# 02 — NAS Mounts

> **Run this guide before starting Docker.** The NFS mounts must be in place before any container that depends on them is started. This guide covers NAS-side configuration, homelab-side fstab setup, resilience tuning, and verification.

**Prerequisites:** Guide 01 (OS Installation) complete — Ubuntu is installed, static IP is set, `nfs-common` is installed.

---

## Overview

| What | Where |
|------|-------|
| NAS IP | `172.20.20.10` (Servers VLAN) |
| NAS shared folder | `homelab` (contains `cloud/`, `immich/`, `media/` subfolders) |
| NAS export | `/homelab` |
| Homelab mount point | `/mnt/nas/homelab` |
| Protocol | NFSv4 |
| Mount strategy | `x-systemd.automount` — lazy mount, survives NAS downtime at boot |

---

## Part 1 — NAS Configuration

> Perform these steps in your NAS admin web UI. The exact menus differ slightly between UGreen NAS and UniFi UNAS 4, but the concepts are identical.

### 1.1 Create Shared Folder and Subfolders

Create one shared folder on the NAS:

| Folder Name | Purpose |
|-------------|---------|
| `homelab` | Parent folder for all homelab storage |

After the first mount (or via the NAS admin UI), create three subfolders within `homelab`:

| Subfolder | Purpose |
|-----------|---------|
| `homelab/cloud` | Nextcloud user data |
| `homelab/immich` | Immich photo/video library |
| `homelab/media` | General media (future use) |

Leave permissions at default for now — you'll set ownership after the first mount.

### 1.2 Enable NFS Service

In the NAS admin panel, navigate to **Services → NFS** (or **File Services → NFS**) and enable the NFS server. Enable **NFSv4** specifically.

### 1.3 Configure NFS Export

Add one NFS rule for the `homelab` shared folder with the following settings:

| Setting | Value |
|---------|-------|
| **Allowed host** | `172.20.20.5` (homelab only) |
| **Permissions** | Read/Write |
| **Sync** | Synchronous |
| **Root squash** | No root squash |
| **Subtree check** | No subtree check |

> **On `no_root_squash`:** Normally NFS maps root (UID 0) inside a container to an unprivileged `nobody` user on the NAS, which breaks Docker container writes. Since access is restricted to `172.20.20.5` on a private VLAN, disabling root squash is a reasonable trade-off. The alternative is configuring UID/GID mappings, which adds complexity.

If you're configuring via the NAS terminal (SSH), the `/etc/exports` entry looks like:

```
/homelab   172.20.20.5(rw,sync,no_subtree_check,no_root_squash)
```

After saving, apply the exports:
```bash
# On the NAS (if SSH access available)
sudo exportfs -ra
```

### 1.4 Verify Exports Are Visible

From the **homelab**, check that the NAS is advertising its exports:

```bash
showmount -e 172.20.20.10
```

Expected output:
```
Export list for 172.20.20.10:
/homelab   172.20.20.5
```

If `showmount` hangs or returns nothing, check that the NFS service is running on the NAS and that firewall rules on the NAS allow the homelab IP on TCP/UDP port 2049.

---

## Part 2 — Homelab Mount Setup

### 2.1 Install NFS Client

```bash
sudo apt install -y nfs-common
```

### 2.2 Create Mount Point

```bash
sudo mkdir -p /mnt/nas/homelab
```

### 2.3 Test Manual Mount First

Before adding to fstab, verify the share mounts manually:

```bash
sudo mount -t nfs4 172.20.20.10:/homelab /mnt/nas/homelab
ls /mnt/nas/homelab        # should show cloud, immich, media subfolders
sudo umount /mnt/nas/homelab
```

If the manual mount fails, resolve it before continuing. Common issues:
- NFS service not started on NAS
- Export not configured for `172.20.20.5`
- NAS firewall blocking port 2049

### 2.4 Add Persistent fstab Entry

Open fstab:
```bash
sudo nano /etc/fstab
```

Append this line at the bottom:

```
# NAS NFS mount (172.20.20.10 — Servers VLAN)
172.20.20.10:/homelab  /mnt/nas/homelab  nfs4  _netdev,nofail,x-systemd.automount,x-systemd.mount-timeout=30,soft,timeo=100,retrans=3,rsize=131072,wsize=131072,noatime  0 0
```

**What each option does:**

| Option | Effect |
|--------|--------|
| `nfs4` | Explicitly use NFSv4 |
| `_netdev` | Defer mount until network is up (critical for boot order) |
| `nofail` | If NAS is unreachable at boot, continue booting — don't hang |
| `x-systemd.automount` | Register a lazy automount unit; actual connection made on first access |
| `x-systemd.mount-timeout=30` | If mount attempt takes more than 30 seconds, fail gracefully |
| `soft` | NFS I/O operations return an error after timeout (vs. blocking forever) |
| `timeo=100` | NFS client waits 10 seconds per operation before retrying (units of 0.1s) |
| `retrans=3` | Retry a failed NFS operation 3 times before giving up |
| `rsize=131072,wsize=131072` | 128 KB transfer blocks — well-suited for 2.5 GbE |
| `noatime` | Do not update file access timestamps — reduces unnecessary NAS writes |

### 2.5 Reload systemd and Activate Automount

After saving fstab, tell systemd to regenerate units from it:

```bash
sudo systemctl daemon-reload
```

Activate the automount unit (without rebooting):

```bash
sudo systemctl start mnt-nas-homelab.automount
```

> **Note:** Systemd derives unit names from mount paths by replacing `/` with `-`. So `/mnt/nas/homelab` becomes `mnt-nas-homelab.automount`.

### 2.6 Trigger and Verify the Mount

Access the path to trigger the automount:

```bash
ls /mnt/nas/homelab
```

Verify the mount is live:

```bash
mount | grep nfs
# Expected output:
# 172.20.20.10:/homelab on /mnt/nas/homelab type nfs4 (rw,relatime,...)
```

Verify subfolders exist:

```bash
ls /mnt/nas/homelab/cloud /mnt/nas/homelab/immich /mnt/nas/homelab/media
```

Check disk space is visible from the NAS:

```bash
df -h /mnt/nas/homelab
```

---

## Part 3 — Set File Ownership on NAS

Now that the share is mounted, set the correct ownership so Docker containers can write to their respective subfolders. First, create the subfolders if they don't already exist:

```bash
sudo mkdir -p /mnt/nas/homelab/{cloud,immich,media}
```

Then set ownership:

```bash
# Nextcloud container writes as www-data (UID 33)
sudo chown -R 33:33 /mnt/nas/homelab/cloud
sudo chmod -R 755 /mnt/nas/homelab/cloud

# Immich container writes as UID 1000 (node user)
sudo chown -R 1000:1000 /mnt/nas/homelab/immich
sudo chmod -R 755 /mnt/nas/homelab/immich

# Media — owned by current user for now, revisit when media server is added
sudo chown -R $USER:$USER /mnt/nas/homelab/media
sudo chmod -R 755 /mnt/nas/homelab/media
```

Verify:

```bash
ls -la /mnt/nas/homelab/
```

---

## Part 4 — Docker Dependency on Network

Docker must start after the network is fully up so that automount units can function when containers first access NAS paths. By default Ubuntu's Docker service depends on `network.target`, which is weaker than `network-online.target`. Strengthen this:

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

This ensures Docker (and therefore all containers) only starts once the network stack is fully operational — including DNS resolution and routing to the NAS subnet.

Verify the override is active:

```bash
systemctl show docker | grep -E "After|Wants" | tr ' ' '\n' | grep network
```

---

## Part 5 — Resilience Testing

### Test 1: Boot Without NAS

1. Shut down the NAS
2. Reboot the homelab
3. Confirm the homelab boots fully and Docker starts:
   ```bash
   systemctl status docker
   docker ps
   ```
4. Confirm the automount unit is registered (not mounted):
   ```bash
   systemctl status mnt-nas-homelab.automount
   # Should show: active (waiting)
   ```
5. Try accessing a mount point — it should fail quickly (not hang):
   ```bash
   timeout 35 ls /mnt/nas/homelab
   # Should return I/O error within ~30 seconds
   ```

### Test 2: NAS Comes Back While Homelab is Running

1. Start the NAS while the homelab is running
2. Wait for the NAS to fully boot (~2–3 minutes)
3. Trigger re-mount:
   ```bash
   ls /mnt/nas/homelab
   ```
4. Verify it's mounted:
   ```bash
   mount | grep nfs
   ```
5. Restart any affected containers:
   ```bash
   docker restart nextcloud immich-server
   ```

### Test 3: NAS Goes Down During Operation

1. While Nextcloud/Immich are running, shut down the NAS
2. Observe: containers stay running, but file operations fail
3. Check container logs:
   ```bash
   docker logs nextcloud --tail 20
   docker logs immich-server --tail 20
   ```
4. Confirm the homelab itself is unaffected (auth, dashboard, HA still work)
5. Bring NAS back up, re-trigger mount, restart containers

---

## Part 6 — Monitoring Mounts

Add these checks to your routine health checks (or Netdata alerts):

```bash
# Check if the NAS mount is active
if mountpoint -q /mnt/nas/homelab; then
  echo "✓ /mnt/nas/homelab is mounted"
else
  echo "✗ /mnt/nas/homelab is NOT mounted"
fi
```

Verify subfolder accessibility:

```bash
ls /mnt/nas/homelab/cloud /mnt/nas/homelab/immich
```

Check automount unit status:

```bash
systemctl status mnt-nas-homelab.automount
```

Check NFS statistics (useful for diagnosing performance):

```bash
nfsstat -c
```

---

## Troubleshooting

### Mount hangs at boot

Symptom: Boot takes a very long time (>2 minutes) or drops to emergency shell.

Cause: `nofail` may not be set, or `_netdev` is missing.

Fix: Double-check fstab entry has both `nofail` and `_netdev`. Run `sudo mount -fav` to dry-run fstab without actually mounting and check for errors.

---

### `mount: /mnt/nas/homelab: bad option; for several filesystems (e.g. nfs, cifs) you might need a /sbin/mount.helper program`

Cause: `nfs-common` not installed.

Fix:
```bash
sudo apt install -y nfs-common
```

---

### Permission denied when container writes to NAS

Symptom: Container logs show `Permission denied` on `/var/www/html/data` (Nextcloud) or `/usr/src/app/upload` (Immich).

Cause: NAS file ownership doesn't match the container UID.

Fix: Re-run the ownership commands from Part 3, or on the NAS side set `all_squash,anonuid=33,anongid=33` for Nextcloud.

---

### NFS mount drops after period of inactivity

Symptom: NFS mount disappears after hours of no access.

Cause: NAS goes into sleep/idle mode and drops connections.

Fix on NAS side: Disable drive sleep/idle timeout, or increase it. Alternatively, add a keepalive cron on the homelab:

```bash
# Add to crontab: touch NAS paths every 15 minutes to prevent idle disconnect
*/15 * * * * ls /mnt/nas/homelab > /dev/null 2>&1
```

---

### After NAS reboot, mount doesn't auto-recover

Symptom: After NAS comes back up, `ls /mnt/nas/homelab` still fails.

Fix: Restart the automount unit to reset its state:

```bash
sudo systemctl restart mnt-nas-homelab.automount
ls /mnt/nas/homelab  # this triggers the fresh mount attempt
```

---

## Reference: fstab Entry Template

```
# Format:
# <server>:<export>  <mountpoint>  <fstype>  <options>  <dump>  <pass>
#
172.20.20.10:/homelab  /mnt/nas/homelab  nfs4  _netdev,nofail,x-systemd.automount,x-systemd.mount-timeout=30,soft,timeo=100,retrans=3,rsize=131072,wsize=131072,noatime  0 0
```

> ⚠️ After any fstab change, always run `sudo mount -fav` (dry run) before `sudo mount -a` (live) to catch typos that could make the system unbootable.
