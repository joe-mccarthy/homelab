# Core Deployments

[![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white&style=flat-square)](https://docs.ansible.com/) [![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/compose/) [![Traefik](https://img.shields.io/badge/Traefik-Core%20Dependency-24A1C1?logo=traefikproxy&logoColor=white&style=flat-square)](https://doc.traefik.io/traefik/) ![DDNS](https://img.shields.io/badge/DDNS-Core%20Dependency-1F6FEB?style=flat-square)

Before deploying other services in this repository, deploy these core services first. The playbook imports Traefik, DDNS, and NFS Backup in sequence on the single `nfs_servers` host. Traefik and DDNS use Docker Compose, while NFS Backup runs short-lived containers scheduled by systemd timers.

## Traefik

[Traefik](https://traefik.io/) is responsible for acting as a reverse proxy for other services, enabling name resolution instead of relying on IP addresses and ports. Additionally, it integrates with DNS providers to perform challenges and issue valid certificates for HTTPS connections. For more details, refer to the [Traefik deployment README](../traefik/README.md).

## Dynamic DNS

Dynamic DNS ensures that services hosted on the home lab cluster are accessible by updating DNS records to reflect the public IP address of your internet gateway. This service uses the Cloudflare API (assuming Cloudflare is your DNS provider) to update domain name entries dynamically. For more details, refer to the [Dynamic DNS deployment README](../ddns/README.md).

## NFS Backup

NFS Backup schedules encrypted Restic backups of the local application data to S3-compatible storage, along with retention and integrity checks. For configuration and repository initialization instructions, refer to the [NFS Backup deployment README](../nfs-backup/README.md).
