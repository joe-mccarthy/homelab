# Omni Tools Deployment

[![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white&style=flat-square)](https://docs.ansible.com/) [![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/compose/) [![Traefik](https://img.shields.io/badge/Traefik-HTTPS%20Access-24A1C1?logo=traefikproxy&logoColor=white&style=flat-square)](https://doc.traefik.io/traefik/) ![Omni Tools](https://img.shields.io/badge/Omni%20Tools-0.6.0-5C5C5C?style=flat-square)

An Ansible-managed, single-host Docker Compose deployment of **Omni Tools**, a self-hosted, browser-based collection of everyday utility tools. Traefik provides HTTPS access at `https://omni.<domain>`.

## Runtime Model

The `omni` Compose project runs one service named `omni`, with the fixed container name `omni-tools`, on the host passed through `-e target=...`.

The container joins the external local `proxy` bridge network. Service-level labels let Traefik's Docker provider route traffic to its internal HTTP listener on port `80`, redirect HTTP to HTTPS, and obtain certificates through the `letsencrypt` resolver.

Omni Tools is stateless and requires no persistent application storage. The rendered Compose project remains at `/opt/omni/compose.yaml` for normal `docker compose` operations.

The image is pinned to `iib0011/omni-tools:0.6.0`, corresponding to [upstream release v0.6.0](https://github.com/iib0011/omni-tools/releases/tag/v0.6.0). This tag provides Linux AMD64 and ARM64 images.

## Prerequisites

- A reachable inventory hostname, DNS name, or IP address passed as `target`.
- Docker Engine and the Docker Compose v2 plugin on that host.
- The `community.docker` collection from [`requirements.yml`](../../requirements.yml).
- An existing local `proxy` bridge network and a [Traefik](../traefik/README.md) container attached to it.
- DNS for `omni.<domain>` directed to Traefik.
- `vault.shared.general.domain` defined in the encrypted vault.

The [home-server bootstrap](../../home_server/README.md) prepares Docker and the local proxy bridge on a standalone host.

## Configuration

Settings live in [`group_vars/all.yml`](group_vars/all.yml):

| Variable | Default | Purpose |
| --- | --- | --- |
| `omni.compose_dir` | `/opt/omni` | Retained Compose project directory. |
| `omni.proxy_network` | `proxy` | External local bridge shared with Traefik. |
| `omni.version` | `0.6.0` | Explicit Docker image version. |

Set the domain in the vault using [`vault.template.yml`](../../vault.template.yml):

```yaml
vault:
  shared:
    general:
      domain: example.com
```

## Deploy

Run from the repository root, replacing the example address and SSH user:

```bash
ansible-playbook deployments/omni/deploy.yml \
  -e target=192.168.1.50 -u pi \
  --extra-vars @vault.yml --ask-vault-pass --ask-become-pass
```

For inventory aliases or passwordless sudo, see the [command conventions](../../home_server/README.md#command-conventions). The vault is explicitly loaded by `--extra-vars @vault.yml`.

The [`omni` role](roles/omni/tasks/main.yml) follows five ordered stages:

| Stage | Task file | Responsibility |
| --- | --- | --- |
| Validate | [`validate.yml`](roles/omni/tasks/validate.yml) | Validate the Compose path, proxy network setting, and domain. |
| Filesystem | [`filesystem.yml`](roles/omni/tasks/filesystem.yml) | Create the root-owned Compose project directory. |
| Prepare | [`prepare.yml`](roles/omni/tasks/prepare.yml) | Prepare Docker, validate the local proxy bridge, and render and validate Compose. |
| Pull | [`pull.yml`](roles/omni/tasks/pull.yml) | Pull the pinned image before interrupting the existing container. |
| Deploy | [`deploy.yml`](roles/omni/tasks/deploy.yml) | Replace the existing container, start Compose, and verify Omni Tools is running. |

Each deployment recreates the `omni-tools` container using the validated Compose project and the already-pulled image.

## Operations

```bash
sudo docker compose --project-directory /opt/omni ps
sudo docker compose --project-directory /opt/omni logs -f
sudo docker compose --project-directory /opt/omni restart omni
sudo docker compose --project-directory /opt/omni config --quiet
```

To upgrade, set `omni.version` to a published version in `group_vars/all.yml` and rerun the playbook. Ansible replaces the rendered Compose file on each deployment.

## Troubleshooting

- **Service not accessible**: inspect the local bridge with `docker network inspect proxy` and confirm both Omni Tools and Traefik are attached.
- **Certificate not issued**: verify DNS resolves `omni.<domain>` to the Traefik host and its `letsencrypt` resolver is configured.
- **Container fails to start**: inspect `sudo docker compose --project-directory /opt/omni logs` and confirm the pinned image is available for the host architecture.
