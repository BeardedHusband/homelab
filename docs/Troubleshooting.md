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
- Verify folder structure matches what Komga expects per series (one folder per series, volumes/chapters named consistently) — Komf's matching accuracy depends heavily on clean, consistent naming, same underlying lesson as the ABS folder work in item #7.
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
