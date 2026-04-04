# SSL/TLS Certificates Guide: Wildcard Certs with acme.sh and Vercel DNS

This guide covers setting up TLS certificates for `in.alybadawy.com` and `*.in.alybadawy.com` using acme.sh with the Vercel DNS plugin. The DNS-01 challenge is used because the homelab is not publicly exposed.

**Environment:**

- Domain: `alybadawy.com` (DNS managed by Vercel)
- Target certificate: `in.alybadawy.com` and `*.in.alybadawy.com` (SAN cert)
- acme.sh runs on Ubuntu host (not containerized)
- Certificates deployed to: `/opt/certs/`
- Certificate manager: Nginx Proxy Manager (NPM)

---

## Step 1: Obtain a Vercel API Token

⚠️ **Warning:** This token grants full access to your Vercel account, including DNS management. Keep it secure and never commit it to version control.

1. Go to **vercel.com** → **Account Settings** → **Tokens**
2. Click **Create Token**
3. Name it `homelab-acme` (or similar)
4. Set scope to **Full Access**
5. Click **Create**
6. **Copy the token value immediately** — you'll only see it once

Store the token securely. You'll use it in Step 3.

---

## Step 2: Install acme.sh on the Ubuntu Host

Ubuntu Server 24.04 minimal does not include `cron` by default. acme.sh requires it to schedule automatic certificate renewals. Install it first:

```bash
sudo apt install -y cron
sudo systemctl enable cron
```

Then run the acme.sh installation script:

```bash
curl https://get.acme.sh | sh -s email=alybadawy@icloud.com
```

After installation completes, reload your shell:

```bash
source ~/.bashrc
```

Or close and reopen the terminal.

Verify installation:

```bash
~/.acme.sh/acme.sh --version
```

---

## Step 3: Configure the Vercel DNS Plugin

Export your Vercel API token so acme.sh can authenticate:

```bash
export VERCEL_TOKEN="your_token_here"
```

Replace `your_token_here` with the token from Step 1.

acme.sh will save this token to `~/.acme.sh/account.conf` automatically after the first certificate issuance, so you won't need to export it again.

---

## Step 4: Issue the Certificate

Request a SAN certificate covering both the root subdomain and the wildcard:

```bash
~/.acme.sh/acme.sh --issue \
  --dns dns_vercel \
  -d "in.alybadawy.com" \
  -d "*.in.alybadawy.com" \
  --server letsencrypt
```

> The first `-d` is the primary domain; the second is the wildcard SAN. Both are included in the same certificate, so a single cert covers `in.alybadawy.com` itself and every subdomain under it.

**What happens:**

1. acme.sh contacts Let's Encrypt to begin the DNS-01 challenge
2. acme.sh adds temporary TXT records to your Vercel DNS for both domains
3. Let's Encrypt validates ownership of both
4. The TXT records are automatically cleaned up
5. The certificate is saved to `~/.acme.sh/in.alybadawy.com_ecc/`

**Expected time:** 30–60 seconds (includes DNS propagation)

Check for success:

```bash
ls -la ~/.acme.sh/in.alybadawy.com_ecc/
```

You should see: `fullchain.cer`, `ca.cer`, `in.alybadawy.com.cer`, `in.alybadawy.com.key`

---

## Step 5: Create the Certificate Deployment Hook Script

The deployment hook is called by acme.sh whenever the certificate is renewed. It restarts all running Docker containers so every service picks up the new certificate automatically — no per-service copying or custom logic required.

First, create the `/opt/certs/` directory — this is the centralized certificate location for both NPM and Kanidm:

```bash
sudo mkdir -p /opt/certs
sudo chmod 755 /opt/certs
sudo chown $USER:$USER /opt/certs
```

The `chown` is required because acme.sh runs as your user (not root) and needs write access to deploy cert files into this directory.

Create the script at `/opt/certs/deploy-cert.sh`:

```bash
#!/bin/bash
# Called by acme.sh after cert installation or renewal.
# Restarts all running Docker containers so they pick up the renewed certificate.
# Using docker ps -q | xargs -r docker restart handles the case where some
# containers don't exist yet (xargs -r skips the command if input is empty).

# Restart all running containers
docker ps -q | xargs -r docker restart

# Log the deployment
echo "[$(date)] Certificate deployed successfully" >> /opt/certs/deploy.log
```

> **Note:** `$CERT_PATH` and similar variables are not available in `--reloadcmd`. They are only set in acme.sh deploy hooks. The cert files are already installed by acme.sh before this script runs, so no copying is needed here. Because all services (NPM, Kanidm, etc.) mount `/opt/certs/` directly, a single restart is all that's needed.

Make the script executable:

```bash
chmod +x /opt/certs/deploy-cert.sh
```

Verify the script:

```bash
ls -la /opt/certs/deploy-cert.sh
```

---

## Step 6: Install the Certificate with Deploy Hook

Install the certificate to its final location and register the deployment hook:

```bash
~/.acme.sh/acme.sh --install-cert \
  -d "in.alybadawy.com" \
  --cert-file /opt/certs/cert.pem \
  --key-file /opt/certs/key.pem \
  --ca-file /opt/certs/chain.pem \
  --fullchain-file /opt/certs/fullchain.pem \
  --reloadcmd "/opt/certs/deploy-cert.sh"
```

This command:

- Copies the certificate files from the acme.sh cache to `/opt/certs/`
- Registers the deploy hook to run on every renewal
- Creates symlinks in the acme.sh cache for management

Verify the files were created:

```bash
ls -la /opt/certs/
```

You should see: `cert.pem`, `key.pem`, `chain.pem`, `fullchain.pem`

---

## Step 7: Verify acme.sh Cron Job

acme.sh installs a cron job to automatically renew certificates 30 days before expiry. Verify it's active:

```bash
crontab -l | grep acme
```

Expected output:

```
0 0 * * * /home/homelab/.acme.sh/acme.sh --cron --home /home/homelab/.acme.sh > /dev/null 2>&1
```

This runs daily at midnight and checks if any certificates need renewal.

---

## Step 8: Manual Renewal Test

To test the renewal process before the certificate actually expires:

```bash
~/.acme.sh/acme.sh --renew -d "in.alybadawy.com" --force
```

The `--force` flag skips the "not yet due for renewal" check. This is useful for:

- Testing the renewal workflow
- Verifying the deployment hook runs
- Confirming NPM reloads correctly

Check the log:

```bash
tail -f /opt/certs/deploy.log
```

---

## Step 9: Verify Certificate Details

Check the certificate information:

```bash
~/.acme.sh/acme.sh --list
```

Shows all managed certificates and their expiry dates.

Inspect the certificate file:

```bash
openssl x509 -in /opt/certs/cert.pem -text -noout | grep -E "Subject:|DNS:|Not Before:|Not After"
```

Expected output:

```
Subject: CN = in.alybadawy.com
Not Before: Jan  1 12:00:00 2026 GMT
Not After : Apr  1 12:00:00 2026 GMT
                DNS:in.alybadawy.com
                DNS:*.in.alybadawy.com
```

Both SANs must be present — the cert covers the root subdomain and all services under it.

---

## Troubleshooting

### Certificate issuance fails with "DNS error"

**Problem:** acme.sh cannot create the TXT record in Vercel DNS.

**Solutions:**

1. Verify the Vercel API token is correct and has full access scope:
   ```bash
   echo $VERCEL_TOKEN
   ```
2. Ensure your domain is added to Vercel and DNS is delegated to Vercel nameservers
3. Check that `in.alybadawy.com` is not already in use as a Vercel project name
4. Manually check DNS propagation:
   ```bash
   dig _acme-challenge.in.alybadawy.com TXT
   ```

### Deployment hook doesn't run or NPM doesn't reload

**Problem:** Certificate is renewed but NPM doesn't pick up the new cert.

**Solutions:**

1. Verify the script is executable:
   ```bash
   ls -la /opt/certs/deploy-cert.sh
   ```
2. Check if NPM container is running:
   ```bash
   docker ps | grep npm
   ```
3. Check the deploy log:
   ```bash
   cat /opt/certs/deploy.log
   ```
4. Manually trigger renewal:
   ```bash
   ~/.acme.sh/acme.sh --renew -d "in.alybadawy.com" --force
   ```

### Vercel API token expires or is rotated

If your Vercel token is rotated or expires, update it in the acme.sh config:

```bash
# Edit the account config file
nano ~/.acme.sh/account.conf
```

Find the line starting with `VERCEL_TOKEN=` and update it with the new token:

```
VERCEL_TOKEN=your_new_token_here
```

Save the file and test renewal:

```bash
~/.acme.sh/acme.sh --renew -d "in.alybadawy.com" --force
```

---

## Reference: Certificate Lifecycle

1. **Issuance (Step 4):** Certificate is created and valid for 90 days
2. **Installation (Step 6):** Certificate files are deployed to `/opt/certs/`; deploy hook is registered with acme.sh
3. **Automatic Renewal:** acme.sh cron runs daily; if cert is within 30 days of expiry, it renews automatically
4. **Deployment:** Upon successful renewal, the deploy hook (Step 5) runs `docker ps -q | xargs -r docker restart` — all running containers pick up the new cert from `/opt/certs/` automatically
5. **Verification (Step 9):** Check cert expiry and renewal status anytime

---

## Next Steps

Once the certificate is issued and verified:

- Proceed to guide **06-core-infrastructure.md** to bootstrap Portainer and deploy core services
- NPM mounts `/opt/certs/` directly — no per-service cert copying is needed
- When the cert renews, the deploy hook restarts all running containers automatically
