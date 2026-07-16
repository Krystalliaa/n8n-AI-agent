# Repository Memory

## Nextcloud Home Server (Raspberry Pi 4B)

**Project:** Self-hosted Nextcloud instance on a Raspberry Pi 4B (4GB RAM), headless, WiFi-only.
**Doc file:** `docs/nextcloud-home-server.md`
**Last major session:** Boot recovery → full stack rebuild → backup/restore automation → WiFi fix → Claude Code installation.

**Hardware:**
- Boot: microSD card (Raspberry Pi OS, headless)
- Storage: 1.9TB USB SSD mounted at `/mnt/ssd`
- Network: WiFi only (wlan0), power-save disabled persistently
- Remote access: Tailscale

**Stack:** Four Docker containers via Compose — `db` (MariaDB), `redis`, `app` (Nextcloud, port 8080), `cron` (Nextcloud background jobs). All persistent data uses bind mounts under `/mnt/ssd/nextcloud/{db,redis,html,data}`. Secrets in a single `.env` shared by Compose and automation scripts.

**Automation:**
- `/usr/local/bin/backup_nextcloud.sh` — daily 03:00 cron; maintenance mode → mariadb-dump → rsync html/ + data/ → datestamped archive in `/mnt/ssd/backups/`; 30-day retention
- `/usr/local/bin/restore_nextcloud.sh` — manual; keeps db container running; mariadb-admin ping readiness loop; imports SQL dump; rsync html/ + data/; restarts app+cron services only
- `/mnt/ssd/network-monitor/` — continuous 5-second polling log (uptime, wlan link, ping router, ping 1.1.1.1, SSH port, docker ps, routes, neighbours)

**On-device tooling:** Claude Code installed on the Pi for AI-assisted troubleshooting over SSH/Tailscale.

**Key lessons (operational):**
- Raspberry Pi Imager regenerates PARTUUIDs on re-flash but does NOT update `/etc/fstab` — must reconcile after any re-flash
- `docker compose` subcommands require Compose service names (`app`, `cron`), not `container_name` values (`nextcloud`, `nextcloud-cron`)
- WiFi power-save mode causes multi-minute reachability drops with no trace in standard logs — disable it as baseline for any headless Pi server
- Nextcloud container ships pristine reference copy at `/usr/src/nextcloud/` — use for diff and recovery of missing files after restore
- Bind mounts require rsync-based backup; named-volume backup steps silently back up nothing against this stack
- `config.php` must be included in backup (inside `html/`) — it carries instance ID, secret, and password salts
- `e2fsck` against a live mounted filesystem produces misleading output; must unmount or use separate machine
- `docker logs` for the Nextcloud container shows Apache access logs only; application errors are in `data/nextcloud.log`

**Known gaps / future work:**
- Wrap network monitor in a systemd service with daily restart for correct log rollover
- Add backup integrity check (non-empty dump/archive assertion) with failure notification
- Prune stale Tailscale peers (2 peers offline 17–22 days at last check)
- Consider excluding `3rdparty/*/data/` from rsync backup and copying from container reference post-restore instead
- Consider UPS or undervoltage monitoring
- Document `.env` schema and Compose file structure in repo
