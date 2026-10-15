# Secure Communication Proxies — Production VPS

> This is my first personal project that actually serves a real purpose. It's used every day by my family, and I keep improving it as I learn.

A set of secure communication proxies running on a Linux VPS (OVH). The services are containerized (or run as hardened system services), sit behind an Nginx reverse proxy with TLS, and have been up in production for more than 6 months for 5 users.

## Services

| Service         | Setup                                    | Purpose                   |
|-----------------|------------------------------------------|---------------------------|
| Signal Proxy    | Docker Compose + Nginx + TLS             | Secure messaging relay    |
| Telegram Proxy  | Docker container (official MTProto image)| Encrypted messaging relay |
| Xray (Reality)  | systemd service                          | Secure tunneling proxy    |

## Stack

- OS: Ubuntu (OVH VPS)
- Signal and Telegram: Docker Compose, Nginx, Let's Encrypt (TLS)
- Xray: installed as a systemd service
- DNS: custom domain with an A record
- SSH: key-only access on a custom port

## What I did

- Provisioned and configured the VPS (Ubuntu, OVH)
- Set up DNS (A record, subdomain)
- Issued and configured TLS certificates with Let's Encrypt
- Hardened SSH access (key-only authentication, custom port)
- Kept it running and monitored it in production for 6+ months

## Current architecture

```
                Production VPS
                       │
        ┌──────────────┼──────────────┐
        │              │              │
    Port 443       Port 443       Port 8888
   (active OR)   (active OR)      (always)
        │              │              │
        ▼              ▼              ▼
   Signal Proxy   Xray Reality   Telegram MTProto
   (Docker)       (systemd)      (Docker)
```

## Security measures

- SSH restricted to key-based authentication only
- Custom SSH port (non-default)
- TLS certificates with Let's Encrypt (automated by the Signal proxy setup)
- Isolated Docker network for communication between containers
- Exposed ports: 80 (TLS renewal), 443 (HTTPS), 8888 (Telegram), SSH on a custom port

## Design notes

This is a personal project for my family (5 users). I focused on the essentials: SSH hardening (key-only, custom port) and TLS on all the web services. For this use case, the security level matches the actual risk.

I set the services up one at a time, as I needed them. That's why they don't share a single orchestration yet. The next section is how I want to rebuild it into a cleaner, more production-like stack.

## What I want to improve next

Right now the setup is a mix of Docker and systemd that I configured by hand. The goal for the next version is to turn it into a single, reproducible deployment, closer to what I'd set up in a company.

What I want to add:

- A single `docker-compose.yml` that orchestrates all the services, instead of the current mix.
- A WAF (ModSecurity with Nginx) in front of the web services to filter malicious traffic.
- Centralized logging (ELK or Loki) so I can audit and investigate incidents.
- Monitoring and alerting (Prometheus and Grafana) to catch anomalies and attacks in real time.
- Tighter network rules: restrict sensitive ports by IP and segment the Docker networks per service.
- TLS renewal handled by its own certbot container instead of being tied to one service.
- Proper secrets management (Docker secrets or Vault) instead of plain config and env files.

Target architecture I'm aiming for:

```
                          Internet
                             │
                             ▼
                    ┌─────────────────┐
                    │   Nginx + WAF   │   reverse proxy, TLS,
                    │  (ModSecurity)  │   rate limiting
                    └────────┬────────┘
                             │   (isolated Docker network)
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                     ▼
   Signal Proxy         Telegram Proxy          Xray
   (container)          (container)          (container)
        │                    │                     │
        └────────────────────┴─────────┬───────────┘
                                        ▼
                          ┌──────────────────────────┐
                          │  Observability sidecars  │
                          │  Prometheus, Grafana     │
                          │  Centralized logs (ELK)  │
                          └──────────────────────────┘

   All services in one docker-compose.yml
   Secrets injected via Docker secrets or Vault
   A certbot container handles TLS renewal for everything
```

I built it the current way because I needed something working and secure enough, fast. Cleaning it up into one IaC-managed stack with a WAF and real monitoring is the obvious next step, both to improve the design and to practice the production patterns I want to work with.
