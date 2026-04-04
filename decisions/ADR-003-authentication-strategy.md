# ADR-003: Authentication and Identity Management Strategy

**Status:** Accepted (Revised 2026-04-02)
**Date:** 2026-03-29 | **Revised:** 2026-03-31, 2026-04-01, 2026-04-02
**Deciders:** Homelab Architecture Team
**Affected Components:** Nextcloud, Immich, Home Assistant, Portainer, NPM, Netdata, Ugreen NAS

---

## Context

The homelab deployment consists of multiple services that require user authentication and authorization:
- **Nextcloud** (file storage and collaboration)
- **Immich** (photo management)
- **Home Assistant** (home automation)
- **Portainer** (Docker container management)
- **NPM** (Nginx Proxy Manager, reverse proxy and SSL termination)
- **Netdata** (system monitoring and observability)
- **UGreen NAS** (network attached storage)

### Hardware Constraints

The homelab infrastructure has a hard constraint of **8GB RAM** across all services. Identity management solutions add meaningful overhead:

- **Keycloak:** Requires 512MB+ RAM minimum; JVM overhead makes it impractical
- **Authentik:** ~400-700MB for both containers; adds significant complexity
- **Kanidm:** ~50-100MB; built-in OIDC/LDAP; but requires careful service account management
- **LLDAP:** ~20-30MB; simple web UI; but lacks POSIX compatibility for the NAS

### Why Centralized Identity Was Removed

After evaluating and partially implementing both LLDAP+Authentik and Kanidm, centralized identity management was found to add operational overhead disproportionate to the scale of this homelab:

1. **Single-user scale** — This homelab serves one primary user. The main benefit of centralized identity (provisioning/deprovisioning users across services in one place) provides minimal value at this scale.

2. **POSIX compatibility rabbit hole** — Getting a directory server to work correctly with the UGreen NAS required fighting POSIX attribute requirements (uidNumber, gidNumber, homeDirectory), NFS uid mapping, and ACL interactions. Each service has its own integration quirks.

3. **Failure blast radius** — A centralized identity service going down locks users out of all services simultaneously. Per-service accounts mean each service is independently accessible.

4. **Complexity budget** — Time spent debugging LDAP sync, objectSid errors, NFS uid mapping, and OIDC redirect loops is time not spent on actual homelab goals (running services, automating the home, storing photos).

---

## Decision

**Use each service's own built-in user management system.** No centralized identity provider.

| Service        | Authentication Method         |
|----------------|-------------------------------|
| Nextcloud      | Built-in local accounts       |
| Immich         | Built-in local accounts       |
| Home Assistant | Built-in local accounts       |
| UGreen NAS     | NAS built-in user management  |
| Portainer      | Built-in local admin account  |
| NPM            | Built-in local admin account  |
| Netdata        | Built-in local admin account  |

Passwords are stored in a password manager (1Password/Bitwarden). All services are LAN+VPN only — no public internet exposure — so the attack surface for per-service credentials is limited.

---

## Why This Architecture: Alternatives Considered

### Alternative 1: Kanidm (Single-Tier LDAP + OIDC)

**Description:** Kanidm as combined user directory and OAuth2/OIDC provider. Nextcloud and NAS authenticate via LDAPS; Immich and Home Assistant via OIDC.

**Pros:**
- Single source of truth for users; one provisioning/deprovisioning step
- POSIX-native (uidNumber, gidNumber auto-assigned) — compatible with NAS
- Low memory footprint (~50-100MB)
- Built-in OAuth2/OIDC — no middleware layer needed

**Cons:**
- Kanidm as single point of failure for all service logins
- NFS uid mapping between Kanidm UIDs and NAS filesystem ownership requires careful alignment
- LDAPS-only (no plain LDAP) — requires TLS cert management for LDAP clients
- Service account tokens expire and must be rotated
- Complexity cost high relative to a one-user homelab

**Decision:** Rejected (2026-04-02). The operational complexity exceeds the benefit at single-user scale.

### Alternative 2: LLDAP + Authentik (Two-Tier)

**Description:** LLDAP as the directory, Authentik as OIDC/OAuth2 middleware syncing from LLDAP.

**Pros:**
- Authentik has a rich policy engine and forward auth proxy for apps without native OIDC
- LLDAP is very lightweight (~20-30MB)

**Cons:**
- ~500-700MB combined RAM for Authentik server + worker + PostgreSQL + Redis
- LLDAP does not populate POSIX attributes (uidNumber, gidNumber) — incompatible with NAS
- LDAP sync introduces a second source of truth; sync failures leave users locked out
- objectSid errors from Authentik's sync base class caused persistent sync failures with LLDAP
- Two separate systems to configure, debug, and maintain

**Decision:** Rejected. POSIX incompatibility with the NAS and persistent sync errors made this approach impractical.

### Alternative 3: Authentik Alone

**Description:** Authentik's built-in user database without a separate directory server.

**Pros:**
- Single component for OIDC/OAuth2

**Cons:**
- No POSIX LDAP interface — NAS cannot authenticate against it
- ~500MB RAM just for the auth middleware

**Decision:** Rejected. NAS integration requires a proper directory server.

### Alternative 4: Keycloak

**Description:** Industry-standard identity platform.

**Pros:**
- Extensive feature set; battle-tested at scale

**Cons:**
- 512MB–2GB RAM due to JVM overhead
- Severe overkill for a single-user homelab

**Decision:** Rejected. Hardware constraints make Keycloak impractical.

---

## Implementation Consequences

### Positive Consequences

**Simplicity and resilience** — No identity service to maintain, back up, or debug. Each service is independently accessible regardless of the state of other services.

**No RAM overhead** — The ~50-500MB that would go to an identity stack is available for application services.

**Faster setup** — Deploying a service means deploying one container, not configuring LDAP binds, OIDC providers, redirect URIs, and sync schedules.

**Isolated failure** — A password change or account lockout in one service doesn't affect any other service.

### Negative Consequences

**No SSO** — Each service has separate credentials. Mitigated by a password manager (all passwords stored in one place; just not a single login flow).

**Manual provisioning across services** — Adding a new user requires creating accounts in each service. Acceptable at one-user scale.

**No unified audit trail** — Access logs are per-service only.

---

## Backup and Recovery

Each service backs up its own user database as part of its normal container volume backup. No separate identity backup needed.

---

## Related Decisions

- **ADR-001:** OS selection
- **ADR-002:** SSL certificates and reverse proxy strategy
- **ADR-004:** Storage strategy
- **ADR-005:** NAS folder layout

---

## Document History

| Date       | Version | Change                                                                                          |
|------------|---------|-------------------------------------------------------------------------------------------------|
| 2026-03-29 | 1.0     | Initial ADR — LLDAP + Authentik two-tier                                                        |
| 2026-03-31 | 1.1     | Replace LLDAP with Kanidm — POSIX incompatibility with UGreen NAS                              |
| 2026-04-01 | 2.0     | Remove Authentik — Kanidm built-in OAuth2/OIDC covers all app needs                            |
| 2026-04-02 | 3.0     | Remove all centralized identity — per-service built-in auth at single-user homelab scale        |
