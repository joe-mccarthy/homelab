# Paperless Deployment

[![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white&style=flat-square)](https://docs.ansible.com/) [![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/compose/) [![Traefik](https://img.shields.io/badge/Traefik-HTTPS-24A1C1?logo=traefikproxy&logoColor=white&style=flat-square)](https://doc.traefik.io/traefik/) ![paperless-ngx](https://img.shields.io/badge/paperless--ngx-v3.1.0-5C5C5C?style=flat-square)

An Ansible-managed, single-host Docker Compose deployment of Paperless-ngx with PostgreSQL, Redis, Gotenberg, and Apache Tika.

## Services

| Service | Version | Purpose |
| --- | ---: | --- |
| `paperless` | `3.1.0` | Web UI, API, document ingestion, and OCR. |
| `paperless-broker` | `8.10.1` | Redis task broker. |
| `paperless-database` | `18.6` | PostgreSQL application database. |
| `paperless-gotenberg` | `8.36.0` | PDF and email rendering. |
| `paperless-tika` | `3.3.1.0` | Office document and metadata parsing. |

All five containers run on the host passed through `-e target=...`. Paperless
joins the external local `proxy` bridge network for Traefik. Backend traffic uses
an internal Compose network.

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

The legacy floating `redis:8` image can write RDB format 14. Redis versions before 8.8 cannot read that format, so the broker must not be downgraded below the pinned 8.8 release while retaining this directory.

## Prerequisites

- A reachable inventory hostname, DNS name, or IP address passed as `target`.
- Docker Engine and the Docker Compose v2 plugin on that host.
- The `community.docker` collection from [`requirements.yml`](../../requirements.yml).
- `/exports/docker` on local storage, or `paperless.data_dir` changed to another local path.
- The machine bootstrap's external `proxy` network.
- A local Traefik container attached to that bridge.
- DNS for `paperless.<domain>` directed to Traefik.
- Vault values defined from [`vault.template.yml`](../../vault.template.yml).

## Existing Data

The play requires an initialized PostgreSQL 18 cluster under `/exports/docker/paperless/database` and the existing `data` and `media` directories. It verifies the cluster's `PG_VERSION` marker before replacing containers, preventing an incorrect path from silently creating an empty database.

Older Paperless installations used SQLite at
`/exports/docker/paperless/data/db.sqlite3`. Do not use
`paperless_allow_fresh_install=true` while that database is still present. The
role refuses the cutover until the following migration is complete:

1. Back up the complete `/exports/docker/paperless` tree.
2. While the legacy Paperless service is running, use its
   `document_exporter ../export` command to create an export in the persistent
   `/usr/src/paperless/export` mount.
3. Verify the export completed successfully and contains `manifest.json`.
4. Stop the legacy Paperless service and archive `data/db.sqlite3` outside
   `/exports/docker/paperless`; do not delete the backup.
5. Run this deployment once with both `paperless_allow_fresh_install=true` and
   `paperless_sqlite_migration=true`. The latter flag requires the persistent
   export manifest before PostgreSQL can be initialized.
6. Run `docker exec paperless document_importer ../export`, then verify users,
   documents, tags, correspondents, and document counts before retiring the
   SQLite backup.

Use both flags only for the first deployment that initializes PostgreSQL from
the SQLite export. Retain the export as a backup, but omit both flags on later
deployments; an initialized PostgreSQL cluster takes precedence over it.

The Compose settings retain automatic OCR, archive generation, and duplicate rejection. Tika remains on the latest release pinned by Paperless upstream because Tika 4 is not yet the supported conversion image.

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

Run from the repository root. Include `--extra-vars @vault.yml` when the vault is not already loaded by inventory:

```bash
ansible-playbook \
  -i inventory.yml \
  deployments/paperless/deploy.yml \
  -e target=odin \
  --extra-vars @vault.yml \
  --ask-vault-pass
```

The `paperless` role is organized into five ordered task files:

| Stage | Task file | Purpose |
| ---: | --- | --- |
| 1 | `validate.yml` | Guard the data root, database, media, and secret key. |
| 2 | `filesystem.yml` | Create persistent application, broker, PostgreSQL, Compose, and secrets directories. |
| 3 | `prepare.yml` | Prepare Docker, write secrets, render Compose, and validate it. |
| 4 | `pull.yml` | Pull each missing image sequentially. |
| 5 | `deploy.yml` | Replace and start containers, then verify services and conversion endpoints. |

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
