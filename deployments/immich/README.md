# Immich Deployment

[![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white&style=flat-square)](https://docs.ansible.com/) [![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/compose/) [![Traefik](https://img.shields.io/badge/Traefik-HTTPS-24A1C1?logo=traefikproxy&logoColor=white&style=flat-square)](https://doc.traefik.io/traefik/) ![Immich](https://img.shields.io/badge/Immich-v3.1.0-5C5C5C?style=flat-square)

An Ansible-managed, single-host Docker Compose deployment of Immich, PostgreSQL, Valkey, and Immich Machine Learning.

## Services

| Service | Image | Purpose |
| --- | --- | --- |
| `immich-server` | `ghcr.io/immich-app/immich-server:v3.1.0` | Web application, API, and background jobs. |
| `immich-machine-learning` | `ghcr.io/immich-app/immich-machine-learning:v3.1.0` | Face recognition and smart search. |
| `immich-redis` | `valkey/valkey:9` pinned by digest | Cache and job coordination. |
| `immich-database` | `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0` pinned by digest | Metadata and vector search database. |

The complete database and Valkey image tags and digests are in [`group_vars/all.yml`](group_vars/all.yml).

All containers run on the host passed through `-e target=...` using Docker
Compose. The private project network and external proxy bridge are local to
that host.

The server joins the external `proxy` network for Traefik. Database, Valkey, and
machine learning traffic remains on the private Compose project network.

## Paths

| Path | Purpose |
| --- | --- |
| `/opt/immich/compose.yaml` | Retained Compose project rendered by Ansible. |
| `/opt/immich/secrets/` | Root-protected database credential sources. |
| `/exports/docker/immich/upload` | Photo and video library mounted at Immich's `/data`. |
| `/exports/docker/immich/backups` | Immich database backups mounted at `/data/backups`. |
| `/exports/docker/immich/database` | PostgreSQL data. |
| `/exports/docker/immich/ml` | Downloaded machine learning models. |

`immich.data_dir` and `immich.compose_dir` can be changed in [`group_vars/all.yml`](group_vars/all.yml).

## Prerequisites

- A reachable inventory hostname, DNS name, or IP address passed as `target`.
- Docker Engine and the Docker Compose v2 plugin on that host.
- The `community.docker` collection from [`requirements.yml`](../../requirements.yml).
- `/exports/docker` on local storage, or `immich.data_dir` changed to another local path.
- The [home-server bootstrap's](../../home_server/README.md#bootstrap) external local `proxy` bridge.
- A local Traefik container attached to that bridge.
- DNS for `immich.<domain>` directed to Traefik.
- At least 6 GB RAM; 8 GB and four CPU cores are recommended.
- An x86-64-v2 or newer CPU when using x86 machine-learning images.

PostgreSQL must not run on NFS. The play checks the database filesystem against the local filesystems supported by the pinned Immich image before starting Compose.

## Existing Data

Routine deployments reuse the media, automatic backups, and PostgreSQL data at the paths listed above. The upload directory is mounted at `/data` in the server container, with database backups mounted at `/data/backups`.

The play requires `database/PG_VERSION` and these six media markers beneath `immich.data_dir`:

```text
upload/upload/.immich
upload/library/.immich
upload/thumbs/.immich
upload/encoded-video/.immich
upload/profile/.immich
backups/.immich
```

These checks prevent a wrong path from becoming a fresh, empty installation. Review upstream upgrade instructions and create a native PostgreSQL backup before changing image versions.

## Configuration

Define these values in the encrypted vault:

```yaml
vault:
  shared:
    general:
      domain: example.com

  services:
    immich:
      database:
        user: immich
        password: replace-with-a-long-random-password
```

The database username and password are written beneath `/opt/immich/secrets` and mounted as file-backed Compose secrets. They are not embedded in the retained Compose file or container environment.

Changing either vault credential does not update roles inside an initialized PostgreSQL database. Rename the role or rotate its password inside PostgreSQL first, then update the vault and redeploy.

For an intentional new installation, add `--extra-vars immich_allow_fresh_install=true` to the first run only. Later runs should use the default data guards.

## Deploy

Run from the repository root, replacing the example address and SSH user:

```bash
ansible-playbook deployments/immich/deploy.yml \
  -e target=192.168.1.50 -u pi \
  --extra-vars @vault.yml --ask-vault-pass --ask-become-pass
```

For a new installation, append `-e immich_allow_fresh_install=true` to this command. For inventory aliases or passwordless sudo, see the [command conventions](../../home_server/README.md#command-conventions).

The role validates the project, pulls images, starts Compose without a second registry request, waits for health checks, and fails unless all four services are running.

Immediately before startup, the role stops and removes every container using Immich's fixed container names, including containers from an earlier run of this Compose project. Compose then runs with `recreate: always`, so every successful playbook run creates fresh containers. Bind-mounted application data remains untouched, anonymous volumes are retained, and unrelated containers are not pruned.

| Stage | Task file | Responsibility |
| --- | --- | --- |
| Validate | [`validate.yml`](roles/immich/tasks/validate.yml) | Guard existing data and database credentials. |
| Filesystem | [`filesystem.yml`](roles/immich/tasks/filesystem.yml) | Create data and Compose directories and require local database storage. |
| Prepare | [`prepare.yml`](roles/immich/tasks/prepare.yml) | Prepare Docker, secrets, and the validated Compose project. |
| Pull | [`pull.yml`](roles/immich/tasks/pull.yml) | Pull each missing project image sequentially. |
| Deploy | [`deploy.yml`](roles/immich/tasks/deploy.yml) | Replace, start, and verify the Immich containers. |

## Operations

```bash
sudo docker compose --project-directory /opt/immich ps
sudo docker compose --project-directory /opt/immich logs -f
sudo docker compose --project-directory /opt/immich restart immich-server
sudo docker compose --project-directory /opt/immich config --quiet
```

After deployment, verify login, library counts, a new mobile upload, thumbnail generation, smart search, and creation of a new database backup.

Redeploy after changing variables or templates by rerunning Ansible. Do not edit `/opt/immich/compose.yaml` directly because Ansible replaces it.

## Backups

Back up the complete data root, but treat `/exports/docker/immich/upload`, `/exports/docker/immich/backups`, and `/exports/docker/immich/database` as critical. A filesystem copy of a running PostgreSQL directory is not a substitute for a verified native database backup.

See the [Immich documentation](https://immich.app/docs) for application backup and restore procedures.
