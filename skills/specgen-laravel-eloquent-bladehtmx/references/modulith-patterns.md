# Laravel Modules Patterns — Module Structure & Event Architecture

This reference describes the nwidart/laravel-modules conventions used in the spec. Include
this content in Sections 5 and 19 of the generated specification.

---

## Module Boundary Rules

nwidart/laravel-modules enforces modular boundaries by treating each module as an
independent package with its own namespace, routes, views, migrations, and service
providers. This spec uses a strict encapsulation model: public API via contracts and
DTOs, all implementation internal to the module.

### What constitutes a module

```
app/
├── Http/Middleware/         ← NOT a module (shared middleware)
├── Services/               ← NOT a module (shared services)
├── Models/                 ← NOT a module (shared base models like User)
├── Traits/                 ← NOT a module (shared traits)
├── Constants/              ← NOT a module (shared constants)
├── Exceptions/             ← NOT a module (shared exceptions)
└── Providers/              ← NOT a module (shared service providers)

Modules/
├── Order/                  ← MODULE: Order bounded context
├── Inventory/              ← MODULE: Inventory bounded context
├── Scheduling/             ← MODULE: Scheduling bounded context
└── ...
```

Each module is generated via `php artisan module:make <ModuleName>` and follows the
nwidart/laravel-modules directory convention.

### Visibility rules — Public API vs Internal Implementation

Each module has a strict two-layer structure:

```
Modules/Order/                                      ← MODULE ROOT
├── Contracts/                                      ← PUBLIC API (visible to other modules)
│   └── OrderServiceInterface.php                   ← Service interface
├── DTOs/                                           ← PUBLIC API
│   └── OrderData.php                               ← spatie/laravel-data DTO
├── Events/                                         ← PUBLIC API
│   └── OrderCreatedEvent.php                       ← Domain event
├── Exceptions/                                     ← PUBLIC API
│   └── OrderException.php                          ← Module-specific exception
│
├── Services/                                       ← INTERNAL
│   └── OrderService.php                            ← Implements OrderServiceInterface
├── Models/                                         ← INTERNAL
│   └── Order.php                                   ← Eloquent model
├── Repositories/                                   ← INTERNAL (optional)
│   └── OrderRepository.php                         ← Query abstraction
├── Http/                                           ← INTERNAL
│   ├── Controllers/
│   │   ├── OrderPageController.php                 ← Page controller (full HTML)
│   │   └── OrderFragmentController.php             ← Fragment controller (htmx)
│   └── Requests/
│       ├── StoreOrderRequest.php                   ← Form validation
│       └── UpdateOrderRequest.php
├── ViewModels/                                     ← INTERNAL
│   ├── OrderListViewModel.php                      ← List page view model
│   └── OrderDetailViewModel.php                    ← Detail page view model
├── database/
│   └── migrations/                                 ← INTERNAL
│       └── 2024_01_01_000000_create_orders_table.php
├── resources/
│   └── views/                                      ← INTERNAL
│       ├── pages/
│       │   ├── index.blade.php                     ← List page
│       │   ├── show.blade.php                      ← Detail page
│       │   ├── create.blade.php                    ← Create form
│       │   └── edit.blade.php                      ← Edit form
│       └── fragments/
│           ├── order-row.blade.php                 ← Table row fragment
│           └── search-results.blade.php            ← Search results fragment
├── routes/
│   └── web.php                                     ← Module routes
├── Config/
│   └── config.php                                  ← Module config
├── Providers/
│   └── OrderServiceProvider.php                    ← Module service provider
└── Tests/
    ├── Feature/
    └── Unit/
```

**Public API (Contracts, DTOs, Events, Exceptions)** — only these types are visible to
other modules:
- `OrderServiceInterface` — Interface. The sole programmatic entry point for other modules.
  Other modules inject this interface; Laravel wires the internal `OrderService` via the
  module's service provider.
- `OrderData` — The data contract. When `OrderServiceInterface` returns data, it returns
  this DTO, never the internal `Order` Eloquent model. This prevents leaking persistence
  details.
- `OrderException` — Callers can catch module-specific failures.
- `OrderCreatedEvent` — Domain events must be in the public namespace so listeners in
  other modules can reference the event type.

**Internal implementation** — invisible to other modules by convention:
- `OrderService` — Contains all business logic, Eloquent queries, event dispatching.
- `Order` — The Eloquent model is internal. It may have casts, relationships, or scopes
  that are irrelevant to consumers.
- `OrderRepository` — Data access abstraction (optional — some teams query directly in
  the service).
- `OrderPageController` — The controller is an **inbound adapter**. Other modules never
  call controllers.
- `OrderListViewModel` — View models are internal to the module.

### Module Service Provider Binding

Each module's service provider binds the interface to the implementation:

```php
namespace Modules\Order\Providers;

use Illuminate\Support\ServiceProvider;
use Modules\Order\Contracts\OrderServiceInterface;
use Modules\Order\Services\OrderService;

class OrderServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(OrderServiceInterface::class, OrderService::class);
    }

    public function boot(): void
    {
        $this->loadMigrationsFrom(module_path('Order', 'database/migrations'));
        $this->loadViewsFrom(module_path('Order', 'resources/views'), 'order');
        $this->loadRoutesFrom(module_path('Order', 'routes/web.php'));
    }
}
```

### Inter-module dependency direction

```
OrderPageController (internal) → OrderServiceInterface (public contract)
                                        ↑
InventoryService (another module)  → OrderServiceInterface (public contract)
```

Both the internal controller and external modules depend on the same public interface.
The `OrderService` implementation is wired by Laravel's service container but never
directly referenced.

### What modules must NOT do

- Never inject a Service class from another module — use the Contract interface
- Never reference another module's `Models/`, `Services/`, or `Http/` namespace
- Never reference another module's Eloquent Model directly
- Never create circular dependencies between modules
- Never bypass the service interface by calling another module's controller

---

## Inter-Module Communication via Events

When one module needs to notify others about a domain event (e.g., an order was created
and inventory needs to adjust), use Laravel's event system.

### Event Class Convention

Events are plain PHP classes, immutable, and carry only the data the listener needs.

Naming: `<AggregateRoot><PastTenseVerb>Event`

```php
// In Modules/Order/Events/ (module public — visible to other modules)
namespace Modules\Order\Events;

use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderCreatedEvent
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public readonly string $orderId,
        public readonly string $customerId,
        public readonly \DateTimeImmutable $occurredAt,
    ) {}
}
```

Why readonly properties: they're immutable and make the event's contract explicit.
Using `DateTimeImmutable` establishes when the event happened.

### Publishing Events

The service implementation (internal) publishes events using Laravel's `event()` helper
or `Event` facade:

```php
// In Modules/Order/Services/ (INTERNAL)
namespace Modules\Order\Services;

use Modules\Order\Contracts\OrderServiceInterface;
use Modules\Order\DTOs\OrderData;
use Modules\Order\Events\OrderCreatedEvent;
use Modules\Order\Models\Order;

class OrderService implements OrderServiceInterface
{
    public function create(array $data): OrderData
    {
        $order = Order::create($data);

        event(new OrderCreatedEvent(
            orderId: $order->id,
            customerId: $order->customer_id,
            occurredAt: new \DateTimeImmutable(),
        ));

        return OrderData::from($order);
    }
}
```

### Listening to Events

Other modules listen using Laravel listeners in their service provider:

```php
// In Modules/Inventory/Listeners/ (INTERNAL)
namespace Modules\Inventory\Listeners;

use Illuminate\Contracts\Queue\ShouldQueue;
use Modules\Order\Events\OrderCreatedEvent;

class AdjustInventoryOnOrderCreated implements ShouldQueue
{
    public bool $afterCommit = true;

    public function handle(OrderCreatedEvent $event): void
    {
        // Business logic to reserve inventory
        logger()->info("Adjusting inventory for order: {$event->orderId}");
    }
}
```

Register in the module's `EventServiceProvider` or `boot()`:
```php
// In Modules/Inventory/Providers/InventoryServiceProvider.php
use Illuminate\Support\Facades\Event;
use Modules\Inventory\Listeners\AdjustInventoryOnOrderCreated;
use Modules\Order\Events\OrderCreatedEvent;

public function boot(): void
{
    Event::listen(OrderCreatedEvent::class, AdjustInventoryOnOrderCreated::class);
}
```

The listener references `OrderCreatedEvent` which is in the `Order` module's public
`Events/` namespace, so this cross-module reference is valid. The listener implementation
itself is internal to the `Inventory` module.

`ShouldQueue` + `$afterCommit = true` means:
- The listener runs in a separate queue job
- It executes after the publishing transaction commits
- Failures in the listener do not roll back the publisher

### Event Publication Persistence

For at-least-once delivery semantics, consider:
1. Using Laravel's queue system with database driver (failed jobs are retried)
2. Implementing a custom `event_publications` table for tracking delivery status
3. Using `spatie/laravel-event-sourcing` for full event sourcing

Configuration in `config/queue.php`:
```php
'connections' => [
    'database' => [
        'driver' => 'database',
        'table' => 'jobs',
        'queue' => 'default',
        'retry_after' => 90,
        'after_commit' => true,
    ],
],
```

---

## Module Verification

To verify module boundaries are respected, use `phpat/phpat` architecture tests:

```php
namespace Tests\Architecture;

use PHPat\Selector\Selector;
use PHPat\Test\Builder\Rule;
use PHPat\Test\PHPat;

class ModuleBoundaryTest
{
    public function test_modules_only_depend_on_public_api(): Rule
    {
        return PHPat::rule()
            ->classes(Selector::inNamespace('Modules\Inventory'))
            ->canOnlyDependOn()
            ->classes(
                Selector::inNamespace('Modules\Order\Contracts'),
                Selector::inNamespace('Modules\Order\DTOs'),
                Selector::inNamespace('Modules\Order\Events'),
                Selector::inNamespace('Modules\Order\Exceptions'),
                // ... other public namespaces
                Selector::inNamespace('Illuminate'),
                Selector::inNamespace('App'),
            );
    }
}
```

This test catches:
- Direct dependencies between modules that bypass contracts
- Circular module dependencies
- References to another module's internal types (Models, Services, Http)

Include this test in the Testing Strategy section of the spec. It should run as part
of CI to enforce architectural integrity continuously.

---

## Web Application Module Adaptations

For server-rendered web applications, each module includes `pages/` and `fragments/`
view directories, page and fragment controllers, and view models.

### Key Patterns

1. **No REST controllers**: Web modules use standard `Controller` returning Blade views,
   not JSON API responses.

2. **Direct data ownership**: Each module owns its Eloquent models and migrations.
   The service layer queries the database directly via Eloquent.

3. **View models**: Each page has a dedicated view model class (`spatie/laravel-view-models`)
   that encapsulates all data needed by the Blade template. View models are internal.

4. **Page vs Fragment separation**: `pages/` holds Blade views for full page renders;
   `fragments/` holds Blade views that return partial HTML for htmx requests.

5. **Form Requests**: Controllers use Form Request classes for validation. Validation
   rules are defined per-action (Store, Update, etc.).

### Web Page Controller Pattern

```php
namespace Modules\Order\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\View\View;
use Modules\Order\Contracts\OrderServiceInterface;
use Modules\Order\ViewModels\OrderListViewModel;
use Modules\Order\ViewModels\OrderDetailViewModel;

class OrderPageController extends Controller
{
    public function __construct(
        private readonly OrderServiceInterface $orderService,
    ) {}

    public function index(Request $request): View
    {
        $items = $this->orderService->list(
            page: $request->integer('page', 1),
            perPage: 20,
        );

        return (new OrderListViewModel(
            items: $items,
            theme: $request->cookie('theme', 'light'),
            user: auth()->user(),
        ))->view('order::pages.index');
    }

    public function show(string $id, Request $request): View
    {
        $order = $this->orderService->get($id);

        return (new OrderDetailViewModel(
            order: $order,
            theme: $request->cookie('theme', 'light'),
            user: auth()->user(),
        ))->view('order::pages.show');
    }
}
```

### Web Fragment Controller Pattern

```php
namespace Modules\Order\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Illuminate\View\View;
use Modules\Order\Contracts\OrderServiceInterface;

class OrderFragmentController extends Controller
{
    public function __construct(
        private readonly OrderServiceInterface $orderService,
    ) {}

    public function row(string $id): View
    {
        $order = $this->orderService->get($id);
        return view('order::fragments.order-row', ['item' => $order]);
    }

    public function search(Request $request): View
    {
        $results = $this->orderService->search($request->input('q', ''));
        return view('order::fragments.search-results', ['items' => $results]);
    }

    public function destroy(string $id): Response
    {
        $this->orderService->delete($id);
        return response('', 200);
    }
}
```

### Module Route File

```php
// Modules/Order/routes/web.php
use Illuminate\Support\Facades\Route;
use Modules\Order\Http\Controllers\OrderPageController;
use Modules\Order\Http\Controllers\OrderFragmentController;

Route::middleware(['auth', 'role:HUB_ADMINISTRATOR|HUB_OPERATION_SUPPORT'])
    ->group(function () {
        // Page routes
        Route::get('/orders', [OrderPageController::class, 'index'])->name('order.index');
        Route::get('/orders/create', [OrderPageController::class, 'create'])->name('order.create');
        Route::post('/orders', [OrderPageController::class, 'store'])->name('order.store');
        Route::get('/orders/{id}', [OrderPageController::class, 'show'])->name('order.show');
        Route::get('/orders/{id}/edit', [OrderPageController::class, 'edit'])->name('order.edit');
        Route::put('/orders/{id}', [OrderPageController::class, 'update'])->name('order.update');
        Route::delete('/orders/{id}', [OrderPageController::class, 'destroy'])->name('order.destroy');

        // Fragment routes (htmx)
        Route::get('/orders/fragments/search', [OrderFragmentController::class, 'search'])->name('order.fragments.search');
        Route::get('/orders/fragments/row/{id}', [OrderFragmentController::class, 'row'])->name('order.fragments.row');
        Route::delete('/orders/fragments/{id}', [OrderFragmentController::class, 'destroy'])->name('order.fragments.destroy');
    });
```
