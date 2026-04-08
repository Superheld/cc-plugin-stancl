# Multi-Database Tenancy

## Setup

Keep `DatabaseTenancyBootstrapper` in `tenancy.bootstrappers`. The Tenant model must implement `TenantWithDatabase` and use `HasDatabase`:

```php
use Stancl\Tenancy\Database\Models\Tenant as BaseTenant;
use Stancl\Tenancy\Contracts\TenantWithDatabase;
use Stancl\Tenancy\Database\Concerns\HasDatabase;
use Stancl\Tenancy\Database\Concerns\HasDomains;

class Tenant extends BaseTenant implements TenantWithDatabase
{
    use HasDatabase, HasDomains;
}
```

## Event Pipeline

The `TenancyServiceProvider` maps `TenantCreated` to database setup jobs:

```php
Events\TenantCreated::class => [
    JobPipeline::make([
        Jobs\CreateDatabase::class,
        Jobs\MigrateDatabase::class,
        Jobs\SeedDatabase::class,
    ])->send(fn (TenantCreated $event) => $event->tenant)
      ->shouldBeQueued(false),
],
```

Remove `SeedDatabase` if seeding is not needed on tenant creation.

## Tenant Migrations

Place tenant-specific migrations in `database/migrations/tenant/`. Central migrations stay in `database/migrations/`.

```bash
php artisan tenants:migrate
php artisan tenants:migrate --tenants=foo --tenants=bar
php artisan tenants:migrate-fresh
php artisan tenants:rollback
php artisan tenants:seed
```

## Database Naming

Configure in `tenancy.database`:

```php
'prefix' => 'tenant_',
'suffix' => '',
// Result: tenant_<tenant_id>
```

## Database Managers

Configure in `tenancy.database.managers`:

| Manager | Use Case |
|---|---|
| `MySQLDatabaseManager` | Standard MySQL |
| `PostgreSQLDatabaseManager` | Standard PostgreSQL |
| `SQLiteDatabaseManager` | SQLite (file per tenant) |
| `MicrosoftSQLDatabaseManager` | SQL Server |
| `PostgreSQLSchemaManager` | Schema-based separation (single DB, separate schemas) |
| Permission-controlled variants | Create per-tenant DB users with limited privileges |

## Template Connection

The `template_tenant_connection` config defines a database connection template. The package clones this template and overrides the database name dynamically for each tenant.

Define the template connection in `config/database.php`:

```php
'connections' => [
    'tenant' => [
        'driver' => 'mysql',
        'host' => env('DB_HOST', '127.0.0.1'),
        'database' => null, // overridden dynamically
        'username' => env('DB_USERNAME'),
        'password' => env('DB_PASSWORD'),
    ],
],
```

Reference it in `config/tenancy.php`:

```php
'database' => [
    'template_tenant_connection' => 'tenant',
],
```

## Running Code Across Tenants

```php
// Run closure for each tenant
Tenant::all()->runForEach(function () {
    User::factory()->create();
});

// Run for specific tenant
$tenant->run(function () {
    // in tenant context
});
```

## Resource Syncing

For sharing resources (e.g., users) across tenant databases:

**Central model**: Implements `SyncMaster`, uses `ResourceSyncing` + `CentralConnection`
**Tenant model**: Implements `Syncable`, uses `ResourceSyncing`

Both define `getSyncedAttributeNames()`, `getGlobalIdentifierKey()`, `getGlobalIdentifierKeyName()`.

Requires a pivot table mapping global IDs to tenant IDs. For production, queue the sync listener:

```php
UpdateOrCreateSyncedResource::$shouldQueue = true;
```

## Important Limitations

- Only the **default** DB connection is switched. Explicit `DB::connection('name')` calls use the specified connection unchanged.
- The `central_connection` config must point to a named connection in `config/database.php` that always reaches the central database.
- `drop_tenant_databases_on_migrate_fresh` controls whether `tenants:migrate-fresh` drops and recreates databases or just wipes tables.
