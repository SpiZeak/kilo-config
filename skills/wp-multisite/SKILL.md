---
name: wp-multisite
description: "Use when working with WordPress Multisite/Networks: enabling and configuring multisite (wp-config constants, subdomain vs subdirectory), network admin, site lifecycle, domain mapping and sunrise.php, and writing multisite-aware code (switch_to_blog, network activation, per-site tables, site options, super admins)."
compatibility: "Targets WordPress 7.0+ (PHP 7.4.0+). Filesystem-based agent with bash + node. Some workflows require WP-CLI."
---

# WP Multisite

## When to use

Use this skill for Multisite (network) work such as:

- enabling/converting an install to multisite (`WP_ALLOW_MULTISITE`, generated constants)
- choosing subdomain vs subdirectory, wildcard DNS/vhosts, rewrite rules
- site lifecycle: create, archive, spam, deactivate, delete; site status side effects
- domain mapping, `sunrise.php`, `COOKIE_DOMAIN` and cross-domain login/cookies
- writing multisite-aware plugins/themes: `switch_to_blog()`, site vs network options,
  network activation, multisite hooks, per-site tables and uploads
- super admins vs per-site roles and capabilities

WP-CLI targeting (`--url`, `--network`, `wp site list` iteration) lives in
`wp-wpcli-and-ops` (`references/multisite.md`); this skill covers the concepts
that make those flags meaningful.

## Inputs required

- Whether the install is already multisite (`is_multisite()`, `wp core is-installed --network`,
  `MULTISITE` in wp-config.php) or being converted.
- Subdomain or subdirectory decision (subdomain needs wildcard DNS + vhost).
- Where config can be edited (wp-config.php, webserver config) and whether that is
  in-repo or on the server.

## Procedure

### 0) Detect the network shape (deterministic)

1. Grep wp-config.php for `WP_ALLOW_MULTISITE`, `MULTISITE`, `SUBDOMAIN_INSTALL`,
   `DOMAIN_CURRENT_SITE`, `SUNRISE`.
2. `wp site list --fields=blog_id,url,archived,spam,deleted` (or
   `wp-cli.yml` defaults) to see the sites involved.
3. For code tasks, identify main site ID (usually 1) and the blogs the code touches.

### 1) Enable or configure multisite

Follow the canonical sequence; do not hand-invent the constants block:

- `WP_ALLOW_MULTISITE` → admin Network Setup → copy the generated block into wp-config.php.
- Match rewrite rules to the server (Apache mod_rewrite vs nginx `try_files`).

Read:
- `references/setup-and-config.md`

### 2) Site lifecycle and network admin

- Prefer explicit status transitions; deleting a site drops its tables unless kept.
- Users are global; roles/capabilities are per-site (usermeta keys like
  `wp_{$blog_id}_capabilities`).

Read:
- `references/sites-users-and-network-admin.md`

### 3) Write multisite-aware code

Key correctness rules:

- Every `switch_to_blog()` is paired with `restore_current_blog()`; avoid switching
  in tight loops (it re-initializes large parts of WP).
- Network options via `get_site_option()`/`update_site_option()`; blog options via
  `get_option()`. Know which one your feature needs.
- Activation hooks run once when network-activated; per-site setup must iterate sites
  explicitly.

Read:
- `references/development-patterns.md`

### 4) Domain mapping / sunrise

- Native per-site domains first; `sunrise.php` only when you need cross-domain
  cookies or legacy mapping behavior.
- Mapped domains change cookie behavior: test login + admin on a mapped domain
  before shipping.

Read:
- `references/setup-and-config.md` (domain mapping section)

## Verification

- Create a throwaway site and verify its front end, admin, and login; then remove it.
- For code changes: exercise the code against both the main site and a secondary site
  (or via `switch_to_blog()` in a test), then run the repo's tests.
- After config changes: `wp site list` still resolves, admin loads on the network
  domain, and one site's content did not leak into another.
- Uploads: upload a file on a non-main site and confirm the URL resolves under
  `wp-content/uploads/sites/<id>/`.

## Failure modes / debugging

- Login redirect loop on a mapped domain: `COOKIE_DOMAIN` / sunrise not configured.
- Subdirectory subsite 404s: missing rewrite rules or pretty permalinks off.
- `switch_to_blog()` "loses" plugins/options: something cached across the switch, or
  `restore_current_blog()` missing on an early-return path.
- Network-activated plugin missing per-site data: activation hook ran once;
  per-site migration never executed.
- Deleting a site removed its tables unexpectedly: use `wp site delete --keep-tables`
  when data must survive.

## Escalation

- For canonical detail, consult the Multisite Network Handbook before inventing patterns.
- Environment-specific runs (Playground `multisite=yes`, wp-env, DevWP containers):
  use `wp-playground` / `devwp` for the runtime, this skill for the concepts.
