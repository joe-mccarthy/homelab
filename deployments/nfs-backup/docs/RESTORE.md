# Restore Runbook

This runbook explains how to recover files from the NFS Restic repository. It is written for both routine tests and real incidents.

## The Most Important Rule

Restore into `/restore`, inspect the result, and only then decide how to return files to production.

Do not point Restic directly at `/exports/docker` during the first recovery attempt. Restoring over live data can overwrite newer files, combine old and new application state, or damage a running database.

## What The Runner Mounts

Manual commands use:

```bash
sudo nfs-backup-job restic <restic arguments>
```

The runner provides the container with:

- the repository location
- the repository encryption password
- S3 credentials
- the persistent Restic cache
- configured host `restore_path` mounted as container directory `/restore`

It does not mount the live source for writing. The examples use the default
host `restore_path` of `/restore`. If that setting was changed, replace
host-side paths in commands such as `df`, `mkdir`, `ls`, and `du`; Restic
commands still use the fixed container path `/restore`.

## How Restored Paths Are Laid Out

The snapshots were created from the absolute path `/exports/docker`.

When restoring to `/restore`, Restic normally recreates that absolute hierarchy below the target:

```text
Original snapshot path:
/exports/docker/app/config.yml

Restored host path:
/restore/exports/docker/app/config.yml
```

Always inspect the resulting directory tree instead of assuming the restored file appears directly under `/restore`.

## Recovery Types

| Recovery type | Typical use |
| --- | --- |
| Single file | A configuration file was deleted or edited incorrectly. |
| Single directory | One application's data needs to be rolled back. |
| Full snapshot | The NFS dataset or server was lost. |
| Comparison only | Determine when a file changed before choosing a snapshot. |
| Test restore | Prove that backup data can be decrypted and reconstructed. |

## Before Starting

Confirm:

- no unrelated restore is already using `/restore`
- `/restore` has enough free space
- the Restic encryption key is available
- S3 credentials can read the repository
- the NFS server can reach the S3 endpoint
- you know approximately what path and time you need
- production applications will not read partially restored data

Check free space:

```bash
df -h /restore
```

Check whether another local runner job is active:

```bash
systemctl status 'nfs-backup@*.service'
docker ps --filter 'label=homelab.job'
```

The runner waits for another local job, but confirming the current state avoids surprising delays.

## Step 1: Confirm Repository Access

```bash
sudo nfs-backup-job restic cat config
```

A successful command prints repository configuration such as its version and ID. It must not prompt for an unknown password.

If this fails, stop and use the [Troubleshooting Guide](TROUBLESHOOTING.md). Do not initialize the location during a recovery incident: initialization creates a new repository; it does not repair access to the old one.

## Step 2: List Candidate Snapshots

List snapshots for this deployment:

```bash
sudo nfs-backup-job restic snapshots \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup
```

List only recent snapshots:

```bash
sudo nfs-backup-job restic snapshots \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup \
  --latest 10
```

The output includes:

- snapshot ID
- backup time
- hostname
- tags
- source path

Record the full or short snapshot ID you intend to use.

Do not automatically choose the newest snapshot. If data was corrupted or deleted several days ago, the newest snapshot may already contain the bad state.

## Step 3: Inspect A Snapshot Without Restoring

List all files in a snapshot:

```bash
sudo nfs-backup-job restic ls <snapshot-id>
```

List a specific directory recursively:

```bash
sudo nfs-backup-job restic ls \
  --recursive \
  <snapshot-id> \
  /exports/docker/app
```

Show ownership, permissions, size, and modification time:

```bash
sudo nfs-backup-job restic ls \
  --long \
  <snapshot-id> \
  /exports/docker/app
```

This confirms that the desired file existed in that snapshot before downloading data.

## Step 4: Prepare A Clean Restore Destination

Create a unique incident or test directory:

```bash
sudo mkdir -p /restore/incident-YYYYMMDD
```

With the default configuration, the helper mounts host `/restore` as container
`/restore`. Therefore a host directory such as
`/restore/incident-YYYYMMDD` is addressed inside Restic with the same path. If
the host setting differs, create the directory below that host path but still
use `/restore/incident-YYYYMMDD` as the Restic target.

Avoid deleting an earlier restore until it has been reviewed. If `/restore` already contains important files, choose a new unique target.

## Restore A Single File

```bash
sudo nfs-backup-job restic restore <snapshot-id> \
  --include /exports/docker/app/config.yml \
  --target /restore/incident-YYYYMMDD
```

Expected result:

```text
/restore/incident-YYYYMMDD/exports/docker/app/config.yml
```

Inspect it:

```bash
sudo ls -l /restore/incident-YYYYMMDD/exports/docker/app/config.yml
sudo file /restore/incident-YYYYMMDD/exports/docker/app/config.yml
```

For a text configuration file, compare it with the live copy:

```bash
sudo diff -u \
  /exports/docker/app/config.yml \
  /restore/incident-YYYYMMDD/exports/docker/app/config.yml
```

`diff` exits non-zero when files differ; differences are expected during recovery analysis.

## Restore A Directory

```bash
sudo nfs-backup-job restic restore <snapshot-id> \
  --include /exports/docker/app \
  --target /restore/incident-YYYYMMDD
```

Inspect restored size and contents:

```bash
sudo du -sh /restore/incident-YYYYMMDD/exports/docker/app
sudo ls -la /restore/incident-YYYYMMDD/exports/docker/app
```

## Restore A Complete Snapshot

Estimate the required restore size first:

```bash
sudo nfs-backup-job restic stats <snapshot-id>
df -h /restore
```

Restore:

```bash
sudo nfs-backup-job restic restore <snapshot-id> \
  --target /restore/incident-YYYYMMDD
```

The complete dataset should appear under:

```text
/restore/incident-YYYYMMDD/exports/docker
```

## Follow Restore Progress

The manual runner writes to the current terminal. To preserve output, run it from a managed shell session such as tmux or redirect through an approved logging process.

In another terminal, confirm the container is active:

```bash
docker ps --filter 'label=homelab.job=nfs-backup-restic'
```

The manual mode uses job label `nfs-backup-restic` because `restic` is the runner's selected job name.

When the command finishes, `docker ps` should no longer list the container.

## Validate Restored Files

Validation depends on the data type. At minimum:

```bash
sudo du -sh /restore/incident-YYYYMMDD/exports/docker
sudo find /restore/incident-YYYYMMDD/exports/docker -maxdepth 2 -type d
```

Check ownership and numeric IDs:

```bash
sudo ls -ln /restore/incident-YYYYMMDD/exports/docker
```

Check representative files:

```bash
sudo file /restore/incident-YYYYMMDD/exports/docker/path/to/file
sudo sha256sum /restore/incident-YYYYMMDD/exports/docker/path/to/file
```

Useful application-specific validation includes:

- parse YAML, JSON, or TOML configuration
- open images or media files
- test archive integrity
- run a database engine's native verification tool against a copy
- start an isolated application instance using only restored data
- compare expected file counts and directory sizes

A successful Restic command proves reconstruction completed. It does not prove the application-level contents are logically correct.

## Database Warning

Restic reads ordinary files. A database may modify several related files while backup is running. The resulting snapshot can be crash-consistent, inconsistent, or valid depending on the database and storage behavior.

Preferred database recovery sources are:

- native database dumps created before backup
- database-supported physical backups
- application-consistent filesystem snapshots
- database replication or point-in-time recovery logs

Never assume that copying restored live database files over a running database is safe.

## Returning A File To Production

The exact application procedure varies, but a safe general sequence is:

1. Identify application owners and dependencies.
2. Stop writes to the affected application.
3. Take a fresh manual backup of the current state.
4. Copy the live file or directory to a separate rollback location.
5. Copy the validated restored data into place.
6. Restore expected owner, group, mode, ACLs, and labels if required.
7. Start the application.
8. Check application logs and functionality.
9. Keep both the pre-change backup and restore workspace until validation ends.

Create a fresh safety snapshot before production changes:

```bash
sudo systemctl start nfs-backup@backup.service
```

Do not use a generic recursive copy command without first understanding ownership, symlinks, hardlinks, ACLs, sparse files, and application requirements.

## Compare Two Snapshots

To determine what changed between two backup points:

```bash
sudo nfs-backup-job restic diff <older-snapshot-id> <newer-snapshot-id>
```

Include metadata differences:

```bash
sudo nfs-backup-job restic diff \
  --metadata \
  <older-snapshot-id> \
  <newer-snapshot-id>
```

This can help identify when deletion or corruption first appeared.

## Find A File Across Snapshots

```bash
sudo nfs-backup-job restic find config.yml
```

Narrow to this backup set:

```bash
sudo nfs-backup-job restic find \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup \
  config.yml
```

Quote wildcard patterns so the host shell does not expand them:

```bash
sudo nfs-backup-job restic find '*.sqlite'
```

## Quarterly Restore Drill

Perform this at least quarterly:

1. Select the latest snapshot.
2. Select one small known text file.
3. Select one medium binary or media file.
4. Select one application directory.
5. Restore all selections to a dated test directory.
6. Compare text content and configuration syntax.
7. Open or checksum binary files.
8. Verify owner, group, and permissions.
9. Record snapshot ID, date, commands, duration, and outcome.
10. Remove test data only after recording success.

Example test target:

```bash
sudo mkdir -p /restore/test-YYYYMMDD
sudo nfs-backup-job restic restore latest \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup \
  --include /exports/docker/path/to/test-data \
  --target /restore/test-YYYYMMDD
```

When using `latest`, include the host, path, and tag filters so another backup
set cannot be selected accidentally.

## Disaster Recovery On A Replacement Host

Do not run the deployment playbook on a blank replacement host. It requires the
real source directory and can enable future backup timers. Disaster recovery
needs only a temporary, manual Restic container.

### 1. Prepare Protected Recovery Paths

Install Docker, attach enough recovery storage, and create root-only paths:

```bash
sudo install -d -m 0700 \
  /root/nfs-backup-recovery \
  /var/cache/nfs-backup-recovery \
  /restore
sudo install -m 0600 /dev/null \
  /root/nfs-backup-recovery/repository
sudo install -m 0600 /dev/null \
  /root/nfs-backup-recovery/password
sudo install -m 0600 /dev/null \
  /root/nfs-backup-recovery/restic.env
```

If the recovery filesystem is mounted somewhere other than `/restore`, replace
the host side of the restore bind mount in later commands. Keep the container
side as `/restore`.

### 2. Enter Existing Repository Settings

Use `sudoedit` so secrets are not command-line arguments or shell-history
entries:

```bash
sudoedit /root/nfs-backup-recovery/repository
sudoedit /root/nfs-backup-recovery/password
sudoedit /root/nfs-backup-recovery/restic.env
```

The `repository` file contains the exact existing repository URL, for example:

```text
s3:https://s3.example.com/homelab-restic
```

The `password` file contains only the existing Restic encryption password.

The environment file contains the existing backend settings:

```text
RESTIC_REPOSITORY_FILE=/run/secrets/restic-repository
RESTIC_PASSWORD_FILE=/run/secrets/restic-password
RESTIC_CACHE_DIR=/root/.cache/restic
AWS_DEFAULT_REGION=us-east-1
AWS_ACCESS_KEY_ID=REPLACE_WITH_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY=REPLACE_WITH_SECRET_ACCESS_KEY
TZ=Europe/London
```

Confirm permissions without printing contents:

```bash
sudo stat -c '%a %U:%G %n' \
  /root/nfs-backup-recovery/repository \
  /root/nfs-backup-recovery/password \
  /root/nfs-backup-recovery/restic.env
```

Every file should report mode `600` and owner `root:root`.

### 3. Open The Existing Repository

Pull the same pinned image used by the deployment:

```bash
sudo docker pull mazzolino/restic:1.8.2
```

Use this read-only metadata command to list matching snapshots:

```bash
sudo docker run --rm \
  --security-opt no-new-privileges:true \
  --env SKIP_INIT=true \
  --env-file /root/nfs-backup-recovery/restic.env \
  --mount type=bind,src=/root/nfs-backup-recovery/repository,dst=/run/secrets/restic-repository,readonly \
  --mount type=bind,src=/root/nfs-backup-recovery/password,dst=/run/secrets/restic-password,readonly \
  --mount type=bind,src=/var/cache/nfs-backup-recovery,dst=/root/.cache/restic \
  mazzolino/restic:1.8.2 \
  snapshots \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup
```

If this reports an empty repository or wrong password, stop. Recheck the
repository URL and encryption key. Never run `restic init` during disaster
recovery: initialization creates a new repository and does not reconnect or
repair the old one.

### If The Repository Is Locked

A replacement host has no installed `nfs-backup-job` wrapper. List lock objects
with the same protected recovery files:

```bash
sudo docker run --rm \
  --security-opt no-new-privileges:true \
  --env SKIP_INIT=true \
  --env-file /root/nfs-backup-recovery/restic.env \
  --mount type=bind,src=/root/nfs-backup-recovery/repository,dst=/run/secrets/restic-repository,readonly \
  --mount type=bind,src=/root/nfs-backup-recovery/password,dst=/run/secrets/restic-password,readonly \
  --mount type=bind,src=/var/cache/nfs-backup-recovery,dst=/root/.cache/restic \
  mazzolino/restic:1.8.2 \
  list locks
```

Do not remove a lock merely because the original host is unreachable. Confirm
that no backup, prune, check, restore, or other Restic client can still be
running against this repository. A network partition can make an active host
look failed from the recovery site.

Only after proving every listed lock is stale, remove stale locks:

```bash
sudo docker run --rm \
  --security-opt no-new-privileges:true \
  --env SKIP_INIT=true \
  --env-file /root/nfs-backup-recovery/restic.env \
  --mount type=bind,src=/root/nfs-backup-recovery/repository,dst=/run/secrets/restic-repository,readonly \
  --mount type=bind,src=/root/nfs-backup-recovery/password,dst=/run/secrets/restic-password,readonly \
  --mount type=bind,src=/var/cache/nfs-backup-recovery,dst=/root/.cache/restic \
  mazzolino/restic:1.8.2 \
  unlock
```

Run `list locks` again, then retry the filtered `snapshots` command. Unlocking
does not repair repository corruption; it only removes stale lock objects.

### 4. Restore A Known-Good Snapshot

Select a snapshot ID based on incident time, then restore it into a dated
directory:

```bash
sudo mkdir -p /restore/disaster-YYYYMMDD
sudo docker run --rm \
  --security-opt no-new-privileges:true \
  --env SKIP_INIT=true \
  --env-file /root/nfs-backup-recovery/restic.env \
  --mount type=bind,src=/root/nfs-backup-recovery/repository,dst=/run/secrets/restic-repository,readonly \
  --mount type=bind,src=/root/nfs-backup-recovery/password,dst=/run/secrets/restic-password,readonly \
  --mount type=bind,src=/var/cache/nfs-backup-recovery,dst=/root/.cache/restic \
  --mount type=bind,src=/restore,dst=/restore \
  mazzolino/restic:1.8.2 \
  restore <snapshot-id> \
  --target /restore/disaster-YYYYMMDD
```

The recovered dataset is normally under:

```text
/restore/disaster-YYYYMMDD/exports/docker
```

Validate files, ownership, databases, and application-level behavior before
rebuilding NFS exports or applications from this copy. Keep the protected
recovery configuration until recovery and verification are complete, then
remove it using the host's approved secret-disposal process.

## Cleaning Up A Restore Workspace

Before deletion:

- confirm recovered data has been validated
- confirm required data has been copied elsewhere
- record the snapshot ID and result
- confirm the path is beneath `/restore`

Then remove only the specific dated workspace using an approved host-administration process.

Do not delete paths from `/exports/docker` or objects from the S3 repository as part of restore cleanup.

## Common Restore Problems

### Snapshot ID Not Found

Refresh the snapshot list and ensure retention has not expired it:

```bash
sudo nfs-backup-job restic snapshots \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup
```

### Included Path Restores Nothing

Inspect exact paths stored in the snapshot:

```bash
sudo nfs-backup-job restic ls <snapshot-id>
```

Use the absolute snapshot path beginning with `/exports/docker`.

### Permission Denied Under `/restore`

The runner and restore container operate as root, and `/restore` is mode `0700`. Run operator commands through `sudo`.

### No Space Left On Device

Stop and free or add restore storage. Previous repository snapshots remain safe because the restore target is separate from S3.

```bash
df -h /restore
sudo du -sh /restore/*
```

### Wrong Password

Verify that the Vault value is the exact encryption key used to initialize the repository. S3 credentials and Restic encryption passwords are different.

### Repository Locked

Check for active jobs first:

```bash
docker ps --filter 'label=homelab.job'
systemctl status 'nfs-backup@*.service'
sudo nfs-backup-job restic list locks
```

Only if no Restic process is active:

```bash
sudo nfs-backup-job restic unlock
```

On a blank replacement host, use the standalone container commands under
[If The Repository Is Locked](#if-the-repository-is-locked). See the
[Troubleshooting Guide](TROUBLESHOOTING.md) for more detail.

## Restore Completion Checklist

A restore is complete only when:

- the intended snapshot and path were selected
- Restic completed without errors
- restored paths are understood
- file sizes and counts are plausible
- representative content is readable
- ownership and permissions are correct
- application-specific validation passed
- production changes were controlled and reversible
- the recovery result was recorded

The ability to list snapshots is not proof of recovery. A validated restored file is the minimum evidence.
