# Configuration Reference — config/tenancy.php

## models

```php
'models' => [
    'tenant' => \App\Models\Tenant::class,
    'domain' => \Stancl\Tenancy\Database\Models\Domain::class,
    'tenant_key_column' => 'tenant_id',  // column name on tenant-scoped models
    'id_generator' => Stancl\Tenancy\UUIDGenerator::class, // or null for autoincrement
],
```

Available ID generators: `UUIDGenerator`, `ULIDGenerator`, `UUIDv7Generator`, `RandomHexGenerator`, `RandomIntGenerator`, `RandomStringGenerator`, or `null`.

## identification

```php
'identification' => [
    'central_domains' => [
        'saas.test',
        'saas.com',
    ],
    'default_middleware' => \Stancl\Tenancy\Middleware\InitializeTenancyByDomain::class,
    'resolvers' => [
        Resolvers\DomainTenantResolver::class => [
            'cache' => false,
            'cache_ttl' => 3600,
            'cache_store' => null,
        ],
        Resolvers\PathTenantResolver::class => [
            'tenant_parameter_name' => 'tenant',
            'tenant_model_column' => null, // null = find by primary key
            'cache' => false,
            'cache_ttl' => 3600,
            'cache_store' => null,
        ],
        Resolvers\RequestDataTenantResolver::class => [
            'header' => 'X-Tenant',
            'cookie' => 'tenant',
            'query_parameter' => 'tenant',
            'cache' => false,
            'cache_ttl' => 3600,
            'cache_store' => null,
        ],
    ],
],
```

Set individual resolver fields to `null` to disable (e.g., `'cookie' => null` to not check cookies).

Cache invalidation is automatic on tenant save. Clear manually: `app(DomainTenantResolver::class)->invalidateCache($tenant)`.

## bootstrappers

```php
'bootstrappers' => [
    Stancl\Tenancy\Bootstrappers\DatabaseTenancyBootstrapper::class,
    Stancl\Tenancy\Bootstrappers\CacheTenancyBootstrapper::class,
    Stancl\Tenancy\Bootstrappers\FilesystemTenancyBootstrapper::class,
    Stancl\Tenancy\Bootstrappers\QueueTenancyBootstrapper::class,
    // Stancl\Tenancy\Bootstrappers\RedisTenancyBootstrapper::class,
],
```

For single-database mode, remove `DatabaseTenancyBootstrapper`.

## database

```php
'database' => [
    'central_connection' => env('DB_CONNECTION', 'mysql'),
    'template_tenant_connection' => null, // connection name from config/database.php
    'prefix' => 'tenant_',
    'suffix' => '',
    'managers' => [
        'mysql' => Stancl\Tenancy\TenantDatabaseManagers\MySQLDatabaseManager::class,
        'pgsql' => Stancl\Tenancy\TenantDatabaseManagers\PostgreSQLDatabaseManager::class,
        'sqlite' => Stancl\Tenancy\TenantDatabaseManagers\SQLiteDatabaseManager::class,
        'sqlsrv' => Stancl\Tenancy\TenantDatabaseManagers\MicrosoftSQLDatabaseManager::class,
    ],
    'drop_tenant_databases_on_migrate_fresh' => false,
],
```

## cache

```php
'cache' => [
    'prefix' => 'tenant_%tenant%_',
    'stores' => [], // which cache stores to scope
    'scope_sessions' => false,
    'tag_base' => 'tenant',
],
```

## filesystem

```php
'filesystem' => [
    'suffix_base' => 'tenant',
    'disks' => ['local', 'public'],
    'root_override' => [
        'local' => '%storage_path%/app/',
        'public' => '%storage_path%/app/public/',
    ],
    'url_override' => [
        'public' => 'public-%tenant%',
    ],
    'suffix_storage_path' => true,
    'asset_helper_override' => true,
],
```

## redis

```php
'redis' => [
    'prefix' => 'tenant_%tenant%_',
    'prefixed_connections' => ['default'],
],
```

Do NOT include queue Redis connections here if using Redis for queues.

## features

```php
'features' => [
    // Stancl\Tenancy\Features\UserImpersonation::class,
    // Stancl\Tenancy\Features\TenantConfig::class,
    // Stancl\Tenancy\Features\CrossDomainRedirect::class,
    // Stancl\Tenancy\Features\ViteBundler::class,
    // Stancl\Tenancy\Features\TelescopeTags::class,
    // Stancl\Tenancy\Features\UniversalRoutes::class,
],
```

## default_route_mode

```php
'default_route_mode' => Stancl\Tenancy\Enums\RouteMode::CENTRAL,
// Options: RouteMode::CENTRAL, RouteMode::TENANT, RouteMode::UNIVERSAL
```

## rls (Row-Level Security)

```php
'rls' => [
    'manager' => Stancl\Tenancy\RLS\TableRLSManager::class,
    'user' => [
        'username' => 'tenant_user',
        'password' => env('TENANT_DB_PASSWORD'),
    ],
    'session_variable_name' => 'tenancy.tenant_id',
],
```

PostgreSQL-only. Alternative to application-level scoping with BelongsToTenant.

## migration_parameters / seeder_parameters

```php
'migration_parameters' => [
    '--force' => true,
    '--path' => [database_path('migrations/tenant')],
],
'seeder_parameters' => [
    '--class' => 'TenantDatabaseSeeder',
],
```

Default parameters passed to `tenants:migrate` and `tenants:seed`.
