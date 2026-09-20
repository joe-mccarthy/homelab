# Paperless Deployment

[![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white&style=flat-square)](https://docs.ansible.com/) [![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/compose/) [![Traefik](https://img.shields.io/badge/Traefik-HTTPS-24A1C1?logo=traefikproxy&logoColor=white&style=flat-square)](https://doc.traefik.io/traefik/) ![paperless-ngx](https://img.shields.io/badge/paperless--ngx-v3.1.0-5C5C5C?style=flat-square)

An Ansible-managed, single-host Docker Compose deployment of Paperless-ngx with PostgreSQL, Redis, Gotenberg, and Apache Tika.

## Services

| Service | Image tag | Purpose |
| --- | ---: | --- |
| `paperless` | `3.1.0` | Web UI, API, document ingestion, and OCR. |
| `paperless-broker` | `8.10.1` | Redis task broker. |
| `paperless-database` | `18.6-bookworm` | PostgreSQL application database. |
| `paperless-gotenberg` | `8.36.0` | PDF and email rendering. |
| `paperless-tika` | `3.3.1.0` | Office document and metadata parsing. |

All five containers run on the host passed through `-e target=...`. Paperless
joins the external local `proxy` bridge network for Traefik. Backend traffic uses
the project's default bridge network.

## Paths

| Path | Purpose |
| --- | --- |
| `/opt/paperless/compose.yaml` | Retained Compose project rendered by Ansible. |
| `/opt/paperless/secrets/paperless_secret_key` | Stable Django signing key. |
| `/opt/paperless/secrets/paperless_database_password` | PostgreSQL password. |
| `/exports/docker/paperless/data` | Search index and application state. |
| `/exports/docker/paperless/database` | PostgreSQL 18 data. Must remain on local storage. |
| `/exports/docker/paperless/media` | Original documents, archived documents, and thumbnails. |
| `/exports/docker/paperless/consume` | Document ingestion directory. |
| `/exports/docker/paperless/export` | Generated exports. |
| `/exports/docker/paperless/broker` | Redis persistence. |

Redis persists broker state in `broker/` using the pinned image listed above. The complete image tags and path settings are in [`group_vars/all.yml`](group_vars/all.yml).

## Prerequisites

- A reachable inventory hostname, DNS name, or IP address passed as `target`.
- Docker Engine and the Docker Compose v2 plugin on that host.
- The `community.docker` collection from [`requirements.yml`](../../requirements.yml).
- `/exports/docker` on local storage, or `paperless.data_dir` changed to another local path.
- The [home-server bootstrap's](../../home_server/README.md#bootstrap) external local `proxy` bridge.
- A local Traefik container attached to that bridge.
- DNS for `paperless.<domain>` directed to Traefik.
- Vault values defined from [`vault.template.yml`](../../vault.template.yml).

## Existing Data

The play requires an initialized PostgreSQL 18 cluster under `/exports/docker/paperless/database` and the existing `data` and `media` directories. It verifies `database/18/docker/PG_VERSION` and checks that it contains the configured major version before replacing containers.

`paperless.database.data_dir` is configured separately from `paperless.data_dir`. Update both settings when relocating application and database storage.

Normal deployments use the existing PostgreSQL database. For a new installation,
add `-e paperless_allow_fresh_install=true` to the first run to initialize the
database and application directories. Omit that option on subsequent runs.

The Compose settings enable automatic OCR, archive generation, duplicate rejection, and conversion through the pinned Gotenberg and Tika services.

## Configuration

Define stable Paperless and PostgreSQL secrets and the public domain in the encrypted vault:

Generate each secret once:

```bash
openssl rand -hex 32
```

```yaml
vault:
  shared:
    general:
      domain: example.com

  services:
    paperless:
      secret_key: replace-with-at-least-32-random-characters
      database_password: replace-with-at-least-32-random-characters
```

Do not rotate `secret_key` during a routine deployment. It signs browser sessions and other Django data. Both credentials are mounted as file-backed Compose secrets and are not embedded in the Compose file or Docker configuration environment.

Paperless runs application files as UID/GID `1000`. Change `paperless.uid` and `paperless.gid` together if the existing files use another identity.

## Deploy

Run from the repository root, replacing the example address and SSH user:

```bash
ansible-playbook deployments/paperless/deploy.yml \
  -e target=192.168.1.50 -u pi \
  --extra-vars @vault.yml --ask-vault-pass --ask-become-pass
```

For a new installation, append `-e paperless_allow_fresh_install=true` to this command. For inventory aliases or passwordless sudo, see the [command conventions](../../home_server/README.md#command-conventions).

The `paperless` role is organized into five ordered task files:

| Stage | Task file | Purpose |
| ---: | --- | --- |
| 1 | [`validate.yml`](roles/paperless/tasks/validate.yml) | Guard the data root, database version, media, and credentials. |
| 2 | [`filesystem.yml`](roles/paperless/tasks/filesystem.yml) | Create persistent directories and require local database storage. |
| 3 | [`prepare.yml`](roles/paperless/tasks/prepare.yml) | Prepare Docker, write secrets, render Compose, and validate it. |
| 4 | [`pull.yml`](roles/paperless/tasks/pull.yml) | Pull each missing image sequentially. |
| 5 | [`deploy.yml`](roles/paperless/tasks/deploy.yml) | Replace and start containers, then verify services and conversion endpoints. |

The role validates the project, pulls images, starts Compose without a second registry request, waits for the containers, and verifies that PostgreSQL, Tika, and Gotenberg are available.

Immediately before startup, the role stops and removes every container using Paperless's fixed container names, including containers from an earlier run of this Compose project. Compose then runs with `recreate: always`, so every successful playbook run creates fresh containers. Bind-mounted application data remains untouched, anonymous volumes are retained, and unrelated containers are not pruned.

## Operations

```bash
sudo docker compose --project-directory /opt/paperless ps
sudo docker compose --project-directory /opt/paperless logs -f
sudo docker compose --project-directory /opt/paperless restart paperless
sudo docker compose --project-directory /opt/paperless config --quiet
```

After deployment, verify existing users and document counts, ingest a test document, process an Office document through Tika and Gotenberg, and create an export.

Redeploy after changing variables or templates by rerunning Ansible. Do not edit `/opt/paperless/compose.yaml` directly because Ansible replaces it.

## Backups

Back up PostgreSQL with `pg_dump` and retain the complete `/exports/docker/paperless` tree. Database and filesystem backups should represent the same point in time. Do not use a live copy of the PostgreSQL data directory as the database backup.

See the [Paperless-ngx documentation](https://docs.paperless-ngx.com/) for exporter and restore procedures.
