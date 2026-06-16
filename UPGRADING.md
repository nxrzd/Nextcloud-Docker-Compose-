# Upgrading

Nextcloud upgrades **one major version at a time** — you can't skip. Going from 30 to 33 means stepping 30 → 31 → 32 → 33, pulling each tag in turn. The official image detects the version gap on container start and runs the upgrade itself; your job is to bump the tag and back up first.

## Before any upgrade: back up

```bash
# 1. Database dump (root password is already in the db container's env)
docker exec nextcloud-db sh -c \
  'exec mariadb-dump --single-transaction -u root -p"$MARIADB_ROOT_PASSWORD" nextcloud' \
  > backup-db-$(date +%F).sql

# 2. config.php and apps
docker cp nextcloud-app:/var/www/html/config ./backup-config-$(date +%F)

# 3. User data is your NEXTCLOUD_DATA_PATH bind mount — back that dir up
#    with your normal snapshot/rsync routine.
```

If you run Proxmox/ZFS snapshots on the VM, a snapshot right before the upgrade is the fastest rollback path.

## Patch / minor upgrade (same major, e.g. 33.0.4 → 33.0.5)

No tag change needed if you track the major tag (`33-apache`):

```bash
docker compose pull
docker compose up -d
docker compose logs -f app
```

## Major upgrade (e.g. 33 → 34)

Do **one** major per cycle. Edit the image tags in `docker-compose.yml`:

```yaml
# app AND cron services
image: nextcloud:34-apache   # was 33-apache
```

Then:

```bash
docker compose pull
docker compose up -d
docker compose logs -f app   # watch the upgrade run on first boot
```

To cross several majors, repeat the whole block for each step (33 → 34, then 34 → 35, …). Don't jump straight to the newest tag.

## After upgrading

```bash
# Confirm version + maintenance mode is off
docker exec -u www-data nextcloud-app php occ status

# Apply any indices/columns the upgrade recommends
docker exec -u www-data nextcloud-app php occ db:add-missing-indices
docker exec -u www-data nextcloud-app php occ db:add-missing-columns
docker exec -u www-data nextcloud-app php occ db:add-missing-primary-keys

# Optional: full integrity/setup check
docker exec -u www-data nextcloud-app php occ setupchecks
```

Then load the web UI and check **Administration settings → Overview** for any remaining warnings.

## Rollback

If an upgrade goes sideways:

1. Stop the stack: `docker compose down`.
2. Restore the VM snapshot (fastest), **or** restore `config/`, the DB dump, and the data dir from your backups.
3. Pin the image tag back to the previous major in `docker-compose.yml`.
4. `docker compose up -d`.

Restoring data without restoring the matching DB (or vice versa) will leave the instance inconsistent — always roll back all three together.
