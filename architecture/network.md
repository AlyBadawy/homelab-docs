# Homelab Network Architecture

## Physical Network Topology

```
                    ┌─────────────┐
                    │  ISP Modem  │
                    └──────┬──────┘
                           │ WAN
                    ┌──────┴────────────────────────────────────────┐
                    │          Unifi Dream Router 7 (UDR7)          │
                    │  Gateway / Router / Firewall / VPN / DNS | AP │
                    └──────┬────────────────────────────────────────┘
                           |
                           │ LAN (trunk — all VLANs)
                           |
              ┌────────────┴──────────────────────────────────────────────────────┐
              │       8-Port Switch (+1 uplink)                                   │
              └──┬──────┬─────────────┬────────────┬────────────┬───────────┬─────┘
                 │      │             │            │            │           │
          VLAN 10│    VLAN 30      VLAN 20     VLAN 20       VLAN 40      VLAN 41
                 │      │             │            │            │           │
         Personal PC  Work PC     Homelab         NAS       AREDN WAN     AREDN LAN
                                172.20.20.5 172.20.20.10   172.20.40.2   10.6.229.9
```

> **AREDN-LAN (VLAN 41):** The UDR7 does not run a DHCP server on this VLAN. Instead, it acts as a **DHCP relay** — forwarding DHCP requests to the AREDN node at `10.6.229.9`, which decides the subnet and IP assignments internally. The AREDN node is the authoritative DHCP server and default gateway for the `10.6.229.8/29` segment.

---

## VLAN Design

The home network uses UniFi VLAN segmentation for isolation between device classes. The homelab server and NAS both reside on the **Servers VLAN**.

| Name        | VLAN ID | Subnet             | Gateway         | Purpose                                   |
| ----------- | ------- | ------------------ | --------------- | ----------------------------------------- |
| Default     | 1       | 172.20.1.0/24      | 172.20.1.1      | Management / primary devices              |
| Personal    | 10      | 172.20.10.0/24     | 172.20.10.1     | Personal computers, phones                |
| **Servers** | **20**  | **172.20.20.0/24** | **172.20.20.1** | **Homelab server + NAS**                  |
| Work        | 30      | 172.20.30.0/24     | 172.20.30.1     | Work devices (isolated)                   |
| Aredn-Wan   | 40      | 172.20.40.0/24     | 172.20.40.1     | AREDN mesh WAN uplink                     |
| Aredn-Lan   | 41      | 10.6.229.8/29      | —               | AREDN mesh LAN — DHCP relay to AREDN node |
| IoT         | 100     | 172.20.100.0/24    | 172.20.100.1    | Smart home / IoT devices                  |
| Guest       | 200     | 172.20.200.0/24    | 172.20.200.1    | Guest Wi-Fi (isolated)                    |

### Key Device Assignments

| Device                   | VLAN           | IP           | Notes                                    |
| ------------------------ | -------------- | ------------ | ---------------------------------------- |
| UDR7 (Servers gateway)   | Servers (20)   | 172.20.20.1  | Main DNS server for all VLANs            |
| Homelab server (Beelink) | Servers (20)   | 172.20.20.5  | OS hostname: `dc.id.in.alybadawy.com`    |
| UniFi NAS                | Servers (20)   | 172.20.20.10 | SMB and NFS exports                      |
| AREDN node (WAN port)    | Aredn-Wan (40) | 172.20.40.2  | AREDN mesh WAN uplink                    |
| AREDN node (LAN port)    | Aredn-Lan (41) | 10.6.229.9   | AREDN mesh LAN + DHCP server for VLAN 41 |

> [!CAUTION]
> **Note on hostname:** Samba AD requires the machine's OS hostname to match the DC FQDN (`dc.id.in.alybadawy.com`). All service subdomains (`*.in.alybadawy.com`) continue to work via UDR7 local DNS records pointing to `172.20.20.5`.

> [!TIP]
> **Why Servers VLAN for both homelab and NAS?** Placing both on the same VLAN means NFS traffic stays entirely within VLAN 20 at LAN speeds, without traversing firewall rules between VLANs. Other devices access services through NPM via inter-VLAN routing controlled by the UDR7 firewall.

---

## DNS Architecture

### Public DNS (Vercel)

Only `in.alybadawy.com` is exposed publicly. Individual service subdomains are internal-only. There is no DNS record for those as they are only reachable from LAN or VPN.

```
in.alybadawy.com          A       → UDR7 public IP (auto-updated by DDNS script)
```

- No public ports are forwarded to the homelab — all services are LAN + VPN only
- The A record is updated automatically when the Homelab's public IP changes

### Internal DNS (UniFi UDR7 Built-in DNS)

The UDR7 DNS server overrides public DNS for internal clients, ensuring direct routing without hairpin NAT. Configured under **UniFi → Network → DNS → Local DNS Records**.

**Host (A) Records:**

| Domain                 | IP             | Notes                                            |
| ---------------------- | -------------- | ------------------------------------------------ |
| `lab.in.alybadawy.com` | `172.20.20.5`  | Homelab server — all service CNAMEs resolve here |
| `nas.in.alybadawy.com` | `172.20.20.10` | UniFi NAS direct hostname                        |

**Alias (CNAME) Records — all point to `lab.in.alybadawy.com`:**

| Domain                     | → Target               |
| -------------------------- | ---------------------- |
| `proxy.in.alybadawy.com`   | `lab.in.alybadawy.com` |
| `dockers.in.alybadawy.com` | `lab.in.alybadawy.com` |
| `monitor.in.alybadawy.com` | `lab.in.alybadawy.com` |
| `photos.in.alybadawy.com`  | `lab.in.alybadawy.com` |
| `cloud.in.alybadawy.com`   | `lab.in.alybadawy.com` |
| `ha.in.alybadawy.com`      | `lab.in.alybadawy.com` |

Each service subdomain resolves to `lab.in.alybadawy.com` → `172.20.20.5`, landing on NPM. New services require a new CNAME entry here.

**DNS Forward Zones:**

Configured in **UniFi → Network → DNS → DNS Forwarding**:

| Zone                  | Forwarded To  | Purpose                                                    |
| --------------------- | ------------- | ---------------------------------------------------------- |
| `id.in.alybadawy.com` | `172.20.20.5` | Delegates all AD DNS queries to Samba DC on port 53        |
| `*.local.mesh`        | `10.6.229.9`  | Forwards all AREDN mesh hostname queries to the AREDN node |

- **AD zone:** Samba handles all records under `id.in.alybadawy.com` (DC hostname, Kerberos SRV, LDAP SRV). Samba forwards non-AD queries back to UDR7 at `172.20.20.1`.
- **AREDN zone:** The AREDN node manages `*.local.mesh` hostnames internally. UDR7 forwards queries to it, allowing homelab clients to resolve AREDN mesh services by hostname without any manual configuration.

### Why This Two-Level Setup Works for TLS

1. **Let's Encrypt Validation** — acme.sh performs DNS-01 challenge via Vercel API. Vercel adds the TXT record, Let's Encrypt validates it, and issues the wildcard cert.
2. **Internal Resolution** — LAN clients query UDR7 DNS; `photos.in.alybadawy.com` resolves directly to `172.20.20.5`, bypassing any internet roundtrip.
3. **TLS Validity** — The wildcard cert is publicly signed by Let's Encrypt, trusted by all major clients. No browser warnings.
4. **VPN Access** — VPN clients receive LAN IPs and use UDR7 DNS, so they resolve and access services identically to LAN clients.

---

## VPN Access

- **VPN Endpoint**: UDR7 (the router — not the homelab)
- **VPN Type**: WireGuard (UniFi built-in)
- **Client IP Pool**: Personal VLAN (172.20.10.0/24)

VPN clients receive a LAN IP, query UDR7 DNS, and access homelab services identically to local clients — same DNS, same TLS certs, no per-service configuration needed.

---

## Port Exposure

### Homelab Server Ports

| Port          | Service                | Scope                     | Purpose                                            |
| ------------- | ---------------------- | ------------------------- | -------------------------------------------------- |
| 53 (TCP/UDP)  | Samba AD (DNS)         | LAN — forwarded from UDR7 | AD zone DNS for `id.in.alybadawy.com`              |
| 88 (TCP/UDP)  | Samba AD (Kerberos)    | LAN                       | Kerberos ticket issuance for AD clients            |
| 389 (TCP)     | Samba AD (LDAP)        | LAN (Docker + LAN)        | LDAP bind for services (Nextcloud, NAS, etc.)      |
| 445 (TCP)     | Samba AD (SMB)         | LAN                       | AD DC management (samba-tool, Windows admin tools) |
| 464 (TCP/UDP) | Samba AD (Kerberos pw) | LAN                       | Kerberos password change                           |
| 636 (TCP)     | Samba AD (LDAPS)       | LAN (Docker + LAN)        | Secure LDAP for services                           |
| 49152–65535   | Samba AD (RPC)         | LAN                       | Dynamic RPC ports for AD DC management             |
| 80            | NPM (HTTP)             | LAN + VPN                 | Redirect HTTP → HTTPS                              |
| 443           | NPM (HTTPS)            | LAN + VPN                 | All web services via reverse proxy                 |
| 45876         | Beszel Agent           | Host (loopback to hub)    | Beszel hub → agent metrics stream (internal only)  |
| All others    | Docker containers      | Docker networks only      | Internal communication only                        |

> **NPM port 81 (admin UI):** Exposed temporarily during initial setup only. Once NPM is accessible at `https://proxy.in.alybadawy.com`, port 81 is removed from the stack and closed via UFW.

### Firewall Notes

- **No Port Forwarding** from internet to homelab — the UDR7 does not forward any external ports
- **Internet-Facing**: Only the UDR7 itself (VPN endpoint)
- **Homelab**: LAN + authenticated VPN access only
- **Implicit Deny**: Traffic not explicitly allowed by UDR7 rules is dropped

---

## Docker Network Topology

### Network Definitions

Three pre-created external bridge networks are used. All Docker networks are created before any stack is deployed and are shared across stacks.

#### 1. `proxy` Network

Services that need to be reachable through Nginx Proxy Manager:

- Nginx Proxy Manager
- Portainer
- Beszel Hub
- Nextcloud (front-end)
- Immich Server (front-end)

#### 2. `databases` Network

Shared data services — only PostgreSQL and Redis:

- PostgreSQL (global)
- Redis (global)

Services that need database access (Nextcloud, Immich Server) also join this network.

#### 3. `apps` Network

Application-layer communication — services that need to talk to each other:

- Nextcloud
- Immich Server
- Immich Machine Learning

#### Host Network

- **Beszel Agent** — runs with `network_mode: host` to observe real host-level CPU, memory, disk, and network interface stats

### Network Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                          Docker Host                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [Samba 4 AD DC] (bare-metal — ports 53/88/389/445/636)          │
│  [acme.sh]       (bare-metal — cron, writes to /opt/certs/)      │
│  [Beszel Agent]  (host network — port 45876)                     │
│                                                                  │
│  ┌──────────────── proxy ──────────────────────────────┐         │
│  │  [NPM]  [Portainer]  [Beszel Hub]                   │         │
│  │  [Nextcloud]  [Immich Server]                       │         │
│  └──────────────────────────┬──────────────────────────┘         │
│                             │ (ports 80, 443)                    │
│  ┌──────── databases ───────┼──────────────────────────┐         │
│  │  [PostgreSQL]  [Redis]   │                          │         │
│  │  [Nextcloud]  [Immich Server]  ← also on this net   │         │
│  └──────────────────────────┼──────────────────────────┘         │
│                             │                                    │
│  ┌──────────── apps ────────┴──────────────────────────┐         │
│  │  [Nextcloud]  [Immich Server]  [Immich ML]          │         │
│  └─────────────────────────────────────────────────────┘         │
│                                                                  │
│  [Home Assistant] → network_mode: host                           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

> **NPM port 81:** Active only during initial setup. Removed from the stack once NPM is proxied through itself at `https://proxy.in.alybadawy.com`.

---

## Traffic Flow Summary

### Inbound (Client → Homelab Service)

```
1. Client queries UDR7 DNS for photos.in.alybadawy.com
2. UDR7 responds with homelab IP (172.20.20.5)
3. Client initiates TLS to NPM (homelab:443)
4. NPM terminates TLS, checks Host header
5. NPM routes to Immich Server container (proxy network)
6. Immich validates user session (OIDC or built-in auth)
7. Immich queries PostgreSQL (databases network)
8. Immich fetches media from NAS (/mnt/nas/homelab/immich via NFS)
9. Response returns through NPM → TLS → Client
```

### Service → Authentication (LDAP example)

```
1. Nextcloud receives login request
2. Nextcloud connects to Samba AD at 172.20.20.5:636 (LDAPS)
3. Samba validates credentials against AD user database
4. Samba returns user attributes (uid, groups, email)
5. Nextcloud grants access and establishes a session
```

### AREDN Mesh DNS Resolution

```
1. Client queries UDR7 DNS for hostname.local.mesh
2. UDR7 DNS forward zone matches *.local.mesh
3. Query forwarded to AREDN node at 10.6.229.9
4. AREDN node resolves the mesh hostname and returns IP
5. Client connects directly to the AREDN mesh service
```

---

## Security Considerations

### Network Isolation

- **Samba AD ports (389, 636, 88, etc.)** — LAN-accessible; not forwarded to internet
- **PostgreSQL port 5432** — internal-only; containers on the `databases` network only
- **Redis port 6379** — internal-only; containers on the `databases` network only
- **NPM port 81** — temporary during initial setup only; removed once NPM is self-proxied

### TLS & Encryption

- **External traffic (ports 80, 443)** — always encrypted via TLS after HTTP→HTTPS redirect
- **Internal Docker traffic** — unencrypted but isolated to Docker bridge networks (not routable to LAN)
- **LDAPS (port 636)** — services connecting to Samba AD use LDAPS for encrypted credential exchange
- **NAS traffic (NFS)** — unencrypted but on trusted Servers VLAN; consider NFS over Kerberos for future hardening

### Authentication

- **Samba 4 AD** is the centralized identity provider; credentials are never stored per-service
- **LDAPS (636)** used by Nextcloud and NAS for secure directory lookups
- **Samba AD is not internet-facing** — LDAP/Kerberos ports are LAN-only
- **VPN access** requires WireGuard keys; VPN clients are treated as LAN users

### Firewall Defense-in-Depth

- **UDR7 firewall** — first line of defense; blocks unsolicited inbound traffic
- **Docker bridge isolation** — second line; prevents direct access to internal ports from LAN
- **Samba AD auth** — third line; services validate credentials against AD regardless of network path

---

## Future Enhancements

1. **NFS over Kerberos** — encrypt NFS traffic using AD-integrated Kerberos for the NAS mounts
2. **mTLS between services** — add client certificates for Docker-to-Docker communication if zero-trust posture is required
3. **IoT VLAN automation** — UniFi automation rules between Home Assistant and IoT VLAN devices
4. **Audit logging** — centralize logs from NPM and services for security review
