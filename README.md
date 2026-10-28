# tout-pris-docker

Déploiement de la stack **Tout Pris** :

- l'API [`tout-pris-api`](https://github.com/AxineTeam/tout-pris-api)
- le front [`tout-pris-front`](https://github.com/AxineTeam/tout-pris-front)

Ce dépôt ne contient aucun code applicatif, seulement de quoi faire tourner les images publiées.

|                       |                                                                    |
| --------------------- | ------------------------------------------------------------------ |
| `docker-compose.yaml` | les trois conteneurs : `api`, `front`, `memcached`                 |
| `extra/nginx/`        | le vhost à installer sur le nginx de l'hôte                        |
| `extra/backup/`       | le script de sauvegarde, son timer systemd, sa rétention logrotate |

## Architecture

```
internet ──▶ RP frontal (hors de ce dépôt) ──▶ nginx de l'hôte
                                                 ├── /          ──▶ 127.0.0.1:8180  front (SPA)
                                                 ├── /api/      ─┐
                                                 ├── /admin/    ─┤
                                                 ├── /accounts/ ─┼▶ 127.0.0.1:8100  api (Django)
                                                 └── /static/   ─┘
```

Le reverse proxy est le nginx de l'hôte, pas un conteneur, et il n'est lui-même joignable qu'à travers le RP frontal — c'est le pare-feu qui le garantit. Les deux ports des conteneurs sont publiés sur la loopback, donc inaccessibles autrement que par nginx.

Front et API sont servis depuis la même origine : le SPA appelle `/api/<chemin>`, nginx transmet tel quel, Django monte ses routes sous `/api/`. Pas de CORS, pas d'URL d'API à configurer, cookies de session et de CSRF d'un même domaine. Django sert aussi `/admin/`, `/accounts/` (callbacks OAuth) et `/static/` (par WhiteNoise, depuis le processus) ; tout le reste va au front.

La base est un fichier SQLite dans `data/`, monté en bind mount et non en volume nommé pour que les scripts de l'hôte le lisent directement.

## Installation

Les chemins ci-dessous supposent le dépôt dans `/srv/tout-pris` ; ailleurs, deux fichiers sont à ajuster — `WorkingDirectory`/`ExecStart` dans `extra/backup/tout-pris-backup.service` et le chemin dans `extra/backup/logrotate-tout-pris`.

```sh
sudo git clone https://github.com/AxineTeam/tout-pris-docker /srv/tout-pris
cd /srv/tout-pris
sudo mkdir -p data backups
sudo chown 999:999 data   # l'utilisateur non-root de l'image api
```

Puis le `.env`, que git ignore :

```sh
cp .env.example .env
sudo chmod 600 .env
```

| Variable                            | Rôle                                                                                                                                  |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `DJANGO_SECRET_KEY`                 | Signe sessions et jetons e-mail. `openssl rand -base64 48`. La changer déconnecte tout le monde et invalide les liens en circulation. |
| `DJANGO_ALLOWED_HOSTS`              | Le domaine public ; Django répond 400 à tout autre `Host`. Le compose y ajoute `127.0.0.1` pour son healthcheck.                      |
| `FRONTEND_URL`                      | L'URL publique du front, vers laquelle pointent les liens des e-mails.                                                                |
| `BREVO_API_KEY`                     | La clé Brevo, par où partent les e-mails transactionnels.                                                                             |
| `MAIL_FROM_EMAIL`, `MAIL_FROM_NAME` | L'expéditeur, optionnels : le compose a des valeurs par défaut. L'adresse doit être validée dans Brevo.                               |

Les quatre premières n'ont pas de défaut : sans elles `docker compose up` s'arrête en nommant celle qui manque. `DJANGO_DEBUG` est fixé à `false` dans le compose et n'a rien à faire dans le `.env`. La liste complète des variables lues par l'image est dans le [README de l'API](https://github.com/AxineTeam/tout-pris-api#configuration).

```sh
sudo docker compose up -d
sudo docker compose ps
```

Les migrations Django sont appliquées par l'entrypoint de l'image à chaque démarrage.

Enfin le vhost :

```sh
sudo cp extra/nginx/tout-pris.conf /etc/nginx/sites-available/tout-pris
sudo sed -i 's/tout-pris.example.com/VOTRE-DOMAINE/' /etc/nginx/sites-available/tout-pris
sudo ln -s /etc/nginx/sites-available/tout-pris /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d VOTRE-DOMAINE   # si c'est ce nginx qui porte le certificat
```

## Les trois réglages non évidents

### `FORWARDED_ALLOW_IPS: "*"`

> TODO: à revoir pas clair

Sans lui, le site entier part en boucle de redirection. L'API ne définit pas `SECURE_PROXY_SSL_HEADER` : c'est le serveur qui traduit le `X-Forwarded-Proto` posé par le vhost, et dans cette image c'est le `ProxyHeadersMiddleware` d'uvicorn qui le fait, pour les seules adresses de confiance de gunicorn — `127.0.0.1` par défaut. Or nginx entre par la passerelle du réseau Docker : l'en-tête est ignoré, Django se croit en clair, et comme `SECURE_SSL_REDIRECT` est actif chaque requête repart en 301 vers nginx, qui la repasse à Django.

Tout croire est sans risque ici : le port n'étant publié que sur la loopback, personne n'atteint l'API autrement qu'à travers nginx, qui réécrit l'en-tête à chaque requête.

### `WEB_CONCURRENCY` et `memcached`

Le nombre de processus n'est pas dans l'image : gunicorn lit `WEB_CONCURRENCY`, donc c'est le compose qui dimensionne.

Au-delà d'un seul processus, `MEMCACHED_LOCATION` devient obligatoire : le cache de Django est privé à chaque processus, django-allauth y tient les compteurs de ses limitations, et sans cache partagé chaque worker les applique pour lui seul. Django refuse de démarrer en multithread sans un cache.

Deux vérifications pour s'assurer que ça fonctionne bien :

```sh
# renvoie plusieurs ligne, une master gunicorn + une par WEB_CONCURRENCY workers
sudo docker top tout-pris-api
# si ça renvoie des 0 c'est que le cache ne marche pas
sudo docker compose exec memcached sh -c 'echo stats | nc -w 1 localhost 11211' | grep -E 'cmd_get|cmd_set'
```

### Limitation de débit

> TODO: à revoir allauth est sensé recevoir les IP clientes justement, mais je confirme que ça n'a pas l'air de marcher
> TODO: il n'y a rien dans les logs d'ailleurs pour s'en assurer

Django ne voit pas l'adresse du client. `REMOTE_ADDR` lui vient de `X-Forwarded-For`, que le RP frontal ajoute _derrière_ ce que l'appelant a envoyé, et uvicorn — à qui l'on dit de tout croire — retient la première valeur, donc celle de l'appelant. Les limitations par IP de django-allauth (`signup`, `login`, `reset_password_from_key`) ne protègent par conséquent de rien : la limitation de débit a sa place sur le RP frontal, le seul à voir l'adresse réelle. Celles portées par l'identifiant, dont le `5/300s/key` de `login_failed`, tiennent dans tous les cas.

## Compression des réponses de l'API

Le bloc `location /api/` du vhost compresse le JSON : sans ses quatre lignes aucune réponse ne l'est, le `gzip_types` par défaut de nginx ne couvrant que `text/html` et `gzip_proxied` lui interdisant les réponses d'un _upstream_.

En compressant, nginx affaiblit l'`ETag` (`"abc"` devient `W/"abc"`), que le navigateur renvoie tel quel dans `If-None-Match`. L'API doit donc comparer les empreintes faiblement, sinon elle répond `200` avec le corps entier là où elle répondait `304` — sans rien signaler. Vérification :

```sh
curl -sS -o /dev/null -D - -H 'Accept-Encoding: gzip' -b cookies.txt \
  https://VOTRE-DOMAINE/api/households/1/trips/1/items/
```

`Content-Encoding: gzip` doit apparaître, et une seconde requête portant l'`ETag` reçu doit répondre `304`.

## Sauvegarde

> TODO: Jamais vérifiée sur la machine de déploiement : voir [#12](https://github.com/AxineTeam/tout-pris-docker/issues/12).

`extra/backup/backup.sh` écrit toujours le même fichier, `backups/tout_pris.sqlite.gz` ; c'est logrotate qui le date et en garde 14. Le script passe par `sqlite3 .backup`, l'API de sauvegarde en ligne de SQLite, qui produit un fichier cohérent pendant que l'API écrit — un `cp` du `.db` ne garantit rien, les dernières transactions vivant dans le `-wal`. Chaque archive est relue avec `PRAGMA integrity_check` avant d'être renommée, donc la précédente survit à un échec.

```sh
sudo apt install sqlite3
sudo cp extra/backup/tout-pris-backup.service extra/backup/tout-pris-backup.timer /etc/systemd/system/
sudo cp extra/backup/logrotate-tout-pris /etc/logrotate.d/tout-pris
sudo systemctl daemon-reload
sudo systemctl enable --now tout-pris-backup.timer

sudo systemctl start tout-pris-backup.service   # déclenche une sauvegarde
ls -lh backups/
sudo logrotate -d /etc/logrotate.d/tout-pris    # simulation
```

Le timer tourne à 03:00 et `logrotate.timer` vers 00:00 : si vous changez l'heure, gardez-la après celle de logrotate, pour que la rotation date le fichier de la veille avant l'écriture du jour.

Restent à votre charge le hors-site — les archives sont sur le même disque que la base, un `restic` dans un second timer est le complément minimal — et l'alerting, `OnFailure=` dans l'unité étant le point d'accroche prévu.

## Restaurer la base

```sh
cd /srv/tout-pris
ls -lh backups/
sudo docker compose stop api

gzip -dc backups/tout_pris.sqlite.gz-20260817 > /tmp/restore.sqlite
sqlite3 /tmp/restore.sqlite 'PRAGMA integrity_check;'   # ne continuez que sur « ok »

sudo mkdir -p backups/avant-restauration
sudo cp -a data/tout_pris.db* backups/avant-restauration/

sudo rm -f data/tout_pris.db data/tout_pris.db-wal data/tout_pris.db-shm
sudo cp /tmp/restore.sqlite data/tout_pris.db
sudo chown 999:999 data/tout_pris.db

sudo docker compose start api
curl -fsS https://VOTRE-DOMAINE/api/health/
```

On valide l'archive avant de toucher à la base, et on met la base courante de côté : elle peut contenir des écritures plus récentes. Les `-wal`/`-shm` résiduels sont à supprimer, sinon SQLite rejoue au démarrage un journal qui ne correspond plus au fichier restauré ; sans le `chown`, l'API démarre mais échoue à la première écriture.

> Testez cette procédure une fois **avant** d'en avoir besoin. C'est le seul moyen de savoir que les archives sont exploitables.

## Exploitation

```sh
sudo docker compose pull && sudo docker compose up -d   # mise à jour
sudo docker compose logs -f api
sudo docker compose exec api python manage.py createsuperuser
```

L'admin Django est exposé sur `/admin/`, protégé par le seul mot de passe de ce compte ; le restreindre davantage se fait dans le bloc `location /admin/` du vhost. La rotation des logs Docker relève du démon, via `log-opts` dans `/etc/docker/daemon.json`.

Un service `watchtower` est fourni commenté en fin de compose pour automatiser les mises à jour : `api` et `front` portent déjà le label qu'il attend. Deux réserves avant de l'activer — le socket Docker vaut un accès root sur l'hôte, et une nouvelle image peut embarquer une migration appliquée sans supervision.

Pour travailler sur le code, ce dépôt n'est pas le bon : utilisez les `docker-compose.yml` des dépôts applicatifs, qui montent les sources et rechargent à chaud.
