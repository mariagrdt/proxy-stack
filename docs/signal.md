# Signal TLS Proxy

Notes sur le `docker-compose.yml` du dépôt officiel [signalapp/Signal-TLS-Proxy](https://github.com/signalapp/Signal-TLS-Proxy), et sur la façon de l'intégrer dans proxy-stack.

## Le principe

Le téléphone ouvre une connexion TLS vers `signal.ton-domaine`. **À l'intérieur** de cette connexion, il y a une deuxième connexion TLS, chiffrée directement vers les serveurs de Signal :

```
Téléphone ══TLS #1══▶ nginx-terminate ──▶ nginx-relay ══TLS #2 (intact)══▶ serveurs Signal
                      (enlève TLS #1)     (lit le SNI de TLS #2,
                                           envoie au bon serveur Signal)
```

Le serveur enlève la première couche, mais la deuxième reste fermée : **il est impossible de lire les messages**. Le proxy ne fait que relayer.

## Les 3 services

### 1. `nginx-terminate` : celui qui déchiffre (TLS #1)

- **`image: nginx:stable-alpine@sha256:...`** : Nginx en version légère. Le `@sha256:...` fige l'image exacte : même si une nouvelle version est publiée sous le même tag, c'est toujours celle-là qui est utilisée. C'est une bonne pratique de sécurité.
- **`volumes`** :
  - `./data/nginx-terminate` → sa config Nginx
  - `./data/certbot/conf` → les certificats Let's Encrypt (partagés avec `certbot`)
  - `./data/certbot/www` → prévu pour le challenge HTTP-01, mais **pas utilisé** dans la config actuelle : Certbot tourne en mode `--standalone`, et `nginx-terminate` transmet `/.well-known/acme-challenge/` directement à `certbot:80`
- **`ports: 443 et 80`** : c'est lui qui reçoit le trafic venant de l'extérieur. Le 443 sert pour Signal, le 80 pour la validation Let's Encrypt.
- **`command`** fait deux choses en même temps :
  - une boucle en arrière-plan qui attend 6 h puis lance `nginx -s reload`, pour que Nginx prenne en compte le certificat après son renouvellement, sans redémarrer ;
  - le lancement de Nginx au premier plan (`daemon off;`), obligatoire dans un conteneur, sinon il s'arrête aussitôt.

### 2. `nginx-relay` : l'aiguilleur vers Signal

- **Pas de `ports:`** : il n'est joignable que par `nginx-terminate`, sur le réseau interne.
- Il lit le SNI de la connexion TLS #2 (`ssl_preread`) et l'envoie vers le bon serveur Signal.
- **`${NGINX_RELAY_CONFIG:-nginx.conf}`** : il utilise la variable `NGINX_RELAY_CONFIG` si elle existe, sinon `nginx.conf` par défaut. Ça permet de choisir une autre config sans modifier le fichier.
- Il a la même boucle de reload toutes les 6 h.

### 3. `certbot` : le renouvellement des certificats

- Il partage les deux dossiers `certbot/` avec `nginx-terminate`.
- **`entrypoint`** : une boucle infinie qui lance `certbot renew` toutes les 12 h. Certbot ne renouvelle vraiment que si le certificat expire bientôt, donc le lancer souvent ne pose pas de problème.
- **`trap exit TERM`** : quand Docker demande au conteneur de s'arrêter (`docker compose down`), il s'arrête proprement au lieu d'attendre 10 s puis d'être tué de force.

## Les astuces shell

- **`$${!}`** : dans un compose, `$` sert à lire des variables. On écrit `$$` pour obtenir un vrai `$`. Une fois traduit, ça donne `${!}` en shell, c'est-à-dire l'identifiant du dernier processus lancé en arrière-plan.
- **`sleep 6h & wait $!`** au lieu de `sleep 6h` : le conteneur peut réagir à un signal d'arrêt pendant qu'il attend. Avec un simple `sleep`, il ne répondrait qu'au bout des 6 h.

## Intégration dans proxy-stack

**Le problème :** `nginx-terminate` prend les ports 443 et 80, que le Nginx principal de la stack doit avoir.

Ce qu'il faut changer :

1. Reprendre les 3 services, mais **sans les `ports:`** de `nginx-terminate`.
2. Le Nginx principal (`stream`) envoie `signal.ton-domaine` → `nginx-terminate:443`.
3. **Port 80** (challenge Let's Encrypt) : le Nginx principal transmet tout le port 80 à `nginx-terminate:80`, qui sait déjà répondre au challenge. C'est un simple relais TCP, sans SNI à lire.
4. Les chemins des volumes changent : `./data/...` devient `./signal/data/...`, pour que tout reste dans le dossier `signal/`.

Le reste (la boucle de reload, Certbot, le relay) peut rester **tel quel**.

## Les fichiers du dépôt

| Fichier | Rôle | Pour proxy-stack |
|---|---|---|
| `docker-compose.yml` | Les 3 services | Recopier les blocs dans le compose principal, sans les `ports:` |
| `data/nginx-terminate/nginx.conf` | Bloc `http` sur le port 80 (envoie le challenge à `certbot:80`) + bloc `stream` sur le 443 qui déchiffre TLS #1 et transmet à `nginx-relay:4433` | **À garder** → `signal/data/nginx-terminate/nginx.conf` |
| `data/nginx-relay/nginx.conf` | `map` du SNI → serveurs Signal (`chat.signal.org`, `cdn.signal.org`…), `default` → `deny` | **À garder tel quel** → `signal/data/nginx-relay/nginx.conf` |
| `data/nginx-relay/nginx-staging.conf` | Même chose vers les serveurs de test de Signal | Inutile (sert aux développeurs de Signal) |
| `init-certificate.sh` | Obtient le **premier** certificat (Certbot en `--standalone` sur le port 80), puis crée le lien `/etc/letsencrypt/active` | Inutile si on récupère les certificats existants du VPS (voir plus bas) |
| `LICENSE` | Licence MIT | **À copier tel quel** → `signal/LICENSE`. La licence MIT exige que la notice de copyright accompagne toute copie ou partie substantielle du code, ce qui est le cas des configs reprises. |
| `README.md` | Doc du dépôt | Ne pas copier. Citer la source dans le README de proxy-stack. |
| `.gitignore` | Ignore `/data/certbot` | Ajouter `signal/data/certbot/` au `.gitignore` de proxy-stack (certificats et clé privée) |
| `.github/` | Dependabot pour les images | Inutile |

### Les certificats

Le VPS a déjà un certificat valide, obtenu avec `init-certificate.sh`. Le plus simple : copier le dossier `data/certbot/` de l'installation actuelle vers `~/proxy-stack/signal/data/certbot/` sur le VPS. Les **liens symboliques** (`active` → `live/<domaine>`) doivent être conservés : utiliser `cp -a` ou `rsync -a`.

Si un jour il faut relancer `init-certificate.sh`, attention : il publie le port 80 sur le conteneur `certbot`, ce qui entre en conflit avec le Nginx principal. Il faut alors arrêter le Nginx principal le temps du script.

## Différences : installation d'origine (VPS) → proxy-stack

**Référence :** dépôt officiel au commit `6442976` (2 novembre 2026). L'installation actuelle sur le VPS a été clonée plus tôt et peut avoir quelques différences avec ce commit (voir « Vérifier sur le VPS » plus bas).

### Vue d'ensemble

```
AVANT (VPS)                                   APRÈS (proxy-stack)

Internet                                      Internet
   │ 443 / 80                                    │ 443 / 80
   ▼                                             ▼
nginx-terminate  (publie 443 et 80)           Nginx principal  (seul à publier 443 et 80)
   │                                             │ SNI signal.<domaine> / tout le port 80
   ▼                                             ▼
nginx-relay                                   nginx-terminate  (plus de ports publiés)
   │                                             │
   ▼                                             ▼
serveurs Signal                               nginx-relay
                                                 │
                                                 ▼
                                              serveurs Signal
```

Avant, Signal occupait seul les ports 443 et 80 du serveur. Maintenant, il partage le 443 avec MTProxy et Xray grâce au Nginx principal.

### `docker-compose.yml`

| Élément | Avant (original) | Après (proxy-stack) | Pourquoi |
|---|---|---|---|
| `ports:` de `nginx-terminate` | `443:443` et `80:80` | **Supprimés** | Seul le Nginx principal publie des ports. `nginx-terminate` n'est joignable que par le réseau interne. |
| Chemins des volumes | `./data/...` | `./signal/data/...` | Tout ce qui concerne Signal reste dans le dossier `signal/`. |
| Config de `nginx-relay` | `${NGINX_RELAY_CONFIG:-nginx.conf}` | `nginx.conf` | La config de staging n'est pas copiée : la variable n'avait plus qu'une valeur possible. |
| Emplacement | `docker-compose.yml` propre à Signal | Section `# CONFIG SIGNAL` du compose unique | Un seul fichier pour toute la stack. |
| Images (`@sha256`) | Identiques | Identiques | Aucune raison de changer. |
| `command` / `entrypoint` | Identiques | Identiques | La boucle de reload et le renouvellement Certbot fonctionnent tels quels. |
| Volume `certbot/www` | Présent | Présent | Inutilisé en mode `--standalone`, gardé pour rester proche de l'original. |

### Configs Nginx

| Fichier | Changement |
|---|---|
| `nginx-terminate/nginx.conf` | **Aucun**, copie exacte |
| `nginx-relay/nginx.conf` | **Aucun**, copie exacte |

Les noms `nginx-relay:4433` et `certbot:80` utilisés dans ces configs restent valables : les noms de services sont les mêmes, et tous les services sont sur le même réseau Docker.

### Fichiers du dépôt

| Fichier | Avant | Après |
|---|---|---|
| `LICENSE` | À la racine du dépôt Signal | `signal/LICENSE` (obligatoire, licence MIT) |
| `nginx-relay/nginx-staging.conf` | Présent | Non copié |
| `init-certificate.sh` | Utilisé pour obtenir le premier certificat | Non copié : les certificats existants sont réutilisés |
| `README.md`, `.github/` | Présents | Non copiés |
| `.gitignore` | `/data/certbot` | `signal/data/certbot/` dans le `.gitignore` de proxy-stack |

### Certificats

| | Avant | Après |
|---|---|---|
| Emplacement | `<dossier Signal>/data/certbot/` | `~/proxy-stack/signal/data/certbot/` |
| Obtention | `init-certificate.sh` | Copiés depuis l'ancienne installation avec `cp -a` (garde le lien `active`) |
| Renouvellement | Certbot du compose Signal | Le même conteneur Certbot, maintenant dans la stack |
| Challenge HTTP-01 (port 80) | Arrive directement sur `nginx-terminate` | Le Nginx principal transmet le port 80 à `nginx-terminate:80`, qui l'envoie à `certbot:80` |

### Vérifier sur le VPS

Pour comparer l'installation réelle avec cette référence, dans le dossier Signal du VPS :

```bash
git log -1 --format='%h %cs'     # commit installé
git status                       # fichiers modifiés à la main
git diff                         # détail des modifications locales
git fetch && git diff HEAD origin/main --stat   # écart avec la version actuelle
```

Si `git diff` montre des modifications locales (domaine, config), il faut les reporter dans `signal/`.

## Prochaine étape

Lire le `nginx.conf` de `nginx-relay` : c'est un vrai exemple de `ssl_preread` + `map`, le même mécanisme que celui à écrire pour le Nginx principal.
