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
- **Tailwind/Headless UI report UI** — report list page, parameter forms (React Hook Form + Zod),
  preview dialog (Headless UI `Dialog`), and format selection
- **Report HTML templates** use Tailwind CSS CDN for standalone rendering inside Puppeteer
- All report layouts are programmatically developed by the AI coding agent — no manual
  template authoring required

> **Note on consistency:** Unlike the MUI variant (where report layouts deliberately avoided
> MUI's Emotion runtime), this Tailwind variant uses the *same* utility-class vocabulary in
> both the app UI and the Puppeteer report HTML. The app's compiled Tailwind CSS does not reach
> the standalone Puppeteer document, so the report HTML loads Tailwind via CDN — but the classes
> you write are identical, which keeps report authoring intuitive.

---

## Architecture Overview

```
React SPA (Browser)                    Report Service (Node.js)
┌──────────────────────────┐          ┌──────────────────────────┐
│  ReportListPage          │          │  Express.js              │
│  ReportGeneratePage      │   POST   │  POST /api/reports/pdf   │
│  ReportPreviewDialog     │─────────►│    receives: HTML string │
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

**Key design rule:** Report layout components are plain React components that use Tailwind
CSS classes (loaded via CDN in the HTML wrapper). They render as a standalone HTML document
inside Puppeteer that has no access to the application's compiled CSS bundle.

---

## npm Dependencies

### React SPA (main application package.json)

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
      args: ['--no-sandbox', '--disable-setuid-sandbox', '--disable-dev-shm-usage', '--disable-gpu'],
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
  const { html, landscape = false, format = 'A4', filename = 'report.pdf' } = req.body;

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
      margin: { top: '10mm', bottom: '10mm', left: '10mm', right: '10mm' },
    });

    res.setHeader('Content-Type', 'application/pdf');
    res.setHeader('Content-Disposition', `attachment; filename="${filename}"`);
    res.setHeader('Content-Length', String(pdf.byteLength));
    res.send(pdf);
  } catch (error) {
    console.error('PDF generation failed:', error);
    res.status(500).json({ error: 'PDF generation failed.' });
  } finally {
    if (page) await page.close();
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

```typescript
// vite.config.ts (add to existing server.proxy)
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: { alias: { '@': path.resolve(__dirname, './src') } },
  server: {
    port: {{DEV_PORT}},
    proxy: {
      '/api/v1': { target: '{{API_BASE_URL_DEV}}', changeOrigin: true },
      '/api/reports': { target: 'http://localhost:3001', changeOrigin: true },
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
  reportId: string;
  name: string;
  description: string;
  domain: string;
  parameters: ReportParameter[];
  supportedFormats: ReportFormat[];
}

export interface ReportParameter {
  name: string;
  label: string;
  /** Input type — maps to Tailwind/Headless UI form controls */
  type: 'text' | 'date' | 'number' | 'select' | 'boolean';
  required: boolean;
  options?: ReportParameterOption[];
  defaultValue?: string;
}

export interface ReportParameterOption {
  label: string;
  value: string;
}

export type ReportFormat = 'pdf' | 'xlsx' | 'csv';

export type ReportDataRow = Record<string, unknown>;

export interface ReportColumn {
  key: string;
  header: string;
  width?: string;
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
  list: async (): Promise<ReportDefinition[]> => {
    const { data } = await axiosInstance.get<ReportDefinition[]>('/v1/reports');
    return data;
  },
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

export function useReportList() {
  return useQuery({ queryKey: reportKeys.list(), queryFn: () => reportApi.list() });
}

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
  generatePdf: (
    html: string,
    filename: string,
    options?: { landscape?: boolean; format?: 'A4' | 'A3' | 'Letter' | 'Legal' },
  ) => Promise<void>;
  generateXlsx: (data: ReportDataRow[], columns: ReportColumn[], filename: string) => void;
  generateCsv: (data: ReportDataRow[], columns: ReportColumn[], filename: string) => void;
  isGenerating: boolean;
}

export function useReportGeneration(): UseReportGenerationReturn {
  const [isGenerating, setIsGenerating] = useState(false);
  const push = useNotificationStore((s) => s.push);

  // PDF — send HTML to the Puppeteer report service, download the binary response
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
          { html, landscape: options?.landscape ?? false, format: options?.format ?? 'A4', filename },
          { responseType: 'blob' },
        );

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
        const message = error instanceof Error ? error.message : 'PDF generation failed.';
        push('error', message);
        throw error;
      } finally {
        setIsGenerating(false);
      }
    },
    [push],
  );

  // XLSX — generate entirely in the browser using the xlsx library
  const generateXlsx = useCallback(
    (data: ReportDataRow[], columns: ReportColumn[], filename: string): void => {
      try {
        exportToXlsx(data, columns, filename);
        push('success', `Report "${filename}" downloaded successfully.`);
      } catch (error) {
        push('error', error instanceof Error ? error.message : 'XLSX export failed.');
      }
    },
    [push],
  );

  // CSV — generate entirely in the browser using PapaParse
  const generateCsv = useCallback(
    (data: ReportDataRow[], columns: ReportColumn[], filename: string): void => {
      try {
        exportToCsv(data, columns, filename);
        push('success', `Report "${filename}" downloaded successfully.`);
      } catch (error) {
        push('error', error instanceof Error ? error.message : 'CSV export failed.');
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

export function exportToXlsx(data: ReportDataRow[], columns: ReportColumn[], filename: string): void {
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

  worksheet['!cols'] = columns.map((col, index) => {
    const maxLength = Math.max(col.header.length, ...rows.map((row) => String(row[index] ?? '').length));
    return { wch: Math.min(maxLength + 2, 50) };
  });

  for (let colIdx = 0; colIdx < headers.length; colIdx++) {
    const cellRef = XLSX.utils.encode_cell({ r: 0, c: colIdx });
    if (worksheet[cellRef]) worksheet[cellRef].s = { font: { bold: true } };
  }

  const workbook = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(workbook, worksheet, 'Report');
  XLSX.writeFile(workbook, filename);
}
```

### CSV Export

```typescript
// src/features/reports/utils/exportCsv.ts
import Papa from 'papaparse';
import type { ReportDataRow, ReportColumn } from '../reports.types';

export function exportToCsv(data: ReportDataRow[], columns: ReportColumn[], filename: string): void {
  const transformedData = data.map((row) => {
    const ordered: Record<string, string> = {};
    for (const col of columns) {
      const value = row[col.key];
      ordered[col.header] = value === null || value === undefined ? '' : String(value);
    }
    return ordered;
  });

  const csv = Papa.unparse(transformedData, { quotes: true, header: true });

  const blob = new Blob(['﻿' + csv], { type: 'text/csv;charset=utf-8;' });
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
They use Tailwind CSS classes (loaded via CDN in the HTML wrapper).

### renderReportHtml Utility

```typescript
// src/features/reports/layouts/renderReportHtml.ts
import { renderToStaticMarkup } from 'react-dom/server';
import type { ReactElement } from 'react';

/**
 * Render a React element to a complete HTML document string suitable for Puppeteer.
 * The output includes Tailwind CSS via CDN and print-optimized CSS.
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
    body { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
    tr { page-break-inside: avoid; }
    .page-break { page-break-after: always; }
  </style>
</head>
<body class="bg-white text-gray-900 text-sm p-8">
  ${body}
</body>
</html>`;
}

function escapeHtml(text: string): string {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
}
```

### Base Report Layout

```tsx
// src/features/reports/layouts/ReportLayout.tsx
import type { ReactNode } from 'react';

interface ReportLayoutProps {
  title: string;
  subtitle?: string;
  generatedAt: string;
  children: ReactNode;
}

/**
 * Base report layout with header, title, and footer.
 * Uses Tailwind CSS classes (rendered via CDN in the HTML wrapper).
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
      <header className="border-b border-gray-300 pb-4 mb-6">
        <div className="flex justify-between items-start">
          <div>
            <h1 className="text-xl font-bold text-gray-900">{title}</h1>
            {subtitle && <p className="text-sm text-gray-600 mt-1">{subtitle}</p>}
          </div>
          <div className="text-right text-xs text-gray-500">
            <p>Generated: {formattedDate}</p>
          </div>
        </div>
      </header>

      <main className="flex-1">{children}</main>

      <footer className="border-t border-gray-300 pt-3 mt-8 text-xs text-gray-400 text-center">
        <p>{title} &mdash; Generated on {formattedDate}</p>
      </footer>
    </div>
  );
}
```

### Table Report Layout

```tsx
// src/features/reports/layouts/TableReportLayout.tsx
import { ReportLayout } from './ReportLayout';
import type { ReportDataRow, ReportColumn } from '../reports.types';

interface TableReportLayoutProps {
  title: string;
  subtitle?: string;
  generatedAt: string;
  columns: ReportColumn[];
  data: ReportDataRow[];
  showTotalCount?: boolean;
}

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
                style={{ width: col.width ?? 'auto', textAlign: col.align ?? 'left' }}
              >
                {col.header}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {data.length === 0 ? (
            <tr>
              <td colSpan={columns.length} className="border border-gray-300 px-3 py-8 text-center text-gray-500 italic">
                No data available for the selected parameters.
              </td>
            </tr>
          ) : (
            data.map((row, rowIndex) => (
              <tr key={rowIndex} className={rowIndex % 2 === 1 ? 'bg-gray-50' : ''}>
                {columns.map((col) => (
                  <td key={col.key} className="border border-gray-300 px-3 py-1.5" style={{ textAlign: col.align ?? 'left' }}>
                    {row[col.key] !== null && row[col.key] !== undefined ? String(row[col.key]) : ''}
                  </td>
                ))}
              </tr>
            ))
          )}
        </tbody>
      </table>

      {showTotalCount && data.length > 0 && (
        <p className="mt-3 text-xs text-gray-500">Total records: {data.length}</p>
      )}
    </ReportLayout>
  );
}
```

---

## Report Parameter Schema (Zod)

```typescript
// src/features/reports/reports.schema.ts
import { z } from 'zod';
import type { ReportParameter, ReportFormat } from './reports.types';

export function buildReportParameterSchema(
  parameters: ReportParameter[],
): z.ZodObject<Record<string, z.ZodTypeAny>> {
  const shape: Record<string, z.ZodTypeAny> = {};

  for (const param of parameters) {
    let fieldSchema: z.ZodTypeAny;

    switch (param.type) {
      case 'number':
        fieldSchema = z.coerce.number();
        break;
      case 'boolean':
        fieldSchema = z.boolean();
        break;
      case 'text':
      case 'date':
      case 'select':
      default:
        fieldSchema = z.string();
    }

    if (param.required) {
      if (param.type === 'boolean') {
        shape[param.name] = fieldSchema;
      } else {
        shape[param.name] = (fieldSchema as z.ZodString).min(1, `${param.label} is required`);
      }
    } else {
      shape[param.name] = fieldSchema.optional();
    }
  }

  return z.object(shape);
}

export const reportGenerateSchema = z.object({
  format: z.enum(['pdf', 'xlsx', 'csv'] as const satisfies readonly ReportFormat[]),
});

export type ReportGenerateFormValues = z.infer<typeof reportGenerateSchema>;
```

---

## Report Pages (Tailwind + Headless UI)

### Report List Page

```tsx
// src/features/reports/pages/ReportListPage.tsx
import { useState, useMemo } from 'react';
import { useNavigate } from 'react-router-dom';
import { MagnifyingGlassIcon, DocumentTextIcon, TableCellsIcon, DocumentIcon } from '@heroicons/react/24/outline';
import { Button } from '../../../shared/components/Button';
import { Card } from '../../../shared/components/Card';
import { PageHeader } from '../../../shared/components/PageHeader';
import { LoadingOverlay } from '../../../shared/components/LoadingOverlay';
import { EmptyState } from '../../../shared/components/EmptyState';
import { StatusBadge } from '../../../shared/components/StatusBadge';
import { useReportList } from '../hooks/useReports';
import type { ReportFormat } from '../reports.types';

const FORMAT_ICON: Record<ReportFormat, typeof DocumentTextIcon> = {
  pdf: DocumentTextIcon,
  xlsx: TableCellsIcon,
  csv: DocumentIcon,
};

const FORMAT_VARIANT: Record<ReportFormat, 'danger' | 'success' | 'primary'> = {
  pdf: 'danger',
  xlsx: 'success',
  csv: 'primary',
};

export function ReportListPage() {
  const navigate = useNavigate();
  const { data: reports, isLoading, isError } = useReportList();

  const [search, setSearch] = useState('');
  const [domainFilter, setDomainFilter] = useState('');

  const domains = useMemo(() => {
    if (!reports) return [];
    return [...new Set(reports.map((r) => r.domain))].sort();
  }, [reports]);

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

  const groupedReports = useMemo(() => {
    const groups: Record<string, typeof filteredReports> = {};
    for (const report of filteredReports) {
      (groups[report.domain] ??= []).push(report);
    }
    return groups;
  }, [filteredReports]);

  if (isLoading) return <LoadingOverlay />;
  if (isError) return <p className="text-danger">Failed to load reports.</p>;

  return (
    <div>
      <PageHeader title="Reports" breadcrumbs={[{ label: 'Dashboard', path: '/dashboard' }, { label: 'Reports' }]} />

      {/* Filters */}
      <div className="mb-6 flex flex-wrap gap-3">
        <div className="relative">
          <MagnifyingGlassIcon className="pointer-events-none absolute left-3 top-2.5 h-4 w-4 text-slate-400" />
          <input
            value={search}
            onChange={(e) => setSearch(e.target.value)}
            placeholder="Search reports…"
            className="w-72 rounded border border-slate-300 bg-surface py-2 pl-9 pr-3 text-sm focus:border-primary focus:ring-1 focus:ring-primary dark:border-slate-600"
          />
        </div>
        <select
          value={domainFilter}
          onChange={(e) => setDomainFilter(e.target.value)}
          className="rounded border border-slate-300 bg-surface px-3 py-2 text-sm focus:border-primary focus:ring-1 focus:ring-primary dark:border-slate-600"
        >
          <option value="">All Domains</option>
          {domains.map((domain) => (
            <option key={domain} value={domain}>
              {domain}
            </option>
          ))}
        </select>
      </div>

      {filteredReports.length === 0 ? (
        <EmptyState message="No reports match your search criteria." />
      ) : (
        Object.entries(groupedReports).map(([domain, domainReports]) => (
          <section key={domain} className="mb-8">
            <h2 className="mb-3 text-base font-semibold text-slate-800 dark:text-slate-100">{domain}</h2>
            <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">
              {domainReports.map((report) => (
                <Card key={report.reportId} className="flex flex-col p-4">
                  <h3 className="text-sm font-semibold text-slate-900 dark:text-slate-100">{report.name}</h3>
                  <p className="mt-1 flex-1 text-sm text-slate-600 dark:text-slate-300">{report.description}</p>
                  <div className="mt-3 flex flex-wrap gap-1">
                    {report.supportedFormats.map((format) => {
                      const Icon = FORMAT_ICON[format];
                      return (
                        <span
                          key={format}
                          className="inline-flex items-center gap-1 rounded-full bg-slate-100 px-2 py-0.5 text-xs font-medium text-slate-700 dark:bg-slate-700 dark:text-slate-200"
                        >
                          <Icon className="h-3.5 w-3.5" />
                          {format.toUpperCase()}
                        </span>
                      );
                    })}
                  </div>
                  <div className="mt-4">
                    <Button size="sm" onClick={() => navigate(`/reports/${report.reportId}`)}>
                      Generate
                    </Button>
                  </div>
                </Card>
              ))}
            </div>
          </section>
        ))
      )}
    </div>
  );
}
```

### Report Generate Page

```tsx
// src/features/reports/pages/ReportGeneratePage.tsx
import { useState, useMemo, useCallback } from 'react';
import { useParams, useNavigate } from 'react-router-dom';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { ArrowLeftIcon, EyeIcon, ArrowDownTrayIcon } from '@heroicons/react/24/outline';
import { Button } from '../../../shared/components/Button';
import { Card } from '../../../shared/components/Card';
import { PageHeader } from '../../../shared/components/PageHeader';
import { LoadingOverlay } from '../../../shared/components/LoadingOverlay';
import { TextFieldController } from '../../../shared/form/TextFieldController';
import { SelectController } from '../../../shared/form/SelectController';
import { useReportDetails } from '../hooks/useReports';
import { useReportGeneration } from '../hooks/useReportGeneration';
import { buildReportParameterSchema } from '../reports.schema';
import { ReportPreviewDialog } from '../components/ReportPreviewDialog';
import { cn } from '../../../lib/utils/cn';
import type { ReportFormat, ReportParameter } from '../reports.types';

export function ReportGeneratePage() {
  const { reportId } = useParams<{ reportId: string }>();
  const navigate = useNavigate();
  const { data: report, isLoading } = useReportDetails(reportId ?? '');
  const { generatePdf, isGenerating } = useReportGeneration();

  const [selectedFormat, setSelectedFormat] = useState<ReportFormat>('pdf');
  const [previewOpen, setPreviewOpen] = useState(false);
  const [previewHtml, setPreviewHtml] = useState('');

  const parameterSchema = useMemo(
    () => (report ? buildReportParameterSchema(report.parameters) : null),
    [report],
  );

  const { control, handleSubmit } = useForm({
    resolver: parameterSchema ? zodResolver(parameterSchema) : undefined,
    defaultValues: useMemo(() => {
      if (!report) return {};
      const defaults: Record<string, string> = {};
      for (const param of report.parameters) defaults[param.name] = param.defaultValue ?? '';
      return defaults;
    }, [report]),
  });

  /**
   * Build the report HTML for PDF generation or preview.
   * Each report implementation provides its own buildReportHtml that:
   * 1. Fetches data from the API with the parameters
   * 2. Renders a layout component (TableReportLayout, etc.) with the data
   * 3. Returns the HTML string via renderReportHtml()
   */
  const buildReportHtml = useCallback(async (_parameters: Record<string, unknown>): Promise<string> => {
    throw new Error('buildReportHtml must be implemented for each report. See Module Integration.');
  }, []);

  const handleGenerate = handleSubmit(async (values) => {
    if (!report) return;
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-').slice(0, 19);
    const baseFilename = `${report.reportId}-${timestamp}`;
    if (selectedFormat === 'pdf') {
      const html = await buildReportHtml(values);
      await generatePdf(html, `${baseFilename}.pdf`);
    }
    // xlsx/csv handled by the module-specific report hook (see Module Integration)
  });

  const handlePreview = handleSubmit(async (values) => {
    const html = await buildReportHtml(values);
    setPreviewHtml(html);
    setPreviewOpen(true);
  });

  if (isLoading || !report) return <LoadingOverlay />;

  return (
    <div>
      <PageHeader
        title={report.name}
        breadcrumbs={[
          { label: 'Dashboard', path: '/dashboard' },
          { label: 'Reports', path: '/reports' },
          { label: report.name },
        ]}
        actions={
          <Button variant="outline" onClick={() => navigate('/reports')}>
            <ArrowLeftIcon className="h-4 w-4" />
            Back to Reports
          </Button>
        }
      />

      <Card className="max-w-2xl p-6">
        <p className="mb-6 text-sm text-slate-600 dark:text-slate-300">{report.description}</p>

        {/* Parameter Form */}
        {report.parameters.length > 0 && (
          <>
            <h3 className="mb-3 text-sm font-semibold text-slate-800 dark:text-slate-100">Report Parameters</h3>
            <div className="mb-6 space-y-4">
              {report.parameters.map((param: ReportParameter) =>
                param.type === 'select' && param.options ? (
                  <SelectController
                    key={param.name}
                    name={param.name}
                    control={control}
                    label={param.label}
                    required={param.required}
                    options={param.options.map((o) => ({ value: o.value, label: o.label }))}
                  />
                ) : (
                  <TextFieldController
                    key={param.name}
                    name={param.name}
                    control={control}
                    label={param.label}
                    required={param.required}
                    type={param.type === 'number' ? 'number' : 'text'}
                  />
                ),
              )}
            </div>
            <hr className="mb-6 border-slate-200 dark:border-slate-700" />
          </>
        )}

        {/* Format Selection */}
        <h3 className="mb-3 text-sm font-semibold text-slate-800 dark:text-slate-100">Export Format</h3>
        <div className="mb-6 inline-flex rounded border border-slate-300 p-0.5 dark:border-slate-600">
          {report.supportedFormats.map((format) => (
            <button
              key={format}
              type="button"
              onClick={() => setSelectedFormat(format)}
              className={cn(
                'rounded px-4 py-1.5 text-sm font-medium transition-colors',
                selectedFormat === format
                  ? 'bg-primary text-white'
                  : 'text-slate-600 hover:bg-slate-100 dark:text-slate-300 dark:hover:bg-slate-700',
              )}
            >
              {format.toUpperCase()}
            </button>
          ))}
        </div>

        {/* Action Buttons */}
        <div className="flex gap-3">
          {selectedFormat === 'pdf' && (
            <Button variant="outline" onClick={handlePreview} disabled={isGenerating}>
              <EyeIcon className="h-4 w-4" />
              Preview
            </Button>
          )}
          <Button onClick={handleGenerate} loading={isGenerating}>
            <ArrowDownTrayIcon className="h-4 w-4" />
            Generate {selectedFormat.toUpperCase()}
          </Button>
        </div>
      </Card>

      <ReportPreviewDialog
        open={previewOpen}
        onClose={() => setPreviewOpen(false)}
        html={previewHtml}
        title={report.name}
      />
    </div>
  );
}
```

---

## Report Preview Dialog (Headless UI Dialog)

```tsx
// src/features/reports/components/ReportPreviewDialog.tsx
import { Dialog, DialogBackdrop, DialogPanel, DialogTitle } from '@headlessui/react';
import { XMarkIcon } from '@heroicons/react/24/outline';
import { Button } from '../../../shared/components/Button';

interface ReportPreviewDialogProps {
  open: boolean;
  onClose: () => void;
  html: string;
  title: string;
}

/**
 * Full-screen dialog that renders a report HTML preview inside an iframe.
 * The iframe uses a data URI to display the report without a network request.
 */
export function ReportPreviewDialog({ open, onClose, html, title }: ReportPreviewDialogProps) {
  const dataUri = `data:text/html;charset=utf-8,${encodeURIComponent(html)}`;

  return (
    <Dialog open={open} onClose={onClose} className="relative z-[1000]">
      <DialogBackdrop className="fixed inset-0 bg-black/50" />
      <div className="fixed inset-0 flex flex-col">
        <DialogPanel className="flex h-full w-full flex-col bg-surface">
          <div className="flex items-center justify-between border-b border-slate-200 px-4 py-3 dark:border-slate-700">
            <DialogTitle className="text-sm font-semibold text-slate-900 dark:text-slate-100">
              Preview: {title}
            </DialogTitle>
            <button
              type="button"
              onClick={onClose}
              aria-label="Close"
              className="rounded p-1.5 text-slate-500 hover:bg-slate-100 dark:hover:bg-slate-700"
            >
              <XMarkIcon className="h-5 w-5" />
            </button>
          </div>
          <iframe src={dataUri} title={`Preview: ${title}`} className="flex-1 border-0 bg-white" />
          <div className="flex justify-end border-t border-slate-200 px-4 py-3 dark:border-slate-700">
            <Button variant="outline" onClick={onClose}>
              Close
            </Button>
          </div>
        </DialogPanel>
      </div>
    </Dialog>
  );
}
```

---

## Zustand Report Store (Optional)

```typescript
// src/features/reports/store/report.store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import type { ReportFormat } from '../reports.types';

interface RecentReport {
  reportId: string;
  name: string;
  format: ReportFormat;
  generatedAt: string;
}

interface ReportState {
  lastFormat: ReportFormat;
  recentReports: RecentReport[];
  setLastFormat: (format: ReportFormat) => void;
  addRecentReport: (report: RecentReport) => void;
  clearRecent: () => void;
}

const MAX_RECENT = 10;

export const useReportStore = create<ReportState>()(
  persist(
    (set) => ({
      lastFormat: 'pdf',
      recentReports: [],
      setLastFormat: (format) => set({ lastFormat: format }),
      addRecentReport: (report) =>
        set((state) => ({ recentReports: [report, ...state.recentReports].slice(0, MAX_RECENT) })),
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
  import('../features/reports/pages/ReportListPage').then((m) => ({ default: m.ReportListPage })),
);
const ReportGeneratePage = lazy(() =>
  import('../features/reports/pages/ReportGeneratePage').then((m) => ({ default: m.ReportGeneratePage })),
);

// Inside the protected dashboard routes children array:
// { path: '/reports', element: <SuspenseWrapper><ReportListPage /></SuspenseWrapper> },
// { path: '/reports/:reportId', element: <SuspenseWrapper><ReportGeneratePage /></SuspenseWrapper> },
```

Navigation item for the sidebar:

```typescript
// Add to src/shared/components/navigationConfig.ts
import { ChartBarIcon } from '@heroicons/react/24/outline';
// { label: 'Reports', path: '/reports', icon: ChartBarIcon },
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

  getReportData: async (params: { department?: string; status?: string }): Promise<ReportDataRow[]> => {
    const { data } = await axiosInstance.get<ReportDataRow[]>('/v1/staff/report', { params });
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

export function StaffListReportLayout({ data, parameters, generatedAt }: StaffListReportLayoutProps) {
  const subtitle =
    [parameters.department && `Department: ${parameters.department}`, parameters.status && `Status: ${parameters.status}`]
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
  generate: (parameters: { department?: string; status?: string }, format: ReportFormat) => Promise<void>;
  preview: (parameters: { department?: string; status?: string }) => Promise<string>;
  isGenerating: boolean;
}

export function useStaffListReport(): UseStaffListReportReturn {
  const { generatePdf, generateXlsx, generateCsv, isGenerating } = useReportGeneration();

  const fetchData = useCallback(
    (parameters: { department?: string; status?: string }) =>
      staffApi.getReportData({ department: parameters.department || undefined, status: parameters.status || undefined }),
    [],
  );

  const buildHtml = useCallback(
    async (parameters: { department?: string; status?: string }): Promise<string> => {
      const data = await fetchData(parameters);
      const element = <StaffListReportLayout data={data} parameters={parameters} generatedAt={new Date().toISOString()} />;
      return renderReportHtml(element, 'Staff List Report');
    },
    [fetchData],
  );

  const generate = useCallback(
    async (parameters: { department?: string; status?: string }, format: ReportFormat): Promise<void> => {
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
    (parameters: { department?: string; status?: string }) => buildHtml(parameters),
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
│   │   └── ReportPreviewDialog.tsx       # Full-screen preview dialog (Headless UI)
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
│   │   └── report.store.ts               # (Optional) Persisted report preferences
│   ├── utils/
│   │   ├── exportXlsx.ts                 # Client-side XLSX export
│   │   └── exportCsv.ts                  # Client-side CSV export
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
| **Server-side Puppeteer for PDF** | Full CSS support (Tailwind, flexbox, grid), pixel-perfect rendering, page breaks, headers/footers. Browser-based PDF libraries (jsPDF, html2canvas) produce inferior results. |
| **Client-side CSV/XLSX** | No server round-trip needed — data is already available from the API or TanStack Query cache. `xlsx` and `PapaParse` are mature libraries. Instant export with no network latency. |
| **ReactDOMServer for HTML generation** | Consistent React component model for report layouts. Full TypeScript type safety. Report layouts are just React components — no separate template language. |
| **Tailwind CSS CDN in report HTML** | Puppeteer renders a standalone HTML document with no access to the app's compiled CSS bundle. Tailwind CDN provides the same utility classes used elsewhere, with no build step. |
| **Same Tailwind vocabulary everywhere** | The app UI and the report HTML both use Tailwind utility classes, so authoring a report layout is intuitive for anyone who built the app — no second styling system to learn. |
| **Separate report service** | Puppeteer requires Node.js + headless Chrome which cannot run in the browser. A lightweight Express server isolates the concern and can be scaled independently. |
| **Dynamic Zod schemas** | Report parameters are defined as data (ReportDefinition), so the Zod schema is built dynamically at runtime. New reports can be added without changing the form infrastructure. |

---

## Report Service Deployment

### Development

- Start the Vite dev server: `npm run dev` (port {{DEV_PORT}})
- Start the report service: `cd report-service && npm run dev` (port 3001)
- Vite proxy forwards `/api/reports` to `http://localhost:3001`

### Production

- Deploy the report service as a sidecar container or separate microservice
- The React SPA connects via the `VITE_REPORT_SERVICE_URL` environment variable (or an API gateway / ingress rule)

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

- **Browser singleton**: Reuse a single Chrome instance across requests (implemented above). Each request opens a new page (tab) and closes it when done.
- **Memory limit**: Set `--max-old-space-size` for the Node.js process and container memory limits in Kubernetes.
- **Page cleanup**: Always close pages in a `finally` block to prevent memory leaks from failed requests.

### Concurrent Report Generation

For production workloads with many concurrent report requests, limit concurrent Puppeteer
pages with a semaphore (e.g., max 5), or use a job queue (BullMQ + Redis) for high volume:

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

The report service implements `SIGTERM` and `SIGINT` handlers that stop accepting new
connections, close the Puppeteer browser, and exit cleanly — ensuring Kubernetes can
gracefully terminate the pod during deployments.

### Health Check

The `/api/reports/health` endpoint returns `{ "status": "ok" }` and can be used as a
Kubernetes liveness/readiness probe.
