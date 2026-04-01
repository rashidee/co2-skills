---
name: util-usanalyzer
description: >
  Analyze PRD.md files for quality issues: incomplete sentences, non-existent references,
  cross-module inconsistencies, contradictory requirements, and duplicate roles. Adds inline
  [TODO] annotations directly into the PRD.md file for each issue found.
  Trigger on keywords: "analyze user story", "analyze user stories", "check user story quality",
  "validate user stories", "find user story issues", "audit user stories", "review user stories",
  "check requirements quality", "find inconsistencies", "find contradictions".
  Accepts an application name as input (e.g., `/util-usanalyzer mainapp`,
  `/util-usanalyzer "Main App"`).
---

# Util US Analyzer

Analyze PRD.md files for quality issues and add `[TODO]` annotations for each issue found.

## Input Resolution

This skill requires one mandatory argument: `<application>`.

| Argument | Required | Description |
|----------|----------|-------------|
| `<application>` | Yes | Application name — matched against root-level application folders |
| `<module>` | No | Optional module name to limit analysis to a single module |

### Application Folder Resolution

1. List root-level application folders that may have a numeric prefix (e.g., `1_hub_middleware`, `2_hc_adapter`) or no prefix (e.g., `mainapp`)
2. Strip the leading `<number>_` prefix from each folder name if present (e.g., `1_hub_middleware` -> `hub_middleware`)
3. Match the provided application name **case-insensitively** against the stripped folder names
4. Accept `snake_case`, `kebab-case`, or title-case input (e.g., `hub_middleware`, `hub-middleware`, `Hub Middleware` all match `1_hub_middleware`)
5. If no match is found, list all available application names and **stop**

### Auto-Resolved Paths

| File | Resolved Path |
|------|---------------|
| PRD.md | `<matched_app_folder>/context/PRD.md` |

Example invocations:
- `/util-usanalyzer mainapp`
- `/util-usanalyzer "Main App"`
- `/util-usanalyzer mainapp module:Home`

## Workflow

### 1. Read and Parse PRD.md

Read the entire PRD.md file and parse its structure:
- Identify all module sections (delimited by `## <Module Name>` headers)
- Within each module, identify subsections: `### User Story`, `### Non Functional Requirement`, `### Constraint`, `### Reference`, `### Bug`
- Record line numbers for every item for precise TODO annotations

### 2. Collect All Roles

Scan the entire document and collect every role mentioned in user stories (the part after "As a" / "As" and before the comma or "I want"). Build a normalized role registry mapping each unique role to its canonical form and the line numbers where it appears.

### 3. Collect All Cross-References

Scan the entire document for:
- References to other modules (e.g., mentions of `Batch Job`, `Audit Trail`, `Headcount`, etc.)
- References to external documents or sections (e.g., `[PROJECT.md](../PROJECT.md)`)
- References to pages, sections, or features by name within user stories, NFRs, constraints

Build a registry of all outbound references per module.

### 4. Run Analysis Checks

**Skip empty modules entirely.** If a module has no content in any of its subsections (User Story, Non Functional Requirement, Constraint, Reference are all empty — no bullet items), skip that module completely. Do not flag empty sections or run any checks on empty modules.

Execute the following checks in order. For each issue found, record the line number, issue category, and a description.

#### 4.1 Incomplete User Stories

Check every top-level bullet under `### User Story` sections:
- Must follow the Agile format: **"As a [role], I want [feature] so that [benefit]"** or **"As [role], I want [feature] so that [benefit]"**
- Flag items missing "I want" clause
- Flag items missing "so that" clause (benefit/justification)
- Flag items that appear to be truncated or end abruptly (e.g., ending with a colon but no sub-bullets, or trailing whitespace with no content)
**Exception:** User stories that end with a colon (`:`) followed by sub-bullets describing the detail are acceptable — they provide the "what" through their sub-items. Only flag these if the sub-bullets are missing.

#### 4.2 Non-Existent References

For each reference found in step 3:
- If it references an external file (e.g., `[PROJECT.md](../PROJECT.md)`), verify the file exists at the resolved path relative to the PRD.md location
- If it references a module by name (e.g., backtick-quoted `Batch Job`), verify that module exists as a `## <Module>` section in the document
- If it references a page or feature (e.g., "redirect to the Headcount list page"), verify that the referenced module exists

#### 4.3 Cross-Module Inconsistencies

For each module, check:
- If a user story, NFR, or constraint mentions a feature/entity from another module (e.g., "Headcount Fulfillment Status"), verify that feature is actually defined in the referenced module
- If a user story mentions navigating to another module's page (e.g., "redirect to the Sub Project details page"), verify that view/page is described in the target module's user stories
- If an NFR or constraint references a concept from another module (e.g., "using `Batch Job`"), verify the referenced module defines that concept

#### 4.4 Contradictory Requirements

Compare requirements within the same module and across modules:
- Look for NFRs or constraints that directly contradict each other (e.g., one says "will be automatically deleted" while another says "will not be automatically deleted")
- Look for user stories that grant permissions contradicted by constraints (e.g., a user story allows editing but a constraint says the data is read-only)
- Look for status values or enums defined differently in different places (e.g., status list in user story differs from status list in constraint)
- Look for contradictions between NFR and Constraint sections within the same module (e.g., Sub Project NFR says it has status values but Constraint says it will not have status)

#### 4.5 Duplicate Roles

Using the role registry from step 2:
- Identify roles that are semantically similar but named differently (e.g., "Project Manager" vs "PM", "user" vs "User", "staff" vs "Staff")
- Check the `## Role` module's constraint section for the canonical role list
- Flag any role used in user stories that doesn't match the canonical list
- Flag inconsistent capitalization of the same role (e.g., "a user" vs "a User")

#### 4.6 Design System Reference Validation

If a `# Design System` section exists in PRD.md:
- Extract the referenced file path (e.g., from `[DESIGN_SYSTEM.md](reference/DESIGN_SYSTEM.md)`)
- Resolve the path relative to PRD.md's location
- Verify the referenced file actually exists at the resolved path
- If the file does not exist, flag as `BAD_REF`

#### 4.7 Architecture Principle Consistency

If an `# Architecture Principle` section exists in PRD.md:
- Extract declared architectural patterns (e.g., "stateless", "event-driven", "document based database", "message driven", "monolithic")
- Cross-reference with NFRs and Constraints across all modules:
  - If architecture declares "stateless" but an NFR implies server-side session storage, flag as `CONTRADICTION`
  - If architecture declares "document based database" but constraints reference SQL joins or foreign key enforcement, flag as `CONTRADICTION`
  - If architecture declares "event-driven" inter-module communication but an NFR describes synchronous direct calls between modules, flag as `CONTRADICTION`
  - If architecture declares "message driven" processing but an NFR or constraint implies synchronous external API polling, flag as `CONTRADICTION`
- Only flag clear, unambiguous contradictions — patterns that directly oppose the stated architectural principle

#### 4.8 Process Flow Consistency

If a `# High Level Process Flow` section exists in PRD.md:
- Parse all process flows and their steps. Each flow is a named subsection (e.g., `## Recruitment Agent Sync`, `## Job Demand`) with ordered steps as bullet items
- For each step that references a module or entity by name, verify:
  - **Flow-Module Consistency**: The referenced module exists as a `## <Module>` section under `# System Module` or `# Business Module`. If not, flag as `CROSS_MODULE`
  - **Flow-NFR Coverage**: Each step describing message publishing, queue consumption, or asynchronous processing should have a corresponding NFR in the relevant module that describes that behavior. If a step describes "publishes ACK message" but no module NFR mentions ACK publishing, flag as `CROSS_MODULE`
  - **Flow Completeness**: If a flow describes an error or failure path (e.g., "on validation failure") but no NFR or constraint in the relevant module handles that error condition, flag as `CROSS_MODULE`
- Do not flag flows whose steps are fully covered by existing NFRs

### 5. Insert TODO Annotations

For each issue found, insert a `[TODO]` annotation in the PRD.md file:

**Placement rules:**
- Insert the `[TODO]` as a new line **immediately above** (before) the problematic line, at the **same indentation level** as the problematic line
- If the issue relates to a section-level problem (e.g., empty section, cross-module inconsistency), insert the `[TODO]` as a top-level bullet above the first item after the version tag line

**Format:**
```
- [TODO] <CATEGORY>: <description> (line <N>)
- <the problematic line stays unchanged below>
```

**Categories:**
| Category | Code |
|----------|------|
| Incomplete User Story | `INCOMPLETE` |
| Non-Existent Reference | `BAD_REF` |
| Cross-Module Inconsistency | `CROSS_MODULE` |
| Contradictory Requirement | `CONTRADICTION` |
| Duplicate Role | `DUP_ROLE` |
| Architecture Violation | `ARCH_VIOLATION` |

**Examples:**
```markdown
- [TODO] INCOMPLETE: User story is missing content after the colon — no sub-bullets describing what the dashboard shows. Add the dashboard items or remove the trailing colon. (line 25)
- As Management Team, I want to be able to view the dashboard showing:
- [TODO] INCOMPLETE: User story is missing "so that [benefit]" clause. Add the business justification. (line 304)
- [TODO] INCOMPLETE: Possible typo — "be to manage" should likely be "be able to manage". (line 304)
- As Super Admin, I want to be to manage divisions
```

### 6. Output Summary

After inserting all TODOs, print a summary table:

```
## Analysis Summary for <Application Name>

| Category              | Count | Modules Affected                  |
|-----------------------|-------|-----------------------------------|
| Incomplete            | X     | Home, Division, ...               |
| Bad Reference         | X     | ...                               |
| Cross-Module          | X     | ...                               |
| Contradiction         | X     | ...                               |
| Duplicate Role        | X     | ...                               |
| Arch Violation        | X     | ...                               |
| **Total**             | **X** |                                   |

### Details by Module

#### <Module Name>
- [INCOMPLETE] line X: <brief description>
- [BAD_REF] line Y: <brief description>
...
```

## Important Rules

- **NEVER remove, modify, or delete existing content.** Only add new `[TODO]` annotation lines.
- **NEVER modify existing tags** (e.g., `[USHM00003]`, `[v1.0.0]`). These are immutable identifiers.
- Preserve all existing content, formatting, and indentation exactly.
- Each TODO must reference the specific line number where the issue was found for easy navigation.
- Be precise and actionable in TODO descriptions — state what the issue is AND what needs to be done to fix it.
- Do not flag stylistic preferences — only flag genuine quality issues that could lead to implementation ambiguity.
- If a module filter is provided, only analyze that module but still check cross-module references involving it.
- Do not insert duplicate TODOs — if the same issue applies to multiple lines, annotate each line individually but do not repeat the same TODO on the same line.
- Tables, blank lines, version tags (e.g., `[v1.0.0]`), and section headers are never annotated.
- When checking for contradictions, consider the full context — an NFR in one module may intentionally override a general rule stated elsewhere. Only flag clear, unambiguous contradictions.
- When checking roles, the canonical role list is defined in the `## Role > ### Constraint` section under the Role module. Roles not in this list should be flagged.
