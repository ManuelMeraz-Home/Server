# Server backups

Bash, Python 3, Borg 1.2 and rsync copy `/data` to `/data_sync`.
Borg stores `/etc`, `/usr/local` and `/var` in `/data/server`. Archive pruning
is disabled. Ordinary files preserve times/symlinks; destination-only ordinary
files are retained. NTFS ownership/permissions are not a POSIX replica.

## Install

Review the mount UUIDs/types in `backup`. Install it root:root 0755 as
`/usr/local/bin/backup`; install `backup-secondary` root:root 0755 as
`/usr/local/libexec/backup-secondary`. Install units root:root 0644 under
`/etc/systemd/system/`. Preserve restricted originals before replacing anything.

Keep the existing passphrase in `/etc/backup/borg-passphrase` (root:root 0600),
inside root-owned 0700 `/etc/backup`. State belongs in `/etc/backup/state`
(root-only 0700, files 0600). Never commit credentials, state, archive listings,
backup data or private operational reports. Keep recovery credentials separately.

Check Bash/Python syntax, `backup --check-mounts`, and `systemd-analyze verify`
on both units. Reload systemd after unit changes. Keep the timer stopped during
validation; the service is static and must not be enabled independently.

## Copy safety

Both exact mountpoints, UUIDs, filesystem types and writable mounts are checked.
Pinned directory descriptors prevent writes falling through to root after unmount.
The server flock serializes jobs; failures and Borg warnings return nonzero.

All known repositories are directly below `/data`. Directories containing
`config` and `data/` are excluded from ordinary rsync and copied under exclusive
native `borg with-lock` locks, with a 60-second acquisition timeout. No client
key is needed for locking, but source bookkeeping changes and clients may wait
or fail on contention. Do not create repositories during a copy. Nested
repositories need a separate reviewed policy before adding them.

Each repository has fixed current, `.NAME.staging` and `.NAME.previous` slots.
Stages copy current source metadata independently and hard-link only reusable
segment files. Changed data uses rsync temporary-file replacement, never in-place
or append writes. Obsolete files are omitted from fresh stages; deletion is
limited to staging/data. Current and previous contents remain protected during
staging. Shared segment timestamps may change; contents must not.

Normal runs use timestamp/size checks for historical segments and checksums for
the server repository. Metadata is checksummed; index integrity, committed
transaction and segment references are checked. The server receives a full
repository-only check and selected authenticated restores before/after promotion.
Historical same-size payload corruption is not exhaustively checked. Do not run
Borg writes/maintenance independently on either secondary generation.

## One-previous retention (explicit authorization required)

Without `/etc/backup/retire-previous-approved` (root-owned 0600), a retained
previous generation blocks another rotation. The marker is operator authorization;
installation alone must not create it. Only enable after explicit retention approval.

With authorization: validate recorded current identity and the incoming stage,
record durable cleanup/promotion intent, retire only the identified older previous,
rename current to previous, then promote staging. One verified predecessor remains.
Current remains intact during staging, validation and older-previous cleanup.
Interrupted promotion is recovered from the root-only journal and directory identities.
Unexpected identities or unresolved unverified stages fail closed. No numbered
generations accumulate. Physical power-loss/controller behavior is not proven by
process-interruption tests.

`fast` is the one-time repair mode and requires private incident artifacts outside
Git. It trusts same-size historical segments and never retires a previous generation.
Use normal mode through `backup` for scheduled operation.

## Schedule

The timer is daily at 02:00 America/New_York with persistent catch-up; activation
may immediately run a missed job. Require successful production backup/restore
validation, approved retention, and explicit timer activation approval before
starting it. Coordinate laptop schedules to avoid source-lock contention.
