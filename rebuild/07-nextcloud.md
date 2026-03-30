# 07 — Nextcloud

Personal cloud storage and file sync, authenticated through Authentik.

## Prerequisites

- Guide 06 complete — LLDAP and Authentik running, LDAP sync working
- Docker networks: `proxy` and `apps` present
- PostgreSQL running with `nextcloud` database created (Guide 05 init script)
- Redis running on the `apps` network
- NAS mounted at `/mnt/nas/homelab/cloudnext`

> **How to deploy a stack in Portainer:** Go to **Stacks → + Add stack**, enter the stack name shown, paste the compose content into the **Web editor**, add environment variables in the **Environment variables** section below the editor, then click **Deploy the stack**.

---

## Section 1: Deploy Nextcloud

Stack name: `nextcloud`

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

volumes:
  nextcloud_config:

networks:
  proxy:
    external: true
  apps:
    external: true
```

In the **Environment variables** section, add:

| Variable                    | Value                                                                              |
| --------------------------- | ---------------------------------------------------------------------------------- |
| `NEXTCLOUD_DB_PASSWORD`     | _(must match the password set in the PostgreSQL init SQL for user `nextcloud`)_    |
| `NEXTCLOUD_ADMIN_USER`      | `aly`                                                                              |
| `NEXTCLOUD_ADMIN_PASSWORD`  | _(strong temporary password — you'll disable this account after OIDC is working)_ |

Click **Deploy the stack**. Wait 1–2 minutes for initialization. Check **Containers → nextcloud → Logs**. Watch for: `New nextcloud instance initialized`.

---

## Section 2: Configure NPM Proxy

In NPM → **Proxy Hosts → Add Proxy Host**:

- **Domain Names:** `cloud.in.alybadawy.com`
- **Scheme:** `http`
- **Forward Hostname/IP:** `nextcloud`
- **Forward Port:** `80`
- **SSL Certificate:** `wildcard-in-alybadawy-com`
- **Force SSL:** On
- **HTTP/2 Support:** On
- **Websockets Support:** On

In the **Advanced** tab, paste:

```nginx
client_max_body_size 10G;
proxy_read_timeout 600s;
proxy_connect_timeout 600s;
proxy_send_timeout 600s;
```

These settings allow large file uploads and long-running sync operations.

---

## Section 3: Create Authentik OIDC Provider

In **Authentik Admin → Applications → Providers**:

1. Click **Create → OAuth2/OpenID Connect Provider**
2. Fill in:
   - **Name:** `Nextcloud OIDC`
   - **Client Type:** `Confidential`
   - **Redirect URIs:** `https://cloud.in.alybadawy.com/apps/user_oidc/code`
3. Click **Create**
4. Copy the **Client ID** and **Client Secret** (click the eye icon) — you'll need both in Section 5

Then create the Authentik Application:

In **Authentik Admin → Applications → Applications**:

1. Click **Create**
2. Fill in:
   - **Name:** `Nextcloud`
   - **Slug:** `nextcloud`
   - **Provider:** `Nextcloud OIDC`
3. Click **Create**

---

## Section 4: Install Nextcloud OIDC App

1. Open `https://cloud.in.alybadawy.com`
2. Log in with the admin credentials you set in Section 1
3. Click the admin avatar (top right) → **Apps**
4. Search for `openid connect user backend` (app name: `user_oidc`)
5. Click **Install**

---

## Section 5: Configure OIDC in Nextcloud

1. Go to **Administration** (avatar → Administration)
2. In the left sidebar, scroll to **OpenID Connect user backend**
3. Fill in:
   - **Provider name:** `Authentik`
   - **Client ID:** _(from Section 3)_
   - **Client Secret:** _(from Section 3)_
   - **Discovery URL:** `https://auth.in.alybadawy.com/application/o/nextcloud/.well-known/openid-configuration`
4. Click **Verify and save**

---

## Section 6: Test OIDC Login

1. Log out from Nextcloud (avatar → Log out)
2. On the login page, click **Log in with Authentik** (or the OpenID Connect button)
3. You'll be redirected to Authentik — log in with your LLDAP account
4. Authorize the app
5. You'll be logged into Nextcloud as your LLDAP user

Once OIDC is confirmed working, you can disable the initial admin account in **Settings → Users** for security.

---

## Verification Checklist

- [ ] Nextcloud running: `docker ps | grep nextcloud` shows `Up`
- [ ] `https://cloud.in.alybadawy.com` loads, SSL valid
- [ ] Admin login works with initial credentials
- [ ] OIDC app installed
- [ ] OIDC login redirects to Authentik
- [ ] LLDAP user can log in via OIDC
- [ ] Upload a file → confirm it lands at `/mnt/nas/homelab/cloudnext`

---

## Troubleshooting

**OIDC button not visible on login page:**

```bash
docker exec nextcloud php occ app:list | grep user_oidc
# If missing:
docker exec nextcloud php occ app:install user_oidc
```

**"Discovery URL returned an error" in OIDC settings:**

```bash
docker exec nextcloud curl -v https://auth.in.alybadawy.com/application/o/nextcloud/.well-known/openid-configuration
```

Nextcloud must be able to reach Authentik — confirm both are on the `proxy` network: `docker network inspect proxy`.

**Large file uploads fail:**

Verify the `client_max_body_size 10G` line is saved in NPM's Advanced settings. Check `docker logs nextcloud | tail -20` for PHP upload limit errors.
