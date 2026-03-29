# Homelab Architecture Overview

## System Summary

- **Hardware**: Beelink Mini S12 (Intel N95, 8GB DDR4, 256GB SSD)
- **OS**: Ubuntu Server 24.04 LTS
- **Container Runtime**: Docker (all services containerized)
- **NAS**: UGreen NAS (upgrading to UniFi UNAS 4) for bulk storage
- **Domain**: alybadawy.com (DNS via Vercel, internal services via *.inside.alybadawy.com)
- **Access**: LAN + VPN only (no public internet exposure)

---

## Design Principles

### Core Architecture Rules

1. **Everything in Docker**
   - No bare-metal service installations on the host
   - Portainer for container orchestration and management
   - Facilitates easy updates, backups, and isolation

2. **Single Entry Point: Nginx Proxy Manager**
   - All web UI traffic flows through NPM
   - No direct service port exposure (except NPM itself and Netdata)
   - TLS termination at NPM layer
   - Centralized reverse proxy with web UI for easy management

3. **Unified TLS with Wildcard Certificates**
   - Single wildcard cert: `*.inside.alybadawy.com`
   - Provisioned via acme.sh using Vercel DNS API
   - Covers all subdomains with one certificate (immich.inside.alybadawy.com, nextcloud.inside.alybadawy.com, etc.)
   - Auto-renewal built into Docker stack

4. **Centralized Authentication**
   - Authentik as OIDC/OAuth2 provider
   - LLDAP as LDAP backend (simpler than FreeIPA)
   - Services connect to Authentik for forward auth (HTTP header) or OIDC flows
   - Single source of truth for user management

5. **NAS for Persistent Data**
   - Large files (photos, videos, documents) stored on NAS via NFS
   - Homelab SSD reserved for OS, containers, and databases
   - Reduces SSD wear, simplifies backups, enables future hardware upgrades

6. **No Internet Exposure**
   - Zero ports forwarded from internet to homelab
   - VPN endpoint on UDR7 (not homelab)
   - Access via LAN or authenticated VPN only

---

## System Layers

### Network & Traffic Flow Diagram

```
                         Internet
                            |
                          [UDR7]
                       (Firewall, VPN,
                        DNS, Gateway)
                         /        \
                        /          \
                  LAN Clients    VPN Clients
                       |               |
                       └───────┬───────┘
                               |
                     (Route via internal DNS)
                               |
                      [Nginx Proxy Manager]
                      (Port 443, TLS termination)
                               |
                 ┌─────────────┼─────────────┐
                 |             |             |
            [Authentik]  [Services]   [PostgreSQL]
            (Auth layer)  (Nextcloud, [Redis]
                          Immich,
                          Home Assistant)
                 |             |
                 └─────────────┼─────────────┘
                               |
                          [NAS via NFS]
                        (User data, media)
```

### Layer Descriptions

**Layer 1: Gateway & VPN**
- UDR7 handles internet firewall, DHCP, internal DNS, and VPN endpoint
- Internal DNS resolves `*.inside.alybadawy.com` to homelab server LAN IP
- VPN clients receive LAN IPs and see the same internal DNS

**Layer 2: Reverse Proxy**
- Nginx Proxy Manager listens on ports 80 (HTTP redirect) and 443 (HTTPS)
- Terminates TLS using the wildcard cert
- Routes requests to appropriate backend services based on hostname

**Layer 3: Authentication & Authorization**
- Authentik validates user identity via OIDC or forward auth
- LLDAP provides LDAP interface for Authentik and other integrations
- PostgreSQL backs both Authentik and LLDAP

**Layer 4: Application Services**
- Nextcloud, Immich, Home Assistant, and other apps run in isolated Docker containers
- Services may use local databases (PostgreSQL, Redis) for sessions and caching
- Large persistent data stored on NAS

**Layer 5: Data Persistence**
- PostgreSQL: global relational database (shared by identity and app services)
- Redis: cache and session store
- NAS (NFS mount): user files, media library, backups

---

## Typical Request Flow

### Example: User accessing Nextcloud

1. **DNS Resolution**
   - User browses to `nextcloud.inside.alybadawy.com`
   - UDR7's internal DNS resolves to homelab IP (e.g., 172.20.20.15)

2. **TLS Handshake**
   - Client connects to NPM on port 443
   - NPM presents wildcard cert for `*.inside.alybadawy.com`
   - TLS tunnel established

3. **Request Routing**
   - NPM examines Host header, looks up routing rule for `nextcloud`
   - NPM proxies request to Nextcloud container (e.g., http://nextcloud:80)

4. **Authentication Check**
   - Nextcloud is configured for forward auth via Authentik
   - Nextcloud forwards auth request to Authentik via special header
   - Authentik checks session/token; if valid, adds `Remote-User` header and allows request

5. **Data Operations**
   - If request accesses user files, Nextcloud reads from NAS (NFS mount at /mnt/nfs/nextcloud)
   - Session data cached in Redis
   - User metadata queried from PostgreSQL

6. **Response**
   - Nextcloud renders response
   - NPM sends back through encrypted TLS tunnel
   - Browser displays content

---

## Resource Usage Estimates

### Per-Service Memory Consumption (Peak)

| Component | Estimated RAM | Notes |
|-----------|---------------|-------|
| Ubuntu base system | 300 MB | Kernel + essential daemons |
| Docker daemon | 100 MB | Container runtime overhead |
| Portainer | 50 MB | Container management UI |
| Netdata | 150 MB | Metrics collection & dashboard |
| Nginx Proxy Manager | 50 MB | Reverse proxy & routing |
| LLDAP | 30 MB | LDAP server (lightweight) |
| Authentik (server) | 400 MB | Auth server + request handler |
| Authentik (worker) | 200 MB | Background job processor |
| PostgreSQL (global) | 150 MB | Runs with shared_buffers=256MB |
| Redis | 50 MB | Session & cache store |
| Nextcloud | 300 MB | PHP-FPM container |
| Immich (server) | 400 MB | API server |
| Immich (ML service) | 800 MB – 1.5 GB | ML model inference (heaviest) |
| Home Assistant | 400 MB | Automation engine |
| **Total (peak)** | **~3.5 – 4.0 GB** | Under 8GB RAM + 8GB swap |

### Why This Works

- **Immich ML is the memory peak**: At ~1.5GB, it's the single largest consumer, but not always running
- **Swap headroom**: 8GB swap prevents OOM kills during brief spikes
- **N95 CPU is adequate**: Most services are I/O-bound, not CPU-bound; N95's multi-core handles typical workloads
- **256GB SSD is ample**: OS (~10GB) + Docker images (~20–30GB) + small databases (~10GB) + overhead leaves room for caches and logs

### Recommendations

- Monitor memory with Netdata; set alerts if usage exceeds 6GB sustained
- If adding heavy workloads (e.g., Plex transcoding), consider increasing SSD or adding RAM
- Current setup is sustainable for 1–3 users, typical homelab scale

---

## Hardware Adequacy Summary

The Beelink Mini S12 is **well-suited** for this architecture:

- **CPU**: Intel N95 (4-core, N-series efficiency) handles container orchestration and service requests comfortably
- **RAM**: 8GB is sufficient with disciplined container limits and 8GB swap as overflow buffer
- **Storage**: 256GB SSD adequate for OS and Docker layers; NAS holds user data
- **Form factor**: Fanless Mini PC is silent, reliable, and consumes <25W

This setup scales well for a home office, small team, or hobby projects. For 10+ concurrent users or high-throughput workloads, consider a larger/more powerful system.
