# Repository Guidelines

## Project Overview

Organizr is a PHP web portal for collecting self-hosted services into one dashboard with tabs, user/group access control, homepage widgets, SSO/auth helpers, and plugin/theme support. The frontend is a PHP-rendered Bootstrap/jQuery single-page interface; the backend exposes legacy handlers plus the current Slim-based `/api/v2` API.

## Architecture & Data Flow

- `index.php` is the main browser entry point. It includes `api/functions.php`, constructs `new Organizr(true)`, then renders HTML/CSS/JS and plugin/theme assets.
- `api/functions.php` is the global bootstrap. It sets UTC, requires Composer autoloading, then loads traits/functions/classes/pages and plugin/custom plugin files with `glob()` and recursive directory iterators.
- `api/classes/organizr.class.php` is the central application class. It composes most behavior through traits from `api/functions/*.php` and homepage traits from `api/homepage/*.php`, initializes config, paths, logging, database access, current user, upgrade checks, and auth state.
- `api/v2/index.php` creates the Slim 4 app, attaches an `Organizr` instance to each request, registers routes from `api/v2/routes/*.php`, `data/routes/*.php`, and plugin route files, then emits JSON responses.
- API responses generally use `$GLOBALS['api']['response']` with `result`, `message`, and `data`, plus `$GLOBALS['responseCode']`. Route handlers write `jsonE($GLOBALS['api'])` and return `Content-Type: application/json`.
- API authorization commonly goes through `$Organizr->checkRoute($request)` and `$Organizr->qualifyRequest(<group>, true)`. Mutating endpoints often read request data with `$Organizr->apiData($request)`.
- Frontend API calls use `organizrAPI2()` from `js/functions.js`; it sends `Token` and `formKey` headers, aborts duplicate in-flight requests by path, and uses `/api/v2/...` endpoints.
- Pages are registered by appending to `$GLOBALS['organizrPages']` and defining `get_page_<name>($Organizr = null)` in `api/pages/*.php`.

## Key Directories

- `api/` - PHP backend, Composer dependencies, config defaults, page renderers, core traits/classes, and API v2 routes.
- `api/classes/` - Core classes; `organizr.class.php` is the main integration point.
- `api/functions/` - Trait-based feature modules such as auth, config, homepage, plugins, logging, updates, and SSO/OIDC.
- `api/homepage/` - Homepage widget trait implementations for integrated services.
- `api/pages/` - PHP functions returning page HTML for homepage, settings, login, wizard, and related views.
- `api/v2/routes/` - Slim route definitions. Add current API behavior here rather than in legacy handlers.
- `api/plugins/` and `plugins/` - Plugin code/assets and third-party frontend dependencies. Treat `plugins/bower_components/` as vendored.
- `js/` - Application JavaScript plus vendored browser libraries. Core application helpers are in `js/functions.js`; much UI behavior is in `js/custom.js`.
- `css/`, `less/` - Theme and application styles. Runtime pages load minified CSS such as `css/dark.min.css` and `css/organizr.min.css`.
- `docs/` - API documentation assets, including `docs/api.json` OpenAPI 3.0 output.
- `scripts/` - Manual update scripts for Linux and Windows.
- `bootstrap/` - Vendored Bootstrap 3.3.6 source/package files, not the primary application package.

## Development Commands

Run commands from the directory shown.

```bash
# Backend dependencies
cd api
composer install

# Bootstrap vendor package tasks only, if touching bootstrap/ assets
cd bootstrap
npm install
npm test        # runs grunt test

# Manual update scripts
./scripts/linux-update.sh [v2-master|v2-develop]
scripts/windows-update.bat [-m|-d]
```

Notes:

- There is no observed root `package.json`, root test runner, or Composer script block.
- `api/composer.json` pins Composer's platform PHP to `7.4.0`; `Organizr::$minimumPHP` is also `7.4`. Prefer PHP 7.4+ even though older README text mentions PHP 7.2+.
- Docker deployment is documented in `README.md`; the app is commonly run via the `organizr/organizr` image with `/config` volume mapping.

## Code Conventions & Common Patterns

- PHP style is legacy procedural/object hybrid: tabs for indentation, global arrays for response/page registries, and traits mixed into the `Organizr` class.
- Prefer extending existing traits/classes over adding parallel bootstrap systems. If adding a feature method, place it in the relevant `api/functions/*-functions.php` trait or homepage trait and call it through `Organizr`.
- For new API endpoints, add a route file or route in `api/v2/routes/`, retrieve the request-scoped instance with `$request->getAttribute('Organizr')`, gate permissions with existing auth helpers, update `$GLOBALS['api']['response']`, and return JSON consistently.
- Keep OpenAPI annotations near API routes when adding or changing documented endpoints; `docs/api.json` is the generated/static API artifact consumers see.
- For UI pages, follow the `api/pages/*.php` pattern: register the page name globally and return the HTML string from `get_page_<name>()`. Check `hasDB()` where page rendering depends on initialized storage.
- Frontend code is jQuery-centric. Use delegated handlers (`$(document).on(...)`) for dynamic content and `organizrAPI2(method, 'api/v2/...', data, async)` for API calls.
- `organizrAPI2()` expects data objects for POST/PUT because it adds `formKey`; avoid passing `null` to mutating calls.
- Use existing UI helpers such as `message`, `messageSingle`, `ajaxloader`, `OrganizrApiError`, `organizrConsole`, and `loadSettingsPage2` instead of inventing new notification/loading flows.
- CSS/asset changes should account for minified files loaded by `index.php` (`*.min.css`, `*.min.js`). Do not edit vendored/minified dependencies unless the change is intentionally in that artifact.
- Plugin loading is filename-based (`plugin.php`, `page.php`, `cron.php`, `api.php`, `routes.php` in recognized plugin directories). Be conservative with file names because bootstrap discovery is broad.

## Important Files

- `index.php` - Main web UI entry point and asset loader.
- `api/functions.php` - Bootstrap/load order for backend, pages, plugins, and custom runtime extensions.
- `api/classes/organizr.class.php` - Core stateful application class and trait composition.
- `api/config/default.php` - Large flat default config array; user/runtime config is selected by application logic, so do not treat defaults as live instance state.
- `api/v2/index.php` - Slim app setup, API response defaults, auth bypass list, route inclusion, and 404 fallback.
- `api/v2/routes/*.php` - Current API endpoint definitions.
- `js/functions.js` - Core JavaScript utilities, including `organizrAPI2()`.
- `js/custom.js` - Main UI event handlers and settings/homepage interactions.
- `docs/api.json` - OpenAPI 3.0 API documentation artifact.
- `api/composer.json` - PHP dependency and platform constraints.
- `CONTRIBUTING.md` - Contribution rules: rebase from develop and open PRs to `v2-develop`, never master.

## Runtime/Tooling Preferences

- Backend runtime: PHP 7.4+ with Composer dependencies installed under `api/vendor/`.
- Backend framework: Slim 4 with PSR-7 (`slim/slim`, `slim/psr7`).
- Storage/config: default driver is SQLite (`api/config/default.php`), with runtime data paths under `data/` used by `Organizr`.
- Frontend runtime: server-rendered PHP plus Bootstrap 3.3.6, jQuery, and plugins from `plugins/bower_components/`.
- Package managers: Composer for `api/`; npm/Grunt only for the vendored `bootstrap/` package.
- Branch conventions: default runtime/update branch is `v2-master`; contributions target `v2-develop`.
- Avoid sweeping edits in `api/vendor/`, `plugins/bower_components/`, generated/minified assets, and runtime `data/` content unless explicitly required.

## Testing & QA

- No repository-owned test suite or local test command was observed in the inspected project files. Do not claim full coverage from vendored tests.
- `bootstrap/package.json` has `npm test` -> `grunt test`, but that applies to the Bootstrap vendor package, not the Organizr application as a whole.
- For PHP changes, at minimum run targeted syntax checks on changed PHP files, for example:

```bash
php -l api/v2/routes/groups.php
php -l api/functions/auth-functions.php
```

- For API behavior changes, exercise the affected `/api/v2/...` endpoint with the required `Token` and `formKey` context or through the UI path that calls it.
- For frontend changes, verify the affected page in a browser, including success and failure paths for `organizrAPI2()` calls and visible notification/loading behavior.
- For config/auth/session changes, test both authorized and unauthorized paths; many routes return a JSON envelope even when permission checks fail.
- For docs-only changes, verify Markdown renders and that commands/paths match observed files.
