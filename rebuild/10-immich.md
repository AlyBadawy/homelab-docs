# 10 — Immich

Self-hosted photo and video library with AI-powered features. Users are managed with Immich's built-in accounts.

## Prerequisites

- Guide 07 complete — PostgreSQL and Redis running
- Docker networks: `proxy` and `databases` present
- NAS mounted at `/mnt/nas/homelab/immich`

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
      - DB_HOSTNAME=postgres
      - DB_USERNAME=postgres
      - DB_PASSWORD=${POSTGRES_PASSWORD}
      - DB_DATABASE_NAME=immich
      - REDIS_HOSTNAME=redis
    networks:
      - proxy
      - databases

  immich-machine-learning:
    image: ghcr.io/immich-app/immich-machine-learning:release
    container_name: immich-machine-learning
    restart: unless-stopped
    volumes:
      - immich_model_cache:/cache
    mem_limit: 2g

volumes:
  immich_model_cache:

networks:
  proxy:
    external: true
  databases:
    external: true
```

In the **Environment variables** section, add:

| Variable            | Value                                                                              |
| ------------------- | ---------------------------------------------------------------------------------- |
| `POSTGRES_PASSWORD` | _(the global `postgres` superuser password set in Guide 07)_                       |

Immich will create the `immich` database in the global PostgreSQL instance on first run.

Click **Deploy the stack**. Wait 1–2 minutes for initialization. Check **Containers → immich-server → Logs**. Watch for: `Immich Server is listening`.

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

- [ ] Immich containers running: `docker ps | grep immich` — both `immich-server` and `immich-machine-learning` show `Up`
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

Usually a database connection error — confirm the global `postgres` container is running and healthy (`docker ps | grep postgres`), and that `POSTGRES_PASSWORD` matches the value set in Guide 07.

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
