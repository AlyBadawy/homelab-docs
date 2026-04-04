# Homelab Restore Guide (v3)

## ⚙️ Prerequisites

```bash
sudo apt update
sudo apt install -y rsync docker.io
```

---

## 📡 Mount NAS

```bash
sudo mount -t nfs <nas-ip>:/path /mnt/nas
```

---

## 📁 Select Backup

```bash
BACKUP="/mnt/nas/homelab/backups/backup-YYYY-MM-DD"
```

---

## 🛑 Stop Docker

```bash
sudo systemctl stop docker
```

---

## 🔁 Restore

### System

```bash
sudo rsync -aAX --numeric-ids $BACKUP/etc/ /etc/
```

### Users

```bash
sudo rsync -aAX --numeric-ids $BACKUP/home/ /home/
sudo rsync -aAX --numeric-ids $BACKUP/root/ /root/
```

### Cron

```bash
sudo rsync -aAX --numeric-ids $BACKUP/cron/ /var/spool/cron/
sudo systemctl restart cron
```

### Docker

```bash
sudo rsync -aAX --numeric-ids $BACKUP/docker/ /var/lib/docker/
```

---

## 🔁 Reboot

```bash
sudo reboot
```

---

## ✅ Verify

- hostname
- ssh (no fingerprint warning)
- docker ps
- crontab -l

---

## 🧠 Notes

If cron fails:

```bash
sudo systemctl restart cron
```
