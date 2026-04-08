# Single-Database Tenancy

## Setup

1. Remove `DatabaseTenancyBootstrapper` from `tenancy.bootstrappers`
2. Remove `CreateDatabase`, `MigrateDatabase`, `SeedDatabase` from `TenantCreated` event pipeline in `TenancyServiceProvider`
3. Tenant model should NOT implement `TenantWithDatabase` and should NOT use `HasDatabase`

## Model Types

### Primary Models — BelongsToTenant

Add to any model queried directly that needs tenant isolation:

```php
use Stancl\Tenancy\Database\Concerns\BelongsToTenant;

class Post extends Model
{
    use BelongsToTenant;
}
```

This adds:
- A `TenantScope` Global Scope filtering by `tenant_id` on all queries
- Automatic `tenant_id` fill on model creation via `FillsCurrentTenant`

### Secondary Models — BelongsToPrimaryModel

For models accessed only through a parent relationship (e.g., comments through posts):

```php
use Stancl\Tenancy\Database\Concerns\BelongsToPrimaryModel;

class Comment extends Model
{
    use BelongsToPrimaryModel;

    public function getRelationshipToPrimaryModel(): string
    {
        return 'post';
    }

    public function post()
    {
        return $this->belongsTo(Post::class);
    }
}
```

If a secondary model is ever queried directly (search, reports, standalone controllers), promote it to a primary model with `BelongsToTenant` and its own `tenant_id` column.

### Global Models

Models without tenant scoping (e.g., plans, feature flags). No trait needed.

## Custom Tenant Key Column

The default column name is `tenant_id`. Override globally:

```php
// In AppServiceProvider::boot()
BelongsToTenant::$tenantIdColumn = 'team_id';

// Or via config
// tenancy.models.tenant_key_column = 'team_id'
```

## Migrations

Every primary model migration must include the tenant key column:

```php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained('tenants')->cascadeOnDelete();
    $table->index('tenant_id');
    // ... other columns
});
```

For UUID tenants:

```php
$table->foreignUuid('tenant_id')->constrained('tenants')->cascadeOnDelete();
$table->index('tenant_id');
```

Always match the ID type to the tenant model's primary key type.

## Unique Indexes

Scope unique constraints to tenant:

```php
// Primary models — scope to tenant
$table->unique(['tenant_id', 'slug']);

// Secondary models — scope to parent
$table->unique(['post_id', 'user_id']);
```

## Validation Rules

### Manual Approach

```php
use Illuminate\Validation\Rule;

$rules = [
    'slug' => Rule::unique('posts', 'slug')->where('tenant_id', tenant('id')),
];
```

### HasScopedValidationRules Trait

Add `HasScopedValidationRules` to the Tenant model for cleaner syntax:

```php
// In Tenant model
use Stancl\Tenancy\Database\Concerns\HasScopedValidationRules;

class Tenant extends BaseTenant
{
    use HasScopedValidationRules;
}

// In validation
$rules = [
    'slug' => tenant()->unique('posts'),
    'email' => tenant()->exists('users', 'email'),
];
```

## Bypassing Tenant Scope

```php
// All tenants — admin/platform context only
Post::withoutTenancy()->get();
```

Never use in tenant routes. Only appropriate for admin panels, reports, or platform-level operations where cross-tenant access is intentional.

## Important Limitations

- Only Eloquent queries are scoped. Raw `DB::` queries, `DB::table()`, and `DB::select()` are NOT affected by `BelongsToTenant`.
- The `TenantScope` only applies when `tenancy()->initialized` is true. Queries before initialization or after `tenancy()->end()` return unscoped results.
- Relationships defined on a BelongsToTenant model are automatically scoped through the parent. No need to add BelongsToTenant to the related model unless it is also queried directly.
