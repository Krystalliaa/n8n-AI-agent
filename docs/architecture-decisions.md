Architecture Decisions

ADR-001
Date: 2025-07-10
Status: Accepted
Decision: All persistent Docker container data for the Nextcloud stack uses bind mounts into /mnt/ssd/nextcloud/{db,redis,html,data} rather than named Docker volumes. A single .env file at /mnt/ssd/nextcloud/.env is shared between the Compose stack and all automation scripts as the single source of truth for credentials and paths.
Reason: Bind mounts make backup and restore straightforward with standard rsync and mariadb-dump tooling, without requiring docker run intermediaries or volume inspection. A shared .env eliminates credential drift between the Compose stack and automation scripts. Separating html/ (application code + config.php) from data/ (user files) ensures Nextcloud updates do not overwrite user data.
Impact: Generic backup script templates written against named-volume assumptions (docker run -v named_volume:/volume ...) will silently back up nothing and must not be reused without verification. Restore scripts must use Compose service names (app, cron) not container_name values (nextcloud, nextcloud-cron) when invoking docker compose subcommands. The database container must remain running during restore to accept mariadb SQL imports.
Alternatives Considered: Named Docker volumes (rejected — harder to inspect, back up, and restore without Docker-specific tooling); separate .env files per script (rejected — credential drift risk); storing html/ and data/ in the same directory (rejected — Nextcloud updates would overwrite user files).


Decisions
