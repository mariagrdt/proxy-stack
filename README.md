# Proxy Stack — Censorship-resistant messaging proxies on a Linux VPS

Self-hosted proxies that give my family (5 users) reliable access to **Signal** and **Telegram**, and soon the open web, from a country that blocks them. The first version ran in production for more than 6 months. I am rebuilding it as a reproducible, infrastructure-as-code stack: a blank server becomes the full stack with one command.

**Tech:** Ansible · Ansible Vault · Docker Compose · Nginx (`stream`, SNI routing) · Let's Encrypt · GitHub Actions · UFW · fail2ban · Ubuntu (OVH VPS)

## Status

| Part | State |
|---|---|
| Server built by **Ansible**: packages, firewall, fail2ban, code, secrets, certificate, containers | ✅ Running on the VPS; full rebuild on a blank server still to be tested |
| **Nginx** single entry point, SNI routing on port 443 | ✅ Running |
| **Signal** TLS Proxy | ✅ Running, tested from a phone |
| **Telegram** MTProxy (fake-TLS) | ✅ Running, tested from a phone |
| Secrets in **Ansible Vault** | ✅ No secret or private domain in the repository |
| **CI/CD** (GitHub Actions): validate, then deploy | ✅ Running on every push to `main` |
| **Xray** (VLESS-REALITY) | 🚧 Next |
| v1 (hand-built server) | Archived as an OVH snapshot after 6+ months in production |

---

## Where I started (v1)

I set up each service on its own, following its documentation, as I needed it:

| Service | How it was installed |
|---|---|
| Signal TLS Proxy | Docker Compose (official repository), with its own Nginx and Let's Encrypt |
| Xray (VLESS-REALITY) | systemd service, installed with the official script |
| Telegram MTProxy | Docker container, on a separate port |

```
                    VPS (configured by hand)
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
    Port 443             Port 443             Port 8888
  (one OR the other)  (one OR the other)       (always)
        │                    │                    │
        ▼                    ▼                    ▼
   Signal Proxy         Xray REALITY       Telegram MTProxy
     (Docker)            (systemd)            (Docker)
```

It worked, and it was secured (key-only SSH on a custom port, TLS with Let's Encrypt). But it had real limits:

- **Signal and Xray both needed port 443**, so only one of them could run at a time.
- **Three installation methods** (Docker Compose, systemd, a lone container) to maintain and remember.
- **Nothing was reproducible.** The configuration only existed on the server. To rebuild, I had to take a snapshot and list by hand what was running.

## Architecture (v2)

The new VPS started empty. Everything on it is described in this repository.

```
                                         Phones
                                            │
                                        Port 443
                                            │
                                            ▼
                                          Nginx
                           (routes by domain, never decrypts)
                                            │
           ┌─────────────────────┬──────────┴──────────┬─────────────────────┐
           │                     │                     │                     │
     Signal domain       Camouflage domain      Unknown domain            Port 80
                                                                      (Let's Encrypt)
           │                     │                     │                     │
           ▼                     ▼                     ▼                     ▼
    nginx-terminate           MTProxy            Xray REALITY             certbot
  (removes outer TLS)       (fake-TLS)             (planned)         (renews the cert)
           │                     │                     │
           ▼                     │                     │
      nginx-relay                │                     │
  (Signal hosts only)            │                     │
           │                     │                     │
           ▼                     ▼                     ▼
    Signal servers       Telegram servers          Open web
```

- **Only Nginx is reachable from the Internet.** Everything else stays on the internal Docker network.
- **Until Xray is deployed**, unknown domains go to the Signal proxy.
- **Port 443** carries the proxy traffic. Nginx sorts it by SNI and passes it on still encrypted.
- **Port 80** is used **only** by Let's Encrypt to renew the Signal certificate. Any other request gets a 404: there is no website to redirect to HTTPS.

| Layer | Tool |
|---|---|
| Server configuration | **Ansible** |
| Secrets | **Ansible Vault** (encrypted) → `.env` on the server |
| Services | **Docker Compose**, one file for the whole stack |
| Entry point | **Nginx** (`stream`), SNI routing with TLS passthrough |
| Deployment | **GitHub Actions**: secret scan, config checks, deploy over SSH, post-deploy check |

### How the traffic is routed

The first message of every TLS connection contains, in clear text, the name of the server the client wants to reach: the **SNI**. Nginx reads only that name (`ssl_preread`) and forwards the still-encrypted connection to the right container.

| Service | SNI sent by the client | Destination |
|---|---|---|
| Signal | my own subdomain | `nginx-terminate` |
| Telegram MTProxy | the camouflage domain encoded in the fake-TLS secret | MTProxy container |
| Xray REALITY | its own camouflage domain, **not** mine | default route |

Xray is the **default** route on purpose: when REALITY receives a connection that is not a real client, it forwards it to the real camouflage website. A scanner probing the server sees an ordinary site.

### Why Signal goes through two Nginx containers

A Signal connection is wrapped in **two TLS layers**. The outer layer goes to my domain and hides the fact that the user is talking to Signal: that is what defeats SNI-based blocking. The inner layer goes from the phone straight to Signal's servers.

```
Phone ══TLS #1══▶ nginx-terminate ──▶ nginx-relay ══TLS #2 (untouched)══▶ Signal
                  (my certificate)    (reads the inner SNI,
                                       allows only Signal hosts)
```

The server removes the outer layer only. It can never read messages.

---

## Design decisions

- **`stream`, not `http`, on port 443.** These are encrypted tunnels, not web requests. Nginx must not decrypt them: REALITY and Signal need the original TLS handshake.
- **No WAF, on purpose.** A WAF inspects HTTP requests, and there is no HTTP to inspect here. I used ModSecurity in a previous web project, where it made sense.
- **One entry point.** Only Nginx publishes ports. Docker bypasses UFW for published ports, so a single published service keeps the firewall meaningful.
- **No secret and no private domain in the repository.** The camouflage domain must stay private: knowing it would reveal which connections are Telegram. Secrets and domains live in an Ansible Vault file (AES-256). The compose file and the Nginx config only contain `${VARIABLES}`; the Nginx config is a template that the official image fills at startup (`envsubst`).
- **Images pinned by digest** (`@sha256:…`), so a moved tag can never silently change what runs.
- **No Nginx logs** (`access_log off`, and `error_log /dev/null` on port 443). The users rely on this service for their privacy; without logs, debugging port 443 means turning them back on temporarily.
- **Right-sized.** One VPS doesn't need Kubernetes or Terraform. I use Terraform and Ansible on AWS in another project; here the value is in Ansible and Compose.

## Operations

### Two paths to the server

| I change… | Tool | Triggered by | What happens |
|---|---|---|---|
| The **server** or a **secret** | **Ansible** | Me, after a dry run (`--check --diff`) | The server is brought to the state described in the playbook |
| The **stack's code** (compose file, Nginx template, a new service) | **GitHub Actions** | `git push` to `main` | Checks run first, then the server updates and restarts Nginx (details below) |
| Nothing: the server is **brand new** | **Ansible** | Me | One command builds the whole stack |

The pipeline never touches secrets: it uses the `.env` that Ansible wrote. Code changes are deployed automatically; infrastructure changes stay human-triggered, because a mistake there (firewall, SSH) could lock me out of the server. In a team, I would run them through a pipeline too, with review and approval.

### The pipeline

On every push to `main`, the `deploy` job only starts if `validate` is green.

| Job | Step | What it catches |
|---|---|---|
| `validate` | **gitleaks** on the full history | A secret committed by mistake |
| `validate` | `docker compose config` with fake values | An invalid compose file |
| `validate` | Render the Nginx template with the real compose settings, then `nginx -t` | A broken config, or a template that is not rendered |
| `validate` | **ansible-lint** (not blocking yet) | Mistakes and bad practices in the playbook |
| `deploy` | Over SSH: `git pull --ff-only`, `docker compose up -d`, restart Nginx | Stops if the server copy has diverged from GitHub |
| `deploy` | TLS check of Signal and MTProxy through the SNI routing | A service that no longer answers after the deployment |

Nginx is restarted on every deployment because `docker compose up -d` does not detect changes inside mounted files, and the template is only rendered at startup. It costs a few seconds of downtime, which is acceptable for 5 users. There is no automatic rollback: a failed check alerts me, and I revert with a `git revert`, which goes through the same pipeline.

### What the playbook does

| Step | Detail |
|---|---|
| Packages | Docker and the Compose plugin; the admin user joins the `docker` group |
| Firewall | UFW opens the SSH port, 80 and 443, **then** is enabled (deny by default) |
| fail2ban | Protects SSH, configured for the custom port (otherwise bans would hit port 22) |
| Code | Clones the repository over HTTPS (no key needed on a new server) |
| Secrets | Writes `.env` from the vault: mode `0600`, `no_log` so values never appear in the output |
| TLS | Downloads Certbot's recommended TLS settings; requests the first Signal certificate **only if none exists** (`creates:`) |
| Stack | Starts the containers |

```bash
ansible-playbook -i ansible/inventory.ini ansible/playbook.yml --check --diff --ask-vault-pass   # dry run
ansible-playbook -i ansible/inventory.ini ansible/playbook.yml --ask-vault-pass                  # apply
```

### How secrets flow

**Vault** (encrypted, on my machine) → decrypted **in memory** by Ansible → **`.env`** on the server (git-ignored) → **Docker Compose** substitutes `${…}` → the **Nginx image** fills its config template at startup.

`.env_example` lists the expected variables with fake values; the pipeline uses it to validate the compose file without any real secret.

## Repository layout

```
.
├── docker-compose.yml          # the whole stack
├── nginx/nginx.conf.template   # main entry point (SNI routing), filled at startup
├── signal/                     # Signal TLS Proxy configs (+ upstream MIT license)
├── ansible/                    # playbook, inventory example (vault and inventory are git-ignored)
├── .env_example                # variables expected by the stack, with fake values
└── .github/workflows/          # validate + deploy pipeline
```

## Credits

The Signal configuration comes from [signalapp/Signal-TLS-Proxy](https://github.com/signalapp/Signal-TLS-Proxy) (MIT license, kept in [`signal/LICENSE`](signal/LICENSE)).
