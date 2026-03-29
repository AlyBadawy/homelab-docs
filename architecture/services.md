# Homelab Services Architecture

**Environment:** Ubuntu Server 24.04 LTS
**Stack Directory:** `/opt/stacks/`
**Docker Networks:** `proxy`, `identity`, `apps`
**Base Domain:** `*.inside.alybadawy.com`

---

## Core Infrastructure

| Service | Image | Networks | Exposed Ports | Volumes | Subdomain |
|---------|-------|----------|---------------|---------|-----------|
| Portainer | portainer/portainer-ce:latest | proxy | 9000 (internal to proxy net) | portainer_data, /var/run/docker.sock | portainer.inside.alybadawy.com |
| Netdata | netdata/netdata:latest | host network | 19999 | host mounts (proc, sys, etc.) | netdata.inside.alybadawy.com |
| NPM | jc21/nginx-proxy-manager:latest | proxy | 80, 443, 81 (host) | npm_data, npm_letsencrypt, certs volume | npm.inside.alybadawy.com (port 81 admin) |

**Notes:**
- Portainer is the primary orchestration interface for stack management
- Netdata provides real-time system monitoring and metrics
- NPM (Nginx Proxy Manager) handles TLS termination and reverse proxying; wildcard certificate deployed by acme.sh to `certs/` volume

---

## Database Layer

| Service | Image | Networks | Purpose | Notes |
|---------|-------|----------|---------|-------|
| PostgreSQL (Global) | postgres:16 | identity, apps | Multi-tenant database | One instance, multiple databases: authentik, nextcloud, lldap (optional) |
| Redis (Global) | redis:7-alpine | identity, apps | Distributed caching | Shared instance, different DB indexes per service |
| PostgreSQL (Immich) | ghcr.io/immich-app/postgres:latest | apps | Immich-specific database | Separate instance; pgvecto.rs extension required for vector similarity search |

**Notes:**
- Global PostgreSQL: use `POSTGRES_MULTIPLE_DATABASES` init script to create separate databases
- Redis: configure each service to use unique database index (0-15) to avoid key collisions
- Immich PostgreSQL: must be on same network as Immich Server for pgvecto.rs functionality

---

## Identity Stack

| Service | Image | Networks | Purpose | Notes |
|---------|-------|----------|---------|-------|
| LLDAP | lldap/lldap:stable | identity | LDAP directory | LDAP service on port 3890, Web UI on port 17170 |
| Authentik Server | ghcr.io/goauthentik/server:latest | proxy, identity | OIDC/OAuth2 provider | Primary authentication backend for apps; sits behind NPM proxy |
| Authentik Worker | ghcr.io/goauthentik/server:latest | identity | Background task processing | Needs `/var/run/docker.sock` mount for outpost management |

**Notes:**
- LLDAP provides LDAP interface for legacy app integration
- Authentik Server connects to global PostgreSQL and Redis for session/config storage
- Authentik Worker runs background sync, notification, and outpost tasks
- Both Authentik containers share same database and Redis configuration

---

## Applications

| Service | Image | Networks | Purpose | Notes |
|---------|-------|----------|---------|-------|
| Nextcloud | nextcloud:latest | proxy, apps | File sync & collaboration | Data directory mounted from NAS via NFS; uses global PostgreSQL |
| Immich Server | ghcr.io/immich-app/immich-server:release | proxy, apps | Photo/video library | Uses separate Immich PostgreSQL; library mounted from NAS via NFS |
| Immich Machine Learning | ghcr.io/immich-app/immich-machine-learning:release | apps | ML inference (CPU) | Memory limited to 2GB; processes photo/video for recognition, classification, EXIF extraction |
| Home Assistant | ghcr.io/home-assistant/home-assistant:stable | proxy | Home automation hub | Runs on host network or bridged with port 8123; requires host access for certain integrations |

**Notes:**
- All app containers authenticate via Authentik OAuth2/OIDC proxy (NPM)
- NFS mounts to `/mnt/nas/{nextcloud,immich}` must be mounted on host before container start
- Immich ML container is CPU-bound; consider pinning to specific cores on high-core systems

---

## Stack Directory Structure

```
/opt/stacks/
├── core/
│   ├── portainer/
│   │   └── docker-compose.yml
│   └── netdata/
│       └── docker-compose.yml
├── proxy/
│   └── npm/
│       ├── docker-compose.yml
│       └── certs/                    ← acme.sh deploys wildcard cert here
├── db/
│   ├── postgres/
│   │   ├── docker-compose.yml
│   │   └── init/
│   │       └── 01-databases.sql
│   └── redis/
│       └── docker-compose.yml
├── identity/
│   ├── lldap/
│   │   ├── docker-compose.yml
│   │   └── .env                      ← secrets (not in git)
│   └── authentik/
│       ├── docker-compose.yml
│       └── .env                      ← secrets (not in git)
└── apps/
    ├── nextcloud/
    │   ├── docker-compose.yml
    │   └── .env                      ← secrets (not in git)
    ├── immich/
    │   ├── docker-compose.yml
    │   └── .env                      ← secrets (not in git)
    └── homeassistant/
        └── docker-compose.yml
```

---

## Secrets Management

**Critical:** Each stack directory with sensitive configuration contains a `.env` file that is:
- **Never committed to git** (add to .gitignore)
- **Backed up to a password manager** (1Password, Bitwarden, etc.)
- **Referenced in docker-compose.yml** via `env_file: .env` or inline `${VAR_NAME}`

Example `.env` structure:
```bash
# identity/lldap/.env
LLDAP_LDAP_USER_PASS=<strong-random-password>
LLDAP_JWT_SECRET=<strong-random-token>
LLDAP_DB_URL=postgres://user:password@postgres:5432/lldap

# identity/authentik/.env
AUTHENTIK_SECRET_KEY=<strong-random-key>
AUTHENTIK_BOOTSTRAP_TOKEN=<strong-random-token>
AUTHENTIK_BOOTSTRAP_PASSWORD=<strong-random-password>
POSTGRES_PASSWORD=<strong-random-password>
POSTGRES_USER=authentik
REDIS_PASSWORD=<strong-random-password>

# apps/nextcloud/.env
NEXTCLOUD_ADMIN_USER=admin
NEXTCLOUD_ADMIN_PASSWORD=<strong-random-password>
POSTGRES_PASSWORD=<strong-random-password>
REDIS_PASSWORD=<strong-random-password>

# apps/immich/.env
IMMICH_SERVER_API_KEY=<auto-generated-by-immich>
DB_PASSWORD=<strong-random-password>
REDIS_PASSWORD=<strong-random-password>
```

Backup procedure: Export `.env` files to your password manager with full directory path noted (e.g., "identity/authentik/.env") for quick recovery.

---

## Deployment Order

1. **Core Infrastructure** (core/): Portainer, Netdata
2. **Database Layer** (db/): PostgreSQL (global), Redis (global), PostgreSQL (Immich)
3. **Proxy** (proxy/): NPM + acme.sh wildcard cert setup
4. **Identity** (identity/): LLDAP, Authentik Server, Authentik Worker
5. **Applications** (apps/): Nextcloud, Immich, Home Assistant

This order ensures dependencies are available before downstream services attempt to connect.

---

## Monitoring & Health Checks

- **Portainer Dashboard:** Monitor container status, resource usage, logs
- **Netdata Dashboard:** Real-time system metrics, alerting
- **Authentik Health:** Check `/api/v3/admin/system/` for backend status
- **Docker Compose Health:** Use `healthcheck:` in docker-compose.yml for critical services

Recommended container restart policy: `restart_policy: always` for production stacks.
