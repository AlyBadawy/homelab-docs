# 03 — Docker and Portainer

This guide installs Docker, configures it for the homelab environment, and bootstraps Portainer. After this guide, all further services are deployed and managed through the Portainer UI.

**Prerequisites:** Guide 02 (NAS Mounts) complete — `/mnt/nas/homelab` is mounted and accessible.

---

## Part 1 — Install Docker CE

Ubuntu repositories carry outdated Docker versions. Install from the official Docker repository.

### 1.1 Add Docker's GPG Key and Repository

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 1.2 Install Docker Packages

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 1.3 Add User to Docker Group

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### 1.4 Verify Installation

```bash
docker run hello-world
docker compose version
```

Expected: `Hello from Docker!` and a Compose version string.

---

## Part 2 — Configure Docker Daemon

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "userland-proxy": false
}
EOF

sudo systemctl restart docker
sudo systemctl status docker
```

- `max-size` / `max-file` — Log rotation; caps each container at ~30 MB of logs
- `userland-proxy: false` — Better performance via iptables rules instead of userland proxy

---

## Part 3 — Configure Docker Network Dependency

Docker must start after the network is fully up so that NFS automount units can function when containers first access NAS paths. By default Ubuntu's Docker service depends on `network.target`, which is weaker than `network-online.target`. Strengthen this:

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/network-wait.conf > /dev/null << 'EOF'
[Unit]
After=network-online.target
Wants=network-online.target
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
```

Verify the override is active:

```bash
systemctl show docker | grep -E "After|Wants" | tr ' ' '\n' | grep network
```

---

## Part 4 — Create Docker Networks

Create the three bridge networks used for service isolation:

```bash
# Proxy network (for reverse proxy and external-facing services)
docker network create proxy

# Identity network (for authentication services and backends)
docker network create identity

# Apps network (for application services and their backends)
docker network create apps
```

| Network    | Purpose                                                       |
| ---------- | ------------------------------------------------------------- |
| `proxy`    | Reverse proxy and externally-exposed services                 |
| `identity` | Authentication services (LLDAP, Authentik) and their backends |
| `apps`     | User-facing applications (Nextcloud, Immich, Home Assistant)  |

Verify:

```bash
docker network ls
```

---

## Part 5 — Create Stack Directory Structure

```bash
sudo mkdir -p /opt/stacks/{core/{portainer,netdata},proxy/npm/certs,db/{postgres/init,redis},identity/{lldap,authentik},apps/{nextcloud,immich,homeassistant}}
sudo chown -R $USER:$USER /opt/stacks
```

```
/opt/stacks/
├── core/
│   ├── portainer/
│   └── netdata/
├── proxy/
│   └── npm/
│       └── certs/          ← acme.sh deploys wildcard cert here
├── db/
│   ├── postgres/
│   │   └── init/
│   └── redis/
├── identity/
│   ├── lldap/
│   └── authentik/
└── apps/
    ├── nextcloud/
    ├── immich/
    └── homeassistant/
```

---

## Part 6 — Install Portainer

Portainer is the only service bootstrapped from the CLI. Everything else is deployed through its UI after this step.

### 6.1 Create the Compose File

```bash
cat > /opt/stacks/core/portainer/docker-compose.yml << 'EOF'
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      - proxy

volumes:
  portainer_data:

networks:
  proxy:
    external: true
EOF
```

### 6.2 Start Portainer

```bash
cd /opt/stacks/core/portainer
docker compose up -d
```

Verify:

```bash
docker ps | grep portainer
```

### 6.3 Initial Setup

Open in your browser:

```
http://172.20.20.5:9000
```

> Portainer prompts you to create an admin account on first access. If you wait more than a few minutes, it locks down for security and you'll need to restart the container: `docker restart portainer`

1. Enter an admin username and a strong password
2. Click **Create user**
3. On the next screen, select **Get Started** (local Docker environment)

Portainer will now show the local Docker environment with the networks, volumes, and containers you've already created.

---

## Part 7 — Verify

```bash
# Docker daemon
sudo systemctl status docker

# Networks
docker network ls | grep -E "proxy|identity|apps"

# Directories
ls /opt/stacks/

# Portainer
docker ps --format "table {{.Names}}\t{{.Status}}" | grep portainer
```

All three networks should be present, `/opt/stacks/` should show all subdirectories, and Portainer should show as `Up`.

---

## Part 8 — Reboot Test

Before moving on, verify that Docker and Portainer survive a reboot automatically. This is critical — all future services depend on this being reliable.

### 8.1 Pre-reboot Checks

Confirm Docker is enabled to start on boot and Portainer has the correct restart policy:

```bash
# Should return: enabled
sudo systemctl is-enabled docker

# Should return: unless-stopped
docker inspect portainer --format '{{.HostConfig.RestartPolicy.Name}}'
```

### 8.2 Reboot

```bash
sudo reboot
```

### 8.3 Post-reboot Verification

After the homelab comes back up (allow ~30–60 seconds), SSH back in and run:

```bash
# Docker started automatically
sudo systemctl status docker

# Portainer is running (check Status column — should say "Up X seconds")
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Networks survived (Docker recreates them on start, but verify)
docker network ls | grep -E "proxy|identity|apps"
```

Then open Portainer in your browser to confirm it's fully functional:

```
http://172.20.20.5:9000
```

### 8.4 What to Expect

| Check | Expected result |
|-------|----------------|
| `systemctl is-enabled docker` | `enabled` |
| `docker ps` shows portainer | `Up X seconds` (not `Exited`) |
| All three networks present | `proxy`, `identity`, `apps` |
| Portainer UI accessible | Login page loads at `http://172.20.20.5:9000` |

> If Portainer shows `Exited` after reboot, the restart policy wasn't applied. Fix with: `docker update --restart unless-stopped portainer && sudo reboot`

---

## Troubleshooting

**`permission denied` running docker commands**

The docker group change requires a new session to take effect:

```bash
newgrp docker
# or log out and back in
```

**Portainer locks down before you set the admin password**

```bash
docker restart portainer
```

Then immediately open `http://172.20.20.5:9000` and complete setup.

**`docker daemon fails to start` after editing daemon.json**

```bash
# Validate JSON syntax
cat /etc/docker/daemon.json | python3 -m json.tool

# If invalid, remove and recreate
sudo rm /etc/docker/daemon.json
sudo systemctl restart docker
```

---

## Next Steps

Portainer is running at `http://172.20.20.5:9000`. All remaining services — NPM, Netdata, PostgreSQL, Redis, LLDAP, Authentik, Nextcloud, Immich, and Home Assistant — are deployed through Portainer's **Stacks** interface.

Proceed to **Guide 04** to issue the wildcard SSL certificate before deploying any web services.
