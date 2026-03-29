# Homelab Documentation

> **Owner:** Aly Badawy
> **Domain:** alybadawy.com
> **Internal prefix:** `*.inside.alybadawy.com`
> **Last updated:** 2026-03-29

This repository is the single source of truth for the homelab server — covering every architectural decision made, the final design, and a complete step-by-step rebuild guide. If the server ever needs to be rebuilt from scratch, this document set is sufficient to reproduce the entire setup.

---

## Hardware

| Component | Spec |
|-----------|------|
| **Host** | Beelink Mini PC Mini S12 |
| **CPU** | Intel N95 (12th Gen, 4-core, up to 3.4 GHz) |
| **RAM** | 8 GB DDR4 |
| **Storage** | 256 GB SSD (system + Docker) |
| **Network** | Gigabit Ethernet (primary), Wi-Fi 5 (backup) |
| **NAS** | UGreen NAS → planned migration to UniFi UNAS 4 |

---

## Network Summary

| Layer | Detail |
|-------|--------|
| **Router / Firewall** | UniFi Dream Router 7 (UDR7) — `172.20.1.1` |
| **Servers VLAN** | VLAN 20 — `172.20.20.0/24` — homelab + NAS |
| **Homelab IP** | `172.20.20.15` (static, Servers VLAN) |
| **NAS IP** | `172.20.20.10` (static, Servers VLAN) |
| **DNS (internal)** | UDR7 built-in DNS — `172.20.20.1` for Servers VLAN |
| **VPN** | UniFi VPN server on UDR7 (remote access, assigns Personal VLAN IPs) |
| **Domain registrar** | Vercel (alybadawy.com) |
| **Public entry point** | `inside.alybadawy.com` → UDR public IP (A record, auto-updated) |
| **Public wildcard** | `*.inside.alybadawy.com` CNAME → `inside.alybadawy.com` |
| **Internal resolution** | `*.inside.alybadawy.com` → `172.20.20.15` (homelab) |
| **Public exposure** | None — homelab is LAN-only + VPN |

---

## Services

| Service | Purpose | Subdomain |
|---------|---------|-----------|
| Nginx Proxy Manager | Reverse proxy + SSL termination | `npm.inside.alybadawy.com` |
| Portainer | Docker container management | `portainer.inside.alybadawy.com` |
| Netdata | System monitoring (CPU/RAM/disk/network) | `netdata.inside.alybadawy.com` |
| LLDAP | Lightweight LDAP user directory | *(internal only, port 3890)* |
| Authentik | SSO / Identity Provider (OIDC + LDAP outpost) | `auth.inside.alybadawy.com` |
| Nextcloud | Self-hosted file sync and productivity | `nextcloud.inside.alybadawy.com` |
| Immich | Self-hosted photo management | `immich.inside.alybadawy.com` |
| Home Assistant | Home automation hub | `ha.inside.alybadawy.com` |
| PostgreSQL | Shared relational database | *(internal only)* |
| Redis | Shared cache / queue | *(internal only)* |

---

## Document Index

### Architectural Decisions

- [ADR-001 — OS Selection](decisions/ADR-001-os-selection.md)
- [ADR-002 — SSL Certificates & Reverse Proxy](decisions/ADR-002-ssl-certificates.md)
- [ADR-003 — Authentication Strategy](decisions/ADR-003-authentication-strategy.md)
- [ADR-004 — Storage Strategy](decisions/ADR-004-storage-strategy.md)

### Architecture

- [Overview](architecture/overview.md)
- [Network Design](architecture/network.md)
- [Services & Docker Networks](architecture/services.md)

### Rebuild Guide

> Follow these in order for a full rebuild from bare metal.

1. [OS Installation](rebuild/01-os-installation.md)
2. [Docker Setup](rebuild/02-docker-setup.md)
3. [SSL Certificates (acme.sh + Vercel DNS)](rebuild/03-ssl-certificates.md)
4. [Core Infrastructure (NPM, Portainer, Netdata)](rebuild/04-core-infrastructure.md)
5. [Identity Stack (LLDAP + Authentik)](rebuild/05-identity-stack.md)
6. [Application Services (Nextcloud, Immich, Home Assistant)](rebuild/06-application-services.md)

---

## Key Conventions

- All Docker stacks live under `/opt/stacks/`
- Each stack has its own `.env` file for secrets (never commit secrets)
- Docker networks are pre-created externally and shared across stacks
- All web UIs are accessed exclusively through Nginx Proxy Manager
- Persistent data that belongs on the NAS is mounted via NFS
