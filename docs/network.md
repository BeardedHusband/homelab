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

## Troubleshooting

### Mousehole + qBittorrent VPN IP Mismatch (MAM)

#### Problem

MyAnonamouse (MAM) flagged my account for connecting from two different IPs — my home IP from browsing and a PIA VPN IP from qBittorrent seeding. MAM requires a registered "seedbox" session to allow this, kept updated via their Dynamic Seedbox API. Mousehole automates this, but must run on qBittorrent's network namespace to detect the correct VPN IP. PIA also periodically rotates VPN IPs, breaking the MAM session.

#### Diagnosis

Mousehole was running on Docker's default bridge network, causing it to report the home IP instead of the PIA VPN IP. Confirmed by comparing:

```bash
docker exec Mousehole curl -s ifconfig.me
docker exec binhex-qbittorrentvpn curl -s ifconfig.me
```

When IPs differ, Mousehole is on the wrong network. MAM logs showed `Invalid session - IP mismatch` or `ASN mismatch` errors.

Additional issues encountered:
- PIA dropped `ca-ontario.privacy.network` from port forwarding support, preventing qBittorrent from launching
- Mousehole container updates reset environment variables, breaking the auth config
- MAM session created from browser locked to home ASN instead of PIA ASN

#### Resolution

Mousehole must share qBittorrent's network namespace using `--network=container:binhex-qbittorrentvpn`. This cannot be set via UnRAID's Docker GUI and requires a terminal script.

Rebuild script saved at `/mnt/user/appdata/mousehole/rebuild-mousehole.sh`:

```bash
#!/bin/bash
docker stop Mousehole 2>/dev/null
docker rm Mousehole 2>/dev/null

docker run -d \
  --name Mousehole \
  --network=container:binhex-qbittorrentvpn \
  --restart unless-stopped \
  -v /mnt/user/appdata/mousehole/data:/srv/mousehole \
  -e MOUSEHOLE_PORT=5010 \
  -e MOUSEHOLE_STATE_DIR_PATH=/srv/mousehole \
  -e MOUSEHOLE_CHECK_INTERVAL_SECONDS=300 \
  -e MOUSEHOLE_STALE_RESPONSE_SECONDS=86400 \
  -e TZ=America/Chicago \
  -e MOUSEHOLE_USER_AGENT=mousehole-by-timtimtim \
  -e MOUSEHOLE_INSECURE_ALLOW_NO_AUTH=true \
  tmmrtn/mousehole:latest
```

MAM session cookie stored and updated via:

```bash
cat > /mnt/user/appdata/mousehole/data/state.json << 'EOF'
{
  "currentCookie": "YOUR_COOKIE_HERE"
}
EOF
docker restart Mousehole
```

MAM session must be created with the PIA IP entered manually and set to **ASN locked** (not IP locked) to survive IP rotations within PIA's network.

If PIA drops port forwarding support for an endpoint, change `VPN_REMOTE_SERVER` in the container to a supported endpoint such as `ca-toronto.privacy.network`.

#### Lessons Learned

- Always create MAM sessions using ASN locking, not IP locking, to survive VPN IP rotations
- Mousehole must run on the VPN container's network namespace, not the Docker bridge
- Store the rebuild script in appdata so it survives and is easy to find
- After any qBittorrent container update, verify Mousehole is still seeing the PIA IP
- Check `/tmp/getvpnport` exists before assuming qBittorrent is fully started; its absence means port forwarding failed

