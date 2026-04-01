# Homelab Services Architecture

**Environment:** Ubuntu Server 24.04 LTS
**Stack Directory:** `/opt/stacks/`
**Docker Networks:** `proxy`, `identity`, `apps`
**Base Domain:** `*.in.alybadawy.com`

---

## Core Infrastructure

| Service   | Image                           | Networks     | Exposed Ports                | Volumes                              | Subdomain                                |
| --------- | ------------------------------- | ------------ | ---------------------------- | ------------------------------------ | ---------------------------------------- |
| Dashboard | custom Rails app                | proxy        | 3000 (internal)              | dashboard_data                       | `dashboard.in.alybadawy.com`             |
| NPM       | jc21/nginx-proxy-manager:latest | proxy        | 80, 443, 81 (host)           | npm_data, certs volume               | `proxy.in.alybadawy.com` (port 81 admin) |
| Portainer | portainer/portainer-ce:latest   | proxy        | 9000 (internal to proxy net) | portainer_data, /var/run/docker.sock | `dockers.in.alybadawy.com``              |
| Netdata   | netdata/netdata:latest          | host network | 19999                        | host mounts (proc, sys, etc.)        | `netdata.in.alybadawy.com`               |

**Notes:**

- Dashboard is a custom Rails application serving as the homelab landing page/control panel
- NPM handles TLS termination and reverse proxying; wildcard cert deployed by acme.sh to shared `certs/` volume
- Portainer is the primary orchestration interface for stack management
- Netdata provides real-time system monitoring and metrics; runs in host network mode so it can observe all interfaces and processes

---

## Database Layer

| Service             | Image                              | Networks       | Purpose                  | Notes                                                                         |
| ------------------- | ---------------------------------- | -------------- | ------------------------ | ----------------------------------------------------------------------------- |
| PostgreSQL (Global) | postgres:16                        | identity, apps | Multi-tenant database    | One instance, multiple databases: nextcloud (and others as needed)            |
| Redis (Global)      | redis:7-alpine                     | identity, apps | Distributed caching      | Shared instance, different DB indexes per service                             |
| PostgreSQL (Immich) | ghcr.io/immich-app/postgres:latest | apps           | Immich-specific database | Separate instance; pgvecto.rs extension required for vector similarity search |

**Notes:**

- Global PostgreSQL: use `POSTGRES_MULTIPLE_DATABASES` init script to create separate databases
- Redis: configure each service to use unique database index (0-15) to avoid key collisions
- Immich PostgreSQL: must be on same network as Immich Server for pgvecto.rs functionality

---

## Identity Stack

| Service | Image                | Networks        | Exposed Ports          | Purpose                             | Notes                                                                            |
| ------- | -------------------- | --------------- | ---------------------- | ----------------------------------- | -------------------------------------------------------------------------------- |
| Kanidm  | kanidm/server:latest | proxy, identity | 8443 (UI), 636 (LDAPS) | User directory + POSIX LDAP + OIDC  | LDAPS-only; wildcard cert mounted from `/opt/certs/`; base DN `dc=in,dc=alybadawy,dc=com` |

**Notes:**

- Kanidm is the single source of truth for all user and group identities. It exposes `posixAccount` and `posixGroup` over LDAPS — compatible with the NAS and any POSIX LDAP client without schema workarounds.
- Kanidm requires TLS for all LDAP connections. The shared wildcard cert (`*.in.alybadawy.com`) is mounted directly from `/opt/certs/` — not proxied through NPM for LDAPS. NPM only proxies the web UI on port 8443.
- Kanidm also serves as the OAuth2/OIDC provider for all web apps (Immich, Home Assistant, Portainer BE). Nextcloud connects via LDAP directly. NPM and Netdata use local admin accounts.

---

## Applications

| Service                 | Image                                              | Networks    | Purpose                   | Notes                                                                                         |
| ----------------------- | -------------------------------------------------- | ----------- | ------------------------- | --------------------------------------------------------------------------------------------- |
| Nextcloud               | nextcloud:latest                                   | proxy, apps | File sync & collaboration | Data directory mounted from NAS via NFS; uses global PostgreSQL                               |
| Immich Server           | ghcr.io/immich-app/immich-server:release           | proxy, apps | Photo/video library       | Uses separate Immich PostgreSQL; library mounted from NAS via NFS                             |
| Immich Machine Learning | ghcr.io/immich-app/immich-machine-learning:release | apps        | ML inference (CPU)        | Memory limited to 2GB; processes photo/video for recognition, classification, EXIF extraction |
| Home Assistant          | ghcr.io/home-assistant/home-assistant:stable       | proxy       | Home automation hub       | Runs on host network or bridged with port 8123; requires host access for certain integrations |

**Notes:**

- All app containers authenticate via Authentik OAuth2/OIDC proxy (NPM)
- NFS mounts to `/mnt/nas/{nextcloud,immich}` must be mounted on host before container start
- Immich ML container is CPU-bound; consider pinning to specific cores on high-core systems

---

## Stack Directory Structure

```
/opt/stacks/
├── core/
│   ├── dashboard/
│   │   ├── docker-compose.yml
│   │   └── .env                      ← secrets (not in git)
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
│   └── kanidm/
│       ├── docker-compose.yml
│       └── server.toml               ← Kanidm config (domain, ports, TLS paths)
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
# identity/kanidm — no .env needed; secrets are in server.toml and API tokens stored in password manager

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
4. **Identity** (identity/): Kanidm (user directory + OAuth2/OIDC provider)
5. **Applications** (apps/): Nextcloud, Immich, Home Assistant

This order ensures dependencies are available before downstream services attempt to connect.

---

## Monitoring & Health Checks

- **Portainer Dashboard:** Monitor container status, resource usage, logs
- **Netdata Dashboard:** Real-time system metrics, alerting
- **Authentik Health:** Check `/api/v3/admin/system/` for backend status
- **Docker Compose Health:** Use `healthcheck:` in docker-compose.yml for critical services

Recommended container restart policy: `restart_policy: always` for production stacks.
