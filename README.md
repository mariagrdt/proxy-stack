# Proxy Stack — Secure communication proxies on a Linux VPS

> My first personal project that serves a real purpose: censorship-resistant access to **Signal**, **Telegram** and the open web for my family (5 users). The first version ran in production for more than 6 months. I'm now rebuilding it as a reproducible, infrastructure-as-code stack.

## Status

| Part | State |
|---|---|
| v1 (hand-built server) | ✅ Ran in production for 6+ months, now archived as an OVH snapshot |
| Server configuration with **Ansible** (Docker, UFW, fail2ban) | ✅ Applied to the new VPS |
| **Nginx** single entry point (`stream` on 443, `http` on 80) | ✅ Running |
| **Signal** TLS Proxy in the compose stack | ✅ **Running** on the new VPS (Let's Encrypt certificate, tested from a phone) |
| **Xray** (VLESS-REALITY) and **Telegram** (MTProxy) | 🚧 In progress |
| SNI routing between the three services | 🚧 In progress (needs Xray and MTProxy) |
| CI/CD deployment with GitHub Actions | 🚧 Pipeline exists; `docker compose up` is not enabled yet |

---

## Where I started (v1)

I set up each service on its own, the way its documentation suggested, as I needed it:

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
- **Nothing was reproducible.** The configuration only existed on the server. When I decided to rebuild, I had to take a snapshot and list by hand what was running.

## Current architecture (v2)

The new VPS starts empty. Everything on it is described in this repository.

```
                               Internet
                      │                          │
                  port 443                    port 80
               proxy traffic            Let's Encrypt only
                      │                          │
                      ▼                          ▼
   ┌────────────────────────────────────────────────────────┐
   │                      Nginx (stream)                    │
   │      only container reachable from the Internet        │
   │      reads the SNI, never decrypts                     │
   └─────┬──────────────┬──────────────┬──────────────┬─────┘
         │              │              │              │
  signal.<domain>  fake-TLS domain  any other     port 80
         │              │              │              │
         ▼              ▼              ▼              ▼
   ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
   │  Signal   │  │  MTProxy  │  │   Xray    │  │  certbot  │
   │  proxy    │  │           │  │  REALITY  │  │           │
   │           │  │  (to do)  │  │  (to do)  │  │ (via the  │
   │           │  │           │  │           │  │  Signal   │
   │           │  │           │  │           │  │  proxy)   │
   └─────┬─────┘  └───────────┘  └───────────┘  └───────────┘
         │
         ▼
   Signal servers
```

- **Port 443** carries the proxy traffic. Nginx sorts it by SNI (the domain name the client asks for) and passes it on still encrypted.
- **Port 80** is used **only** by Let's Encrypt to renew the Signal certificate. Any other request on port 80 gets a 404. There is no website to redirect to HTTPS.
- The "Signal proxy" box is two containers (`nginx-terminate` and `nginx-relay`), detailed [below](#why-signal-goes-through-two-nginx-containers).

| Layer | Tool |
|---|---|
| Server configuration | **Ansible**: Docker, UFW firewall, fail2ban |
| Services | **Docker Compose**: one file for the whole stack |
| Entry point | **Nginx** in `stream` mode, with SNI routing and TLS passthrough |
| Deployment | **GitHub Actions** over SSH, with a dedicated deploy key |

### How the traffic is routed

The first message of every TLS connection contains, in clear text, the name of the server the client wants to reach (the **SNI**). Nginx reads only that name (`ssl_preread`) and forwards the still-encrypted connection to the right container:

| Service | SNI sent by the client | Destination |
|---|---|---|
| Signal | my own subdomain | `nginx-terminate` |
| Telegram MTProxy | the domain encoded in the fake-TLS secret (`ee…`) | MTProxy container |
| Xray REALITY | the camouflage domain (`serverNames`), **not** mine | default route |

Xray is the **default** route on purpose: when REALITY receives a connection that is not a real client, it forwards it to the real camouflage website. A scanner probing the server therefore sees an ordinary site.

### Why Signal goes through two Nginx containers

A Signal connection is wrapped in **two TLS layers**. The outer layer goes to my domain and hides the fact that the user is talking to Signal (that is what defeats SNI-based blocking). The inner layer goes from the phone straight to Signal's servers.

```
Phone ══TLS #1══▶ nginx-terminate ──▶ nginx-relay ══TLS #2 (untouched)══▶ Signal
                  (my certificate)    (reads the inner SNI,
                                       allows only Signal hosts)
```

The server removes the outer layer only. It can never read messages.

---

## Design decisions

- **`stream`, not `http`.** These are encrypted tunnels, not web requests. Nginx must not decrypt them: REALITY and Signal need the original TLS handshake.
- **One entry point.** Only Nginx publishes ports. Docker bypasses UFW for published ports, so keeping a single published service keeps the firewall meaningful.
- **Images pinned by digest** (`@sha256:…`), so a moved tag can never silently change what runs.
- **No connection logs.** The users rely on this service for their privacy (same choice as the Signal proxy: `access_log off`).
- **Secrets stay out of Git.** The IP and SSH port live in `ansible/inventory.ini` (git-ignored). The MTProxy secret and the domains (including the camouflage domain, which must stay private) live in an **Ansible Vault** file, encrypted with AES-256. Ansible writes them to a git-ignored `.env` on the server; the compose file and the Nginx template only contain `${VARIABLES}`.
- **Right-sized.** One VPS doesn't need Kubernetes or Terraform. I use Terraform and Ansible on AWS in another project; here the value is in Ansible and Compose.

## Server configuration (Ansible)

The VPS is never changed by hand: every change goes through the playbook and is committed.

| Section | What it does |
|---|---|
| Docker | Installs Docker and the Compose plugin; adds the admin user to the `docker` group |
| UFW | Opens the SSH port, 80 and 443, **then** enables the firewall (deny by default) |
| fail2ban | Protects SSH, configured for the custom port (otherwise bans would hit port 22) |
| Secrets | Writes `.env` on the server from the encrypted vault (mode `0600`, `no_log` so values never appear in the output) |

```bash
ansible-playbook -i ansible/inventory.ini ansible/playbook.yml --check --diff --ask-vault-pass   # dry run
ansible-playbook -i ansible/inventory.ini ansible/playbook.yml --ask-vault-pass                  # apply
```

How secrets flow: **vault** (encrypted, on my machine) → decrypted **in memory** by Ansible → **`.env`** on the server → **Docker Compose** substitutes `${…}` in the compose file → the **Nginx image** fills its config template at startup.

For now I type the vault password at each run (`--ask-vault-pass`). That's fine because the playbook only runs when the server configuration or a secret changes, not on every deployment. Next, I want to automate it with a small `vault_password_file` script that reads the password from the macOS Keychain, so it is never typed or stored in plain text, plus an `ansible.cfg` to drop the repeated flags.

## Repository layout

```
.
├── docker-compose.yml        # the whole stack
├── nginx/nginx.conf          # main entry point (stream, SNI routing)
├── signal/                   # Signal TLS Proxy configs (+ upstream MIT license)
├── xray/  mtproxy/           # 🚧
├── ansible/                  # server configuration
└── .github/workflows/        # deployment pipeline
```

## Next steps

- Add Xray and MTProxy, then switch Nginx to SNI routing
- Reconnect the users to the new Signal proxy
- Deploy the stack with Ansible (clone, first certificate, `docker compose up`)
- Read the vault password from the macOS Keychain through a `vault_password_file` script instead of typing it
- CI: validate the compose file, scan for leaked secrets (gitleaks) and vulnerable images (Trivy), then deploy
- Container hardening (`no-new-privileges`, dropped capabilities, read-only filesystems)
- Monitoring: ship metrics to my existing Prometheus/Grafana stack on AWS, with no client IPs collected


## Credits

The Signal configuration comes from [signalapp/Signal-TLS-Proxy](https://github.com/signalapp/Signal-TLS-Proxy) (MIT license, kept in [`signal/LICENSE`](signal/LICENSE)).
