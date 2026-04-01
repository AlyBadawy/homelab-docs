# 06 — Identity Stack

Complete guide to deploying Kanidm as the centralized identity and authentication service for the homelab.

## Prerequisites

- Guide 05 complete — NPM running, wildcard cert deployed to `/opt/certs/` (see guide 04)
- Docker networks `identity` and `proxy` present

> **How to deploy a stack in Portainer:** Go to **Stacks → + Add stack**, enter the stack name shown, paste the compose content into the **Web editor**, add environment variables in the **Environment variables** section below the editor, then click **Deploy the stack**.

---

## Section 1: Kanidm

Stack name: `kanidm`

Kanidm is the single identity service for the homelab — it handles both the POSIX user directory (user/group storage, LDAPS for the NAS) and OAuth2/OIDC authentication for all web applications. No separate SSO middleware is needed.

**Configuration:**

- Domain: `in.alybadawy.com`
- Base DN: `dc=in,dc=alybadawy,dc=com`
- Web UI: `https://id.in.alybadawy.com` (port 8443)
- LDAP: port 636 on host → 3636 in container (LDAPS — TLS only, no plain LDAP)
- TLS: wildcard cert mounted directly from `/opt/certs/`

> **Why LDAPS only?** Kanidm enforces encrypted LDAP for all clients. The wildcard cert from Let's Encrypt covers `*.in.alybadawy.com` so all clients trust it automatically without any cert import.

### Step 1: Create Config Directory and Config File

```bash
sudo mkdir -p /opt/stacks/kanidm
```

Create `/opt/stacks/kanidm/server.toml`:

```toml
bindaddress = "0.0.0.0:8443"
ldapbindaddress = "0.0.0.0:3636"
domain = "in.alybadawy.com"
origin = "https://id.in.alybadawy.com"
db_path = "/data/kanidm.db"
tls_chain = "/etc/kanidm/tls/fullchain.pem"
tls_key = "/etc/kanidm/tls/key.pem"
log_level = "info"
```

### Step 2: Deploy Kanidm

In Portainer, create a new stack named `kanidm`:

```yaml
services:
  kanidm:
    image: kanidm/server:latest
    container_name: kanidm
    restart: unless-stopped
    command: /sbin/kanidmd server -c /etc/kanidm/server.toml
    volumes:
      - kanidm_data:/data
      - /opt/stacks/kanidm/server.toml:/etc/kanidm/server.toml:ro
      - /opt/certs:/etc/kanidm/tls:ro
    ports:
      - "8443:8443" # Web UI (HTTPS)
      - "636:3636" # LDAPS — NAS and any LDAP client
    networks:
      - proxy
      - identity

volumes:
  kanidm_data:

networks:
  proxy:
    external: true
  identity:
    external: true
```

> **Portainer note:** Relative paths (`./`) are unreliable in Portainer stacks. Absolute paths are used here intentionally. The `command` override is required so Kanidm loads the config from `/etc/kanidm/` rather than `/data/`, avoiding a conflict with the named data volume.

Check logs after starting:

```bash
docker logs kanidm --tail 30
```

Expected in logs: `Kanidm is ready to rock!`

### Step 3: Open Firewall Ports

```bash
sudo ufw allow from 172.20.20.0/24 to any port 636 proto tcp comment "Kanidm LDAPS - Servers VLAN"
```

### Step 4: Configure NPM Proxy for Kanidm Web UI

In NPM → **Proxy Hosts → Add Proxy Host**:

| Field                   | Value                       |
| ----------------------- | --------------------------- |
| **Domain Names**        | `id.in.alybadawy.com`       |
| **Scheme**              | `https`                     |
| **Forward Hostname/IP** | `kanidm`                    |
| **Forward Port**        | `8443`                      |
| **SSL Certificate**     | `wildcard-in-alybadawy-com` |
| **Force SSL**           | On                          |
| **HTTP/2 Support**      | On                          |

> Kanidm serves HTTPS natively — set scheme to `https`, not `http`.

### Step 5: Bootstrap Admin Accounts

Kanidm has two built-in administrator accounts:

- `admin` — server admin (manages OAuth2 resources, server config). Recovery-only; not for daily use.
- `idm_admin` — identity admin (manages users and groups). Use this for all identity management.

Recover both accounts:

```bash
docker exec -i kanidm kanidmd recover-account admin -c /etc/kanidm/server.toml
docker exec -i kanidm kanidmd recover-account idm_admin -c /etc/kanidm/server.toml
```

Each command outputs a one-time temporary password. Log in to `https://id.in.alybadawy.com` for each account and set a permanent password via **Profile → Change Password**.

Store both passwords in your password manager.

### Step 6: Create Groups

Using the Kanidm CLI tools container (run from the homelab host):

```bash
# Create groups
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm group create --name idm_admin homelab_users

docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm group create --name idm_admin homelab_admins

# Enable POSIX on each group (required for NAS and all POSIX LDAP clients)
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm group posix set --name idm_admin homelab_users

docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm group posix set --name idm_admin homelab_admins
```

Kanidm auto-assigns a `gidNumber` to each POSIX-enabled group. This is what exposes them as `posixGroup` over LDAP.

### Step 7: Create User Accounts

For each user, create the account and enable POSIX. Repeat for every user:

```bash
# Create person account
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm person create --name idm_admin alybadawy "Aly Badawy"

# Set email
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm person update --name idm_admin alybadawy --mail alybadawy@icloud.com

# Enable POSIX (assigns uidNumber, gidNumber, homeDirectory automatically)
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm person posix set --name idm_admin alybadawy --shell /bin/bash

# Add to groups
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm group add-members --name idm_admin homelab_users alybadawy

docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm group add-members --name idm_admin homelab_admins alybadawy

# Generate a password reset token so the user can set their own password
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm person credential create-reset-token --name idm_admin alybadawy
```

The reset token URL lets the user set their own password without the admin knowing it.

### Step 8: Create NAS Service Account

The NAS needs a read-only LDAP bind account. Use a Kanidm service account with an API token — this avoids using personal credentials for machine-to-machine auth.

```bash
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm service-account create --name idm_admin nas_reader "NAS LDAP Reader"

# Save this token — you won't see it again
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm service-account api-token generate --name idm_admin nas_reader "nas-ldap"
```

Store the token in your password manager. When binding over LDAP, use:

- **Bind DN:** `dn=token`
- **Password:** _(the token value)_

### Section 1 Verification

- [ ] Kanidm running: `docker ps | grep kanidm`
- [ ] Web UI accessible: `https://id.in.alybadawy.com`
- [ ] LDAPS reachable: `openssl s_client -connect 172.20.20.5:636 -servername id.in.alybadawy.com`
- [ ] Users created with POSIX enabled
- [ ] Groups created with POSIX enabled
- [ ] NAS service account token saved in password manager

---

## Section 2: Connect the Ugreen NAS to Kanidm

The NAS authenticates users and resolves groups directly against Kanidm LDAPS. Kanidm exposes proper `posixAccount` and `posixGroup` over LDAP for all POSIX-enabled accounts — no schema workarounds needed.

In the Ugreen NAS Control Panel → **Domain/LDAP** → begin setup:

**Step 1 — Server info:**

| Field          | Value                 |
| -------------- | --------------------- |
| Server address | `id.in.alybadawy.com` |
| DNS server     | `172.20.20.1`         |

**Step 2 — Validation & Settings:**

| Field           | Value                          |
| --------------- | ------------------------------ |
| Bind DN/Account | `dn=token`                     |
| Password        | _(the `nas_reader` API token)_ |
| Encrypt         | SSL                            |
| BASE DN         | `dc=in,dc=alybadawy,dc=com`    |

After saving, click the **Domain user** and **Domain user group** tabs — your Kanidm POSIX users and groups should appear.

---

## Section 3: Configure Kanidm OAuth2/OIDC for Apps

Kanidm includes a built-in OAuth2/OIDC provider. Each application gets its own OAuth2 resource server with a unique client ID and secret. Use the `admin` account (not `idm_admin`) to manage OAuth2 resources.

The OIDC discovery URL for any Kanidm OAuth2 client follows this pattern:

```
https://id.in.alybadawy.com/oauth2/openid/<client-name>/.well-known/openid-configuration
```

### Nextcloud

Nextcloud connects to Kanidm via LDAP using the **LDAP User and Group Backend** app. This is simpler and more reliable than OIDC for file sync because it preserves stable user UIDs across sessions.

First, create a dedicated service account for Nextcloud's LDAP bind (same pattern as the NAS reader):

```bash
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm service-account create --name idm_admin nextcloud_reader "Nextcloud LDAP Reader"

docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm service-account api-token generate --name idm_admin nextcloud_reader "nextcloud-ldap"
```

Enable and configure the **LDAP/AD Integration** app in Nextcloud:

| Field               | Value                            |
| ------------------- | -------------------------------- |
| Server              | `ldaps://172.20.20.5`            |
| Port                | `636`                            |
| Bind DN             | `dn=token`                       |
| Bind Password       | _(the `nextcloud_reader` token)_ |
| Base DN             | `dc=in,dc=alybadawy,dc=com`      |
| User object filter  | `(objectClass=posixaccount)`     |
| Group object filter | `(objectClass=group)`            |

Users log in to Nextcloud with their Kanidm credentials (username + password).

### Immich

Immich has native OIDC support. Create the OAuth2 resource server in Kanidm first:

```bash
# Create Immich OAuth2 client (admin account manages OAuth2 resources)
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm system oauth2 create --name admin \
    immich "Immich" "https://photos.in.alybadawy.com/auth/login"

# Allow homelab_users to access Immich via OIDC
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm system oauth2 update-scope-map --name admin \
    immich homelab_users openid profile email

# Get the client secret (save this — you'll need it in Immich settings)
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm system oauth2 show-basic-secret --name admin immich
```

In Immich → **Administration → Settings → OAuth**:

| Field               | Value                                              |
| ------------------- | -------------------------------------------------- |
| Issuer URL          | `https://id.in.alybadawy.com/oauth2/openid/immich` |
| Client ID           | `immich`                                           |
| Client Secret       | _(from `show-basic-secret`)_                       |
| Scope               | `openid profile email`                             |
| Button text         | `Login with Kanidm`                                |
| Auto-register users | On                                                 |

### Home Assistant

Home Assistant supports OIDC via the `generic_oauth` auth provider. Create the client in Kanidm:

```bash
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm system oauth2 create --name admin \
    homeassistant "Home Assistant" "https://ha.in.alybadawy.com/auth/oidc/callback"

docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm system oauth2 update-scope-map --name admin \
    homeassistant homelab_users openid profile email

docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm system oauth2 show-basic-secret --name admin homeassistant
```

In Home Assistant `configuration.yaml`:

```yaml
homeassistant:
  auth_providers:
    - type: homeassistant
    - type: oidc
      client_id: homeassistant
      client_secret: <secret from show-basic-secret>
      discovery_url: https://id.in.alybadawy.com/oauth2/openid/homeassistant/.well-known/openid-configuration
```

### Portainer

Portainer CE uses local admin accounts — OAuth2 is a Business Edition feature. For CE, create a local admin account in Portainer and skip OAuth2 configuration. NPM and Netdata are accessed with their own local admin credentials and do not require Kanidm integration.

If you upgrade to Portainer BE, create the client:

```bash
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm system oauth2 create --name admin \
    portainer "Portainer" "https://dockers.in.alybadawy.com"

docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm system oauth2 update-scope-map --name admin \
    portainer homelab_admins openid profile email

docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm system oauth2 show-basic-secret --name admin portainer
```

In Portainer → **Settings → Authentication → OAuth**:

| Field             | Value                                                          |
| ----------------- | -------------------------------------------------------------- |
| Client ID         | `portainer`                                                    |
| Client Secret     | _(from `show-basic-secret`)_                                   |
| Authorization URL | `https://id.in.alybadawy.com/ui/oauth2`                        |
| Access Token URL  | `https://id.in.alybadawy.com/oauth2/token`                     |
| Resource URL      | `https://id.in.alybadawy.com/oauth2/openid/portainer/userinfo` |
| Redirect URL      | `https://dockers.in.alybadawy.com`                             |
| User identifier   | `preferred_username`                                           |

---

## Verification Checklist

- [ ] Kanidm running: `docker ps | grep kanidm`
- [ ] Kanidm web UI loads: `https://id.in.alybadawy.com`
- [ ] LDAPS reachable from host: `openssl s_client -connect 172.20.20.5:636 -servername id.in.alybadawy.com`
- [ ] Users and groups created with POSIX enabled
- [ ] NAS Domain user tab shows Kanidm users
- [ ] NAS Domain user group tab shows Kanidm groups
- [ ] Nextcloud LDAP integration configured and users visible
- [ ] Immich OIDC login works end-to-end
- [ ] Home Assistant OAuth2 configured

---

## Troubleshooting

**Kanidm won't start — TLS error:**

```bash
ls -la /opt/certs/
docker logs kanidm | grep -i "tls\|cert\|error"
```

Verify the cert files exist and are readable:

```bash
openssl x509 -in /opt/certs/fullchain.pem -text -noout | grep -E "Subject:|Not After"
```

**NAS shows no users after connecting:**

Users must have POSIX enabled in Kanidm — non-POSIX accounts are not exposed over LDAP to POSIX clients. Check:

```bash
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  kanidm person posix show --name idm_admin <username>
```

If no POSIX attributes are shown, run `kanidm person posix set` for that user (Section 1, Step 7).

**LDAPS connection refused from NAS or Nextcloud:**

```bash
# From homelab host:
openssl s_client -connect 172.20.20.5:636 -servername id.in.alybadawy.com

# Check UFW:
sudo ufw status | grep 636
```

**Kanidm CLI — "Is a directory" error for `/root/.config/kanidm`:**

This happens when Docker creates the config path as a directory instead of a file. Fix inside the running tools container:

```bash
docker run --rm -it \
  -e KANIDM_URL=https://id.in.alybadawy.com \
  kanidm/tools:latest \
  sh

# Inside the container:
rm -rf /root/.config/kanidm
mkdir -p /root/.config
printf 'uri = "https://id.in.alybadawy.com"\nverify_ca = true\n' > /root/.config/kanidm
kanidm login --name idm_admin
```

**Cert not updated after renewal:**

The deploy hook at `/opt/certs/deploy-cert.sh` restarts all running containers automatically. Verify it ran:

```bash
cat /opt/certs/deploy.log
```

Force a manual renewal test:

```bash
~/.acme.sh/acme.sh --renew -d "in.alybadawy.com" --force
```
