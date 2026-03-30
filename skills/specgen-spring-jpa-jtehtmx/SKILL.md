---
name: specgen-spring-jpa-jtehtmx
description: >
  Generate a detailed specification document for building a monolith Spring Boot 3 web
  application with server-rendered views (JTE), Tailwind CSS, Alpine.js, htmx, and
  Spring Modulith packaging. Database (MongoDB, PostgreSQL, MySQL, or none), authentication
  (Keycloak OAuth2 Client, Spring Security form login, or none), scheduling (Quartz +
  Spring Batch or none), and messaging (RabbitMQ pub/sub or none) are configurable
  based on user input.
  Standardized input: application name (mandatory), version (mandatory), module (optional).
  Use this skill whenever the user asks to create a spec, specification, blueprint, or
  technical design document for a new Spring Boot web application with server-side rendering.
  Also trigger when the user says things like "spec out a new web project",
  "design a Spring Boot web skeleton", "write a technical spec for my new web app",
  "scaffold spec for a monolith web app", or any request for a specification document
  describing a Spring Boot + JTE + Tailwind application. Even if the user only mentions
  a subset of the stack (e.g., "Spring Boot web app" or "Spring web with Mongo" or
  "Spring Boot with Keycloak"), this skill likely applies — ask and confirm.
---

# Spring Boot Web Application Specification Generator

This skill generates a comprehensive specification document (Markdown) that serves as a
blueprint for building a monolith Spring Boot 3 web application with server-rendered
views. The spec is intended to be followed by a developer or a coding agent to produce
a fully functional project scaffold.

The specification does NOT generate code. It produces a detailed, opinionated technical
document describing every layer of the application — from Maven configuration to JTE
layouts to security filter chains — so that implementation becomes a mechanical exercise.

## Technology Stack

### Core Stack (Always Included)

These are the fixed versions the spec targets. Do not deviate unless the user explicitly
requests different versions.

| Component          | Version   |
|--------------------|-----------|
| Java JDK           | 21        |
| Spring Boot        | 3.5.7     |
| Maven              | 4.0.0     |
| JTE                | 3.2.1     |
| Tailwind CSS       | 4.x       |
| Alpine.js          | 3.x       |
| htmx               | 2.x       |
| Vite               | 6.x       |

### Optional Integration Versions

Include in the version table only when the corresponding integration is selected.

| Component          | Version   | When Selected        |
|--------------------|-----------|----------------------|
| MongoDB            | 8.0.19    | Database = MongoDB   |
| PostgreSQL         | 17.x      | Database = PostgreSQL|
| MySQL              | 8.4.x     | Database = MySQL     |
| Keycloak           | 26.5.3    | Auth = Keycloak      |
| RabbitMQ           | 4.x       | Messaging = yes OR Remote Partitioning = yes |

## Core Dependencies

The spec must include these in the Maven configuration section (always):

- Spring Web (starter-web)
- Spring Modulith (core, events-api)
- JTE Spring Boot Starter (`gg.jte:jte-spring-boot-starter-3`)
- JTE precompiler (`gg.jte:jte-maven-plugin` with `precompile` goal at `process-classes` phase, output to `target/classes` so precompiled templates are included in the Spring Boot fat JAR)
- Lombok
- Spring Boot DevTools
- MapStruct (with annotation processor)
- frontend-maven-plugin (for Node/Vite build)

### Conditional Dependencies

**If Database = MongoDB:**
- Spring Data MongoDB (`spring-boot-starter-data-mongodb`)
- MongoDB Driver
- Spring Modulith starter-mongodb (`spring-modulith-starter-mongodb`)

**If Database = PostgreSQL or MySQL:**
- Spring Data JPA (`spring-boot-starter-data-jpa`)
- PostgreSQL driver (`org.postgresql:postgresql`) or MySQL driver (`com.mysql:mysql-connector-j`)
- Flyway (`org.flywaydb:flyway-core`) for schema migration
- Spring Modulith starter-jpa (`spring-modulith-starter-jpa`)

**If Auth = Keycloak:**
- Spring Security (`spring-boot-starter-security`)
- Spring Security OAuth2 Client (`spring-boot-starter-oauth2-client`)

**If Auth = Spring Security (form login):**
- Spring Security (`spring-boot-starter-security`)

**If Scheduling = yes:**
- Spring Quartz (`spring-boot-starter-quartz`)
- If Database = MongoDB: `io.fluidsonic.mirror:fluidsonic-mirror-quartz` (MongoDB job store for Quartz)
- If Database = PostgreSQL/MySQL: Quartz JDBC job store (built-in)

**If Scheduling = yes AND Spring Batch = yes:**
- Spring Batch (`spring-boot-starter-batch`)

**If Scheduling = yes AND Spring Batch = yes AND Remote Partitioning = yes:**
- Spring Batch Integration (`spring-batch-integration`)
- Spring Integration AMQP (`spring-integration-amqp`)
- Spring Boot AMQP Starter (`spring-boot-starter-amqp`)

**If Messaging = yes:**
- Spring Boot AMQP Starter (`spring-boot-starter-amqp`)
  *(shared with Remote Partitioning — if both are selected, include the dependency once)*

**If Reporting = yes:**
- JasperReports (`net.sf.jasperreports:jasperreports:7.0.3`)
- JasperReports Fonts (`net.sf.jasperreports:jasperreports-fonts:7.0.3`)
- OpenPDF (`com.github.librepdf:openpdf:2.0.4`) — PDF export engine for JasperReports 7.x
- Apache POI OOXML (`org.apache.poi:poi-ooxml:5.4.1`) — XLSX export support

## When the Skill Triggers

Generate the spec when the user provides an **application name** and **version** that
corresponds to one of the custom applications defined in `CLAUDE.md`. The skill
reads all required inputs from the project's context files — no interactive Q&A is needed
for the core inputs.

The user invokes this skill by specifying the target application and version, for example:
- `/specgen-spring-jpa-jtehtmx hub_middleware v1.0.3`
- `/specgen-spring-jpa-jtehtmx hub_middleware v1.0.3 module:Location Information`
- `/specgen-spring-jpa-jtehtmx "Hub Middleware" v1.0.3`

The skill then locates the matching context folder and reads all input files automatically.

## Version Gate

Before starting any work, check `CHANGELOG.md` in the project root:

1. If `CHANGELOG.md` does not exist, skip this check (first-ever execution).
2. If `CHANGELOG.md` exists, scan all `## vX.Y.Z` headings and determine the **highest version** using semantic versioning comparison.
3. Compare the requested version against the highest version:
   - If requested version **>=** highest version: proceed normally.
   - If requested version **<** highest version: **STOP immediately**. Print: `"Version {requested} is lower than the current project version {highest} recorded in CHANGELOG.md. Execution rejected."` Do NOT proceed with any work.

## Input Resolution

This skill uses standardized input resolution. Provide:

| Argument | Required | Example | Description |
|----------|----------|---------|-------------|
| `<application>` | Yes | `hub_middleware` | Application name to locate the context folder |
| `<version>` | Yes | `v1.0.3` | Version to scope processing |
| `module:<name>` | No | `module:Location Information` | Limit generation to a single module |

### Application Folder Resolution

The application name is matched against root-level application folders:
1. Strip any leading `<number>_` prefix from folder names (e.g., `1_hub_middleware` → `hub_middleware`)
2. Match case-insensitively against the provided application name
3. Accept snake_case, kebab-case, or title-case input (all match the same folder)
4. If no match found, list available applications and stop

### Auto-Resolved Paths

| File | Resolved Path |
|------|---------------|
| PRD.md | `<app_folder>/context/PRD.md` |
| Module Models | `<app_folder>/context/model/` |
| HTML Mockups | `<app_folder>/context/mockup/` |
| Output (specification) | `<app_folder>/context/specification/` |

### Example Invocations

- `/specgen-spring-jpa-jtehtmx hub_middleware v1.0.3` (all modules)
- `/specgen-spring-jpa-jtehtmx hub_middleware v1.0.3 module:Location Information` (one module)
- `/specgen-spring-jpa-jtehtmx "Hub Middleware" v1.0.3`

### Version Filtering

When a version is provided, only include user stories, NFRs, and constraints from versions
<= the provided version. For example, if `v1.0.3` is specified:
- Include items tagged `[v1.0.0]`, `[v1.0.1]`, `[v1.0.2]`, `[v1.0.3]`
- Exclude items tagged `[v1.0.4]` or later
- Version comparison uses semantic versioning order

### Module Filtering

When `module:<name>` is provided:
- Only generate the `SPEC.md` for that specific module
- Other existing module spec files remain untouched
- `SPECIFICATION.md` (root) gets a partial update — only that module's entry in the TOC
  is added or updated; all other TOC entries are preserved as-is

## Gathering Input

The specification is driven by **six input sources** that are read from the project's
context files. The skill does NOT ask the user for database, authentication, scheduling,
or messaging choices — it **determines** these automatically from the context.

### Input 1: Application Name (from CLAUDE.md)

From CLAUDE.md (already loaded in context), locate the target application under the **Custom Applications**
section. Extract:

- **Application name**: The section heading (e.g., "Hub Middleware", "HC Adapter")
- **Application description**: The description paragraph below the heading
- **Dependencies**: The "Depends on" list — this is the primary source for determining
  optional components (see [Determining Optional Components](#determining-optional-components))

The application name is used to derive:
- **Artifact ID**: Kebab-case of the application name (e.g., `hub-middleware`)
- **Group ID**: `com.bestinet.urp` (project-level constant)
- **Base package**: `com.bestinet.urp.<artifactid_no_hyphens>` (e.g., `com.bestinet.urp.hubmiddleware`)

### Input 2: User Stories (from PRD.md)

Read `<app_folder>/context/PRD.md`. This file contains all user stories
organized by module. Extract:

- **System modules**: Modules under the `# System Module` heading (e.g., User, Notification,
  Activities, Audit Trail). These become system-level modules in the spec.
- **Business modules**: Modules under the `# Business Module` heading (e.g., Location
  Information, Corridor, Employer). These become business-level modules.
- **User stories per module**: Each `### User Story` section contains tagged items like
  `[USHM00108] As a user, I want to...`. These define the functional requirements for
  each module's service interface and page controllers.

The user stories directly inform:
- Which CRUD operations each module's service must expose
- Which page controllers and JTE views are needed
- Which form fields and validation rules apply

**Important:** Items with strikethrough (`~~text~~`) are deprecated — do NOT include them
as active requirements. Instead, list them in the "Removed / Replaced" subsection of the
traceability table (see spec-template.md) so that developers can see what was removed and
which version removed it. If a deprecated item has a replacement (e.g., `USHM00015` replaced
by `USHM00222`), note the replacement ID.

- **Version tags per item**: Each `### User Story`, `### Non Functional Requirement`,
  and `### Constraint` section contains one or more version blocks formatted as `[v1.0.x]`.
  Items listed under each version tag belong to that version. The skill must track the
  version tag for each item (user story, NFR, constraint) and carry it through to the
  generated specification's traceability section. When a version block explicitly lists
  "Removed ... from previous version", record those removals in a "Removed / Replaced"
  subsection with the version that removed them and the replacement ID (if any).

### Input 3: Non-Functional Requirements (from PRD.md)

Within the same `PRD.md`, each module has a `### Non Functional Requirement` section
with tagged items like `[NFRHM0120]`. These inform:

- Data storage strategies (e.g., "stored in JWT token", "stored in the database")
- Integration patterns (e.g., "API call to SSO service", "sent asynchronously")
- Performance constraints (e.g., "automatically deleted after 30 days")
- Security requirements (e.g., "require user to relogin")

NFRs should be mapped to specific technical decisions in the spec — for example, an NFR
stating "stored in JWT token" confirms stateless auth, while "sent asynchronously" confirms
event-driven processing.

### Input 4: Constraints (from PRD.md)

Within the same `PRD.md`, each module has a `### Constraint` section with tagged
items like `[CONSHM042]`. These define hard boundaries that the spec must enforce:

- Field-level restrictions (e.g., "can only change name and phone number")
- Feature boundaries (e.g., "only 2 delivery channels supported")
- Scope limitations (e.g., "does not manage any user, permissions and roles")

Constraints are embedded directly into the relevant module blueprint — they inform
service interface contracts, validation rules, and UI form configurations.

### Input 5: Module Model (from model/ folder)

Read `<app_folder>/context/model/MODEL.md` first as the index, then read the
individual module model files in each module subfolder.

**MODEL.md** provides:
- Summary table of all modules with collection counts and design decisions
- Links to each module's detailed model files

**Per-module files** (e.g., `model/location-information/model.md`):
- Complete document/entity field definitions with types and constraints
- Embedded vs. referenced relationships
- Collection/table names
- Index specifications
- Audit field patterns

**Per-module schema** (e.g., `model/location-information/schemas.json`):
- JSON schemas defining exact field types, required fields, and validation rules

**Per-module diagram** (e.g., `model/location-information/document-model.mermaid`):
- Visual representation of document structure and relationships

The module model directly maps to:
- MongoDB documents or JPA entities (field-for-field, not placeholder)
- Repository methods and query patterns
- MapStruct mapper definitions
- DTO structures matching the actual module fields
- Service interface method signatures

- **Version tracking**: The MODEL.md summary table includes a "Versions" column listing
  which versions each module participates in (e.g., "1.0.0, 1.0.1, 1.0.3"). Per-module
  model.md files may also include version annotations on fields and indexes. The skill
  must carry these version tags into the generated specification.

### Input 6: HTML Mockup Screens (from mockup/ folder)

Read `<app_folder>/context/mockup/MOCKUP.html` first as the index page, then
read the HTML files organized by role in subfolders.

**MOCKUP.html** provides:
- Application identity (name, version, short name)
- Design tokens: fonts, colors, spacing from Tailwind config
- List of user roles and their screen sets
- Setup instructions for the mockup server

**Role-specific subfolders** (e.g., `mockup/hub_administrator/content/`):
- Individual HTML screens for each page in the application
- Screen layout: which components are used (tables, forms, cards, tabs, modals)
- Navigation structure from sidebar HTML files
- Data display patterns (list pages, detail pages, create/edit forms)

**IMPORTANT — Role folders inform access control, NOT URL paths.** The role-specific
folder structure (e.g., `mockup/hub_administrator/content/corridor.html`) determines:
1. Which role can access the page → `@PreAuthorize("hasRole('HUB_ADMINISTRATOR')")`
2. Which sidebar navigation items appear for each role
It does NOT determine the URL path. The URL path is always module-based:
- `@RequestMapping("/corridor")` — NOT `@RequestMapping("/hub_administrator/corridor")`
- Fragment URL: `@RequestMapping("/corridor/fragments")` — NOT `@RequestMapping("/api/content/hub_administrator/corridor")`

**Shared partials** (e.g., `mockup/partials/`):
- `header.html` — Header bar layout and elements
- `footer.html` — Footer layout
- `sidebar-<role>.html` — Per-role navigation menus
- `shell.html` — Page shell/wrapper structure

The mockup screens directly map to:
- JTE page templates (one per HTML screen)
- JTE fragment templates (for htmx partial updates)
- Page controller endpoints (one per screen)
- View model classes (with fields matching the data shown in each screen)
- Sidebar navigation items per role
- Form field layouts and validation display
- Design tokens for Tailwind configuration (colors, fonts from MOCKUP.html)

- **Version tracking**: The MOCKUP.html index page shows a version tag on each screen
  card (e.g., `v1.0.0`, `v1.0.3`). Individual mockup screens may include version
  annotations. The skill must associate each mockup screen with its version and carry
  this through to the generated specification.

## Determining Optional Components

Instead of asking the user, the skill determines optional components by analyzing the
dependencies listed in `CLAUDE.md` and cross-referencing with PRD.md NFRs and
constraints.

### Database Detection

Examine the "Depends on" list in CLAUDE.md for the target application:

| Dependency Pattern | Database Selection |
|---|---|
| References "Hub Database" (MongoDB) | Database = MongoDB |
| References "HC Database" or "SC Database" (MySQL) | Database = MySQL |
| No database dependency listed | Database = none |

Also check `CLAUDE.md`'s database section for the exact database name, and read
`SECRET.md` (in the project root) for host, port, and credentials to use in the
spec's application configuration.

### Authentication Detection

| Dependency Pattern | Auth Selection |
|---|---|
| References "Hub Single Sign On" (Keycloak) | Auth = Keycloak |
| PRD.md constraint says "does not manage any user, permissions and roles" | Auth = none |
| PRD.md NFRs reference "SSO service" or "JWT token" | Auth = Keycloak |
| No SSO/Keycloak dependency but has user management stories | Auth = form |

If Auth = Keycloak, also extract from CLAUDE.md:
- Keycloak version from "Hub Single Sign On" section
- Keycloak realm: Default derived from project name
- Keycloak client ID: Default `<artifact-id>-web`
- Keycloak issuer URI: Default `http://localhost:8180/realms/<realm>`
- Keycloak roles: Infer from mockup sidebar roles (e.g., `hub_administrator` → `HUB_ADMINISTRATOR`)

### Messaging Detection

| Dependency Pattern | Messaging Selection |
|---|---|
| References "Hub to HC Adapter Message Queue" or "Hub to SC Adapter Message Queue" (RabbitMQ) | Messaging = yes |
| No message queue dependency listed | Messaging = no |

If Messaging = yes, also extract the RabbitMQ version from the corresponding message
queue section in CLAUDE.md.

### Scheduling Detection

Scheduling is determined from PRD.md content:

| Content Pattern | Scheduling Selection |
|---|---|
| NFRs mention "automatically deleted after X days", "scheduled", "periodic", "batch processing" | Scheduling = yes |
| User stories describe recurring jobs, cleanup tasks, or time-triggered operations | Scheduling = yes |
| No scheduling-related requirements found | Scheduling = no |

**If Scheduling = yes**, further determine:
- **Spring Batch = yes** if NFRs mention "batch processing", "ETL", "bulk import/export",
  or processing large volumes of data with reader/processor/writer patterns
- **Remote Partitioning = yes** if NFRs mention "distributed processing", "horizontal
  scaling", or "partitioned batch jobs"

### Reporting Detection

Reporting is determined from PRD.md content:

| Content Pattern | Reporting Selection |
|---|---|
| NFRs mention "report", "Report interface", "generate report", "report generation" | Reporting = yes |
| User stories describe generating/downloading PDF, Excel, or CSV reports | Reporting = yes |
| A "Report" module exists in PRD.md with NFRs defining a Report interface | Reporting = yes |
| No reporting-related requirements found | Reporting = no |

**If Reporting = yes**, the spec includes:
- Jasper Reports compilation and export infrastructure
- `ReportDefinition` interface for modules to implement
- `ReportService` orchestrating compile → fill → export via `JRBeanCollectionDataSource`
- Report registry persisted in the database
- Report page controller with parameter form and download endpoint
- JTE templates for report list and parameter form pages
- Multi-format export: PDF (OpenPDF), XLSX (Apache POI), CSV

### Summary of Determination

After analyzing all inputs, produce a determination summary before generating the spec.
Present it to the user for confirmation:

```
Optional Component Determination:
- Database:      MongoDB (from CLAUDE.md → depends on Hub Database)
- Authentication: Keycloak (from CLAUDE.md → depends on Hub Single Sign On)
- Scheduling:    yes (from PRD.md → NFR mentions automatic deletion)
  - Spring Batch: no
  - Remote Partitioning: no
- Messaging:     yes (from CLAUDE.md → depends on Hub to HC/SC Adapter Message Queue)
- Reporting:     yes (from PRD.md → Report module with Report interface NFR)
```

If the user disagrees with any determination, allow them to override before proceeding.

## Required Inputs

After determination, these values are needed. Most are derived automatically:

**Auto-derived from context files:**
- **Application name**: From CLAUDE.md section heading
- **Artifact ID**: Kebab-case of application name
- **Group ID**: `com.bestinet.urp`
- **Base package**: `com.bestinet.urp.<artifactid>`
- **Application description**: From CLAUDE.md description
- **Modules**: From PRD.md module headings + model/MODEL.md
- **Database**: Auto-determined (see above)
- **Authentication**: Auto-determined (see above)
- **Scheduling**: Auto-determined (see above)
- **Messaging**: Auto-determined (see above)
- **Database name/credentials**: From SECRET.md (root-level file with local environment credentials)
- **User roles**: From mockup sidebar files
- **Design tokens**: From MOCKUP.html Tailwind config

**Optional (use sensible defaults if not found in context):**
- **Server port**: Default `8080`
- **Default theme**: Default `light` (supports `light`/`dark`)
- **Log level**: Default `INFO` for application, `WARN` for frameworks

## Generating the Specification

Once inputs are gathered from context files and optional components are determined,
generate the specification as a **multi-file output split by module**. Read the spec
template at `references/spec-template.md` for the exact structure and content of each
section. The template is the authoritative guide — follow it closely.

The specification is split into two categories:

1. **Root `SPECIFICATION.md`** — Contains the Table of Contents, shared infrastructure,
   and application-level sections that apply across all modules.
2. **Per-module `<module-name>/SPEC.md`** — Each module gets its own folder with
   a self-contained specification file covering that module's complete blueprint.

This split enables a coding agent to:
- First, scaffold the shared infrastructure from `SPECIFICATION.md`
- Then, implement each module independently by picking up its `<module>/SPEC.md`

**Important:** The generated spec must use **real module data** from the context files,
not generic placeholders. Specifically:

- **Modules** must use the actual module names from PRD.md and MODEL.md
  (e.g., `locationInformation`, `corridor`, `employer` — not `module1`, `module2`)
- **Document/Entity fields** must match the actual fields defined in the module model
  files (e.g., `model/location-information/model.md`), not placeholder `fieldOne`/`fieldTwo`
- **Service interfaces** must expose methods matching the actual user stories (e.g., if
  a story says "view provinces by country", the service needs `listProvincesByCountry()`)
- **Page controllers** must map to the actual mockup screens (e.g., if
  `hub_administrator/content/location_information.html` exists, there must be a matching
  controller endpoint and JTE page). The controller URL is module-based (e.g.,
  `/location-information`), NOT role-prefixed (e.g., NOT `/hub_administrator/location-information`).
  The mockup's role folder determines `@PreAuthorize` annotations, not URL structure.
- **Sidebar navigation** must match the mockup sidebar files per role. Sidebar hrefs use
  module-based paths (e.g., `/corridor`, `/quota`) — the role determines which items
  appear in the sidebar, not the URL prefix
- **Form fields** must match what the mockup screens display
- **Design tokens** (colors, fonts) must match the MOCKUP.html Tailwind configuration
- **Version tags**: Every user story ID, NFR ID, constraint ID, and mockup screen in
  the traceability section must include its version tag (e.g., `USHM00228 [v1.0.3]`).
  This enables incremental implementation by version. **ALL traceability sub-tables
  (User Stories, NFRs, AND Constraints) MUST include the `| Version |` column.** Do not
  omit the Version column from any table — even if a module only has v1.0.0 items.
- **Removed / Replaced items**: The traceability section must include a "Removed / Replaced"
  subsection listing any deprecated items from previous versions — showing the removed ID,
  the version that removed it, the replacement ID (if any), and a brief reason. This
  ensures developers can see what was removed and understand the evolution of requirements.
  If a module has no removed items, include the subsection with `_None._` to make it
  explicit that nothing was removed.

### Output Structure

```
<app_folder>/context/specification/
├── SPECIFICATION.md                    ← TOC + shared/application-level specs
├── location-information/
│   └── SPEC.md                         ← Module blueprint for Location Information
├── corridor/
│   └── SPEC.md                         ← Module blueprint for Corridor
├── employer/
│   └── SPEC.md                         ← Module blueprint for Employer
├── ...                                 ← One folder per module from PRD.md
```

### What Goes in `SPECIFICATION.md` (Root)

The root file contains the TOC and all shared/application-level sections. These are the
sections that a coding agent implements **first** before any module work:

#### 1. Project Overview
Project metadata, application description, technology stack summary, the complete
list of user roles extracted from mockup sidebar files, and the **Module Index** —
a table listing every module with a link to its `<module>/SPEC.md` file.

#### 2. Maven Configuration
Complete `pom.xml` structure with all dependencies (core + selected conditional),
plugin configurations (MapStruct annotation processor, Spring Boot Maven plugin,
frontend-maven-plugin for Vite build, `maven-clean-plugin` to delete on-demand
`jte-classes/` folder, `jte-maven-plugin` with `precompile` goal at `process-classes`
phase outputting to `${project.build.directory}/classes`), and property management.

#### 3. Application Configuration *(conditional content varies)*
A single `application.yml` (no profile-specific files like `application-dev.yml` or
`application-prod.yml`) covering database connection (MongoDB URI or JDBC datasource
depending on selection), auth settings (Keycloak/OAuth2 if selected, or form login if
selected), JTE configuration (with `JTE_DEV_MODE` and `JTE_PRECOMPILED` env vars),
scheduling config (if selected), theme defaults, and logging configuration. **All environment-sensitive values (ports, hostnames, credentials,
URIs) MUST use Spring's `${ENV_VAR:default}` syntax** to allow externalization via
environment variables while keeping sensible defaults for local development. Do NOT use
Spring profiles or profile-specific YAML files — environment differences are handled
entirely through environment variables (e.g., via `.env` file locally or system
environment variables in deployment).

#### 3a. Application Version Configuration
The `application.yml` MUST include an `app.version` property set via environment variable
with a default derived from the version argument provided during skill invocation. If
multiple versions were provided, use the highest one.

```yaml
app:
  version: ${APP_VERSION:1.0.0}
```

The application MUST expose this version in the **footer** of every page. The shared footer
layout (`footer.jte` or `footer.html`) must read `app.version` from a Spring `@Value`-injected
bean or controller model attribute and render it as: `v{version}` (e.g., `v1.0.3`).

The `.env` file must include `APP_VERSION={version}` with the actual version value.

The `pom.xml` `<version>` element MUST also be set to the version value (e.g., `1.0.3`).

#### 3b. `.env` File Generation from SECRET.md
Generate a `.env` file at the project root by reading `SECRET.md` from the project root.
The `.env` file maps SECRET.md credential and platform values to the environment variable
names referenced in `application.yml`. The spec must define the complete `.env` content
with actual values from SECRET.md.

**Process:**
1. Read `SECRET.md` from the project root
2. Extract relevant values from `# Credential` section (database hosts, ports, usernames,
   passwords) and `# Platform` section (JDK path, Maven path, Node.js path, etc.)
3. Map each value to the corresponding `${ENV_VAR}` name used in `application.yml`
4. Generate the `.env` file with `KEY=value` pairs

**Example `.env` output (derived from SECRET.md):**
```properties
# Database
DB_HOST=localhost
DB_PORT=3306
DB_NAME=hub_supp
DB_USERNAME=root
DB_PASSWORD=B3st1n3t@2025

# Authentication (Keycloak)
KEYCLOAK_HOST=http://localhost:8180
KEYCLOAK_REALM=urp
KEYCLOAK_CLIENT_ID=hub-middleware-web

# Messaging (RabbitMQ)
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USERNAME=guest
RABBITMQ_PASSWORD=guest

# Server
SERVER_PORT=8080

# Platform
JAVA_HOME=C:\Users\rashidee.rashid.BESTINET\.jdks\azul-21.0.9
MAVEN_HOME=C:\Users\rashidee.rashid.BESTINET\apache-maven-3.9.12
```

**Rules:**
- Only include variables that are actually referenced in `application.yml`
- Use actual values from SECRET.md — never use placeholders or `TODO`
- If SECRET.md does not exist or a value is not found, use sensible defaults for local
  development (e.g., `localhost`, default ports)
- The `.env` file is gitignored (already covered in `.gitignore`)

#### 4. Build & Tooling
Vite configuration, Tailwind CSS setup (using design tokens from MOCKUP.html), PostCSS
config, frontend directory structure, and Maven integration for building JS/CSS assets.

#### 4b. `.gitignore`
Generate a `.gitignore` file at the project root that excludes all generated, downloaded,
and environment-specific files from version control. The spec must include the complete
`.gitignore` content. At minimum, include:

```
# Build output
target/

# IDE
.idea/
*.iml
.vscode/
.settings/
.project
.classpath
*.swp
*~

# OS
.DS_Store
Thumbs.db

# Frontend
frontend/node_modules/
frontend/.vite/
src/main/resources/static/dist/

# JTE generated
jte-classes/

# E2E
e2e/node_modules/
e2e/test-results/
e2e/playwright-report/
e2e/visual-baselines/

# Logs
*.log
logs/

# Environment
.env
.env.*
```

**Conditional additions:**
- **If Database = PostgreSQL/MySQL:** No additional entries needed (Flyway migrations are committed)
- **If Scheduling = yes AND Spring Batch = yes:** Add `*.tmp` for batch temp files
- **If Reporting = yes:** Add `*.jasper` for compiled JasperReports templates (regenerated at build time)

#### 5. Package Structure
The complete directory tree rooted at the base package. The structure follows
Domain-Driven Design with Spring Modulith conventions, adapted for server-rendered web
with `page` and `fragment` subpackages. Use actual module names from the context.
Read `references/modulith-patterns.md` for the detailed module layout and inter-module
communication rules.

#### 6. Security Configuration *(conditional — include only if Auth != none)*
**If Auth = Keycloak:** OAuth2 Client setup with Keycloak OIDC, SecurityFilterChain
with `oauth2Login()`, `KeycloakGrantedAuthoritiesMapper` to extract roles from ID token
claims, CSP nonce filter, automatic redirect to Keycloak login, and public endpoint
configuration. Session-based (Spring default). Read `references/security-patterns.md`
for the full security architecture.

**If Auth = form:** Spring Security form login configuration, `SecurityFilterChain` with
`formLogin()`, `UserDetailsService` implementation backed by the selected database, password
encoding with BCrypt, role-based access control, CSP nonce filter, and remember-me support.

#### 7. Layouts (JTE)
Three layouts: MainLayout (header/sidebar/footer for authenticated pages), AuthLayout
(split-screen for login), ErrorLayout (centered error display). Each with Java view
model class and JTE template. Layout structure should match the mockup shell/partials.
*Note: If Auth = none, AuthLayout may be omitted or simplified.*

#### 8. Fragments
Reusable JTE fragments: HeaderFragment, SidebarFragment, FooterFragment,
ThemeSwitcherFragment. Each with view model and template. Sidebar must include the actual
navigation items from the mockup sidebar files, organized per role.

#### 9. UI Components (Tailwind-based)
Pure Tailwind CSS components (no DaisyUI): Alert, Avatar, Badge, Breadcrumb, Button,
Card, Drawer, Dropdown, FormControl/Input/Select/Textarea/Checkbox/Radio/Toggle,
Loading, Menu, Modal, Navbar, Pagination, Progress, Stats, Steps, Table, Tabs,
Toast, Tooltip. Each with Java view model and JTE template using Tailwind utility classes.

#### 10. Frontend JS Structure
Alpine.js stores (toast, modal, drawer, theme), htmx extensions (loading states, toast
errors), ES module organization, and CSP nonce integration.

#### 11. Styles
Tailwind CSS organization: base (reset, typography), components (buttons, forms, cards,
tables, modals, navigation), layouts (sidebar, navbar, footer responsive), themes
(light/dark CSS custom properties). Use the actual design tokens (colors, fonts) from
MOCKUP.html.

#### 12. Authentication Pages *(conditional — include only if Auth != none)*
**If Auth = Keycloak:** No custom login page needed. Spring Security automatically
redirects unauthenticated users to Keycloak login. Keycloak handles the login form,
social login providers, and credential management. After successful authentication,
Keycloak redirects back with an authorization code which Spring exchanges for tokens.

**If Auth = form:** Login page with email/password form, CSRF token, remember-me checkbox.
Logout endpoint. Optionally a registration page if user self-registration is desired.

#### 13. Data Access *(conditional content varies)*
**If Database = MongoDB:** MongoDB repositories per module with Spring Data MongoRepository.
DTO mapping via MapStruct. Pagination helpers with `PaginationAware` mixin. All list
endpoints paginated. Use actual collection names from the module model.

**If Database = PostgreSQL/MySQL:** JPA repositories per module with Spring Data
JpaRepository. JPA entities with `@Entity`/`@Table` annotations extending `BaseEntity`.
`BaseEntity` uses `@GeneratedValue(strategy = GenerationType.UUID)` with
`@Column(name = "id", columnDefinition = "CHAR(36)")`. Flyway migration scripts.
DTO mapping via MapStruct. Pagination helpers. All list endpoints paginated. Use actual
table names from the module model.
**MySQL UUID rule:** All UUID primary keys and foreign keys use `CHAR(36)` in DDL.
In JPA entities, map with `columnDefinition = "CHAR(36)"` (not `length = 36`) to
pass Hibernate `ddl-auto: validate`.

**If Database = none:** In-memory data structures or stubs. The spec should note that a
database integration can be added later.

#### 14. Error Handling & Exceptions
Base `WebApplicationException` with status/code/userMessage. `GlobalExceptionHandler`
returning error pages for full requests and toast fragments for htmx requests.

#### 15. Theming
Light/dark theme switching via cookie. Server reads cookie and passes theme to layout
view model. Alpine store syncs toggle with cookie and `data-theme` attribute.

#### 16. Pagination Support
PaginationRequest, PaginatedResult, and the shared pagination JTE component.

#### 17. Logging Strategy
SLF4J with Logback configuration, correlation ID (from JWT if auth is enabled, or
request-scoped UUID otherwise), MDC context filter, per-module log levels.

#### 18. Scheduling and Batch Processing *(conditional — include only if Scheduling = yes)*
Quartz scheduler configuration with job store matching the selected database (MongoDB
job store via fluidsonic-mirror if MongoDB, JDBC job store if relational, in-memory if
no database). If Spring Batch = no, Quartz runs standalone with direct service-call jobs.
If Spring Batch = yes, Quartz triggers Spring Batch jobs with chunk-oriented
reader/processor/writer processing. If Remote Partitioning = yes (requires Spring Batch),
also includes Spring Batch remote partitioning with RabbitMQ for horizontal scaling of
batch jobs across multiple worker nodes. Read `references/batch-patterns.md` for the full
scaffolding spec (Quartz-only, Quartz+Batch, and remote partitioning patterns).

#### 19. Event-Driven Architecture
Spring Modulith application events for inter-module communication. Read
`references/modulith-patterns.md` for details.

#### 20. Messaging (RabbitMQ Pub/Sub) *(conditional — include only if Messaging = yes)*
Standalone RabbitMQ publisher/consumer services for inter-system communication.
Topic exchange for event broadcasting, direct exchange for point-to-point commands.
Read `references/messaging-patterns.md` for the full messaging architecture.

#### 21. MapStruct Usage
Per-module MapStruct mapper conventions and mapping flow patterns.

#### 22. Testing Strategy
Overview of testing approach — unit tests, module tests, security test utilities
(JWT mocking if Keycloak, or `@WithMockUser` if form login), view model tests.

#### 23. Reporting (Jasper Reports) *(conditional — include only if Reporting = yes)*
JasperReports infrastructure with DTO-based data sources via `JRBeanCollectionDataSource`.
Includes `ReportDefinition` interface for modules to implement, `ReportService`
orchestrating compile → fill → export, `ReportRegistry` for auto-discovering and persisting
report definitions at startup, report page controller with parameter form and download
endpoint, JTE templates for report list and parameter form, multi-format export (PDF via
OpenPDF, XLSX via Apache POI, CSV). Modules register reports by creating `@Component`
classes implementing `ReportDefinition` — each report calls module services to produce DTOs
(never repositories directly), preserving Spring Modulith module boundaries. Read
`references/jasper-patterns.md` for the full reporting architecture.

### What Goes in Each `<module>/SPEC.md` (Per-Module)

For EACH module from PRD.md and MODEL.md, create a folder named after the
module (kebab-case, e.g., `location-information/`) and generate a `SPEC.md` inside it.

Each module `SPEC.md` is a **self-contained** blueprint that a coding agent can pick up
and implement independently (after the shared infrastructure is in place). It must include:

- **Header** with module name and a back-reference to the root `SPECIFICATION.md`
- **Traceability**: User story IDs, NFR IDs, constraint IDs, collection/table names,
  mockup screen filenames
- **Service interface** with methods derived from user stories
- **DTOs** with fields matching the module model's document/entity fields
- **Exception class** for module-specific errors
- **Document/Entity class** with fields from `model/<module>/model.md` and
  `model/<module>/schemas.json`
- **Repository** with query methods matching user stories' data access patterns
- **MapStruct mapper** with mappings matching actual field names
- **Service implementation** with full CRUD logic
- **Page controllers** with endpoints matching mockup screens for that module
- **Fragment controllers** for htmx partial updates
- **View models** with fields matching what mockup screens display
- **JTE templates** (list, detail, create, edit, row fragments)
- **Complete code samples** for every component — continuous and copy-pasteable

**Separating UI Layer from Messaging/Async Pipeline:**

When a module has BOTH user-facing screens (user stories) AND async processing NFRs
(e.g., RabbitMQ message consumption, message validation, ACK publishing, forwarding),
the SPEC.md MUST clearly separate these into distinct sections:

1. **UI Layer sections** — service methods for read-only queries (search, getById,
   getHistory), page controllers, fragment controllers, JTE templates. These are driven
   by user stories (USHMxxxxx).
2. **Messaging Pipeline sections** — message consumer (`@RabbitListener`), message
   validator, ACK publisher, forward publisher, queue configuration, domain events,
   `processIncomingMessage()` service method. These are driven by NFRs (NFRHMxxxx).

This separation enables the implementation orchestrator to track and implement each
concern independently. A module may have its UI layer fully implemented while its
messaging pipeline remains pending (e.g., waiting for an upstream adapter). The
IMPLEMENTATION_MODULE.md checklist tracks each user story and NFR individually, so
the status accurately reflects what is done vs. what remains.

See `references/spec-template.md` Section "Module SPEC.md Template" for the
exact template structure.

## Changelog Append

After all specification files are successfully generated, append an entry to `CHANGELOG.md` in the project root:

1. Read `CHANGELOG.md` from the project root. If it does not exist, create it with:
   ```markdown
   # Changelog

   - This file tracks all skill executions by version across all applications.
   - The highest version recorded here is the current project version.
   - Skills MUST NOT execute for a version lower than the highest version in this file.

   ---
   ```
2. Search for a `## {version}` heading matching the current version.
3. If the section **exists**: append a new row to its table.
4. If the section **does not exist**: insert a new section after the `---` below the context header and before any existing `## vX.Y.Z` section (newest-first ordering), with a new table header and the first row.
5. Row format: `| {YYYY-MM-DD} | {application_name} | specgen-spring-jpa-jtehtmx | {module or "All"} | Generated Spring Boot web application technical specification |`
6. **Never modify or delete existing rows.**

## Output Format

The generated specification is a **folder of files**, not a single document:

```
<app_folder>/context/specification/
├── SPECIFICATION.md                    ← Root: TOC + shared/application-level specs
├── <module-1>/
│   └── SPEC.md                         ← Module blueprint (self-contained)
├── <module-2>/
│   └── SPEC.md
├── <module-N>/
│   └── SPEC.md
```

- **Root folder**: `<app_folder>/context/specification/`
- **Root file**: `SPECIFICATION.md` — contains the Table of Contents (with links to
  each module's `SPEC.md`), all shared infrastructure sections (1–9), and all
  application-level cross-cutting sections (10–22, excluding module blueprints)
- **Module folders**: One folder per module from PRD.md, named in kebab-case
  (e.g., `location-information/`, `corridor/`, `employer/`)
- **Module files**: Each `SPEC.md` is self-contained with full code samples for that
  module's module — service, DTOs, document/entity, repository, mapper, controllers,
  views, and JTE templates

**Design goal:** A coding agent can implement the application module-by-module:
1. First pass: implement shared infrastructure from `SPECIFICATION.md`
2. Subsequent passes: pick up each `<module>/SPEC.md` independently and implement it

**Sample code is mandatory.** Every component described in any spec file must include a
complete, self-explanatory code sample inside a fenced code block (Java, JTE, JavaScript,
YAML, or HTML as appropriate). The sample code must be continuous (not fragments stitched
with "// ..." gaps) and usable as a direct reference by a coding agent or developer. A
section that only has bullet-point descriptions without accompanying code is incomplete.
The code samples in `references/spec-template.md` and the other reference files show the
expected level of detail — reproduce that level in the generated spec, substituting the
actual project values from the context files.

## Constraints

These constraints are non-negotiable. Every code sample in the generated spec must
follow them. If any constraint is violated, the spec is incorrect.

### Universal Constraints (Always Apply)

**Do NOT use Lombok's `@Data` annotation — anywhere.** Use `@Getter` and `@Setter`
explicitly on every class that needs accessors. `@Data` generates `equals()`, `hashCode()`,
and `toString()` which cause problems with persistent documents/entities, lazy-loaded
collections, and circular references. The spec must use `@Getter @Setter` consistently.

**Do NOT use field injection.** Every Spring-managed bean must use constructor injection.
Use Lombok's `@RequiredArgsConstructor` with `private final` fields. Never use `@Autowired`
on fields. This applies to services, controllers, configurations, filters, and any other
Spring component.

**Do NOT use DaisyUI.** All UI components must be built with pure Tailwind CSS utility
classes. No DaisyUI plugin, no DaisyUI class names (no `btn`, `card`, `navbar`, `alert`,
`badge`, `modal`, `drawer`, `menu`, `dropdown`, `tabs`, `toast`, `tooltip`, `table`,
`stats`, `steps`, `progress`, `avatar`, `breadcrumbs`, `swap`, `join`, `indicator`,
`input`, `select`, `textarea`, `checkbox`, `radio`, `toggle`, `range`, `file-input`,
`form-control`, etc.). Build equivalent components using Tailwind utility classes only.

**Use MapStruct for all object mapping.** Define MapStruct mappers per module with
`@Mapper(componentModel = "spring")`. Avoid handwritten mapping beyond simple passthroughs.

**URL routing must be module-based, NOT role-prefixed.** Controller `@RequestMapping`
paths use only the module name (e.g., `/corridor`, `/quota`). Never embed role names
in URL paths (e.g., NOT `/hub_administrator/corridor`). Role-based access control is
enforced via `@PreAuthorize` annotations on controllers, derived from the mockup's
role folder structure. Fragment controllers use `/{module}/fragments` (e.g.,
`/corridor/fragments`), NOT `/api/content/{role}/{module}`.

**All data listings must be paginated.** Every page and fragment that renders a list
must use the shared pagination component. Never render unbounded lists.

**Use real module data from context files.** The spec must use actual field names,
collection names, module names, and user stories from the context files. Generic
placeholder names (`fieldOne`, `module1`, etc.) are only acceptable in the shared
infrastructure sections (layouts, components, error handling) — never in modules.

**Alpine.js v3 requires `'unsafe-eval'` in CSP `script-src`.** Alpine.js v3 uses
`new Function()` for expression evaluation. The CspNonceFilter MUST include
`'unsafe-eval'` in the `script-src` directive alongside the nonce. Without this,
Alpine.js expressions will be silently blocked and all `x-data`, `x-show`, `@click`
etc. will fail. The CSP `font-src` must also include `data:` for bundled font data
URIs.

**Tailwind CSS v4 requires an explicit `@source` directive to scan JTE templates.**
In `app.css`, add `@source "../../../jte";` after the `@import "tailwindcss";` line.
Without this, Tailwind v4's automatic content detection only scans the frontend
`src/` directory and misses all utility classes used in JTE template files, resulting
in a minimal CSS output missing most styling.

**Do NOT generate UI elements without backend support.** Every interactive UI element
in the spec (buttons, dropdowns, toggles) must be backed by a requirement in PRD.md
or a functional backend endpoint. Specifically: do NOT include a locale/language
switcher unless PRD.md has i18n requirements. Do NOT include features "for future use"
— they will be non-functional and confuse users.

**HTMX navigation in sidebar and header fragments MUST use direct module URLs**, NOT
`/api/content/...` prefixed paths. The pattern is: `hx-get="${item.getHref()}"` with
`hx-select="#content-area"` and `hx-target="#content-area"` to extract the content area
from the full page response. Never use `/api/content/` as a URL prefix — this pattern
creates a dependency on a non-existent content API and causes navigation failures.

**Sidebar navigation must be filtered by the current user's roles.** The LayoutService
(or equivalent) that builds `NavItem` lists MUST check the authenticated user's roles
and only include menu items that the user is authorized to access. The role-to-menu
mapping is derived from the mockup sidebar files (each role folder defines which menu
items that role sees). Never show all menu items to all users — this exposes
unauthorized functionality in the UI.

### Conditional Constraints

**If Auth = Keycloak:**
- **Use OAuth2 Client (not Resource Server).** The application uses `oauth2Login()` with
  Authorization Code flow. After successful login, the `OidcUser` principal is stored in
  the HTTP session (Spring Security default session management).
- **Do NOT manage users and roles in the application.** The application does not create,
  update, or delete users or roles. Keycloak (the external identity provider) owns all
  user and role management. The application only reads roles from the OIDC ID token to
  enforce access control. There are no user registration endpoints, no role assignment
  endpoints, and no user entity/document in the database.

**If Auth = form:**
- The application manages its own user authentication. A `UserDetailsService` backed by
  the selected database provides user lookup. Passwords must be encoded with BCrypt.
  Session management uses Spring Security defaults (not stateless).

**If Scheduling = yes and Database = MongoDB:**
- **Use `io.fluidsonic.mirror` for Quartz MongoDB job store.** Instead of the default
  in-memory or JDBC job store, Quartz is configured to persist job metadata in MongoDB
  using the `fluidsonic-mirror-quartz` library.

**If Scheduling = yes and Database = PostgreSQL/MySQL:**
- Use the built-in Quartz JDBC job store with the selected relational database.

**If Messaging = yes:**
- Queue and exchange names for pub/sub messaging (`app.*`) must not collide with batch
  partition queues (`batch.partition.*`). Use distinct prefixes for each concern.

**If Reporting = yes:**
- Report definitions must call **module services** to produce DTOs, never repositories
  directly. This preserves Spring Modulith module boundaries.
- Report DTO field names must match `.jrxml` template `<field name="...">` declarations
  exactly (JavaBean getter convention).
- `.jrxml` templates must NOT contain `<queryString>` SQL — all data comes via
  `JRBeanCollectionDataSource` from DTOs. No direct database access in report templates.
- Report implementations must be `@Component` beans implementing `ReportDefinition` so
  that `ReportRegistry` auto-discovers them at startup.

**If Scheduling = yes and Remote Partitioning = yes:**
- Manager and worker nodes must share the **same `JobRepository` database**. The manager
  writes partition metadata (`StepExecution` entries) and workers update them on completion.
- **If Database = none and Remote Partitioning = yes**: This is an **invalid combination**.
  The spec must warn that remote partitioning requires a shared database. Either select a
  database (MongoDB, PostgreSQL, or MySQL) or disable remote partitioning.

## Principles Embedded in the Spec

These principles should be woven into every section of the spec rather than listed
separately.

### Always-Applicable Principles
- Server-rendered views with JTE templating and view-model-first design
- Tailwind CSS utility-first styling (no component framework)
- Alpine.js for client-side interactivity, htmx for partial page updates
- Vite build pipeline with manifest-based asset resolution
- Event-driven inter-module communication via Spring Modulith events
- Domain-driven package structure (one module per bounded context)
- Separation of concerns (controller -> service -> repository, view model -> template)
- Constructor injection everywhere (`@RequiredArgsConstructor` + `private final`)
- `@Getter @Setter` instead of `@Data` on all classes
- Single `application.yml` with environment variables for all environment-specific values (no profile-specific YAML files)
- Custom exception hierarchy with global error handling
- CSP nonce support for inline scripts
- Light/dark theme switching persisted in cookies
- Structured logging of important events and errors

### Conditional Principles
- **If Auth = Keycloak:** OAuth2 Client with Keycloak OIDC for all authentication and
  authorization (session-based). No user/role management — external IdP only.
- **If Auth = form:** Spring Security form-based login with database-backed user store.
  Session-based authentication.
- **If Auth = none:** No security filter chain. All endpoints are publicly accessible.
  Auth can be layered on later.
