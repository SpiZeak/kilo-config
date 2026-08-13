---
name: devwp
description: "Use when working on DevWP projects — Local by Flywheel WordPress sites running in Docker containers (devwp_php, devwp_mariadb, devwp_nginx, devwp_redis, devwp_mailpit). Covers calling WP-CLI and MariaDB via docker exec, discovering the site path / DB name, and container gotchas."
compatibility: "Requires Docker and a running Local by Flywheel stack (containers prefixed devwp_). Filesystem-based agent with bash."
---

# DevWP (Local by Flywheel) Ops

## When to use

Use this skill whenever the task needs to reach a WordPress site's CLI or
database in a Local by Flywheel / DevWP environment. The bare host shell cannot
run `wp` or `mariadb` directly and cannot resolve container hostnames — every
command goes through `docker exec` into one of the `devwp_*` containers.

## Container topology

One shared Local stack serves **multiple** WordPress sites. There is a single
set of containers:

| Container       | Role                      | Published ports       |
| --------------- | ------------------------- | --------------------- |
| `devwp_php`     | PHP-FPM + WP-CLI + site files | 9000/tcp (internal) |
| `devwp_mariadb` | Single MySQL/MariaDB instance for all sites | 3306 |
| `devwp_nginx`   | Web server (vhosts per site) | 80, 443 |
| `devwp_redis`   | Redis object cache        | 6379 |
| `devwp_mailpit` | Mail catcher (SMTP + web UI) | 1025 (SMTP), 8025 (UI) |

Each site is a directory under `/src/www/<site>.test/web/` inside
`devwp_php`, and has its own database in `devwp_mariadb`.

## Step 0) Discover what is running

```bash
docker ps --format '{{.Names}}\t{{.Ports}}'
```

List the sites (paths inside `devwp_php`):

```bash
docker exec devwp_php sh -c 'ls -d /src/www/*'
```

List the databases (all in the one MariaDB instance, creds `root` / `root`):

```bash
docker exec devwp_mariadb mariadb -uroot -proot -e "SHOW DATABASES;"
```

**Do not guess the site path or DB name — derive them from the listings above.**
The database name is the site domain slug with dots/hyphens replaced by
underscores plus a `_test` suffix (e.g. `unison.test` → `unison_test`,
`nya-otheme.test` → `nya_otheme_test`), but always confirm against
`SHOW DATABASES` since the suffix/slug can vary per install.

## Step 1) WP-CLI

The canonical invocation targets a site's WordPress root inside `devwp_php`:

```bash
docker exec devwp_php wp --path=/src/www/<site>.test/web/wp <command>
```

Example (list plugins for `unison.test`):

```bash
docker exec devwp_php wp --path=/src/www/unison.test/web/wp plugin list
```

### Gotchas

- **`--path` is required and must be the container path** (`/src/www/...`), not
  the host path. The host repo path (`/home/max/www/unison.test`) is bind-mounted
  into the container, but `wp` resolves relative to `/src/www/`.
- **Deprecated noise**: WP core + plugin autoloaders emit `PHP Deprecated:`
  warnings to stderr. Filter them:
  ```bash
  docker exec devwp_php wp --path=/src/www/unison.test/web/wp <cmd> 2>&1 | grep -v Deprecated
  ```
- **Commands that need an authenticated user** (e.g. WooCommerce `wc ...`):
  pass `--user=<login>` (e.g. `--user=frontend`) or the command returns
  empty/error:
  ```bash
  docker exec devwp_php wp --path=/src/www/unison.test/web/wp wc shipping_zone list --user=frontend
  ```
- No `wp` binary exists on the host and `mariadb` hostname does not resolve
  from the host — never run these bare.

## Step 2) Database

Use the `mariadb` client (the binary is `mariadb`, **not** `mysql`) inside
`devwp_mariadb`, creds `root` / `root`, targeting the site's database:

```bash
docker exec devwp_mariadb mariadb -uroot -proot <db_name> -e "<SQL>"
```

Example:

```bash
docker exec devwp_mariadb mariadb -uroot -proot unison_test -e "SELECT option_value FROM wp_options WHERE option_name='siteurl';"
```

For imports/exports pipe files in/out:

```bash
docker exec -i devwp_mariadb mariadb -uroot -proot unison_test < dump.sql
docker exec devwp_mariadb mariadb -uroot -proot unison_test -e "SHOW TABLES;"
```

## Step 3) Other services

- **Mail**: Mailpit catches all outbound mail. UI at `http://localhost:8025`.
  SMTP relay from PHP uses host `devwp_mailpit` port `1025` (internal).
- **Redis**: `devwp_redis` on `6379`; object-cache connection string uses the
  `devwp_redis` hostname from inside `devwp_php`.
- **Nginx**: `devwp_nginx` serves `https://<site>.test` (self-signed). Browsers
  need a hosts entry or Local's DNS/trust. Site only resolves if `.env` +
  DB + media are in place.

## Verification

- After running WP-CLI writes, re-run a read (`wp option get`, `wp plugin list`,
  `wp db check`) to confirm the change landed.
- For DB changes, re-query the affected rows with `mariadb -e`.
- If a site does not respond, confirm its directory exists under `/src/www/`
  and its DB exists in `SHOW DATABASES`.

## Failure modes

- `This does not seem to be a WordPress installation` → wrong `--path` (must be
  `/src/www/<site>.test/web/wp`, not the host path).
- `Access denied for user 'root'` → wrong creds (expected `root`/`root`) or
  wrong host (always `docker exec devwp_mariadb`, not `-h 127.0.0.1`).
- WooCommerce/WP-CLI returns empty → missing `--user`.
- Unknown database → the site slug/suffix was guessed; confirm via
  `SHOW DATABASES`.
