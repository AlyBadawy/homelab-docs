# Homelab Architecture Overview

## System Summary

- **Hardware**: Beelink Mini S12 (Intel N95, 8GB DDR4, 256GB SSD, 2.5 GbE, Wi-Fi 7)
- **OS**: Ubuntu Server 24.04 LTS
- **Identity**: Samba 4 Active Directory DC (bare-metal, provisioned before Docker)
- **Container Runtime**: Docker (all application services containerized)
- **NAS**: UGreen NAS (upgrading to UniFi UNAS 4) for bulk storage
- **AREDN Node**: Mikrotik hAC 2 Lite running WireGuard mesh tunnel (VLAN 40/41)
- **Domain**: alybadawy.com (DNS via Vercel, internal services via *.in.alybadawy.com)
- **AD Realm**: `ID.IN.ALYBADAWY.COM` (Samba handles DNS for the `id.in.alybadawy.com` zone)
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
   - Single wildcard cert: `*.in.alybadawy.com`
   - Provisioned via acme.sh using Vercel DNS API
   - Covers all subdomains with one certificate (photos.in.alybadawy.com, cloud.in.alybadawy.com, etc.)
   - Auto-renewal built into Docker stack

4. **Centralized Identity via Active Directory**
   - Samba 4 AD DC is the single identity provider — provisioned bare-metal before Docker
   - Services authenticate via LDAP/LDAPS (Nextcloud, NAS) or OIDC/OAuth2 backed by AD
   - AD is set up first in the rebuild sequence because Docker and other services depend on it for auth
   - All services are LAN+VPN only; AD credentials managed centrally and stored in a password manager

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
                 ┌─────────────┴──────────────┐
                 |                            |
      [Samba 4 AD DC]              [Nginx Proxy Manager]
     (LDAP/Kerberos/DNS)           (Port 443, TLS termination)
     (bare-metal, pre-Docker)               |
                 |              ┌───────────┼───────────┐
                 |              |           |           |
                 └──────→ [PostgreSQL] [Services]   [Redis]
                                       (Nextcloud,
                                        Immich,
                                        Home Assistant)
                                            |
                                       [NAS via NFS]
                                     (User data, media)
```

### Layer Descriptions

**Layer 1: Gateway & VPN**
- UDR7 handles internet firewall, DHCP, internal DNS, and VPN endpoint
- Internal DNS resolves `*.in.alybadawy.com` to homelab server LAN IP
- UDR7 forwards the `id.in.alybadawy.com` DNS zone to Samba AD at 172.20.20.5
- VPN clients receive LAN IPs and see the same internal DNS

**Layer 1b: Identity (Samba 4 Active Directory)**
- Runs bare-metal on the homelab server — provisioned before Docker during initial setup
- Provides LDAP/LDAPS for service authentication and Kerberos for AD clients
- Runs its own DNS server (port 53) for the `id.in.alybadawy.com` zone; forwards other queries to UDR7
- Must be healthy before any auth-dependent Docker services start

**Layer 2: Reverse Proxy**
- Nginx Proxy Manager listens on ports 80 (HTTP redirect) and 443 (HTTPS)
- Terminates TLS using the wildcard cert
- Routes requests to appropriate backend services based on hostname

**Layer 3: Application Services**
- Nextcloud, Immich, Home Assistant, and other apps run in isolated Docker containers
- Services authenticate users against Samba AD via LDAP or OIDC
- Services may use local databases (PostgreSQL, Redis) for sessions and caching
- Large persistent data stored on NAS

**Layer 4: Data Persistence**
- PostgreSQL: global relational database (shared by identity and app services)
- Redis: cache and session store
- NAS (NFS mount): user files, media library, backups

---

## Typical Request Flow

### Example: User accessing Nextcloud

1. **DNS Resolution**
   - User browses to `cloud.in.alybadawy.com`
   - UDR7's internal DNS resolves to homelab IP (e.g., 172.20.20.5)

2. **TLS Handshake**
   - Client connects to NPM on port 443
   - NPM presents wildcard cert for `*.in.alybadawy.com`
   - TLS tunnel established

3. **Request Routing**
   - NPM examines Host header, looks up routing rule for `nextcloud`
   - NPM proxies request to Nextcloud container (e.g., http://nextcloud:80)

4. **Authentication Check**
   - Nextcloud checks the user's session cookie against its own local user database
   - If no session, Nextcloud presents its own login page

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
| Samba 4 AD DC | 200–350 MB | Bare-metal; runs samba-ad-dc service |
| Docker daemon | 100 MB | Container runtime overhead |
| Dashboard (Rails) | 100 MB | Custom homelab dashboard |
| Portainer | 50 MB | Container management UI |
| Netdata | 150 MB | Metrics collection & dashboard |
| Nginx Proxy Manager | 50 MB | Reverse proxy & routing |
| PostgreSQL (global) | 150 MB | Runs with shared_buffers=256MB |
| Redis | 50 MB | Session & cache store |
| Nextcloud | 300 MB | PHP-FPM container |
| Immich (server) | 400 MB | API server |
| Immich (ML service) | 800 MB – 1.5 GB | ML model inference (heaviest) |
| Home Assistant | 400 MB | Automation engine |
| **Total (peak)** | **~3.1 – 3.9 GB** | Under 8GB RAM + 8GB swap |

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
- **Network**: 2.5 GbE wired connection ensures fast NFS transfers to the NAS (on the same VLAN)
- **AREDN**: Mikrotik hAC 2 Lite handles the AREDN mesh uplink on its own dedicated VLANs (40/41), isolated from homelab services
- **Form factor**: Mini PC is silent, reliable, and consumes <25W

This setup scales well for a home office, small team, or hobby projects. For 10+ concurrent users or high-throughput workloads, consider a larger/more powerful system.
