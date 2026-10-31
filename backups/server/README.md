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

Top-level directories containing `config` and `data/` are excluded from the
ordinary rsync pass and copied separately under native `borg with-lock` exclusive
locks. Source `lock.exclusive` and `lock.roster` files are excluded. Lock acquisition
waits up to 60 seconds; lock or copy failure fails the backup. Native locking needs
no laptop decryption key, but updates source repository bookkeeping. Client jobs
may wait or fail while a copy holds the lock; initial checksum comparison can take
hours. Prefer an off-hours window.

The current layout places all repositories directly under `/data`. Do not create
new repositories during copying. Nested repositories need an explicit copying
policy before scheduling; they are not discovered automatically. The bounded
layout inspection and copy policy must be revisited when adding laptop repositories.
Do not break source locks or run Borg writes independently on both encrypted copies.
Existing stale destination locks are not removed automatically. An interrupted
repository copy must not be assumed usable. No destination-only files are deleted.

Before the first production run: review a checksum dry run, approve any genuine
content replacements, reconcile old timestamp differences using checksum-based
copying, and validate a real configuration backup and selected restores. A tiny
synthetic restore alone does not validate the full system or laptop recoverability.
Confirm time synchronization and enough space on both filesystems and root.

The timer runs daily at 02:00 America/New_York with persistent catch-up. Starting
it may immediately launch a missed run. Only after explicit operational approval,
enable/start `backup.timer`; do not enable the service independently.
