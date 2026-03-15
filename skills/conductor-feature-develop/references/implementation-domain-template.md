# IMPLEMENTATION_MODULE.md Template

Use this template when creating per-module implementation tracking files in Phase 3.

---

```markdown
# Implementation - {Module Name}

**Module**: {Module Name}
**Layer**: {Layer}
**Status**: IN PROGRESS
**Started**: {YYYY-MM-DD}
**Completed**: -

---

## Resources

| Resource | Path |
|----------|------|
| User Stories | {Comma-separated IDs} |
| Model | `model/{module-slug}/model.md` |
| Schema | `model/{module-slug}/schemas.json` |
| Specification | `specification/{module-slug}/SPEC.md` |
| Test Spec | `test/{module-slug}/TEST_SPEC.md` |
| Mockup | `mockup/{role}/content/{screen}.html` |

---

## Implementation Checklist

- [ ] 1. Read and analyze module resources
- [ ] 2. Implement document/entity classes
- [ ] 3. Implement repository layer
- [ ] 4. Implement service layer
- [ ] 5. Implement MapStruct mappers (if applicable)
- [ ] 6. Implement controller layer
- [ ] 7. Implement JTE view templates
- [ ] 8. Implement message listeners (if applicable)
- [ ] 9. Implement scheduled jobs (if applicable)
- [ ] 10. Write Playwright E2E tests
- [ ] 11. Run E2E tests and verify
- [ ] 12. Visual consistency testing (mockup vs application)
- [ ] 13. Fix visual deviations (if any)
- [ ] 14. Update tracking files

---

## Source Files Created

| File | Purpose |
|------|---------|
{Populated as files are created during implementation}

---

## Implementation Log

{Append entries as work progresses}

### Step 1: Analyze Module Resources
**{timestamp}** - Started

Findings:
- Collections: {list}
- Key entities: {list}
- User stories covered: {IDs}
- Test scenarios count: {count}
- Dependencies satisfied: {Yes/No — list any missing}

---

### Step 2: Document/Entity Classes
**{timestamp}** - Started

Files created:
- `{package}/{ModuleEntity}.java`
...

---

{Continue for each step...}

### Step 11: E2E Test Results
**{timestamp}** - Executed

```
Test results:
  {count} passed
  {count} failed
  {count} skipped

Failed tests (if any):
  - {test name}: {failure reason}
```

### Resolution (if tests failed)
**{timestamp}** - Fixed

- Issue: {description}
- Fix: {what was changed}
- Re-run result: {pass/fail}

---

### Step 12: Visual Consistency Testing
**{timestamp}** - Executed

Screens compared against mockup baselines:

| Screen | Mockup Source | App Route | Result | Notes |
|--------|--------------|-----------|--------|-------|
| {screen-name} | `mockup/{role}/content/{file}.html` | `/{app-route}` | PASS/FAIL | {deviation details if any} |

Aesthetic checks:
- [ ] Color scheme matches (backgrounds, borders, text, buttons)
- [ ] Spacing/alignment matches (padding, margins, gaps)
- [ ] Layout structure matches (grid/flex, sidebar/header/content)
- [ ] Typography matches (font sizes, weights, line heights)
- [ ] Component styling matches (buttons, tables, forms, cards)

### Step 13: Visual Deviation Fixes (if any)
**{timestamp}** - Fixed

| Deviation | File Changed | Fix Applied |
|-----------|-------------|-------------|
| {e.g. "button color mismatch"} | `{jte/css file}` | {what was changed} |

Re-run result: {pass/fail}
```
