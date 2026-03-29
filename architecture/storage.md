# Storage Architecture

## Design Philosophy

Storage is divided into two tiers based on access pattern and data class:

| Tier | Device | Used For | Mount |
|------|--------|----------|-------|
| **Hot** | Homelab SSD (256 GB) | OS, Docker images, databases, app state | `/` (local) |
| **Cold/Bulk** | NAS (172.20.20.10) | User files, photo library, media | `/mnt/nas/*` (NFS) |

The homelab SSD is intentionally kept free of user data. Everything a user creates or uploads lives on the NAS. This means:

- The homelab can be fully wiped and rebuilt without losing a single file
- NAS hardware can be swapped (UGreen → UniFi UNAS 4) by updating the fstab entry and remounting — no Docker config changes needed
- Backups are simpler: back up the NAS, not the homelab

---

## NAS Export Layout

The NAS exposes a single NFS share (`/homelab`) containing service-specific subfolders, plus independent SMB shares for personal user data:

```
NAS (172.20.20.10)
├── homelab/              ← single NFS export, restricted to 172.20.20.5, NFS only
│   ├── cloud/            → Nextcloud user data
│   ├── immich/           → Immich photo/video library
│   └── media/            → General media (future use)
│
├── [Personal: Aly]       ← SMB only (independent of homelab services)
├── [Personal: User 2]    ← SMB only
└── [Shared folder]       ← SMB shared between users
```

This is mounted on the homelab at:

```
/mnt/nas/homelab/        ← single NFS mount
├── cloud/               ← bind-mounted into Nextcloud container
├── immich/              ← bind-mounted into Immich container
└── media/               ← available for future media services
```

---

## Service → Storage Mapping

### Nextcloud (`cloud.in.alybadawy.com`)

| Data Type | Location | Notes |
|-----------|----------|-------|
| User files | `/mnt/nas/homelab/cloudnext/` (NAS) | All documents, photos synced by users |
| App config & PHP state | Docker volume `nextcloud_config` (SSD) | Themes, installed apps, settings |
| Database | PostgreSQL on SSD | Metadata, shares, users |
| Session cache | Redis on SSD | Ephemeral, rebuilt on restart |

Docker bind mount:
```yaml
volumes:
  - /mnt/nas/homelab/cloudnext:/var/www/html/data   # NAS
  - nextcloud_config:/var/www/html              # SSD (config/code)
```

### Immich (`photo.in.alybadawy.com`)

| Data Type | Location | Notes |
|-----------|----------|-------|
| Original photos/videos | `/mnt/nas/homelab/immich/library/` (NAS) | Source files, never modified by Immich |
| Thumbnails & transcodes | `/mnt/nas/homelab/immich/thumbs/` (NAS) | Regeneratable, but large — keep on NAS |
| ML model cache | Docker volume `immich_model_cache` (SSD) | CLIP, face recognition models (~2–4 GB) |
| Database | Immich PostgreSQL (SSD) | Metadata, face tags, albums |

Docker bind mount:
```yaml
volumes:
  - /mnt/nas/homelab/immich:/usr/src/app/upload  # NAS (library + thumbs)
```

> Keeping thumbnails on the NAS avoids regenerating them after a rebuild. Regeneration can take hours for large libraries.

---

## NFS Mount Strategy

### Why NFS over SMB

| Criterion | NFS | SMB |
|-----------|-----|-----|
| Linux Docker container compatibility | Native | Requires extra packages |
| Unix file permissions (UID/GID) | First-class | Mapped (lossy) |
| Performance on LAN | Slightly faster | Slightly slower |
| Protocol overhead | Lower | Higher |
| Best suited for | Linux-to-Linux | Windows clients |

NFS is the right choice for Linux containers talking to a NAS on the same VLAN.

### Single Mount vs. Multiple Mounts

The `/homelab` NFS export is mounted once at `/mnt/nas/homelab` on the homelab. Docker containers then use **bind mounts** to access specific subfolders:

```bash
# Single NFS mount
172.20.20.10:/volume1/homelab  →  /mnt/nas/homelab  (mounted once in fstab)

# Docker isolation via bind mounts (not separate NFS mounts)
/mnt/nas/homelab/cloudnext  →  /var/www/html/data (Nextcloud container)
/mnt/nas/homelab/immich →  /usr/src/app/upload (Immich container)
```

This simplification reduces complexity: one NFS mount, one network operation, one entry in fstab. The NAS export itself (`/homelab`) is restricted to a single IP (`172.20.20.5`), making multi-service isolation a NAS-level concern, not a mounting concern.

### NFS Version

Use **NFSv4** (`nfs4` in fstab). It offers:
- Stateful connections (better error handling)
- Better security (no need to expose `rpcbind` on the NAS)
- Single TCP port (2049) — simpler firewall rules
- Stronger file locking semantics (important for Nextcloud)

---

## Access Model

The NAS storage is split into two completely separate access patterns:

### Service Data (`/homelab` — NFS only)

- **Protocol:** NFS only (no SMB), mounted at `/mnt/nas/homelab` on the homelab
- **Users:** Nextcloud and Immich containers only
- **Access:** Containers access data via bind mounts to subfolders
- **Human access:** Users access their files through the Nextcloud and Immich web apps, never by browsing the NAS directly

Example:
```
User uploads a photo via Immich web app
  → Immich container writes to /mnt/nas/homelab/immich/
  → NAS stores it in /homelab/immich/
  → User never sees the NAS share directly
```

### Personal & Shared Data (SMB only)

- **Protocol:** SMB only (completely separate from the `homelab` NFS export)
- **Users:** Human users (Aly, User 2, etc.)
- **Access:** Users browse folders on their computers using SMB (Windows File Explorer, macOS Finder, Linux Nautilus)
- **Isolation:** Independent of homelab services — if Nextcloud or Immich is down, personal SMB shares remain accessible

Example:
```
Aly accesses [Personal: Aly] share via SMB
  → Browse files as if they were on a network drive
  → Completely separate from the homelab NFS export
```

This separation ensures:
- Service failures don't block user access to personal files
- No confusion between app-managed data and user-managed data
- NFS (Linux-optimal) for services, SMB (user-friendly) for personal access

---

## Resilience Design

The NAS is a dependency of two services (Nextcloud, Immich). If the NAS goes offline, those services lose access to user data. The goal is to make this failure mode **graceful** — not catastrophic.

### Boot Sequence (NAS Available)

```
1. Ubuntu boots
2. systemd brings up network (network-online.target)
3. systemd registers NFS automount units (instantly ready, not yet mounted)
4. Docker daemon starts
5. Nextcloud container starts
6. Container first accesses /mnt/nas/homelab → automount triggers NFS mount
7. NFS mount succeeds → Nextcloud serves files normally
```

### Boot Sequence (NAS Unavailable)

```
1. Ubuntu boots
2. systemd brings up network
3. systemd registers NFS automount units (instantly ready)
4. Docker daemon starts  ← no hang, no failure
5. Nextcloud container starts  ← no hang, no failure
6. Container first accesses /mnt/nas/homelab → automount triggers NFS mount
7. NFS mount attempt times out (30 sec) → returns I/O error
8. Nextcloud shows 503/error for file operations
9. Auth, settings, and non-file features still work
```

### Recovery When NAS Comes Back

```bash
# Re-trigger the automount (no reboot needed)
sudo systemctl restart mnt-nas-homelab.automount

# Or simply touch the mount point to trigger automount
ls /mnt/nas/homelab

# Then restart affected containers
docker restart nextcloud immich-server
```

### Mount Options Explained

| Option | Effect | Why Used |
|--------|--------|----------|
| `nfs4` | Use NFSv4 protocol | Better performance and security |
| `_netdev` | Mark as network device | Tells systemd to wait for network before mounting |
| `nofail` | Don't fail boot if mount unavailable | Homelab continues booting if NAS is down |
| `x-systemd.automount` | Lazy mount — only connect on first access | Docker starts immediately; no boot hang |
| `x-systemd.mount-timeout=30` | Give up mounting after 30 seconds | Prevents indefinite hang if NAS unreachable |
| `soft` | NFS ops return error on timeout (don't block forever) | Containers get I/O errors, not infinite hangs |
| `timeo=100` | 10-second NFS operation timeout (units: 0.1s) | Quick failure detection |
| `retrans=3` | Retry failed NFS ops 3 times before giving up | Tolerates brief NAS hiccups |
| `rsize=131072,wsize=131072` | 128 KB read/write block size | Optimized for 2.5 GbE throughput |
| `noatime` | Skip updating file access timestamps | Reduces write amplification on NAS drives |

---

## File Ownership & Permissions

Docker containers access NFS shares as specific Unix UIDs. The NAS must allow those UIDs to write.

| Service | Container UID | NAS Directory | Required Permission |
|---------|--------------|---------------|---------------------|
| Nextcloud | `www-data` (UID 33) | `/homelab/cloudnext` | UID 33 owns the directory, `rwx` |
| Immich | `node` (UID 1000) | `/homelab/immich` | UID 1000 owns the directory, `rwx` |

On the NAS, before first use:
```bash
# Run on the NAS (via SSH or NAS terminal)
chown -R 33:33 /homelab/cloudnext        # www-data for Nextcloud
chown -R 1000:1000 /homelab/immich   # node for Immich
chmod -R 755 /homelab/cloudnext /homelab/immich
```

Alternatively, configure the NAS NFS export with `all_squash,anonuid=1000,anongid=1000` and run both containers as UID 1000 — but this requires overriding the Nextcloud container's UID, which adds complexity.

> **Simplest homelab option:** Configure the NAS `/homelab` export with `no_root_squash` (access is restricted to `172.20.20.5` anyway) and let the containers handle their own permission needs. Document this as a deliberate trade-off on a private, VLAN-isolated network.

---

## SSD Storage Budget

With NAS handling all user data, the local SSD footprint is predictable:

| Category | Estimated Size |
|----------|---------------|
| Ubuntu Server | ~5 GB |
| Docker images (all services) | ~15–25 GB |
| Docker volumes (databases, app state) | ~10–15 GB |
| Immich ML model cache | ~3–5 GB |
| Swap file | 8 GB |
| Logs + misc | ~2 GB |
| **Total used** | **~43–60 GB** |
| **Remaining headroom** | **~196–213 GB** |

The SSD will not be a bottleneck. All growth goes to the NAS.

---

## NAS Migration Path (UGreen → UniFi UNAS 4)

When replacing the NAS hardware, no Docker or application configuration changes are needed if the same IP (`172.20.20.10`) and export path (`/homelab`) are preserved.

Migration procedure:
```
1. Set up UniFi UNAS 4 with the same static IP (172.20.20.10)
2. Create the same NFS export (/homelab) with subfolders (cloud/, immich/, media/)
3. rsync data from old NAS to new NAS (see rebuild/02-nas-mounts.md)
4. Update /etc/fstab on homelab only if paths changed
5. Run: sudo umount /mnt/nas/homelab && sudo mount -a
6. Verify: ls /mnt/nas/homelab
7. Decommission old NAS
```
