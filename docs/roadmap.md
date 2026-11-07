# Roadmap — Nouvelle architecture proxy-stack

**Situation de départ :** l'ancien VPS est sauvegardé (snapshot OVH). Le nouveau VPS est **vide**. On construit tout de zéro, de façon reproductible :

- **Ansible** configure le serveur (utilisateur, SSH, pare-feu, fail2ban, Docker) ;
- **docker-compose** décrit les services ;
- **Nginx** est le point d'entrée unique et trie le trafic par SNI.

```
client ──443──▶ Nginx (stream + ssl_preread)
                  ├─ signal.<domaine>        → nginx-terminate (Signal)
                  ├─ <domaine fake-TLS>      → MTProxy
                  └─ tout le reste (default) → Xray (REALITY)
client ──80───▶ Nginx → nginx-terminate:80 → certbot (certificat Signal)
```

Chaque étape se termine par un **✅ critère de fin** : ne passe pas à la suivante tant qu'il n'est pas atteint.

> ⚠️ Si ta famille n'a plus de proxy pendant les travaux, vise d'abord une **première version qui marche** (étapes 1 → 6), puis améliore.

---

## Étape 1 — Organiser le dépôt

- [x] Renommer `mtpproxy/` → `mtproxy/`
- [x] Créer `nginx/`
- [x] Créer `ansible/`
- [ ] Lister dans `.env_example` **les noms** des variables secrètes, avec des valeurs bidon
- [ ] Vérifier que `.gitignore` exclut `.env`, les certificats, `ansible/inventory.ini` et toute config qui contient des secrets

✅ L'arborescence est en place et `git status` ne montre aucun secret.

---

## Étape 2 — Ansible : préparer le VPS vide

Le VPS est vide : si une erreur te bloque l'accès, tu peux le réinstaller sans rien perdre. C'est le bon moment pour tester Ansible directement dessus.

Structure de départ (un seul playbook, les rôles viendront plus tard) :

```
ansible/
├── inventory.ini.example   ← modèle, commité
├── inventory.ini           ← vrai VPS, dans le .gitignore
└── playbook.yml            ← sections commentées, dans l'ordre ci-dessous
```

- [x] `inventory.ini` : IP, port SSH, utilisateur du VPS
- [x] Vérifier la connexion : `ansible -i inventory.ini all -m ping`
- [ ] **Paquets** : mises à jour, paquets de base, `unattended-upgrades`
- [ ] **Utilisateur** : un utilisateur dédié avec ta clé SSH et `sudo`
- [x] **Docker** : Docker + le plugin compose (reprendre ton projet AWS)
- [ ] **SSH** : clé uniquement, `root` interdit, port personnalisé
- [x] **UFW** : tout refuser par défaut ; ouvrir le port SSH, 80 et 443
- [x] **fail2ban** : protection SSH

Détails et pièges rencontrés : [ansible.md](ansible.md).

> 💡 Avant chaque nouvelle section : `ansible-playbook -i inventory.ini playbook.yml --check --diff` (simulation). Garde une session SSH ouverte quand tu appliques les sections SSH et UFW.
>
> 💡 Docker contourne UFW pour les ports publiés : seul Nginx doit avoir des `ports:` dans le compose.

✅ Le playbook s'exécute deux fois de suite sans erreur, et la 2ᵉ fois il n'affiche **aucun** `changed` (idempotence).

---

## Étape 3 — Décider : anciens ou nouveaux secrets ?

- **Récupérer** depuis le snapshot (UUID Xray, clé REALITY, secret MTProxy) → rien à changer sur les téléphones.
- **Générer de nouveaux** → plus propre, mais il faudra redonner les liens à tout le monde.

⚠️ MTProxy doit utiliser un secret **fake-TLS (`ee…`)**, sinon Nginx ne peut pas le trier par SNI. Si l'ancien secret n'est pas `ee`, il faut en créer un nouveau de toute façon.

- [ ] Décision prise pour Xray
- [ ] Décision prise pour MTProxy
- [ ] Domaines choisis (Signal, MTProxy) et SNI de camouflage Xray (`serverNames`)

✅ Une fiche (hors Git) avec les domaines, les ports et où sont stockés les secrets.

---

## Étape 4 — Compléter le `docker-compose.yml`

### Déjà fait
- [x] Signal : `nginx-terminate`, `nginx-relay`, `certbot` (sans `ports:`, volumes dans `signal/data/`)
- [x] Nginx principal : ports 80 et 443, `nginx.conf` monté en lecture seule, `depends_on`
- [x] `nginx.conf` v1 : tout le 80 et le 443 vers `nginx-terminate`

### Xray
- [ ] Choisir l'image (onglet Packages du dépôt `XTLS/Xray-core`) et l'épingler par digest
- [ ] Monter `./xray/config.json` en lecture seule (`config.json` dans le `.gitignore`, `config.json.example` commité)
- [ ] Noter le port de l'inbound : c'est la destination pour Nginx

> 💡 Le SNI de Xray n'est **pas** ton domaine : c'est le domaine de camouflage (`serverNames`). Xray sera la route **`default`** de Nginx.

### MTProxy
- [ ] Choisir l'image : `telegrammessenger/proxy` ou `nineseconds/mtg` (fake-TLS)
- [ ] Secret `ee…` dans `.env`
- [ ] Noter le port interne

### Nginx : tri par SNI
- [ ] Ajouter `ssl_preread on;` dans le `server` du 443
- [ ] Ajouter un `map $ssl_preread_server_name` : Signal → `nginx-terminate:443`, MTProxy → son service, `default` → Xray
- [ ] Ajouter `resolver 127.0.0.11;` dans `stream` (obligatoire dès que `proxy_pass` utilise une variable)
- [ ] Ajouter `xray` et `mtproxy` dans le `depends_on` de `nginx`

✅ `docker compose config` ne signale aucune erreur.

---

## Étape 5 — Premier démarrage sur le VPS

- [ ] **DNS** : faire pointer les domaines vers l'IP du nouveau VPS
- [ ] Cloner le dépôt sur le VPS et créer le `.env` et `xray/config.json`
- [ ] **Premier certificat Signal** : lancer `init-certificate.sh` du dépôt Signal **avant** de démarrer le Nginx principal (il a besoin du port 80 libre). Seulement la première fois.
- [ ] `docker compose up -d`
- [ ] `docker compose ps` : tous les services sont `running`
- [ ] Tester depuis un vrai téléphone : Signal, Telegram, Xray

✅ Les trois proxies fonctionnent depuis un téléphone.

---

## Étape 6 — Donner l'accès à la famille

- [ ] Envoyer les liens (Signal : `https://signal.tube/#<domaine>`, Telegram : lien `tg://proxy…`, Xray : lien VLESS)
- [ ] Vérifier avec chaque personne que ça marche

✅ Les 5 utilisateurs sont reconnectés.

---

## Étape 7 — Ansible : déployer la stack

- [ ] Section **proxy_stack** dans le playbook : cloner le dépôt, déposer `.env` et `xray/config.json` depuis **Ansible Vault**, `docker compose up -d`
- [ ] Tester sur un serveur jetable (une instance EC2 créée par ton projet Terraform) : un VPS vide doit devenir une stack complète avec une seule commande

✅ `ansible-playbook` reconstruit un serveur complet à partir de zéro (hors certificat initial).

---

## Étape 8 — CI/CD + DevSecOps

- [ ] Validation avant déploiement : `docker compose config` (avec un `.env` factice construit à partir de `.env_example`)
- [ ] **gitleaks** : bloque un secret commité par erreur
- [ ] **Trivy** : scanne les images Docker (CVE)
- [ ] **ansible-lint** sur le playbook
- [ ] Décommenter `docker compose up -d` dans la commande SSH, puis réactiver le workflow
- [ ] **Dependabot** pour mettre à jour les digests des images

✅ Un `git push` sur `main` vérifie, scanne puis déploie.

---

## Étape 9 — Durcissement des conteneurs

- [ ] `security_opt: [no-new-privileges:true]` sur chaque service
- [ ] `cap_drop: [ALL]`, puis ne rajouter que les capacités nécessaires
- [ ] `read_only: true` quand c'est possible
- [ ] Toutes les images épinglées par digest
- [ ] Limiter la taille des logs Docker (`logging:` avec `max-size`)

✅ La stack tourne avec ces restrictions, sans régression.

---

## Étape 10 — Observabilité (avec le projet AWS)

- [ ] `node_exporter` + `cAdvisor` dans le compose, **sans `ports:`**
- [ ] Un agent (Grafana Alloy) qui **envoie** les métriques (`remote_write`) vers le Prometheus de l'EC2
- [ ] Côté AWS : Security Group ouvert **uniquement** à l'IP du VPS, géré dans Terraform
- [ ] Seulement des métriques globales : **pas d'IP de clients**

✅ Un tableau de bord Grafana affiche l'état du VPS et des conteneurs.

---

## Plus tard

- Rôles Ansible (découper le playbook)
- Secrets : Docker secrets, puis éventuellement Vault
- AIOps : détection d'anomalies sur les métriques
- `resolve` sur les upstreams Nginx, `proxy_protocol` pour Xray
