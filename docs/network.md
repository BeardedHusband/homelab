# Network Overview

This document describes the physical and logical network architecture of my homelab, including infrastructure, management devices, and self-hosted services.

## Internet

```text
Internet
    │
White River Connect Fiber
    │
Netgear BE9300 Router
    │
2.5 GbE Switch
    ├───────────────┐
    │               │
Unraid Server   Linux Mint Laptop
                    │
             Pi-hole / WireGuard
                    │
               MoCA Backbone
                    │
             Gaming Desktop
                    │
       Steam Deck (Moonlight)
```

---

## Technical Skills

**Operating Systems**
- Linux Mint
- Unraid
- Windows 11

**Networking**
- Static IP addressing
- WireGuard VPN
- Pi-hole DNS
- Cloudflare Tunnel
- MoCA
- SSH

**Virtualization & Containers**
- Docker
- Docker Compose
- Container troubleshooting

**Storage**
- SMB Shares
- RAID/Parity
- Backup & Restore
- UPS monitoring

**Monitoring**
- Grafana
- Plex Dash
- System monitoring

## Infrastructure

- 2.5 GbE LAN
- 2.5 GbE Switch
- MoCA Backbone
- Static IP Assignments
- Pi-hole DNS
- WireGuard VPN
- Cloudflare Tunnel

---

## Core Devices

| Device | Address | Purpose |
|---------|---------|---------|
| Netgear BE9300 | 192.168.1.254 | Router |
| Unraid Server | 192.168.1.66 | Primary server |
| Linux Mint Laptop | 192.168.1.20 | Headless Linux management server |
| Gaming Desktop | 192.168.1.125 | Primary workstation |

---

## Management

### Linux Mint Laptop

- Headless
- NoMachine remote desktop
- SSH administration
- Hosts Pi-hole
- Hosts WireGuard

### Unraid Server

- Web GUI
- SSH access
- Docker host
- UPS monitored

---

## Self Hosted Services

| Service | Purpose |
|---------|---------|
| Plex | Media streaming |
| Audiobookshelf | Audiobook server |
| Nextcloud | Personal cloud |
| Vaultwarden | Password management |
| Immich | Photo backup |
| Pi-hole | DNS filtering |
| WireGuard | Remote VPN |
| Grafana | Monitoring |
| Nginx Proxy Manager | Reverse proxy |
| Cloudflare Tunnel | Secure remote access |

---

## Current Scale

- 56 TB storage
- 23 Plex users
- 30+ Docker containers
- 2.5 GbE LAN
- Static IP addressing
- Daily backups
- UPS monitoring
- Remote VPN access

---

## Docker

The Docker environment currently hosts more than 30 containers supporting:

- Media streaming
- Media automation
- Monitoring
- Networking
- Databases
- Password management
- Personal cloud
- Reverse proxy

Major containers include:

- Plex
- Audiobookshelf
- Sonarr
- Radarr
- Bazarr
- SABnzbd
- qBittorrent
- Nextcloud
- Vaultwarden
- Immich
- Grafana
- PostgreSQL
- MariaDB
- Matrix Synapse

---

## Troubleshooting Example

### Plex Database Corruption

#### Problem

Plex database became corrupted after an unexpected shutdown.

#### Diagnosis

Reviewed Docker logs and identified SQLite database corruption.

#### Resolution

1. Stopped Plex container.
2. Restored previous database backup.
3. Restarted container.
4. Verified library integrity.

#### Lessons Learned

- Increased backup frequency.
- Always verify database integrity after an unexpected shutdown.
