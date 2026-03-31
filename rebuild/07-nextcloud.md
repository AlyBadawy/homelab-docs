# 07 — Nextcloud

Personal cloud storage and file sync, authenticated through Authentik.

## Prerequisites

- Guide 06 complete — LLDAP and Authentik running, LDAP sync working
- Docker networks: `proxy` and `apps` present
- PostgreSQL running on the `apps` network with `nextcloud` database created
- Redis running on the `apps` network
- NAS mounted at `/mnt/nas/homelab/cloudnext`

> **How to deploy a stack in Portainer:** Go to **Stacks → + Add stack**, enter the stack name shown, paste the compose content into the **Web editor**, add environment variables in the **Environment variables** section below the editor, then click **Deploy the stack**.

---

## Section 1: Prepare NAS Directory

The Nextcloud container runs as uid `1000` (PUID). The NAS-mounted directory must be owned by that user or Nextcloud will fail to initialize with permission errors.

SSH into the NAS and fix ownership there directly (NFS root_squash prevents `chown` from the server side):

```bash
ssh alybadawy@nas.in.alybadawy.com
chown -R 1000:10 /volume1/homelab/cloudnext
```

> **Why uid 1000, not 33?** The official Nextcloud image runs as `www-data` (uid 33), which happens to be a system service account on the UGreen NAS. The NAS blocks service accounts from writing to user-managed shared folders regardless of POSIX permissions. The LinuxServer image used here supports `PUID`/`PGID` env vars, so we run it as uid 1000 (your NAS user) to avoid this entirely.

---

## Section 2: Deploy Nextcloud

Stack name: `nextcloud`

```yaml
services:
  nextcloud:
    image: lscr.io/linuxserver/nextcloud:latest
    container_name: nextcloud
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=10
      - TZ=America/New_York
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
      - nextcloud_config:/config
      - /mnt/nas/homelab/cloudnext:/data
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

| Variable                   | Value                                                                               |
| -------------------------- | ----------------------------------------------------------------------------------- |
| `NEXTCLOUD_DB_PASSWORD`    | _(must match the password set in the PostgreSQL init SQL for user `nextcloud`)_     |
| `NEXTCLOUD_ADMIN_USER`     | `aly`                                                                               |
| `NEXTCLOUD_ADMIN_PASSWORD` | _(strong temporary password — you'll disable this account after OIDC is working)_  |

Click **Deploy the stack**. Wait 1–2 minutes for initialization. Check **Containers → nextcloud → Logs**. Watch for: `New nextcloud instance initialized`.

> **Note:** The LinuxServer image uses `/config` (named volume) for the app and `/data` for user files, instead of the official image's `/var/www/html`. All `occ` commands use the path `/config/www/nextcloud/occ` and must be run as user `abc`.

---

## Section 3: Configure NPM Proxy

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

proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

The `proxy_set_header` lines are required so Nextcloud knows it is behind HTTPS. Without them, mobile and desktop clients get stuck in a grant-access loop.

---

## Section 4: Configure Reverse Proxy Settings

After the container is running, set the external URL and trusted proxy subnets so Nextcloud handles HTTPS redirects correctly. These are required for the iOS/macOS clients to authenticate without looping.

```bash
docker exec -u abc nextcloud php /config/www/nextcloud/occ config:system:set overwrite.cli.url --value="https://cloud.in.alybadawy.com"

docker exec -u abc nextcloud php /config/www/nextcloud/occ config:system:set overwriteprotocol --value="https"

docker exec -u abc nextcloud php /config/www/nextcloud/occ config:system:set trusted_proxies 0 --value="172.16.0.0/12"
docker exec -u abc nextcloud php /config/www/nextcloud/occ config:system:set trusted_proxies 1 --value="172.17.0.0/16"
docker exec -u abc nextcloud php /config/www/nextcloud/occ config:system:set trusted_proxies 2 --value="172.18.0.0/16"
docker exec -u abc nextcloud php /config/www/nextcloud/occ config:system:set trusted_proxies 3 --value="172.19.0.0/16"
docker exec -u abc nextcloud php /config/www/nextcloud/occ config:system:set trusted_proxies 4 --value="172.20.0.0/16"
docker exec -u abc nextcloud php /config/www/nextcloud/occ config:system:set trusted_proxies 5 --value="172.21.0.0/16"
```

No restart needed — `occ` writes directly to `config.php`. Verify with:

```bash
curl -IL https://cloud.in.alybadawy.com
```

You should see `HTTP/2 200` with no `http://` redirects in the chain.

---

## Section 5: Create Authentik OIDC Provider

In **Authentik Admin → Applications → Providers**:

1. Click **Create → OAuth2/OpenID Connect Provider**
2. Fill in:
   - **Name:** `Nextcloud OIDC`
   - **Client Type:** `Confidential`
   - **Redirect URIs:** `https://cloud.in.alybadawy.com/apps/user_oidc/code`
3. Click **Create**
4. Copy the **Client ID** and **Client Secret** (click the eye icon) — you'll need both in Section 6

Then create the Authentik Application:

In **Authentik Admin → Applications → Applications**:

1. Click **Create**
2. Fill in:
   - **Name:** `Nextcloud`
   - **Slug:** `nextcloud`
   - **Provider:** `Nextcloud OIDC`
3. Click **Create**

---

## Section 6: Install Nextcloud OIDC App

1. Open `https://cloud.in.alybadawy.com`
2. Log in with the admin credentials you set in Section 2
3. Click the admin avatar (top right) → **Apps**
4. Search for `openid connect user backend` (app name: `user_oidc`)
5. Click **Install**

---

## Section 7: Configure OIDC in Nextcloud

1. Go to **Administration** (avatar → Administration)
2. In the left sidebar, scroll to **OpenID Connect user backend**
3. Fill in:
   - **Provider name:** `Authentik`
   - **Client ID:** _(from Section 5)_
   - **Client Secret:** _(from Section 5)_
   - **Discovery URL:** `https://auth.in.alybadawy.com/application/o/nextcloud/.well-known/openid-configuration`
4. Click **Verify and save**

---

## Section 8: Test OIDC Login

1. Log out from Nextcloud (avatar → Log out)
2. On the login page, click **Log in with Authentik** (or the OpenID Connect button)
3. You'll be redirected to Authentik — log in with your LLDAP account
4. Authorize the app
5. You'll be logged into Nextcloud as your LLDAP user

Once OIDC is confirmed working, disable the initial admin account in **Settings → Users** for security.

---

## Verification Checklist

- [ ] Nextcloud running: `docker ps | grep nextcloud` shows `Up`
- [ ] `https://cloud.in.alybadawy.com` loads, SSL valid
- [ ] `curl -IL https://cloud.in.alybadawy.com` shows `HTTP/2 200` with no HTTP redirects
- [ ] Admin login works with initial credentials
- [ ] OIDC app installed
- [ ] OIDC login redirects to Authentik and returns successfully
- [ ] iOS/macOS Nextcloud client authenticates without looping
- [ ] LLDAP user can log in via OIDC
- [ ] Upload a file → confirm it lands at `/mnt/nas/homelab/cloudnext`

---

## Troubleshooting

**`Permission denied` errors / "Your data directory is not writable":**

The NFS mount ownership must be fixed on the NAS directly — `chown` from the server won't work due to NFS root_squash, and the UGreen NAS blocks its internal `www-data` (uid 33) from accessing user shares:

```bash
ssh alybadawy@nas.in.alybadawy.com
chown -R 1000:10 /volume1/homelab/cloudnext
```

Then redeploy the stack in Portainer.

**iOS/macOS client stuck in grant-access loop:**

Ensure Section 3 (NPM `proxy_set_header` lines) and Section 4 (`occ` commands) are both complete. The `overwriteprotocol` setting is the most common missing piece.

**OIDC button not visible on login page:**

```bash
docker exec -u abc nextcloud php /config/www/nextcloud/occ app:list | grep user_oidc
# If missing:
docker exec -u abc nextcloud php /config/www/nextcloud/occ app:install user_oidc
```

**"Discovery URL returned an error" in OIDC settings:**

```bash
docker exec nextcloud curl -v https://auth.in.alybadawy.com/application/o/nextcloud/.well-known/openid-configuration
```

Nextcloud must be able to reach Authentik — confirm both are on the `proxy` network: `docker network inspect proxy`.

**Large file uploads fail:**

Verify the `client_max_body_size 10G` line is saved in NPM's Advanced settings. Check `docker logs nextcloud | tail -20` for PHP upload limit errors.
