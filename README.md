# Secure Communication Proxies — Production VPS

Deployment and maintenance of multiple secure communication proxies on a production Linux VPS (OVH).

All services are containerized, served behind a reverse proxy with TLS, and have been running in production for over 6 months.

---

## Services

| Service | Setup | Purpose |
|---------|-------|---------|
| Signal Proxy | Docker Compose + Nginx + TLS | Secure messaging relay |
| Telegram Proxy | Docker container (official image) | Encrypted messaging relay |
| Xray | Systemd service | Secure tunneling proxy |

---

## Stack

- **OS**: Ubuntu (OVH VPS)
- **Signal & Telegram**: Docker Compose, Nginx, Let's Encrypt (TLS)
- **Xray**: Installed as systemd service
- **DNS**: Custom domain with A record
- **SSH**: Key-only access, custom port

## What I did

- Provisioned and configured the VPS (Ubuntu, OVH)
- Configured DNS (A record, subdomain)
- Configured TLS certificates via Let's Encrypt
- Hardened SSH access (key-only, custom port)
- Ongoing maintenance and monitoring in production (6+ months)
---

## Architecture

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

---

## Security Measures

- SSH access restricted to key-based authentication only
- Custom SSH port (non-default)
- TLS certificates via Let's Encrypt (automated by Signal proxy setup)
- Isolated Docker network for inter-container communication
- Exposed ports: 80 (TLS renewal), 443 (HTTPS), 8888 (Telegram) — SSH on custom port

---


## Design Notes & Production Considerations

This is a personal project serving my family (5 users). I secured the
essentials — SSH hardening (key-only, custom port) and TLS on all web services.

For a production environment in a company, I would additionally:
- Add a WAF (Web Application Firewall) in front of the web services
- Set up monitoring and real-time alerting to detect attacks
- Restrict access by IP on sensitive ports (least privilege at network level)
- Centralize logs for auditing and incident response

For the current use case, the security level is proportionate to the risk.
