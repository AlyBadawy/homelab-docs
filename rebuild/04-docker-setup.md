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

| Network    | Purpose                                                      |
| ---------- | ------------------------------------------------------------ |
| `proxy`    | Reverse proxy and externally-exposed services                |
| `identity` | Authentication services (Kanidm) and their backends          |
| `apps`     | User-facing applications (Nextcloud, Immich, Home Assistant) |

Verify:

```bash
docker network ls
```

---

## Part 5 — Verify

```bash
# Docker daemon running and enabled
sudo systemctl status docker
sudo systemctl is-enabled docker

# All three networks present
docker network ls | grep -E "proxy|identity|apps"
```

Expected: docker is `active (running)` and `enabled`; all three networks listed.

---

## Part 6 — Reboot Test

Verify Docker survives a reboot before adding any services. This is critical — all future services depend on Docker auto-starting reliably.

```bash
# Confirm Docker is enabled
sudo systemctl is-enabled docker   # → enabled

# Reboot
sudo reboot
```

After the homelab comes back up (~30–60 seconds), SSH back in:

```bash
sudo systemctl status docker
docker network ls | grep -E "proxy|identity|apps"
```

Expected: Docker is running, all three networks are present.

---

## Troubleshooting

**`permission denied` running docker commands**

The docker group change requires a new session to take effect:

```bash
newgrp docker
# or log out and back in
```

**`docker daemon fails to start` after editing daemon.json**

```bash
# Validate JSON syntax
cat /etc/docker/daemon.json | python3 -m json.tool

# If invalid, remove and recreate
sudo rm /etc/docker/daemon.json
sudo systemctl restart docker
```
