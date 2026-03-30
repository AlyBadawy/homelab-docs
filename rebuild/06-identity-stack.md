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
      - proxy
      - identity

volumes:
  lldap_data:

networks:
  proxy:
    external: true
  identity:
    external: true
```

In the **Environment variables** section, add:

| Variable               | Value                                                  |
| ---------------------- | ------------------------------------------------------ |
| `LLDAP_JWT_SECRET`     | _(64-char random secret — run `openssl rand -hex 32`)_ |
| `LLDAP_ADMIN_PASSWORD` | _(strong password — store in password manager)_        |

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
- Password: _(the `LLDAP_ADMIN_PASSWORD` you set)_

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

| Variable                | Value                                                                           |
| ----------------------- | ------------------------------------------------------------------------------- |
| `AUTHENTIK_SECRET_KEY`  | _(64-char random secret — run `openssl rand -hex 32`)_                          |
| `AUTHENTIK_DB_PASSWORD` | _(must match the password set in the PostgreSQL init SQL for user `authentik`)_ |

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
2. Fill in the **Connection** settings:

| Setting                 | Value                                                                                                               |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **Name**                | `LLDAP`                                                                                                             |
| **Slug**                | `lldap`                                                                                                             |
| **Server URI**          | `ldap://lldap:3890`                                                                                                 |
| **Connection Security** | `No TLS` (LLDAP does not support STARTTLS — both containers are on the same internal network so plain LDAP is fine) |
| **Bind CN**             | `uid=admin,ou=people,dc=in,dc=alybadawy,dc=com`                                                                     |
| **Bind Password**       | _(your `LLDAP_ADMIN_PASSWORD`)_                                                                                     |
| **Base DN**             | `dc=in,dc=alybadawy,dc=com`                                                                                         |

3. Fill in the **LDAP Attribute Mapping** settings:

| Setting                     | Value                        |
| --------------------------- | ---------------------------- |
| **Addition User DN**        | `ou=people`                  |
| **Addition Group DN**       | `ou=groups`                  |
| **User object filter**      | `(objectClass=person)`       |
| **Group object filter**     | `(objectClass=groupOfNames)` |
| **Group membership field**  | `member`                     |
| **Object uniqueness field** | `uid`                        |

4. Set **User Property Mappings** (Selected column should contain exactly):
   - `authentik default LDAP Mapping: DN to User Path`
   - `authentik default LDAP Mapping: Name`
   - `authentik default LDAP Mapping: mail`
   - `authentik default OpenLDAP Mapping: cn`
   - `authentik default OpenLDAP Mapping: uid`

   Remove all Active Directory mappings from the Selected column.

5. Set **Group Property Mappings** (Selected column should contain exactly):
   - `authentik default OpenLDAP Mapping: cn`

6. Click **Update**
7. Open the LDAP source → click **Run sync again**

After sync completes, verify in the Authentik Admin Interface:

- **Directory → Users** — your LLDAP user accounts appear here (alongside `akadmin`), with **Sources** column showing `lldap`
- **Directory → Groups** — `homelab_users` and `homelab_admins` appear here

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

**LDAP sync shows "Not synced yet" after running:**

Sync runs in the worker — check worker logs first:

```bash
docker logs authentik-worker 2>&1 | grep -i "ldap\|sync\|error" | tail -20
```

Verify the worker can reach LLDAP:

```bash
docker exec authentik-worker python3 -c "import socket; s=socket.create_connection(('lldap', 3890), timeout=5); print('LDAP port reachable'); s.close()"
```

**`LDAPUnwillingToPerformResult - Unsupported extended operation: 1.3.6.1.4.1.1466.20037`**

Authentik is trying STARTTLS, which LLDAP doesn't support. In the LDAP source settings, set **Connection Security** to **No TLS**. Both containers are on the same Docker network so plain LDAP is safe.

**Wrong Bind CN** — LLDAP uses `uid=`, not `cn=`:

```
uid=admin,ou=people,dc=in,dc=alybadawy,dc=com   ✓
cn=admin,ou=people,dc=in,dc=alybadawy,dc=com    ✗
```

Update the Bind CN in **Authentik Admin → Directory → Federation & Social Login → LLDAP → Edit**, save, then run sync again.

**`LDAPAttributeError: invalid attribute type objectSid`**

Authentik requests the `objectSid` attribute (Active Directory-specific) from LLDAP, which doesn't support it. The fix is to remove all Active Directory property mappings from both User and Group Property Mappings in the LDAP source. Keep only the LDAP and OpenLDAP mappings listed in Section 3 above. If the error persists even with all AD mappings removed, go to **Customization → Property Mappings**, find any mapping whose expression references `objectSid`, and edit it to guard against the missing attribute:

```python
if not ldap.get("objectSid"):
    return None
return bytes_to_int(list_flatten(ldap.get("objectSid")))
```
