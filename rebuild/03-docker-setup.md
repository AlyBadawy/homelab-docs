# Docker Installation and Configuration Guide

**OS:** Ubuntu Server 24.04 LTS (post-installation)
**Docker Version:** Latest CE (Community Edition)
**Setup:** Docker networks, daemon configuration, stack directory structure

This guide covers Docker CE installation from the official repository, daemon configuration, and preparation of the stack directory structure for homelab services.

---

## 1. Install Docker CE

Ubuntu repositories may carry outdated Docker versions. Install from the official Docker repository to get the latest release.

### Step 1a: Add Docker's GPG Key and Repository

```bash
# Update package manager
sudo apt-get update

# Install required certificates and curl
sudo apt-get install -y ca-certificates curl

# Create directory for Docker's GPG key
sudo install -m 0755 -d /etc/apt/keyrings

# Download and save Docker's GPG key
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

# Make the key readable
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### Step 1b: Add Docker Repository

```bash
# Add Docker's official repository to apt sources
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Step 1c: Install Docker Packages

```bash
# Update package manager with Docker repository
sudo apt-get update

# Install Docker Engine, CLI, and plugins
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**Package descriptions:**
- `docker-ce`: Docker Engine (core runtime)
- `docker-ce-cli`: Docker command-line client
- `containerd.io`: Container runtime (dependency)
- `docker-buildx-plugin`: Extended build capabilities
- `docker-compose-plugin`: Docker Compose v2 (native plugin, faster than standalone)

### Step 1d: Add User to Docker Group

```bash
# Add current user to docker group (allows running docker without sudo)
sudo usermod -aG docker $USER

# Apply group changes (either logout/login or run:)
newgrp docker
```

**Security Note:** Users in the docker group can access the Docker daemon socket. Treat membership like sudo access. Only add trusted users.

### Step 1e: Verify Installation

```bash
# Test Docker installation
docker run hello-world
```

Expected output:
```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
...
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

Also verify Compose:

```bash
docker compose version
```

Expected output:
```
Docker Compose version v2.x.x
```

---

## 2. Configure Docker Daemon

Configure Docker for optimized logging and performance.

### Step 2a: Create daemon.json

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
```

**Configuration explanation:**
- `log-driver: json-file`: Use JSON file logging (standard for Docker logs)
- `max-size: 10m`: Individual log files max 10MB
- `max-file: 3`: Keep only 3 rotated log files (total max ~30MB per container)
- `userland-proxy: false`: Disable userland proxy for better performance (rely on iptables rules instead)

### Step 2b: Reload Docker Configuration

```bash
# Restart Docker daemon to apply changes
sudo systemctl restart docker

# Verify the service is running
sudo systemctl status docker
```

Expected output: `Active: active (running)`

---

## 3. Create Docker Networks

Create the three bridge networks for service isolation and communication:

```bash
# Proxy network (for reverse proxy and external-facing services)
docker network create proxy

# Identity network (for authentication services and backends)
docker network create identity

# Apps network (for application services and their backends)
docker network create apps
```

Verify networks were created:

```bash
docker network ls
```

Expected output:
```
NETWORK ID     NAME      DRIVER    SCOPE
xxxxxxxx       proxy     bridge    local
xxxxxxxx       identity  bridge    local
xxxxxxxx       apps      bridge    local
xxxxxxxx       bridge    bridge    local
xxxxxxxx       host      host      local
xxxxxxxx       none      null      local
```

**Network isolation strategy:**
- `proxy`: Contains NPM, Portainer, Netdata (externally exposed services)
- `identity`: Contains LLDAP, Authentik (authentication backends)
- `apps`: Contains Nextcloud, Immich, Home Assistant (user applications)
- Services requiring cross-network communication connect to multiple networks (see services.md)

---

## 4. Create Stack Directory Structure

Create the organized directory hierarchy for all service docker-compose files:

```bash
# Create all directories at once
sudo mkdir -p /opt/stacks/{core/{portainer,netdata},proxy/npm/certs,db/{postgres/init,redis},identity/{lldap,authentik},apps/{nextcloud,immich,homeassistant}}

# Change ownership to the current user (for easier editing)
sudo chown -R $USER:$USER /opt/stacks
```

### Directory Structure

```
/opt/stacks/
├── core/                          # Core infrastructure services
│   ├── portainer/
│   │   └── docker-compose.yml     # Container management UI
│   └── netdata/
│       └── docker-compose.yml     # System monitoring
├── proxy/                         # Reverse proxy layer
│   └── npm/
│       ├── docker-compose.yml     # Nginx Proxy Manager
│       └── certs/                 # TLS certificates (acme.sh deploys here)
├── db/                            # Shared database services
│   ├── postgres/
│   │   ├── docker-compose.yml     # Global PostgreSQL 16
│   │   └── init/
│   │       └── 01-databases.sql   # Multi-database initialization
│   └── redis/
│       └── docker-compose.yml     # Global Redis cache
├── identity/                      # Authentication and authorization
│   ├── lldap/
│   │   ├── docker-compose.yml     # LDAP directory
│   │   └── .env                   # Secrets (not in git)
│   └── authentik/
│       ├── docker-compose.yml     # OIDC/OAuth2 provider
│       └── .env                   # Secrets (not in git)
└── apps/                          # User-facing applications
    ├── nextcloud/
    │   ├── docker-compose.yml     # File sync and collaboration
    │   └── .env                   # Secrets (not in git)
    ├── immich/
    │   ├── docker-compose.yml     # Photo/video library
    │   └── .env                   # Secrets (not in git)
    └── homeassistant/
        └── docker-compose.yml     # Home automation
```

Verify the structure was created:

```bash
tree /opt/stacks/
# or
find /opt/stacks -type d | sort
```

### Set Appropriate Permissions

Ensure the directory is owned by your user for easy editing, but protected from unauthorized access:

```bash
# Verify ownership
ls -ld /opt/stacks
# Output should show: drwxr-xr-x user:user /opt/stacks

# Optional: restrict to user only (remove group/other read)
# chmod 700 /opt/stacks
```

---

## 5. Verify Docker Compose v2 Functionality

Test Docker Compose with a simple service:

```bash
# Create a test directory
mkdir -p /tmp/docker-test
cd /tmp/docker-test

# Create a minimal docker-compose.yml
cat > docker-compose.yml <<'EOF'
version: '3.8'
services:
  test:
    image: alpine:latest
    command: echo "Docker Compose is working!"
EOF

# Test the compose file
docker compose config

# Run the test service
docker compose run --rm test

# Clean up
cd /tmp
rm -rf /tmp/docker-test
```

Expected output: Should show "Docker Compose is working!" without errors.

---

## 6. Enable Docker Restart Policy

To ensure Docker services survive system reboots and daemon restarts, configure the restart policy at container creation time.

### Recommended Restart Policy

In each service's `docker-compose.yml`, add the restart policy:

```yaml
services:
  myservice:
    image: some-image:latest
    restart_policy:
      condition: always
      delay: 5s
      max_attempts: 3
      window: 120s
```

Or the shorthand (equivalent to always):

```yaml
services:
  myservice:
    image: some-image:latest
    restart: always
```

**Restart policy options:**
- `no`: Do not automatically restart (default)
- `always`: Always restart unless explicitly stopped
- `unless-stopped`: Always restart unless explicitly stopped
- `on-failure`: Restart only if container exits with non-zero code

For homelab services, `restart: always` is recommended for critical infrastructure (databases, proxy, auth) and optional for applications.

---

## 7. Configure Log Rotation

Verify Docker's log rotation is working correctly:

```bash
# Check Docker logs directory
ls -lh /var/lib/docker/containers/*/

# View a container's logs
docker logs <container-id>

# Follow logs in real-time
docker logs -f <container-id>

# View logs with timestamps
docker logs --timestamps <container-id>
```

The `daemon.json` configuration (Step 2) ensures logs don't consume excessive disk space.

---

## 8. Optional: Configure Docker Registry Mirror

If you're behind a slow/blocked Docker Hub connection, configure a registry mirror (e.g., Aliyun, DaoCloud, or your own):

```bash
# Edit daemon.json
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "userland-proxy": false,
  "registry-mirrors": [
    "https://mirror.aliyun.com"
  ]
}
EOF

# Restart Docker
sudo systemctl restart docker
```

---

## 9. Set Up Stack Deployment with Portainer

Once Docker is running, you'll deploy services in this order:

1. **Core Infrastructure** → Deploy Portainer and Netdata
2. **Databases** → Deploy PostgreSQL and Redis
3. **Proxy** → Deploy NPM (Nginx Proxy Manager)
4. **Identity** → Deploy LLDAP and Authentik
5. **Applications** → Deploy Nextcloud, Immich, Home Assistant

### Portainer as Primary Orchestration

Portainer will serve as the primary interface for managing stacks:

- **Docker Containers/Stacks** section: View, manage, restart services
- **Stacks** section: Deploy from `docker-compose.yml` files by specifying the git repo (if using git) or the `/opt/stacks/` directory
- **Volumes** section: Manage persistent data
- **Networks** section: View network connections
- **Events/Logs** section: Monitor container activity

Each docker-compose.yml in `/opt/stacks/*/` can be deployed as a Portainer stack through the UI after Portainer is running.

---

## 10. Verify Docker and Stack Readiness

Run these final checks before deploying services:

```bash
# Verify Docker daemon is running
sudo systemctl status docker

# List Docker networks
docker network ls

# List created directories
ls -la /opt/stacks/

# Verify permissions
touch /opt/stacks/test.txt && rm /opt/stacks/test.txt && echo "Write permission OK"

# Check disk space available
df -h /opt

# Check available memory
free -h
```

Expected results:
- Docker service: `Active: active (running)`
- Networks: `proxy`, `identity`, `apps` present
- Stacks directory: All subdirectories present, owned by your user
- Write permission: Test file creation succeeds
- Disk space: At least 50GB free recommended
- Memory: At least 4GB free (for running containers)

---

## Troubleshooting

**Problem: Docker command fails with "permission denied"**

You likely need to log out and back in for the docker group change to take effect:

```bash
# Verify your user is in docker group
groups $USER
# Should output: user ... docker ...

# If not listed, try:
newgrp docker

# Or log out and back in
exit
# Then log back in and test
docker run hello-world
```

**Problem: Docker daemon fails to start after daemon.json edit**

The JSON may have syntax errors:

```bash
# Validate the JSON
cat /etc/docker/daemon.json | jq .

# If errors, restore a backup or recreate the file carefully
sudo rm /etc/docker/daemon.json
sudo systemctl restart docker
```

**Problem: Networks already exist error**

If you get "network already exists" when creating networks, they're already created (likely from a previous attempt). This is not a problem. Verify with `docker network ls`.

**Problem: Cannot access /opt/stacks as regular user**

Fix permissions:

```bash
sudo chown -R $USER:$USER /opt/stacks
sudo chmod 755 /opt/stacks
```

**Problem: Docker running out of disk space**

Check and clean up unused images/containers:

```bash
# Show disk usage
docker system df

# Remove unused containers/images/networks
docker system prune -a

# Remove unused volumes
docker volume prune
```

---

## Next Steps

Docker is now ready for service deployment. Proceed with creating docker-compose.yml files for:

1. **Portainer** → management interface
2. **PostgreSQL + Redis** → database backends
3. **NPM** → reverse proxy and TLS termination
4. **LLDAP + Authentik** → authentication
5. **Nextcloud, Immich, Home Assistant** → applications

Refer to **services.md** for service specifications and network memberships.
