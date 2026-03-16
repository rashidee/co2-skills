# Specification Template — Laravel Web Monolith

This is the authoritative template for the generated specification. The specification is
split into **two types of files**:

1. **`SPECIFICATION.md`** (root) — Table of Contents, shared infrastructure, and all
   application-level sections. Generated once per application.
2. **`<module>/SPEC.md`** (per-module) — Self-contained module blueprint.
   Generated once per module from PRD.md.

```
spec/
├── SPECIFICATION.md
├── location-information/
│   └── SPEC.md
├── corridor/
│   └── SPEC.md
└── ...
```

Placeholders use `{{VARIABLE}}` syntax and must be replaced with actual values gathered
from context files.

Sections marked **[CONDITIONAL]** should only be included when the corresponding
integration is selected. Within unconditional sections, some content blocks are
conditional — follow the inline guidance.

**CRITICAL: Sample code requirement.** Every component in the spec MUST include a complete,
continuous code sample in a fenced code block (PHP, Blade, JavaScript, YAML, or HTML).
The code must be self-explanatory and directly usable as a reference for a coding agent.
Do not describe components with bullet points alone — always accompany descriptions with
full code.

---

# Part A: Root SPECIFICATION.md

Everything from this point until "Part B" goes into the root `SPECIFICATION.md` file.
A coding agent implements these shared sections **first**, before any module work.

---

## Table of Contents

Generate a TOC with clickable Markdown anchor links to every H2 and H3 section in this
file. Additionally, include a **Modules** section in the TOC that links to each
module's `SPEC.md` file:

```markdown
## Table of Contents

### Shared Infrastructure
- [1. Project Overview](#1-project-overview)
- [2. Composer & npm Configuration](#2-composer--npm-configuration)
- ...all shared sections...

### Modules
- [Location Information](location-information/SPEC.md)
- [Corridor](corridor/SPEC.md)
- [Employer](employer/SPEC.md)
- ...(one link per module)...
```

Only include conditional sections that apply based on user selections.

---

## Section 1: Project Overview

```
# {{APPLICATION_NAME}} — Technical Specification

## 1. Project Overview

**Application Name**: {{APPLICATION_NAME}}   (from CLAUDE.md Custom Applications section)
**Project Slug**: {{PROJECT_SLUG}}           (kebab-case of application name)
**PHP Version**: 8.3+
**Laravel Version**: 12.x
**Description**: {{APP_DESCRIPTION}}         (from CLAUDE.md application description)
**Versions Covered**: v1.0.0 — v{{LATEST_VERSION}}

### Optional Components (Auto-Determined)
**Database**: {{DATABASE}}       (from CLAUDE.md dependencies)
**Authentication**: {{AUTH}}     (from CLAUDE.md dependencies + PRD.md NFRs)
**Scheduling**: {{SCHEDULING}}   (from PRD.md NFRs)
**Messaging**: {{MESSAGING}}     (from CLAUDE.md dependencies)

### Technology Stack
(Render the core stack version table from SKILL.md, plus selected optional integration versions)

### User Roles
(List each user role extracted from mockup sidebar files, e.g., Hub Administrator, Hub Operation Support.
Include a role-to-module mapping table showing which modules each role can access.
This mapping drives middleware('role:...') on routes — NOT URL path prefixes.)

| Role | Modules | Permission Role |
|------|---------|----------------|
| Hub Administrator | Location Information, Corridor, Recruitment Step, ... | `HUB_ADMINISTRATOR` |
| Hub Operation Support | Employer, Quota, Quota Allocation, ... | `HUB_OPERATION_SUPPORT` |

### Sidebar Navigation Items
(For each role, list the navigation items with module-based hrefs — NOT role-prefixed hrefs.)

**Hub Administrator:**
| Label | Href | Section |
|-------|------|---------|
| Location Information | `/location-information` | Geography |
| Corridor | `/corridor` | Administration |
| ... | ... | ... |

### Modules
(List each module from PRD.md organized by System Module and Business Module,
with description from the module heading and user story count from MODEL.md summary table.
Each module links to its dedicated SPEC.md file.)

| Module | Type | Stories | Versions | Spec |
|--------|------|---------|----------|------|
| Location Information | Business | 12 | 1.0.0, 1.0.1, 1.0.3 | [SPEC](location-information/SPEC.md) |
| Corridor | Business | 8 | 1.0.0, 1.0.3 | [SPEC](corridor/SPEC.md) |

### Input Sources
(List the context files read to produce this specification)
- CLAUDE.md: {{path}}
- PRD.md: {{path}}
- Module Model: {{path to MODEL.md}}
- Mockup Screens: {{path to MOCKUP.html}}
```

---

## Section 2: Composer & npm Configuration

### 2.1 composer.json

Generate a complete `composer.json` inside a code block. This section is critical — it must
be copy-pasteable and immediately functional.

Include:
- `"require"`:
  - **Core**: `laravel/framework`, `nwidart/laravel-modules`, `spatie/laravel-data`, `spatie/laravel-view-models`
  - **[If Database = MongoDB]** `mongodb/laravel-mongodb`
  - **[If Auth = Keycloak]** `laravel/socialite`, `socialiteproviders/keycloak`
  - **[If Auth = form]** `laravel/breeze`
  - **RBAC**: `spatie/laravel-permission`
  - **Audit**: `owen-it/laravel-auditing`
  - **Health**: `spatie/laravel-health`
  - **[If Messaging = yes]** `vladimir-yuldashev/laravel-queue-rabbitmq`, `php-amqplib/php-amqplib`
  - **[If Reporting = yes]** `barryvdh/laravel-dompdf`, `maatwebsite/excel`
- `"require-dev"`:
  - `phpunit/phpunit`, `phpat/phpat`, `laravel/pint`

### 2.2 package.json

```json
{
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build"
  },
  "devDependencies": {
    "laravel-vite-plugin": "^1.2",
    "vite": "^6.0",
    "tailwindcss": "^4.0",
    "@tailwindcss/vite": "^4.0",
    "autoprefixer": "^10.4",
    "postcss": "^8.4"
  },
  "dependencies": {
    "alpinejs": "^3.14",
    "htmx.org": "^2.0"
  }
}
```

---

## Section 3: Application Configuration

Generate complete configuration files.

**IMPORTANT: Environment variable externalization.** All configuration values that may
change between environments (ports, hostnames, credentials, URIs) MUST be externalized
to environment variables. Laravel's `env('VAR', 'default')` helper is used in config
files. The `.env` file provides local development defaults, while production deployments
override via system environment variables or `.env.production`.

### .env (default)

**Always include:**
```env
APP_NAME={{APPLICATION_NAME}}
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost:8000

LOG_CHANNEL=stack
LOG_LEVEL=debug
```

**[If Database = MongoDB] add:**
```env
DB_CONNECTION=mongodb
DB_URI=mongodb://localhost:27017/{{DB_NAME}}
DB_DATABASE={{DB_NAME}}
```

**[If Database = PostgreSQL] add:**
```env
DB_CONNECTION=pgsql
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE={{DB_NAME}}
DB_USERNAME={{DB_USER}}
DB_PASSWORD={{DB_PASSWORD}}
```

**[If Database = MySQL] add:**
```env
DB_CONNECTION=mysql
DB_HOST=localhost
DB_PORT=3306
DB_DATABASE={{DB_NAME}}
DB_USERNAME={{DB_USER}}
DB_PASSWORD={{DB_PASSWORD}}
```

**[If Auth = Keycloak] add:**
```env
KEYCLOAK_CLIENT_ID={{KEYCLOAK_CLIENT_ID}}
KEYCLOAK_CLIENT_SECRET={{KEYCLOAK_CLIENT_SECRET}}
KEYCLOAK_REALM={{KEYCLOAK_REALM}}
KEYCLOAK_BASE_URL=http://localhost:8180
KEYCLOAK_REDIRECT_URI=${APP_URL}/auth/callback
```

**[If Messaging = yes] add:**
```env
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USER=guest
RABBITMQ_PASSWORD=guest
RABBITMQ_VHOST=/
RABBITMQ_QUEUE=default
```

### config/database.php

**[If Database = MongoDB] add connection:**
```php
'mongodb' => [
    'driver' => 'mongodb',
    'dsn' => env('DB_URI', 'mongodb://localhost:27017'),
    'database' => env('DB_DATABASE', '{{DB_NAME}}'),
],
```

**[If Database = PostgreSQL] add connection:**
```php
'pgsql' => [
    'driver' => 'pgsql',
    'host' => env('DB_HOST', 'localhost'),
    'port' => env('DB_PORT', '5432'),
    'database' => env('DB_DATABASE', '{{DB_NAME}}'),
    'username' => env('DB_USERNAME', '{{DB_USER}}'),
    'password' => env('DB_PASSWORD', ''),
    'charset' => env('DB_CHARSET', 'utf8'),
    'prefix' => '',
    'schema' => 'public',
    'sslmode' => env('DB_SSLMODE', 'prefer'),
],
```

**[If Database = MySQL] add connection:**
```php
'mysql' => [
    'driver' => 'mysql',
    'host' => env('DB_HOST', 'localhost'),
    'port' => env('DB_PORT', '3306'),
    'database' => env('DB_DATABASE', '{{DB_NAME}}'),
    'username' => env('DB_USERNAME', '{{DB_USER}}'),
    'password' => env('DB_PASSWORD', ''),
    'unix_socket' => env('DB_SOCKET', ''),
    'charset' => env('DB_CHARSET', 'utf8mb4'),
    'collation' => env('DB_COLLATION', 'utf8mb4_unicode_ci'),
    'prefix' => '',
    'strict' => true,
    'engine' => null,
],
```

**[If Auth = Keycloak] add to config/services.php:**
```php
'keycloak' => [
    'client_id' => env('KEYCLOAK_CLIENT_ID'),
    'client_secret' => env('KEYCLOAK_CLIENT_SECRET'),
    'realm' => env('KEYCLOAK_REALM'),
    'base_url' => env('KEYCLOAK_BASE_URL', 'http://localhost:8180'),
    'redirect' => env('KEYCLOAK_REDIRECT_URI', env('APP_URL') . '/auth/callback'),
],
```

**[If Messaging = yes] add to config/messaging.php:**
```php
return [
    'rabbitmq' => [
        'host' => env('RABBITMQ_HOST', 'localhost'),
        'port' => env('RABBITMQ_PORT', 5672),
        'user' => env('RABBITMQ_USER', 'guest'),
        'password' => env('RABBITMQ_PASSWORD', 'guest'),
        'vhost' => env('RABBITMQ_VHOST', '/'),
        'queue' => env('RABBITMQ_QUEUE', 'default'),
    ],
];
```

### config/logging.php

```php
'channels' => [
    'stack' => [
        'driver' => 'stack',
        'channels' => ['daily'],
    ],
    'daily' => [
        'driver' => 'daily',
        'path' => storage_path('logs/laravel.log'),
        'level' => env('LOG_LEVEL', 'debug'),
        'days' => 14,
    ],
],
```

---

## Section 4: Build & Tooling

### 4.1 Resource Directory Structure
```
resources/
├── css/
│   ├── app.css                  # Main entry — imports Tailwind + partials
│   ├── base/
│   │   ├── reset.css            # Tailwind @layer base
│   │   └── typography.css       # Font imports, text defaults
│   ├── components/
│   │   ├── buttons.css          # @layer components — button styles
│   │   ├── forms.css            # Input, select, textarea styles
│   │   ├── cards.css            # Card component styles
│   │   ├── tables.css           # Table styles
│   │   ├── modals.css           # Modal/dialog styles
│   │   └── navigation.css       # Nav, breadcrumb, tabs styles
│   ├── layouts/
│   │   ├── sidebar.css          # Sidebar responsive behavior
│   │   ├── navbar.css           # Header/navbar styles
│   │   └── footer.css           # Footer styles
│   └── themes/
│       ├── light.css            # Light theme CSS custom properties
│       └── dark.css             # Dark theme CSS custom properties
├── js/
│   ├── app.js                   # Entry — registers Alpine, htmx, global stores
│   ├── alpine/
│   │   ├── stores/
│   │   │   ├── toast.js         # Toast notification store
│   │   │   ├── modal.js         # Modal state store
│   │   │   ├── drawer.js        # Sidebar drawer store
│   │   │   └── theme.js         # Theme toggle store
│   │   └── components/
│   │       ├── dropdown.js      # Dropdown behavior
│   │       ├── tabs.js          # Tab switching
│   │       └── collapse.js      # Accordion/collapse
│   └── htmx/
│       ├── index.js             # Global error/loading handlers
│       └── extensions/
│           ├── toast-errors.js  # Map htmx errors to toast notifications
│           └── loading-states.js # Toggle loading classes during requests
└── views/
    ├── layouts/
    │   ├── app.blade.php        # Main authenticated layout
    │   ├── auth.blade.php       # [If Auth != none] Login layout
    │   └── error.blade.php      # Error layout
    ├── partials/
    │   ├── header.blade.php     # Header bar
    │   ├── sidebar.blade.php    # Sidebar navigation
    │   ├── footer.blade.php     # Footer
    │   └── theme-switcher.blade.php
    └── components/
        ├── alert.blade.php
        ├── avatar.blade.php
        ├── badge.blade.php
        ├── breadcrumb.blade.php
        ├── button.blade.php
        ├── card.blade.php
        ├── form-control.blade.php
        ├── input.blade.php
        ├── select.blade.php
        ├── loading.blade.php
        ├── modal.blade.php
        ├── pagination.blade.php
        ├── table.blade.php
        ├── tabs.blade.php
        └── toast-container.blade.php
```

### 4.2 Vite Configuration
```js
// vite.config.js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
    plugins: [
        laravel({
            input: ['resources/css/app.css', 'resources/js/app.js'],
            refresh: true,
        }),
        tailwindcss(),
    ],
});
```

### 4.3 Tailwind Configuration
```js
// tailwind.config.js
export default {
    content: [
        './resources/**/*.blade.php',
        './resources/**/*.js',
        './Modules/*/resources/views/**/*.blade.php',
    ],
    darkMode: ['class', '[data-theme="dark"]'],
    theme: {
        extend: {
            colors: {
                primary: { 50: '#eff6ff', /* ... full scale */ 900: '#1e3a5f' },
                accent: { 50: '#f0fdf4', /* ... full scale */ 900: '#14532d' },
            },
        },
    },
    plugins: [],
};
```

---

## Section 4b: `.gitignore`

See SKILL.md Section 4b for the complete `.gitignore` content.

---

## Section 5: Directory Structure

Generate the full directory tree. The structure must follow nwidart/laravel-modules
conventions adapted for server-rendered web applications.

```
{{PROJECT_SLUG}}/
├── app/
│   ├── Console/
│   │   └── Commands/
│   │       ├── [If Scheduling] CleanupExpiredRecordsCommand.php
│   │       ├── [If Batch] ProcessDataExportCommand.php
│   │       └── [If Messaging] SetupRabbitMQCommand.php
│   ├── Constants/
│   │   └── Roles.php
│   ├── Contracts/
│   │   └── [If Reporting] ReportDefinition.php
│   ├── Exceptions/
│   │   ├── Handler.php
│   │   └── WebApplicationException.php
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── [If Auth = Keycloak] Auth/KeycloakAuthController.php
│   │   │   └── [If Reporting] ReportController.php
│   │   └── Middleware/
│   │       ├── CspNonceMiddleware.php
│   │       ├── ThemeMiddleware.php
│   │       └── CorrelationIdMiddleware.php
│   ├── Models/
│   │   ├── User.php
│   │   └── [If Reporting] Report.php
│   ├── Providers/
│   │   ├── AppServiceProvider.php
│   │   └── EventServiceProvider.php
│   ├── Services/
│   │   ├── [If Auth = Keycloak] KeycloakRoleMapper.php
│   │   ├── SecurityContext.php
│   │   ├── [If Messaging] Messaging/
│   │   │   ├── RabbitMQInfrastructure.php
│   │   │   └── MessagePublisher.php
│   │   └── [If Reporting] Reporting/
│   │       ├── ReportService.php
│   │       ├── ReportRegistry.php
│   │       └── GenericReportExport.php
│   ├── Traits/
│   │   └── Blameable.php
│   └── Jobs/
│       ├── [If Batch] ProcessExportChunkJob.php
│       └── [If Messaging] Consumers/
│           └── OrderEventConsumer.php
│
├── Modules/
│   ├── {{Module1}}/
│   │   ├── Contracts/
│   │   │   └── {{Module1}}ServiceInterface.php
│   │   ├── DTOs/
│   │   │   └── {{Module1}}Data.php
│   │   ├── Events/
│   │   │   └── {{Module1}}CreatedEvent.php
│   │   ├── Exceptions/
│   │   │   └── {{Module1}}Exception.php
│   │   ├── Services/
│   │   │   └── {{Module1}}Service.php
│   │   ├── Models/
│   │   │   └── {{Module1}}.php
│   │   ├── Http/
│   │   │   ├── Controllers/
│   │   │   │   ├── {{Module1}}PageController.php
│   │   │   │   └── {{Module1}}FragmentController.php
│   │   │   └── Requests/
│   │   │       ├── Store{{Module1}}Request.php
│   │   │       └── Update{{Module1}}Request.php
│   │   ├── ViewModels/
│   │   │   ├── {{Module1}}ListViewModel.php
│   │   │   └── {{Module1}}DetailViewModel.php
│   │   ├── database/
│   │   │   └── migrations/
│   │   ├── resources/
│   │   │   └── views/
│   │   │       ├── pages/
│   │   │       │   ├── index.blade.php
│   │   │       │   ├── show.blade.php
│   │   │       │   ├── create.blade.php
│   │   │       │   └── edit.blade.php
│   │   │       └── fragments/
│   │   │           └── {{module1}}-row.blade.php
│   │   ├── routes/
│   │   │   └── web.php
│   │   ├── Providers/
│   │   │   └── {{Module1}}ServiceProvider.php
│   │   └── Tests/
│   │
│   └── {{Module2}}/
│       └── (same structure)
│
├── resources/
│   ├── css/                    # (see Section 4.1)
│   ├── js/                     # (see Section 4.1)
│   └── views/                  # Shared views (see Section 4.1)
│
├── routes/
│   ├── web.php                 # Core routes (auth, reports)
│   └── console.php             # Scheduled commands
│
├── config/
│   ├── app.php
│   ├── auth.php
│   ├── database.php
│   ├── logging.php
│   ├── queue.php
│   ├── services.php
│   ├── [If Messaging] messaging.php
│   └── modules.php
│
├── database/
│   └── migrations/             # Core migrations (users, roles, etc.)
│
├── bootstrap/
│   └── app.php                 # Middleware registration
│
├── public/
│   └── build/                  # Vite build output (gitignored)
│
├── storage/
├── tests/
│   ├── Architecture/
│   │   └── ModuleBoundaryTest.php
│   ├── Feature/
│   └── Unit/
│
├── composer.json
├── package.json
├── vite.config.js
├── tailwind.config.js
├── .env.example
└── .gitignore
```

---

## Sections 6-23

For Sections 6 through 23, follow the same structure as described in `SKILL.md` under
"What Goes in SPECIFICATION.md (Root)". Each section must include complete code samples
in the appropriate language (PHP, Blade, JavaScript, CSS, YAML).

Reference the corresponding `references/` files for detailed patterns:
- Section 6 (Security): `references/security-patterns.md`
- Section 7-12 (Web/UI): `references/web-patterns.md`
- Section 18 (Scheduling): `references/batch-patterns.md`
- Section 19 (Events): `references/modulith-patterns.md`
- Section 20 (Messaging): `references/messaging-patterns.md`
- Section 23 (Reporting): `references/reporting-patterns.md`

---

# Part B: Per-Module SPEC.md Template

Each module from PRD.md gets its own `<module-name>/SPEC.md` file.

---

## Module SPEC.md Template

```markdown
# {{MODULE_NAME}} — Module Specification

> Back to [SPECIFICATION.md](../SPECIFICATION.md)

## 1. Traceability

### User Stories
| ID | Description | Version |
|----|-------------|---------|
| USHM00108 | As a user, I want to view... | v1.0.0 |

### Non-Functional Requirements
| ID | Description | Version |
|----|-------------|---------|
| NFRHM0120 | Data must be stored in... | v1.0.0 |

### Constraints
| ID | Description | Version |
|----|-------------|---------|
| CONSHM042 | Can only change name and... | v1.0.0 |

### Removed / Replaced
| ID | Removed In | Replaced By | Reason |
|----|-----------|-------------|--------|
| USHM00015 | v1.0.3 | USHM00222 | Consolidated into... |
_(or "None." if no removals)_

### Data Sources
| Artifact | Reference |
|----------|-----------|
| Collection/Table | `{{collection_or_table_name}}` |
| Mockup Screen | `mockup/{{role}}/content/{{screen}}.html` |

## 2. Service Contract

```php
namespace Modules\{{Module}}\Contracts;

interface {{Module}}ServiceInterface
{
    public function list(int $page = 1, int $perPage = 20): LengthAwarePaginator;
    public function get(string $id): {{Module}}Data;
    public function create(array $data): {{Module}}Data;
    public function update(string $id, array $data): {{Module}}Data;
    public function delete(string $id): void;
    // Additional methods from user stories...
}
```

## 3. DTO (spatie/laravel-data)

```php
namespace Modules\{{Module}}\DTOs;

use Spatie\LaravelData\Data;

class {{Module}}Data extends Data
{
    public function __construct(
        public readonly ?string $id,
        // Fields from model/<module>/model.md...
        public readonly ?string $created_by,
        public readonly ?string $updated_by,
        public readonly ?\DateTimeImmutable $created_at,
        public readonly ?\DateTimeImmutable $updated_at,
    ) {}
}
```

## 4. Exception

```php
namespace Modules\{{Module}}\Exceptions;

use App\Exceptions\WebApplicationException;

class {{Module}}Exception extends WebApplicationException
{
    public static function notFound(string $id): self
    {
        return new self("{{Module}} not found: {$id}", 404);
    }
}
```

## 5. Eloquent Model

```php
namespace Modules\{{Module}}\Models;

use App\Traits\Blameable;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;

class {{Module}} extends Model
{
    use HasUuids, Blameable;

    protected $table = '{{table_name}}';

    protected $fillable = [
        // Fields from model/<module>/model.md...
    ];

    protected $casts = [
        // Type casts...
    ];
}
```

## 6. Migration [If Database = PostgreSQL/MySQL]

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('{{table_name}}', function (Blueprint $table) {
            $table->uuid('id')->primary();
            // Fields from model/<module>/model.md...
            $table->string('created_by', 100)->nullable();
            $table->string('updated_by', 100)->nullable();
            $table->timestamps();
            // Indexes from model/<module>/model.md...
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('{{table_name}}');
    }
};
```

## 7. Service Implementation

```php
namespace Modules\{{Module}}\Services;

use Modules\{{Module}}\Contracts\{{Module}}ServiceInterface;
use Modules\{{Module}}\DTOs\{{Module}}Data;
use Modules\{{Module}}\Events\{{Module}}CreatedEvent;
use Modules\{{Module}}\Exceptions\{{Module}}Exception;
use Modules\{{Module}}\Models\{{Module}};

class {{Module}}Service implements {{Module}}ServiceInterface
{
    public function list(int $page = 1, int $perPage = 20): LengthAwarePaginator
    {
        return {{Module}}::orderByDesc('created_at')->paginate($perPage);
    }

    public function get(string $id): {{Module}}Data
    {
        $model = {{Module}}::findOrFail($id);
        return {{Module}}Data::from($model);
    }

    public function create(array $data): {{Module}}Data
    {
        $model = {{Module}}::create($data);

        event(new {{Module}}CreatedEvent(
            id: $model->id,
            occurredAt: new \DateTimeImmutable(),
        ));

        return {{Module}}Data::from($model);
    }

    public function update(string $id, array $data): {{Module}}Data
    {
        $model = {{Module}}::findOrFail($id);
        $model->update($data);
        return {{Module}}Data::from($model->fresh());
    }

    public function delete(string $id): void
    {
        $model = {{Module}}::findOrFail($id);
        $model->delete();
    }
}
```

## 8. Form Requests

```php
namespace Modules\{{Module}}\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class Store{{Module}}Request extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            // Validation rules from model + mockup screens...
        ];
    }
}
```

## 9. Page Controller

```php
namespace Modules\{{Module}}\Http\Controllers;

use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\View\View;
use Modules\{{Module}}\Contracts\{{Module}}ServiceInterface;
use Modules\{{Module}}\Http\Requests\Store{{Module}}Request;
use Modules\{{Module}}\Http\Requests\Update{{Module}}Request;
use Modules\{{Module}}\ViewModels\{{Module}}ListViewModel;
use Modules\{{Module}}\ViewModels\{{Module}}DetailViewModel;

class {{Module}}PageController extends Controller
{
    public function __construct(
        private readonly {{Module}}ServiceInterface $service,
    ) {}

    public function index(Request $request): View
    {
        $items = $this->service->list(
            page: $request->integer('page', 1),
        );
        return (new {{Module}}ListViewModel(items: $items))
            ->view('{{module}}::pages.index');
    }

    public function show(string $id): View
    {
        $item = $this->service->get($id);
        return (new {{Module}}DetailViewModel(item: $item))
            ->view('{{module}}::pages.show');
    }

    public function create(): View
    {
        return view('{{module}}::pages.create');
    }

    public function store(Store{{Module}}Request $request): RedirectResponse
    {
        $this->service->create($request->validated());
        return redirect()->route('{{module}}.index')
            ->with('success', '{{Module}} created successfully.');
    }

    public function edit(string $id): View
    {
        $item = $this->service->get($id);
        return view('{{module}}::pages.edit', compact('item'));
    }

    public function update(Update{{Module}}Request $request, string $id): RedirectResponse
    {
        $this->service->update($id, $request->validated());
        return redirect()->route('{{module}}.show', $id)
            ->with('success', '{{Module}} updated successfully.');
    }

    public function destroy(string $id): RedirectResponse
    {
        $this->service->delete($id);
        return redirect()->route('{{module}}.index')
            ->with('success', '{{Module}} deleted successfully.');
    }
}
```

## 10. Fragment Controller

```php
namespace Modules\{{Module}}\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Illuminate\View\View;
use Modules\{{Module}}\Contracts\{{Module}}ServiceInterface;

class {{Module}}FragmentController extends Controller
{
    public function __construct(
        private readonly {{Module}}ServiceInterface $service,
    ) {}

    public function row(string $id): View
    {
        $item = $this->service->get($id);
        return view('{{module}}::fragments.{{module}}-row', compact('item'));
    }

    public function search(Request $request): View
    {
        $items = $this->service->search($request->input('q', ''));
        return view('{{module}}::fragments.search-results', compact('items'));
    }

    public function destroy(string $id): Response
    {
        $this->service->delete($id);
        return response('', 200);
    }
}
```

## 11. View Models

```php
namespace Modules\{{Module}}\ViewModels;

use Illuminate\Pagination\LengthAwarePaginator;
use Spatie\ViewModels\ViewModel;

class {{Module}}ListViewModel extends ViewModel
{
    public function __construct(
        public readonly LengthAwarePaginator $items,
    ) {}
}
```

## 12. Blade Templates

### pages/index.blade.php
```blade
<x-app-layout title="{{Module}} List">
    <div class="space-y-6">
        <div class="flex items-center justify-between">
            <h1 class="text-2xl font-bold text-gray-900 dark:text-white">{{Module}}</h1>
            <x-button label="Create" href="{{ route('{{module}}.create') }}" tone="primary" />
        </div>

        <div class="flex gap-4">
            <input type="search" name="q" placeholder="Search..."
                   hx-get="{{ route('{{module}}.fragments.search') }}"
                   hx-target="#{{module}}-list"
                   hx-trigger="keyup changed delay:300ms"
                   class="block w-full rounded-lg border border-gray-300 px-3 py-2 text-sm" />
        </div>

        <div id="{{module}}-list">
            @include('{{module}}::fragments.{{module}}-table', ['items' => $items])
        </div>

        <x-pagination :paginator="$items" />
    </div>
</x-app-layout>
```

### fragments/{{module}}-row.blade.php
```blade
<tr class="hover:bg-gray-50 dark:hover:bg-gray-800/50 transition-colors"
    id="{{module}}-row-{{ $item->id }}">
    <td class="px-6 py-4 text-sm text-gray-900 dark:text-gray-300">{{ $item->field_one }}</td>
    {{-- Additional fields from mockup... --}}
    <td class="px-6 py-4 text-sm">
        <a href="{{ route('{{module}}.show', $item->id) }}"
           class="text-blue-600 hover:text-blue-800 dark:text-blue-400">View</a>
        <button hx-delete="{{ route('{{module}}.fragments.destroy', $item->id) }}"
                hx-target="#{{module}}-row-{{ $item->id }}"
                hx-swap="outerHTML"
                hx-confirm="Are you sure?"
                class="text-red-600 hover:text-red-800 dark:text-red-400 ml-3">Delete</button>
    </td>
</tr>
```

## 13. Module Routes

```php
// Modules/{{Module}}/routes/web.php
use Illuminate\Support\Facades\Route;
use Modules\{{Module}}\Http\Controllers\{{Module}}PageController;
use Modules\{{Module}}\Http\Controllers\{{Module}}FragmentController;

Route::middleware(['auth', 'role:{{ROLE}}'])->group(function () {
    Route::resource('{{route-prefix}}', {{Module}}PageController::class);

    Route::prefix('{{route-prefix}}/fragments')->name('{{module}}.fragments.')->group(function () {
        Route::get('/search', [{{Module}}FragmentController::class, 'search'])->name('search');
        Route::get('/row/{id}', [{{Module}}FragmentController::class, 'row'])->name('row');
        Route::delete('/{id}', [{{Module}}FragmentController::class, 'destroy'])->name('destroy');
    });
});
```

## 14. Module Service Provider

```php
namespace Modules\{{Module}}\Providers;

use Illuminate\Support\ServiceProvider;
use Modules\{{Module}}\Contracts\{{Module}}ServiceInterface;
use Modules\{{Module}}\Services\{{Module}}Service;

class {{Module}}ServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind({{Module}}ServiceInterface::class, {{Module}}Service::class);
    }

    public function boot(): void
    {
        $this->loadMigrationsFrom(module_path('{{Module}}', 'database/migrations'));
        $this->loadViewsFrom(module_path('{{Module}}', 'resources/views'), '{{module}}');
        $this->loadRoutesFrom(module_path('{{Module}}', 'routes/web.php'));
    }
}
```

## 15. [If Messaging] Messaging Pipeline

(Only include if the module has async processing NFRs)

### Consumer
```php
// Queue job consuming from RabbitMQ...
```

### Publisher
```php
// Publishing domain events via MessagePublisher...
```

---

**IMPORTANT — Separating UI Layer from Messaging/Async Pipeline:**

When a module has BOTH user-facing screens AND async processing NFRs, the SPEC.md
MUST clearly separate these into distinct sections as shown above (Sections 9-12 for
UI, Section 15 for messaging). This enables the implementation orchestrator to track
and implement each concern independently.
```

---

End of template.
