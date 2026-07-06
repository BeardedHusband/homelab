# Homelab Troubleshooting Log

A running reference of issues encountered in this homelab setup, how they were diagnosed, and how they were resolved. Written for future-me under time pressure — skip to the relevant section, don't re-derive from scratch.

**Environment context:**
- Server1: UnRAID, i7-12700K, 32GB DDR4 @ 3600MHz, ~56TB array, 40–50+ Docker containers, `192.168.1.66`
- Secondary host: Dell Inspiron 15 (svc tag 8JN5DL2), i3-7130U, 16GB RAM, Linux Mint 22.2, `192.168.1.20`, managed via NoMachine
- VPN provider: PIA (Private Internet Access) via WireGuard
- Torrent client: qBittorrent in `binhex-qbittorrentvpn` Docker container
- Private tracker: MyAnonamouse (MAM), session management via Mousehole (dual-IP)

---

## 1. Pi-hole / Unbound / WireGuard migration (MacBook Pro → Dell Inspiron)

**Context:**
Migrating these services off an aging mid-2010 MacBook Pro onto a 2018 Dell Inspiron 15 (interim solution; long-term plan is a Raspberry Pi 4).

**Symptom / motivation:**
Old hardware unreliable/underpowered for running Pi-hole + Unbound + WireGuard reliably as always-on infrastructure.

**Migration checklist:**
1. Install base OS (Linux Mint 22.2) on the Inspiron, set static IP / DHCP reservation (`192.168.1.20`), confirm hardwired (not WiFi) for reliability.
2. Set up remote access first (NoMachine) so the box can be headless-administered going forward.
3. Install Pi-hole, point it at Unbound as the upstream recursive resolver (not a third-party DNS) for privacy.
4. Install and configure Unbound as a local recursive resolver — verify root hints and DNSSEC validation are working (`dig` against Unbound directly to confirm before wiring Pi-hole to it).
5. Install WireGuard server config, port over existing peer configs (server keys will differ — every client peer needs updated endpoint / server public key).
6. Update the router/DHCP server to point DNS at the new Pi-hole IP (`192.168.1.20`) instead of the old MacBook IP.
7. Update port forwarding rules on the router for WireGuard to point at the new box.
8. Decommission old MacBook only after confirming DNS + WireGuard both work from multiple client devices.

**Gotchas encountered:**
- Client WireGuard configs (phone, laptop, etc.) needed updated `Endpoint` and the new host's public key — old configs silently fail to connect rather than erroring clearly.
- A Chromebook (CB315-4H, MAGMA board) was considered as an alternative host but set aside — ChromeOS/crostini limitations made it a worse fit than the Inspiron; returned to stock ChromeOS.

**Prevention / future work:**
- Raspberry Pi 4 is the planned long-term home for these services (lower power draw, dedicated single-purpose box). Migration steps above should transfer directly.

---

## 2. Linux WiFi drops / regression on ASUS ROG Zephyrus G15 (MT7921 chipset)

**Symptom:**
WiFi (MediaTek MT7921 chipset) becomes unstable, drops, or fails to reconnect reliably after upgrading to kernel 6.14+.

**Diagnosis steps:**
1. Check kernel version: `uname -r`
2. Check for MT7921-related errors in kernel log: `dmesg | grep -i mt7921` or `journalctl -k | grep -i mt7921`
3. Check driver/firmware version in use vs. what's available: `dmesg | grep -i firmware`

**Root cause:**
Known regression in the `mt7921e` driver introduced in kernel 6.14+ affecting stability on this chipset — not a hardware fault or misconfiguration.

**Fix (workarounds, pick based on distro):**
- Pin/boot an older kernel known to work (pre-6.14) until upstream fixes land.
- Check for and apply any backported firmware/driver fixes from your distro's kernel package updates.
- Monitor the linux-wireless mailing list / kernel bugzilla for the specific regression fix landing, then update.

**Prevention:**
- Before doing a major kernel upgrade on this laptop specifically, check recent kernel changelogs / Arch or Ubuntu forums for MT7921 regression reports first.

---

## 3. Mousehole + qBittorrent VPN IP Mismatch (MAM)

**Problem:**

MyAnonamouse (MAM) flagged my account for connecting from two different IPs — my home IP from browsing and a PIA VPN IP from qBittorrent seeding. MAM requires a registered "seedbox" session to allow this, kept updated via their Dynamic Seedbox API. Mousehole automates this, but must run on qBittorrent's network namespace to detect the correct VPN IP. PIA also periodically rotates VPN IPs, breaking the MAM session.

**Diagnosis:**

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

**Resolution:**

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

**Lessons Learned:**

- Always create MAM sessions using ASN locking, not IP locking, to survive VPN IP rotations
- Mousehole must run on the VPN container's network namespace, not the Docker bridge
- Store the rebuild script in appdata so it survives and is easy to find
- After any qBittorrent container update, verify Mousehole is still seeing the PIA IP
- Check `/tmp/getvpnport` exists before assuming qBittorrent is fully started; its absence means port forwarding failed

## Template for new entries

```
## [Issue title]

**Symptom:**

**Diagnosis steps:**
1.

**Root cause:**

**Fix:**

**Prevention:**
```

