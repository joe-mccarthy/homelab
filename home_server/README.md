# Home Server

Target-based Ansible playbooks for preparing and operating one Debian or Ubuntu
Docker host. The host can be an inventory alias, DNS name, or IP address.

`setup.yml` installs Docker Engine and the Compose plugin from Docker's official APT
repository, enables bounded container logs, adds the Ansible connection user to
the `docker` group, creates `/exports/docker` and `/opt`, creates the local
`proxy` network, logs in to every Docker registry configured in the vault, and
verifies the Docker installation. The `proxy` network is created as a local
bridge when absent; an existing network is validated as a local bridge before
deployment continues.

## Playbooks

| Playbook | Purpose |
| --- | --- |
| [`setup.yml`](setup.yml) | Install and configure Docker, storage directories, registry access, and the proxy bridge. |
| [`update.yml`](update.yml) | Upgrade APT packages and report the OS reboot flag. |
| [`status.yml`](status.yml) | Report host resources, Docker versions, container health, filesystem usage, and backup timers. |
| [`reboot.yml`](reboot.yml) | Reboot the selected host and wait for it to return. |
| [`storage.yml`](storage.yml) | Mount an existing application-data filesystem by UUID and persist the mount in `/etc/fstab`. |

## Prerequisites

- The target account must have SSH and sudo access.
- The target must run Debian or Ubuntu.
- Install the repository collections with
  `ansible-galaxy collection install -r requirements.yml`.
- For `setup.yml`, create and encrypt `vault.yml` with a `vault.docker_registries` list based on
  [`vault.template.yml`](../vault.template.yml).
- The operational playbooks use the Python runtime installed by `setup.yml`.

All home-server playbooks and the six application deployment playbooks accept
`-e target=...`. Each command selects a single host.

## Command Conventions

Run commands from the repository root. Replace `192.168.1.50` and `pi` in the
examples with the host address and SSH user. Add `--ask-pass` for SSH password
authentication, and omit `--ask-become-pass` when sudo is passwordless.

To use an inventory alias, add `-i inventory.yml` for your own inventory and use
that alias as `target`. The inventory can supply the SSH user and connection settings.

## Bootstrap

```bash
ansible-playbook home_server/setup.yml \
  -e target=192.168.1.50 -u pi \
  --extra-vars @vault.yml --ask-vault-pass --ask-become-pass
```

The first play adds the address to an in-memory inventory for the duration of
the run. Reconnect afterward if the connection user needs its new Docker group
membership in an interactive shell.

The bootstrap and service roles default to `/exports/docker`, `/opt`, and the
`proxy` network. Change the corresponding service variables as well if these
paths or the network name need to differ. Changing a bootstrap variable alone
does not change the paths in the individual deployment variable files.

Each registry entry must contain non-empty credentials:

```yaml
vault:
  docker_registries:
    - registry_url: ghcr.io
      username: example-user
      password: example-access-token
```

Docker stores these credentials in root's Docker configuration because the
deployment playbooks perform image pulls with privilege escalation.

For public image pulls without registry authentication, set
`vault.docker_registries: []`. `setup.yml` still requires the key to exist.
The update, status, reboot, and storage playbooks do not require vault values.

## Update Packages

```bash
ansible-playbook home_server/update.yml \
  -e target=192.168.1.50 -u pi --ask-become-pass
```

The playbook refreshes APT metadata, performs a distribution package upgrade,
and reports `packages_changed` and `reboot_required`. The latter reflects the
OS-provided `/var/run/reboot-required` flag. Use `reboot.yml` when a restart is
needed.

Package upgrades use the configured APT repositories, including Docker's
repository. Container image versions are managed through each application's
deployment variables and playbook.

Add `--check` to preview package changes. In check mode, the reboot flag reports
the host's current state rather than predicting the result of an upgrade.

## Check Status

```bash
ansible-playbook home_server/status.yml \
  -e target=192.168.1.50 -u pi --ask-become-pass
```

The read-only report includes:

- OS version, uptime, memory usage, and the OS reboot flag.
- Docker service state and installed Docker/Compose versions.
- The container count and running/stopped states, with health details when a container has a health check.
- Filesystem usage for `/`, `/exports/docker`, and `/opt`.
- Load, enablement, activity, last-trigger, and next-trigger information for the
  NFS Backup `backup`, `prune`, and `check` timers.

Unavailable Docker tools, missing storage paths, and absent backup timers appear
in the report. Diagnostic commands also run when `--check` is supplied.

## Reboot

```bash
ansible-playbook home_server/reboot.yml \
  -e target=192.168.1.50 -u pi --ask-become-pass
```

Ansible verifies the host has rebooted and waits for the SSH connection to return.
The reboot module's timeout defaults to 600 seconds and can be changed with
`-e home_server_reboot_timeout=900`.

## Mount Application Storage (Optional)

Use this playbook when application data should live on a separately mounted
NVMe/SSD filesystem. Run it after bootstrapping and before deploying applications.
Use an already-formatted ext2, ext3, ext4, XFS, Btrfs, or F2FS filesystem, and obtain
its filesystem UUID on the server with:

```bash
lsblk -f
```

Replace the sample UUID below with the filesystem's UUID:

```bash
ansible-playbook home_server/storage.yml \
  -e target=192.168.1.50 -u pi --ask-become-pass \
  -e home_server_storage_uuid=01234567-89ab-cdef-0123-456789abcdef
```

The playbook detects the filesystem type, mounts it at `/exports/docker`, writes
the UUID-based entry to `/etc/fstab`, and verifies the mounted UUID. An unmounted
target directory must be readable and empty. The filesystem may be unmounted or
mounted only at the requested path, and an existing mount at that path must use
the requested UUID. The root filesystem `/` is not an application mount target.

Set `home_server_storage_mount_options` for filesystem-specific options such as
`defaults,noatime` or a Btrfs subvolume. If you change `home_server_data_root`,
update the application data paths and backup source configuration accordingly.

## Operational Variables

| Variable | Default | Used by |
| --- | --- | --- |
| `home_server_update_cache_valid_time` | `3600` seconds | Update: maximum APT cache age. |
| `home_server_reboot_timeout` | `600` seconds | Reboot: reboot module timeout. |
| `home_server_data_root` | `/exports/docker` | Setup, status, and storage. |
| `home_server_compose_root` | `/opt` | Setup and status. |
| `home_server_status_paths` | `/`, data root, Compose root | Status: paths passed to `df`. |
| `home_server_storage_uuid` | Required | Storage: existing filesystem UUID. |
| `home_server_storage_mount_options` | `defaults` | Storage: mount options recorded in `/etc/fstab`. |
| `home_server_proxy_network` | `proxy` | Setup: name of the external local bridge. |

## Deploy Applications

After configuring the required values in the Ansible Vault, deploy Traefik
before the web applications. The commands below target the same Docker host:

```bash
ansible-playbook deployments/traefik/deploy.yml \
  -e target=192.168.1.50 -u pi --extra-vars @vault.yml --ask-vault-pass --ask-become-pass
ansible-playbook deployments/ddns/deploy.yml \
  -e target=192.168.1.50 -u pi --extra-vars @vault.yml --ask-vault-pass --ask-become-pass
ansible-playbook deployments/omni/deploy.yml \
  -e target=192.168.1.50 -u pi --extra-vars @vault.yml --ask-vault-pass --ask-become-pass
ansible-playbook deployments/home-assistant/deploy.yml \
  -e target=192.168.1.50 -u pi --extra-vars @vault.yml --ask-vault-pass --ask-become-pass
ansible-playbook deployments/paperless/deploy.yml \
  -e target=192.168.1.50 -u pi --extra-vars @vault.yml --ask-vault-pass --ask-become-pass
ansible-playbook deployments/immich/deploy.yml \
  -e target=192.168.1.50 -u pi --extra-vars @vault.yml --ask-vault-pass --ask-become-pass
```

The commands above are routine deployments. For a new Paperless installation,
add `-e paperless_allow_fresh_install=true` to its first run. For a new Immich
installation, add `-e immich_allow_fresh_install=true` to its first run. Later
runs validate the existing database and application data.

NFS Backup has its own deployment and requires one host in the `nfs_servers`
inventory group. Follow its [host and inventory](../deployments/nfs-backup/README.md#host-and-inventory)
and [repository initialization](../deployments/nfs-backup/README.md#initialize-a-brand-new-repository)
instructions to configure the scheduled jobs.
