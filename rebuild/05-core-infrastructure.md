# 05 — Core Infrastructure

This guide bootstraps Portainer (the first container), then deploys NPM, Kanidm, and the shared database layer. The deployment order matters: Portainer and Kanidm are brought up first because everything else either depends on or is managed through them.

**Deployment order in this guide:**

1. **Portainer** — first container; manages all subsequent stacks
2. **Kanidm** — second container; identity service needed by NAS and apps (deployed via Guide 06 immediately after Portainer is set up)
3. **NPM** — reverse proxy; makes all web UIs accessible via subdomain
4. **PostgreSQL + Redis** — shared database layer for apps

**Prerequisites:**

- Guide 03 complete — Docker installed, networks created (`proxy`, `identity`, `apps`)
- Guide 04 complete — wildcard cert in `/opt/certs/`

> **About `/opt/stacks/`:** Service config files (compose files, config files) live under `/opt/stacks/<service>/`. Create the root directory once here, then each service's subdirectory is created without `sudo` as you go:
>
> ```bash
> sudo mkdir -p /opt/stacks
> sudo chown $USER:$USER /opt/stacks
> ```
>
> Run this now before proceeding. After that, all `mkdir -p /opt/stacks/<service>` commands in this and future guides work without `sudo`.

> **About Portainer data:** Portainer stores all of its runtime data — stacks, users, settings, secrets, endpoints — in the Docker named volume `portainer_data` (at `/var/lib/docker/volumes/portainer_data/`). The compose file used to bootstrap it is a one-time script; after first launch, everything is managed through the Portainer UI.

---

## Section 0: Bootstrap Portainer

Portainer is the only service started from the CLI. Everything after this is deployed through the Portainer UI.

### 0.1: Start Portainer

Create the directory and compose file:

```bash
sudo mkdir -p /opt/stacks/portainer
sudo chown -r $USER:$USER /opt/stacks

cat > /opt/stacks/portainer/docker-compose.yml << 'EOF'
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      - proxy

volumes:
  portainer_data:

networks:
  proxy:
    external: true
EOF

cd /opt/stacks/portainer
docker compose up -d
docker ps | grep portainer
```

### 0.2: Initial Setup

Open in your browser:

```
http://172.20.20.5:9000
```

> Portainer prompts you to create an admin account on first access. If you wait more than a few minutes it locks down for security — restart with `docker restart portainer` to reset the timer.

1. Enter an admin username and a strong password
2. Click **Create user**
3. On the next screen, select **Get Started** (local Docker environment)

Portainer will now show the local Docker environment with the networks and containers you've already created.

### 0.3: Reboot Verification

Before proceeding, confirm Portainer survives a reboot:

```bash
# Confirm restart policy
docker inspect portainer --format '{{.HostConfig.RestartPolicy.Name}}'
# → should say: unless-stopped

sudo reboot
```

After the server comes back up, SSH in and verify:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

Portainer should show `Up X seconds`. Then open `http://172.20.20.5:9000` to confirm the UI is accessible.

---

> **Next: deploy Kanidm before NPM.** Kanidm is the second core service and should be running before NPM so its proxy host can be configured immediately. Proceed to **Guide 06, Section 1** now, then return here for NPM.

---

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

### Disable port 81 on UFW

```bash
sudo ufw delete allow 81/tcp
```

---

## Section 2: Portainer Proxy Host

Add an NPM proxy host so Portainer is accessible at `https://dockers.in.alybadawy.com`.

1. **Proxy Hosts → Add Proxy Host**
2. **Details** tab:
   - **Domain Names:** `dockers.in.alybadawy.com`
   - **Scheme:** `http`
   - **Forward Hostname/IP:** `portainer`
   - **Forward Port:** `9000`
   - **Block Common Exploits:** On
3. **SSL** tab:
   - **SSL Certificate:** `wildcard-in-alybadawy-com`
   - **Force SSL:** On
4. Click **Save**

Portainer is now accessible at `https://dockers.in.alybadawy.com`.

---

## Section 3: PostgreSQL and Redis

These are the shared database services used by Nextcloud and other apps. They are container-only — no external ports exposed.

Apps connect using the default `postgres` superuser — no init scripts or dedicated per-app users are needed. Each app creates its own database on first run if it doesn't exist.

### 3.1: Deploy PostgreSQL

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

| Variable            | Value                                                                            |
| ------------------- | -------------------------------------------------------------------------------- |
| `POSTGRES_PASSWORD` | _(strong generated password — this is the `postgres` superuser password)_        |

Generate a strong password:

```bash
openssl rand -base64 32
```

Save this password in your password manager — apps that use PostgreSQL will reference it.

Click **Deploy the stack**. Check **Containers → postgres → Logs** — wait for the healthcheck to show `healthy`.

### 3.2: Deploy Redis

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

### 3.3: Verify Database Connectivity

```bash
# PostgreSQL
docker exec postgres pg_isready -U postgres

# Redis
docker exec redis redis-cli ping
```

Expected: `accepting connections` and `PONG`.

---

## Deployment Summary

| Service             | Stack name  | Network        | Access                             |
| ------------------- | ----------- | -------------- | ---------------------------------- |
| Portainer           | `portainer` | proxy          | `https://dockers.in.alybadawy.com` |
| Nginx Proxy Manager | `npm`       | proxy          | `https://proxy.in.alybadawy.com`   |
| PostgreSQL          | `postgres`  | identity, apps | `postgres:5432` (internal only)    |
| Redis               | `redis`     | identity, apps | `redis:6379` (internal only)       |

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

- Volume already exists with old data: remove it in Portainer under **Volumes** and redeploy
- Container exits immediately: check logs for permission errors on the data volume

### Proxy hosts return 502 Bad Gateway

NPM cannot reach the backend container:

```bash
docker ps | grep <service_name>
docker network inspect proxy
docker exec npm ping -c 3 <service_name>
```

Verify the container is on the `proxy` network and the forward port in NPM matches the container's port.
