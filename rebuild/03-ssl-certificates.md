# SSL/TLS Certificates Guide: Wildcard Certs with acme.sh and Vercel DNS

This guide covers setting up wildcard TLS certificates for `*.inside.alybadawy.com` using acme.sh with the Vercel DNS plugin. The DNS-01 challenge is used because the homelab is not publicly exposed.

**Environment:**
- Domain: `alybadawy.com` (DNS managed by Vercel)
- Target certificate: `*.inside.alybadawy.com`
- acme.sh runs on Ubuntu host (not containerized)
- Certificates deployed to: `/opt/stacks/proxy/npm/certs/`
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

Run the installation script:

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
export VERCEL_API_TOKEN="your_token_here"
```

Replace `your_token_here` with the token from Step 1.

acme.sh will save this token to `~/.acme.sh/account.conf` automatically after the first certificate issuance, so you won't need to export it again.

---

## Step 4: Issue the Wildcard Certificate

Request the wildcard certificate using the Vercel DNS plugin:

```bash
~/.acme.sh/acme.sh --issue \
  --dns dns_vercel \
  -d "*.inside.alybadawy.com" \
  --server letsencrypt
```

**What happens:**
1. acme.sh contacts Let's Encrypt to begin the DNS-01 challenge
2. acme.sh adds a temporary TXT record to your Vercel DNS (under `inside.alybadawy.com`)
3. acme.sh verifies the TXT record exists
4. Let's Encrypt validates ownership
5. The TXT record is automatically cleaned up
6. The certificate is saved to `~/.acme.sh/inside.alybadawy.com_ecc/`

**Expected time:** 30–60 seconds (includes DNS propagation)

Check for success:

```bash
ls -la ~/.acme.sh/inside.alybadawy.com_ecc/
```

You should see: `fullchain.cer`, `ca.cer`, `inside.alybadawy.com.cer`, `inside.alybadawy.com.key`

---

## Step 5: Create the Certificate Deployment Hook Script

The deployment hook is called by acme.sh whenever the certificate is renewed. It copies the certificate files to NPM's location and restarts NPM.

Create the script at `/opt/stacks/proxy/npm/certs/deploy-cert.sh`:

```bash
#!/bin/bash
# Certificate deployment hook for acme.sh
# Called after successful cert issuance or renewal
# Variables set by acme.sh: $CERT_PATH, $KEY_PATH, $CA_CERT_PATH, $FULLCHAIN_PATH

CERT_DIR="/opt/stacks/proxy/npm/certs"

# Copy certificate files
cp "$CERT_PATH" "$CERT_DIR/cert.pem"
cp "$KEY_PATH" "$CERT_DIR/key.pem"
cp "$CA_CERT_PATH" "$CERT_DIR/chain.pem"
cp "$FULLCHAIN_PATH" "$CERT_DIR/fullchain.pem"

# Set proper permissions
chmod 644 "$CERT_DIR"/*.pem

# Reload NPM to pick up the new certificate
docker restart npm 2>/dev/null || true

# Log the deployment
echo "[$(date)] Certificate deployed successfully" >> /var/log/acme-deploy.log
```

Make the script executable:

```bash
chmod +x /opt/stacks/proxy/npm/certs/deploy-cert.sh
```

Verify the script:

```bash
ls -la /opt/stacks/proxy/npm/certs/deploy-cert.sh
```

---

## Step 6: Install the Certificate with Deploy Hook

Install the certificate to its final location and register the deployment hook:

```bash
~/.acme.sh/acme.sh --install-cert \
  -d "*.inside.alybadawy.com" \
  --cert-file /opt/stacks/proxy/npm/certs/cert.pem \
  --key-file /opt/stacks/proxy/npm/certs/key.pem \
  --ca-file /opt/stacks/proxy/npm/certs/chain.pem \
  --fullchain-file /opt/stacks/proxy/npm/certs/fullchain.pem \
  --reloadcmd "/opt/stacks/proxy/npm/certs/deploy-cert.sh"
```

This command:
- Copies the certificate files from the acme.sh cache to `/opt/stacks/proxy/npm/certs/`
- Registers the deploy hook to run on every renewal
- Creates symlinks in the acme.sh cache for management

Verify the files were created:

```bash
ls -la /opt/stacks/proxy/npm/certs/
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
0 0 * * * /root/.acme.sh/acme.sh --cron --home /root/.acme.sh > /dev/null 2>&1
```

This runs daily at midnight and checks if any certificates need renewal.

---

## Step 8: Manual Renewal Test

To test the renewal process before the certificate actually expires:

```bash
~/.acme.sh/acme.sh --renew -d "*.inside.alybadawy.com" --force
```

The `--force` flag skips the "not yet due for renewal" check. This is useful for:
- Testing the renewal workflow
- Verifying the deployment hook runs
- Confirming NPM reloads correctly

Check the log:

```bash
tail -f /var/log/acme-deploy.log
```

---

## Step 9: Import Certificate into Nginx Proxy Manager

After NPM is running (see guide 04), import the certificate into NPM for use across all proxy hosts.

1. Open the NPM admin panel at `http://172.20.20.15:81`
2. Navigate to **SSL Certificates**
3. Click **Add SSL Certificate** → **Custom**
4. Fill in the form:
   - **Certificate:** Paste the contents of `/opt/stacks/proxy/npm/certs/fullchain.pem`
   - **Private Key:** Paste the contents of `/opt/stacks/proxy/npm/certs/key.pem`
   - **Name:** `wildcard-inside-alybadawy-com`
5. Click **Save**

To view file contents:

```bash
cat /opt/stacks/proxy/npm/certs/fullchain.pem
cat /opt/stacks/proxy/npm/certs/key.pem
```

Once imported, you can select this certificate for all proxy hosts.

---

## Step 10: Verify Certificate Details

Check the certificate information:

```bash
~/.acme.sh/acme.sh --list
```

Shows all managed certificates and their expiry dates.

Inspect the certificate file:

```bash
openssl x509 -in /opt/stacks/proxy/npm/certs/cert.pem -text -noout | grep -E "Subject:|Not Before:|Not After"
```

Expected output:

```
Subject: CN = *.inside.alybadawy.com
Not Before: Jan  1 12:00:00 2026 GMT
Not After : Apr  1 12:00:00 2026 GMT
```

---

## Troubleshooting

### Certificate issuance fails with "DNS error"

**Problem:** acme.sh cannot create the TXT record in Vercel DNS.

**Solutions:**
1. Verify the Vercel API token is correct and has full access scope:
   ```bash
   echo $VERCEL_API_TOKEN
   ```
2. Ensure your domain is added to Vercel and DNS is delegated to Vercel nameservers
3. Check that `inside.alybadawy.com` is not already in use as a Vercel project name
4. Manually check DNS propagation:
   ```bash
   dig _acme-challenge.inside.alybadawy.com TXT
   ```

### Deployment hook doesn't run or NPM doesn't reload

**Problem:** Certificate is renewed but NPM doesn't pick up the new cert.

**Solutions:**
1. Verify the script is executable:
   ```bash
   ls -la /opt/stacks/proxy/npm/certs/deploy-cert.sh
   ```
2. Check if NPM container is running:
   ```bash
   docker ps | grep npm
   ```
3. Check acme.sh renewal log:
   ```bash
   cat ~/.acme.sh/inside.alybadawy.com_ecc/inside.alybadawy.com.log
   ```
4. Manually trigger renewal:
   ```bash
   ~/.acme.sh/acme.sh --renew -d "*.inside.alybadawy.com" --force
   ```

### Vercel API token expires or is rotated

If your Vercel token is rotated or expires, update it in the acme.sh config:

```bash
# Edit the account config file
nano ~/.acme.sh/account.conf
```

Find the line starting with `VERCEL_API_TOKEN=` and update it with the new token:

```
VERCEL_API_TOKEN=your_new_token_here
```

Save the file and test renewal:

```bash
~/.acme.sh/acme.sh --renew -d "*.inside.alybadawy.com" --force
```

---

## Reference: Certificate Lifecycle

1. **Issuance (Step 4):** Certificate is created and valid for 90 days
2. **Installation (Step 6):** Certificate is copied to NPM's location; cron job is registered
3. **Automatic Renewal:** acme.sh cron runs daily; if cert is within 30 days of expiry, it renews automatically
4. **Deployment:** Upon successful renewal, the deploy hook (Step 5) copies new cert to NPM and restarts the container
5. **Verification (Step 10):** Check cert expiry and renewal status anytime

---

## Next Steps

Once the certificate is issued and verified:
- Proceed to guide **04-core-infrastructure.md** to deploy Nginx Proxy Manager
- NPM will use the wildcard certificate for all internal services
- Subsequent services will automatically use the same cert for HTTPS
