# Ubuntu Server 24.04 LTS Installation Guide

**Target Hardware:** Beelink Mini S12
**OS:** Ubuntu Server 24.04 LTS
**Storage:** 256GB SSD
**Network:** Gigabit Ethernet (enp1s0)

This guide walks through a clean Ubuntu Server installation on the Beelink Mini S12, from ISO download to post-installation hardening.

---

## 1. Download Ubuntu Server ISO

1. Visit [ubuntu.com/download/server](https://ubuntu.com/download/server)
2. Download **Ubuntu Server 24.04 LTS** (the latest LTS release)
3. Verify the SHA256 checksum (provided on the download page) to ensure file integrity

---

## 2. Create Bootable USB

### Option A: balenaEtcher (Recommended for all platforms)

1. Download [balenaEtcher](https://www.balena.io/etcher/)
2. Plug in a USB drive (8GB or larger)
3. Open balenaEtcher
4. Click **Flash from file** and select the Ubuntu ISO
5. Click **Select target** and choose your USB drive
6. Click **Flash** and wait for completion

### Option B: Rufus (Windows)

1. Download [Rufus](https://rufus.ie/)
2. Plug in a USB drive
3. Open Rufus
4. Select ISO Image mode and choose the Ubuntu ISO
5. Keep default settings and click **Start**
6. Confirm overwriting the USB drive

### Option C: Command line (Linux/Mac)

```bash
# Identify the USB device
lsblk
# or on Mac:
diskutil list

# Write the ISO (replace sdX with your device, e.g., sda, sdb)
sudo dd if=~/Downloads/ubuntu-24.04-live-server-amd64.iso of=/dev/sdX bs=1M status=progress
sudo sync
```

**Warning:** Make sure you target the correct device. Using the wrong device will overwrite that disk.

---

## 3. Boot from USB on Beelink Mini S12

1. Plug the USB drive into the Beelink
2. Power on the Beelink
3. Press **F7** (or **Del** if F7 doesn't work) to enter the boot menu
4. Select the USB drive (may be labeled "UEFI: [USB device name]")
5. Press Enter and the Ubuntu installer will start

---

## 4. Ubuntu Installation Steps

### Language Selection
- Select **English** and press Enter

### Keyboard Configuration
- Select your keyboard layout (default is fine for most users)
- Press Enter to continue

### Network Configuration
- Ubuntu will auto-detect the Gigabit Ethernet interface (likely `enp1s0`)
- Select **DHCP** (automatic IP configuration) for now
- You will configure a static IP after the OS is installed
- Press Enter to continue

### Storage Configuration
- The installer will detect the 256GB SSD
- Select **Use entire disk** (the default option)
- Select the detected SSD device
- **Do NOT enable LVM** unless you have specific reasons (advanced feature)
- **Do NOT enable ZFS** (not needed for a single-disk setup)
- Press Enter to confirm and format the disk

**Warning:** This step is destructive. Verify you are formatting the correct disk.

### Profile Setup
- **Hostname:** Enter `lab` (the full DNS name will be `lab.in.alybadawy.com`, registered on the UDR7 internal DNS)
- **Username:** Enter `aly`
- **Password:** Create a strong password (mix of uppercase, lowercase, numbers, special chars)
- Confirm the password
- Press Enter to continue

### SSH Server
- Select **Install OpenSSH server** (checkmark should be present)
- When prompted, **Generate SSH keypair** (recommended) or use password auth only
- Press Enter to continue

### Featured Server Snaps
- This step offers optional snap packages (VSCode, LXD, Docker, etc.)
- **Select NONE** — do not install any snaps
  - Docker will be installed via the official Docker repository (cleaner, more flexible)
  - Other tools can be installed later if needed
- Press Enter to skip

### Installation Complete
- Once the installer finishes, press Enter or wait for automatic reboot
- Remove the USB drive after the system reboots

---

## 5. First Login and Basic Updates

Log in with username `aly` and the password you created.

Update the package manager and install security updates:

```bash
sudo apt update && sudo apt upgrade -y
```

Install essential tools:

```bash
sudo apt install -y curl wget git htop vim ufw net-tools
```

**Package descriptions:**
- `curl`: Command-line HTTP client
- `wget`: File downloader
- `git`: Version control
- `htop`: Interactive process monitor
- `vim`: Text editor
- `ufw`: Firewall management
- `net-tools`: Network utilities (ifconfig, netstat, etc.)

---

## 6. Configure Static IP Address

Ubuntu 24.04 uses Netplan for network configuration. Edit the Netplan configuration file:

```bash
sudo vim /etc/netplan/00-installer-config.yaml
```

Replace the contents with your static IP configuration:

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    enp1s0:
      dhcp4: no
      addresses:
        - 172.20.20.5/24           # Homelab static IP on Servers VLAN
      routes:
        - to: default
          via: 172.20.20.1           # UDR7 gateway for Servers VLAN (VLAN 20)
      nameservers:
        addresses: [172.20.20.1]     # UDR7 internal DNS (handles all VLANs)
```

> **Note:** `enp1s0` is the typical Ethernet interface name on the Beelink. Confirm it with `ip link show` during installation — it may also appear as `eth0` or `eno1`.

Apply the configuration:

```bash
sudo netplan apply
```

Verify the static IP is assigned:

```bash
ip addr show enp1s0
```

---

## 7. Disable Snapd (Optional, for a Lean System)

If you want to remove Snapd entirely (saves disk space and boot time):

```bash
sudo systemctl disable snapd --now
sudo apt purge -y snapd
sudo rm -rf /snap /var/snap /var/lib/snapd
```

This is optional but recommended for a minimal homelab server.

---

## 8. Configure Swap File

Create an 8GB swap file to handle memory pressure (useful if running many containers):

```bash
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Add swap to `/etc/fstab` so it persists across reboots:

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Set kernel swap behavior to prefer RAM over disk (swappiness=10 = use swap only when necessary):

```bash
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Verify swap is active:

```bash
free -h
# Should show 8G in the Swap row
```

---

## 9. Enable UFW Firewall

Configure the Uncomplicated Firewall to block unauthorized access:

```bash
# Set default policies: deny all incoming, allow all outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (required, or you'll lock yourself out)
sudo ufw allow ssh

# Allow HTTP and HTTPS (needed for NPM reverse proxy)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow NPM Admin Dashboard
sudo ufw allow 81/tcp comment 'NPM Admin'

# Allow Netdata monitoring
sudo ufw allow 19999/tcp comment 'Netdata'

# Enable the firewall (this will prompt for confirmation)
sudo ufw enable
```

Verify firewall rules:

```bash
sudo ufw status
```

You should see all rules listed with `ALLOW` status.

---

## 10. Enable Automatic Security Updates

Install and configure unattended-upgrades to apply security patches automatically:

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

This will prompt for a mail notification setting. You can skip it for a home server (select "none").

---

## 11. NFS Client Setup (for NAS Mounts)

Install the NFS client package — this is all that's needed at this stage:

```bash
sudo apt install -y nfs-common
```

Create the mount point directories so they exist before fstab is configured:

```bash
sudo mkdir -p /mnt/nas/cloud /mnt/nas/immich /mnt/nas/media
```

> **Full NAS mount configuration is covered in [Guide 02 — NAS Mounts](02-nas-mounts.md).** That guide covers NAS-side NFS export setup, resilient fstab options (automount, nofail, soft mounts), Docker service dependency ordering, ownership setup, and resilience testing. Complete guide 02 before starting guide 03 (Docker).

---

## 12. Verify the Installation

Run these checks to ensure everything is set up correctly:

```bash
# Display network configuration
ip addr show

# Verify static IP is assigned
ip route show

# Check disk space
df -h

# Check memory and swap
free -h

# Verify SSH is running
systemctl status ssh

# Verify firewall is active
sudo ufw status

# Verify swap is active
swapon --show
```

Expected output:
- Static IP should be assigned to enp1s0
- Disk should show the 256GB SSD
- Memory should match your system (e.g., ~12GB on Beelink Mini S12)
- Swap should show 8G
- SSH should be running (active)
- UFW should show enabled with rules listed
- NFS mounts (if configured) should show in `df -h`

---

## Next Steps

The OS is now ready for Docker installation. Proceed to **03-docker-setup.md** to install and configure Docker CE.

---

## Troubleshooting

**Problem: Network interface is not `enp1s0`**
```bash
ip link show
```
Note the actual interface name and update Netplan configuration accordingly.

**Problem: Static IP not applying after `netplan apply`**
```bash
# Restart networking service
sudo systemctl restart systemd-networkd

# Check for Netplan errors
sudo netplan validate
```

**Problem: NFS mount fails**

See [Guide 02 — NAS Mounts](02-nas-mounts.md) for full NFS troubleshooting. Quick connectivity check:
```bash
ping 172.20.20.10
showmount -e 172.20.20.10
```

**Problem: Firewall blocking SSH**
```bash
# If locked out, reboot and press Escape at boot to enter GRUB recovery
# Or disable UFW temporarily (requires physical access)
sudo ufw disable
```

**Problem: Swap not persisting after reboot**
Verify the `/etc/fstab` line is correct:
```bash
cat /etc/fstab | grep swapfile
# Should return: /swapfile none swap sw 0 0
```
