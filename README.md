# 🏠 Home Lab

[![Issues and PRs](https://img.shields.io/github/issues/joe-mccarthy/homelab?style=flat-square)](https://github.com/joe-mccarthy/homelab/issues)
[![Release](https://img.shields.io/github/v/release/joe-mccarthy/homelab?style=flat-square)](https://github.com/joe-mccarthy/homelab/releases)
[![Last commit](https://img.shields.io/github/last-commit/joe-mccarthy/homelab?style=flat-square)](https://github.com/joe-mccarthy/homelab/commits/main)
[![License](https://img.shields.io/github/license/joe-mccarthy/homelab?style=flat-square)](LICENSE)
[![ansible-lint](https://img.shields.io/github/actions/workflow/status/joe-mccarthy/homelab/ansible-linter.yml?style=flat-square&label=ansible%20lint)](https://github.com/joe-mccarthy/homelab/actions/workflows/ansible-linter.yml)

An Ansible-powered Raspberry Pi home lab for running and maintaining self-hosted services with Docker Compose.

This repository contains the playbooks, roles, templates, and documentation I use to bootstrap machines, deploy services, manage local persistent storage, and schedule backups. It is built for learning and experimentation, but it is structured like real infrastructure so it stays repeatable instead of becoming a pile of one-off shell commands.

> [!WARNING]
> This repository is intended for home lab, learning, testing, and development use. Review every variable, secret, network rule, and exposed service before adapting anything for a public or production environment.

---

## ✨ What This Lab Does

- 🐳 Installs Docker Engine and runs services with [Docker Compose](https://docs.docker.com/compose/) on standalone hosts.
- 🤖 Uses [Ansible](https://docs.ansible.com/ansible/latest/index.html) for repeatable host provisioning and service deployment.
- 💾 Keeps application data in local persistent directories and backs it up with Restic.
- 🌐 Routes services through [Traefik](https://doc.traefik.io/traefik/) with domain-based access and HTTPS.
- 🔐 Keeps sensitive values in [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html).
- 🧰 Includes ready-to-run deployments for self-hosted apps, backups, and host bootstrapping.

---

## 🧭 Repository Map

| Path | Purpose |
| --- | --- |
| [`home_server/`](home_server/README.md) | Bootstrap Docker hosts, update packages, report status, reboot, and mount application storage. |
| [`deployments/`](deployments/README.md) | Service deployments, Docker Compose templates, roles, and per-service docs. |
| [`vault.template.yml`](vault.template.yml) | Complete reference for expected secret values. |
| [`requirements.yml`](requirements.yml) | Required Ansible collections. |

---

## 🧱 Architecture

The lab uses Raspberry Pi hosts running Debian or Ubuntu. Applications run with Docker Compose on a selected host, with Traefik routing web traffic over a local `proxy` bridge network. Persistent application data lives under `/exports/docker`, and Compose projects are retained beneath `/opt`.

### Current Hardware

- 8 Raspberry Pi 4s with 8 GB RAM and 64 GB SD cards.
- 1 Raspberry Pi 5 with 8 GB RAM and a 2 TB NVMe drive on a [Pimoroni NVMe Base](https://shop.pimoroni.com/products/nvme-base?variant=41219587178579).
- PoE HATs for the Raspberry Pi 4 nodes.
- An 8-port PoE switch for power and networking.

### Deployment Targets

The bootstrap and application deployment playbooks select a host with `-e target=<host-or-address>`. A reachable DNS name or IP address can be used directly, with `-u <ssh-user>` for the connection account. Pass your own inventory with `-i inventory.yml` when using inventory aliases or host-specific settings.

NFS Backup requires an inventory defining exactly one host in `nfs_servers`. See its [host and inventory instructions](deployments/nfs-backup/README.md#host-and-inventory) for the required structure.

---

## 🚀 Quick Start

These commands assume you are running from the root of this repository and have already installed an operating system on the target host. Replace `192.168.1.50` and `pi` with its address and SSH user.

### 1. Install Ansible Collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Create and Encrypt Your Vault

```bash
cp vault.template.yml vault.yml
# edit vault.yml with your real values
ansible-vault encrypt vault.yml
```

The vault template documents every secret expected by the deployment stack, including Cloudflare credentials, registry credentials, service passwords, backup keys, and application-specific values.

### 3. Bootstrap the Docker Host

The target must be reachable over SSH with a sudo-capable account. Add `--ask-pass` if password authentication is required.

```bash
ansible-playbook home_server/setup.yml \
  -e target=192.168.1.50 -u pi \
  --extra-vars @vault.yml --ask-vault-pass --ask-become-pass
```

This installs Docker Engine and the Compose plugin, configures the Docker daemon, creates local storage roots and the `proxy` bridge, and logs in to configured registries. See the [home-server bootstrap guide](home_server/README.md) for details.

For a separate NVMe/SSD data filesystem, configure the [application storage mount](home_server/README.md#mount-application-storage-optional) before deploying services.

### 4. Deploy Routing and DNS

```bash
ansible-playbook deployments/traefik/deploy.yml \
  -e target=192.168.1.50 -u pi --extra-vars @vault.yml --ask-vault-pass
ansible-playbook deployments/ddns/deploy.yml \
  -e target=192.168.1.50 -u pi --extra-vars @vault.yml --ask-vault-pass
```

Deploy Traefik before the web applications so HTTPS routing is available. Configure scheduled backups separately using the [NFS Backup guide](deployments/nfs-backup/README.md), including its inventory and repository setup instructions.

---

## 📦 Service Catalog

The full service catalog lives in [`deployments/README.md`](deployments/README.md). Each deployment has its own README, variables, templates, and playbook.

| Service | What It Provides |
| --- | --- |
| [Traefik](deployments/traefik/README.md) | Reverse proxy, routing, and HTTPS certificate handling. |
| [DDNS](deployments/ddns/README.md) | Dynamic DNS updates for home internet connections. |
| [Home Assistant](deployments/home-assistant/README.md) | Smart home automation with Zigbee, MQTT, and Wi-Fi Matter support. |
| [Immich](deployments/immich/README.md) | Self-hosted photo and video management. |
| [Paperless](deployments/paperless/README.md) | Document management and OCR workflow. |
| [Omni Tools](deployments/omni/README.md) | Self-hosted everyday browser utilities. |
| [NFS Backup](deployments/nfs-backup/README.md) | Restic-based backups for local application data. |

To deploy an application:

```bash
ansible-playbook deployments/<service>/deploy.yml \
  -e target=<host-or-address> -u <ssh-user> --extra-vars @vault.yml --ask-vault-pass
```

For example:

```bash
ansible-playbook deployments/immich/deploy.yml \
  -e target=192.168.1.50 -u pi --extra-vars @vault.yml --ask-vault-pass
```

---

## 🔐 Secrets and Configuration

Sensitive values are intentionally kept out of normal group variable files. The pattern used throughout the repository is:

```yml
cf_token: "{{ vault.shared.cloudflare.token }}"
```

If you do not want to use Ansible Vault, you can replace vault lookups with literal values, but those files should stay private.

The most important shared value is the base domain used by Traefik:

```yml
vault:
  shared:
    general:
      domain: "example.com"
```

You can keep the domain in `vault.yml`, put it in `inventory.yml`, or encrypt it as an individual vault string. Prefer the smallest amount of plain-text configuration that still keeps your workflow practical.

---

## 🧪 Philosophy

This lab is intentionally practical:

- Keep infrastructure documented in the same repository as the automation.
- Prefer repeatable playbooks over manual node-by-node changes.
- Keep application data in explicit local paths with tested backups.
- Keep secret material centralized and encrypted.
- Use the lab as a place to learn real operational patterns without pretending it is production.

---

## 🤝 Contributions

This is a personal home lab built in public. I am sharing what I have learned, and there will always be room to improve the structure, playbooks, defaults, documentation, and service templates.

Issues, suggestions, and pull requests are welcome:

### Issue Types

Use the guided issue forms when opening work:

| Type | Use It For |
| --- | --- |
| 🐛 Bug Report | Broken playbooks, deployments, automation, or unexpected behavior. |
| ✨ Feature Request | New playbooks, roles, automation, or improvements to existing workflows. |
| 🚀 Deployment Request | New self-hosted services that should live under `deployments/`. |
| 📚 Documentation | README, usage, runbook, or troubleshooting improvements. |
| 🔧 Maintenance | Refactoring, cleanup, linting, workflow, or repository maintenance. |
| 📦 Dependency Update | Ansible collections, container images, GitHub Actions, or upstream versions. |

### Pull Request Types

Use the default pull request template for most changes. Dedicated PR templates
are also available for deployments, documentation-only changes, and maintenance
work.

Before opening a pull request:

1. Fork the project.
2. Create a feature branch: `git checkout -b feature/amazing-feature`.
3. Commit your changes: `git commit -m "Add amazing feature"`.
4. Run validation where relevant, especially `ansible-lint`.
5. Push the branch: `git push origin feature/amazing-feature`.
6. Open a pull request and choose the template that best matches the change.

---

## 📄 License

This project is licensed under the MIT License. See [`LICENSE`](LICENSE) for details.
