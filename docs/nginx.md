# Nginx : les blocs de configuration

Un fichier Nginx est fait de **blocs emboîtés**, comme des boîtes dans des boîtes. Chaque bloc a un rôle et ne peut se trouver qu'à certains endroits.

```
nginx.conf
├── events { }                 ← réglages techniques (nb de connexions)
│
├── http { }                   ← MODE HTTP (comprend les requêtes web)
│   ├── upstream { }           ← un groupe de serveurs de destination
│   └── server { }             ← un « site » : un port (+ un nom de domaine)
│       └── location { }       ← une règle selon l'URL
│
└── stream { }                 ← MODE TCP (fait passer des octets)
    ├── map { }                ← une table de correspondance
    ├── upstream { }           ← un groupe de serveurs de destination
    └── server { }             ← un port d'écoute
        (pas de location : pas d'URL à lire)
```

## Les blocs

| Bloc | Rôle | Exemple |
|---|---|---|
| **`events`** | Réglages internes, obligatoire | `worker_connections 1024;` |
| **`http`** | Contient tout ce qui comprend le web | Transcendance, et le port 80 de `nginx-terminate` |
| **`stream`** | Contient tout ce qui relaie du TCP sans le comprendre | Le Nginx principal, `nginx-relay` |
| **`server`** | « J'écoute sur ce port », dans `http` comme dans `stream` | `listen 443;` |
| **`location`** | « Pour cette URL, fais ça ». **Seulement dans `http`** | `location /api/ { ... }` |
| **`upstream`** | Donne un nom à une destination | `upstream relay { server nginx-relay:4433; }` |
| **`map`** | « Si la valeur est X, alors Y » | SNI `chat.signal.org` → `signal-service` |

## `http` ou `stream` ?

**`http` (couche 7)** : Nginx lit les requêtes (méthode, URL, en-têtes) et peut trier **selon l'URL** avec `location`. Exemple dans [`nginx-terminate/nginx.conf`](../signal/data/nginx-terminate/nginx.conf), sur le port 80 :

```
http {
    server {
        listen 80;
        location /.well-known/acme-challenge/ { → envoyer à certbot }
        location / { → répondre 404 }
    }
}
```

**`stream` (couche 4, TCP)** : Nginx ne comprend pas ce qui passe. Pas d'URL, donc **pas de `location`**. Il dit seulement « ce qui arrive sur tel port → tel serveur ». Avec `ssl_preread`, il peut en plus lire le **SNI** pour choisir la destination.

## L'analogie

- `http` = un **guichet de poste** qui ouvre les lettres, lit l'adresse précise (l'URL) et les trie avec `location`.
- `stream` = un **tapis roulant** qui transporte des colis fermés. Il ne regarde que l'étiquette extérieure (le port, ou le SNI avec `ssl_preread`).
- `server` = le numéro du guichet ou du tapis (le port).
- `map` = le tableau affiché au mur : « étiquette X → tapis Y ».
- `upstream` = l'adresse de livraison.

## Le Nginx principal de proxy-stack

| Port | Rôle | Bloc |
|---|---|---|
| 443 | Lire le SNI, envoyer à Signal, MTProxy ou Xray, **sans déchiffrer** | `stream` + `ssl_preread` + `map` |
| 80 | Tout transmettre à `nginx-terminate:80` (qui trie lui-même le challenge Let's Encrypt) | `stream`, simple relais |

Structure du fichier :

```
events { }        ← obligatoire, même vide

stream {
    map ...       ← SNI → nom du service
    server { écoute 443, ssl_preread, envoie selon le map }
    server { écoute 80, envoie tout à nginx-terminate:80 }
}
```

Pas de bloc `http`, donc pas de `location`. Modèle à suivre : [`nginx-relay/nginx.conf`](../signal/data/nginx-relay/nginx.conf), qui fait le même travail sur le port 443.

## Comparaison : stack web (`http`) contre stack TCP (`stream`)

Même logiciel Nginx, deux usages opposés.

- **App full stack (Transcendance)** : Nginx est le **destinataire final** du TLS. Il ouvre le colis, lit le contenu et le trie.
- **proxy-stack** : Nginx **transporte des colis fermés**. Il ne lit que l'étiquette (le SNI).

### Pourquoi on ne voit pas `http {` dans la config de l'app web

Le fichier de l'app est un **template** : l'image officielle le copie dans `/etc/nginx/conf.d/`, et le `nginx.conf` par défaut l'inclut **à l'intérieur** de son bloc `http { }`. Tout le fichier est donc dans `http`, même si ce n'est pas écrit.

Dans proxy-stack, le `nginx.conf` est monté **à la place** du fichier par défaut : il faut écrire soi-même `events { }` et `stream { }`.

### Ce que fait l'app web, et si c'est possible en `stream`

| Dans l'app web | Ce que ça demande | Possible dans proxy-stack ? |
|---|---|---|
| `ssl_certificate` + `listen ... ssl` | Nginx **déchiffre** le TLS | ❌ REALITY et Signal doivent recevoir le TLS intact |
| `location /api/`, `/socket.io/`, `/grafana`… | Lire **l'URL** | ❌ Elle est chiffrée, Nginx ne la voit pas |
| `return 301 https://...` | Répondre en HTTP | ❌ Ce n'est pas du HTTP |
| `proxy_set_header`, `add_header` (HSTS, CSP…) | Modifier les **en-têtes HTTP** | ❌ Aucun en-tête lisible |
| `log_format` avec `$request`, `$status`, `$http_user_agent` | Lire la requête et la réponse | ❌ Seulement IP, port, SNI, octets, durée |
| WAF (ModSecurity) | Inspecter le contenu HTTP | ❌ Rien à inspecter |
| `upstream` + `server ... resolve` | Résoudre le nom du conteneur | ✅ Fonctionne aussi en `stream` |

### Vue d'ensemble

| | Stack web (`http`) | Stack TCP (`stream`) |
|---|---|---|
| Couche | 7 (application) | 4 (transport) |
| TLS | **Terminé** par Nginx (il a le certificat) | **Passthrough** (Nginx n'a pas de certificat) |
| Tri du trafic | Par URL (`location`) | Par port, ou par SNI (`ssl_preread` + `map`) |
| En-têtes de sécurité | Oui (HSTS, CSP, X-Frame-Options…) | Non |
| WAF | Oui | Non |
| Limitation | `limit_req` (requêtes) et `limit_conn` | `limit_conn` seulement (connexions) |
| Logs | Requête, code HTTP, navigateur… | IP, SNI, octets, durée de la session |
| Config | Template dans `conf.d/`, inclus dans `http` | `nginx.conf` complet avec `events` + `stream` |

### Ce qui se reprend de la stack web

**`resolve` sur les upstreams.** Quand un conteneur est recréé, il change d'IP et Nginx garde l'ancienne, ce qui provoque des erreurs. L'image (Nginx 1.30) gère `zone` + `server ... resolve` aussi dans `stream`.

Différence : dans l'app web, le `resolver` était fourni par le système de templates de l'image. Avec un `nginx.conf` monté directement, il faut l'écrire soi-même dans le bloc `stream` :

```
resolver 127.0.0.11 valid=5s;    # 127.0.0.11 = DNS interne de Docker
```

**`log_format`**, avec les variables de `stream` : `$remote_addr`, `$ssl_preread_server_name`, `$upstream_addr`, `$bytes_sent`, `$session_time`.

⚠️ **Vie privée :** les utilisateurs du proxy comptent sur la discrétion du service. Éviter d'enregistrer les IP, ou désactiver les logs comme le fait Signal (`access_log off`).
