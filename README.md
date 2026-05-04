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
Clients → Domain (DNS) → Nginx (TLS termination) → Docker containers
                                                      ├── Signal Proxy
                                                      ├── Telegram Proxy
                                                      └── Xray
```

---

## Security Measures

- SSH access restricted to key-based authentication only
- Custom SSH port (non-default)
- TLS certificates via Let's Encrypt
- Isolated Docker network for inter-container communication
- Only ports 443 (HTTPS) exposed externally

---

## Screenshots

> Screenshots available upon request (IP, domain, and personal info redacted for security reasons).
