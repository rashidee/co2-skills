---
name: util-modulesync
description: >
  Synchronize module headers and module sections across all PRD.md and BUG.md files
  to match the canonical module structure defined in the project-level CLAUDE.md.
  Ensures every application context folder has the same module categories (Common, System Module,
  Business Module) and module sections in the same order as CLAUDE.md.
  Trigger on keywords: "sync modules", "synchronize modules", "module sync", "align modules",
  "check module structure", "validate module structure", "module consistency".
  No arguments required — operates on all application context folders automatically.
---

# Util Module Sync

Synchronize module headers and module sections across all `PRD.md` and `BUG.md` files to match the canonical module structure defined in the project-level `CLAUDE.md`.

## Overview

The project-level `CLAUDE.md` defines the canonical list of module categories and modules under the `# Domains` section. Each application has its own `context/PRD.md` and `context/BUG.md` that must mirror this module structure. This skill ensures consistency by:

1. **Phase 1 (Analysis):** Detecting modules in any `PRD.md` or `BUG.md` that do NOT exist in `CLAUDE.md`. If found, stop and report — the user must first update `CLAUDE.md` or remove the rogue module before syncing.
2. **Phase 2 (Sync):** Adding any missing module categories or module sections from `CLAUDE.md` into every application's `PRD.md` and `BUG.md`, in the correct order.

## Input

No arguments required. The skill automatically discovers all application context folders.

Example invocations:
- `/util-modulesync`

## Canonical Module Structure (from CLAUDE.md)

The canonical structure is extracted from CLAUDE.md under the `# Domains` section. The heading levels in CLAUDE.md are:
- `## <Category>` — module category headers (e.g., `## Common`, `## System Module`, `## Business Module`)
- `## <Module>` or `### <Module>` — individual module names (e.g., `## UI/UX Standards`, `### User`, `### Notification`)

**Heading level mapping between CLAUDE.md and context files:**

| CLAUDE.md | PRD.md / BUG.md |
|-----------|------------------------|
| `## Common` (category) | `# Common` |
| `## UI/UX Standards` (module under Common) | `## UI/UX Standards` |
| `## System Module` (category) | `# System Module` |
| `### User` (module under System) | `## User` |
| `## Business Module` (category) | `# Business Module` |
| `### Location Information` (module under Business) | `## Location Information` |

In CLAUDE.md, `## Common` is a category and `## UI/UX Standards` is listed as a standalone `##` heading right after it (same level). In context files, `UI/UX Standards` is a `##` under `# Common`. The skill must treat `UI/UX Standards` as a module belonging to the `Common` category.

## Workflow

### Step 1: Read CLAUDE.md Canonical Structure

1. Read the project root `CLAUDE.md`
2. Find the `# Domains` section
3. Extract the ordered list of module categories and their child modules:

```
Common:
  - UI/UX Standards

System Module:
  - User
  - Notification
  - Audit Trail
  - Document Management

Business Module:
  - Location Information
  - Corridor
  - Recruitment Step
  - Employer
  - Recruitment Agent
  - Industrial Classification
  - Occupation Classification
  - Job Demand
  - Candidate Registration
```

**Parsing rules for CLAUDE.md:**
- Module categories are `##` headings that are one of: `Common`, `System Module`, `Business Module`
- Under `## Common`, the next `##` heading(s) before `## System Module` are modules belonging to Common (e.g., `## UI/UX Standards`)
- Under `## System Module` and `## Business Module`, modules are `###` headings
- Stop parsing at the next `#` heading that is not part of Domains (e.g., `# Folder structure`)

### Step 2: Discover Application Context Folders

1. Glob for all `*/context/PRD.md` files at the project root
2. Glob for all `*/context/BUG.md` files at the project root
3. Build a list of unique application folders that have at least one of these files
4. For each application, note which files exist (PRD.md, BUG.md, or both)

### Step 3: Parse Context File Module Structures

For each discovered `PRD.md` and `BUG.md`, extract the module structure:

**Parsing rules for PRD.md:**
- Module categories are `#` headings: `# Common`, `# System Module`, `# Business Module`
- Modules are `##` headings under each category (e.g., `## User`, `## Notification`)
- Everything before `# Common` is preamble/context (preserve as-is)
- Record the exact line content and position of each category and module

**Parsing rules for BUG.md:**
- Same heading structure as PRD.md (`#` for categories, `##` for modules)
- Typically has less content per module (just bug reports or empty sections)

### Step 4: Phase 1 — Detect Rogue Modules

Compare every module found in every context file against the canonical list from CLAUDE.md:

1. For each module found in any `PRD.md` or `BUG.md`:
   - Check if the module name exists in the canonical list (case-insensitive match)
   - If a module exists in a context file but NOT in CLAUDE.md, record it as a **rogue module**
2. If ANY rogue modules are found:
   - Print a clear error report listing each rogue module, the file it was found in, and the category it was found under
   - Suggest the user either add the module to CLAUDE.md or remove it from the context file
   - **STOP — do not proceed to Phase 2**

**Report format:**
```
## Phase 1: Rogue Module Detection

FAILED — Found modules in context files that do not exist in CLAUDE.md:

| Application | File | Category | Rogue Module |
|-------------|------|----------|--------------|
| hub_middleware | PRD.md | System Module | Activities |
| hub_middleware | BUG.md | System Module | Activities |

Action required:
- Add "Activities" to CLAUDE.md under "## System Module", OR
- Remove "## Activities" from the affected files

Re-run /util-modulesync after resolving.
```

3. If NO rogue modules are found, print success and proceed to Phase 2:
```
## Phase 1: Rogue Module Detection

PASSED — All modules in context files exist in CLAUDE.md.

Proceeding to Phase 2...
```

### Step 5: Phase 2 — Sync Missing Modules

For each application context file (`PRD.md` and `BUG.md`), compare against the canonical structure and add any missing module categories or modules.

#### 5a. Determine Insertion Content

**For PRD.md**, a new module section looks like:

```markdown
## <Module Name>

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

A missing module category header looks like:
```markdown
# <Category Name>
```

**For BUG.md**, a new module section looks like:

```markdown
## <Module Name>
[v1.0.0]

---
```

A missing module category header looks like:
```markdown
# <Category Name>
```

#### 5b. Insertion Rules

1. **Preserve all existing content.** Never modify, delete, or reorder existing sections or their content.
2. **Insert missing modules in the correct position** according to the canonical order from CLAUDE.md.
3. **Insert missing category headers** if a category is entirely absent.
4. If a module category exists but is missing some modules:
   - Find the correct insertion point based on the canonical order
   - Insert the new module section between existing modules to maintain canonical order
   - Ensure `---` separators are present between module sections
5. If a module already exists (by name match), skip it — do not duplicate.
6. The `[v1.0.0]` version tag is always used for newly inserted sections.

#### 5c. Order Enforcement

The canonical order from CLAUDE.md must be maintained. When inserting a missing module, find its position relative to existing modules:

Example: If the canonical order under Business Module is `[Location Information, Corridor, Recruitment Step, Employer, ...]` and a file has `Location Information` and `Employer` but is missing `Corridor` and `Recruitment Step`, insert them between `Location Information` and `Employer`.

### Step 6: Output Summary

Print a summary of all changes made:

```
## Phase 2: Module Sync Summary

| Application | File | Action | Module |
|-------------|------|--------|--------|
| hub_middleware | PRD.md | Added | Activities (System Module) |
| hub_middleware | BUG.md | Added | Activities (System Module) |
| hc_adapter | PRD.md | No changes needed | — |
| hc_adapter | BUG.md | No changes needed | — |

Total: 2 files modified, 2 modules added.
```

## Important Rules

- **NEVER remove, modify, or delete existing content.** Only add new sections.
- **NEVER reorder existing sections.** Only insert new sections in the correct position.
- **Preserve all existing content, formatting, and indentation exactly.**
- Module name matching is **case-insensitive** but insertion uses the exact casing from CLAUDE.md.
- The `# Context` preamble at the top of each file must never be modified.
- Always maintain `---` horizontal rule separators between module sections.
- If a file has content under a module (user stories, bugs, etc.), that content is preserved exactly.
- Phase 1 is a hard gate — if it fails, Phase 2 must NOT execute.
