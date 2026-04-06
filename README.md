# Homelab Documentation

> [!INFO]
>
> **Owner:** Aly Badawy \
> **Domain:** alybadawy.com \
> **Internal prefix:** `*.in.alybadawy.com` \
> **Last updated:** 2026-04-06

This repository is the single source of truth for the homelab server — covering every architectural decision made, the final design, and a complete step-by-step rebuild guide. If the server ever needs to be rebuilt from scratch, this document set is sufficient to reproduce the entire setup.

---

## Hardware

| Component   | Spec                                              |
| ----------- | ------------------------------------------------- |
| **Host**    | Beelink Mini PC Mini S12                          |
| **CPU**     | Intel N95 (12th Gen, 4-core, up to 3.4 GHz)       |
| **RAM**     | 8 GB DDR4                                         |
| **Storage** | 256 GB SSD (system + Docker)                      |
| **Network** | 2.5 Gigabit Ethernet (primary), Wi-Fi 7 (backup)  |
| **NAS**     | UGreen NAS → planned migration to UniFi UNAS 4    |
| **AREDN**   | Mikrotik hAC 2 Lite running WireGuard mesh tunnel |

---

## Network Summary

| Layer                   | Detail                                                                        |
| ----------------------- | ----------------------------------------------------------------------------- |
| **Router / Firewall**   | UniFi Dream Router 7 (UDR7) — `172.20.1.1`                                    |
| **Servers VLAN**        | VLAN 20 — `172.20.20.0/24` — homelab + NAS                                    |
| **Homelab OS hostname** | `dc.id.in.alybadawy.com` (required by Samba AD DC)                            |
| **Homelab IP**          | `172.20.20.5` (static, Servers VLAN)                                          |
| **NAS IP**              | `172.20.20.10` (static, Servers VLAN)                                         |
| **DNS (internal)**      | UDR7 built-in DNS — forwards `id.in.alybadawy.com` zone to Samba at 172.20.20.5 |
| **AD Realm**            | `ID.IN.ALYBADAWY.COM` — Samba 4 DC, bare-metal on homelab server              |
| **VPN**                 | UniFi VPN server on UDR7 (remote access, assigns Personal VLAN IPs)           |
| **Domain registrar**    | Vercel (alybadawy.com)                                                        |
| **Public entry point**  | `in.alybadawy.com` → UDR public IP (A record, auto-updated)                   |
| **Public wildcard**     | `*.in.alybadawy.com` CNAME → `in.alybadawy.com`                               |
| **Internal resolution** | `*.in.alybadawy.com` → `172.20.20.5` (homelab)                                |
| **Public exposure**     | None — homelab is LAN-only + VPN                                              |

---

## Services

| Service             | Purpose                                                        | Subdomain / Access                  |
| ------------------- | -------------------------------------------------------------- | ----------------------------------- |
| **Samba 4 AD DC**   | Centralized identity — LDAP, Kerberos, DNS for AD zone         | _(bare-metal, ports 389/636/88/53)_ |
| Nginx Proxy Manager | Reverse proxy + SSL termination                                | `proxy.in.alybadawy.com`            |
| Portainer           | Docker container management                                    | `dockers.in.alybadawy.com`          |
| Beszel              | Host + container monitoring dashboard                          | `monitor.in.alybadawy.com`          |
| Netdata             | System monitoring (CPU/RAM/disk/network)                       | `netdata.in.alybadawy.com`          |
| Nextcloud           | Self-hosted file sync and productivity                         | `cloud.in.alybadawy.com`            |
| Immich              | Self-hosted photo management                                   | `photos.in.alybadawy.com`           |
| Home Assistant      | Home automation hub                                            | `ha.in.alybadawy.com`               |
| PostgreSQL          | Shared relational database                                     | _(internal only)_                   |
| Redis               | Shared cache / queue                                           | _(internal only)_                   |

---

## Document Index

### Architectural Decisions

- [ADR-001 — OS Selection](decisions/ADR-001-os-selection.md)
- [ADR-002 — SSL Certificates & Reverse Proxy](decisions/ADR-002-ssl-certificates.md)
- [ADR-003 — Authentication Strategy](decisions/ADR-003-authentication-strategy.md)
- [ADR-004 — Storage Strategy](decisions/ADR-004-storage-strategy.md)
- [ADR-005 — NAS Folder Layout & Access Strategy](decisions/ADR-005-nas-layout.md)

### Architecture

- [Overview](architecture/overview.md)
- [Network Design](architecture/network.md)
- [Services & Docker Networks](architecture/services.md)
- [Storage Design](architecture/storage.md)

### Rebuild Guide

> Follow these in order for a full rebuild from bare metal.

1. [OS Installation](rebuild/01-os-installation.md)
2. [Active Directory — Samba 4 (bare-metal, before Docker)](rebuild/02-active-directory.md)
   - [DC Setup & Provisioning](rebuild/active-directory/01-samba-ad-dc-setup.md)
   - [User & Group Management](rebuild/active-directory/02-samba-ad-user-group-management.md)
3. [NAS Mounting (NFS — must run before Docker)](rebuild/03-nas-mounting.md)
4. [Docker + Portainer](rebuild/04-docker-setup.md)
5. [SSL Certificates (acme.sh + Vercel DNS)](rebuild/05-ssl-certificates.md)
6. [Core Infrastructure (NPM, Netdata, PostgreSQL, Redis)](rebuild/06-core-infrastructure.md)
7. [Nextcloud](rebuild/07-nextcloud.md)
8. [Immich](rebuild/08-immich.md)
9. [Home Assistant](rebuild/09-home-assistant.md)
10. [Beszel (Host Monitoring)](rebuild/10-beszel.md)

---

## Key Conventions

- All Docker stacks live under `/opt/stacks/`
- Each stack has its own `.env` file for secrets (never commit secrets)
- Docker networks are pre-created externally and shared across stacks
- All web UIs are accessed exclusively through Nginx Proxy Manager
- Persistent data that belongs on the NAS is mounted via NFS
- Samba 4 AD is the centralized identity provider — all services authenticate against it via LDAP/LDAPS or OIDC
- Samba AD is a bare-metal service (not Docker) and must be provisioned before Docker is installed
