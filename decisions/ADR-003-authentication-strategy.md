# ADR-003: Authentication and Identity Management Strategy

**Status:** Accepted
**Date:** 2026-03-29
**Deciders:** Homelab Architecture Team
**Affected Components:** Nextcloud, Immich, Home Assistant, Portainer, NPM, Netdata

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

1. **User Management Burden:** Administrators must create and maintain user accounts across six separate services. Adding new users requires six separate operations.
2. **Password Fatigue:** Users must remember and manage different credentials for each service, increasing support burden and security risk.
3. **Inconsistent Access Control:** Different services implement access control differently, with no centralized audit trail or unified identity model.
4. **Service Scaling:** As the homelab grows to include additional services, the user management problem compounds.

### Hardware Constraints

The homelab infrastructure has a hard constraint of **8GB RAM** across all services. This severely limits choices for identity management:

- **Keycloak:** Requires 512MB+ RAM minimum, makes scaling difficult on an 8GB system
- **Authentik (standalone):** ~400-700MB when optimized, but lacks native LDAP interface for older services
- **OpenLDAP:** ~100-200MB, but complex to configure and lacks modern UI for user management
- **Custom solutions:** Not maintainable for long-term operations

### Service Authentication Capabilities

Services support varying authentication methods:

| Service | OIDC | OAuth2 | LDAP | SAML | Proxy Auth |
|---------|------|--------|------|------|-----------|
| Nextcloud | ✓ (via app) | - | ✓ | ✓ | - |
| Immich | ✓ | - | - | - | - |
| Home Assistant | - | ✓ | - | - | - |
| Portainer | - | ✓ | - | - | ✓ |
| NPM | - | - | - | - | ✓ |
| Netdata | - | - | - | - | ✓ |

---

## Decision

**Implement a two-tier authentication architecture using LLDAP as the user directory backend and Authentik as the centralized SSO Identity Provider.**

This decision establishes:
1. **LLDAP** as the lightweight directory service (user/group storage)
2. **Authentik** as the single point of authentication and authorization
3. **Service integrations** through OIDC, OAuth2, and LDAP outposts as appropriate

---

## Architecture Overview

### Component Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      Homelab Services                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │Nextcloud │ │  Immich  │ │  Home    │ │Portainer│       │
│  │(OIDC)    │ │ (OIDC)   │ │Assistant │ │(OAuth2) │       │
│  │          │ │          │ │(OAuth2)  │ │         │       │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬────┘       │
│       │            │            │            │            │
│  ┌────┴────────────┴────────────┴────────────┴────────┐   │
│  │                   Authentik (IdP)                   │   │
│  │  - OIDC Provider                                    │   │
│  │  - OAuth2 Provider                                  │   │
│  │  - Forward Auth Proxy (NPM, Netdata)               │   │
│  │  - Connects to LLDAP for user sync                 │   │
│  │  - PostgreSQL backend                              │   │
│  └────┬──────────────────────────────────────────────┘   │
│       │                                                    │
│  ┌────┴──────────────────────────────────────────────┐   │
│  │           LLDAP (User Directory)                   │   │
│  │  - User accounts and groups                        │   │
│  │  - LDAP protocol interface                         │   │
│  │  - Web UI for user management                      │   │
│  │  - SQLite backend                                  │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

#### LLDAP (Lightweight Directory Access Protocol Server)

**Purpose:** Centralized user and group directory

**Characteristics:**
- Lightweight implementation of LDAP protocol
- ~20-30MB RAM footprint
- SQLite-based storage
- Built-in web UI for user/group management
- Supports LDAP bind, search, and modify operations
- Simpler to understand and maintain than OpenLDAP

**Responsibilities:**
- Store user accounts (username, email, password hash, attributes)
- Manage group memberships
- Provide LDAP interface for directory queries
- Serve as single source of truth for identity data

#### Authentik (SSO Identity Provider)

**Purpose:** Centralized authentication and authorization service

**Characteristics:**
- Modern identity provider with OIDC/OAuth2 support
- PostgreSQL-backed (shared with other services if needed)
- Dual-container architecture (server + worker)
- ~400-700MB RAM when running both containers
- Stateless and horizontally scalable
- LDAP outpost capability for backward compatibility

**Responsibilities:**
- Authenticate users against LLDAP directory
- Expose OIDC endpoints for services that support it
- Expose OAuth2 endpoints for services that support it
- Run LDAP outpost for services requiring LDAP auth
- Provide forward auth proxy for cookie-based services (NPM, Netdata)
- Manage sessions and tokens
- Enforce authorization policies and access control
- Audit and log all authentication events

### Authentication Flows

#### Flow 1: OIDC-based Services (Nextcloud, Immich)

```
User Browser
    │
    ├─→ [Nextcloud/Immich] Requests Protected Resource
    │
    ├─→ [Service] Redirects to Authentik OIDC Provider
    │       (client_id, redirect_uri, scope, state)
    │
    ├─→ [Authentik] Shows Login Form (if not authenticated)
    │
    ├─→ [User] Enters Credentials
    │
    ├─→ [Authentik] Validates Against LLDAP
    │       LLDAP checks username/password
    │
    ├─→ [Authentik] Issues ID Token + Access Token
    │       (signed JWT tokens)
    │
    ├─→ [Service] Validates Token Signature, Extracts Claims
    │       (user ID, email, groups)
    │
    └─→ [Service] Creates Session, Grants Access
```

#### Flow 2: OAuth2-based Services (Home Assistant, Portainer)

```
Similar to OIDC, but:
- Service uses Access Token to call Authentik /userinfo endpoint
- Authentik returns user attributes from LLDAP
- Service creates local session based on returned attributes
```

#### Flow 3: Forward Auth Proxy (NPM, Netdata)

```
User Browser
    │
    ├─→ [Reverse Proxy] Receives Request to /admin (NPM) or /
    │       (Netdata) with no Authentik session cookie
    │
    ├─→ [Reverse Proxy] Checks Authentik /authorize endpoint
    │
    ├─→ [Authentik] Returns 401 Unauthorized
    │
    ├─→ [Reverse Proxy] Redirects to Authentik Login
    │
    ├─→ [User] Authenticates Against LLDAP (via Authentik)
    │
    ├─→ [Authentik] Issues Session Cookie
    │
    ├─→ [User] Redirected Back to Service
    │
    ├─→ [Reverse Proxy] Validates Session Cookie with Authentik
    │
    └─→ [Service] Receives X-Remote-User Headers, Grants Access
```

#### Flow 4: LDAP-based Services (future legacy app support)

```
Application
    │
    ├─→ [Authentik LDAP Outpost] Receives LDAP Bind Request
    │
    ├─→ [LDAP Outpost] Forwards to Authentik Backend
    │
    ├─→ [Authentik] Validates Against LLDAP User Store
    │
    └─→ [LDAP Outpost] Returns LDAP Response (Success/Failure)
```

---

## Service-Specific Integration Details

### Nextcloud

**Authentication Method:** OIDC via `user_oidc` app

**Configuration:**
```
Client ID: [from Authentik provider]
Client Secret: [from Authentik provider]
Discovery URL: https://authentik.homelab/application/o/nextcloud/.well-known/openid-configuration
Scope: openid profile email
User claim: preferred_username
Email claim: email
Groups claim: groups
```

**Benefits:**
- Native OIDC support via official app
- Automatic user provisioning on first login
- Group-based access control
- Low overhead on Nextcloud performance

### Immich

**Authentication Method:** OIDC (native support)

**Configuration:**
```
OIDC_ISSUER_URL: https://authentik.homelab
OIDC_CLIENT_ID: [from Authentik provider]
OIDC_CLIENT_SECRET: [from Authentik provider]
OIDC_SCOPE: openid profile email
OIDC_STORAGE_LABEL_CLAIM: name
```

**Benefits:**
- Built-in OIDC support
- No additional app installation required
- Simple configuration

### Home Assistant

**Authentication Method:** OAuth2 via generic_oauth2 integration

**Configuration:**
```yaml
homeassistant:
  auth_providers:
    - type: generic_oauth2
      client_id: [from Authentik provider]
      client_secret: [from Authentik provider]
      authorize_url: https://authentik.homelab/application/o/authorize/
      token_url: https://authentik.homelab/application/o/token/
      userinfo_url: https://authentik.homelab/application/o/userinfo/
      scope: openid profile email
      user_attr_name: email
```

**Benefits:**
- Native OAuth2 support in Home Assistant
- Flexible claims mapping
- Supports admin/user role assignment via groups

### Portainer

**Authentication Method:** OAuth2 via Authentik

**Configuration:**
```
OAuth2 Provider: Custom
Client ID: [from Authentik provider]
Client Secret: [from Authentik provider]
Authorization URL: https://authentik.homelab/application/o/authorize/
Access Token URL: https://authentik.homelab/application/o/token/
Resource URL (userinfo): https://authentik.homelab/application/o/userinfo/
Redirect URL: https://portainer.homelab/auth/oauth2/callback
```

**Benefits:**
- Centralized Docker access control
- Team-based access policies via LDAP groups
- Audit trail of all Docker operations

### NPM (Nginx Proxy Manager)

**Authentication Method:** Authentik Forward Auth Proxy

**Configuration:**
```
In NPM Admin Interface:
- Enable authentication for admin paths
- Proxy type: Forward Authentication
- Authentik URL: https://authentik.homelab/outpost.goauthproxy/
- Client ID: [from Authentik outpost config]
```

**How it works:**
- Reverse proxy intercepts requests to /admin and other protected paths
- Checks with Authentik forward auth endpoint
- If user not authenticated, redirects to Authentik login
- After successful login, sets X-authentik-* headers with user info
- NPM receives authenticated requests with user context

**Benefits:**
- Protects NPM admin panel without app modification
- Single session cookie for all proxied services
- Audit log of who accessed admin functions

### Netdata

**Authentication Method:** Authentik Forward Auth Proxy

**Configuration:**
```
Via Docker Compose environment:
NETDATA_CLAIM_TOKEN: [homelab claim token]
Protected by reverse proxy with same Authentik forward auth setup as NPM
```

**Alternative:** If Netdata is accessed directly (not via reverse proxy):
- Use read-only API token for metrics collection
- Restrict network access to metrics port (19999)
- Public dashboards require reverse proxy authentication

**Benefits:**
- Protects sensitive system metrics
- Prevents unauthorized system monitoring access
- Integrates seamlessly with NPM proxy setup

---

## Why This Architecture: Alternatives Considered

### Alternative 1: Authentik Alone (No LLDAP)

**Description:** Use Authentik's built-in user database, without separate LLDAP server

**Pros:**
- Single component to manage
- Simplifies deployment and updates
- All user management in one interface

**Cons:**
- Authentik's internal user store lacks advanced LDAP features
- Some services that only support LDAP authentication would require LDAP outpost anyway
- No separate LDAP interface means limited interoperability
- LLDAP is more memory-efficient (~30MB vs. full Authentik overhead)

**Decision:** Rejected. LLDAP provides a cleaner, more extensible user directory that is lighter weight and offers better separation of concerns.

### Alternative 2: Keycloak

**Description:** Use Keycloak as the identity provider with built-in LDAP support

**Pros:**
- Industry-standard identity platform
- Extensive feature set for enterprise deployments
- Strong community and documentation
- Native LDAP backend support

**Cons:**
- **Minimum 512MB RAM required**, often 1-2GB in practice
- Complex configuration for homelab use cases
- Overhead of Java Runtime Environment
- Steeper learning curve
- Overkill for small number of services
- On an 8GB system, Keycloak + LLDAP + all services leaves little headroom

**Decision:** Rejected. Hardware constraints make Keycloak impractical for this homelab scale.

### Alternative 3: Authelia

**Description:** Lightweight SSO server focused on simplicity

**Pros:**
- Minimal memory footprint (~100MB)
- Easy configuration
- Good forward auth proxy support
- Supports LDAP backend

**Cons:**
- Does not expose OIDC provider endpoints as well as Authentik
- Limited to forward auth and LDAP for applications
- Smaller ecosystem and community
- Fewer integrations available
- Would require workarounds for OIDC-based services

**Decision:** Rejected. While lightweight, Authelia cannot serve as an OIDC provider, which limits integration with modern services like Immich and Nextcloud's user_oidc app.

### Alternative 4: Individual Per-Service Authentication

**Description:** Keep current model where each service maintains its own user database

**Pros:**
- No new components to deploy or manage
- Services are independent
- No single point of failure for authentication
- Lower initial complexity

**Cons:**
- No SSO capability — users manage multiple credentials
- User provisioning/deprovisioning requires manual work per service
- No consistent audit trail across services
- Painful to add new users
- Impossible to enforce consistent password policies
- No way to bulk deactivate users

**Decision:** Rejected. This approach doesn't meet the goal of unified user management and single sign-on.

---

## Implementation Consequences

### Positive Consequences

**Pro: Single Unified Identity**
- Users remember one username and password
- New users provisioned once in LLDAP, available to all services automatically
- Offboarding involves a single user deactivation in LLDAP

**Pro: Memory-Efficient Design**
- LLDAP: ~20-30MB RAM (extremely lightweight)
- Authentik (optimized): ~400-700MB RAM combined (server + worker)
- Total authentication stack: ~430-730MB, acceptable on 8GB system
- Leaves ~7GB for application services

**Pro: Simple User Management**
- LLDAP web UI makes user management accessible without LDAP expertise
- Self-service password resets can be enabled in Authentik
- Group management in LLDAP translates to role-based access in services

**Pro: Extensibility**
- Adding new services only requires Authentik provider configuration
- No need to touch LLDAP or recreate users
- LDAP interface allows future services that require LDAP to work without changes

**Pro: Operational Simplicity**
- Clear separation of concerns: LLDAP stores data, Authentik handles auth flows
- Well-documented pattern in self-hosting community with proven reliability
- Standard protocols (LDAP, OIDC, OAuth2) reduce vendor lock-in

**Pro: Audit and Compliance**
- All authentication events logged in Authentik
- Centralized view of who accessed what and when
- Easier to implement password policies, session timeouts, MFA
- Single place to audit user lifecycle events

### Negative Consequences

**Con: Increased Setup Complexity**
- More components to configure than per-service auth
- Initial integration with each service requires OIDC/OAuth2 setup
- LLDAP + Authentik coordination adds learning curve

**Mitigation:** Create runbooks and documentation for each service integration. Invest time upfront in getting configuration right.

**Con: Authentik as Critical Dependency**
- If Authentik is down, users cannot authenticate to any service
- Even if individual services remain running, they're inaccessible without SSO
- Single point of failure for the entire homelab

**Mitigation:**
- Implement robust health monitoring on Authentik containers
- Set aggressive restart policies (always restart on failure)
- Back up Authentik PostgreSQL database regularly
- Document manual recovery procedure if database is corrupted
- Consider using a separate, dedicated machine for Authentik if possible

**Con: LLDAP Data Loss Risk**
- If LLDAP SQLite database is corrupted or lost, all users are lost
- No automatic recovery without backups

**Mitigation:**
- Include LLDAP data directory in automated backup strategy
- Regular backup testing (restore to separate container, verify data)
- Document recovery procedure
- Consider replicating LLDAP data to secondary storage

**Con: Debugging Complexity**
- Authentication issues require checking both LLDAP and Authentik logs
- Service-specific issues harder to diagnose when auth layer is in between
- Requires deeper understanding of OIDC/OAuth2/LDAP flows

**Mitigation:**
- Enable debug logging in both components during troubleshooting
- Create diagnostic playbooks for common issues
- Document authentication flows clearly
- Maintain detailed logs with centralized log aggregation (e.g., in Immich or Netdata)

---

## Backup and Recovery Strategy

### Critical Data to Back Up

1. **Authentik PostgreSQL Database**
   - Contains all authentication policies, providers, applications, sessions
   - Location: PostgreSQL data directory (typically `/var/lib/postgresql`)
   - Frequency: Daily
   - Retention: 30 days
   - Method: PostgreSQL `pg_dump` or filesystem backup if database not actively written

2. **LLDAP SQLite Database**
   - Contains all user accounts, groups, passwords
   - Location: `/lldap/data/lldap.db`
   - Frequency: Daily
   - Retention: 30 days
   - Method: Simple file copy (SQLite is single-file)

3. **Authentik Configuration**
   - `.env` file with secrets and configuration
   - Docker Compose file
   - Any custom Python scripts or outpost configurations
   - Location: `/opt/authentik/` or wherever deployed
   - Frequency: On every configuration change
   - Method: Version control or encrypted backup

### Recovery Procedures

**If Authentik PostgreSQL is lost:**
1. Stop Authentik containers
2. Restore PostgreSQL backup
3. Start PostgreSQL, verify it comes up cleanly
4. Verify Authentik can connect to database
5. Restart Authentik server and worker
6. Test OIDC provider endpoint responds correctly

**If LLDAP database is lost:**
1. Stop LLDAP container
2. Restore LLDAP data directory
3. Start LLDAP container
4. Verify LDAP bind works: `ldapsearch -x -H ldap://lldap:3890 -D cn=admin,dc=homelab -W -b dc=homelab`
5. Verify Authentik can still sync users
6. Test user login in one service

**If both are lost:**
1. Restore both from backups following procedures above
2. Verify LDAP connectivity first
3. Verify Authentik can connect to LDAP
4. Restart all services with LDAP/Authentik dependencies
5. Verify authentication works end-to-end

---

## Deployment Checklist

- [ ] Deploy LLDAP container with persistent storage
- [ ] Create initial admin user in LLDAP
- [ ] Verify LLDAP is accessible on port 3890 (LDAP) and 17170 (Web UI)
- [ ] Deploy Authentik server and worker containers with PostgreSQL
- [ ] Create Authentik superuser account
- [ ] Verify Authentik is accessible on configured port
- [ ] Create LDAP source in Authentik pointing to LLDAP
- [ ] Verify user sync from LLDAP to Authentik works
- [ ] For each service:
  - [ ] Create OAuth2/OIDC provider in Authentik
  - [ ] Generate client credentials
  - [ ] Configure service to use provider
  - [ ] Test authentication with non-admin user
  - [ ] Verify groups/roles sync correctly
- [ ] Set up automated backups of Authentik DB and LLDAP data
- [ ] Document recovery procedures
- [ ] Create monitoring/alerting for Authentik health
- [ ] Test failover: stop Authentik, verify services remain accessible (read-only)

---

## Related Decisions

- **ADR-001:** Infrastructure container runtime and orchestration
- **ADR-002:** Persistent storage strategy and backup approach
- **ADR-004:** (Future) Multi-factor authentication expansion

---

## Glossary

- **LDAP (Lightweight Directory Access Protocol):** Industry-standard protocol for accessing centralized directory services. Used for storing and querying user/group information.
- **LLDAP:** Simplified, lightweight LDAP server implementation optimized for homelab use.
- **OIDC (OpenID Connect):** Modern identity protocol built on top of OAuth2. Provides authentication (not just authorization) and standard claims about users.
- **OAuth2:** Authorization protocol allowing services to delegate user authentication to a trusted provider.
- **Authentik:** Open-source identity provider implementing OIDC, OAuth2, SAML, and LDAP protocols.
- **SSO (Single Sign-On):** Authentication model where users sign in once and gain access to multiple services.
- **IdP (Identity Provider):** Service that authenticates users and provides claims/attributes about them to other services.
- **Forward Auth Proxy:** Authentication pattern where a reverse proxy validates requests with an identity provider before forwarding to backend service.
- **JWT (JSON Web Token):** Digitally signed token containing user claims, used in OIDC/OAuth2 flows.

---

## Document History

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-29 | 1.0 | Homelab Architecture Team | Initial ADR creation and acceptance |

