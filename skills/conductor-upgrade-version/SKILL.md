---
name: conductor-upgrade-version
model: claude-sonnet-4-6
effort: high
description: >
  Version upgrade orchestrator — orchestrates both feature development (conductor-feature-develop)
  and bug fixing (conductor-defect) for a new version in a single session. First implements new
  features from user stories, then resolves all bugs from BUG.md — ensuring the upgraded version
  is both feature-complete and stable. Takes an application name (mandatory), version (optional —
  supports single version, comma-separated list, "all", or omit for all), and optional module
  filter. When multiple versions are resolved, they are processed SEQUENTIALLY in ascending
  semver order — both Phase A (features) and Phase B (bugs) for version N are fully completed
  before version N+1 begins. Use this skill when the user asks to "upgrade version", "version
  upgrade", "upgrade to new version", "new version with bugs and features", "upgrade and fix
  bugs", or any request to systematically implement new features and fix bugs for a version
  upgrade. Also trigger when user says "resume upgrade" to continue from where a previous
  session left off using UPGRADE_MASTER.md progress file.
---

# Version Upgrade Conductor

Orchestrates a full version upgrade by running feature development (conductor-feature-develop)
followed by bug fixing (conductor-defect) in sequence. Ensures the upgraded version is both
feature-complete (all modules implemented) and stable (all bugs resolved). Development runs
first so that code exists before bug reproduction and fixing is attempted.

## Ralph Loop Integration (AUTO-START — MANDATORY)

This skill AUTOMATICALLY starts a Ralph Loop to ensure the complete upgrade workflow across
both phases. Without Ralph Loop, the upgrade may stop prematurely due to context window limits,
API usage limits, or the agent incorrectly concluding work is "done enough". Ralph Loop ensures
the same prompt is re-fed after each session exit, and the agent picks up where it left off
using the UPGRADE_MASTER.md tracking file.

### FIRST ACTION: Start Ralph Loop

**BEFORE doing anything else** (before Phase 0, before reading any files), you MUST invoke the
Ralph Loop skill using the Skill tool. This is a blocking requirement — do NOT proceed with
any work until Ralph Loop is active.

**Invoke this immediately:**
```
Skill(skill: "ralph-loop:ralph-loop", args: "/conductor-upgrade-version <application> [version:<version>] [module:<module>] --completion-promise \"VERSION UPGRADE COMPLETED\" --max-iterations 150")
```

Replace `<application>` and optional arguments with the actual arguments provided by the user.

**Example:** If the user invokes:
```
/conductor-upgrade-version mainapp version:v2.0.0
```

Then invoke:
```
Skill(skill: "ralph-loop:ralph-loop", args: "/conductor-upgrade-version mainapp version:v2.0.0 --completion-promise \"VERSION UPGRADE COMPLETED\" --max-iterations 150")
```

If the user provides a version list:
```
/conductor-upgrade-version mainapp version:v1.0.3,v1.0.4,v2.0.0
```

Then invoke:
```
Skill(skill: "ralph-loop:ralph-loop", args: "/conductor-upgrade-version mainapp version:v1.0.3,v1.0.4,v2.0.0 --completion-promise \"VERSION UPGRADE COMPLETED\" --max-iterations 150")
```

If the user omits version (all versions):
```
/conductor-upgrade-version mainapp
```

Then invoke:
```
Skill(skill: "ralph-loop:ralph-loop", args: "/conductor-upgrade-version mainapp --completion-promise \"VERSION UPGRADE COMPLETED\" --max-iterations 150")
```

If the user provides optional arguments:
```
/conductor-upgrade-version mainapp version:v2.0.0 module:user
```

Then invoke:
```
Skill(skill: "ralph-loop:ralph-loop", args: "/conductor-upgrade-version mainapp version:v2.0.0 module:user --completion-promise \"VERSION UPGRADE COMPLETED\" --max-iterations 150")
```

After Ralph Loop is active, proceed with Phase 0 (Resume Check) and continue normally.

### How It Works

1. This skill auto-starts a Ralph Loop with the upgrade orchestrator prompt as the loop body
2. On each iteration, the agent reads UPGRADE_MASTER.md to determine which phase is active
3. Phase A (Feature Development): delegates to conductor-feature-develop — agent implements modules until all are completed
4. Phase B (Bug Fixing): delegates to conductor-defect — agent fixes bugs until all are resolved
5. When the agent tries to exit, Ralph Loop re-feeds the same prompt automatically
6. The next iteration resumes from where the last one left off (tracked in UPGRADE_MASTER.md)
7. When BOTH phases are COMPLETED, the agent outputs the completion promise to exit the loop

### Completion Promise

When BOTH phases — bug fixing AND feature development — are fully completed, output the
following promise tag to signal the Ralph Loop that the version upgrade is done:

```
<promise>VERSION UPGRADE COMPLETED</promise>
```

**CRITICAL**: Only output this promise when:
- ALL versions in the Version Processing Order table have status `COMPLETED`
- For each version: Phase A is `COMPLETED` and Phase B has terminal status (`COMPLETED` or `SKIPPED`)
- Deployment artifacts (Phase 4) have been generated after the last version

Do NOT output the promise prematurely. Do NOT output it to escape the loop.

### Iteration Awareness

At the START of every iteration (including the first), the agent MUST:
1. Check if Ralph Loop is already active (if `.claude/ralph-loop.local.md` exists, skip re-invoking)
2. If NOT active, silently invoke Ralph Loop as described above
3. Read UPGRADE_MASTER.md to determine the current version and phase status
4. Resume from the active version and phase — do NOT restart completed versions or phases
5. If ALL versions are COMPLETED (all phases for all versions done), output the completion promise and stop

### Never Stop Prematurely

Within a Ralph Loop iteration, the agent MUST:
- Continue working through the current phase until context limits force a stop
- After Phase A (features) completes for the current version, IMMEDIATELY transition to Phase B (bugs)
- After Phase B (bugs) completes for the current version, IMMEDIATELY advance to the next version
- Do NOT stop between phases or between versions "to let the user review" — Ralph Loop handles iteration
- Do NOT output the completion promise until ALL versions have BOTH phases verified complete
- If approaching context limits mid-phase, save progress to UPGRADE_MASTER.md so the
  next iteration can resume from the exact point

## Inputs

The skill expects these arguments:

```
/conductor-upgrade-version <application> [version:<version>] [module:<module>]
```

| Argument | Required | Example | Description |
|----------|----------|---------|-------------|
| `<application>` | Yes | `hub_middleware` | Application name to locate the context folder |
| `version:<version>` | No | `version:v2.0.0` or `version:v1.0.3,v2.0.0` or `version:all` | Target version(s) for the upgrade. Supports single version, comma-separated list, `all`, or omit for all versions. Multiple versions are processed sequentially in ascending semver order |
| `module:<module>` | No | `module:location-information` | Filter both bugs and features by module |

### Input Resolution

The application name is matched against root-level application folders:
1. Strip any leading `<number>_` prefix from folder names (e.g., `1_hub_middleware` -> `hub_middleware`)
2. Match case-insensitively against the provided application name
3. Accept snake_case, kebab-case, or title-case input
4. If no match found, list available applications and stop

### Auto-Resolved Paths

| File | Resolved Path |
|------|---------------|
| BUG.md | `<app_folder>/context/BUG.md` |
| PRD.md | `<app_folder>/context/PRD.md` |
| Bug Tracking Output | `<app_folder>/context/bug/` |
| Module Models | `<app_folder>/context/model/` |
| HTML Mockups | `<app_folder>/context/mockup/` |
| Specifications | `<app_folder>/context/specification/` |
| Test Specs | `<app_folder>/context/test/` |
| Development Output | `<app_folder>/context/develop/` |
| Upgrade Tracking | `<app_folder>/context/develop/UPGRADE_MASTER.md` |

### Argument Forwarding

This skill forwards arguments to its sub-skills. When processing multiple versions sequentially,
each sub-skill invocation receives a **single version** (the current version in the loop):

| Sub-Skill | Forwarded Arguments |
|-----------|-------------------|
| `conductor-feature-develop` | `<application> version:<current-version> [module:<module>]` |
| `conductor-defect` | `<application> version:<current-version> [module:<module>]` |

The `source:` argument for `conductor-feature-develop` is NOT accepted by this skill.
It defaults to the resolved `<app_folder>` as per conductor-feature-develop's default behavior.

**IMPORTANT**: Sub-skills always receive a SINGLE version, never a list. The version loop
in this skill handles the sequential iteration — sub-skills are unaware of multi-version
processing.

### Version Resolution

The `version:` argument supports four forms:

| Form | Example | Behavior |
|------|---------|----------|
| Single version | `version:v2.0.0` | Process only v2.0.0 |
| Comma-separated list | `version:v1.0.3,v1.0.4,v2.0.0` | Process each version sequentially in ascending semver order |
| Explicit all | `version:all` | Discover all versions from PRD.md and BUG.md, process sequentially in ascending semver order |
| Omitted | _(no version arg)_ | Same as `version:all` |

#### Version Discovery

When `version:all` or omitted:
1. Scan BOTH PRD.md and BUG.md for all `[vX.Y.Z]` version tags
2. Collect unique versions from both files (union)
3. Sort in ascending semantic version order (v1.0.0 < v1.0.1 < v1.1.0 < v2.0.0)
4. This becomes the ordered version list for sequential processing

#### Sequential Version Processing Rule

**Versions are ALWAYS processed one at a time, in ascending semver order.** Both Phase A
(feature development) and Phase B (bug fixing) for version N must fully complete before
version N+1 begins. This ensures:
- Features are implemented before bug reproduction and fixing is attempted
- Bug fixes target code that actually exists (developed in Phase A)
- Feature implementations build incrementally on prior version work
- The codebase is progressively upgraded version by version

## Pre-Requisite: Project Information from CLAUDE.md (MANDATORY)

**CLAUDE.md is automatically loaded into context** at the start of every session. It contains
project details, infrastructure paths, credentials, and configuration. You do NOT need to read
it manually — the information is already available in your context.

**Before executing ANY tool command** (Maven build, Spring Boot run, database CLI, Keycloak CLI,
Playwright test, npm start, etc.), use the following from CLAUDE.md (already in context):

- **JDK path** — Use the exact `JAVA_HOME` path specified in CLAUDE.md
- **Maven path** — Use the exact Maven binary path specified in CLAUDE.md
- **Database credentials** — Host, port, username, password
- **Keycloak configuration** — Host, admin credentials, CLI path
- **Any other infrastructure details** — Ports, URLs, connection strings

**WHY**: CLAUDE.md contains the actual system paths, credentials, and configuration for the
developer's machine. Every shell command MUST use the values from CLAUDE.md.

## PRD.md Extended Sections

### High Level Process Flow Awareness

If PRD.md contains a `# High Level Process Flow` section, be aware that process flow changes between versions may affect both bug fixing and feature development:
- **Phase A (Feature Development)**: Process flows serve as implementation blueprints for message-driven modules. conductor-feature-develop uses them to implement flow steps in the correct order.
- **Phase B (Bug Fixing)**: Process flows provide step-by-step debugging context for message-driven bugs. conductor-defect uses them to trace which flow step is failing.
- If process flows have changed between the previous version and the target version (new steps, modified steps), both phases should be aware of these changes.

---

## Pre-Requisites

Before the upgrade can proceed, verify the following exist:

### For Phase A (Feature Development)
- Context artifacts MUST exist (from conductor-feature-prepare):
  - `<app_folder>/context/model/` — Module models
  - `<app_folder>/context/mockup/` — HTML mockups
  - `<app_folder>/context/specification/` — Technical specifications
  - `<app_folder>/context/test/` — Test specifications
- If context artifacts are missing, **stop and inform the user** to run `conductor-feature-prepare` first

### For Phase B (Bug Fixing)
- `<app_folder>/context/BUG.md` — Bug reports for the target version(s)
- Phase B applicability is checked **per version** during the version loop, AFTER Phase A
  (feature development) completes:
  - If BUG.md does not exist OR has no bugs for the current version being processed, Phase B
    is **skipped** for that version (marked SKIPPED in the Version Processing Order table)
  - Phase B may be skipped for some versions but not others

### Bug Regression Awareness (Redo/Redevelop Scenario)
- PRD.md modules may contain `### Bug` sections listing previously fixed bugs from prior
  development cycles. Both Phase A (conductor-feature-develop) and Phase B (conductor-defect)
  must take these into account:
  - **Phase A (conductor-feature-develop)**: When implementing modules, the `### Bug` section
    entries are treated as supplementary requirements to prevent the same bugs from reappearing
    during redevelopment. conductor-feature-develop reads these entries alongside user stories,
    NFRs, and constraints for each module. This means many previously reported bugs are
    proactively prevented during development, reducing the number of bugs that Phase B
    needs to address.
  - **Phase B (conductor-defect)**: Bug fixes recorded in PRD.md `### Bug` sections are
    already-resolved issues from past versions — they do not need re-fixing but inform context.
    Phase B focuses on bugs in BUG.md that were NOT already addressed by the Bug Regression
    Awareness in Phase A.

## UPGRADE_MASTER.md — Central Tracking File

The UPGRADE_MASTER.md file tracks the overall upgrade progress. It is created during Phase 1
and updated throughout the workflow. Location: `<app_folder>/context/develop/UPGRADE_MASTER.md`

### Template

```markdown
# Version Upgrade: {application}

> Upgrade started: {date}
> Resolved Versions: {comma-separated sorted version list, e.g., "v1.0.3, v1.0.4, v2.0.0"}
> Status: {IN_PROGRESS | COMPLETED}

## Version Processing Order

| # | Version | Phase A (Features) | Phase B (Bugs) | Status | Started | Completed |
|---|---------|-------------------|---------------|--------|---------|-----------|
| 1 | v1.0.3 | NEW | NEW | NEW | - | - |
| 2 | v1.0.4 | NEW | NEW | NEW | - | - |
| 3 | v2.0.0 | NEW | NEW | NEW | - | - |

> **Processing Rule**: Both Phase A and Phase B for version N must complete before version N+1 begins.
> **Phase order**: Phase A (develop features) runs first, then Phase B (fix bugs) — code must exist before bugs can be reproduced and fixed.

## Current Version: {current version being processed}

### Phase Summary

| Phase | Name | Status | Started | Completed | Notes |
|-------|------|--------|---------|-----------|-------|
| A | Feature Development | {NEW / IN_PROGRESS / COMPLETED} | {date} | {date} | {notes} |
| B | Bug Fixing | {NEW / IN_PROGRESS / COMPLETED / SKIPPED} | {date} | {date} | {notes} |

### Phase A: Feature Development

- **Sub-Skill**: conductor-feature-develop
- **Invocation**: `/conductor-feature-develop {application} version:{current-version} [module:{module}]`
- **Tracking File**: `<app_folder>/context/develop/IMPLEMENTATION_MASTER.md`
- **Status**: {detailed status}

### Phase B: Bug Fixing

- **Sub-Skill**: conductor-defect
- **Invocation**: `/conductor-defect {application} version:{current-version} [module:{module}]`
- **Tracking File**: `<app_folder>/context/bug/BUG_MASTER.md`
- **Status**: {detailed status}

## Upgrade Log

| Timestamp | Version | Phase | Event | Details |
|-----------|---------|-------|-------|---------|
```

**IMPORTANT — Single version shortcut**: When only a single version is resolved, the Version
Processing Order table has a single row. The behavior is identical to the original single-version
flow — no extra complexity.

**IMPORTANT — Current Version section**: The `## Current Version` section is updated each time
the version loop advances. When Phase A and Phase B for the current version are both COMPLETED
(or Phase B is SKIPPED), the current version row in the Version Processing Order table is marked
`COMPLETED`, the `## Current Version` heading is updated to the next version, and the Phase
Summary table is reset to `NEW` for the next version.

## Redo/Redevelop Guard

This guard prevents accidental re-execution of already-completed work while allowing
incremental processing of new versions. It runs ONCE before the version loop starts and uses
a **partition and filter** approach.

1. Read `CHANGELOG.md` from the project root. If it does not exist, skip this check.
2. Resolve the version list (see Version Resolution).
3. **Partition** the resolved versions into two groups:
   - `completed_versions` — versions that have a matching `conductor-upgrade-version` OR
     `conductor-feature-develop` entry in CHANGELOG.md for this application
   - `new_versions` — versions with NO matching entry
4. **Decision**:

   | `new_versions` | `completed_versions` | Artifacts/code exist? | Action |
   |---------------|---------------------|----------------------|--------|
   | Not empty | Any (including empty) | Yes (expected — prior versions built them) | **Proceed with `new_versions` only** — filter out completed versions. Existing code/artifacts are the base for the next version's upgrade. |
   | Not empty | Any | No | **Proceed with all resolved versions** — no prior artifacts, start from scratch. |
   | Empty | Not empty | Yes | **STOP**. Print: `"All requested versions ({list}) for {application} were already upgraded (recorded in CHANGELOG.md) and artifacts/code still exist. To redo, first delete the existing tracking files (UPGRADE_MASTER.md, IMPLEMENTATION_MASTER.md) and source code, then re-run this skill."` |
   | Empty | Not empty | No | **Proceed with all resolved versions** — code was cleaned up, this is a legitimate redo. |

   **Artifacts/code exist check**: `<app_folder>/context/develop/UPGRADE_MASTER.md` or
   `<app_folder>/context/develop/IMPLEMENTATION_MASTER.md` exists, OR source code files exist
   in `<app_folder>/` (e.g., `pom.xml`, `composer.json`, `package.json`, or `src/` directory).

5. **Update the resolved version list** to contain only the versions that will be processed
   (either `new_versions` or all versions for redo). This filtered list is what the Version
   Processing Order table in UPGRADE_MASTER.md will track.

## Processing Workflow

### Phase 0: Resume Check (runs every Ralph Loop iteration)

1. Ensure Ralph Loop is active (check `.claude/ralph-loop.local.md`)
2. **Resolve the version list** using the Version Resolution rules:
   - Single version → `[v2.0.0]`
   - Comma-separated → parse and sort ascending by semver → `[v1.0.3, v1.0.4, v2.0.0]`
   - `all` or omitted → scan PRD.md and BUG.md for ALL `[vX.Y.Z]` tags, deduplicate, sort ascending
3. Check if `UPGRADE_MASTER.md` exists:
   - **If YES**: Read it and determine the current state:
     - Scan the **Version Processing Order** table for the FIRST version with status != `COMPLETED`
     - If such a version exists, this is the **current version**. Read its Phase Summary:
       - If Phase A is `IN_PROGRESS` or `NEW` → resume Phase A for this version (go to Phase 2)
       - If Phase A is `COMPLETED` and Phase B is `IN_PROGRESS` or `NEW` → resume Phase B for this version (go to Phase 3)
       - If BOTH phases for this version are `COMPLETED` (or Phase B is `SKIPPED`) → mark this version as
         `COMPLETED` in the Version Processing Order table, advance to the NEXT version
         (update `## Current Version`), and go to Phase 1.5 (Version Initialization) for it
     - If ALL versions are `COMPLETED` → go to Phase 4 (deployment artifacts), then Phase 5 (completion)
   - **If NO**: Proceed to Phase 1 (fresh start)

### Phase 1: Initialization

1. **Resolve application folder** using input resolution rules
2. **Resolve the version list** (if not already done in Phase 0)
3. **Verify pre-requisites**:
   - Check if context artifacts exist for feature development
   - If context artifacts are missing, **stop and inform the user**
4. **Create UPGRADE_MASTER.md** with the Version Processing Order table populated from the
   resolved version list. All versions start with status `NEW`.
5. **Set the FIRST version as the current version** in `## Current Version`
6. **Log** the initialization event
7. Proceed to Phase 1.5 (Version Initialization) for the first version

### Phase 1.5: Version Initialization (runs for each version in the loop)

This phase runs at the start of each new version in the Version Processing Order:

1. **Update `## Current Version`** in UPGRADE_MASTER.md to the current version
2. **Phase A (Feature Development) is always required** — set Phase A status = `NEW`
3. **Determine Phase B applicability for the current version**:
   - Read BUG.md and check for bugs matching the current version (and optional module filter)
   - If bugs exist → Phase B status = `NEW`
   - If no bugs for the current version → Phase B status = `SKIPPED`
4. **Reset the Phase Summary table** for the current version:
   - Phase A = `NEW`
   - Phase B = `NEW` (or `SKIPPED` if no bugs)
5. **Update the Version Processing Order table**: set current version status to `IN_PROGRESS`
6. **Log** the version initialization event
7. Proceed to Phase 2 (Feature Development)

### Phase 2: Feature Development (Phase A)

This phase delegates entirely to `conductor-feature-develop`. The delegation works as follows:

1. **Update UPGRADE_MASTER.md**: Set Phase A status to `IN_PROGRESS`, record start date
2. **Invoke conductor-feature-develop** by calling the Skill tool with the **current version only**:
   ```
   Skill(skill: "conductor-feature-develop", args: "<application> version:<current-version> [module:<module>]")
   ```
   - Pass the application, the **current version from the version loop** (single version),
     and module arguments
   - conductor-feature-develop will handle its own Ralph Loop internally — **do NOT start a nested Ralph Loop**

   **IMPORTANT — Ralph Loop Delegation**:
   When invoking conductor-feature-develop from within this skill, the conductor-feature-develop
   skill will attempt to start its own Ralph Loop. Since this skill's Ralph Loop is ALREADY
   ACTIVE, conductor-feature-develop will detect the existing `.claude/ralph-loop.local.md` and
   skip re-invoking Ralph Loop. This is the expected behavior — both skills share the same
   Ralph Loop session.

3. **Monitor completion**: After conductor-feature-develop returns or the iteration ends:
   - Check `<app_folder>/context/develop/IMPLEMENTATION_MASTER.md`
   - If ALL modules for the **current version** have status `COMPLETED`:
     - Update UPGRADE_MASTER.md: Phase A = `COMPLETED`, record completion date
     - Log the completion event
     - Proceed to Phase 3 for the **same version** (or skip if Phase B is `SKIPPED`)
   - If modules remain pending for the current version:
     - UPGRADE_MASTER.md remains Phase A = `IN_PROGRESS`
     - Ralph Loop will re-feed the prompt, and Phase 0 will resume Phase A for this version
4. **Log** progress in the Upgrade Log table (include the version column)

### Phase 3: Bug Fixing (Phase B)

This phase delegates entirely to `conductor-defect`. It runs AFTER Phase A (Feature Development)
so that code exists before bug reproduction and fixing is attempted.

1. **Update UPGRADE_MASTER.md**: Set Phase B status to `IN_PROGRESS`, record start date
2. **Invoke conductor-defect** by calling the Skill tool with the **current version only**:
   ```
   Skill(skill: "conductor-defect", args: "<application> version:<current-version> [module:<module>]")
   ```
   - Pass the application, the **current version from the version loop** (single version),
     and module arguments
   - conductor-defect will handle its own Ralph Loop internally — **do NOT start a nested Ralph Loop**

   **IMPORTANT — Ralph Loop Delegation**:
   Same as Phase A — conductor-defect will detect the existing Ralph Loop and skip
   re-invoking. Both skills share the same Ralph Loop session.

3. **Monitor completion**: After conductor-defect returns or the iteration ends:
   - Check `<app_folder>/context/bug/BUG_MASTER.md`
   - If ALL bugs for the **current version** have terminal status (`FIXED`, `CANNOT_REPRODUCE`, `HIGH_IMPACT`):
     - Update UPGRADE_MASTER.md: Phase B = `COMPLETED`, record completion date
     - **Mark the current version as `COMPLETED`** in the Version Processing Order table
     - Log the completion event
     - **Check if there are MORE versions** in the Version Processing Order table:
       - If YES → go to Phase 1.5 (Version Initialization) for the NEXT version. Do NOT
         proceed to Phase 4 yet — deployment artifacts are generated only after the LAST version.
       - If NO (this was the last version) → proceed to Phase 4
   - If bugs remain unresolved for the current version:
     - UPGRADE_MASTER.md remains Phase B = `IN_PROGRESS`
     - Ralph Loop will re-feed the prompt, and Phase 0 will resume Phase B for this version
4. **Log** progress in the Upgrade Log table (include the version column)

### Phase 4: Generate Deployment Artifacts

After ALL versions in the Version Processing Order are `COMPLETED` (each version's Phase A
is `COMPLETED`, and Phase B is `COMPLETED` or `SKIPPED`), regenerate deployment artifacts
to ensure they reflect the final state of all new features and bug fixes across all versions.

#### Step 4.1: Invoke depgen-k8s

Invoke the deployment artifact generator:
```
Skill(skill: "depgen-k8s", args: "<application>")
```

The `depgen-k8s` skill auto-detects the technology stack from the application's project files
(pom.xml, composer.json, or package.json) and will regenerate:
- `Dockerfile` — production-ready, multi-stage Docker build in the application folder
- `<source-code-path>/k8s/<environment>/` — Per-environment Kubernetes manifests

#### Step 4.2: Verify Deployment Artifacts

Confirm that both files were generated:
- `<source-code-path>/Dockerfile` exists
- `<source-code-path>/k8s/` folder exists with at least one environment subfolder

If either file is missing, log a warning in UPGRADE_MASTER.md and proceed to Phase 5
(Completion) — do NOT block completion on deployment artifacts.

### Phase 5: Completion

1. **Verify ALL versions in the Version Processing Order table are `COMPLETED`**:
   - For each version: Phase A is `COMPLETED`, AND Phase B is `COMPLETED` or `SKIPPED`
2. **Update UPGRADE_MASTER.md**: Set top-level status to `COMPLETED`, record completion date
3. **Log** the final completion event
4. **Output the completion promise**:
   ```
   <promise>VERSION UPGRADE COMPLETED</promise>
   ```

## Critical Rules

1. **CLAUDE.md is source of truth** — All tool commands (Maven, database CLI, Keycloak, Playwright,
   etc.) MUST use paths and credentials from CLAUDE.md, which is already loaded in context.

2. **Phase order is strict, version order is strict** — Phase A (feature development) MUST
   complete before Phase B (bug fixing) begins for each version. Code must exist before bugs
   can be reproduced and fixed. When processing multiple versions, both Phase A and Phase B
   for version N must fully complete before version N+1 begins. This ensures the codebase is
   progressively upgraded and stabilized version by version.

3. **UPGRADE_MASTER.md is the master checkpoint** — Every iteration starts by reading this file to
   determine where to resume. The Version Processing Order table and Current Version section
   track cross-version progress. All phase and version transitions are recorded here.

4. **No nested Ralph Loops** — This skill starts ONE Ralph Loop. When invoking conductor-defect
   and conductor-feature-develop, they will detect the existing Ralph Loop and share it.
   The completion promise for the OUTER loop is `VERSION UPGRADE COMPLETED`. The sub-skills'
   own completion promises (`ALL BUGS RESOLVED`, `ALL MODULES IMPLEMENTED`) are NOT used —
   instead, this skill checks the sub-skills' tracking files directly.

5. **Phase A cannot be skipped** — Feature development is always required for each version in
   the upgrade. If context artifacts are missing, stop and inform the user.

6. **Phase B can be skipped per version** — If BUG.md does not exist or has no bugs for the
   current version being processed, Phase B is marked `SKIPPED` for that version and the
   workflow proceeds to the next version (or Phase 4 if this is the last version). Phase B
   may be skipped for some versions but not others.

7. **Context window awareness** — Ralph Loop handles recovery across context window limits.
   Always save progress to UPGRADE_MASTER.md before context runs out.

8. **Ralph Loop discipline** — NEVER stop prematurely. Continue working through phases and
   versions until context limits force a stop or ALL versions are completed. After Phase A
   (features) completes, IMMEDIATELY start Phase B (bugs). After both phases for the current
   version complete, IMMEDIATELY advance to the next version — do NOT stop between phases
   or versions.

9. **NO creative alternatives for 3rd party apps** — Use ONLY the tools, commands, and
   configurations specified in CLAUDE.md. Do NOT invent alternative approaches.

10. **Argument forwarding** — Always forward the application, the **current single version**
    from the version loop, and module arguments to both sub-skills. Sub-skills always receive
    a single version, never a list. The version loop in this skill handles the sequential
    iteration — sub-skills are unaware of multi-version processing.
