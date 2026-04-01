# 07 — Nextcloud

Personal cloud storage and file sync, with users authenticated via Kanidm LDAP.

## Prerequisites

- Guide 05 complete — PostgreSQL and Redis running
- Guide 06 complete — Kanidm running, `nextcloud_reader` service account created
- Docker networks: `proxy` and `apps` present
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
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
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

| Variable                   | Value                                                                                 |
| -------------------------- | ------------------------------------------------------------------------------------- |
| `POSTGRES_PASSWORD`        | _(the `postgres` superuser password set when deploying the postgres stack in Guide 05)_ |
| `NEXTCLOUD_ADMIN_USER`     | `aly`                                                                                 |
| `NEXTCLOUD_ADMIN_PASSWORD` | _(strong temporary password — you'll disable this account after LDAP is working)_    |

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

## Section 5: Configure Kanidm LDAP Authentication

Nextcloud uses the **LDAP User and Group Backend** app to authenticate users directly against Kanidm. The `nextcloud_reader` service account created in Guide 06 is used for the LDAP bind.

### 5.1: Install the LDAP App

1. Log in to `https://cloud.in.alybadawy.com` with the admin credentials from Section 2
2. Click the admin avatar (top right) → **Apps**
3. Search for `LDAP user and group backend`
4. Click **Enable**

### 5.2: Configure the LDAP Connection

Go to **Administration** (avatar → Administration) → **LDAP/AD Integration**.

**Server tab:**

| Field    | Value                 |
| -------- | --------------------- |
| Host     | `172.20.20.5`         |
| Port     | `636`                 |
| User DN  | `dn=token`            |
| Password | _(the `nextcloud_reader` API token from Guide 06)_ |
| Base DN  | `dc=in,dc=alybadawy,dc=com` |

Check **Use TLS** (StartTLS) or select **Use SSL** depending on the option shown — for port 636 it should be SSL/LDAPS.

Click **Test Base DN** — you should see a green confirmation.

**Users tab:**

- User object classes: `posixaccount`
- Click **Verify settings and count users** — your Kanidm users should be detected

**Groups tab:**

- Group object classes: `group`
- Click **Verify settings and count groups** — `homelab_users` and `homelab_admins` should appear

**Advanced tab → Special Attributes:**

| Field         | Value  |
| ------------- | ------ |
| Internal Username | `uid` |
| Email field   | `mail` |

Click **Save** on each tab.

### 5.3: Test LDAP Login

1. Log out from Nextcloud (avatar → Log out)
2. Log in with your Kanidm username and password
3. You should be logged in as the Kanidm user

Once LDAP login is confirmed working, disable the initial local admin account in **Administration → Users** for security.

---

## Verification Checklist

- [ ] Nextcloud running: `docker ps | grep nextcloud` shows `Up`
- [ ] `https://cloud.in.alybadawy.com` loads, SSL valid
- [ ] `curl -IL https://cloud.in.alybadawy.com` shows `HTTP/2 200` with no HTTP redirects
- [ ] Admin login works with initial credentials
- [ ] LDAP app installed and connected — users count shows correctly
- [ ] Kanidm user can log in with their username and password
- [ ] iOS/macOS Nextcloud client authenticates without looping
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

**LDAP app can't connect / "Invalid credentials":**

Verify the `nextcloud_reader` token is correct and Kanidm is reachable from the Nextcloud container:

```bash
docker exec nextcloud curl -v --insecure ldaps://172.20.20.5:636
```

Check that the token hasn't expired in Kanidm (`kanidm service-account api-token status --name idm_admin nextcloud_reader`).

**"No users found" after LDAP configuration:**

Verify users have POSIX enabled in Kanidm:

```bash
docker run --rm -it -e KANIDM_URL=https://id.in.alybadawy.com kanidm/tools:latest \
  kanidm person posix show --name idm_admin alybadawy
```

Non-POSIX accounts are not visible to LDAP clients. Run `kanidm person posix set` if POSIX attributes are missing.

**Large file uploads fail:**

Verify the `client_max_body_size 10G` line is saved in NPM's Advanced settings. Check `docker logs nextcloud | tail -20` for PHP upload limit errors.
