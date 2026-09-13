# Immich Deployment

[![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white&style=flat-square)](https://docs.ansible.com/) [![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/compose/) [![Traefik](https://img.shields.io/badge/Traefik-HTTPS-24A1C1?logo=traefikproxy&logoColor=white&style=flat-square)](https://doc.traefik.io/traefik/) ![Immich](https://img.shields.io/badge/Immich-v3.1.0-5C5C5C?style=flat-square)

An Ansible-managed, single-host Docker Compose deployment of Immich, PostgreSQL, Valkey, and Immich Machine Learning.

## Services

| Service | Image | Purpose |
| --- | --- | --- |
| `immich-server` | `ghcr.io/immich-app/immich-server:v3.1.0` | Web application, API, and background jobs. |
| `immich-machine-learning` | `ghcr.io/immich-app/immich-machine-learning:v3.1.0` | Face recognition and smart search. |
| `immich-redis` | `valkey/valkey:9` pinned by digest | Cache and job coordination. |
| `immich-database` | Immich PostgreSQL 14 image pinned by digest | Metadata and vector search database. |

All containers run on the host passed through `-e target=...`. Docker Swarm placement and overlay networks are not used by the resulting deployment.

The server joins the local external `proxy` bridge for Traefik. Database, Valkey, and machine learning traffic remains on the private Compose project network.

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
- The machine bootstrap's local `proxy` bridge network.
- A local Traefik container attached to the `proxy` bridge.
- DNS for `immich.<domain>` directed to Traefik.
- At least 6 GB RAM; 8 GB and four CPU cores are recommended.
- An x86-64-v2 or newer CPU when using x86 machine-learning images.

PostgreSQL must not run on NFS. The play checks the database filesystem against the local filesystems supported by the pinned Immich image before starting Compose.

## Existing Data

The former template mounted the host upload directory at `/usr/src/app/upload`, while this project uses Immich's current `/data` mount.

Immich v3 supports direct upgrades from this deployment's former v2.7.5 pin. The database already uses VectorChord, as required by v3. Create and test a native PostgreSQL backup before the first v3 deployment.

Before the first Compose start:

1. Place the existing media tree under `/exports/docker/immich/upload`.
2. Place automatic backups under `/exports/docker/immich/backups`.
3. Place the PostgreSQL cluster under `/exports/docker/immich/database`.
4. Create and test a native PostgreSQL backup.

The play requires the existing PostgreSQL `PG_VERSION` file and all six Immich `.immich` media markers before deployment. These checks prevent a wrong path from becoming a fresh, empty installation.

During migration, the deployment removes the four known legacy `immich_*` Swarm services. It does not prune unrelated services, stacks, or overlay networks.

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

Run from the repository root. Include `--extra-vars @vault.yml` when the vault is not already loaded by inventory:

```bash
ansible-playbook \
  -i inventory.yml \
  deployments/immich/deploy.yml \
  -e target=odin \
  --extra-vars @vault.yml \
  --ask-vault-pass
```

The role validates the project, pulls images, starts Compose without a second registry request, waits for health checks, and fails unless all four services are running.

Immediately before startup, the role stops and removes every container using Immich's fixed container names, including containers from an earlier run of this Compose project. Compose then runs with `recreate: always`, so every successful playbook run creates fresh containers. Bind-mounted application data remains untouched, anonymous volumes are retained, and unrelated containers are not pruned.

| Stage | Task file | Responsibility |
| --- | --- | --- |
| Validate | `validate.yml` | Guard existing data and database credentials. |
| Filesystem | `filesystem.yml` | Create data and Compose directories and require local database storage. |
| Prepare | `prepare.yml` | Prepare Docker, secrets, and the validated Compose project. |
| Pull | `pull.yml` | Pull each project image sequentially. |
| Deploy | `deploy.yml` | Replace, start, and verify the Immich containers. |

## Operations

```bash
sudo docker compose --project-directory /opt/immich ps
sudo docker compose --project-directory /opt/immich logs -f
sudo docker compose --project-directory /opt/immich restart immich-server
sudo docker compose --project-directory /opt/immich config --quiet
```

After migration, verify login, library counts, a new mobile upload, thumbnail generation, smart search, and creation of a new database backup.

Redeploy after changing variables or templates by rerunning Ansible. Do not edit `/opt/immich/compose.yaml` directly because Ansible replaces it.

## Backups

Back up the complete data root, but treat `/exports/docker/immich/upload`, `/exports/docker/immich/backups`, and `/exports/docker/immich/database` as critical. A filesystem copy of a running PostgreSQL directory is not a substitute for a verified native database backup.

See the [Immich documentation](https://immich.app/docs) for application backup and restore procedures.
