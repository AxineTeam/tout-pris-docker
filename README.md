# tout-pris-docker

Déploiement de la stack **Tout Pris** : l'API
[`tout-pris-api`](https://github.com/AxineTeam/tout-pris-api) (Django, servie
en ASGI par gunicorn supervisant des workers uvicorn) et le front
[`tout-pris-front`](https://github.com/AxineTeam/tout-pris-front) (SvelteKit
statique).

Ce dépôt ne contient pas de code applicatif :

| | |
| --- | --- |
| `docker-compose.yaml` | les deux conteneurs applicatifs |
| `extra/nginx/` | le vhost à installer sur le nginx de l'hôte |
| `extra/backup/` | le script de sauvegarde, son timer systemd et sa rétention logrotate |

## Architecture

```
  internet ──▶ nginx (hôte, 80/443)
                 ├── /          ──▶ 127.0.0.1:8180  conteneur front (SPA)
                 ├── /api/      ─┐
                 ├── /admin/    ─┤
                 ├── /accounts/ ─┼▶ 127.0.0.1:8100  conteneur api (Django)
                 └── /static/   ─┘
                                                        │
                                                        ▼
                                            /srv/tout-pris/data/tout_pris.db
                                                        ▲
                    systemd timer ──▶ backup.sh ──▶ /srv/tout-pris/backups/
```

Deux conteneurs seulement. **Le reverse proxy est le nginx de l'hôte**, pas un
conteneur : le TLS, les autres vhosts et les certificats restent gérés là où ils
le sont déjà. Les deux ports sont publiés sur la loopback, donc inaccessibles
depuis l'extérieur autrement que par nginx.

Le front et l'API sont servis **depuis la même origine** : le SPA appelle
`/api/<chemin>` et nginx transmet le chemin tel quel à Django, qui monte
lui-même ses routes sous `/api/`. Pas de CORS, pas d'URL d'API à configurer dans
le front, et les cookies de session et de CSRF d'un même domaine.

Django ne se contente pas de `/api/` : le vhost route aussi `/admin/` (l'admin
Django), `/accounts/` (les callbacks OAuth des fournisseurs externes) et
`/static/` (les fichiers statiques, servis par WhiteNoise depuis le processus
applicatif, pas depuis le disque de l'hôte). Tout le reste va au front.

La base est un **SQLite** — un fichier dans `data/`, pas de serveur de base.
C'est un bind mount et non un volume nommé, pour que les scripts de l'hôte
puissent le lire directement.

## Installation

La suite suppose le dépôt déployé dans `/srv/tout-pris`. Si vous le mettez
ailleurs, deux fichiers sont à ajuster : `WorkingDirectory`/`ExecStart` dans
`extra/backup/tout-pris-backup.service`, et le chemin dans
`extra/backup/logrotate-tout-pris`. Le script de sauvegarde, lui, travaille en
chemins relatifs et n'a rien à changer.

```sh
sudo git clone https://github.com/AxineTeam/tout-pris-docker /srv/tout-pris
cd /srv/tout-pris

# Le conteneur api tourne en 999:999 (utilisateur non-root de son image) et
# doit pouvoir écrire dans le bind mount.
sudo mkdir -p data backups
sudo chown 999:999 data
```

### Configuration

L'API Django lit sa configuration dans l'environnement. `docker compose`
la prend dans un fichier `.env` à la racine du dépôt, que git ignore :

```sh
sudo tee /srv/tout-pris/.env > /dev/null <<'EOF'
DJANGO_SECRET_KEY=REMPLACEZ-MOI
DJANGO_ALLOWED_HOSTS=VOTRE-DOMAINE
FRONTEND_URL=https://VOTRE-DOMAINE
BREVO_API_KEY=REMPLACEZ-MOI
MAIL_FROM_EMAIL=no-reply@tout-pris.app
MAIL_FROM_NAME=Tout Pris
EOF
sudo chmod 600 /srv/tout-pris/.env
```

| Variable | Rôle |
| --- | --- |
| `DJANGO_SECRET_KEY` | Signe les sessions et les jetons envoyés par e-mail. `openssl rand -base64 48` en produit une. La changer déconnecte tout le monde et invalide les liens de vérification en circulation. |
| `DJANGO_ALLOWED_HOSTS` | Le domaine public. Django répond 400 à toute requête portant un autre `Host`. Le compose y ajoute `127.0.0.1` pour son propre *healthcheck*. |
| `FRONTEND_URL` | L'URL publique du front : c'est vers elle que pointent les liens des e-mails de vérification d'adresse et de mot de passe oublié. |
| `BREVO_API_KEY` | La clé Brevo, par où partent les e-mails transactionnels. Sans elle, Django refuse de démarrer avec `DJANGO_DEBUG=false` plutôt que d'écrire les e-mails dans les logs. |
| `MAIL_FROM_EMAIL` | L'expéditeur, qui doit être une adresse validée dans Brevo. |
| `MAIL_FROM_NAME` | Le nom affiché de l'expéditeur. |

Les quatre premières n'ont pas de valeur par défaut : sans elles, `docker
compose up` s'arrête en nommant celle qui manque plutôt que de démarrer une API
mal configurée. `DJANGO_DEBUG` est fixé à `false` dans le compose et n'a rien à
faire dans le `.env` — c'est lui qui active les cookies `Secure`, la
redirection HTTPS et le HSTS.

La liste complète des variables lues par l'image est dans le
[README de l'API](https://github.com/AxineTeam/tout-pris-api#configuration).

### Démarrage

```sh
sudo docker compose up -d
sudo docker compose ps
```

Les migrations Django sont appliquées par l'entrypoint de l'image à chaque
démarrage du conteneur.

### nginx

```sh
sudo cp extra/nginx/tout-pris.conf /etc/nginx/sites-available/tout-pris
sudo sed -i 's/tout-pris.example.com/VOTRE-DOMAINE/' /etc/nginx/sites-available/tout-pris
sudo ln -s /etc/nginx/sites-available/tout-pris /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Pour le HTTPS, `certbot` écrit lui-même le bloc 443 et la redirection :

```sh
sudo certbot --nginx -d VOTRE-DOMAINE
```

### Compression des réponses de l'API

Le bloc `location /api/` du vhost compresse les réponses JSON. Sans ses quatre
lignes, aucune ne l'est : le `gzip_types` par défaut de nginx ne couvre que
`text/html`, et `gzip_proxied` lui interdit par défaut de compresser quoi que ce
soit qui vienne d'un *upstream*.

**Ce réglage exige une API qui compare les `ETag` faiblement**, c'est-à-dire une
image qui embarque au moins
[tout-pris-api#94](https://github.com/AxineTeam/tout-pris-api/pull/94). Le
déployer avant elle casse la revalidation en silence.

Dès qu'il compresse, nginx n'a plus le droit de promettre que la réponse est
identique octet pour octet à celle de l'origine : il affaiblit l'empreinte,
`ETag: "abc"` devient `ETag: W/"abc"`. Le navigateur renvoie cette forme
affaiblie dans `If-None-Match` au sondage suivant, et une API qui compare les
empreintes comme des chaînes brutes ne la reconnaît plus : elle répond `200`
avec le corps entier là où elle répondait `304`. Rien n'échoue, rien
n'apparaît dans les logs — on aurait activé la compression et désactivé la
revalidation dans le même geste.

Après déploiement, `Content-Encoding: gzip` doit apparaître sur une réponse JSON
de plus d'un kilooctet — la liste des lignes d'un voyage, que le front sonde
toutes les trois secondes, en est une :

```sh
curl -sS -o /dev/null -D - -H 'Accept-Encoding: gzip' -b cookies.txt \
  https://VOTRE-DOMAINE/api/households/1/trips/1/items/
```

Et une seconde requête portant l'`ETag` reçu doit répondre `304`, pas `200` :
c'est elle qui vérifie la dépendance ci-dessus.

### Sauvegardes

Trois fichiers, un par responsabilité : le script sauvegarde, le timer
déclenche, logrotate gère la rétention.

```sh
sudo apt install sqlite3

sudo cp extra/backup/tout-pris-backup.service extra/backup/tout-pris-backup.timer /etc/systemd/system/
sudo cp extra/backup/logrotate-tout-pris /etc/logrotate.d/tout-pris

sudo systemctl daemon-reload
sudo systemctl enable --now tout-pris-backup.timer
```

Vérifier :

```sh
sudo systemctl start tout-pris-backup.service   # déclenche une sauvegarde
sudo systemctl status tout-pris-backup.service
sudo systemctl list-timers tout-pris-backup.timer
ls -lh backups/
sudo logrotate -d /etc/logrotate.d/tout-pris    # simulation, sans rien écrire
```

Hors systemd, le script se lance directement depuis la racine du dépôt :

```sh
sudo ./extra/backup/backup.sh
```

## Fonctionnement du compose

`docker-compose.yaml` ne décrit que les deux conteneurs applicatifs, et ne porte
aucun commentaire : ce qu'il faut savoir pour le relire est ici.

Il ne construit aucune image et ne dicte aucune commande. L'image publiée
`estb/tout-pris-api` démarre gunicorn d'elle-même, après avoir appliqué les
migrations Django dans son entrypoint — d'où l'absence de `build`, de `command`
et de toute étape de migration. Le compose ne fait que la configurer, lui donner
un volume et publier son port.

Les deux ports sont publiés sur `127.0.0.1` et non sur toutes les interfaces :
`8100` pour l'API (`8000` dans le conteneur) et `8180` pour le front (`80`).
Seul le nginx de l'hôte peut donc les atteindre, et ce sont ces deux numéros que
reprennent les `upstream` du vhost. La base, elle, est montée en `./data`
plutôt que dans un volume nommé, pour que `backup.sh` lise le fichier
directement depuis l'hôte ; le dossier doit appartenir à `999:999`,
l'utilisateur non-root de l'image.

Le service `watchtower` commenté en fin de fichier est décrit plus bas, dans
[Mise à jour](#mise-à-jour).

### `FORWARDED_ALLOW_IPS`

C'est la variable la moins évidente du fichier, et celle sans laquelle le site
entier part en boucle de redirection.

L'API ne définit pas `SECURE_PROXY_SSL_HEADER` : c'est donc le serveur qui
décide si Django se croit en HTTPS, en traduisant l'en-tête `X-Forwarded-Proto`
que pose le vhost. L'image sert l'application en ASGI — gunicorn supervise, et
c'est uvicorn qui parle le protocole : la traduction est faite par le
`ProxyHeadersMiddleware` d'uvicorn, qui lit la liste des adresses de confiance
dans la configuration de gunicorn, `127.0.0.1` par défaut. Les requêtes de
nginx entrent dans le conteneur par la passerelle du réseau Docker et non par
la loopback : l'en-tête est donc ignoré, et Django se croit en clair derrière le
TLS. Comme `SECURE_SSL_REDIRECT` est actif dès que `DJANGO_DEBUG` vaut `false`,
chaque requête repart alors en 301 vers `https`, c'est-à-dire vers nginx, qui la
repasse à Django, qui la redirige encore : une boucle de redirection sur la
totalité du site.

`FORWARDED_ALLOW_IPS: "*"` lève la restriction. Le port n'étant publié que sur
la loopback de l'hôte, aucun client extérieur ne peut atteindre l'API
autrement qu'à travers nginx, qui réécrit `X-Forwarded-Proto` à chaque requête :
personne n'est en position de mentir sur le protocole.

### Limitation de débit

Django ne voit pas l'adresse du client. `REMOTE_ADDR` lui vient de
`X-Forwarded-For`, que le RP frontal ajoute *derrière* ce que l'appelant a
envoyé, et uvicorn — à qui `FORWARDED_ALLOW_IPS: "*"` dit de tout croire —
retient la première valeur de la liste, donc celle que l'appelant a posée
lui-même. Les limitations par IP de django-allauth (`signup`, `login`,
`reset_password_from_key`) ne protègent par conséquent de rien : la limitation
de débit a sa place sur le RP frontal, le seul à voir l'adresse réelle. Celles
portées par l'identifiant, dont le `5/300s/key` de `login_failed`, tiennent
dans tous les cas.

### Le healthcheck de l'API

Il appelle `/api/health/` — et non `/health`, qui était le chemin du temps de
FastAPI — en se déclarant lui-même en HTTPS, faute de quoi la redirection
ci-dessus l'enverrait vers `https://127.0.0.1/`, où rien n'écoute : il
échouerait à chaque passage et le conteneur resterait indéfiniment `unhealthy`.

C'est aussi pour lui que le compose ajoute `127.0.0.1` à
`DJANGO_ALLOWED_HOSTS`, plutôt que de le laisser au `.env` : la requête part
avec `Host: 127.0.0.1`, que Django rejetterait en 400 s'il ne figurait pas dans
la liste.

## Fonctionnement des sauvegardes

`extra/backup/backup.sh` produit `backups/tout_pris.sqlite.gz` — **toujours le
même nom**. C'est logrotate qui le date et décide combien de copies conserver
(`rotate 14` par défaut), plutôt qu'une logique de rétention dans le script.

Le script travaille en chemins relatifs et attend d'être lancé depuis la racine
du dépôt ; c'est `WorkingDirectory` dans l'unité systemd qui le garantit.

Le script utilise `sqlite3 .backup`, c'est-à-dire l'**API de sauvegarde en
ligne** de SQLite : elle produit un fichier cohérent pendant que l'API continue
d'écrire. Un `cp` du `.db` ne garantit rien — les dernières transactions vivent
dans le `-wal`, qui n'est pas copié atomiquement avec le fichier principal, et
rien ne signale le problème au moment de la copie. Chaque sauvegarde est relue
avec `PRAGMA integrity_check` et n'est renommée sous son nom définitif qu'en cas
de succès : la sauvegarde précédente survit donc à un échec.

Le timer tourne à 03:00 et `logrotate.timer` vers 00:00. L'ordre compte : la
rotation date le fichier de la veille avant que celui du jour ne soit écrit. Si
vous changez l'heure du timer, gardez-la après celle de logrotate.

Deux points restent à votre charge :

- **Le hors-site.** Les sauvegardes sont sur le même disque que la base, ce qui
  ne protège que de l'erreur humaine. Un `rclone`/`restic` vers un stockage
  distant, dans un second timer, est le complément minimal.
- **L'alerting.** `OnFailure=` dans `tout-pris-backup.service` est le point
  d'accroche prévu : sans lui, une sauvegarde qui échoue le fait en silence.

## Restaurer la base

Procédure manuelle. Chaque étape est vérifiable avant de passer à la suivante,
et rien n'est écrasé avant que l'archive n'ait été validée.

**1. Choisir l'archive.**

```sh
cd /srv/tout-pris
ls -lh backups/
# tout_pris.sqlite.gz            <- la plus récente
# tout_pris.sqlite.gz-20260817   <- datées par logrotate
```

**2. Arrêter l'API**, pour que plus personne n'écrive dans la base. nginx peut
rester en place, il renverra des 502 le temps de l'opération.

```sh
sudo docker compose stop api
```

**3. Décompresser et vérifier l'archive.** On travaille sur une copie, et on
valide *avant* de toucher à la base : une restauration qui échoue après avoir
écrasé la base en place est une double panne.

```sh
gzip -dc backups/tout_pris.sqlite.gz-20260817 > /tmp/restore.sqlite
sqlite3 /tmp/restore.sqlite 'PRAGMA integrity_check;'   # doit répondre : ok
```

Ne continuez que si la réponse est exactement `ok`.

**4. Mettre la base courante de côté.** Même une base qu'on croit perdue peut
contenir des écritures plus récentes que l'archive. L'API étant arrêtée, les
trois fichiers forment un ensemble cohérent et une copie simple suffit.

```sh
sudo mkdir -p backups/avant-restauration
sudo cp -a data/tout_pris.db* backups/avant-restauration/
```

**5. Installer l'archive.** Les `-wal` et `-shm` résiduels appartiennent à
l'ancienne base : les laisser ferait rejouer à SQLite un journal qui ne
correspond plus au fichier restauré. Le `chown` rétablit l'utilisateur non-root
de l'image, sans lequel l'API redémarre sur une base qu'elle ne peut pas écrire.

```sh
sudo rm -f data/tout_pris.db data/tout_pris.db-wal data/tout_pris.db-shm
sudo cp /tmp/restore.sqlite data/tout_pris.db
sudo chown 999:999 data/tout_pris.db
sudo chmod 644 data/tout_pris.db
```

**6. Relancer et vérifier.**

```sh
sudo docker compose start api
sudo docker compose logs -f api        # les migrations Django se rejouent ici
curl -fsS https://VOTRE-DOMAINE/api/health/
curl -sS -o /dev/null -w '%{http_code}\n' https://VOTRE-DOMAINE/api/households/   # 401 : l'API répond et exige une session
```

Une fois la vérification faite, `rm /tmp/restore.sqlite`.

### Ce qui peut mal se passer

Points de vigilance, sans solution toute faite — chacun demande une décision au
cas par cas :

- **L'archive est corrompue** : l'`integrity_check` de l'étape 3 ne répond pas
  `ok`. Le cas est attrapé avant tout écrasement, mais il faut alors une autre
  archive.
- **La base courante est elle-même illisible** au moment de l'étape 4.
- **Le disque est plein** pendant la décompression : `/tmp/restore.sqlite` est
  tronqué, et l'`integrity_check` peut passer sur un fichier incomplet.
- **Un `docker compose up -d` est lancé pendant l'opération** : l'`api` est
  recréée et se remet à écrire au milieu de la restauration.
- **Les `-wal`/`-shm` de l'étape 5 sont oubliés** : SQLite rejoue un journal
  orphelin au démarrage.
- **Le `chown` est oublié** : l'API démarre mais échoue à la première écriture.
- **L'archive vient d'une version plus récente de l'API** : les migrations
  Django manquantes sont appliquées vers l'avant, jamais vers l'arrière.

> Testez cette procédure au moins une fois **avant** d'en avoir besoin. C'est le
> seul moyen de savoir que vos archives sont exploitables.

## Mise à jour

```sh
sudo docker compose pull
sudo docker compose up -d
```

Les migrations Django sont appliquées automatiquement au démarrage de l'API,
par l'entrypoint de l'image.

Pour que ce soit automatique, un service `watchtower` est fourni **commenté** en
fin de `docker-compose.yaml` : `api` et `front` portent déjà le label
`com.centurylinklabs.watchtower.enable`, et `WATCHTOWER_LABEL_ENABLE` restreint
Watchtower à eux seuls. Deux points avant de l'activer : le socket Docker donne
au conteneur un accès équivalent à root sur l'hôte, et une nouvelle image de
l'API peut embarquer une migration appliquée sans supervision au redémarrage.

## Exploitation

```sh
sudo docker compose ps
sudo docker compose logs -f api
sudo docker compose down

# L'admin Django, servi sur https://VOTRE-DOMAINE/admin/, demande un compte
# superutilisateur — il n'en existe aucun au premier démarrage.
sudo docker compose exec api python manage.py createsuperuser
```

L'admin est exposé publiquement, protégé par le seul mot de passe de ce compte.
Le restreindre davantage — filtrage par IP, authentification supplémentaire — se
fait dans le bloc `location /admin/` du vhost.

La rotation des logs Docker n'est pas configurée ici : elle relève du démon, via
`log-opts` dans `/etc/docker/daemon.json`.

## Développement local

Ce dépôt déploie les **images publiées**. Pour travailler sur le code, utilisez
les `docker-compose.yml` de chaque dépôt applicatif, qui montent les sources et
activent le rechargement à chaud.
