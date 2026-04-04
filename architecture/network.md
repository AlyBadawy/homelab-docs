# Homelab Network Architecture

## VLAN Design

The home network uses UniFi VLAN segmentation for isolation between device classes. The homelab server and NAS both reside on the **Servers VLAN**.

| Name        | VLAN ID | Subnet             | Gateway         | Purpose                      |
| ----------- | ------- | ------------------ | --------------- | ---------------------------- |
| Default     | 1       | 172.20.1.0/24      | 172.20.1.1      | Management / primary devices |
| Personal    | 10      | 172.20.10.0/24     | 172.20.10.1     | Personal computers, phones   |
| **Servers** | **20**  | **172.20.20.0/24** | **172.20.20.1** | **Homelab server + NAS**     |
| Work        | 30      | 172.20.30.0/24     | 172.20.30.1     | Work devices (isolated)      |
| Aredn-Wan   | 40      | 172.20.40.0/24     | 172.20.40.1     | AREDN mesh WAN uplink        |
| Aredn-Lan   | 41      | 10.6.229.8/29      | —               | AREDN mesh LAN segment       |
| IoT         | 100     | 172.20.100.0/24    | 172.20.100.1    | Smart home / IoT devices     |
| Guest       | 200     | 172.20.200.0/24    | 172.20.200.1    | Guest Wi-Fi (isolated)       |

### Key Device Assignments (Servers VLAN — 172.20.20.0/24)

| Device                      | IP           | Notes                                                                 |
| --------------------------- | ------------ | --------------------------------------------------------------------- |
| UDR7 (Servers VLAN gateway) | 172.20.20.1  | Main DNS server for all VLANs; forwards `id.in.alybadawy.com` to DC  |
| NAS (UGreen → UniFi UNAS 4) | 172.20.20.10 | NFS exports for Nextcloud + Immich                                    |
| Homelab server (Beelink)    | 172.20.20.5  | OS hostname: `dc.id.in.alybadawy.com` (required by Samba AD); also reachable as `lab.in.alybadawy.com` via UDR7 DNS alias |

> **Note on hostname:** Samba AD requires the machine's OS hostname to match the DC FQDN (`dc.id.in.alybadawy.com`). All existing DNS aliases (`lab.in.alybadawy.com`, `*.in.alybadawy.com` service subdomains) continue to work via UDR7 local DNS records pointing to `172.20.20.5`.

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
- The wildcard CNAME allows any subdomain (photos.in.alybadawy.com, cloud.in.alybadawy.com, etc.) to resolve through the same public A record
- Let's Encrypt DNS-01 challenges use the Vercel API to validate domain ownership for wildcard cert issuance

### Internal DNS (UniFi UDR7 Built-in DNS)

The UDR7 DNS server overrides public DNS for internal clients, ensuring direct routing without hairpin NAT. Records are configured under **UniFi → Network → DNS → Local DNS Records**.

**Host (A) Records:**

| Domain                 | IP             | Notes                                            |
| ---------------------- | -------------- | ------------------------------------------------ |
| `lab.in.alybadawy.com` | `172.20.20.5`  | Homelab server — all CNAMEs resolve through this |
| `nas.in.alybadawy.com` | `172.20.20.10` | NAS direct hostname                              |
| `localnode.local.mesh` | `10.6.229.9`   | AREDN mesh node                                  |

**Alias (CNAME) Records — all point to `lab.in.alybadawy.com`:**

| Domain                     | →                      |
| -------------------------- | ---------------------- |
| `proxy.in.alybadawy.com`   | `lab.in.alybadawy.com` |
| `dockers.in.alybadawy.com` | `lab.in.alybadawy.com` |
| `netdata.in.alybadawy.com` | `lab.in.alybadawy.com` |
| `nas-ui.in.alybadawy.com`  | `lab.in.alybadawy.com` |
| `photos.in.alybadawy.com`  | `lab.in.alybadawy.com` |
| `cloud.in.alybadawy.com`   | `lab.in.alybadawy.com` |

All TTLs are set to Auto. Rather than a single wildcard `*.in.alybadawy.com` A record, each service subdomain is explicitly registered as a CNAME. This makes DNS intent visible and avoids resolving non-existent services.

- Internal clients query UDR7's DNS
- Each subdomain CNAME resolves to `lab.in.alybadawy.com`, which resolves to `172.20.20.5`
- Traffic arrives directly at NPM on the homelab — no internet roundtrip (no hairpin NAT)
- New services require a new CNAME entry in UDR7's Local DNS Records

**DNS Forward Zone (Active Directory):**

| Zone                    | Forwarded To  | Purpose                                              |
| ----------------------- | ------------- | ---------------------------------------------------- |
| `id.in.alybadawy.com`   | `172.20.20.5` | Delegates all AD DNS queries to Samba DC on port 53  |

Configured in UniFi → Network → DNS → DNS Forwarding. This allows all VLANs to resolve AD records (DC hostname, Kerberos SRV records, LDAP SRV records) through the UDR7, which in turn asks Samba. Samba forwards non-AD queries back to `172.20.20.1` (UDR7).

### Why This Two-Level Setup Works for TLS

1. **Let's Encrypt Validation**
   - acme.sh (running on homelab) performs DNS-01 challenge via Vercel API
   - Vercel API updates the public A and CNAME records
   - Let's Encrypt validates ownership and issues the wildcard cert for `*.in.alybadawy.com`

2. **Internal Client Resolution**
   - LAN clients query UDR7 DNS
   - `photos.in.alybadawy.com` resolves to homelab server IP (172.20.20.5)
   - Request hits NPM directly on the local network (no internet roundtrip)

3. **TLS Validity**
   - The wildcard cert is publicly signed (by Let's Encrypt)
   - All major browsers and OS trust Let's Encrypt as a root CA
   - Clients connecting to `photos.in.alybadawy.com` receive a valid cert matching the hostname
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

| Port          | Service        | Scope                        | Purpose                                              |
| ------------- | -------------- | ---------------------------- | ---------------------------------------------------- |
| 53 (TCP/UDP)  | Samba AD (DNS) | LAN (Servers VLAN + forwarded from UDR7) | AD zone DNS — `id.in.alybadawy.com`    |
| 88 (TCP/UDP)  | Samba AD (Kerberos) | LAN                    | Kerberos ticket issuance for AD clients              |
| 389 (TCP)     | Samba AD (LDAP) | LAN (Docker + LAN)          | LDAP bind for services (Nextcloud, NAS, etc.)        |
| 445 (TCP)     | Samba AD (SMB) | LAN                          | AD DC management (samba-tool, Windows admin tools)   |
| 464 (TCP/UDP) | Samba AD (Kerberos pw) | LAN               | Kerberos password change                             |
| 636 (TCP)     | Samba AD (LDAPS) | LAN (Docker + LAN)         | Secure LDAP for services                             |
| 49152–65535   | Samba AD (RPC) | LAN                          | Dynamic RPC ports for AD DC replication/management   |
| 80            | NPM (HTTP)     | LAN + VPN                    | Redirect HTTP → HTTPS                                |
| 443           | NPM (HTTPS)    | LAN + VPN                    | All web services via reverse proxy                   |
| 19999         | Netdata        | Host network, LAN accessible | Metrics dashboard (also proxied via NPM)             |
| All other service ports | (Docker containers) | Docker networks only | Internal communication only                 |

### Firewall Notes

- **UDR7 Firewall Rules**: User has pre-configured firewall rules on the UDR7
- **No Port Forwarding**: The UDR7 does NOT forward any ports from the internet to the homelab
- **Internet-Facing Endpoint**: Only the UDR7 itself is internet-facing (VPN service)
- **Homelab Isolation**: The homelab is only accessible via LAN or authenticated VPN
- **Implicit Deny**: Any traffic not explicitly allowed by UDR7 rules is dropped

---

## Docker Network Topology

### Network Design Goals

- Isolate services by function (proxy, apps)
- Minimize cross-network traffic (services talk over bridges)
- Identity (Samba AD) runs bare-metal — not inside Docker — and is reachable on host LAN ports

### Network Definitions

#### 1. `proxy` Network

**Purpose**: HTTP/HTTPS traffic entry point
**Services on this network**:

- Nginx Proxy Manager (main traffic handler)
- Any services that need external HTTP access via NPM

**Rationale**: NPM sits on the proxy network to receive incoming traffic and route it to backend services.

#### 2. `apps` Network

**Purpose**: Application services and their shared data stores
**Services on this network**:

- Nextcloud
- Immich (server + ML)
- Home Assistant
- PostgreSQL (global shared instance)
- Redis (global shared instance)

**Rationale**: All apps and their infrastructure share this bridge for low-latency internal communication.

> **Note on identity:** Samba 4 AD is a bare-metal service running on the host, not in Docker. Services that need LDAP/LDAPS authentication connect to `172.20.20.5:389` or `172.20.20.5:636` directly — or use `host.docker.internal` if needed — rather than via a Docker network.

### Network Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      Docker Host                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Samba 4 AD DC] (bare-metal, ports 53/88/389/445/636)      │
│        ↑ ↓  (LAN reachable; Docker containers connect via   │
│              host IP 172.20.20.5 or host.docker.internal)   │
│                                                             │
│  ┌──────────────── proxy network ────────────────┐          │
│  │                                               │          │
│  │           [Nginx Proxy Manager]               │          │
│  │                                               │          │
│  └─────────────────────────────────────────────-─┘          │
│           ↑                                                 │
│           │ (External traffic: ports 80, 443)               │
│                                                             │
│  ┌──────────────── apps network ────────────────────┐       │
│  │                                                  │       │
│  │  [Nextcloud]  ←→  [Immich]  ←→  [Home Assistant] │       │
│  │       ↓              ↓              ↓            │       │
│  │    [PostgreSQL] ←→ [Redis]                       │       │
│  │                                                  │       │
│  └─────────────────────────────────────────────---──┘       │
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

  apps:
    driver: bridge
    ipam:
      config:
        - subnet: 172.22.0.0/16

services:
  npm:
    networks:
      - proxy

  postgres:
    networks:
      - apps

  redis:
    networks:
      - apps

  nextcloud:
    networks:
      - proxy
      - apps

  immich-server:
    networks:
      - proxy
      - apps

  immich-ml:
    networks:
      - apps

  home-assistant:
    networks:
      - proxy
```

---

## Traffic Flow Summary

### Inbound (External/VPN Client → Homelab Service)

```
1. Client queries UDR7 DNS for photos.in.alybadawy.com
2. UDR7 responds with homelab IP (172.20.20.5)
3. Client initiates TLS to NPM (homelab:443)
4. NPM terminates TLS, checks Host header
5. NPM routes to Immich container on apps network
6. Immich validates the user session (OIDC or built-in auth backed by Samba AD)
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

### Service → Authentication Check (LDAP example)

```
1. Nextcloud receives login request
2. Nextcloud connects to Samba AD at 172.20.20.5:636 (LDAPS)
3. Samba validates credentials against the AD user database
4. Samba returns user attributes (uid, groups, email)
5. Nextcloud grants access and establishes a session
```

---

## Security Considerations

### Network Isolation

- **Samba AD ports (389, 636, 88, etc.)** are LAN-accessible within Servers VLAN; not forwarded to internet
- **PostgreSQL port 5432** is internal-only; accessed only by containers on the `apps` network
- **Redis port 6379** is internal-only; accessed only by containers on the `apps` network
- **NPM port 81 (admin UI)** is accessible from LAN but should use strong credentials; consider further restricting with firewall rules

### TLS & Encryption

- **External traffic (ports 80, 443)**: Always encrypted via TLS after HTTP→HTTPS redirect
- **Internal Docker traffic**: Unencrypted but isolated to local bridges (not routable to LAN)
- **LDAPS (port 636)**: Services connecting to Samba AD use LDAPS for encrypted credential exchange
- **NAS traffic (NFS)**: Unencrypted but on trusted LAN; consider NFS over TLS in future if security posture changes

### Authentication

- **Samba 4 AD** is the centralized identity provider; credentials are never stored per-service
- **LDAPS (636)** used by services like Nextcloud and NAS for secure directory lookups
- **Samba AD is not internet-facing**; LDAP/Kerberos ports are LAN-only
- **VPN access** requires WireGuard/OpenVPN keys; users on VPN are treated as LAN users

### Firewall Defense-in-Depth

- **UDR7 firewall**: First line of defense; blocks unsolicited inbound traffic
- **Docker bridge isolation**: Second line; prevents direct access to internal ports
- **Samba AD auth**: Third line; services validate credentials against AD regardless of network path

---

## Future Enhancements

1. **NFS over TLS**: If storing highly sensitive data, encrypt NFS with TLS-based protocols (e.g., NFSv4.2 with Kerberos)
2. **mTLS between services**: Add client certificates for Docker-to-Docker communication if zero-trust is desired
3. **Network segmentation**: Create separate VLANs for cameras, IoT devices, trusted clients (Home Assistant integration with UniFi automation)
4. **DDoS protection**: If domain becomes public, enable Cloudflare or similar CDN with DDoS mitigation
5. **Audit logging**: Collect and centralize logs from NPM, Authentik, and services for compliance/debugging
