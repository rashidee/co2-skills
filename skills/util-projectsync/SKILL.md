---
name: util-projectsync
description: >
  Synchronize project folder structure, PRD.md and BUG.md files based on the canonical application
  and module definitions in CLAUDE.md. Validates dependencies (circular, missing, logical) and
  deployment environments (coverage, build/deploy consistency) across all applications. Creates
  missing application folders, scaffolds new PRD.md and BUG.md files from templates, and adds
  missing module sections to existing files. Inserts [TODO] annotations into CLAUDE.md for
  validation failures.
  Trigger on keywords: "sync project", "project sync", "sync folders", "sync modules",
  "sync PRD", "sync BUG", "initialize project structure", "scaffold project",
  "create project folders", "sync application structure", "validate dependencies",
  "validate deployment".
  Accepts no arguments — reads all configuration from CLAUDE.md.
---

# Util Project Sync

Synchronize project folder structure, PRD.md and BUG.md files to match the canonical structure defined in CLAUDE.md. Validates dependencies and deployment environments before syncing.

## Input Resolution

This skill requires no arguments. All configuration is read from CLAUDE.md in the project root.

### Required Sections in CLAUDE.md

| Section | Purpose |
|---------|---------|
| `# Custom Applications` | List of custom applications to create folders for and validate |
| `# Supporting 3rd Party Applications` | List of 3rd party applications referenced as dependencies |
| `# Environment` | Deployment environments defined for the project |
| `# Modules` | Module definitions grouped under `## System Module` and `## Business Module` |

### Application Detection

1. Read the `# Custom Applications` section in CLAUDE.md
2. Each application is listed as a `## <Application Name>` heading under the section
3. If the section contains `**No Custom Applications**` or similar "none" indicator, report that there are no applications to sync and **stop**

### 3rd Party Application Detection

1. Read the `# Supporting 3rd Party Applications` section in CLAUDE.md
2. Each 3rd party application is listed as a `## <Application Name>` heading under the section
3. If the section contains `**No Supporting 3rd Party Applications**` or similar "none" indicator, treat as having zero 3rd party applications

### Environment Detection

1. Read the `# Environment` section in CLAUDE.md
2. Environments are listed as bullet items under the section (e.g., `- home_server:`, `- abc_cloud:`)
3. Extract the environment identifier (the part before the colon) for each environment
4. If the section does not exist or contains no environments, skip deployment validation and record a warning

### Module Detection

1. Read the `# Modules` section in CLAUDE.md
2. Modules are organized under `## System Module` and `## Business Module` subsections
3. Each module is listed as a `### <Module Name>` heading under its parent section
4. If a module section contains `**There is no System Module**` or `**There is no Business Module**` or similar "none" indicator, treat that category as having zero modules

## Workflow

### 1. Parse CLAUDE.md

Read CLAUDE.md from the project root and extract:
- **Custom Application list**: names and their full content (description, dependencies, build strategy, deployment strategy) from `## <Name>` headings under `# Custom Applications`
- **3rd Party Application list**: names and their full content from `## <Name>` headings under `# Supporting 3rd Party Applications`
- **All Application names**: combined list of both custom and 3rd party application names (used as the valid dependency pool)
- **Environment list**: environment identifiers from `# Environment` section
- **System Modules**: names from `### <Name>` headings under `## System Module`
- **Business Modules**: names from `### <Name>` headings under `## Business Module`

### 2. Validate Dependencies and Deployment

Run all validation checks across **both** `# Custom Applications` and `# Supporting 3rd Party Applications`. Collect all issues found. After all checks are complete, insert `[TODO]` annotations into CLAUDE.md for each issue.

#### 2a. Dependency Validation

For each application (custom and 3rd party) that has a `- Depends on:` section, extract its dependency list. Each dependency is a bullet item under "Depends on:" — the dependency name is the text before any parenthetical qualifier or "for"/"to"/"when" clause.

**Check 1: No Circular Dependencies**

Build a directed dependency graph from all applications. Detect cycles using depth-first traversal. A circular dependency exists when application A depends on B, and B depends (directly or transitively) on A.

Example of a circular dependency:
- Hub Middleware depends on HC Adapter Message Queue
- HC Adapter Message Queue depends on Hub Single Sign On
- Hub Single Sign On depends on Hub Middleware (cycle!)

For each cycle found, record the full cycle path (e.g., `Hub Middleware -> HC Adapter Message Queue -> Hub Single Sign On -> Hub Middleware`).

**Check 2: All Dependencies Exist**

For each dependency listed, verify that a matching application exists in either `# Custom Applications` or `# Supporting 3rd Party Applications`. Match by name, case-insensitively. The dependency name may include a database name qualifier in parentheses (e.g., `Hub Support Database (urp_hub_kc)`) — match against the application name only (e.g., `Hub Support Database`).

**Check 3: Dependencies Are Logical and Consistent**

Check for logical issues:
- **Duplicate dependencies**: The same dependency listed more than once under the same application (e.g., `Hub Cache` listed twice under Hub Middleware)
- **Self-dependency**: An application listing itself as a dependency
- **Dependency on application in wrong tier**: A 3rd party application depending on a custom application (3rd party should not depend on custom apps; custom apps depend on 3rd party or other custom apps)

#### 2b. Deployment Environment Validation

**Check 4: All Applications Have Deployment Strategy for All Environments**

For each application (custom and 3rd party), check that its `- Deployment Strategy:` section contains a sub-section for every environment defined in `# Environment`. Each environment should appear as a bullet item like `- <environment_name>:` under the deployment strategy.

Flag applications that:
- Are missing a deployment strategy for one or more environments
- Have a deployment strategy for an environment not listed in `# Environment` (unknown environment)

**Check 5: All Applications Have Build and Deploy Strategy Defined**

For each application (custom and 3rd party), verify that both sections exist:
- `- Build Strategy:` with at least one bullet of content
- `- Deployment Strategy:` with at least one bullet of content

Flag applications missing either section entirely, or where the section exists but has no content.

**Check 6: Build Strategy and Deployment Strategy Consistency**

Verify that the build strategy is compatible with the deployment strategy:

| Build Strategy Pattern | Expected Deployment Pattern |
|------------------------|-----------------------------|
| "Build as Docker image" or "Build as a Docker image" | Deployment must mention "Kubernetes Deployment", "Docker", "container", or similar container-based deployment |
| "Use official ... Docker image" | Deployment must mention "Kubernetes Deployment", "Docker", "container", or similar container-based deployment |

Flag inconsistencies where:
- Build strategy produces a Docker image but deployment does not use container-based deployment
- Build strategy does not produce a Docker image but deployment references Kubernetes/Docker/container deployment

**Check 7: Environments Are Consistent**

Verify that the set of environments referenced across all applications' deployment strategies is consistent:
- Collect all unique environment names found across all deployment strategies
- Compare against the environments defined in `# Environment`
- Flag any application using an environment not defined in `# Environment`
- Flag if any environment defined in `# Environment` is not used by any application

#### 2c. Insert TODO Annotations into CLAUDE.md

For each validation issue found, insert a `[TODO]` annotation line **immediately above** the `## <Application Name>` heading of the affected application in CLAUDE.md.

**Format:**
```
- [TODO] <CATEGORY>: <description>
## <Application Name>
```

**Categories:**
| Category | Code |
|----------|------|
| Circular Dependency | `CIRCULAR_DEP` |
| Missing Dependency | `MISSING_DEP` |
| Duplicate Dependency | `DUP_DEP` |
| Self Dependency | `SELF_DEP` |
| Illogical Dependency | `ILLOGICAL_DEP` |
| Missing Build Strategy | `MISSING_BUILD` |
| Missing Deploy Strategy | `MISSING_DEPLOY` |
| Missing Environment Coverage | `MISSING_ENV` |
| Unknown Environment | `UNKNOWN_ENV` |
| Build/Deploy Inconsistency | `BUILD_DEPLOY_MISMATCH` |
| Unused Environment | `UNUSED_ENV` |

**Examples:**
```markdown
- [TODO] CIRCULAR_DEP: Circular dependency detected: Hub Middleware -> HC Adapter Message Queue -> Hub Single Sign On -> Hub Middleware
- [TODO] MISSING_DEP: Dependency "Hub Analytics" does not exist in Supporting 3rd Party Applications or Custom Applications
- [TODO] DUP_DEP: Dependency "Hub Cache" is listed more than once
- [TODO] MISSING_BUILD: No Build Strategy defined
- [TODO] MISSING_DEPLOY: No Deployment Strategy defined
- [TODO] MISSING_ENV: Missing deployment strategy for environment "abc_cloud"
- [TODO] BUILD_DEPLOY_MISMATCH: Build strategy produces Docker image but deployment for "home_server" does not reference container-based deployment
## Hub Middleware
```

**Note:** If an application has multiple issues, insert multiple `[TODO]` lines above its heading — one per issue.

### 3. Ensure Application Folders Exist

For each application in the `# Custom Applications` list:
1. Convert the application name to `snake_case` for the folder name (e.g., "Hub Middleware" -> `hub_middleware`)
2. Check if a root-level folder exists matching that name (also check for numeric-prefixed variants like `1_hub_middleware`)
3. If the folder **does not exist**, create it at the project root using the snake_case name
4. If the folder **already exists**, skip creation

**Note:** Folders are only created for Custom Applications, not for Supporting 3rd Party Applications.

### 4. Sync PRD.md for Each Application

For each custom application folder:

#### 4a. PRD.md Does Not Exist

Create a new `PRD.md` inside the application folder using the template below:
- Populate the `# System Module` section with all system modules from CLAUDE.md
- Populate the `# Business Module` section with all business modules from CLAUDE.md
- Each module gets the standard subsections: `### User Story`, `### Non Functional Requirement`, `### Constraint`, `### Reference` — all versioned with `[v1.0.0]`
- The `# Standards` section is included with placeholder text

#### 4b. PRD.md Already Exists

1. Parse the existing PRD.md to find all module headings (`## <Module Name>`) under both `# System Module` and `# Business Module`
2. Compare against the module list from CLAUDE.md:
   - **Module exists in both CLAUDE.md and PRD.md**: Do nothing
   - **Module exists in CLAUDE.md but NOT in PRD.md**: Append the missing module section at the end of its parent section (`# System Module` or `# Business Module`) using the template structure with empty versioned subsections `[v1.0.0]`
   - **Module exists in PRD.md but NOT in CLAUDE.md**: Do nothing to the file. Record a warning for the summary output

### 5. Sync BUG.md for Each Application

For each custom application folder:

#### 5a. BUG.md Does Not Exist

Create a new `BUG.md` inside the application folder using the template below:
- Include the `# Common Module` section
- Populate the `# System Module` section with all system modules from CLAUDE.md
- Populate the `# Business Module` section with all business modules from CLAUDE.md
- Each module gets an empty section with a separator

#### 5b. BUG.md Already Exists

1. Parse the existing BUG.md to find all module headings (`## <Module Name>`) under both `# System Module` and `# Business Module`
2. Compare against the module list from CLAUDE.md:
   - **Module exists in both CLAUDE.md and BUG.md**: Do nothing
   - **Module exists in CLAUDE.md but NOT in BUG.md**: Append the missing module section at the end of its parent section using the template structure
   - **Module exists in BUG.md but NOT in CLAUDE.md**: Do nothing to the file. Record a warning for the summary output

### 6. Resolve PRD.md and BUG.md Paths

PRD.md and BUG.md files are located inside the application folder. The exact path depends on whether the project uses a `context/` subfolder:
- If `<app_folder>/context/` exists, use `<app_folder>/context/PRD.md` and `<app_folder>/context/BUG.md`
- Otherwise, use `<app_folder>/PRD.md` and `<app_folder>/BUG.md`

Follow the folder structure defined in CLAUDE.md's `# Folder structure` section to determine the correct path.

### 7. Resolve Latest Version

Scan all PRD.md and BUG.md files across all custom application folders and extract every version tag matching the pattern `[v{major}.{minor}.{patch}]`. Compare all found versions using semantic versioning rules and determine the highest version.

Write (or overwrite) a file named `VERSION` in the project root directory with the highest version number as raw content — no brackets, no prefix, no trailing newline beyond the version string.

**Example:** If the highest version found across all files is `[v1.2.3]`, the VERSION file content is:
```
1.2.3
```

**Rules:**
- If no version tags are found in any file, write `0.0.0` as the default
- Compare versions using semantic versioning: major takes precedence, then minor, then patch (e.g., `v2.0.0` > `v1.9.9`, `v1.2.0` > `v1.1.5`)
- Only consider version tags in the format `[v{major}.{minor}.{patch}]` — ignore other bracketed content like tag codes (`[USHM00003]`)

### 8. Output Summary

Print a summary of all actions taken, validation results, and warnings:

```
## Project Sync Summary

### Dependency & Deployment Validation
| Check | Status | Issues |
|-------|--------|--------|
| Circular Dependencies | PASS/FAIL | 0 or description |
| Missing Dependencies | PASS/FAIL | 0 or count |
| Duplicate Dependencies | PASS/FAIL | 0 or count |
| Self Dependencies | PASS/FAIL | 0 or count |
| Illogical Dependencies | PASS/FAIL | 0 or count |
| Missing Build Strategy | PASS/FAIL | 0 or count |
| Missing Deploy Strategy | PASS/FAIL | 0 or count |
| Missing Environment Coverage | PASS/FAIL | 0 or count |
| Unknown Environments | PASS/FAIL | 0 or count |
| Build/Deploy Consistency | PASS/FAIL | 0 or count |
| Unused Environments | PASS/FAIL | 0 or count |

### Validation TODOs Inserted
| Application | Section | Issues |
|-------------|---------|--------|
| Hub Middleware | Custom Applications | DUP_DEP: "Hub Cache" listed twice |
| SC Adapter | Custom Applications | MISSING_BUILD, MISSING_DEPLOY |

### Application Folders
| Application | Folder | Status |
|-------------|--------|--------|
| Hub Middleware | hub_middleware | Created |
| HC Adapter | hc_adapter | Already exists |

### PRD.md Sync
| Application | Status | Modules Added | Warnings |
|-------------|--------|---------------|----------|
| hub_middleware | Updated | Payment, Billing | Module "Legacy Auth" exists in PRD.md but not in CLAUDE.md |
| hc_adapter | Created | (all modules) | - |

### BUG.md Sync
| Application | Status | Modules Added | Warnings |
|-------------|--------|---------------|----------|
| hub_middleware | Updated | Payment, Billing | - |
| hc_adapter | Created | (all modules) | - |

### VERSION
- Latest version: 1.2.3 (written to VERSION file)
- Sources: hub_middleware/PRD.md [v1.2.3], hc_adapter/BUG.md [v1.1.0], ...

### Warnings
- [hub_middleware/PRD.md] Module "Legacy Auth" exists in PRD.md but not in CLAUDE.md — manual review recommended
```

## PRD.md Template

When creating a new PRD.md or adding module sections, use this structure:

```markdown
# Context
- This document contain user stories, non-functional requirements, constraints and references by module about this application
- The application name can be inferred from the root folder name where this file is located.
- This is a version controlled document, the version is indicated in the square bracket after each section title, and it will be updated whenever there is any change in the content of that section. The versioning format is [v{major}.{minor}.{patch}] where:
  - Major version will be updated when there is a significant change in the content that may affect the overall understanding of the module or the system.
  - Minor version will be updated when there is a minor change in the content that may affect some details but not the overall understanding of the module or the system.
  - Patch version will be updated when there is a small change in the content that does not affect the overall understanding of the module or the system, such as fixing typos or formatting issues.
- For user any item which is no longer valid or applicable, it will be marked with strikethrough and indicated in the new version.

---

# Standards
- Technical standards which should be referenced all the time in the development of the system.

## UI/UX
- No UI/UX standard for now

## Coding
- No coding standard for now

---

# System Module

## {{System Module Name}}

### User Story
[v1.0.0]

### Non Functional Requirement
[v1.0.0]

### Constraint
[v1.0.0]

### Reference
[v1.0.0]

---

# Business Module

## {{Business Module Name}}

### User Story
[v1.0.0]

### Non Functional Requirement
[v1.0.0]

### Constraint
[v1.0.0]

### Reference
[v1.0.0]

---
```

### Adding a Missing Module to Existing PRD.md

When a module exists in CLAUDE.md but not in PRD.md, append this block at the end of the appropriate parent section (before the next `#` heading or end of file):

```markdown

## {{Module Name}}

### User Story
[v1.0.0]

### Non Functional Requirement
[v1.0.0]

### Constraint
[v1.0.0]

### Reference
[v1.0.0]

---
```

## BUG.md Template

When creating a new BUG.md or adding module sections, use this structure:

```markdown
# Context
- This document contain list of bugs reported by users or testers about this application
- All previously reported bugs have been resolved and merged into the PRD.md as NFRs or constraints.

---

# Common Module

[v1.0.0]

---

# System Module

## {{System Module Name}}

[v1.0.0]

---

# Business Module

## {{Business Module Name}}

[v1.0.0]

---
```

### Adding a Missing Module to Existing BUG.md

When a module exists in CLAUDE.md but not in BUG.md, append this block at the end of the appropriate parent section:

```markdown

## {{Module Name}}

[v1.0.0]

---
```

## Important Rules

- **NEVER remove, modify, or delete existing content** in PRD.md or BUG.md. Only add new sections.
- **NEVER modify existing version tags** (e.g., `[v1.0.0]`). These are immutable.
- **NEVER modify existing module sections** — only append new modules that are missing.
- Preserve all existing content, formatting, and indentation exactly.
- Application folder names must use `snake_case`.
- Folders are only created for `# Custom Applications`, not for `# Supporting 3rd Party Applications`.
- PRD.md and BUG.md are only created/synced for `# Custom Applications`.
- Dependency and deployment validation covers **both** `# Custom Applications` and `# Supporting 3rd Party Applications`.
- When matching existing folders, account for numeric prefixes (e.g., `1_hub_middleware` matches application "Hub Middleware").
- When adding modules to existing files, maintain the correct order: system modules under `# System Module`, business modules under `# Business Module`.
- Always include separator lines (`---`) between module sections.
- New module sections added to existing files use `[v1.0.0]` as the initial version.
- Modules in PRD.md/BUG.md that don't exist in CLAUDE.md are **never removed** — only flagged as warnings in the summary.
- If CLAUDE.md has no custom applications defined, report this clearly and stop without making any changes.
- When inserting `[TODO]` annotations in CLAUDE.md, insert them **immediately above** the `## <Application Name>` heading — never inside the application's content.
- Do not insert duplicate `[TODO]` annotations. If re-running the skill, check for existing `[TODO]` lines above each application heading and skip if the same issue is already annotated.
- When matching dependency names, strip parenthetical qualifiers (e.g., `Hub Support Database (urp_hub_kc)` matches `Hub Support Database`).
- Validation is advisory — it does not block folder creation or PRD.md/BUG.md syncing. All steps run regardless of validation results.
