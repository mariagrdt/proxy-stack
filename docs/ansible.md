# Ansible : configuration du VPS

Le VPS est configuré **uniquement par Ansible**. Plus aucune modification à la main : chaque changement passe par le playbook, puis est commité dans Git.

**Pourquoi :**
- **Reconstruire** un serveur identique en une commande si le VPS meurt
- **Documenter** : le playbook décrit exactement l'état du serveur
- **Modifier proprement** : chaque changement est dans l'historique Git
- **Tester avant** : `--check --diff` montre ce qui va changer
- **Dupliquer** : un 2ᵉ VPS = une ligne de plus dans l'inventaire

## Fichiers

```
ansible/
├── inventory.ini.example   ← modèle, commité
├── inventory.ini           ← vrai VPS (IP, port, clé), dans le .gitignore
└── playbook.yml            ← les tâches
```

### L'inventaire

Il dit à Ansible **sur quelle machine** travailler et **comment s'y connecter** :

```ini
[vps]
YOUR_VPS_IP ansible_user=ubuntu ansible_port=YOUR_SSH_PORT ansible_ssh_private_key_file=~/.ssh/YOUR_PRIVATE_KEY
```

| Élément | Rôle |
|---|---|
| `[vps]` | Le groupe ciblé par le playbook (`hosts: vps`) |
| `ansible_user` | L'utilisateur SSH (`ubuntu` sur l'image OVH) |
| `ansible_port` | Le port SSH personnalisé |
| `ansible_ssh_private_key_file` | Le chemin de la **clé privée** (sur l'ordinateur, jamais dans le dépôt) |

Ces valeurs sont aussi dans `~/.ssh/config` : c'est pour ça qu'un simple `ssh <IP>` fonctionne sans préciser le port ni la clé.

**Principe :** le code (playbook) est public, les valeurs sensibles (IP, port) restent dans `inventory.ini`, ignoré par Git. Le playbook y accède avec `{{ ansible_port }}`.

## Ce que fait le playbook

| Section | Tâches | Pourquoi |
|---|---|---|
| **Système** | Mise à jour de la liste des paquets | Prérequis aux installations |
| **Docker** | `docker.io` + `docker-compose-v2` (paquets Ubuntu) ; `ubuntu` ajouté au groupe `docker` | Lancer la stack sans `sudo` |
| **UFW** | Ouvre le port SSH (`{{ ansible_port }}`), 80 et 443, **puis** active le pare-feu | Tout le reste est refusé par défaut |
| **fail2ban** | Installation ; `/etc/fail2ban/jail.local` ; service démarré et activé au boot ; handler de redémarrage | Bannit les IP qui échouent trop souvent en SSH |

### Les pièges rencontrés

**UFW et le port SSH.** Ouvrir le port 22 alors que SSH écoute sur un port personnalisé = **se bloquer l'accès au serveur** dès l'activation d'UFW. Toujours ouvrir le port SSH **avant** `state: enabled`, avec `{{ ansible_port }}`.

**fail2ban et le port SSH.** Par défaut, fail2ban bannit sur le port 22. Avec un port personnalisé, il faut l'indiquer (`port = {{ ansible_port }}`), sinon l'attaquant banni continue sur le vrai port.

**`jail.local` et pas `jail.conf`.** `jail.conf` appartient au paquet et peut être écrasé à une mise à jour. `jail.local` contient nos réglages et il est prioritaire.

**Handler.** `notify: Restart fail2ban` ne redémarre le service **que si** `jail.local` a changé. Le nom du `notify` doit être identique à celui du handler, et `handlers:` est aligné comme `tasks:`.

**`--check` et le groupe `docker`.** En simulation, l'installation de Docker n'a pas vraiment lieu, donc le groupe `docker` n'existe pas, et la tâche suivante échoue. Ce n'est pas un bug du playbook : il passe en exécution réelle.

**Docker contourne UFW.** Les ports publiés par Docker passent avant les règles UFW. Seul le Nginx principal doit avoir des `ports:` dans le compose.

## Commandes

```bash
# Tester la connexion
ansible -i ansible/inventory.ini all -m ping

# Vérifier la syntaxe
ansible-playbook -i ansible/inventory.ini ansible/playbook.yml --syntax-check

# Simuler (ne change rien)
ansible-playbook -i ansible/inventory.ini ansible/playbook.yml --check --diff

# Appliquer
ansible-playbook -i ansible/inventory.ini ansible/playbook.yml
```

Après une exécution réelle, relancer : le `PLAY RECAP` doit afficher **`changed=0`** (idempotence). Pour la tâche « Update apt cache », `cache_valid_time: 3600` évite qu'elle compte comme un changement à chaque fois.

### Vérifier sur le serveur

```bash
docker --version && docker compose version
id ubuntu                          # doit contenir "docker"
sudo ufw status
sudo fail2ban-client status sshd   # IP bannies
```

## État au 7 novembre 2026

| Élément | État |
|---|---|
| Docker 29.1 + Compose 2.40 | ✅ |
| `ubuntu` dans le groupe `docker` | ✅ |
| UFW actif : SSH, 80, 443 | ✅ |
| fail2ban actif sur SSH | ✅ |
| Règle `22/tcp DENY` dans UFW | Reste de l'ancienne config manuelle, sans effet gênant |

## À faire

- [ ] `cache_valid_time: 3600` sur « Update apt cache » (idempotence)
- [ ] Mise à jour du système (`upgrade: dist`) et `unattended-upgrades`
- [ ] Durcissement SSH : clé uniquement, `root` interdit
- [ ] Utilisateur `deploy` dédié, avec sa clé SSH
- [ ] Déploiement de la stack (clone du dépôt, `.env` depuis Ansible Vault, `docker compose up -d`)
- [ ] Découper le playbook en rôles quand il grossira
