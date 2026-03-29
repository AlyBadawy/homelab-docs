# ADR-005 — NAS Folder Layout and Access Strategy

**Status:** Accepted
**Date:** 2026-03-29

---

## Context

The NAS (172.20.20.10) serves two distinct audiences:

1. **Human users** — two people who store and share personal files, accessing them directly via SMB from their computers
2. **Homelab services** — Nextcloud and Immich, which need dedicated backend storage for application-managed data

These two use cases have fundamentally different access patterns and permission requirements, and needed to be designed separately rather than mixed together.

Three decisions were made during the design discussion:

1. How to structure the homelab service storage on the NAS (one shared folder vs. several)
2. Whether homelab service data should live in a shared folder or a dedicated NAS user's personal folder
3. Whether the homelab service folders should also be accessible directly over SMB

---

## Decision 1 — One `homelab` shared folder with subfolders, not separate exports per service

**Decided:** Create a single shared folder named `homelab` on the NAS, containing subfolders `cloud`, `immich`, and `media`. Export the entire `homelab` folder as a single NFS share. Mount it once on the homelab at `/mnt/nas/homelab`.

**Alternatives considered:**

- **Separate shared folder per service** (`cloud`, `immich`, `media` each as top-level NAS shares) — three NFS exports, three fstab entries, three mount points. More granular control over per-service quotas and snapshots, but unnecessary overhead for a two-service homelab. Docker bind mounts already provide container-level isolation even within a single mount, so there is no meaningful security difference.

**Rationale:**

A single export with subfolders is simpler in every dimension — one NFS rule to configure on the NAS, one fstab entry to maintain, one systemd automount unit to monitor, one mount point to check in health scripts. The subfolders (`cloud/`, `immich/`, `media/`) provide clear organization without the operational overhead of separate shares. If per-service quotas or independent snapshot schedules become necessary in the future, this can be revisited and split without any changes to Docker or application configuration — only the fstab and NAS export rules would change.

**Consequences:**

- Pro: NAS admin has one NFS rule to manage instead of three
- Pro: One fstab entry, one systemd unit, one mount health check
- Pro: Clear grouping — everything under `homelab/` is service data, everything else is human data
- Con: Cannot apply independent NAS-level quotas per service without restructuring; mitigated by the fact that all growth is monitored at the application level (Nextcloud and Immich both have their own storage reporting)
- Con: A single I/O bottleneck at the mount level if services compete for NAS bandwidth; acceptable for a two-user homelab

---

## Decision 2 — Shared folder, not a homelab user's personal folder

**Decided:** The `homelab` folder is a NAS shared folder, not the personal/home folder of a dedicated NAS user account.

**Alternatives considered:**

- **Dedicated NAS user (`homelab`) with data in their personal folder** — create a NAS user called `homelab`, store all service data in their home directory (`/homes/homelab/`), export that path via NFS.

**Rationale:**

On NAS systems (UGreen, UniFi UNAS), personal/home folders are designed for human users accessing their own files interactively over SMB. They are managed by the NAS user system — their paths, permissions, and availability can be affected by user account changes (password resets, account suspension, software updates that alter home folder behavior). Using a personal folder as backend service storage couples application data to user account management in a way that can cause unexpected breakage.

Shared folders are the correct NAS abstraction for this purpose. They exist independently of any user account, have their own permission rules, can be assigned their own snapshot and backup policies, and are explicitly designed to be mounted and accessed by systems rather than people.

**Consequences:**

- Pro: `homelab/` is a first-class NAS object with its own lifecycle, independent of any user account
- Pro: No risk of service data being affected by user management operations
- Pro: Snapshots and backups can be targeted precisely at service data vs. user data
- Con: Requires creating a dedicated shared folder (minor — this is one click in any NAS admin UI)

---

## Decision 3 — NFS only, no SMB access on the `homelab` folder

**Decided:** The `homelab` shared folder is exported via NFS to `172.20.20.5` (homelab server) only. No SMB access is configured on this folder for any user.

**Alternatives considered:**

- **Dual access: NFS for services + SMB for direct user browsing** — allows users to open Finder or Explorer and browse the raw Nextcloud or Immich file trees directly on the NAS.

**Rationale:**

Both Nextcloud and Immich manage their storage directories internally. Immich in particular maintains a strict folder structure (`library/`, `thumbs/`, `encoded-video/`, `profile/`) and expects to be the sole writer to its upload directory. Direct SMB writes by a user — even accidental ones like a rename or a delete — can silently corrupt Immich's internal state or cause files to become orphaned (present on disk but not indexed in the database).

Nextcloud's data directory has a similar concern: files added outside of Nextcloud's own sync mechanism are not automatically indexed and will not appear in the app without manually running a file scan. This creates a confusing user experience where files appear to exist on the NAS but not in Nextcloud.

Users access their files through the applications — Nextcloud for documents and sync, Immich for photos. The NAS is backend storage, not a user-facing share for these folders.

**Consequences:**

- Pro: Application data integrity is guaranteed — no external writes to app-managed directories
- Pro: No SMB/NFS permission conflicts to manage
- Pro: Cleaner mental model — users know they use the app, not the NAS, for their photos and synced files
- Con: Users cannot access files directly if an application is down; mitigated by the fact that the NAS personal and shared SMB folders remain fully accessible independently of the homelab services

---

## Final NAS Layout

```
NAS (172.20.20.10)
│
├── homelab/              ← Shared folder — NFS only, restricted to 172.20.20.5
│   ├── cloud/            ← Nextcloud user data (written by www-data, UID 33)
│   ├── immich/           ← Immich photo/video library (written by node, UID 1000)
│   └── media/            ← General media — future use
│
├── [Personal: Aly]       ← SMB personal folder — Aly only
├── [Personal: User 2]    ← SMB personal folder — User 2 only
└── [Shared folder]       ← SMB shared folder — both users
```

NFS export (single entry on NAS):
```
/homelab   172.20.20.5(rw,sync,no_subtree_check,no_root_squash)
```

Homelab mount (single fstab entry):
```
172.20.20.10:/volume1/homelab  /mnt/nas/homelab  nfs4  _netdev,nofail,x-systemd.automount,...  0 0
```

Docker bind mounts reference subfolders within the single mount:
```
/mnt/nas/homelab/cloudnext   → Nextcloud container data directory
/mnt/nas/homelab/immich  → Immich container upload directory
/mnt/nas/homelab/media   → (future)
```
