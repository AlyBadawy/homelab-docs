# 06 — Identity Stack

Complete guide to deploying LLDAP and Authentik for centralized identity and authentication across the homelab.

## Prerequisites

- Guide 05 complete — NPM, PostgreSQL, and Redis running
- Docker networks: `identity` and `proxy` present
- PostgreSQL `authentik` database created by the init script

> **How to deploy a stack in Portainer:** Go to **Stacks → + Add stack**, enter the stack name shown, paste the compose content into the **Web editor**, add environment variables in the **Environment variables** section below the editor, then click **Deploy the stack**.

---

## Section 1: LLDAP

Stack name: `lldap`

LLDAP is a lightweight LDAP server — the user directory for the homelab. All accounts are created here and flow into Authentik.

**Configuration:**
- Base DN: `dc=in,dc=alybadawy,dc=com`
- Web UI: `lldap.in.alybadawy.com`
- LDAP port: `3890` (internal only)
- Admin user: `admin` (directory admin, not an app account)

### Step 1: Deploy LLDAP

In Portainer, create a new stack named `lldap` with the following compose content:

```yaml
services:
  lldap:
    image: lldap/lldap:stable
    container_name: lldap
    restart: unless-stopped
    volumes:
      - lldap_data:/data
    environment:
      - LLDAP_JWT_SECRET=${LLDAP_JWT_SECRET}
      - LLDAP_LDAP_USER_PASS=${LLDAP_ADMIN_PASSWORD}
      - LLDAP_LDAP_BASE_DN=dc=in,dc=alybadawy,dc=com
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

In the **Environment variables** section, add:

| Variable | Value |
|----------|-------|
| `LLDAP_JWT_SECRET` | *(64-char random secret — run `openssl rand -hex 32`)* |
| `LLDAP_ADMIN_PASSWORD` | *(strong password — store in password manager)* |

Click **Deploy the stack**. Check **Containers → lldap → Logs**. Expected: `LLDAP successfully started!`

### Step 2: Configure NPM Proxy for LLDAP

In NPM → **Proxy Hosts → Add Proxy Host**:

- **Domain Names:** `lldap.in.alybadawy.com`
- **Scheme:** `http`
- **Forward Hostname/IP:** `lldap`
- **Forward Port:** `17170`
- **SSL Certificate:** `wildcard-in-alybadawy-com`
- **Force SSL:** On
- **HTTP/2 Support:** On

### Step 3: Initialize LLDAP

Access `https://lldap.in.alybadawy.com` and log in:
- Username: `admin`
- Password: *(the `LLDAP_ADMIN_PASSWORD` you set)*

**Create user groups** (navigate to Groups):
- `homelab_users` — standard access to services
- `homelab_admins` — elevated/admin access across services

**Create your personal account** (Groups → Users → Add User):
- Set your username and a strong password
- Add to both `homelab_users` and `homelab_admins`

Your LLDAP directory is ready. Users created here sync into Authentik.

---

## Section 2: Authentik

Stack name: `authentik`

Authentik is the central authentication platform — it connects to LLDAP for user data and provides OIDC/OAuth2 login for all services.

**Configuration:**
- Domain: `auth.in.alybadawy.com`
- PostgreSQL database: `authentik` (created in Guide 05 init script)
- Redis: shared instance on `identity` network

### Step 1: Deploy Authentik

In Portainer, create a new stack named `authentik` with the following compose content:

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

In the **Environment variables** section, add:

| Variable | Value |
|----------|-------|
| `AUTHENTIK_SECRET_KEY` | *(64-char random secret — run `openssl rand -hex 32`)* |
| `AUTHENTIK_DB_PASSWORD` | *(must match the password set in the PostgreSQL init SQL for user `authentik`)* |

Click **Deploy the stack**. Wait ~1 minute for DB migrations. Check **Containers → authentik-server → Logs**. Watch for: `Starting application server for authentik...`

### Step 2: Configure NPM Proxy for Authentik

In NPM → **Proxy Hosts → Add Proxy Host**:

- **Domain Names:** `auth.in.alybadawy.com`
- **Scheme:** `http`
- **Forward Hostname/IP:** `authentik-server`
- **Forward Port:** `9000`
- **SSL Certificate:** `wildcard-in-alybadawy-com`
- **Force SSL:** On
- **HTTP/2 Support:** On
- **Websockets Support:** On

### Step 3: Complete Authentik Initial Setup

Access `https://auth.in.alybadawy.com/if/flow/initial-setup/`

1. Create the `akadmin` account with a very strong password (20+ chars)
2. **Store this password securely** — it's the Authentik super-admin
3. After setup, click the avatar icon → **Admin Interface**

⚠️ The `akadmin` account has complete control over Authentik and all integrated services.

---

## Section 3: Connect Authentik to LLDAP

This makes LLDAP the source of truth for users and groups in Authentik.

In **Authentik Admin Interface → Directory → Federation & Social Login**:

1. Click **Create → LDAP Source**
2. Fill in:

| Setting | Value |
|---------|-------|
| **Name** | `LLDAP` |
| **Slug** | `lldap` |
| **Server URI** | `ldap://lldap:3890` |
| **Bind CN** | `cn=admin,ou=people,dc=in,dc=alybadawy,dc=com` |
| **Bind Password** | *(your `LLDAP_ADMIN_PASSWORD`)* |
| **Base DN for User search** | `dc=in,dc=alybadawy,dc=com` |
| **Base DN for Group search** | `dc=in,dc=alybadawy,dc=com` |
| **User object filter** | `(|(uid=*)(cn=*))` |
| **Group object filter** | `(|(objectClass=groupOfNames)(objectClass=groupOfUniqueNames))` |
| **Group membership field** | `member` |

3. Under **Sync** settings, enable **Sync users** and **Sync groups**
4. Click **Create**
5. Open the LDAP source → **Sync** tab → **Run sync now**

Verify your LLDAP users and groups appear in Authentik under **Directory → Users** and **Groups**.

---

## Section 4: Create OIDC Providers

For each service that authenticates through Authentik, create an OIDC provider. You'll do this per-service in Guide 07, but the general flow is:

In **Authentik Admin → Applications → Providers**:

1. **Create → OAuth2/OpenID Connect Provider**
2. **Name:** (e.g., `Nextcloud OIDC`)
3. **Client Type:** `Confidential`
4. **Redirect URIs:** (application-specific — see Guide 07)
5. Click **Create**
6. Copy the **Client ID** and **Client Secret** — you'll need them in the app configuration

---

## Verification Checklist

- [ ] LLDAP running: check Portainer Containers list
- [ ] LLDAP web UI: `https://lldap.in.alybadawy.com` loads and login works
- [ ] Your user account + groups created in LLDAP
- [ ] Authentik running: check Portainer Containers list
- [ ] Authentik accessible: `https://auth.in.alybadawy.com` loads
- [ ] LDAP sync complete: users visible in Authentik **Directory → Users**

---

## Troubleshooting

**LLDAP won't start:**

```bash
docker logs lldap
docker network ls | grep identity
```

**Authentik stuck on startup:**

```bash
docker logs authentik-server | tail -20
```

Wait up to 2 minutes for DB migrations. Check that the `AUTHENTIK_DB_PASSWORD` matches what was set in the PostgreSQL init SQL.

**LDAP sync not working:**

```bash
docker logs lldap | grep -i ldap
```

Verify the Bind CN and password in the Authentik LDAP source config exactly match your LLDAP admin credentials.
