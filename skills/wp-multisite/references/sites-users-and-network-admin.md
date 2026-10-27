# Sites, users, and network admin

## Site lifecycle and status flags

Sites are rows in `wp_blogs` with status flags. Status changes are soft (site kept),
deletion is hard (tables dropped by default).

| Action | Effect |
| --- | --- |
| Archive | Public visitors see "site archived" notice; admins can still log in |
| Spam | Marked spam; excluded from listings; may affect network protections |
| Deleted | Marked deleted; front end stops resolving |
| Delete (permanent) | Tables dropped unless `--keep-tables`; uploads NOT removed |

- Restore flow: unarchive/unspam/undelete (admin: Sites → edit site status; CLI: `wp site unarchive` etc.).
- Permanent deletion: `wp site delete <id> [--keep-tables]` — verify the site ID twice
  before running; there is no undo without a backup.
- Sites created later: WP-CLI `wp site create --slug=<slug> --title=<title> [--email=<admin>]`,
  or Network Admin → Sites → Add New.

## Network admin surface

- Lives at `/wp-admin/network/`; requires super admin (or a user with the network-level caps).
- Screens: Sites, Users (network-wide), Themes (network enable/disable — per-site activation
  is separate), Plugins (network activation), Settings (`wpmu_options`), Updates.
- Network-enable vs activate: network-enabled themes/plugins are *available* to sites;
  individual sites still activate them (or the plugin is network-activated, which loads
  it on every site with no per-site activation).

## Users across sites

- `wp_users` and `wp_usermeta` are shared across the whole network; per-site role via
  `wp_{$blog_id}_capabilities` usermeta keys.
- Adding an existing user to another site: Network Admin → Sites → Edit → Users tab, or
  `wp user add-role <user> <role> --url=<site-url>` / `add_user_to_blog()`.
- Removing from one site does not remove the account from the network; deleting a user
  from Network Admin → Users removes them everywhere.
- Super admin: `wp super-admin add <user>` / `wp super-admin list` — bypasses all
  capability checks; grant sparingly.

## Operating across sites

For "run X on every site" work, iterate explicitly and prefer per-site commands over
bulk magic:

```
wp site list --field=url | xargs -n1 -I{} wp <command> --url={}
```

Read the targeting guardrails first: `wp-wpcli-and-ops/references/multisite.md`
(`--url` when you mean one site; `--network` where supported).

## Common ops checks

- `wp site list --fields=blog_id,url,archived,spam,deleted,registered,last_updated` —
  inventory plus stale-site detection (`last_updated` only moves when a site posts/updates).
- `wp site meta list <id>` (site meta API, `wp_blogs` metadata) for per-site flags that
  must not live in options.
- Users not assigned to any site after cleanup: `wp user list --fields=ID,user_email`
  cross-referenced with `wp site list --field=url` membership, or SQL against
  `wp_usermeta` capabilities keys.

## Watch-outs

- Site ID vs blog ID are the same thing; URLs differ. Confirm you are operating on the
  ID you think (`--url=` and `--blog=` both select context).
- Moving a site between networks or merging networks is not a core operation — treat as
  a migration project (export/import or scripted table moves), not a setting.
- `wp site delete` does not delete `uploads/sites/<id>/` — clean up explicitly if required.
