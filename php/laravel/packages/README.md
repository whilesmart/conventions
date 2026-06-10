# PHP / Laravel package conventions

How we design PHP packages for Laravel at WhileSmart. Every `whilesmart/eloquent-*` package
follows this so they stay interchangeable: same skeleton, same tenancy model, same release
flow, same CI. A new package should be a copy-and-adapt of a currently maintained, in-use
package (pick a recently released one from the
[Packagist listing](https://packagist.org/packages/whilesmart/)) plus this document.

---

## 1. Naming

- Repo + package: `whilesmart/eloquent-<noun>`, the noun plural and kebab-case
  (`eloquent-invoices`, `eloquent-customers`, `eloquent-assets`).
- Namespace: `Whilesmart\<Noun>\…` (`Whilesmart\Invoices\Models\Invoice`).
- Avoid names that collide with Laravel concepts: not `eloquent-resources` (API Resources),
  not `eloquent-inventory` if `products` already covers stock. A domain noun like "Assets" is
  fine (there is no Laravel `Asset` class); just don't name a DB column `attributes` (collides
  with Eloquent's internal `$attributes`) — use `details`/`metadata` instead.

## 2. Repository & branch model

- **`dev`** is the default branch and where development happens. Feature branches open PRs into
  `dev`.
- **`main`** is the release branch. You promote `dev` → `main` to cut a release.
- Never commit straight to `dev`/`main`; always PR.

## 3. Versioning & releases

Releases are automated by the shared workflow `whilesmart/workflows/package/release@main`,
which runs on **push to `main`**. It reads the version from `composer.json` and the notes from
`CHANGELOG.md`, then tags `vX.Y.Z` and drafts a GitHub release. That tag is what
[Packagist](https://packagist.org/packages/whilesmart/) serves: a released package is one
consumers can pull by version from Packagist. Browse the published packages at
<https://packagist.org/packages/whilesmart/>.

- Put the version in the **standard `version` field** of `composer.json` (not the non-standard
  `package-version`; the workflow falls back to it, but `version` is correct).
- Every release needs a matching `CHANGELOG.md` entry: `## [X.Y.Z] - YYYY-MM-DD` followed by
  bullet points. The release fails if the entry is missing or the tag `vX.Y.Z` already exists.
- `pre-release.yml` runs the same checks on PRs into `main`, so a `dev` → `main` PR tells you up
  front whether the version/changelog are ready.
- SemVer: `fix` → patch, `feat` → minor, behaviour you remove/restrict → major. Bump the
  `version` field and add the changelog entry in the same PR as the change.

## 4. composer.json template

```json
{
    "name": "whilesmart/eloquent-<noun>",
    "description": "...",
    "type": "library",
    "version": "1.0.0",
    "license": "MIT",
    "authors": [{ "name": "Whilesmart Team" }],
    "require": {
        "php": "^8.2",
        "laravel/framework": "^11.0|^12.0"
    },
    "require-dev": {
        "orchestra/testbench": "^9.0|^10.0",
        "laravel/pint": "^1.22",
        "fakerphp/faker": "^1.24"
    },
    "config": { "platform": { "php": "8.2" } },
    "autoload": {
        "psr-4": {
            "Whilesmart\\<Noun>\\": "src/",
            "Whilesmart\\<Noun>\\Database\\Factories\\": "database/factories/"
        }
    },
    "autoload-dev": { "psr-4": { "Tests\\": "tests/" } },
    "extra": { "laravel": { "providers": ["Whilesmart\\<Noun>\\<Noun>ServiceProvider"] } },
    "scripts": {
        "test": "vendor/bin/testbench package:test",
        "pint": ["./vendor/bin/pint"],
        "pint:test": ["./vendor/bin/pint --test"]
    },
    "minimum-stability": "dev",
    "prefer-stable": true
}
```

Cross-package dependencies are pinned `dev-dev` (or `dev-main`) while in active development,
e.g. `eloquent-invoices` requires `eloquent-customers`, `eloquent-owner-access`,
`eloquent-payments`. Dependency direction matters: lower-level packages (`owner-access`,
`files`, `payments`) must not depend on higher-level domain packages.

## 5. File layout

```
src/
  <Noun>ServiceProvider.php       # merge config, load migrations, publish, register routes
  Models/<Model>.php
  Enums/<Model>Status.php
  Http/Controllers/<Model>Controller.php
  Http/Requests/{Store,Update}<Model>Request.php
  Http/Resources/<Model>Resource.php
  Traits/Has<Models>.php          # morphMany for owners
  Events/…                        # domain events the host app can listen to
config/<noun>.php
routes/api.php
database/migrations/…  database/factories/<Model>Factory.php
tests/{TestCase.php, Feature/…}
.github/workflows/{checks,commits,pre-release,release}.yml
phpunit.xml  .gitignore  docker-compose.yml  Makefile  README.md  CHANGELOG.md
```

The ServiceProvider always: `mergeConfigFrom`, `loadMigrationsFrom`, `publishes` (config +
migrations), and **conditional route registration** gated by `config('<noun>.register_routes')`
with `route_middleware`/`route_prefix` from config (default `['api','auth:sanctum']` + `api`).

## 6. Tenancy: owner-access is mandatory

Every record is scoped to a polymorphic owner (`$table->morphs('owner')`). Authorization is
delegated to [`whilesmart/eloquent-owner-access`](https://github.com/whilesmartphp/eloquent-owner-access);
the host app binds an `OwnerAuthorizer` (e.g. a `WorkspaceMemberAuthorizer`). Packages never
decide tenancy themselves and never trust a client-supplied owner without these checks.

- Controller: `use AuthorizesOwnerController;`
  - `index`: `$this->scopeAccessibleOwners(Model::query(), $request->user())`
  - `show`/`update`/`destroy`/custom actions: `$this->authorizeAccessTo($record, $request->user())`
- `StoreXRequest::authorize()` → `$this->authorizeOwnerInRequest()`
- `UpdateXRequest::authorize()` → `$this->authorizeOwnerOfBoundModel('x')`

This is the single most important convention. `eloquent-customers` once shipped without it and
let any user read another tenant's records; do not repeat that.

## 7. Model conventions

- `protected $guarded = ['id'];`
- `getTable()` returns the configurable table name (`config('<noun>.<noun>_table', '<table>')`).
- `owner(): MorphTo`, `SoftDeletes`, `HasFactory` + `newFactory()`.
- `casts`: enums for status, `date`/`datetime` for timestamps, `array` for JSON columns.
- Per-owner natural keys (invoice number, asset reference) are **unique per owner, not
  globally**: `$table->unique(['owner_type','owner_id','<key>'])`, and a `booted() creating`
  hook generates the next value per owner (`INV-00001`, `AST-00001`). Never a global unique on
  the natural key — it breaks multi-tenant numbering.
- Keep packages host-agnostic: emit domain events (`InvoiceSent`, `InvoicePaid`) and let the
  host app bridge them to its own systems. Don't import app classes into a package.

## 8. API shape

- `apiResource('<nouns>', Controller::class)` plus custom `POST` actions for transitions
  (`/send`, `/mark-paid`, `/renew`, `/retire`).
- Response envelope: `{ "success": true, "data": … }`. Index returns the paginated resource
  collection: `Resource::collection($paginated)->response()->getData(true)` (so clients read
  `data.data`, `data.meta`).
- A `Resource` class shapes every model's JSON; never return raw models.

## 9. Cross-database compatibility

Tests run on SQLite, dev runs on MySQL, prod may be MySQL or Postgres. Write queries that work
on all three: no Postgres-only `ilike`. For case-insensitive search use
`whereRaw('lower(col) like ?', ['%'.strtolower($term).'%'])`.

## 10. Testing (testbench)

- `phpunit.xml`: SQLite `:memory:`, a fixed test `APP_KEY`.
- `Tests\` namespace; `TestCase extends Orchestra\Testbench\TestCase` with `RefreshDatabase`,
  `defineDatabaseMigrations()` (load this package's migrations + any dependency migrations it
  needs), `getPackageProviders()` (OwnerAccess + this package + any dep providers), and
  `getEnvironmentSetUp()` setting `<noun>.route_middleware` to `['api']` (drop sanctum in tests).
- Two feature tests minimum:
  - `<Model>ApiTest`: CRUD + custom actions through the **HTTP boundary** (default
    `AllowAllAuthorizer` permits).
  - `<Model>AuthorizationTest`: bind a denying `OwnerAuthorizer` in `setUp`, assert `403` on
    store/show/update/destroy/custom actions and an empty scoped `index` (`data.meta.total` 0).
- Drive tests the way the user does (through the controller), not by calling models directly.

## 11. Linting (pint)

`laravel/pint` with the default Laravel preset. `composer pint` to fix, `composer pint:test` in
CI. Watch: `method_chaining_indentation` (chained `->` lines use a 4-space continuation indent,
not alignment), ordered imports, trailing commas in multiline arrays.

## 12. Commits

Follow the org [commit conventions](../../../commits.md). CI enforces them with `commits.yml`
(extends `@commitlint/config-conventional`). Two rules that bite in packages specifically:
subject must be **sentence-case** and **body lines must be ≤ 100 characters** (wrap the body,
or commitlint fails the PR).

## 13. CI workflows (the four, copied verbatim per package)

| Workflow | Trigger | Does |
|----------|---------|------|
| `checks.yml` | PR → `main`/`dev` | `composer install`, `testbench package:test`, `pint:test` on PHP 8.2/8.3/8.4 |
| `commits.yml` | PR → `main`/`dev` | commitlint on the PR commits |
| `pre-release.yml` | PR → `main` | `whilesmart/workflows/package/pre-release@main` (version + changelog + tag-free) |
| `release.yml` | push → `main` | `whilesmart/workflows/package/release@main` (tag `vX.Y.Z` + release) |

`checks.yml` runs from the PR head branch, so a PR that introduces `checks.yml` validates
itself. A package without a test/pint harness (no `tests/`, no `pint` script) must add that
harness before adding `checks.yml`, or CI fails on every PR.

## 14. Local dev loop

`docker-compose.yml` uses `ghcr.io/whilesmartphp/laravel-dev:package-8.4`; the `Makefile` pulls
`package.mk` from the image. Common commands:

```
make bootstrap                                  # one-time: fetch package.mk
docker compose run --rm app composer test       # testbench tests
docker compose run --rm app composer pint       # autofix style
```

## 15. Consuming a package from a host app

The constraint a host app pins moves through the package's lifecycle:

- **Released (ideal):** once a package is released on **Packagist** (a `vX.Y.Z` tag from
  `release.yml`), pin a tagged caret constraint, e.g. `"whilesmart/eloquent-<noun>": "^1.2"`.
  This is the production state.
- **During development, before the Packagist release is done:** pin the package by its branch
  so the host app can depend on it ahead of publication — `dev-main` to track the release
  branch, or `dev-dev` for active unreleased work. (`minimum-stability: dev` + `prefer-stable`
  in the host app allow these alongside stable deps.)
- After a package change lands: merge/release it, then in the host app
  `composer update whilesmart/eloquent-<noun>` and `php artisan migrate`.
- The host app provides the tenancy decision by binding `OwnerAuthorizer`, and bridges package
  domain events to its own features (automations, notifications) via its `EventServiceProvider`.

---

## Checklist for a new package

1. Copy a currently maintained, in-use package as the skeleton; rename namespace/files.
2. Owner-access on the controller + both requests.
3. Config-driven table name + route registration; `Resource` envelope.
4. Per-owner unique natural key with a `creating` generator (if applicable).
5. `version` + `CHANGELOG.md` first entry; the four CI workflows.
6. `<Model>ApiTest` + `<Model>AuthorizationTest`, green on 8.2/8.3/8.4 + pint.
7. Repo: `dev` default, `main` release; open the scaffold as a PR into `dev`.
8. Host app: require `dev-dev`, add the owning trait, bridge any events.
