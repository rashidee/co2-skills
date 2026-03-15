# Web Application Patterns — Layouts, Components, and Frontend

This reference describes the web-specific patterns for the server-rendered monolith.
Include this content in Sections 7-12 of the generated specification.

---

## Architecture Overview

The web application uses a layered composition model:

```
Request → SecurityFilter → CspNonceFilter → @Controller
    → Service (queries MongoDB via Repository)
    → View Model (encapsulates all template data)
    → JTE Template (renders HTML using Tailwind CSS)
    → Response (full page or htmx fragment)
```

JTE templates are organized to mirror Java package names:
- `src/main/jte/{{BASE_PACKAGE_PATH}}/shared/layout/` — Layouts
- `src/main/jte/{{BASE_PACKAGE_PATH}}/shared/fragment/` — Shared fragments
- `src/main/jte/{{BASE_PACKAGE_PATH}}/shared/component/` — Reusable components
- `src/main/jte/{{BASE_PACKAGE_PATH}}/{{module}}/page/` — Module page templates
- `src/main/jte/{{BASE_PACKAGE_PATH}}/{{module}}/fragment/` — Module fragment templates

---

## View Model Pattern

Every JTE template receives a strongly-typed view model. Never pass raw `Map` objects
or unstructured data to templates.

```java
// View models are records or classes with all data the template needs
public record OrderListView(
    List<OrderDTO> items,
    Pagination pagination,
    String theme,
    UserProfile user
) {}
```

Layouts are also view models:

```java
@Getter @Setter
public class MainLayout {
    private final String title;
    private final List<NavItem> navItems;
    private final UserProfile user;
    private final String theme;
    private final String cspNonce;
    private final ViteManifestService vite;

    public static MainLayout defaultPage(String title, HttpServletRequest req,
            ViteManifestService vite, List<NavItem> navItems) {
        // Extract theme from cookie, user from JWT, nonce from request attribute
    }
}
```

---

## Page Composition Flow

1. Controller extracts pagination, theme, and user from request
2. Controller calls service to get data
3. Controller builds view model with all needed data
4. Controller adds layout and view to Spring Model
5. JTE layout wraps the page content
6. JTE page renders view model data using shared components

```
MainLayout.jte (@param Content content)
  ├── HeaderFragment.jte (@template. call)
  ├── SidebarFragment.jte (@template. call)
  ├── <main> content area
  │     └── ${content}  ← page body injected via content = @`...`
  │           └── OrderListPage.jte
  │                 └── Pagination.jte (@template. call)
  │                 └── OrderListFragment.jte (@template. call)
  │                 └── Pagination.jte (@template. call)
  ├── FooterFragment.jte (@template. call, inside <main>)
  └── ToastContainer.jte (@template. call)
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
- Java view model (record or class) defining the component's data contract
- JTE template using Tailwind utility classes with `dark:` variants
- Conditional styling via JTE expressions (switch on type/tone/variant)

Example pattern:
```java
// Java view model
public record Alert(String type, String message) {}
```
```jte
// JTE template — all styling is Tailwind utilities
@param Alert alert
!{var colors = switch(alert.type()) {
    case "success" -> "bg-green-50 text-green-800 dark:bg-green-900/20 dark:text-green-400";
    case "error" -> "bg-red-50 text-red-800 dark:bg-red-900/20 dark:text-red-400";
    default -> "bg-blue-50 text-blue-800 dark:bg-blue-900/20 dark:text-blue-400";
};}
<div class="p-4 rounded-lg border ${colors}" role="alert">
  <span class="text-sm font-medium">${alert.message()}</span>
</div>
```

---

## htmx Integration Pattern

For partial page updates, controllers return HTML fragments:

```java
// Fragment controller
@GetMapping("/search")
String search(@RequestParam String q, Model model) {
    var results = service.search(q);
    model.addAttribute("items", results);
    return "order/fragment/OrderSearchResults";
}
```

```jte
// In the page template — use @template. syntax, never @include()
<input type="search" name="q"
       hx-get="/orders/fragments/search"
       hx-target="#results"
       hx-trigger="keyup changed delay:300ms" />
<div id="results">
  @for(var item : view.items())
    @template.{{BASE_PACKAGE_PATH_DOT}}.order.fragment.OrderRowFragment(item = item)
  @endfor
</div>
```

### htmx Error Handling

Global error handler detects htmx requests and returns toast triggers instead of error pages:

```java
if ("true".equals(req.getHeader("HX-Request"))) {
    return ResponseEntity.status(ex.getStatus())
        .header("HX-Trigger", "{\"showToast\":{\"type\":\"error\",\"message\":\"" + ex.getUserMessage() + "\"}}")
        .body("");
}
```

---

## Asset Resolution

`ViteManifestService` reads Vite's `manifest.json` to resolve hashed asset filenames
for production. In development mode, assets are served from Vite's dev server.

Templates include assets via the service:
```jte
<link rel="stylesheet" href="${layout.getVite().resolveAsset("css/app.css")}" />
<script src="${layout.getVite().resolveAsset("js/app.js")}" nonce="${layout.getCspNonce()}" defer></script>
```

---

## Pagination Contract

All list endpoints must be paginated. The pagination flow:

1. Controller implements `PaginationAware` to parse `page` and `size` query params
2. Service returns `PaginatedResult<T>` wrapping data + pagination metadata
3. `PaginatedResult.toPagination(baseHref)` creates the `Pagination` view model
4. JTE template renders `Pagination` component above and below the listing

Default page size: 20. Maximum page size: 100.
