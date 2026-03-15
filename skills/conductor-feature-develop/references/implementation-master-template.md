# IMPLEMENTATION_MASTER.md Template

Use this template when creating the master implementation tracking file in Phase 1.

---

```markdown
# Implementation Master - {Application Name}

**Started**: {YYYY-MM-DD}
**Source Code**: `{source-code-path}/`
**Context**: `{context-path}/`
**Status**: IN PROGRESS

---

## Execution Order

{Copy verbatim from TEST_PLAN.md Section 5 "Execution Order"}

---

## Pre-Implementation (Scaffolding)

| Task | Status | Notes |
|------|--------|-------|
| Project skeleton (Maven/Gradle) | PENDING | |
| Build configuration (pom.xml) | PENDING | |
| Application configuration (YAML) | PENDING | |
| Security configuration (Keycloak) | PENDING | |
| JTE layouts + shared partials | PENDING | |
| UI components (Tailwind) | PENDING | |
| JS + CSS structure | PENDING | |
| Data access configuration | PENDING | |
| Error handling | PENDING | |
| Theming | PENDING | |
| Pagination support | PENDING | |
| Messaging (RabbitMQ) config | PENDING | |
| Scheduling (Quartz/Batch) config | PENDING | |
| Playwright E2E project setup | PENDING | |
| Application compiles + starts | PENDING | |

---

## Module Implementation Status

| # | Module | Layer | Status | Started | Completed | Notes |
|---|--------|-------|--------|---------|-----------|-------|
{For each module in execution order:}
| {n} | {Module Name} | {Layer} | PENDING | - | - | |

---

## Module Details

{For each module in execution order:}

### {n}. {Module Name}

**Layer**: {Layer}
**Dependencies**: {List from TEST_PLAN.md}

**Resources**:

| Resource | Path | Exists |
|----------|------|--------|
| User Stories | {IDs} | Yes |
| Model | `model/{module-slug}/model.md` | {Yes/No} |
| Schema | `model/{module-slug}/schemas.json` | {Yes/No} |
| Specification | `specification/{module-slug}/SPEC.md` | {Yes/No} |
| Test Spec | `test/{module-slug}/TEST_SPEC.md` | {Yes/No} |
| Mockup | `mockup/{role}/content/{screen}.html` | {Yes/No} |

**Test Summary** (from TEST_SPEC.md):
- Total scenarios: {count}
- Seeding strategy: {strategy from TEST_PLAN.md}

---
```

## Status Values

| Status | Meaning |
|--------|---------|
| PENDING | Not yet started |
| IN PROGRESS | Currently being implemented |
| COMPLETED | Implementation done, tests passing |
| BLOCKED | Cannot proceed due to dependency or issue |
| FAILED | Tests failing, needs investigation |
