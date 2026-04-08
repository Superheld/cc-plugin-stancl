# Optional Features

Enable features in `tenancy.features` config array.

## UserImpersonation

Impersonate users across tenants from a central admin panel:

```php
// Generate impersonation token (from central context)
$token = tenancy()->impersonate($tenant, $userId, '/dashboard');

// In tenant routes — handle the impersonation redirect
Route::get('/impersonate/{token}', function ($token) {
    return \Stancl\Tenancy\Features\UserImpersonation::makeResponse($token);
});
```

Configure TTL: `UserImpersonation::$ttl = 120;` (seconds)

## TenantConfig

Map tenant attributes to Laravel config values. Changes apply when tenancy initializes:

```php
use Stancl\Tenancy\Features\TenantConfig;

TenantConfig::$storageToConfigMap = [
    'paypal_api_key' => 'services.paypal.api_key',
    'locale' => ['app.locale', 'locales.default'], // maps to multiple config keys
];
```

Store values on the tenant: `$tenant->update(['paypal_api_key' => 'sk_...'])`.

## CrossDomainRedirect

Redirect between tenant domains:

```php
return redirect()->route('home')->domain($domain);
return redirect(tenant_route($domain, 'home'));
```

## ViteBundler

Ensures Vite generates correct asset paths per tenant. Enable when using Vite with tenancy.

## TelescopeTags

Adds tenant ID tags to Laravel Telescope entries for filtering.

## UniversalRoutes

Routes that work in both central and tenant context:

```php
Route::get('/foo', function () {
    $tenantId = tenant('id'); // null in central, set in tenant
})->middleware(['universal', InitializeTenancyByDomain::class]);
```

Custom `$onFail` logic on identification middleware is incompatible with universal routes.

## MaintenanceMode

Per-tenant maintenance mode:

```php
// Put tenant in maintenance
$tenant->putDownForMaintenance();

// Bring back up
$tenant->update(['maintenance_mode' => null]);
```

Add `CheckTenantForMaintenanceMode` middleware to tenant routes.

## Helper Functions

```php
tenant_asset($path)          // tenant-aware asset URL
global_asset($path)          // global (non-tenant) asset URL
global_cache()               // access global cache store (bypasses tenant prefix)
tenant_route($domain, $route, $params)  // generate URL on specific tenant domain
tenant_channel($name, $cb)   // tenant-prefixed broadcast channel
global_channel($name, $cb)   // global broadcast channel
```

## Livewire v3 Integration

Override the Livewire update route in `TenancyServiceProvider::boot()`:

```php
use Livewire\Livewire;

Livewire::setUpdateRoute(function ($handle) {
    return Route::post('/livewire/update', $handle)
        ->middleware(['web', 'universal', Tenancy::defaultMiddleware()]);
});
```

This ensures Livewire AJAX requests go through tenant identification.
