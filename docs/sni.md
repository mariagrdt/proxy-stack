# Le SNI et le trajet du trafic, expliqués simplement

## 1. Le problème

Le serveur a **un seul port 443**, mais **trois services** veulent l'utiliser : Signal, Telegram et Xray. Il faut quelqu'un à l'entrée qui envoie chaque connexion au bon service. C'est le rôle de Nginx.

Mais comment Nginx sait-il à quel service une connexion est destinée ? Grâce au **SNI**.

## 2. C'est quoi le SNI ?

Toute connexion chiffrée (TLS, le « S » de HTTPS) commence par un petit message de bonjour. Dans ce message, le client écrit **en clair** le nom du site qu'il veut joindre :

```
« Bonjour, je veux parler à signal.mondomaine.fr »
                            └──────── le SNI ────────┘
```

**SNI** = *Server Name Indication*, « indication du nom du serveur ». Tout le reste de la connexion est chiffré. Seul ce nom est lisible.

📮 **L'analogie de la lettre**
- La connexion est une **lettre dans une enveloppe fermée**.
- Le SNI est **l'adresse écrite sur l'enveloppe**.
- Tout le monde peut lire l'adresse. Personne ne peut lire la lettre sans l'ouvrir.

## 3. Ce que fait Nginx

Nginx est le **gardien de l'immeuble** :

1. Il **lit l'adresse** sur l'enveloppe (le SNI). C'est la directive `ssl_preread on;`.
2. Il **n'ouvre pas** l'enveloppe. C'est ce qu'on appelle le *TLS passthrough*.
3. Il **apporte** l'enveloppe au bon appartement (le bon conteneur). C'est le `map` et le `proxy_pass`.

```
         enveloppe fermée, adresse = signal.mondomaine.fr
client ─────────────────────────────────────────────▶ Nginx
                                                        │ lit l'adresse
                                                        │ n'ouvre pas
                                                        ▼
                                                   conteneur Signal
```

## 4. Quelle adresse pour quel service ?

| Service | Adresse (SNI) écrite par le client | Où Nginx l'envoie |
|---|---|---|
| **Signal** | Mon sous-domaine, ex. `signal.mondomaine.fr` | Signal (`nginx-terminate`) |
| **Telegram** | Le domaine caché dans le secret MTProxy (qui commence par `ee`) | MTProxy |
| **Xray** | Un **faux** domaine, celui d'un grand site connu (camouflage) | Xray |
| **N'importe quoi d'autre** | Robots, curieux… | Xray aussi (route par défaut) |

**Pourquoi Xray a un faux domaine ?** Pour se cacher. Quelqu'un qui surveille le réseau croit voir une visite vers un site connu.

**Pourquoi Xray reçoit aussi « tout le reste » ?** Si un robot vient sonder le serveur, Xray le renvoie vers le vrai site connu. Le robot voit un site normal, et le serveur ne se fait pas repérer.

## 5. Le cas de Signal : deux enveloppes

Pour Signal, la lettre est dans **deux enveloppes** :

```
┌─ enveloppe extérieure : adresse = signal.mondomaine.fr ──────┐
│   ┌─ enveloppe intérieure : adresse = chat.signal.org ─────┐ │
│   │                 message Signal (chiffré)               │ │
│   └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

- **De l'extérieur**, on ne voit que `signal.mondomaine.fr` : personne ne sait que c'est du Signal. C'est ce qui contourne la censure.
- **`nginx-terminate`** ouvre l'enveloppe extérieure, grâce à mon certificat.
- **`nginx-relay`** lit l'adresse de l'enveloppe intérieure (`chat.signal.org`) et l'envoie chez Signal. Il ne peut pas l'ouvrir.

Résultat : le serveur ne peut **jamais** lire les messages.

## 6. Et le port 80 ?

Il ne sert **qu'à une chose** : prouver à Let's Encrypt que le domaine m'appartient, pour obtenir et renouveler le certificat de Signal.

**Comment Let's Encrypt vérifie :**
1. Certbot demande un certificat pour `signal.mondomaine.fr`.
2. Let's Encrypt répond : « Prouve-le : mets ce code à cette adresse » :
   `http://signal.mondomaine.fr/.well-known/acme-challenge/<code>`
3. Certbot met le code à disposition.
4. Let's Encrypt vient vérifier. Le code est là ? ✅ Il délivre le certificat.

🏠 C'est comme : « Si c'est ta maison, colle ce papier sur ta porte, je passe vérifier. »

| Mot | Signification |
|---|---|
| **ACME** | Le nom du protocole de Let's Encrypt. `acme-challenge` = « l'épreuve » : le code à montrer. |
| **`.well-known`** | Un dossier standard, réservé aux fichiers techniques que des services extérieurs viennent lire |

**Le trajet sur le port 80 :**
```
Let's Encrypt ──80──▶ Nginx ──▶ nginx-terminate ──▶ certbot
                     (ne trie     (adresse de vérification → certbot,
                      rien)        toute autre adresse → 404)
```

Il n'y a pas de site web sur ce serveur, donc rien à rediriger vers HTTPS.

## 7. Le voir de ses propres yeux

Pour **voir** le SNI, cette commande ouvre une connexion TLS en choisissant l'adresse écrite sur l'enveloppe :

```bash
openssl s_client -connect <IP du serveur>:443 -servername signal.mondomaine.fr
```

- `-connect` : où on se connecte (l'immeuble)
- `-servername` : le SNI (l'adresse sur l'enveloppe)

En changeant uniquement `-servername`, on arrive sur un service différent, **avec la même IP et le même port**. C'est exactement le tri que fait Nginx.

Pour aller plus loin : capturer une connexion avec **Wireshark** et chercher le champ `server_name` dans le message « Client Hello ». On y voit le SNI passer en clair.

## Résumé en 4 phrases

1. Le **SNI** est l'adresse écrite en clair sur une connexion chiffrée.
2. **Nginx** lit cette adresse sans ouvrir la connexion, et l'envoie au bon service.
3. **Signal** utilise deux enveloppes pour cacher qu'on parle à Signal.
4. Le **port 80** sert seulement à Let's Encrypt, pour vérifier le domaine.
