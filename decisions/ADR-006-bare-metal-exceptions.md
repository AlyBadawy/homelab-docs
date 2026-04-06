# ADR-006: Bare-Metal Exceptions — Samba 4 AD DC and acme.sh

## Status
**Accepted**

## Date
2026-04-06

---

## Context

The core architecture principle is that all services run in Docker containers. This maximizes portability, isolation, and ease of management via Portainer. However, two services were identified as poor fits for containerization and are intentionally run bare-metal on the Ubuntu host:

1. **Samba 4 Active Directory Domain Controller**
2. **acme.sh (wildcard certificate management)**

---

## Decision

**Run Samba 4 AD DC and acme.sh directly on the Ubuntu host, outside of Docker.**

---

## Rationale

### Samba 4 Active Directory DC

**Kerberos requires a stable, OS-integrated hostname**

Kerberos — the authentication protocol at the core of Active Directory — is tightly coupled to the system hostname. It issues tickets tied to the FQDN of the DC (`dc.id.in.alybadawy.com`). Docker containers have ephemeral, dynamically assigned identities. Even with static container names, the host/container boundary creates subtle mismatches in Kerberos principal resolution, clock synchronization, and DNS that cause intermittent authentication failures.

**AD-integrated DNS requires direct OS binding**

Samba's DNS server (for the `id.in.alybadawy.com` zone) must bind to the host's network interface and respond on port 53. Docker's networking stack introduces NAT and port-mapping layers that break the direct IP binding AD clients expect. Running Samba bare-metal allows it to bind directly to `172.20.20.5:53` without translation.

**File permission and system clock integration**

Samba AD stores Kerberos keytabs and AD database files that require specific OS-level permissions and tight integration with `/etc/krb5.conf`, `/etc/hosts`, and system time (`chrony`/`ntpd`). Containerizing these interactions adds complexity without any benefit — the DC is not something that needs to be updated frequently or replicated across multiple hosts.

**Provisioning must happen before Docker**

The AD DC is provisioned first in the rebuild sequence because Docker services (Nextcloud, etc.) depend on it for authentication. The OS hostname must be set to the DC FQDN (`dc.id.in.alybadawy.com`) before Samba is provisioned — a requirement that predates Docker installation. Running it bare-metal is the natural fit for this first-in-sequence role.

---

### acme.sh (Certificate Management)

**Cron-based renewal requires host scheduling**

acme.sh renews certificates via a cron job on the host. The renewal process uses the Vercel DNS API to complete the DNS-01 challenge and writes the resulting files directly to `/opt/certs/`. A containerized acme.sh would require either:
- Mounting the host's crontab (not supported in Docker), or
- Using a custom entrypoint with an internal cron daemon (adds complexity and a second process inside a container)

Neither approach is meaningfully better than simply running acme.sh as a host-level cron job, which is the standard, documented deployment method.

**Direct file access to the certificate store**

acme.sh writes renewed certificates to `/opt/certs/`, which is a host directory mounted read-only into NPM. Running acme.sh on the host makes this a simple file write. Containerizing it would require a bind-mount back to the host, negating any isolation benefit and adding an extra volume mapping to reason about.

**acme.sh is a shell script, not a daemon**

acme.sh is not a long-running service — it's a shell script invoked periodically by cron. There is no meaningful benefit to containerizing a script that runs for a few seconds every 60 days.

---

## Alternatives Considered

### Samba 4 in Docker

Several community Docker images exist for Samba AD (e.g., `nowsci/samba-domain`). These were evaluated and rejected because:

- They require `--privileged` mode and host networking, which eliminate nearly all isolation benefits of containerization
- Kerberos clock sync issues are reported frequently in containerized Samba AD setups
- The DNS binding issues described above require `network_mode: host`, which further collapses the container boundary
- Community support and documentation quality for containerized Samba AD is significantly weaker than bare-metal Samba AD guides

### acme.sh in Docker

A containerized acme.sh (`neilpang/acme.sh`) is available and functional, but:

- It still requires a bind-mount to the host cert directory
- It requires scheduling via an internal cron daemon inside the container
- The operational complexity is higher than a native cron entry with no tangible benefit

---

## Consequences

### Positive

- **Samba AD is stable and well-integrated**: Direct OS binding eliminates networking and clock edge cases. The DC works exactly as documented in official Samba and Microsoft guides.
- **Certificate renewal is simple**: A host cron entry and a shell script. No container lifecycle to manage, no volume mounts to reason about.
- **Both services survive Docker restarts**: Because they are bare-metal, a full `docker system prune` or Docker daemon restart does not affect identity or certificate availability.

### Negative

- **Two services exist outside Portainer's view**: Samba AD and acme.sh are not visible or manageable in the Portainer UI. They must be managed directly via SSH.
- **Slight philosophical inconsistency**: The "everything in Docker" principle has two documented exceptions. This is explicitly acknowledged in the architecture overview and is not expected to expand.
- **OS hostname constraint**: The host must be set to `dc.id.in.alybadawy.com` (the DC FQDN), which means the homelab server has two effective names — the AD DC name and the service hostname (`lab.in.alybadawy.com`, via DNS alias).

---

## Related Decisions

- [ADR-001 — OS Selection](ADR-001-os-selection.md)
- [ADR-002 — SSL Certificates & Reverse Proxy](ADR-002-ssl-certificates.md)
- [ADR-003 — Authentication Strategy](ADR-003-authentication-strategy.md)
