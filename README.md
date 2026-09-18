# 🏠 Home Lab

[![Issues and PRs](https://img.shields.io/github/issues/joe-mccarthy/homelab?style=flat-square)](https://github.com/joe-mccarthy/homelab/issues)
[![Release](https://img.shields.io/github/v/release/joe-mccarthy/homelab?style=flat-square)](https://github.com/joe-mccarthy/homelab/releases)
[![Last commit](https://img.shields.io/github/last-commit/joe-mccarthy/homelab?style=flat-square)](https://github.com/joe-mccarthy/homelab/commits/main)
[![License](https://img.shields.io/github/license/joe-mccarthy/homelab?style=flat-square)](LICENSE)
[![ansible-lint](https://img.shields.io/github/actions/workflow/status/joe-mccarthy/homelab/ansible-linter.yml?style=flat-square&label=ansible%20lint)](https://github.com/joe-mccarthy/homelab/actions/workflows/ansible-linter.yml)

An Ansible-powered Raspberry Pi home lab for building, running, and maintaining a small Docker Swarm cluster.

This repository contains the playbooks, roles, templates, and documentation I use to bootstrap machines, create a Swarm, deploy services, manage persistent storage, and keep the cluster healthy. It is built for learning and experimentation, but it is structured like real infrastructure so it stays repeatable instead of becoming a pile of one-off shell commands.

> [!WARNING]
> This repository is intended for home lab, learning, testing, and development use. Review every variable, secret, network rule, and exposed service before adapting anything for a public or production environment.

---

## ✨ What This Lab Does

- 🐳 Creates and manages a multi-node [Docker Swarm](https://docs.docker.com/engine/swarm/) cluster.
- 🤖 Uses [Ansible](https://docs.ansible.com/ansible/latest/index.html) for repeatable provisioning, deployment, and maintenance.
- 💾 Provides shared persistent storage through [NFS](https://en.wikipedia.org/wiki/Network_File_System).
- 🌐 Routes services through [Traefik](https://doc.traefik.io/traefik/) with domain-based access and HTTPS.
- 🔐 Keeps sensitive values in [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html).
- 🧰 Includes ready-to-run deployments for self-hosted apps, backups, automation, and cluster operations.

---

## 🧭 Repository Map

| Path | Purpose |
| --- | --- |
| [`docker-swarm/`](docker-swarm/README.md) | Create, manage, and destroy the Docker Swarm cluster. |
| [`deployments/`](deployments/README.md) | Service deployments, Docker Compose templates, roles, and per-service docs. |
| [`maintenance/`](maintenance/README.md) | Operational playbooks for updates, setup, shutdown, restarts, and Docker registry login. |
| [`inventory.example.yml`](inventory.example.yml) | Example Ansible inventory for managers, workers, and the NFS host. |
| [`vault.template.yml`](vault.template.yml) | Complete reference for expected secret values. |
| [`requirements.yml`](requirements.yml) | Required Ansible collections. |

---

## 🧱 Architecture

The lab is designed around Raspberry Pi nodes running Ubuntu and participating in a Docker Swarm.

### Current Hardware

- 8 Raspberry Pi 4s with 8 GB RAM and 64 GB SD cards.
- 1 Raspberry Pi 5 with 8 GB RAM and a 2 TB NVMe drive on a [Pimoroni NVMe Base](https://shop.pimoroni.com/products/nvme-base?variant=41219587178579).
- PoE HATs for the Raspberry Pi 4 nodes.
- An 8-port PoE switch for power and networking.

### Cluster Shape

The example inventory models the cluster with three Ansible groups:

| Group | Role |
| --- | --- |
| `nfs_servers` | Hosts the NFS export used for persistent Docker volumes. |
| `manager` | Docker Swarm manager nodes. |
| `docker` | Docker Swarm worker nodes. |

Both `manager` and `docker` sit under the `cluster` group so maintenance playbooks can target every node together.

In the sample setup, `odin` acts as the first manager and NFS server, `thor` and `loki` are additional managers, and the remaining nodes are workers. The Raspberry Pi 5 is the natural fit for NFS because the NVMe drive gives services durable storage without hammering SD cards.

> [!TIP]
> Use an odd number of Swarm managers. A cluster with `N` managers can tolerate the loss of at most `(N - 1) / 2` managers, and Docker recommends no more than seven manager nodes.

---

## 🚀 Quick Start

These commands assume you are running from the root of this repository and have already installed an operating system on each node.

### 1. Install Ansible Collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Create Your Inventory

```bash
cp inventory.example.yml inventory.yml
```

Edit `inventory.yml` with your node names, IP addresses, SSH users, and Swarm labels.

### 3. Create and Encrypt Your Vault

```bash
cp vault.template.yml vault.yml
# edit vault.yml with your real values
ansible-vault encrypt vault.yml
```

The vault template documents every secret expected by the deployment stack, including Cloudflare credentials, registry credentials, service passwords, backup keys, and application-specific values.

### 4. Generate a Cluster SSH Key

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com" -f ~/.ssh/homelab
```

### 5. Prepare the Machines

```bash
ansible-playbook -i inventory.yml maintenance/set-up-machine/setup.yml --ask-pass --ask-become-pass
```

This bootstraps new nodes with SSH access, package updates, common tools, and any required reboot.

### 6. Create the Swarm

```bash
ansible-playbook -i inventory.yml docker-swarm/create.yml --ask-vault-pass
```

This installs Docker, configures NFS, initializes the Swarm, joins managers and workers, applies labels, and creates the shared proxy network.

### 7. Deploy the Core Services

```bash
ansible-playbook -i inventory.yml deployments/core-deployments/deploy.yml \
  --extra-vars @vault.yml --ask-vault-pass
```

The core deployment brings up the foundation used by the rest of the lab, including routing and DNS support.

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
| [NFS Backup](deployments/nfs-backup/README.md) | Restic-based backups for shared NFS data. |

To deploy a single service:

```bash
ansible-playbook -i inventory.yml deployments/<service>/deploy.yml \
  -e target=<host-or-address> --extra-vars @vault.yml --ask-vault-pass
```

For example:

```bash
ansible-playbook -i inventory.yml deployments/immich/deploy.yml \
  -e target=odin --extra-vars @vault.yml --ask-vault-pass
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

## 🛠️ Day-to-Day Operations

### Update Every Node

```bash
ansible-playbook -i inventory.yml maintenance/update.yml --ask-become-pass
```

### Log Into Private Docker Registries

```bash
ansible-playbook -i inventory.yml maintenance/docker-login/login.yml --ask-vault-pass
```

### Gracefully Shut Down the Cluster

```bash
ansible-playbook -i inventory.yml maintenance/shutdown-cluster.yml --ask-become-pass
```

### Tear Down the Swarm

```bash
ansible-playbook -i inventory.yml docker-swarm/destroy.yml --ask-vault-pass
```

Use destructive playbooks with care. Back up important data first, especially anything under shared NFS volumes.

---

## 🧪 Philosophy

This lab is intentionally practical:

- Keep infrastructure documented in the same repository as the automation.
- Prefer repeatable playbooks over manual node-by-node changes.
- Make services movable by using shared storage and Swarm placement rules.
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
