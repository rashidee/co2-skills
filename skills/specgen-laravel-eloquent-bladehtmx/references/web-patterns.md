# Web Application Patterns — Layouts, Components, and Frontend

This reference describes the web-specific patterns for the server-rendered monolith.
Include this content in Sections 7-12 of the generated specification.

---

## Architecture Overview

The web application uses a layered composition model:

```
Request → Middleware (Auth, CSP Nonce, Theme) → Controller
    → Service (queries DB via Eloquent Model / Repository)
    → View Model (spatie/laravel-view-models — encapsulates all template data)
    → Blade Template (renders HTML using Tailwind CSS)
    → Response (full page or htmx fragment)
```

Blade templates are organized following Laravel conventions with module separation:

- `resources/views/layouts/` — Layouts (app, auth, error)
- `resources/views/partials/` — Shared partials (header, sidebar, footer)
- `resources/views/components/` — Reusable Blade components
- `Modules/<Module>/resources/views/pages/` — Module page templates
- `Modules/<Module>/resources/views/fragments/` — Module htmx fragment templates

---

## View Model Pattern

Every Blade page receives a strongly-typed view model via `spatie/laravel-view-models`.
Never pass raw arrays or unstructured data to templates.

```php
namespace Modules\Order\ViewModels;

use Spatie\ViewModels\ViewModel;

class OrderListViewModel extends ViewModel
{
    public function __construct(
        public readonly \Illuminate\Pagination\LengthAwarePaginator $items,
        public readonly string $theme,
        public readonly array $user,
    ) {}
}
```

Usage in controller:
```php
public function index(Request $request): View
{
    $items = $this->orderService->list($request->integer('page', 1));
    return (new OrderListViewModel(
        items: $items,
        theme: $request->cookie('theme', 'light'),
        user: auth()->user(),
    ))->view('order::pages.index');
}
```

---

## Page Composition Flow

1. Request hits middleware (auth, CSP nonce, theme injection)
2. Controller calls service to get data
3. Controller builds view model with all needed data
4. View model resolves to Blade view
5. Blade layout wraps the page content via `@extends` or component slots
6. Blade page renders view model data using shared components

```
<x-app-layout :title="$title" :theme="$theme">
  ├── @include('partials.header')
  ├── @include('partials.sidebar', ['navItems' => $navItems])
  ├── <main> content area
  │     └── {{ $slot }}  ← page body injected via component slot
  │           └── pages/index.blade.php
  │                 └── <x-pagination :paginator="$items" />
  │                 └── @include('order::fragments.order-row', ['item' => $item])
  ├── @include('partials.footer')
  └── <x-toast-container />
</x-app-layout>
```

---

## Tailwind CSS Component Strategy

All UI components use pure Tailwind CSS utility classes. No DaisyUI or other component
framework is used.

### Design Tokens via CSS Custom Properties

Define a semantic color system using CSS custom properties that Tailwind references:

```css
/* themes/light.css */
[data-theme="light"] {
  --color-bg-primary: theme('colors.white');
  --color-bg-secondary: theme('colors.gray.50');
  --color-text-primary: theme('colors.gray.900');
  --color-text-secondary: theme('colors.gray.500');
  --color-border: theme('colors.gray.200');
  --color-accent: theme('colors.blue.600');
}

/* themes/dark.css */
[data-theme="dark"] {
  --color-bg-primary: theme('colors.gray.900');
  --color-bg-secondary: theme('colors.gray.800');
  --color-text-primary: theme('colors.gray.100');
  --color-text-secondary: theme('colors.gray.400');
  --color-border: theme('colors.gray.700');
  --color-accent: theme('colors.blue.400');
}
```

### Component Convention

Each Tailwind component follows this pattern:
- Anonymous Blade component (`resources/views/components/`) defining the component's props
- Blade template using Tailwind utility classes with `dark:` variants
- Conditional styling via Blade expressions (match on type/tone/variant)

Example pattern:
```blade
{{-- resources/views/components/alert.blade.php --}}
@props(['type' => 'info', 'message'])

@php
$colors = match($type) {
    'success' => 'bg-green-50 text-green-800 dark:bg-green-900/20 dark:text-green-400',
    'error' => 'bg-red-50 text-red-800 dark:bg-red-900/20 dark:text-red-400',
    default => 'bg-blue-50 text-blue-800 dark:bg-blue-900/20 dark:text-blue-400',
};
@endphp

<div class="p-4 rounded-lg border {{ $colors }}" role="alert">
    <span class="text-sm font-medium">{{ $message }}</span>
</div>
```

Usage:
```blade
<x-alert type="success" message="Order saved successfully" />
```

---

## htmx Integration Pattern

For partial page updates, controllers return Blade fragment views:

```php
// Fragment controller method
public function search(Request $request): View
{
    $results = $this->orderService->search($request->input('q'));
    return view('order::fragments.search-results', ['items' => $results]);
}
```

```blade
{{-- In the page template --}}
<input type="search" name="q"
       hx-get="{{ route('order.fragments.search') }}"
       hx-target="#results"
       hx-trigger="keyup changed delay:300ms" />
<div id="results">
    @each('order::fragments.order-row', $items, 'item')
</div>
```

### htmx Request Detection

Detect htmx requests to return fragments vs full pages:

```php
// In a controller or middleware
if ($request->header('HX-Request')) {
    return view('order::fragments.order-row', ['item' => $order]);
}
return view('order::pages.show', new OrderDetailViewModel($order));
```

### htmx Error Handling

Global error handler detects htmx requests and returns toast triggers instead of error pages:

```php
// In app/Exceptions/Handler.php
public function render($request, Throwable $e)
{
    if ($request->header('HX-Request')) {
        return response('', $e->getCode() ?: 500)
            ->header('HX-Trigger', json_encode([
                'showToast' => [
                    'type' => 'error',
                    'message' => $e->getMessage(),
                ],
            ]));
    }
    return parent::render($request, $e);
}
```

---

## Asset Resolution

Laravel's Vite plugin handles manifest resolution automatically. Templates include assets
via Blade directives:

```blade
@vite(['resources/css/app.css', 'resources/js/app.js'])
```

In development, Vite's dev server provides HMR. In production, `@vite` reads the manifest
and generates hashed URLs automatically.

For CSP nonce integration:
```blade
@vite(['resources/css/app.css', 'resources/js/app.js'])
{{-- Vite automatically includes nonce when configured --}}
```

Configure in `app/Providers/AppServiceProvider.php`:
```php
use Illuminate\Support\Facades\Vite;

public function boot(): void
{
    Vite::useCspNonce();
}
```

---

## Pagination Contract

All list endpoints must be paginated. The pagination flow:

1. Controller calls `->paginate($perPage)` on Eloquent query
2. `LengthAwarePaginator` is returned with data + pagination metadata
3. Blade template renders the custom pagination component

Default page size: 20. Maximum page size: 100.

```php
// In controller
$items = $this->service->list(
    page: $request->integer('page', 1),
    perPage: min($request->integer('per_page', 20), 100),
);

// In Blade
<x-pagination :paginator="$items" />
```

---

## Form Request Validation

Each form submission uses a dedicated Form Request class for validation:

```php
namespace Modules\Order\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StoreOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true; // Authorization handled by middleware
    }

    public function rules(): array
    {
        return [
            'customer_id' => ['required', 'string', 'max:36'],
            'items' => ['required', 'array', 'min:1'],
            'items.*.product_id' => ['required', 'string'],
            'items.*.quantity' => ['required', 'integer', 'min:1'],
        ];
    }
}
```

Validation errors are automatically available in Blade via `$errors`:
```blade
<x-form-control label="Customer" name="customer_id" :required="true">
    <x-input name="customer_id" :value="old('customer_id')" />
    @error('customer_id')
        <p class="text-xs text-red-500 mt-1">{{ $message }}</p>
    @enderror
</x-form-control>
```
