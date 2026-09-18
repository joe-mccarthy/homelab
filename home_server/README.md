# Home Server Bootstrap

This playbook prepares one Debian or Ubuntu host for the Home Assistant,
Traefik, Paperless, Immich, DDNS, and Omni Tools Docker Compose deployments. The
host can be an inventory alias, DNS name, or IP address.

It installs Docker Engine and the Compose plugin from Docker's official APT
repository, enables bounded container logs, adds the Ansible connection user to
the `docker` group, creates `/exports/docker` and `/opt`, creates the local
`proxy` network, logs in to every Docker registry configured in the vault, and
verifies the Docker installation. An existing attachable Swarm overlay is
preserved so Compose and legacy Swarm services can share it. The playbook stops
with migration guidance rather than replacing a non-attachable overlay.

## Prerequisites

- The target account must have sudo access.
- The target must run Debian or Ubuntu.
- A Swarm-active target must be a manager so the migration can remove legacy
  services and retain Traefik discovery for services not yet migrated.
- Install the repository collections with
  `ansible-galaxy collection install -r requirements.yml`.
- Create and encrypt `vault.yml` with a `vault.docker_registries` list based on
  [`vault.template.yml`](../vault.template.yml).

The six service deployment playbooks accept the same `-e target=...` value, so
the host does not need to belong to a particular inventory group.

## Usage

From the repository root, bootstrap an inventory host:

```bash
ansible-playbook -i inventory.yml home_server/setup.yml \
  -e target=odin --extra-vars @vault.yml \
  --ask-vault-pass --ask-become-pass
```

An inventory is optional when targeting an address directly. Supply the SSH
user with `-u` and add `--ask-pass` if password authentication is required:

```bash
ansible-playbook home_server/setup.yml \
  -e target=192.168.1.50 -u pi --ask-pass \
  --extra-vars @vault.yml --ask-vault-pass --ask-become-pass
```

The first play adds the address to an in-memory inventory for the duration of
the run. Reconnect afterward if the connection user needs its new Docker group
membership in an interactive shell.

The bootstrap and service roles default to `/exports/docker`, `/opt`, and the
`proxy` network. Change the corresponding service variables as well if these
paths or the network name need to differ.

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

After configuring the required values in the Ansible Vault, deploy services in
dependency order:

```bash
ansible-playbook -i inventory.yml deployments/traefik/deploy.yml -e target=odin \
  --extra-vars @vault.yml --ask-vault-pass
ansible-playbook -i inventory.yml deployments/ddns/deploy.yml -e target=odin \
  --extra-vars @vault.yml --ask-vault-pass
ansible-playbook -i inventory.yml deployments/omni/deploy.yml -e target=odin \
  --extra-vars @vault.yml --ask-vault-pass
ansible-playbook -i inventory.yml deployments/home-assistant/deploy.yml -e target=odin \
  --extra-vars @vault.yml --ask-vault-pass
ansible-playbook -i inventory.yml deployments/paperless/deploy.yml \
  -e target=odin -e paperless_allow_fresh_install=true \
  --extra-vars @vault.yml --ask-vault-pass
ansible-playbook -i inventory.yml deployments/immich/deploy.yml \
  -e target=odin -e immich_allow_fresh_install=true \
  --extra-vars @vault.yml --ask-vault-pass
```

The Paperless and Immich commands above intentionally initialize empty data
directories. For subsequent deployments or restored installations, omit the
corresponding `allow_fresh_install` variable to use the existing database and
application data.
