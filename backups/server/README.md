# Server backups

Bash, Borg 1.2, rsync, and systemd configuration for `/data` -> `/data_sync`.
Borg stores `/etc`, `/usr/local`, and `/var` in `/data/server`. Automatic pruning
is disabled. Rsync preserves modification times and symbolic links without
preserving POSIX ownership/permissions or deleting destination-only files.

## Installation

Review the expected mount UUIDs and filesystem types in `backup` for the target
machine. These are storage identities, not encryption credentials. NTFS currently
reports `fuseblk`; a driver change requires updating validation deliberately.
Back up installed configuration securely before replacing it.

Install `backup` as root:root mode 0755 at `/usr/local/bin/backup` and the units
as root:root mode 0644 in `/etc/systemd/system/`. Store the existing Borg
passphrase in `/etc/backup/borg-passphrase` (root:root 0600), inside a root-owned
0700 `/etc/backup` directory. Never add this file, encryption keys, archive
listings, backup data, or private operational reports to Git. Keep an independent
secure recovery copy of the repository encryption key and passphrase.

Run `bash -n /usr/local/bin/backup`, `backup --check-mounts`,
`systemd-analyze verify /etc/systemd/system/backup.{service,timer}`, and
`systemctl daemon-reload` as appropriate. The service is static and is started
by the timer. Leave the timer disabled/stopped during validation.

## Safety and activation prerequisites

The script checks both exact mountpoints, UUIDs, filesystem types, filesystem
roots, and writable mount options. systemd requires/binds both mount units and
asserts mountpoints. Open directory descriptors keep copy paths on the intended
filesystems even after a lazy unmount. A single flock prevents overlapping server
runs; command failures and Borg warnings cause nonzero exits. After mount
validation, Borg relocation acceptance is scoped to the expected pinned alias
of the same repository. This avoids an interactive prompt for the fd path.

**The server flock does not lock laptop Borg repositories.** This script copies
the whole source tree, so repository writers must remain quiescent throughout
copying. Ordinary rsync against a live Borg repository is not a consistent backup.
Before scheduling, arrange an enforced quiet window or add a reviewed per-repository
`borg with-lock` copy procedure. `with-lock` does not require decryption keys,
but does update repository bookkeeping and copies lock files. Do not break locks
on source repositories; copied stale locks require deliberate handling on restore.
An interrupted copy must not be assumed usable. Never write independently to both
copies of the same encrypted Borg repository.

Before the first production run: review a checksum dry run, approve any genuine
content replacements, reconcile old timestamp differences using checksum-based
copying, and validate a real configuration backup and selected restores. A tiny
synthetic restore alone does not validate the full system or laptop recoverability.
Confirm time synchronization and enough space on both filesystems and root.

The timer runs daily at 02:00 America/New_York with persistent catch-up. Starting
it may immediately launch a missed run. Only after explicit operational approval,
enable/start `backup.timer`; do not enable the service independently.
