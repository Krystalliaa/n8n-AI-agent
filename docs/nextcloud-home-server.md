# Nextcloud Home Server — Boot Recovery, Docker Rebuild, Backup Automation & WiFi Connectivity Fix

## Introduction

I run a personal Nextcloud instance on a Raspberry Pi 4B (4GB RAM), headless, connected via WiFi only — no ethernet is available at the installation location. The OS boots from a microSD card, and all user data and the MariaDB database live on a 1.9TB SSD connected via USB enclosure, mounted at `/mnt/ssd`. The application stack runs as four Docker containers — Nextcloud app, Nextcloud cron, MariaDB, and Redis — managed via Docker Compose. Tailscale provides stable remote access independent of local network conditions.

This document covers a maintenance session that began with a boot failure, progressed through a full rebuild of the boot media and application stack, added automated backup and restore tooling, and concluded with diagnosing and resolving an intermittent WiFi connectivity issue.

---

## Objectives

- Recover from a boot failure caused by a PARTUUID mismatch in `/etc/fstab`
- Evaluate a spare SD card for reuse and decide on a repair-vs-reformat path
- Rebuild the Nextcloud Docker Compose stack on clean boot media
- Implement reliable, tested backup and restore automation
- Diagnose and resolve intermittent loss of SSH and web UI access
- Add persistent network monitoring to capture context for future incidents

---

## Technologies Used

| Technology | Role |
|---|---|
| Raspberry Pi 4B (4GB) | Host hardware |
| Raspberry Pi OS (headless) | Operating system |
| Docker Engine + Compose plugin | Container runtime and orchestration |
| Nextcloud | Self-hosted file sync and sharing application |
| MariaDB | Relational database backend |
| Redis | In-memory cache for Nextcloud session and file locking |
| Tailscale | Stable remote access via WireGuard mesh VPN |
| microSD card | Boot media |
| 1.9TB USB SSD (`/mnt/ssd`) | Persistent data storage |
| WSL2 + usbipd-win | Offline SD card inspection from a Windows machine |
| `blkid`, `e2fsck`, `badblocks` | Storage diagnostics |
| `rsync`, `mariadb-dump` | Backup and restore tooling |
| `iw`, `ss`, `ip`, `ping` | Network diagnostics |

---

## Architecture Overview

```
Raspberry Pi 4B (headless, WiFi only)
│
├── Boot: microSD card (Raspberry Pi OS)
│     └── /etc/fstab — PARTUUIDs verified against blkid output
│
├── Storage: 1.9TB USB SSD, mounted at /mnt/ssd (nofail in fstab)
│     └── /mnt/ssd/nextcloud/
│           ├── .env          (shared secrets: DB credentials, paths)
│           ├── db/           (MariaDB data, bind-mounted)
│           ├── redis/        (Redis data, bind-mounted)
│           ├── html/         (Nextcloud application code + config.php)
│           └── data/         (user files)
│
├── Docker Compose stack
│     ├── db service       → container_name: mariadb
│     ├── redis service    → container_name: redis
│     ├── app service      → container_name: nextcloud  (port 8080)
│     └── cron service     → container_name: nextcloud-cron
│
├── Remote access: Tailscale (wlan0, power-save disabled)
│
└── Automation
      ├── /usr/local/bin/backup_nextcloud.sh   (cron: daily 03:00)
      ├── /usr/local/bin/restore_nextcloud.sh  (manual invocation)
      └── /mnt/ssd/network-monitor/            (continuous connectivity log)
```

All persistent container data uses bind mounts into `/mnt/ssd/nextcloud/` rather than named Docker volumes. Application code (`html/`) and user files (`data/`) are kept in separate directories so Nextcloud updates do not overwrite user data. Secrets are stored in a single `.env` file shared between the Compose stack and both automation scripts to prevent credential drift.

---

## Implementation Process

### 1. Boot Media Recovery

The Pi dropped into emergency mode after I ran `e2fsck` on the primary SD card. The boot log showed `local-fs.target` timing out waiting for a device by PARTUUID, with `boot-firmware.mount` failing as a dependency.

I mounted the card on a separate Windows machine, passed the USB card reader through to WSL2 using `usbipd-win`, ran `blkid` to read the actual current PARTUUIDs, and corrected the stale entries in `/etc/fstab` to match. The card booted normally after this fix.

### 2. Spare SD Card Evaluation

I had a 32GB card from an earlier incident that I suspected might be faulty. I ran `badblocks -sv` (read-only) — zero bad sectors. I then ran `e2fsck -f -n` (report-only, no repair) which aborted with `Directory inode ... block #0, offset 0: directory corrupted`. This confirmed genuine filesystem-level corruption rather than a PARTUUID mismatch.

Because no data on the card needed to be preserved, I reformatted it from scratch using Raspberry Pi Imager rather than attempting a repair, which carries a risk of partial data loss.

### 3. Nextcloud Stack Rebuild on the 32GB Card

With the freshly-imaged card as the new primary boot media, I rebuilt the full stack:

- Ran a full system update, then installed Docker Engine and the Compose plugin via `get-docker.sh`
- Formatted the SSD and added a UUID-based entry to `/etc/fstab` with the `nofail` option, so a disconnected SSD does not block the boot sequence
- Created the Docker Compose stack with four services: `db` (MariaDB), `redis`, `app` (Nextcloud), and `cron` (Nextcloud background jobs)
- Bind-mounted all persistent data under `/mnt/ssd/nextcloud/{db,redis,html,data}`, keeping application code and user files in separate directories
- Installed and authenticated Tailscale, then configured `trusted_domains` and Redis caching in `config.php`

### 4. Backup and Restore Automation

I wrote `backup_nextcloud.sh` and `restore_nextcloud.sh` and installed them under `/usr/local/bin/`. Both scripts source credentials and paths from the same `.env` file used by the Compose stack.

**Backup script behaviour:**
- Puts Nextcloud into maintenance mode
- Dumps the database using `mariadb-dump`
- Rsyncs `data/` (user files) and `html/` (application code and `config.php`)
- Archives everything to `/mnt/ssd/backups/` with a datestamped filename
- Enforces 30-day retention by deleting older archives
- Runs daily at 03:00 via cron

**Restore script behaviour:**
- Accepts a backup archive path as an argument
- Keeps the database container running, waits for readiness via a `mariadb-admin ping` loop
- Imports the SQL dump directly into the running MariaDB container
- Restores `html/` and `data/` via rsync to the correct bind-mount paths
- Stops and restarts only the application and cron services around the restore

### 5. Network Monitoring

After diagnosing the WiFi power-save issue (see Challenges below), I added a persistent network monitoring script that runs in an infinite loop, polling every 5 seconds and appending timestamped snapshots to a daily log file at `/mnt/ssd/network-monitor/network_YYYY-MM-DD.log`.

Each snapshot captures: system uptime, WiFi link state (signal strength, bitrate, association via `iw dev wlan0 link`), reachability to the local router and to `1.1.1.1` (one ping each, to distinguish a local WiFi fault from a wider network issue), SSH daemon port state, Docker container status, the kernel default route table, and the ARP/neighbour table.

```bash
#!/usr/bin/env bash
LOGDIR="/mnt/ssd/network-monitor"
mkdir -p "$LOGDIR"
LOGFILE="$LOGDIR/network_$(date +%F).log"

while true
do
  {
    echo "============================================================"
    date "+%F %T"
    echo

    echo "=== UPTIME ==="
    uptime
    echo

    echo "=== WLAN LINK ==="
    iw dev wlan0 link
    echo

    echo "=== PING ROUTER ==="
    ping -c1 -W1 192.168.3.1
    echo

    echo "=== PING INTERNET ==="
    ping -c1 -W1 1.1.1.1
    echo

    echo "=== SSH PORT ==="
    ss -ltn | grep ":22" || echo "SSH NOT LISTENING"
    echo

    echo "=== DOCKER ==="
    docker ps --format "table {{.Names}}\t{{.Status}}"
    echo

    echo "=== DEFAULT ROUTE ==="
    ip route
    echo

    echo "=== NEIGHBOURS ==="
    ip neigh show
  } >> "$LOGFILE" 2>&1
  sleep 5
done
```

> **Note:** `LOGFILE` is evaluated once at script start. If the process runs past midnight it continues appending to the previous day's file. For short diagnostic runs this is acceptable; for long-term continuous use the script should be wrapped in a systemd service with a daily restart, or the log path should be re-evaluated inside the loop.

---

## Challenges Encountered

### Challenge 1: Boot failure — PARTUUID mismatch after SD card re-flash

The Pi dropped into emergency mode after I ran `e2fsck` on the boot card. The root cause was that Raspberry Pi Imager had previously been used to re-flash the card, which regenerated the PARTUUIDs for each partition and updated `/boot/firmware/cmdline.txt` accordingly — but left `/etc/fstab` inside the root filesystem untouched. My hand-edited `fstab` entries (added to mount the SSD) still referenced the old PARTUUIDs from before the re-flash.

### Challenge 2: Filesystem corruption on the spare SD card

The 32GB spare card showed genuine directory-level filesystem corruption (`e2fsck -f -n` aborted with `directory corrupted`), distinct from the PARTUUID issue on the primary card. Since no data needed to be preserved, I chose reformat over repair.

### Challenge 3: docker-compose.yml validation error

After hand-editing `docker-compose.yml` in nano, `docker compose up -d` failed with `additional properties 'amlservices' not allowed`. A stray keypress had corrupted the top-level `services:` key. I rewrote the file cleanly and validated it with `docker compose config` before bringing the stack up.

### Challenge 4: Backup script path, credential, and logic errors

The initial backup script draft (adapted from a generic template) had several mismatches with my actual environment:

- Referenced `/mnt/nvme/backups` — a path that does not exist on this machine
- Referenced `/mnt/ssd/nextcloud_data` instead of the actual `/mnt/ssd/nextcloud/data`
- Hardcoded the database password separately from `.env`, which would silently drift out of sync if the password changed
- Included a named-volume backup step using `docker run -v named_volume:/volume ...` — meaningless for a bind-mount setup, which would have silently created empty, unused volumes rather than backing up anything
- Wrote its log to `/var/log/`, which requires root and would fail under a non-root cron job

### Challenge 5: Restore script sequencing bug and service-name confusion

The initial restore script draft shared the same path and credential errors, plus two additional bugs:

- Tried to restore a `db_volume.tar.gz` that the corrected backup script never produces — the backup uses a SQL dump via `mariadb-dump`, not a raw volume archive
- Stopped both the database and application containers before the restore, when the database container must be running to accept the import

After fixing both issues, a second bug appeared on the first real-world run: `docker compose stop` failed with `no such service: nextcloud`. The script was passing the container names (`nextcloud`, `nextcloud-cron`, as set via `container_name:` in the Compose file) to `docker compose` subcommands, which require the Compose service names (`app`, `cron`) instead.

### Challenge 6: Internal Server Error after a successful restore

After completing a full backup-and-restore cycle, the Nextcloud web UI returned a generic Internal Server Error. Running `occ status` showed the installation as healthy (`installed: true`, `maintenance: false`, `needsDbUpgrade: false`). The error was only visible in the Nextcloud application log at `data/nextcloud.log`, not in the Apache access log that `docker logs` surfaces by default.

The application log showed repeated `Punic\Exception\DataFileNotFound` errors for `weekData` and `parentLocales`. I ran `diff -rq` between the live `html/` tree and the pristine reference copy bundled inside the container at `/usr/src/nextcloud`, which revealed that six `data/` subdirectories were missing under `3rdparty/` — covering `punic`, `aws-sdk-php`, `libphonenumber`, `symfony/string`, `symfony/translation`, and `wapmorgan/mp3info`. These are third-party library data directories, not user files, and they had not survived the rsync pass.

### Challenge 7: Intermittent loss of SSH and web UI access

For roughly 15 minutes the Pi became unreachable over SSH and the web UI timed out, while the local console continued to work normally. Container uptimes and system uptime confirmed no reboot had occurred.

I systematically worked through possible causes with log evidence: the DHCP lease was stable (no IP change), no reboot or OOM events had occurred, `vcgencmd get_throttled` returned `0x0` (no undervoltage or thermal throttling), Docker showed no abnormal resource usage, the nightly backup had completed hours earlier, WiFi signal was strong (-47dBm, full bitrate), and there were no disconnect or roaming events in the wireless logs. Tailscale logs showed route flapping around the same period, but the same pattern was present across the entire monitoring window, not just during the outage, so I treated it as inconclusive.

---

## Troubleshooting

### Boot failure: correcting the PARTUUID mismatch

I passed the USB SD card reader through to WSL2 using `usbipd-win`, ran `blkid` to read the actual PARTUUIDs currently written on the card, then opened `/etc/fstab` and corrected the stale PARTUUID values in the SSD mount entry and the boot partition entry to match `blkid` output. The card booted normally after remounting.

### Spare card: reformat rather than repair

`badblocks -sv` (read-only scan) reported zero bad sectors, confirming the storage medium itself was healthy. `e2fsck -f -n` (no-repair, report-only) aborted with directory-level corruption. Because no data needed to be preserved, I reformatted the card using Raspberry Pi Imager rather than running `e2fsck` in repair mode, which risks turning filesystem corruption into data loss.

### docker-compose.yml corruption

Rather than hunting through the file for a single corrupted character, I rewrote it from scratch and ran `docker compose config` to validate the YAML structure before attempting to bring the stack up.

### Backup and restore script rewrites

I rewrote both scripts from scratch rather than patching the templates:

- Replaced all hardcoded paths and credentials with variables sourced from the same `.env` the Compose stack uses
- Replaced the broken named-volume backup step with an `rsync` of the `html/` bind-mount directory, which preserves `config.php`
- Moved log output to a directory the service account owns
- Kept the database container running during restore and added a `mariadb-admin ping` readiness loop before importing the SQL dump
- Introduced separate variables for Compose service names (used with `docker compose` subcommands) and container names (used with `docker exec`)

### Internal Server Error after restore: missing 3rdparty data directories

I confirmed the application layer was healthy via `occ status`, then checked `data/nextcloud.log` directly (not `docker logs`, which only shows Apache access logs). The `Punic\Exception\DataFileNotFound` errors pointed to missing data files under `3rdparty/`.

I ran `diff -rq /mnt/ssd/nextcloud/html/3rdparty /usr/src/nextcloud/3rdparty` (comparing the live tree against the pristine copy bundled in the container) to identify exactly which directories were absent. I copied the six missing `data/` subdirectories from `/usr/src/nextcloud/3rdparty/` into the live `html/` tree, then ran `chown -R www-data:www-data` on the affected paths. No external download was required — the Nextcloud image ships the full reference tree under `/usr/src/nextcloud/` for use by its own update mechanism.

### WiFi power-save: disabling persistently

After ruling out all other causes, I identified WiFi power-save mode as the root cause. I disabled it persistently so the setting survives reboot.

---

## Lessons Learned

1. **Raspberry Pi Imager regenerates PARTUUIDs on re-flash but does not update `/etc/fstab`.** Any hand-edited `fstab` entries — such as an added SSD mount — will silently reference stale PARTUUIDs after a re-flash. After any re-flash, I need to reconcile `fstab` against `blkid` output before booting.

2. **`docker compose` subcommands take the Compose service name, not `container_name`.** These can differ — in this stack `app` and `cron` are the service names, while `nextcloud` and `nextcloud-cron` are the container names. Passing the wrong one fails with `no such service` and gives no hint that the actual issue is a name mismatch rather than a missing service.

3. **Bind mounts and named Docker volumes require different backup strategies.** A generic backup script written against named-volume assumptions will fail silently against a bind-mount setup — the volume step runs without error but backs up nothing. Before reusing any template script, I need to verify which storage model the Compose file actually uses.

4. **A Nextcloud backup must include `config.php`, not just user data.** `config.php` carries the instance ID, secret, and password salts. A backup that omits it produces a database dump that cannot be restored correctly against a fresh Nextcloud install — the credentials won't match.

5. **Running `e2fsck` against a live, mounted filesystem produces misleading output.** Apparent inconsistencies (wrong free block counts, `deleted inode has zero dtime`) are an artefact of the filesystem changing during the read-only scan, not evidence of real corruption. A meaningful `e2fsck` check requires the filesystem to be unmounted or inspected from a separate machine.

6. **WiFi power-save mode on a headless Pi causes multi-minute reachability drops that leave no trace in standard logs.** There are no disconnect events, no reassociation events, the signal stays strong, and the IP lease remains stable. For any WiFi-only Pi deployment acting as a server, disabling power-save mode is a baseline step, not an afterthought.

7. **Purpose-built, always-on monitoring is more useful than reconstructing events from general-purpose logs after the fact.** Several hours were spent piecing together what happened during the connectivity outage from logs that were not designed to capture that context. The network monitor script I added afterward would have shown the exact moment reachability dropped and whether it correlated with a WiFi event.

8. **The Nextcloud container ships a pristine reference copy of the application at `/usr/src/nextcloud/`.** This is the source the container's update mechanism uses. It is also useful as a diff target to identify files missing from the live `html/` tree after a restore, and as a source to copy them back from without any external download.

---

## Future Improvements

- **Wrap the network monitor in a systemd service** with a daily restart to handle midnight log rollover correctly, rather than running it as a background script that appends to the previous day's file indefinitely.
- **Add an integrity check to the backup script** that verifies the SQL dump and the rsync archives are non-empty before marking a backup run as successful, and sends a notification on failure.
- **Prune inactive Tailscale peers** — two peers have been offline for 17 and 22 days at the time of writing. They are not causing operational problems, but they add background retry noise to `tailscaled` logs and represent stale access credentials that should be removed.
- **Investigate migrating the restore script's rsync step** to explicitly exclude the `3rdparty/*/data/` directories from backup and instead copy them from the container's `/usr/src/nextcloud/` reference after each restore, which would avoid the missing-data-directory class of error entirely.
- **Consider adding a UPS or at least undervoltage monitoring** — the Pi runs on WiFi with no ethernet fallback and no battery backup. A power blip or undervoltage event that corrupts the SD card would require the full rebuild process documented here to repeat.
- **Document the `.env` schema and the Compose file** in this repository so the rebuild process can be completed from documentation alone without relying on memory.
