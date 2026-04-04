# Step 2 — Active Directory (Samba 4)

**Must complete before:** NAS Mounts (03), Docker (04), and all application services.

Samba 4 Active Directory is a **bare-metal** service — it runs directly on the host OS, not in Docker. It must be provisioned and verified before anything else is installed, because application services authenticate against it on first run.

---

## What This Step Covers

By the end of this step you will have:

- A fully provisioned Samba 4 AD domain (`ID.IN.ALYBADAWY.COM`)
- Samba's internal DNS server running on port 53, answering queries for the `id.in.alybadawy.com` zone
- Kerberos working and verified (valid TGT obtainable)
- The UDR7 configured with a DNS forward zone for `id.in.alybadawy.com` → `172.20.20.5`
- An initial set of users and groups ready for services to consume

---

## Guides

Follow these two guides in order:

1. [01 — DC Setup & Provisioning](active-directory/01-samba-ad-dc-setup.md)
   Installs Samba, provisions the domain, configures Kerberos, starts the `samba-ad-dc` service, and verifies DNS and Kerberos are working.

2. [02 — User & Group Management](active-directory/02-samba-ad-user-group-management.md)
   Reference cheatsheet for creating and managing users and groups via `samba-tool`. Use this to set up the initial user accounts and service groups before connecting applications.

---

## Key Details

| Setting       | Value                          |
| ------------- | ------------------------------ |
| Realm         | `ID.IN.ALYBADAWY.COM`          |
| Domain        | `id.in.alybadawy.com`          |
| NetBIOS name  | `ID`                           |
| DC hostname   | `dc.id.in.alybadawy.com`       |
| DC IP         | `172.20.20.5`                  |
| DNS backend   | Samba Internal (port 53)       |
| DNS forwarder | `172.20.20.1` (UDR7)           |
| LDAP          | Port 389 — LAN accessible      |
| LDAPS         | Port 636 — LAN accessible      |
| Kerberos      | Port 88 — LAN accessible       |

---

## UDR7 DNS Forward Zone

After Samba is running, configure a DNS forward zone in UniFi so all VLANs can resolve AD records:

- **UniFi Network → Settings → Networks → DNS → DNS Forwarding**
- Zone: `id.in.alybadawy.com`
- Forward to: `172.20.20.5`

This allows any client (or Docker container) to resolve `dc.id.in.alybadawy.com`, Kerberos SRV records, and LDAP SRV records through UDR7, which delegates to Samba.

---

## Next Step

Once AD is provisioned and verified, proceed to **[03 — NAS Mounts](03-nas-mounting.md)**.
