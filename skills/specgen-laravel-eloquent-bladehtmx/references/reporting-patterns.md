# Reporting Patterns — Puppeteer (Browsershot) + Laravel Excel with DTO Data Source

This reference describes the reporting infrastructure for the spec. Include this content
in the Reporting section of the generated specification. All code samples must be reproduced
in the generated spec with full use statements and constructor injection.

The reporting feature provides:
- **Report Interface** for modules to define and register their own reports
- **Puppeteer via Browsershot** for PDF generation from Blade templates — uses headless Chrome for full CSS support including Tailwind CSS, flexbox, and grid
- **Laravel Excel** for XLSX and CSV export
- **DTO-based data source** — reports receive data via DTOs from services, never query DB directly
- **Multi-format export** — PDF, XLSX, CSV
- **Report registry** persisted in the database for discoverability
- **Report parameter forms** rendered via Blade for user input before generation
- **Blade templates with Tailwind CSS** — PDF templates use the same design system as the web application since Puppeteer renders full CSS

---

## Architecture Overview

```
Module                           Shared Reporting Infrastructure
┌─────────────────────┐         ┌─────────────────────────────────────────┐
│  StaffReport        │         │  ReportService                          │
│  implements         │────────►│    renderPdf(Browsershot + Blade + data)│
│  ReportDefinition   │         │    exportExcel(Laravel Excel + data)    │
│                     │         │    exportCsv(Laravel Excel + data)      │
│  getReportId()      │         │                                         │
│  getName()          │         │  ReportRegistry                         │
│  getDescription()   │         │    discovers tagged ReportDefinition    │
│  getParameters()    │         │    persists to reports table            │
│  generateData(...)  │         │    provides report lookup               │
│                     │         │                                         │
│  --- Data Flow ---  │         │  ReportPageController                   │
│  Service → DTO[]    │         │    list all available reports            │
│  (never Repository) │         │    render parameter form                │
└─────────────────────┘         │    generate + download report           │
                                └─────────────────────────────────────────┘
```

**Key design rule:** Reports obtain data through module service interfaces (which return
DTOs), never by injecting repositories or querying the database directly. This preserves
module boundaries — reports are consumers of the module's public API.

**Rendering engine:** Browsershot renders Blade templates using headless Chrome (Puppeteer).
Templates are self-contained HTML documents that include Tailwind CSS via CDN, enabling
the same utility classes used in the web application. This eliminates the CSS limitations
of traditional PDF libraries (no flexbox, no grid, limited font support).

---

## Composer Dependencies

```json
{
    "require": {
        "spatie/browsershot": "^4.0",
        "maatwebsite/excel": "^3.1"
    }
}
```

Additionally, install the Puppeteer Node.js package on the server:

```bash
npm install puppeteer
```

This installs both the Puppeteer library and a compatible headless Chrome binary.

---

## Server Requirements

Browsershot requires Node.js and a Chrome/Chromium binary to render PDFs. The setup
depends on the deployment environment:

**Local development:**
- Install Node.js (v18+) and run `npm install puppeteer` in the project root.
- Puppeteer automatically downloads a compatible Chrome binary into `node_modules`.
- Browsershot auto-detects the Node.js binary and Chrome path from `node_modules`.

**Production / Docker:**
- The Dockerfile must include Node.js and either install Puppeteer (which bundles Chrome)
  or install Chromium via the system package manager.
- When running Chrome inside Docker, the `--no-sandbox` flag is required (Browsershot
  applies this automatically when it detects a non-TTY environment, but it can also be
  set explicitly).
- Set the environment variable `PUPPETEER_EXECUTABLE_PATH` to use a system-installed
  Chrome instead of the one bundled with Puppeteer.

**Environment variables (optional):**
- `BROWSERSHOT_NODE_BINARY` — absolute path to the Node.js binary
- `BROWSERSHOT_NPM_BINARY` — absolute path to the npm binary
- `BROWSERSHOT_CHROME_PATH` — absolute path to the Chrome/Chromium binary
- `BROWSERSHOT_NODE_MODULES_PATH` — absolute path to the `node_modules` directory

---

## Package Structure

```
app/
├── Contracts/
│   └── ReportDefinition.php              # Interface for module reports
├── Services/
│   └── Reporting/
│       ├── ReportService.php             # Orchestrates render/export via Browsershot
│       └── ReportRegistry.php            # Discovers and persists report definitions
├── Models/
│   └── Report.php                        # Eloquent model for reports table
├── Http/
│   └── Controllers/
│       └── ReportController.php          # Report list, form, download endpoints
├── config/
│   └── browsershot.php                   # Browsershot binary path configuration
└── resources/
    └── views/
        └── reports/
            ├── index.blade.php           # Report list page
            ├── parameters.blade.php      # Parameter form page
            └── templates/                # PDF Blade templates (self-contained HTML)
                └── sample-report.blade.php
```

---

## ReportDefinition Interface

```php
namespace App\Contracts;

interface ReportDefinition
{
    /**
     * Unique identifier for this report (e.g., 'staff-list-report').
     */
    public function getReportId(): string;

    /**
     * Human-readable display name.
     */
    public function getName(): string;

    /**
     * Description shown in the report list.
     */
    public function getDescription(): string;

    /**
     * Parameter definitions for the report form.
     * Each parameter: ['name' => string, 'label' => string, 'type' => string, 'required' => bool]
     */
    public function getParameters(): array;

    /**
     * Generate report data as an array of DTOs.
     * Parameters are the user-provided form inputs.
     */
    public function generateData(array $parameters): array;

    /**
     * Supported export formats: ['pdf', 'xlsx', 'csv']
     */
    public function getSupportedFormats(): array;

    /**
     * Blade template path for PDF rendering (e.g., 'reports.templates.staff-list').
     */
    public function getPdfTemplate(): string;

    /**
     * Column headers for XLSX/CSV export.
     */
    public function getExportHeaders(): array;

    /**
     * Whether the PDF should be rendered in landscape orientation.
     */
    public function isLandscape(): bool;
}
```

---

## Report Model (Eloquent)

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Concerns\HasUuids;

class Report extends Model
{
    use HasUuids;

    protected $fillable = [
        'report_id',
        'name',
        'description',
        'parameters',
        'supported_formats',
        'module',
    ];

    protected $casts = [
        'parameters' => 'array',
        'supported_formats' => 'array',
    ];
}
```

Migration:
```php
Schema::create('reports', function (Blueprint $table) {
    $table->uuid('id')->primary();
    $table->string('report_id')->unique();
    $table->string('name');
    $table->text('description')->nullable();
    $table->json('parameters')->nullable();
    $table->json('supported_formats');
    $table->string('module')->nullable();
    $table->timestamps();
});
```

---

## ReportRegistry

Auto-discovers all `ReportDefinition` implementations and syncs to the database:

```php
namespace App\Services\Reporting;

use App\Contracts\ReportDefinition;
use App\Models\Report;

class ReportRegistry
{
    /** @var array<string, ReportDefinition> */
    private array $definitions = [];

    /**
     * Register a report definition (called from service providers).
     */
    public function register(ReportDefinition $definition): void
    {
        $this->definitions[$definition->getReportId()] = $definition;
    }

    /**
     * Get a report definition by ID.
     */
    public function get(string $reportId): ?ReportDefinition
    {
        return $this->definitions[$reportId] ?? null;
    }

    /**
     * Sync all registered definitions to the database.
     */
    public function sync(): void
    {
        foreach ($this->definitions as $definition) {
            Report::updateOrCreate(
                ['report_id' => $definition->getReportId()],
                [
                    'name' => $definition->getName(),
                    'description' => $definition->getDescription(),
                    'parameters' => $definition->getParameters(),
                    'supported_formats' => $definition->getSupportedFormats(),
                ]
            );
        }

        // Remove reports no longer registered
        Report::whereNotIn('report_id', array_keys($this->definitions))->delete();
    }

    /**
     * Get all registered definitions.
     */
    public function all(): array
    {
        return $this->definitions;
    }
}
```

Register as singleton in `AppServiceProvider`:
```php
$this->app->singleton(ReportRegistry::class);
```

Modules register their reports in their service provider:
```php
// In Modules/Staff/Providers/StaffServiceProvider.php
public function boot(): void
{
    $registry = app(ReportRegistry::class);
    $registry->register(app(StaffListReport::class));
}
```

---

## ReportService

Orchestrates report generation across all formats. Uses Browsershot (Puppeteer) for PDF
rendering — the Blade template is rendered to HTML, then Browsershot converts it to PDF
using headless Chrome:

```php
namespace App\Services\Reporting;

use App\Contracts\ReportDefinition;
use Illuminate\Http\Response;
use Maatwebsite\Excel\Facades\Excel;
use Spatie\Browsershot\Browsershot;
use Symfony\Component\HttpFoundation\BinaryFileResponse;

class ReportService
{
    public function __construct(
        private readonly ReportRegistry $registry,
    ) {}

    /**
     * Generate and download a report in the specified format.
     */
    public function generate(string $reportId, array $parameters, string $format): Response|BinaryFileResponse
    {
        $definition = $this->registry->get($reportId)
            ?? throw new \RuntimeException("Report not found: {$reportId}");

        $data = $definition->generateData($parameters);

        return match ($format) {
            'pdf' => $this->generatePdf($definition, $data, $parameters),
            'xlsx' => $this->generateExcel($definition, $data, 'xlsx'),
            'csv' => $this->generateExcel($definition, $data, 'csv'),
            default => throw new \InvalidArgumentException("Unsupported format: {$format}"),
        };
    }

    private function generatePdf(ReportDefinition $definition, array $data, array $parameters): Response
    {
        $html = view($definition->getPdfTemplate(), [
            'data' => $data,
            'parameters' => $parameters,
            'reportName' => $definition->getName(),
            'generatedAt' => now(),
        ])->render();

        $filename = $definition->getReportId() . '-' . now()->format('Y-m-d-His') . '.pdf';

        $browsershot = Browsershot::html($html)
            ->format('A4')
            ->margins(10, 10, 10, 10)
            ->showBackground()
            ->waitUntilNetworkIdle();

        if ($definition->isLandscape()) {
            $browsershot->landscape();
        }

        $this->applyBrowsershotConfig($browsershot);

        $pdfContent = $browsershot->pdf();

        return response($pdfContent)
            ->header('Content-Type', 'application/pdf')
            ->header('Content-Disposition', 'attachment; filename="' . $filename . '"');
    }

    private function generateExcel(ReportDefinition $definition, array $data, string $format): BinaryFileResponse
    {
        $export = new GenericReportExport($definition->getExportHeaders(), $data);
        $extension = $format === 'csv' ? 'csv' : 'xlsx';
        $filename = $definition->getReportId() . '-' . now()->format('Y-m-d-His') . '.' . $extension;

        $writerType = $format === 'csv'
            ? \Maatwebsite\Excel\Excel::CSV
            : \Maatwebsite\Excel\Excel::XLSX;

        return Excel::download($export, $filename, $writerType);
    }

    /**
     * Apply optional Browsershot configuration from config/browsershot.php.
     */
    private function applyBrowsershotConfig(Browsershot $browsershot): void
    {
        $nodeBinary = config('browsershot.node_binary');
        $npmBinary = config('browsershot.npm_binary');
        $chromePath = config('browsershot.chrome_path');
        $nodeModulesPath = config('browsershot.node_modules_path');

        if ($nodeBinary) {
            $browsershot->setNodeBinary($nodeBinary);
        }

        if ($npmBinary) {
            $browsershot->setNpmBinary($npmBinary);
        }

        if ($chromePath) {
            $browsershot->setChromePath($chromePath);
        }

        if ($nodeModulesPath) {
            $browsershot->setNodeModulePath($nodeModulesPath);
        }

        if (config('browsershot.no_sandbox', false)) {
            $browsershot->noSandbox();
        }
    }
}
```

---

## Generic Report Export (Laravel Excel)

```php
namespace App\Services\Reporting;

use Maatwebsite\Excel\Concerns\FromArray;
use Maatwebsite\Excel\Concerns\WithHeadings;
use Maatwebsite\Excel\Concerns\WithStyles;
use PhpOffice\PhpSpreadsheet\Worksheet\Worksheet;

class GenericReportExport implements FromArray, WithHeadings, WithStyles
{
    public function __construct(
        private readonly array $headers,
        private readonly array $data,
    ) {}

    public function headings(): array
    {
        return $this->headers;
    }

    public function array(): array
    {
        return array_map(function ($row) {
            if (is_object($row) && method_exists($row, 'toArray')) {
                return $row->toArray();
            }
            return (array) $row;
        }, $this->data);
    }

    public function styles(Worksheet $sheet): array
    {
        return [
            1 => ['font' => ['bold' => true]],
        ];
    }
}
```

---

## Report Controller

```php
namespace App\Http\Controllers;

use App\Models\Report;
use App\Services\Reporting\ReportRegistry;
use App\Services\Reporting\ReportService;
use Illuminate\Http\Request;
use Illuminate\View\View;

class ReportController extends Controller
{
    public function __construct(
        private readonly ReportService $reportService,
        private readonly ReportRegistry $registry,
    ) {}

    public function index(): View
    {
        $reports = Report::orderBy('name')->paginate(20);
        return view('reports.index', compact('reports'));
    }

    public function parameters(string $reportId): View
    {
        $definition = $this->registry->get($reportId)
            ?? abort(404, "Report not found: {$reportId}");

        return view('reports.parameters', [
            'report' => $definition,
            'reportId' => $reportId,
        ]);
    }

    public function generate(Request $request, string $reportId)
    {
        $definition = $this->registry->get($reportId)
            ?? abort(404, "Report not found: {$reportId}");

        $validated = $request->validate(
            collect($definition->getParameters())
                ->mapWithKeys(fn ($param) => [
                    $param['name'] => $param['required'] ? 'required' : 'nullable',
                ])
                ->toArray()
        );

        $format = $request->input('format', 'pdf');

        return $this->reportService->generate($reportId, $validated, $format);
    }
}
```

---

## Sample Module Report Implementation

```php
namespace Modules\Staff\Reports;

use App\Contracts\ReportDefinition;
use Modules\Staff\Contracts\StaffServiceInterface;

class StaffListReport implements ReportDefinition
{
    public function __construct(
        private readonly StaffServiceInterface $staffService,
    ) {}

    public function getReportId(): string
    {
        return 'staff-list-report';
    }

    public function getName(): string
    {
        return 'Staff List Report';
    }

    public function getDescription(): string
    {
        return 'List all staff members with their department and status.';
    }

    public function getParameters(): array
    {
        return [
            ['name' => 'department', 'label' => 'Department', 'type' => 'select', 'required' => false],
            ['name' => 'status', 'label' => 'Status', 'type' => 'select', 'required' => false],
        ];
    }

    public function generateData(array $parameters): array
    {
        // Uses the service interface (public API), NOT the repository
        return $this->staffService->getStaffForReport(
            department: $parameters['department'] ?? null,
            status: $parameters['status'] ?? null,
        );
    }

    public function getSupportedFormats(): array
    {
        return ['pdf', 'xlsx', 'csv'];
    }

    public function getPdfTemplate(): string
    {
        return 'staff::reports.staff-list';
    }

    public function getExportHeaders(): array
    {
        return ['Name', 'Email', 'Department', 'Position', 'Status', 'Joined Date'];
    }

    public function isLandscape(): bool
    {
        return false;
    }
}
```

---

## PDF Blade Template

PDF templates are self-contained HTML documents. Because Browsershot uses headless Chrome,
templates have full CSS support — Tailwind CSS classes work exactly as they do in the
browser. Include the Tailwind CSS CDN in the `<head>` so the template renders correctly
when Browsershot processes it outside the web application context.

```blade
{{-- Modules/Staff/resources/views/reports/staff-list.blade.php --}}
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>{{ $reportName }}</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @page { margin: 0; }
        body { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
    </style>
</head>
<body class="bg-white text-gray-900 text-sm p-8">
    {{-- Report Header --}}
    <div class="text-center mb-6">
        <h1 class="text-xl font-bold">{{ $reportName }}</h1>
        <p class="text-xs text-gray-500">Generated: {{ $generatedAt->format('Y-m-d H:i:s') }}</p>
    </div>

    {{-- Report Parameters Summary --}}
    @if (!empty(array_filter($parameters)))
        <div class="mb-4 flex flex-wrap gap-3 text-xs text-gray-600">
            @foreach ($parameters as $key => $value)
                @if ($value)
                    <span class="bg-gray-100 rounded px-2 py-1">
                        <span class="font-medium">{{ ucfirst(str_replace('_', ' ', $key)) }}:</span>
                        {{ $value }}
                    </span>
                @endif
            @endforeach
        </div>
    @endif

    {{-- Data Table --}}
    <table class="w-full border-collapse">
        <thead>
            <tr class="bg-gray-100">
                <th class="border border-gray-300 px-3 py-2 text-left font-semibold">Name</th>
                <th class="border border-gray-300 px-3 py-2 text-left font-semibold">Email</th>
                <th class="border border-gray-300 px-3 py-2 text-left font-semibold">Department</th>
                <th class="border border-gray-300 px-3 py-2 text-left font-semibold">Position</th>
                <th class="border border-gray-300 px-3 py-2 text-left font-semibold">Status</th>
                <th class="border border-gray-300 px-3 py-2 text-left font-semibold">Joined Date</th>
            </tr>
        </thead>
        <tbody>
            @foreach ($data as $staff)
                <tr class="even:bg-gray-50">
                    <td class="border border-gray-300 px-3 py-2">{{ $staff->name }}</td>
                    <td class="border border-gray-300 px-3 py-2">{{ $staff->email }}</td>
                    <td class="border border-gray-300 px-3 py-2">{{ $staff->department }}</td>
                    <td class="border border-gray-300 px-3 py-2">{{ $staff->position }}</td>
                    <td class="border border-gray-300 px-3 py-2">
                        <span class="inline-block rounded-full px-2 py-0.5 text-xs font-medium
                            {{ $staff->status === 'Active' ? 'bg-green-100 text-green-800' : 'bg-red-100 text-red-800' }}">
                            {{ $staff->status }}
                        </span>
                    </td>
                    <td class="border border-gray-300 px-3 py-2">{{ $staff->joined_date }}</td>
                </tr>
            @endforeach
        </tbody>
    </table>

    {{-- Footer --}}
    <div class="flex justify-between items-center mt-6 text-xs text-gray-500">
        <p>Total records: {{ count($data) }}</p>
        <p>Page 1</p>
    </div>
</body>
</html>
```

---

## Browsershot Configuration

Create a configuration file at `config/browsershot.php` to allow customizing binary paths
per environment. This is especially useful for Docker deployments where Node.js and Chrome
are installed at non-standard paths:

```php
<?php

return [

    /*
    |--------------------------------------------------------------------------
    | Node.js Binary Path
    |--------------------------------------------------------------------------
    |
    | Absolute path to the Node.js binary. When null, Browsershot auto-detects
    | the Node.js binary from the system PATH.
    |
    */
    'node_binary' => env('BROWSERSHOT_NODE_BINARY', null),

    /*
    |--------------------------------------------------------------------------
    | npm Binary Path
    |--------------------------------------------------------------------------
    |
    | Absolute path to the npm binary. When null, Browsershot auto-detects
    | the npm binary from the system PATH.
    |
    */
    'npm_binary' => env('BROWSERSHOT_NPM_BINARY', null),

    /*
    |--------------------------------------------------------------------------
    | Chrome / Chromium Binary Path
    |--------------------------------------------------------------------------
    |
    | Absolute path to the Chrome or Chromium binary. When null, Browsershot
    | uses the Chrome binary bundled with Puppeteer in node_modules.
    |
    */
    'chrome_path' => env('BROWSERSHOT_CHROME_PATH', null),

    /*
    |--------------------------------------------------------------------------
    | node_modules Path
    |--------------------------------------------------------------------------
    |
    | Absolute path to the node_modules directory containing Puppeteer.
    | When null, Browsershot looks for node_modules in the project root.
    |
    */
    'node_modules_path' => env('BROWSERSHOT_NODE_MODULES_PATH', null),

    /*
    |--------------------------------------------------------------------------
    | Disable Chrome Sandbox
    |--------------------------------------------------------------------------
    |
    | Set to true when running inside Docker or other containerized environments
    | where Chrome's sandbox cannot be used. This passes --no-sandbox to Chrome.
    |
    */
    'no_sandbox' => env('BROWSERSHOT_NO_SANDBOX', false),

];
```

---

## Key Design Decisions

**Why Puppeteer / Browsershot instead of DomPDF:**
- DomPDF has limited CSS support — no flexbox, no grid, incomplete Tailwind CSS compatibility.
- Puppeteer uses headless Chrome, which provides full CSS support and pixel-perfect rendering.
- Report Blade templates use Tailwind CSS classes, the same design system as the web application,
  ensuring visual consistency between the UI and generated PDFs.
- Browsershot provides a clean PHP API around Puppeteer, handling process management and
  binary detection automatically.

**Why DTO as data source:**
- Modules expose data through service interfaces that return DTOs.
- Reports consume the module's public API, never bypassing it to query the database directly.
- This preserves module boundaries — a report is a consumer of the module, not part of its internals.
- DTOs are easily testable (construct them in tests without database setup).

**Template management:**
- Each module owns its PDF Blade templates in its `resources/views/reports/` directory.
- Templates are self-contained HTML documents (not partials) — they include their own `<head>`,
  Tailwind CDN, and print styles so Browsershot can render them independently.
- The AI coding agent writes these templates programmatically as part of module development.

---

## Operational Notes

**Chrome memory usage:**
- Each Browsershot PDF generation spawns a headless Chrome process. For reports with large
  datasets (thousands of rows), Chrome may consume significant memory (200-500 MB per render).
- For very large reports, consider paginating the data and generating multiple pages rather
  than rendering a single enormous table.

**Concurrent report generation:**
- Multiple simultaneous PDF requests each spawn their own Chrome process. On servers with
  limited memory, consider using a queue (Laravel Queue) to serialize report generation
  and prevent memory exhaustion.
- Example: dispatch a job to generate the report, then redirect the user to a download page
  that polls for completion.

**Docker setup:**
- The Dockerfile must install Node.js and either Puppeteer (which bundles Chrome) or
  a system Chromium package.
- Chrome inside Docker requires the `--no-sandbox` flag. Set `BROWSERSHOT_NO_SANDBOX=true`
  in the container environment.
- Required system dependencies for Chrome in Debian/Ubuntu-based containers:
  `libnss3`, `libatk1.0-0`, `libatk-bridge2.0-0`, `libcups2`, `libxcomposite1`,
  `libxrandr2`, `libgbm1`, `libpango-1.0-0`, `libcairo2`, `libasound2`.

**Tailwind CDN in templates:**
- The `<script src="https://cdn.tailwindcss.com"></script>` tag in PDF templates loads
  Tailwind CSS at render time. Browsershot's `waitUntilNetworkIdle()` ensures the script
  is fully loaded before generating the PDF.
- For air-gapped environments without internet access, bundle a pre-built Tailwind CSS file
  and reference it with an absolute `file://` path or inline the CSS directly in the template.
