# Retention Guide

This guide explains how Restic chooses snapshots to keep, how pruning affects storage, and how to select a policy for this homelab.

## Retention In Plain English

Backups accumulate snapshots. Retention answers:

> How many recovery points should remain available, and how far back should they reach?

The policy should balance:

- how quickly accidental deletion is noticed
- how quickly corruption is noticed
- how often source data changes
- how much storage and S3 bandwidth are affordable
- legal or personal history requirements
- how many independent backup copies exist

Retention is a recovery decision, not only a storage decision.

## Snapshot Does Not Mean Full Copy

Each Restic snapshot presents a complete filesystem view, but Restic stores file content in deduplicated chunks.

If a 100 GB source changes by 500 MB between daily backups, the next snapshot does not normally add another 100 GB. It mostly adds new chunks for changed content plus metadata.

Storage growth is influenced by:

- initial unique source size
- daily unique changed content
- large files that change internally
- database and log churn
- compression effectiveness
- retention duration
- unused data waiting for prune
- repository pack-repacking behavior

Snapshot count alone cannot predict repository size.

## Current Policy

The deployment runs one scheduled backup per day and applies:

```text
keep every snapshot within 1 day of the newest snapshot
keep 14 daily snapshots
keep 8 weekly snapshots
keep 12 monthly snapshots
keep 3 yearly snapshots
```

Equivalent Restic options:

```bash
--keep-within 1d
--keep-daily 14
--keep-weekly 8
--keep-monthly 12
--keep-yearly 3
```

This is the recommended balanced homelab policy.

## Rules Are Combined With OR

Restic keeps a snapshot if it matches any policy rule.

Suppose the latest snapshot is also:

- today's daily snapshot
- this week's weekly snapshot
- this month's monthly snapshot
- this year's yearly snapshot

That is one retained snapshot with four reasons, not four separate snapshots.

For this reason, adding `14 + 8 + 12 + 3` overestimates the retained count.

## How Calendar Rules Select Snapshots

Calendar rules retain the most recent snapshot in each natural calendar period that contains at least one snapshot.

Examples:

- an hour runs from minute `00` through `59`
- a day runs from `00:00` through `23:59`
- a week runs Monday through Sunday
- a month follows the calendar month
- a year follows the calendar year

`--keep-daily 14` means 14 days that have snapshots. It does not necessarily mean only snapshots from the last 14 elapsed calendar days.

If the host was off for a week, Restic still keeps 14 available backup days, potentially spanning more than 14 calendar days.

## How `keep-within` Works

`--keep-within 1d` retains every snapshot whose timestamp is within one day of the newest snapshot in the group.

It is relative to the newest snapshot, not the wall clock when prune runs.

This protects dense recent history. If other clients have produced multiple snapshots per day, the rule preserves roughly the latest 24 hours of that history while older snapshots are reduced to daily, weekly, monthly, and yearly representatives.

After the system settles at one snapshot per day, `keep-within 1d` usually adds little beyond the daily policy.

## Snapshot Grouping

The prune job uses:

```bash
--host nfs-backup
--path /exports/docker
--tag nfs-backup
--group-by host,paths
```

The host, path, and tag filters select only snapshots created by this backup
job. A hostname alone is not sufficient because another backup set could use
the same generic `nfs-backup` hostname in a shared repository.

A dedicated repository or S3 prefix remains the simplest and safest design.
Filters reduce accidental cross-policy deletion, but separate repositories
also isolate locks, checks, maintenance duration, and credentials.

Grouping by host and paths prevents snapshots of different source paths from competing for the same daily or weekly slot.

The backup job deliberately uses stable values:

```text
host: nfs-backup
path: /exports/docker
tag: nfs-backup
```

These are fixed values inside the temporary container. Changing the host-side
`source_path` bind-mount source does not change the path recorded by Restic; it
still appears as `/exports/docker`.

Changing the fixed hostname, in-container path, or tag in the implementation
requires a retention review. Old snapshots are not automatically
merged into a new group or selected by new filters.

## Forget And Prune Are Different

### Forget

`restic forget` removes snapshot records that the policy no longer keeps.

After forget, data chunks referenced only by removed snapshots become unreferenced, but they may still occupy object storage.

### Prune

`restic prune` analyzes repository packs, deletes fully unused packs, and may repack partially used packs.

Prune can be expensive because it may:

- list large numbers of S3 objects
- download pack data
- upload replacement packs
- rebuild indexes
- delete old objects
- hold an exclusive repository lock

The scheduled command uses `forget --prune`. Restic invokes prune when snapshots were forgotten. If no snapshot expires, the job exits without an unnecessary full prune.

## Suggested Profiles

All profiles assume one backup per day and retain all snapshots from the latest day.

| Profile | Daily | Weekly | Monthly | Yearly | Suggested use |
| --- | ---: | ---: | ---: | ---: | --- |
| Lean | 7 | 4 | 6 | 1 | Re-creatable data, short detection windows, or strict storage limits. |
| Balanced | 14 | 8 | 12 | 3 | General homelab data and the recommended default. |
| Conservative | 30 | 12 | 24 | 7 | Important personal data or problems that may remain unnoticed for months. |

### Lean

```yaml
retention:
  keep_within: 1d
  keep_daily: 7
  keep_weekly: 4
  keep_monthly: 6
  keep_yearly: 1
```

Advantages:

- lower long-term repository growth
- shorter prune analysis set

Risks:

- a deletion noticed after seven backup days may have only weekly recovery points
- a bad configuration propagated for months may have limited monthly history

### Balanced

```yaml
retention:
  keep_within: 1d
  keep_daily: 14
  keep_weekly: 8
  keep_monthly: 12
  keep_yearly: 3
```

Advantages:

- two weeks of daily choices
- roughly two months of weekly choices
- one year of monthly choices
- several annual reference points

This is suitable when data matters but storage is not unlimited.

### Conservative

```yaml
retention:
  keep_within: 1d
  keep_daily: 30
  keep_weekly: 12
  keep_monthly: 24
  keep_yearly: 7
```

Advantages:

- more time to discover slow corruption or accidental deletion
- two years of monthly choices
- long annual history

Risks:

- higher storage usage
- more metadata and a potentially larger prune working set
- old sensitive data remains recoverable longer

## Choosing A Policy

Ask these questions:

1. How long could a deletion remain unnoticed?
2. How long could silent corruption remain unnoticed?
3. How frequently does the source change?
4. Which data is irreplaceable?
5. Is there another independent backup?
6. Does the S3 provider charge for reads, writes, deletes, or egress?
7. Is old personal or sensitive data supposed to expire?
8. How often are restore drills performed?

Practical guidance:

- Keep daily points longer than the normal time needed to notice a problem.
- Keep monthly points for slow-moving configuration and personal-data mistakes.
- Keep yearly points only when old state has real recovery or historical value.
- Prefer reducing very old history before reducing the recent recovery window.
- Keep more history for irreplaceable photos, documents, and configuration.
- Keep less history for caches, generated artifacts, and easily recreated data.

Ideally, exclude disposable data from the backup source instead of relying on aggressive retention to control its cost.

## Estimate Storage From Change Rate

A rough planning model is:

```text
repository size approximately equals
  compressed unique initial data
  + compressed unique changes retained over time
  + metadata
  + temporary unused/repacked data
```

Example only:

```text
Initial unique data:       500 GB
Average unique changes:      2 GB/day
Effective retained change window: approximately 180 days
Change history:            360 GB before compression/dedup overlap
```

Real results vary significantly. Databases, virtual-disk images, encrypted files, and compressed archives can cause more churn than ordinary documents.

Estimate unique blob data referenced by repository snapshots:

```bash
sudo nfs-backup-job restic stats --mode raw-data
```

This is not necessarily physical or billed S3 usage. Unreferenced packs,
repository overhead, provider object versions, and delayed provider metrics can
make storage-provider usage larger. Use the provider's metrics for physical
usage.

Inspect a snapshot's logical restore size:

```bash
sudo nfs-backup-job restic stats latest \
  --host nfs-backup \
  --path /exports/docker \
  --tag nfs-backup
```

The logical restore size and raw repository data answer different questions and should not be expected to match.

## Always Preview Policy Changes

Use `--dry-run` before reducing retention:

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

Review:

- the selected snapshots have host `nfs-backup`, path `/exports/docker`, and
  tag `nfs-backup`
- recent snapshots have expected reasons
- monthly and yearly points reach as far back as intended
- snapshots marked for removal are acceptable
- no unrelated backup group is included

Save the dry-run output before a major policy reduction.

`--dry-run` does not delete snapshots or repository data.

## Existing Hourly Snapshots

If the repository already contains hourly snapshots, this deployment creates one snapshot per day going forward. At the first prune:

- hourly snapshots within one day of the newest snapshot are kept
- older hourly snapshots usually collapse to one daily representative
- weekly, monthly, and yearly representatives are retained
- expired snapshot records are forgotten
- data unique to expired snapshots becomes eligible for prune

If hourly recovery points older than one day are important, change `keep_within` or add an intentional transitional rule before the first prune.

Do not discover this requirement after prune has deleted unreferenced data.

## S3 Versioning And Object Lock

Restic expects exclusive control over its repository objects. Storage-side protection can help recover from accidental or malicious deletion, but it changes costs and operations.

### Bucket Versioning

Versioning may preserve deleted or replaced object versions. It can also cause provider storage usage to remain high after Restic prune because old object versions still exist.

Plan lifecycle rules carefully and make sure they cannot remove active Restic data prematurely.

### Object Lock Or Immutability

Immutability can protect against ransomware and compromised deletion credentials. However, normal Restic prune needs to delete expired objects. A locked repository may produce repeated prune failures until retention periods expire.

Common safer designs include:

- a writable primary repository plus an independently protected second copy
- storage credentials that can append but not delete for regular backup clients
- separate administrative credentials for maintenance
- provider snapshots or replication outside the backup host's credentials

Test the exact provider behavior with a non-production repository.

## Retention Is Not Backup Independence

Keeping years of snapshots in one repository protects against time-based mistakes, but not every threat.

One repository can still be affected by:

- lost encryption key
- compromised delete credentials
- provider account loss
- billing or quota issues
- repository-wide corruption
- operator error

For irreplaceable data, apply a 3-2-1 style strategy:

- at least three data copies
- on at least two storage types or administrative domains
- with at least one copy off-site or otherwise isolated

## Applying A New Policy

1. Edit `retention` in `group_vars/all.yml`.
2. Run the proposed policy with `--dry-run`.
3. Review the output carefully.
4. Run Ansible from the repository root.
5. Allow the next Sunday prune or run it manually.
6. Review prune logs and repository statistics.

Deploy:

```bash
ansible-playbook \
  -i inventory.yml \
  deployments/nfs-backup/deploy.yml \
  --extra-vars @vault.yml \
  --ask-vault-pass
```

Run retention manually after approval:

```bash
sudo systemctl start nfs-backup@prune.service
```

Inspect logs:

```bash
journalctl -u nfs-backup@prune.service -n 300 --no-pager
```

## When Prune Fails

Do not manually delete repository pack, index, snapshot, or key objects.

Check:

- repository lock status
- S3 delete permission
- S3 quota and available space
- network stability
- provider request throttling
- prune log output

Commands:

```bash
sudo nfs-backup-job restic list locks
journalctl -u nfs-backup@prune.service -n 300 --no-pager
sudo nfs-backup-job restic check
```

See the [Troubleshooting Guide](TROUBLESHOOTING.md) before using advanced or unsafe Restic repair options.

## Recommended Default

Keep the balanced profile unless actual storage measurements, recovery requirements, or provider costs justify a change:

```yaml
retention:
  keep_within: 1d
  keep_daily: 14
  keep_weekly: 8
  keep_monthly: 12
  keep_yearly: 3
```

Review the policy after observing at least one month of repository growth and after every significant change in source workload.
