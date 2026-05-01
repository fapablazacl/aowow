# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

AoWoW is a PHP 8.2+ web database tool for World of Warcraft 3.3.5 (build 12340), targeting AzerothCore / TrinityCore world databases. It is a server-side rewrite that drives a vintage Wowhead-style frontend (the JS/CSS/templates were imported from a ~2013 layout and are not original to this repo). Requires MySQL ≥ 5.7 / MariaDB ≥ 10.6 and PHP extensions `SimpleXML`, `gd`, `mysqli`, `mbstring`, `fileinfo`. The `Intl` extension must be **disabled** — it ships its own `Locale` class that collides with this project's.

## Common commands

There is no test suite, no linter config, and no build system. All "build"-like work is the CLI setup tool `php aowow`, which must be run from the repo root.

```bash
php aowow --help                   # list every available subcommand
php aowow --setup                  # initial setup wizard (run once after install)
php aowow --database               # (re)write config/config.php with DB credentials
php aowow --configure              # edit site config in DB (HOST_URL, STATIC_URL, etc.)
php aowow --account                # create/manage admin/user accounts
php aowow --sql                    # regenerate aowow_* SQL tables from world DB + DBC
php aowow --sql=items,quests       # regenerate a subset (see setup/tools/sqlgen/*.ss.php)
php aowow --dbc                    # (re)import DBC files from setup/mpqdata/<locale>/DBFilesClient/
php aowow --build                  # generate static assets (JS, CSS, images) into static/
php aowow --sync                   # sync world DB changes into aowow_* mirror tables
php aowow --update                 # apply pending SQL migrations from setup/updates/
php aowow --locales=enus,dede ...  # restrict any of the above to specific locales
php aowow -f ...                   # force overwrite of existing files
```

The character profiler runs out-of-band — `./prQueue` is a long-running CLI worker that drains `?_profiler_sync` and pulls character/guild/arena data from the realm `characters` DB. The web layer enqueues; `prQueue` processes. Don't expect synchronous profile fetches.

`bash setup/generate-db.sh` is the CI path used by `.github/workflows/generate-aowow-database.yml`: it stands up acore-docker, fetches DBCs from the `wowgaming/client-data` release, runs `php aowow --sql`, and dumps `aowow_db.sql.zip`. It writes to `config/config.php` with hardcoded docker creds — only run inside the workflow's container, never against a real DB.

## Architecture

### Entry points and request flow
- `index.php` — single web entry. The first segment of the query string (`?<pageCall>=<pageParam>`) is the route. The router first tries `Ajax<PageCall>` (in `includes/ajaxHandler/`); if that throws, it falls back to `<PageCall>Page` (in `pages/`). Static "more" pages route to `MorePage`, dynamic listings (latest-comments, random, etc.) to `UtilityPage`. Unknown routes → `GenericPage::error()`. URLs with `?power` always emit a JS stub instead of HTML — Wowhead-style tooltip embeds rely on this.
- `aowow` — CLI entry; loads `includes/kernel.php` then `setup/setup.php`.
- `prQueue` — CLI entry for the profiler queue worker (single-instance lock via `Profiler::queueLock()`).
- `includes/kernel.php` is bootstrap for **all three**: it loads config, registers two `spl_autoload_register` chains (one for SmartAI/Conditions components, one that maps `*List`/`*ListFilter` → `includes/types/*.class.php`, `Ajax*` → `includes/ajaxHandler/*.class.php`, `*Page` → `pages/*.php`), wires error/exception/shutdown handlers (errors are persisted to `?_errors` in the AoWoW DB), and starts the session.

### Databases
Four logical DBs, all accessed through the `DB::` wrapper in `includes/database.class.php` (a thin wrapper over `includes/libs/DbSimple/`). Connection identifiers are the `DB_*` constants from `includes/defines.php`:
- `DB_AOWOW` — this project's own DB. Tables are prefixed (typically `aowow_`); SQL uses `?_table` and the DbSimple layer rewrites `?_` to the configured prefix. **Always** use `?_` in queries, never hardcode the prefix.
- `DB_WORLD` — the AzerothCore/TrinityCore world DB (read-only in spirit; ideally only `read` grants).
- `DB_AUTH` — the auth DB (used by the profiler; optional).
- `DB_CHARACTERS` + realm id — per-realm characters DB. Multiple realms are loaded as `DB_CHARACTERS . $realmId`. Access via `DB::Characters($realm)`.

`?_` placeholders, plus `?d` (int), `?s` (string), `?a` (array), etc. come from DbSimple — see existing queries before inventing your own substitution patterns.

### Type system
`includes/types/*.class.php` is the heart of the data model. Each file defines a *List* class (e.g. `ItemList`, `SpellList`, `QuestList`) extending `BaseType` from `includes/basetype.class.php`. A List both queries the DB and renders/exposes its rows; the matching `*ListFilter` class (auto-resolved by the autoloader stripping `Filter`) handles search/filter form parsing. Detail pages and Ajax handlers consume Lists rather than writing raw SQL. New entity types belong here.

### Pages and templates
- `pages/<route>.php` defines `<Route>Page extends GenericPage`. The `display()` method assembles tab data and template variables, then renders `template/pages/<tpl>.tpl.php`.
- `template/` is plain PHP templates (no engine): `pages/` for top-level pages, `bricks/` for shared partials, `listviews/` for the JS-driven list tables, `localized/` for per-locale string includes.
- Compiled/cached output lands in `cache/template/`.

### Ajax & JSON-P
`includes/ajaxHandler.class.php` is the base; subclasses live in `includes/ajaxHandler/`. The router *prefers* the Ajax handler — a route is treated as a page only if no Ajax handler exists or it declines the request. Many "page" URLs double as data endpoints (`?data=`, `?cookie=`, `?filter=`, `?power`) — keep both paths in mind when changing routing.

### Setup tooling (`setup/`)
- `setup/tools/clisetup/*.us.php` — top-level CLI utilities (`UtilityScript`). Each declares `COMMAND`, `DESCRIPTION`, `optGroup`, and is auto-discovered by `CLISetup::loadScripts()`. `OPT_GRP_SETUP` commands appear under "AoWoW Setup", `OPT_GRP_UTIL` under "Utility Functions".
- `setup/tools/sqlgen/*.ss.php` — `SetupScript` subclasses, registered against an invoker (almost always `sql`). These are the per-table generators that build `aowow_*` from world DB + DBC data. Adding a new generated table = new `*.ss.php` here.
- `setup/tools/dbc/` — DBC binary parsing.
- `setup/tools/filegen/` — static asset generation (JS/CSS templating, images).
- `setup/updates/<unixtime>_NN.sql` — schema migrations applied by `php aowow --update`. Always create a new file; never edit a shipped migration. The numeric prefix is the unix timestamp at authoring time.

### Components
`includes/components/SmartAI/` and `includes/components/Conditions/` parse and render TrinityCore's `smart_scripts` and `conditions` tables — these are loaded lazily by their own autoloader branch, not the generic one.

### Locale and configuration
- Six locales are supported: `enus`, `frfr`, `dede`, `eses`, `ruru`, `zhcn`. The `--locales=` flag scopes setup commands; in the web layer `Lang::load()` is driven by `User::$preferedLoc` or `?locale=`.
- `Cfg::load()` pulls runtime config from the `?_config` table after the DB is up. `config/config.php` only holds DB credentials. Use `Cfg::get('KEY')` to read site config (HOST_URL, STATIC_URL, DEBUG, PROFILER_*, etc.) — don't hardcode.
- DEBUG ≥ `CLI::LOG_INFO` enables per-request DB query logging for users in `U_GROUP_DEV | U_GROUP_ADMIN`.

### Caching
Two backends, selected by the `CACHE_MODE_*` flags: file cache under `cache/` (default) and Memcached. `CACHE_TYPE_*` distinguishes pages, tooltips, search, and item XML caches. Invalidation is mostly time-based; setup commands that mutate `aowow_*` data are expected to clear the relevant cache themselves.

## Conventions worth knowing

- Every PHP file checks `if (!defined('AOWOW_REVISION')) die('illegal access');` near the top — keep this guard when adding files in `includes/` or `setup/`.
- `AOWOW_REVISION` in `includes/kernel.php` is bumped when schema or generated-data shape changes; migrations in `setup/updates/` are paired with revisions.
- Don't add `mkdir`/permission logic ad-hoc — `Util::writeDir()` + the `--setup` script handle the seven required writable dirs (`cache/`, `config/`, `static/download/`, `static/widgets/`, `static/js/`, `static/uploads/`, `static/images/wow/`, `datasets/`).
- Wowhead deep links use the `WOWHEAD_LINK` format string — don't hand-construct external URLs.
- The world DB is treated as upstream truth. AoWoW does **not** write to `DB_WORLD`. The one exception is the install step that imports `setup/spell_learn_spell.sql` once.
