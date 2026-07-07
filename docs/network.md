# Network Architecture

This document describes the physical and logical network architecture of my homelab, including infrastructure, management systems, networking decisions, and remote access strategy.

---

# Design Goals

This network was built with the following objectives:

- Reliable self-hosted services
- Secure remote administration
- High-speed local file transfers
- Simple management
- Practical systems administration experience
- Minimal public attack surface
- Easy disaster recovery

---

# Physical Topology

```text
                      Internet
                          │
              White River Connect Fiber
                          │
                  Netgear BE9300 Router
                          │
                    2.5 GbE Switch
          ┌───────────────┴───────────────┐
          │                               │
     Unraid Server                Linux Mint Server
          │                               │
          │                     Pi-hole / Unbound
          │                     WireGuard VPN
          │
     MoCA Backbone
          │
     Gaming Desktop
          │
      Steam Deck
```

---

# Core Infrastructure

| Device | Purpose |
|----------|----------|
| Netgear BE9300 | Primary router |
| 2.5 GbE Switch | High-speed LAN connectivity |
| Unraid Server | Primary virtualization and storage platform |
| Linux Mint Server | Network services, DNS, VPN, remote management |
| Gaming Desktop | Primary workstation |
| Steam Deck | Remote game streaming client |

---

# IP Addressing

Static IPs are assigned to all infrastructure devices to simplify management and ensure services remain accessible after reboots.

| Device | Address |
|----------|----------|
| Router | 192.168.1.254 |
| Linux Mint | 192.168.1.20 |
| Unraid | 192.168.1.66 |
| Gaming Desktop | 192.168.1.125 |

---

# Networking Services

## Pi-hole

Purpose

- Network-wide DNS filtering
- Centralized DNS management
- Local DNS resolution

Why

Provides centralized DNS administration while reducing unwanted advertising and telemetry across the network.

---

## Unbound

Purpose

Recursive DNS resolver.

Why

Removes dependency on third-party upstream DNS providers and improves privacy.

---

## WireGuard

Purpose

Secure VPN access to the home network.

Why

Allows encrypted remote administration without exposing management interfaces directly to the Internet.

Primary uses:

- SSH
- NoMachine
- Unraid
- Internal web interfaces

---

## Cloudflare Tunnel

Purpose

Securely publish selected services without opening inbound firewall ports.

Why

Cloudflare Tunnel allows remote access while significantly reducing the public attack surface compared to traditional port forwarding.

---

# Infrastructure Services

Primary Docker services currently include:

| Service | Purpose |
|----------|----------|
| Plex | Media streaming |
| Audiobookshelf | Audiobook server |
| Nextcloud | Personal cloud storage |
| Vaultwarden | Password management |
| Immich | Photo backup |
| Grafana | Infrastructure monitoring |
| Nginx Proxy Manager | Reverse proxy |
| PostgreSQL | Database services |
| MariaDB | Database services |
| Matrix Synapse | Messaging |

---

# Storage

Current storage platform:

- ~56 TB protected storage
- Dual parity
- 2 TB NVMe cache
- SMB shares
- Docker appdata
- Daily backups
- SMART monitoring
- UPS battery backup

---

# Remote Administration

Infrastructure is managed remotely using:

- SSH
- NoMachine
- WireGuard VPN
- Unraid Web UI
- Cloudflare Tunnel (selected services only)

Management interfaces are not exposed directly to the Internet.

---

# Monitoring

Infrastructure health is monitored using:

- Grafana
- Plex Dash
- SMART monitoring
- UPS monitoring
- Docker status
- System resource utilization

---

# Security Design

Current security practices include:

- Static IP assignments
- WireGuard VPN
- Cloudflare Tunnel
- Cloudflare Access
- Pi-hole DNS
- Password management with Vaultwarden
- Regular Docker updates
- Daily backups

Future improvements:

- VLAN segmentation
- Windows Active Directory lab
- Centralized authentication
- Infrastructure as Code
- Enhanced monitoring

---

# Current Scale

- ~56 TB protected storage
- 30–40 Docker containers
- 23 active Plex users
- Multiple self-hosted services
- 2.5 GbE LAN
- Secure remote administration
- Daily operational use

---

# Design Philosophy

This network is designed to provide practical systems administration experience through daily operation and maintenance.

Rather than building isolated demonstrations, services are deployed for real-world use. Problems are documented, root causes investigated, and recovery procedures recorded to continuously improve both the infrastructure and my understanding of Linux, networking, and systems administration.
