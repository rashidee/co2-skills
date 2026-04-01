---
name: conductor-upgrade-version
description: >
  Version upgrade orchestrator — orchestrates both bug fixing (conductor-defect) and feature
  development (conductor-feature-develop) for a new version in a single session. First resolves
  all bugs from BUG.md, then implements new features from user stories — ensuring the upgraded
  version is both stable and feature-complete. Takes an application name (mandatory), version
  (mandatory), and optional module filter. Use this skill when the user asks to "upgrade version",
  "version upgrade", "upgrade to new version", "new version with bugs and features", "upgrade
  and fix bugs", or any request to systematically fix bugs and implement new features for a
  version upgrade. Also trigger when user says "resume upgrade" to continue from where a
  previous session left off using UPGRADE_MASTER.md progress file.
---

# Version Upgrade Conductor

Orchestrates a full version upgrade by running bug fixing (conductor-defect) followed by
feature development (conductor-feature-develop) in sequence. Ensures the upgraded version
is both stable (all bugs resolved) and feature-complete (all modules implemented).

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
Skill(skill: "ralph-loop:ralph-loop", args: "/conductor-upgrade-version <application> version:<version> [module:<module>] --completion-promise \"VERSION UPGRADE COMPLETED\" --max-iterations 150")
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
3. Phase A (Bug Fixing): delegates to conductor-defect — agent fixes bugs until all are resolved
4. Phase B (Feature Development): delegates to conductor-feature-develop — agent implements modules until all are completed
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
- Phase A: ALL bugs in BUG_MASTER.md have a terminal status (`FIXED`, `CANNOT_REPRODUCE`, or `HIGH_IMPACT`)
- Phase B: ALL modules in IMPLEMENTATION_MASTER.md have status `COMPLETED` (including deployment artifacts and README generation)

Do NOT output the promise prematurely. Do NOT output it to escape the loop.

### Iteration Awareness

At the START of every iteration (including the first), the agent MUST:
1. Check if Ralph Loop is already active (if `.claude/ralph-loop.local.md` exists, skip re-invoking)
2. If NOT active, silently invoke Ralph Loop as described above
3. Read UPGRADE_MASTER.md to determine the current phase and status
4. Resume from the active phase — do NOT restart completed phases
5. If BOTH phases are COMPLETED, output the completion promise and stop

### Never Stop Prematurely

Within a Ralph Loop iteration, the agent MUST:
- Continue working through the current phase until context limits force a stop
- After Phase A completes, IMMEDIATELY transition to Phase B
- Do NOT stop between phases "to let the user review" — Ralph Loop handles iteration
- Do NOT output the completion promise until BOTH phases are verified complete
- If approaching context limits mid-phase, save progress to UPGRADE_MASTER.md so the
  next iteration can resume from the exact point

## Inputs

The skill expects these arguments:

```
/conductor-upgrade-version <application> version:<version> [module:<module>]
```

| Argument | Required | Example | Description |
|----------|----------|---------|-------------|
| `<application>` | Yes | `hub_middleware` | Application name to locate the context folder |
| `version:<version>` | Yes | `version:v2.0.0` | Target version for the upgrade |
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

This skill forwards arguments to its sub-skills:

| Sub-Skill | Forwarded Arguments |
|-----------|-------------------|
| `conductor-defect` | `<application> version:<version> [module:<module>]` |
| `conductor-feature-develop` | `<application> version:<version> [module:<module>]` |

The `source:` argument for `conductor-feature-develop` is NOT accepted by this skill.
It defaults to the resolved `<app_folder>` as per conductor-feature-develop's default behavior.

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
- **Phase A (Bug Fixing)**: Process flows provide step-by-step debugging context for message-driven bugs. conductor-defect uses them to trace which flow step is failing.
- **Phase B (Feature Development)**: Process flows serve as implementation blueprints for message-driven modules. conductor-feature-develop uses them to implement flow steps in the correct order.
- If process flows have changed between the previous version and the target version (new steps, modified steps), both phases should be aware of these changes.

---

## Pre-Requisites

Before the upgrade can proceed, verify the following exist:

### For Phase A (Bug Fixing)
- `<app_folder>/context/BUG.md` — Bug reports for the target version
- If BUG.md does not exist OR has no bugs for the target version, Phase A is **skipped** (marked SKIPPED)

### For Phase B (Feature Development)
- Context artifacts MUST exist (from conductor-feature-prepare):
  - `<app_folder>/context/model/` — Module models
  - `<app_folder>/context/mockup/` — HTML mockups
  - `<app_folder>/context/specification/` — Technical specifications
  - `<app_folder>/context/test/` — Test specifications
- If context artifacts are missing, **stop and inform the user** to run `conductor-feature-prepare` first

### Bug Regression Awareness (Redo/Redevelop Scenario)
- PRD.md modules may contain `### Bug` sections listing previously fixed bugs from prior
  development cycles. Both Phase A (conductor-defect) and Phase B (conductor-feature-develop)
  must take these into account:
  - **Phase A (conductor-defect)**: Bug fixes recorded in PRD.md `### Bug` sections are
    already-resolved issues from past versions — they do not need re-fixing but inform context.
  - **Phase B (conductor-feature-develop)**: When implementing modules, the `### Bug` section
    entries are treated as supplementary requirements to prevent the same bugs from reappearing
    during redevelopment. conductor-feature-develop reads these entries alongside user stories,
    NFRs, and constraints for each module.

## UPGRADE_MASTER.md — Central Tracking File

The UPGRADE_MASTER.md file tracks the overall upgrade progress. It is created during Phase 1
and updated throughout the workflow. Location: `<app_folder>/context/develop/UPGRADE_MASTER.md`

### Template

```markdown
# Version Upgrade: {application} — {version}

> Upgrade started: {date}
> Status: {IN_PROGRESS | COMPLETED}

## Phase Summary

| Phase | Name | Status | Started | Completed | Notes |
|-------|------|--------|---------|-----------|-------|
| A | Bug Fixing | {NEW / IN_PROGRESS / COMPLETED / SKIPPED} | {date} | {date} | {notes} |
| B | Feature Development | {NEW / IN_PROGRESS / COMPLETED} | {date} | {date} | {notes} |

## Phase A: Bug Fixing

- **Sub-Skill**: conductor-defect
- **Invocation**: `/conductor-defect {application} version:{version} [module:{module}]`
- **Tracking File**: `<app_folder>/context/bug/BUG_MASTER.md`
- **Status**: {detailed status}

## Phase B: Feature Development

- **Sub-Skill**: conductor-feature-develop
- **Invocation**: `/conductor-feature-develop {application} version:{version} [module:{module}]`
- **Tracking File**: `<app_folder>/context/develop/IMPLEMENTATION_MASTER.md`
- **Status**: {detailed status}

## Upgrade Log

| Timestamp | Phase | Event | Details |
|-----------|-------|-------|---------|
```

## Redo/Redevelop Guard

If the requested version already has a `conductor-upgrade-version` entry in CHANGELOG.md for
the same application, this means the version was previously upgraded. Before proceeding, check
whether the previous artifacts and code still exist:

1. Read `CHANGELOG.md` from the project root. If it does not exist, skip this check.
2. Scan for rows matching the requested version, application, AND skill name
   `conductor-upgrade-version` OR `conductor-feature-develop`.
3. If a matching entry is found:
   - Check if `<app_folder>/context/develop/UPGRADE_MASTER.md` or
     `<app_folder>/context/develop/IMPLEMENTATION_MASTER.md` still exists, OR
   - Check if source code files exist in `<app_folder>/` (e.g., `pom.xml`, `composer.json`,
     `package.json`, or `src/` directory)
   - If **either** exists: **STOP immediately**. Print: `"Version {version} for {application} was already upgraded (recorded in CHANGELOG.md) and artifacts/code still exist. To redo, first delete the existing tracking files (UPGRADE_MASTER.md, IMPLEMENTATION_MASTER.md) and source code, then re-run this skill."` Do NOT proceed.
   - If **neither** exists (artifacts and code have been cleaned up): proceed normally — this is a legitimate redo/redevelop scenario.
4. If no matching entry is found, proceed normally.

## Processing Workflow

### Phase 0: Resume Check (runs every Ralph Loop iteration)

1. Ensure Ralph Loop is active (check `.claude/ralph-loop.local.md`)
2. Check if `UPGRADE_MASTER.md` exists:
   - **If YES**: Read it and determine the current state:
     - If Phase A is `IN_PROGRESS` or `NEW` → resume Phase A (go to Phase 2)
     - If Phase A is `COMPLETED` or `SKIPPED` and Phase B is `IN_PROGRESS` or `NEW` → resume Phase B (go to Phase 3)
     - If BOTH phases are `COMPLETED` or `SKIPPED` → go to Phase 4 (deployment artifacts), then Phase 5 (completion)
   - **If NO**: Proceed to Phase 1 (fresh start)

### Phase 1: Initialization

1. **Resolve application folder** using input resolution rules
2. **Verify pre-requisites**:
   - Check if BUG.md exists and has bugs for the target version
   - Check if context artifacts exist for feature development
   - If context artifacts are missing, **stop and inform the user**
3. **Determine Phase A applicability**:
   - Read BUG.md and check for bugs matching the target version (and optional module filter)
   - If bugs exist → Phase A status = `NEW`
   - If no bugs for the target version → Phase A status = `SKIPPED`
4. **Create UPGRADE_MASTER.md** with initial status
5. **Log** the initialization event
6. Proceed to Phase 2 (or Phase 3 if Phase A is SKIPPED)

### Phase 2: Bug Fixing (Phase A)

This phase delegates entirely to `conductor-defect`. The delegation works as follows:

1. **Update UPGRADE_MASTER.md**: Set Phase A status to `IN_PROGRESS`, record start date
2. **Invoke conductor-defect** by calling the Skill tool:
   ```
   Skill(skill: "conductor-defect", args: "<application> version:<version> [module:<module>]")
   ```
   - Pass the same application, version, and module arguments received by this skill
   - conductor-defect will handle its own Ralph Loop internally — **do NOT start a nested Ralph Loop**

   **IMPORTANT — Ralph Loop Delegation**:
   When invoking conductor-defect from within this skill, the conductor-defect skill will
   attempt to start its own Ralph Loop. Since this skill's Ralph Loop is ALREADY ACTIVE,
   conductor-defect will detect the existing `.claude/ralph-loop.local.md` and skip
   re-invoking Ralph Loop. This is the expected behavior — both skills share the same
   Ralph Loop session.

3. **Monitor completion**: After conductor-defect returns or the iteration ends:
   - Check `<app_folder>/context/bug/BUG_MASTER.md`
   - If ALL bugs have terminal status (`FIXED`, `CANNOT_REPRODUCE`, `HIGH_IMPACT`):
     - Update UPGRADE_MASTER.md: Phase A = `COMPLETED`, record completion date
     - Log the completion event
     - Proceed to Phase 3
   - If bugs remain unresolved:
     - UPGRADE_MASTER.md remains Phase A = `IN_PROGRESS`
     - Ralph Loop will re-feed the prompt, and Phase 0 will resume Phase A
4. **Log** progress in the Upgrade Log table

### Phase 3: Feature Development (Phase B)

This phase delegates entirely to `conductor-feature-develop`. The delegation works as follows:

1. **Update UPGRADE_MASTER.md**: Set Phase B status to `IN_PROGRESS`, record start date
2. **Invoke conductor-feature-develop** by calling the Skill tool:
   ```
   Skill(skill: "conductor-feature-develop", args: "<application> version:<version> [module:<module>]")
   ```
   - Pass the same application, version, and module arguments received by this skill
   - conductor-feature-develop will handle its own Ralph Loop internally — **do NOT start a nested Ralph Loop**

   **IMPORTANT — Ralph Loop Delegation**:
   Same as Phase A — conductor-feature-develop will detect the existing Ralph Loop and
   skip re-invoking. Both skills share the same Ralph Loop session.

3. **Monitor completion**: After conductor-feature-develop returns or the iteration ends:
   - Check `<app_folder>/context/develop/IMPLEMENTATION_MASTER.md`
   - If ALL modules have status `COMPLETED` (and deployment artifacts + README are generated):
     - Update UPGRADE_MASTER.md: Phase B = `COMPLETED`, record completion date
     - Log the completion event
     - Proceed to Phase 4
   - If modules remain pending:
     - UPGRADE_MASTER.md remains Phase B = `IN_PROGRESS`
     - Ralph Loop will re-feed the prompt, and Phase 0 will resume Phase B
4. **Log** progress in the Upgrade Log table

### Phase 4: Generate Deployment Artifacts

After BOTH phases are complete (Phase A: `COMPLETED` or `SKIPPED`, Phase B: `COMPLETED`),
regenerate deployment artifacts to ensure they reflect the final state of both bug fixes
and new features.

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

1. **Verify both phases**:
   - Phase A: `COMPLETED` or `SKIPPED`
   - Phase B: `COMPLETED`
2. **Update UPGRADE_MASTER.md**: Set top-level status to `COMPLETED`, record completion date
3. **Log** the final completion event
4. **Output the completion promise**:
   ```
   <promise>VERSION UPGRADE COMPLETED</promise>
   ```

## Critical Rules

1. **CLAUDE.md is source of truth** — All tool commands (Maven, database CLI, Keycloak, Playwright,
   etc.) MUST use paths and credentials from CLAUDE.md, which is already loaded in context.

2. **Phase order is strict** — Phase A (bug fixing) MUST complete before Phase B (feature development)
   begins. This ensures the codebase is stable before new features are layered on top.

3. **UPGRADE_MASTER.md is the master checkpoint** — Every iteration starts by reading this file to
   determine where to resume. All phase transitions are recorded here.

4. **No nested Ralph Loops** — This skill starts ONE Ralph Loop. When invoking conductor-defect
   and conductor-feature-develop, they will detect the existing Ralph Loop and share it.
   The completion promise for the OUTER loop is `VERSION UPGRADE COMPLETED`. The sub-skills'
   own completion promises (`ALL BUGS RESOLVED`, `ALL MODULES IMPLEMENTED`) are NOT used —
   instead, this skill checks the sub-skills' tracking files directly.

5. **Phase A can be skipped** — If BUG.md does not exist or has no bugs for the target version,
   Phase A is marked `SKIPPED` and the workflow proceeds directly to Phase B.

6. **Phase B cannot be skipped** — Feature development is always required for a version upgrade.
   If context artifacts are missing, stop and inform the user.

7. **Context window awareness** — Ralph Loop handles recovery across context window limits.
   Always save progress to UPGRADE_MASTER.md before context runs out.

8. **Ralph Loop discipline** — NEVER stop prematurely. Continue working through phases until
   context limits force a stop or both phases are completed.

9. **NO creative alternatives for 3rd party apps** — Use ONLY the tools, commands, and
   configurations specified in CLAUDE.md. Do NOT invent alternative approaches.

10. **Argument forwarding** — Always forward the exact same application, version, and module
    arguments to both sub-skills. Do not modify or omit arguments during forwarding.
