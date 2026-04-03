---
name: conductor-defect
model: claude-opus-4-6
effort: max
description: >
  Fix bugs reported by humans from a BUG.md file. Takes an application name (mandatory),
  version (optional), and module (optional), resolves the context folder automatically from
  root-level application folders. Tags untagged bugs, creates a BUG_MASTER.md tracking checklist, then fixes each bug
  one at a time: reproduce with Playwright, write a test spec, plan the fix, apply the fix,
  verify, and update related artifacts (mockups, specifications, module models, user stories).
  Integrates with Ralph Loop to ensure all bugs are fully fixed across sessions. Use this skill
  when the user asks to "fix bugs", "fix bug", "start bug fixing", "resolve bugs from BUG.md",
  "bug fix session", or any request to systematically fix bugs from a BUG.md file. Also trigger
  when user says "resume bug fixing" to continue from where a previous session left off using
  BUG_MASTER.md progress file.
---

# Bug Fixer

Fix bugs reported by humans one at a time, tracked via BUG_MASTER.md and per-bug BUG_FIX_PLAN.md.

## Ralph Loop Integration (INTERNAL — FULLY AUTOMATIC)

This skill internally manages Ralph Loop. The user does NOT need to invoke `/ralph-loop` or know
about Ralph Loop at all. The skill handles starting, iterating, and completing the loop transparently.

### Internal Auto-Start Mechanism

When this skill is invoked (e.g., `/conductor-defect hub_middleware version:v1.0.4`), the
**very first action** — before reading any files, before Phase 0, before ANYTHING — is to
silently start Ralph Loop by invoking the Skill tool internally:

```
Skill(skill: "ralph-loop:ralph-loop", args: "<original-user-invocation-with-all-args> --completion-promise \"ALL BUGS RESOLVED\" --max-iterations 50")
```

**Construction rule**: Take the EXACT text the user typed (e.g., `/conductor-defect hub_middleware
version:v1.0.4 module:location-information`) and pass it as the `args` value, appending the
`--completion-promise` and `--max-iterations` flags.

| User types | Ralph Loop args |
|------------|----------------|
| `/conductor-defect hub_middleware` | `/conductor-defect hub_middleware --completion-promise "ALL BUGS RESOLVED" --max-iterations 50` |
| `/conductor-defect hub_middleware version:v1.0.4` | `/conductor-defect hub_middleware version:v1.0.4 --completion-promise "ALL BUGS RESOLVED" --max-iterations 50` |
| `/conductor-defect hub_middleware version:v1.0.4 module:location-information` | `/conductor-defect hub_middleware version:v1.0.4 module:location-information --completion-promise "ALL BUGS RESOLVED" --max-iterations 50` |

**Skip if already active**: If `.claude/ralph-loop.local.md` already exists, Ralph Loop is
already running (this is a resumed iteration). Do NOT re-invoke — proceed directly to Phase 0.

**BLOCKING**: Do NOT proceed with ANY work until Ralph Loop is confirmed active (either freshly
started or already running from a previous iteration).

### How It Works (Transparent to User)

1. User invokes `/conductor-defect` with their arguments — they never see or interact with Ralph Loop
2. This skill silently starts Ralph Loop with the conductor-defect prompt as the loop body
3. On each iteration, the agent reads BUG_MASTER.md to find the next unresolved bug
4. The agent fixes one or more bugs until context runs out or all bugs are resolved
5. When the agent tries to exit, Ralph Loop re-feeds the same prompt automatically
6. The next iteration resumes from where the last one left off (tracked in BUG_MASTER.md)
7. When ALL bugs are resolved, the agent outputs the completion promise to exit the loop
8. The user only sees bug-fixing progress — Ralph Loop is an invisible persistence layer

### Completion Promise

When ALL bugs in BUG_MASTER.md have a terminal status (`FIXED`, `CANNOT_REPRODUCE`, or
`HIGH_IMPACT`), output the following promise tag to signal Ralph Loop that bug fixing is done:

```
<promise>ALL BUGS RESOLVED</promise>
```

**CRITICAL**: Only output this promise when EVERY bug in BUG_MASTER.md has a terminal status.
Do NOT output the promise prematurely. Do NOT output it to escape the loop.

### Iteration Awareness (Internal)

At the START of every iteration (including the first), the agent MUST:
1. Check if Ralph Loop is already active (if `.claude/ralph-loop.local.md` exists, skip re-invoking)
2. If NOT active, silently invoke Ralph Loop as described above
3. Read BUG_MASTER.md to determine what is already resolved
4. Find the FIRST bug with status `NEW` or `IN_PROGRESS`
5. If that bug has a BUG_FIX_PLAN.md, read it to find the last incomplete step
6. Resume from exactly that point — do NOT re-fix already-resolved bugs
7. If ALL bugs have terminal status, output the completion promise and stop

## Inputs

The skill expects these arguments:

```
/conductor-defect <application> [version:<version>] [module:<module>]
```

| Argument | Required | Example | Description |
|----------|----------|---------|-------------|
| `<application>` | Yes | `hub_middleware` | Application name to locate the context folder |
| `version:<version>` | No | `version:v1.0.4` | Filter bugs by version tag |
| `module:<module>` | No | `module:location-information` | Filter bugs by module |

### Input Resolution

The application name is matched against root-level application folders:
1. Strip any leading `<number>_` prefix from folder names
2. Match case-insensitively
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

### Argument Combinations

| Provided | Behavior |
|----------|----------|
| `<application>` only | All versions, all modules |
| `<application>` + `version:` | Specific version, all modules |
| `<application>` + `version:` + `module:` | Specific version, specific module |

### Context Folder Structure (Expected)

```
<app_folder>/context/
  BUG.md                     # Bug reports grouped by module, optionally versioned
  PRD.md              # User stories (for recording bug fixes)
  bug/                       # Bug tracking folder (BUG_MASTER.md + per-module subfolders)
    BUG_MASTER.md            # Master checklist (created by this skill)
    <module-slug>/           # Per-module folder
      <BUG-XXX>/             # Per-bug folder (named by bug tag)
        screenshot_*.png     # Reproduction screenshots
        BUG_TEST_SPEC.md     # Test spec for verification
        BUG_FIX_PLAN.md      # Fix plan with checklist
  model/                     # Module models (updated if fix involves model changes)
  mockup/                    # HTML mockups (updated if fix involves UI changes)
  specification/             # Technical specifications (updated if fix involves logic changes)
```

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

## BUG.md Format

The BUG.md file follows a hierarchical structure mirroring the module structure in CLAUDE.md:

- **Top-level groups** are H1 headers: `# Common`, `# System Module`, `# Business Module`
- **Modules** are H2 headers under their respective group (e.g., `## UI/UX Standards` under `# Common`,
  `## User` under `# System Module`, `## Employer` under `# Business Module`)
- Under each module, bugs are listed as top-level bullet items. Each bug may optionally have a version
  tag (e.g., `[v1.0.4]`) on the line immediately before the bug items for that version.

```markdown
# Common

## UI/UX Standards
[v1.0.4]
- Bug description here
  - Priority: High
  - Steps to Reproduce:
    1. Step 1
    2. Step 2
  - Expected Result: ...

[v1.0.5]
- Another bug for a different version

---

# System Module

## User
[v1.0.5]
- Bug description here

---

## Notification

---

## Activities

---

## Audit Trail

---

## Document Management

---

# Business Module

## Location Information

---

## Corridor

---

## Recruitment Step

---

## Employer

---

## Recruitment Agent

---

## Industrial Classification

---

## Occupation Classification

---

## Job Demand
[v1.0.6]
- Bug description here

---

## Candidate Registration
```

### Version Filtering Logic

- Version tags appear as `[vX.Y.Z]` on their own line within a module section
- When `version:` filter is provided, only include bugs that appear AFTER the matching version tag
  and BEFORE the next version tag or module header
- When no `version:` filter is provided, include ALL bugs from all versions

### Module Filtering Logic

- Module headers are H2 (`## Module Name`) under their parent group (H1) in BUG.md
- The parent groups (`# Common`, `# System Module`, `# Business Module`) are NOT modules themselves
  — they are organizational headers
- When `module:` filter is provided, only include bugs under the matching H2 module section
- Module matching: convert filter value to title case for matching (e.g., `location-information`
  matches `## Location Information`, `ui-ux-standards` matches `## UI/UX Standards`)
- When no `module:` filter is provided, include ALL modules across all groups

## Bug Tagging Convention

Bug tags follow the format: `BUG-XXX` where `XXX` is a zero-padded running number with interval
of 1, starting from `001`.

- Tag format: `BUG-001`, `BUG-002`, `BUG-003`, ...
- Tags are inserted as `[BUG-XXX]` at the start of the bug description (after the `- `)
- Only tag bugs that do NOT already have a `[BUG-XXX]` tag
- Scan the entire BUG.md to find the highest existing tag number before assigning new ones
- Continue numbering from the highest existing number + 1

Example before tagging:
```markdown
- Table formatting in document management is not consistent...
```

Example after tagging:
```markdown
- [BUG-001] Table formatting in document management is not consistent...
```

## Version Gate

Before starting any work, check `CHANGELOG.md` in the project root:

1. If `CHANGELOG.md` does not exist, skip this check (first-ever execution).
2. If `CHANGELOG.md` exists, scan all `## vX.Y.Z` headings and determine the **highest version** using semantic versioning comparison.
3. Compare the requested version against the highest version:
   - If requested version **>=** highest version: proceed normally.
   - If requested version **<** highest version: **STOP immediately**. Print: `"Version {requested} is lower than the current project version {highest} recorded in CHANGELOG.md. Execution rejected."` Do NOT proceed with any work.
4. If no version argument was provided (version filter = "All"), skip this check.

## PRD.md Extended Sections

When fixing bugs, check PRD.md for the following extended sections and use them as diagnostic context:

### Design System

If PRD.md contains a `# Design System` section referencing a `DESIGN_SYSTEM.md` file:
- When fixing UI bugs (wrong color, incorrect styling, layout issues), consult the design system to determine the **correct** appearance before applying a fix
- The design system is the authoritative source for visual expectations

### Architecture Principle

If PRD.md contains an `# Architecture Principle` section:
- Use architectural patterns for root cause analysis context
- If a bug reports "data inconsistency between modules" and architecture declares "event-driven", focus on fixing event handling (retry, idempotency) rather than adding direct cross-module DB queries
- If architecture declares "stateless" and a bug relates to session state, the fix should ensure no server-side session storage

### High Level Process Flow

If PRD.md contains a `# High Level Process Flow` section:
- **Trace the bug against the process flow** to identify which step is failing
- This provides systematic root cause analysis: Was the message received? Validated? Stored? Was the ACK step reached? Which step failed?
- The process flow serves as a step-by-step debugging guide for message-driven bugs

---

## Workflow

### Phase 0: Resume Check (Runs Every Ralph Loop Iteration)

This phase runs at the START of every iteration, including the first.

0. **Ensure Ralph Loop is active (INTERNAL — silent)** — Check if `.claude/ralph-loop.local.md`
   exists. If it does NOT exist, silently invoke Ralph Loop using the Skill tool as described
   in the "Internal Auto-Start Mechanism" section above. The user should NOT be informed about
   this step — it is an internal implementation detail. If the file already exists, skip this step.

1. **Use project information from CLAUDE.md (already in context)** — extract JDK path, Maven path, database credentials,
   and all infrastructure details. These values are required for every subsequent tool command.

2. Check if `<app_folder>/context/bug/BUG_MASTER.md` exists

3. If it exists, read it and determine the current state:
   - Scan the bug table for the FIRST bug with status `NEW` or `IN_PROGRESS`
   - If ALL bugs have terminal status (`FIXED`, `CANNOT_REPRODUCE`, `HIGH_IMPACT`) →
     output `<promise>ALL BUGS RESOLVED</promise>` and stop
   - Otherwise, read its `BUG_FIX_PLAN.md` (if exists) for detailed progress
   - Resume from the last incomplete step

4. If it does not exist, proceed to Phase 1 (fresh start)

### Phase 1: Pre-Implementation — Analyze, Tag, and Create Master Checklist

#### Step 1.1: Read and Filter BUG.md

1. Read `<app_folder>/context/BUG.md`
2. Apply version and module filters based on the provided arguments
3. Identify all bugs that match the filter criteria
4. Count the total bugs to process

#### Step 1.2: Tag Untagged Bugs

1. Scan the ENTIRE BUG.md for existing `[BUG-XXX]` tags to find the highest number
2. For each untagged bug (matching the filter), assign the next `[BUG-XXX]` tag
3. Write the updated BUG.md with new tags applied
4. **IMPORTANT**: Only tag bugs that do NOT already have a tag. Never modify existing tags.

#### Step 1.3: Create BUG_MASTER.md

Create `<app_folder>/context/bug/BUG_MASTER.md` with this structure:

```markdown
# Bug Master — <Application Name>

**Started**: <date>
**Context**: <app_folder>/context
**Version Filter**: <version or "All">
**Module Filter**: <module or "All">
**Status**: IN PROGRESS

---

## <Module Name>

| Code | Version | Description | Status | Remark |
|------|---------|-------------|--------|--------|
| BUG-001 | v1.0.4 | Short description of the bug | NEW | |
| BUG-002 | v1.0.4 | Short description of the bug | NEW | |

---

## <Another Module>

| Code | Version | Description | Status | Remark |
|------|---------|-------------|--------|--------|
| BUG-003 | v1.0.5 | Short description of the bug | NEW | |

---

## Summary

| Status | Count |
|--------|-------|
| NEW | X |
| IN_PROGRESS | 0 |
| FIXED | 0 |
| CANNOT_REPRODUCE | 0 |
| HIGH_IMPACT | 0 |
| **Total** | **X** |
```

**Status Values:**
- `NEW` — Bug has been tagged but not yet worked on
- `IN_PROGRESS` — Bug is currently being investigated/fixed
- `FIXED` — Bug has been fixed and verified
- `CANNOT_REPRODUCE` — Bug could not be reproduced via Playwright
- `HIGH_IMPACT` — Fix would potentially break other working functionalities; deferred

### Phase 2: Implementation — Fix Each Bug (One at a Time)

For each bug with status `NEW` in BUG_MASTER.md, in order:

#### Step 2.1: Initialize Bug Folder

1. Update BUG_MASTER.md: set the current bug's status to `IN_PROGRESS`
2. Create folder: `<app_folder>/context/bug/<module-slug>/<BUG-XXX>/`
3. This folder will contain all artifacts for this specific bug

#### Step 2.2: Reproduce the Bug with Playwright

1. Read the bug description from BUG.md (including reproduction steps and expected result)
2. Write a Playwright script to reproduce the bug — save the script file in the bug folder:
   `<app_folder>/context/bug/<module-slug>/<BUG-XXX>/reproduce.spec.ts`
3. In the Playwright script, use **explicit screenshot paths** pointing to the bug folder.
   Do NOT rely on Playwright's default screenshot location. Use `page.screenshot()` with an
   absolute or project-relative path:
   ```typescript
   await page.screenshot({
     path: '<app_folder>/context/bug/<module-slug>/<BUG-XXX>/screenshot_reproduce.png',
     fullPage: true
   });
   ```
4. Run the Playwright script: `npx playwright test <path-to-script> --config=<playwright-config>`

**CRITICAL**: Screenshots MUST be saved to `<app_folder>/context/bug/<module-slug>/<BUG-XXX>/`.
Do NOT save screenshots in the application source folder, test output folder, or Playwright's
default results directory. The bug folder in the application's `context/` folder is the single source of truth for all
bug artifacts.

**IF able to reproduce:**
- Update BUG_MASTER.md remark: "Reproduced successfully"
- Proceed to Step 2.3

**IF NOT able to reproduce:**
- Update BUG_MASTER.md: set status to `CANNOT_REPRODUCE`, add remark with details
- Save the screenshot showing the current (non-buggy) state to the bug folder
- Move to the next bug

#### Step 2.3: Write Test Spec

Create `<app_folder>/context/bug/<module-slug>/<BUG-XXX>/BUG_TEST_SPEC.md`:

```markdown
# Test Spec — <BUG-XXX>

**Bug**: <Short description>
**Module**: <Module Name>
**Reproduced**: Yes

---

## Pre-Conditions

- <List any required state, data, or login credentials>

## Steps to Verify Fix

1. <Step-by-step instructions to verify the bug is fixed>
2. <Navigate to...>
3. <Assert that...>

## Expected Result After Fix

- <What the user should see when the bug is fixed>

## Playwright Verification Script

```typescript
// Playwright test to verify the fix
test('<BUG-XXX>: <description>', async ({ page }) => {
  // Steps to verify...

  // IMPORTANT: Save screenshots to the bug folder, NOT the source folder
  await page.screenshot({
    path: '<app_folder>/context/bug/<module-slug>/<BUG-XXX>/screenshot_fixed.png',
    fullPage: true
  });
});
```
```

#### Step 2.4: Analyze and Plan the Fix

1. Analyze the bug in detail — read relevant source code, templates, configurations
2. Determine the root cause
3. Plan how to fix it
4. Assess impact on other functionalities

Create `<app_folder>/context/bug/<module-slug>/<BUG-XXX>/BUG_FIX_PLAN.md`:

```markdown
# Fix Plan — <BUG-XXX>

**Bug**: <Short description>
**Module**: <Module Name>
**Root Cause**: <What is causing the bug>
**Impact Assessment**: <Low/Medium/High — does fixing this affect other features?>

---

## Fix Checklist

- [ ] 1. <First change to make>
- [ ] 2. <Second change to make>
- [ ] 3. Verify fix with BUG_TEST_SPEC.md
- [ ] 4. Update artifacts (mockups/specs/models/user stories)

---

## Files to Modify

| File | Change Description |
|------|-------------------|
| `path/to/file.jte` | Fix table styling |
| `path/to/file.java` | Update method logic |

---

## Fix Log

### Step 1: <description>
<timestamp> - Started
- Changes made: ...
- Result: ...
```

**IF the fix potentially affects other working functionalities (HIGH_IMPACT):**
- Update BUG_MASTER.md: set status to `HIGH_IMPACT`, add remark explaining the risk
- Record the analysis in BUG_FIX_PLAN.md
- Do NOT apply the fix
- Move to the next bug

**IF safe to fix:**
- Proceed to Step 2.5

#### Step 2.5: Apply the Fix

1. Make the code changes as planned in BUG_FIX_PLAN.md
2. Update BUG_FIX_PLAN.md: check off completed items, log changes in the Fix Log section

#### Step 2.6: Verify the Fix

1. Run the verification from BUG_TEST_SPEC.md (Playwright test)
2. Capture post-fix screenshot using an explicit path in the Playwright script:
   ```typescript
   await page.screenshot({
     path: '<app_folder>/context/bug/<module-slug>/<BUG-XXX>/screenshot_fixed.png',
     fullPage: true
   });
   ```
   Do NOT save screenshots in the application source folder — always use the bug folder under `<app_folder>/context/`.

**IF verified (test passes):**
- Update BUG_MASTER.md: set status to `FIXED`
- Update BUG_FIX_PLAN.md: mark all checklist items as completed
- Proceed to Step 2.7

**IF NOT verified (test fails):**
- Log the failure in BUG_FIX_PLAN.md
- Go back to Step 2.4 to re-analyze and re-plan
- Apply a different fix approach

#### Step 2.7: Update Related Artifacts

Analyze the fix to determine which artifacts need updating:

**A. UI Fix → Update Mockups**

If the fix changed the visual appearance (HTML/CSS/layout changes in JTE templates):
1. Navigate to `<app_folder>/context/mockup/` and find the relevant module screens
2. Update the HTML mockup files to reflect the fix
3. Log the mockup changes in BUG_FIX_PLAN.md

**B. Code Logic Fix → Update Specifications**

If the fix involved code logic changes (service layer, controller logic, validation):
1. Navigate to `<app_folder>/context/specification/<module-slug>/`
2. Update the SPEC.md to reflect the changed behavior
3. Log the specification changes in BUG_FIX_PLAN.md

**C. Module Model Fix → Update Models**

If the fix involved module model changes (new/removed fields, collection changes):
1. Navigate to `<app_folder>/context/model/<module-slug>/`
2. Update `model.md`, `schemas.json`, and `document-model.mermaid` as needed
3. Log the model changes in BUG_FIX_PLAN.md

**D. Record in PRD.md**

If the bug fix is NOT already documented in PRD.md:
1. Open `<app_folder>/context/PRD.md`
2. Find the specific module section
3. Look for a `### Bug` section (positioned AFTER the `### Reference` section)
4. If the `### Bug` section does NOT exist, create one after `### Reference`
5. Add the version tag on the line immediately after the `### Bug` header, using the bug's
   version from BUG.md (e.g., `[v1.0.3]`). This follows the same format as all other sections
   in PRD.md (User Story, Non Functional Requirement, Constraint, Reference) where the
   version tag `[vX.Y.Z]` appears on its own line directly after the `###` header.
6. Add a new bullet point with the bug tag and fix description after the version tag:
   ```markdown
   ### Bug
   [v1.0.3]
   - [BUG-001] Fixed table formatting inconsistency between document management and location information screens
   ```
7. If the `### Bug` section already exists:
   - Check if the bug's version tag already appears in the section
   - If the version tag already exists, append the new bug fix bullet point under that version
   - If the version tag does NOT exist, add the new version tag and bullet point after the
     last existing entry (following the same multi-version pattern used in other sections):
     ```markdown
     ### Bug
     [v1.0.3]
     - [BUG-001] Fixed table formatting inconsistency
     [v1.0.4]
     - [BUG-003] Fixed another issue
     ```

#### Step 2.8: Record Summary in BUG_MASTER.md

1. Update the bug's `Remark` column with a brief summary of what was fixed
2. Include chain effects (what other artifacts were updated)
3. Update the Summary table counts
4. Move to the next bug with status `NEW`

### Phase 3: Generate Deployment Artifacts

After ALL bugs have been processed (every bug has a terminal status), regenerate deployment
artifacts (Dockerfile and Kubernetes manifests) to reflect any code changes from bug fixes.

#### Step 3.1: Invoke depgen-k8s

Invoke the deployment artifact generator:
```
Skill(skill: "depgen-k8s", args: "<application>")
```

The `depgen-k8s` skill auto-detects the technology stack from the application's project files
(pom.xml, composer.json, or package.json) and will regenerate:
- `Dockerfile` — production-ready, multi-stage Docker build in the application folder
- `<source-code-path>/k8s/<environment>/` — Per-environment Kubernetes manifests

#### Step 3.2: Verify Deployment Artifacts

Confirm that both files were generated:
- `<source-code-path>/Dockerfile` exists
- `<source-code-path>/k8s/` folder exists with at least one environment subfolder

If either file is missing, log a warning in BUG_MASTER.md and proceed to Phase 4
(Completion) — do NOT block completion on deployment artifacts.

### Phase 4: Completion

After all bugs have been processed (every bug has a terminal status):

1. Update BUG_MASTER.md:
   - Set top-level `**Status**:` to `COMPLETED`
   - Update all Summary table counts
2. Append an entry to `CHANGELOG.md` in the project root:
   - Read `CHANGELOG.md` from the project root. If it does not exist, create it with context header.
   - Search for a `## {version}` heading matching the current version (use the version filter; if "All", use the highest version found in BUG.md).
   - If the section **exists**: append a new row to its table.
   - If the section **does not exist**: insert a new section after the `---` below the context header and before any existing `## vX.Y.Z` section (newest-first ordering), with a new table header and the first row.
   - Row format: `| {YYYY-MM-DD} | {application_name} | conductor-defect | {module or "All"} | Fixed {count} bugs ({list of BUG codes}) |`
   - **Never modify or delete existing rows.**
3. Output the Ralph Loop completion promise: `<promise>ALL BUGS RESOLVED</promise>`

## Critical Rules

1. **CLAUDE.md is the source of truth for all tool commands** — CLAUDE.md is automatically
   loaded into context. Use the exact JDK path, Maven path, database credentials,
   and all other infrastructure details from CLAUDE.md. NEVER hardcode or guess paths.

2. **One bug at a time** — Fix bugs sequentially. Complete all steps for one bug before moving
   to the next. Never work on multiple bugs simultaneously.

3. **BUG_MASTER.md is the master checkpoint** — Ralph Loop uses BUG_MASTER.md to determine
   which bugs have been fixed and which are pending. Always keep it up to date. For detailed
   status of each bug, refer to the BUG_FIX_PLAN.md in each bug's folder.

4. **Never skip reproduction** — Always attempt to reproduce the bug with Playwright first.
   If it can't be reproduced, mark it as `CANNOT_REPRODUCE` and move on.

5. **HIGH_IMPACT defers, not blocks** — If a fix would break other features, mark it as
   `HIGH_IMPACT` and move to the next bug. Do not attempt risky fixes.

6. **Track everything** — Every action should be logged in BUG_FIX_PLAN.md so that any
   future session (or Ralph Loop iteration) can understand what was done and what remains.

7. **Preserve existing tags** — Never modify or remove existing `[BUG-XXX]` tags in BUG.md.
   Only add new tags to untagged bugs.

8. **Update artifacts faithfully** — When a fix changes UI, logic, or models, update the
   corresponding mockups, specifications, and module models. Keep all artifacts in sync.

9. **Context window awareness** — If approaching context limits, save progress to
   BUG_MASTER.md and BUG_FIX_PLAN.md. Ralph Loop will resume from where you left off.

10. **Ralph Loop discipline — NEVER stop prematurely** — After fixing one bug, IMMEDIATELY
    check for the next `NEW` bug and start it. Do NOT stop "to let the user review". The only
    valid reasons to stop within an iteration are: (a) context window approaching limit,
    (b) all bugs resolved (output promise), or (c) an unrecoverable error requiring user input.

11. **Auto-start Ralph Loop (INTERNAL)** — The FIRST action MUST be to silently check if Ralph
    Loop is active (`.claude/ralph-loop.local.md` exists). If not, silently invoke it using the
    Skill tool. The user should NEVER be asked to start Ralph Loop manually — this is an internal
    implementation detail handled entirely by the skill. Do NOT mention Ralph Loop to the user.

12. **NO creative alternatives for 3rd party applications (CRITICAL)** — Use the EXACT methods,
    connection strings, CLIs, and credentials described in `CLAUDE.md`. NEVER use Docker
    containers, alternative databases, or different CLIs than what CLAUDE.md specifies.

13. **Screenshots MUST be saved in the bug folder (CRITICAL)** — All Playwright screenshots
    (reproduction, verification, fixed) MUST be saved to
    `<app_folder>/context/bug/<module-slug>/<BUG-XXX>/` using explicit `page.screenshot({ path: ... })`
    calls. NEVER save screenshots in the application source folder, Playwright's default test-results
    directory, or any other location. The Playwright script files themselves should also be saved in
    the bug folder, not in the source tree.
