# Setup and configuration

## Enabling multisite (canonical sequence)

1. Backup DB + wp-config.php.
2. Add above `/* That's all, stop editing! */`:
   ```php
   define( 'WP_ALLOW_MULTISITE', true );
   ```
3. Admin → Tools → Network Setup. Choose subdirectories or subdomains.
   - Subdirectories require pretty permalinks.
   - Subdomains require a wildcard DNS record and a wildcard vhost/server block
     (`*.example.com` pointing at the install).
4. Network Setup prints a constants block; paste it into wp-config.php:
   ```php
   define( 'MULTISITE', true );
   define( 'SUBDOMAIN_INSTALL', false ); // true = subdomains
   define( 'DOMAIN_CURRENT_SITE', 'example.com' );
   define( 'PATH_CURRENT_SITE', '/' );
   define( 'SITE_ID_CURRENT_SITE', 1 );
   define( 'BLOG_ID_CURRENT_SITE', 1 );
   ```
5. Add the rewrite rules it prints (Apache: mod_rewrite block for .htaccess;
   nginx: `try_files $uri $uri/ /index.php?$args;` plus the multisite `map`/server setup).

Do not hand-write the constants from memory; use the generated block so values match
the install. Converting single → multisite is effectively one-way: back out first if unsure.

## Constants reference

| Constant | Meaning |
| --- | --- |
| `WP_ALLOW_MULTISITE` | Shows the Network Setup tool |
| `MULTISITE` | Network is on |
| `SUBDOMAIN_INSTALL` | `true` subdomains, `false` subdirectories |
| `DOMAIN_CURRENT_SITE` / `PATH_CURRENT_SITE` | Main site host + path |
| `SITE_ID_CURRENT_SITE` / `BLOG_ID_CURRENT_SITE` | Main network/site IDs (usually 1) |
| `SUNRISE` | Loads `wp-content/sunrise.php` before multisite bootstraps |
| `NOBLOGREDIRECT` | URL for nonexistent subdomains (signup page or custom) |
| `UPLOADBLOGSDIR` / `UPLOADS` | Legacy/custom uploads layout; leave default unless migrating old installs |
| `WP_INSTALL_NETWORK` | Set by core during network creation, not by you |

## Tables added by multisite

- `wp_blogs` — one row per site (domain, path, registered, last_updated, status flags).
- `wp_site` — the network record (one row; multi-network is not core).
- `wp_sitemeta` — network-wide options (`get_site_option()` storage).
- Per-site tables: `$wpdb->get_blog_prefix( $blog_id )` (main site uses the bare
  `wp_` prefix; site 2 uses `wp_2_`).

## Domain mapping

- Native: since 4.5 a site can have its own domain (Network Admin → Sites → Edit site
  → Info tab: Domain/Path). DNS for the mapped domain must point at the install.
- Cross-domain cookies/logins do not work out of the box because core sets cookies for
  `DOMAIN_CURRENT_SITE`. `sunrise.php` (enabled via `define( 'SUNRISE', true );`) runs
  before multisite loads and can rewrite `COOKIE_DOMAIN` / current blog per host.
- Prefer native mapping; use `sunrise.php` only for cross-domain login, legacy mapping
  plugins, or multi-network setups. Keep sunrise minimal and version-controlled: it runs
  on every request.

## wp-config placement rules

- All multisite constants above the stop line; `SUNRISE` before `MULTISITE` is fine —
  sunrise itself loads from `wp-content/sunrise.php`.
- Never set `WP_ALLOW_MULTISITE`/`MULTISITE` conditionally by hostname in production;
  it changes table routing and cookie behavior per request.

## Verify after enabling

- `wp site list` returns the main site.
- Network admin loads at `/wp-admin/network/`.
- Create one test subsite; front end + admin resolve; delete it after.
