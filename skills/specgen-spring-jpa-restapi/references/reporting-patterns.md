# Reporting Patterns — JasperReports with JRDesign API (Programmatic Layout)

This reference describes the reporting infrastructure for the spec. Include this content
in the Reporting section of the generated specification. All code samples must be reproduced
in the generated spec with full import statements and constructor injection.

The reporting feature provides:
- **JRDesign API** for building report layouts 100% programmatically — no `.jrxml` XML files
- **DTO-based data source** via `JRBeanCollectionDataSource` — no direct DB queries in reports
- **Multi-format export** — PDF, XLSX, CSV
- **Report registry** persisted in the database for discoverability
- **REST API endpoints** for listing reports, retrieving report metadata, and generating downloads
- **OpenAPI annotations** on all endpoints for Swagger documentation

> **Why JRDesign API instead of `.jrxml` templates?**
> Programmatic report design via `JRDesign*` classes is AI-agent-friendly: no XML parsing,
> compile-time type safety, dynamic layout construction, and no separate template files to
> manage. The entire report definition lives in Java code, making it easy to review, test,
> and refactor.

---

## Architecture Overview

```
Module                    Shared Reporting Infrastructure
┌─────────────────────┐   ┌─────────────────────────────────────────┐
│  StaffReport        │   │  ReportService                          │
│  implements         │──►│    compile(JasperDesign → JasperReport)  │
│  ReportDefinition   │   │    fill(JasperReport + DTO Collection)  │
│                     │   │    export(JasperPrint → PDF/XLSX/CSV)   │
│  buildDesign()      │   │                                         │
│  getParameters()    │   │  ReportRegistry                         │
│  generateData(...)  │   │    discovers @Component ReportDefinition│
│                     │   │    persists to report table             │
│  DTO: StaffReportDTO│   │                                         │
│  (data source bean) │   │  ReportController (@RestController)     │
│                     │   │    GET /api/v1/reports — list reports   │
└─────────────────────┘   │    GET /api/v1/reports/{id} — details   │
                          │    POST /api/v1/reports/{id}/generate   │
                          └─────────────────────────────────────────┘
```

**Data flow:**
1. Client calls `GET /api/v1/reports` to list available reports (JSON response)
2. Client calls `GET /api/v1/reports/{reportId}` to retrieve report metadata and parameter descriptors (JSON response)
3. Client calls `POST /api/v1/reports/{reportId}/generate` with parameters in the request body and desired format as a query parameter
4. `ReportController` dispatches to the matching `ReportDefinition`
5. `ReportDefinition.buildDesign(params)` constructs a `JasperDesign` programmatically using JRDesign API
6. `ReportDefinition.generateData(params)` calls module services to produce a `List<DTO>`
7. `ReportService` compiles the `JasperDesign` into a `JasperReport`, fills it with `JRBeanCollectionDataSource`, and exports
8. The exported file (PDF/XLSX/CSV) is returned as a binary download with `Content-Disposition: attachment`

---

## Maven Dependencies

```xml
<!-- Reporting (included when Reporting = yes) -->
<dependency>
    <groupId>net.sf.jasperreports</groupId>
    <artifactId>jasperreports</artifactId>
    <version>7.0.3</version>
</dependency>
<dependency>
    <groupId>net.sf.jasperreports</groupId>
    <artifactId>jasperreports-fonts</artifactId>
    <version>7.0.3</version>
</dependency>
<!-- PDF export via OpenPDF (JasperReports 7.x default) -->
<dependency>
    <groupId>com.github.librepdf</groupId>
    <artifactId>openpdf</artifactId>
    <version>2.0.4</version>
</dependency>
<!-- XLSX export -->
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>5.4.1</version>
</dependency>
```

> **Note:** JasperReports 7.x uses OpenPDF instead of iText for PDF generation. The
> `jasperreports` artifact pulls in transitive dependencies, but `openpdf` and `poi-ooxml`
> should be declared explicitly for version control.

---

## Application Configuration

```yaml
# [If Reporting = yes] add to application.yml:
app:
  reporting:
    temp-path: ${java.io.tmpdir}/reports   # temp directory for export files
    default-format: PDF                    # PDF, XLSX, CSV
```

### application-prod.yml overrides

```yaml
app:
  reporting:
    temp-path: ${REPORT_TEMP_PATH:/tmp/reports}
    default-format: PDF
```

> **Note:** No `template-path` or `compiled-path` configuration is needed. Report layouts
> are built programmatically via `JRDesign*` API and compiled in memory. Only a temp path
> for intermediate export files is required.

---

## Package Structure

```
shared/
└── reporting/
    ├── ReportDefinition.java          # Interface with buildDesign() for programmatic layout
    ├── ReportParameter.java           # Report parameter descriptor record
    ├── ReportFormat.java              # Enum: PDF, XLSX, CSV
    ├── ReportResult.java              # Holds generated bytes + metadata
    ├── ReportDesignHelper.java        # Static builder for common JRDesign patterns
    ├── ReportService.java             # Compile, fill, export orchestrator
    ├── ReportRegistry.java            # Auto-discovers and persists report definitions
    ├── ReportConfig.java              # Configuration properties binding
    ├── ReportEntity.java              # [If PostgreSQL/MySQL] Database entity for report registry
    ├── ReportRepository.java          # [If PostgreSQL/MySQL] Spring Data JPA repository
    ├── ReportDocument.java            # [If MongoDB] MongoDB document for report registry
    ├── ReportGenerationException.java # Runtime exception for report failures
    ├── ReportInfoDTO.java             # REST response DTO for report metadata
    └── ReportController.java          # @RestController for report API endpoints
```

---

## Report Definition Interface

Modules implement this interface to define their reports. Each implementation is
a Spring `@Component` automatically discovered by the `ReportRegistry`.

The key difference from template-based reporting is the `buildDesign()` method, which
returns a `JasperDesign` built programmatically using JRDesign API classes.

```java
package {{BASE_PACKAGE}}.shared.reporting;

import net.sf.jasperreports.engine.design.JasperDesign;

import java.util.Collection;
import java.util.List;
import java.util.Map;

/**
 * Contract for domain-defined reports. Each domain module creates one or more
 * implementations as Spring @Component beans. The ReportRegistry auto-discovers
 * them at startup and persists their metadata to the database.
 *
 * Report layouts are built programmatically via JRDesign API — no .jrxml files.
 */
public interface ReportDefinition {

    /**
     * Unique identifier for this report (e.g., "staff-allocation-summary").
     * Used as the report ID in URLs and database records.
     */
    String getReportId();

    /**
     * Human-readable report name displayed in the report list.
     */
    String getName();

    /**
     * Brief description of what this report contains.
     */
    String getDescription();

    /**
     * The domain this report belongs to (e.g., "Staff", "Project").
     * Used for grouping reports in the API responses.
     */
    String getDomain();

    /**
     * Defines the parameters this report accepts (date range, filters, etc.).
     * The API returns these descriptors so clients can build parameter forms.
     * Return an empty list if the report takes no parameters.
     */
    List<ReportParameter> getParameters();

    /**
     * Builds the report layout programmatically using JRDesign API.
     * Called by ReportService before compilation. The returned JasperDesign
     * is compiled into a JasperReport and cached by reportId.
     *
     * @param parameters user-supplied parameter values (may influence layout)
     * @return a fully constructed JasperDesign ready for compilation
     */
    JasperDesign buildDesign(Map<String, Object> parameters);

    /**
     * Executes domain-specific logic to produce the report data as a collection
     * of DTOs. The returned collection is wrapped in JRBeanCollectionDataSource
     * by the ReportService.
     *
     * @param parameters user-supplied parameter values keyed by parameter name
     * @return collection of DTO beans to be used as the Jasper data source
     */
    Collection<?> generateData(Map<String, Object> parameters);
}
```

---

## Report Parameter Descriptor

```java
package {{BASE_PACKAGE}}.shared.reporting;

import java.util.List;

/**
 * Describes a single parameter that a report accepts.
 * Used by the REST API to expose parameter metadata so clients can build forms.
 *
 * @param name        parameter key (e.g., "startDate", "departmentId")
 * @param label       human-readable label for the form field
 * @param type        input type: TEXT, DATE, NUMBER, SELECT, BOOLEAN
 * @param required    whether the parameter is mandatory
 * @param options     for SELECT type: list of allowed values (label-value pairs)
 * @param defaultValue default value if user doesn't provide one
 */
public record ReportParameter(
        String name,
        String label,
        ParameterType type,
        boolean required,
        List<Option> options,
        String defaultValue
) {
    public enum ParameterType {
        TEXT, DATE, NUMBER, SELECT, BOOLEAN
    }

    public record Option(String label, String value) {}

    public static ReportParameter text(String name, String label, boolean required) {
        return new ReportParameter(name, label, ParameterType.TEXT, required, List.of(), null);
    }

    public static ReportParameter date(String name, String label, boolean required) {
        return new ReportParameter(name, label, ParameterType.DATE, required, List.of(), null);
    }

    public static ReportParameter number(String name, String label, boolean required) {
        return new ReportParameter(name, label, ParameterType.NUMBER, required, List.of(), null);
    }

    public static ReportParameter select(String name, String label, boolean required, List<Option> options) {
        return new ReportParameter(name, label, ParameterType.SELECT, required, options, null);
    }

    public static ReportParameter bool(String name, String label, String defaultValue) {
        return new ReportParameter(name, label, ParameterType.BOOLEAN, false, List.of(), defaultValue);
    }
}
```

---

## Report Format Enum

```java
package {{BASE_PACKAGE}}.shared.reporting;

public enum ReportFormat {
    PDF("application/pdf", ".pdf"),
    XLSX("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet", ".xlsx"),
    CSV("text/csv", ".csv");

    private final String contentType;
    private final String extension;

    ReportFormat(String contentType, String extension) {
        this.contentType = contentType;
        this.extension = extension;
    }

    public String getContentType() { return contentType; }
    public String getExtension() { return extension; }
}
```

---

## Report Result Record

```java
package {{BASE_PACKAGE}}.shared.reporting;

/**
 * Holds the generated report output.
 *
 * @param data        the exported report bytes
 * @param format      the export format used
 * @param fileName    suggested file name for download
 * @param contentType MIME type for the HTTP response
 */
public record ReportResult(
        byte[] data,
        ReportFormat format,
        String fileName,
        String contentType
) {
    public static ReportResult of(byte[] data, ReportFormat format, String reportId) {
        String fileName = reportId + format.getExtension();
        return new ReportResult(data, format, fileName, format.getContentType());
    }
}
```

---

## Report Design Helper

Static utility class providing common JRDesign building blocks. Modules use these helpers
inside their `buildDesign()` implementations to avoid repetitive low-level JRDesign API calls.

```java
package {{BASE_PACKAGE}}.shared.reporting;

import net.sf.jasperreports.engine.design.JRDesignBand;
import net.sf.jasperreports.engine.design.JRDesignExpression;
import net.sf.jasperreports.engine.design.JRDesignField;
import net.sf.jasperreports.engine.design.JRDesignStaticText;
import net.sf.jasperreports.engine.design.JRDesignStyle;
import net.sf.jasperreports.engine.design.JRDesignTextField;
import net.sf.jasperreports.engine.design.JasperDesign;
import net.sf.jasperreports.engine.type.HorizontalTextAlignEnum;
import net.sf.jasperreports.engine.type.VerticalTextAlignEnum;

import java.awt.Color;
import java.util.List;

/**
 * Static helpers for common JRDesign report building patterns.
 * All methods return JRDesign objects that can be added to a JasperDesign.
 */
public final class ReportDesignHelper {

    private ReportDesignHelper() {
        // utility class
    }

    // ── Standard page dimensions ──

    /** A4 Portrait: 595 x 842 points (72 dpi), margins 20 each side. */
    public static final int A4_PORTRAIT_WIDTH = 595;
    public static final int A4_PORTRAIT_HEIGHT = 842;

    /** A4 Landscape: 842 x 595 points (72 dpi), margins 20 each side. */
    public static final int A4_LANDSCAPE_WIDTH = 842;
    public static final int A4_LANDSCAPE_HEIGHT = 595;

    private static final int DEFAULT_MARGIN = 20;

    /**
     * Column definition for table-style reports.
     *
     * @param label     the column header label
     * @param fieldName the DTO field name (must match JRDesignField name)
     * @param width     the column width in points
     */
    public record ColumnDef(String label, String fieldName, int width) {}

    // ── Design factory methods ──

    /**
     * Create a new A4 Portrait JasperDesign with standard margins.
     *
     * @param name the report name (used as the design name)
     * @return a JasperDesign configured for A4 portrait orientation
     */
    public static JasperDesign createA4PortraitDesign(String name) {
        JasperDesign design = new JasperDesign();
        design.setName(name);
        design.setPageWidth(A4_PORTRAIT_WIDTH);
        design.setPageHeight(A4_PORTRAIT_HEIGHT);
        design.setLeftMargin(DEFAULT_MARGIN);
        design.setRightMargin(DEFAULT_MARGIN);
        design.setTopMargin(DEFAULT_MARGIN);
        design.setBottomMargin(DEFAULT_MARGIN);
        design.setColumnWidth(A4_PORTRAIT_WIDTH - DEFAULT_MARGIN * 2);
        return design;
    }

    /**
     * Create a new A4 Landscape JasperDesign with standard margins.
     *
     * @param name the report name (used as the design name)
     * @return a JasperDesign configured for A4 landscape orientation
     */
    public static JasperDesign createA4LandscapeDesign(String name) {
        JasperDesign design = new JasperDesign();
        design.setName(name);
        design.setPageWidth(A4_LANDSCAPE_WIDTH);
        design.setPageHeight(A4_LANDSCAPE_HEIGHT);
        design.setLeftMargin(DEFAULT_MARGIN);
        design.setRightMargin(DEFAULT_MARGIN);
        design.setTopMargin(DEFAULT_MARGIN);
        design.setBottomMargin(DEFAULT_MARGIN);
        design.setColumnWidth(A4_LANDSCAPE_WIDTH - DEFAULT_MARGIN * 2);
        return design;
    }

    // ── Field helpers ──

    /**
     * Add a field declaration to the JasperDesign. Fields map to DTO properties
     * and are referenced in detail band text fields via $F{fieldName} expressions.
     *
     * @param design the JasperDesign to add the field to
     * @param name   the field name (must match the DTO property name)
     * @param type   the field Java type (e.g., String.class, Integer.class)
     */
    public static void addField(JasperDesign design, String name, Class<?> type) {
        try {
            JRDesignField field = new JRDesignField();
            field.setName(name);
            field.setValueClass(type);
            design.addField(field);
        } catch (Exception e) {
            throw new ReportGenerationException("Failed to add field '" + name + "' to design", e);
        }
    }

    // ── Band builders ──

    /**
     * Create a title band with a centered, bold heading.
     *
     * @param title  the report title text
     * @param height the band height in points
     * @return a JRDesignBand configured as a title band
     */
    public static JRDesignBand createTitleBand(String title, int height) {
        JRDesignBand band = new JRDesignBand();
        band.setHeight(height);

        JRDesignStaticText titleText = new JRDesignStaticText();
        titleText.setX(0);
        titleText.setY(0);
        titleText.setWidth(802); // full column width for landscape; adjusted by caller if needed
        titleText.setHeight(height);
        titleText.setText(title);
        titleText.setFontSize(18f);
        titleText.setBold(true);
        titleText.setHorizontalTextAlign(HorizontalTextAlignEnum.CENTER);
        titleText.setVerticalTextAlign(VerticalTextAlignEnum.MIDDLE);

        band.addElement(titleText);
        return band;
    }

    /**
     * Create a column header band with styled header labels based on column definitions.
     * Each column header is a static text element with bold font and a light gray background.
     *
     * @param columns the list of column definitions
     * @return a JRDesignBand configured as a column header band
     */
    public static JRDesignBand createColumnHeaderBand(List<ColumnDef> columns) {
        JRDesignBand band = new JRDesignBand();
        band.setHeight(30);

        int xOffset = 0;
        for (ColumnDef col : columns) {
            JRDesignStaticText header = new JRDesignStaticText();
            header.setX(xOffset);
            header.setY(0);
            header.setWidth(col.width());
            header.setHeight(30);
            header.setText(col.label());
            header.setFontSize(10f);
            header.setBold(true);
            header.setVerticalTextAlign(VerticalTextAlignEnum.MIDDLE);
            header.setMode(net.sf.jasperreports.engine.type.ModeEnum.OPAQUE);
            header.setBackcolor(new Color(232, 232, 232)); // light gray #E8E8E8

            band.addElement(header);
            xOffset += col.width();
        }

        return band;
    }

    /**
     * Create a detail band with text fields mapped to DTO properties via $F{fieldName} expressions.
     * Each text field is positioned according to the corresponding column definition.
     *
     * @param columns the list of column definitions (fieldName must match declared JRDesignField names)
     * @return a JRDesignBand configured as a detail band
     */
    public static JRDesignBand createDetailBand(List<ColumnDef> columns) {
        JRDesignBand band = new JRDesignBand();
        band.setHeight(25);

        int xOffset = 0;
        for (ColumnDef col : columns) {
            JRDesignTextField textField = new JRDesignTextField();
            textField.setX(xOffset);
            textField.setY(0);
            textField.setWidth(col.width());
            textField.setHeight(25);
            textField.setFontSize(9f);
            textField.setVerticalTextAlign(VerticalTextAlignEnum.MIDDLE);
            textField.setBlankWhenNull(true);

            JRDesignExpression expression = new JRDesignExpression();
            expression.setText("$F{" + col.fieldName() + "}");
            textField.setExpression(expression);

            band.addElement(textField);
            xOffset += col.width();
        }

        return band;
    }

    /**
     * Create a page footer band with page number display ("Page X of Y").
     * Uses built-in JasperReports variables $V{PAGE_NUMBER}.
     *
     * @return a JRDesignBand configured as a page footer band
     */
    public static JRDesignBand createPageFooterBand() {
        JRDesignBand band = new JRDesignBand();
        band.setHeight(30);

        // "Page X" — evaluates at page time
        JRDesignTextField pageNumberField = new JRDesignTextField();
        pageNumberField.setX(0);
        pageNumberField.setY(5);
        pageNumberField.setWidth(100);
        pageNumberField.setHeight(20);
        pageNumberField.setFontSize(8f);
        pageNumberField.setHorizontalTextAlign(HorizontalTextAlignEnum.LEFT);

        JRDesignExpression pageExpr = new JRDesignExpression();
        pageExpr.setText("\"Page \" + $V{PAGE_NUMBER}");
        pageNumberField.setExpression(pageExpr);

        band.addElement(pageNumberField);

        // "of Y" — evaluates at report time (after all pages rendered)
        JRDesignTextField totalPagesField = new JRDesignTextField();
        totalPagesField.setX(100);
        totalPagesField.setY(5);
        totalPagesField.setWidth(60);
        totalPagesField.setHeight(20);
        totalPagesField.setFontSize(8f);
        totalPagesField.setHorizontalTextAlign(HorizontalTextAlignEnum.LEFT);
        totalPagesField.setEvaluationTime(net.sf.jasperreports.engine.type.EvaluationTimeEnum.REPORT);

        JRDesignExpression totalExpr = new JRDesignExpression();
        totalExpr.setText("\" of \" + $V{PAGE_NUMBER}");
        totalPagesField.setExpression(totalExpr);

        band.addElement(totalPagesField);

        return band;
    }

    // ── Style helpers ──

    /**
     * Create a JRDesignStyle with the specified font properties.
     *
     * @param name     the style name (used to reference the style in design elements)
     * @param fontName the font family name (e.g., "SansSerif", "DejaVu Sans")
     * @param fontSize the font size in points
     * @param bold     whether the font is bold
     * @return a JRDesignStyle configured with the specified properties
     */
    public static JRDesignStyle createStyle(String name, String fontName, int fontSize, boolean bold) {
        JRDesignStyle style = new JRDesignStyle();
        style.setName(name);
        style.setFontName(fontName);
        style.setFontSize((float) fontSize);
        style.setBold(bold);
        return style;
    }
}
```

---

## Report Configuration

```java
package {{BASE_PACKAGE}}.shared.reporting;

import lombok.Getter;
import lombok.Setter;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConfigurationProperties(prefix = "app.reporting")
@Getter
@Setter
public class ReportConfig {

    private String tempPath = System.getProperty("java.io.tmpdir") + "/reports";
    private ReportFormat defaultFormat = ReportFormat.PDF;
}
```

> **Note:** Unlike the template-based variant, there are no `templatePath` or `compiledPath`
> properties. Report layouts are built in memory via `JRDesign*` API and compiled on the fly.
> Only `tempPath` is retained for any temporary file operations during export.

---

## Report Service

The central orchestrator that compiles `JasperDesign` objects returned by `ReportDefinition.buildDesign()`,
fills them with DTO collections via `JRBeanCollectionDataSource`, and exports to the requested format.

Compiled `JasperReport` objects are cached by `reportId` to avoid re-compilation on every request.

```java
package {{BASE_PACKAGE}}.shared.reporting;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import net.sf.jasperreports.engine.JRException;
import net.sf.jasperreports.engine.JasperCompileManager;
import net.sf.jasperreports.engine.JasperFillManager;
import net.sf.jasperreports.engine.JasperPrint;
import net.sf.jasperreports.engine.JasperReport;
import net.sf.jasperreports.engine.data.JRBeanCollectionDataSource;
import net.sf.jasperreports.engine.design.JasperDesign;
import net.sf.jasperreports.engine.export.JRCsvExporter;
import net.sf.jasperreports.engine.export.ooxml.JRXlsxExporter;
import net.sf.jasperreports.export.SimpleCsvExporterConfiguration;
import net.sf.jasperreports.export.SimpleExporterInput;
import net.sf.jasperreports.export.SimpleOutputStreamExporterOutput;
import net.sf.jasperreports.export.SimpleXlsxReportConfiguration;
import net.sf.jasperreports.pdf.JRPdfExporter;
import org.springframework.stereotype.Service;

import java.io.ByteArrayOutputStream;
import java.util.Collection;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
@RequiredArgsConstructor
@Slf4j
public class ReportService {

    private final ReportConfig config;

    // Cache compiled JasperReport objects by reportId to avoid re-compilation
    private final Map<String, JasperReport> compiledCache = new ConcurrentHashMap<>();

    /**
     * Generate a report using the specified ReportDefinition, user parameters, and format.
     *
     * @param definition the report definition (provides design builder and data generation)
     * @param parameters user-supplied parameter values
     * @param format     desired export format (PDF, XLSX, CSV)
     * @return ReportResult containing the exported bytes and metadata
     */
    public ReportResult generate(ReportDefinition definition,
                                 Map<String, Object> parameters,
                                 ReportFormat format) {
        log.info("Generating report [{}] in format [{}]", definition.getReportId(), format);

        // 1. Generate the data from the domain module
        Collection<?> data = definition.generateData(parameters);
        log.debug("Report [{}] produced {} data records", definition.getReportId(), data.size());

        // 2. Build and compile the JasperDesign (cached by reportId)
        JasperReport jasperReport = compiledCache.computeIfAbsent(
                definition.getReportId(),
                id -> compileDesign(definition, parameters)
        );

        // 3. Create the DTO-based data source
        JRBeanCollectionDataSource dataSource = new JRBeanCollectionDataSource(data);

        // 4. Build Jasper parameters (pass user params + any report-level params)
        Map<String, Object> jasperParams = new HashMap<>(parameters);

        // 5. Fill the report
        JasperPrint jasperPrint = fillReport(jasperReport, jasperParams, dataSource);

        // 6. Export to the requested format
        byte[] exportedBytes = exportReport(jasperPrint, format);

        log.info("Report [{}] generated successfully: {} bytes", definition.getReportId(), exportedBytes.length);
        return ReportResult.of(exportedBytes, format, definition.getReportId());
    }

    /**
     * Build and compile a JasperDesign from a ReportDefinition.
     * The definition's buildDesign() method constructs the layout programmatically.
     */
    private JasperReport compileDesign(ReportDefinition definition, Map<String, Object> parameters) {
        try {
            log.debug("Building and compiling design for report: {}", definition.getReportId());
            JasperDesign design = definition.buildDesign(parameters);
            return JasperCompileManager.compileReport(design);
        } catch (JRException e) {
            throw new ReportGenerationException(
                    "Failed to compile design for report: " + definition.getReportId(), e);
        }
    }

    /**
     * Fill a compiled JasperReport with parameters and data source.
     */
    private JasperPrint fillReport(JasperReport report,
                                   Map<String, Object> parameters,
                                   JRBeanCollectionDataSource dataSource) {
        try {
            return JasperFillManager.fillReport(report, parameters, dataSource);
        } catch (JRException e) {
            throw new ReportGenerationException("Failed to fill report", e);
        }
    }

    /**
     * Export JasperPrint to the specified format.
     */
    private byte[] exportReport(JasperPrint jasperPrint, ReportFormat format) {
        try (ByteArrayOutputStream outputStream = new ByteArrayOutputStream()) {
            switch (format) {
                case PDF -> exportToPdf(jasperPrint, outputStream);
                case XLSX -> exportToXlsx(jasperPrint, outputStream);
                case CSV -> exportToCsv(jasperPrint, outputStream);
            }
            return outputStream.toByteArray();
        } catch (Exception e) {
            throw new ReportGenerationException("Failed to export report as " + format, e);
        }
    }

    private void exportToPdf(JasperPrint print, ByteArrayOutputStream out) throws JRException {
        JRPdfExporter exporter = new JRPdfExporter();
        exporter.setExporterInput(new SimpleExporterInput(print));
        exporter.setExporterOutput(new SimpleOutputStreamExporterOutput(out));
        exporter.exportReport();
    }

    private void exportToXlsx(JasperPrint print, ByteArrayOutputStream out) throws JRException {
        JRXlsxExporter exporter = new JRXlsxExporter();
        exporter.setExporterInput(new SimpleExporterInput(print));
        exporter.setExporterOutput(new SimpleOutputStreamExporterOutput(out));

        SimpleXlsxReportConfiguration xlsxConfig = new SimpleXlsxReportConfiguration();
        xlsxConfig.setOnePagePerSheet(false);
        xlsxConfig.setRemoveEmptySpaceBetweenRows(true);
        xlsxConfig.setDetectCellType(true);
        xlsxConfig.setWhitePageBackground(false);
        exporter.setConfiguration(xlsxConfig);

        exporter.exportReport();
    }

    private void exportToCsv(JasperPrint print, ByteArrayOutputStream out) throws JRException {
        JRCsvExporter exporter = new JRCsvExporter();
        exporter.setExporterInput(new SimpleExporterInput(print));
        exporter.setExporterOutput(new SimpleOutputStreamExporterOutput(out));

        SimpleCsvExporterConfiguration csvConfig = new SimpleCsvExporterConfiguration();
        csvConfig.setFieldDelimiter(",");
        exporter.setConfiguration(csvConfig);

        exporter.exportReport();
    }

    /**
     * Evict a cached compiled report (useful when the report design changes at runtime).
     *
     * @param reportId the report ID whose cached compilation should be removed
     */
    public void evictReport(String reportId) {
        compiledCache.remove(reportId);
        log.info("Evicted cached compiled report: {}", reportId);
    }
}
```

---

## Report Generation Exception

```java
package {{BASE_PACKAGE}}.shared.reporting;

public class ReportGenerationException extends RuntimeException {

    public ReportGenerationException(String message, Throwable cause) {
        super(message, cause);
    }

    public ReportGenerationException(String message) {
        super(message);
    }
}
```

---

## Report Registry

The registry auto-discovers all `ReportDefinition` beans at startup and persists their
metadata to the database. This allows the report list API to return all available reports
without scanning components at runtime.

### [If PostgreSQL/MySQL] Report Entity

```java
package {{BASE_PACKAGE}}.shared.reporting;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import lombok.Getter;
import lombok.Setter;

@Entity
@Table(name = "report")
@Getter
@Setter
public class ReportEntity {

    @Id
    @Column(name = "report_id", length = 100)
    private String reportId;

    @Column(name = "name", nullable = false, length = 255)
    private String name;

    @Column(name = "description", length = 500)
    private String description;

    @Column(name = "domain", nullable = false, length = 100)
    private String domain;

    @Column(name = "active", nullable = false)
    private boolean active = true;
}
```

### [If PostgreSQL/MySQL] Flyway Migration

```sql
-- V{N}__create_report_table.sql
CREATE TABLE report (
    report_id   VARCHAR(100) PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    description VARCHAR(500),
    domain      VARCHAR(100) NOT NULL,
    active      BOOLEAN NOT NULL DEFAULT TRUE
);
```

### [If PostgreSQL/MySQL] Report Repository

```java
package {{BASE_PACKAGE}}.shared.reporting;

import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface ReportRepository extends JpaRepository<ReportEntity, String> {

    List<ReportEntity> findByActiveTrue();

    List<ReportEntity> findByDomainAndActiveTrue(String domain);
}
```

### [If MongoDB] Report Document

```java
package {{BASE_PACKAGE}}.shared.reporting;

import lombok.Getter;
import lombok.Setter;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection = "report")
@Getter
@Setter
public class ReportDocument {

    @Id
    private String reportId;
    private String name;
    private String description;
    private String domain;
    private boolean active = true;
}
```

### [If MongoDB] Report Repository

```java
package {{BASE_PACKAGE}}.shared.reporting;

import org.springframework.data.mongodb.repository.MongoRepository;

import java.util.List;

public interface ReportRepository extends MongoRepository<ReportDocument, String> {

    List<ReportDocument> findByActiveTrue();

    List<ReportDocument> findByDomainAndActiveTrue(String domain);
}
```

### Registry Service

> **Note for [If PostgreSQL/MySQL]:** The code below references `ReportEntity` and
> `ReportRepository` from the JPA variant. For [If MongoDB], replace `ReportEntity` with
> `ReportDocument` and use the MongoDB `ReportRepository`.

```java
package {{BASE_PACKAGE}}.shared.reporting;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

@Service
@RequiredArgsConstructor
@Slf4j
public class ReportRegistry {

    private final List<ReportDefinition> reportDefinitions;
    private final ReportRepository reportRepository;

    // In-memory lookup for fast dispatch during report generation
    private final Map<String, ReportDefinition> definitionMap = new ConcurrentHashMap<>();

    /**
     * On application startup, register all ReportDefinition beans in the database
     * and build the in-memory lookup map.
     */
    @EventListener(ApplicationReadyEvent.class)
    public void registerReports() {
        log.info("Registering {} report definitions", reportDefinitions.size());

        for (ReportDefinition definition : reportDefinitions) {
            definitionMap.put(definition.getReportId(), definition);

            // [If PostgreSQL/MySQL] Upsert to database
            ReportEntity entity = reportRepository.findById(definition.getReportId())
                    .orElseGet(ReportEntity::new);
            entity.setReportId(definition.getReportId());
            entity.setName(definition.getName());
            entity.setDescription(definition.getDescription());
            entity.setDomain(definition.getDomain());
            entity.setActive(true);
            reportRepository.save(entity);

            // [If MongoDB] Upsert to database
            // ReportDocument doc = reportRepository.findById(definition.getReportId())
            //         .orElseGet(ReportDocument::new);
            // doc.setReportId(definition.getReportId());
            // doc.setName(definition.getName());
            // doc.setDescription(definition.getDescription());
            // doc.setDomain(definition.getDomain());
            // doc.setActive(true);
            // reportRepository.save(doc);

            log.info("Registered report: [{}] {} ({})",
                    definition.getReportId(), definition.getName(), definition.getDomain());
        }

        // Deactivate reports that no longer have a definition
        List<?> allReports = reportRepository.findAll();
        for (Object reportObj : allReports) {
            // [If PostgreSQL/MySQL]
            ReportEntity entity = (ReportEntity) reportObj;
            if (!definitionMap.containsKey(entity.getReportId())) {
                entity.setActive(false);
                reportRepository.save(entity);
                log.info("Deactivated report: [{}] (no definition found)", entity.getReportId());
            }

            // [If MongoDB]
            // ReportDocument doc = (ReportDocument) reportObj;
            // if (!definitionMap.containsKey(doc.getReportId())) {
            //     doc.setActive(false);
            //     reportRepository.save(doc);
            //     log.info("Deactivated report: [{}] (no definition found)", doc.getReportId());
            // }
        }
    }

    /**
     * Get a ReportDefinition by its ID for report generation.
     */
    public Optional<ReportDefinition> getDefinition(String reportId) {
        return Optional.ofNullable(definitionMap.get(reportId));
    }

    /**
     * List all active reports from the database.
     */
    public List<?> listActiveReports() {
        return reportRepository.findByActiveTrue();
    }

    /**
     * List active reports for a specific domain.
     */
    public List<?> listReportsByDomain(String domain) {
        return reportRepository.findByDomainAndActiveTrue(domain);
    }
}
```

---

## Report DTO for REST Responses

This DTO is returned by the REST controller for report listing and detail endpoints.
It exposes report metadata including parameter descriptors so API clients can build
dynamic parameter forms.

```java
package {{BASE_PACKAGE}}.shared.reporting;

import java.util.List;

/**
 * REST response DTO for report metadata.
 * Returned by GET /api/v1/reports and GET /api/v1/reports/{reportId}.
 *
 * @param reportId    unique report identifier
 * @param name        human-readable report name
 * @param description brief description of the report contents
 * @param domain      the domain this report belongs to (for grouping)
 * @param parameters  list of parameter descriptors for building input forms
 */
public record ReportInfoDTO(
        String reportId,
        String name,
        String description,
        String domain,
        List<ReportParameter> parameters
) {

    /**
     * Create from a ReportDefinition bean.
     */
    public static ReportInfoDTO from(ReportDefinition definition) {
        return new ReportInfoDTO(
                definition.getReportId(),
                definition.getName(),
                definition.getDescription(),
                definition.getDomain(),
                definition.getParameters()
        );
    }

    /**
     * [If PostgreSQL/MySQL] Create from a ReportEntity (without parameters — used for list view).
     */
    public static ReportInfoDTO fromEntity(ReportEntity entity) {
        return new ReportInfoDTO(
                entity.getReportId(),
                entity.getName(),
                entity.getDescription(),
                entity.getDomain(),
                List.of()
        );
    }

    /**
     * [If MongoDB] Create from a ReportDocument (without parameters — used for list view).
     */
    public static ReportInfoDTO fromDocument(ReportDocument document) {
        return new ReportInfoDTO(
                document.getReportId(),
                document.getName(),
                document.getDescription(),
                document.getDomain(),
                List.of()
        );
    }
}
```

---

## Report Controller

REST controller exposing report management and generation endpoints. All responses are JSON
except the generate endpoint which returns binary file downloads.

```java
package {{BASE_PACKAGE}}.shared.reporting;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/v1/reports")
@RequiredArgsConstructor
@Slf4j
@Tag(name = "Reports", description = "Report generation and management")
class ReportController {

    private final ReportRegistry reportRegistry;
    private final ReportService reportService;

    /**
     * List all available reports. Returns report metadata without parameter details.
     * Use GET /api/v1/reports/{reportId} to retrieve full parameter descriptors.
     */
    @GetMapping
    @Operation(
            summary = "List all available reports",
            description = "Returns a list of all active reports with basic metadata. "
                    + "Parameter details are omitted — use the detail endpoint to get parameters."
    )
    @ApiResponse(
            responseCode = "200",
            description = "List of available reports",
            content = @Content(
                    mediaType = "application/json",
                    schema = @Schema(implementation = ReportInfoDTO.class)
            )
    )
    public ResponseEntity<List<ReportInfoDTO>> listReports() {
        // [If PostgreSQL/MySQL]
        List<ReportInfoDTO> reports = reportRegistry.listActiveReports().stream()
                .map(r -> ReportInfoDTO.fromEntity((ReportEntity) r))
                .toList();

        // [If MongoDB]
        // List<ReportInfoDTO> reports = reportRegistry.listActiveReports().stream()
        //         .map(r -> ReportInfoDTO.fromDocument((ReportDocument) r))
        //         .toList();

        return ResponseEntity.ok(reports);
    }

    /**
     * Get report details including parameter descriptors.
     * Clients use the returned parameter metadata to build dynamic input forms.
     */
    @GetMapping("/{reportId}")
    @Operation(
            summary = "Get report details by ID",
            description = "Returns full report metadata including parameter descriptors. "
                    + "Clients should use the parameter list to build a dynamic input form."
    )
    @ApiResponse(
            responseCode = "200",
            description = "Report details with parameters",
            content = @Content(
                    mediaType = "application/json",
                    schema = @Schema(implementation = ReportInfoDTO.class)
            )
    )
    @ApiResponse(
            responseCode = "404",
            description = "Report not found"
    )
    public ResponseEntity<ReportInfoDTO> getReport(
            @Parameter(description = "Unique report identifier")
            @PathVariable String reportId) {

        ReportDefinition definition = reportRegistry.getDefinition(reportId)
                .orElseThrow(() -> new ReportGenerationException("Report not found: " + reportId));

        ReportInfoDTO dto = ReportInfoDTO.from(definition);
        return ResponseEntity.ok(dto);
    }

    /**
     * Generate and download a report.
     * Accepts report parameters in the request body and returns the generated file
     * as a binary download with appropriate Content-Type and Content-Disposition headers.
     */
    @PostMapping("/{reportId}/generate")
    @Operation(
            summary = "Generate and download a report",
            description = "Generates the specified report with the provided parameters and returns "
                    + "the file as a binary download. Supported formats: PDF, XLSX, CSV."
    )
    @ApiResponse(
            responseCode = "200",
            description = "Report generated successfully — binary file download",
            content = {
                    @Content(mediaType = "application/pdf"),
                    @Content(mediaType = "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"),
                    @Content(mediaType = "text/csv")
            }
    )
    @ApiResponse(
            responseCode = "400",
            description = "Invalid parameters or report generation failed"
    )
    @ApiResponse(
            responseCode = "404",
            description = "Report not found"
    )
    public ResponseEntity<byte[]> generateReport(
            @Parameter(description = "Unique report identifier")
            @PathVariable String reportId,
            @Parameter(description = "Export format: PDF (default), XLSX, or CSV")
            @RequestParam(defaultValue = "PDF") ReportFormat format,
            @io.swagger.v3.oas.annotations.parameters.RequestBody(
                    description = "Report parameters as key-value pairs",
                    content = @Content(schema = @Schema(implementation = Map.class))
            )
            @RequestBody(required = false) Map<String, Object> parameters) {

        Map<String, Object> params = parameters != null ? parameters : Map.of();

        ReportDefinition definition = reportRegistry.getDefinition(reportId)
                .orElseThrow(() -> new ReportGenerationException("Report not found: " + reportId));

        log.info("Generating report [{}] with format [{}] and {} parameters",
                reportId, format, params.size());

        ReportResult result = reportService.generate(definition, params, format);

        return ResponseEntity.ok()
                .contentType(MediaType.parseMediaType(result.contentType()))
                .header(HttpHeaders.CONTENT_DISPOSITION,
                        "attachment; filename=\"" + result.fileName() + "\"")
                .header(HttpHeaders.CONTENT_LENGTH, String.valueOf(result.data().length))
                .body(result.data());
    }
}
```

---

## Module Integration — Sample Report Implementation

Each module creates `@Component` classes implementing `ReportDefinition` to register
their reports. The implementation calls module services (never repositories directly) to
produce DTO collections, and builds the report layout programmatically using `JRDesign*` API
with helpers from `ReportDesignHelper`.

### Sample: Staff Allocation Summary Report

```java
package {{BASE_PACKAGE}}.staff.internal;

import {{BASE_PACKAGE}}.shared.reporting.ReportDefinition;
import {{BASE_PACKAGE}}.shared.reporting.ReportDesignHelper;
import {{BASE_PACKAGE}}.shared.reporting.ReportDesignHelper.ColumnDef;
import {{BASE_PACKAGE}}.shared.reporting.ReportGenerationException;
import {{BASE_PACKAGE}}.shared.reporting.ReportParameter;
import {{BASE_PACKAGE}}.staff.StaffService;
import lombok.RequiredArgsConstructor;
import net.sf.jasperreports.engine.design.JRDesignBand;
import net.sf.jasperreports.engine.design.JasperDesign;
import net.sf.jasperreports.engine.type.SectionTypeEnum;
import org.springframework.stereotype.Component;

import java.util.Collection;
import java.util.List;
import java.util.Map;

/**
 * Staff Allocation Summary report — lists all staff with their current
 * allocation status, department, and position.
 *
 * Data source: StaffReportDTO (flat DTO projected from Staff entity/document).
 * Layout: A4 landscape, built programmatically via JRDesign API.
 */
@Component
@RequiredArgsConstructor
public class StaffAllocationSummaryReport implements ReportDefinition {

    private final StaffService staffService;

    @Override
    public String getReportId() {
        return "staff-allocation-summary";
    }

    @Override
    public String getName() {
        return "Staff Allocation Summary";
    }

    @Override
    public String getDescription() {
        return "Lists all staff members with their department, position, and current allocation status.";
    }

    @Override
    public String getDomain() {
        return "Staff";
    }

    @Override
    public List<ReportParameter> getParameters() {
        return List.of(
                ReportParameter.select("departmentId", "Department", false,
                        List.of(
                                new ReportParameter.Option("All Departments", ""),
                                new ReportParameter.Option("Software Engineering", "dept-se"),
                                new ReportParameter.Option("Quality Assurance", "dept-qa")
                        )),
                ReportParameter.select("allocationStatus", "Allocation Status", false,
                        List.of(
                                new ReportParameter.Option("All", ""),
                                new ReportParameter.Option("Available", "AVAILABLE"),
                                new ReportParameter.Option("Allocated", "ALLOCATED")
                        ))
        );
    }

    @Override
    public JasperDesign buildDesign(Map<String, Object> parameters) {
        try {
            // 1. Create A4 landscape design
            JasperDesign design = ReportDesignHelper.createA4LandscapeDesign("staff-allocation-summary");

            // 2. Declare fields matching StaffReportDTO properties
            ReportDesignHelper.addField(design, "staffNumber", String.class);
            ReportDesignHelper.addField(design, "fullName", String.class);
            ReportDesignHelper.addField(design, "email", String.class);
            ReportDesignHelper.addField(design, "departmentName", String.class);
            ReportDesignHelper.addField(design, "positionName", String.class);
            ReportDesignHelper.addField(design, "allocationStatus", String.class);
            ReportDesignHelper.addField(design, "allocationPercentage", String.class);

            // 3. Define column layout
            List<ColumnDef> columns = List.of(
                    new ColumnDef("Staff No.", "staffNumber", 80),
                    new ColumnDef("Full Name", "fullName", 150),
                    new ColumnDef("Email", "email", 170),
                    new ColumnDef("Department", "departmentName", 130),
                    new ColumnDef("Position", "positionName", 120),
                    new ColumnDef("Status", "allocationStatus", 80),
                    new ColumnDef("Allocation %", "allocationPercentage", 72)
            );

            // 4. Title band
            JRDesignBand titleBand = ReportDesignHelper.createTitleBand("Staff Allocation Summary", 50);
            design.setTitle(titleBand);

            // 5. Column header band
            JRDesignBand columnHeaderBand = ReportDesignHelper.createColumnHeaderBand(columns);
            design.setColumnHeader(columnHeaderBand);

            // 6. Detail band
            JRDesignBand detailBand = ReportDesignHelper.createDetailBand(columns);
            ((net.sf.jasperreports.engine.design.JRDesignSection) design.getDetailSection())
                    .addBand(detailBand);

            // 7. Page footer band with page numbers
            JRDesignBand pageFooterBand = ReportDesignHelper.createPageFooterBand();
            design.setPageFooter(pageFooterBand);

            return design;
        } catch (Exception e) {
            throw new ReportGenerationException(
                    "Failed to build design for report: " + getReportId(), e);
        }
    }

    @Override
    public Collection<?> generateData(Map<String, Object> parameters) {
        String departmentId = (String) parameters.getOrDefault("departmentId", "");
        String allocationStatus = (String) parameters.getOrDefault("allocationStatus", "");

        // Call domain service to get filtered data as DTOs
        // The service returns List<StaffReportDTO> — flat DTO suitable for Jasper
        return staffService.getStaffForReport(departmentId, allocationStatus);
    }
}
```

---

## Sample Report DTO (Jasper Data Source Bean)

```java
package {{BASE_PACKAGE}}.staff.internal;

import lombok.Getter;
import lombok.Setter;

/**
 * Flat DTO used as the data source bean for JasperReports.
 * Each field maps to a JRDesignField declared in the report's buildDesign() method.
 * Field names must match exactly between this DTO and the JRDesignField declarations.
 *
 * Uses @Getter/@Setter (Lombok) so that JRBeanCollectionDataSource can read
 * properties via standard JavaBean getters.
 */
@Getter
@Setter
public class StaffReportDTO {

    private String staffNumber;
    private String fullName;
    private String email;
    private String departmentName;
    private String positionName;
    private String allocationStatus;
    private String allocationPercentage;
}
```

---

## Key Design Decisions

### Why JRDesign API (Programmatic Layout) Instead of .jrxml Templates

| Concern                  | .jrxml Templates                         | JRDesign API (Programmatic)                |
|--------------------------|------------------------------------------|--------------------------------------------|
| AI-agent friendliness    | XML is verbose, error-prone for AI       | Pure Java — AI agents excel at Java code   |
| Compile-time safety      | Errors caught at runtime only            | Field/type mismatches caught at compile    |
| Dynamic layouts          | Requires JRXML manipulation or Scriptlets| Natural Java logic (conditionals, loops)   |
| Template management      | Separate .jrxml files under resources/   | No extra files — layout lives in Java code |
| Versioning               | Must track XML changes separately        | Layout changes appear in standard diffs    |
| Testability              | Hard to unit test XML templates          | buildDesign() returns testable objects     |

### Why DTO as Data Source

- **Module boundary**: Reports consume DTOs produced by domain services, not raw entities. This prevents
  reports from coupling to persistence layer details (table structure, lazy loading, etc.).
- **Testability**: Report data generation can be unit tested by mocking the service layer and verifying
  the returned DTO collection, independent of the database.
- **No SQL in reports**: All data access goes through the module's service layer, maintaining the
  single-responsibility principle and keeping business logic centralized.

---

## Operational Notes

### Memory Management

- **Report compilation caching**: The `ReportService` caches compiled `JasperReport` objects in a
  `ConcurrentHashMap` keyed by `reportId`. For applications with many reports, monitor heap usage.
  The `evictReport()` method allows clearing individual cached compilations.
- **Large data sets**: For reports generating tens of thousands of rows, consider paginating the data
  in the `generateData()` method or using JasperReports virtualizers to offload pages to disk:
  ```java
  JRSwapFileVirtualizer virtualizer = new JRSwapFileVirtualizer(
      50, new JRSwapFile(config.getTempPath(), 2048, 1024));
  jasperParams.put(JRParameter.REPORT_VIRTUALIZER, virtualizer);
  ```
- **ByteArrayOutputStream**: The entire exported report is held in memory before being written to
  the HTTP response. For very large reports (> 50 MB), consider streaming directly to the response
  output stream.

### Concurrency

- **Thread safety**: `ReportService` is a singleton Spring bean. The `ConcurrentHashMap` for compilation
  cache is thread-safe. `JasperFillManager.fillReport()` is stateless and safe to call concurrently
  with different parameters.
- **Rate limiting**: For CPU-intensive reports, consider adding `@RateLimiter` (Resilience4j) or a
  semaphore to limit concurrent report generation:
  ```java
  private final Semaphore reportSemaphore = new Semaphore(4); // max 4 concurrent reports
  ```

### Timeouts

- Report generation requests can be slow for large datasets. Configure a generous timeout in the
  REST client and consider async generation for reports exceeding 30 seconds:
  ```yaml
  spring:
    mvc:
      async:
        request-timeout: 120000  # 2 minutes for report generation
  ```

### Font Embedding

- JasperReports requires fonts to be available at runtime for PDF export. The `jasperreports-fonts`
  dependency bundles DejaVu Sans (the default). For custom fonts:
  1. Place `.ttf` files under `src/main/resources/fonts/`
  2. Create `jasperreports_extension.properties` with font family definitions
  3. Reference the font name in `ReportDesignHelper.createStyle()` calls
- When using programmatic design, set fonts explicitly on text elements:
  ```java
  textField.setFontName("DejaVu Sans");
  textField.setFontSize(10f);
  textField.setPdfEncoding("Identity-H");
  textField.setPdfEmbedded(true);
  ```
