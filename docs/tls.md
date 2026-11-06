# Comment le certificat permet de déchiffrer (TLS, version simple)

## Les deux fichiers

| Fichier | C'est quoi | Public ou secret |
|---|---|---|
| `fullchain.pem` | Le **certificat** : une carte d'identité « ce serveur est bien `signal.ton-domaine.fr` », signée par Let's Encrypt | Public |
| `privkey.pem` | La **clé privée** qui va avec le certificat | **Secret**, ne quitte jamais le serveur |

🔐 Le certificat, c'est le **nom sur la boîte aux lettres**. La clé privée, c'est **la clé de la boîte**.

## La connexion en 4 étapes

```
1. Téléphone → serveur : « je veux parler à signal.ton-domaine.fr »
2. Serveur → téléphone : « voici mon certificat »
3. Le téléphone vérifie le certificat ; le serveur prouve avec sa clé privée qu'il en est bien le propriétaire
4. Les deux créent une clé de session secrète → tout est chiffré avec
```

Un imposteur peut copier le certificat (il est public), mais **pas** la clé privée : il échoue à l'étape 3.

## Dans proxy-stack

- **`nginx-terminate`** a le certificat + la clé privée → il peut ouvrir **TLS #1**.
- Il n'a pas la clé de Signal → il ne peut **pas** ouvrir **TLS #2**.
- **Le Nginx principal** n'a aucun certificat → il n'ouvre rien, il lit seulement l'adresse (le SNI).

⚠️ C'est pour ça que `*.pem` et `signal/data/certbot/` sont dans le `.gitignore` : quelqu'un qui a la clé privée peut se faire passer pour le serveur.
