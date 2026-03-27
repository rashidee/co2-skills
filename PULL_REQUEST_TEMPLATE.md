## Summary

<!-- One paragraph: what does this PR add or fix, and why? -->

## Type of Change

<!-- Check all that apply -->

- [ ] `feat` — New skill
- [ ] `fix` — Bug fix in an existing skill
- [ ] `docs` — Documentation only
- [ ] `refactor` — Non-breaking internal improvement
- [ ] `chore` — Repo config or CI change

## Related Issue

<!-- Required for feat and fix PRs. Use "Closes #<issue>" to auto-close. -->

Closes #

---

## For New Skills (`feat`) — Complete This Section

**Skill name:** `<prefix>-<name>` (e.g. `specgen-django-postgres`)

**Skill family:**
- [ ] `specgen-*` — Technical specification generator
- [ ] `modelgen-*` — Data model generator
- [ ] `mockgen-*` — UI mockup generator
- [ ] `testgen-*` — Test scaffold generator
- [ ] `util-*` — Utility / clean-up
- [ ] `conductor-*` — Orchestrator

**Technology stack targeted:**
<!-- e.g. Django 4.2 + PostgreSQL 16 + Django REST Framework 3.15 -->

**CO2 phase this skill covers:**
<!-- Artifact Generation / PRD Clean-Up / Orchestration -->

**Input artifacts required (prerequisites):**
<!-- e.g. PRD.md, model/, mockup/ -->

**Output files produced:**
<!-- List all files and their exact paths relative to <app_folder>/context/ -->

---

## Checklist

### Structure
- [ ] `SKILL.md` added to `skills/<n>/`
- [ ] `SKILL.md` added to `.claude/skills/<n>/` (both locations in sync)
- [ ] `package.json` present in both locations with correct metadata
- [ ] No hardcoded absolute paths
- [ ] No credentials or secrets

### `SKILL.md` Content
- [ ] Front-matter: `name`, `description`, `version`, `author` all present
- [ ] `When to Use This Skill` is unambiguous
- [ ] `Prerequisites` lists all required context artifacts
- [ ] `Input Resolution` references the CO2 standard `<app_folder>/context/` convention
- [ ] `Instructions` are numbered and each step produces a verifiable output
- [ ] `Output` section specifies every file with its exact path
- [ ] `Constraints` section is present

### Output Contract
- [ ] Output follows the CO2 directory structure
- [ ] Traceability tags (`[USL000001]`, `[NFRL000001]`, `[CONL000001]`) preserved end-to-end
- [ ] Skill does not overwrite files owned by other skills

### Testing
- [ ] Validated against a real `PRD.md` with ≥ 2 modules
- [ ] `examples/input/` and `examples/output/` added (if applicable)

---

## Test Evidence

<!-- Paste the directory listing of your output after running the skill, e.g.: -->

```
<app_folder>/context/specification/
├── auth-module/
│   └── SPEC.md
├── product-module/
│   └── SPEC.md
└── SPECIFICATION.md
```

## Notes for Reviewer

<!-- Anything that warrants extra scrutiny — edge cases handled, deliberate deviations, known limitations. -->
