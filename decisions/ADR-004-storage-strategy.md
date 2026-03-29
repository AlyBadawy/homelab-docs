# ADR-004: Storage Strategy for Homelab

**Date:** 2026-03-29
**Status:** Accepted
**Author:** Homelab Architecture Team
**Stakeholders:** Homelab Operations, Service Owners (Nextcloud, Immich, Home Assistant)

---

## Problem Statement

The homelab server operates with a fixed 256GB SSD, while the infrastructure hosts several storage-intensive services:

- **Nextcloud** (file synchronization and sharing)
- **Immich** (photo management and library)
- **Home Assistant** (home automation and configuration)
- **PostgreSQL** (primary database service)
- **Docker containers and images** (numerous containerized applications)

The challenge is to balance the following constraints and requirements:

1. **Limited SSD capacity** (256GB total)
2. **Growing data volumes** from photos and media files
3. **Performance requirements** for the OS, Docker daemon, and database services
4. **High availability** for critical services
5. **Scalability** as the homelab expands
6. **Data persistence and reliability** independent of the homelab server

Without a deliberate storage strategy, the SSD would quickly fill with large binary data (photos, media files), degrading performance for critical services and leaving no room for system operation or growth.

---

## Decision

Implement a **tiered storage strategy** separating hot storage (SSD) and cold storage (NAS), with the following architecture:

### Hot Storage (SSD - 256GB)
The homelab server SSD is reserved for:
- Operating system (Ubuntu Server)
- Docker daemon and container images
- Docker volumes with stateful service data (databases, application state)
- System-level configurations and runtime directories

### Cold Storage (NAS)
The NAS (currently UGreen, planned migration to UniFi UNAS 4) is responsible for:
- Nextcloud data directory (user files and shares)
- Immich photo library (images, thumbnails, metadata)
- Future media files (Plex/Jellyfin libraries)

### Mount Strategy
NAS storage is mounted on the homelab server via **NFS (Network File System)** at the following paths:
- `/mnt/nas/homelab/cloudnext` — Nextcloud user files and shares
- `/mnt/nas/homelab/immich` — Immich photo library
- `/mnt/nas/homelab/media` — Reserved for future media services

Services access these mount points as regular filesystem paths, with container volumes bound to NFS mounts.

---

## Detailed Specifications

### SSD Allocation (256GB Total)

| Component | Size | Purpose |
|-----------|------|---------|
| OS (Ubuntu Server) | ~8 GB | System installation, kernel, boot |
| Docker images | ~20 GB | Cached container images (Nextcloud, Immich, PostgreSQL, etc.) |
| Docker volumes | ~15–20 GB | Stateful data (PostgreSQL, Authentik media, configs, app state) |
| Swap file | 8 GB | Memory overflow and OOM prevention |
| Miscellaneous (logs, caches, temp) | ~5–10 GB | System logs, container logs, temporary files |
| **Free headroom** | **~150 GB** | Buffer for system operation, updates, and growth |

**Rationale for headroom:** Modern filesystems degrade in performance below 10–15% free space. With 150GB free (59% capacity utilization), the system maintains adequate performance and space for growth without active management.

### NAS Storage Allocation

The NAS serves as the primary storage tier for large binary assets:

- **Nextcloud data directory:** User-uploaded files, shares, folder structures, and metadata
- **Immich library:** Photo and video files, extracted metadata, generated thumbnails
- **Media library:** Space reserved for Plex/Jellyfin media, organized by content type

The NAS should be sized based on expected data growth:
- **Nextcloud:** Typically 1–2TB for a household of 4–6 people with historical files
- **Immich:** Photo libraries grow over time; estimate 50–100GB per 10,000 high-resolution photos
- **Media:** Streaming libraries typically range from 500GB to 2TB for a home setup

### Docker Volume Strategy

#### Stateful Services (Named Volumes)
PostgreSQL, Authentik, and other stateful services use named Docker volumes stored on the SSD:

```dockerfile
services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data

  authentik:
    volumes:
      - authentik_media:/media

volumes:
  postgres_data:
  authentik_media:
```

These volumes are created and managed by Docker, stored in `/var/lib/docker/volumes/`.

#### Configuration Files (Bind Mounts)
Service configurations, docker-compose files, and non-volatile settings are stored in `/opt/stacks/<service>/`:

```dockerfile
services:
  nextcloud:
    volumes:
      - /opt/stacks/nextcloud/config:/var/www/html/config
```

This structure facilitates backup, version control, and migration of service configurations.

#### Large Data (NFS Mounts)
Nextcloud and Immich mount NAS storage via NFS, keeping large binary data off the SSD:

```dockerfile
services:
  nextcloud:
    volumes:
      - /mnt/nas/homelab/cloudnext:/var/www/html/data

  immich-server:
    volumes:
      - /mnt/nas/homelab/immich:/photos
```

---

## Why NFS over SMB?

The decision to use NFS (instead of SMB/CIFS) is based on the following factors:

### Performance
- **NFS** is optimized for Unix-like systems and uses more efficient networking protocols
- **SMB** incurs higher overhead on Linux and is designed primarily for Windows environments
- NFS is preferred for Linux-to-NAS traffic patterns

### Unix File Permissions
- **NFS** preserves and respects traditional Unix file permissions (rwx, ownership)
- **SMB** has weaker Unix permission support and requires workarounds (mapping, ACLs)
- Critical for **Nextcloud**, which relies on Unix permissions for access control
- Important for **Immich**, which manages file ownership for user data isolation

### Container Integration
- Docker volumes mounted via NFS properly respect user/group UID/GID
- Services running in containers (UID 1000, UID 33 for www-data) can properly access files
- SMB requires additional configuration to handle container UID mapping

### Use Case Alignment
- NFS is designed for LAN-local network storage (homelab environment)
- SMB is better suited for environments with Windows clients or WAN access
- This homelab is all-Linux, making NFS the natural choice

---

## Swap Configuration

The 8GB swap file is provisioned on the SSD to protect against out-of-memory conditions:

### Justification
- **RAM capacity:** 8GB is adequate for most services but can be insufficient during peak usage
- **Memory-intensive services:** Immich machine learning (ML inference), Authentik, and Nextcloud uploads can consume significant memory
- **Protection:** Swap prevents OOM (out-of-memory) kills, which would crash services and corrupt data
- **SSD performance:** Modern SSDs provide acceptable swap performance; not ideal but functional

### Configuration
Swap is configured with **low swappiness** to minimize SSD wear and prioritize RAM usage:

```bash
vm.swappiness=10
```

This kernel parameter causes the kernel to prefer freeing cache over using swap, reducing unnecessary disk I/O.

### Monitoring
Monitor swap usage via:
```bash
free -h
vmstat 1 5
```

If swap consistently exceeds 2–3GB, this indicates insufficient RAM and should trigger an upgrade.

---

## NAS Migration Path (UGreen → UniFi UNAS 4)

The architecture is designed to support NAS migration with minimal disruption:

### Migration Strategy

1. **Setup new NAS:**
   - Deploy UniFi UNAS 4
   - Configure NFS exports with identical paths (`/homelab`, `/homelab`, `/homelab`)
   - Ensure network connectivity and firewall rules

2. **Data migration:**
   - Mount both NAS devices simultaneously on the homelab server
   - Use `rsync` to copy data from old NAS to new NAS
   - Verify data integrity and permissions

   ```bash
   rsync -avz --delete /mnt/nas_old/ /mnt/nas_new/
   ```

3. **Update fstab:**
   - Modify `/etc/fstab` to point to the new NAS IP and exports
   - No changes required to Docker configurations if mount paths remain identical

4. **Service restart:**
   - Brief downtime to unmount old NAS and remount new NAS
   - Docker services automatically reconnect to NFS mounts upon restart

5. **Decommission old NAS:**
   - Verify all services operating normally on new NAS
   - Safely power down and remove old NAS hardware

### Benefits
- Services are decoupled from specific NAS hardware
- Migration can be performed with planned downtime (no unexpected failures)
- Same mount paths mean zero configuration changes in Docker Compose files

---

## Alternatives Considered

### 1. Store Everything on SSD
**Rejected** because:
- 256GB total capacity is insufficient for growing photo libraries and media files
- Nextcloud + Immich alone would consume 100–200GB within 12 months
- No capacity for system growth, updates, or temporary files
- SSD performance would degrade as utilization approaches 90%+

### 2. Attach USB External Drive
**Rejected** because:
- USB drives lack RAID redundancy and hot-swap capability
- Slower than NAS (USB 3.0 vs. Gigabit Ethernet)
- No native network access; would require NFS or SMB server
- Poor reliability for long-term data storage
- Hot-plugging introduces complexity and risk

### 3. Run NAS on Homelab Server (ZFS)
**Rejected** because:
- The N95 CPU/motherboard lacks ECC RAM, which is strongly recommended for ZFS
- ZFS is a fault-intolerant filesystem; non-ECC RAM can cause silent data corruption
- Running ZFS storage on the same server that runs critical services creates resource contention
- Dedicated NAS hardware provides better isolation, RAID, and reliability
- More appropriate to use dedicated appliance (UGreen, UniFi UNAS) designed for this purpose

### 4. Use SMB Instead of NFS
**Rejected** because:
- SMB has worse performance on Linux and in containerized environments
- Unix file permission handling is inadequate for Nextcloud and Immich
- Requires additional SMB-to-UID mapping configuration
- NFS is the Linux-native standard for network storage

---

## Consequences

### Positive Consequences

**Performance:**
- SSD remains unencumbered by large binary data, maintaining fast OS and database performance
- Swap remains available for emergency memory overflow
- Docker daemon operates with adequate disk space, preventing performance degradation

**Scalability:**
- NAS storage can be upgraded or expanded independently
- Homelab server SSD can be managed with predictable capacity (minimal growth)
- Services can grow with data volumes without impacting system stability

**Maintainability:**
- NAS hardware is replaceable without affecting homelab Docker configurations
- Configurations remain identical across NAS migrations
- Clear separation of concerns: compute (homelab) vs. storage (NAS)

**Reliability:**
- Nextcloud and Immich data persists independently of homelab server uptime
- NAS can be managed separately (backups, RAID, firmware updates)
- Services can be restored by recreating containers; data remains on NAS

### Negative Consequences

**Network Dependency:**
- If the NAS becomes unreachable (network outage, NAS failure), Nextcloud and Immich services become unavailable
- Requires robust network design and monitoring to detect connectivity issues
- Mitigation: redundant network connections, NAS heartbeat monitoring, alerts

**Setup Complexity:**
- NAS requires configuration of NFS exports and proper permission/ownership setup
- Homelab server requires `/etc/fstab` entries and NFS client configuration
- Initial setup is more complex than single-server storage
- Mitigation: document procedures, automate with Ansible or similar tools

**Brief Downtime for NAS Migration:**
- Migrating from UGreen to UniFi UNAS requires careful rsync and brief unavailability
- Not a zero-downtime operation
- Estimated downtime: 30–60 minutes depending on data volume
- Mitigation: schedule migration during maintenance windows, communicate with users

**Network Bandwidth Limitation:**
- Gigabit Ethernet has finite throughput (~125 MB/s in practice)
- Large Immich ML operations, Nextcloud syncs, and media transfers may be bandwidth-limited
- Not a problem for typical usage but noticeable during bulk operations
- Mitigation: consider 10Gbps networking if performance becomes critical (future upgrade)

---

## Implementation Notes

### NFS Mount Configuration

Add the following to `/etc/fstab` on the homelab server:

```fstab
# NAS mounts (adjust IP address and paths for your environment)
172.20.20.10:/volume1/homelab /mnt/nas/homelab/cloudnext nfs defaults,nofail,noatime 0 0
172.20.20.10:/volume1/homelab /mnt/nas/homelab/immich nfs defaults,nofail,noatime 0 0
172.20.20.10:/volume1/homelab /mnt/nas/homelab/media nfs defaults,nofail,noatime 0 0
```

**Mount options:**
- `defaults` — standard NFS options
- `nofail` — don't fail boot if NAS is unreachable (graceful degradation)
- `noatime` — disable access time updates (reduces I/O)

### NAS Export Configuration (UGreen example)

On the NAS, configure NFS exports:

```
/homelab -alldirs -mapall=nobody:nogroup 192.168.1.0/24
/homelab -alldirs -mapall=nobody:nogroup 192.168.1.0/24
/homelab -alldirs -mapall=nobody:nogroup 192.168.1.0/24
```

Adjust the NAS IP range and permissions based on your homelab network topology.

### Docker Compose Integration

Example Nextcloud service with NFS mount:

```yaml
services:
  nextcloud:
    image: nextcloud:latest
    container_name: nextcloud
    ports:
      - "80:80"
    volumes:
      - /opt/stacks/nextcloud/config:/var/www/html/config
      - /mnt/nas/homelab/cloudnext:/var/www/html/data
    environment:
      - NEXTCLOUD_ADMIN_USER=admin
      - NEXTCLOUD_ADMIN_PASSWORD=secure_password
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  postgres:
    image: postgres:15
    container_name: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=nextcloud
      - POSTGRES_PASSWORD=secure_password
    restart: unless-stopped

volumes:
  postgres_data:
```

---

## Monitoring and Maintenance

### Storage Monitoring

Monitor storage utilization on both tiers:

```bash
# SSD utilization
df -h /

# NAS utilization
df -h /mnt/nas/

# NAS connectivity
nfsstat -c

# NFS mount status
mount | grep nfs
```

### Health Checks

Implement monitoring and alerting:

- **SSD utilization:** Alert if > 80% (leaves 51GB free for buffer)
- **NAS connectivity:** Monitor NFS mount responsiveness; alert on timeouts
- **Docker volume health:** Monitor Docker volume usage; alert if > 85%

### Backup Strategy

- **NAS data:** Back up NAS to external storage or cloud (separate from homelab strategy)
- **Database backups:** Export PostgreSQL regularly to NAS; back up NAS separately
- **Docker configs:** Back up `/opt/stacks/` to version control (Git) and NAS

---

## Related Decisions

- **ADR-001: OS Selection** — Ubuntu Server as the host operating system
- **ADR-002: Containerization Strategy** — Docker as the container runtime (implications for volume management)
- **ADR-003: Database Strategy** — PostgreSQL as primary database (storage requirements inform this decision)

---

## Approval and Sign-off

- **Approved by:** Homelab Architecture Team
- **Approved date:** 2026-03-29
- **Review cycle:** Annual or upon significant infrastructure changes

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-29 | Architecture Team | Initial decision: separated hot/cold storage with NFS mounts |
