---
name: conductor-feature-develop
description: >
  Application development orchestrator — orchestrates full-stack code implementation
  module-by-module (code + Playwright E2E tests), tracking progress in IMPLEMENTATION_MASTER.md
  and per-module IMPLEMENTATION_MODULE.md. Takes an application name (mandatory), with optional
  source code path, version and module filters. Requires context artifacts (module models, HTML
  mockups, technical specifications, test specifications) to already exist — use
  "conductor-feature-prepare" first if they don't. Use this skill when the user asks to
  "implement the application", "start development", "build the app module by module",
  "orchestrate implementation", "develop from specs", "implement from test specs", or any
  request to systematically develop a full application from existing specs. Also trigger when
  user says "resume implementation" to continue from where a previous session left off using
  IMPLEMENTATION_MASTER.md and IMPLEMENTATION_MODULE.md progress files.
---

# Feature Conductor — Develop

Application development orchestrator — implements full-stack code module-by-module, driven by
test specs and tracked via implementation checklists. Requires context artifacts to already
exist (use `conductor-feature-prepare` first).

## Ralph Loop Integration (AUTO-START — MANDATORY)

This skill AUTOMATICALLY starts a Ralph Loop to ensure complete implementation across all modules.
Without Ralph Loop, the implementation may stop prematurely due to context window limits, API
usage limits, or the agent incorrectly concluding work is "done enough". Ralph Loop ensures
the same prompt is re-fed after each session exit, and the agent picks up where it left off
using the IMPLEMENTATION_MASTER.md and IMPLEMENTATION_MODULE.md tracking files.

### FIRST ACTION: Start Ralph Loop

**BEFORE doing anything else** (before Phase 0, before reading any files), you MUST invoke the
Ralph Loop skill using the Skill tool. This is a blocking requirement — do NOT proceed with
any implementation work until Ralph Loop is active.

**Invoke this immediately:**
```
Skill(skill: "ralph-loop:ralph-loop", args: "/conductor-feature-develop <application> [source:<source-code-path>] [version:<version>] [module:<module>] --completion-promise \"ALL MODULES IMPLEMENTED\" --max-iterations 100")
```

Replace `<application>` and optional arguments with the actual arguments provided by the user.

**Example:** If the user invokes:
```
/conductor-feature-develop mainapp
```

Then invoke:
```
Skill(skill: "ralph-loop:ralph-loop", args: "/conductor-feature-develop mainapp --completion-promise \"ALL MODULES IMPLEMENTED\" --max-iterations 100")
```

If the user provides optional arguments:
```
/conductor-feature-develop mainapp version:v2 module:user
```

Then invoke:
```
Skill(skill: "ralph-loop:ralph-loop", args: "/conductor-feature-develop mainapp version:v2 module:user --completion-promise \"ALL MODULES IMPLEMENTED\" --max-iterations 100")
```

After Ralph Loop is active, proceed with Phase 0 (Resume Check) and continue normally.

### How It Works

1. This skill auto-starts a Ralph Loop with the orchestrator prompt as the loop body
2. On each iteration, the agent reads IMPLEMENTATION_MASTER.md to find the next pending module
3. The agent implements one or more modules until context runs out or a module completes
4. When the agent tries to exit, the Ralph Loop stop hook re-feeds the same prompt
5. The next iteration resumes from where the last one left off (tracked in IMPLEMENTATION_MASTER.md)
6. When ALL modules are COMPLETED, the agent outputs the completion promise to exit the loop

### Completion Promise

When ALL modules in IMPLEMENTATION_MASTER.md have status `COMPLETED`, output the following
promise tag to signal the Ralph Loop that implementation is finished:

```
<promise>ALL MODULES IMPLEMENTED</promise>
```

**CRITICAL**: Only output this promise when EVERY module in the execution order has:
- Status = COMPLETED in IMPLEMENTATION_MASTER.md
- All E2E tests passing
- IMPLEMENTATION_MODULE.md fully updated

Do NOT output the promise prematurely. Do NOT output it to escape the loop. The Ralph Loop
will verify this tag and only exit when it is present.

### Iteration Awareness

At the START of every iteration (including the first), the agent MUST:
1. Check if Ralph Loop is already active (if `.claude/ralph-loop.local.md` exists, skip re-invoking)
2. Read IMPLEMENTATION_MASTER.md to determine what is already completed
3. Find the FIRST module with status != COMPLETED
4. If that module has an IMPLEMENTATION_MODULE.md, read it to find the last incomplete step
5. Resume from exactly that point — do NOT re-implement completed work
6. If ALL modules are COMPLETED, output the completion promise and stop

### Never Stop Prematurely

Within a Ralph Loop iteration, the agent MUST:
- Continue implementing modules sequentially until context limits force a stop
- After completing one module, IMMEDIATELY start the next pending module
- Do NOT stop after a single module "to let the user review" — Ralph Loop handles iteration
- Do NOT output the completion promise until ALL modules are verified complete
- If approaching context limits mid-module, save progress to IMPLEMENTATION_MODULE.md so the
  next iteration can resume from the exact step

## Inputs

The skill expects these arguments:

```
/conductor-feature-develop <application> [source:<source-code-path>] [version:<version>] [module:<module>]
```

| Argument | Required | Example | Description |
|----------|----------|---------|-------------|
| `<application>` | Yes | `mainapp` | Application name to locate the context folder |
| `source:<path>` | No | `source:mainapp` | Path where source code resides. Defaults to `<app_folder>` (same as the resolved application folder) |
| `version:<version>` | No | `version:v2` | If provided, filter user stories and artifacts for this version only. If omitted, process all versions |
| `module:<module>` | No | `module:user` | If provided, process only this module. If omitted, process all modules |

### Input Resolution

The application name is matched against root-level application folders:
1. Strip any leading `<number>_` prefix from folder names (e.g., `1_hub_middleware` → `hub_middleware`)
2. Match case-insensitively against the provided application name
3. Accept snake_case, kebab-case, or title-case input
4. If no match found, list available applications and stop

### Auto-Resolved Paths

| File | Resolved Path |
|------|---------------|
| PRD.md | `<app_folder>/context/PRD.md` |
| Module Models | `<app_folder>/context/model/` |
| HTML Mockups | `<app_folder>/context/mockup/` |
| Specifications | `<app_folder>/context/specification/` |
| Test Specs | `<app_folder>/context/test/` |
| References | `<app_folder>/context/reference/` |
| Development Output | `<app_folder>/context/develop/` |

### Application Folder Structure (Expected)

Source code and context artifacts coexist in the same `<app_folder>`. The `context/` subfolder
holds all generated artifacts (models, mockups, specs, tests, tracking). All other files and
folders at the root of `<app_folder>` are **source code** (e.g., `app/`, `Modules/`, `resources/`,
`composer.json`, `pom.xml`, `src/`, etc.).

**CRITICAL**: When scaffolding a new project (e.g., `composer create-project`, `mvn archetype:generate`),
the source code MUST be placed directly in `<source-code-path>/` — NOT in a nested subdirectory.
For example, with `composer create-project laravel/laravel`, you must either:
- Create in a temp directory and move all files (including dotfiles) up to `<source-code-path>/`, OR
- Use a technique that installs directly into the existing directory

The `context/` folder already exists in `<app_folder>` and must NOT be overwritten or deleted.

```
<app_folder>/                      # = <source-code-path> (by default)
  context/                         # Context artifacts (NOT source code)
    PRD.md
    model/
      MODEL.md
      <module-slug>/
        model.md
        schemas.json
        document-model.mermaid
    mockup/
      MOCKUP.html
      server.js
      <role>/content/
    specification/
      SPECIFICATION.md
      <module-slug>/SPEC.md
    test/
      TEST_PLAN.md
      <module-slug>/TEST_SPEC.md
    reference/
    develop/                       # Implementation tracking files
  (source code files)              # All other files are source code
    app/                           # Laravel: app directory
    Modules/                       # Laravel: nwidart modules
    resources/                     # Laravel: views, CSS, JS
    routes/                        # Laravel: route files
    config/                        # Laravel: config files
    composer.json                  # Laravel: PHP dependencies
    package.json                   # Laravel: JS dependencies
    ...                            # (or src/, pom.xml for Spring Boot, etc.)
```

## Pre-Requisite: Project Information from CLAUDE.md (MANDATORY)

**CLAUDE.md is automatically loaded into context** at the start of every session. It contains
project details, infrastructure paths, credentials, and configuration. You do NOT need to read
it manually — the information is already available in your context.

**Before executing ANY tool command** (Maven build, Spring Boot run, database CLI, Keycloak CLI,
Playwright test, npm start, etc.), use the following from CLAUDE.md (already in context):

- **JDK path** — Use the exact `JAVA_HOME` path specified in CLAUDE.md
- **Maven path** — Use the exact Maven binary path specified in CLAUDE.md
- **Database credentials** — Host, port, username, password for MongoDB, MySQL, etc.
- **Message queue credentials** — RabbitMQ host, port, username, password
- **Keycloak configuration** — Host, admin credentials, CLI path
- **Mailcatcher configuration** — SMTP host/port, web UI URL
- **Any other infrastructure details** — Ports, URLs, connection strings

**WHY**: CLAUDE.md contains the actual system paths, credentials, and configuration for the
developer's machine. Hardcoding or guessing these values will cause commands to fail. Every shell
command that involves JDK, Maven, database access, or any external service MUST use the values
from CLAUDE.md.

## Pre-Requisite: Context Artifacts Must Exist

Before starting implementation, verify that all required context artifacts exist:
- `<app_folder>/context/model/` — must contain module model files
- `<app_folder>/context/mockup/` — must contain HTML mockup files
- `<app_folder>/context/specification/` — must contain specification files
- `<app_folder>/context/test/` — must contain test specification files

If any artifacts are missing, **stop and inform the user** to run `/conductor-feature-prepare`
first. Do NOT attempt to generate artifacts — that is the responsibility of the prepare skill.

## Workflow

### Phase 0: Resume Check (Runs Every Ralph Loop Iteration)

This phase runs at the START of every iteration, including the first. In a Ralph Loop,
each iteration begins fresh with the same prompt, so the agent MUST read the tracking
files to understand what has already been completed.

0. **Auto-Start Ralph Loop** — Check if `.claude/ralph-loop.local.md` exists. If it does NOT
   exist, Ralph Loop is not yet active. Invoke it NOW using the Skill tool:
   ```
   Skill(skill: "ralph-loop:ralph-loop", args: "<the full /conductor-feature-develop invocation with args> --completion-promise \"ALL MODULES IMPLEMENTED\" --max-iterations 100")
   ```
   If `.claude/ralph-loop.local.md` already exists, Ralph Loop is active — skip this step.

1. **Use project information from CLAUDE.md (already in context)** — extract JDK path, Maven path, database credentials,
   message queue credentials, Keycloak config, and all infrastructure details. These values
   are required for every subsequent tool command in this session.
2. **Verify context artifacts exist** — Check that model/, mockup/, specification/, and test/
   folders contain the required files. If missing, stop and inform user to run
   `/conductor-feature-prepare` first.
3. Check if `<app_folder>/context/develop/IMPLEMENTATION_MASTER.md` exists
4. If it exists, read it and determine the current state:
   - Scan the Module Implementation Status table for the FIRST module with status != COMPLETED
   - If ALL modules are COMPLETED:
     - **Version increment check**: If a `version` argument was provided, check whether
       IMPLEMENTATION_MASTER.md already tracks that version (look for `**Version**: <version>`
       in the header). If the version is NOT tracked yet, this is a **version increment** —
       update IMPLEMENTATION_MASTER.md to reset affected modules to PENDING status, add the new
       version to the header, and then proceed to Phase 2 (Implementation) for the affected modules.
     - **README check**: If `<source-code-path>/README.md` does not exist, proceed to
       Phase 5 (Generate README.md) — all modules are done but the README hasn't been created yet.
     - If no version argument (or version already tracked) AND README.md exists → output
       `<promise>ALL MODULES IMPLEMENTED</promise>` and stop
   - Otherwise, read its `IMPLEMENTATION_MODULE.md` for detailed progress
   - Resume from the last incomplete step in the checklist
5. If it does not exist, proceed to Phase 1 (Planning — fresh start)

### Phase 1: Planning

1. **Use project information from CLAUDE.md (already in context)** — extract all paths, credentials,
   and infrastructure configuration. This is the single source of truth for JDK, Maven, database,
   message queue, Keycloak, and all other tool configurations.
2. Read `<app_folder>/context/test/TEST_PLAN.md`
3. Extract the **Execution Order** (Section 5) and **Layer Classification** (Section 4)
4. The execution order defines the module sequence — use it as-is
5. Read `<app_folder>/context/specification/SPECIFICATION.md` for shared infrastructure context

Create `<app_folder>/context/develop/IMPLEMENTATION_MASTER.md` with this structure:

```markdown
# Implementation Master - <Application Name>

**Started**: <date>
**Source Code**: <source-code-path>
**Context**: <app_folder>/context
**Status**: IN PROGRESS

---

## Execution Order

<Copy the execution order tree from TEST_PLAN.md>

---

## Module Implementation Status

| # | Module | Layer | Status | Started | Completed | Notes |
|---|--------|-------|--------|---------|-----------|-------|
| 1 | User | L1 | PENDING | - | - | |
| 2 | Location Information | L2 | PENDING | - | - | |
...

---

## Module Details

### 1. User

**Resources**:
- User Story: <list relevant story IDs>
- Model: `model/user/model.md`
- Specification: `specification/user/SPEC.md`
- Test Spec: `test/user/TEST_SPEC.md`
- Mockup: `mockup/<role>/content/<screen>.html`

**Dependencies**: None

---

### 2. Location Information
...
```

### Phase 2: Pre-Implementation (Scaffolding)

Read the SPECIFICATION.md shared infrastructure sections and scaffold the project.

**CRITICAL — Source Code Placement Rule:**
All source code MUST be placed directly in `<source-code-path>/` (which defaults to `<app_folder>/`).
The `context/` folder already exists there and must be preserved. When using project creation tools
like `composer create-project` or `mvn archetype:generate`, ensure you do NOT create a nested
subdirectory. Instead:
- For Laravel: Create the project in a temporary directory (e.g., `<source-code-path>/_temp_scaffold`),
  then move ALL files (including dotfiles) from that temp directory up to `<source-code-path>/`,
  then remove the empty temp directory. This avoids overwriting the existing `context/` folder.
- For Spring Boot: Same approach — scaffold into a temp dir, then move files up.
- NEVER use the project slug/name as the target directory if it would create a nested folder.

**Scaffolding Checklist** (adapt to the technology stack from SPECIFICATION.md):

1. **Project structure**: Create the project skeleton directly in `<source-code-path>/`
2. **Build & dependency configuration**: composer.json / pom.xml / build.gradle with all dependencies
3. **Application configuration**: .env, config files, or application.yml as appropriate
4. **Security configuration**: Keycloak/OAuth2 or other auth provider setup
5. **Shared layouts**: Blade / JTE / other template layout files (header, footer, sidebar)
6. **Shared components**: UI components (Tailwind), JS structure, CSS
7. **Data access layer**: Base repository / model configuration
8. **Error handling**: Global exception handlers
9. **Theming**: Theme configuration from spec
10. **Pagination**: Shared pagination support
11. **Messaging**: Message queue configuration if applicable
12. **Scheduling**: Scheduled task configuration if applicable
13. **Playwright test project**: Initialize Playwright in `<source-code-path>/e2e/` with:
    - `package.json` with Playwright dependency
    - `playwright.config.ts` with base URL from TEST_PLAN.md
    - Helper utilities for login, navigation, data seeding
14. **Mockup baseline screenshots**: Capture baseline screenshots from HTML mockups for visual consistency testing:
    - Start the mockup server (`npm start` in `<app_folder>/context/mockup/`)
    - For each role/screen in the mockup, capture a screenshot to `<source-code-path>/e2e/visual-baselines/`
    - Stop the mockup server after capture
    - These baselines will be compared against the application output during module testing

After scaffolding, verify the application compiles/starts. **Use the exact paths, CLIs, and
credentials from `CLAUDE.md`** — do NOT use generic commands or assume default paths:
```bash
# Examples (actual paths come from CLAUDE.md):

# Laravel:
<php-path-from-CLAUDE.md>/php.exe artisan --version
<php-path-from-CLAUDE.md>/php.exe artisan serve

# Spring Boot:
JAVA_HOME="<jdk-path>" <maven-path>/mvn -f <source-code-path>/pom.xml clean compile
```

Update IMPLEMENTATION_MASTER.md: mark scaffolding as COMPLETED.

### Phase 3: Implementation (Per Module)

For each module in execution order:

#### Step 3.1: Initialize Module Tracking

Create `<app_folder>/context/develop/<module-slug>/IMPLEMENTATION_MODULE.md`:

```markdown
# Implementation - <Module Name>

**Module**: <Module Name>
**Layer**: <Layer>
**Status**: IN PROGRESS
**Started**: <date>

---

## Resources

| Resource | Path |
|----------|------|
| User Stories | <IDs from PRD.md> |
| Model | `model/<module-slug>/model.md` |
| Specification | `specification/<module-slug>/SPEC.md` |
| Test Spec | `test/<module-slug>/TEST_SPEC.md` |
| Mockup | `mockup/<role>/content/<screen>.html` |

---

## Implementation Checklist

### UI Layer

- [ ] 1. Read and analyze module resources
- [ ] 2. Implement module model (entities/documents)
- [ ] 3. Implement repository layer
- [ ] 4. Implement service layer
- [ ] 5. Implement controller layer
- [ ] 6. Implement view templates (list, detail, form pages)
- [ ] 7. Write Playwright E2E tests (UI scenarios)
- [ ] 8. Run E2E tests and verify

### User Stories

<For each user story ID from the module's SPEC.md traceability section, add a checklist item:>
- [ ] USxxxx: <description>

### Non-Functional Requirements

<For each NFR ID from the module's SPEC.md traceability section, add a checklist item:>
- [ ] NFRxxxx: <description>

### Messaging Pipeline (if applicable — include only if module has messaging NFRs)

- [ ] Implement message consumer
- [ ] Implement message validator
- [ ] Implement ACK publisher
- [ ] Implement forward publisher
- [ ] Implement queue configuration
- [ ] Implement module events
- [ ] Write E2E tests for message processing flow

### Scheduled Jobs (if applicable — include only if module has scheduling NFRs)

- [ ] Implement scheduled job
- [ ] Write E2E tests for scheduled job

### Visual Consistency

- [ ] Visual consistency testing (mockup vs application)
- [ ] Fix visual deviations (if any)

---

## Implementation Log

### Step 1: Analyze Module Resources
<timestamp> - Started
- Read model.md, SPEC.md, TEST_SPEC.md, mockup HTML
- Key findings: ...
```

#### Step 3.2: Analyze Module Resources

Read ALL module-specific resources:
- `model/<module-slug>/model.md` — document structure, collections, fields
- `model/<module-slug>/schemas.json` — JSON schema examples
- `specification/<module-slug>/SPEC.md` — full technical specification
- `test/<module-slug>/TEST_SPEC.md` — test scenarios, seeding scripts, assertions
- `mockup/<role>/content/<module_screen>.html` — UI mockup for visual reference
- Relevant entries from `PRD.md` — user stories for this module
- Relevant message files from `reference/message/` if applicable

Update IMPLEMENTATION_MODULE.md: mark step 1 complete with findings summary.

#### Step 3.3: Implement Module Code

Follow the module SPEC.md to implement, in order:
1. **Entity/Document classes** — from model.md + schemas.json
2. **Repository interfaces** — from SPEC.md data access section
3. **Service classes** — business logic from SPEC.md
4. **Mappers** — if specified in SPEC.md
5. **Controller classes** — routes, request handling from SPEC.md
6. **View templates** — from SPEC.md view section + mockup HTML
7. **Message listeners** — from SPEC.md messaging section (if applicable)
8. **Scheduled jobs** — from SPEC.md scheduling section (if applicable)

After each major component, update IMPLEMENTATION_MODULE.md checklist.

#### Step 3.4: Implement Playwright E2E Tests

From the module's TEST_SPEC.md:

1. **Create test file**: `<source-code-path>/e2e/tests/<module-slug>.spec.ts`
2. **Implement data seeding**: Use the seeding scripts from TEST_SPEC.md Section 4
3. **Implement test scenarios**: Convert each scenario from TEST_SPEC.md Section 5 into Playwright tests
4. **DO NOT implement cleanup scripts** — test data must persist for downstream modules

Pattern for test file:
```typescript
import { test, expect } from '@playwright/test';

test.describe('<Module Name>', () => {
  // Data seeding (runs once before all tests in this module)
  test.beforeAll(async () => {
    // Execute seeding script from TEST_SPEC.md Section 4
  });

  // DO NOT add afterAll cleanup — data persists for dependent modules

  test('<scenario-id>: <scenario-name>', async ({ page }) => {
    // Steps from TEST_SPEC.md scenario
  });
});
```

#### Step 3.5: Run E2E Tests

```bash
cd <source-code-path>/e2e && npx playwright test tests/<module-slug>.spec.ts
```

- If tests pass: proceed to visual consistency testing (Step 3.6)
- If tests fail: analyze failures, fix code, re-run (do NOT clean test data unless re-seeding is required for retest)

#### Step 3.6: Visual Consistency Testing (Mockup vs Application)

After functional E2E tests pass, compare the UI output of the application against the approved HTML mockup screens. The goal is to verify **aesthetic consistency** — colors, alignment, padding, margins, font sizes, layout structure — NOT content (data values will differ).

**Process:**

1. **Identify mockup screens** for this module from `<app_folder>/context/mockup/<role>/content/` that correspond to the implemented views.

2. **Capture mockup baselines** (if not already captured during scaffolding):
   - Start the mockup server: `cd <app_folder>/context/mockup && npm start`
   - Navigate to each screen for the module and capture a screenshot
   - Save to `<source-code-path>/e2e/visual-baselines/<module-slug>/<screen-name>.png`
   - Stop the mockup server

3. **Create visual comparison test**: `<source-code-path>/e2e/tests/<module-slug>.visual.spec.ts`
   - For each screen in the module, navigate to the corresponding application page
   - Capture a screenshot of the application output
   - Use Playwright's `toHaveScreenshot()` with a **threshold** to allow content differences while catching layout/styling deviations
   - Compare against the mockup baseline

4. **Visual test pattern**:
```typescript
import { test, expect } from '@playwright/test';
import { loginAs } from '../helpers/auth';
import path from 'path';

test.describe('<Module Name> - Visual Consistency', () => {
  test('<screen-name> matches mockup layout', async ({ page }) => {
    await loginAs(page, '<role>');
    await page.goto('<app-route-for-screen>');
    await page.waitForLoadState('networkidle');

    // Compare against mockup baseline — threshold allows content differences
    // but catches color, alignment, padding, margin, and layout deviations
    await expect(page).toHaveScreenshot('<module-slug>-<screen-name>.png', {
      maxDiffPixelRatio: 0.15,   // Allow up to 15% pixel difference (content varies)
      threshold: 0.3,            // Per-pixel color threshold (0-1, higher = more lenient)
      animations: 'disabled',
    });
  });
});
```

5. **On first run**, Playwright generates actual screenshots. Manually review them:
   - If the layout aesthetics match the mockup (colors, spacing, alignment), approve by updating the snapshots
   - If deviations are found, fix the templates / CSS and re-run

6. **What to check** (aesthetic consistency):
   - Color scheme matches (backgrounds, borders, text colors, button colors)
   - Spacing and alignment (padding, margins, gaps between elements)
   - Layout structure (grid/flex arrangement, sidebar/header/content proportions)
   - Typography (font sizes, weights, line heights — relative, not exact)
   - Component styling (buttons, tables, forms, cards, badges)
   - Responsive breakpoints if applicable

7. **What to ignore** (content differences are expected):
   - Actual text content / data values
   - Number of rows in tables
   - Dynamic timestamps, IDs, or generated values

Update IMPLEMENTATION_MODULE.md: mark visual consistency testing step complete with findings.

- If visual tests pass: update IMPLEMENTATION_MODULE.md status to COMPLETED
- If visual tests reveal layout issues: fix templates/CSS, re-run both functional and visual tests

### Phase 4: Post-Implementation (Per Module)

After each module completes:

1. Update `<app_folder>/context/develop/<module-slug>/IMPLEMENTATION_MODULE.md`:
   - Set Status to COMPLETED
   - Record completion date
   - Record test results summary

2. Update `<app_folder>/context/develop/IMPLEMENTATION_MASTER.md`:
   - Update module row: Status = COMPLETED, Completed = date
   - Add notes about any issues encountered

3. **Check if ALL modules are now COMPLETED**:
   - Scan the Module Implementation Status table in IMPLEMENTATION_MASTER.md
   - If ALL modules have Status = COMPLETED:
     - **Proceed to Phase 5 (Generate README.md)** before outputting the completion promise
   - If there are still PENDING modules:
     - **IMMEDIATELY proceed to the next module** in execution order — do NOT stop
     - Continue implementing until context limits force a natural stop
     - The Ralph Loop will re-feed the prompt, and the next iteration will resume via Phase 0

### Phase 5: Generate README.md

After ALL modules are COMPLETED, generate a `README.md` file in the application root folder
(`<source-code-path>/README.md`). This file serves as the primary documentation for developers
to understand the application architecture, navigate the codebase, and run the application.

**CRITICAL**: The README content MUST be derived from `SPECIFICATION.md` — the technical
specification generated by the chosen `specgen-*` skill. Do NOT invent or assume technology
details. Extract all architecture, stack, configuration, and run instructions directly from
the specification document.

#### Step 5.1: Read SPECIFICATION.md

Read `<app_folder>/context/specification/SPECIFICATION.md` and extract:
- **Technology stack** — framework, language, template engine, CSS framework, JS libraries, database, auth provider, messaging, scheduling
- **Application architecture** — packaging structure (e.g., Spring Modulith, Laravel Modules), layer conventions (controller, service, repository, view), shared infrastructure components
- **Configuration** — environment variables, config files, database connection, auth provider setup, message queue setup
- **Build and run commands** — how to install dependencies, compile, run the application, run tests

#### Step 5.2: Generate README.md

Create `<source-code-path>/README.md` with the following structure:

```markdown
# <Application Name>

<One-paragraph description from SPECIFICATION.md project overview>

## Technology Stack

<Table of technologies, versions, and purposes — extracted from SPECIFICATION.md technology stack section>

| Technology | Version | Purpose |
|-----------|---------|---------|
| ... | ... | ... |

## Architecture

<Description of the application architecture from SPECIFICATION.md — packaging approach,
layer conventions, module organization. Include a text-based or Mermaid diagram if the
specification describes the architecture visually.>

## Folder Structure

<Tree representation of the actual source code folder structure as implemented. Use the
real directory layout from `<source-code-path>/`, excluding `context/`, `node_modules/`,
`.git/`, and other non-essential directories. Annotate key directories with their purpose.>

```
<source-code-path>/
  app/                    # (Laravel) Application core
  Modules/                # (Laravel) Feature modules
  resources/              # (Laravel) Views, CSS, JS
  src/                    # (Spring) Java source
  e2e/                    # Playwright E2E tests
  ...
```

## Prerequisites

<List of software prerequisites needed to run the application — JDK, PHP, Composer, Maven,
Node.js, database, message queue, auth provider, etc. Extracted from SPECIFICATION.md.>

## Getting Started

### Installation

<Step-by-step instructions to install dependencies — e.g., `composer install`, `mvn clean install`,
`npm install`. Derived from SPECIFICATION.md build configuration section.>

### Configuration

<Instructions for setting up configuration — .env file, application.yml, database setup,
auth provider configuration. Reference the specific config files and environment variables
from SPECIFICATION.md.>

### Running the Application

<Commands to start the application — e.g., `php artisan serve`, `mvn spring-boot:run`.
Include the default URL where the application will be accessible.>

### Running Tests

<Commands to run the Playwright E2E test suite:>

```bash
cd e2e && npx playwright test
```

<Any additional test commands — unit tests, integration tests — if applicable from the spec.>

## Modules

<Table listing all implemented modules, their layer classification, and a brief description.
Extracted from IMPLEMENTATION_MASTER.md execution order and module details.>

| Module | Layer | Description |
|--------|-------|-------------|
| ... | ... | ... |
```

**Adapt the template above** to match the actual technology stack from SPECIFICATION.md:
- For **Spring Boot** apps: use Maven/Gradle commands, `src/main/java` paths, `application.yml` config
- For **Laravel** apps: use Composer/Artisan commands, `app/` paths, `.env` config
- For **REST API** apps: omit view/template sections, focus on API endpoints and Swagger/OpenAPI docs
- Include only sections that are relevant to the actual specification — do NOT add sections for
  features that are not part of the spec (e.g., skip messaging section if no messaging is configured)

#### Step 5.3: Update Tracking and Complete

1. Update IMPLEMENTATION_MASTER.md:
   - Set top-level `**Status**:` to `COMPLETED`
   - Add a note: `README.md generated at <source-code-path>/README.md`
2. Output the Ralph Loop completion promise: `<promise>ALL MODULES IMPLEMENTED</promise>`
3. This signals the Ralph Loop to exit

## Mockup Interpretation Guide (CRITICAL)

The HTML mockups in `<app_folder>/context/mockup/` are organized into **role-based subfolders**
(e.g., `hub_administrator/content/`, `hub_operation_support/content/`). This folder structure
represents which screens each role can access — it does **NOT** dictate URL patterns or imply
that URLs should contain role names.

### Rule: Module-Based URLs, NOT Role-Based URLs

**WRONG** (role in URL — NEVER do this):
```
/hub-administrator/employer
/hub-operation-support/quota
/hub-administrator/industrial-classification/create
```

**CORRECT** (module-based URL):
```
/employer
/quota
/industrial-classification/create
```

Controllers MUST use module-based `@RequestMapping` paths. Role enforcement is handled via
`@PreAuthorize` annotations on the controller class or method level — NOT through URL segregation.

### Template Strategy for Role-Based UI Differences

When a module's mockup exists under multiple role folders, the screens may differ. Follow this
decision framework to determine the template strategy:

#### Step 1: Classify Each Screen

For each screen in the module, compare the mockup versions across roles and classify:

| Classification | Description | Example |
|---------------|-------------|---------|
| **Role-Exclusive** | Screen exists under only ONE role folder | `audit_trail.html` (admin only), `job_demand.html` (ops only) |
| **Shared — Minor Differences** | Same layout and structure, but some elements are shown/hidden per role (e.g., action buttons, edit toggles, "Add" button) | `document_classification.html` — admin has toggles + edit; ops has read-only badges |
| **Shared — Major Differences** | Fundamentally different layout, columns, or purpose despite same module name | `occupation_classification.html` — admin sees CRUD list; ops sees corridor mapping view |

#### Step 2: Apply the Appropriate Strategy

**A. Role-Exclusive Screens → Single template, single controller method**

One template, one route. Only the authorized role can access it. No conditional logic needed.

**B. Shared — Minor Differences → Single template with role-based conditionals**

Use **one template** with conditional blocks to show/hide elements based on the user's role.
This is the preferred approach when the page layout is structurally the same but some UI
elements (buttons, action columns, edit controls, banners) differ.

The controller method is accessible to both roles with appropriate authorization annotations.

**C. Shared — Major Differences → Separate templates, controller selects at runtime**

When the two role versions have **fundamentally different layouts, columns, or purposes**
(not just show/hide of a few elements), create separate templates and let the controller
choose which to render based on the authenticated user's role.

#### Step 3: Document the Decision

In `IMPLEMENTATION_MODULE.md`, under the analysis step, record the template strategy chosen
for each screen and why:

```markdown
### Template Strategy

| Screen | Roles | Classification | Strategy |
|--------|-------|---------------|----------|
| employer list | ops-only | Role-Exclusive | Single template |
| document_classification | both | Minor Differences | Single template + conditional |
| occupation_classification | both | Major Differences | Separate templates |
```

### How to Read Mockups During Implementation

When implementing a module (Step 3.2 — Analyze Module Resources):

1. **List all mockup files** for this module across ALL role folders:
   ```
   mockup/<role1>/content/<module>*.html
   mockup/<role2>/content/<module>*.html
   ```

2. **Read each version** and compare structure, columns, buttons, and layout

3. **Classify** each screen using the framework above

4. **Extract visual design** (colors, spacing, components) from the mockup but
   **ignore the URL paths** embedded in the mockup HTML (e.g., `href="/hub_administrator/..."`)
   — these are mockup navigation links, NOT the actual application routes

5. **Map mockup URLs to module URLs**:
   - Mockup: `/hub_administrator/employer` → App: `/employer`
   - Mockup: `/hub_operation_support/quota_allocation` → App: `/quota-allocation`

## Critical Rules

1. **CLAUDE.md is the source of truth for all tool commands** — CLAUDE.md is automatically
   loaded into context. Use the exact JDK path, Maven path, database credentials,
   message queue credentials, Keycloak CLI path, and all other infrastructure details from
   CLAUDE.md. NEVER hardcode, guess, or use generic paths like `./mvnw` or `java`.

2. **No test data cleanup** — DO NOT clean up test data after module tests pass. Test data
   from earlier modules is required as seed data for downstream modules. Only clean up if
   you need to re-seed for a retest.

3. **Context window awareness (Ralph Loop handles recovery)** — If approaching context limits,
   save progress gracefully:
   - Update IMPLEMENTATION_MODULE.md with current progress (mark completed steps, log findings)
   - Update IMPLEMENTATION_MASTER.md with current status
   - The Ralph Loop will automatically re-feed the prompt, and the next iteration will resume
     from exactly where you left off via Phase 0 resume check
   - Do NOT worry about "losing work" — the tracking files ARE your persistence mechanism

4. **Usage limit handling** — If you hit API usage limits, wait and resume. The tracking
   files ensure no work is lost. Ralph Loop will re-feed the prompt on next iteration.

5. **Module order is strict** — Follow the execution order from TEST_PLAN.md exactly.
   Dependencies mean earlier modules must complete before later ones can start.

6. **Track everything** — Every action should be logged in IMPLEMENTATION_MODULE.md so
   that any future session (or Ralph Loop iteration) can understand what was done and what
   remains. This is CRITICAL for Ralph Loop — without accurate tracking, iterations will
   repeat already-completed work.

7. **Test-driven** — Implementation is guided by TEST_SPEC.md. The test spec defines
   what the code must do. The SPEC.md defines how to build it.

8. **Mockup fidelity** — View templates MUST match the approved HTML mockup screens aesthetically.
   Use the mockup as the visual reference for layout, components, and styling. After functional
   E2E tests pass, run visual consistency tests comparing app screenshots against mockup
   baselines to verify colors, alignment, padding, margins, and layout structure match. Content
   differences (data values, row counts) are expected and acceptable — only aesthetic deviations
   should be flagged and fixed.

9. **Specification compliance** — Follow the SPECIFICATION.md shared infrastructure sections
   for all cross-cutting concerns (security, theming, pagination, error handling, etc.).

10. **Module-based URLs only** — NEVER use role names in URL paths. Mockup folder structure
    is for organizing mockups by role visibility, NOT for defining URL routes. All controllers
    use module-based paths. Role enforcement is via authorization annotations. See the
    "Mockup Interpretation Guide" section above for the full template strategy framework.

11. **Ralph Loop discipline — NEVER stop prematurely** — After completing a module, IMMEDIATELY
    check for the next pending module and start it. Do NOT output the completion promise
    (`<promise>ALL MODULES IMPLEMENTED</promise>`) until EVERY module is COMPLETED. Do NOT
    stop "to let the user review" — Ralph Loop handles multi-iteration execution automatically.
    The only valid reasons to stop within an iteration are: (a) context window approaching limit,
    (b) all modules completed (output promise), or (c) an unrecoverable error requiring user input.

12. **Complete implementation per module** — Each module must have ALL of these before marking
    COMPLETED: (a) all source files (entities, repositories, services, controllers, mappers),
    (b) all view templates (page + fragments), (c) E2E test file with all scenarios from TEST_SPEC,
    (d) all E2E tests passing, (e) ALL user stories checked off in IMPLEMENTATION_MODULE.md,
    (f) ALL NFRs checked off in IMPLEMENTATION_MODULE.md. Do NOT mark a module COMPLETED with
    partial implementation or failing tests — this would cause the next Ralph Loop iteration to
    skip it. If a module has messaging/async NFRs that cannot be implemented yet (e.g., upstream
    adapter not available), mark the module as **PARTIALLY COMPLETED** and leave the unchecked
    NFR items visible in the checklist. Only mark COMPLETED when every user story AND every NFR
    is implemented and tested.

13. **Auto-start Ralph Loop** — When this skill is triggered, the FIRST action (Phase 0, Step 0)
    MUST be to check if Ralph Loop is active (`.claude/ralph-loop.local.md` exists). If not active,
    invoke `Skill(skill: "ralph-loop:ralph-loop", args: "...")` with the full orchestrator prompt
    and `--completion-promise "ALL MODULES IMPLEMENTED" --max-iterations 100`. Do NOT proceed
    with any implementation work until Ralph Loop is confirmed active.

14. **NO creative alternatives for 3rd party applications (CRITICAL)** — You MUST use the EXACT
    methods, connection strings, CLIs, and credentials described in `CLAUDE.md` for
    accessing all external infrastructure. This includes but is not limited to:

    **NEVER do any of the following:**
    - Connect to databases via Docker (`docker exec`, `docker run`, CLI inside a container)
    - Start or restart services with alternative databases
    - Spin up services via Docker Compose or Docker run
    - Use `docker exec` to access any service that is described as running natively in CLAUDE.md
    - Install or use alternative CLI tools when CLAUDE.md specifies a different access method
    - Assume services run in Docker when CLAUDE.md says they run natively (or vice versa)
    - Use embedded/in-memory databases as substitutes for the actual configured database
    - Guess connection strings, ports, or credentials — ALWAYS read them from CLAUDE.md

    **ALWAYS do the following:**
    - Use database connection strings EXACTLY as specified in CLAUDE.md
    - Use service instances EXACTLY as configured in CLAUDE.md
    - Use CLIs at the EXACT paths specified in CLAUDE.md
    - For E2E test data seeding, connect to the database using the EXACT connection details from
      CLAUDE.md

    **WHY**: Creative alternatives (Docker containers, alternative databases, alternative CLIs) connect to
    DIFFERENT data stores or service instances than the ones the application is configured to use.
    This produces wrong test results, missing data, and phantom failures. The developer's machine
    has all services running natively as described in CLAUDE.md — use them as-is.

15. **Source code goes directly in `<source-code-path>/` — NO nested subdirectories (CRITICAL)** —
    The `<source-code-path>` (which defaults to `<app_folder>`) already contains a `context/` folder
    with all generated artifacts. Source code files MUST be placed directly alongside `context/` in the same
    directory — NOT inside a nested project subfolder. When using project scaffolding tools,
    these tools create a new subdirectory by default. You MUST work around this by:
    - Creating the project in a temporary subdirectory (e.g., `<source-code-path>/_temp_scaffold`)
    - Moving ALL files (including dotfiles) up to `<source-code-path>/`
    - Removing the empty temporary directory
    - Verifying the `context/` folder was NOT overwritten or deleted

    **WHY**: The folder structure convention is that `<app_folder>` contains both `context/` (artifacts)
    and source code at the same level. A nested subdirectory breaks all relative paths, makes the
    project structure confusing, and separates context from code unnecessarily.

16. **Context artifacts must pre-exist** — This skill does NOT generate context artifacts. If
    module models, mockups, specifications, or test specs are missing, stop and instruct the
    user to run `/conductor-feature-prepare` first. Do NOT attempt to generate artifacts inline.

17. **README.md is mandatory before completion** — After ALL modules are COMPLETED, Phase 5
    (Generate README.md) MUST run before outputting the completion promise. The README content
    MUST be derived from SPECIFICATION.md — do NOT invent stack details, run commands, or
    architecture descriptions. If SPECIFICATION.md describes a Spring Boot app, the README
    reflects Spring Boot conventions. If it describes a Laravel app, the README reflects
    Laravel conventions. The README documents the actual implemented application, not a generic
    template.
