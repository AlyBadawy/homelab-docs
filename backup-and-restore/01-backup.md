# Homelab Backup Strategy

## 🎯 Goal

Create a **weekly, incremental backup** of the homelab server to an
NFS-mounted NAS while preserving:

-   System identity (hostname)
-   Users & passwords
-   SSH keys + server identity (host keys)
-   Firewall rules (UFW)
-   Docker containers and volumes
-   System configuration

Backups retain **last 4 snapshots only**.

## 📁 Backup Location

NAS mount:

    /mnt/nas/homelab

Backup directory:

    /mnt/nas/homelab/backups/

Snapshot format:

    backup-YYYY-MM-DD/

## 📦 What Gets Backed Up

### Critical system state

-   `/etc`
-   `/home`
-   `/root`

### Docker

-   `/var/lib/docker`

## ⚙️ Backup Script

Location:

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
)

LAST_BACKUP=$(ls -1dt $BACKUP_ROOT/backup-* 2>/dev/null | head -n 1)

mkdir -p "$DEST"

RSYNC_OPTS="-aAX --delete --numeric-ids"

if [ -n "$LAST_BACKUP" ]; then
  LINK_DEST="--link-dest=$LAST_BACKUP"
else
  LINK_DEST=""
fi

docker stop $(docker ps -q) || true

for SRC in "${SOURCES[@]}"; do
  rsync $RSYNC_OPTS $LINK_DEST "$SRC" "$DEST/"
done

docker start $(docker ps -aq) || true

ls -1dt $BACKUP_ROOT/backup-* | tail -n +5 | xargs -r rm -rf
```
