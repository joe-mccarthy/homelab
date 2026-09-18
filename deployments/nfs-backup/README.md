# NFS Backup Solution

Encrypted, incremental backups of `/exports/docker` to S3-compatible object storage using Restic, short-lived Docker containers, systemd timers, and Ansible.

This document assumes the reader has never used Restic or systemd timers before. It explains the design, installation, daily operation, safety checks, and recovery workflow.

## Read This First

The most important facts are:

1. No backup container stays running between jobs.
2. The backup timer activates at 00:00. The runner starts a temporary container
   as soon as any earlier local job has released its lock.
3. Prune and check use the same start-work-exit lifecycle on their own schedules.
4. Backups are encrypted before they leave the NFS server.
5. The Restic encryption key is required for every restore. Losing it means losing the backups.
6. The source is mounted read-only, and restores go to `/restore`, not over live data.
7. Repository initialization is intentionally manual so a typo cannot silently create a new empty backup history.
8. A successful backup command is not enough. Test restores are required to prove recoverability.

Additional runbooks:

- [Restore Runbook](docs/RESTORE.md)
- [Retention Guide](docs/RETENTION.md)
- [Troubleshooting Guide](docs/TROUBLESHOOTING.md)

## What This Deployment Does

This deployment protects the host directory `/exports/docker`. In this homelab, that directory contains application data exported by the NFS server.

It provides:

- one encrypted Restic snapshot every day at midnight
- incremental uploads, so unchanged file content is not uploaded again
- content deduplication across snapshots
- maximum Restic compression
- configurable snapshot retention
- weekly repository pruning
- weekly structural and sampled data-integrity checks
- a protected, persistent Restic metadata cache
- manual snapshot, check, maintenance, and restore commands
- deployment safety checks before scheduled jobs are enabled
- systemd journal logs for every scheduled operation

## What This Deployment Does Not Do

This distinction is important for a new operator.

- It does not keep a full Docker container running as a scheduler.
- It does not back up every filesystem on the machine.
- It does not automatically stop applications before reading their files.
- It does not automatically create database-native dumps.
- It does not guarantee application consistency for a database that changes during backup.
- It does not make S3 immutable by itself.
- It does not replace a second independent backup copy.
- It does not automatically prove that restored applications can start.

For databases, use application-specific dumps or filesystem snapshots if crash-consistent files are not sufficient. Store those dumps beneath `/exports/docker` so Restic includes them.

## Key Terms

| Term | Plain-English meaning |
| --- | --- |
| Ansible | Automation that installs and configures the backup system on the correct hosts. |
| Docker image | The packaged software used to start a temporary Restic container. |
| Container | A temporary isolated process. These containers exist only while a job is running. |
| systemd | The Linux service manager already running on the NFS host. It schedules these jobs without another permanent container. |
| Service unit | A systemd definition describing what command to run. Here it runs one Restic job and exits. |
| Timer unit | A systemd definition describing when to start a service unit. |
| Restic | The backup program that encrypts, deduplicates, uploads, checks, and restores data. |
| Repository | The encrypted collection of Restic objects in the configured S3 bucket or prefix. |
| Snapshot | Restic's record of the source filesystem at one backup time. |
| Incremental backup | A backup that uploads only new content, while still presenting each snapshot as a complete point-in-time view. |
| Deduplication | Reusing already stored content instead of storing identical bytes again. |
| Forget | Remove snapshot references according to retention rules. |
| Prune | Delete repository data no longer referenced by any retained snapshot. |
| Check | Validate repository structure and, optionally, read stored data to verify integrity. |
| Lock | A repository object used by Restic to stop unsafe concurrent operations. |
| S3 | An object-storage API used by AWS and many compatible providers. |

## Architecture

The normal scheduled path is:

```text
systemd timer
    |
    v
nfs-backup@<job>.service
    |
    v
/usr/local/sbin/nfs-backup-job <job>
    |
    v
docker run --rm mazzolino/restic:1.8.2
    |                         |
    | read-only              | encrypted network traffic
    v                         v
/exports/docker         S3 Restic repository
```

The restore path is separate:

```text
S3 Restic repository
    |
    | decrypt and download
    v
/restore on the NFS server
```

The live source and restore destination are deliberately different. An operator must inspect restored data before choosing whether to copy it back into production.

## Why systemd Schedules the Jobs

The NFS server already runs systemd. A systemd timer consumes negligible resources and starts a temporary Docker container only when work is due. This avoids an always-running scheduling container and uses only the host's local Docker daemon.

## Job Lifecycle

### While Idle

- Three systemd timer units are active.
- No Restic container is running.
- The Docker image remains cached locally for the next run.
- The Restic metadata cache remains under `/var/cache/nfs-backup`.
- No source files are being read and no S3 requests are made.

### When a Job Is Due

1. The timer asks systemd to start a service instance.
2. The service invokes `/usr/local/sbin/nfs-backup-job` with `backup`, `prune`, or `check`.
3. The runner waits for its local lock so two local jobs cannot overlap.
4. The runner starts a temporary Docker container.
5. Resticker checks that the configured repository exists.
6. Restic performs the requested operation.
7. Output streams to the systemd journal.
8. Restic exits and returns its status to Docker and systemd.
9. Docker removes the container because `--rm` is set.
10. The local lock is released.

### If Another Restic Client Is Active

The local `flock` only coordinates commands launched by this host-side runner. Restic's repository lock coordinates all clients that use the repository.

Each command retries a repository lock for up to `30m`. If the repository remains locked, the job fails instead of bypassing the lock.

## Jobs And Schedule

Schedules are defined in [`group_vars/all.yml`](group_vars/all.yml) and interpreted in `Europe/London` even if the host uses UTC.

| Job | systemd calendar | Local time | Purpose |
| --- | --- | --- | --- |
| Backup | `*-*-* 00:00:00` | Every day at 00:00 | Create an encrypted snapshot of `/exports/docker`. |
| Prune | `Sun *-*-* 03:30:00` | Sunday at 03:30 | Apply retention and prune when snapshots expire. |
| Check | `Sun *-*-* 08:30:00` | Sunday at 08:30 | Check all metadata and a random 10% of data packs. |

The gaps allow the midnight backup to finish before prune and allow prune to finish before check. If a prior local job is still running, the later service waits on `/run/lock/nfs-backup.lock` without starting another container.

### Missed Schedules

`persistent_timers` is `false` by default.

With `false`:

- a host that is off at 00:00 misses that day's backup
- starting the host at 08:00 does not trigger a catch-up backup
- the next backup remains the next 00:00 occurrence

With `true`:

- systemd records missed timer occurrences
- one missed activation runs shortly after timers become active again
- a server boot may therefore start a backup outside 00:00

The default is `false` so a missed midnight activation is not replayed later.
This controls timer catch-up, not lock contention: a timer activated at 00:00
can still wait for an earlier local job before its container starts.

## Backup Details

The scheduled backup command is equivalent to:

```bash
restic backup /exports/docker \
  --host nfs-backup \
  --tag nfs-backup \
  --verbose
```

The fixed hostname is important. Docker normally gives temporary containers random names, but Restic uses the hostname and source path to find a suitable parent snapshot. A stable hostname preserves incremental scanning behavior between runs.

The `nfs-backup` tag makes this backup set easier to identify.

The source bind mount is read-only inside the container. Restic can read files and metadata but cannot modify `/exports/docker` through that mount.

## Retention Summary

The balanced default keeps:

- every snapshot made within one day of the newest snapshot
- one daily snapshot for 14 backup days
- one weekly snapshot for 8 backup weeks
- one monthly snapshot for 12 months
- one yearly snapshot for 3 years

Retention rules are combined with OR logic. A snapshot matching any rule is kept. Daily, weekly, monthly, and yearly selections often refer to the same snapshot, so do not add the numbers together.

Restic snapshots are not independent full copies. Identical content is stored once, so retaining more snapshot metadata usually costs much less than retaining the same number of traditional full backups.

See the [Retention Guide](docs/RETENTION.md) before changing policy or estimating storage.

## Check Summary

Every scheduled check validates:

- repository configuration
- indexes
- snapshot metadata
- directory trees
- blob references
- pack presence and structure

`--read-data-subset=10%` also downloads and cryptographically verifies a random 10% of data packs. Random sampling lowers S3 bandwidth but does not guarantee complete coverage within a fixed number of runs.

Periodically run a complete data check when bandwidth and S3 charges permit:

```bash
sudo nfs-backup-job restic check --read-data
```

## Source Repository Layout

| Path | Purpose |
| --- | --- |
| `deploy.yml` | Coordinates safe preparation and timer activation on the NFS host. |
| `group_vars/all.yml` | Non-secret configuration plus references to Vault secrets. |
| `roles/deploy_backup/tasks/main.yml` | Installs files and performs safety checks without enabling timers. |
| `roles/deploy_backup/tasks/enable_timers.yml` | Activates timers only after validation succeeds. |
| `roles/deploy_backup/templates/nfs-backup-job.j2` | Builds the host-side Docker runner. |
| `roles/deploy_backup/templates/nfs-backup@.service.j2` | Builds the one-shot systemd service template. |
| `roles/deploy_backup/templates/nfs-backup.timer.j2` | Builds each systemd timer. |
| `roles/deploy_backup/templates/restic.env.j2` | Builds protected container environment settings. |
| `docs/` | Operator runbooks for restores, retention, and troubleshooting. |

## Files Installed On The NFS Server

| Installed path | Mode | Purpose |
| --- | ---: | --- |
| `/usr/local/sbin/nfs-backup-job` | `0700` | Root-only wrapper that launches temporary Restic containers. |
| `/etc/nfs-backup/restic.env` | `0600` | Restic, AWS, cache, compression, and timezone environment. |
| `/etc/nfs-backup/repository` | `0600` | Restic repository URL mounted into containers. |
| `/etc/nfs-backup/password` | `0600` | Restic encryption password mounted into containers. |
| `/etc/systemd/system/nfs-backup@.service` | `0644` | One-shot service template shared by all jobs. |
| `/etc/systemd/system/nfs-backup-backup.timer` | `0644` | Daily backup schedule. |
| `/etc/systemd/system/nfs-backup-prune.timer` | `0644` | Weekly retention schedule. |
| `/etc/systemd/system/nfs-backup-check.timer` | `0644` | Weekly integrity-check schedule. |
| `/var/cache/nfs-backup` | `0700` | Persistent Restic metadata cache. |
| `/restore` | `0700` | Safe destination for restored files. |
| `/run/lock/nfs-backup.lock` | Runtime | Local lock held while the runner is active. |

Do not edit generated files on the server. Edit this repository and rerun Ansible, otherwise the next deployment overwrites manual changes.

## Prerequisites

Confirm all items before deployment.

### Host And Inventory

- Ansible collections from `requirements.yml` are installed on the controller.
- The NFS server runs a systemd-based Linux distribution.
- Docker is installed on the NFS server.
- Docker starts successfully with `systemctl status docker`.
- Exactly one inventory host belongs to `nfs_servers`.
- The NFS host has `/exports/docker` as a real directory.
- The NFS host can resolve and reach the S3 endpoint.
- Host time is synchronized, normally with systemd-timesyncd, chrony, or NTP.

Install the required Ansible collections from the repository root:

```bash
ansible-galaxy collection install -r requirements.yml
```

Example inventory structure:

```yaml
all:
  children:
    nfs_servers:
      hosts:
        odin:
          ansible_host: 192.168.1.11
          ansible_user: pi

```

### S3 Repository

- The bucket or prefix exists.
- The repository or prefix is dedicated to this deployment where practical.
- An existing deployment has at least one snapshot with host `nfs-backup`, path
  `/exports/docker`, and tag `nfs-backup`.
- The S3 access key can list, read, write, and delete repository objects.
- The region and endpoint match the provider.
- The configured repository URL uses `s3:https:` so data is encrypted in
  transit as well as encrypted by Restic before upload.

Typical required API capabilities are:

- `s3:ListBucket`
- `s3:GetObject`
- `s3:PutObject`
- `s3:DeleteObject`

Prune cannot reclaim storage without delete permission.

### Secrets And Ansible Vault

The playbook expects these values under `vault.services.nfs_backup`:

```yaml
vault:
  services:
    nfs_backup:
      encryption_key: REPLACE_WITH_A_LONG_RANDOM_VALUE
      s3:
        bucket: s3:https://s3.example.com/homelab-restic
        region: us-east-1
        key_id: REPLACE_WITH_ACCESS_KEY_ID
        access_key: REPLACE_WITH_SECRET_ACCESS_KEY
```

The exact expected repository-wide Vault schema is documented in [`vault.template.yml`](../../vault.template.yml).

If `vault.yml` does not exist yet, create it encrypted from the beginning:

```bash
ansible-vault create vault.yml
```

Enter at least the `vault.services.nfs_backup` structure shown above. If the
repository-wide Vault already exists, update it with
`ansible-vault edit vault.yml` instead.

Do not place real secrets in `group_vars/all.yml`, README files, shell history, issues, or commit messages.

The encryption key must not have leading or trailing whitespace. Restic trims password files, so the deployment rejects a value whose meaning would change after trimming.

An encrypted `vault.yml` is not loaded merely because it exists. Every real
deployment command in this guide explicitly passes it as extra variables:

```bash
ansible-playbook \
  -i inventory.yml \
  deployments/nfs-backup/deploy.yml \
  --extra-vars @vault.yml \
  --ask-vault-pass
```

If a securely managed Vault password file is already part of your workflow,
use `--vault-password-file` instead of `--ask-vault-pass`.

## Configuration Reference

All primary settings are under `nfs_backup` in [`group_vars/all.yml`](group_vars/all.yml).

| Variable | Default | Meaning | Change carefully? |
| --- | --- | --- | --- |
| `encryption_key` | Vault reference | Password used to encrypt and decrypt the repository. | Yes. Changing it does not automatically change an existing repository key. |
| `s3.bucket` | Vault reference | Full Restic S3 repository URL. | Yes. A different value points to different backup history. |
| `s3.region` | Vault reference | S3 provider region. | Yes. A wrong region can cause authentication or redirect errors. |
| `s3.key_id` | Vault reference | S3 access-key identifier. | Rotate together with provider credentials. |
| `s3.access_key` | Vault reference | S3 secret access key. | Keep secret and rotate together with the key ID. |
| `version` | `1.8.2` | Resticker Docker image version. | Review release notes before updating. |
| `source_path` | `/exports/docker` | Host directory mounted read-only at container path `/exports/docker`. | Changing it changes host data, not the path recorded in snapshots. |
| `restore_path` | `/restore` | Host destination mounted at container path `/restore`. | Use the host value for disk checks and the container value in Restic commands. |
| `cache_path` | `/var/cache/nfs-backup` | Host-persistent Restic cache. | Safe if moved with correct permissions. |
| `config_path` | `/etc/nfs-backup` | Root-only generated configuration directory. | Avoid shared or user-writable locations. |
| `timezone` | `Europe/London` | Timezone appended to systemd calendar expressions. | Validate with `systemd-analyze calendar`. |
| `persistent_timers` | `false` | Whether missed jobs catch up after downtime. | `true` permits jobs outside normal wall-clock times. |
| `jobs.backup.calendar` | `*-*-* 00:00:00` | Daily backup calendar. | Use systemd, not cron, syntax. |
| `jobs.prune.calendar` | `Sun *-*-* 03:30:00` | Weekly retention calendar. | Leave enough time after backup. |
| `jobs.check.calendar` | `Sun *-*-* 08:30:00` | Weekly check calendar. | Leave enough time after prune. |
| `retention.keep_within` | `1d` | Keep every recent snapshot within this duration. | Higher values retain dense recent history. |
| `retention.keep_daily` | `14` | Number of backup days represented. | Preview before reducing. |
| `retention.keep_weekly` | `8` | Number of backup weeks represented. | Preview before reducing. |
| `retention.keep_monthly` | `12` | Number of months represented. | Preview before reducing. |
| `retention.keep_yearly` | `3` | Number of years represented. | Preview before reducing. |
| `check_read_data_subset` | `10%` | Random data-pack percentage read by scheduled checks. | Higher values use more time and S3 reads. |
| `retry_lock` | `30m` | Maximum wait for another Restic repository lock. | Keep below the gap between scheduled jobs where practical. |

### Host Paths And Container Paths

The left side of a Docker bind mount is configurable on the host; the right
side is intentionally fixed inside each temporary container:

| Host setting | Fixed container path | Used for |
| --- | --- | --- |
| `source_path` | `/exports/docker` | Backup input and recorded snapshot path. |
| `restore_path` | `/restore` | Restic restore target. |
| `cache_path` | `/root/.cache/restic` | Persistent metadata cache. |

For example, changing `source_path` to `/srv/application-data` still records
the Restic snapshot path as `/exports/docker`. Changing `restore_path` to
`/mnt/recovery` means host disk checks use `/mnt/recovery`, while Restic still
uses `--target /restore` inside the container.

Keep `restore_path`, `cache_path`, and `config_path` outside `source_path` and
outside one another. Otherwise backups can ingest restore output or cache data,
and placing `config_path` below the source can copy credentials into snapshots.
Also account for symlinks and separate mounts that may make apparently
different paths refer to the same storage.

## Validate Before Deployment

Run commands from the repository root so Ansible loads the root `ansible.cfg` and shared role paths.

Check syntax without changing hosts:

```bash
ansible-playbook \
  -i inventory.yml \
  deployments/nfs-backup/deploy.yml \
  --syntax-check
```

Confirm Ansible sees the expected hosts:

```bash
ansible-playbook \
  -i inventory.yml \
  deployments/nfs-backup/deploy.yml \
  --list-hosts
```

Check basic connectivity:

```bash
ansible -i inventory.yml nfs_servers -m ansible.builtin.ping
```

Preview retention before the first prune:

```bash
sudo nfs-backup-job restic forget \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup \
  --keep-within 1d \
  --keep-daily 14 \
  --keep-weekly 8 \
  --keep-monthly 12 \
  --keep-yearly 3 \
  --group-by host,paths \
  --dry-run
```

## Deploy An Existing Repository

Run the playbook:

```bash
ansible-playbook \
  -i inventory.yml \
  deployments/nfs-backup/deploy.yml \
  --extra-vars @vault.yml \
  --ask-vault-pass
```

### What The Playbook Does

The deployment performs these steps in order:

1. Confirm exactly one NFS host is configured.
2. Ensure Docker and `flock` are available.
3. If this scheduler is already installed, stop and disable its timers.
4. Refuse to replace files while a systemd or manual runner job is active.
5. Validate non-empty, safe configuration values.
6. Verify `/exports/docker` exists and is a directory.
7. Create protected configuration, cache, and restore directories.
8. Install the runner and systemd units.
9. Ask `systemd-analyze verify` to parse installed units.
10. Pull the image if missing and open the configured repository.
11. Require an existing snapshot for host `nfs-backup`, path `/exports/docker`,
   and tag `nfs-backup`.
12. Refuse activation if any repository lock exists.
13. Enable and start the three systemd timers.

No timer is enabled until all validation succeeds. On a redeployment, timers
remain paused if deployment fails after they are stopped. Correct the reported
problem and rerun the playbook before manually re-enabling timers.

## Initialize A Brand-New Repository

Automatic initialization is disabled. This prevents a misspelled bucket or prefix from becoming a valid but empty repository while the real backup history remains elsewhere.

For a genuinely new repository:

1. Run the playbook with `--extra-vars @vault.yml --ask-vault-pass`.
2. It installs the runner, then safely stops when repository or expected
   snapshot validation fails.
3. Confirm the S3 URL carefully.
4. Initialize the repository explicitly.
5. Create and inspect the first backup.
6. Rerun the playbook to complete timer activation.

Commands on the NFS host:

```bash
sudo nfs-backup-job restic init
sudo nfs-backup-job backup
sudo nfs-backup-job restic snapshots \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup
```

Complete deployment from the repository root:

```bash
ansible-playbook \
  -i inventory.yml \
  deployments/nfs-backup/deploy.yml \
  --extra-vars @vault.yml \
  --ask-vault-pass
```

Immediately store a secure offline copy of the encryption key.

## Verify After Deployment

### Confirm Timers Are Active

```bash
systemctl list-timers --all 'nfs-backup-*'
```

Expected timer names:

- `nfs-backup-backup.timer`
- `nfs-backup-prune.timer`
- `nfs-backup-check.timer`

Inspect one timer in detail:

```bash
systemctl status nfs-backup-backup.timer
systemctl show nfs-backup-backup.timer \
  --property=ActiveState \
  --property=LastTriggerUSec \
  --property=NextElapseUSecRealtime
```

### Confirm No Idle Container Exists

```bash
docker ps --filter 'label=homelab.job'
```

No output between jobs is the expected healthy state.

### Run A Manual Backup

```bash
sudo systemctl start nfs-backup@backup.service
```

`systemctl start` waits for the one-shot service to finish. In another terminal, follow progress:

```bash
journalctl -fu nfs-backup@backup.service
```

### Confirm A New Snapshot Exists

```bash
sudo nfs-backup-job restic snapshots \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup \
  --latest 5
```

Also confirm the manual service returned success. Restic exit status 3 can
create an incomplete snapshot when some source files were unreadable:

```bash
systemctl show nfs-backup@backup.service \
  --property=ExecMainStartTimestamp \
  --property=ExecMainExitTimestamp \
  --property=Result \
  --property=ExecMainStatus
```

Expected values are `Result=success`, `ExecMainStatus=0`, and nonempty recent
start/exit timestamps corresponding to the snapshot time. Status values alone
can look successful before a service has run.

### Perform A Test Restore

Do not postpone restore testing until an emergency. Follow the [Restore Runbook](docs/RESTORE.md) and restore at least one known file into `/restore`.

## Routine Operations

### View Timer Status

```bash
systemctl status nfs-backup-backup.timer
systemctl status nfs-backup-prune.timer
systemctl status nfs-backup-check.timer
```

### View Service Results

```bash
systemctl status nfs-backup@backup.service
systemctl status nfs-backup@prune.service
systemctl status nfs-backup@check.service
```

### View Logs

Latest backup logs:

```bash
journalctl -u nfs-backup@backup.service -n 200 --no-pager
```

Logs since yesterday:

```bash
journalctl -u nfs-backup@backup.service --since yesterday
```

Follow the active job:

```bash
journalctl -fu nfs-backup@backup.service
```

### Start Jobs Manually

```bash
sudo systemctl start nfs-backup@backup.service
sudo systemctl start nfs-backup@prune.service
sudo systemctl start nfs-backup@check.service
```

To return immediately and watch logs separately:

```bash
sudo systemctl start --no-block nfs-backup@backup.service
journalctl -fu nfs-backup@backup.service
```

### List Snapshots

Manual wrapper syntax always places the Restic subcommand first, followed by
its flags: `nfs-backup-job restic <command> [options...]`. For example, use
`restic snapshots --latest 5`, not global flags before `snapshots`.

```bash
sudo nfs-backup-job restic snapshots
```

Filter to this backup set:

```bash
sudo nfs-backup-job restic snapshots \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup
```

### Show Repository Statistics

```bash
sudo nfs-backup-job restic stats
sudo nfs-backup-job restic stats --mode raw-data
```

`raw-data` estimates unique repository blobs referenced by selected snapshots.
It is not the provider's physical billed usage because unreferenced packs,
repository overhead, and retained S3 object versions may also consume space.

### Inspect Files In A Snapshot

```bash
sudo nfs-backup-job restic ls latest \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup
```

### Check The Repository

Structural check only:

```bash
sudo nfs-backup-job restic check
```

Full data read:

```bash
sudo nfs-backup-job restic check --read-data
```

### Inspect Repository Locks

```bash
sudo nfs-backup-job restic list locks
```

Do not remove a lock until you have confirmed that no Restic process is active.

Remove stale locks:

```bash
sudo nfs-backup-job restic unlock
```

## Restoring Data

The short version is:

```bash
sudo nfs-backup-job restic snapshots \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup
sudo nfs-backup-job restic restore <snapshot-id> --target /restore
```

For a single path:

```bash
sudo nfs-backup-job restic restore <snapshot-id> \
  --include /exports/docker/path/to/data \
  --target /restore
```

Restic recreates the snapshot's absolute path below the target. A file originally stored as `/exports/docker/app/config.yml` is normally restored as:

```text
/restore/exports/docker/app/config.yml
```

Do not restore directly over `/exports/docker` until the restored files have been inspected. Follow the complete [Restore Runbook](docs/RESTORE.md).

## Monitoring Recommendations

At minimum, monitor:

- failed `nfs-backup@*.service` units
- absence of a recent matching snapshot or a successful recent service result
- repeated repository-lock failures
- S3 authentication or network failures
- unexpected repository growth
- integrity-check failures
- free space beneath `/restore` before recovery

Useful commands for a simple health check:

```bash
systemctl is-active nfs-backup-backup.timer
systemctl is-active nfs-backup-prune.timer
systemctl is-active nfs-backup-check.timer
systemctl show nfs-backup@backup.service \
  --property=ExecMainStartTimestamp \
  --property=ExecMainExitTimestamp \
  --property=Result \
  --property=ExecMainStatus
sudo nfs-backup-job restic snapshots \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup \
  --latest 1
```

A timer being active only means the schedule is loaded. Correlate a recent
snapshot with nonempty recent service timestamps, `Result=success`, and
`ExecMainStatus=0`. Result fields can default to success before first execution
and execution metadata can reset after reboot, so durable monitoring should
record service results and snapshot times outside transient unit state.

## Security Model

### What Is Protected

- Restic encrypts file content and metadata before upload.
- The source bind mount is read-only.
- Generated credential files are root-owned with mode `0600`.
- The runner is root-owned with mode `0700`.
- Containers use `no-new-privileges`.
- The repository password is mounted as a file, not passed as a command argument.
- Containers are removed after use.

### What Root Can Still See

Root and anyone with Docker daemon access can inspect running containers, mounts, and environment configuration. Docker access is effectively root-level access. Restrict membership of the `docker` group and access to the NFS server.

### S3 Delete Permission

Prune needs delete permission. That also means a compromised backup host could delete repository objects. For important data, add protection outside this host:

- S3 object lock or immutability where supported
- bucket versioning with protected lifecycle rules
- a second repository using different credentials
- an offline or disconnected backup copy

### Encryption-Key Handling

Keep the key in at least two secure locations. Do not store the only copy:

- only in the repository being protected
- only on the NFS server
- only in an untested Vault file
- only in one person's memory

## Changing The Schedule

Edit the `calendar` fields in `group_vars/all.yml`, then validate each expression locally:

```bash
systemd-analyze calendar '*-*-* 00:00:00 Europe/London'
systemd-analyze calendar 'Sun *-*-* 03:30:00 Europe/London'
```

Redeploy:

```bash
ansible-playbook \
  -i inventory.yml \
  deployments/nfs-backup/deploy.yml \
  --extra-vars @vault.yml \
  --ask-vault-pass
```

The role restarts a timer only when its rendered unit changed.

## Changing Retention

1. Read the [Retention Guide](docs/RETENTION.md).
2. Decide how far back accidental deletion or corruption may need to be discovered.
3. Estimate storage using actual source change rate.
4. Run `forget --dry-run` with the proposed policy.
5. Save the dry-run output for review.
6. Update `group_vars/all.yml`.
7. Redeploy.
8. Review the next prune logs.

Never test retention by manually deleting files from the S3 repository.

## Updating The Container Image

`version: "1.8.2"` is the Resticker image version. Before changing it:

1. Read Resticker release notes.
2. Identify the bundled Restic version.
3. Check Restic repository-format compatibility.
4. Test `snapshots`, `backup`, `check`, and a restore against a non-production repository if possible.
5. Change the pinned version.
6. Deploy and inspect logs.

Do not use `latest` for unattended backups.

## Moving The Backup To Another Host

The deployment intentionally supports one `nfs_servers` host. Ansible cannot disable timers on a machine removed from inventory.

Before changing the inventory host:

```bash
sudo systemctl disable --now nfs-backup-backup.timer
sudo systemctl disable --now nfs-backup-prune.timer
sudo systemctl disable --now nfs-backup-check.timer
```

Disabling a timer does not stop a service that already started. Check for an
active job and normally wait for it to finish before moving the deployment:

```bash
systemctl status 'nfs-backup@*.service'
docker ps --filter 'label=homelab.job'
sudo nfs-backup-job restic list locks
```

Then update inventory, verify the new host sees the correct source data, and
deploy. Keeping the same Restic hostname, tag, and in-container path preserves
snapshot selection and grouping.

## Temporarily Pause Scheduling

Pause all automatic jobs without deleting configuration:

```bash
sudo systemctl disable --now nfs-backup-backup.timer
sudo systemctl disable --now nfs-backup-prune.timer
sudo systemctl disable --now nfs-backup-check.timer
```

This prevents future activations but does not stop a service that is already
running. Inspect `nfs-backup@*.service` and the repository lock before assuming
all work has stopped.

Re-enable them:

```bash
sudo systemctl enable --now nfs-backup-backup.timer
sudo systemctl enable --now nfs-backup-prune.timer
sudo systemctl enable --now nfs-backup-check.timer
```

Disabling timers does not delete snapshots or the repository.

## Removing The Host-Side Scheduler

Only do this if backups have moved elsewhere or are intentionally retired.

```bash
sudo systemctl disable --now nfs-backup-backup.timer
sudo systemctl disable --now nfs-backup-prune.timer
sudo systemctl disable --now nfs-backup-check.timer
```

Before deleting any installed file, confirm that no service or container is
active and that no repository lock belongs to this host:

```bash
systemctl status 'nfs-backup@*.service'
docker ps --filter 'label=homelab.job'
sudo nfs-backup-job restic list locks
```

The following installed files may then be removed through a dedicated Ansible decommission task or careful manual administration:

- `/etc/systemd/system/nfs-backup@.service`
- `/etc/systemd/system/nfs-backup-*.timer`
- `/usr/local/sbin/nfs-backup-job`
- `/etc/nfs-backup`
- `/var/cache/nfs-backup`

Do not delete the S3 repository as part of scheduler removal. Repository deletion is a separate, destructive decision.

## Disaster Recovery Overview

If the original cluster is unavailable:

1. Obtain the offline Restic encryption key.
2. Obtain S3 repository credentials and endpoint details.
3. Prepare a Linux host with Docker and enough restore space.
4. Use the standalone recovery-container procedure in the restore runbook.
5. List snapshots and select the intended host, path, and tag.
6. Restore into a separate recovery directory.
7. Validate files before rebuilding applications.

Do not run the deployment playbook on a blank recovery host. It requires the
real backup source and can activate future backup timers. The restore runbook
deliberately avoids those deployment actions.

The detailed procedure is in the [Restore Runbook](docs/RESTORE.md).

## Frequently Asked Questions

### Why Is No Container Visible Most Of The Time?

That is the intended design. `docker run --rm` creates a container only while a job is active and removes it at completion. Inspect timers and journal logs instead of expecting a permanent container.

### Does Every Snapshot Upload All Files Again?

No. Restic scans metadata, reuses unchanged content, chunks changed files, and uploads only data not already present in the repository.

### Does One Snapshot Consume The Full Source Size?

Not usually. Snapshot metadata points to deduplicated repository data. Storage growth mostly follows unique changed content plus metadata and unused data awaiting prune.

### Why Is Repository Initialization Manual?

A wrong but writable S3 prefix looks like an empty location. Automatic initialization could make that mistake appear successful while the actual repository remains elsewhere. Requiring `restic init` is a deliberate safety decision.

### Why Does Deployment Require An Existing Snapshot?

Opening a Restic repository only proves that some repository exists. Requiring
a snapshot for host `nfs-backup`, path `/exports/docker`, and tag `nfs-backup`
provides stronger evidence that it is the expected history before scheduled
jobs are enabled.

### Why Did Deployment Stop Because A Lock Exists?

The playbook refuses to enable scheduled jobs while Restic may be active. Check running containers and processes. If no operation exists, follow the stale-lock procedure in the [Troubleshooting Guide](docs/TROUBLESHOOTING.md).

### Why Does Prune Take A Long Time?

Prune scans repository usage and may download and re-upload partially used packs before deleting old objects. Runtime depends on repository size, churn, network speed, S3 latency, and how much data expired.

### Can I Restore Directly Over Live Data?

Technically Restic can write to any mounted target, but this runner deliberately exposes only `/restore` for manual commands. Inspect and validate recovered files before copying them into production.

### Is A 10% Weekly Check Enough?

It is a cost-conscious baseline, not proof that every data pack is read within a fixed period. Add periodic full `--read-data` checks and, more importantly, test restores.

### Is S3 The Same As A Second Backup?

It is off-host storage, which is valuable, but the backup host has credentials that can delete objects for prune. Strong protection adds immutability or another independently credentialed copy.

## Further Reading

- [Restic documentation](https://restic.readthedocs.io/)
- [Restic backup command](https://restic.readthedocs.io/en/stable/040_backup.html)
- [Restic forget and prune](https://restic.readthedocs.io/en/stable/060_forget.html)
- [Restic repository checks](https://restic.readthedocs.io/en/stable/045_working_with_repos.html#checking-integrity-and-consistency)
- [systemd timer documentation](https://www.freedesktop.org/software/systemd/man/systemd.timer.html)
- [systemd calendar syntax](https://www.freedesktop.org/software/systemd/man/systemd.time.html)
- [Resticker source and documentation](https://github.com/djmaze/resticker)

## Final Operating Principle

A backup system is healthy only when all of the following are true:

- jobs run on schedule
- recent snapshots exist
- repository checks succeed
- the encryption key is available outside the failed system
- restored files can be read and validated
- application recovery has been rehearsed

The final test is always a restore.
