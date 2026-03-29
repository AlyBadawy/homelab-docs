# ADR-001: Operating System Selection for Homelab Server

## Status
**Accepted**

## Date
2026-03-29

---

## Context

### Hardware Specification
- **Device**: Beelink Mini PC Mini S12
- **Processor**: Intel N95 (12th Generation, 4-core, up to 3.4 GHz)
- **RAM**: 8 GB DDR4
- **Storage**: 256 GB SSD
- **Power Profile**: Ultra-low consumption mini PC ideal for 24/7 operation

### Use Case
This homelab server will run containerized services exclusively via Docker. The system will host the following applications:

**Infrastructure & Monitoring:**
- Portainer (Docker container management UI)
- Netdata (system monitoring and performance analytics)
- Nginx Proxy Manager (reverse proxy and SSL certificate management)

**Identity & Authentication:**
- LLDAP (LDAP directory service)
- Authentik (modern identity provider with OIDC/SAML support)

**Data & Storage:**
- Nextcloud (file synchronization and sharing)
- Immich (photo and video management)
- PostgreSQL (primary relational database)
- Redis (in-memory caching and sessions)

**Home Automation:**
- Home Assistant (home automation hub)

### Network Context
- **Not publicly exposed** to the internet
- **Access**: Restricted to local area network (LAN)
- **Remote Access**: Via VPN only
- **Availability Requirements**: High availability for local services; expected 24/7 uptime

### Operational Constraints
- Owner has no strong OS preference
- Minimal administrator overhead desired
- Preference for hands-off security patching
- Docker is the primary application container platform
- Preference for well-documented self-hosting community support

---

## Decision

**Adopt Ubuntu Server 24.04 LTS as the operating system for the homelab server.**

---

## Rationale

### Long-Term Support & Security
Ubuntu Server 24.04 LTS provides a **5-year standard support window** (until April 2029), with optional **Extended Security Maintenance (ESM)** extending coverage to **10 years** (until April 2034). This extended window aligns well with the 24/7 nature of a homelab server and eliminates the need for OS upgrades over a decade.

For comparison:
- Debian 12 has approximately 3 years of standard support
- Fedora Server has only 13 months of support per release
- Rocky Linux LTS is enterprise-focused, not optimized for small infrastructure

The 10-year ESM window ensures all security vulnerabilities on this hardware can be patched without requiring a full OS migration, reducing operational toil.

### Container & Docker Ecosystem
Docker CE (Community Edition) has **first-class, official support for Ubuntu** via:
- **Official Docker APT repository** with curated Ubuntu packages
- **Upstream Docker community prioritizes Ubuntu** for release testing and validation
- **Canonical maintains strong Docker partnerships**, ensuring rapid integration of new features

The vast majority of self-hosted application documentation (Authentik, Immich, Nextcloud, Home Assistant) assumes an Ubuntu base, with Docker installation instructions explicitly tested on Ubuntu. This reduces the risk of encountering undocumented incompatibilities.

### Self-Hosting Community & Documentation
Ubuntu dominates the self-hosted and homelab community ecosystem. The services planned for this infrastructure have:
- **Native Ubuntu support**: Authentik, Immich, Nextcloud, Home Assistant all provide Ubuntu-first installation guides
- **Strongest troubleshooting community**: The majority of self-hosting communities, subreddits, Discord channels, and forums have significantly more Ubuntu-specific troubleshooting resources
- **Established best practices**: Configuration patterns, security hardening, and Docker Compose examples for these services are predominantly tested on Ubuntu

This ecosystem advantage reduces debugging time and provides accessible solutions for common issues.

### Hardware Compatibility: Intel N95 & Kernel Support
The Intel N95 processor is a **12th-generation mobile-class processor** with:
- Modern instruction sets (Intel AVX-512, newer graphics encoders)
- Recent microcode security updates
- Potential GPU acceleration for transcoding (if Jellyfin or Immich add it in future)

Ubuntu Server 24.04 ships with **Linux kernel 6.8**, which includes:
- **Improved Intel N-series GPU driver support** via the Intel i915 driver (iGPU acceleration)
- **Better power management** for ultra-mobile processors
- **Newer security mitigations** (including Spectre/Meltdown patches) optimized for 12th-gen Intel

Debian 12 ships with kernel 6.1, which has adequate but less optimized support for this newer hardware. If transcoding GPU acceleration is added in the future (e.g., Jellyfin with Immich), Ubuntu's kernel would provide better device driver support out of the box.

### Unattended Security Updates
Ubuntu Server 24.04 includes **`unattended-upgrades`** by default, with:
- **Reliable, well-tested out-of-the-box configuration** for automated security patching
- **Systemd integration** for scheduled patches with minimal disruption
- **Rollback capability** on update failures
- **Notification support** for update logs

This eliminates manual security patching for a 24/7 homelab server, reducing operational toil.

### Lean System Option
While Ubuntu is slightly larger than Debian at install time (~1.5 GB vs ~1.2 GB), this is negligible on a 256 GB SSD. Ubuntu's default snap daemon can be **completely disabled** without impacting core system functionality, reducing memory overhead and creating a system as lean as Debian if desired.

---

## Alternatives Considered

### Debian 12 (Bookworm)
**Pros:**
- Minimal, stable base system with small install footprint (~1.2 GB)
- Exceptional long-term stability and predictability
- Official Docker CE support via repository
- True "Debian stable" philosophy—no snap daemon by default

**Cons:**
- Only ~3 years of standard support (until June 2026); requires migration sooner
- Kernel 6.1 is less optimized for 12th-gen Intel hardware
- Smaller self-hosting community ecosystem; less Ubuntu-specific documentation available
- Less aggressive upstream driver backports for newer hardware

**Verdict:** Debian 12 is objectively stable and efficient, but lacks the support window duration and community ecosystem advantages of Ubuntu 24.04. Recommended as a secondary choice if resource constraints or minimalism were higher priorities.

### Proxmox VE 8
**Pros:**
- Full virtualization platform with KVM hypervisor
- Built on Debian 12, inheriting its stability
- Enterprise-grade VM and container management

**Cons:**
- **Significant overkill for this use case**—all services run containerized in Docker; virtualization adds zero value
- Adds substantial complexity: requires learning Proxmox UI, VM provisioning, storage pools, and clustering concepts
- Consumes additional RAM and CPU overhead for hypervisor layer (~2-3 GB of the 8 GB RAM)
- Larger attack surface for a non-publicly exposed server
- Maintenance burden: requires Proxmox-specific troubleshooting and community (smaller than Ubuntu)

**Verdict:** Proxmox is ideal for homelabs running multiple distinct workloads in VMs (e.g., Windows, multiple Linux distros, experimental environments). This use case runs all containers on a single Docker host, making Proxmox unnecessary.

### Fedora Server 40
**Pros:**
- Very recent kernels (6.8+) with excellent hardware support
- Regular updates with new features and improved driver support
- Large developer community

**Cons:**
- Only **13 months of support per release**—requires OS upgrades every 6-12 months
- Constant churn of dependency updates; less predictable stability for long-running services
- Smaller self-hosting community compared to Ubuntu/Debian
- Minimal documentation for self-hosted applications targeting Fedora first
- Unsuitable for a 24/7 homelab expecting minimal maintenance

**Verdict:** Fedora's rapid release cycle is inappropriate for a long-running homelab server requiring hands-off operation. Better suited for development machines or frequent experimenters.

### Rocky Linux 9
**Pros:**
- 10-year support window (matches Ubuntu 24.04 ESM)
- Enterprise-grade stability (derived from RHEL)
- Strong security record

**Cons:**
- Enterprise-focused ecosystem; minimal self-hosting community support
- Very few self-hosted application guides target Rocky Linux
- Docker CE support present but less prioritized than Ubuntu
- Larger base install and more aggressive default hardening rules for a homelab context

**Verdict:** Rocky Linux is excellent for enterprise infrastructure but provides no advantages over Ubuntu 24.04 for a self-hosted homelab. The self-hosting community ecosystem is significantly smaller.

---

## Consequences

### Positive Consequences

1. **Extended Support Horizon**: 5-10 year support window eliminates OS migration overhead for a decade, aligning with the long-term nature of homelab infrastructure.

2. **Optimal Docker Integration**: Official Ubuntu Docker repository and community validation ensure zero friction deploying containerized services. Portainer, Nginx Proxy Manager, and other admin tools are thoroughly tested on Ubuntu.

3. **Strong Self-Hosting Community**: Authentik, Immich, Nextcloud, and Home Assistant have largest communities on Ubuntu, providing rapid troubleshooting resources and established best practices.

4. **Hardware-Optimized Kernel**: Linux 6.8 provides optimized support for Intel N95's modern instruction sets and iGPU, improving performance and reducing power consumption. Future GPU-accelerated transcoding in Immich would benefit from this.

5. **Hands-Off Security**: `unattended-upgrades` provides reliable, out-of-the-box automated security patching, critical for a 24/7 homelab.

6. **Excellent Documentation**: Self-hosted application guides, Docker Compose examples, and security hardening guides are overwhelmingly available for Ubuntu, reducing research and debugging time.

7. **Low Migration Risk**: If the homelab needs to grow or scale, Ubuntu Server knowledge and tooling are ubiquitous in the Linux ecosystem, making future migrations or expansions straightforward.

### Negative Consequences

1. **Larger Base Install**: Ubuntu Server 24.04 base install is approximately 1.5 GB, compared to Debian's 1.2 GB. However, on a 256 GB SSD, this 300 MB difference is negligible.

2. **Snap Daemon Overhead**: Ubuntu includes `snapd` by default, consuming ~15-30 MB of RAM and ~1-2% CPU for periodic refresh checks. This is easily mitigated by disabling snapd entirely (see **Notes** section).

3. **Slight Learning Curve**: If the administrator has existing Debian knowledge, Ubuntu's apt-based package management is identical, but systemd integration and some default configurations differ slightly.

4. **Less Minimal Philosophy**: Ubuntu targets user-friendliness over absolute minimalism. Some administrators may find Debian's stripped-down approach more philosophically aligned.

---

## Implementation Notes

### Post-Installation Recommendations

1. **Disable snapd (Optional, for minimal footprint)**:
   ```bash
   sudo systemctl disable snapd snapd.socket snapd.seeded
   sudo systemctl stop snapd snapd.socket snapd.seeded
   sudo apt remove --purge snapd
   ```
   This reduces memory usage and aligns the system with Debian-like minimalism if preferred.

2. **Enable Unattended Upgrades** (should be installed by default):
   ```bash
   sudo apt update
   sudo apt install unattended-upgrades apt-listchanges
   sudo dpkg-reconfigure unattended-upgrades
   ```

3. **Install Docker CE** via official Docker repository:
   ```bash
   curl -fsSL https://get.docker.com -o get-docker.sh
   sudo sh get-docker.sh
   ```

4. **Configure Automatic Reboot** for kernel updates (edit `/etc/apt/apt.conf.d/50unattended-upgrades`):
   ```
   Unattended-Upgrade::Automatic-Reboot "true";
   Unattended-Upgrade::Automatic-Reboot-Time "03:00";
   ```

5. **Install Portainer** for container management (recommended for this use case):
   ```bash
   docker run -d -p 8000:8000 -p 9000:9000 --name portainer \
     --restart always \
     -v /var/run/docker.sock:/var/run/docker.sock \
     -v portainer_data:/data \
     portainer/portainer-ce:latest
   ```

### Hardware-Specific Tuning

The Intel N95's power efficiency should be leveraged:

1. **Enable CPU frequency scaling** (Linux powersave governor):
   ```bash
   echo powersave | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
   ```

2. **Monitor system temperature** with Netdata (already planned) to ensure passive cooling is adequate.

3. **Consider iGPU acceleration** for Immich transcoding in future upgrades; Linux 6.8 kernel support is already present.

---

## Related Decisions

- **ADR-002**: Docker networking and reverse proxy strategy (Nginx Proxy Manager configuration)
- **ADR-003**: Backup and disaster recovery strategy for persistent volumes
- **ADR-004**: Network architecture and VPN access controls

---

## Approval & Sign-Off

- **Decided By**: Homelab Owner / Administrator
- **Date**: 2026-03-29
- **Status**: Accepted and ready for implementation

---

## References

- [Ubuntu Server 24.04 LTS Release Notes](https://wiki.ubuntu.com/NobleNumbat/ReleaseNotes)
- [Docker CE Official Installation Guide](https://docs.docker.com/engine/install/ubuntu/)
- [Linux Kernel 6.8 Release Notes](https://kernelnewbies.org/Linux_6.8)
- [Unattended Upgrades Ubuntu Documentation](https://wiki.ubuntu.com/UnattendedUpgrades)
- [Authentik Official Installation Guide](https://docs.goauthentik.io/)
- [Immich Self-Hosted Documentation](https://immich.app/docs/)
- [Nextcloud Administrator's Manual](https://docs.nextcloud.com/)
