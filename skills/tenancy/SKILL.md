---
name: tenancy
description: "This skill should be used when the user asks to 'set up multi-tenancy', 'add tenant isolation', 'configure stancl/tenancy', 'create a tenant model', 'add BelongsToTenant', 'scope models to tenant', 'set up tenant identification', 'configure bootstrappers', 'create tenant routes', 'test tenant isolation', 'dispatch tenant-aware jobs', 'run tenant migrations', or works with stancl/tenancy in any capacity. Also activates when creating models that need tenant scoping, writing migrations with tenant_id, or debugging cross-tenant data leaks."
---

# stancl/tenancy v3 — Multi-Tenancy for Laravel

stancl/tenancy provides automatic multi-tenancy for Laravel. It supports two primary modes: **multi-database** (each tenant gets a separate database) and **single-database** (all tenants share one database, isolated via Global Scopes). Hybrid approaches are also possible.

## Installation

```bash
composer require stancl/tenancy
php artisan tenancy:install
php artisan migrate
```

Register the provider in `bootstrap/providers.php`:

```php
return [
    App\Providers\AppServiceProvider::class,
    App\Providers\TenancyServiceProvider::class,
];
```

The `tenancy:install` command creates `config/tenancy.php`, a `TenancyServiceProvider` stub, migrations for `tenants`/`domains` tables, and `routes/tenant.php`.

## Core Concepts

### Tenant Model

Always create a custom Tenant model. For multi-database mode, implement `TenantWithDatabase` and use `HasDatabase`. For single-database mode, omit both. Add `HasDomains` if using domain-based identification.

The base model uses `stancl/virtualcolumn` — columns in `getCustomColumns()` are real DB columns, everything else goes into a JSON `data` column.

### Accessing the Current Tenant

```php
tenant()              // Tenant|null
tenant('id')          // specific attribute
tenancy()             // Tenancy singleton
tenancy()->initialize($tenant)  // manual initialization
tenancy()->end()                // revert to central context
$tenant->run(function () { })   // run in specific tenant context
```

### Tenant Identification

Identification middleware resolves the tenant from the request:

| Middleware | Identifies by |
|---|---|
| `InitializeTenancyByDomain` | Full hostname |
| `InitializeTenancyBySubdomain` | Subdomain part only |
| `InitializeTenancyByDomainOrSubdomain` | Dots = domain, no dots = subdomain |
| `InitializeTenancyByPath` | URL path segment `/{tenant}` |
| `InitializeTenancyByRequestData` | Header / query param / cookie |

Each middleware has a static `$onFail` callback for custom error handling.

### Routes

Central routes go in `routes/web.php` (wrapped in domain group), tenant routes in `routes/tenant.php` (with identification middleware). Configure `tenancy.identification.central_domains` to distinguish them.

Route modes (`RouteMode::CENTRAL`, `TENANT`, `UNIVERSAL`) control default behavior. Override per-route with middleware.

### Bootstrappers

Bootstrappers activate when tenancy initializes and revert when it ends. Configure in `tenancy.bootstrappers`. Key bootstrappers:

| Bootstrapper | Purpose |
|---|---|
| `DatabaseTenancyBootstrapper` | Switches default DB connection (multi-db only) |
| `CacheTenancyBootstrapper` | Prefixes cache keys |
| `FilesystemTenancyBootstrapper` | Suffixes storage paths and disk roots |
| `QueueTenancyBootstrapper` | Preserves tenant context in queued jobs |
| `RedisTenancyBootstrapper` | Prefixes Redis keys (requires phpredis) |

Custom bootstrappers implement `TenancyBootstrapper` with `bootstrap()` and `revert()` methods.

### Event System

Tenancy is event-driven. The `TenancyServiceProvider` maps events to listener pipelines via `stancl/jobpipeline`. Key event groups: tenant lifecycle (`TenantCreated`, etc.), domain lifecycle, database lifecycle, and tenancy lifecycle (`TenancyInitialized`, `TenancyEnded`, etc.).

## Single-Database Mode

Disable `DatabaseTenancyBootstrapper`. Remove `CreateDatabase`/`MigrateDatabase`/`SeedDatabase` from the `TenantCreated` pipeline. Use `BelongsToTenant` trait on primary models, `BelongsToPrimaryModel` on secondary models accessed only through parent relationships.

Bypass scoping with `Model::withoutTenancy()` — only in admin/platform contexts.

For detailed setup, patterns, and validation rules, consult **`references/single-database.md`**.

## Multi-Database Mode

Keep `DatabaseTenancyBootstrapper` enabled. The `TenantCreated` event pipeline handles database creation, migration, and seeding. Place tenant migrations in `database/migrations/tenant/`.

For detailed setup and database manager options, consult **`references/multi-database.md`**.

## Jobs & Queues

`QueueTenancyBootstrapper` serializes tenant ID into job payloads and re-initializes tenancy on processing. The queue connection MUST point to the central database — otherwise jobs go to tenant DBs and are lost.

Dispatch from non-tenant context using `$tenant->run()`.

## Testing

Initialize tenancy in test setup: `tenancy()->initialize(Tenant::create())`. Never use `Event::fake()` without arguments — it breaks tenancy initialization. Always use selective faking: `Event::fake([SpecificEvent::class])`.

For multi-database testing, `:memory:` SQLite and `RefreshDatabase` are incompatible.

For detailed testing patterns, consult **`references/testing.md`**.

## Console Commands

| Command | Purpose |
|---|---|
| `tenants:migrate` | Run tenant migrations |
| `tenants:seed` | Seed tenant databases |
| `tenants:run` | Run any artisan command in tenant context |
| `tenants:list` | List all tenants |
| `tenants:tinker` | Tinker in tenant context |

Target specific tenants with `--tenants=id` (repeatable).

## Critical Rules

1. **Never add manual tenant filters** (`where('tenant_id', ...)`) when `BelongsToTenant` is used — the Global Scope handles it
2. **Never use `Event::fake()`** without arguments — it breaks tenancy
3. **Match the ID type** in migrations — `foreignUuid` for UUID tenants, `foreignId` for integer
4. **Always initialize tenancy** in artisan commands, seeders, and tests before querying scoped models
5. **Queue connections must use the central database** — never the tenant connection
6. **Policies check permissions, not tenant** — isolation is handled by the Global Scope
7. **Raw `DB::` queries are NOT scoped** — only Eloquent is affected by `BelongsToTenant`
8. **`withoutTenancy()` only in admin/platform contexts** — never in tenant routes

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Cross-tenant data leak on new model | Add `BelongsToTenant` trait + `tenant_id` column + index |
| `foreignId()` with UUID tenants | Use `foreignUuid('tenant_id')` |
| Jobs disappear or go to wrong DB | Set `'connection' => 'central'` in queue config |
| Tenant data mixed in artisan commands | Call `tenancy()->initialize()` before scoped queries |
| Tests fail with tenancy | Add `tenancy()->initialize()` to `beforeEach` |

## Additional Resources

### Reference Files

For detailed guidance on specific topics, consult:

- **`references/single-database.md`** — BelongsToTenant, BelongsToPrimaryModel, validation rules, unique indexes, scoping details
- **`references/multi-database.md`** — Database managers, tenant database naming, migration workflow, resource syncing
- **`references/testing.md`** — Test setup patterns, cross-tenant isolation tests, Event::fake pitfalls
- **`references/config.md`** — Full config/tenancy.php reference with all sections and options
- **`references/features.md`** — Optional features: UserImpersonation, TenantConfig, CrossDomainRedirect, ViteBundler, MaintenanceMode

### Related Skills

This plugin also includes skills for tenancy's dependencies:

- **jobpipeline** — `stancl/jobpipeline` converts jobs into event listeners, used for tenancy's event pipeline (TenantCreated, etc.)
- **virtualcolumn** — `stancl/virtualcolumn` powers the Tenant model's flexible JSON data storage
