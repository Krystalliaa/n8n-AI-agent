# Nextcloud Home Server on Raspberry Pi

## Introduction

I built a self-hosted family cloud server using Nextcloud on a Raspberry Pi 4, designed to replace commercial cloud storage services with a private, locally controlled alternative. The server is accessible both on the local network and remotely through Tailscale, with automated daily backups and a tested full-system restore capability. The project prioritized stability, security, and operational reliability over raw performance.

---

## Objectives

- Deploy Nextcloud using Docker on a Raspberry Pi 4 with persistent storage on an external SSD
- Enable secure remote access through Tailscale without exposing ports to the public internet
- Implement automated daily backups covering the database, data directory, Docker volumes, and Redis state
- Build a tested restore script capable of recovering the full system from a backup archive
- Configure Redis for distributed caching and file locking to improve performance and reliability

---

## Technologies Used

| Technology | Role |
|---|---|
| Raspberry Pi 4 (4GB) | Host hardware |
| Raspberry Pi OS Lite 64-bit | Host operating system |
| Docker + Docker Compose | Container runtime and orchestration |
| Nextcloud (latest) | Cloud server application |
| MariaDB 10.11 | Relational database backend |
| Redis 7 Alpine | Memory cache and file locking |
| Tailscale | Zero-config VPN for remote access |
| Kingston SSD 2TB (ext4) | Primary storage for data and backups |
| rsync | Data directory synchronization during backup |
| bash + cron | Automation scripting and scheduling |

---

## Architecture Overview

```
Raspberry Pi 4 (4GB)
├── OS: Raspberry Pi OS Lite (64-bit)
├── Docker
│   ├── nextcloud          (port 8080)
│   ├── mariadb:10.11
│   └── redis:7-alpine
├── Kingston SSD 2TB (ext4, mounted at /mnt/ssd)
│   ├── /mnt/ssd/nextcloud_data   → Nextcloud user data
│   └── /mnt/ssd/backups          → Backup archives
├── Tailscale
│   ├── IP:     100.105.62.64
│   └── Domain: pi.tail1920a9.ts.net
└── Cron Jobs
    ├── Daily backup at 03:00
    └── Nextcloud background jobs every 5 minutes
```

All three Docker containers communicate over a dedicated bridge network (`nextcloud-net`). Nextcloud user data is stored via a bind mount from the container path `/var/www/html/data` to `/mnt/ssd/nextcloud_data` on the host SSD, making data directly accessible to backup scripts without requiring a container intermediary. The MariaDB data directory and Redis data are stored in named Docker volumes. Tailscale Serve acts as a local TLS termination proxy, forwarding HTTPS traffic on port 443 to the Nextcloud container on port 8080.

---

## Implementation Process

### Phase 1 — Docker Stack Deployment

I installed Docker and Docker Compose on Raspberry Pi OS, then created a `docker-compose.yml` defining three services: the Nextcloud web application exposed on port 8080, MariaDB connected only to the internal bridge network, and Redis used for both `memcache.locking` and `memcache.distributed`. I initially stored user data on an external NVMe drive at `/mnt/nvme/nextcloud_data` using a bind mount.

**docker-compose.yml:**

```yaml
networks:
  nextcloud-net:
    driver: bridge

services:
  mariadb:
    image: mariadb:10.11
    container_name: mariadb
    restart: always
    networks:
      - nextcloud-net
    environment:
      MYSQL_ROOT_PASSWORD: strongpassword
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: nextcloudpass
    volumes:
      - mariadb_data:/var/lib/mysql

  redis:
    image: redis:7-alpine
    container_name: redis
    restart: always
    networks:
      - nextcloud-net
    volumes:
      - redis_data:/data

  nextcloud:
    image: nextcloud
    container_name: nextcloud
    restart: always
    networks:
      - nextcloud-net
    ports:
      - "8080:80"
    depends_on:
      - mariadb
      - redis
    volumes:
      - nextcloud_app:/var/www/html
      - /mnt/ssd/nextcloud_data:/var/www/html/data
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379

volumes:
  mariadb_data:
  redis_data:
  nextcloud_app:
```

### Phase 2 — Nextcloud Configuration

I configured `config.php` to enable Redis caching, register trusted domains for both local and Tailscale access, and set the overwrite settings required for Tailscale Serve to function correctly as a reverse proxy.

**config.php (final):**

```php
<?php
$CONFIG = array (
  'htaccess.RewriteBase' => '/',
  'memcache.local' => '\OC\Memcache\APCu',
  'apps_paths' =>
  array (
    0 =>
    array (
      'path' => '/var/www/html/apps',
      'url' => '/apps',
      'writable' => false,
    ),
    1 =>
    array (
      'path' => '/var/www/html/custom_apps',
      'url' => '/custom_apps',
      'writable' => true,
    ),
  ),
  'memcache.distributed' => '\OC\Memcache\Redis',
  'memcache.locking' => '\OC\Memcache\Redis',
  'redis' =>
  array (
    'host' => 'redis',
    'password' => '',
    'port' => 6379,
  ),
  'upgrade.disable-web' => true,
  'trusted_domains' =>
  array (
    0 => '192.168.3.190:8080',
    1 => '192.168.3.217',
    3 => '192.168.3.211',
    4 => '100.127.162.87',
    5 => '100.105.62.64',
    6 => '100.86.45.65',
    7 => '100.97.163.6',
    8 => '100.87.144.43',
    9 => 'pi.tail1920a9.ts.net',
  ),
  'datadirectory' => '/var/www/html/data',
  'dbtype' => 'mysql',
  'version' => '34.0.0.12',
  'overwrite.cli.url' => 'https://pi.tail1920a9.ts.net/',
  'overwriteprotocol' => 'https',
  'trusted_proxies' => ['100.64.0.0/10'],
  'dbname' => 'nextcloud',
  'dbhost' => 'mariadb',
  'dbtableprefix' => 'oc_',
  'mysql.utf8mb4' => true,
  'dbuser' => 'nextcloud',
  'dbpassword' => 'nextcloudpass',
  'installed' => true,
  'maintenance' => false,
);
```

### Phase 3 — SSD Migration

The original NVMe drive developed filesystem corruption and persistent I/O errors after `fsck` repair. I replaced it with a Kingston SSD 2TB and performed a clean migration:

```bash
# Partition and format the new SSD
parted /dev/sda mklabel gpt
parted /dev/sda mkpart primary ext4 0% 100%
mkfs.ext4 /dev/sda1

# Mount and set correct ownership for the www-data user (UID 33)
mount /dev/sda1 /mnt/ssd
chown -R 33:33 /mnt/ssd/nextcloud_data
chmod -R 770 /mnt/ssd/nextcloud_data
```

I restored the system from the last known good backup (`nextcloud_2026-06-29_20-05-34.tar.gz`) using the corrected restore script, updated `docker-compose.yml` to point the bind mount to `/mnt/ssd/nextcloud_data`, and configured persistent auto-mount via `/etc/fstab`.

**fstab entry:**

```
# System partitions (DO NOT REMOVE)
PARTUUID=f1a02bdc-02  /               ext4  defaults,noatime  0  1
PARTUUID=f1a02bdc-01  /boot/firmware  vfat  defaults          0  2

# External SSD (Nextcloud storage)
UUID=148091b9-0858-4722-9c4a-fe1d70b96d89  /mnt/ssd  ext4  defaults,nofail,x-systemd.device-timeout=10,noatime  0  2
```

The `nofail` and `x-systemd.device-timeout=10` options prevent the Pi from hanging at boot if the SSD is temporarily unavailable. After confirming the new installation was fully operational, I removed the old mount points:

```bash
rm -rf /mnt/nvme_data
rm -rf /mnt/nvme
```

### Phase 4 — Remote Access via Tailscale

I enabled Tailscale Serve to terminate TLS and proxy HTTPS traffic to the local Nextcloud container:

```bash
sudo tailscale serve --https=443 localhost:8080
```

This created a valid HTTPS endpoint at `https://pi.tail1920a9.ts.net` using Tailscale's managed certificates, without requiring any port forwarding or public internet exposure.

### Phase 5 — Backup and Restore Automation

I wrote two scripts deployed to `/usr/local/bin/`.

**backup_nextcloud.sh** enables maintenance mode, dumps the MariaDB database using a temporary `mariadb:10.11` container, rsyncs the data directory, archives the Docker app volume and Redis state into a timestamped `.tar.gz`, verifies the archive integrity, and enforces a 30-day retention policy.

**restore_nextcloud.sh** accepts a specific backup file path or the `--latest` flag to auto-select the most recent archive. It extracts the backup, stops all containers, drops and recreates the database, restores the data directory and Docker volumes, restarts all containers in dependency order, and disables maintenance mode.

Both scripts use `mariadb:10.11` as a temporary container for all database operations and connect via `-h 127.0.0.1` over TCP rather than a Unix socket.

**backup_nextcloud.sh:**

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

BACKUP_DIR="/mnt/ssd/backups"
DATE=$(date +"%Y-%m-%d_%H-%M-%S")
TMP_DIR="$BACKUP_DIR/tmp_$DATE"
ARCHIVE="$BACKUP_DIR/nextcloud_$DATE.tar.gz"
RETENTION_DAYS=30
DB_CONTAINER="mariadb"
NC_CONTAINER="nextcloud"
REDIS_CONTAINER="redis"
DB_NAME="nextcloud"
DB_USER="nextcloud"
DB_PASSWORD="nextcloudpass"
DATA_DIR="/mnt/ssd/nextcloud_data"
LOG_FILE="/var/log/nextcloud-backup.log"
LOCK_FILE="/tmp/nextcloud-backup.lock"

log() { echo "[$(date '+%F %T')] $1" | tee -a "$LOG_FILE" }

cleanup() {
  docker exec --user www-data "$NC_CONTAINER" php occ maintenance:mode --off >/dev/null 2>&1 || true
  rm -rf "$TMP_DIR" 2>/dev/null || true
  rm -f "$LOCK_FILE" 2>/dev/null || true
}
trap cleanup EXIT

if [ -f "$LOCK_FILE" ]; then log "Backup already running"; exit 1; fi
touch "$LOCK_FILE"
mkdir -p "$TMP_DIR"

# Check containers
for c in "$DB_CONTAINER" "$NC_CONTAINER"; do
  if ! docker ps --format '{{.Names}}' | grep -q "^${c}$"; then
    log "ERROR: Container '$c' is not running"; exit 1
  fi
done

# Enable maintenance mode
log "Enabling maintenance mode..."
docker exec --user www-data "$NC_CONTAINER" php occ maintenance:mode --on

# Database backup
log "Backing up database..."
docker run --rm \
  --network container:"$DB_CONTAINER" \
  mariadb:10.11 \
  mysqldump -h 127.0.0.1 -u"$DB_USER" -p"$DB_PASSWORD" "$DB_NAME" > "$TMP_DIR/database.sql"
log "Database backup saved."

# Data directory backup
log "Backing up Nextcloud data from $DATA_DIR ..."
rsync -a --delete "$DATA_DIR/" "$TMP_DIR/data/"
log "Data backup completed."

# Docker app volume backup
log "Backing up Docker volumes..."
docker run --rm \
  -v nextcloud_app:/volume \
  -v "$TMP_DIR":/backup \
  busybox \
  tar czf /backup/nextcloud_app.tar.gz -C /volume .
log "Docker volumes backup completed."

# Redis backup
if docker ps --format '{{.Names}}' | grep -q "^${REDIS_CONTAINER}$"; then
  log "Backing up Redis data..."
  docker run --rm \
    -v redis_data:/data \
    -v "$TMP_DIR":/backup \
    busybox \
    tar czf /backup/redis_data.tar.gz -C /data .
  log "Redis backup completed."
else
  log "WARNING: Redis container not found. Skipping."
fi

# Create and verify archive
log "Creating compressed archive: $ARCHIVE"
tar -czf "$ARCHIVE" -C "$TMP_DIR" .
if tar -tzf "$ARCHIVE" >/dev/null 2>&1; then
  log "Archive verification successful."
else
  log "ERROR: Archive verification failed!"; exit 1
fi

rm -rf "$TMP_DIR"
log "Temporary files cleaned."

# Retention policy
find "$BACKUP_DIR" -name "nextcloud_*.tar.gz" -type f -mtime +"$RETENTION_DAYS" -exec rm -f {} \;
log "Retention policy applied."

log "========================================="
log " Backup completed successfully!"
log "Archive: $ARCHIVE"
log "========================================="
```

**restore_nextcloud.sh:**

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

BACKUP_DIR="/mnt/ssd/backups"
NC_CONTAINER="nextcloud"
DB_CONTAINER="mariadb"
REDIS_CONTAINER="redis"
DB_NAME="nextcloud"
DB_USER="nextcloud"
DB_PASSWORD="nextcloudpass"
DATA_DIR="/mnt/ssd/nextcloud_data"
APP_VOLUME="nextcloud_app"
REDIS_VOLUME="redis_data"
DB_IMAGE="mariadb:10.11"
TMP="/tmp/nc-restore"

find_latest_backup() {
  LATEST=$(ls -t "$BACKUP_DIR"/nextcloud_*.tar.gz 2>/dev/null | head -1)
  if [ -z "$LATEST" ]; then echo "ERROR: No backups found in $BACKUP_DIR"; exit 1; fi
  echo "$LATEST"
}

BACKUP_FILE=""
while [[ $# -gt 0 ]]; do
  case $1 in
    -h|--help)   echo "Usage: $0 [--latest | --list | BACKUP_FILE]"; exit 0 ;;
    -l|--list)   ls -lh "$BACKUP_DIR"/nextcloud_*.tar.gz 2>/dev/null; exit 0 ;;
    -L|--latest) BACKUP_FILE=$(find_latest_backup); shift ;;
    *)           BACKUP_FILE="$1"; shift ;;
  esac
done
[ -z "$BACKUP_FILE" ] && BACKUP_FILE=$(find_latest_backup)

[ ! -f "$BACKUP_FILE" ] && { echo "ERROR: Backup file '$BACKUP_FILE' not found!"; exit 1; }
tar -tzf "$BACKUP_FILE" >/dev/null 2>&1 || { echo "ERROR: Backup file is corrupted!"; exit 1; }

echo "Using backup: $BACKUP_FILE"
echo ""
echo " WARNING: This will DELETE all current Nextcloud data!"
read -p "Continue? (yes/no): " CONFIRM
[ "$CONFIRM" != "yes" ] && { echo "Restore cancelled."; exit 0; }

# Extract
echo "[1/8] Extracting backup to $TMP"
rm -rf "$TMP"; mkdir -p "$TMP"
tar -xzf "$BACKUP_FILE" -C "$TMP"
[ ! -f "$TMP/database.sql" ] && { echo "ERROR: database.sql not found in backup!"; exit 1; }
[ ! -d "$TMP/data" ] && { echo "ERROR: data directory not found in backup!"; exit 1; }

# Stop containers
echo "[2/8] Stopping containers"
docker stop "$NC_CONTAINER" "$DB_CONTAINER" "$REDIS_CONTAINER" 2>/dev/null || true

# Restore database
echo "[3/8] Restoring database"
docker start "$DB_CONTAINER"; sleep 10
docker run --rm --network container:"$DB_CONTAINER" "$DB_IMAGE" \
  mysql -h 127.0.0.1 -u"$DB_USER" -p"$DB_PASSWORD" \
  -e "DROP DATABASE IF EXISTS $DB_NAME; CREATE DATABASE $DB_NAME CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;"
cat "$TMP/database.sql" | docker run --rm -i --network container:"$DB_CONTAINER" "$DB_IMAGE" \
  mysql -h 127.0.0.1 -u"$DB_USER" -p"$DB_PASSWORD" "$DB_NAME"
echo "Database restored."

# Restore data directory
echo "[4/8] Restoring data directory"
rsync -a --delete "$TMP/data/" "$DATA_DIR/"
echo "Data directory restored."

# Restore app volume
echo "[5/8] Restoring app volume"
if [ -f "$TMP/nextcloud_app.tar.gz" ]; then
  docker run --rm -v "$APP_VOLUME":/volume -v "$TMP":/backup \
    busybox sh -c "rm -rf /volume/* && tar -xzf /backup/nextcloud_app.tar.gz -C /volume"
  echo "App volume restored."
else
  echo "WARNING: No app volume backup found. Skipping."
fi

# Restore Redis
echo "[6/8] Restoring Redis data"
if [ -f "$TMP/redis_dump.rdb" ]; then
  docker run --rm -v "$REDIS_VOLUME":/data -v "$TMP":/backup \
    busybox sh -c "cp /backup/redis_dump.rdb /data/dump.rdb"
  echo "Redis data restored."
else
  echo "WARNING: No Redis backup found. Skipping."
fi

# Start containers
echo "[7/8] Starting containers"
docker start "$DB_CONTAINER"; sleep 10
docker start "$REDIS_CONTAINER"; sleep 5
docker start "$NC_CONTAINER"
echo "Containers started."

# Disable maintenance mode
echo "[8/8] Disabling maintenance mode"
docker exec --user www-data "$NC_CONTAINER" php occ maintenance:mode --off

echo "========================================="
echo " Restore completed successfully!"
echo "========================================="
```

**Cron schedule:**

```
# Daily Nextcloud backup at 03:00
0 3 * * * /usr/local/bin/backup_nextcloud.sh

# Nextcloud background jobs every 5 minutes
*/5 * * * * docker exec -u www-data nextcloud php -f /var/www/html/cron.php
```

---

## Challenges Encountered

### Read-Only Filesystem on NVMe Drive

The original NVMe drive developed filesystem corruption. Linux automatically remounted it as read-only (`emergency_ro`), causing Nextcloud to fail all write operations with `Read-only file system` and `Permission denied` errors in the application logs. Running `fsck` confirmed and repaired the filesystem errors, but persistent I/O errors continued to appear in the kernel logs afterward, indicating underlying hardware failure rather than logical corruption. I replaced the drive with a Kingston SSD 2TB.

### mysqldump Not Found in MariaDB 11 Image

The backup script failed immediately with `mysqldump: not found` because the `mariadb:11` container image does not include client tools by default. The `mariadb:10.11` image bundles the full client toolset including `mysqldump` and `mysql`. I updated all references in both the backup and restore scripts to use `mariadb:10.11`.

### Database Socket Connection Error During Restore

The restore script failed with `Can't connect to local server through socket '/run/mysqld/mysqld.sock'`. The script used `--network container:mariadb` to share the database container's network namespace, but this flag shares only the network stack, not the filesystem. The Unix socket file lives on the container's filesystem and is not accessible through the shared network namespace. Switching all `mysql` commands to use `-h 127.0.0.1` forced TCP connections over the shared loopback interface, which resolved the issue.

### Nextcloud Not Responding on Tailscale Domain

After enabling Tailscale Serve, the instance responded correctly on the Tailscale IP with port 8080 but returned errors on the `ts.net` domain. The domain was missing from `trusted_domains`, `overwriteprotocol` was not configured, and `trusted_proxies` did not include the Tailscale CGNAT range. I added the domain to `trusted_domains`, set `overwrite.cli.url` and `overwriteprotocol`, and added `100.64.0.0/10` to `trusted_proxies`.

### Loss of Local HTTP Access After Domain Configuration

After setting `overwriteprotocol` to `https`, local access via `http://192.168.3.190:8080` stopped working because Nextcloud redirected all connections to HTTPS, but the local address had no TLS certificate. I retained the local IP in `trusted_domains` and accepted that local clients use plain HTTP while remote clients use HTTPS through Tailscale Serve.

---

## Troubleshooting

| Symptom | Root Cause | Resolution |
|---|---|---|
| `Read-only file system` errors in Nextcloud logs | NVMe filesystem corruption; kernel remounted drive as read-only | Ran `fsck`, confirmed persistent I/O errors indicating hardware failure, replaced with Kingston SSD 2TB |
| `mysqldump: not found` in backup script | `mariadb:11` image does not include client tools | Changed all script references to `mariadb:10.11` |
| `Can't connect to local server through socket` during restore | `--network container:mariadb` shares network namespace only, not filesystem; Unix socket inaccessible | Used `-h 127.0.0.1` to force TCP connection over shared loopback |
| Nextcloud returns error on `ts.net` domain | Domain absent from `trusted_domains`; `overwriteprotocol` and `trusted_proxies` not configured | Added domain to `trusted_domains`, set `overwriteprotocol` and `trusted_proxies` with `100.64.0.0/10` |
| Local IP access broken after domain setup | `overwriteprotocol=https` redirected all connections including local HTTP | Retained local IP in `trusted_domains`; local access remains HTTP, remote access uses HTTPS via Tailscale |
| Browser redirect loop | Tailscale CGNAT range missing from `trusted_proxies` | Added `100.64.0.0/10` to `trusted_proxies` in `config.php` |

---

## Lessons Learned

- **MariaDB image version determines which tools are available.** The `mariadb:11` image does not ship `mysqldump` or `mysql` client binaries. The `mariadb:10.11` image does. When using a temporary container for database operations in automation scripts, the image version must be chosen based on tooling requirements, not just server compatibility.

- **Docker `--network container:X` shares only the network namespace, not the filesystem.** Unix sockets from the target container are part of its filesystem and are not accessible through this flag. TCP connections using `-h 127.0.0.1` are the correct approach for cross-container database operations when sharing a network namespace.

- **Tailscale Serve requires explicit `trusted_proxies` configuration in Nextcloud.** Without adding the Tailscale CGNAT range (`100.64.0.0/10`) to `trusted_proxies`, Nextcloud does not trust the forwarded `X-Forwarded-Proto` header, which produces redirect loops or incorrect protocol detection regardless of how `overwriteprotocol` is set.

- **`overwriteprotocol` is a global setting that affects all connections.** Setting it to `https` causes Nextcloud to force HTTPS redirects for every client, including those on the local network accessing via plain HTTP. I needed to explicitly account for both access patterns rather than assuming a single configuration would cover both.

- **Filesystem errors do not always indicate a failing drive.** `fsck` can repair logical corruption caused by unclean shutdowns or power loss. However, when I/O errors persist in kernel logs after a successful `fsck`, that is a reliable indicator of physical hardware failure requiring replacement, not further software-level repair.

- **Keeping the old drive available until the new installation is fully verified prevents data loss.** During the migration, the old NVMe remained the source of the backup archive used to restore onto the new SSD. Decommissioning it only after confirming a successful restore and passing access tests eliminated any risk of losing the only copy of data.

- **The `nofail` fstab option is critical on single-board computers.** Without it, a slow or absent external drive causes the Pi to stall at boot indefinitely, requiring physical access to recover. Pairing it with `x-systemd.device-timeout=10` sets an explicit timeout so the system continues booting even if the drive does not respond immediately.

---

## Future Improvements

- Add offsite backup replication using `rclone` to a second location or object storage such as Backblaze B2 to protect against local hardware failure
- Implement backup job notifications via email or a webhook to detect silent failures in the cron-scheduled backup script
- Migrate database credentials and passwords out of plaintext configuration files into Docker secrets or a restricted environment file with `chmod 600` permissions
- Explore replacing Tailscale Serve with a Caddy reverse proxy for more granular access control, request logging, and header management
- Set up a monitoring stack with Prometheus and Grafana to track disk usage trends, container health, and backup archive sizes over time
- Establish a documented update procedure for keeping Nextcloud, MariaDB, and Redis container images patched on a regular schedule
- Evaluate adding Nextcloud Talk for self-hosted video calling to further reduce reliance on third-party communication platforms
