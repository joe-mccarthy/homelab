# DDNS Deployment

[![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white&style=flat-square)](https://docs.ansible.com/) [![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/compose/) [![Cloudflare](https://img.shields.io/badge/Cloudflare-DNS%20API-F38020?logo=cloudflare&logoColor=white&style=flat-square)](https://developers.cloudflare.com/dns/) ![cloudflare-ddns](https://img.shields.io/badge/cloudflare--ddns-v1.17.0-5C5C5C?style=flat-square)

An Ansible-managed, single-host Docker Compose deployment of Favonia Cloudflare DDNS. It periodically reconciles configured DNS records with the gateway's public IPv4 address.

## Runtime Model

The container runs on the host passed through `-e target=...`. It has no inbound ports, persistent application data, NFS dependency, or Traefik attachment. Compose retains the project at `/opt/ddns`.

The container runs as UID/GID `1000`, uses a read-only root filesystem, drops all Linux capabilities, and enables `no-new-privileges`.

## Prerequisites

- A reachable inventory hostname, DNS name, or IP address passed as `target`.
- Docker Engine and the Docker Compose v2 plugin on that host.
- The `community.docker` collection from [`requirements.yml`](../../requirements.yml).
- Outbound DNS and HTTPS access.
- A Cloudflare API token with zone read and DNS edit permissions for the configured records.
- A public IPv4 address. DDNS does not bypass carrier-grade NAT.

## Configuration

Define the token and comma-separated domain list in the encrypted vault:

```yaml
vault:
  shared:
    cloudflare:
      token: replace-with-a-scoped-token

  services:
    ddns:
      dns_domains: home.example.com,another.example.com
```

The token is stored beneath `/opt/ddns/secrets` and mounted through `CLOUDFLARE_API_TOKEN_FILE`; it is not embedded in the Compose file or container environment. Token changes recreate the container.

`PROXIED=true` is retained, and IPv6 updates are disabled with `IP6_PROVIDER=none`. The obsolete `ZONE_ID` setting has been removed because this image discovers zones from `DOMAINS`.

## Deploy

Run from the repository root. Include `--extra-vars @vault.yml` when the vault is not already loaded by inventory:

```bash
ansible-playbook \
  -i inventory.yml \
  deployments/ddns/deploy.yml \
  -e target=odin \
  --extra-vars @vault.yml \
  --ask-vault-pass
```

The role validates the Compose project, pulls the image, starts the container,
and verifies that the DDNS service is running.

Immediately before startup, the role stops and removes any existing `cloudflare-ddns` container, including one from an earlier run of this Compose project. Compose then runs with `recreate: always`, so every successful playbook run creates a fresh container. The removal retains volumes and does not perform broad container pruning.

DDNS can be deployed independently of Traefik and the web applications.

| Stage | Task file | Responsibility |
| --- | --- | --- |
| 1 | `validate.yml` | Validate configuration and the Cloudflare token |
| 2 | `filesystem.yml` | Create Compose and secrets directories |
| 3 | `prepare.yml` | Prepare Docker, secrets, and Compose configuration |
| 4 | `pull.yml` | Pull the pinned image |
| 5 | `deploy.yml` | Replace, start, and verify the container |

## Operations

```bash
sudo docker compose --project-directory /opt/ddns ps
sudo docker compose --project-directory /opt/ddns logs -f
sudo docker compose --project-directory /opt/ddns restart cloudflare-ddns
sudo docker compose --project-directory /opt/ddns config --quiet
```

After deployment, confirm the logs show successful public IPv4 detection and Cloudflare record reconciliation. Also verify that exactly one DDNS updater is running.

Redeploy after changing variables or templates by rerunning Ansible. Do not edit `/opt/ddns/compose.yaml` directly because Ansible replaces it.
