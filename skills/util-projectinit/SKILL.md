---
name: util-projectinit
model: claude-sonnet-4-6
effort: medium
description: >
  Initialize a new CO2 project by analyzing a free-form prompt or markdown brief and generating
  the three foundational documents at the project root: `CLAUDE.md` (project detail, terminology,
  supporting 3rd party applications, custom applications, port allocation, system modules,
  business modules, rules), `DEVTOOL.md` (skeleton of development tools required by the inferred
  technology stack) and `ENVIRONMENT.md` (skeleton of local environment variables and credentials
  matching the 3rd party applications and external services declared in CLAUDE.md). Infers
  required infrastructure (databases, message queues, caches, SSO, file storage, SMTP, etc.),
  the number and names of custom applications, the adequacy of the standard system modules and
  the project-specific business modules from the input. This skill seeds the project — it is the
  first skill to run before any `modelgen-*`, `mockgen-*`, `specgen-*`, `testgen-*`,
  `util-projectsync` or `util-preparek8senv` invocation.
  Trigger on keywords: "init project", "initialize project", "start project", "bootstrap project",
  "scaffold project", "seed CLAUDE.md", "generate CLAUDE.md", "create CLAUDE.md",
  "project init", "new project from brief", "init from prompt", "init from markdown".
  Accepts an optional argument: a free-form prompt describing the project, OR a path to a
  markdown file containing the brief. If no argument is given, the skill asks the user for one.
---

# Util Project Init

Bootstrap a new CO2 project by transforming a free-form prompt or markdown brief into the three
foundational documents at the project root:

- `CLAUDE.md` — canonical project context: project detail, terminology, infrastructure,
  applications, port allocation, modules, rules.
- `DEVTOOL.md` — skeleton of the local development tools the inferred technology stack will need
  (JDK, Maven, Node, PNPM, database/queue CLIs, kubectl, etc.).
- `ENVIRONMENT.md` — skeleton of local environment variables and credentials for every
  supporting 3rd party application and external service declared in `CLAUDE.md`.

This skill is the **first** skill in the CO2 workflow. Once these three files exist, downstream
skills (`util-projectsync`, `util-preparek8senv`, `modelgen-*`, `mockgen-*`, `specgen-*`,
`testgen-*`, `conductor-*`) can take over.

## Input Resolution

This skill accepts a single optional argument:

```
/util-projectinit [prompt-or-markdown-path]
```

| Argument | Required | Example | Description |
|----------|----------|---------|-------------|
| `<input>` | No | `"A universal recruitment platform between countries"` or `./brief.md` | Free-form prompt OR path to a markdown file describing the project |

### Input Detection

1. If the argument is omitted or empty → ask the user: *"Provide a one-paragraph project brief, or the path to a markdown file describing the project."* Stop until the user responds.
2. If the argument **points to an existing file** with a `.md`, `.markdown` or `.txt` extension (relative to the project root or absolute) → read it as the brief.
3. Otherwise → treat the argument verbatim as the free-form prompt.
4. If the resolved brief is fewer than ~20 words and contains no recognizable project metadata (project name, domain, applications, modules), ask one clarifying question before proceeding.

### Pre-Flight Checks

Before doing any analysis, inspect the project root:

| File | Action |
|------|--------|
| `CLAUDE.md` exists | **Stop.** Print: *"CLAUDE.md already exists at the project root. This skill only initializes a new project. To evolve an existing CLAUDE.md, use `util-projectsync` (sync folders/modules) or edit it manually."* |
| `DEVTOOL.md` exists | Warn and skip generation of DEVTOOL.md. Do not overwrite. |
| `ENVIRONMENT.md` exists | Warn and skip generation of ENVIRONMENT.md. Do not overwrite. |

The skill is **non-destructive**: it never overwrites an existing root-level CLAUDE.md, DEVTOOL.md
or ENVIRONMENT.md. If all three files exist, stop after printing the existence summary.

## Workflow

### 1. Analyze the Brief

Parse the prompt or markdown file and extract / infer the following. Where the brief is
ambiguous, prefer a sensible default and emit a `[TODO]` annotation in the generated file rather
than blocking the workflow.

#### 1a. Project Metadata
- **Project Code** — a short uppercase acronym derived from the project name (e.g., `URP` for
  "Universal Recruitment Platform", `HMS` for "Hospital Management System"). 3–5 letters
  preferred. If the brief gives one, use it verbatim.
- **Project Name** — title-case full name of the project.
- **Project Description** — one to three sentences derived from the brief.
- **Goals** — three to five bullets summarising the outcome the project must achieve. Convert
  problem statements in the brief into outcome-oriented goal statements.
- **Terminology** — domain-specific terms (acronyms, role names, country/region pairs, etc.).
  Extract from the brief; include obvious additions (e.g., `IC: Industrial Classification`)
  only when the brief mentions the concept.

#### 1b. Infrastructure Inference

From the brief, infer the **3rd party supporting applications** the project needs. Apply this
heuristic table (these are starting points — adjust to what the brief actually says):

| Capability in brief | Default 3rd party application |
|---------------------|------------------------------|
| Email / notifications | **SMTP Server** — Mailcatcher (dev) |
| Document upload, files, attachments | **File Storage** — MinIO |
| Caching, session, rate-limit | **Cache** — Redis |
| Unstructured / document data, JSON, profiles | **Document Database** — MongoDB |
| Structured / relational data, transactions | **Relational Database** — MySQL or PostgreSQL |
| User login, SSO, OIDC, roles | **Single Sign On** — Keycloak |
| Inter-service async messaging, events | **Message Queue** — RabbitMQ |
| Full-text search | **Search** — Meilisearch |
| API gateway, rate-limit at edge | **API Gateway** — Kong |
| Background jobs, scheduling | (handled by custom apps; no 3rd party unless the brief says so) |

Pick the **minimum** set that satisfies the brief. Do not invent infrastructure the brief did
not imply. For every 3rd party application picked:

- Use the exact same heading layout as the reference `CLAUDE.md` (`## <Name>`, technology &
  version bullet, optional `Databases:` / `Buckets:` / `Vhosts:` bullets, optional
  `Depends on:` bullet).
- Group databases by tier (e.g., `Hub Core Database`, `Hub Support Database`,
  `HC Database`, `SC Database`) when the brief implies multiple bounded contexts.
- For every database, message queue or cache, list the **database names / bucket names /
  vhosts** as sub-bullets so they survive into ENVIRONMENT.md.

#### 1c. External Services Inference

External services are SaaS or remote APIs the project consumes but does **not** deploy itself:
- Google Maps / Mapbox → location service
- Firebase Cloud Messaging → push notifications
- Google Cloud / OpenAI / Anthropic → AI service
- Stripe / PayPal → payments
- WAF / CDN providers

If the brief mentions any of these, declare them under a `## External Services` sub-section
**inside** `# Supporting 3rd Party Applications` (see reference layout — `## External Services`
sits as a sibling sub-heading under the 3rd party section). Each service becomes a
`## <Service Name>` heading immediately under `## External Services`.

If none are implied, omit `## External Services` (do not insert a stub).

#### 1d. Custom Application Inference

From the brief, infer the custom applications the project will deliver. Heuristic:

- **One backend "middleware" / "core" service** is almost always present. Name it after the
  primary domain (e.g., `Hub Middleware`, `Order Service`, `Booking Core`).
- **One support / admin portal** is common for internal users.
- **Adapter / connector services** appear when the brief describes integration between two
  parties (e.g., Hiring Country ↔ Source Country, B2B partners). Each side gets its own
  adapter, named with a side prefix (`HC Adapter`, `SC Adapter`).
- **SDK libraries** appear when the brief mentions third-party developers integrating with the
  platform. Pair each SDK with the adapter it wraps (`HC Adapter SDK`, `SC Adapter SDK`).
- **Web / mobile frontend** apps appear when the brief mentions specific user-facing channels.

For each custom application, write:
- `## <Application Name>` heading.
- One to three bullet description of what it does.
- A `Depends on:` bulleted list referencing the 3rd party applications, external services or
  other custom applications it relies on. Match the dependency names **exactly** to the
  `## <Name>` headings used elsewhere in CLAUDE.md so `util-projectsync` validation passes.

#### 1e. Port Allocation

Assign one local port per **server-style custom application** (backends, portals, adapters,
gateways — not SDK libraries). Group by tier when the brief describes tiers (Public / Hub /
HC / SC, or Frontend / Backend, etc.):

| Tier | Port Range |
|------|-----------|
| Public Layer | 8080 |
| Tier 1 / Hub Layer | 9001–9019 |
| Tier 2 / HC Layer | 9020–9039 |
| Tier 3 / SC Layer | 9040–9059 |

If the project has no tier split, allocate sequential ports from `8080` upward.

Emit the allocation as the same markdown table format used in the reference CLAUDE.md, with a
`Sub Domain` column. The sub-domain follows the pattern
`<type-abbr>-<tier-abbr>-<project-code-lower>` (e.g., `mw-hub-urp`, `sup-hub-urp`,
`ad-hc-urp`, `ad-sc-urp`). If no clear pattern fits, use the application snake_case name as
the sub-domain.

#### 1f. Module Inference

CO2 distinguishes two module tiers — **System Modules** (cross-cutting, framework-level) and
**Business Modules** (domain-specific).

**System Modules — default seven.** Always include all seven unless the brief makes one
clearly inapplicable. Use these names verbatim so downstream skills recognise them:

| Module | Always include unless… |
|--------|------------------------|
| Authentication and Authorization | Brief says no users (rare). |
| User | Brief says no users (rare). |
| Notification | Brief explicitly says no notifications. |
| Activities | Brief says no audit need (rare). |
| Audit Trail | Brief says no audit need (rare). |
| Document Management | Brief says no documents/files. |
| Batch Job | Brief says no scheduled jobs. |

**Business Modules — derive from the brief.** Extract every domain concept that warrants its
own bounded context: entities, workflows, taxonomies, sync operations. Common patterns:
- A `Dashboard` module if any user-facing portal is mentioned.
- A `Location Information` module if geography (countries, cities, postcodes) appears.
- One module per major entity (`Employer`, `Candidate Profile`, `Job Demand`, …).
- One module per cross-system synchronisation flow (`Employer Sync`, `Recruitment Agent Sync`).
- A `News and Announcement` module if the brief mentions broadcast content.
- A `Support Ticket` module if the brief mentions support / help desk.

Each module becomes a `### <Module Name>` heading (three hashes) under either `## System Module`
or `## Business Module` (which themselves sit under `# Modules`). The body is a one-line
description of what the module covers — short enough that `util-projectsync` later expands it
into `PRD.md` skeletons.

#### 1g. Technology Stack Inference (for DEVTOOL.md)

From the inferred applications, decide which tools DEVTOOL.md must list. Apply this mapping
(starting points — extend if the brief calls out a specific stack):

| Custom app technology | DEVTOOL.md entries |
|-----------------------|--------------------|
| Spring Boot / Java backend | JDK (Azul Zulu 21), Maven 3.9, Docker Hub |
| Laravel / PHP backend | PHP 8.3, Composer 2.x, Docker Hub |
| Node.js / TypeScript backend or CLI | NVM, Node.js 22 LTS, PNPM 10 |
| React / Vue / Angular SPA | NVM, Node.js 22 LTS, PNPM 10 |
| Flutter mobile / desktop | FVM, Flutter (resolved by FVM) |
| Python service | Python 3.x |

For every 3rd party application picked in step 1b, add the matching CLI:

| 3rd party app | DEVTOOL.md CLI entry |
|---------------|----------------------|
| MySQL / MariaDB | MySQL Shell |
| MongoDB | Mongo Shell (`mongosh`) |
| PostgreSQL | psql |
| Redis | Redis CLI (`rdcli`) |
| RabbitMQ | RabbitMQ Admin CLI (`rabbitmqadmin`) |
| MinIO / S3 | S3 Compatible CLI (`mc`) |
| Keycloak | Keycloak CLI (`kc`), Keycloak Admin CLI (`kcadm`), Keycloak Registration CLI (`kcreg`) |
| Firebase Cloud Messaging | Firebase CLI |

If a Kubernetes target is mentioned anywhere in the brief, include `Kubectl`.

For tool versions and paths, use placeholder values (see DEVTOOL.md template) — the developer
fills them in for their machine. Do **not** copy paths from the reference file; those are
Windows-specific to one developer.

### 2. Generate CLAUDE.md

Write `CLAUDE.md` at the project root using the **CLAUDE.md Template** (below), substituting
the analysis from step 1. Order of top-level sections is fixed:

1. `# Context`
2. `# Project Detail`
3. `# Goals`
4. `# Terminology`
5. `# Development Tool` (link to DEVTOOL.md)
6. `# Environment` (link to ENVIRONMENT.md; no per-environment sub-sections in this skill —
   environments are added later by `util-preparek8senv`)
7. `# Supporting 3rd Party Applications` (with `## External Services` sub-section if any)
8. `# Custom Applications`
9. `# Port Allocation`
10. `# Modules` (`## System Module` then `## Business Module`)
11. `# Application Folder structure`
12. `# Rules`

For empty sections (e.g., no external services, no business modules) use the
`**No <category>** for this project since …` sentence pattern shown in the project root
CLAUDE.md, **not** an empty section.

### 3. Generate DEVTOOL.md

If DEVTOOL.md already exists at the project root, skip this step and log a warning.

Otherwise write DEVTOOL.md using the **DEVTOOL.md Template** (below). Include only the toolchain
entries derived from step 1g. Every version, path and credential field is a placeholder
(`<version>`, `<path>`, `<to be filled by developer>`) — DEVTOOL.md is per-developer and
machine-specific.

### 4. Generate ENVIRONMENT.md

If ENVIRONMENT.md already exists at the project root, skip this step and log a warning.

Otherwise write ENVIRONMENT.md using the **ENVIRONMENT.md Template** (below). The structure
mirrors CLAUDE.md's 3rd party application and external services list:

- `# External Services` — one `## <Service>` sub-section per external service from step 1c,
  with the credential keys appropriate to the service (`API Key`, `Project ID`, etc.) all set
  to `<to be filled by developer>`.
- `# Environment` — one `## <Application Name>` sub-section per 3rd party application from
  step 1b. Use the **Technology-Specific Config Fields** table from `util-preparek8senv`
  (reproduced below in the ENVIRONMENT.md template) to choose which keys to include. All
  values default to placeholders or technology defaults (e.g., RabbitMQ user/password
  `guest`/`guest`, no auth for Redis/MongoDB unless the brief implies otherwise).

`util-projectinit` writes only the **localhost / single-developer** skeleton. Multi-environment
sections (e.g., Home Server, Kubernetes targets) are added later by `util-preparek8senv` when
the developer chooses a deployment target. Do **not** wrap services in a `# <Environment Name>`
heading here — the top-level `# Environment` heading is sufficient for the skeleton.

### 5. Output Summary

Print a single summary block:

```
## Project Init Summary

### Project Detail
| Field | Value |
|-------|-------|
| Project Code | URP |
| Project Name | Universal Recruitment Platform |
| Description | <one-line> |

### Inferred Infrastructure
| Category | Count | Names |
|----------|-------|-------|
| 3rd Party Applications | 8 | SMTP Server, Hub File Storage, Hub Cache, Hub Core Database, Hub Support Database, Hub Single Sign On, HC Adapter Message Queue, SC Adapter Message Queue |
| External Services | 0 | — |
| Custom Applications | 6 | Hub Middleware, Hub Support Portal, HC Adapter, HC Adapter SDK, SC Adapter, SC Adapter SDK |
| System Modules | 7 | Authentication and Authorization, User, Notification, Activities, Audit Trail, Document Management, Batch Job |
| Business Modules | 14 | Dashboard, Location Information, Corridor, Recruitment Step, Employer, Recruitment Agent, … |

### Port Allocation
| Port | Application | Sub Domain |
|------|-------------|------------|
| 9001 | Hub Middleware | mw-hub-urp |
| 9002 | Hub Support Portal | sup-hub-urp |
| 9020 | HC Adapter | ad-hc-urp |
| 9040 | SC Adapter | ad-sc-urp |

### Files Generated
| File | Status | Notes |
|------|--------|-------|
| CLAUDE.md | Created | — |
| DEVTOOL.md | Created | JDK, Maven, Node, PNPM, MySQL Shell, mongosh, rdcli, rabbitmqadmin, mc, Keycloak CLIs, Kubectl |
| ENVIRONMENT.md | Created | 8 services, 0 external services |

### Next Steps
1. Fill in DEVTOOL.md version and path fields for your machine.
2. Fill in ENVIRONMENT.md credentials and any `<to be filled by developer>` values.
3. Review CLAUDE.md and adjust any `[TODO]` annotations.
4. Run `/util-projectsync` to scaffold per-application folders, PRD.md and BUG.md files.
5. (Optional) Run `/util-preparek8senv <env>` to generate K8s manifests for 3rd party services.
```

If any of the three files were skipped because they already existed, mark their row in the
`Files Generated` table as `Skipped (already exists)` and omit the corresponding "Next Steps"
sub-step.

## CLAUDE.md Template

When generating CLAUDE.md, use this structure verbatim, substituting `{{...}}` placeholders.

```markdown
# Context
- This document is for coding agent to understand the project as a whole, including the project detail, goals and terminology.
- This document will be used as reference in any of the tasks related to this project.
- This document also contain the development environment deployment strategy ONLY. Local and production deployment strategy will be handled separately and not included in this document.

# Project Detail
- Project Code: {{PROJECT_CODE}}
- Project Name: {{PROJECT_NAME}}
- Project Description: {{PROJECT_DESCRIPTION}}

# Goals
{{#each GOALS}}
- {{this}}
{{/each}}

# Terminology
{{#each TERMS}}
- {{name}}: {{definition}}
{{/each}}

# Development Tool
- Refer to [DEVTOOL.md](DEVTOOL.md) for all the development tools to be used by AI coding agent to build and execute applications developed in this project.
- **Load all the information in this section into the context window of the AI coding agent, as it will be used as reference for building and executing the applications in this project.**

# Environment
- Refer to [ENVIRONMENT.md](ENVIRONMENT.md) for the local environment variables and credentials used for this project.
- **Load all the information in this section into the context window of the AI coding agent, as it will be used as reference for building and executing the applications in this project.**

# Supporting 3rd Party Applications
- Below are the list of 3rd party applications which will be used as part of this project:

{{#each THIRD_PARTY_APPS}}
## {{name}}
- {{technology}} version {{version}}.
{{#if databases}}
- Databases:
{{#each databases}}
  - {{this}}
{{/each}}
{{/if}}
{{#if buckets}}
- Buckets:
{{#each buckets}}
  - {{this}}
{{/each}}
{{/if}}
{{#if dependsOn}}
- Depends on:
{{#each dependsOn}}
  - {{this}}
{{/each}}
{{/if}}

{{/each}}

{{#if EXTERNAL_SERVICES}}
## External Services

{{#each EXTERNAL_SERVICES}}
## {{name}}
- {{description}}

{{/each}}
{{/if}}

# Custom Applications
- Below are the list of applications which will be custom developed as part of this project.:

{{#each CUSTOM_APPS}}
## {{name}}
- {{description}}
{{#if dependsOn}}
- Depends on:
{{#each dependsOn}}
  - {{this}}
{{/each}}
{{/if}}

{{/each}}

# Port Allocation
- Each application runs on a unique port to avoid conflicts during local development.
- Port ranges are grouped by layer:
{{#each TIERS}}
  - {{name}}: {{range}}
{{/each}}

| Port | Application            | Sub Domain   |
|------|------------------------|--------------|
{{#each PORT_ROWS}}
| {{port}} | {{application}} | {{subDomain}} |
{{/each}}

# Modules

## System Module

{{#each SYSTEM_MODULES}}
### {{name}}
- {{description}}

{{/each}}

## Business Module

{{#each BUSINESS_MODULES}}
### {{name}}
- {{description}}

{{/each}}

# Application Folder structure
- Each application will have its own folder in the project repository, with the folder name as the application name in snake case format (e.g., `hub_middleware`, `hc_adapter`).
- The application folder structure is as follows:
  - context: The context folder for the application.
    - model: The module data model for the application. Output by the "modelgen-*" skill.
    - mockup: The HTML mockups for the UI design. Output by the "mockgen-*" skill.
    - specification: The detailed specification for the application. Output by the "specgen-*" skill.
    - test: The test specification for the application. Output by the "testgen-*" skill.
    - develop: The planning and tracking output during the development phase. Output by the "conductor-feature".
    - bug: The planning and tracking output during the bug fixing phase. Output by the "conductor-defect".
    - reference: The reference materials for the application.
    - PRD.md: The user stories, NFRs and Constraints (by modules) input by user. Used by all the skills above as reference.
    - BUG.md: The bug reported by user. Used by "conductor-defect" skill as reference.
  - source: The source code for the application.
    - Dockerfile: The Dockerfile for the application.
    - DEPLOYMENT.md: The deployment strategy for the application.
    - README.md: The readme file for the application.

# Rules

## Sudo Privilege
- For tasks that require elevated privileges, refer to the `# Sudo Privilege` section in the user-level `~/CLAUDE.md` (`/home/<user>/CLAUDE.md`) for the sudo username and password if it exists.
- The user-level `CLAUDE.md` is per-developer and is NOT committed to version control. NEVER copy sudo credentials into this project-level `CLAUDE.md` or any other file under the project repository.
- If the user-level `CLAUDE.md` does not contain a `# Sudo Privilege` section, STOP and ask the user to provide the credentials before proceeding.

## Environment Related Task/Command
- If the task/command is related to environment, infer which environment it's targeted to based on the information in `Environment` section above:
  - If not specified, STOP and ask user to clarify which environment it's targeted to before proceeding.

## 3rd Party Platform or Services Username and Password Standard format
- When creating new user accounts for new database, message queue, API gateway, or any other services that require authentication and authorization, use the following standard format for username and password:
    - Username:
      - If user id is required, use `bestrnd`
      - If email is required, use `bestrnd@gmail.com`
    - Password: `B3st1n3t@2025`

## Docker Image Build & Deployment
- ALWAYS rebuild and overwrite docker images during a deploy task, even when a tag with the same version already exists in the local docker daemon or in a private registry. Developers in this project frequently update source code without bumping the version, so skipping a rebuild deploys stale code.
- After `docker push`, capture the registry digest from the push output (`<tag>: digest: sha256:...`) and confirm running pods reference it.
- All Kubernetes deployment manifests (`*/k8s/deployment.yaml`) MUST set `imagePullPolicy: Always` on application containers. With `IfNotPresent`, k8s reuses node-cached layers when the tag is unchanged and silently runs old code.

## Playwright Screenshots
- When taking screenshots using Playwright MCP `browser_take_screenshot`, ALWAYS save to the `.playwright-mcp/` folder by prefixing the filename (e.g., `.playwright-mcp/my-screenshot.png`).
- NEVER use bare relative filenames — they will pollute the repository root.
```

**Rules for empty sections:**

- If there are no 3rd party applications, replace the body of `# Supporting 3rd Party Applications` with `**No Supporting 3rd Party Applications for this project**` (omit the bullet header).
- If there are no external services, omit the `## External Services` heading entirely.
- If there are no custom applications, replace the body of `# Custom Applications` with `**No Custom Applications for this project**` and omit the `# Port Allocation` section.
- If there are no system or business modules, replace the body of the relevant sub-section with `**There is no {{System|Business}} Module for this project**`.

## DEVTOOL.md Template

When generating DEVTOOL.md, use this structure. Include **only** the entries derived from the
inferred technology stack (step 1g) — drop the rest. Every version and path is a placeholder.

```markdown
# Context
- This document contain the development tool to be used by AI coding agent to build and execute applications developed in this project.
- **This file must NOT be committed to version control.**

{{#if NEEDS_JDK}}
# JDK
- Version: <jdk-version>
- Path: <to be filled by developer>
- JAVA_HOME: <to be filled by developer>
{{/if}}

{{#if NEEDS_MAVEN}}
# Maven
- Version: <maven-version>
- Path: <to be filled by developer>
- Maven Local Repository: <to be filled by developer>
{{/if}}

{{#if NEEDS_NVM}}
# Node Version Manager (NVM)
- Version: <nvm-version>
- Path: <to be filled by developer>
{{/if}}

{{#if NEEDS_NODE}}
# Node.js
- Version: <node-version>
- Path: <to be filled by developer>
- NODE_HOME: <to be filled by developer>
{{/if}}

{{#if NEEDS_PNPM}}
# PNPM
- Version: <pnpm-version>
- Path: <to be filled by developer>
{{/if}}

{{#if NEEDS_FVM}}
# Flutter Version Manager (FVM)
- Version: <fvm-version>
- Path: <to be filled by developer>
{{/if}}

{{#if NEEDS_PYTHON}}
# Python
- Version: <python-version>
- Path: <to be filled by developer>
{{/if}}

{{#if NEEDS_FIREBASE_CLI}}
# Firebase CLI
- Version: <firebase-cli-version>
- Path: <to be filled by developer>
{{/if}}

{{#if NEEDS_MYSQL_SHELL}}
# MySQL Shell
- Version: <mysql-shell-version>
- Path: <to be filled by developer>
{{/if}}

{{#if NEEDS_MONGOSH}}
# Mongo Shell
- Version: <mongosh-version>
- Path: <to be filled by developer>
{{/if}}

{{#if NEEDS_PSQL}}
# PostgreSQL Client (psql)
- Version: <psql-version>
- Path: <to be filled by developer>
{{/if}}

{{#if NEEDS_RABBITMQ_ADMIN}}
# RabbitMQ Admin CLI
- Version: <rabbitmqadmin-version>
- Path: <to be filled by developer>
{{/if}}

{{#if NEEDS_REDIS_CLI}}
# Redis CLI
- Version: <redis-cli-version>
- Path: <to be filled by developer>
- Reference: `https://github.com/lujiajing1126/redis-cli`
{{/if}}

{{#if NEEDS_MC}}
# S3 Compatible CLI
- Version: <mc-version>
- Path: <to be filled by developer>
- Reference: `https://github.com/minio/mc`
{{/if}}

{{#if NEEDS_KEYCLOAK_CLI}}
# Keycloak CLI
- Path: <to be filled by developer>

# Keycloak Admin CLI
- Path: <to be filled by developer>

# Keycloak Registration CLI
- Path: <to be filled by developer>
{{/if}}

{{#if NEEDS_DOCKER_HUB}}
# Docker Hub
- Username: <to be filled by developer>
- Password: <to be filled by developer>
{{/if}}

{{#if NEEDS_KUBECTL}}
# Kubectl
- Version: <kubectl-version>
{{/if}}
```

## ENVIRONMENT.md Template

When generating ENVIRONMENT.md, use this structure. The `# External Services` section is
omitted entirely when there are no external services. The `# Environment` section lists one
`## <Application Name>` sub-section per 3rd party application, using the
**Technology-Specific Config Fields** table below.

```markdown
# Context
- This document contain the local environment variables and credentials used for this project.
- The variables is intended for AI coding agent to use to build and execute applications developed in this project.
- This document is per developer, and should not be shared with anyone else.
- **This file must NOT be committed to version control.**

{{#if EXTERNAL_SERVICES}}
# External Services

{{#each EXTERNAL_SERVICES}}
## {{name}}
{{#each credentialKeys}}
- {{this}}: `<to be filled by developer>`
{{/each}}

{{/each}}
{{/if}}

# Environment

{{#each THIRD_PARTY_APPS}}
## {{name}}
- {{technology}} version {{version}}.
{{#each configKeys}}
- {{key}}: {{value}}
{{/each}}
{{#if databases}}
- Databases:
{{#each databases}}
  - {{this}}
{{/each}}
{{/if}}

{{/each}}
```

### Technology-Specific Config Fields

| Technology | Config Fields (in order) | Default Values |
|-----------|--------------------------|----------------|
| MongoDB | Host, Port, Username, Password | `localhost`, `27017`, (none), (none) |
| MySQL | Host, Port, Username, Password | `localhost`, `3306`, `root`, `B3st1n3t@2025` |
| PostgreSQL | Host, Port, Username, Password | `localhost`, `5432`, `postgres`, `B3st1n3t@2025` |
| Redis | Host, Port, Password, Database Index | `localhost`, `6379`, (none), `0` |
| RabbitMQ | Host, AMQP Port, Admin Port, Username, Password, Vhost | `localhost`, `5672`, `15672`, `guest`, `guest`, `/` |
| Keycloak | Host, HTTP Port, Admin Username, Admin Password, Realm | `localhost`, `8180`, `bestrnd@gmail.com`, `B3st1n3t@2025`, `<project-code-lower>` |
| Meilisearch | Host, Port, Master Key | `localhost`, `7700`, `<to be filled by developer>` |
| Kong | Proxy Host, Proxy Port, Admin Host, Admin Port | `localhost`, `8000`, `localhost`, `8001` |
| Mailcatcher | SMTP Host, SMTP Port, HTTP Port | `localhost`, `1025`, `1080` |
| MinIO / S3 | Host, API Port, Web Port, Username, Password, Bucket | `localhost`, `9000`, `9001`, `minioadmin`, `minioadmin`, `<project-code-lower>` |

### External Service Credential Keys

| Service | Credential keys |
|---------|-----------------|
| Google Maps / Mapbox | API Key |
| Firebase Cloud Messaging | (note: `Credential already configured with Firebase CLI`) |
| Google Cloud / AI | API Key |
| OpenAI / Anthropic | API Key |
| Stripe / PayPal | Publishable Key, Secret Key, Webhook Secret |

For each external service, list the keys with value `<to be filled by developer>`.

## Important Rules

- **Non-destructive.** Never overwrite an existing root-level `CLAUDE.md`, `DEVTOOL.md`, or
  `ENVIRONMENT.md`. If a file exists, skip it and log a warning. If `CLAUDE.md` exists, stop
  the entire skill — the project is already initialised.
- **Single-shot.** This skill seeds the project; it does not maintain it. To evolve the
  project, use `util-projectsync` (folders, modules, PRD/BUG sync) or
  `util-preparek8senv` (multi-environment expansion of ENVIRONMENT.md and K8s manifests).
- **Names must match across files.** Every `## <Application Name>` heading in CLAUDE.md must
  appear with the **same name** in ENVIRONMENT.md and (where applicable) in dependency lists.
  This is what `util-projectsync` validation relies on.
- **Module heading levels.** System and Business modules in CLAUDE.md sit at `### <Module>`
  (three hashes), because they are nested under `# Modules` → `## System Module` /
  `## Business Module`. Do not flatten them — `util-projectsync` matches on this level.
- **Project Code is uppercase, namespace is lowercase.** Use the uppercase code in
  `Project Code:`, but use the lowercase form for sub-domains, K8s namespaces, default bucket
  names and Keycloak realm names.
- **No credential leaks.** DEVTOOL.md and ENVIRONMENT.md must never contain real secrets.
  Always use `<to be filled by developer>` (or sensible technology defaults like
  `guest`/`guest` for RabbitMQ) — never invent credentials.
- **TODO over guess.** When the brief is silent on a detail (port, technology version,
  dependency direction), prefer a sensible default plus a `[TODO]` line above the relevant
  heading, rather than silently making up a fact.
- **Do not create per-application folders.** Folder scaffolding (`hub_middleware/`,
  `hc_adapter/`, etc.) is the job of `util-projectsync`. This skill only writes the three
  root files.
- **Do not create PRD.md or BUG.md.** Those are scaffolded by `util-projectsync` from
  CLAUDE.md.
- **Do not create K8s manifests.** Those are generated by `util-preparek8senv`.
- **Keep the brief.** If the brief came from a markdown file, leave the source file
  untouched. Do not move or rename it.
- **Idempotent re-run.** Re-running the skill after CLAUDE.md exists is a no-op apart from
  the "already initialised" message — never partially overwrite.
