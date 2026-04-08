---
name: virtualcolumn
description: "This skill should be used when the user asks to 'store flexible attributes on a model', 'use VirtualColumn trait', 'add a JSON data column', 'configure getCustomColumns', 'query virtual columns', or works with stancl/virtualcolumn. Also activates when building Eloquent models that need schemaless attributes alongside real columns, customizing the Tenant model's data storage, or debugging attribute encoding/decoding issues."
---

# stancl/virtualcolumn — Schemaless Eloquent Attributes via JSON

stancl/virtualcolumn is a single-trait package that stores arbitrary model attributes in a JSON database column while exposing them as normal Eloquent attributes. Real database columns and virtual (JSON-stored) columns are seamless — application code treats them identically.

## Core Pattern

```php
use Illuminate\Database\Eloquent\Model;
use Stancl\VirtualColumn\VirtualColumn;

class Tenant extends Model
{
    use VirtualColumn;

    public $guarded = [];

    public static function getCustomColumns(): array
    {
        return ['id', 'name', 'created_at', 'updated_at'];
    }
}
```

```php
// 'name' goes to its own column, 'plan' goes to JSON `data` column
$tenant = Tenant::create(['name' => 'Acme', 'plan' => 'premium']);
$tenant->plan;  // 'premium' — transparent access
$tenant->name;  // 'Acme' — real column, same API
```

## How It Works

1. **On save**: Attributes NOT in `getCustomColumns()` are removed from the model, packed into a JSON `data` column, and the model is saved
2. **On load**: The JSON `data` column is decoded, its keys are spread onto the model as regular attributes, and the `data` attribute is set to null
3. **At runtime**: Application code sees no difference between real and virtual columns

## Migration Setup

The table needs a JSON column (default name `data`):

```php
Schema::create('tenants', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->timestamps();
    $table->json('data')->nullable();  // stores all virtual attributes
});
```

## `getCustomColumns()`

Return an array of ALL real database columns. Everything not listed here goes to JSON.

**Default** (from the trait):
```php
public static function getCustomColumns(): array
{
    return ['id', static::CREATED_AT, static::UPDATED_AT];
}
```

Override with all real columns on the table. Forgetting a real column causes it to be packed into JSON and the actual column to be null — resulting in database errors.

## Custom Data Column Name

Override `getDataColumn()` to use a different column name:

```php
public static function getDataColumn(): string
{
    return 'metadata';  // instead of 'data'
}
```

Update the migration column name accordingly.

## Querying Virtual Columns

Use `getColumnForQuery()` to get the correct query syntax:

```php
$tenant = new Tenant;
$tenant->getColumnForQuery('name');  // 'name' (real column)
$tenant->getColumnForQuery('plan');  // 'data->plan' (JSON path)

Tenant::where($tenant->getColumnForQuery('plan'), 'premium')->get();
// WHERE `data`->'plan' = 'premium'
```

JSON path queries work on MySQL 5.7+, PostgreSQL 9.3+, SQLite 3.38+.

## Casts

Standard Eloquent casts work on virtual columns:

```php
protected $casts = [
    'settings' => 'array',
    'is_active' => 'boolean',
    'trial_ends_at' => 'datetime',
];
```

Encrypted casts (`encrypted`, `encrypted:array`, etc.) receive special handling to prevent double-encryption on decode. Custom encrypted castables must be registered:

```php
MyModel::$customEncryptedCastables = [MyEncryptedCast::class];
```

## Model Inheritance

Child models extending a VirtualColumn parent can define their own `getCustomColumns()`:

```php
class SpecialTenant extends Tenant
{
    public static function getCustomColumns(): array
    {
        return ['id', 'name', 'tier', 'created_at', 'updated_at'];
    }
}
```

## Critical Rules

1. **`getCustomColumns()` must list ALL real columns** — missing a real column causes data loss on save
2. **`$model->data` is always null** at the application level — the trait spreads JSON keys onto the model and nullifies the data attribute. To use an attribute literally named `data`, rename the data column via `getDataColumn()`
3. **No automatic JSON indexing** — add JSON indexes manually for frequently queried virtual columns
4. **No schema validation** — any key-value pair not in `getCustomColumns()` will be stored in JSON
5. **Overrides `getCasts()` and `fireModelEvent()`** — may conflict with other traits overriding the same methods
6. **Performance** — complex queries (ordering, aggregating, joining) on JSON columns have database-specific limitations

## Common Pitfall: Forgotten Columns

```php
// Migration has: id, name, email, data, timestamps
// But getCustomColumns() only lists:
public static function getCustomColumns(): array
{
    return ['id', 'name', 'created_at', 'updated_at'];
}
// 'email' will be packed into JSON — the real email column will be NULL!
```

Fix: Always include every real column in `getCustomColumns()`.

## Version Compatibility

- **Latest**: v1.5.0
- **Laravel**: 10, 11, 12, 13
- **PHP**: 8.1+
- **Install**: `composer require stancl/virtualcolumn`
