# Reporting Patterns — DomPDF + Laravel Excel with DTO Data Source

This reference describes the reporting infrastructure for the spec. Include this content
in the Reporting section of the generated specification. All code samples must be reproduced
in the generated spec with full use statements and constructor injection.

The reporting feature provides:
- **Report Interface** for modules to define and register their own reports
- **DomPDF** for PDF generation from Blade templates
- **Laravel Excel** for XLSX and CSV export
- **DTO-based data source** — reports receive data via DTOs from services, never query DB directly
- **Multi-format export** — PDF, XLSX, CSV
- **Report registry** persisted in the database for discoverability
- **Report parameter forms** rendered via Blade for user input before generation

---

## Architecture Overview

```
Module                           Shared Reporting Infrastructure
┌─────────────────────┐         ┌─────────────────────────────────────────┐
│  StaffReport        │         │  ReportService                          │
│  implements         │────────►│    renderPdf(Blade template + data)     │
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

---

## Composer Dependencies

```json
{
    "require": {
        "barryvdh/laravel-dompdf": "^3.0",
        "maatwebsite/excel": "^3.1"
    }
}
```

---

## Package Structure

```
app/
├── Contracts/
│   └── ReportDefinition.php              # Interface for module reports
├── Services/
│   └── Reporting/
│       ├── ReportService.php             # Orchestrates render/export
│       └── ReportRegistry.php            # Discovers and persists report definitions
├── Models/
│   └── Report.php                        # Eloquent model for reports table
├── Http/
│   └── Controllers/
│       └── ReportController.php          # Report list, form, download endpoints
└── resources/
    └── views/
        └── reports/
            ├── index.blade.php           # Report list page
            ├── parameters.blade.php      # Parameter form page
            └── templates/                # PDF Blade templates
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

Orchestrates report generation across all formats:

```php
namespace App\Services\Reporting;

use App\Contracts\ReportDefinition;
use Barryvdh\DomPDF\Facade\Pdf;
use Illuminate\Http\Response;
use Maatwebsite\Excel\Facades\Excel;
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
        $pdf = Pdf::loadView($definition->getPdfTemplate(), [
            'data' => $data,
            'parameters' => $parameters,
            'reportName' => $definition->getName(),
            'generatedAt' => now(),
        ]);

        $filename = $definition->getReportId() . '-' . now()->format('Y-m-d-His') . '.pdf';

        return $pdf->download($filename);
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
}
```

---

## PDF Blade Template

```blade
{{-- Modules/Staff/resources/views/reports/staff-list.blade.php --}}
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>{{ $reportName }}</title>
    <style>
        body { font-family: DejaVu Sans, sans-serif; font-size: 10px; }
        table { width: 100%; border-collapse: collapse; margin-top: 20px; }
        th, td { border: 1px solid #ddd; padding: 6px 8px; text-align: left; }
        th { background-color: #f5f5f5; font-weight: bold; }
        .header { text-align: center; margin-bottom: 20px; }
        .header h1 { font-size: 16px; margin: 0; }
        .meta { font-size: 9px; color: #666; }
    </style>
</head>
<body>
    <div class="header">
        <h1>{{ $reportName }}</h1>
        <p class="meta">Generated: {{ $generatedAt->format('Y-m-d H:i:s') }}</p>
    </div>

    <table>
        <thead>
            <tr>
                <th>Name</th>
                <th>Email</th>
                <th>Department</th>
                <th>Position</th>
                <th>Status</th>
                <th>Joined Date</th>
            </tr>
        </thead>
        <tbody>
            @foreach ($data as $staff)
                <tr>
                    <td>{{ $staff->name }}</td>
                    <td>{{ $staff->email }}</td>
                    <td>{{ $staff->department }}</td>
                    <td>{{ $staff->position }}</td>
                    <td>{{ $staff->status }}</td>
                    <td>{{ $staff->joined_date }}</td>
                </tr>
            @endforeach
        </tbody>
    </table>

    <p class="meta" style="margin-top: 20px;">Total records: {{ count($data) }}</p>
</body>
</html>
```
