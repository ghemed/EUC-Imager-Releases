# Changelog

Notable changes, newest first. Versions follow `build_package/version.txt`.

## 1.1.0 (2026-11-17)

**HTTPS**
* The Imager serves HTTPS itself; nginx is no longer part of the Windows installer or the Docker setup. Port 80 only redirects to HTTPS.
* Install your own certificate in Settings, Security, Server certificate: upload PEM or PFX, or create a signing request on the server and install the answer. A restart (button in the same card) applies it.
* Session cookies are Secure when HTTPS is served; HSTS is sent with a certificate that is not self-signed; TLS 1.2 or newer.
* Alerts for an expired, soon-to-expire or self-signed Imager certificate.
* Upgrade from an nginx-based install: the installer keeps the existing certificate, removes the nginx service and opens ports 443 and 80 in Windows Firewall.
* New settings: `HTTPS_PORT`, `HTTP_REDIRECT_PORT`, `HTTPS_PUBLIC_PORT`; `PORT` is no longer used by the production server.

**Dialogs and tables**
* Master Images: the snapshot count ("N versions") opens the snapshot window; the separate Manage Snapshots button is gone.
* Esc closes the pop-up window on top, in every dialog (and an open menu); it does not interrupt a running sync.
* The Cancel button of the first step of the Push Image wizard now closes it.
* Table headings that stay in view while a list scrolls are opaque, so rows no longer show through them.
* Dialogs cover the whole window: a layout rule used to push them 24 px down and leave an undimmed strip above.

**Dashboard**
* Automatic sync: data older than the configured number of hours (default 3) is refreshed by the Imager itself, with the interval in Settings, System. The old setting that did nothing is replaced by this.
* Status column shows the real state (Disconnected is no longer counted as an error; faults show Horizon's reason); the user of a disconnected VM is shown.
* Every table can be resized by dragging a column edge; widths are remembered.
* Sessions saved by earlier versions without a pod are removed on the next sync of their pod.
* Lists: VM name first, Environment / Pod column, Export on every list including sessions, an Error column with Horizon's message, and uptime read from vCenter.
* Sessions that Horizon no longer reports are removed on every sync (per pod); the old data some installs showed is cleaned up.
* Outdated agent versions list scrolls.
* Optional auto refresh (Settings, System).
* Sync button beside the last sync time; a failed sync shows its reason.
* Pool Utilization rebuilt as readable per-pool bars; clicking a pool opens its VM inventory.
* Agent versions are compared per pod; pods running different connection server versions are flagged.
* The "VMs in error state" alert opens the error list.
* Alert center (bell) with per-user unread state, linked to where each issue is fixed.
* Global View adds the environment name in brackets to pool names.

**Pools and images**
* Cross-vCenter replication through the Imager (export, import, progress); the Replicate button now does something.
* Pools list: paging with a page size, and a search box that works.
* Activity log: filter by status and between two dates and times.
* Pool window compares the pools of an entitlement when image drift is detected; header badges have hover hints.
* Image push fixed and validated: correct parent VM, snapshot checked against the app's data, Horizon's error shown,
  last push time recorded.
* Snapshot manager: create, revert and delete snapshots with in-app dialogs.
* Delete a master image from disk: Admin only, two confirmations.

**Cleanup**
* Removed API routes and service methods nothing used (including two developer debug endpoints) and unused imports and helpers.

**Security and administration**
* Per-connection certificate verification with Fetch / Trust, Security tab, AD over LDAPS or StartTLS.
* Permissions are enforced by the server and hide the matching buttons and Settings pages: `console_access`,
  `snapshot_manage`, `pool_push`, `image_manage`, `image_delete`, `vm_power`, `env_manage`, `user_manage` and
  `user_view`. The read-only screens stay open to every signed-in user.
* Orphaned connections can no longer be left behind when an environment dialog is closed or a connection is removed.
* Unsaved-changes tracking; connection test errors next to the field.
* First-run setup wizard, console (WebMKS) proxy, Windows installer, server-side paging.
