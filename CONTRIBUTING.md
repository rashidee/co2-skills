# Contributing to co2-skills

Thank you for your interest in contributing to **Compound Context (CO2)** — an opinionated, structured AI-assisted development workflow powered by Claude Code. This guide covers everything you need to contribute a new skill, fix a bug in an existing one, or improve documentation.

> 🌐 Website: [compound-context.com](https://compound-context.com)  
> 📦 Install: `/plugin marketplace add rashidee/co2-skills` → `/plugin install co2-skills@co2`

---

## Table of Contents

1. [Code of Conduct](#code-of-conduct)
2. [Repository Structure](#repository-structure)
3. [Skill Taxonomy](#skill-taxonomy)
4. [Skill Anatomy](#skill-anatomy)
5. [Contribution Types & Priority Areas](#contribution-types--priority-areas)
6. [Branch Strategy](#branch-strategy)
7. [Development Workflow](#development-workflow)
8. [Skill Quality Checklist](#skill-quality-checklist)
9. [Pull Request Process](#pull-request-process)
10. [Commit Message Convention](#commit-message-convention)

---

## Code of Conduct

This project follows a standard open-source code of conduct. Be respectful, constructive, and collaborative. Discrimination of any kind is not tolerated. When in doubt, default to kindness.

---

## Repository Structure

```
co2-skills/
├── .claude/
│   └── skills/              # Skills loaded by Claude Code agent at runtime
│       ├── util-ustagger/
│       ├── modelgen-relational/
│       └── ...              # All installed skills mirror skills/ structure
├── .claude-plugin/          # Plugin manifest for /plugin install
├── skills/                  # Canonical source for all CO2 skills
│   ├── util-ustagger/
│   ├── util-usanalyzer/
│   ├── util-projectsync/
│   ├── util-preparek8senv/
│   ├── util-modulesync/
│   ├── modelgen-relational/
│   ├── modelgen-nosql/
│   ├── mockgen-tailwind/
│   ├── specgen-spring-jpa-jtehtmx/
│   ├── specgen-spring-jpa-restapi/
│   ├── specgen-laravel-eloquent-bladehtmx/
│   ├── specgen-react-mui/
│   ├── specgen-ts-cli/
│   ├── testgen-functional/
│   ├── conductor-feature-prepare/
│   ├── depgen-k8s/
│   ├── conductor-feature-develop/
│   ├── conductor-defect/
│   └── conductor-upgrade-version/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   ├── new_skill.md
│   │   └── feature_request.md
│   └── PULL_REQUEST_TEMPLATE.md
├── CLAUDE.md                # Agent context file — project-level
├── README.md
├── co2.svg
└── .gitignore
```

> **Important:** Every skill exists in both `skills/` (canonical source) and `.claude/skills/` (agent-loaded copy). Your PR must update **both** locations.

---

## Skill Taxonomy

Every skill belongs to one of the following families. If your contribution does not fit, open a **Feature Request** issue before starting work.

| Prefix | CO2 Phase | Purpose | Example Invocation |
|---|---|---|---|
| `util-` | Setup & Validation | Validate, tag, sync context files, and prepare infrastructure | `/util-projectsync`, `/util-preparek8senv` |
| `modelgen-` | Artifact Generation | Generate domain data models from PRD.md | `/modelgen-relational <app_name>` |
| `mockgen-` | Artifact Generation | Generate HTML UI mockup screens | `/mockgen-tailwind <app_name>` |
| `specgen-` | Artifact Generation | Generate stack-specific technical specifications | `/specgen-spring-jpa-jtehtmx <app_name>` |
| `testgen-` | Artifact Generation | Generate Playwright E2E test plans and specs | `/testgen-functional <app_name>` |
| `depgen-` | Deployment | Generate Dockerfile and deployment specifications | `/depgen-k8s <app_name>` |
| `conductor-` | Orchestration | Orchestrate multi-skill phase pipelines | `/conductor-feature-develop <app_name>` |

---

## Skill Anatomy

Every skill is a self-contained directory under `skills/`. The canonical structure is:

```
skills/
└── specgen-spring-jpa-restapi/
    ├── SKILL.md            # REQUIRED — Agent instruction file
    ├── package.json        # REQUIRED — Skill metadata
    ├── templates/          # OPTIONAL — Reusable Markdown fragments
    │   └── *.md
    └── examples/           # OPTIONAL — Sample inputs and expected outputs
        ├── input/
        │   ├── PRD.md
        │   └── model.md
        └── output/
            ├── SPEC.md
            └── SPECIFICATION.md
```

### `SKILL.md` Requirements

`SKILL.md` is the only file Claude Code reads at invocation time. It must be precise, sequential, and self-contained. Required sections:

```markdown
---
name: specgen-spring-jpa-restapi
description: >
  One sentence describing what this skill generates.
  Include the exact technology stack and versions.
version: 1.0.0
author: your-github-handle
---

# [Skill Name]

## When to Use This Skill
[Exact trigger conditions. Stack versions and prerequisite artifacts.]

## Prerequisites
[Artifacts that MUST exist before this skill runs, e.g. PRD.md, model/, mockup/]

## Input Resolution
[How the skill locates CLAUDE.md, PRD.md, and context folders.
Reference the standard CO2 context path: <app_folder>/context/]

## Instructions
[Numbered, unambiguous steps. Each step produces a verifiable output.]

## Output
[Exact files produced, their paths, and naming conventions.
Always follow the CO2 output structure:
  <app_folder>/context/specification/<module_folder>/SPEC.md
  <app_folder>/context/specification/SPECIFICATION.md]

## Constraints
[Hard rules — version pins, patterns the skill MUST NOT deviate from.]
```

**Writing principles:**
- Write for a stateless agent that has never seen the codebase
- Every instruction must be verifiable — agent can confirm it completed
- Never assume the agent will remember a previous step
- Prefer explicit paths over relative references
- Keep token footprint lean — remove anything that does not change agent behaviour

### `package.json` Requirements

```json
{
  "name": "specgen-spring-jpa-restapi",
  "version": "1.0.0",
  "description": "Short description matching SKILL.md front-matter.",
  "author": "your-github-handle",
  "repository": "rashidee/co2-skills",
  "keywords": ["co2", "specgen", "spring-boot", "rest-api", "jpa"],
  "license": "MIT"
}
```

---

## Contribution Types & Priority Areas

### 1. New Skill — `specgen-*` (Highest Impact)

The CO2 workflow is technology-agnostic by design. Every `specgen-*` skill takes the same inputs (PRD.md, data models, HTML mockups) and produces `SPECIFICATION.md` + per-module `SPEC.md` files tailored to a target stack. Adding support for a new stack is the single highest-impact contribution.

**Currently supported:**

| Skill | Stack |
|---|---|
| `specgen-spring-jpa-jtehtmx` | Spring Boot 3 + JTE + Tailwind + htmx |
| `specgen-spring-jpa-restapi` | Spring Boot 3 REST API |
| `specgen-laravel-eloquent-bladehtmx` | Laravel 12 + Blade + Tailwind + htmx |
| `specgen-react-mui` | React 19 + TypeScript 5 + Vite 6 + Material UI v6 |
| `specgen-ts-cli` | Node.js CLI + TypeScript + Commander.js + tsup + pkg |

**Priority gaps — PRs welcome:**

| Category | Wanted Skills |
|---|---|
| **Backend** | `specgen-django-postgres`, `specgen-fastapi-sqlalchemy`, `specgen-nestjs-typeorm`, `specgen-aspnet-efcore`, `specgen-rails-activerecord`, `specgen-phoenix-ecto`, `specgen-gin-gorm` |
| **Frontend / Full-stack** | `specgen-nextjs-prisma`, `specgen-nuxtjs`, `specgen-sveltekit`, `specgen-remix` |
| **Mobile** | `specgen-flutter-firebase`, `specgen-reactnative-supabase`, `specgen-kotlin-springboot` |

### 2. New Skill — Other Families

| Family | Priority Gaps |
|---|---|
| `modelgen-*` | Neo4j (graph), InfluxDB (time-series), Cassandra |
| `mockgen-*` | Bootstrap 5, Material UI, Ant Design, shadcn/ui |
| `testgen-*` | Cypress, Selenium, JUnit 5 + Mockito |
| `depgen-*` | Docker Compose, Terraform, AWS ECS/Fargate, Google Cloud Run, Serverless |

### 3. Version Gate & Changelog Append Protocol

All new skills that accept a `version` argument **MUST** include two protocol sections:

1. **Version Gate** (`## Version Gate`) — placed before `## Input Resolution` or `## Workflow`. Reads `CHANGELOG.md` from the project root, determines the highest recorded version, and rejects execution if the requested version is lower.
2. **Changelog Append** (`## Changelog Append`) — placed before `## Important Rules` or `## Output Format`. After successful completion, appends a row to the matching version section in `CHANGELOG.md` (or creates a new section if it doesn't exist).

See any existing skill (e.g., `specgen-spring-jpa-jtehtmx/SKILL.md`) for the exact protocol text to copy.

### 4. Bug Fix

Corrections to existing skill logic — wrong output structure, incorrect version assumptions, broken input resolution, or instructions that produce inconsistent agent behaviour.

### 4. Documentation

Improvements to `README.md`, `CONTRIBUTING.md`, or any `SKILL.md`. Clear documentation lowers the barrier for new contributors.

### 5. Refactor

Non-breaking improvements — reduced token footprint, clearer instruction phrasing, better examples. Must not change the observable output contract of the skill.

---

## Branch Strategy

| Branch | Purpose | Who pushes |
|---|---|---|
| `main` | Stable, released skills — **protected** | Maintainer only via PR merge |
| `feat/<skill-name>` | New skill development | Contributor |
| `fix/<skill-name>-<issue-id>` | Bug fix for a specific skill | Contributor |
| `docs/<short-description>` | Documentation-only changes | Contributor |
| `refactor/<skill-name>` | Non-breaking refactors | Contributor |

**All PRs target `main`.** Direct pushes to `main` are blocked.

Branch naming examples:
```
feat/specgen-django-postgres
fix/util-ustagger-42
docs/contributing-branch-strategy
refactor/mockgen-tailwind-token-reduction
```

---

## Development Workflow

### Step 1 — Fork and create your branch

```bash
git clone https://github.com/<your-handle>/co2-skills.git
cd co2-skills
git checkout -b feat/<your-skill-name>
```

### Step 2 — Create the skill in both locations

```bash
# Canonical source (edit here)
mkdir -p skills/<your-skill-name>

# Agent-loaded copy (must stay in sync with skills/)
mkdir -p .claude/skills/<your-skill-name>
```

Create `SKILL.md` and `package.json` in `skills/<your-skill-name>/`. Copy both files to `.claude/skills/<your-skill-name>/` as well.

### Step 3 — Validate locally

Install your skill into a CO2 project and run it against a minimal `PRD.md` with at least 2 modules:

```bash
/<your-skill-name> <test_app_name>
```

Verify:
- All output files are written to the expected `<app_folder>/context/` paths
- Output structure matches the CO2 directory convention
- No hardcoded paths, credentials, or host-machine assumptions

### Step 4 — Add examples (strongly recommended)

Add at least one `examples/input/` + `examples/output/` pair under `skills/<your-skill-name>/examples/`. This serves as both documentation and a regression baseline for reviewers.

### Step 5 — Submit the PR

Push your branch and open a PR against `main` using the provided PR template.

---

## Skill Quality Checklist

Before marking your PR as ready for review, confirm all of the following:

**Structure**
- [ ] `SKILL.md` is present in both `skills/<name>/` and `.claude/skills/<name>/`
- [ ] `package.json` is present and complete in both locations
- [ ] No hardcoded absolute paths in `SKILL.md`
- [ ] No credentials, secrets, or environment-specific values anywhere in the skill

**`SKILL.md` content**
- [ ] Front-matter includes `name`, `description`, `version`, `author`
- [ ] `When to Use This Skill` is specific and unambiguous
- [ ] `Prerequisites` lists all required context artifacts
- [ ] `Input Resolution` references the CO2 standard context path convention
- [ ] `Instructions` are numbered and produce verifiable outputs
- [ ] `Output` section lists every file produced with its exact path
- [ ] `Constraints` section is present and documents hard rules

**Output contract**
- [ ] Output follows the CO2 directory structure (`<app_folder>/context/<phase>/`)
- [ ] Traceability tags (`[USL000001]`, `[NFRL000001]`, `[CONL000001]`) are preserved end-to-end
- [ ] Skill does not overwrite or delete files owned by other skills

**Testing**
- [ ] Skill validated against at least one real PRD.md with ≥ 2 modules
- [ ] Example input/output pair added under `examples/` (if applicable)

---

## Pull Request Process

1. Open a PR against `main` using the provided PR template
2. Fill in every section of the template — incomplete PRs will be returned
3. Link the related issue (required for bug fixes and new skills with prior discussion)
4. A maintainer will review within 7 days
5. Address all review comments before requesting re-review
6. Maintainer merges when all checks pass and at least one approval is given

**PR size guidance:** Keep PRs focused on a single skill or a single fix. PRs that touch multiple unrelated skills will be asked to be split.

---

## Commit Message Convention

This project uses [Conventional Commits](https://www.conventionalcommits.org/).

```
<type>(<scope>): <short summary in present tense, lowercase>

[optional body — what and why, not how]

[optional footer: Closes #<issue>]
```

**Types:**

| Type | When to use |
|---|---|
| `feat` | Adding a new skill |
| `fix` | Correcting existing skill logic |
| `docs` | Documentation only |
| `refactor` | Non-breaking internal improvement |
| `chore` | Repo config, `.gitignore`, CI changes |

**Scope** is the skill name (without prefix for brevity) or `*` for repo-wide changes.

**Examples:**
```
feat(specgen-django-postgres): add initial Django 4.2 + PostgreSQL spec skill

fix(util-ustagger): handle empty NFR sections without crashing

Closes #42

docs(contributing): add branch strategy and commit convention

refactor(mockgen-tailwind): reduce SKILL.md token footprint by 30%
```

---

## Questions?

Open a [GitHub Discussion](https://github.com/rashidee/co2-skills/discussions) or file an [Issue](https://github.com/rashidee/co2-skills/issues). Happy to help you scope your contribution before you start building.
