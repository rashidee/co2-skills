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

---

## Sidebar Fragment Pattern

The sidebar fragment renders navigation items using direct module URLs. **Never use
`/api/content/...` prefixed paths** — this pattern creates a dependency on a non-existent
content API and causes navigation failures.

Each sidebar link uses `hx-get` with the direct module URL and `hx-select="#content-area"`
to extract just the content area from the full page response. This allows HTMX to swap
the content without a full page reload.

```jte
@import {{BASE_PACKAGE}}.shared.web.NavItem
@import java.util.List

@param List<NavItem> navItems

<aside class="fixed top-16 left-0 bottom-0 w-64 bg-white dark:bg-slate-800 border-r border-slate-200 dark:border-slate-700 overflow-y-auto z-30">
    <nav class="p-4 space-y-1">
        @for(var item : navItems)
            <%-- DO NOT use /api/content/ prefix — use direct module URL --%>
            <a href="${item.getHref()}"
               hx-get="${item.getHref()}"
               hx-target="#content-area"
               hx-select="#content-area"
               hx-swap="innerHTML"
               hx-push-url="true"
               class="${item.isActive()
                   ? "flex items-center gap-3 px-3 py-2 rounded-lg bg-blue-50 dark:bg-blue-900/20 text-blue-700 dark:text-blue-400 font-medium"
                   : "flex items-center gap-3 px-3 py-2 rounded-lg text-slate-600 dark:text-slate-400 hover:bg-slate-100 dark:hover:bg-slate-700"}">
                <span class="text-sm">${item.getLabel()}</span>
            </a>
        @endfor
    </nav>
</aside>
```

---

## Header Fragment Pattern

The header fragment renders the top bar with logo, notifications, and user menu. All
navigation links use direct URLs, NOT `/api/content/...` prefixed paths.

**Do NOT include a locale/language switcher** unless PRD.md has i18n requirements.
Non-functional UI elements confuse users and waste development effort.

```jte
@import {{BASE_PACKAGE}}.shared.web.MainLayoutViewModel

@param MainLayoutViewModel layout

<header class="fixed top-0 left-0 right-0 z-40 h-16 bg-white dark:bg-slate-800 border-b border-slate-200 dark:border-slate-700 flex items-center justify-between px-6"
        x-data="{ notifOpen: false, userOpen: false }">
    <!-- Left: Logo & App Name -->
    <div class="flex items-center gap-3">
        <%-- DO NOT use /api/content/home — use direct URL --%>
        <a href="/home"
           hx-get="/home"
           hx-target="#content-area"
           hx-select="#content-area"
           hx-swap="innerHTML"
           hx-push-url="true"
           class="flex items-center gap-3">
            <div class="w-9 h-9 rounded-lg bg-gradient-to-br from-primary to-secondary flex items-center justify-center shadow-sm">
                <!-- App icon SVG -->
            </div>
            <span class="text-lg font-semibold text-slate-800 dark:text-white tracking-tight">{{APP_NAME}}</span>
        </a>
    </div>

    <!-- Right: Actions -->
    <div class="flex items-center gap-2">
        <!-- Dark Mode Toggle — MUST use $store.theme, NOT local variables -->
        <button @click="$store.theme.toggle()"
                class="p-2 rounded-lg text-slate-500 dark:text-slate-400 hover:bg-slate-100 dark:hover:bg-slate-700 transition-colors">
            <!-- Sun icon (shown in dark mode) -->
            <svg x-show="$store.theme.dark" class="w-5 h-5" ...><!-- sun --></svg>
            <!-- Moon icon (shown in light mode) -->
            <svg x-show="!$store.theme.dark" class="w-5 h-5" ...><!-- moon --></svg>
        </button>

        <!-- Notification Bell -->
        <div class="relative" @click.away="notifOpen = false">
            <button @click="notifOpen = !notifOpen" class="relative p-2 rounded-lg ...">
                <!-- Bell icon SVG -->
            </button>
            <div x-show="notifOpen" class="absolute right-0 mt-2 w-80 bg-white dark:bg-slate-800 rounded-xl shadow-lg ...">
                <!-- Notification dropdown content -->
                <div class="px-4 py-2.5 border-t">
                    <%-- DO NOT use /api/content/notification — use direct URL --%>
                    <a href="/notifications"
                       hx-get="/notifications"
                       hx-target="#content-area"
                       hx-select="#content-area"
                       hx-swap="innerHTML"
                       hx-push-url="true"
                       class="text-sm text-blue-600 hover:text-blue-700 font-medium">
                        View All Notifications
                    </a>
                </div>
            </div>
        </div>

        <!-- User Menu -->
        <div class="relative" @click.away="userOpen = false">
            <button @click="userOpen = !userOpen" class="flex items-center gap-2 ...">
                <span class="text-sm font-medium">${layout.getUser().fullName()}</span>
            </button>
            <div x-show="userOpen" class="absolute right-0 mt-2 w-48 bg-white dark:bg-slate-800 rounded-xl shadow-lg ...">
                <%-- DO NOT use /api/content/profile — use direct URL --%>
                <a href="/profile"
                   hx-get="/profile"
                   hx-target="#content-area"
                   hx-select="#content-area"
                   hx-swap="innerHTML"
                   hx-push-url="true"
                   class="block px-4 py-2 text-sm ...">My Profile</a>
                <a href="/logout" class="block px-4 py-2 text-sm ...">Logout</a>
            </div>
        </div>

        <%-- NO locale dropdown — only include if PRD.md specifies i18n requirements --%>
    </div>
</header>
```

---

## Theme Switching Pattern

Theme switching uses an Alpine.js **global store** (`$store.theme`) that persists the
preference in a cookie and toggles the `dark` class on `<html>`.

**Critical:** Do NOT use local Alpine.js `x-data` variables (`darkMode`, `toggleDark`)
for theme state. Alpine.js v3 nested `x-data` components are separate scopes — a local
variable in `<html x-data="appShell()">` cannot be accessed from a nested
`<header x-data="{ ... }">`. This causes the dark mode toggle to silently fail.

### Alpine Store (in app.js)

```javascript
import Alpine from 'alpinejs';

Alpine.store('theme', {
    dark: document.cookie.includes('theme=dark'),
    toggle() {
        this.dark = !this.dark;
        document.documentElement.classList.toggle('dark', this.dark);
        document.cookie = `theme=${this.dark ? 'dark' : 'light'};path=/;max-age=31536000`;
    },
    apply() {
        document.documentElement.classList.toggle('dark', this.dark);
    }
});

// Apply theme on load
document.addEventListener('alpine:init', () => {
    Alpine.store('theme').apply();
});
```

### MainLayout.jte

```html
<%-- Use $store.theme.dark, NOT a local darkMode variable --%>
<html lang="en" x-data :class="{ 'dark': $store.theme.dark }">
```

### Header Toggle Button

```html
<%-- Use $store.theme.toggle(), NOT a local toggleDark() function --%>
<button @click="$store.theme.toggle()" ...>
    <svg x-show="$store.theme.dark" ...><!-- sun icon --></svg>
    <svg x-show="!$store.theme.dark" ...><!-- moon icon --></svg>
</button>
```

---

## Layout — Footer Positioning

When the sidebar uses `position: fixed`, the `<main>` element must have its own
`min-height` to ensure the footer stays at the bottom of the viewport even when content
is short. Without this, the footer floats up because the fixed sidebar is out of the
normal document flow.

```html
<div class="flex pt-16 min-h-[calc(100vh-4rem)]">
    <!-- Sidebar is position:fixed — it's out of flow -->
    @template.fragment.Sidebar(layout = layout)

    <!-- min-h required on <main> because sidebar is position:fixed (out of flow) -->
    <main class="flex-1 ml-64 flex flex-col min-w-0 min-h-[calc(100vh-4rem)]">
        <div id="content-area" class="flex-1 flex flex-col">
            ${content}
        </div>
        @template.fragment.Footer()
    </main>
</div>
```

Key points:
- `min-h-[calc(100vh-4rem)]` on BOTH the wrapper div AND `<main>`
- `flex-col` on `<main>` with `flex-1` on `#content-area` pushes footer to bottom
- Footer is INSIDE `<main>`, not outside, so it scrolls with content

---

## Role-Based Navigation Filtering

The `LayoutService` (or equivalent) that builds sidebar navigation items MUST filter
by the current user's roles. The role-to-menu mapping is derived from the mockup
sidebar files — each role folder defines which menu items that role sees.

```java
package {{BASE_PACKAGE}}.shared.web;

import {{BASE_PACKAGE}}.shared.security.Roles;
import {{BASE_PACKAGE}}.shared.security.SecurityContextUtil;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;

@Service
@RequiredArgsConstructor
public class LayoutService {

    private final SecurityContextUtil securityContextUtil;

    public List<NavItem> buildNavItems(String currentPath) {
        Set<String> userRoles = securityContextUtil.getCurrentRoles();
        List<NavItem> items = new ArrayList<>();

        // Home — visible to all authenticated users
        items.add(NavItem.builder()
            .label("Home").href("/home").section("")
            .active("/home".equals(currentPath))
            .build());

        // Role-specific menu items
        // Derive from mockup sidebar files: mockup/<role>/sidebar-<role>.html

        if (userRoles.contains(Roles.HUB_ADMINISTRATOR)) {
            // System / Configuration items for admin
            // e.g., Audit Trail, Corridor, Recruitment Step
        }

        if (userRoles.contains(Roles.HUB_OPERATION_SUPPORT)) {
            // Operations items for support
            // e.g., Employer, Recruitment Agent, Job Demand, etc.
        }

        return items;
    }
}
```

**Never show all menu items to all users.** Even if the backend enforces access control
via `@PreAuthorize`, showing menu items the user cannot access creates a confusing UX
and may expose unauthorized functionality.

---

## JSON Display Pattern

When rendering raw JSON content (e.g., message payloads, audit trail changes, API
responses), use Alpine.js to pretty-print on initialization. This provides a readable,
well-formatted display without requiring server-side formatting.

```html
<!-- For raw JSON display, use Alpine.js to pretty-print on init -->
<pre class="bg-slate-50 dark:bg-slate-800 p-4 rounded-lg overflow-auto text-sm font-mono max-h-96">
    <code x-data x-init="try { $el.textContent = JSON.stringify(JSON.parse($el.textContent), null, 2) } catch(e) {}">
        ${rawJsonContent}
    </code>
</pre>
```

Key points:
- `x-init` runs once when the element is initialized
- `JSON.parse` + `JSON.stringify(..., null, 2)` produces indented output
- `try/catch` gracefully handles non-JSON content (falls back to raw text)
- Use `$el.textContent` (not `innerHTML`) to avoid XSS and HTML entity issues
- For `<pre><code>` pattern: format the `<code>` element's textContent
- For `<pre>` only pattern: format the `<pre>` element's textContent directly
