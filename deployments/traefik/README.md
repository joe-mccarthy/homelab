# Traefik Deployment

[![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white&style=flat-square)](https://docs.ansible.com/) [![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/compose/) [![Let's Encrypt](https://img.shields.io/badge/Let's%20Encrypt-TLS-003A70?logo=letsencrypt&logoColor=white&style=flat-square)](https://letsencrypt.org/docs/) [![Cloudflare](https://img.shields.io/badge/Cloudflare-DNS%20Challenge-F38020?logo=cloudflare&logoColor=white&style=flat-square)](https://developers.cloudflare.com/dns/) ![Traefik](https://img.shields.io/badge/Traefik-v3.7.12-5C5C5C?style=flat-square)

An Ansible-managed, single-host Docker Compose deployment of Traefik. It discovers local containers through the Docker provider and legacy Swarm services through the Swarm provider, routes HTTP, HTTPS, and SSH traffic, and obtains wildcard certificates with a Cloudflare DNS-01 challenge.

## Runtime Model

Traefik runs on the host passed through `-e target=...` using ordinary Docker Compose, Docker and Swarm providers, a shared external network, and file-backed Compose secrets.

The container joins the external `proxy` network. On a Swarm manager this can be the existing attachable overlay, allowing Traefik to reach both Compose containers and legacy Swarm services. Other Compose services must join the same network and place their Traefik labels directly under the service-level `labels` key.

The rendered project remains at `/opt/traefik` so normal `docker compose` commands can manage it after deployment. Persistent certificates and logs live under `/exports/docker/traefik` by default.

## Prerequisites

- A reachable inventory hostname, DNS name, or IP address passed as `target`.
- Docker Engine and the Docker Compose v2 plugin on that host.
- The `community.docker` Ansible collection from [`requirements.yml`](../../requirements.yml).
- Ports `80`, `443`, and `2222` available, unless overridden in the vault.
- Public DNS records directed to the host.
- Cloudflare credentials and the public domain defined in Ansible Vault.

The Cloudflare token should be limited to DNS zone read and edit permissions for the required zone.

The external `proxy` network must already exist; this deployment does not manage it. [`home_server/setup.yml`](../../home_server/setup.yml) creates it when absent and safely rejects an existing non-attachable Swarm overlay. New Swarm setups create the overlay as attachable. Existing clusters must recreate a non-attachable overlay during a maintenance window before Compose containers can join it.

## Deploy

Run from the repository root:

```bash
ansible-playbook -i inventory.yml deployments/traefik/deploy.yml \
  -e target=odin --extra-vars @vault.yml --ask-vault-pass
```

Add `--ask-become-pass` if the remote user requires a sudo password.

The deployment uses one Traefik role with five ordered task files:

| Stage | Task file | Responsibility |
| --- | --- | --- |
| Validate | [`validate.yml`](roles/traefik/tasks/validate.yml) | Validates required settings, ports, and the types of existing managed paths without modifying the host. Missing directories are valid. |
| Filesystem | [`filesystem.yml`](roles/traefik/tasks/filesystem.yml) | Creates parent directories, persistent data, logs, ACME storage, Compose, and secret paths with their required ownership and modes. |
| Prepare | [`prepare.yml`](roles/traefik/tasks/prepare.yml) | Ensures Docker is running, writes secrets, and renders and validates the Compose project. |
| Pull | [`pull.yml`](roles/traefik/tasks/pull.yml) | Pulls the pinned Traefik image before the existing proxy is interrupted. |
| Deploy | [`deploy.yml`](roles/traefik/tasks/deploy.yml) | Removes the legacy Swarm service and old container, starts the replacement, and verifies that it is running. |

[`roles/traefik/tasks/main.yml`](roles/traefik/tasks/main.yml) imports these files in deployment order. The top-level playbook includes only the `traefik` role.

The role removes the legacy `traefik_traefik` Swarm service and the fixed container name `traefik`; it does not prune unrelated services or containers. The external `proxy` network and other workloads attached to it remain intact during Traefik recreation.

## Configuration

The deployment reads these vault values:

```yaml
vault:
  shared:
    cloudflare:
      email: admin@example.com
      token: replace-with-a-scoped-token
    general:
      domain: example.com

  services:
    traefik:
      ports:
        web: "80"
        websecure: "443"
        ssh: "2222"
      dashboard:
        enabled: true
        allowed_ips: "192.168.1.0/24"
```

Cloudflare values are mounted as Compose secrets at `/run/secrets/cf_email` and `/run/secrets/cf_token`; they are not embedded in the rendered Compose file or container environment.

Every playbook run recreates the Traefik container, so file-backed credential changes are always picked up immediately.

The dashboard is available at `https://traefik.<domain>` when enabled. It is not published on port `8080`, and both HTTP and HTTPS dashboard routes enforce the configured source-range allowlist.

## Persistent Files

```text
/exports/docker/traefik
├── acme.json
└── logs
    └── traefik.log
```

The role preserves the existing `acme.json` at this path across container removal and recreation.

## Operations

Check the container:

```bash
sudo docker compose --project-directory /opt/traefik ps
```

Follow logs:

```bash
sudo docker compose --project-directory /opt/traefik logs -f
```

Validate the rendered project:

```bash
sudo docker compose --project-directory /opt/traefik config --quiet
```

Redeploy after changing variables or templates by rerunning the Ansible playbook. Every run removes and recreates the `traefik` container while preserving `traefik.data_dir`. Do not edit the rendered files because Ansible replaces them.

## Security

- Keep the Docker socket mount read-only, but treat the Traefik container as host-root equivalent because Docker API access is privileged.
- Keep `/opt/traefik`, its Compose file, and credential files readable only by root.
- Restrict the dashboard allowlist to trusted networks.
- Back up `acme.json`; it contains private certificate keys.
- Do not publish an insecure dashboard listener.
