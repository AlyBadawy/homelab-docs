# Homelab Restore Guide

## 🎯 Goal

Restore a homelab server such that it is **identical to the original**.

## 🧱 Step 1: Install Base System

Install Ubuntu Server 24.04.

## 📡 Step 2: Mount NAS

``` bash
sudo mount -t nfs <nas-ip>:/path /mnt/nas
```

## 📦 Step 3: Install Required Tools

``` bash
sudo apt update
sudo apt install rsync docker.io -y
```

## 📁 Step 4: Select Backup

``` bash
BACKUP="/mnt/nas/homelab/backups/backup-YYYY-MM-DD"
```

## 🛑 Step 5: Stop Docker

``` bash
sudo systemctl stop docker
```

## 🔁 Step 6: Restore

### Restore system

``` bash
sudo rsync -aAX --numeric-ids $BACKUP/etc/ /etc/
```

### Restore users

``` bash
sudo rsync -aAX --numeric-ids $BACKUP/home/ /home/
sudo rsync -aAX --numeric-ids $BACKUP/root/ /root/
```

### Restore Docker

``` bash
sudo rsync -aAX --numeric-ids $BACKUP/var/lib/docker/ /var/lib/docker/
```

## 🔁 Step 7: Reboot

``` bash
sudo reboot
```

## ✅ Verification

-   hostname
-   ssh access (no warnings)
-   docker ps
