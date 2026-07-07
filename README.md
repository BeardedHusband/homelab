# Homelab Infrastructure

> A production-inspired self-hosted homelab built to develop practical Linux, networking, Docker, and systems administration skills through real-world deployment, maintenance, troubleshooting, and documentation.

---

## Overview

This repository documents the design, deployment, operation, and continuous improvement of my personal homelab.

Rather than simply running self-hosted applications, the goal of this lab is to gain hands-on systems administration experience by building infrastructure I depend on daily, troubleshooting real failures, documenting root causes, and maintaining long-term operational reliability.

Everything here has been deployed, configured, maintained, and documented by me.

---

## Current Environment

### Infrastructure

- Intel i7-12700K Unraid Server
- ~56 TB parity-protected storage
- 32 GB DDR4
- 2 TB NVMe cache
- APC UPS battery backup
- 2.5 GbE LAN
- Static IP addressing
- MoCA backbone
- Dell Linux Mint management server
- Raspberry Pi migration planned

### Scale

- 40–50+ Docker containers
- 23 active Plex users
- Multiple externally accessible services
- WireGuard remote administration
- Cloudflare Tunnel & Access
- Network-wide DNS filtering
- Daily use self-hosted infrastructure

---

# Technologies

## Operating Systems

- Unraid
- Linux Mint
- Windows 11

## Networking

- TCP/IP
- DNS
- DHCP
- WireGuard
- Pi-hole
- Unbound
- Cloudflare Tunnel
- Cloudflare Access
- SSH
- SMB
- NFS
- Static IP Management
- MoCA
- 2.5 Gb Ethernet

## Infrastructure

- Docker
- Docker Compose
- Reverse Proxy
- UPS Monitoring
- SMART Monitoring
- Storage Arrays
- Disaster Recovery
- Backup & Restore

## Services

- Plex
- Nextcloud
- Vaultwarden
- Immich
- Audiobookshelf
- Komga
- Calibre-Web
- Grafana
- PostgreSQL
- MariaDB
- Matrix Synapse
- Pi-hole
- WireGuard

---

# Architecture

Documentation includes:

- Network topology
- Physical infrastructure
- Logical network layout
- Service relationships
- Troubleshooting documentation
- Disaster recovery procedures

> **Architecture diagrams are continually updated as the lab evolves.**

---

## Network Architecture

The following diagram illustrates the physical layout of the homelab infrastructure.

![Physical Network](diagrams/physical-network.png)

For more information see:

- [Network Documentation](docs/network.md)

# Documentation

## Repository Contents

```
docs/
│
├── network.md
├── troubleshooting.md
├── services.md          (planned)
├── security.md          (planned)
├── operations.md        (planned)
└── storage.md           (planned)
```

### Current Documentation

| Document | Description |
|----------|-------------|
| Network | Physical and logical network architecture |
| Troubleshooting | Detailed root-cause analyses and resolutions for real infrastructure issues |

---

# Featured Projects

## Infrastructure Deployment

Designed and maintain an Unraid-based self-hosted environment providing storage, networking, media, backup, and cloud services.

---

## Linux Administration

Manage Linux systems via SSH and NoMachine while hosting critical network infrastructure including Pi-hole, Unbound, and WireGuard.

---

## Docker Infrastructure

Deploy and maintain dozens of Docker containers supporting media, networking, cloud storage, monitoring, databases, and authentication services.

---

## Disaster Recovery

Document recovery procedures for database corruption, failed upgrades, networking issues, VPN failures, Docker failures, and hardware related problems.

---

## Documentation

Every significant issue encountered in this homelab is documented with:

- Symptoms
- Diagnosis
- Root Cause
- Resolution
- Prevention

This repository serves as both operational documentation and a personal knowledge base.

---

# What I've Learned

Through this homelab I've gained practical experience with:

- Linux administration
- Docker management
- Network troubleshooting
- DNS infrastructure
- VPN configuration
- Reverse proxies
- Cloudflare services
- Storage management
- Backup strategies
- Database recovery
- Service monitoring
- Root-cause analysis
- Documentation
- Remote systems administration

---

# Current Roadmap

The lab continues to evolve.

Current projects include:

- Windows Server
- Active Directory
- Entra ID
- Group Policy
- VLAN segmentation
- Ansible
- Kubernetes
- Infrastructure as Code
- PowerShell automation
- Monitoring improvements

---

# Philosophy

This homelab exists for one reason:

> Learn by building.

Every service is deployed with the intention of understanding how it works, how it fails, and how to recover it.

Problems aren't deleted, they're documented.

Failures become runbooks.

The goal isn't simply to host services, but to continuously develop practical systems administration skills through real-world experience.

---

## Contact

**GitHub:** https://github.com/BeardedHusband

```
