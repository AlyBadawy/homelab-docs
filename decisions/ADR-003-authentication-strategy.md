# ADR-003: Authentication and Identity Management Strategy

**Status:** Accepted (Revised 2026-04-01)
**Date:** 2026-03-29 | **Revised:** 2026-03-31, 2026-04-01
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

Currently, each service maintains its own user database and authentication mechanism. This creates several operational challenges:

1. **User Management Burden:** Administrators must create and maintain user accounts across multiple separate services. Adding new users requires per-service operations.
2. **Password Fatigue:** Users must remember and manage different credentials for each service.
3. **Inconsistent Access Control:** No centralized audit trail or unified identity model.
4. **Service Scaling:** As the homelab grows, the user management problem compounds.

### Hardware Constraints

The homelab infrastructure has a hard constraint of **8GB RAM** across all services. This severely limits choices for identity management:

- **Keycloak:** Requires 512MB+ RAM minimum; JVM overhead makes it impractical on an 8GB system
- **Authentik:** ~400-700MB for both containers; adds complexity without benefit if OIDC can come from the directory itself
- **OpenLDAP:** ~100-200MB, but complex to configure, no modern UI, no built-in OAuth2/OIDC
- **Kanidm:** ~50-100MB; written in Rust; includes POSIX LDAP, web UI, and built-in OAuth2/OIDC provider

### Service Authentication Capabilities

| Service        | OIDC         | OAuth2 | LDAP         |
|----------------|--------------|--------|--------------|
| Nextcloud      | ✓ (via app)  | -      | ✓            |
| Immich         | ✓ (native)   | -      | -            |
| Home Assistant | -            | ✓      | -            |
| Portainer CE   | -            | -      | -            |
| NPM            | -            | -      | -            |
| Netdata        | -            | -      | -            |

---

## Decision

**Implement a single-tier authentication architecture using Kanidm as both the user directory and the OAuth2/OIDC identity provider.**

This decision establishes:
1. **Kanidm** as the sole identity service — user directory (POSIX LDAP) and OAuth2/OIDC provider combined
2. **Service integrations** via direct LDAPS (Nextcloud, NAS) or Kanidm's built-in OIDC (Immich, Home Assistant)
3. **NPM and Netdata** use local admin accounts — they have no native OIDC/OAuth2 support

> **Revision note (2026-03-31):** Originally written with LLDAP as the directory backend. LLDAP was replaced by Kanidm after discovering that LLDAP does not expose `posixAccount` with required attributes (`uidNumber`, `gidNumber`, `homeDirectory`) and does not support `posixGroup` at all — making it incompatible with the Ugreen NAS and any other POSIX LDAP client.

> **Revision note (2026-04-01):** Authentik removed from the architecture. Kanidm's built-in OAuth2/OIDC provider covers all app integration needs. Removing Authentik saves ~400-700MB RAM, eliminates two containers, a PostgreSQL database dependency, and a Redis dependency, and removes the LDAP sync complexity between Kanidm and Authentik.

---

## Architecture Overview

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Homelab Services                          │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │Nextcloud │  │  Immich  │  │  Home    │  │   Ugreen     │   │
│  │  (LDAP)  │  │  (OIDC)  │  │Assistant │  │    NAS       │   │
│  │          │  │          │  │ (OAuth2) │  │  (LDAPS)     │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │
│       │              │             │                │            │
│  ┌────┴──────────────┴─────────────┴────────────────┴────────┐  │
│  │                    Kanidm                                  │  │
│  │  - POSIX LDAP (LDAPS port 636)                            │  │
│  │  - OAuth2/OIDC Provider (port 8443)                       │  │
│  │  - Web UI for user/group management                       │  │
│  │  - Integrated database (no PostgreSQL/Redis needed)       │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  NPM: local admin    Netdata: local admin   Portainer: local    │
└─────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

#### Kanidm (Identity Management Server)

**Purpose:** Unified user directory and OAuth2/OIDC identity provider

**Characteristics:**
- Modern identity server written in Rust; ~50-100MB RAM footprint
- Integrated database (no external DB dependency)
- Built-in web UI for user/group management
- LDAPS-only (no plain LDAP) — correct security posture
- First-class POSIX support: `posixAccount`, `posixGroup`, auto-assigned `uidNumber`/`gidNumber`
- Built-in OAuth2/OIDC provider — no middleware layer needed
- Compatible with NAS, Linux PAM, sssd, and any POSIX LDAP client without schema workarounds
- Uses the homelab wildcard cert directly for both LDAPS and HTTPS

**Responsibilities:**
- Store user accounts (username, email, password hash, POSIX attributes)
- Manage group memberships
- Expose LDAPS interface for NAS and LDAP-based apps (Nextcloud)
- Serve as OAuth2/OIDC provider for Immich, Home Assistant, and Portainer BE
- Single source of truth for all identity data

### Authentication Flows

#### Flow 1: LDAP-based Services (Nextcloud, NAS)

```
Nextcloud / NAS
    │
    ├─→ [Kanidm LDAPS] Bind request with service account token (Bind DN: dn=token)
    │
    ├─→ [Kanidm] Validates token, returns bind success
    │
    ├─→ [Service] Queries users/groups (objectClass=posixaccount / objectClass=group)
    │
    └─→ [Kanidm] Returns posixAccount users with uidNumber, gidNumber, homeDirectory
```

#### Flow 2: OIDC-based Services (Immich, Home Assistant)

```
User Browser
    │
    ├─→ [Immich/HA] Requests protected resource; no session
    │
    ├─→ [Service] Redirects to Kanidm OIDC authorization endpoint
    │       (client_id, redirect_uri, scope, state)
    │
    ├─→ [Kanidm] Shows login form
    │
    ├─→ [User] Enters Kanidm credentials
    │
    ├─→ [Kanidm] Validates credentials, issues ID Token + Access Token (signed JWT)
    │
    ├─→ [Service] Validates token, extracts claims (sub, email, name)
    │
    └─→ [Service] Creates session, grants access
```

---

## Service-Specific Integration Details

### Nextcloud

**Authentication Method:** LDAP (via LDAP User and Group Backend app)

Using LDAP rather than OIDC preserves stable user UIDs across sessions, which is important for file ownership consistency in Nextcloud.

**Configuration:**
```
Server: ldaps://172.20.20.5
Port: 636
Bind DN: dn=token
Bind Password: <nextcloud_reader service account token>
Base DN: dc=in,dc=alybadawy,dc=com
User object filter: (objectClass=posixaccount)
Group object filter: (objectClass=group)
```

### Immich

**Authentication Method:** OIDC (native support)

**Configuration:**
```
Issuer URL: https://id.in.alybadawy.com/oauth2/openid/immich
Client ID: immich
Client Secret: <from kanidm system oauth2 show-basic-secret>
Scope: openid profile email
Auto-register users: enabled
```

### Home Assistant

**Authentication Method:** OIDC via generic_oauth auth provider

**Configuration (configuration.yaml):**
```yaml
homeassistant:
  auth_providers:
    - type: homeassistant
    - type: oidc
      client_id: homeassistant
      client_secret: <from kanidm system oauth2 show-basic-secret>
      discovery_url: https://id.in.alybadawy.com/oauth2/openid/homeassistant/.well-known/openid-configuration
```

### Portainer

**Authentication Method:** Local admin (Community Edition has no OAuth2 support)

If upgraded to Portainer BE, Kanidm OAuth2 can be configured using the authorization URL `https://id.in.alybadawy.com/ui/oauth2` and token URL `https://id.in.alybadawy.com/oauth2/token`.

### NPM and Netdata

**Authentication Method:** Local admin accounts only

Neither NPM nor Netdata support OIDC or OAuth2 natively. Both are accessed via local administrator credentials. Access is restricted to the Servers VLAN (172.20.20.0/24) via UFW rules.

---

## Why This Architecture: Alternatives Considered

### Alternative 1: Kanidm + Authentik (Two-Tier)

**Description:** Use Authentik as an OIDC/OAuth2 middleware layer sitting in front of Kanidm, syncing users from Kanidm via LDAP.

**Pros:**
- Authentik has a richer policy engine and flow customization
- Forward auth proxy for apps without native OIDC (NPM, Netdata)
- More detailed audit logging UI

**Cons:**
- Two containers (~400-700MB RAM), a PostgreSQL database, and a Redis instance just for auth middleware
- LDAP sync introduces a second source of truth — users must be in Kanidm AND synced to Authentik
- Sync lag means new users aren't immediately available to OIDC apps
- Authentik as SPOF: if Authentik is down, all OIDC-protected services become inaccessible even if Kanidm is healthy
- Kanidm already includes a perfectly capable OAuth2/OIDC provider — Authentik adds no unique value for this use case

**Decision:** Rejected. Kanidm's built-in OIDC provider meets all requirements without the operational overhead of a second identity middleware layer.

### Alternative 2: Authentik Alone (No Separate Directory)

**Description:** Use Authentik's built-in user database without a separate directory server.

**Pros:**
- Single component to manage

**Cons:**
- Authentik's internal user store does not expose a POSIX LDAP interface
- No `posixAccount`/`posixGroup` support — NAS and Linux systems cannot authenticate against it
- No path to NAS integration without a second LDAP server anyway

**Decision:** Rejected. POSIX LDAP compatibility for the NAS requires a proper directory server.

### Alternative 3: LLDAP as Directory Backend

**Description:** Use LLDAP as the directory, with Authentik syncing from it.

**Pros:**
- Very lightweight (~20-30MB RAM)
- Simple web UI

**Cons:**
- Declares `posixAccount` objectClass but does not populate required attributes (`uidNumber`, `gidNumber`, `homeDirectory`)
- Does not support `posixGroup` — groups cannot be found by POSIX LDAP clients
- Ugreen NAS cannot enumerate users or groups
- No fix possible without replacing LLDAP

**Decision:** Rejected. LLDAP was the original choice but was replaced after discovering its POSIX incompatibility during NAS integration (2026-03-31).

### Alternative 4: Keycloak

**Description:** Use Keycloak as the identity provider.

**Pros:**
- Industry-standard platform; extensive feature set

**Cons:**
- Minimum 512MB RAM, typically 1-2GB in practice due to JVM overhead
- Overkill for a small homelab
- On an 8GB system, Keycloak alone would consume 15-25% of available RAM

**Decision:** Rejected. Hardware constraints make Keycloak impractical.

### Alternative 5: Individual Per-Service Authentication

**Description:** Keep current model where each service maintains its own user database.

**Pros:**
- No new components; services are independent

**Cons:**
- No SSO; users manage multiple credentials
- Manual provisioning/deprovisioning per service
- No consistent audit trail; no unified password policies

**Decision:** Rejected. Does not meet the goal of unified user management.

---

## Implementation Consequences

### Positive Consequences

**Single Unified Identity**
- Users remember one username and password
- New users provisioned once in Kanidm, available to all services automatically
- Offboarding involves a single account deactivation

**Memory-Efficient Design**
- Kanidm: ~50-100MB RAM (including built-in OAuth2/OIDC)
- No Authentik server (~200-400MB) + worker (~200-300MB) + PostgreSQL (~50MB) + Redis (~20MB)
- Total authentication stack: ~50-100MB, leaving ~7.9GB for application services

**Operational Simplicity**
- Single container to manage, monitor, and back up for identity
- No LDAP sync to configure or debug between two systems
- Clear ownership: Kanidm is the single source of truth with no derived copies

**No SSO Middleware as Single Point of Failure**
- Kanidm is the only auth dependency; if it goes down, services using local auth (NPM, Netdata, Portainer) remain accessible
- Services using Kanidm OIDC (Immich, HA) gracefully degrade — existing sessions typically remain valid

**POSIX-Native**
- NAS, future Linux PAM integration, and any POSIX LDAP client work without schema workarounds
- uidNumber and gidNumber are auto-assigned and consistent across all clients

### Negative Consequences

**No Forward Auth Proxy for NPM/Netdata**
- Without Authentik, NPM and Netdata cannot be placed behind SSO
- Mitigation: restrict access via UFW to Servers VLAN only; use strong local passwords stored in password manager

**Kanidm as Single Point of Failure**
- All LDAP and OIDC authentication depends on Kanidm being available
- Mitigation: `restart: unless-stopped` policy; regular backups of the `kanidm_data` volume; monitor with Netdata

**Less Rich Policy Engine**
- Kanidm's auth flows are less customizable than Authentik's
- Mitigation: Acceptable for a homelab with a small number of trusted users; no enterprise policy requirements

---

## Backup and Recovery Strategy

### Critical Data to Back Up

1. **Kanidm Database**
   - Contains all user accounts, groups, passwords, POSIX attributes, OAuth2 clients
   - Location: Docker volume `kanidm_data` (internally `/data/kanidm.db`)
   - Frequency: Daily
   - Retention: 30 days
   - Method: `docker exec kanidm kanidmd backup /data/kanidm-backup.db` then copy file out

2. **Kanidm Config**
   - `/opt/stacks/kanidm/server.toml`
   - Frequency: On every configuration change
   - Method: Included in general `/opt/stacks/` backup

### Recovery Procedure

**If Kanidm database is lost:**
1. Stop Kanidm container
2. Restore `kanidm.db` from backup into the named volume
3. Start Kanidm, verify it comes up cleanly
4. Verify LDAPS bind works from the host: `openssl s_client -connect 172.20.20.5:636`
5. Verify OIDC discovery endpoint responds: `curl https://id.in.alybadawy.com/oauth2/openid/immich/.well-known/openid-configuration`
6. Test user login in one LDAP-connected service and one OIDC-connected service

---

## Deployment Checklist

- [ ] Deploy Kanidm container with persistent storage and `/opt/certs/` TLS mount
- [ ] Open UFW ports 636 and 8443 for Servers VLAN
- [ ] Configure NPM proxy for `id.in.alybadawy.com` with scheme `https`
- [ ] Recover `admin` and `idm_admin` accounts
- [ ] Create `homelab_users` and `homelab_admins` groups with POSIX enabled
- [ ] Create user accounts with POSIX enabled
- [ ] Create `nas_reader` service account token; configure NAS LDAP
- [ ] Create `nextcloud_reader` service account token; configure Nextcloud LDAP
- [ ] Create `immich` OAuth2 resource server; configure Immich OIDC
- [ ] Create `homeassistant` OAuth2 resource server; configure Home Assistant
- [ ] Verify LDAPS connectivity from NAS and Nextcloud
- [ ] Verify OIDC login works from Immich and Home Assistant
- [ ] Set up automated backup of `kanidm_data` volume
- [ ] Test backup restore procedure

---

## Related Decisions

- **ADR-001:** Infrastructure container runtime and orchestration
- **ADR-002:** Persistent storage strategy and backup approach
- **ADR-004:** (Future) Multi-factor authentication expansion

---

## Glossary

- **LDAP (Lightweight Directory Access Protocol):** Industry-standard protocol for accessing centralized directory services.
- **LDAPS:** LDAP over TLS. Kanidm supports LDAPS only — no plain LDAP.
- **OIDC (OpenID Connect):** Modern identity protocol built on OAuth2. Provides authentication and standard user claims.
- **OAuth2:** Authorization protocol allowing services to delegate user authentication to a trusted provider.
- **Kanidm:** Modern identity server implementing LDAPS, OAuth2/OIDC, and POSIX attributes natively. Written in Rust.
- **posixAccount / posixGroup:** LDAP objectClasses required by POSIX-compliant systems (NAS, Linux PAM, sssd).
- **SSO (Single Sign-On):** Authentication model where users sign in once and gain access to multiple services.
- **IdP (Identity Provider):** Service that authenticates users and provides claims to other services.
- **JWT (JSON Web Token):** Digitally signed token containing user claims, used in OIDC/OAuth2 flows.

---

## Document History

| Date       | Version | Author                     | Change                                                                                              |
|------------|---------|----------------------------|-----------------------------------------------------------------------------------------------------|
| 2026-03-29 | 1.0     | Homelab Architecture Team  | Initial ADR creation and acceptance (LLDAP + Authentik two-tier)                                   |
| 2026-03-31 | 1.1     | Homelab Architecture Team  | Replace LLDAP with Kanidm — POSIX incompatibility with Ugreen NAS                                  |
| 2026-04-01 | 2.0     | Homelab Architecture Team  | Remove Authentik — Kanidm built-in OAuth2/OIDC covers all app needs; saves ~500MB RAM              |
