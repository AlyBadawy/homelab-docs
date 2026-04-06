# 07 — Databases (PostgreSQL + Redis)

Shared database services used by Nextcloud and other apps. Both are container-only — no external ports exposed. Apps connect to them by name over the `databases` network.

Apps connect to PostgreSQL using the default `postgres` superuser — no init scripts or dedicated per-app users are needed. Each app creates its own database on first run if it doesn't exist.

**Prerequisites:**

- Guide 06 complete — Portainer running

> **How to deploy a stack in Portainer:** Go to **Stacks → + Add stack**, enter the stack name shown, paste the compose content into the **Web editor**, add environment variables in the **Environment variables** section below the editor, then click **Deploy the stack**.

---

## Section 1: Deploy PostgreSQL

Stack name: `postgres`

```yaml
services:
  postgres:
    image: postgres:16
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_INITDB_ARGS: "-c max_connections=200"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - databases
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:

networks:
  databases:
    external: true
```

In the **Environment variables** section, add:

| Variable            | Value                                                                     |
| ------------------- | ------------------------------------------------------------------------- |
| `POSTGRES_PASSWORD` | _(strong generated password — this is the `postgres` superuser password)_ |

Generate a strong password:

```bash
openssl rand -base64 32
```

Save this password in your password manager — apps that use PostgreSQL will reference it.

Click **Deploy the stack**. Check **Containers → postgres → Logs** — wait for the healthcheck to show `healthy`.

---

## Section 2: Deploy Redis

Stack name: `redis`

```yaml
services:
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    command: redis-server --save 60 1 --loglevel warning --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - databases
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  redis_data:

networks:
  databases:
    external: true
```

No environment variables needed. Click **Deploy the stack**.

---

## Section 3: Verify Connectivity

```bash
# PostgreSQL
docker exec postgres pg_isready -U postgres

# Redis
docker exec redis redis-cli ping
```

Expected: `accepting connections` and `PONG`.

---

## Deployment Summary

| Service    | Stack name | Network   | Access                          |
| ---------- | ---------- | --------- | ------------------------------- |
| PostgreSQL | `postgres` | databases | `postgres:5432` (internal only) |
| Redis      | `redis`    | databases | `redis:6379` (internal only)    |

---

## Troubleshooting

### PostgreSQL won't initialize

```bash
docker logs postgres
```

Common issues:

- Volume already exists with old data: remove it in Portainer under **Volumes** and redeploy
- Container exits immediately: check logs for permission errors on the data volume

### Redis won't start

```bash
docker logs redis
```

Common issues:

- Port conflict: Redis doesn't expose any host ports, so conflicts are rare — check for a duplicate container name with `docker ps -a`

### App can't reach postgres or redis

Verify the app's stack is also on the `databases` network:

```bash
docker network inspect databases
```

The connecting containers must appear in the `Containers` list. If not, add `databases` as an external network in the app's compose file and redeploy.
