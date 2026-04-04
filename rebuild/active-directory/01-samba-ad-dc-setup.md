# Samba 4 Active Directory DC — Build Runbook

## Environment

| Setting | Value |
|---|---|
| Realm | `ID.IN.ALYBADAWY.COM` |
| Domain | `id.in.alybadawy.com` |
| NetBIOS Name | `ID` |
| Hostname | `dc.id.in.alybadawy.com` |
| IP Address | `172.20.20.5` (fixed DHCP lease) |
| DNS Backend | Samba Internal |
| Role | Single DC |
| OS | Ubuntu Server 24.04 (minimized) |
| DNS Forwarder | `172.20.20.1` (UDR) |

---

## Step 1 — Prepare the System

### 1.1 Update and reboot

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

### 1.2 Install a text editor

```bash
sudo apt install -y nano vim
```

### 1.3 Set the hostname

```bash
sudo hostnamectl set-hostname dc.id.in.alybadawy.com
```

### 1.4 Edit `/etc/hosts`

```bash
sudo vim /etc/hosts
```

Remove any existing `127.0.1.1` lines. File should contain:

```
127.0.0.1   localhost
172.20.20.5 dc.id.in.alybadawy.com dc
```

Verify:

```bash
hostname -f
# Expected: dc.id.in.alybadawy.com
```

---

## Step 2 — Configure DNS Resolution for Installation

Disable `systemd-resolved` (it conflicts with Samba's DNS on port 53):

```bash
sudo systemctl disable --now systemd-resolved
sudo rm /etc/resolv.conf
```

Point temporarily at a public DNS to install packages:

```bash
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```

> We'll switch this to `127.0.0.1` after provisioning.

---

## Step 3 — Install Required Packages

```bash
sudo apt install -y \
  samba \
  winbind \
  libnss-winbind \
  libpam-winbind \
  krb5-user \
  krb5-config \
  dnsutils \
  net-tools \
  acl \
  attr
```

When prompted for Kerberos configuration:

- **Default Kerberos version 5 realm:** `ID.IN.ALYBADAWY.COM` *(all caps)*
- **Kerberos servers:** `dc.id.in.alybadawy.com`
- **Administrative server:** `dc.id.in.alybadawy.com`

---

## Step 4 — Stop and Clear Default Samba Services

```bash
sudo systemctl stop smbd nmbd winbind
sudo systemctl disable smbd nmbd winbind
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

---

## Step 5 — Provision the Domain

```bash
sudo samba-tool domain provision \
  --use-rfc2307 \
  --realm=ID.IN.ALYBADAWY.COM \
  --domain=ID \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL \
  --adminpass='YourStrongP@ssw0rd!'
```

> ⚠️ Password must meet complexity requirements: uppercase, lowercase, number, symbol, 8+ chars.

Expected output ending:

```
Server Role:           active directory domain controller
Hostname:              dc
NetBIOS Domain:        ID
DNS Domain:            id.in.alybadawy.com
DOMAIN SID:            S-1-5-21-...
```

---

## Step 6 — Configure Kerberos

```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

> Do not symlink — copy the file.

---

## Step 7 — Enable and Start Samba AD DC

```bash
sudo systemctl unmask samba-ad-dc
sudo systemctl enable samba-ad-dc
sudo systemctl start samba-ad-dc
sudo systemctl is-active samba-ad-dc
# Expected: active
```

> Note: `WERR_DNS_ERROR_RECORD_ALREADY_EXISTS` warnings in the logs on first start are harmless.

---

## Step 8 — Point DNS at Itself

```bash
sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolv.conf'
sudo chattr +i /etc/resolv.conf
```

The `chattr +i` makes the file immutable so nothing can overwrite it on reboot.

---

## Step 9 — Add DNS Forwarder

```bash
sudo vim /etc/samba/smb.conf
```

Add this line in the `[global]` section:

```
dns forwarder = 172.20.20.1
```

Restart Samba:

```bash
sudo systemctl restart samba-ad-dc
```

---

## Step 10 — Verify

### DNS — from the DC itself

```bash
host -t A dc.id.in.alybadawy.com 127.0.0.1
host -t SRV _kerberos._udp.id.in.alybadawy.com 127.0.0.1
host -t SRV _ldap._tcp.id.in.alybadawy.com 127.0.0.1
```

### DNS — from another machine (via UDR)

```bash
nslookup dc.id.in.alybadawy.com
nslookup -type=SRV _kerberos._udp.id.in.alybadawy.com
nslookup -type=SRV _ldap._tcp.id.in.alybadawy.com
nslookup -type=SRV _kerberos._tcp.id.in.alybadawy.com
nslookup -type=SRV _gc._tcp.id.in.alybadawy.com
```

### Internet resolution (via forwarder)

```bash
nslookup google.com
```

### Kerberos

```bash
kinit Administrator@ID.IN.ALYBADAWY.COM
klist
# Should show a valid TGT
```

### Domain level

```bash
sudo samba-tool domain level show
# Expected: Windows 2008 R2
```

### User list

```bash
sudo samba-tool user list
# Expected: Guest, krbtgt, Administrator
```

---

## Step 11 — (Optional) Firewall Rules

If `ufw` is active:

```bash
sudo ufw allow from any to any port 53
sudo ufw allow from any to any port 88
sudo ufw allow from any to any port 135
sudo ufw allow from any to any port 139
sudo ufw allow from any to any port 389
sudo ufw allow from any to any port 445
sudo ufw allow from any to any port 464
sudo ufw allow from any to any port 636
sudo ufw allow 49152:65535/tcp
sudo ufw reload
```

---

## UDR DNS Configuration

In the UniFi Dream Router, configure a DNS forward zone for `id.in.alybadawy.com` pointing to `172.20.20.5` so that all clients on the network can resolve AD DNS records through the UDR.

---

## Notes

- The `samba-ad-dc` service is the only service needed — `smbd`, `nmbd`, and `winbind` are not used in AD DC mode.
- The `/etc/resolv.conf` file is intentionally made immutable with `chattr +i`. To edit it in the future, run `sudo chattr -i /etc/resolv.conf` first.
- Kerberos password for Administrator expires every 42 days by default. To change the password policy: `sudo samba-tool domain passwordsettings set --max-pwd-age=0` (disables expiry).
