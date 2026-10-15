# Secure Communication Proxies — Production VPS

Deployment and maintenance of multiple secure communication proxies on a production Linux VPS (OVH).

All services are containerized (or run as hardened system services), served behind a reverse proxy with TLS, and have been running in production for 6+ months for a small group of users (5).

---

## Services

| Service         | Setup                                   | Purpose                   |
|-----------------|-----------------------------------------|---------------------------|
| Signal Proxy    | Docker Compose + Nginx + TLS            | Secure messaging relay    |
| Telegram Proxy  | Docker container (official MTProto image)| Encrypted messaging relay |
| Xray (Reality)  | systemd service                         | Secure tunneling proxy    |

## Stack

- **OS:** Ubuntu (OVH VPS)
- **Signal & Telegram:** Docker Compose, Nginx, Let's Encrypt (TLS)
- **Xray:** Installed as a systemd service
- **DNS:** Custom domain with A record
- **SSH:** Key-only access, custom port

## What I did

- Provisioned and configured the VPS (Ubuntu, OVH)
- Configured DNS (A record, subdomain)
- Issued and configured TLS certificates via Let's Encrypt
- Hardened SSH access (key-only authentication, custom port)
- Ongoing maintenance and monitoring in production (6+ months)

## Current Architecture

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

## Security Measures

- SSH access restricted to key-based authentication only
- Custom SSH port (non-default)
- TLS certificates via Let's Encrypt (automated by the Signal proxy setup)
- Isolated Docker network for inter-container communication
- Exposed ports: 80 (TLS renewal), 443 (HTTPS), 8888 (Telegram) — SSH on a custom port

---

## Design Notes & Production Considerations

This is a personal project serving my family (5 users). I secured the essentials — SSH hardening (key-only, custom port) and TLS on all web services. For the current use case, the security level is proportionate to the risk.

The services were originally set up incrementally (each proxy added as needed), which is why they don't yet share a single orchestration. The section below describes how I intend to rebuild it as a clean, production-grade stack.

---

## Roadmap — Target Architecture (work in progress)

The goal of the next iteration is to move from an incrementally-built setup to a **single, reproducible, infrastructure-as-code deployment**, closer to what I would ship in a company.

### Objectives

- **One unified `docker-compose.yml`** orchestrating all services (instead of mixed Docker + systemd, set up by hand)
- **WAF** (e.g. ModSecurity / Nginx) in front of the web-facing services to filter malicious traffic
- **Centralized logging** (ELK or Loki) for auditing and incident response
- **Monitoring & real-time alerting** (Prometheus + Grafana) to detect anomalies and attacks
- **Network-level least privilege**: restrict sensitive ports by IP, segment Docker networks per service
- **Automated TLS renewal** decoupled from a single service (dedicated certbot container)
- **Secrets management** (Docker secrets / Vault) instead of plain config/env files

### Target architecture (planned)

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
                          │  Prometheus · Grafana    │
                          │  Centralized logs (ELK)  │
                          └──────────────────────────┘

   All services defined in a single docker-compose.yml
   Secrets injected via Docker secrets / Vault
   certbot container handles TLS renewal for all services
```

### Why not already done this way?

The current setup grew service by service to meet an immediate need, and prioritizing a working, secure-enough deployment over a perfect architecture. Consolidating it into a single IaC-managed stack with a WAF and full observability is the explicit next step — both to clean up the design and to practice the production patterns I want to work with professionally.
