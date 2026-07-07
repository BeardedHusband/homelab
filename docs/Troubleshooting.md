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
- A Chromebook (CB315-4H, MAGMA board) was also considered as a host — see entry #15 for the full conversion process, which was ultimately carried out rather than staying on stock ChromeOS.

**Prevention / future work:**
- Raspberry Pi 4 is the planned long-term home for these services (lower power draw, dedicated single-purpose box). Migration steps above should transfer directly.

---

## 2. WiFi dead after kernel auto-upgrade — ASUS ROG Zephyrus G15 (MT7921 / mt76 driver)

**Symptom:**
WiFi connects briefly after boot then drops entirely. `nmcli dev wifi list` returns empty (no networks visible at all). Interface shows as `DORMANT` in `ip link`. Rebooting restores WiFi temporarily but it dies again within minutes to an hour. Password-related errors are a red herring — the adapter isn't scanning at all.

**Diagnosis steps:**

1. Confirm adapter is visible but dormant:
```bash
ip link show
```
Look for `wlp4s0` (or similar) with state `DORMANT` — means adapter is up but not functional.

2. Confirm scan returns nothing:
```bash
nmcli dev wifi list
sudo nmcli dev wifi rescan && nmcli dev wifi list
```
If both return only headers with no networks listed, it's a driver issue, not a password/SSID issue.

3. Check which kernel is running:
```bash
uname -r
```
If 6.14+ is shown, this is the root cause.

4. Check which MT76 modules are loaded:
```bash
lsmod | grep mt79
```
Expected output on affected kernels: `mt792x_lib`, `mt76_connac_lib`, `mt76`, `mac80211` — all loaded but driver in bad state.

5. Confirm module name changed in 6.14+ (mt7921e no longer exists):
```bash
find /lib/modules/$(uname -r) -name "mt79*"
```

6. Attempt driver reload (temporary fix, will break again):
```bash
sudo modprobe -r mt76 mt76_connac_lib mt792x_lib
sudo modprobe mt792x_lib mt76_connac_lib mt76
sudo systemctl restart NetworkManager
nmcli dev wifi list
```
Note: module unload will fail with "in use" errors because mac80211 holds the stack — a full reboot is cleaner than fighting the unload order.

7. Check what kernel versions are installed:
```bash
apt list --installed | grep linux-image
```

**Root cause:**
Linux Mint's HWE (Hardware Enablement) stack automatically upgraded the kernel from a stable version to 6.14 (and then 6.17) during a routine update. Kernel 6.14+ introduced a regression in the `mt76` driver stack (specifically the MT7921 chipset used in the G15's MediaTek WiFi card) that causes the driver to enter a bad state intermittently. The driver initializes on boot, works briefly, then dies. The module name also changed in 6.14+ — `mt7921e` no longer exists, replaced by `mt792x_lib`. The issue is not hardware, not password, not router-side — confirmed by the empty scan list and DORMANT interface state.

**Fix:**

Install kernel 6.8 (last known stable for MT7921):
```bash
sudo apt install linux-image-6.8.0-107-generic linux-headers-6.8.0-107-generic
sudo update-grub
```

Reboot and select **Advanced options for Linux Mint → 6.8.0-107-generic** in GRUB.

**Important:** First boot on 6.8 with NVIDIA drivers present may stall at a blinking cursor for 2–3 minutes. This is normal — wait it out. Do not hard-reboot during this window; it will complete.

Once confirmed stable on 6.8, remove the broken kernels (must be booted into 6.8 first — removing the currently running kernel will prompt an abort warning):
```bash
sudo apt remove --purge linux-image-6.14.0-29-generic linux-image-6.14.0-37-generic linux-image-6.17.0-20-generic
sudo apt remove --purge linux-headers-6.14.0-29-generic linux-headers-6.14.0-37-generic linux-headers-6.17.0-20-generic
sudo apt autoremove
sudo update-grub
```

Set 6.8 as default boot kernel in `/etc/default/grub`:
```
GRUB_DEFAULT="Advanced options for Linux Mint>Linux Mint, with Linux 6.8.0-107-generic"
```
Then:
```bash
sudo update-grub
```

**Prevention:**
- Update Manager will continue offering kernel upgrades via the HWE stack. Do not install kernel updates without first checking whether the target version has MT7921 regressions reported on Ubuntu/Arch forums or the linux-wireless mailing list.
- The Update Manager will show kernel updates as a lightning-bolt icon item. Uncheck or right-click → ignore any kernel update to 6.14+ until the MT76 regression is confirmed fixed upstream.
- Before removing an old kernel, always verify the running kernel with `uname -r` — removing the currently booted kernel mid-session triggers an abort prompt and is safe to cancel.
- If WiFi dies again on a future kernel, the fallback is: plug in ethernet → install 6.8 → reboot into it via GRUB Advanced options.

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

---

## 4. Audiobookshelf (ABS) library organization at scale

**Context:**
800+ audiobooks accumulated over time with inconsistent folder naming/structure, causing ABS to mis-match metadata, split series incorrectly, or fail to scan books at all.

**Symptom:**
Books show up as "missing metadata," duplicate entries appear for the same book, series get split across multiple entries, or ABS's scanner skips folders entirely.

**Root cause:**
ABS expects a fairly consistent folder structure to reliably match metadata (typically `Author/Series/Book Title (Year)/` or `Author/Book Title (Year)/`, one book per folder, audio files inside). Years of ad-hoc downloading/ripping had produced inconsistent naming: missing years, inconsistent author name formatting (e.g. "Last, First" vs "First Last"), series info embedded inconsistently or missing, and some multi-file books split across nested subfolders ABS didn't expect.

**Fix (approach used):**
- Wrote iterative PowerShell scripts to walk the existing library tree and normalize folder/file naming in batches rather than by hand:
  - Standardize author folder naming to a single convention.
  - Detect and tag series info into folder names where identifiable from existing metadata/filenames.
  - Flatten unexpected nested subfolder structures so each book lives in one folder.
  - Run scripts against a *copy* or small test batch first, verify ABS picks it up correctly, then run against the full library.
- Re-scanned the library in ABS after each batch and spot-checked problem entries before moving to the next batch, rather than running one giant rename pass across everything at once.

**Prevention:**
- Adopt the target folder convention (`Author/Series/Book Title (Year)/`) at ingest time going forward — normalize new downloads before they land in the ABS library folder rather than letting inconsistencies accumulate again.
- Keep the PowerShell scripts around and versioned (this repo) so future cleanup passes don't start from scratch.

---

## 5. Ebook/audiobook delivery pipeline: Kavita → Calibre-Web migration + Kindle delivery

**Context:**
Originally used Kavita for ebook serving; migrated to Calibre-Web for better OPDS support and metadata handling, with delivery to a jailbroken Kindle Paperwhite (10th gen) running KOReader.

**Symptom / motivation for migration:**
Kavita's OPDS implementation and metadata handling weren't a good fit for the way books needed to sync to KOReader on the jailbroken Kindle; Calibre-Web's tighter Calibre-library integration and mature OPDS feed support were a better match.

**Migration steps:**
1. Stand up Calibre-Web pointed at a proper Calibre library (not just a raw folder of ebooks) — this is what enables Calibre-Web's metadata and OPDS features.
2. Import/convert the existing ebook collection into the Calibre library structure.
3. Expose Calibre-Web externally via Cloudflare Tunnel at `books.stephens-group.net` (rather than direct port forwarding) so it's reachable from the Kindle without opening ports on the router.
4. On the Kindle: confirm jailbreak + KOReader are functioning, then add the Calibre-Web OPDS feed URL (`books.stephens-group.net/opds` or equivalent) as a catalog source inside KOReader.
5. Test browsing/downloading a book directly on-device via the OPDS catalog before decommissioning Kavita.
6. Decommission/stop the Kavita container once Calibre-Web + OPDS + KOReader path is confirmed working end-to-end.

**Gotchas encountered:**
- OPDS feeds require the underlying library to have clean, consistent metadata (author/series/title) to browse/search well — worth cleaning up metadata in Calibre itself before relying heavily on the OPDS feed.
- Cloudflare Tunnel needed to be configured for this service specifically (see item below on Cloudflare Access) — don't expose it without auth in front of it.

**Prevention:**
- Any new ebook additions should go into the Calibre library (not a loose folder) so metadata and OPDS stay consistent.

---

## 6. Manga metadata enrichment: Komga + Komf

**Context:**
Manga collection (Berserk, Vinland Saga, Blame!, Pluto, Monster, Akira, Blade of the Immortal, Hellboy, Sin City, Walking Dead, Attack on Titan, and others) served via Komga, with Komf used to automate metadata/cover enrichment.

**Symptom / motivation:**
Manually tagging series metadata (covers, summaries, reading order) for a large and growing manga library is slow and error-prone.

**Fix (setup used):**
- Deploy Komga as the manga server (library scanning, reading, series organization).
- Deploy Komf alongside it configured against Komga's API to automatically pull and apply metadata/cover art from configured metadata providers.
- Verify folder structure matches what Komga expects per series (one folder per series, volumes/chapters named consistently) — Komf's matching accuracy depends heavily on clean, consistent naming, same underlying lesson as the ABS folder work in item #4.
- Spot-check a handful of series after each Komf run before trusting it across the whole library, since mismatches (wrong series matched) are easier to catch early than after they've propagated.

**Prevention:**
- Keep new manga acquisitions named consistently at import time (series name + volume number) so Komf's auto-match stays reliable without manual correction.

---

## 7. Cloudflare Access for previously-unauthenticated exposed services

**Context:**
Several self-hosted services were reachable via Cloudflare Tunnel (public hostnames on `stephens-group.net`) but had no authentication layer in front of them — reachable by anyone who found/guessed the URL.

**Symptom / risk:**
Services with sensitive data or admin controls (media servers, dashboards, management UIs) were exposed to the public internet with only the app's own (sometimes weak or nonexistent) login, rather than a proper access-control layer.

**Fix:**
- Enabled Cloudflare Access (Zero Trust) in front of the exposed hostnames rather than relying on the app's own auth.
- Created Access policies per-hostname (or per-group of hostnames) requiring authentication (e.g., email OTP or identity provider login) before Cloudflare will even proxy the request through to the tunnel/origin.
- Verified that services meant to stay fully public (e.g., an OPDS feed a Kindle needs to reach without an interactive login step) were either excluded from the Access policy or handled with a service-token/bypass approach appropriate to non-interactive clients, so device-based feeds didn't break.
- Tested from an unauthenticated browser/session to confirm Access actually blocks the request before reaching the origin, not just hides a login page.

**Prevention:**
- Treat "exposed via Cloudflare Tunnel" and "protected" as two separate steps going forward — any new hostname added to the tunnel gets an explicit decision about whether it needs an Access policy before it's considered done, not exposed by default.

---

## 8. GBA ROM organizer script

**Context:**
A folder of GBA ROM files with inconsistent naming (region tags, revision numbers, abbreviations, mixed casing) needed to be organized into a clean, consistently-named, browsable structure.

**Symptom:**
Frontend/emulator library scanners either failed to match box art and metadata, or listed the same game multiple times under slightly different filenames (different region tags, rev numbers, etc.).

**Fix (approach used):**
- Wrote a script to walk the ROM folder and normalize filenames to a single consistent convention (consistent title casing, standardized region/revision tag placement, stripped redundant tags).
- Grouped/deduplicated near-identical entries (same game, different region dump) where only one copy was actually wanted, flagging ambiguous cases for manual review rather than silently deleting.
- Ran against a test copy first, spot-checked results in the target frontend/emulator before running against the full ROM set.

**Prevention:**
- Apply the same naming convention to any newly added ROMs at import time rather than letting inconsistent naming creep back in.
- Keep the script versioned here so the same normalization can be re-run if new inconsistencies show up.

---

## 9. Plex hardware transcoding + Radarr/Sonarr custom format scoring (legacy codec blocking)

**Context:**
Plex server handling a large library (~4,680 movies, 433 shows) needed reliable hardware transcoding, and Radarr/Sonarr needed to stop pulling releases encoded with legacy/inefficient codecs that caused transcoding problems or wasted storage.

**Symptom:**
Playback for certain files required CPU-based transcoding (high load, potential stuttering for remote/friend access) instead of hardware-accelerated transcoding; some downloaded releases used older codecs (e.g., legacy H.264 profiles or worse) that were larger and more transcode-hungry than necessary.

**Fix:**
- **Plex hardware transcoding:** Enabled and verified Intel QuickSync hardware transcoding in Plex settings, confirmed the UnRAID Docker container had proper GPU/iGPU passthrough configured (device mappings for `/dev/dri` or equivalent) so Plex could actually access QuickSync rather than falling back to software transcoding silently.
- **Radarr/Sonarr custom format scoring:** Set up custom formats in Radarr and Sonarr to detect and penalize/block legacy codecs and undesirable encodes, scoring releases so preferred (modern, efficient codec) releases are picked over legacy ones automatically during the release selection process, rather than needing manual review per download.

**Verification:**
- Confirmed hardware transcoding is actually being used (not falling back to software) by checking Plex's active transcoder session details during playback of a file that requires transcoding.
- Spot-checked new grabs in Radarr/Sonarr against the custom format scores to confirm legacy-codec releases were being deprioritized/rejected as expected.

**Prevention:**
- Periodically re-check custom format scores against new release-group naming conventions, since format-detection rules can drift out of date as scene/P2P release naming conventions change over time.
- Confirm GPU passthrough survives UnRAID updates/container recreation — this is a common thing to silently break after an update.

---

## 10. Jitsi self-hosted — mobile clients disconnecting immediately on join

> ⚠️ **UNRESOLVED** — Root cause suspected but not confirmed. Fix section incomplete. Return to this when ready to troubleshoot TURN configuration.

**Symptom:**
Mobile clients (iOS/Android, via browser at `meet.stephens-group.net`) receive "You have been disconnected — check your network connection" immediately or within seconds of joining a Jitsi room. Desktop clients on LAN connect without issue.

**Diagnosis steps:**
1. Confirm TURN is generating relay candidates — go to `https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/`, add your TURN server URI (`turn:yourdomain.com:3478`), enter credentials, click "Gather candidates." You need to see `relay` type candidates appear. If only `host` and `srflx` appear, TURN is unreachable or misconfigured.
2. Verify firewall and router have UDP 3478 and relay port range 49152–65535 open and forwarded.
3. Check `turnserver.conf` for correct `external-ip` value pointing to your public IP.
4. Check Jitsi `config.js` for `p2p: { enabled: true }` — mobile clients attempt P2P first, fail, and may not fall back to TURN correctly:
```javascript
p2p: {
    enabled: false
}
```
5. Verify Synapse `homeserver.yaml` TURN block is correct:
```yaml
turn_uris:
  - "turn:yourdomain.com:3478?transport=udp"
  - "turn:yourdomain.com:3478?transport=tcp"
turn_shared_secret: "your_secret"
turn_user_lifetime: 86400000
turn_allow_guests: false
```

**Root cause:**
*(Unconfirmed)* Mobile devices behind NAT/cellular cannot establish direct peer-to-peer WebRTC connections and are fully dependent on TURN relay. TURN server is either unreachable, misconfigured, or not correctly wired into Jitsi config. Desktop LAN clients mask the problem because they can connect directly.

**Fix:**
*(Incomplete — return to finish)*

**Prevention:**
- Always verify TURN with the Trickle ICE tool after initial setup and after any firewall or router changes — TURN failures are silent on LAN but immediately fatal for mobile.
- Disable P2P in `config.js` on self-hosted instances.
- Mobile clients should use the Jitsi Meet app pointed at `meet.stephens-group.net` rather than the browser — set server in app settings once, never touch again.

---

## 11. Wake-on-LAN not working remotely (WireGuard + iPhone)

**Symptom:**
WOL works from Xbox on LAN but fails to wake gaming PC when triggered remotely over WireGuard from iPhone. No response, PC stays off.

**Diagnosis steps:**
1. Confirm WOL works locally first — if Xbox on LAN can wake PC, hardware/BIOS config is correct.
2. Confirm WireGuard tunnel is actually up on iPhone — verify you can reach other LAN resources (UnRAID web UI, PiHole, etc.) before blaming WOL.
3. Understand the architectural reason: WOL magic packets are Layer 2 LAN broadcasts. They cannot be sent directly from a remote WireGuard client across the tunnel — a device physically on the LAN must broadcast the packet.

**Root cause:**
WOL magic packets are subnet broadcasts and cannot originate from outside the LAN, even over a routed VPN tunnel. The iPhone on WireGuard is logically "outside" the LAN from a broadcast perspective. The Xbox works because it is physically on the same Layer 2 segment.

**Fix:**

**Option A — etherwake via UnRAID SSH (immediate, no setup):**
SSH into UnRAID via Termius while on WireGuard, then:
```bash
etherwake AA:BB:CC:DD:EE:FF
```
`etherwake` is built into UnRAID — no install needed. Save as a Termius snippet for one-tap access.

**Option B — WOL Docker container (web UI, no SSH needed):**
Install `ghcr.io/trugamr/wol:latest` on UnRAID via Community Apps.

Create config file *before* starting the container (container will fail with "is a directory" error if this file doesn't exist):
```bash
rm -rf /mnt/user/appdata/wol/config.yaml
nano /mnt/user/appdata/wol/config.yaml
```

Paste config:
```yaml
machines:
  - name: gaming desktop
    mac: "AA:BB:CC:DD:EE:FF"
    ip: "192.168.1.x"

server:
  listen: ":7777"

ping:
  privileged: false
```

In UnRAID Docker template, verify:
- Volume mapping host path: `/mnt/user/appdata/wol/config.yaml`
- Container path: `/etc/wol/config.yaml`
- Network type: Host
- Remove any auto-added TailScale Fallback State Directory entry pointing at `/etc/wol/config.yaml` — this conflicts

Access web UI at `http://192.168.1.66:7777` over WireGuard to wake PC without SSH.

**Verification:**
Shut gaming PC down → connect WireGuard on iPhone (confirm tunnel is up by reaching UnRAID) → send WOL via etherwake or web UI → wait 45–60 seconds → connect Moonlight/Apollo.

**Prevention:**
- Always-on LAN device (UnRAID) is the correct place to host WOL functionality, not the remote client
- The WOL container config file must exist on the host before container start — UnRAID will create a directory instead of a file if the path doesn't exist, causing startup failure
- Remove auto-added Community Apps TailScale fields from the container template if not using Tailscale — they can conflict with volume mappings

---

## 12. WOL Docker container fails to start — "is a directory" error

**Symptom:**
WOL container repeatedly crashes on start. Logs show:
```
Error: failed to load config file: read /etc/wol/config.yaml: is a directory
```

**Diagnosis steps:**
1. Check container logs:
```bash
docker logs WOL
```
2. Confirm the issue — check what `/mnt/user/appdata/wol/config.yaml` actually is:
```bash
ls -la /mnt/user/appdata/wol/
```
If `config.yaml` shows as a directory (`drwxr-xr-x`) rather than a file (`-rw-r--r--`), this is the cause.

**Root cause:**
When the container volume mapping path doesn't exist at container creation time, Docker creates the target as a directory rather than a file. The WOL container then tries to read a directory as a config file and fails.

**Fix:**
```bash
rm -rf /mnt/user/appdata/wol/config.yaml
nano /mnt/user/appdata/wol/config.yaml
```

Paste valid config, save (Ctrl+X → Y → Enter), then restart container. File must exist and be a valid YAML file before container starts.

**Prevention:**
- Create the config file manually before installing/starting any container that requires a config file at a specific path
- Verify with `ls -la` that the path is a file not a directory before starting the container
- The Community Apps template warning at the top of the WOL template states this explicitly — read it before applying

---

## 13. NoMachine Inspiron going to sleep / unreachable without physical interaction

**Symptom:**
Inspiron drops off the network and becomes unreachable via NoMachine. Requires physically opening the lid and pressing power button to wake. Not fully off — just suspended/sleeping.

**Diagnosis steps:**
1. Confirm it's sleep not shutdown — if it responds immediately after physical interaction it's suspend, not a crash or power failure.
2. Check current logind config:
```bash
cat /etc/systemd/logind.conf | grep -i handle
```
3. Check power manager settings in Mint GUI — System Settings → Power Manager.

**Root cause:**
Linux Mint default power management suspends the system on lid close and after idle timeout. On a headless/always-on machine this is never appropriate.

**Fix:**
```bash
sudo nano /etc/systemd/logind.conf
```

Set (uncomment if prefixed with `#`):
```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleSuspendKey=ignore
IdleAction=ignore
```

Then apply:
```bash
sudo systemctl restart systemd-logind
```

Also set all sleep/suspend/hibernate options to "Never" in Mint GUI Power Manager for both battery and plugged-in profiles. Both changes are needed — Mint's power manager can override logind settings.

**Verification:**
```bash
sudo reboot
```
Attempt NoMachine connection from another device without physically touching the Inspiron. If reachable within ~60 seconds of boot, fix is confirmed.

**Prevention:**
- Apply these settings immediately after OS install on any machine intended as a headless always-on appliance
- Set all services to autostart via systemd at the same time:
```bash
sudo systemctl enable ssh
sudo systemctl enable pihole-FTL
sudo systemctl enable unbound
sudo systemctl enable wg-quick@wg0
```
- Verify autostart with:
```bash
systemctl is-enabled pihole-FTL
systemctl is-enabled unbound
systemctl is-enabled wg-quick@wg0
```

---

## 14. qBittorrent causing high CPU load / elevated UnRAID load average

**Symptom:**
UnRAID dashboard shows sustained high CPU load (load average ~18 on a 20-thread i7-12700K). Individual CPU threads hitting 80-89% in the UnRAID dashboard graph. Server feels sluggish. No obvious cause from the dashboard alone.

**Diagnosis steps:**
1. Check UnRAID dashboard CPU graph for which specific threads are pegged — look for sustained 80%+ on individual cores.
2. Run `htop` in UnRAID terminal, press `P` to sort by CPU:
```bash
htop
```
3. If `htop` is interrupted (SIGINT), fall back to docker stats snapshot:
```bash
docker stats --no-stream
```
4. Identify the offending container by CPU% column. In this case `binhex-qbittorrentvpn` was consuming ~43.7% CPU sustained.
5. Cross-reference outbound traffic on the UnRAID dashboard — high outbound (108 Mbps in this case) alongside high CPU confirms active torrent swarm activity, not a runaway process.

**Root cause:**
qBittorrent was running with uncapped connection limits managing 1721 simultaneously active torrents across a library of 1765 total. Default/uncapped settings were:
- Global max connections: 1000
- Max connections per torrent: 100
- Global max upload slots: 500
- Max upload slots per torrent: 20
- Max active torrents: 200

With 1721 active torrents competing for those slots, qBittorrent was maintaining thousands of simultaneous connection states, saturating CPU. Additionally, on every restart qBittorrent rechecks/hashes all torrents for integrity — with 1765 torrents this produced a sustained 43%+ CPU spike lasting the duration of the recheck.

**Fix:**
Restarted the `binhex-qbittorrentvpn` container to immediately drop load while diagnosing. Then applied the following limits in qBittorrent WebUI under Tools → Options:

**Connection tab:**
- Global max connections: 300–400
- Max connections per torrent: 20–30
- Global max upload slots: 100–150
- Max upload slots per torrent: 4–6

**BitTorrent tab:**
- Max active downloads: 10
- Max active uploads: 30–40
- Max active torrents: 150
- Inactive seeding time limit: 60–120 minutes (down from 1440)

**Note on private trackers:** These limits are safe for H&R compliance — ratio requirements are satisfied by upload volume and seeding time, not connection count. 4–6 upload slots per torrent is sufficient to satisfy H&R requirements on any private tracker.

**Prevention:**
- Cap connection limits immediately after any fresh qBittorrent install/container deployment — defaults are tuned for a small library, not 1700+ torrents
- The startup recheck spike is unavoidable with a large library — avoid unnecessary container restarts
- To identify dead-weight public tracker torrents burning active slots: in qBittorrent WebUI enable the **Private** column (right-click any column header) and sort by it — non-private torrents with dead trackers (rarbg, etc.) can be safely paused or removed with no ratio risk
- Alternatively use the left sidebar tracker groups to bulk-select and pause torrents by tracker domain

---

## 15. Chromebook → Linux Mint via MrChromebox UEFI Full ROM (Acer CB315-4H, MAGMA board)

**Symptom / motivation:**
Acer Chromebook 315 (CB315-4H, MAGMA board, Pentium Silver N6000, 128GB eMMC) repurposed from ChromeOS to Linux Mint 22 for use as a homelab dashboard / Moonlight streaming client.

**Diagnosis steps:**

1. Confirm board identity and UEFI Full ROM support via `chrome://system` in ChromeOS browser. Key fields:
   - `CHROMEOS_FIRMWARE_VERSION`: `Google_Magolor.13606.710.0`
   - `CHROMEOS_RELEASE_BOARD`: `dedede-signed-mp-v58keys`
   - `HWID`: `MAGMA-QZPR C3B-F2F-E4E-L4R-Q9Y`
   - `FREE_DISK_SPACE`: `93265756160` (~128GB eMMC confirmed)
   - `bios_info → RO firmware`: `protected` (WP enabled, expected)
   - `bios_info → Developer mode`: `not enabled` (expected)

2. Cross-reference board against MrChromebox supported devices page — MAGMA (Dedede family, Jasper Lake) shows two green checkmarks: RW_LEGACY and UEFI Full ROM both supported. WP method listed as `CR50 (SuzyQ) | jumper`.

3. Confirm CPU architecture is x86_64 (not ARM) — required for MrChromebox UEFI Full ROM:
   From Crostini terminal:
   ```bash
   lscpu
   ```
   Confirms: `Architecture: x86_64`, `Model name: Intel(R) Pentium(R) Silver N6000 @ 1.10GHz`, `Virtualization: VT-x`

**Root cause / context:**
ChromeOS uses proprietary Google firmware instead of standard UEFI/BIOS. Installing Linux requires replacing this with a UEFI Full ROM via MrChromebox's firmware utility script. Hardware write protect must be disabled before flashing. For the MAGMA board, WP can be disabled via software (CR50) without opening the device — the physical jumper method is an alternative but was not needed here.

**Fix:**

**Step 1 — Enable Developer Mode:**
- Power off → hold `Esc + Refresh + Power` → at recovery screen press `Ctrl + D` → confirm → wait for wipe/reboot (~10 min)
- After reboot: press `Ctrl + D` at the "OS verification is OFF" screen to boot into ChromeOS

**Step 2 — Get a root shell:**
- At ChromeOS login screen, press `Ctrl + Alt + F2`
- Login as `chronos` (no password)
- This bypasses the `no new privileges` restriction encountered when using Crosh (`Ctrl + Alt + T` → `shell` → `sudo`) in some configurations

**Step 3 — Run MrChromebox firmware utility script:**
```bash
cd; curl -LOf https://mrchromebox.tech/firmware-util.sh && sudo bash firmware-util.sh
```

- Script detects software WP enabled and prompts to disable it — press `Y` then Enter
- Device reboots, re-run the same command after reboot
- Select **Install/Update UEFI (Full ROM) Firmware** from the menu
- When prompted to create firmware backup: press `Y`, select USB device (PNY USB 2.0 FD, 28.9GB listed as device `1`)
- Backup saved as `stock-firmware-MAGMA-20260415.rom` — copied off USB to NAS before proceeding
- Script downloads `coreboot_edk2-magolor-mrchromebox_20260409.rom`, persists HWID, extracts VPD, disables software WP, flashes Full ROM
- Confirms: `Full ROM firmware successfully installed/updated.`
- Power off using the script's menu option — do not reboot immediately

**Step 4 — Install Linux Mint:**

Boot from Linux Mint USB. Installer fails twice before successful install:

*Error 1 — GRUB install fatal error:*
Installer defaulted to wrong target device for bootloader. Fix: use manual partitioning ("Something else").

*Error 2 — `ubi-partman` crashed with exit code 10:*
Partition table on eMMC was in a bad state from the failed install attempt. Fix — wipe the disk from terminal in the live environment:
```bash
sudo umount /dev/mmcblk1* 2>/dev/null
sudo swapoff -a
sudo wipefs -a /dev/mmcblk1
sudo sgdisk --zap-all /dev/mmcblk1
```

*Successful install — manual partitioning ("Something else"):*

| Partition | Device | Type | Mount | Size |
|---|---|---|---|---|
| EFI | `/dev/mmcblk1p1` | EFI System Partition | — | 512 MB |
| Root | `/dev/mmcblk1p2` | ext4 | `/` | remainder (~124 GB) |

Set **Device for boot loader installation** to `/dev/mmcblk1` (not a partition — the whole disk).

Proceed through installer normally. On completion, remove USB when prompted and reboot.

**Verification:**
System boots directly into Linux Mint. No GRUB issues. WiFi connects. Updates applied via Update Manager.

**Prevention:**
- Always use **Something else** (manual partitioning) when installing Linux on Chromebook eMMC — the automatic partitioner frequently targets the wrong bootloader device or produces exit code 10 errors on eMMC
- Run `wipefs` and `sgdisk --zap-all` on the eMMC before any install attempt, especially after a failed install
- Set bootloader device explicitly to the whole disk (`/dev/mmcblk1`) not a partition
- Firmware backup (`stock-firmware-MAGMA-YYYYMMDD.rom`) should be saved to NAS or secondary storage immediately — required if ever reverting to ChromeOS or recovering from a bad flash
- First boot after MrChromebox UEFI flash may take 30+ seconds with a black screen — this is normal, do not hard-reboot
- The `chronos` shell method (`Ctrl + Alt + F2` at login screen) is more reliable than Crosh for running the firmware script — avoids the `no new privileges` flag issue encountered in Crosh shell with some account configurations
- After install, disable suspend-on-lid-close if using as a clamshell/kiosk device (see entry #13)

---

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
