# Core Deployments

[![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white&style=flat-square)](https://docs.ansible.com/) [![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/compose/) [![Docker Swarm](https://img.shields.io/badge/Docker%20Swarm-Legacy-2496ED?logo=docker&logoColor=white&style=flat-square)](https://docs.docker.com/engine/swarm/) [![Traefik](https://img.shields.io/badge/Traefik-Core%20Dependency-24A1C1?logo=traefikproxy&logoColor=white&style=flat-square)](https://doc.traefik.io/traefik/) ![DDNS](https://img.shields.io/badge/DDNS-Core%20Dependency-1F6FEB?style=flat-square)

Before deploying other services in this repository, deploy these core services first. Traefik and DDNS run with Docker Compose on the single `nfs_servers` host; Cioban remains a legacy Swarm deployment. This playbook imports each individual deployment in sequence.

## Traefik

[Traefik](https://traefik.io/) is responsible for acting as a reverse proxy for other services, enabling name resolution instead of relying on IP addresses and ports. Additionally, it integrates with DNS providers to perform challenges and issue valid certificates for HTTPS connections. For more details, refer to the [Traefik deployment README](../traefik/README.md).

## Dynamic DNS

Dynamic DNS ensures that services hosted on the home lab cluster are accessible by updating DNS records to reflect the public IP address of your internet gateway. This service uses the Cloudflare API (assuming Cloudflare is your DNS provider) to update domain name entries dynamically. For more details, refer to the [Dynamic DNS deployment README](../ddns/README.md).

## Cioban

[Cioban](https://github.com/cioban) is responsible for automatically updating Docker services, including itself. It requires access to the Docker socket and relies on labels applied to services to determine which ones to update. When a new image for a deployed service is detected, Cioban updates the service to use the new image, adhering to the deployment policies defined for that service. For more details, refer to the [Cioban deployment README](../cioban/README.md).
