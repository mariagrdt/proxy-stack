# Roadmap — Migration vers une stack docker-compose unique

Objectif : passer des trois services installés à la main (systemd, Docker, compilation) à **un seul `docker-compose.yml`**, avec **Nginx en point d'entrée unique** qui route par SNI vers les conteneurs.

```
client ──443──▶ Nginx (stream + ssl_preread)
                  ├─ signal.<domaine>        → conteneur Signal
                  ├─ <domaine fake-TLS>      → conteneur MTProxy
                  └─ tout le reste (default) → conteneur Xray (REALITY)
```

Chaque étape se termine par un **✅ critère de fin** : ne passe pas à la suivante tant qu'il n'est pas atteint.

---

## Étape 0 — Faire l'inventaire de l'existant (sur le VPS)

Avant d'écrire quoi que ce soit, note comment ça tourne **aujourd'hui**. Les nouvelles configs partiront de là.

- [ ] Qui écoute sur quels ports : `sudo ss -tlnp`
- [ ] **Xray** : où est sa config (`systemctl cat xray` te montre le chemin) ? Note l'UUID, la clé privée REALITY, le `dest` et les `serverNames`.
- [ ] **Signal** : quel dossier, quel `docker-compose`, quel domaine, où sont les certificats ?
- [ ] **MTProxy** : quel binaire, quel secret, quel port ? Le secret commence-t-il par `ee` (fake-TLS) ?
- [ ] Les domaines et sous-domaines utilisés, et vers quoi pointe le DNS
- [ ] **Sauvegarde** de toutes ces configs **hors du dépôt Git** (elles contiennent des secrets)

✅ Tu as une fiche par service : port, domaine, secrets, chemin de config.

---

## Étape 1 — Organiser le dépôt

- [✅] Renommer `mtpproxy/` → `mtproxy/` (petite faute de frappe dans le nom actuel)
- [✅] Créer `nginx/` (il contiendra `nginx.conf`)
- [ ] Lister dans `.env_example` **les noms** de toutes les variables secrètes, avec des valeurs bidon
- [ ] Vérifier que `.gitignore` exclut bien `.env`, les certificats et toute config qui contient des secrets

> 💡 **Piège Xray :** son `config.json` contient l'UUID et la clé privée, et Xray **ne lit pas** les variables d'environnement dans ce fichier. Deux options :
> - committer un `config.json.example` sans secrets et garder le vrai `config.json` dans `.gitignore` (le plus simple pour commencer) ;
> - générer le fichier au démarrage avec `envsubst` à partir d'un template (plus propre, à garder pour plus tard).

✅ L'arborescence est en place et `git status` ne montre aucun secret.

---

## Étape 2 — Le squelette `docker-compose.yml`

- [ ] Déclarer un **réseau interne** (section `networks:` en bas du fichier)
- [ ] Service `nginx` : image officielle, `restart: unless-stopped`, ports `80` et `443` publiés, branché sur le réseau
- [ ] Un service vide par proxy (`xray`, `mtproxy`, `signal`), branché sur le réseau, **sans `ports:`**
- [ ] Vérifier la syntaxe : `docker compose config`

✅ `docker compose config` affiche le fichier sans erreur.

---

## Étape 3 — Xray

- [ ] Choisir l'image (cherche l'image officielle dans le dépôt GitHub `XTLS/Xray-core`)
- [ ] Monter `./xray/config.json` dans le conteneur, en lecture seule (`:ro`). Regarde dans la doc de l'image à quel chemin Xray cherche sa config.
- [ ] Reprendre ta config actuelle et vérifier le port d'écoute de l'inbound. C'est ce port que Nginx visera sur le réseau interne.

> 💡 **Point clé :** avec REALITY, le client n'envoie **pas** ton domaine dans le SNI. Il envoie le domaine de camouflage (celui de `serverNames`, par exemple un grand site connu). Nginx ne peut donc pas reconnaître Xray par ton domaine : Xray sera la **route par défaut** de Nginx (tout ce qui ne correspond ni à Signal ni à MTProxy).

✅ Le conteneur démarre et `docker compose logs xray` ne montre pas d'erreur.

---

## Étape 4 — Nginx en mode `stream`

- [ ] Écrire `nginx/nginx.conf` avec un bloc `stream { }`. Dans ce mode, **pas** de bloc `http` pour le 443.
- [ ] Activer `ssl_preread on;` pour que Nginx lise le SNI
- [ ] Faire une table `map $ssl_preread_server_name ...` : chaque domaine → un service, et `default` → xray
- [ ] Le `proxy_pass` utilise le résultat du `map`. Les conteneurs se joignent **par leur nom de service** (`xray:443`, etc.).
- [ ] Ajouter `limit_conn` (nombre de connexions simultanées par IP)
- [ ] Monter `nginx.conf` dans le conteneur, en lecture seule
- [ ] Tester la config : `docker compose exec nginx nginx -t`

> 💡 **IP des clients :** derrière Nginx, les conteneurs voient l'IP de Nginx, plus celle du vrai client. Ce n'est pas bloquant pour démarrer. Plus tard, regarde `proxy_protocol` (Nginx l'envoie, Xray sait le lire) si tu veux de vraies IP dans les logs ou pour fail2ban.

✅ Avec seulement Xray derrière Nginx, un client Xray se connecte et navigue.

---

## Étape 5 — MTProxy

- [ ] Choisir l'image : l'officielle `telegrammessenger/proxy`, ou **mtg** (`nineseconds/mtg`, dépôt `9seconds/mtg`), très active et pensée pour le fake-TLS. Compare les deux.
- [ ] Utiliser un secret **fake-TLS** (qui commence par `ee`). Le domaine encodé dans ce secret est le SNI que Nginx verra.
- [ ] Ajouter ce domaine dans le `map` de Nginx → `mtproxy`
- [ ] Mettre le secret dans `.env`

✅ Telegram se connecte via le proxy.

---

## Étape 6 — Signal + Certbot

- [ ] Lire le dépôt officiel `signalapp/Signal-TLS-Proxy`. Il a son propre `docker-compose` avec plusieurs services (un Nginx qui gère le TLS, un relais, certbot).
- [ ] Reprendre ces services dans **ton** compose, **sans** leurs `ports:` (c'est ton Nginx qui reçoit le 443)
- [ ] Ajouter `signal.<domaine>` dans le `map` de Nginx → le service Signal qui gère le TLS
- [ ] **Certbot** : le challenge HTTP-01 passe par le port **80**. Ton Nginx doit donc écouter sur 80 (bloc `http`) et servir le dossier du challenge, ou transmettre le port 80 à Certbot.
- [ ] Un **volume partagé** pour les certificats, entre Certbot (qui écrit) et Signal (qui lit)
- [ ] Prévoir le renouvellement automatique (Certbot qui tourne en boucle, ou une tâche planifiée)

✅ Signal se connecte via le proxy et `certbot renew --dry-run` réussit.

---

## Étape 7 — Bascule en production

C'est l'étape risquée : 5 personnes utilisent le service tous les jours.

- [ ] Choisir un créneau calme et prévenir les utilisateurs
- [ ] Écrire le **plan de retour arrière** *avant* de commencer (comment relancer les anciens services)
- [ ] Arrêter les anciens services (`systemctl stop xray`, ancien compose de Signal, MTProxy)
- [ ] `docker compose up -d` dans `~/proxy-stack`
- [ ] Tester les trois services depuis un vrai téléphone
- [ ] Une fois que tout marche : `systemctl disable` les anciens services pour qu'ils ne redémarrent pas au reboot
- [ ] Vérifier UFW : seuls 80, 443 et ton port SSH sont ouverts

✅ Les trois services marchent via la nouvelle stack, et l'ancienne installation est désactivée.

---

## Étape 8 — CI/CD

- [ ] Ajouter une étape de **validation** avant le déploiement dans [`.github/workflows/deploy.yml`](../.github/workflows/deploy.yml) : `docker compose config` (avec un `.env` factice construit à partir de `.env_example`)
- [ ] Décommenter `docker compose up -d` dans la commande SSH
- [ ] Faire un petit changement, pousser, et vérifier que le déploiement passe

✅ Un `git push` sur `main` valide puis déploie la stack automatiquement.

---

## Étape 9 — Durcissement des conteneurs

- [ ] `security_opt: [no-new-privileges:true]` sur chaque service
- [ ] `cap_drop: [ALL]`, puis ne rajouter que les capacités nécessaires
- [ ] `read_only: true` quand c'est possible
- [ ] Fixer les versions des images (pas de `:latest`)
- [ ] Limiter la taille des logs Docker (`logging:` avec `max-size`)

✅ La stack tourne avec ces restrictions, sans régression.

---

## Plus tard (déjà dans le README)

- Observabilité : Prometheus + Grafana. Grafana sera la seule partie HTTP ; une WAF y aurait du sens.
- Logs centralisés (Loki, plus léger qu'ELK pour un petit VPS)
- Secrets : Docker secrets, puis éventuellement Vault
