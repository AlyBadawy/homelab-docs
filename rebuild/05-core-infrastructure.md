# 05 — Core Infrastructure

This guide deploys NPM (reverse proxy), Netdata (monitoring), and the shared database layer (PostgreSQL and Redis). All services are deployed as Portainer stacks.

**Prerequisites:**
- Guide 03 complete — Docker installed, networks created, Portainer running at `http://172.20.20.5:9000`
- Guide 04 complete — wildcard cert for `*.in.alybadawy.com` in `/opt/stacks/proxy/npm/certs/`

> **How to deploy a stack in Portainer:** Go to **Stacks → + Add stack**, give it a name, paste the compose file content into the **Web editor**, then click **Deploy the stack**.

---

## Section 1: Nginx Proxy Manager (NPM)

Nginx Proxy Manager is the reverse proxy for all internal services. It terminates TLS, routes traffic by hostname, and provides a web UI for configuration.

### 1.1: Create NPM Compose File

Create the NPM docker-compose file:

```bash
mkdir -p /opt/stacks/proxy/npm/certs
cd /opt/stacks/proxy/npm
```

Create `/opt/stacks/proxy/npm/docker-compose.yml`:

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
      - ./certs:/certs:ro
    networks:
      - proxy
    environment:
      DB_SQLITE_FILE: /data/database.sqlite

volumes:
  npm_data:

networks:
  proxy:
    external: true
```

**Port mapping:**
- `80:80` — HTTP traffic (redirects to HTTPS)
- `443:443` — HTTPS traffic (where the wildcard cert terminates)
- `81:81` — NPM admin panel (web UI for managing proxy hosts)

### 1.2: Deploy NPM via Portainer

In Portainer, go to **Stacks → + Add stack**:
- **Name:** `npm`
- **Web editor:** paste the compose file above
- Click **Deploy the stack**

Wait 10–15 seconds for NPM to initialize its database. Verify in Portainer under **Containers** that `npm` is running.

### 1.3: Initial Login

Access the NPM admin panel in your browser:

```
http://172.20.20.5:81
```

Replace `172.20.20.5` with your server's local IP (e.g., `192.168.1.50`).

⚠️ **Warning:** The default credentials grant full access to all proxy hosts. Change them immediately.

**Default credentials:**
- Email: `admin@example.com`
- Password: `changeme`

1. Log in
2. Go to **Users** → click the admin user
3. Change the email to your actual email
4. Change the password to a strong, unique password
5. Save

### 1.4: Import the Wildcard Certificate

Now import the wildcard certificate from guide 03.

1. Go to **SSL Certificates** in the left menu
2. Click **Add SSL Certificate** → **Custom**
3. Fill in the form:
   - **Certificate:** Paste the full contents of `/opt/stacks/proxy/npm/certs/fullchain.pem`
   - **Private Key:** Paste the full contents of `/opt/stacks/proxy/npm/certs/key.pem`
   - **Name:** `wildcard-inside-alybadawy-com`
4. Click **Save**

To retrieve the certificate contents:

```bash
cat /opt/stacks/proxy/npm/certs/fullchain.pem
cat /opt/stacks/proxy/npm/certs/key.pem
```

### 1.5: Create First Proxy Host (NPM Admin)

Create a proxy host for NPM's own admin panel, accessible at `proxy.in.alybadawy.com`:

1. Go to **Proxy Hosts** in the left menu
2. Click **Add Proxy Host**
3. Fill in the **Details** tab:
   - **Domain Names:** `proxy.in.alybadawy.com`
   - **Scheme:** `http`
   - **Forward Hostname/IP:** `npm`
   - **Forward Port:** `81`
   - **Cache Assets:** On
   - **Block Common Exploits:** On
4. Click the **SSL** tab:
   - **SSL Certificate:** Select `wildcard-inside-alybadawy-com`
   - **Force SSL:** On
   - **HTTP/2 Support:** On
   - **HSTS Enabled:** On
5. Click **Save**

Test access:

```bash
curl -k https://proxy.in.alybadawy.com
```

Or open in browser: `https://proxy.in.alybadawy.com`

### 1.6: Disable Direct Port 81 Access (Optional but Recommended)

Once NPM is proxying itself, you don't need direct access to port 81. Disable it:

⚠️ **Warning:** After this step, you must access NPM admin at `https://proxy.in.alybadawy.com` only. If DNS fails, you'll need to use the server's console.

1. Remove port 81 from the docker-compose:
   ```bash
   nano /opt/stacks/proxy/npm/docker-compose.yml
   ```
   Delete the line: `- "81:81"`

2. Reload the container:
   ```bash
   cd /opt/stacks/proxy/npm
   docker compose up -d
   ```

3. Verify port 81 is closed:
   ```bash
   netstat -tlnp | grep 81
   ```
   No output means it's closed.

---

## Section 2: Portainer Proxy Host

Portainer is already running (deployed in Guide 03). Add an NPM proxy host so it's accessible at `https://docker.in.alybadawy.com`.

1. Go to NPM admin → **Proxy Hosts** → **Add Proxy Host**
2. **Details** tab:
   - **Domain Names:** `docker.in.alybadawy.com`
   - **Scheme:** `http`
   - **Forward Hostname/IP:** `portainer`
   - **Forward Port:** `9000`
   - **Block Common Exploits:** On
3. **SSL** tab:
   - **SSL Certificate:** `wildcard-in-alybadawy-com`
   - **Force SSL:** On
4. Click **Save**

Portainer is now accessible at `https://docker.in.alybadawy.com`.

---

## Section 3: Netdata

Netdata provides real-time system monitoring and performance metrics (CPU, memory, disk, network, etc.). It's lightweight and can monitor the host and all containers.

### 3.1: Create Netdata Compose File

Create the Netdata directory:

```bash
mkdir -p /opt/stacks/core/netdata
cd /opt/stacks/core/netdata
```

Create `/opt/stacks/core/netdata/docker-compose.yml`:

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

**Key configuration notes:**

- **`pid: host`** — Allows Netdata to access process info from the host
- **`network_mode: host`** — Netdata runs on the host network, accessible at `localhost:19999`
- **Capabilities and security:** Required to monitor system metrics without full root access
- **Volume mounts:** Read-only access to host system files and Docker socket

### 3.2: Deploy Netdata via Portainer

In Portainer, go to **Stacks → + Add stack**:
- **Name:** `netdata`
- **Web editor:** paste the compose file above
- Click **Deploy the stack**

Wait 5–10 seconds for Netdata to initialize and gather metrics. Verify in Portainer under **Containers** that `netdata` is running.

### 3.3: Test Direct Access

Test direct access to Netdata (via host network):

```bash
curl http://localhost:19999/api/v1/info
```

Should return JSON with Netdata version and capabilities.

### 3.4: Create NPM Proxy Host for Netdata

⚠️ **Important:** Because Netdata uses `network_mode: host`, it's accessible on the host's localhost, not via the container network. Use `127.0.0.1` as the forward address.

1. Go to NPM admin → **Proxy Hosts** → **Add Proxy Host**
2. Fill in the **Details** tab:
   - **Domain Names:** `netdata.in.alybadawy.com`
   - **Scheme:** `http`
   - **Forward Hostname/IP:** `127.0.0.1` ← localhost because of host network mode
   - **Forward Port:** `19999`
   - **Cache Assets:** Off (Netdata has frequent updates)
   - **Block Common Exploits:** On
3. Click the **SSL** tab:
   - **SSL Certificate:** `wildcard-inside-alybadawy-com`
   - **Force SSL:** On
4. Click **Save**

### 3.5: Access Netdata

Open in browser:

```
https://netdata.in.alybadawy.com
```

You'll see real-time system metrics:
- CPU, memory, disk usage
- Network I/O
- Docker container metrics
- Processes and system load
- And much more

---

## Section 4: PostgreSQL and Redis (Shared Database Layer)

PostgreSQL and Redis are the shared database services used by applications like Authentik and Nextcloud. They run on the `identity` and `apps` networks, accessible only from other containers.

### 4.1: Create PostgreSQL Initialization Script

Create the PostgreSQL init directory:

```bash
mkdir -p /opt/stacks/db/postgres/init
cd /opt/stacks/db/postgres
```

Create `/opt/stacks/db/postgres/init/01-databases.sql`:

```sql
-- PostgreSQL initialization script
-- Creates databases and users for Authentik, Nextcloud, and other services

-- Authentik database
CREATE USER authentik WITH PASSWORD 'CHANGE_ME_authentik_pass';
CREATE DATABASE authentik OWNER authentik;

-- Nextcloud database
CREATE USER nextcloud WITH PASSWORD 'CHANGE_ME_nextcloud_pass';
CREATE DATABASE nextcloud OWNER nextcloud;

-- Grant necessary privileges
ALTER USER authentik CREATEDB;
ALTER USER nextcloud CREATEDB;
```

⚠️ **Warning:** Change the passwords! These are database credentials for sensitive applications.

Better practice: Use a password manager to generate strong, unique passwords:

```bash
openssl rand -base64 32
```

Update the SQL file with the generated passwords.

### 4.2: Create PostgreSQL Environment File

Create `/opt/stacks/db/postgres/.env`:

```
POSTGRES_ROOT_PASSWORD=CHANGE_ME_strong_root_password
```

⚠️ **Warning:** Use a strong, unique password for the root PostgreSQL user. Example:

```bash
openssl rand -base64 32
```

This file is only used at initial setup. After the first run, you must access PostgreSQL directly to change the root password.

### 4.3: Create PostgreSQL Compose File

Create `/opt/stacks/db/postgres/docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:16
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_ROOT_PASSWORD}
      POSTGRES_INITDB_ARGS: "-c max_connections=200"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d:ro
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

**Network configuration:**
- `identity` — For Authentik, LDAP, and auth services
- `apps` — For Nextcloud and general applications

Services can query PostgreSQL at `postgres:5432` if they're on the same network.

### 4.4: Create Redis Compose File

Create `/opt/stacks/db/redis/docker-compose.yml`:

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

**Redis configuration:**
- **`--save 60 1`** — Save to disk every 60 seconds if at least 1 key changed (persistence)
- **`--appendonly yes`** — Append-only file for crash recovery
- **`--loglevel warning`** — Minimal logging

Services can connect to Redis at `redis:6379`.

### 4.5: Deploy PostgreSQL and Redis via Portainer

Deploy PostgreSQL first — in Portainer, go to **Stacks → + Add stack**:
- **Name:** `postgres`
- **Web editor:** paste the PostgreSQL compose file above
- Click **Deploy the stack**

Check logs in Portainer under **Containers → postgres → Logs** for any initialization errors. Wait for the healthcheck to show `healthy` in the Containers list.

Then deploy Redis — in Portainer, go to **Stacks → + Add stack**:
- **Name:** `redis`
- **Web editor:** paste the Redis compose file above
- Click **Deploy the stack**

### 4.6: Verify Database Connectivity

Test PostgreSQL from the host:

```bash
docker exec postgres pg_isready -U postgres
```

Expected output: `accepting connections`

Test from another container (if you have one running on the `identity` or `apps` network):

```bash
docker exec <container_name> psql -h postgres -U postgres -d postgres -c "SELECT 1"
```

Test Redis:

```bash
docker exec redis redis-cli ping
```

Expected output: `PONG`

---

## Deployment Summary

You now have a complete core infrastructure:

| Service | Container | Network | Access |
|---------|-----------|---------|--------|
| **Nginx Proxy Manager** | npm | proxy | `https://proxy.in.alybadawy.com` (port 80, 443) |
| **Portainer** | portainer | proxy | `https://docker.in.alybadawy.com` |
| **Netdata** | netdata | host (19999) | `https://netdata.in.alybadawy.com` |
| **PostgreSQL** | postgres | identity, apps | `postgres:5432` (container-only) |
| **Redis** | redis | identity, apps | `redis:6379` (container-only) |

---

## Next Steps

1. **Access your services:**
   - NPM admin: `https://proxy.in.alybadawy.com`
   - Portainer: `https://docker.in.alybadawy.com`
   - Netdata: `https://netdata.in.alybadawy.com`

2. **Monitor the databases:**
   - Use Portainer's UI to view container logs and stats
   - Use Netdata to monitor system and container performance

3. **Deploy applications:**
   - Use PostgreSQL and Redis for Authentik, Nextcloud, and other services
   - Use NPM to add proxy hosts for each new application
   - All services will automatically use HTTPS with the wildcard certificate

4. **Backup your data:**
   - PostgreSQL data: `/opt/stacks/db/postgres/postgres_data/` (via Docker volume)
   - Redis data: `/opt/stacks/db/redis/redis_data/` (via Docker volume)
   - NPM data: `/opt/stacks/proxy/npm/npm_data/` (via Docker volume)

5. **Security hardening:**
   - Close port 81 (NPM admin direct access) once everything is working
   - Use strong, unique passwords for all services
   - Enable 2FA in Portainer (if available)
   - Regularly check Netdata for anomalies

---

## Troubleshooting

### NPM won't start or keeps restarting

Check the logs:

```bash
docker logs npm
```

Common issues:
- Port 80 or 443 already in use: `lsof -i :80,443`
- Disk space full: `df -h /opt`
- Docker network proxy not created: `docker network ls`

### PostgreSQL won't initialize

Check logs:

```bash
docker logs postgres
```

Common issues:
- Volume already exists with incompatible data: Remove and restart
- Bad SQL syntax in init script: Check `/opt/stacks/db/postgres/init/01-databases.sql`
- Port 5432 in use: `lsof -i :5432`

### Netdata not showing metrics

Netdata requires access to host `/proc` and `/sys`:

```bash
docker exec netdata ls -la /host/proc
```

If empty, the volume mounts are not working. Rebuild the container:

```bash
cd /opt/stacks/core/netdata
docker compose down
docker compose up -d
```

### Proxy hosts return 502 Bad Gateway

This means NPM cannot reach the backend service:

1. Verify the container is running and on the correct network:
   ```bash
   docker ps | grep <service_name>
   docker network inspect proxy
   ```

2. Test connectivity from NPM container:
   ```bash
   docker exec npm ping -c 3 <service_name>
   ```

3. Check the forward port in NPM matches the exposed port in the container's Dockerfile or compose file
