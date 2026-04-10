---
name: conductor-feature-prepare
model: claude-sonnet-4-6
effort: high
description: >
  Context artifacts preparation orchestrator — generates all context artifacts (module models,
  HTML mockups, technical specifications, test specifications) by invoking the appropriate
  sub-skills. Takes an application name (mandatory), with optional version and module filters.
  Version supports single version, comma-separated list, "all", or omit for all versions.
  When multiple versions are resolved, they are processed SEQUENTIALLY in ascending semver
  order — all artifacts for version N are fully generated before version N+1 begins.
  Use this skill when the user asks to "prepare artifacts", "generate context", "prepare for
  development", "generate models and specs", "create mockups and specs", or any request to
  systematically generate all context artifacts from user stories before implementation begins.
  Also trigger when user says "resume preparation" to continue from where a previous session
  left off.
---

# Feature Conductor — Prepare

Context artifacts preparation orchestrator — generates all context artifacts (module models,
HTML mockups, technical specifications, test specifications) needed before code implementation
can begin.

## Ralph Loop Integration (AUTO-START — MANDATORY)

This skill AUTOMATICALLY starts a Ralph Loop to ensure complete artifact generation across all steps.
Without Ralph Loop, the preparation may stop prematurely due to context window limits, API
usage limits, or the agent incorrectly concluding work is "done enough". Ralph Loop ensures
the same prompt is re-fed after each session exit, and the agent picks up where it left off.

### FIRST ACTION: Start Ralph Loop

**BEFORE doing anything else** (before Phase 0, before reading any files), you MUST invoke the
Ralph Loop skill using the Skill tool. This is a blocking requirement — do NOT proceed with
any work until Ralph Loop is active.

**Invoke this immediately:**
```
Skill(skill: "ralph-loop:ralph-loop", args: "/conductor-feature-prepare <application> [version:<version>] [module:<module>] --completion-promise \"ALL ARTIFACTS GENERATED\" --max-iterations 20")
```

Replace `<application>` and optional arguments with the actual arguments provided by the user.

**Example:** If the user invokes:
```
/conductor-feature-prepare mainapp
```

Then invoke:
```
Skill(skill: "ralph-loop:ralph-loop", args: "/conductor-feature-prepare mainapp --completion-promise \"ALL ARTIFACTS GENERATED\" --max-iterations 20")
```

If the user provides optional arguments:
```
/conductor-feature-prepare mainapp version:v2 module:user
```

Then invoke:
```
Skill(skill: "ralph-loop:ralph-loop", args: "/conductor-feature-prepare mainapp version:v2 module:user --completion-promise \"ALL ARTIFACTS GENERATED\" --max-iterations 20")
```

After Ralph Loop is active, proceed with Phase 0 (Resume Check) and continue normally.

### How It Works

1. This skill auto-starts a Ralph Loop with the orchestrator prompt as the loop body
2. On each iteration, the agent checks which artifacts have already been generated
3. The agent generates one or more artifacts until context runs out or all are done
4. When the agent tries to exit, the Ralph Loop stop hook re-feeds the same prompt
5. The next iteration resumes from where the last one left off
6. When ALL artifacts are generated, the agent outputs the completion promise to exit the loop

### Completion Promise

When ALL artifacts have been generated (or confirmed to already exist), output the following
promise tag to signal the Ralph Loop that preparation is finished:

```
<promise>ALL ARTIFACTS GENERATED</promise>
```

**CRITICAL**: Only output this promise when EVERY artifact step has completed:
- User story items tagged
- Module models generated
- HTML mockups generated
- Technical specifications generated
- Test specifications generated

Do NOT output the promise prematurely. Do NOT output it to escape the loop.

### Iteration Awareness

At the START of every iteration (including the first), the agent MUST:
1. Check if Ralph Loop is already active (if `.claude/ralph-loop.local.md` exists, skip re-invoking)
2. Check which artifacts already exist in the `<app_folder>/context/` subfolders
3. Skip steps for artifacts that already exist (unless version increment mode)
4. Resume from the first incomplete step
5. If ALL artifacts exist, output the completion promise and stop

## Inputs

The skill expects these arguments:

```
/conductor-feature-prepare <application> [version:<version>] [module:<module>]
```

| Argument | Required | Example | Description |
|----------|----------|---------|-------------|
| `<application>` | Yes | `mainapp` | Application name to locate the context folder |
| `version:<version>` | No | `version:v2` or `version:v1,v2` or `version:all` | Filter user stories and artifacts by version. Supports single version, comma-separated list, `all`, or omit for all versions. Multiple versions are processed sequentially in ascending semver order |
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

### Version Resolution

The `version:` argument supports four forms:

| Form | Example | Behavior |
|------|---------|----------|
| Single version | `version:v2` | Process only v2 |
| Comma-separated list | `version:v1,v2,v3` | Process each version sequentially in ascending semver order |
| Explicit all | `version:all` | Discover all versions from PRD.md, process sequentially in ascending semver order |
| Omitted | _(no version arg)_ | Same as `version:all` |

#### Version Discovery

When `version:all` or omitted:
1. Scan PRD.md for all `[vX.Y.Z]` version tags across all module sections
2. Collect unique versions
3. Sort in ascending semantic version order (v1.0.0 < v1.0.1 < v1.1.0 < v2.0.0)
4. This becomes the ordered version list for sequential processing

#### Sequential Version Processing Rule

**Versions are ALWAYS processed one at a time, in ascending semver order.** All artifacts for
version N must be fully generated (or confirmed to already exist) before version N+1 begins.
This ensures:
- Module models from earlier versions are in place before later version models are layered on
- Mockups, specifications, and test specs build incrementally on prior version artifacts
- Each version's artifacts reflect the cumulative state up to that version

## Pre-Requisite: Project Information from CLAUDE.md (MANDATORY)

**CLAUDE.md is automatically loaded into context** at the start of every session. It contains
project details, infrastructure paths, credentials, and configuration. You do NOT need to read
it manually — the information is already available in your context.

## Redo/Redevelop Guard

This guard prevents accidental re-execution of already-completed work while allowing
incremental processing of new versions. It uses a **partition and filter** approach.

1. Read `<app_folder>/CHANGELOG.md` (the application folder's changelog). If it does not exist, skip this check.
2. Resolve the version list (see Version Resolution).
3. **Partition** the resolved versions into two groups:
   - `completed_versions` — versions that have a matching `conductor-feature-prepare` entry
     in `<app_folder>/CHANGELOG.md`
   - `new_versions` — versions with NO matching entry
4. **Decision**:

   | `new_versions` | `completed_versions` | Artifacts exist? | Action |
   |---------------|---------------------|-----------------|--------|
   | Not empty | Any (including empty) | Yes (expected — prior versions built them) | **Proceed with `new_versions` only** — filter out completed versions from the processing list. Existing artifacts are the base for version increment. |
   | Not empty | Any | No | **Proceed with all resolved versions** — no prior artifacts, start from scratch. |
   | Empty | Not empty | Yes | **STOP**. Print: `"All requested versions ({list}) for {application} already have context artifacts (recorded in <app_folder>/CHANGELOG.md) and they still exist. To redo, first delete the existing context artifact folders (model/, mockup/, specification/, test/), then re-run this skill."` |
   | Empty | Not empty | No | **Proceed with all resolved versions** — artifacts were cleaned up, this is a legitimate redo. |

5. **Update the resolved version list** to contain only the versions that will be processed
   (either `new_versions` or all versions for redo). This filtered list is what the Sequential
   Version Loop in Phase 1 will iterate over.

## Workflow

### Phase 0: Resume Check (Runs Every Ralph Loop Iteration)

This phase runs at the START of every iteration, including the first. In a Ralph Loop,
each iteration begins fresh with the same prompt, so the agent MUST check existing artifacts
to understand what has already been completed.

0. **Auto-Start Ralph Loop** — Check if `.claude/ralph-loop.local.md` exists. If it does NOT
   exist, Ralph Loop is not yet active. Invoke it NOW using the Skill tool:
   ```
   Skill(skill: "ralph-loop:ralph-loop", args: "<the full /conductor-feature-prepare invocation with args> --completion-promise \"ALL ARTIFACTS GENERATED\" --max-iterations 20")
   ```
   If `.claude/ralph-loop.local.md` already exists, Ralph Loop is active — skip this step.

1. **Use project information from CLAUDE.md (already in context)** — extract database type,
   design system, technology stack, and all infrastructure details.
2. **Resolve the version list** using the Version Resolution rules:
   - Single version → `[v2]`
   - Comma-separated → parse and sort ascending by semver → `[v1, v2, v3]`
   - `all` or omitted → scan PRD.md for ALL `[vX.Y.Z]` tags, deduplicate, sort ascending
3. Check which artifacts already exist:
   - Check `<app_folder>/context/model/` for module model files
   - Check `<app_folder>/context/mockup/` for HTML mockup files
   - Check `<app_folder>/context/specification/` for spec files
   - Check `<app_folder>/context/test/` for test spec files
4. If ALL artifacts exist AND only a single version was resolved AND this is NOT a version increment:
   - Output `<promise>ALL ARTIFACTS GENERATED</promise>` and stop
5. If multiple versions were resolved, proceed to Phase 1 — the version loop will determine
   which versions still need processing (see Phase 1 Sequential Version Loop)
6. Otherwise, proceed to Phase 1, starting from the first step with missing artifacts

### Phase 1: Context Artifacts Generation

This phase generates all required context artifacts before code implementation begins. Each step
checks if the artifact already exists — if it does, the step is skipped. This ensures the phase
is idempotent and can be resumed safely.

**IMPORTANT**: If `module` argument was provided, pass it through to each sub-skill invocation
so that only the relevant subset of artifacts is generated.

#### Sequential Version Loop

When multiple versions are resolved (list, `all`, or omitted), Phase 1 wraps Steps 1.1–1.9 in
a **version loop**:

```
resolved_versions = [v1.0.0, v1.0.1, v1.0.2]  # ascending semver order

For each version in resolved_versions:
  1. Run Steps 1.1–1.9 with this single version as the version argument
  2. For the FIRST version: artifacts are generated from scratch (Steps 1.2–1.9)
  3. For SUBSEQUENT versions: artifacts already exist from the prior version — operate in
     Version Increment Mode (update, not skip)
  4. All artifacts for this version must be complete before moving to the next version
```

When only a single version is resolved, the loop runs once — no behavioral difference from
the original single-version flow.

#### Version Increment Mode

When artifacts exist from a previous version but the current version in the loop has new/changed
user stories, Steps 1.4–1.9 operate in **update mode**:

- **DO NOT skip steps just because artifacts already exist.** The existing artifacts are from a
  previous version and need to be updated with the current version's changes.
- Re-invoke ALL sub-skills (modelgen, mockgen, specgen, testgen) with BOTH the current `version`
  AND `module` arguments so they can update the existing artifacts incrementally.
- The sub-skills are responsible for reading the `[v<version>]` sections in PRD.md and
  updating their output files accordingly (appending new fields, modifying existing diagrams,
  updating specs and test scenarios).

#### Bug Regression Awareness (Redo/Redevelop Scenario)

PRD.md modules may contain a `### Bug` section listing previously fixed bugs (e.g.,
`[BUG-024] Fixed Message ID link...`). These entries represent validated fixes from prior
development cycles. When generating or updating artifacts, sub-skills MUST treat bug fix
entries as supplementary requirements:

- **modelgen-***: If a bug fix involved model changes (new fields, changed relationships),
  the model output must reflect the corrected structure.
- **mockgen-tailwind**: If a bug fix involved UI changes (layout, navigation, styling),
  the mockup must reflect the corrected design.
- **specgen-***: If a bug fix involved logic changes (event listeners, processing flow,
  validation), the specification must include the corrected behavior.
- **testgen-functional**: Bug fix descriptions should inform test scenarios to ensure
  regression coverage — each bug fix should have at least one corresponding test assertion.

This ensures that when an application is redeveloped from scratch, previously reported bugs
do not reappear in the new implementation.

#### Step 1.1: Resolve Application Folder

1. Match the `<application>` argument against root-level folders (same logic as Input Resolution)
2. The resolved `<app_folder>` is the root for all context artifacts
3. Verify `<app_folder>/context/PRD.md` exists — if not, stop with error

#### Step 1.2: Tag User Stories

Tag all untagged items in PRD.md with unique ID codes before any artifact generation begins.
This ensures all user stories, NFRs, constraints, and references have stable IDs that downstream
artifacts (module models, specs, tests) can reference for traceability.

**This step runs ONCE before the version loop starts** (not per-version), because tagging
applies to all items regardless of version.

1. Determine the version label to pass:
   - If processing a single version → use that version (e.g., `v2`)
   - If processing multiple versions → use the HIGHEST version in the resolved list
   - If no `version` argument (all) → use `v1` as the default
2. Invoke the tagging skill:
   - `Skill(skill: "util-ustagger", args: "<application> <version>")`
   - Example: `Skill(skill: "util-ustagger", args: "mainapp v1")`
3. Wait for the skill to complete. It will tag untagged items in-place in PRD.md.
4. This step is idempotent — if all items are already tagged, the skill will report no changes.

#### Step 1.3: Infer Database Type

1. **Check PRD.md `# Architecture Principle` section first** (primary signal):
   - If Architecture Principle explicitly mentions "document based database", "document-oriented", "NoSQL", "MongoDB", or similar → **nosql**
   - If Architecture Principle explicitly mentions "relational database", "SQL", "PostgreSQL", "MySQL", or similar → **relational**
2. **Fallback to CLAUDE.md** (if Architecture Principle is absent or does not mention database type):
   - Read application details from CLAUDE.md (already in context)
   - Identify the database from the application's dependency list:
     - If the application depends on MySQL, PostgreSQL, or another relational database → **relational**
     - If the application depends on MongoDB or another NoSQL database → **nosql**
3. Record the inferred database type for the next step

#### Step 1.4: Generate Module Model

1. **Check if artifacts exist**: List files in `<app_folder>/context/model/`
   - If `.md` or `.json` files already exist AND this is NOT a version increment → **SKIP** this step
   - If this IS a version increment, **proceed** to re-invoke the skill with version/module args
2. Invoke the appropriate model generation skill:
   - If database type is **relational** → `Skill(skill: "modelgen-relational", args: "<app_folder> [version:<version>] [module:<module>]")`
   - If database type is **nosql** → `Skill(skill: "modelgen-nosql", args: "<app_folder> [version:<version>] [module:<module>]")`
3. Wait for the skill to complete. The output will be module model files in `<app_folder>/context/model/`

#### Step 1.5: Infer Design System

1. **Check PRD.md `# Design System` section first** (primary signal):
   - If the section exists and references a `DESIGN_SYSTEM.md` file (e.g., `[DESIGN_SYSTEM.md](reference/DESIGN_SYSTEM.md)`), resolve the file path relative to PRD.md
   - Record the resolved design system file path for downstream skills (mockgen, specgen)
   - The referenced file contains design tokens (colors, typography, components) that sub-skills will use
2. **Fallback to CLAUDE.md** (if Design System section is absent or has no reference):
   - Read application details from CLAUDE.md (already in context)
   - Identify the design system / CSS framework:
     - Look for mentions of Tailwind CSS, Bootstrap, Material UI, etc.
     - If no design system is explicitly identified → default to **Tailwind CSS**
3. Record the inferred design system for the next step

#### Step 1.6: Generate HTML Mockups

1. **Check if artifacts exist**: List files in `<app_folder>/context/mockup/`
   - If `.html` files already exist AND this is NOT a version increment → **SKIP** this step
   - If this IS a version increment, **proceed** to re-invoke the skill with version/module args
2. Determine the appropriate mockup generator:
   - Check available skills matching `mockgen-*` pattern
   - For Tailwind CSS → `Skill(skill: "mockgen-tailwind", args: "<app_folder> [version:<version>] [module:<module>]")`
   - Default to `mockgen-tailwind` if no specific generator matches the design system
3. Wait for the skill to complete. The output will be HTML mockup files in `<app_folder>/context/mockup/`

#### Step 1.7: Infer Technology Stack

1. **Check PRD.md `# Architecture Principle` section first** (primary signal):
   - If Architecture Principle explicitly mentions a framework (e.g., "Spring Boot", "Laravel"), use it
   - If it mentions a view engine (e.g., "JTE", "Blade"), use it
   - If it mentions frontend tooling (e.g., "HTMX", "Alpine.js", "Vite"), use it
   - If it mentions architectural style (e.g., "monolithic", "REST API"), use it to disambiguate specgen variant
2. **Fallback to CLAUDE.md** (if Architecture Principle is absent or incomplete):
   - Read application details from CLAUDE.md (already in context)
   - Identify the technology stack:
     - Framework (Laravel, Spring Boot, etc.)
     - ORM (Eloquent, JPA/Hibernate, etc.)
     - View engine (Blade, JTE, etc.)
     - Frontend enhancement (HTMX, Alpine.js, etc.)
3. Record the inferred technology stack for the next step

#### Step 1.8: Generate Technical Specification

1. **Check if artifacts exist**: List files in `<app_folder>/context/specification/`
   - If `.md` files already exist AND this is NOT a version increment → **SKIP** this step
   - If this IS a version increment, **proceed** to re-invoke the skill with version/module args
2. Determine the appropriate specification generator:
   - Check available skills matching `specgen-*` pattern
   - For Laravel + Eloquent + Blade + HTMX → `Skill(skill: "specgen-laravel-eloquent-bladehtmx", args: "<app_folder> [version:<version>] [module:<module>]")`
   - For Spring Boot + JPA + JTE + HTMX → `Skill(skill: "specgen-spring-jpa-jtehtmx", args: "<app_folder> [version:<version>] [module:<module>]")`
   - Match the skill name against the inferred technology stack components
3. Wait for the skill to complete. The output will be specification files in `<app_folder>/context/specification/`

#### Step 1.9: Generate Test Specification

1. **Check if artifacts exist**: List files in `<app_folder>/context/test/`
   - If `.md` files already exist AND this is NOT a version increment → **SKIP** this step
   - If this IS a version increment, **proceed** to re-invoke the skill with version/module args
2. Invoke the test specification generator:
   - `Skill(skill: "testgen-functional", args: "<app_folder> [version:<version>] [module:<module>]")`
3. Wait for the skill to complete. The output will be test specification files in `<app_folder>/context/test/`

#### Phase 1 Completion

After all artifacts for ALL versions in the resolved version list are generated (or confirmed
to already exist):
1. Output a summary of what was generated/skipped per version
2. Output `<promise>ALL ARTIFACTS GENERATED</promise>` to signal Ralph Loop completion

**IMPORTANT — Version loop completion**: The completion promise is ONLY output after the LAST
version in the resolved list has been fully processed. Do NOT output it after completing an
intermediate version — proceed to the next version immediately.

## Critical Rules

1. **CLAUDE.md is the source of truth** — CLAUDE.md is automatically loaded into context. Use
   it for all project details, database types, technology stacks, and infrastructure configuration.

2. **Idempotent execution** — Each step checks if artifacts already exist before generating.
   This ensures the skill can be safely re-run or resumed without duplicating work.

3. **Version increment awareness** — When processing multiple versions sequentially, artifacts
   from the prior version already exist when the next version begins. ALL sub-skills must be
   re-invoked in update mode for each subsequent version.

4. **Pass through filters** — The current version from the version loop AND the `module`
   argument (if provided) MUST be passed to every sub-skill invocation. Each sub-skill
   receives a SINGLE version at a time, never a list.

5. **Sequential version order** — When multiple versions are resolved, they are processed
   in ascending semver order. All artifacts for version N must complete before version N+1
   begins. Never process versions in parallel or merge them into a single invocation.

6. **Context window awareness (Ralph Loop handles recovery)** — If approaching context limits,
   the Ralph Loop will automatically re-feed the prompt, and the next iteration will resume
   from the first step with missing artifacts.

7. **Auto-start Ralph Loop** — When this skill is triggered, the FIRST action (Phase 0, Step 0)
   MUST be to check if Ralph Loop is active (`.claude/ralph-loop.local.md` exists). If not active,
   invoke the Ralph Loop skill. Do NOT proceed with any work until Ralph Loop is confirmed active.

8. **AUTONOMOUS CONTINUATION — NEVER STOP BETWEEN STEPS OR VERSIONS.** This is an orchestrator
   skill that runs a multi-step pipeline (Steps 1.1 through 1.9) across potentially multiple
   versions. When a sub-skill (e.g., util-ustagger, modelgen-relational, mockgen-tailwind,
   specgen-*, testgen-functional) completes and returns control, you MUST immediately proceed
   to the NEXT step in the pipeline. When all steps for the current version are done, you MUST
   immediately proceed to the NEXT version. Do NOT stop, do NOT wait for user input, do NOT
   treat a sub-skill or version completion as the end of your task. The task is only complete
   when ALL steps for ALL versions have been executed and the completion promise
   `<promise>ALL ARTIFACTS GENERATED</promise>` has been output. **Stopping after any
   intermediate step or version is a bug in execution.**
