# Deployments

[![ansible-lint](https://img.shields.io/github/actions/workflow/status/joe-mccarthy/homelab/ansible-linter.yml?style=flat-square&label=ansible%20lint)](https://github.com/joe-mccarthy/homelab/actions/workflows/ansible-linter.yml) [![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white&style=flat-square)](https://docs.ansible.com/) [![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/compose/) [![Docker Swarm](https://img.shields.io/badge/Docker%20Swarm-Legacy-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/engine/swarm/) [![Traefik](https://img.shields.io/badge/Traefik-Reverse%20Proxy-24A1C1?logo=traefikproxy&logoColor=white&style=flat-square)](https://doc.traefik.io/traefik/)

The `deployments` directory contains standalone Ansible playbooks for services in the home lab. Newer deployments run with Docker Compose on a single target host; legacy deployments still use Docker Swarm. Web applications rely on [Traefik](traefik/README.md), while background services such as DDNS do not.

## Overview

This directory includes deployments for a variety of services, ranging from personal applications to essential cluster management tools. These deployments are designed to:
- Simplify deployment and management across the current Compose and legacy Swarm environments.
- Provide examples of best practices for deploying containerized applications.
- Ensure services are configured with proper proxying, DNS resolution, and HTTPS certificates.

## Compose Files

Single-host Compose deployments retain root-owned projects beneath `/opt/<service>` for normal `docker compose` operations. Legacy deployments that use the shared `copy_and_deploy` role still render a temporary `/home/docker-compose/docker-compose.yaml`, apply it with `community.docker.docker_stack`, and remove it afterward.

## Deployments

### 1. [Cioban](cioban/README.md)
- **Description**: Automates the process of updating Docker services when new image versions are available. It monitors services with specific labels and updates them according to their deployment policies.
- **Use Case**: Ensures services remain up-to-date with minimal manual intervention.
- **Dependencies**: Requires access to the Docker socket and Traefik for proxying.

### 2. [Core Deployments](core-deployments/README.md)
- **Description**: A collection of essential services required for the rest of the deployments to function properly. This includes Traefik and Dynamic DNS.
- **Use Case**: Deploy this playbook first to set up the foundational services for the cluster.
- **Dependencies**: None, but it is recommended to run this playbook before deploying other services.

### 3. [DDNS](ddns/README.md)
- **Description**: Dynamically updates DNS records to reflect the public IP address of the home lab's internet gateway. This ensures services are accessible via domain names.
- **Use Case**: Useful for home labs with dynamic IP addresses.
- **Dependencies**: Runs with Docker Compose on the single `nfs_servers` host and requires a scoped Cloudflare API token. It does not use Traefik.

### 4. [Home Assistant](home-assistant/README.md)
- **Description**: Deploys Home Assistant, an open-source platform for home automation, with Zigbee2MQTT, Mosquitto, and Matter Server for Wi-Fi Matter devices.
- **Use Case**: Perfect for managing and automating smart home devices.
- **Dependencies**: Runs with Docker Compose on the single `nfs_servers` host and requires local Traefik proxying, LAN IPv6/mDNS, and a local or TCP-connected Zigbee coordinator.

### 5. [Immich](immich/README.md)
- **Description**: Immich is a high-performance self-hosted photo and video management solution that serves as a complete alternative to Google Photos. Features include:
  - Web interface and mobile apps for photo browsing and automatic backup
  - AI-powered features including face recognition and object detection
  - Timeline view and album organization
  - External sharing capabilities
- **Services**:
  - **immich-server**: Main application with web interface and API
  - **immich-database**: PostgreSQL with vector extensions for AI features
  - **immich-redis**: Cache and session storage for performance
  - **immich-machine-learning**: AI processing for smart features
- **Use Case**: Ideal for users looking to manage and organize their photo and video collections with advanced AI capabilities.
- **Dependencies**: Runs with Docker Compose on the target host and requires local persistent storage plus the external Traefik `proxy` local bridge network.

### 6. [Paperless](paperless/README.md)
- **Description**: Deploys Paperless-ngx with Redis, Gotenberg, and Tika for document management, OCR, and Office document conversion.
- **Use Case**: Ideal for searchable archival and automated ingestion of scanned documents.
- **Dependencies**: Runs with Docker Compose on the target host and requires local persistent storage plus the external Traefik `proxy` local bridge network.

### 7. [NFS Backup](nfs-backup/README.md)
- **Description**: Runs encrypted Restic backups to S3 from short-lived containers scheduled by systemd; no backup container remains running between jobs.
- **Schedule**: Daily backup at 00:00, weekly retention/prune, and weekly integrity checking.
- **Retention**: Keeps the latest 24 hours plus 14 daily, 8 weekly, 12 monthly, and 3 yearly snapshots.
- **Use Case**: Critical data protection for containerized applications with secure off-site storage
- **Security**: All backups are strongly encrypted and services operate with least-privilege principles
- **Dependencies**: Requires `/exports/docker` on the NFS server, Docker, systemd, and S3-compatible storage credentials

### 8. [Omni Tools](omni/README.md)
- **Description**: Deploys Omni Tools, a self-hosted browser-based collection of everyday utility tools. It provides a lightweight, privacy-friendly alternative to scattered online services.
- **Use Case**: Ideal for users who want a single, self-hosted destination for common utility tasks without relying on third-party websites.
- **Dependencies**: Runs with Docker Compose on the target host and requires a local Traefik container on the external `proxy` bridge network.

### 9. [Traefik](traefik/README.md)
- **Description**: Acts as a reverse proxy for other services, enabling name resolution instead of relying on IP addresses and ports. It also integrates with DNS providers to issue valid HTTPS certificates.
- **Use Case**: A critical component for managing traffic and securing connections in the cluster.
- **Dependencies**: None, but it is recommended to deploy Traefik first.

## Service Versions

| Service | Component | Version |
|---------|-----------|:-------:|
| [Cioban](cioban/README.md) | cioban | `0.17.14` |
| [DDNS](ddns/README.md) | cloudflare-ddns | `1.17.0` |
| [Home Assistant](home-assistant/README.md) | home-assistant | `2026.8.3` |
| [Home Assistant](home-assistant/README.md) | matter-server | `1.4.0` |
| [Home Assistant](home-assistant/README.md) | zigbee2mqtt | `2.13.0` |
| [Home Assistant](home-assistant/README.md) | mosquitto | `2.1.2-alpine` |
| [Immich](immich/README.md) | immich-server | `v3.1.0` |
| [Immich](immich/README.md) | immich-machine-learning | `v3.1.0` |
| [NFS Backup](nfs-backup/README.md) | resticker | `1.8.2` |
| [Paperless](paperless/README.md) | paperless-ngx | `3.1.0` |
| [Paperless](paperless/README.md) | redis | `8.10.1` |
| [Paperless](paperless/README.md) | gotenberg | `8.36.0` |
| [Paperless](paperless/README.md) | tika | `3.3.1.0` |
| [Omni Tools](omni/README.md) | omni-tools | `0.6.0` |
| [Traefik](traefik/README.md) | traefik | `3.7.12` |

## Prerequisites

Before deploying any services, ensure the following:
1. **Docker Runtime**:
   - Compose deployments require a target host with Docker Engine and the Compose v2 plugin, selected with `-e target=<host-or-address>`.
   - Legacy Swarm deployments still require an initialized cluster and a manager node.

2. **Traefik Deployment**:
   - Deploy Traefik first to handle proxying and certificate management for other services.

3. **Ansible Inventory**:
   - Ensure [`inventory.yml`](../inventory.example.yml) defines all nodes in the cluster and groups them appropriately. Use [`inventory.example.yml`](../inventory.example.yml) at the repo root as a starting point.

4. **DNS Configuration**:
   - Set up DNS records for the services you plan to deploy. Use Dynamic DNS if your public IP address changes frequently.

## Usage

Run the service playbook from the repository root. For example, to deploy Omni Tools:

```bash
ansible-playbook -i inventory.yml deployments/omni/deploy.yml \
  -e target=odin --extra-vars @vault.yml --ask-vault-pass
```
