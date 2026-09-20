# Deployments

[![ansible-lint](https://img.shields.io/github/actions/workflow/status/joe-mccarthy/homelab/ansible-linter.yml?style=flat-square&label=ansible%20lint)](https://github.com/joe-mccarthy/homelab/actions/workflows/ansible-linter.yml) [![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white&style=flat-square)](https://docs.ansible.com/) [![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/compose/) [![Traefik](https://img.shields.io/badge/Traefik-Reverse%20Proxy-24A1C1?logo=traefikproxy&logoColor=white&style=flat-square)](https://doc.traefik.io/traefik/)

The `deployments` directory contains standalone Ansible playbooks for services in the home lab. Applications run with Docker Compose on a single target host, and NFS Backup uses systemd-scheduled Docker containers. Web applications rely on [Traefik](traefik/README.md), while background services such as DDNS do not.

## Overview

This directory includes deployments for personal applications, routing, DNS, and backups. These deployments are designed to:
- Simplify deployment and management on a single Docker host.
- Provide examples of best practices for deploying containerized applications.
- Ensure services are configured with proper proxying, DNS resolution, and HTTPS certificates.

## Compose Files

Single-host Compose deployments retain root-owned projects beneath `/opt/<service>` for normal `docker compose` operations.

Long-running web, database, cache, broker, and conversion containers expose
service-specific health checks. Compose startup waits for healthy hard
dependencies, while independently reconnectable integrations are not ordered.
The DDNS container is the exception: its hardened scratch image contains only
the updater binary and exposes no local readiness interface, so Docker's
running state and the updater logs are its health signals.

## Deployments

### 1. [DDNS](ddns/README.md)
- **Description**: Dynamically updates DNS records to reflect the public IP address of the home lab's internet gateway. This ensures services are accessible via domain names.
- **Use Case**: Useful for home labs with dynamic IP addresses.
- **Dependencies**: Runs with Docker Compose on the target host and requires a scoped Cloudflare API token. It does not use Traefik.

### 2. [Home Assistant](home-assistant/README.md)
- **Description**: Deploys Home Assistant, an open-source platform for home automation, with Zigbee2MQTT, Mosquitto, and Matter Server for Wi-Fi Matter devices.
- **Use Case**: Perfect for managing and automating smart home devices.
- **Dependencies**: Runs with Docker Compose on the target host and requires local Traefik proxying, LAN IPv6/mDNS, and a local or TCP-connected Zigbee coordinator.

### 3. [Homebox](homebox/README.md)
- **Description**: Deploys Homebox, a self-hosted home inventory and organization system, with PostgreSQL.
- **Use Case**: Ideal for cataloging household items, locations, warranties, and attachments.
- **Dependencies**: Runs with Docker Compose on the target host and requires local persistent storage plus the external Traefik `proxy` local bridge network.

### 4. [Immich](immich/README.md)
- **Description**: Immich is a high-performance self-hosted photo and video management solution that serves as a complete alternative to Google Photos. Features include:
  - Web interface and mobile apps for photo browsing and automatic backup
  - AI-powered features including face recognition and object detection
  - Timeline view and album organization
  - External sharing capabilities
- **Services**:
  - **immich-server**: Main application with web interface and API
  - **immich-database**: PostgreSQL with vector extensions for AI features
  - **immich-redis**: Valkey cache and job coordination
  - **immich-machine-learning**: AI processing for smart features
- **Use Case**: Ideal for users looking to manage and organize their photo and video collections with advanced AI capabilities.
- **Dependencies**: Runs with Docker Compose on the target host and requires local persistent storage plus the external Traefik `proxy` local bridge network.

### 5. [Paperless](paperless/README.md)
- **Description**: Deploys Paperless-ngx with PostgreSQL, Redis, Gotenberg, and Tika for document management, OCR, and Office document conversion.
- **Use Case**: Ideal for searchable archival and automated ingestion of scanned documents.
- **Dependencies**: Runs with Docker Compose on the target host and requires local persistent storage plus the external Traefik `proxy` local bridge network.

### 6. [NFS Backup](nfs-backup/README.md)
- **Description**: Runs encrypted Restic backups to S3 from short-lived containers scheduled by systemd; no backup container remains running between jobs.
- **Schedule**: Daily backup at 00:00, Sunday retention/prune at 03:30, and Sunday integrity checking at 08:30, all in `Europe/London`.
- **Retention**: Keeps every snapshot within one day of the newest snapshot, plus 14 daily, 8 weekly, 12 monthly, and 3 yearly representatives. These selections overlap.
- **Use Case**: Off-site protection for persistent application data.
- **Data Handling**: Restic encrypts snapshots, and the source directory is mounted read-only in backup containers.
- **Dependencies**: Requires local application data (default `/exports/docker`) on the single `nfs_servers` inventory host, Docker, systemd, and S3-compatible storage credentials.

### 7. [Omni Tools](omni/README.md)
- **Description**: Deploys Omni Tools, a self-hosted browser-based collection of everyday utility tools. It provides a lightweight, privacy-friendly alternative to scattered online services.
- **Use Case**: Ideal for users who want a single, self-hosted destination for common utility tasks without relying on third-party websites.
- **Dependencies**: Runs with Docker Compose on the target host and requires a local Traefik container on the external `proxy` bridge network.

### 8. [Traefik](traefik/README.md)
- **Description**: Routes requests to local containers using hostname rules and obtains HTTPS certificates through Cloudflare DNS-01 challenges.
- **Use Case**: Manages traffic and secures connections to the home lab applications.
- **Dependencies**: Requires Docker Compose, the external local `proxy` bridge, Cloudflare credentials, and a configured domain. Deploy it before the web applications.

## Service Versions

These are the image tags configured in each service's `group_vars/all.yml`.
Immich's database and Valkey images also have digest pins in that file.

| Service | Component | Image tag |
|---------|-----------|:-------:|
| [DDNS](ddns/README.md) | cloudflare-ddns | `1.17.0` |
| [Home Assistant](home-assistant/README.md) | home-assistant | `2026.8.3` |
| [Home Assistant](home-assistant/README.md) | matter-server | `1.4.0` |
| [Home Assistant](home-assistant/README.md) | zigbee2mqtt | `2.13.0` |
| [Home Assistant](home-assistant/README.md) | mosquitto | `2.1.2-alpine` |
| [Homebox](homebox/README.md) | homebox | `0.26.2-rootless` |
| [Homebox](homebox/README.md) | PostgreSQL | `17.6-alpine` |
| [Immich](immich/README.md) | immich-server | `v3.1.0` |
| [Immich](immich/README.md) | immich-machine-learning | `v3.1.0` |
| [Immich](immich/README.md) | immich-redis (Valkey) | `9` (digest-pinned) |
| [Immich](immich/README.md) | immich-database | `14-vectorchord0.4.3-pgvectors0.2.0` (digest-pinned) |
| [NFS Backup](nfs-backup/README.md) | resticker | `1.8.2` |
| [Paperless](paperless/README.md) | paperless-ngx | `3.1.0` |
| [Paperless](paperless/README.md) | redis | `8.10.1` |
| [Paperless](paperless/README.md) | PostgreSQL | `18.6-bookworm` |
| [Paperless](paperless/README.md) | gotenberg | `8.36.0` |
| [Paperless](paperless/README.md) | tika | `3.3.1.0` |
| [Omni Tools](omni/README.md) | omni-tools | `0.6.0` |
| [Traefik](traefik/README.md) | traefik | `3.7.12` |

## Prerequisites

Before deploying any services, ensure the following:
1. **Docker Runtime**:
   - Compose deployments require a target host with Docker Engine and the Compose v2 plugin, selected with `-e target=<host-or-address>`.
   - NFS Backup requires exactly one host in `nfs_servers` with Docker Engine and systemd.

2. **Traefik Deployment**:
   - Bootstrap the local `proxy` bridge with [`home_server/setup.yml`](../home_server/setup.yml), then deploy Traefik before the web applications. Traefik and the applications it routes must share that bridge on the same Docker host.

3. **Ansible Inventory**:
   - Application deployments can target a reachable hostname or IP address directly with `-e target=...` and `-u <ssh-user>`. Supply your own inventory with `-i inventory.yml` when using inventory aliases or host-specific settings.
   - NFS Backup requires an inventory with one host in `nfs_servers`; see its [host and inventory instructions](nfs-backup/README.md#host-and-inventory).

4. **DNS Configuration**:
   - Set up DNS records for the services you plan to deploy. Use Dynamic DNS if your public IP address changes frequently.

5. **Configuration and Data**:
   - Pass the encrypted vault explicitly with `--extra-vars @vault.yml --ask-vault-pass`, unless your inventory already loads the same variables.
   - Homebox, Immich, and Paperless require existing data by default. Use their service-specific fresh-install option only when initializing a new installation.

## Usage

Run the service playbook from the repository root. For example, to deploy Omni Tools:

```bash
ansible-playbook deployments/omni/deploy.yml \
  -e target=192.168.1.50 -u pi \
  --extra-vars @vault.yml --ask-vault-pass --ask-become-pass
```

Replace the address and SSH user. Add `--ask-pass` for SSH password authentication;
omit `--ask-become-pass` when sudo is passwordless. See the service README for
first-run settings and operations.

## Retained Compose Projects

Run Compose commands on the application host using these defaults:

| Deployment | Project name | Project directory |
| --- | --- | --- |
| DDNS | `ddns` | `/opt/ddns` |
| Home Assistant | `home_assistant` | `/opt/home-assistant` |
| Homebox | `homebox` | `/opt/homebox` |
| Immich | `immich` | `/opt/immich` |
| Omni Tools | `omni` | `/opt/omni` |
| Paperless | `paperless` | `/opt/paperless` |
| Traefik | `traefik` | `/opt/traefik` |

Home Assistant's directory and project names differ, so specify its project name:

```bash
sudo docker compose --project-directory /opt/home-assistant \
  --project-name home_assistant ps
```

NFS Backup is managed through its systemd timers and `nfs-backup-job` runner;
see the [backup operations guide](nfs-backup/README.md#routine-operations).
