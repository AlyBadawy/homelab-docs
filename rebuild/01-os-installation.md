# Ubuntu Server 24.04 LTS Installation Guide

**Target Hardware:** Beelink Mini S12
**OS:** Ubuntu Server 24.04 LTS
**Storage:** 256GB SSD
**Network:** 2.5 Gigabit Ethernet (enp1s0)

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

- **Hostname:** Enter `dc` (the full FQDN `dc.id.in.alybadawy.com` is set in Section 7 below — Samba AD requires this exact format)
- **Username:** Enter `alybadawy`
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

Log in with username `alybadawy` and the password you created.

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

## 6. Verify IP Address

The homelab's IP (`172.20.20.5`) is assigned by the UDR7's DHCP server as a **fixed/reserved lease** — the router always hands out the same IP to this machine based on its MAC address. No static IP configuration is needed on the OS side.

> **Before this step:** Ensure the DHCP reservation is already configured on the UDR7. In UniFi Network, go to **Network → [Servers VLAN] → DHCP → Fixed IPs** and create a reservation for the Beelink's MAC address with IP `172.20.20.5`.

After the first boot, verify the server received the correct IP:

```bash
ip addr show
```

Look for `172.20.20.5/24` on the Ethernet interface (typically `enp1s0` on the Beelink — confirm with `ip link show` if unsure).

Also verify the default route points to the UDR7 gateway:

```bash
ip route show
# Expected: default via 172.20.20.1
```

---

## 7. Set Hostname and Hosts File

> **Why this matters for Samba AD:** Samba 4 Active Directory requires the machine's OS hostname to exactly match the DC's FQDN (`dc.id.in.alybadawy.com`). This must be done before Samba is installed. Setting it here during OS setup avoids having to reconfigure anything later.

### Set the full FQDN as the system hostname

```bash
sudo hostnamectl set-hostname dc.id.in.alybadawy.com
```

Verify:

```bash
hostname          # → dc.id.in.alybadawy.com
hostname -f       # → dc.id.in.alybadawy.com
```

### Update /etc/hosts

Open the hosts file:

```bash
sudo nano /etc/hosts
```

It will look something like this by default:

```
127.0.0.1   localhost
127.0.1.1   dc
```

Remove the `127.0.1.1` line entirely. The file should contain only:

```
127.0.0.1       localhost
172.20.20.5     dc.id.in.alybadawy.com  dc
```

> **Important:** Samba AD provisioning (`samba-tool domain provision`) reads `/etc/hosts` and will fail or misbehave if the loopback line `127.0.1.1` is present. Remove it.

Save and close (`Ctrl+O`, `Enter`, `Ctrl+X`).

Verify the FQDN resolves correctly:

```bash
hostname -f
# Expected: dc.id.in.alybadawy.com
```

### Register DNS records on UDR7

For other devices on the network to resolve the homelab by name, add records in the UDR7's internal DNS under **Settings → Networks → [DNS / Local DNS Records]**:

| Record type | Name                       | Value           | Purpose                                          |
| ----------- | -------------------------- | --------------- | ------------------------------------------------ |
| A           | `lab.in.alybadawy.com`     | `172.20.20.5`   | Friendly alias for the homelab server            |
| A           | `dc.id.in.alybadawy.com`   | `172.20.20.5`   | DC hostname (used before Samba DNS is live)      |

> **Note:** After Samba AD is provisioned (next guide), a DNS forward zone for `id.in.alybadawy.com` is added to UDR7 so that Samba's own DNS answers AD queries. The `dc.id.in.alybadawy.com` A record above is only needed during the initial Samba setup before that forward zone exists.

---

## 8. Disable Snapd (Optional, for a Lean System)

If you want to remove Snapd entirely (saves disk space and boot time):

```bash
sudo systemctl disable snapd --now
sudo apt purge -y snapd
sudo rm -rf /snap /var/snap /var/lib/snapd
```

This is optional but recommended for a minimal homelab server.

---

## 9. Configure Swap File

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

swapon --show
# Should show: /swapfile  file  8G  0B   -2
```

### Verify disk partition layout and free space

After creating the swap file, confirm the overall disk usage looks as expected:

```bash
df -h
```

Expected output (approximate):

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2       240G  8.5G  230G   4% /
/dev/sda1       1.1G  6.1M  1.1G   1% /boot/efi
```

> The root partition (`/`) should show roughly 8–9 GB used (OS + packages + swap file) and ~230 GB available. If it looks significantly different, check that the installer used the full disk.

For a full picture of partition structure including types and UUIDs:

```bash
lsblk -f
```

---

## 10. Enable UFW Firewall

Configure the Uncomplicated Firewall to block unauthorized access. All Samba AD ports are included here since UFW is set up once at the OS level before any services are installed.

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

# --- Samba 4 Active Directory ports ---
# DNS (Samba internal DNS server for the AD zone)
sudo ufw allow 53 comment 'Samba AD DNS'

# Kerberos (ticket issuance and password change)
sudo ufw allow 88 comment 'Samba AD Kerberos'
sudo ufw allow 464 comment 'Samba AD Kerberos pw'

# LDAP and LDAPS (directory lookups from services and NAS)
sudo ufw allow 389/tcp comment 'Samba AD LDAP'
sudo ufw allow 636/tcp comment 'Samba AD LDAPS'

# SMB/NetBIOS (DC management, samba-tool, Windows admin)
sudo ufw allow 139/tcp comment 'Samba AD NetBIOS'
sudo ufw allow 445/tcp comment 'Samba AD SMB'

# RPC endpoint mapper + dynamic RPC range
sudo ufw allow 135/tcp comment 'Samba AD RPC'
sudo ufw allow 49152:65535/tcp comment 'Samba AD RPC dynamic'
# --- end Samba AD ports ---

# Enable the firewall (this will prompt for confirmation)
sudo ufw enable
```

Verify firewall rules:

```bash
sudo ufw status verbose
```

You should see all rules listed with `ALLOW` status.

---

## 11. Enable Automatic Security Updates

Install and configure unattended-upgrades to apply security patches automatically:

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

This will prompt for a mail notification setting. You can skip it for a home server (select "none").

---

## 12. NFS Client Setup (for NAS Mounts)

Install the NFS client package — this is all that's needed at this stage:

```bash
sudo apt install -y nfs-common
```

Create the mount point directories so they exist before fstab is configured:

```bash
sudo mkdir -p /mnt/nas/homelab
```

> **Full NAS mount configuration is covered in [Guide 03 — NAS Mounting](03-nas-mounting.md).** That guide covers NAS-side NFS export setup, resilient fstab options (automount, nofail, soft mounts), Docker service dependency ordering, ownership setup, and resilience testing. Complete guide 03 before starting guide 04 (Docker).

---

## 13. Verify the Installation

Run these checks to ensure everything is set up correctly:

```bash
# Verify IP address (should show 172.20.20.5 on enp1s0)
ip addr show

# Verify default route (should point to 172.20.20.1)
ip route show

# Verify hostname and FQDN
hostname && hostname -f
# → dc.id.in.alybadawy.com
# → dc.id.in.alybadawy.com

# Verify /etc/hosts has no 127.0.1.1 line and correct entry
cat /etc/hosts

# Check disk partitions and free space
df -h
lsblk -f

# Check memory and swap
free -h
swapon --show

# Verify SSH is running
systemctl status ssh

# Verify firewall is active with all rules
sudo ufw status verbose
```

Expected output:

- `ip addr` shows `172.20.20.5/24` on the Ethernet interface
- `hostname -f` returns `dc.id.in.alybadawy.com`
- `/etc/hosts` contains `172.20.20.5 dc.id.in.alybadawy.com dc` and **no** `127.0.1.1` line
- Root partition (`/`) shows ~230 GB available on the 256 GB SSD
- `free -h` shows 8 GB in the Swap row
- `swapon --show` shows `/swapfile  file  8G`
- SSH is `active (running)`
- UFW is enabled with SSH, HTTP/HTTPS, NPM, Netdata, and all Samba AD ports listed

---

## Next Steps

The OS is now ready for Samba 4 Active Directory provisioning. Proceed to **[02 — Active Directory](02-active-directory.md)** to install and provision the AD domain controller.

> **Do not install Docker yet.** Samba AD is a bare-metal service that must be provisioned, verified, and healthy before Docker is introduced. The AD setup guide covers Samba installation, domain provisioning, Kerberos configuration, and DNS verification. NAS mounts and Docker come after.

---

## Troubleshooting

**Problem: Network interface is not `enp1s0`**

```bash
ip link show
```

Note the actual interface name and update Netplan configuration accordingly.

**Problem: Server didn't receive `172.20.20.5` from DHCP**

```bash
# Check what IP was actually assigned
ip addr show

# Release and renew the DHCP lease
sudo dhclient -r && sudo dhclient
```

If the wrong IP is assigned, verify the DHCP reservation on the UDR7 — make sure the MAC address in the reservation matches the Beelink's actual MAC (`ip link show enp1s0` to find it).

**Problem: NFS mount fails**

See [Guide 03 — NAS Mounting](03-nas-mounting.md) for full NFS troubleshooting. Quick connectivity check:

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
