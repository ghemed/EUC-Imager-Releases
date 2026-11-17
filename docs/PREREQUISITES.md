# Prerequisites

What you need before installing EUC Imager: software, hardware, network access and the
accounts it uses. Pick one deployment method (Windows service or Docker) and one database.

> Sizing numbers below are recommended starting points, not benchmarks. EUC Imager has been
> exercised against small lab environments; validate against your own inventory size.

## 1. Supported platforms

| Component | Supported versions |
|---|---|
| VMware vCenter / ESXi | 7.0 and 8.0 |
| Omnissa Horizon (Connection Server) | 2603 and later (Horizon 7 and older are **not** supported) |
| Active Directory | Any supported Windows Server domain (for user sign-in and group mapping) |
| Browser (end users) | Current Chrome, Edge or Firefox |

## 2. Software

### Option A: Windows (installer / service)

| Requirement | Notes |
|---|---|
| Windows Server 2019 or later (Windows 10/11 for evaluation) | 64-bit |
| Administrator rights to install | The installer registers one Windows service (EUC Imager Backend), which serves HTTPS itself, and opens ports 443 and 80 in Windows Firewall |
| [ODBC Driver 18 for SQL Server](https://learn.microsoft.com/sql/connect/odbc/download-odbc-driver-for-sql-server) | **Only** if you choose Microsoft SQL Server as the database |
| TLS certificate for the Imager hostname | The service creates a self-signed one on first start; install yours in **Settings, Security, Server certificate** (PEM, PFX, or a signing request created there) |

### Option B: Docker

| Requirement | Notes |
|---|---|
| Docker Engine 24+ with Compose v2 (Linux) or Docker Desktop (Windows/macOS) | Windows needs WSL 2 |
| Internet access during `docker compose build` | Pulls the Node, Python and PostgreSQL images and the Microsoft ODBC driver |

The image already contains the SQL Server ODBC driver; PostgreSQL ships in the compose file.

### Option C: Run from source (development; see the source repository)

| Requirement | Version |
|---|---|
| Python | 3.11 or newer |
| Node.js | 20.19 or newer, or 22.12 or newer (22 is used in the Docker build and CI) |
| Git | any recent version |

### VMware HTML Console SDK (WebMKS), required for the in-browser console

The SDK is licensed by Broadcom and is **not** distributed with this repository. Download the
*VMware HTML Console SDK* from Broadcom and copy these two files before building the frontend
(or building the Docker image):

```
frontend/public/wmks/wmks.min.js
frontend/public/wmks/css/wmks-all.css
```

The files are git-ignored. After starting the app, use **Settings → vCenter → Test console**; its
first check ("WebMKS SDK present") tells you whether the files were picked up. Without the SDK
everything else works and the console falls back to VMRC / the vCenter UI.

### Database (choose one)

| Database | When to use | What you need |
|---|---|---|
| SQLite (default) | Lab, small installs, a handful of admins | Nothing. The file lives in the data folder |
| PostgreSQL 13+ (tested with 16) | Large inventories, multiple concurrent admins | A database and a user with rights to create tables in it |
| Microsoft SQL Server 2017+ (tested with 2022) | Customers standardised on SQL Server | A database and a login that can create tables; ODBC Driver 18 on the Imager host |

Schema creation and upgrades are automatic on startup (Alembic migrations). The Docker image was
verified against PostgreSQL 16 and SQL Server 2022 (Linux). SQL Server from a Windows host through
the Windows ODBC driver uses the same code path but has not been exercised yet.

## 3. Hardware / virtual machine sizing

| Deployment | vCPU | RAM | Disk | Database |
|---|---|---|---|---|
| Lab / evaluation (< 2,000 VMs) | 2 | 4 GB | 10 GB | SQLite |
| Standard (up to ~20,000 VMs) | 4 | 8 GB | 40 GB | PostgreSQL or SQL Server, 2 vCPU / 4 GB |
| Large (up to ~100,000 VMs) | 4–8 | 16 GB | 100 GB | PostgreSQL or SQL Server on a dedicated server (4+ vCPU / 16 GB, SSD) |

Large inventories are handled server-side: lists are paged and searched in the database, Horizon
machines and sessions are written in bulk, and the vCenter scan reads the inventory with the
PropertyCollector. In a lab test with 100,000 desktop VMs, 30,000 sessions and 100,000 log rows
(SQLite), a full re-sync of the rows took about 5 seconds and page, search and sort requests
answered in 0.01 to 0.15 seconds. Scanning a vCenter went from 12 to 24 seconds to 0.2 to 1.7 seconds
for about 400 VMs, with identical results. Treat these as indicative, not a guarantee, and use
PostgreSQL or SQL Server above a few thousand VMs.

Disk grows with the activity log and inventory. Keep the data folder (`instance/` or the Docker
volume `/data`) on persistent, backed-up storage: it holds `config.json`, the encryption key
(if you don't supply `AD_ENCRYPTION_KEY`) and the SQLite file.

## 4. Network requirements

### Ports

| From | To | Port / protocol | Purpose |
|---|---|---|---|
| User browser | EUC Imager | 443/TCP (80 only redirects to 443) | Web UI and the console WebSocket (`/api/console/ws/`). Plain HTTP is never served |
| EUC Imager | vCenter Server | 443/TCP | Inventory sync, power, snapshots, console tickets |
| EUC Imager | **Every ESXi host that runs a master VM** | 443/TCP | WebMKS console relay (the browser never talks to ESXi) |
| EUC Imager | Horizon Connection Server | 443/TCP | Pools, push image, health |
| EUC Imager | Domain controllers | 636/TCP (LDAPS, default) or 389/TCP (StartTLS / plain LDAP) | AD sign-in and user/group search |
| EUC Imager | Database server | 5432/TCP (PostgreSQL) or 1433/TCP (SQL Server) | Only for external databases |
| User workstation | vCenter and ESXi hosts | 443/TCP and 902/TCP | **Only** for the optional VMRC fallback (the desktop client connects directly) |

### Cross-vCenter image replication

**Replicate** on the Images page copies a powered-off master image to another vCenter. The Imager exports the VM
(OVF and disks) from the source and imports it into the target, so **all disk data flows through the Imager host**.
No network path between the two sites' vCenters or ESXi hosts is needed, and neither site needs to reach the other.

| From | To | Port / protocol | Purpose |
|---|---|---|---|
| EUC Imager | **Source** vCenter | 443/TCP | Export lease for the VM |
| EUC Imager | **Source ESXi host that runs the VM** | 443/TCP (902/TCP on some builds) | Streams the disks out. The host's certificate is checked against the thumbprint vCenter announces |
| EUC Imager | **Target** vCenter | 443/TCP | Import lease, placement lookup (datastores, pools, folders, networks) and the disk upload, under the vCenter's certificate settings |
| EUC Imager | **Target ESXi host** | 443/TCP (902/TCP on some builds) | Only if vCenter hands out a direct host address for the upload |

Plan for:

* **Bandwidth and time.** The disks are read from one site and written to the other through the Imager, so the
  transfer is limited by the Imager host's link to both sites. A 100 GB image takes about 15 minutes at
  1 Gbit/s sustained; use a host close to the sites or with a fast link to both.
* **Temporary disk space.** The disks are staged in `transfers/` inside the data folder (about the size of the VM's
  used disk space, checked before the copy starts) and removed afterwards, also when the copy fails.
* **Both vCenters configured** in the Imager, each with stored credentials, and **ESXi host names resolvable** from
  the Imager host.
* **Privileges.** Source account: *Virtual machine → Provisioning → Allow disk access / Allow read-only disk access*
  and *Virtual machine → Provisioning → Clone template* (export). Target account: *Virtual machine → Inventory →
  Create new*, *Virtual machine → Configuration → Add new disk*, *Resource → Assign virtual machine to resource pool*,
  *Datastore → Allocate space* and *vApp → Import* on the destination folder, pool and datastore.
* **Permission.** The user needs `image_manage` in the Imager.
* **The VM must be powered off.** It is copied as it is now, including all its disks, thin provisioned; snapshots
  are not part of the copy. The target gets a VM under the name you choose; that name must be unused there.
* **Time and certificates:** the same NTP and certificate rules as above apply to both sites.

Progress is shown in the dialog and the result is written to the Activity Log. The copy runs in the background, so
the dialog can be closed. Jobs are kept in memory: restarting the Imager during a copy stops it, and the target
may need a leftover VM removed by hand.

If you would rather use native vCenter features (Cross vCenter vMotion or a Content Library with subscription), the
data then moves host to host and these Imager paths are not used; the sites need ESXi-to-ESXi connectivity (TCP 8000
for vMotion and 443/902), which is outside the Imager's scope.

### Name resolution and time

* The Imager host must resolve the FQDNs of vCenter, **ESXi hosts**, Horizon Connection Servers
  and domain controllers (the console ticket returns the ESXi host *name*).
* Keep the Imager host, vCenter, ESXi and domain controllers on NTP. Console tickets are
  short-lived and TLS fails with large clock skew.

### Reverse proxies and load balancers

The Imager serves HTTPS itself and needs no proxy. If you put your own in front of it, point it at the Imager's
HTTPS port (`HTTPS_PORT`, default 8443 outside the Windows service; the proxy may skip verifying the certificate or
trust the one you installed), turn the redirect off (`HTTP_REDIRECT_PORT=0`), pass WebSocket upgrades for
`/api/console/ws/` and allow long-lived connections (1 hour or more). Run a single Imager instance: console sessions
are held in memory.

### Certificates

* **The Imager's own certificate (what browsers see):** created as self-signed on first start, so browsers warn.
  To replace it, open **Settings, Security, Server certificate** (needs `user_manage`) and either
  *Upload certificate* (PEM certificate + unencrypted or password-protected key, with the chain, or a PFX/P12 file) or
  *Create signing request* (a key is made on the server and never leaves it; send the request to your CA and install
  the answer with *Install signed certificate*). The certificate must list the name users type in the address bar
  (subject alternative name). A restart applies it (button in the same card; the Windows service and the container
  restart themselves). The files live in `<data folder>/tls` (`cert.pem`, `key.pem`): keep that folder
  protected and backed up. The Imager warns in the alert center 30 days before the certificate expires.
* **vCenter, Horizon, domain controllers:** certificate verification is configurable per connection
  (see [SECURITY.md](../SECURITY.md#transport-security)). To verify against a public or enterprise CA, the
  Imager host must trust that CA and resolve the server by a name listed in the certificate. For a
  self-signed or private-CA certificate, use **Fetch certificate** in the connection form.
* **ESXi (console):** the console relay pins the ESXi host certificate to the thumbprint vCenter announces.
* **LDAPS** needs a certificate on the domain controllers. Active Directory Certificate Services
  issues these automatically; without one, use StartTLS or a CA-issued certificate, or LDAP on an isolated network.

## 5. Accounts and privileges

### Service account in vCenter

Create a dedicated account (e.g. `svc-euc-imager@vsphere.local`) with a role containing:

| Privilege | Used for |
|---|---|
| Read-only on the inventory (propagate to the master-image folders **and to the folders that hold the Horizon desktop VMs**) | Discovering master VMs and snapshots; reading the uptime of desktop VMs |
| Virtual machine → Interaction → Console interaction | WebMKS / VMRC console |
| Virtual machine → Interaction → Power on / Power off | Power buttons |
| Virtual machine → Snapshot management → Create snapshot / Remove snapshot / Revert to snapshot | Snapshot manager |
| Virtual machine → Inventory → Remove | **Only** for *Delete image from disk*. Leave it out of the role if you do not use that feature |

Enter the account in **Settings → vCenter**. If the username has no domain, set the vCenter's
*Domain* field and the Imager appends it (`user@domain`).

### Service account in Horizon

An administrator account with a role that can view and edit desktop pools and schedule
*push image* operations, for example a role based on **Administrators** or a custom role with
those privileges, scoped to the pools you manage. Supply the AD domain with the account.
It also reads the connection server versions (dashboard agent check) and the Cloud Pod / entitlement
data. Pods that belong to one Cloud Pod Architecture should run the same connection server version.

### Active Directory

* A read-only service account that can bind and search the directory (user lookup, group search).
* AD groups (or users) that should be Imager **Admins**, **Operators** or **Viewers**.

### Roles inside EUC Imager

| Permission | Allows | Enforced by the server |
|---|---|---|
| `console_access` | Open VM consoles | yes |
| `snapshot_manage` | Create, revert and delete snapshots | yes |
| `pool_push` | Push images to pools (also allows syncing) | yes |
| `image_manage` | Sync master images and pools | yes |
| `image_delete` | Delete a master image from disk (Admin only by default) | yes |
| `vm_power` | Power master VMs on and off | yes |
| `env_manage` | Environments, vCenters, Connection Servers, certificates, Security tab | yes |
| `user_manage` | AD sign-in, users, roles, group mappings, system settings | yes |
| `user_view` | Read users and roles through the API | yes |
| `image_view`, `env_view` | Reserved: read-only screens are open to every signed-in user | no |
| `all` | Everything | yes |

Buttons and Settings pages a user may not use are hidden in the UI. See [SECURITY.md](../SECURITY.md#permissions).

## 6. Secrets you must provide

| Setting | How to create it |
|---|---|
| `SECRET_KEY` | A long random string (`python -c "import secrets; print(secrets.token_urlsafe(48))"`) |
| `AD_ENCRYPTION_KEY` | `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`. Stored vCenter/Horizon/AD passwords are encrypted with it. **Back it up**: without it they cannot be decrypted |
| `POSTGRES_PASSWORD` | Docker compose only |
