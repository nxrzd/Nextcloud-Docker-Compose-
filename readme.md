Nextcloud (Docker Compose)

Self-hosted Nextcloud stack I run on a Debian VM in my homelab, behind a Caddy reverse proxy. Mirrors a real running setup — clone it, fill in a few values, and you should get something close to mine.

Stack

ServiceImagePurposeappnextcloud:33-apacheNextcloud + Apachedbmariadb:11.4Databaseredisredis:7-alpineCaching + file lockingcronnextcloud:33-apacheBackground jobs (/cron.sh, 5-min schedule)

Pinned to Nextcloud 33 (the current stable/production track) rather than latest, so major-version upgrades stay deliberate. Redis caching, APCu, and distributed file locking are auto-wired by the image once REDIS_HOST is set.

Requirements


A Linux host (this runs on Debian). Docker Engine + Compose plugin.
A reverse proxy in front for TLS (Caddy example below).


Install Docker on Debian

bash# Official Docker apt repo (not Docker Desktop)
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/debian $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

Setup

bashgit clone <this-repo> nextcloud && cd nextcloud
cp .env.example .env

# Generate strong secrets and paste them into .env
openssl rand -base64 24

# Edit .env: secrets, your domain, trusted proxy IP, data path
nano .env

docker compose up -d
docker compose logs -f app    # watch the first-run install complete

Then set your region to clear the admin warning:

bashdocker exec -u www-data nextcloud-app php occ config:system:set default_phone_region --value="CA"

Reverse proxy (Caddy)

nextcloud.example.com {
    reverse_proxy 127.0.0.1:8080
    header Strict-Transport-Security "max-age=31536000;"
    redir /.well-known/carddav /remote.php/dav 301
    redir /.well-known/caldav /remote.php/dav 301
}

If your proxy runs on a different host than Nextcloud, set HTTP_BIND=<lan-ip>:8080 in .env and point reverse_proxy at that address.

Data directory

NEXTCLOUD_DATA_PATH (default ./data) is bind-mounted as Nextcloud's data dir.


Fresh install: the path must be empty. Nextcloud's file index lives in the database, so dropping an existing data dir under a brand-new DB just gives you an empty install sitting on orphaned files.
Production: point it at a dedicated mount (I use a LUKS-encrypted volume, e.g. /mnt/storage/nextcloud/data).
Migrating an existing instance: restore the matching DB dump and config.php alongside the data, and pin the image to the major version that wrote that data — mismatched majors won't boot.


Security notes


.env is gitignored. Never commit real secrets, domains, or your data dir.
The app port binds to 127.0.0.1 by default — TLS is terminated at the reverse proxy.
Upgrade one major version at a time (bump the image tag, then docker compose pull && docker compose up -d).
