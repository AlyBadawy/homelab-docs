# Homelab Network Architecture

## VLAN Design

The home network uses UniFi VLAN segmentation for isolation between device classes. The homelab server and NAS both reside on the **Servers VLAN**.

| Name | VLAN ID | Subnet | Gateway | Purpose |
|------|---------|--------|---------|---------|
| Default | 1 | 172.20.1.0/24 | 172.20.1.1 | Management / primary devices |
| Personal | 10 | 172.20.10.0/24 | 172.20.10.1 | Personal computers, phones |
| **Servers** | **20** | **172.20.20.0/24** | **172.20.20.1** | **Homelab server + NAS** |
| Work | 30 | 172.20.30.0/24 | 172.20.30.1 | Work devices (isolated) |
| Aredn-Wan | 40 | 172.20.40.0/24 | 172.20.40.1 | AREDN mesh WAN uplink |
| Aredn-Lan | 41 | 10.6.229.8/29 | — | AREDN mesh LAN segment |
| IoT | 100 | 172.20.100.0/24 | 172.20.100.1 | Smart home / IoT devices |
| Guest | 200 | 172.20.200.0/24 | 172.20.200.1 | Guest Wi-Fi (isolated) |

### Key Device Assignments (Servers VLAN — 172.20.20.0/24)

| Device | IP | Notes |
|--------|----|-------|
| UDR7 (Servers VLAN gateway) | 172.20.20.1 | Also the DNS server for this VLAN |
| NAS (UGreen → UniFi UNAS 4) | 172.20.20.10 | NFS exports for Nextcloud + Immich |
| Homelab server (Beelink) | 172.20.20.5 | Static IP — hostname `lab.in.alybadawy.com` |

> **Why Servers VLAN?** Placing both the homelab and NAS on the same VLAN means NFS traffic between them stays entirely within VLAN 20 at LAN speeds, without traversing firewall rules between VLANs. Personal and IoT devices access services through NPM via inter-VLAN routing controlled by the UDR7 firewall.

---

## DNS Architecture

### Public DNS (Vercel)

The public DNS records allow external Let's Encrypt validation and map the domain to the home network:

```
in.alybadawy.com          A record    → UDR7 public IP
                                            (auto-updated by user script)

*.in.alybadawy.com        CNAME       → in.alybadawy.com
```

- The A record points to the UDR7's public IP, which is dynamic and updated periodically by a script
- The wildcard CNAME allows any subdomain (photo.in.alybadawy.com, cloud.in.alybadawy.com, etc.) to resolve through the same public A record
- Let's Encrypt DNS-01 challenges use the Vercel API to validate domain ownership for wildcard cert issuance

### Internal DNS (UniFi UDR7 Built-in DNS)

The UDR7 DNS server overrides public DNS for internal clients, ensuring direct routing without hairpin NAT:

```
in.alybadawy.com          A record    → 172.20.20.1  (UDR7 Servers VLAN gateway)
lab.in.alybadawy.com      A record    → 172.20.20.5  (homelab server hostname)
*.in.alybadawy.com        A record    → 172.20.20.5  (wildcard → all services on homelab)
```

- Internal clients query UDR7's DNS
- Requests for `in.alybadawy.com` resolve to the UDR7 itself (gateway)
- Requests for subdomains (e.g., `photo.in.alybadawy.com`) resolve directly to the homelab server
- This avoids "hairpin NAT" — clients don't need to loop traffic through the internet gateway

### Why This Two-Level Setup Works for TLS

1. **Let's Encrypt Validation**
   - acme.sh (running on homelab) performs DNS-01 challenge via Vercel API
   - Vercel API updates the public A and CNAME records
   - Let's Encrypt validates ownership and issues the wildcard cert for `*.in.alybadawy.com`

2. **Internal Client Resolution**
   - LAN clients query UDR7 DNS
   - `photo.in.alybadawy.com` resolves to homelab server IP (172.20.20.5)
   - Request hits NPM directly on the local network (no internet roundtrip)

3. **TLS Validity**
   - The wildcard cert is publicly signed (by Let's Encrypt)
   - All major browsers and OS trust Let's Encrypt as a root CA
   - Clients connecting to `photo.in.alybadawy.com` receive a valid cert matching the hostname
   - No SSL warnings or browser errors

4. **External/VPN Access**
   - External users or VPN-connected devices also resolve to the homelab IP (via UDR7 DNS)
   - The TLS cert is valid for all subdomain patterns
   - Same cert, same security, works seamlessly across all networks

---

## VPN Access

### UniFi VPN Server Configuration

- **VPN Endpoint**: UDR7 (the router/gateway)
- **VPN Type**: Typically WireGuard or OpenVPN (UniFi supports both)
- **Client IP Pool**: Assigned from the same LAN subnet or a separate range within the UDR7's managed subnets

### VPN Client Flow

```
VPN Client (remote)
        |
        → Connects to UDR7 via WireGuard/OpenVPN
        |
        → Receives a LAN IP on the Personal VLAN (172.20.10.0/24)
        |
        → Queries UDR7 DNS for *.in.alybadawy.com
        |
        → Resolves to homelab IP (172.20.20.5)
        |
        → Connects to NPM on homelab, same as LAN client
```

- VPN clients are effectively on the LAN from a network perspective
- They see the same DNS, access the same services, use the same TLS certs
- No additional configuration needed per-service for VPN support

---

## Port Exposure

### Homelab Server Ports

| Port | Service | Scope | Purpose |
|------|---------|-------|---------|
| 80 | NPM (HTTP) | LAN + VPN | Redirect HTTP → HTTPS |
| 443 | NPM (HTTPS) | LAN + VPN | All web services via reverse proxy |
| 81 | NPM Admin UI | Local/LAN only | Web UI for proxy configuration |
| 19999 | Netdata | Host network, LAN accessible | Metrics dashboard (also proxied via NPM) |
| 3890 | LLDAP | Docker internal network only | LDAP protocol (not exposed to LAN) |
| All other service ports | (Application containers) | Docker networks only | Internal communication only |

### Firewall Notes

- **UDR7 Firewall Rules**: User has pre-configured firewall rules on the UDR7
- **No Port Forwarding**: The UDR7 does NOT forward any ports from the internet to the homelab
- **Internet-Facing Endpoint**: Only the UDR7 itself is internet-facing (VPN service)
- **Homelab Isolation**: The homelab is only accessible via LAN or authenticated VPN
- **Implicit Deny**: Any traffic not explicitly allowed by UDR7 rules is dropped

---

## Docker Network Topology

### Network Design Goals

- Isolate services by function (identity, proxy, apps)
- Minimize cross-network traffic (services talk over bridges)
- Ensure critical services (auth, database) are not directly exposed

### Network Definitions

#### 1. `proxy` Network
**Purpose**: HTTP/HTTPS traffic entry point
**Services on this network**:
- Nginx Proxy Manager (main traffic handler)
- Authentik (for authentication checks)
- Any services that need external HTTP access

**Rationale**: NPM and Authentik sit on the proxy network to intercept and route traffic before it reaches backend services.

#### 2. `identity` Network
**Purpose**: Identity and authentication infrastructure
**Services on this network**:
- LLDAP (LDAP server)
- Authentik server (processes auth requests)
- Authentik worker (background jobs)
- PostgreSQL (identity database)
- Redis (session/cache store)

**Rationale**: Isolated network for sensitive authentication data; only accessible to auth-aware services.

#### 3. `apps` Network
**Purpose**: Application services and their data stores
**Services on this network**:
- Nextcloud
- Immich (server + ML)
- Home Assistant
- PostgreSQL (shared by apps, also on identity network)
- Redis (shared by apps, also on identity network)

**Rationale**: Apps talk to each other and shared infrastructure on this bridge.

### Cross-Network Connectivity

**PostgreSQL**: Joined to both `identity` and `apps` networks
- Allows identity services (LLDAP, Authentik) to query their databases
- Allows app services (Nextcloud, Immich) to query their databases

**Redis**: Joined to both `identity` and `apps` networks
- Allows Authentik to cache sessions and tokens
- Allows apps to use Redis for caching and temporary data

**Authentik**: Joined to both `proxy` and `identity` networks
- Communicates with NPM over the proxy network for forward auth checks
- Communicates with LLDAP and PostgreSQL over the identity network

### Network Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      Docker Host                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────── proxy network ────────────────┐         │
│  │                                                 │         │
│  │  [Nginx Proxy Manager]  ←→  [Authentik]      │         │
│  │                                                 │         │
│  └──────────────────────────────────────────────┘         │
│           ↑                                                 │
│           │ (External traffic: ports 80, 443)             │
│                                                             │
│  ┌──────────────── identity network ────────────────┐      │
│  │                                                   │      │
│  │  [LLDAP] ←→ [Authentik] ←→ [PostgreSQL]        │      │
│  │                              ←→ [Redis]         │      │
│  │                                                   │      │
│  └───────────────────────────────────────────────┘      │
│                      ↑                                    │
│          (Authentik bridges proxy & identity)          │
│                                                             │
│  ┌──────────────── apps network ────────────────────┐     │
│  │                                                   │     │
│  │  [Nextcloud]  ←→  [Immich]  ←→  [Home Assistant]│     │
│  │       ↓              ↓              ↓             │     │
│  │    [PostgreSQL] ←→ [Redis]                      │     │
│  │                                                   │     │
│  └───────────────────────────────────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Docker Compose Network Setup (Conceptual)

```yaml
networks:
  proxy:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

  identity:
    driver: bridge
    ipam:
      config:
        - subnet: 172.21.0.0/16

  apps:
    driver: bridge
    ipam:
      config:
        - subnet: 172.22.0.0/16

services:
  npm:
    networks:
      - proxy

  authentik-server:
    networks:
      - proxy
      - identity

  authentik-worker:
    networks:
      - identity

  lldap:
    networks:
      - identity

  postgres:
    networks:
      - identity
      - apps

  redis:
    networks:
      - identity
      - apps

  nextcloud:
    networks:
      - apps

  immich-server:
    networks:
      - apps

  immich-ml:
    networks:
      - apps

  home-assistant:
    networks:
      - apps
```

---

## Traffic Flow Summary

### Inbound (External/VPN Client → Homelab Service)

```
1. Client queries UDR7 DNS for photo.in.alybadawy.com
2. UDR7 responds with homelab IP (172.20.20.5)
3. Client initiates TLS to NPM (homelab:443)
4. NPM terminates TLS, checks Host header
5. NPM routes to Immich container on apps network
6. Immich checks auth via Authentik (proxy & identity networks)
7. If authenticated, Immich queries PostgreSQL (apps network)
8. Immich fetches media from NAS (/mnt/nfs/immich)
9. Response returns through NPM → TLS → Client
```

### Intra-Service (Service → Service)

```
Example: Immich Server → PostgreSQL

1. Immich (apps network) connects to postgres:5432
2. Connection routed via Docker bridge to postgres container (also apps network)
3. Direct LAN-local communication, no TLS needed internally
4. Data transaction completes
```

### Service → Authentication Check

```
1. NPM receives request, configured with forward auth for photo.in.alybadawy.com
2. NPM internally connects to Authentik (proxy network)
3. Authentik validates session cookie/token
4. Authentik responds with 200 OK + Remote-User header
5. NPM allows request to proceed to Immich
6. Immich (optionally) also checks Remote-User for additional auth
```

---

## Security Considerations

### Network Isolation

- **LLDAP port 3890** is internal-only; never exposed to LAN or internet
- **PostgreSQL port 5432** is internal-only; accessed only by containers on identity/apps networks
- **Redis port 6379** is internal-only; accessed only by containers on identity/apps networks
- **NPM port 81 (admin UI)** is accessible from LAN but should use strong credentials; consider further restricting with firewall rules

### TLS & Encryption

- **External traffic (ports 80, 443)**: Always encrypted via TLS after HTTP→HTTPS redirect
- **Internal Docker traffic**: Unencrypted but isolated to local bridges (not routable to LAN)
- **NAS traffic (NFS)**: Unencrypted but on trusted LAN; consider NFS over TLS in future if security posture changes

### Authentication

- **Authentik forward auth** enforces login for all services, even if accessed directly via LAN
- **LLDAP** is not internet-facing; credentials are not transmitted over the internet
- **VPN access** requires WireGuard/OpenVPN keys; users on VPN are treated as LAN users

### Firewall Defense-in-Depth

- **UDR7 firewall**: First line of defense; blocks unsolicited inbound traffic
- **Docker bridge isolation**: Second line; prevents direct access to internal ports
- **Service-level auth**: Third line; Authentik validates user identity regardless of network

---

## Future Enhancements

1. **NFS over TLS**: If storing highly sensitive data, encrypt NFS with TLS-based protocols (e.g., NFSv4.2 with Kerberos)
2. **mTLS between services**: Add client certificates for Docker-to-Docker communication if zero-trust is desired
3. **Network segmentation**: Create separate VLANs for cameras, IoT devices, trusted clients (Home Assistant integration with UniFi automation)
4. **DDoS protection**: If domain becomes public, enable Cloudflare or similar CDN with DDoS mitigation
5. **Audit logging**: Collect and centralize logs from NPM, Authentik, and services for compliance/debugging
