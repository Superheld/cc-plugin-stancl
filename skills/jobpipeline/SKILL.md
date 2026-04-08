---
name: jobpipeline
description: "This skill should be used when the user asks to 'create a job pipeline', 'chain jobs as event listener', 'use JobPipeline::make', 'convert jobs to event listeners', 'set up queued event pipeline', or works with stancl/jobpipeline. Also activates when defining event-to-job mappings in EventServiceProvider or TenancyServiceProvider, debugging pipeline execution order, or handling pipeline failures."
---

# stancl/jobpipeline — Turn Jobs into Event Listeners

stancl/jobpipeline is a single-class package that converts any series of Laravel jobs into an event listener. Define jobs, extract data from the event, optionally queue the pipeline, and register it as a listener. The entire package is one class: `JobPipeline`.

## Core Pattern

```php
use Stancl\JobPipeline\JobPipeline;
use Illuminate\Support\Facades\Event;

Event::listen(OrderPlaced::class, JobPipeline::make([
    ValidateOrder::class,
    ChargePayment::class,
    SendConfirmation::class,
])->send(function (OrderPlaced $event) {
    return $event->order;
})->shouldBeQueued(false)->toListener());
```

## API

### `make(array $jobs): static`

Create a pipeline with an array of job class names or closures.

### `send(callable $callback): static`

Define how to extract data from the event. The callback receives the event arguments and returns data spread into each job's constructor. Return an array for multiple constructor arguments.

```php
->send(function (SomeEvent $event) {
    return [$event->model, $event->config]; // spread as constructor args
})
```

Omitting `send()` passes the event object itself to each job's constructor.

### `shouldBeQueued(bool $queued = true, ?string $queue = null): static`

Toggle async execution. Default is `false` (synchronous). Optionally target a specific queue (v2.x):

```php
->shouldBeQueued(true, 'emails')
// or
->shouldBeQueued(queue: 'high-priority')
```

### `toListener(): Closure`

Return a closure suitable for `Event::listen()` or `EventServiceProvider::$listen`.

## How It Works

1. `toListener()` returns a closure registered as event listener
2. When the event fires, `send()` callback extracts data synchronously
3. The pipeline is either dispatched to queue or executed inline
4. `handle()` iterates jobs sequentially — each job is instantiated with the extracted data as constructor args, then `handle()` is called via the container (enabling method injection)

**Constructor args come from the pipeline. `handle()` args come from the container.**

```php
class MigrateDatabase
{
    public function __construct(protected Tenant $tenant) {}  // from pipeline

    public function handle(DatabaseManager $db)  // from container
    {
        // ...
    }
}
```

## Pipeline Cancellation

Any job returning `false` from `handle()` stops execution of all subsequent jobs:

```php
class ValidateOrder
{
    public function handle(): bool
    {
        if ($this->order->isInvalid()) {
            return false; // ChargePayment, SendConfirmation will NOT run
        }
        return true;
    }
}
```

## Error Handling

If a job class defines a `failed(Throwable $e)` method, the pipeline catches exceptions and calls it. The pipeline then stops. Without `failed()`, exceptions propagate normally.

```php
class ChargePayment
{
    public function handle() { /* may throw */ }

    public function failed(\Throwable $e)
    {
        Log::error('Payment failed: ' . $e->getMessage());
    }
}
```

## Usage in EventServiceProvider

```php
protected $listen = [
    TenantCreated::class => [
        JobPipeline::make([
            CreateDatabase::class,
            MigrateDatabase::class,
            SeedDatabase::class,
        ])->send(fn (TenantCreated $e) => $e->tenant)
          ->shouldBeQueued(false)
          ->toListener(),
    ],
];
```

## Closures as Jobs

Closures work as pipeline steps for simple operations. They receive the passable data and support container injection:

```php
JobPipeline::make([
    function (Order $order) {
        $order->markAsProcessed();
    },
])->send(fn (OrderPlaced $e) => $e->order)->toListener()
```

Closures cannot be queued — they are not serializable. Use class-based jobs for queued pipelines.

## Global Default

Change the default queuing behavior for all pipelines:

```php
// In a service provider boot()
JobPipeline::$shouldBeQueuedByDefault = true;
```

## Critical Rules

1. **Sequential execution** — jobs run in order within a single pipeline; the pipeline is the queued unit, not individual jobs
2. **No closures in queued pipelines** — closures are not serializable
3. **`send()` runs synchronously** — data extraction happens before queue dispatch, even for queued pipelines
4. **`return false` cancels** — any job returning false stops the pipeline
5. **Method injection in `handle()`** — container resolves type-hinted parameters in job `handle()` methods

## Version Compatibility

| Version | Laravel | PHP | Notes |
|---|---|---|---|
| 1.x (stable) | 10, 11, 12, 13 | ^8.0 | `composer require stancl/jobpipeline` |
| 2.x (RC) | 10, 11, 12, 13 | ^8.0 | Adds queue name parameter, `static` return types |
