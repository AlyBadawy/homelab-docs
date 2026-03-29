# ADR-002: SSL Certificate Management for *.inside.alybadawy.com

**Date:** 2026-03-29
**Status:** Accepted
**Author:** Homelab Architecture Team
**Stakeholders:** Infrastructure, Security, DevOps

---

## Context

The homelab infrastructure requires TLS certificates to enable secure HTTPS communication for services accessed via `*.inside.alybadawy.com` subdomains. The current architecture presents several constraints and requirements that significantly impact certificate acquisition and renewal strategy:

### Current Infrastructure

- **Primary Domain:** `alybadawy.com`, registered and managed via Vercel DNS (user preference to maintain Vercel as DNS provider)
- **Public DNS Configuration:**
  - `inside.alybadawy.com` → A record pointing to UDR public IP address
  - `*.inside.alybadawy.com` → CNAME record pointing to `inside.alybadawy.com`
  - Public IP address is dynamically updated via a maintenance script
- **Internal DNS Override:** Local DNS resolver configured to respond with homelab server LAN IP for `*.inside.alybadawy.com` queries
- **Reverse Proxy:** Nginx Proxy Manager (NPM) deployed in Docker, responsible for routing and TLS termination
- **Network Isolation:** Homelab is **not publicly exposed** to the internet; firewall rules block inbound traffic on ports 80 and 443 from WAN

### Certificate Requirements

- **Certificate Type:** Wildcard certificate for `*.inside.alybadawy.com` to cover all current and future subdomains (e.g., `app1.inside.alybadawy.com`, `api.inside.alybadawy.com`, etc.)
- **Auto-renewal:** Certificates must renew automatically before expiry to prevent service disruptions
- **Operational:** Solution must require minimal manual intervention after initial setup
- **Security:** Sensitive credentials (DNS API tokens) must be stored securely on the homelab server

### ACME Challenge Constraints

The architecture eliminates two of the three standard ACME challenge types:

1. **HTTP-01 Challenge (Not Viable)**
   - Requires the ACME client to respond to HTTP requests on port 80 from the Certificate Authority
   - Homelab has no inbound port 80 access from the internet
   - Would require opening firewall rules, violating network security design

2. **TLS-ALPN-01 Challenge (Not Viable)**
   - Requires ACME client to respond to HTTPS requests on port 443 from the Certificate Authority
   - Also blocked by homelab firewall configuration

3. **DNS-01 Challenge (Required)**
   - Proves domain ownership by provisioning DNS TXT records
   - Works regardless of public network accessibility
   - **Mandatory for wildcard certificates** per ACME RFC 8555
   - Requires API access to DNS provider (Vercel in this case)

---

## Decision

**Use `acme.sh` running on the homelab host (via systemd timer) with the Vercel DNS plugin to issue and renew the wildcard certificate `*.inside.alybadawy.com`. Automatically import renewed certificates into Nginx Proxy Manager via a deploy hook.**

### Implementation Strategy

#### 1. Certificate Issuance and Renewal

- **Tool:** `acme.sh` (lightweight ACME client, no external dependencies)
- **DNS Plugin:** Community-maintained `dns_vercel` plugin integrated with acme.sh
- **Authentication:** Vercel API token stored in `/root/.acme.sh/account.conf` on the host
- **Scheduling:** systemd timer (alternative: cron job) checks for expiry and renews 30 days before expiration
- **Certificate Path:** Generated certificates stored in `/root/.acme.sh/inside.alybadawy.com/` directory

#### 2. Certificate Deployment to NPM

- **Deploy Hook:** acme.sh's post-renewal hook automatically executes
- **Action:** Copy certificate files to shared Docker volume at `/opt/stacks/proxy/npm/certs/`
- **Reload:** Restart NPM container to load new certificates
- **Integration:** NPM configured to use custom certificate from the shared volume path

#### 3. Automation Flow

```
[Systemd Timer Triggers]
         ↓
[acme.sh checks expiry]
         ↓
[Certificate expires within 30 days?]
         ├─ NO → Exit, no action
         └─ YES → Execute DNS-01 challenge
                    ↓
                [acme.sh uses Vercel API token]
                    ↓
                [Provision TXT record in Vercel DNS]
                    ↓
                [ACME server validates DNS]
                    ↓
                [New cert issued]
                    ↓
                [Deploy hook triggers]
                    ├─ Copy cert to /opt/stacks/proxy/npm/certs/
                    ├─ Update symlinks if needed
                    └─ Restart NPM container
```

---

## Why acme.sh Over Alternatives

### Comparison with Other Solutions

| Criteria | acme.sh | NPM Built-in LE | certbot | Manual | Cloudflare Switch |
|----------|---------|-----------------|---------|--------|-------------------|
| **Vercel DNS Support** | ✅ Community plugin | ❌ No | ✅ Community plugin | ✅ Possible | N/A (different DNS) |
| **Lightweight** | ✅ ~200KB | ✅ Built-in | ❌ ~15MB | N/A | N/A |
| **Auto-renewal** | ✅ Native cron | ✅ Built-in | ✅ systemd timer | ❌ Manual | N/A |
| **User Preference** | ✅ Vercel DNS | ✅ Vercel DNS | ✅ Vercel DNS | ✅ Vercel DNS | ❌ Requires change |
| **Operational Burden** | Low | Low | Medium | Very High | High (DNS migration) |
| **Docker Dependency** | ❌ Host-level | ✅ Container | ❌ Host-level | N/A | N/A |
| **Post-renewal Hook** | ✅ Flexible | Partial | ✅ Flexible | Manual | N/A |

### Rationale for acme.sh Selection

1. **Vercel DNS Plugin Availability**
   - NPM's built-in Let's Encrypt integration lacks DNS provider plugins, including Vercel
   - acme.sh's ecosystem includes a community-maintained `dns_vercel` plugin with documented Vercel API integration
   - Avoids vendor lock-in or switching to a different DNS provider

2. **Lightweight and Portable**
   - acme.sh is a standalone bash script, minimal dependencies
   - Easy to inspect, audit, and debug
   - Runs efficiently on modest hardware (typical homelab servers)
   - No additional runtime requirements beyond bash and curl

3. **Flexible Deploy Hooks**
   - acme.sh supports custom post-renewal hooks via `--renew-hook` parameter
   - Enables automatic certificate deployment to NPM without manual steps
   - Allows for future extensibility (e.g., additional services beyond NPM)

4. **Native Auto-renewal**
   - acme.sh integrates directly with system cron or systemd timers
   - No need for external orchestration tools
   - Proven track record in production homelab and small infrastructure deployments

5. **Cost and Operational Efficiency**
   - Minimal computational overhead
   - Single point of credential management (Vercel API token)
   - Transparent, auditable process for security review

---

## Alternatives Considered and Rejected

### 1. NPM Built-in Let's Encrypt with DNS Challenge

**Why Rejected:**
- NPM's integrated Let's Encrypt client only supports a limited set of DNS providers (Cloudflare, AWS Route53, etc.)
- Vercel DNS is not supported in the current or recent NPM versions
- Would require switching DNS provider or using an unsupported workaround
- Violates user's explicit preference to keep domain on Vercel

**Trade-offs:**
- Would reduce host-level dependencies
- All certificate management consolidated in NPM UI
- Less flexibility for advanced deployment scenarios

### 2. Manual Certificate Renewal

**Why Rejected:**
- Certificates expire every 90 days (ACME standard)
- Requires operator to manually request new certs, configure DNS, wait for validation, and update NPM
- High operational risk: expired certs cause service outages and security warnings
- Not scalable; impractical for production or semi-production homelab environments
- Violates infrastructure reliability principles

**Trade-offs:**
- Maximum simplicity and control in the moment
- No automation failures possible (because there is no automation)
- Completely impractical for long-term infrastructure stability

### 3. certbot with dns_vercel Plugin

**Why Rejected:**
- certbot is feature-rich but significantly larger (distributed as Python package, ~15MB+ with dependencies)
- Requires Python runtime to be installed on the host
- More complex configuration files and logging infrastructure
- acme.sh achieves the same result with lower resource footprint
- Higher attack surface with more dependencies

**Trade-offs:**
- More widely documented in enterprise environments
- Potentially more mature plugin ecosystem (though dns_vercel is comparable in both tools)
- Official support from EFF (though acme.sh is also well-maintained)

### 4. Switching DNS Provider to Cloudflare

**Why Rejected:**
- User has explicitly stated preference to keep domain on Vercel
- NPM's built-in Cloudflare DNS provider would work, but requires abandoning Vercel DNS
- Introduces unnecessary migration complexity and operational risk
- Violates stated architectural constraints

**Trade-offs:**
- Would enable use of NPM's native DNS plugin
- Potentially better support community
- Not acceptable given user's explicit requirements

### 5. HTTP-01 Challenge with Port Forwarding

**Why Rejected:**
- Requires opening inbound firewall rules for ports 80 and 443 from WAN
- Exposes homelab infrastructure to internet-facing attack surface
- Contradicts security-by-design principle of network isolation
- Public routing via UDR public IP was intended for outbound connectivity, not to expose services
- Places homelab at risk of unwanted enumeration and exploitation attempts

**Trade-offs:**
- Simplest to implement (no DNS plugin required)
- Would work with any ACME client and any certificate tool
- Completely unacceptable from security perspective

---

## Consequences

### Positive Consequences

1. **Fully Automated Renewal Process**
   - No manual intervention required every 90 days
   - Systemd timer or cron runs checks automatically
   - Deploy hook handles certificate deployment without operator action
   - Reduces risk of expired certificate outages

2. **Single Wildcard Certificate**
   - One cert covers `*.inside.alybadawy.com` and all current/future subdomains
   - Simplifies certificate management (only one cert to track)
   - Reduces renewal complexity as infrastructure scales

3. **Vercel DNS Integration**
   - Maintains user's preference to keep domain on Vercel
   - Leverages existing Vercel account and DNS infrastructure
   - Vercel API token stored only on homelab server (not exposed)
   - No need for DNS provider migration or multi-cloud complexity

4. **Low Operational Overhead**
   - acme.sh is lightweight and self-contained
   - Minimal dependencies: bash, curl, standard utilities
   - Transparent, auditable process
   - Easy to debug if issues arise

5. **Flexible Scaling**
   - Deploy hook enables easy addition of future certificate consumers
   - Not locked into NPM; could deploy certs to other services as needed
   - acme.sh plugin ecosystem allows future DNS provider additions if needed

### Negative Consequences

1. **Host-Level Dependency**
   - acme.sh runs on the host OS, not containerized
   - Adds a host-level process that must be maintained and monitored
   - Requires shell script execution at system level (systemd timer or cron)
   - If host OS changes, acme.sh must be reinstalled/reconfigured

2. **Initial Manual Configuration**
   - First-time cert issuance and NPM import requires manual steps
   - Must manually configure NPM to use custom certificate initially
   - Deploy hook setup requires understanding of acme.sh post-renewal mechanics
   - Cannot be fully automated until cert is already deployed to NPM

3. **Plugin Maintenance Risk**
   - `dns_vercel` plugin is community-maintained, not official
   - If Vercel's API changes significantly, plugin may break
   - Community maintenance means no guaranteed SLA for fixes
   - May require manual intervention to update plugin if incompatibilities arise

4. **State Management**
   - acme.sh maintains certificate state in `/root/.acme.sh/` directory
   - Must ensure proper backups of this directory to prevent certificate loss
   - If host filesystem is corrupted, may need to recover from backups
   - Requires documentation of state management and disaster recovery procedures

5. **API Credential Exposure**
   - Vercel API token must be stored on host filesystem
   - If host is compromised, attacker gains access to Vercel DNS control
   - Requires strong host security practices (firewall, access controls, monitoring)
   - Token rotation procedures must be documented

### Operational Implications

- **Monitoring:** Log certificate issuance/renewal events; alert on failures
- **Backup:** Include `/root/.acme.sh/` in regular backup procedures
- **Documentation:** Maintain runbooks for plugin updates, token rotation, emergency cert renewal
- **Testing:** Periodically test the full renewal flow to ensure deploy hook functions

---

## Implementation Notes

### Installation and Configuration

#### 1. Install acme.sh on the Host

```bash
# Download and install acme.sh
curl https://get.acme.sh | sh

# Reload shell profile
source ~/.bashrc  # or ~/.zshrc

# Verify installation
acme.sh --version
```

#### 2. Configure Vercel API Credentials

```bash
# Export Vercel API token for acme.sh to use
# This should be done securely (e.g., reading from a secrets file, not hardcoding)
export VERCEL_API_TOKEN="your_vercel_api_token_here"

# Store in acme.sh config for persistence
acme.sh --set-notify "$(pwd)/deploy_hook.sh"
```

**Security Note:** Store the API token in a restricted file (mode 0600) and source it when needed, or integrate with host secret management system (e.g., HashiCorp Vault, local encrypted file).

#### 3. Issue Initial Certificate

```bash
# Issue certificate for *.inside.alybadawy.com using Vercel DNS plugin
acme.sh --issue \
  --dns dns_vercel \
  -d inside.alybadawy.com \
  -d "*.inside.alybadawy.com" \
  --home /root/.acme.sh \
  --dnssleep 10
```

**Flags Explanation:**
- `--dns dns_vercel`: Use Vercel DNS provider plugin
- `-d`: Domain names (primary + wildcard)
- `--home`: State directory for acme.sh (default `/root/.acme.sh`)
- `--dnssleep`: Wait 10 seconds after DNS provisioning for propagation

#### 4. Create Deploy Hook Script

**File:** `/root/.acme.sh/deploy_hook.sh`

```bash
#!/bin/bash

# Deploy hook called by acme.sh after successful renewal

set -e

# Source environment to get NPM paths
export CERT_NAME="inside.alybadawy.com"
export CERT_SOURCE="${HOME}/.acme.sh/${CERT_NAME}"
export CERT_DEST="/opt/stacks/proxy/npm/certs"
export NPM_CONTAINER="nginx-proxy-manager"

# Verify source certificates exist
if [ ! -f "${CERT_SOURCE}/fullchain.cer" ] || [ ! -f "${CERT_SOURCE}/${CERT_NAME}.key" ]; then
  echo "[ERROR] Certificate files not found in ${CERT_SOURCE}"
  exit 1
fi

# Ensure destination directory exists
mkdir -p "${CERT_DEST}"

# Copy certificate files with timestamp for backup
TIMESTAMP=$(date +%s)
cp "${CERT_SOURCE}/fullchain.cer" "${CERT_DEST}/inside.alybadawy.com.fullchain.cer.${TIMESTAMP}"
cp "${CERT_SOURCE}/${CERT_NAME}.key" "${CERT_DEST}/inside.alybadawy.com.key.${TIMESTAMP}"

# Update active symlinks (NPM reads from these)
ln -sfn "${CERT_DEST}/inside.alybadawy.com.fullchain.cer.${TIMESTAMP}" "${CERT_DEST}/inside.alybadawy.com.crt"
ln -sfn "${CERT_DEST}/inside.alybadawy.com.key.${TIMESTAMP}" "${CERT_DEST}/inside.alybadawy.com.key"

# Restart NPM container to load new certificates
docker restart "${NPM_CONTAINER}" || {
  echo "[ERROR] Failed to restart ${NPM_CONTAINER}"
  exit 1
}

echo "[SUCCESS] Certificate deployed and NPM restarted at $(date)"
```

**Permissions:**
```bash
chmod 700 /root/.acme.sh/deploy_hook.sh
```

#### 5. Register Deploy Hook with acme.sh

```bash
# Tell acme.sh to run the deploy hook after renewal
acme.sh --set-notify "chmod +x /root/.acme.sh/deploy_hook.sh && /root/.acme.sh/deploy_hook.sh"
```

Alternatively, use the `--renew-hook` flag during renewal setup.

#### 6. Configure Auto-renewal with systemd Timer

**File:** `/etc/systemd/system/acme-renew.service`

```ini
[Unit]
Description=acme.sh certificate renewal
After=network.target
Wants=acme-renew.timer

[Service]
Type=oneshot
ExecStart=/root/.acme.sh/acme.sh --cron --home /root/.acme.sh
StandardOutput=journal
StandardError=journal
SyslogIdentifier=acme-renew
```

**File:** `/etc/systemd/system/acme-renew.timer`

```ini
[Unit]
Description=acme.sh certificate renewal timer
Requires=acme-renew.service

[Timer]
# Run renewal check daily at 2 AM
OnCalendar=daily
OnBootSec=10min
Persistent=true

[Install]
WantedBy=timers.target
```

**Enable and start:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable acme-renew.timer
sudo systemctl start acme-renew.timer

# Verify
sudo systemctl status acme-renew.timer
sudo journalctl -u acme-renew -f
```

#### 7. Configure NPM to Use Custom Certificate

1. **Access NPM Web UI** (typically `https://npm-server:81`)
2. **Navigate to:** SSL Certificates → Add Certificate
3. **Select:** Custom Certificate
4. **Paths:**
   - Certificate file: `/certs/inside.alybadawy.com.crt`
   - Private key file: `/certs/inside.alybadawy.com.key`
5. **Save**

NPM Docker Compose must include volume mount:
```yaml
services:
  npm:
    volumes:
      - /opt/stacks/proxy/npm/certs:/certs:ro
```

### Certificate Rotation

The renewal process:
1. **Day 60 of 90:** acme.sh checks expiry via systemd timer
2. **Day 65+:** If cert expires within 30 days, renewal begins
3. **Renewal Process:**
   - acme.sh uses Vercel API token to authenticate
   - Provisions TXT record: `_acme-challenge.inside.alybadawy.com`
   - Let's Encrypt validates DNS ownership
   - New cert issued to `/root/.acme.sh/inside.alybadawy.com/`
4. **Deploy Hook:**
   - Copies new cert to `/opt/stacks/proxy/npm/certs/`
   - Updates symlinks
   - Restarts NPM container
5. **Service Impact:** Minimal; NPM restart is brief, existing connections may be dropped but reconnect with new cert

### Monitoring and Alerting

Recommended monitoring:
```bash
# Check cert expiry in NPM
openssl x509 -in /opt/stacks/proxy/npm/certs/inside.alybadawy.com.crt -noout -enddate

# Check systemd timer status
systemctl status acme-renew.timer

# Monitor renewal logs
journalctl -u acme-renew -n 50 --follow
```

### Troubleshooting

| Issue | Cause | Resolution |
|-------|-------|-----------|
| Renewal fails, DNS update times out | Vercel API latency or token invalid | Check VERCEL_API_TOKEN, increase --dnssleep to 15 or 20 |
| Deploy hook fails, cert not copied | Directory permissions or NPM container not running | Verify `/opt/stacks/proxy/npm/certs/` is writable, check `docker ps` |
| NPM still shows old cert after renewal | NPM cache not cleared, symlink not updated | Manually restart NPM: `docker restart nginx-proxy-manager` |
| ACME challenge fails repeatedly | Vercel DNS API changed or dns_vercel plugin broken | Check plugin repo for updates, run `acme.sh --upgrade` |

### Disaster Recovery

**Scenario: Host filesystem corrupted, `/root/.acme.sh/` lost**

Recovery steps:
1. Restore `/root/.acme.sh/` from backup (should be included in regular backups)
2. Restart acme-renew timer: `systemctl restart acme-renew.timer`
3. Verify cert deployment to NPM: `ls -la /opt/stacks/proxy/npm/certs/`

**Scenario: Vercel API token compromised**

Mitigation:
1. Revoke old token in Vercel dashboard
2. Generate new token
3. Update token in `/root/.acme.sh/account.conf` (or however it's stored)
4. Test renewal with `acme.sh --renew -d inside.alybadawy.com`

---

## Future Considerations

### DNS Provider Migration

If Vercel is replaced in the future:
1. Switch `dns_vercel` plugin to appropriate provider plugin (e.g., `dns_cloudflare`)
2. Update Vercel API token to provider's token
3. Restart acme-renew timer
4. Existing cert files remain in place; next renewal uses new plugin

### Multi-Subdomain Scenarios

If additional domain names are needed:
```bash
# Add additional domains to existing cert
acme.sh --renew -d inside.alybadawy.com -d "*.inside.alybadawy.com" -d "*.example.com"
```

### Integration with Secrets Management

For enhanced security, integrate with HashiCorp Vault or similar:
- Store Vercel API token in Vault
- Have deploy hook fetch token from Vault before renewal
- Reduces risk of token exposure on filesystem

### Containerization of acme.sh

If future architecture moves to full containerization:
- Migrate acme.sh to dedicated renewal container
- Run as scheduled pod/task via Docker or Kubernetes
- Mount `/root/.acme.sh/` volume for state persistence
- Update deploy hook to communicate with Docker daemon

---

## References

- **acme.sh GitHub:** https://github.com/acmesh-official/acme.sh
- **acme.sh Vercel DNS Plugin:** https://github.com/acmesh-official/acme.sh/wiki/dnsapi#dns_vercel
- **ACME RFC 8555:** https://tools.ietf.org/html/rfc8555
- **Nginx Proxy Manager Docs:** https://nginxproxymanager.com/
- **systemd Timers:** https://www.freedesktop.org/software/systemd/man/systemd.timer.html
- **Vercel API Documentation:** https://vercel.com/docs/rest-api

---

## Sign-Off

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Decision Maker | - | 2026-03-29 | Pending Review |
| Infrastructure Lead | - | 2026-03-29 | Pending Review |
| Security Reviewer | - | 2026-03-29 | Pending Review |

---

**Document Version:** 1.0
**Last Updated:** 2026-03-29
**Next Review Date:** 2027-03-29
