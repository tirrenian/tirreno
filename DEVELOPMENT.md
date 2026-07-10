# tirreno developers documentation

This document describes how to integrate [tirreno](https://www.tirreno.com) into your product and how to extend it with your own pages, rules, and presets.

It applies to **tirreno v0.10.0**. The version of a running instance is defined in [`app/Utils/VersionControl.php`](app/Utils/VersionControl.php) and can be read at runtime:

```php
tirreno('utils')->versionControl->fullVersionString();   // "v0.10.0"
```

Database schema migrations are keyed by the same version strings in [`app/Updates/`](app/Updates/) (e.g. `Update009.php` is `v0.10.0`) and run automatically during update.

## Contents

* [How tirreno is put together](#how-tirreno-is-put-together)
* [Event ingestion API (sensor)](#event-ingestion-api-sensor)
* [Blacklist API](#blacklist-api)
* [The tirreno() API](#the-tirreno-api) *(new in v0.10.0)*
* [Query builder: tirreno('queries')](#query-builder-tirrenoqueries) *(work in progress)*
* [Custom pages: assets/pages/](#custom-pages-assetspages) *(work in progress)*
* [Explained example: risk-users](#explained-example-risk-users)
* [Explained example: llm-bots](#explained-example-llm-bots)
* [Custom rules: assets/rules/custom/](#custom-rules-assetsrulescustom)
* [Rule context](#rule-context)
* [Rule presets](#rule-presets)
* [Lists](#lists)
* [UI constants](#ui-constants)
* [Configuration variables](#configuration-variables)
* [RBAC and system operators](#rbac-and-system-operators) *(new in v0.10.0)*
* [Coding standards and tooling](#coding-standards-and-tooling)
* [Links](#links)

## How tirreno is put together

tirreno is a hand-written, few-dependency PHP/PostgreSQL application. There are two entry points:

| Path | Purpose |
|------|---------|
| `index.php` | The dashboard (console) application. Boots the tirreno router, namespace `\Tirreno\`, code in `app/`. |
| `sensor/index.php` | The event ingestion endpoint. A standalone, minimal-overhead application, namespace `\Sensor\`, code in `sensor/src/`. |

Directory overview:

* `app/` — dashboard application: `Controllers/`, `Models/`, `Entities/`, `Utils/`, `Views/`, `Updates/` (migrations), and `Core/` (the service container behind the `tirreno()` API).
* `sensor/` — ingestion API: request validation, device detection, blacklist matching, enrichment, logbook.
* `assets/` — **your** extension points: custom pages, custom rules, rule presets, keyword lists, UI constants, logs. Everything under `assets/` is designed to survive updates.
* `config/` — `config.ini` (defaults), `routes.ini`, `apiEndpoints.ini`, `crons.ini`, and `config/local/` for machine-local overrides.
* `ui/` — dashboard templates (F3 template engine), JavaScript, CSS.
* `install/` — web installer (delete after installation).
* `libs/` — vendored dependencies (Fat-Free core, Ruler, PHPMailer, device-detector), used when no `vendor/` autoloader is present, so Composer is optional at runtime.
* `tests/` — PHPUnit test suite.

Requirements: PHP 8.0–8.3 with `PDO_PGSQL` and `cURL`, PostgreSQL 12+. Background processing (rule checks, counters, enrichment) runs through cron:

```
*/10 * * * * /usr/bin/php /absolute/path/to/tirreno/index.php /cron
```

## Event ingestion API (sensor)

Events are sent as **form-encoded `POST` requests** to `/sensor/` with the API key (from the dashboard *API* page) in the `Api-Key` header. One request describes one event.

```bash
curl -X POST 'https://tirreno.example.com/sensor/' \
    -H 'Api-Key: 0000000000000000000000000000000000000000' \
    --data-urlencode 'eventTime=2026-07-10 12:34:56.789' \
    --data-urlencode 'ipAddress=203.0.113.10' \
    --data-urlencode 'url=https://app.example.com/login' \
    --data-urlencode 'eventType=account_login' \
    --data-urlencode 'userName=jdoe' \
    --data-urlencode 'emailAddress=jdoe@example.com' \
    --data-urlencode 'userAgent=Mozilla/5.0 (X11; Linux x86_64) ...' \
    --data-urlencode 'browserLanguage=en-US,en;q=0.9' \
    --data-urlencode 'httpMethod=post' \
    --data-urlencode 'httpCode=200'
```

Rather than hand-rolling requests, consider the official SDKs: [PHP](https://github.com/tirrenotechnologies/tirreno-php-tracker), [Python](https://github.com/tirrenotechnologies/tirreno-python-tracker), [NodeJS](https://github.com/tirrenotechnologies/tirreno-nodejs-tracker), [WordPress](https://github.com/tirrenotechnologies/tirreno-wordpress-tracker).

### Fields

Required:

| Field | Description |
|-------|-------------|
| `ipAddress` | IP address of the acting user (IPv4 or IPv6). |
| `url` | Full URL of the request being described. |
| `eventTime` | Event timestamp, `Y-m-d H:i:s.v` (milliseconds); `Y-m-d H:i:s` and microsecond precision are also accepted. An unparsable value is replaced with the current UTC time, and the correction is recorded in the logbook. |

Optional:

| Field | Description |
|-------|-------------|
| `userName` | Stable account identifier (user ID, login). If omitted and `emailAddress` is set, an identifier is derived from the email; if both are missing, the event is filed under `N/A`. |
| `emailAddress` | Account email. |
| `phoneNumber` | Account phone number. |
| `firstName`, `lastName`, `fullName` | Account display names. |
| `userCreated` | Account registration timestamp (enables account-age rules). |
| `eventType` | One of the types below; defaults to a plain page visit. |
| `pageTitle` | Human-readable title of the page/action. |
| `userAgent` | Raw `User-Agent` header (enables device, bot, and AI-bot detection). |
| `browserLanguage` | Raw `Accept-Language` header. |
| `httpMethod` | `get`, `post`, `head`, `put`, `patch`, `delete`, ... |
| `httpCode` | HTTP status code your application returned (invalid values become `0`). |
| `httpReferer` | Raw `Referer` header. |
| `payload` | Type-specific payload, accepted for `page_search` and `account_email_change`. |
| `fieldHistory` | Old/new values for `field_edit` events (feeds the field audit trail). |
| `blacklisting` | See [Blacklist check](#inline-blacklist-check-blacklisting). |

### Event types

`page_view`, `page_edit`, `page_delete`, `page_search`, `account_login`, `account_logout`, `account_login_fail`, `account_registration`, `account_email_change`, `account_password_change`, `account_edit`, `page_error`, `field_edit`.

Using precise types pays off: many rules key on them (failed logins, password changes, edit bursts), `page_error` powers API-abuse detection, and `field_edit` populates the field audit trail.

### Responses and limits

* Success: `200` with an empty body — the event is stored, and validation corrections (if any) are recorded per field and shown in the dashboard logbook (`/logbook`).
* `401` — missing or unknown `Api-Key` header.
* `400` — validation error: a required field is missing or empty.
* `429` — API overload protection triggered. The sensor applies a leaky-bucket limiter configured by `LEAKY_BUCKET_RPS` (default 10) and `LEAKY_BUCKET_WINDOW` (default 20 seconds).
* `503` — database unavailable.

Every request, including rejected ones, is written to the logbook, so integration mistakes are visible in the dashboard.

### Inline blacklist check (`blacklisting`)

Sending `blacklisting=true` with an event makes the sensor answer **synchronously** with the blacklist status of the event's identifiers, so your application can block a request in-line:

```json
{"value": "203.0.113.10", "blacklisted": false}
```

`blacklisted` is `true` when the event's IP, email, or phone was earlier blacklisted (manually or via thresholds) under your API key.

### CLI mode

The sensor also runs from the command line, which is handy for backfilling and air-gapped pipelines:

```bash
php sensor/index.php --apiKey=YOUR_KEY \
    --ipAddress='203.0.113.10' \
    --url='https://app.example.com/login' \
    --eventTime='2026-07-10 12:34:56.000'
```

Every field from the table above is available as a `--field=value` long option.

## Blacklist API

An authenticated JSON endpoint of the dashboard application (defined in `config/apiEndpoints.ini`):

```
POST /api/v1/blacklist/search
Api-Key: <your-api-key>

{"value": "userNameToMatch"}
```

It answers `{"value": "...", "blacklisted": true|false}` — whether an account name is blacklisted under your API key — e.g. for a login or signup flow. Errors come back as `{"code": ..., "message": ...}`.

## The tirreno() API

**New in v0.10.0.** The dashboard application exposes all of its building blocks through a single global function, defined in [`app/tirreno.php`](app/tirreno.php):

```php
function tirreno(string $name): object
```

`tirreno($name)` resolves a named service from `\Tirreno\Core\Container` ([`app/Core/Container.php`](app/Core/Container.php)). This is the intended surface for custom pages, local extensions, and coding agents: instead of instantiating internal classes directly, you ask the container for a capability by name. Unknown names throw an exception.

Services are request-scoped singletons, with three exceptions — `users`, `ips`, `resources` — that return a **fresh instance on every call** (they are stateful query builders). `.phpstorm.meta.php` in the repository root teaches PhpStorm-family IDEs to autocomplete the argument and infer return types (also new in v0.10.0).

### Request lifecycle

| Interface | What it gives you |
|-----------|-------------------|
| `tirreno('request')` | Typed access to the current HTTP request: verb checks (`isGet()`, `isPost()`, `isPut()`, `isDelete()`, `isAjax()`, `isCli()`, `isHttps()`), URL parts (`getUri()`, `getPath()`, `getQuery()`, `getFragment()`), headers (`getHeader('X-Foo')`, `getContentType()`), route parameters (`getUrlParam()`, `getIntUrlParam()`), and the merged GET+POST+JSON payload (`getAllPayload()`, `getStringRequestParam()`, `getIntRequestParam()`, `getArrayRequestParam()`, `getDictionaryRequestParam()`). `validateCsrf()` checks the session CSRF token. |
| `tirreno('response')` | Redirects and errors, plus declarative guards: `redirect()`, `error()`, `redirectNotLoggedIn()`, `redirectLoggedIn()`, `errorNotLoggedIn()`, and role gates `redirectImproperRole($allowedRoles, $blockedRoles)` / `errorImproperRole(...)`. |
| `tirreno('session')` | The operator session: `getCurrentOperator()` (an `\Tirreno\Entities\Operator`), `getCurrentKey()` (the active API key — a `\Tirreno\Entities\ApiKey`), and session storage `get()` / `set()` / `remove()` / `clear()`. |
| `tirreno('storage')` | Low-level key–value access to the framework hive (config values, route state): `get()`, `set()`, `remove()`. Prefer the higher-level services when one exists. |
| `tirreno('page')` | Metadata of the page being rendered: `setTitle()`, `setName()`, `setTemplate()`, `setJavascript()`, `setAuthenticated()`, `setAllowedRoles()` / `setBlockedRoles()` (arrays, or per-HTTP-verb maps), and template parameters via `setParams()` / `addParams()` — everything passed here is available as `{{ @KEY }}` in the view. |
| `tirreno('router')` | The tirreno router (introduced in v0.10.0 in place of the Fat-Free router, keeping an F3-compatible interface): `config()`, `route()`, `run()`, `reroute()`, `error()`. |

### Operator and infrastructure

| Interface | What it gives you |
|-----------|-------------------|
| `tirreno('sysop')` | The current operator as a fully loaded `Operator` entity, with roles and permissions resolved; falls back to the guest system operator when nobody is logged in. |
| `tirreno('db')` | `initConnection()` — establishes the PostgreSQL connection (done for you inside pages). |
| `tirreno('log')` | Application logging with `printf`-style formatting: `debug()` (gated by `DEBUG`), `info()`, `error()`, raw `log()`, `logSql()`, and `logbookRequest()` for logbook entries. With `LOG_TO_STDERR = true` messages are duplicated to stderr — useful in containers. |
| `tirreno('helpers')` | Small view helpers, e.g. `formatTitle()`. |

### Data access

| Interface | What it gives you |
|-----------|-------------------|
| `tirreno('user')`, `tirreno('ip')`, `tirreno('resource')`, `tirreno('rule')` | Single-entity lookups scoped to the current operator's API key: `getById($id)` returns `\Tirreno\Entities\User` / `Ip` / `Resource` / `Rule`, or `null`. An explicit key can be passed as the second argument. |
| `tirreno('users')`, `tirreno('ips')`, `tirreno('resources')` | Pre-scoped query builders over accounts, IPs, and URLs — the same fluent interface as `tirreno('queries')` below, already bound to the current API key. A fresh builder is returned on every call. |
| `tirreno('rules')` | Rule sets for the current key: `getAll()`, `getByUserId($userId)`. |
| `tirreno('queries')` | Factory of query builders for all event tables — see [Query builder](#query-builder-tirrenoqueries). |

### Aggregators

The remaining interfaces are lazy aggregators: each exposes a family of internal classes as properties, instantiated on first access. They exist so that extension code can reach any part of the application through one stable entry point — and can even substitute members (`tirreno('models')->override('rules', MyRules::class)`) without touching core files.

| Interface | Members (examples) | Backing namespace |
|-----------|--------------------|-------------------|
| `tirreno('controllers')` | `dashboard`, `users`, `rules`, `settings`, `logbook`, ... — data/service controllers behind each dashboard page | `\Tirreno\Controllers\Services\*` |
| `tirreno('pages')` | `login`, `signup`, `dashboard`, `api`, ... — page controllers | `\Tirreno\Controllers\Pages\*` |
| `tirreno('models')` | `operator`, `rules`, `event`, `apiKeys`, `blacklistItems`, ... — database models | `\Tirreno\Models\*` |
| `tirreno('grids')` | `users`, `events`, `logbook`, ... — server-side DataTables grid models | `\Tirreno\Models\Grid\*\Grid` |
| `tirreno('charts')` | `events`, `users`, `blacklist`, ... — chart data models | `\Tirreno\Models\Chart\*` |
| `tirreno('entities')` | `user`, `users`, `ip`, `rule`, `logbook`, ... — entity classes (static-proxied), e.g. `tirreno('entities')->users->buildFromArray(...)` | `\Tirreno\Entities\*` |
| `tirreno('utils')` | `constants`, `conversion`, `validators`, `dateRange`, `timezones`, `network`, `variables` (typed config getters), `versionControl`, `mailer`, `httpClient`, ... — utility classes exposed through a static proxy, so class constants read as properties: `tirreno('utils')->constants->RULE_WEIGHT_MAP` | `\Tirreno\Utils\*` |
| `tirreno('assets')` | Accessors for the `assets/` extension points: `emailList`, `urlList`, `userAgentList`, `asnList`, `aiBotList`, `fileExtensionsList`, `uiConstants`, `serverConstants`, `context`, `pages`, `rules`, `rulesPresets` | `\Tirreno\Utils\Assets\*` |

A short, practical taste of the API:

```php
$operator = tirreno('session')->getCurrentOperator();
$key      = tirreno('session')->getCurrentKey();

tirreno('log')->info('operator %d opened %s', $operator->id, tirreno('request')->getPath());

$entity  = tirreno('user')->getById(42);                              // ?\Tirreno\Entities\User
$rules   = tirreno('rules')->getByUserId(42);                         // rules matched by account 42
$version = tirreno('utils')->versionControl->fullVersionString();     // "v0.10.0"
$aiBots  = tirreno('assets')->aiBotList->getList();                   // assets/lists/ai-bot.php
```

## Query builder: tirreno('queries')

**Work in progress in v0.10.0** — the interface is functional but still evolving; expect additions in upcoming releases.

`tirreno('queries')` builds parameterized SQL over the event store without hand-written SQL. Every builder is automatically **scoped to the current operator's API key**, values are bound as prepared-statement parameters, and column names are validated against a per-builder whitelist — unknown columns are silently ignored rather than interpolated.

Available builders and their primary tables:

| Builder | Table | Notable columns (aliases) |
|---------|-------|---------------------------|
| `->users` | `event_account` | `user_id`, `user_userid`, `user_score`, `user_score_details`, `user_fraud`, `user_reviewed`, `user_lastseen`, `user_total_ip`, `email_email`, `phone_phone_number`, ... |
| `->ips` | `event_ip` | `ip_ip`, `ip_cidr`, `ip_data_center`, `ip_tor`, `ip_vpn`, `ip_relay`, `ip_blocklist`, `isp_asn`, `isp_name`, `country_iso`, ... |
| `->events` | `event` | per-event records |
| `->urls` | `event_url` | visited resources |
| `->countries`, `->devices`, `->sessions`, `->payloads`, `->queries`, `->referers` | respective `event_*` tables | |

Chainable methods:

```php
where($column, $operator, $value)       // AND condition (alias of andWhere)
andWhere(...) / orWhere(...)            // explicit chaining logic
whereColumn($column, $operator, $col2)  // compare two columns
join($table)                            // LEFT JOIN another event table (e.g. 'event_ip')
groupBy($column)
orderBy($column, 'ASC'|'DESC')
limit($n) / offset($n)
```

Terminal methods:

```php
get()               // run SELECT; returns an entity collection — ->data is an array of entity objects
count($selector)    // SELECT COUNT(*), optionally applying a selector string
find($selector)     // parse a selector string, then get()
```

Supported operators: `=`, `<`, `>`, `<=`, `>=`, `<>`, `!=`, `LIKE`, `ILIKE`, `NOT LIKE`, `NOT ILIKE`, `IN`, `NOT IN`, `BETWEEN`, `NOT BETWEEN`, regex matches `~`, `*~`, `!~`, `!*~`, and unary forms (`IS NULL`, `IS NOT NULL`, `IS TRUE`, ...). String values `'NULL'`, `'TRUE'`, `'FALSE'` with `=` / `!=` are converted to the corresponding `IS ...` forms, and array values expand to placeholder lists for `IN` / `BETWEEN`.

Fluent example — accounts with a trust score of 20 or higher, newest activity first:

```php
$entities = tirreno('queries')->users
    ->where('user_score', '>=', 20)
    ->orderBy('user_lastseen', 'DESC')
    ->limit(50)
    ->get()
    ->data;                            // \Tirreno\Entities\User[]
```

`find()` accepts a compact selector string — comma-separated `column<op>value` conditions plus `sort=`, `limit=`, `start=` controls; `|` between values means OR:

```php
// Tor or VPN addresses seen most recently
$ips = tirreno('queries')->ips->find('ip_tor=TRUE|ip_vpn=TRUE, sort=-ip_lastseen, limit=20')->data;
```

The entity objects returned by `get()->data` expose typed properties. For `users`: `id`, `userid`, `score`, `scoreDetails` (decoded array), `fraud`, `reviewed`, `lastseen`, `created`, totals (`totalVisit`, `totalIp`, `totalCountry`, ...), and nested `email` / `phone` entities.

## Custom pages: assets/pages/

**Work in progress in v0.10.0.** tirreno ingests universal primitives (entities/users, IPs, devices, sessions, events) and exposes them through composable machinery — the rule engine, the query builder, and this page extension system — so you can build views for exactly the risk model your product needs. Custom pages live in `assets/` and survive updates.

### File-based pages

The simplest form. Two files, no classes, no route registration:

```
assets/pages/<name>.php            # controller: runs inside FileBasedPage::index()
assets/pages/views/<name>.html     # view: an F3 template fragment
```

The page is automatically served at `/<name>` (filename characters: `A–Z a–z 0–9 _ -`) and appears in the left navigation menu, titled by the `$page->setTitle('...')` call found in the file. A `.example.php` suffix keeps the file out of the menu and off the router — the two shipped examples are enabled by renaming, e.g. `llm-bots.example.php` → `llm-bots.php` (and its view to `llm-bots.html`).

Inside the controller file these variables are pre-bound (see [`app/Core/FileBasedPage.php`](app/Core/FileBasedPage.php)); everything else is a `tirreno('...')` call away:

```php
$session   $request   $response   $sysop   $utils
$page      $helpers   $db         $log     $user    $ip
```

The database connection, session, and CSRF token are established before your file runs. The token is exposed to templates as `{{ @CSRF }}`; if your page accepts form submissions, validate them with `$request->validateCsrf()` (class-based pages do this automatically for non-GET requests).

### Explained example: risk-users

[`assets/pages/risk-users.example.php`](assets/pages/risk-users.example.php) + [`views/risk-users.example.html`](assets/pages/views/risk-users.example.html) — a "top risk accounts" page in ~40 lines.

The controller, step by step:

```php
$page->setTitle('Risk users');
```
Sets the browser/page title, which doubles as the navigation menu label.

```php
$response->redirectNotLoggedIn('/login');
$response->redirectImproperRole(['operator'], [], '/login');
```
The security gate. Anonymous visitors and operators without the `operator` role (e.g. `guest`) are redirected — see [RBAC](#rbac-and-system-operators). Custom pages are responsible for their own gating, which two lines cover.

```php
$entities = tirreno('queries')->users->where('user_score', '>=', 20)->limit(50)->get()->data;
```
The new query builder in action: accounts with trust score ≥ 20, automatically restricted to the operator's API key, returned as `\Tirreno\Entities\User` objects.

```php
$rows = array_map(static function (\Tirreno\Entities\User $user): array {
    return [
        'id'        => $user->id,
        'userid'    => $user->userid,
        'email'     => $user->email->email,
        'lastseen'  => $user->lastseen,
    ];
}, $entities);
```
Entities are flattened to scalar rows before templating — a deliberate **XSS defence**. In F3 templates, array access (`{{ @entity['userid'] }}`) is HTML-escaped automatically, while object access (`{{ @entity->userid }}`) is *not* — and `userid`/`email` are attacker-supplied, ingested data. Always pass arrays.

```php
$page->addParams([
    'ENTITIES'      => $rows,
    'AB_TITLE'      => 'Top risk users',
    // ... column labels ...
]);
```
Everything a template shows arrives via page params. File-based pages don't load a page dictionary, so undefined `@keys` would render as empty strings — labels are therefore passed as params too.

The view renders a table with F3 constructs — `<repeat>` for iteration, `<check>` for the empty state — and links each row to the entity's built-in detail page:

```html
<repeat group="{{ @ENTITIES }}" value="{{ @entity }}">
    <tr>
        <td><a href="{{ @BASE }}/id/{{ @entity['id'] }}">{{ @entity['userid'] }}</a></td>
        ...
</repeat>
```

### Explained example: llm-bots

[`assets/pages/llm-bots.example.php`](assets/pages/llm-bots.example.php) + [`views/llm-bots.example.html`](assets/pages/views/llm-bots.example.html) — lists accounts whose traffic came from LLM/AI bots. Same skeleton as `risk-users`; the interesting part is the query:

```php
$entities = tirreno('queries')->users
    ->where('user_score_details', 'ILIKE', '%"D13"%')
    ->orderBy('user_lastseen', 'DESC')
    ->limit(50)
    ->get()->data;
```

It leans on the rule engine rather than re-deriving bot logic. Core rule **D13 — "Device is AI bot"** ([`assets/rules/core/D13.php`](assets/rules/core/D13.php)) fires when an account's user agent resolves to a known AI/LLM bot from the [`assets/lists/ai-bot.php`](assets/lists/ai-bot.php) list (ChatGPT-User, Claude-User, PerplexityBot, ...; the separate AI-bot list is new in v0.10.0). Every matched rule is stored on the account in the JSONB column `score_details` as `[{"uid":"D13","score":N}, ...]` — so "accounts flagged by rule X" is a simple `ILIKE '%"X"%'` filter on `user_score_details`. The same pattern works for any rule uid, including your custom ones.

### Class-based pages

When a page outgrows a single file — its own JavaScript, CSS, dictionary, multiple verbs — promote it to a class:

```
assets/pages/<pageName>/<PageName>.php      # class \Tirreno\Pages\<PageName> extends \Tirreno\Core\Page
assets/pages/<pageName>/dictionary.php      # optional: template dictionary entries
assets/pages/<pageName>/ui/templates/       # own templates
assets/pages/<pageName>/ui/js|css|images/   # own static assets, auto-routed under /ui/
```

A class page implements `init()` (configure `tirreno('page')`: template, title, roles) and `getRoute()` (e.g. `'/my-page'`); the base class registers the route for all HTTP verbs, runs the auth/CSRF pipeline, and renders JSON for AJAX requests and the frontend template otherwise. All discovered pages are instantiated at boot (`tirreno('assets')->pages->getAllPagesObjects()` in `index.php`).

## Custom rules: assets/rules/custom/

Rules score account behaviour. Each rule is one PHP class in one file; core rules ship in `assets/rules/core/`, and your rules go to `assets/rules/custom/`. **A custom rule with the same uid as a core rule overrides it.**

The uid is the class *and* file name. Its first letter groups the rule in the UI: `A` account takeover, `B` behaviour, `C` country, `D` device, `E` email, `I` IP, `P` phone, `R` reuse, `X` extra (recommended for custom rules). Weights are assigned per rule in the dashboard (or by preset): `positive` (−20), `none` (0), `medium` (10), `high` (20), `extreme` (70).

The shipped example [`assets/rules/custom/X03.example.php`](assets/rules/custom/X03.example.php) (rename to `X03.php` to activate):

```php
namespace Tirreno\Rules\Custom;

class X03 extends \Tirreno\Assets\Rule {
    public const NAME = '1xx user name';
    public const DESCRIPTION = 'Username starts with digit 1.';
    public const ATTRIBUTES = [];

    protected function defineCondition(): \Ruler\Operator\LogicalOperator {
        return $this->rb->logicalAnd(
            $this->rb['extra_one_digit_userid']->equalTo(true),
        );
    }
}
```

* `NAME` / `DESCRIPTION` are what operators see on `/rules`.
* `defineCondition()` declares the trigger with the [Ruler](https://github.com/bobthecow/Ruler) rule builder (`$this->rb`): combine context variables with `logicalAnd` / `logicalOr` / `logicalNot` and comparisons (`equalTo`, `greaterThan`, `lessThan`, ...).
* Variables like `$this->rb['extra_one_digit_userid']` come from the rule **context** — the per-account fact sheet described next. Core context already covers scores, totals, device, IP, and email traits (e.g. `eup_ai_bot` used by D13).

Rules are evaluated by the cron worker over accounts with fresh activity; `CHECK_RULE_USERS_LIMIT` (default 1000) caps how many accounts are re-scored per run. A rule can be dry-run from the dashboard with the *play* action on `/rules`.

New core rules in v0.10.0: **I13** (IP belongs to suspicious ASN, driven by `assets/lists/asn.php`), **D11** (empty user agent), **D12** (empty browser language).

## Rule context

To feed custom rules with facts that core context doesn't provide, drop a `Context.php` into `assets/rules/custom/`. The shipped [`Context.example.php`](assets/rules/custom/Context.example.php) shows the two-method contract:

```php
namespace Tirreno\Rules\Custom;

class Context extends \Tirreno\Assets\Context {
    protected ?string $DB_TABLE_NAME = 'event_account';
    protected ?bool $uniqueValues = false;

    // 1) batched SQL: fetch raw details for a set of account ids (tenant-scoped by API key)
    protected function getDetails(array $accountIds, int $apiKey): array {
        [$params, $placeHolders] = $this->getRequestParams($accountIds, $apiKey);
        $query = "SELECT event_account.id AS id,
                         event_account.userid AS extra_userid
                  FROM event_account
                  WHERE event_account.id IN ({$placeHolders}) AND event_account.key = :api_key";

        return $this->execQuery($query, $params);
    }

    // 2) derive rule variables from the fetched details
    public function expandContext(array &$extraData, array &$user): void {
        $user['extra_one_digit_userid'] = substr(($extraData['extra_userid'][0][0] ?? ' '), 0, 1) === '1';
    }
}
```

`getDetails()` runs once per batch of accounts (always constrain by `:api_key`), and `expandContext()` turns the fetched rows into variables on the account's context array — which is exactly what `$this->rb['...']` reads inside `defineCondition()`. Prefixing custom variables with `extra_` avoids collisions with core context.

## Rule presets

A preset is a named bundle of rule-uid → weight assignments that an operator can apply in one click (on `/rules` via *Apply preset*, or when signing up). Core presets live in `assets/rules/core/`, custom presets in `assets/rules/custom/`, named `preset-<preset-name>.php` (lowercase alphanumerics and dashes). A custom preset with the same name **overrides** the core one.

A preset file returns a plain array — see [`assets/rules/custom/preset-custom-1.example.php`](assets/rules/custom/preset-custom-1.example.php) (rename to `preset-custom-1.php` to activate):

```php
return [
    'description'   => 'Custom preset I',
    'rules'         => [
        'A01'   => 'extreme',
        'A02'   => 'extreme',
        'A05'   => 'extreme',
    ],
];
```

Weights are the names shown above (`positive`, `none`, `medium`, `high`, `extreme`); entries with unknown uids or weights are skipped. Shipped presets (new set in v0.10.0, matching the README's preset list): `default` (empty), `account-registration`, `account-takeover`, `api-protection`, `bot-detection`, `content-spam`, `credential-stuffing`, `dormant-account`, `fraud-prevention`, `high-risk-regions`, `insider-threat`, `multi-accounting`, `promo-abuse`.

## Lists

`assets/lists/` contains plain PHP files returning arrays of strings that rules and detectors consume; edit them to tune detection to your product:

| File | Used for |
|------|----------|
| `ai-bot.php` | AI/LLM bot user agents — feeds rule D13 (*new in v0.10.0*) |
| `asn.php` | Flagged ASNs — feeds rule I13 (*new in v0.10.0*) |
| `user-agent.php` | Suspicious/vulnerable user-agent substrings |
| `url.php` | Suspicious URL substrings |
| `email.php` | Suspicious email patterns |
| `file-extensions.php` | File-extension filter for the `/resource` grid |

Each file returns a PHP array and overrides the corresponding built-in default. Programmatic access goes through `tirreno('assets')`: `aiBotList`, `asnList`, `userAgentList`, `urlList`, `emailList`, `fileExtensionsList` — all exposing `getList()`.

## UI constants

[`assets/dashboard/Constants.php`](assets/dashboard/Constants.php) collects dashboard UI tuning knobs (string truncation lengths in tables and tiles, etc.) in one update-safe class, available as `tirreno('assets')->uiConstants`. Server-side counterparts are reachable as `tirreno('assets')->serverConstants` and the full application constant set as `tirreno('utils')->constants`.

## Configuration variables

Defaults live in [`config/config.ini`](config/config.ini); machine-local overrides belong in `config/local/config.local.ini` (never edit shipped files in place). A selection relevant to development — the first three are **new in v0.10.0**:

| Variable | Default | Meaning |
|----------|---------|---------|
| `CHECK_RULE_USERS_LIMIT` | `1000` | Max accounts re-scored per cron rule run. |
| `RECALCULATE_TOTALS_ON_VISIT` | `true` | Recalculate entity totals on visit instead of deferring. |
| `LOG_TO_STDERR` | `false` | Duplicate `tirreno('log')` output to stderr (container-friendly). |
| `DEBUG` | `0` | `1` renders a stack trace on the error page (new behaviour in v0.10.0) and enables `tirreno('log')->debug()`. Never enable in production. |
| `LEAKY_BUCKET_RPS` / `LEAKY_BUCKET_WINDOW` | `10` / `20` | Sensor overload protection (requests per second / burst window). |
| `LOGBOOK_LIMIT` | `3000` | Logbook retention size. |
| `USER_QUEUE_EVENTS_LIMIT` | `100000` | Cap on queued events per processing run. |
| `ALLOW_FORGOT_PASSWORD` | `false` | Enable the password-recovery flow (requires mail settings). |
| `ALLOW_EMAIL_PHONE` | `false` | Treat email/phone as first-class searchable identifiers. |
| `FORCE_HTTPS` | `false` | Force secure cookies/redirects behind TLS. |

Typed access from code goes through `tirreno('utils')->variables` (e.g. `getDebug()`, `getLogToStderr()`, `getCheckRuleUsersLimit()`) rather than reading the hive directly.

## RBAC and system operators

**New in v0.10.0.** Operators now carry roles with attached page permissions:

| Role | Purpose |
|------|---------|
| `superuser` | Full control, operator administration. |
| `operator` | Regular analyst work in the dashboard. |
| `guest` | Read-only/anonymous baseline. |

Two **system operators** exist alongside human ones: a *guest* operator (represents unauthenticated access) and a *cron/daemon* operator (background jobs run under it). Operator ids below 100 are reserved for system use.

For your own pages, access control is declarative:

```php
// class-based pages, in init():
tirreno('page')->setAuthenticated(true);
tirreno('page')->setAllowedRoles(['operator']);            // or per-verb: ['GET' => ['guest'], 'POST' => ['operator']]
tirreno('page')->setBlockedRoles(['guest']);

// file-based pages, imperative guards:
$response->redirectNotLoggedIn('/login');
$response->redirectImproperRole(['operator'], [], '/login');   // allowed, blocked, target
```

Role checks are also available on the operator entity itself: `tirreno('session')->getCurrentOperator()->roles`, `->isGuest()`, `->isLoggedIn()`.

## Coding standards and tooling

The codebase is deliberately "low-tech": `declare(strict_types=1)` everywhere, no framework magic beyond the small F3-compatible core, and few dependencies (vendored under `libs/`, so production needs no Composer).

For development:

```bash
composer install                  # dev tools (runtime works without vendor/)

vendor/bin/phpcs                  # code style   — ruleset in phpcs.xml
vendor/bin/phpstan analyse        # static analysis — phpstan.neon
vendor/bin/phpunit                # tests        — phpunit.xml, tests/
npx eslint ui/js                  # JS lint      — eslint.config.js
```

Match the existing style when contributing: PSR-4 under `\Tirreno\` → `app/`, one class per file, aligned array assignments, and the `tirreno('...')` API instead of reaching into classes directly. Please review [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) and [SECURITY.md](SECURITY.md) (vulnerability reports) before submitting changes.

## Links

* [Website](https://www.tirreno.com)
* [User guide](https://docs.tirreno.com)
* [Live demo](https://play.tirreno.com) (*admin/tirreno*)
* [Changelog](CHANGELOG.md) · [Release notes](RELEASE_NOTES.md)
* [PHP SDK](https://github.com/tirrenotechnologies/tirreno-php-tracker) · [Python SDK](https://github.com/tirrenotechnologies/tirreno-python-tracker) · [NodeJS SDK](https://github.com/tirrenotechnologies/tirreno-nodejs-tracker) · [WordPress plugin](https://github.com/tirrenotechnologies/tirreno-wordpress-tracker)
* [Mattermost community](https://chat.tirreno.com)
