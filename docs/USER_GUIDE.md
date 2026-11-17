# User guide

How each screen of Horizon Imager works. Set-up, ports and accounts are in
[PREREQUISITES.md](PREREQUISITES.md); what each role may do is in the
[README](../README.md#roles-and-permissions).

Buttons and Settings pages you have no permission for are not shown (for example no **Sync**, **Power**
or **Settings** without the matching permission). If something you expect is missing, ask an administrator
which role you have. The server refuses the action as well.

Press **Esc** to close the pop-up window on top (a dialog that is busy, such as a running sync, stays open).

## Environments and the Global View

An **environment** groups the vCenters and Horizon pods that belong together. The selector at the top
of the sidebar switches between them. **Global View** shows everything at once; there, pool names carry
their environment in brackets, for example `Windows 10 (Lab)`, so equal names can be told apart.

## Dashboard

* **Last sync / Sync**: the pill shows when data was last refreshed. The small **Sync** button next to
  it reads master images from vCenter and then pools, VMs and sessions from Horizon. A problem (for
  example missing credentials) appears in red next to it.
* **Old data refreshes itself**: when master images or pools were last synced longer ago than the interval (3 hours by
  default, **Settings, System, Refresh the data automatically when it gets old**), the Imager syncs master images from
  vCenter and pools, machines and sessions from Horizon by itself. It checks every 10 minutes, also when nobody has the
  dashboard open; opening the dashboard with old data starts it at once. The dashboard shows "Refreshing old data..."
  while it runs, and "Data is N h old" or "Last sync failed: ..." when it could not. A failed sync is retried after 30
  minutes. Only connections with stored credentials are synced.
* **Auto refresh**: **Settings, System, Dashboard auto refresh** reloads the dashboard from the Imager's stored data
  every 15 seconds to 5 minutes while the page is open (off by default). It does not sync from vCenter or Horizon.
* **Cards**: active sessions, VMs, pools, master images, pods, health score and outdated agent versions.
  Click a card for the list behind it.
* **VM and session lists** (Active Sessions, Total VMs, Available VMs, the error list): VM name comes first, then the
  desktop pool. Unless you narrow the list to one pod, an **Environment / Pod** column says where each row lives.
  **Uptime** is read from vCenter after every pool sync (the uptime VMware Tools reports, else the power-on time) and
  counts up live in the list. The **Status** column shows the real state: Connected, Available, **Disconnected** (a user
  left the VM; not a fault), Maintenance, or for a fault Horizon's reason in plain words, for example *Agent unreachable*.
  Only faults count as "VMs in error state". The **User** column shows who is on a VM, also when they are disconnected.
  Every list has **Export** (CSV of all matching rows).
* **Column widths**: drag the right edge of any table heading to widen or narrow a column, in every list. The widths are
  remembered in your browser; double-click the edge to go back to automatic widths.
* **Pool Utilization**: one bar per pool. Blue is connected, green available, red error, grey other
  (provisioning, maintenance, disconnected). Hover a bar for the numbers; click a pool to open all of its
  VMs with a status filter, search and CSV export. Sort by most users, busiest, errors first or name.
* **Outdated agent versions**: each VM is compared with the connection server version of **its own pod**.
  If pods that should match (a Cloud Pod Architecture) run different versions, a warning lists each pod's
  version and the card shows "Pods differ".
* **Recent Alerts**: VM and pool alerts. Clicking "VMs in error state" opens the error list.
* **Bell (alert center)**: conditions that need an administrator, such as connections without certificate
  checks or failed syncs. The badge counts unread alerts; clicking an alert opens the place to fix it.
  Alerts you have fixed disappear by themselves.

## Desktop Pools

The list shows each pool with its entitlement, type, availability, master image and snapshot, sessions
and status. The status pill is *Available*, *Warning* (VMs in error), *Image drift* or *Pending push*;
the second icon shows a running task.

### Pool window

Click a pool. The header shows its status, **Image drift**, Global Entitlement, ID and **Last Push**
(hover each for what it means).

* **Image drift** means pools of one Global Entitlement do not all run the same master image and
  snapshot. The window then shows a table of every pool in the entitlement with the differing master
  image or snapshot highlighted. To fix it, push the same snapshot to all of them.
* The four tiles open the pool's VMs filtered by total, available, connected or error.
* **Current base image** shows the snapshot the pool runs. If only an ID appears, the name could not
  be resolved: sync the master images and the pools, and check the vCenter folder filter includes the VM.
* **Last Push** is recorded when you push from this app and, when Horizon reports one, from Horizon.

### Push image

Select pools, then **Push Image**. Pick the master image and snapshot (one for all pools or one per pool),
a schedule, and confirm. Every pool is checked first (the snapshot must exist on the chosen image);
Horizon's own error text is shown if it refuses. By default the push waits for users to log off.

## Master Images

* The list is paged, searchable and sortable; **Refresh** reloads and refreshes power state.
* **Console** opens the VM in a browser tab. If it fails, the dialog names the layer that failed and
  offers the VMRC app or the vCenter UI.
* **Snapshot** takes a snapshot (name and optional description).
* Click the **N versions** link of an image to open its snapshot tree. Per snapshot: **Deploy** (push to pools), **Revert** (the
  VM returns to that snapshot; changes since then are lost) and **Delete** (optionally with the snapshots
  below it; refused while a pool runs from it).
* **Replicate** (needs `image_manage`): pick the target vCenter, then a datastore, a cluster or resource pool and
  optionally a folder, network and new name. The VM must be powered off. The Imager exports the VM and imports it
  into the target, showing progress; you can close the dialog and the copy continues. The result is in the Activity
  Log and the new image appears in the target's list after a sync (the Imager starts one when the copy ends).
* **Delete from disk** (red trash icon, administrators): deletes the VM and its files from the datastore.
  Step 1 explains the effect; step 2 asks you to type the image name. It needs the VM powered off and not
  used by a pool, and cannot be undone.

## Settings

* **Environments**: add vCenters and Connection Servers. Saving a connection and saving the environment are
  separate; closing the dialog without saving discards connections you created in it, and removing a
  connection from an environment deletes it (with what was synced from it).
* **Certificates**: each connection shows whether its certificate is checked. **Fetch certificate** shows
  the subject, issuer, validity and SHA-256 fingerprint; compare it with the server's, then **Trust this
  certificate**. **Remove** stops checking.
* **Security**: the **Server certificate (HTTPS)** card shows the certificate browsers get (subject, issuer, names,
  validity, fingerprint, days left) and lets an administrator replace it: **Upload certificate** (PEM with key, or a
  PFX/P12 file with its password), **Create signing request** (the key stays on the server; send the request to your
  certificate authority, then **Install signed certificate**) or **New self-signed**. A new certificate is used after
  **Restart now**; the page reloads by itself when the server is back. Below the card, the list of connections that
  are not protected, each with a **Fix** button that opens that connection (verified ones are behind a toggle).
* **Active Directory**, **Roles**, **Users** and **System**: sign-in, permissions and defaults. A dot on a
  section means unsaved changes.
* **Test console** (on a vCenter): checks the WebMKS files, the vCenter login, a powered-on VM and the
  console ticket, and names the first step that fails.

## Activity Log

Every sign-in, sync, snapshot, push, console and delete action with user, target, status and details.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Console says "InvalidLogin" | The vCenter record the VM belongs to has a wrong user name or password. Check **Settings, Environments** |
| Pool shows a snapshot ID instead of a name | Horizon reports a snapshot the app has not synced, or the VM is outside the vCenter folder filter. Sync both and check the filter |
| "The server answered with a web page instead of data" | The backend is older than the frontend. Restart it |
| Pool images missing after a sync | The vCenter folder or tag filter excludes them |
| Sync button shows a red problem | The message names the connection; open it in Settings and run its test |
| Browser warns "connection not private" | The Imager still uses its self-signed certificate. Install one from your authority in **Settings, Security** |
| After installing a certificate the browser still shows the old one | It is used after a restart: click **Restart now** in the Server certificate card |
| Port 443 or 80 is in use, service does not start | Another program (an old nginx, IIS) holds the port. Stop it; the log names the port |
