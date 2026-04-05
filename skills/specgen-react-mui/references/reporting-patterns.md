# Reporting Patterns — Puppeteer PDF + Client-Side Export

This reference describes the reporting infrastructure for the spec. Include this content
in the Reporting section of the generated specification. All code samples must be reproduced
in the generated spec with full import statements and typed signatures.

The reporting feature provides:
- **Dual architecture** — server-side Puppeteer for pixel-perfect PDF generation, client-side
  libraries for instant CSV/XLSX export
- **React layout components** rendered to static HTML via `ReactDOMServer.renderToStaticMarkup()`
  and sent to a companion Node.js report service that uses Puppeteer (headless Chrome) to
  produce PDF documents
- **Client-side CSV export** via PapaParse (no server round-trip)
- **Client-side XLSX export** via SheetJS `xlsx` library (no server round-trip)
- **Report definitions** — typed metadata describing each report's parameters and supported formats
- **MUI-based report UI** — report list page, parameter forms (React Hook Form + Zod),
  preview dialog, and format selection
- **Report HTML templates** use Tailwind CSS CDN for standalone rendering inside Puppeteer
  (independent of the application's MUI build)
- All report layouts are programmatically developed by the AI coding agent — no manual
  template authoring required

---

## Architecture Overview

```
React SPA (Browser)                    Report Service (Node.js)
┌──────────────────────────┐          ┌──────────────────────────┐
│  ReportListPage          │          │  Express.js              │
│  ReportParameterForm     │   POST   │  POST /api/reports/pdf   │
│  ReportPreview           │─────────►│    receives: HTML string │
│  useReportGeneration()   │          │    renders via Puppeteer  │
│                          │◄─────────│    returns: PDF bytes    │
│  Client-side export:     │  binary  │                          │
│  • xlsx (XLSX)           │          └──────────────────────────┘
│  • Papa Parse (CSV)      │
└──────────────────────────┘
```

**Why this architecture:**
- React is a frontend-only SPA — it cannot generate PDFs natively in the browser
- Puppeteer requires Node.js + headless Chrome (server-side only)
- The report "layout" is an HTML document that Puppeteer renders to PDF with full CSS support
- The React SPA builds report HTML from React layout components using
  `ReactDOMServer.renderToStaticMarkup()`, then sends the HTML string to the report service
  which returns PDF bytes as a binary download
- For CSV/XLSX, client-side libraries handle export directly in the browser with zero
  server round-trip — the data is already available from TanStack Query cache

**Key design rule:** Report layout components are plain React components that use inline
styles or Tailwind CSS classes (loaded via CDN in the HTML wrapper). They do NOT use MUI
components or the application's Emotion styling — Puppeteer renders a standalone HTML
document that has no access to the application's runtime CSS.

---

## npm Dependencies

### React SPA (main application package.json)

**Production:**

```json
{
  "dependencies": {
    "xlsx": "^0.18.5",
    "papaparse": "^5.4.1"
  },
  "devDependencies": {
    "@types/papaparse": "^5.3.14"
  }
}
```

### Report Service (separate package.json — companion service)

```json
{
  "name": "{{PROJECT_SLUG}}-report-service",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "start": "node dist/server.js",
    "dev": "tsx watch src/server.ts",
    "build": "tsc"
  },
  "dependencies": {
    "express": "^4.21.0",
    "puppeteer": "^23.0.0",
    "cors": "^2.8.5"
  },
  "devDependencies": {
    "@types/express": "^5.0.0",
    "@types/cors": "^2.8.17",
    "typescript": "^5.0.0",
    "tsx": "^4.0.0"
  }
}
```

---

## Report Service (Node.js Express)

A lightweight Express server that accepts an HTML string and returns a rendered PDF.
This runs as a companion service alongside the Vite dev server during development, or
as a separate microservice (sidecar container) in production.

### Server Implementation

```typescript
// report-service/src/server.ts
import express, { type Request, type Response } from 'express';
import puppeteer, { type Browser } from 'puppeteer';
import cors from 'cors';

const app = express();
app.use(cors());
app.use(express.json({ limit: '10mb' }));

// ---------------------------------------------------------------------------
// Browser Singleton — reuse a single headless Chrome instance across requests
// ---------------------------------------------------------------------------
let browser: Browser | null = null;

async function getBrowser(): Promise<Browser> {
  if (!browser || !browser.connected) {
    browser = await puppeteer.launch({
      headless: true,
      args: [
        '--no-sandbox',
        '--disable-setuid-sandbox',
        '--disable-dev-shm-usage',
        '--disable-gpu',
      ],
    });
  }
  return browser;
}

// ---------------------------------------------------------------------------
// PDF Generation Endpoint
// ---------------------------------------------------------------------------
interface PdfRequestBody {
  html: string;
  landscape?: boolean;
  format?: 'A4' | 'A3' | 'Letter' | 'Legal';
  filename?: string;
}

app.post('/api/reports/pdf', async (req: Request<object, Buffer, PdfRequestBody>, res: Response): Promise<void> => {
  const {
    html,
    landscape = false,
    format = 'A4',
    filename = 'report.pdf',
  } = req.body;

  if (!html || typeof html !== 'string') {
    res.status(400).json({ error: 'Request body must include an "html" string.' });
    return;
  }

  let page: Awaited<ReturnType<Browser['newPage']>> | null = null;

  try {
    const b = await getBrowser();
    page = await b.newPage();

    // Set content and wait for all network requests (e.g., Tailwind CDN) to finish
    await page.setContent(html, { waitUntil: 'networkidle0' });

    const pdf = await page.pdf({
      format,
      landscape,
      printBackground: true,
      margin: {
        top: '10mm',
        bottom: '10mm',
        left: '10mm',
        right: '10mm',
      },
    });

    res.setHeader('Content-Type', 'application/pdf');
    res.setHeader('Content-Disposition', `attachment; filename="${filename}"`);
    res.setHeader('Content-Length', String(pdf.byteLength));
    res.send(pdf);
  } catch (error) {
    console.error('PDF generation failed:', error);
    res.status(500).json({ error: 'PDF generation failed.' });
  } finally {
    if (page) {
      await page.close();
    }
  }
});

// ---------------------------------------------------------------------------
// Health Check
// ---------------------------------------------------------------------------
app.get('/api/reports/health', (_req: Request, res: Response) => {
  res.json({ status: 'ok' });
});

// ---------------------------------------------------------------------------
// Graceful Shutdown
// ---------------------------------------------------------------------------
const PORT = parseInt(process.env.REPORT_SERVICE_PORT ?? '3001', 10);

const server = app.listen(PORT, () => {
  console.log(`Report service listening on port ${PORT}`);
});

async function gracefulShutdown(): Promise<void> {
  console.log('Shutting down report service...');
  server.close();
  if (browser) {
    await browser.close();
    browser = null;
  }
  process.exit(0);
}

process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);
```

### Vite Proxy Configuration (Development)

During development, Vite proxies report service requests to the companion Node.js server
so the React SPA can call `/api/reports/pdf` without CORS issues:

```typescript
// vite.config.ts (add to existing server.proxy)
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: {{DEV_PORT}},
    proxy: {
      // Existing API proxy
      '/api/v1': {
        target: '{{API_BASE_URL_DEV}}',
        changeOrigin: true,
      },
      // Report service proxy
      '/api/reports': {
        target: 'http://localhost:3001',
        changeOrigin: true,
      },
    },
  },
});
```

### Report Service Environment Variables

```env
# report-service/.env
REPORT_SERVICE_PORT=3001
```

---

## Report Types and Interfaces

```typescript
// src/features/reports/reports.types.ts

/**
 * Metadata definition for a single report.
 * Report definitions are typically provided by the backend API or
 * defined as constants in the frontend for static report catalogues.
 */
export interface ReportDefinition {
  /** Unique identifier (e.g., 'staff-list-report') */
  reportId: string;
  /** Human-readable display name */
  name: string;
  /** Description shown in the report catalogue */
  description: string;
  /** Domain/module grouping (e.g., 'Human Resources', 'Finance') */
  domain: string;
  /** Parameters the user must fill before generating */
  parameters: ReportParameter[];
  /** Supported output formats */
  supportedFormats: ReportFormat[];
}

/**
 * A single parameter definition for a report's input form.
 */
export interface ReportParameter {
  /** Field name used as the form field key */
  name: string;
  /** Display label */
  label: string;
  /** Input type — maps to MUI form controls */
  type: 'text' | 'date' | 'number' | 'select' | 'boolean';
  /** Whether the parameter is required */
  required: boolean;
  /** Options for 'select' type parameters */
  options?: ReportParameterOption[];
  /** Default value pre-filled in the form */
  defaultValue?: string;
}

export interface ReportParameterOption {
  label: string;
  value: string;
}

/** Supported export formats */
export type ReportFormat = 'pdf' | 'xlsx' | 'csv';

/**
 * Represents a row of tabular report data.
 * Each key is a column identifier, value is the cell content.
 */
export type ReportDataRow = Record<string, unknown>;

/**
 * Column definition for tabular report layouts and client-side exports.
 */
export interface ReportColumn {
  /** Column identifier matching the key in ReportDataRow */
  key: string;
  /** Display header label */
  header: string;
  /** Column width hint (for PDF layout) */
  width?: string;
  /** Text alignment */
  align?: 'left' | 'center' | 'right';
}
```

---

## Report API Hooks (TanStack Query)

```typescript
// src/features/reports/api/reportApi.ts
import { axiosInstance } from '../../../lib/api/axiosInstance';
import type { ReportDefinition } from '../reports.types';

export const reportApi = {
  /**
   * Fetch all available report definitions.
   */
  list: async (): Promise<ReportDefinition[]> => {
    const { data } = await axiosInstance.get<ReportDefinition[]>('/v1/reports');
    return data;
  },

  /**
   * Fetch a single report definition by ID.
   */
  getById: async (reportId: string): Promise<ReportDefinition> => {
    const { data } = await axiosInstance.get<ReportDefinition>(`/v1/reports/${reportId}`);
    return data;
  },
};
```

```typescript
// src/features/reports/hooks/useReports.ts
import { useQuery } from '@tanstack/react-query';
import { reportApi } from '../api/reportApi';

export const reportKeys = {
  all: ['reports'] as const,
  lists: () => [...reportKeys.all, 'list'] as const,
  list: () => [...reportKeys.lists()] as const,
  details: () => [...reportKeys.all, 'detail'] as const,
  detail: (reportId: string) => [...reportKeys.details(), reportId] as const,
};

/**
 * Fetch all available report definitions.
 */
export function useReportList() {
  return useQuery({
    queryKey: reportKeys.list(),
    queryFn: () => reportApi.list(),
  });
}

/**
 * Fetch a single report definition by ID.
 */
export function useReportDetails(reportId: string) {
  return useQuery({
    queryKey: reportKeys.detail(reportId),
    queryFn: () => reportApi.getById(reportId),
    enabled: !!reportId,
  });
}
```

---

## useReportGeneration Hook

Custom hook that orchestrates report generation across all three formats. For PDF it
builds HTML from a React layout component and sends it to the report service. For XLSX
and CSV it generates the file entirely in the browser.

```typescript
// src/features/reports/hooks/useReportGeneration.ts
import { useState, useCallback } from 'react';
import axios from 'axios';
import { exportToXlsx } from '../utils/exportXlsx';
import { exportToCsv } from '../utils/exportCsv';
import { useNotificationStore } from '../../../store/notification.store';
import type { ReportDataRow, ReportColumn } from '../reports.types';

interface UseReportGenerationReturn {
  /** Generate a PDF by sending HTML to the report service */
  generatePdf: (
    html: string,
    filename: string,
    options?: { landscape?: boolean; format?: 'A4' | 'A3' | 'Letter' | 'Legal' },
  ) => Promise<void>;
  /** Generate an XLSX file client-side */
  generateXlsx: (
    data: ReportDataRow[],
    columns: ReportColumn[],
    filename: string,
  ) => void;
  /** Generate a CSV file client-side */
  generateCsv: (
    data: ReportDataRow[],
    columns: ReportColumn[],
    filename: string,
  ) => void;
  /** Whether a PDF generation request is in flight */
  isGenerating: boolean;
}

/**
 * Hook providing report generation functions for all supported formats.
 *
 * Usage:
 *   const { generatePdf, generateXlsx, generateCsv, isGenerating } = useReportGeneration();
 */
export function useReportGeneration(): UseReportGenerationReturn {
  const [isGenerating, setIsGenerating] = useState(false);
  const { push } = useNotificationStore();

  // ---------------------------------------------------------------------------
  // PDF — send HTML to the Puppeteer report service, download the binary response
  // ---------------------------------------------------------------------------
  const generatePdf = useCallback(
    async (
      html: string,
      filename: string,
      options?: { landscape?: boolean; format?: 'A4' | 'A3' | 'Letter' | 'Legal' },
    ): Promise<void> => {
      setIsGenerating(true);
      try {
        const response = await axios.post(
          '/api/reports/pdf',
          {
            html,
            landscape: options?.landscape ?? false,
            format: options?.format ?? 'A4',
            filename,
          },
          { responseType: 'blob' },
        );

        // Create a temporary download link and trigger the browser download
        const blob = new Blob([response.data], { type: 'application/pdf' });
        const url = URL.createObjectURL(blob);
        const link = document.createElement('a');
        link.href = url;
        link.download = filename;
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        URL.revokeObjectURL(url);

        push('success', `Report "${filename}" downloaded successfully.`);
      } catch (error) {
        const message =
          error instanceof Error ? error.message : 'PDF generation failed.';
        push('error', message);
        throw error;
      } finally {
        setIsGenerating(false);
      }
    },
    [push],
  );

  // ---------------------------------------------------------------------------
  // XLSX — generate entirely in the browser using the xlsx library
  // ---------------------------------------------------------------------------
  const generateXlsx = useCallback(
    (data: ReportDataRow[], columns: ReportColumn[], filename: string): void => {
      try {
        exportToXlsx(data, columns, filename);
        push('success', `Report "${filename}" downloaded successfully.`);
      } catch (error) {
        const message =
          error instanceof Error ? error.message : 'XLSX export failed.';
        push('error', message);
      }
    },
    [push],
  );

  // ---------------------------------------------------------------------------
  // CSV — generate entirely in the browser using PapaParse
  // ---------------------------------------------------------------------------
  const generateCsv = useCallback(
    (data: ReportDataRow[], columns: ReportColumn[], filename: string): void => {
      try {
        exportToCsv(data, columns, filename);
        push('success', `Report "${filename}" downloaded successfully.`);
      } catch (error) {
        const message =
          error instanceof Error ? error.message : 'CSV export failed.';
        push('error', message);
      }
    },
    [push],
  );

  return { generatePdf, generateXlsx, generateCsv, isGenerating };
}
```

---

## Client-Side Export Utilities

### XLSX Export

```typescript
// src/features/reports/utils/exportXlsx.ts
import * as XLSX from 'xlsx';
import type { ReportDataRow, ReportColumn } from '../reports.types';

/**
 * Generate an XLSX file from tabular data and trigger a browser download.
 *
 * @param data     - Array of data rows (each row is a Record<string, unknown>)
 * @param columns  - Column definitions providing key and header label
 * @param filename - Download filename (should end with .xlsx)
 */
export function exportToXlsx(
  data: ReportDataRow[],
  columns: ReportColumn[],
  filename: string,
): void {
  // Build a 2D array: header row + data rows (column-ordered)
  const headers = columns.map((col) => col.header);
  const rows = data.map((row) =>
    columns.map((col) => {
      const value = row[col.key];
      if (value === null || value === undefined) return '';
      return String(value);
    }),
  );

  const worksheetData = [headers, ...rows];
  const worksheet = XLSX.utils.aoa_to_sheet(worksheetData);

  // Auto-size columns based on content length
  worksheet['!cols'] = columns.map((col, index) => {
    const maxLength = Math.max(
      col.header.length,
      ...rows.map((row) => String(row[index] ?? '').length),
    );
    return { wch: Math.min(maxLength + 2, 50) };
  });

  // Bold header row
  for (let colIdx = 0; colIdx < headers.length; colIdx++) {
    const cellRef = XLSX.utils.encode_cell({ r: 0, c: colIdx });
    if (worksheet[cellRef]) {
      worksheet[cellRef].s = { font: { bold: true } };
    }
  }

  const workbook = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(workbook, worksheet, 'Report');

  // Trigger download
  XLSX.writeFile(workbook, filename);
}
```

### CSV Export

```typescript
// src/features/reports/utils/exportCsv.ts
import Papa from 'papaparse';
import type { ReportDataRow, ReportColumn } from '../reports.types';

/**
 * Generate a CSV file from tabular data and trigger a browser download.
 *
 * @param data     - Array of data rows (each row is a Record<string, unknown>)
 * @param columns  - Column definitions providing key and header label
 * @param filename - Download filename (should end with .csv)
 */
export function exportToCsv(
  data: ReportDataRow[],
  columns: ReportColumn[],
  filename: string,
): void {
  // Transform data to column-ordered objects with header labels as keys
  const transformedData = data.map((row) => {
    const ordered: Record<string, string> = {};
    for (const col of columns) {
      const value = row[col.key];
      ordered[col.header] = value === null || value === undefined ? '' : String(value);
    }
    return ordered;
  });

  const csv = Papa.unparse(transformedData, {
    quotes: true,
    header: true,
  });

  // Create a Blob and trigger download
  const blob = new Blob(['\uFEFF' + csv], { type: 'text/csv;charset=utf-8;' });
  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href = url;
  link.download = filename;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  URL.revokeObjectURL(url);
}
```

---

## Report Layout Components

Report layouts are React components that render report content as plain HTML for Puppeteer.
They use inline styles or Tailwind CSS classes — NOT MUI components — because Puppeteer
renders a standalone HTML document without access to the application's Emotion runtime.

### renderReportHtml Utility

This utility takes a React element, renders it to a static HTML string, and wraps it in
a complete HTML document with Tailwind CSS CDN for styling:

```typescript
// src/features/reports/layouts/renderReportHtml.ts
import { renderToStaticMarkup } from 'react-dom/server';
import type { ReactElement } from 'react';

/**
 * Render a React element to a complete HTML document string suitable for Puppeteer.
 *
 * The output HTML includes:
 * - Tailwind CSS via CDN (for styling in Puppeteer's headless Chrome)
 * - Print-optimized CSS (color-adjust, margin reset)
 * - The rendered React component as the document body
 *
 * @param element - React element (report layout component) to render
 * @param title   - Document <title> for the PDF metadata
 * @returns Complete HTML string ready to send to the report service
 */
export function renderReportHtml(element: ReactElement, title: string): string {
  const body = renderToStaticMarkup(element);

  return `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>${escapeHtml(title)}</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <style>
    @page { margin: 0; }
    body {
      -webkit-print-color-adjust: exact;
      print-color-adjust: exact;
    }
    /* Ensure tables do not break across pages mid-row */
    tr { page-break-inside: avoid; }
    /* Force page breaks where explicitly requested */
    .page-break { page-break-after: always; }
  </style>
</head>
<body class="bg-white text-gray-900 text-sm p-8">
  ${body}
</body>
</html>`;
}

/**
 * Escape HTML special characters to prevent XSS in the document title.
 */
function escapeHtml(text: string): string {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
}
```

### Base Report Layout

A wrapper component that provides standard report chrome: company header, report title,
generation date, and page footer.

```tsx
// src/features/reports/layouts/ReportLayout.tsx
import type { ReactNode } from 'react';

interface ReportLayoutProps {
  /** Report title displayed at the top of every page */
  title: string;
  /** Optional subtitle (e.g., filter description) */
  subtitle?: string;
  /** ISO date string for the generation timestamp */
  generatedAt: string;
  /** Report body content */
  children: ReactNode;
}

/**
 * Base report layout with header, title, and footer.
 * Uses Tailwind CSS classes (rendered via CDN in the HTML wrapper).
 * Do NOT use MUI components here — this renders as standalone HTML for Puppeteer.
 */
export function ReportLayout({ title, subtitle, generatedAt, children }: ReportLayoutProps) {
  const formattedDate = new Date(generatedAt).toLocaleString('en-US', {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
  });

  return (
    <div className="min-h-screen flex flex-col">
      {/* Report Header */}
      <header className="border-b border-gray-300 pb-4 mb-6">
        <div className="flex justify-between items-start">
          <div>
            <h1 className="text-xl font-bold text-gray-900">{title}</h1>
            {subtitle && (
              <p className="text-sm text-gray-600 mt-1">{subtitle}</p>
            )}
          </div>
          <div className="text-right text-xs text-gray-500">
            <p>Generated: {formattedDate}</p>
          </div>
        </div>
      </header>

      {/* Report Body */}
      <main className="flex-1">
        {children}
      </main>

      {/* Report Footer */}
      <footer className="border-t border-gray-300 pt-3 mt-8 text-xs text-gray-400 text-center">
        <p>
          {title} &mdash; Generated on {formattedDate}
        </p>
      </footer>
    </div>
  );
}
```

### Table Report Layout

A table-based report layout for standard tabular reports with column headers and data rows:

```tsx
// src/features/reports/layouts/TableReportLayout.tsx
import { ReportLayout } from './ReportLayout';
import type { ReportDataRow, ReportColumn } from '../reports.types';

interface TableReportLayoutProps {
  /** Report title */
  title: string;
  /** Optional subtitle */
  subtitle?: string;
  /** ISO date string for the generation timestamp */
  generatedAt: string;
  /** Column definitions */
  columns: ReportColumn[];
  /** Data rows */
  data: ReportDataRow[];
  /** Whether to show a total row count at the bottom */
  showTotalCount?: boolean;
}

/**
 * Tabular report layout with headers, data rows, and optional record count.
 * Uses Tailwind CSS classes — do NOT use MUI components here.
 */
export function TableReportLayout({
  title,
  subtitle,
  generatedAt,
  columns,
  data,
  showTotalCount = true,
}: TableReportLayoutProps) {
  return (
    <ReportLayout title={title} subtitle={subtitle} generatedAt={generatedAt}>
      <table className="w-full border-collapse text-xs">
        <thead>
          <tr>
            {columns.map((col) => (
              <th
                key={col.key}
                className="border border-gray-300 bg-gray-100 px-3 py-2 font-semibold text-left"
                style={{
                  width: col.width ?? 'auto',
                  textAlign: col.align ?? 'left',
                }}
              >
                {col.header}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {data.length === 0 ? (
            <tr>
              <td
                colSpan={columns.length}
                className="border border-gray-300 px-3 py-8 text-center text-gray-500 italic"
              >
                No data available for the selected parameters.
              </td>
            </tr>
          ) : (
            data.map((row, rowIndex) => (
              <tr key={rowIndex} className={rowIndex % 2 === 1 ? 'bg-gray-50' : ''}>
                {columns.map((col) => (
                  <td
                    key={col.key}
                    className="border border-gray-300 px-3 py-1.5"
                    style={{ textAlign: col.align ?? 'left' }}
                  >
                    {row[col.key] !== null && row[col.key] !== undefined
                      ? String(row[col.key])
                      : ''}
                  </td>
                ))}
              </tr>
            ))
          )}
        </tbody>
      </table>

      {showTotalCount && data.length > 0 && (
        <p className="mt-3 text-xs text-gray-500">
          Total records: {data.length}
        </p>
      )}
    </ReportLayout>
  );
}
```

---

## Report Parameter Schema (Zod)

Dynamic Zod schema generation from report parameter definitions:

```typescript
// src/features/reports/reports.schema.ts
import { z } from 'zod';
import type { ReportParameter, ReportFormat } from './reports.types';

/**
 * Build a Zod schema dynamically from report parameter definitions.
 * This allows each report to declare its own parameters and have them
 * validated at form submission time.
 */
export function buildReportParameterSchema(
  parameters: ReportParameter[],
): z.ZodObject<Record<string, z.ZodTypeAny>> {
  const shape: Record<string, z.ZodTypeAny> = {};

  for (const param of parameters) {
    let fieldSchema: z.ZodTypeAny;

    switch (param.type) {
      case 'text':
        fieldSchema = z.string();
        break;
      case 'date':
        fieldSchema = z.string();
        break;
      case 'number':
        fieldSchema = z.coerce.number();
        break;
      case 'select':
        fieldSchema = z.string();
        break;
      case 'boolean':
        fieldSchema = z.boolean();
        break;
      default:
        fieldSchema = z.string();
    }

    if (param.required) {
      if (param.type === 'boolean') {
        // Booleans are always valid — no min check
        shape[param.name] = fieldSchema;
      } else {
        shape[param.name] = (fieldSchema as z.ZodString).min(
          1,
          `${param.label} is required`,
        );
      }
    } else {
      shape[param.name] = fieldSchema.optional();
    }
  }

  return z.object(shape);
}

/**
 * Schema for the report generation form (format selection).
 */
export const reportGenerateSchema = z.object({
  format: z.enum(['pdf', 'xlsx', 'csv'] as const satisfies readonly ReportFormat[]),
});

export type ReportGenerateFormValues = z.infer<typeof reportGenerateSchema>;
```

---

## Report Pages (MUI Components)

### Report List Page

```tsx
// src/features/reports/pages/ReportListPage.tsx
import { useState, useMemo } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  Box,
  Card,
  CardContent,
  CardActions,
  Button,
  Typography,
  TextField,
  InputAdornment,
  Stack,
  Chip,
  MenuItem,
  Select,
  type SelectChangeEvent,
} from '@mui/material';
import SearchIcon from '@mui/icons-material/Search';
import PictureAsPdfIcon from '@mui/icons-material/PictureAsPdf';
import TableChartIcon from '@mui/icons-material/TableChart';
import DescriptionIcon from '@mui/icons-material/Description';
import { PageHeader } from '../../../shared/components/PageHeader';
import { LoadingOverlay } from '../../../shared/components/LoadingOverlay';
import { EmptyState } from '../../../shared/components/EmptyState';
import { useReportList } from '../hooks/useReports';
import type { ReportFormat } from '../reports.types';

const FORMAT_ICONS: Record<ReportFormat, React.ReactNode> = {
  pdf: <PictureAsPdfIcon fontSize="small" />,
  xlsx: <TableChartIcon fontSize="small" />,
  csv: <DescriptionIcon fontSize="small" />,
};

const FORMAT_COLORS: Record<ReportFormat, 'error' | 'success' | 'info'> = {
  pdf: 'error',
  xlsx: 'success',
  csv: 'info',
};

export function ReportListPage() {
  const navigate = useNavigate();
  const { data: reports, isLoading, isError } = useReportList();

  const [search, setSearch] = useState('');
  const [domainFilter, setDomainFilter] = useState<string>('');

  // Extract unique domains for the filter dropdown
  const domains = useMemo(() => {
    if (!reports) return [];
    return [...new Set(reports.map((r) => r.domain))].sort();
  }, [reports]);

  // Filter reports by search text and domain
  const filteredReports = useMemo(() => {
    if (!reports) return [];
    return reports.filter((report) => {
      const matchesSearch =
        !search ||
        report.name.toLowerCase().includes(search.toLowerCase()) ||
        report.description.toLowerCase().includes(search.toLowerCase());
      const matchesDomain = !domainFilter || report.domain === domainFilter;
      return matchesSearch && matchesDomain;
    });
  }, [reports, search, domainFilter]);

  // Group filtered reports by domain
  const groupedReports = useMemo(() => {
    const groups: Record<string, typeof filteredReports> = {};
    for (const report of filteredReports) {
      if (!groups[report.domain]) {
        groups[report.domain] = [];
      }
      groups[report.domain].push(report);
    }
    return groups;
  }, [filteredReports]);

  if (isLoading) return <LoadingOverlay />;
  if (isError) return <div>Failed to load reports.</div>;

  return (
    <Box>
      <PageHeader
        title="Reports"
        breadcrumbs={[
          { label: 'Dashboard', path: '/dashboard' },
          { label: 'Reports' },
        ]}
      />

      {/* Filters */}
      <Stack direction="row" spacing={2} sx={{ mb: 3 }}>
        <TextField
          size="small"
          placeholder="Search reports..."
          value={search}
          onChange={(e) => setSearch(e.target.value)}
          slotProps={{
            input: {
              startAdornment: (
                <InputAdornment position="start">
                  <SearchIcon fontSize="small" />
                </InputAdornment>
              ),
            },
          }}
          sx={{ minWidth: 300 }}
        />
        <Select
          size="small"
          value={domainFilter}
          onChange={(e: SelectChangeEvent) => setDomainFilter(e.target.value)}
          displayEmpty
          sx={{ minWidth: 200 }}
        >
          <MenuItem value="">All Domains</MenuItem>
          {domains.map((domain) => (
            <MenuItem key={domain} value={domain}>
              {domain}
            </MenuItem>
          ))}
        </Select>
      </Stack>

      {/* Report Cards grouped by domain */}
      {filteredReports.length === 0 ? (
        <EmptyState message="No reports match your search criteria." />
      ) : (
        Object.entries(groupedReports).map(([domain, domainReports]) => (
          <Box key={domain} sx={{ mb: 4 }}>
            <Typography variant="h6" sx={{ mb: 2, fontWeight: 600 }}>
              {domain}
            </Typography>
            <Box
              sx={{
                display: 'grid',
                gridTemplateColumns: 'repeat(auto-fill, minmax(340px, 1fr))',
                gap: 2,
              }}
            >
              {domainReports.map((report) => (
                <Card key={report.reportId} variant="outlined">
                  <CardContent>
                    <Typography variant="subtitle1" fontWeight={600}>
                      {report.name}
                    </Typography>
                    <Typography variant="body2" color="text.secondary" sx={{ mt: 0.5, mb: 1.5 }}>
                      {report.description}
                    </Typography>
                    <Stack direction="row" spacing={0.5}>
                      {report.supportedFormats.map((format) => (
                        <Chip
                          key={format}
                          icon={FORMAT_ICONS[format] as React.ReactElement}
                          label={format.toUpperCase()}
                          color={FORMAT_COLORS[format]}
                          size="small"
                          variant="outlined"
                        />
                      ))}
                    </Stack>
                  </CardContent>
                  <CardActions>
                    <Button
                      size="small"
                      variant="contained"
                      onClick={() => navigate(`/reports/${report.reportId}`)}
                    >
                      Generate
                    </Button>
                  </CardActions>
                </Card>
              ))}
            </Box>
          </Box>
        ))
      )}
    </Box>
  );
}
```

### Report Generate Page

```tsx
// src/features/reports/pages/ReportGeneratePage.tsx
import { useState, useMemo, useCallback } from 'react';
import { useParams, useNavigate } from 'react-router-dom';
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import {
  Box,
  Button,
  Card,
  CardContent,
  Divider,
  Stack,
  ToggleButton,
  ToggleButtonGroup,
  Typography,
} from '@mui/material';
import PictureAsPdfIcon from '@mui/icons-material/PictureAsPdf';
import TableChartIcon from '@mui/icons-material/TableChart';
import DescriptionIcon from '@mui/icons-material/Description';
import PreviewIcon from '@mui/icons-material/Preview';
import DownloadIcon from '@mui/icons-material/Download';
import ArrowBackIcon from '@mui/icons-material/ArrowBack';
import { PageHeader } from '../../../shared/components/PageHeader';
import { LoadingOverlay } from '../../../shared/components/LoadingOverlay';
import { TextFieldController } from '../../../shared/form/TextFieldController';
import { SelectController } from '../../../shared/form/SelectController';
import { useReportDetails } from '../hooks/useReports';
import { useReportGeneration } from '../hooks/useReportGeneration';
import { buildReportParameterSchema } from '../reports.schema';
import { ReportPreviewDialog } from '../components/ReportPreviewDialog';
import type { ReportFormat, ReportParameter } from '../reports.types';

export function ReportGeneratePage() {
  const { reportId } = useParams<{ reportId: string }>();
  const navigate = useNavigate();
  const { data: report, isLoading } = useReportDetails(reportId ?? '');
  const { generatePdf, generateXlsx, generateCsv, isGenerating } = useReportGeneration();

  const [selectedFormat, setSelectedFormat] = useState<ReportFormat>('pdf');
  const [previewOpen, setPreviewOpen] = useState(false);
  const [previewHtml, setPreviewHtml] = useState('');

  // Build dynamic Zod schema from report parameters
  const parameterSchema = useMemo(
    () => (report ? buildReportParameterSchema(report.parameters) : null),
    [report],
  );

  const { control, handleSubmit, getValues } = useForm({
    resolver: parameterSchema ? zodResolver(parameterSchema) : undefined,
    defaultValues: useMemo(() => {
      if (!report) return {};
      const defaults: Record<string, string> = {};
      for (const param of report.parameters) {
        defaults[param.name] = param.defaultValue ?? '';
      }
      return defaults;
    }, [report]),
  });

  /**
   * Build the report HTML for PDF generation or preview.
   * This function must be implemented per-report by the coding agent,
   * using the appropriate layout component and data fetched from the API.
   *
   * The pattern is:
   * 1. Call the module's report data API with the user-provided parameters
   * 2. Pass the data to a report layout component (e.g., TableReportLayout)
   * 3. Call renderReportHtml() to produce the full HTML string
   */
  const buildReportHtml = useCallback(
    async (_parameters: Record<string, unknown>): Promise<string> => {
      // This is a placeholder — each report implementation provides its own
      // buildReportHtml function that:
      // 1. Fetches data from the appropriate API endpoint with the parameters
      // 2. Renders a layout component (TableReportLayout, etc.) with the data
      // 3. Returns the HTML string via renderReportHtml()
      throw new Error(
        'buildReportHtml must be implemented for each report. ' +
        'See the Module Integration section for a complete example.',
      );
    },
    [],
  );

  const handleGenerate = handleSubmit(async (values) => {
    if (!report) return;

    const timestamp = new Date().toISOString().replace(/[:.]/g, '-').slice(0, 19);
    const baseFilename = `${report.reportId}-${timestamp}`;

    switch (selectedFormat) {
      case 'pdf': {
        const html = await buildReportHtml(values);
        await generatePdf(html, `${baseFilename}.pdf`);
        break;
      }
      case 'xlsx': {
        // For XLSX/CSV, the coding agent implements data fetching and passes
        // it to generateXlsx/generateCsv with the appropriate columns
        break;
      }
      case 'csv': {
        break;
      }
    }
  });

  const handlePreview = handleSubmit(async (values) => {
    const html = await buildReportHtml(values);
    setPreviewHtml(html);
    setPreviewOpen(true);
  });

  if (isLoading || !report) return <LoadingOverlay />;

  return (
    <Box>
      <PageHeader
        title={report.name}
        breadcrumbs={[
          { label: 'Dashboard', path: '/dashboard' },
          { label: 'Reports', path: '/reports' },
          { label: report.name },
        ]}
        actions={
          <Button
            variant="outlined"
            startIcon={<ArrowBackIcon />}
            onClick={() => navigate('/reports')}
          >
            Back to Reports
          </Button>
        }
      />

      <Card>
        <CardContent>
          <Typography variant="body2" color="text.secondary" sx={{ mb: 3 }}>
            {report.description}
          </Typography>

          {/* Parameter Form */}
          {report.parameters.length > 0 && (
            <>
              <Typography variant="subtitle2" sx={{ mb: 2, fontWeight: 600 }}>
                Report Parameters
              </Typography>
              <Stack spacing={2} sx={{ mb: 3, maxWidth: 600 }}>
                {report.parameters.map((param: ReportParameter) =>
                  param.type === 'select' && param.options ? (
                    <SelectController
                      key={param.name}
                      name={param.name}
                      control={control}
                      label={param.label}
                      options={param.options.map((opt) => ({
                        value: opt.value,
                        label: opt.label,
                      }))}
                    />
                  ) : (
                    <TextFieldController
                      key={param.name}
                      name={param.name}
                      control={control}
                      label={param.label}
                      required={param.required}
                      type={param.type === 'date' ? 'date' : param.type === 'number' ? 'number' : 'text'}
                      slotProps={{
                        inputLabel: param.type === 'date' ? { shrink: true } : undefined,
                      }}
                    />
                  ),
                )}
              </Stack>
              <Divider sx={{ mb: 3 }} />
            </>
          )}

          {/* Format Selection */}
          <Typography variant="subtitle2" sx={{ mb: 2, fontWeight: 600 }}>
            Export Format
          </Typography>
          <ToggleButtonGroup
            value={selectedFormat}
            exclusive
            onChange={(_, value) => {
              if (value) setSelectedFormat(value as ReportFormat);
            }}
            size="small"
            sx={{ mb: 3 }}
          >
            {report.supportedFormats.includes('pdf') && (
              <ToggleButton value="pdf">
                <PictureAsPdfIcon sx={{ mr: 0.5 }} fontSize="small" />
                PDF
              </ToggleButton>
            )}
            {report.supportedFormats.includes('xlsx') && (
              <ToggleButton value="xlsx">
                <TableChartIcon sx={{ mr: 0.5 }} fontSize="small" />
                XLSX
              </ToggleButton>
            )}
            {report.supportedFormats.includes('csv') && (
              <ToggleButton value="csv">
                <DescriptionIcon sx={{ mr: 0.5 }} fontSize="small" />
                CSV
              </ToggleButton>
            )}
          </ToggleButtonGroup>

          {/* Action Buttons */}
          <Stack direction="row" spacing={2}>
            {selectedFormat === 'pdf' && (
              <Button
                variant="outlined"
                startIcon={<PreviewIcon />}
                onClick={handlePreview}
                disabled={isGenerating}
              >
                Preview
              </Button>
            )}
            <Button
              variant="contained"
              startIcon={<DownloadIcon />}
              onClick={handleGenerate}
              loading={isGenerating}
            >
              Generate {selectedFormat.toUpperCase()}
            </Button>
          </Stack>
        </CardContent>
      </Card>

      {/* Preview Dialog */}
      <ReportPreviewDialog
        open={previewOpen}
        onClose={() => setPreviewOpen(false)}
        html={previewHtml}
        title={report.name}
      />
    </Box>
  );
}
```

---

## Report Preview Dialog

A MUI Dialog that renders the report layout as an HTML preview in an iframe:

```tsx
// src/features/reports/components/ReportPreviewDialog.tsx
import {
  Dialog,
  DialogTitle,
  DialogContent,
  DialogActions,
  Button,
  IconButton,
  Box,
} from '@mui/material';
import CloseIcon from '@mui/icons-material/Close';

interface ReportPreviewDialogProps {
  /** Whether the dialog is open */
  open: boolean;
  /** Callback to close the dialog */
  onClose: () => void;
  /** The complete HTML string to preview */
  html: string;
  /** Report title shown in the dialog header */
  title: string;
}

/**
 * Full-screen dialog that renders a report HTML preview inside an iframe.
 * The iframe uses a data URI to display the report without making a network request.
 */
export function ReportPreviewDialog({
  open,
  onClose,
  html,
  title,
}: ReportPreviewDialogProps) {
  // Encode the HTML as a data URI for the iframe src
  const dataUri = `data:text/html;charset=utf-8,${encodeURIComponent(html)}`;

  return (
    <Dialog open={open} onClose={onClose} fullScreen>
      <DialogTitle sx={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
        Preview: {title}
        <IconButton edge="end" onClick={onClose} aria-label="close">
          <CloseIcon />
        </IconButton>
      </DialogTitle>
      <DialogContent dividers sx={{ p: 0 }}>
        <Box
          component="iframe"
          src={dataUri}
          sx={{
            width: '100%',
            height: '100%',
            border: 'none',
          }}
          title={`Preview: ${title}`}
        />
      </DialogContent>
      <DialogActions>
        <Button onClick={onClose} variant="outlined">
          Close
        </Button>
      </DialogActions>
    </Dialog>
  );
}
```

---

## Zustand Report Store (Optional)

If reports need client-side state beyond TanStack Query cache (e.g., persisting the last
selected format or recent report history):

```typescript
// src/features/reports/store/report.store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import type { ReportFormat } from '../reports.types';

interface ReportState {
  /** Last selected export format — remembered across page navigations */
  lastFormat: ReportFormat;
  /** Recently generated reports for quick access */
  recentReports: RecentReport[];
  setLastFormat: (format: ReportFormat) => void;
  addRecentReport: (report: RecentReport) => void;
  clearRecent: () => void;
}

interface RecentReport {
  reportId: string;
  name: string;
  format: ReportFormat;
  generatedAt: string;
}

const MAX_RECENT = 10;

export const useReportStore = create<ReportState>()(
  persist(
    (set) => ({
      lastFormat: 'pdf',
      recentReports: [],

      setLastFormat: (format) => set({ lastFormat: format }),

      addRecentReport: (report) =>
        set((state) => ({
          recentReports: [report, ...state.recentReports].slice(0, MAX_RECENT),
        })),

      clearRecent: () => set({ recentReports: [] }),
    }),
    { name: 'report-preferences' },
  ),
);
```

---

## React Router Integration

```typescript
// Add to src/router/index.tsx

import { lazy } from 'react';

const ReportListPage = lazy(() =>
  import('../features/reports/pages/ReportListPage').then((m) => ({
    default: m.ReportListPage,
  })),
);

const ReportGeneratePage = lazy(() =>
  import('../features/reports/pages/ReportGeneratePage').then((m) => ({
    default: m.ReportGeneratePage,
  })),
);

// Inside the protected dashboard routes children array:
// {
//   path: '/reports',
//   element: (
//     <SuspenseWrapper>
//       <ReportListPage />
//     </SuspenseWrapper>
//   ),
// },
// {
//   path: '/reports/:reportId',
//   element: (
//     <SuspenseWrapper>
//       <ReportGeneratePage />
//     </SuspenseWrapper>
//   ),
// },
```

Navigation item for the sidebar:

```typescript
// Add to src/shared/components/navigationConfig.ts
import AssessmentIcon from '@mui/icons-material/Assessment';

// {
//   label: 'Reports',
//   path: '/reports',
//   icon: AssessmentIcon,
//   // Reports are typically available to all authenticated users
// },
```

---

## Module Integration — Complete Sample Report

This section demonstrates how a module defines and registers a report. The example shows
a Staff List Report within a hypothetical `staff` feature module.

### Step 1: Report Data API

```typescript
// src/features/staff/api/staffApi.ts (add report data endpoint)
import { axiosInstance } from '../../../lib/api/axiosInstance';
import type { ReportDataRow } from '../../reports/reports.types';

export const staffApi = {
  // ... existing CRUD methods ...

  /**
   * Fetch staff data formatted for report generation.
   * The backend returns a flat array suitable for tabular display.
   */
  getReportData: async (params: {
    department?: string;
    status?: string;
  }): Promise<ReportDataRow[]> => {
    const { data } = await axiosInstance.get<ReportDataRow[]>('/v1/staff/report', {
      params,
    });
    return data;
  },
};
```

### Step 2: Report Definition Constant

```typescript
// src/features/staff/reports/staffListReport.ts
import type { ReportDefinition, ReportColumn } from '../../reports/reports.types';

export const STAFF_LIST_REPORT: ReportDefinition = {
  reportId: 'staff-list-report',
  name: 'Staff List Report',
  description: 'List all staff members with their department, position, and status.',
  domain: 'Human Resources',
  parameters: [
    {
      name: 'department',
      label: 'Department',
      type: 'select',
      required: false,
      options: [
        { label: 'Engineering', value: 'ENGINEERING' },
        { label: 'Marketing', value: 'MARKETING' },
        { label: 'Sales', value: 'SALES' },
        { label: 'Human Resources', value: 'HR' },
      ],
    },
    {
      name: 'status',
      label: 'Status',
      type: 'select',
      required: false,
      options: [
        { label: 'Active', value: 'ACTIVE' },
        { label: 'Inactive', value: 'INACTIVE' },
        { label: 'On Leave', value: 'ON_LEAVE' },
      ],
    },
  ],
  supportedFormats: ['pdf', 'xlsx', 'csv'],
};

/**
 * Column definitions for the Staff List Report.
 * Used by both the PDF layout and client-side XLSX/CSV exports.
 */
export const STAFF_LIST_COLUMNS: ReportColumn[] = [
  { key: 'name', header: 'Name', width: '20%' },
  { key: 'email', header: 'Email', width: '22%' },
  { key: 'department', header: 'Department', width: '15%' },
  { key: 'position', header: 'Position', width: '18%' },
  { key: 'status', header: 'Status', width: '10%' },
  { key: 'joinedDate', header: 'Joined Date', width: '15%', align: 'center' },
];
```

### Step 3: Report Layout Component

```tsx
// src/features/staff/reports/StaffListReportLayout.tsx
import { TableReportLayout } from '../../reports/layouts/TableReportLayout';
import { STAFF_LIST_COLUMNS } from './staffListReport';
import type { ReportDataRow } from '../../reports/reports.types';

interface StaffListReportLayoutProps {
  data: ReportDataRow[];
  parameters: { department?: string; status?: string };
  generatedAt: string;
}

/**
 * Staff List Report layout component for PDF rendering.
 * This component is rendered to static HTML via renderReportHtml().
 */
export function StaffListReportLayout({
  data,
  parameters,
  generatedAt,
}: StaffListReportLayoutProps) {
  const subtitle = [
    parameters.department && `Department: ${parameters.department}`,
    parameters.status && `Status: ${parameters.status}`,
  ]
    .filter(Boolean)
    .join(' | ') || 'All Staff';

  return (
    <TableReportLayout
      title="Staff List Report"
      subtitle={subtitle}
      generatedAt={generatedAt}
      columns={STAFF_LIST_COLUMNS}
      data={data}
      showTotalCount
    />
  );
}
```

### Step 4: Report Generation Hook (Module-Specific)

```typescript
// src/features/staff/reports/useStaffListReport.ts
import { useCallback } from 'react';
import { staffApi } from '../api/staffApi';
import { useReportGeneration } from '../../reports/hooks/useReportGeneration';
import { renderReportHtml } from '../../reports/layouts/renderReportHtml';
import { StaffListReportLayout } from './StaffListReportLayout';
import { STAFF_LIST_COLUMNS } from './staffListReport';
import type { ReportFormat } from '../../reports/reports.types';

interface UseStaffListReportReturn {
  generate: (
    parameters: { department?: string; status?: string },
    format: ReportFormat,
  ) => Promise<void>;
  preview: (parameters: { department?: string; status?: string }) => Promise<string>;
  isGenerating: boolean;
}

/**
 * Hook that encapsulates the complete staff list report generation flow.
 * Fetches data, builds the layout, and triggers the appropriate export.
 */
export function useStaffListReport(): UseStaffListReportReturn {
  const { generatePdf, generateXlsx, generateCsv, isGenerating } = useReportGeneration();

  const fetchData = useCallback(
    async (parameters: { department?: string; status?: string }) => {
      return staffApi.getReportData({
        department: parameters.department || undefined,
        status: parameters.status || undefined,
      });
    },
    [],
  );

  const buildHtml = useCallback(
    async (parameters: { department?: string; status?: string }): Promise<string> => {
      const data = await fetchData(parameters);
      const element = (
        <StaffListReportLayout
          data={data}
          parameters={parameters}
          generatedAt={new Date().toISOString()}
        />
      );
      return renderReportHtml(element, 'Staff List Report');
    },
    [fetchData],
  );

  const generate = useCallback(
    async (
      parameters: { department?: string; status?: string },
      format: ReportFormat,
    ): Promise<void> => {
      const timestamp = new Date().toISOString().replace(/[:.]/g, '-').slice(0, 19);
      const baseFilename = `staff-list-report-${timestamp}`;

      switch (format) {
        case 'pdf': {
          const html = await buildHtml(parameters);
          await generatePdf(html, `${baseFilename}.pdf`);
          break;
        }
        case 'xlsx': {
          const data = await fetchData(parameters);
          generateXlsx(data, STAFF_LIST_COLUMNS, `${baseFilename}.xlsx`);
          break;
        }
        case 'csv': {
          const data = await fetchData(parameters);
          generateCsv(data, STAFF_LIST_COLUMNS, `${baseFilename}.csv`);
          break;
        }
      }
    },
    [buildHtml, fetchData, generatePdf, generateXlsx, generateCsv],
  );

  const preview = useCallback(
    async (parameters: { department?: string; status?: string }): Promise<string> => {
      return buildHtml(parameters);
    },
    [buildHtml],
  );

  return { generate, preview, isGenerating };
}
```

---

## Directory Structure (Report Feature)

```
src/features/
├── reports/                              # Shared report infrastructure
│   ├── api/
│   │   └── reportApi.ts                  # Report catalogue API
│   ├── components/
│   │   └── ReportPreviewDialog.tsx       # Full-screen preview dialog
│   ├── hooks/
│   │   ├── useReports.ts                 # TanStack Query hooks for report list
│   │   └── useReportGeneration.ts        # PDF/XLSX/CSV generation hook
│   ├── layouts/
│   │   ├── renderReportHtml.ts           # ReactDOMServer HTML wrapper utility
│   │   ├── ReportLayout.tsx              # Base report layout (header + footer)
│   │   └── TableReportLayout.tsx         # Tabular report layout
│   ├── pages/
│   │   ├── ReportListPage.tsx            # Report catalogue page
│   │   └── ReportGeneratePage.tsx        # Parameter form + generate page
│   ├── store/
│   │   └── report.store.ts              # (Optional) Persisted report preferences
│   ├── utils/
│   │   ├── exportXlsx.ts                # Client-side XLSX export
│   │   └── exportCsv.ts                 # Client-side CSV export
│   ├── reports.types.ts                  # Shared report type definitions
│   ├── reports.schema.ts                 # Dynamic Zod schema builder
│   └── index.ts                          # Public API re-exports
│
├── staff/                                # Example module with reports
│   ├── api/
│   │   └── staffApi.ts                   # Includes getReportData() method
│   ├── reports/
│   │   ├── staffListReport.ts            # Report definition + column config
│   │   ├── StaffListReportLayout.tsx     # PDF layout component
│   │   └── useStaffListReport.ts         # Module-specific report generation hook
│   └── ...
```

---

## Key Design Decisions

| Decision | Rationale |
|---|---|
| **Server-side Puppeteer for PDF** | Full CSS support (Tailwind, flexbox, grid), pixel-perfect rendering, support for page breaks, headers/footers, and complex layouts. Browser-based PDF libraries (jsPDF, html2canvas) produce inferior results. |
| **Client-side CSV/XLSX** | No server round-trip needed — data is already available from the API or TanStack Query cache. `xlsx` and `PapaParse` are mature, well-maintained libraries. Instant export with no network latency. |
| **ReactDOMServer for HTML generation** | Consistent React component model for report layouts. Full TypeScript type safety. Report layouts are just React components — no separate template language. |
| **Tailwind CSS CDN in report HTML** | Puppeteer renders a standalone HTML document with no access to the application's Emotion/MUI runtime CSS. Tailwind CDN provides utility classes for styling without any build step. |
| **Separate report service** | Puppeteer requires Node.js + headless Chrome which cannot run in the browser. A lightweight Express server keeps the concern isolated and can be independently scaled. |
| **Dynamic Zod schemas** | Report parameters are defined as data (ReportDefinition), so the Zod schema is built dynamically at runtime. This allows new reports to be added without changing the form infrastructure. |
| **Report layouts avoid MUI** | MUI relies on Emotion CSS-in-JS which requires a browser runtime. Puppeteer renders raw HTML — inline styles and Tailwind classes work reliably without a JS runtime. |

---

## Report Service Deployment

### Development

- Start the Vite dev server: `npm run dev` (port {{DEV_PORT}})
- Start the report service: `cd report-service && npm run dev` (port 3001)
- Vite proxy forwards `/api/reports` to `http://localhost:3001`

### Production

- Deploy the report service as a sidecar container or separate microservice
- The React SPA connects to the report service via the `VITE_REPORT_SERVICE_URL`
  environment variable (or through an API gateway / ingress rule)

### Environment Variables (Production)

```env
# React SPA (.env.production)
VITE_REPORT_SERVICE_URL=${REPORT_SERVICE_URL}

# Report Service
REPORT_SERVICE_PORT=3001
```

### Dockerfile (Report Service)

```dockerfile
# report-service/Dockerfile
FROM node:22-slim

# Install Chromium dependencies required by Puppeteer
RUN apt-get update && apt-get install -y \
    chromium \
    fonts-liberation \
    libappindicator3-1 \
    libasound2 \
    libatk-bridge2.0-0 \
    libatk1.0-0 \
    libcups2 \
    libdbus-1-3 \
    libdrm2 \
    libgbm1 \
    libnspr4 \
    libnss3 \
    libxcomposite1 \
    libxdamage1 \
    libxrandr2 \
    xdg-utils \
    --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

# Tell Puppeteer to use the system-installed Chromium
ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true
ENV PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --production

COPY dist/ ./dist/

EXPOSE 3001

USER node

CMD ["node", "dist/server.js"]
```

---

## Operational Notes

### Chrome Memory Management

Puppeteer's headless Chrome can consume significant memory for large reports. Mitigations:

- **Browser singleton**: Reuse a single Chrome instance across requests (implemented in
  the server above). Each request opens a new page (tab) and closes it when done.
- **Memory limit**: Set `--max-old-space-size` for the Node.js process and container
  memory limits in Kubernetes.
- **Page cleanup**: Always close pages in a `finally` block to prevent memory leaks
  from failed requests.

### Concurrent Report Generation

For production workloads with many concurrent report requests:

- **Semaphore pattern**: Limit the number of concurrent Puppeteer pages to prevent
  Chrome from running out of memory. Example: maximum 5 concurrent pages.
- **Queue**: For high-volume scenarios, use a job queue (e.g., BullMQ with Redis) to
  serialize report generation and return results asynchronously.

```typescript
// Example: Simple concurrency limiter for the report service
class Semaphore {
  private current = 0;
  private queue: (() => void)[] = [];

  constructor(private readonly max: number) {}

  async acquire(): Promise<void> {
    if (this.current < this.max) {
      this.current++;
      return;
    }
    return new Promise<void>((resolve) => {
      this.queue.push(resolve);
    });
  }

  release(): void {
    this.current--;
    const next = this.queue.shift();
    if (next) {
      this.current++;
      next();
    }
  }
}

// Usage in the report service:
// const semaphore = new Semaphore(5);
// app.post('/api/reports/pdf', async (req, res) => {
//   await semaphore.acquire();
//   try { /* generate PDF */ }
//   finally { semaphore.release(); }
// });
```

### Graceful Shutdown

The report service implements `SIGTERM` and `SIGINT` handlers that:
1. Stop accepting new HTTP connections
2. Close the Puppeteer browser instance (terminates headless Chrome)
3. Exit the process cleanly

This ensures Kubernetes can gracefully terminate the pod during deployments.

### Health Check

The `/api/reports/health` endpoint returns `{ "status": "ok" }` and can be used as
a Kubernetes liveness/readiness probe.
