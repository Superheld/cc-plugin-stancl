# Testing with stancl/tenancy

## Basic Tenant Test Setup

Initialize tenancy before testing scoped functionality:

```php
beforeEach(function () {
    $this->tenant = Tenant::create();
    tenancy()->initialize($this->tenant);
});
```

Or with a dedicated base TestCase:

```php
class TenantTestCase extends TestCase
{
    protected $tenancy = false;

    public function setUp(): void
    {
        parent::setUp();
        if ($this->tenancy) {
            $this->initializeTenancy();
        }
    }

    public function initializeTenancy(): void
    {
        $tenant = Tenant::create();
        tenancy()->initialize($tenant);
    }
}

class PostTest extends TenantTestCase
{
    protected $tenancy = true;

    public function test_creates_post(): void
    {
        $post = Post::factory()->create();
        expect($post->tenant_id)->toBe(tenant('id'));
    }
}
```

## Cross-Tenant Isolation Tests

Verify that tenant scoping actually prevents data leaks:

```php
it('prevents access to other tenant data', function () {
    $tenantA = Tenant::create();
    $tenantB = Tenant::create();

    tenancy()->initialize($tenantA);
    $post = Post::factory()->create();

    tenancy()->initialize($tenantB);
    expect(Post::find($post->id))->toBeNull();
    expect(Post::count())->toBe(0);
});
```

## Event::fake() — Critical Pitfall

`Event::fake()` without arguments intercepts ALL events, including tenancy lifecycle events (`TenancyInitialized`, `BootstrapTenancy`, etc.). This prevents tenancy from initializing at all.

```php
// WRONG — breaks tenancy completely
Event::fake();
tenancy()->initialize($tenant); // silently fails

// CORRECT — only fake specific events
Event::fake([PostCreated::class]);
tenancy()->initialize($tenant); // works
```

This is the single most common testing pitfall with stancl/tenancy.

## HTTP Tests with Tenant Context

For feature tests that hit tenant routes:

```php
it('shows dashboard for tenant', function () {
    $tenant = Tenant::create();
    $tenant->domains()->create(['domain' => 'foo.localhost']);

    $this->get('http://foo.localhost/dashboard')
        ->assertOk();
});
```

For path-based identification:

```php
$this->get("/foo/dashboard") // where 'foo' is the tenant identifier
    ->assertOk();
```

## Multi-Database Testing Limitations

- `:memory:` SQLite is NOT compatible — each tenant needs a real database file or server
- `RefreshDatabase` trait does not work for tenant databases
- Create a dedicated `TenantTestCase` that handles database creation and cleanup
- Consider using SQLite file-based databases for faster test execution

## Seeding in Tests

Initialize tenant before running factories or seeders:

```php
beforeEach(function () {
    $this->tenant = Tenant::create();
    tenancy()->initialize($this->tenant);

    // Factories now automatically get tenant_id
    $this->user = User::factory()->create();
    $this->post = Post::factory()->count(3)->create();
});
```

## Testing Jobs

Verify tenant context is preserved across job dispatching:

```php
it('preserves tenant context in queued jobs', function () {
    tenancy()->initialize($tenant);

    ProcessPost::dispatch($post);

    // Process the queued job
    $this->artisan('queue:work', ['--once' => true]);

    // Verify job ran in correct tenant context
    expect($post->fresh()->processed)->toBeTrue();
});
```

## Testing Without Tenancy

For tests that explicitly need to run without tenant context (admin panel, central routes):

```php
it('lists all tenants in admin panel', function () {
    Tenant::factory()->count(3)->create();

    // No tenancy()->initialize() — testing central context
    $this->get('/admin/tenants')
        ->assertOk()
        ->assertJsonCount(3, 'data');
});
```
