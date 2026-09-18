# Home Assistant Deployment

[![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white&style=flat-square)](https://docs.ansible.com/) [![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/compose/) [![Traefik](https://img.shields.io/badge/Traefik-HTTPS-24A1C1?logo=traefikproxy&logoColor=white&style=flat-square)](https://doc.traefik.io/traefik/) [![Matter](https://img.shields.io/badge/Matter-Wi--Fi-00BFB3?style=flat-square)](https://csa-iot.org/all-solutions/matter/)

An Ansible-managed, single-host Docker Compose deployment of Home Assistant, Matter.js Server, Zigbee2MQTT, and Mosquitto.

> [!WARNING]
> Mosquitto uses its upstream no-auth configuration and Matter Server listens on the host's LAN port `5580`. Keep both services on a trusted network.

## Services

| Service | Image | Purpose |
| --- | --- | --- |
| `homeassistant` | `ghcr.io/home-assistant/home-assistant:2026.8.3` | Home automation platform and web UI. |
| `matter-server` | `ghcr.io/matter-js/matterjs-server:1.4.0` | Matter controller and WebSocket API. |
| `zigbee2mqtt` | `ghcr.io/koenkk/zigbee2mqtt:2.13.0` | Zigbee-to-MQTT bridge and web UI. |
| `mqtt` | `eclipse-mosquitto:2.1.2-alpine` | MQTT broker used by Home Assistant and Zigbee2MQTT. |

Docker Compose runs the project as `home_assistant` and assigns stable container names matching the service names above.

## Runtime Model

All four containers run on the host passed through `-e target=...`. The deployment uses ordinary Docker Compose without node labels or placement constraints. Its private project network is local; the external proxy network can be a bridge on a Compose-only host or an attachable overlay on a Swarm host.

Home Assistant, Zigbee2MQTT, and Mosquitto share the Compose project network. Home Assistant and Zigbee2MQTT also join the external `proxy` network so the local Traefik container can route HTTPS traffic to them.

Matter Server uses `network_mode: host` because Matter device discovery and control require LAN IPv6 and mDNS multicast. Home Assistant connects to it through the host's stable LAN address:

```text
ws://<home-assistant-host>:5580/ws
```

## Files

| Path | Purpose |
| --- | --- |
| [`deploy.yml`](deploy.yml) | Targets the single application host and runs the Home Assistant role. |
| [`group_vars/all.yml`](group_vars/all.yml) | Local paths, network name, vault lookups, and image versions. |
| [`templates/docker-compose.yaml`](templates/docker-compose.yaml) | Local Docker Compose project template. |
| [`templates/configuration.yaml`](templates/configuration.yaml) | Initial Home Assistant configuration. |
| [`templates/zigbee/configuration.yaml`](templates/zigbee/configuration.yaml) | Initial Zigbee2MQTT configuration. |
| [`roles/home_assistant/tasks/main.yml`](roles/home_assistant/tasks/main.yml) | Runs the five deployment stages in order. |

The rendered Compose project remains at `/opt/home-assistant/compose.yaml` so normal `docker compose` commands can manage it after deployment.

## Prerequisites

- A reachable inventory hostname, DNS name, or IP address passed as `target`.
- Docker Engine and the Docker Compose v2 plugin on that host.
- The `community.docker` Ansible collection from [`requirements.yml`](../../requirements.yml).
- `/exports/docker` available as local persistent storage, or `home_assistant.data_dir` changed to another local path.
- A local Traefik deployment attached to the external `proxy` network.
- A Zigbee coordinator available through a local `/dev/...` path or a `tcp://...` endpoint.
- Working IPv6 and mDNS/multicast between this host and Matter devices.
- Vault values defined from [`../../vault.template.yml`](../../vault.template.yml).

The machine bootstrap creates the external `proxy` bridge on a Compose-only host or an attachable overlay on a Swarm host; this deployment does not manage it.

## Deploy

Run from the repository root:

```bash
ansible-playbook -i inventory.yml deployments/home-assistant/deploy.yml \
  -e target=odin --extra-vars @vault.yml --ask-vault-pass
```

Add `--ask-become-pass` if the remote user requires a sudo password.

The deployment uses one Home Assistant role with five ordered task files:

| Stage | Task file | Responsibility |
| --- | --- | --- |
| Validate | [`validate.yml`](roles/home_assistant/tasks/validate.yml) | Validates required settings and existing managed directory paths without modifying the host. |
| Filesystem | [`filesystem.yml`](roles/home_assistant/tasks/filesystem.yml) | Creates persistent data, initial configuration, and Compose directories. |
| Prepare | [`prepare.yml`](roles/home_assistant/tasks/prepare.yml) | Ensures Docker is running, renders and validates Compose, and verifies persisted Zigbee settings. |
| Pull | [`pull.yml`](roles/home_assistant/tasks/pull.yml) | Pulls all project images before the existing containers are interrupted. |
| Deploy | [`deploy.yml`](roles/home_assistant/tasks/deploy.yml) | Removes known containers, reapplies Matter ownership, starts Compose, and verifies every service. |

[`roles/home_assistant/tasks/main.yml`](roles/home_assistant/tasks/main.yml) imports these files in deployment order. The top-level playbook includes only the `home_assistant` role.

The deployment:

1. Prepares persistent data under `home_assistant.data_dir`.
2. Installs the Docker Compose plugin and starts Docker.
3. Renders and validates `/opt/home-assistant/compose.yaml` with mode `0600`.
4. Verifies a configured local Zigbee device, when applicable, and confirms that persisted MQTT and coordinator settings match the vault.
5. Pulls every required image while the existing deployment is still running.
6. Stops and removes containers using the deployment's fixed names while retaining volumes.
7. Reapplies Matter Server data ownership after its container has stopped.
8. Runs `docker compose up` with forced recreation and waits for all four containers without another registry request.
9. Fails unless every expected service is running.

Only the known Home Assistant Swarm services and the five container names `homeassistant`, `zigbee2mqtt`, `matter-server`, `mqtt`, and the obsolete `matter-server-proxy` are removed. The role does not broadly prune unrelated services or containers.

## Configuration

The deployment reads these vault-backed values:

```yaml
vault:
  shared:
    general:
      domain: example.com

  services:
    home_assistant:
      proxy: 172.18.0.0/16
      mqtt_server: mqtt://mqtt:1883
      zigbee_serial_port: tcp://zigbee-coordinator.local:6638
```

| Variable | Used for |
| --- | --- |
| `vault.shared.general.domain` | Traefik host rules for Home Assistant and Zigbee2MQTT. |
| `vault.services.home_assistant.proxy` | Home Assistant's trusted proxy address or CIDR. |
| `vault.services.home_assistant.mqtt_server` | Zigbee2MQTT broker URL; normally `mqtt://mqtt:1883`. |
| `vault.services.home_assistant.zigbee_serial_port` | Local device path or TCP coordinator URL used by Zigbee2MQTT. |

The Home Assistant and Zigbee2MQTT configuration templates are initial defaults and are not overwritten after first creation. Edit `/exports/docker/home_assistant/configuration.yaml` directly if `trusted_proxies` needs the local bridge CIDR. If the MQTT URL or coordinator path changes, edit `/exports/docker/home_assistant/zigbee2mqtt/data/configuration.yaml` as well as updating the vault.

## Persistent Data

The default data root is `/exports/docker/home_assistant`:

```text
/exports/docker/home_assistant
├── configuration.yaml
├── automations.yaml
├── scenes.yaml
├── scripts.yaml
├── matter-server/data
├── mosquitto
└── zigbee2mqtt/data
```

The initial Home Assistant and Zigbee2MQTT templates use `force: false`, so subsequent deployments preserve files changed by the applications or by hand.

Matter.js Server runs as UID/GID `1000`. The deployment recursively applies that ownership to `matter-server/data`, including data migrated from Python Matter Server.

## Operations

Check all containers:

```bash
sudo docker compose --project-directory /opt/home-assistant ps
```

Follow logs:

```bash
sudo docker compose --project-directory /opt/home-assistant logs -f
```

Restart one service:

```bash
sudo docker compose --project-directory /opt/home-assistant restart homeassistant
sudo docker compose --project-directory /opt/home-assistant restart matter-server
sudo docker compose --project-directory /opt/home-assistant restart zigbee2mqtt
sudo docker compose --project-directory /opt/home-assistant restart mqtt
```

Redeploy after changing variables or templates by rerunning the Ansible playbook. Every run removes the known containers and recreates all four services; persistent bind-mounted data remains intact. Do not edit the rendered Compose file because Ansible replaces it.

## Zigbee

For a local coordinator, use a stable `/dev/serial/by-id/...` path instead of `/dev/ttyUSB0`; Compose passes `/dev/...` values through as Docker devices. For a network coordinator, use its `tcp://host:port` URL. TCP coordinators are configured only in Zigbee2MQTT and are not added to Compose's `devices` list.

Pairing is disabled by default in the initial configuration. Enable it only while adding devices, then disable it again.

## Matter

Add the Matter integration in Home Assistant under **Settings > Devices & services**. Choose the custom Matter Server option and enter:

```text
ws://<home-assistant-host>:5580/ws
```

Use the host's stable LAN IP or LAN DNS name, not a Compose service name. Matter Server is intentionally outside the project networks and is not routed through Traefik.

This setup supports Wi-Fi Matter devices. Thread devices additionally require a working Thread border router and routable IPv6 connectivity.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Only Matter Server starts | Inspect `proxy` with `docker network inspect proxy`; it must be a bridge on a Compose-only host or an attachable overlay on a Swarm host. |
| Deployment reports a missing service | Run `sudo docker compose --project-directory /opt/home-assistant ps --all` and inspect the failed service's logs. |
| Home Assistant proxy errors | Update `vault.services.home_assistant.proxy` for the local proxy bridge CIDR. |
| Zigbee2MQTT cannot open the coordinator | For local hardware, verify the device path and ownership. For TCP hardware, verify the host and port are reachable. |
| Zigbee2MQTT cannot reach MQTT | Use `mqtt://mqtt:1883` and confirm both services are on the project default network. |
| Home Assistant cannot connect to Matter | Confirm port `5580` is reachable at the host LAN address and Matter Server is healthy. |
| Matter commissioning fails | Check host IPv6, mDNS/multicast, firewall rules, and phone LAN connectivity. |
| Traefik cannot route a service | Confirm Traefik is a local container on the same `proxy` network and its Docker provider is enabled. |

## Security

- Keep `vault.services.home_assistant.proxy` limited to the actual reverse proxy network.
- Do not expose Matter Server port `5580` to the internet.
- Mosquitto currently has no authentication; keep it isolated or add authentication before publishing port `1883`.
- Leave Zigbee pairing disabled outside onboarding windows.
- Back up the entire data root, especially `.storage`, `matter-server/data`, and `zigbee2mqtt/data`.

## Related Documentation

- [Deployments catalog](../README.md)
- [Traefik deployment](../traefik/README.md)
- [Vault template](../../vault.template.yml)
- [Home Assistant Matter integration](https://www.home-assistant.io/integrations/matter/)
- [Matter.js Server Docker documentation](https://github.com/matter-js/matterjs-server/blob/main/docs/docker.md)
