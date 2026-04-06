# 09 — Nextcloud

Personal cloud storage and file sync. Users are managed with Nextcloud's built-in accounts.

## Prerequisites

- Guide 07 complete — PostgreSQL and Redis running
- Docker networks: `proxy` and `databases` present
- NAS mounted at `/mnt/nas/homelab/cloudnext`

> **How to deploy a stack in Portainer:** Go to **Stacks → + Add stack**, enter the stack name shown, paste the compose content into the **Web editor**, add environment variables in the **Environment variables** section below the editor, then click **Deploy the stack**.

---

## Section 1: Prepare NAS Directory

The Nextcloud container must run as the `homelab` NAS user so it can write to the NFS-mounted directory. Look up that user's UID and GID on the NAS first:

```bash
ssh admin@nas.in.alybadawy.com
id homelab
# Example: uid=1026(homelab) gid=100(users) ...
```

Then set ownership of the Nextcloud directory on the NAS:

```bash
# Still on the NAS — replace <UID> and <GID> with values from `id homelab`
chown -R <UID>:<GID> /volume1/homelab/cloudnext
```

Note the UID and GID — you'll need them as `HOMELAB_UID` and `HOMELAB_GID` in Portainer's environment variables in the next section.

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
      - PUID=${HOMELAB_UID}
      - PGID=${HOMELAB_GID}
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
      - databases

volumes:
  nextcloud_config:

networks:
  proxy:
    external: true
  databases:
    external: true
```

In the **Environment variables** section, add:

| Variable                   | Value                                                                                   |
| -------------------------- | --------------------------------------------------------------------------------------- |
| `HOMELAB_UID`              | _(UID of the `homelab` NAS user — from `id homelab` on the NAS)_                        |
| `HOMELAB_GID`              | _(GID of the `homelab` NAS user — from `id homelab` on the NAS)_                        |
| `POSTGRES_PASSWORD`        | _(the `postgres` superuser password set when deploying the postgres stack in Guide 07)_  |
| `NEXTCLOUD_ADMIN_USER`     | `aly`                                                                                   |
| `NEXTCLOUD_ADMIN_PASSWORD` | _(strong password)_                                                                     |

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

## Verification Checklist

- [ ] Nextcloud running: `docker ps | grep nextcloud` shows `Up`
- [ ] `https://cloud.in.alybadawy.com` loads, SSL valid
- [ ] `curl -IL https://cloud.in.alybadawy.com` shows `HTTP/2 200` with no HTTP redirects
- [ ] Admin login works
- [ ] iOS/macOS Nextcloud client authenticates without looping
- [ ] Upload a file → confirm it lands at `/mnt/nas/homelab/cloudnext`

---

## Troubleshooting

**`Permission denied` errors / "Your data directory is not writable":**

The NFS mount ownership must match the `homelab` NAS user's UID/GID. Fix it on the NAS directly (`chown` from the homelab server side won't work due to NFS root_squash):

```bash
ssh admin@nas.in.alybadawy.com
id homelab   # confirm UID and GID
chown -R <UID>:<GID> /volume1/homelab/cloudnext
```

Also confirm `HOMELAB_UID` and `HOMELAB_GID` in Portainer match those values, then redeploy the stack.

**iOS/macOS client stuck in grant-access loop:**

Ensure Section 3 (NPM `proxy_set_header` lines) and Section 4 (`occ` commands) are both complete. The `overwriteprotocol` setting is the most common missing piece.

**Large file uploads fail:**

Verify the `client_max_body_size 10G` line is saved in NPM's Advanced settings. Check `docker logs nextcloud | tail -20` for PHP upload limit errors.
