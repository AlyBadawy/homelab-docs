# Homelab Documentation

> **Owner:** Aly Badawy \
> **Last updated:** 2026-04-06

This is the single source of truth for the homelab server — covering every architectural decision made, the final design, and a complete step-by-step rebuild guide. If the server ever needs to be rebuilt from scratch, this document set is sufficient to reproduce the entire setup.

For hardware specs, network design, and service architecture, see the [Architecture docs](architecture/overview.md).

---

## Services

See [Services & Docker Networks](architecture/services.md) for the full service inventory, network assignments, and deployment order.

---

## Document Index

### Architectural Decisions

- [ADR-001 — OS Selection](decisions/ADR-001-os-selection.md)
- [ADR-002 — SSL Certificates & Reverse Proxy](decisions/ADR-002-ssl-certificates.md)
- [ADR-003 — Authentication Strategy](decisions/ADR-003-authentication-strategy.md)
- [ADR-004 — Storage Strategy](decisions/ADR-004-storage-strategy.md)
- [ADR-005 — NAS Folder Layout & Access Strategy](decisions/ADR-005-nas-layout.md)
- [ADR-006 — Bare-Metal Exceptions (Samba AD & acme.sh)](decisions/ADR-006-bare-metal-exceptions.md)

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
6. [Core Infrastructure (Portainer, NPM)](rebuild/06-core-infrastructure.md)
7. [Databases (PostgreSQL, Redis)](rebuild/07-databases.md)
8. [Monitoring (Beszel)](rebuild/08-monitoring.md)
9. [Nextcloud](rebuild/09-nextcloud.md)
10. [Immich](rebuild/10-immich.md)
11. [Home Assistant](rebuild/11-home-assistant.md)

---

## Key Conventions

- Docker networks are pre-created externally and shared across stacks
- All web UIs are accessed exclusively through Nginx Proxy Manager
- Persistent data that belongs on the NAS is mounted via NFS
- Samba 4 AD is the centralized identity provider — services authenticate via LDAP/LDAPS or OIDC
- Samba 4 AD and acme.sh (cert management) run bare-metal; everything else is Docker (see ADR-006)
