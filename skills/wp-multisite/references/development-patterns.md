# Development patterns for multisite-aware code

## Blog switching

```php
switch_to_blog( $blog_id );
// ... get_option()/query/whatever runs against $blog_id ...
restore_current_blog();
```

Rules:

- Every `switch_to_blog()` must pair with exactly one `restore_current_blog()`,
  including on early returns and exceptions. Wrap in try/finally if control flow is complex.
- Switching re-initializes CPTs/taxonomies, options caches, and object cache keys — it
  is expensive. Batch per-site work: switch once per site, do everything, restore.
- Never switch and leave the switch active past the end of a hook callback.
- `is_main_site( $blog_id )` / `get_main_site_id()` for main-site-only behavior
  (e.g. storing a network-wide singleton on the main site).
- Some plugins cache state at load time; data may not follow a switch. Verify with a
  real secondary site, not just `switch_to_blog( 1 )`.

## Options: site (blog) vs network

- `get_option()` / `update_option()` — current blog's `wp_options` (or `wp_2_options` …).
- `get_site_option()` / `update_site_option()` — network-wide `wp_sitemeta`. Despite the
  name, in multisite "site option" means network option. Single-site installs fall back
  to `wp_options` transparently.
- Decision rule: config shared by every site (API keys, license, defaults) → network
  option, edited from a Network Admin settings page. Per-site overrides → blog option.
- Reading another blog's option without switching: `get_blog_option( $blog_id, $key )`.

## Network activation and activation hooks

- Network-activated plugins load once per request (not per site); `is_plugin_active_for_network( $myplugin )`
  to branch behavior.
- Activation hooks fire once at network activation, not per site. Per-site setup must iterate:

  ```php
  register_activation_hook( __FILE__, function ( $network_wide ) {
      if ( $network_wide ) {
          foreach ( get_sites( [ 'fields' => 'ids', 'deleted' => 0 ] ) as $blog_id ) {
              switch_to_blog( $blog_id );
              my_plugin_setup();
              restore_current_blog();
          }
      } else {
          my_plugin_setup();
      }
  } );
  ```

- Also handle sites created later via `wp_initialize_site` (see hooks below).
- Rewrite rules: flush only on the affected blogs; `flush_rewrite_rules()` affects the
  current blog only.

## Multisite hooks worth knowing

| Hook | Fires |
| --- | --- |
| `wp_initialize_site` / `wp_insert_site` | New site created (use `wp_initialize_site`, which receives a `WP_Site` and args) |
| `wp_delete_site` | Site deleted (before tables drop) |
| `wpmu_new_blog` | Legacy; prefer `wp_initialize_site` |
| `add_user_to_blog` / `remove_user_from_blog` | Site membership changes |
| `network_admin_menu` / `network_admin_notices` | Network admin UI |
| `wpmu_options` / `update_wpmu_options` | Network Settings page render/save |
| `upload_dir` filter | Multisite already rewrites uploads to `sites/<id>`; only adjust when adding mapped-domain URL handling |

## Users, roles, capabilities

- One `wp_users` table per network; membership and roles live in `wp_usermeta` under
  per-site keys: `wp_{$blog_id}_capabilities`, `wp_{$blog_id}_user_level`.
- Super admin bypasses capability checks network-wide: `is_super_admin( $user_id )`,
  `grant_super_admin()` / `revoke_super_admin()`. Do not treat super admin as a role.
- Capability checks are against the current site unless you pass context:
  `current_user_can_for_site( $blog_id, $cap )`, `user_can( $user_id, $cap )`.
- `is_user_member_of_blog( $user_id, $blog_id )` before showing site-scoped UI.

## Data model quick map

- Sites: `get_sites()`, `get_blog_details( $blog_id )`, `WP_Site` objects.
  (`wp_get_sites()` is deprecated.)
- Network: `get_network( $network_id )`, `get_main_site_id( $network_id )`,
  `get_network_option()`.
- Tables: `$wpdb->get_blog_prefix( $blog_id ) . 'posts'` for cross-site SQL; never
  assume `wp_2_` — main site tables have no numeric infix.
- Uploads: main site → `wp-content/uploads/`; other sites → `wp-content/uploads/sites/<id>/`.
  Legacy pre-3.5 installs use `wp-content/blogs.dir`.
- Object cache and `wp_cache_flush()` are network-wide effects — see `wp-performance`
  before flushing on large networks.

## REST and URL contexts

- REST endpoints serve the current site (`/wp-json/...` on each site's URL); network-wide
  data is not exposed by core REST. See `wp-rest-api` for building network endpoints.
- `home_url()` / `admin_url()` are site-scoped; `network_admin_url()` / `network_home_url()`
  for the main site/network admin — wrong one is the usual cause of misdirected links.
