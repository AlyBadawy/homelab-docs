# 07 — Immich

Self-hosted photo and video library with AI-powered features. Users are managed with Immich's built-in accounts.

## Prerequisites

- Guide 05 complete — core infrastructure running (NPM, Redis)
- Docker networks: `proxy` and `apps` present
- NAS mounted at `/mnt/nas/homelab/immich`

> Immich uses its **own dedicated PostgreSQL** instance (with the pgvector extension for AI features). It does not share the global PostgreSQL from Guide 05.

> **How to deploy a stack in Portainer:** Go to **Stacks → + Add stack**, enter the stack name shown, paste the compose content into the **Web editor**, add environment variables in the **Environment variables** section below the editor, then click **Deploy the stack**.

---

## Section 1: Deploy Immich

Stack name: `immich`

```yaml
services:
  immich-server:
    image: ghcr.io/immich-app/immich-server:release
    container_name: immich-server
    restart: unless-stopped
    volumes:
      - /mnt/nas/homelab/immich:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    environment:
      - DB_HOSTNAME=immich-postgres
      - DB_USERNAME=immich
      - DB_PASSWORD=${IMMICH_DB_PASSWORD}
      - DB_DATABASE_NAME=immich
      - REDIS_HOSTNAME=redis
    depends_on:
      - immich-postgres
    networks:
      - proxy
      - apps

  immich-machine-learning:
    image: ghcr.io/immich-app/immich-machine-learning:release
    container_name: immich-machine-learning
    restart: unless-stopped
    volumes:
      - immich_model_cache:/cache
    environment:
      - DB_HOSTNAME=immich-postgres
      - DB_USERNAME=immich
      - DB_PASSWORD=${IMMICH_DB_PASSWORD}
      - DB_DATABASE_NAME=immich
    mem_limit: 2g
    networks:
      - apps

  immich-postgres:
    image: ghcr.io/immich-app/postgres:14-vectorchord0.3.0-pgvectors0.2.0
    container_name: immich-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=immich
      - POSTGRES_PASSWORD=${IMMICH_DB_PASSWORD}
      - POSTGRES_DB=immich
      - POSTGRES_INITDB_ARGS=--encoding=UTF8
    volumes:
      - immich_postgres_data:/var/lib/postgresql/data
    networks:
      - apps

volumes:
  immich_model_cache:
  immich_postgres_data:

networks:
  proxy:
    external: true
  apps:
    external: true
```

In the **Environment variables** section, add:

| Variable             | Value                      |
| -------------------- | -------------------------- |
| `IMMICH_DB_PASSWORD` | _(strong unique password)_ |

Click **Deploy the stack**. Wait 2–3 minutes for the database to initialize. Check **Containers → immich-server → Logs**. Watch for: `Immich Server is listening`.

---

## Section 2: Configure NPM Proxy

In NPM → **Proxy Hosts → Add Proxy Host**:

- **Domain Names:** `photos.in.alybadawy.com`
- **Scheme:** `http`
- **Forward Hostname/IP:** `immich-server`
- **Forward Port:** `2283`
- **SSL Certificate:** `wildcard-in-alybadawy-com`
- **Force SSL:** On
- **HTTP/2 Support:** On
- **Websockets Support:** On

In the **Advanced** tab, paste:

```nginx
client_max_body_size 50G;
proxy_read_timeout 600s;
proxy_connect_timeout 600s;
proxy_send_timeout 600s;
proxy_buffering off;
```

---

## Section 3: Complete Immich Initial Setup

Open `https://photos.in.alybadawy.com`. You'll be taken through an onboarding wizard:

1. Create your admin account
2. Complete the wizard

Additional users can be added later under **Administration → Users**.

---

## Verification Checklist

- [ ] All Immich containers running: `docker ps | grep immich` — all show `Up`
- [ ] `https://photos.in.alybadawy.com` loads, SSL valid
- [ ] Admin login works
- [ ] Upload a photo → confirm it lands at `/mnt/nas/homelab/immich`
- [ ] Machine learning container not OOMKilled: `docker stats immich-machine-learning`

---

## Troubleshooting

**`immich-server` exits immediately after start:**

```bash
docker logs immich-server | tail -30
```

Usually means `immich-postgres` isn't ready yet. Wait 30 seconds and check again — `depends_on` doesn't wait for the DB to be fully ready, only for the container to start.

**Photos not appearing after upload:**

```bash
docker exec immich-server ls /usr/src/app/upload
```

**ML container OOMKilled:**

```bash
# Increase memory limit in Portainer stack editor:
mem_limit: 4g
# Then redeploy the stack
```
