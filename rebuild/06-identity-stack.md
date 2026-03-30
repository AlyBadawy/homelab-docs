# Identity Stack Deployment Guide

Complete guide to deploying LLDAP and Authentik for centralized identity and authentication across your homelab.

## Prerequisites

- Docker networks: `identity` (pre-created)
- PostgreSQL running on `identity` network (container name: `postgres`)
- Redis running on `identity` network (container name: `redis`)
- NPM with wildcard cert configured for `*.in.alybadawy.com`
- Stack directory structure in place: `/opt/stacks/identity/`

## Section 1: LLDAP Deployment

LLDAP is a lightweight LDAP server that acts as your user directory. All homelab services will authenticate against it through Authentik.

**LLDAP Configuration:**
- Base DN: `dc=inside,dc=alybadawy,dc=com`
- Admin user: `admin` (LDAP directory admin, not an app user)
- Web UI domain: `lldap.in.alybadawy.com` (optional but recommended)
- LDAP port: `3890` (internal only, no external access needed)

### Step 1: Create LLDAP Docker Compose File

Create `/opt/stacks/identity/lldap/docker-compose.yml`:

```yaml
services:
  lldap:
    image: lldap/lldap:stable
    container_name: lldap
    restart: unless-stopped
    ports:
      - "3890:3890"   # LDAP — only accessible within Docker network
    volumes:
      - lldap_data:/data
    environment:
      - LLDAP_JWT_SECRET=${LLDAP_JWT_SECRET}
      - LLDAP_LDAP_USER_PASS=${LLDAP_ADMIN_PASSWORD}
      - LLDAP_LDAP_BASE_DN=dc=inside,dc=alybadawy,dc=com
      - LLDAP_HTTP_PORT=17170
      - LLDAP_LDAP_PORT=3890
    networks:
      - identity

volumes:
  lldap_data:

networks:
  identity:
    external: true
```

### Step 2: Create LLDAP Environment File

Create `/opt/stacks/identity/lldap/.env`:

```env
LLDAP_JWT_SECRET=CHANGE_ME_random_64_char_secret
LLDAP_ADMIN_PASSWORD=CHANGE_ME_strong_lldap_admin_password
```

**Generate secure secrets:**

```bash
# Generate JWT secret (64 hex chars)
openssl rand -hex 32

# Use a strong password for admin account (min 12 chars, mixed case + numbers + symbols)
```

⚠️ **Security Warning:** The `LLDAP_ADMIN_PASSWORD` is sensitive. Store it in a password manager. This is the directory admin account, not a user account.

### Step 3: Deploy LLDAP via Portainer

In Portainer, go to **Stacks → + Add stack**:
- **Name:** `lldap`
- **Web editor:** paste the compose file above
- Click **Deploy the stack**

Verify in Portainer under **Containers → lldap → Logs**. Expected: `LLDAP successfully started!`

### Step 4: Configure NPM Proxy for LLDAP Web UI

In Nginx Proxy Manager, add a new proxy host:

- **Domain Name:** `lldap.in.alybadawy.com`
- **Scheme:** `http`
- **Forward Hostname/IP:** `lldap`
- **Forward Port:** `17170`
- **SSL Certificate:** Select wildcard cert for `in.alybadawy.com`
- **Force SSL:** On
- **HTTP/2 Support:** On

### Step 5: Initialize LLDAP

Access the web UI at `https://lldap.in.alybadawy.com`

1. **Login** with credentials:
   - Username: `admin`
   - Password: (the `LLDAP_ADMIN_PASSWORD` from your `.env`)

2. **Create user groups** (in the UI, navigate to Groups):
   - `homelab_users` — users with standard access to services
   - `homelab_admins` — users with elevated/admin access across services

3. **Create your personal user account** (Groups → Users → Add User):
   - Example: Username `aly`
   - Set a strong password
   - **Important:** Check "User is admin" to make this account a directory admin

4. **Add yourself to both groups:**
   - Click your user → Edit → Group memberships
   - Add to `homelab_users` and `homelab_admins`

Your LLDAP directory is now ready. Users created here will sync to Authentik in the next section.

---

## Section 2: Authentik Deployment

Authentik is your central authentication platform. It connects to LLDAP for user/group data and provides OIDC/OAuth2 login for all your applications.

**Authentik Configuration:**
- Domain: `auth.in.alybadawy.com`
- Admin panel port: `9000`
- Connects to PostgreSQL and Redis on the `identity` network
- Uses LLDAP as the user directory source

### Step 1: Create Authentik Docker Compose File

Create `/opt/stacks/identity/authentik/docker-compose.yml`:

```yaml
services:
  authentik-server:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik-server
    command: server
    restart: unless-stopped
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgres
      AUTHENTIK_POSTGRESQL__USER: authentik
      AUTHENTIK_POSTGRESQL__NAME: authentik
      AUTHENTIK_POSTGRESQL__PASSWORD: ${AUTHENTIK_DB_PASSWORD}
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY}
      AUTHENTIK_ERROR_REPORTING__ENABLED: "false"
    volumes:
      - authentik_media:/media
      - authentik_templates:/templates
    networks:
      - proxy
      - identity
    depends_on:
      - authentik-worker

  authentik-worker:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik-worker
    command: worker
    restart: unless-stopped
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgres
      AUTHENTIK_POSTGRESQL__USER: authentik
      AUTHENTIK_POSTGRESQL__NAME: authentik
      AUTHENTIK_POSTGRESQL__PASSWORD: ${AUTHENTIK_DB_PASSWORD}
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - authentik_media:/media
      - authentik_templates:/templates
    networks:
      - identity

volumes:
  authentik_media:
  authentik_templates:

networks:
  proxy:
    external: true
  identity:
    external: true
```

### Step 2: Create Authentik Environment File

Create `/opt/stacks/identity/authentik/.env`:

```env
AUTHENTIK_SECRET_KEY=CHANGE_ME_random_64_char_secret
AUTHENTIK_DB_PASSWORD=CHANGE_ME_authentik_db_password
```

**Generate secrets:**

```bash
# Generate secret key (64 hex chars)
openssl rand -hex 32

# DB password should be strong (min 16 chars recommended)
```

⚠️ **Security Warning:** Both values are cryptographic secrets. Store in password manager and never commit to version control.

### Step 3: Deploy Authentik via Portainer

In Portainer, go to **Stacks → + Add stack**:
- **Name:** `authentik`
- **Web editor:** paste the compose file above
- Click **Deploy the stack**

Wait about 1 minute for migrations to complete. Check **Containers → authentik-server → Logs** in Portainer. Watch for: `Starting application server for authentik...`

### Step 4: Configure NPM Proxy for Authentik

In Nginx Proxy Manager, add a new proxy host:

- **Domain Name:** `auth.in.alybadawy.com`
- **Scheme:** `http`
- **Forward Hostname/IP:** `authentik-server`
- **Forward Port:** `9000`
- **SSL Certificate:** Select wildcard cert for `in.alybadawy.com`
- **Force SSL:** On
- **HTTP/2 Support:** On
- **Websockets Support:** On (toggle in NPM)

### Step 5: Complete Authentik Initial Setup

Access `https://auth.in.alybadawy.com/if/flow/initial-setup/`

1. **Create the akadmin account:**
   - Username: `akadmin`
   - Password: Set a very strong password (20+ chars, all types)
   - **Store this password securely** — it's the Authentik super-admin

⚠️ **Security Warning:** The akadmin account has complete control over Authentik and all integrated services. Protect this password like a root password.

2. **Access the Admin Interface:**
   - After setup completes, click the avatar icon in the top right
   - Select "Admin Interface"

Your Authentik server is running. Now you'll connect it to LLDAP.

---

## Section 3: Connect Authentik to LLDAP

This step makes LLDAP the source of truth for user and group data in Authentik.

### Step 1: Add LLDAP as an LDAP Source

In the **Authentik Admin Interface:**

1. Navigate to **Directory** → **Federation & Social Login**
2. Click **Create** → **LDAP Source**
3. Fill in the following:

| Setting | Value |
|---------|-------|
| **Name** | `LLDAP` |
| **Slug** | `lldap` |
| **Server URI** | `ldap://lldap:3890` |
| **Bind CN** | `cn=admin,ou=people,dc=inside,dc=alybadawy,dc=com` |
| **Bind Password** | (your `LLDAP_ADMIN_PASSWORD`) |
| **Base DN for User search** | `dc=inside,dc=alybadawy,dc=com` |
| **Base DN for Group search** | `dc=inside,dc=alybadawy,dc=com` |
| **User object filter** | `(|(uid=*)(cn=*))` |
| **Group object filter** | `(|(objectClass=groupOfNames)(objectClass=groupOfUniqueNames))` |
| **Group membership field** | `member` |

4. Under **Sync** settings:
   - **Sync users** → Toggle ON
   - **Sync groups** → Toggle ON
   - **Sync parent group** → Leave blank
   - **User path prefix** → `authentik built-in/sources/ldap/`
   - **Group path prefix** → `authentik built-in/sources/ldap/`

5. Click **Create**

### Step 2: Run Initial LDAP Sync

1. In the LDAP source you just created, click the **Sync** tab
2. Click **Run sync now**
3. Watch the Authentik logs for sync messages:

```bash
docker logs -f authentik-server | grep -i ldap
```

Expected: Your LLDAP users and groups appear in Authentik's **Directory** → **Users** and **Groups** sections.

Your LDAP source is now syncing users and groups to Authentik. Users will sync automatically on login going forward.

---

## Section 4: Create OIDC Providers for Applications

For each service that will authenticate through Authentik (Nextcloud, Immich, Home Assistant, etc.), you need to create an OIDC/OAuth2 provider.

### General OIDC Provider Setup

In **Authentik Admin Interface** → **Applications** → **Providers**:

1. Click **Create** → **OAuth2/OpenID Connect Provider**

2. **Basic Settings:**
   - **Name:** (e.g., "Nextcloud OIDC" or "Immich OIDC")
   - **Authorization flow:** (Keep default)
   - **Client type:** `Confidential`

3. **Protocol Settings:**
   - **Client ID:** (Auto-generated, or set custom)
   - **Redirect URIs/Origins:** (Application-specific, see below)

4. **Advanced Protocol Settings:**
   - **Signing Key:** (Keep default)
   - Leave other settings at defaults

5. Click **Create**

6. **Save the credentials:**
   - Copy the **Client ID**
   - Click the eye icon to reveal **Client Secret**
   - Store both securely — you'll need them in each application's configuration

⚠️ **Security Warning:** Client Secrets are sensitive cryptographic credentials. Never commit them to git or share them. Store in `.env` files outside version control.

### Redirect URIs by Application

Each application will need a specific Redirect URI. You'll create the OIDC provider with the correct URI when deploying each service (see next guide).

---

## Verification Checklist

Before moving to the next deployment guide, verify:

- [ ] LLDAP is running: `docker ps | grep lldap`
- [ ] LLDAP web UI accessible: `https://lldap.in.alybadawy.com` (login works)
- [ ] User directory has your account + groups created
- [ ] Authentik is running: `docker ps | grep authentik`
- [ ] Authentik accessible: `https://auth.in.alybadawy.com` (login as akadmin works)
- [ ] LDAP source synced: Users visible in Authentik Directory
- [ ] Created OIDC providers for planned services (or will do per-service)

---

## Troubleshooting

**LLDAP won't start:**
```bash
# Check logs
docker logs lldap

# Verify network exists
docker network ls | grep identity

# Re-create if needed
docker network create identity
```

**Can't reach LLDAP web UI:**
- Verify NPM proxy is configured and SSL cert is valid
- Check LLDAP container is on the `identity` network: `docker inspect lldap | grep identity`

**Authentik stuck on startup:**
```bash
# Wait longer (up to 2 minutes for DB migrations)
docker logs authentik-server | tail -20

# Check database connectivity
docker exec authentik-server /bin/bash -c "psql -h postgres -U authentik -d authentik -c 'SELECT 1'"
```

**LDAP sync not working:**
- Verify LLDAP admin password is correct in Authentik LDAP source config
- Check LLDAP logs: `docker logs lldap | grep -i ldap`
- Manually verify LDAP connectivity: `docker exec authentik-server ldapsearch -x -H ldap://lldap:3890 -D cn=admin,ou=people,dc=inside,dc=alybadawy,dc=com -b dc=inside,dc=alybadawy,dc=com`

---

## Next Steps

Your identity stack is ready. Proceed to **07-application-services.md** to deploy:
- Nextcloud with OIDC login
- Immich with OIDC login
- Home Assistant with OAuth2 login
- Portainer integration
