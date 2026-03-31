# Homelab Backup Strategy (v3)

## 🎯 Goal

Reliable weekly backups including: - System identity (hostname, SSH host
keys) - Users, passwords - Cron jobs (system + user) - Docker data -
Firewall rules

------------------------------------------------------------------------

## ⚙️ Prerequisites

``` bash
sudo apt update
sudo apt install -y rsync docker.io
```

Ensure NAS mounted at:

    /mnt/nas/homelab

------------------------------------------------------------------------

## 📦 Backup Scope

-   /etc
-   /home
-   /root
-   /var/lib/docker
-   /var/spool/cron

------------------------------------------------------------------------

## 🛠️ Script

    /usr/local/bin/homelab-backup.sh

``` bash
#!/bin/bash
set -e

BACKUP_ROOT="/mnt/nas/homelab/backups"
DATE=$(date +%F)
DEST="$BACKUP_ROOT/backup-$DATE"

SOURCES=(
  "/etc"
  "/home"
  "/root"
  "/var/lib/docker"
  "/var/spool/cron"
)

# Pre-flight checks
command -v rsync >/dev/null || { echo "rsync not installed"; exit 1; }
mountpoint -q /mnt/nas || { echo "NAS not mounted"; exit 1; }

LAST_BACKUP=$(ls -1dt $BACKUP_ROOT/backup-* 2>/dev/null | head -n 1)

mkdir -p "$DEST"

RSYNC_OPTS="-aAX --delete --numeric-ids"

[ -n "$LAST_BACKUP" ] && LINK_DEST="--link-dest=$LAST_BACKUP" || LINK_DEST=""

echo "Stopping Docker..."
docker stop $(docker ps -q) || true

for SRC in "${SOURCES[@]}"; do
  rsync $RSYNC_OPTS $LINK_DEST "$SRC" "$DEST/"
done

echo "Starting Docker..."
docker start $(docker ps -aq) || true

# Save crontabs (readable backup)
crontab -l > "$DEST/root-crontab.txt" 2>/dev/null || true
for user in $(cut -f1 -d: /etc/passwd); do
  crontab -u $user -l > "$DEST/cron-$user.txt" 2>/dev/null || true
done

# Retention
ls -1dt $BACKUP_ROOT/backup-* | tail -n +5 | xargs -r rm -rf

echo "Backup complete"
```

Make executable:

``` bash
sudo chmod +x /usr/local/bin/homelab-backup.sh
```

------------------------------------------------------------------------

## ⏱️ Cron

``` bash
sudo crontab -e
```

    0 3 * * 0 /usr/local/bin/homelab-backup.sh >> /var/log/homelab-backup.log 2>&1

------------------------------------------------------------------------

## 🧪 Test

``` bash
sudo /usr/local/bin/homelab-backup.sh
```
