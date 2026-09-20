# Troubleshooting Guide

This guide provides a safe diagnostic sequence for the NFS backup deployment.

The backup runs on local application data on the single `nfs_servers` inventory
host. The name is retained by the playbook and systemd units; an NFS export is
not a prerequisite. Run host commands below on that machine. Deployment commands
run from the repository root on the Ansible controller.

## First Principle: No Container Is Normally Running

Between jobs, this is expected:

```bash
sudo docker ps --quiet --filter 'label=homelab.job'
```

Expected output: no matching containers.

The scheduler is systemd, not a permanent Docker container. Diagnose timers, service results, and journal logs rather than treating the absence of a container as a failure.

## Safety Rules

Before troubleshooting:

- do not run `restic init` against an existing repository problem
- do not delete S3 objects manually
- do not bypass a Restic lock with `--no-lock`
- do not use `unlock --remove-all` without understanding every client
- do not run repair commands merely to see whether they help
- do not expose `/etc/nfs-backup` contents in logs or support requests
- do not restore directly over `/exports/docker` during diagnosis

Start with read-only inspection.

## Quick Diagnostic Sequence

For an overview from the controller, run the [host status playbook](../../../home_server/status.yml):

```bash
ansible-playbook home_server/status.yml \
  -e target=192.168.1.50 -u pi --ask-become-pass
```

For backup-specific diagnosis, run on the backup host:

```bash
systemctl list-timers --all 'nfs-backup-*'
systemctl status nfs-backup-backup.timer
systemctl status nfs-backup@backup.service
sudo journalctl -u nfs-backup@backup.service -n 200 --no-pager
sudo docker ps --filter 'label=homelab.job'
sudo nfs-backup-job restic snapshots \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup \
  --latest 1
```

These answer six different questions:

1. Are timers loaded?
2. Is the backup timer active with a next run?
3. What was the latest service result?
4. What did Restic report?
5. Is a job currently running?
6. Does a recent snapshot exist?

## Understand The Three States

### Timer State

The timer decides when to start work.

```bash
systemctl status nfs-backup-backup.timer
```

Healthy timer state is usually `active (waiting)`.

### Service State

The service process runs only while a job is active; its unit definition stays installed. A successful oneshot service usually returns to `inactive (dead)` after completion.

```bash
systemctl status nfs-backup@backup.service
```

`inactive (dead)` can mean success. Read the result and logs.

### Snapshot State

A snapshot confirms that Restic recorded a restore point, but Restic exit status
3 can still create an incomplete snapshot after source read errors. Confirm
both snapshot presence and a successful service result.

```bash
sudo nfs-backup-job restic snapshots \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup \
  --latest 5
```

## Timer Is Missing

Symptoms:

- `Unit nfs-backup-backup.timer could not be found`
- no NFS backup timers appear in `systemctl list-timers`

Check installed files:

```bash
ls -l /etc/systemd/system/nfs-backup*.timer
ls -l /etc/systemd/system/nfs-backup@.service
```

Reload systemd and verify units:

```bash
sudo systemctl daemon-reload
sudo systemd-analyze verify \
  /etc/systemd/system/nfs-backup@.service \
  /etc/systemd/system/nfs-backup-backup.timer \
  /etc/systemd/system/nfs-backup-prune.timer \
  /etc/systemd/system/nfs-backup-check.timer
```

If files are absent, rerun the Ansible deployment from the repository root.

## Timer Is Disabled Or Inactive

Inspect:

```bash
systemctl is-enabled nfs-backup-backup.timer
systemctl is-active nfs-backup-backup.timer
```

Enable and start:

```bash
sudo systemctl enable --now nfs-backup-backup.timer
```

Repeat for prune and check if required.

If Ansible deployment failed, inspect its recovery output. Before scheduler
activation, the playbook restores the managed files and prior timer enablement
state. If deployment cleanup fails after activation, it keeps the validated
timers active to preserve backup coverage. Fix the original error and rerun the
playbook to finish deployment cleanup.

## Timer Has No Next Run

Inspect the rendered calendar:

```bash
systemctl cat nfs-backup-backup.timer
systemctl list-timers --all nfs-backup-backup.timer
```

Validate the expression from `group_vars/all.yml`:

```bash
systemd-analyze calendar '*-*-* 00:00:00 Europe/London'
```

Common causes:

- invalid systemd calendar syntax
- an invalid timezone name
- the timer is not started
- systemd has not reloaded after a file change

## The Host Was Off At Midnight

With `persistent_timers: false`, the missed backup does not run after boot. This is configured behavior, not a timer fault.

Options:

- wait for the next midnight
- run a manual backup now
- change `persistent_timers` to `true` and redeploy if catch-up behavior is desired

Manual backup:

```bash
sudo systemctl start nfs-backup@backup.service
```

## Service Failed

Inspect status and logs:

```bash
systemctl status nfs-backup@backup.service
sudo journalctl -u nfs-backup@backup.service -n 300 --no-pager
```

Then run the wrapper's read-only snapshot command:

```bash
sudo nfs-backup-job restic snapshots \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup \
  --latest 1
```

This separates repository access failures from source-scanning failures.

## Docker Is Not Running

Symptoms:

- `Cannot connect to the Docker daemon`
- systemd reports `docker.service` dependency failure

Check:

```bash
systemctl status docker
sudo docker version
```

Start Docker:

```bash
sudo systemctl enable --now docker
```

Investigate Docker logs if it does not start:

```bash
sudo journalctl -u docker -n 300 --no-pager
```

## Docker Image Cannot Be Pulled

Symptoms:

- registry timeout
- manifest not found
- no matching platform
- rate-limit error

Check the configured Resticker version in `group_vars/all.yml` and test the pull:

```bash
sudo docker pull mazzolino/restic:1.8.2
```

Possible causes:

- DNS failure
- blocked outbound HTTPS
- incorrect image tag
- unsupported CPU architecture
- Docker Hub rate limits
- registry outage

Do not change to `latest` merely to bypass an unknown tag problem. Confirm the intended release.

## Backup Source Is Missing

The Ansible role deliberately fails if `/exports/docker` is absent or not a directory.

Check:

```bash
sudo stat /exports/docker
sudo ls -la /exports/docker
```

Possible causes:

- application-data preparation is incomplete
- storage is not mounted
- path changed
- permissions prevent traversal
- wrong inventory host was selected

Do not create an empty `/exports/docker` only to satisfy the check. Confirm that it contains the intended live data.

## Repository Does Not Exist

Typical Restic message:

```text
Fatal: repository does not exist
```

Possible explanations:

- wrong bucket
- wrong prefix
- wrong endpoint
- wrong region
- access key cannot see the repository config object
- repository genuinely has not been initialized

For an existing deployment, compare the configured URL with the known provider location. Do not run `init` until you are certain the location is intentionally new.

For a genuinely new repository, follow the [initialization procedure](../README.md#initialize-a-brand-new-repository).

## Wrong Password

Typical messages include:

- `wrong password or no key found`
- exit status 12

The Restic encryption password is independent of the S3 access key.

Check:

- the correct Vault file was loaded
- whitespace was not added or removed
- the repository URL points to the expected repository
- the encryption key matches the key used at repository initialization

There is no password reset that can decrypt a repository without an existing valid key.

## S3 Authentication Or Permission Failure

Typical messages include:

- access denied
- invalid access key
- signature mismatch
- forbidden
- authorization-header error

Check:

- access key ID
- secret access key
- endpoint URL
- region
- system clock
- bucket policy
- provider-specific S3 compatibility settings

Normal Restic S3 operations need to create and remove lock objects as well as
list, read, and write repository objects. Prune also deletes expired snapshot
and data objects. This runner uses the same credential set for every job.

If backup succeeds but prune fails with access denied, check deletion permissions
for snapshot/data objects and any provider object-lock policy.

## TLS Or Certificate Failure

Symptoms:

- certificate signed by unknown authority
- certificate hostname mismatch
- TLS handshake timeout

Check:

- endpoint hostname
- DNS resolution
- reverse-proxy certificate chain
- system date and time
- provider status

Restic makes the TLS connection from inside the temporary container. Installing
a private CA only in the host trust store does not make that CA available in
the image. The current runner has no custom-CA mount; use an endpoint with a
publicly trusted certificate or extend and test the deployment with a read-only
CA mount and `RESTIC_CACERT`.

Do not disable TLS verification as a permanent fix. Correct container trust or
endpoint configuration.

## Network Timeout Or DNS Failure

Check name resolution and HTTPS reachability using approved host networking tools.

Also inspect:

```bash
resolvectl status
timedatectl status
```

Possible causes:

- DNS outage
- firewall rule
- incorrect S3 endpoint
- provider outage
- IPv6 routing problem
- proxy configuration
- severe packet loss

Restic retries backend requests, but persistent network failure eventually fails the job.

## Repository Is Locked

List locks:

```bash
sudo nfs-backup-job restic list locks
```

Check local activity:

```bash
sudo docker ps --filter 'label=homelab.job'
systemctl status 'nfs-backup@*.service'
```

If a job is active, wait. Do not remove its lock.

If no Restic process exists and the lock came from an interrupted operation:

```bash
sudo nfs-backup-job restic unlock
```

Then verify:

```bash
sudo nfs-backup-job restic list locks
sudo nfs-backup-job restic check
```

The deployment checks locks before enabling timers and never unlocks
automatically. If a stale lock remains, investigate it, run `unlock` only when
safe, and rerun the playbook to activate timers.

## Deployment Says Expected Backup History Is Missing

The playbook requires an existing snapshot with:

```text
host: nfs-backup
path: /exports/docker
tag: nfs-backup
```

Inspect:

```bash
sudo nfs-backup-job restic snapshots
```

Possible causes:

- wrong repository URL
- wrong repository password
- repository is new and empty
- historical snapshots used another hostname
- historical snapshots used another source path
- retention already removed the expected history

Do not weaken the assertion until you understand which case applies. For a brand-new repository, initialize it and create the first backup as documented.

## Deployment Requires One Backup Host

The play requires:

- exactly one inventory host in `nfs_servers`

Run the complete playbook with that host in scope, using your own inventory as
shown in the [deployment guide](../README.md#host-and-inventory):

```bash
ansible-playbook \
  -i inventory.yml \
  deployments/nfs-backup/deploy.yml \
  --extra-vars @vault.yml \
  --ask-vault-pass --ask-become-pass
```

The assertion checks the full `nfs_servers` group, so limiting a multi-host group
to one host does not satisfy it. If a limit excludes the backup host completely,
Ansible skips the play and no deployment occurs.

## Backup Completed With Exit Status 3

Restic exit status 3 means some source files could not be read, although an incomplete snapshot may have been created.

Inspect logs for exact paths:

```bash
sudo journalctl -u nfs-backup@backup.service -n 500 --no-pager
```

Common causes:

- permission denied
- file disappeared while being scanned
- transient I/O error
- broken symlink is usually not itself an error, but inaccessible targets may affect application expectations
- filesystem or storage error

Do not treat an incomplete snapshot as equivalent to a complete backup. Correct the source issue and run another backup.

## Backup Is Slow

The first backup is expected to be slow because all unique data must be read, chunked, compressed, encrypted, and uploaded.

Later backups can still be slow because of:

- many files to scan
- unstable inode or metadata values
- large frequently changing database files
- maximum compression
- low CPU performance
- slow source storage
- slow S3 upload
- missing or cold Restic cache
- provider throttling

Inspect Restic summary output and compare scan time with upload time.

## Prune Is Slow

Prune may process the entire repository and repack partially used data. It can take much longer than a normal incremental backup.

Check:

```bash
systemctl status nfs-backup@prune.service
sudo journalctl -fu nfs-backup@prune.service
```

Do not interrupt prune without a clear reason. Restic is designed to preserve repository consistency when interrupted, but future maintenance may be required and a stale lock may remain.

## Prune Does Not Reduce Provider Storage

Possible causes:

- no snapshots expired, so `forget --prune` did not run prune
- retained snapshots still reference the data
- bucket versioning retains deleted object versions
- provider usage metrics update slowly
- partially used packs remain within Restic's allowed unused-data threshold
- another repository or prefix contributes to bucket usage

Review prune logs and provider versioning/lifecycle configuration.

## Check Uses Significant Bandwidth

The scheduled check reads a random 10% of repository data packs in addition to metadata.

For a large repository, 10% can still be substantial. Lower `check_read_data_subset` only after considering integrity-detection goals.

Do not remove data checks entirely without adding another method of detecting stored-data corruption.

## Check Reports Repository Errors

Stop normal maintenance decisions and preserve logs.

Run the default check again only if the original failure may have been a transient network problem:

```bash
sudo nfs-backup-job restic check
```

Do not immediately run repair commands. Identify the error category:

- damaged index
- missing pack
- damaged pack
- invalid snapshot tree
- backend request failure

Read Restic's exact recommendation and official troubleshooting documentation. Test repair actions on a copied repository where practical.

## Cache Permission Failure

Check:

```bash
sudo ls -ld /var/cache/nfs-backup
sudo ls -la /var/cache/nfs-backup
```

Expected directory owner and mode:

```text
root root 0700
```

Rerunning Ansible restores the top-level directory ownership and mode.

Do not make the cache world-writable.

## Restore Fails With No Space Left

Check:

```bash
sudo df -h /restore
sudo du -h --max-depth=1 /restore
```

Use a larger host destination configured through `restore_path`, redeploy, and
retry. That host path is mounted at `/restore` inside the container, so Restic
commands continue to use `/restore`. Partial restored data can be cleaned after
confirming the exact host target.

Repository snapshots remain unchanged by a failed restore.

## Logs Are Missing

Check whether the service ever ran:

```bash
systemctl show nfs-backup@backup.service \
  --property=ActiveEnterTimestamp \
  --property=InactiveEnterTimestamp \
  --property=Result
```

Check journal retention and storage:

```bash
sudo journalctl --disk-usage
systemctl status systemd-journald
```

A newly deployed timer has no service logs until its first activation or manual run.

## Host Time Is Wrong

Incorrect time affects:

- timer execution
- snapshot timestamps
- S3 request signatures
- retention calendar selection
- TLS certificate validation

Inspect:

```bash
timedatectl status
```

Correct time synchronization before backup or prune.

## Duplicate Backups Appear

Possible causes:

- timers remain enabled on a former backup host
- manual and scheduled backups were both run
- another host uses the same fixed Restic hostname

Inspect known backup hosts. Before moving the inventory entry to a new host, disable timers on the old one and let any active job finish, as described in [Moving The Backup To Another Host](../README.md#moving-the-backup-to-another-host).

## Collect A Safe Diagnostic Summary

The following commands avoid printing the protected environment file or password:

```bash
systemctl list-timers --all 'nfs-backup-*'
systemctl status nfs-backup-backup.timer --no-pager
systemctl status nfs-backup@backup.service --no-pager
sudo journalctl -u nfs-backup@backup.service -n 200 --no-pager
sudo docker version
sudo docker ps --filter 'label=homelab.job'
sudo nfs-backup-job restic version
sudo nfs-backup-job restic snapshots \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup \
  --latest 3
sudo nfs-backup-job restic list locks
sudo df -h /exports/docker /restore /var/cache/nfs-backup
timedatectl status
```

Before sharing output, review it for:

- bucket names
- endpoint hostnames
- internal hostnames and IP addresses
- filenames containing sensitive information
- application names

Never include:

- `/etc/nfs-backup/password`
- `/etc/nfs-backup/restic.env`
- Vault plaintext
- AWS access keys
- the Restic encryption key

## Repair Escalation

Use this order:

1. Preserve complete logs.
2. Confirm network and provider health.
3. Confirm the repository is not in active use.
4. Run non-data-reading `restic check`.
5. Read official Restic guidance for the exact error.
6. Protect or copy the repository before risky repair where possible.
7. Use the narrowest recommended repair command.
8. Run check again.
9. Perform a restore test.

Commands such as `repair index`, `repair packs`, and advanced prune recovery exist for specific failure modes. They are not general maintenance commands.

## When To Escalate Immediately

Stop and seek experienced review when:

- the configured repository unexpectedly appears empty
- the encryption password no longer opens known snapshots
- check reports missing or damaged packs
- S3 objects were manually altered
- multiple schedulers wrote concurrently
- a repair command was interrupted
- repository size or snapshot history changed unexpectedly
- all recent snapshots are incomplete
- a required restore fails cryptographic verification

Preserve evidence and avoid making the only repository harder to recover.
