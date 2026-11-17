# Security

## Reporting a vulnerability

Please do not open a public issue for security problems. Report them privately through GitHub's
*Report a vulnerability* feature (Security tab) of the source repository, TerasGuy/EUC-Imager.

## Handling of secrets

* vCenter, Horizon and AD passwords are stored encrypted (Fernet) using `AD_ENCRYPTION_KEY`
  (or an auto-generated `secret.key` in the data folder). Back the key up; rotating it makes
  stored passwords unreadable and they must be re-entered.
* Local user passwords are stored as salted hashes. Older plain-text values are hashed
  automatically on first start.
* `.env`, `config.json`, databases and `secret.key` hold secrets. Keep them out of version control and backups you do not trust.

## Console security

* Console sessions are single-use, expire after 60 seconds and are bound to the signed-in user.
* Opening a console requires the `console_access` permission and is written to the activity log.
* The Imager pins the ESXi host certificate to the thumbprint announced by vCenter before it
  relays traffic.

## Transport security

**The Imager itself** serves HTTPS only; plain HTTP is never offered to the application (an optional port 80
only redirects to HTTPS). TLS 1.2 or newer. On first start it creates a self-signed certificate; replace it in
**Settings, Security, Server certificate** with one from your own authority (PEM or PFX upload, or a signing
request created on the server so that the private key never leaves it). Rules the Imager follows:

* The private key is stored in `<data folder>/tls/key.pem` (readable only by administrators and the service when
  the installer set the folder permissions; keep the folder protected and out of backups you do not trust). It is
  never returned by any API or written to the log or the Activity Log. A signing request's key is kept as
  `pending.key` until the signed certificate is installed.
* Session cookies carry the `Secure` flag whenever the server serves HTTPS. `Strict-Transport-Security` is sent
  only with a certificate that is not self-signed, because HSTS on a self-signed certificate would lock browsers out
  when it changes.
* Installing or replacing the certificate needs the `user_manage` permission, is written to the Activity Log with
  the subject and fingerprint, and takes effect after a restart.
* The alert center warns when the certificate is expired, expires within 30 days, or is self-signed.
* Upgrading from a version that put nginx in front: the installer keeps the certificate from nginx's folder, removes
  the nginx service and serves HTTPS from the app on the same ports (443, 80 redirect-only).

**Connections from the Imager** to other systems:

Every vCenter, Horizon Connection Server and the Active Directory connection has its own
certificate setting (Settings, and the first-run wizard):

* **Verify the server certificate** (default for new connections). The certificate must chain to a CA
  the Imager host trusts (on Windows: the Windows certificate store), or you trust one specific
  certificate yourself with **Fetch certificate -> Trust this certificate**, which shows the subject,
  issuer, validity and SHA-256 fingerprint first. Compare the fingerprint with the server's own.
* **Do not verify.** The connection stays encrypted but the server's identity is not checked, so a
  machine in the middle could impersonate it. Existing connections keep this behaviour when you
  upgrade so nothing stops working; open each one and turn verification on.
* **Active Directory** supports LDAPS (636), StartTLS (389) and plain LDAP. New setups default to
  LDAPS with verification. Plain LDAP sends the service account password and every user's password
  unencrypted and should only be used on an isolated network.

## Permissions

The server checks these permissions on every request, and the UI hides the matching buttons and
Settings pages:

* `console_access`: open a VM console.
* `snapshot_manage`: create, revert to and delete snapshots.
* `pool_push`: push an image to pools. Each push is validated against the app's own data (the snapshot
  must exist on the selected master image) before anything is sent to Horizon.
* `env_manage`: add, change and delete environments, vCenters and Connection Servers, test connections,
  fetch certificates and see the Security tab. `user_manage`: Active Directory settings, users, roles, group
  mappings and system settings (`user_view` allows reading users and roles through the API only).
* `vm_power`: power master VMs on and off. `image_manage`, `env_manage` or `pool_push`: sync from vCenter
  and Horizon.
* `image_delete`: delete a master image and its files from the datastore. Only the Admin role has it
  by default. It needs two confirmations (read the warning, then type the image name; the server checks
  the name again), is refused while the VM is powered on or a pool uses it, and is written to the
  activity log whether it succeeds or fails.

Every state-changing action is written to the activity log with the user name.

## Alert center and Security tab

**Settings, Security** lists every vCenter, Horizon and Active Directory connection that runs
without certificate verification or without encryption, with a **Fix** button that opens that
connection. The bell on the dashboard shows the same issues (and failed syncs) as unread alerts per
user; an alert disappears when its condition is resolved.

## Known limitations

* Read-only screens (dashboard, pools, images, VM lists, sessions, activity log, connection lists) are open to
  every signed-in user; `image_view` and `env_view` do not restrict them. Connection lists never contain
  passwords, but they do show server names and certificate settings.

* Run a single instance; console sessions are kept in memory.
