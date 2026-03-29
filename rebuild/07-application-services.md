# Application Services Deployment Guide

Complete guide to deploying Nextcloud, Immich, and Home Assistant with OIDC authentication through Authentik.

## Prerequisites

- Identity stack deployed (LLDAP + Authentik): See **06-identity-stack.md**
- Docker networks: `proxy` and `apps` (pre-created)
- PostgreSQL and Redis running on `apps` network
- NPM with wildcard cert for `*.in.alybadawy.com`
- NAS mounted at `/mnt/nas/` on the host with directories at `/mnt/nas/homelab/cloudnext`, `/mnt/nas/homelab/immich`, and `/mnt/nas/homelab/media`
- Stack directory structure: `/opt/stacks/apps/`

---

## Section 1: Nextcloud Deployment

Nextcloud is your personal cloud storage and collaboration platform. All users authenticate through Authentik's OIDC provider.

**Nextcloud Configuration:**
- Domain: `cloud.in.alybadawy.com`
- Database: PostgreSQL on `apps` network
- Cache: Redis on `apps` network
- File storage: `/mnt/nas/homelab/cloudnext` (persistent NAS mount)

### Step 1: Create Nextcloud Docker Compose File

Create `/opt/stacks/apps/nextcloud/docker-compose.yml`:

```yaml
services:
  nextcloud:
    image: nextcloud:latest
    container_name: nextcloud
    restart: unless-stopped
    environment:
      - POSTGRES_HOST=postgres
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=nextcloud
      - POSTGRES_PASSWORD=${NEXTCLOUD_DB_PASSWORD}
      - NEXTCLOUD_ADMIN_USER=${NEXTCLOUD_ADMIN_USER}
      - NEXTCLOUD_ADMIN_PASSWORD=${NEXTCLOUD_ADMIN_PASSWORD}
      - NEXTCLOUD_TRUSTED_DOMAINS=cloud.in.alybadawy.com
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    volumes:
      - nextcloud_config:/var/www/html
      - /mnt/nas/homelab/cloudnext:/var/www/html/data
    networks:
      - proxy
      - apps
    depends_on:
      - postgres

volumes:
  nextcloud_config:

networks:
  proxy:
    external: true
  apps:
    external: true
```

### Step 2: Create Nextcloud Environment File

Create `/opt/stacks/apps/nextcloud/.env`:

```env
NEXTCLOUD_DB_PASSWORD=CHANGE_ME_nextcloud_db_password
NEXTCLOUD_ADMIN_USER=aly
NEXTCLOUD_ADMIN_PASSWORD=CHANGE_ME_strong_initial_admin_password
```

⚠️ **Security Note:** The initial admin password is temporary. You'll use Authentik for production logins. Set something strong but temporary here (you'll disable this admin account after OIDC is configured).

### Step 3: Prepare NAS Directory

On your NAS or mounted storage:

```bash
# Create Nextcloud data directory
mkdir -p /mnt/nas/homelab/cloudnext

# Set permissions (adjust as needed for your setup)
chmod 755 /mnt/nas/homelab/cloudnext
```

### Step 4: Start Nextcloud

```bash
cd /opt/stacks/apps/nextcloud
docker compose up -d

# Wait for initialization (1-2 minutes)
docker logs -f nextcloud
```

Watch for: `New nextcloud instance initialized`

### Step 5: Configure NPM Proxy for Nextcloud

In Nginx Proxy Manager, add a new proxy host:

- **Domain Name:** `cloud.in.alybadawy.com`
- **Scheme:** `http`
- **Forward Hostname/IP:** `nextcloud`
- **Forward Port:** `80`
- **SSL Certificate:** Wildcard cert for `in.alybadawy.com`
- **Force SSL:** On
- **HTTP/2 Support:** On
- **Websockets Support:** On

**Critical: Add NPM Advanced Settings**

Click the **Advanced** tab and paste:

```nginx
client_max_body_size 10G;
proxy_read_timeout 600s;
proxy_connect_timeout 600s;
proxy_send_timeout 600s;
```

These settings allow large file uploads and long-running sync operations.

### Step 6: Install Nextcloud OIDC App

1. Access `https://cloud.in.alybadawy.com`
2. Login with your initial admin credentials (from `.env`)
3. Click the admin avatar (top right) → **Administration**
4. Go to **Apps** → **Search for apps**
5. Search: `openid connect user backend` (or `user_oidc`)
6. Click **Install**

### Step 7: Create OIDC Provider in Authentik

In **Authentik Admin Interface** → **Applications** → **Providers**:

1. Click **Create** → **OAuth2/OpenID Connect Provider**
2. **Name:** `Nextcloud OIDC`
3. **Client Type:** `Confidential`
4. **Redirect URIs:** `https://cloud.in.alybadawy.com/apps/user_oidc/code`
5. Leave other settings at defaults
6. Click **Create**
7. **Copy:**
   - Client ID
   - Client Secret (click eye icon)

### Step 8: Configure OIDC in Nextcloud

Back in Nextcloud:

1. **Administration** → **Administration Settings** (sidebar)
2. Scroll to **OpenID Connect user backend** section
3. Fill in:
   - **Provider:** `Authentik` (or generic description)
   - **Client ID:** (paste from Authentik)
   - **Client Secret:** (paste from Authentik)
   - **Discovery URL:** `https://auth.in.alybadawy.com/application/o/nextcloud/.well-known/openid-configuration`

4. Click **Verify and save**

### Step 9: Create Authentik Application (Frontend)

Back in **Authentik Admin** → **Applications** → **Applications**:

1. Click **Create**
2. **Name:** `Nextcloud`
3. **Slug:** `nextcloud`
4. **Provider:** Select the `Nextcloud OIDC` provider you created
5. Click **Create**

### Step 10: Test OIDC Login

1. Logout from Nextcloud (click avatar → Logout)
2. On the login page, you should now see an **OpenID Connect** or provider button
3. Click it → you'll be redirected to Authentik
4. Login with your LLDAP user account
5. Authorize the app
6. You'll be logged into Nextcloud as your LDAP user

Nextcloud is now configured with OIDC authentication. User accounts are automatically created from your LLDAP directory.

⚠️ **Optional Cleanup:** Once OIDC is working, you can disable the initial admin account in Nextcloud settings for security.

---

## Section 2: Immich Deployment

Immich is your photo and video library with AI-powered features. It has its own PostgreSQL instance (with pgvector support) and machine learning container.

**Immich Configuration:**
- Domain: `photo.in.alybadawy.com`
- Database: Dedicated PostgreSQL with pgvector extension
- ML Container: Runs facial recognition and object detection
- Library: `/mnt/nas/homelab/immich` (NAS mount for all media)

### Step 1: Create Immich Docker Compose File

Create `/opt/stacks/apps/immich/docker-compose.yml`:

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

**Why a dedicated Immich PostgreSQL?** Immich uses the pgvector extension for AI features. The shared PostgreSQL may not have this extension, so Immich includes its own pre-configured database.

### Step 2: Create Immich Environment File

Create `/opt/stacks/apps/immich/.env`:

```env
IMMICH_DB_PASSWORD=CHANGE_ME_immich_db_password
```

### Step 3: Prepare NAS Directory

```bash
mkdir -p /mnt/nas/homelab/immich
chmod 755 /mnt/nas/homelab/immich
```

### Step 4: Start Immich Services

```bash
cd /opt/stacks/apps/immich
docker compose up -d

# Wait for database initialization (2-3 minutes)
docker logs -f immich-server
```

Watch for: `Immich Server is running...`

### Step 5: Configure NPM Proxy for Immich

In Nginx Proxy Manager:

- **Domain Name:** `photo.in.alybadawy.com`
- **Scheme:** `http`
- **Forward Hostname/IP:** `immich-server`
- **Forward Port:** `2283`
- **SSL Certificate:** Wildcard cert
- **Force SSL:** On
- **HTTP/2 Support:** On
- **Websockets Support:** On

**Critical: Add NPM Advanced Settings**

In the **Advanced** tab:

```nginx
client_max_body_size 50G;
proxy_read_timeout 600s;
proxy_connect_timeout 600s;
proxy_send_timeout 600s;
proxy_buffering off;
```

These settings support uploading very large video files.

### Step 6: Create OIDC Provider in Authentik

In **Authentik Admin** → **Applications** → **Providers**:

1. Click **Create** → **OAuth2/OpenID Connect Provider**
2. **Name:** `Immich OIDC`
3. **Client Type:** `Confidential`
4. **Redirect URIs:** `https://photo.in.alybadawy.com/auth/callback`
5. Click **Create**
6. **Copy:**
   - Client ID
   - Client Secret

### Step 7: Create Authentik Application

In **Authentik Admin** → **Applications** → **Applications**:

1. Click **Create**
2. **Name:** `Immich`
3. **Slug:** `immich`
4. **Provider:** Select the `Immich OIDC` provider
5. Click **Create**

### Step 8: Enable OIDC in Immich

1. Access `https://photo.in.alybadawy.com`
2. You'll be redirected to setup. Create an initial admin account (temporary).
3. Go to **Administration** (settings icon in top right)
4. Navigate to **User Management** → **Authentication Settings** → **OAuth**
5. **Enable OAuth:** Toggle ON
6. Fill in:
   - **Issuer URL:** `https://auth.in.alybadawy.com/application/o/immich/`
   - **Client ID:** (from Authentik)
   - **Client Secret:** (from Authentik)
   - **Scope:** `openid profile email` (should be auto-filled)
7. Click **Save**

### Step 9: Test Immich OIDC Login

1. Logout and return to login page
2. You should see an **OAuth** or **Login with Authentik** button
3. Click it and authenticate with your LLDAP account
4. You'll be logged into Immich as your LDAP user

---

## Section 3: Home Assistant Deployment

Home Assistant automates your smart home. It uses host network mode for device discovery, and optionally connects to Authentik for SSO.

**Home Assistant Configuration:**
- Domain: `ha.in.alybadawy.com`
- Network Mode: `host` (required for mDNS discovery of smart devices)
- Config storage: Docker volume
- Optional: OIDC login through Authentik

### Step 1: Create Home Assistant Docker Compose File

Create `/opt/stacks/apps/homeassistant/docker-compose.yml`:

```yaml
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: unless-stopped
    privileged: true
    network_mode: host
    volumes:
      - homeassistant_config:/config
      - /etc/localtime:/etc/localtime:ro
    environment:
      - TZ=America/Los_Angeles

volumes:
  homeassistant_config:
```

**Why `network_mode: host`?** Home Assistant needs direct access to mDNS (multicast DNS) and other network protocols to discover smart home devices. Docker bridge networking breaks these features.

### Step 2: Start Home Assistant

```bash
cd /opt/stacks/apps/homeassistant
docker compose up -d

# Wait for startup (2-3 minutes)
docker logs -f homeassistant
```

Watch for: `Home Assistant started at` or similar.

### Step 3: Complete Initial Onboarding

⚠️ **Important:** Because HA uses host network mode, access it directly (not through NPM) for initial setup:

1. Open `http://<172.20.20.5>:8123` in your browser
2. Complete the onboarding wizard
3. Create your admin account (this will be replaced by OIDC)
4. Finish onboarding

### Step 4: Configure NPM Proxy for Home Assistant

Back in **Nginx Proxy Manager**:

- **Domain Name:** `ha.in.alybadawy.com`
- **Scheme:** `http`
- **Forward Hostname/IP:** `127.0.0.1`
- **Forward Port:** `8123`
- **SSL Certificate:** Wildcard cert
- **Force SSL:** On
- **HTTP/2 Support:** On
- **Websockets Support:** On (toggle required for HA frontend)

### Step 5: Add NPM to Home Assistant Trusted Proxies

Because traffic now comes through NPM, Home Assistant needs to trust NPM as a proxy to preserve client IP information and security headers.

SSH into your homelab and edit `/opt/stacks/apps/homeassistant/homeassistant_config/configuration.yaml`:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
    - 172.16.0.0/12
```

Then restart HA:

```bash
docker compose -f /opt/stacks/apps/homeassistant/docker-compose.yml restart homeassistant
```

Now you can access Home Assistant through NPM at `https://ha.in.alybadawy.com`

### Step 6 (Optional): Enable OIDC for Home Assistant

Home Assistant has limited OIDC support. If you want SSO through Authentik:

⚠️ **Limitation:** Home Assistant's OAuth integration doesn't fully support PKCE (the modern OAuth standard). This is a HA limitation, not an Authentik issue.

**Option A: Generic OAuth2 Integration (Recommended)**

1. In **HA Settings** → **Devices & Services** → **Create Automation** → **Integrations**
2. Search for **Generic OAuth2** integration (if available in your HA version)
3. Follow setup and point to Authentik as provider

**Option B: Manual YAML Configuration**

Edit `/opt/stacks/apps/homeassistant/homeassistant_config/configuration.yaml`:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
    - 172.16.0.0/12

homeassistant:
  name: Home
  # ... other config

# Note: Full OAuth2 setup in HA requires manual YAML or available integration
# Consult HA documentation: https://www.home-assistant.io/integrations/oauth2/
```

For full OIDC integration, consult the [Home Assistant documentation](https://www.home-assistant.io) and Authentik's OAuth2 provider configuration.

---

## Section 4: Verify All Services

After deploying all three applications, use this checklist to verify everything works:

### Access & SSL

- [ ] `https://proxy.in.alybadawy.com` — NPM accessible, cert valid
- [ ] `https://auth.in.alybadawy.com` — Authentik login page loads
- [ ] `https://lldap.in.alybadawy.com` — LLDAP web UI loads
- [ ] `https://cloud.in.alybadawy.com` — Nextcloud loads
- [ ] `https://photo.in.alybadawy.com` — Immich loads
- [ ] `https://ha.in.alybadawy.com` — Home Assistant loads

### Authentication

- [ ] Logout of Nextcloud → see OIDC login button
- [ ] Click OIDC button → Authentik login redirects
- [ ] Login with your LLDAP account → redirects back to Nextcloud, logged in
- [ ] Repeat for Immich
- [ ] Home Assistant: manually logged in (OIDC optional)

### Container Health

```bash
docker ps | grep -E "nextcloud|immich|homeassistant"
```

All three should show `STATUS: Up` with no restart loops.

### Storage & Performance

- [ ] Upload a file to Nextcloud → visible at `/mnt/nas/homelab/cloudnext`
- [ ] Upload a photo to Immich → stored at `/mnt/nas/homelab/immich`
- [ ] Immich ML container not using excessive memory: `docker stats immich-machine-learning`

### External Access (VPN)

If you have a VPN configured:

- [ ] Connect from outside LAN
- [ ] Verify all services accessible through VPN
- [ ] Test OIDC login works from remote

---

## Troubleshooting

### Nextcloud Issues

**Can't see OIDC button on login:**
```bash
# Verify app is installed
docker exec nextcloud php occ app:list | grep user_oidc

# Reinstall if needed
docker exec nextcloud php occ app:install user_oidc
```

**"Discovery URL returned an error" in Nextcloud settings:**
```bash
# Test connectivity from Nextcloud container
docker exec nextcloud curl -v https://auth.in.alybadawy.com/application/o/nextcloud/.well-known/openid-configuration
```

**Users not syncing from LDAP:**
- Verify LDAP sync is working in Authentik (06-identity-stack.md troubleshooting)
- Force LDAP resync in Authentik Admin panel

### Immich Issues

**"Connection refused" when starting immich-server:**
```bash
# Wait for Immich PostgreSQL to be ready
docker logs immich-postgres | tail -20

# Check if postgres is listening
docker exec immich-postgres pg_isready
```

**ML container OOMKilled:**
```bash
# Increase memory limit in docker-compose.yml
mem_limit: 4g

# Then restart
docker compose restart immich-machine-learning
```

**OAuth not working:**
- Verify Client Secret is correct (copy-paste carefully)
- Check Issuer URL exactly matches: `https://auth.in.alybadawy.com/application/o/immich/`

### Home Assistant Issues

**Can't reach HA through NPM:**
```bash
# Verify it's listening on host network
docker exec homeassistant netstat -tlnp | grep 8123

# Test direct access first
curl http://127.0.0.1:8123
```

**Losing external connections after NPM proxy:**
- Ensure `trusted_proxies` is set in configuration.yaml
- Restart HA after changes: `docker compose restart homeassistant`

**Device discovery not working:**
- This is expected if HA is in Docker. Smart home devices on your LAN won't auto-discover through host networking in Docker.
- Workaround: Manual device IP configuration, or use a second HA instance on native hardware

---

## Performance & Optimization Tips

1. **Nextcloud:** Monitor RAM usage. Add more to Redis cache if large numbers of users.
2. **Immich:** The ML container is CPU/RAM intensive. Limit it to avoid starving other services.
3. **Home Assistant:** Disable automations/scripts you don't need; they run every minute by default.
4. **All services:** Monitor disk space at `/mnt/nas/`. Add more NAS storage before hitting 80% capacity.

---

## Next Steps

Congratulations! Your homelab stack is complete:

- **Identity:** LLDAP + Authentik for centralized auth
- **Storage:** Nextcloud for files, Immich for photos
- **Automation:** Home Assistant for smart home
- **Monitoring:** (Already deployed: Portainer, Netdata, NPM)

From here:

1. Invite other users: Create accounts in LLDAP, they'll see up in Authentik and all apps
2. Configure smart devices in Home Assistant
3. Organize photos in Immich
4. Share Nextcloud folders with other users
5. Set up automated backups of Nextcloud and Immich data to external storage

---

## Security Reminders

⚠️ Always remember:

- **Passwords:** Use unique, strong passwords. Store in password manager.
- **Secrets:** `.env` files with DB passwords and OIDC client secrets must never be committed to git.
- **Backups:** Regularly backup Nextcloud and Immich NAS directories.
- **SSL Certs:** Monitor wildcard cert expiration (usually 1 year). Renewal is automatic if using Let's Encrypt.
- **Updates:** Regularly update container images: `docker compose pull && docker compose up -d`
