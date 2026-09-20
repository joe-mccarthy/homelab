# Homebox Deployment

[![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white&style=flat-square)](https://docs.ansible.com/) [![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/compose/) [![Traefik](https://img.shields.io/badge/Traefik-HTTPS-24A1C1?logo=traefikproxy&logoColor=white&style=flat-square)](https://doc.traefik.io/traefik/) ![Homebox](https://img.shields.io/badge/Homebox-0.26.2-5C5C5C?style=flat-square)

An Ansible-managed, single-host Docker Compose deployment of Homebox with PostgreSQL.

## Services

| Service | Image | Purpose |
| --- | --- | --- |
| `homebox` | `sysadminsmedia/homebox:0.26.2-rootless` | Inventory application and API. |
| `homebox-database` | `postgres:17.6-alpine` | Inventory metadata database. |

Homebox alone joins the external `proxy` network. PostgreSQL is available only on the private Compose project network.

## Paths

| Path | Purpose |
| --- | --- |
| `/opt/homebox/compose.yaml` | Retained Compose project rendered by Ansible. |
| `/opt/homebox/secrets/` | Root-protected secret sources. |
| `/exports/docker/homebox/data` | Homebox attachments and generated files. |
| `/exports/docker/homebox/database` | PostgreSQL data. |

## Prerequisites

- A reachable inventory hostname, DNS name, or IP address passed as `target`.
- Docker Engine and the Docker Compose v2 plugin on that host.
- The `community.docker` collection from [`requirements.yml`](../../requirements.yml).
- `/exports/docker` on local storage, or `homebox.data_dir` changed to another local path.
- The [home-server bootstrap's](../../home_server/README.md#bootstrap) external local `proxy` bridge and a local Traefik container.
- DNS for `homebox.<domain>` directed to Traefik.

PostgreSQL must not run on NFS. The play rejects common remote filesystems before starting the database.

## Configuration

Define the domain and Homebox secrets in the encrypted vault:

```yaml
vault:
  shared:
    general:
      domain: example.com

  services:
    homebox:
      api_key_pepper: replace-with-at-least-32-random-characters
      database_password: replace-with-at-least-32-random-characters
```

Generate suitable values with `openssl rand -base64 48`. Keep the API key pepper stable because rotating it invalidates all Homebox API keys. Changing the vault database password does not update an initialized PostgreSQL role; rotate it in PostgreSQL first.

The secrets are mounted as file-backed Compose secrets and loaded by Homebox's startup process. They are not embedded in the retained Compose file or its declared environment.

## Deploy

For the first installation, run from the repository root:

```bash
ansible-playbook deployments/homebox/deploy.yml \
  -e target=192.168.1.50 -u pi \
  -e homebox_allow_fresh_install=true \
  --extra-vars @vault.yml --ask-vault-pass --ask-become-pass
```

Omit `-e homebox_allow_fresh_install=true` on later runs. The normal guard requires the existing PostgreSQL cluster and attachment directory before replacing containers.

The role validates storage and configuration, renders Compose, pulls missing images, waits for health checks, and verifies both services are running.

## Operations

```bash
sudo docker compose --project-directory /opt/homebox ps
sudo docker compose --project-directory /opt/homebox logs -f
sudo docker compose --project-directory /opt/homebox restart homebox
sudo docker compose --project-directory /opt/homebox config --quiet
```

After the first deployment, open `https://homebox.<domain>` and create the initial account. Redeploy by rerunning Ansible; do not edit `/opt/homebox/compose.yaml` directly.

## Backups

Back up both `/exports/docker/homebox/data` and `/exports/docker/homebox/database`. Use `pg_dump` for consistent database backups; copying a live PostgreSQL data directory is not a substitute for a native database backup.

See the [Homebox documentation](https://homebox.software/en/) for application usage and restore guidance.
