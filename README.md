# Proxy Stack — Hardened multi-service VPS

Administration of a Linux VPS (OVHcloud) hosting **three proxy services** for a small group of users, in production for over six months. Goal: private, censorship-resistant network access for **Signal**, **Telegram** and general traffic.

This repository also documents the **refactor** of the project: moving from a hand-made installation to a **unified, versioned and auto-deployed `docker-compose` stack**.

---

## Services

| Service | Role |
|---|---|
| **Signal TLS Proxy** | TLS relay for Signal |
| **Telegram MTProxy** | MTProto proxy for Telegram |
| **Xray (VLESS-REALITY)** | general-purpose proxy with traffic obfuscation |

---

## Initial version (simple)

Originally, the three services were each installed **in their own way**:

- **Xray**: installed via a `systemd` script;
- **Signal**: run in a separate Docker container;
- **MTProxy**: **compiled** from source.

The result was **functional** but **heterogeneous**. Three different methods to maintain, hard to reproduce identically on a new server, and no versioned trace of the configuration.

---

## Refactor in progress → a unified stack

Migration to **a single `docker-compose.yml`**:

- **one container per service** (separation of concerns);
- **official images** instead of compiling or scripting the install;
- **configuration mounted as volumes**;
- **secrets in a `.env`**, never committed.

### Why this refactor

- **Reproducibility**: `docker compose up -d` recreates the whole stack on any server.
- **Readability & maintenance**: the entire architecture lives in one declarative file.
- **Versioning**: the infrastructure is tracked in Git (secrets excluded).
- **Best practices**: secrets kept out of the code, separation of concerns, least privilege.

---

## Automated deployment (CI/CD)

**GitHub Actions** pipeline: on every `push` to `main`, a runner connects to the VPS over **SSH** and updates the stack.

- A **dedicated deployment SSH key** (revocable without affecting other access);
- secrets (`VPS_HOST`, `VPS_PORT`, `VPS_USER`, `VPS_SSH_KEY`) stored **encrypted** on GitHub;
- the server's host fingerprint pre-registered with `ssh-keyscan` (no interactive prompt, anti-MITM).

> Currently: continuous deployment (**CD**). The validation step (**CI**: YAML lint, `docker compose config`) will be added **before** deployment.

---

## Security

- **Key-only SSH** access, `root` login disabled, custom port;
- **UFW** firewall (default-deny, only the needed ports open);
- **fail2ban** (bans brute-force IPs);
- **TLS** via Let's Encrypt (HTTP-01 challenge);
- secrets **kept out of the repository** (`.env` ignored by Git).

---

## Status

- [x] SSH / UFW / fail2ban hardening
- [x] GitHub Actions deployment pipeline (CD, working skeleton)
- [ ] `docker-compose.yml` for the three services *(in progress)*
- [ ] CI step (validation before deployment)
- [ ] Full switch from the old install to the unified stack

---

*Personal DevOps / DevSecOps learning project — administration, containerization, security and automation of a real production infrastructure.*
