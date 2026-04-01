# 05 — Core Infrastructure

This guide deploys NPM (reverse proxy), Netdata (monitoring), and the shared database layer (PostgreSQL and Redis). All services are deployed as Portainer stacks.

**Prerequisites:**

- Guide 03 complete — Docker installed, networks created, Portainer running at `http://172.20.20.5:9000`
- Guide 04 complete — wildcard cert in `/opt/certs/`

> **How to deploy a stack in Portainer:** Go to **Stacks → + Add stack**, enter the stack name shown, paste the compose content into the **Web editor**, add any environment variables in the **Environment variables** section below the editor, then click **Deploy the stack**.

---

## Section 1: Nginx Proxy Manager (NPM)

Stack name: `npm`

### 1.1: Deploy NPM

In Portainer, create a new stack named `npm` with the following compose content:

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    volumes:
      - npm_data:/data
      - npm_letsencrypt:/etc/letsencrypt
      - /opt/certs:/certs:ro
    networks:
      - proxy
    environment:
      DB_SQLITE_FILE: /data/database.sqlite

volumes:
  npm_data:
  npm_letsencrypt:

networks:
  proxy:
    external: true
```

No environment variables needed. Click **Deploy the stack**.

Wait 10–15 seconds for NPM to initialize its database. Verify in Portainer under **Containers** that `npm` is running.

**Port mapping:**

- `80:80` — HTTP (redirects to HTTPS)
- `443:443` — HTTPS (wildcard cert terminates here)
- `81:81` — NPM admin panel (temporary — closed in step 1.6)

### 1.2: Initial Login

Access the NPM admin panel:

```
http://172.20.20.5:81
```

**Default credentials:**

- Email: `admin@example.com`
- Password: `changeme`

⚠️ Change these immediately — log in, go to **Users** → admin user → update email and password.

### 1.3: Import the Wildcard Certificate

Import the certificate issued in Guide 04 so it can be selected for all proxy hosts.

1. Go to **SSL Certificates** in the left menu
2. Click **Add SSL Certificate** → **Custom**
3. Fill in the form:
   - **Certificate:** Paste the full contents of `/opt/certs/fullchain.pem`
   - **Private Key:** Paste the full contents of `/opt/certs/key.pem`
   - **Name:** `wildcard-in-alybadawy-com`
4. Click **Save**

To retrieve the cert contents on the homelab:

```bash
cat /opt/certs/fullchain.pem
cat /opt/certs/key.pem
```

### 1.4: Create First Proxy Host (NPM Admin)

Create a proxy host for NPM's own admin panel at `proxy.in.alybadawy.com`:

1. **Proxy Hosts → Add Proxy Host**
2. **Details** tab:
   - **Domain Names:** `proxy.in.alybadawy.com`
   - **Scheme:** `http`
   - **Forward Hostname/IP:** `npm`
   - **Forward Port:** `81`
   - **Block Common Exploits:** On
3. **SSL** tab:
   - **SSL Certificate:** `wildcard-in-alybadawy-com`
   - **Force SSL:** On
   - **HTTP/2 Support:** On
   - **HSTS Enabled:** On
4. Click **Save**

Test: open `https://proxy.in.alybadawy.com` in your browser.

### 1.5: Disable Direct Port 81 Access

Once NPM is accessible via its subdomain, remove direct port 81 exposure.

In Portainer, go to **Stacks → npm → Editor**. Remove the `- "81:81"` line from the ports section, then click **Update the stack**.

⚠️ After this step, NPM admin is only accessible at `https://proxy.in.alybadawy.com`. If DNS fails, you'll need the server console.

Verify port 81 is closed:

```bash
netstat -tlnp | grep 81
```

No output means it's closed.

---

## Section 2: Portainer Proxy Host

Portainer is already running (deployed in Guide 03). Add an NPM proxy host so it's accessible at `https://dockers.in.alybadawy.com``.

1. **Proxy Hosts → Add Proxy Host**
2. **Details** tab:
   - **Domain Names:** `dockers.in.alybadawy.com``
   - **Scheme:** `http`
   - **Forward Hostname/IP:** `portainer`
   - **Forward Port:** `9000`
   - **Block Common Exploits:** On
3. **SSL** tab:
   - **SSL Certificate:** `wildcard-in-alybadawy-com`
   - **Force SSL:** On
4. Click **Save**

Portainer is now accessible at `https://dockers.in.alybadawy.com``.

---

## Section 3: Netdata

Stack name: `netdata`

Netdata provides real-time system monitoring — CPU, memory, disk, network, and all Docker containers.

In Portainer, create a new stack named `netdata` with the following compose content:

```yaml
services:
  netdata:
    image: netdata/netdata:latest
    container_name: netdata
    pid: host
    network_mode: host
    restart: unless-stopped
    cap_add:
      - SYS_PTRACE
      - SYS_ADMIN
    security_opt:
      - apparmor:unconfined
    volumes:
      - netdataconfig:/etc/netdata
      - netdatalib:/var/lib/netdata
      - netdatacache:/var/cache/netdata
      - /etc/passwd:/host/etc/passwd:ro
      - /etc/group:/host/etc/group:ro
      - /etc/localtime:/etc/localtime:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /etc/os-release:/host/etc/os-release:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro

volumes:
  netdataconfig:
  netdatalib:
  netdatacache:
```

No environment variables needed. Click **Deploy the stack**.

Wait 5–10 seconds. Verify in Portainer under **Containers** that `netdata` is running.

**Key configuration notes:**

- `pid: host` — Gives Netdata access to host process info
- `network_mode: host` — Netdata runs on the host network at port `19999`

### Test Direct Access

```bash
curl http://localhost:19999/api/v1/info
```

Should return JSON with Netdata version and capabilities.

### Create NPM Proxy Host for Netdata

⚠️ Because Netdata uses `network_mode: host`, use `127.0.0.1` as the forward address — not the container name.

1. **Proxy Hosts → Add Proxy Host**
2. **Details** tab:
   - **Domain Names:** `netdata.in.alybadawy.com`
   - **Scheme:** `http`
   - **Forward Hostname/IP:** `127.0.0.1`
   - **Forward Port:** `19999`
   - **Cache Assets:** Off
   - **Block Common Exploits:** On
3. **SSL** tab:
   - **SSL Certificate:** `wildcard-in-alybadawy-com`
   - **Force SSL:** On
4. Click **Save**

Access at `https://netdata.in.alybadawy.com`.

---

## Section 4: PostgreSQL and Redis

These are the shared database services used by Authentik, Nextcloud, and others. They are container-only — no external ports exposed.

### 4.1: Create the PostgreSQL Init Script

This file must exist on disk before deploying the stack — it's mounted into the container on first run to create the databases.

```bash
mkdir -p /opt/stacks/db/postgres/init
```

Create `/opt/stacks/db/postgres/init/01-databases.sql`:

```sql
-- Creates databases and users for Authentik and Nextcloud

-- Authentik
CREATE USER authentik WITH PASSWORD 'CHANGE_ME_authentik_pass';
CREATE DATABASE authentik OWNER authentik;
ALTER USER authentik CREATEDB;

-- Nextcloud
CREATE USER nextcloud WITH PASSWORD 'CHANGE_ME_nextcloud_pass';
CREATE DATABASE nextcloud OWNER nextcloud;
ALTER USER nextcloud CREATEDB;
```

⚠️ Replace the passwords with strong generated values:

```bash
openssl rand -base64 32
```

### 4.2: Deploy PostgreSQL

Stack name: `postgres`

In Portainer, create a new stack named `postgres` with the following compose content:

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
      - /opt/stacks/db/postgres/init:/docker-entrypoint-initdb.d:ro
    networks:
      - identity
      - apps
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:

networks:
  identity:
    external: true
  apps:
    external: true
```

In the **Environment variables** section, add:

| Variable            | Value                                            |
| ------------------- | ------------------------------------------------ |
| `POSTGRES_PASSWORD` | _(strong generated password — root DB password)_ |

Click **Deploy the stack**. Check **Containers → postgres → Logs** — wait for the healthcheck to show `healthy`.

### 4.3: Deploy Redis

Stack name: `redis`

In Portainer, create a new stack named `redis` with the following compose content:

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
      - identity
      - apps
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  redis_data:

networks:
  identity:
    external: true
  apps:
    external: true
```

No environment variables needed. Click **Deploy the stack**.

### 4.4: Verify Database Connectivity

```bash
# PostgreSQL
docker exec postgres pg_isready -U postgres

# Redis
docker exec redis redis-cli ping
```

Expected: `accepting connections` and `PONG`.

---

## Deployment Summary

| Service             | Stack name  | Network        | Access                              |
| ------------------- | ----------- | -------------- | ----------------------------------- |
| Nginx Proxy Manager | `npm`       | proxy          | `https://proxy.in.alybadawy.com`    |
| Portainer           | `portainer` | proxy          | `https://dockers.in.alybadawy.com`` |
| Netdata             | `netdata`   | host           | `https://netdata.in.alybadawy.com`  |
| PostgreSQL          | `postgres`  | identity, apps | `postgres:5432` (internal only)     |
| Redis               | `redis`     | identity, apps | `redis:6379` (internal only)        |

---

## Troubleshooting

### NPM won't start or keeps restarting

Check logs in Portainer under **Containers → npm → Logs**, or:

```bash
docker logs npm
```

Common issues:

- Port 80 or 443 already in use: `lsof -i :80,443`
- Docker network `proxy` not created: `docker network ls`

### PostgreSQL won't initialize

```bash
docker logs postgres
```

Common issues:

- Bad SQL in init script: check `/opt/stacks/db/postgres/init/01-databases.sql`
- Volume already exists with old data: remove it in Portainer under **Volumes** and redeploy

### Netdata not showing metrics

```bash
docker exec netdata ls -la /host/proc
```

If empty, the volume mounts failed. In Portainer, go to **Stacks → netdata** → **Stop** then **Start** to recreate the container.

### Proxy hosts return 502 Bad Gateway

NPM cannot reach the backend container:

```bash
docker ps | grep <service_name>
docker network inspect proxy
docker exec npm ping -c 3 <service_name>
```

Verify the container is on the `proxy` network and the forward port in NPM matches the container's port.
