# Plugin Information
- Plugin Name: Compound Context (CO2) Workflow (see below for details)

To install the plugin
~~~bash

~~~


# Compound Context (CO2) Workflow

| Logo                | Description                                                                                                                                                                              |
|---------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ![co2.svg](co2.svg) | an **OPINIONATED** approach to software development using AI coding agents. It emphasizes the importance of comprehensive and structured context to reduce AI hallucination and AI slop: |

# Key Features

📝 User Story to drive UI and UX design + Non-Functional Requirement to drive business logic and technical design + Constraints to drive technical design and implementation.

🔄 Versioning to keep track of the changes and updates in the context as the project progresses.

📦 Multi application support where each application will have its own context. This is to support:
   - Microservices architecture (multiple backend services with REST API or GraphQL API)
   - Mobile application development (mobile frontend + REST API backend)
   - Single Page Application development (web frontend + REST API backend)

🔗 Traceability from the generated code to the artifacts in the context to the User Stories, Non-Functional Requirements and Constraints.

🧠 Comprehensive context to reduce AI hallucination and AI slop, which includes:
   - Data Models
   - HTML Mockups
   - Technical Specifications
   - Test Specifications

# Pre-requisites
- Claude Code
- Claude Code Plugins
  - Playwright Test Generator Plugin (for generating Playwright test specifications and test scripts)
  - Ralph Loop Plugin (for orchestrating the development and bug fixing process)
- Node and npm version 18 or above (for serving the HTML mockups and running the Playwright test scripts)

# Development Workflow

## Context Window Preparation

_This is the initial step where you want to ensure the AI coding agent has all the information needed to work._

### CLAUDE.md
⚠️ **Shared by the team - Checked in the repository**
  - Project detail
  - Project goals
  - Terminology (if any)
  - Applications (1 or more)
  - Modules (System Modules and Business Modules)

### SECRET.md 
⚠️ **For coding agent only - NOT CHECKED IN the repository**
  - Sensitive information which should not be shared publicly, such as API keys, database credentials, etc.

## PRD Development

**PRD.md** is the main input for the development phase, which contains the User Stories, Non-Functional Requirements and Constraints. It is also the main reference for all the skills in this phase. The PRD.md should be updated and versioned as the project progresses.

Example of the PRD.md structure:
~~~markdown
# Business Modules

## <Module 1>
### User Story
[v1.0.0]
- As a <type of user> I want to <perform some task> so that I can <achieve some goal>.
- As a <type of user> I want to <perform some task> so that I can <achieve some goal>.
### Non-Functional Requirement
[v1.0.0]
- <Non-Functional Requirement 1>
- ~~<Non-Functional Requirement 2>~~
[v1.0.1]
- <Updated Non-Functional Requirement 1>
- Removed <Non-Functional Requirement 2> since <reason for removal>
### Constraint
[v1.0.0]
- <Constraint 1>
- <Constraint 2>
### Reference
[v1.0.0]
- <Reference 1>

### <Module 2>
...
~~~

## PRD Clean Up

_Since PRD.md is based on human input, it may require some validation and each new points added need to be tagged with unique ID for traceability:_

The skills to invoke the `clean-up` process are:

| Skill           | Example Skill Invovation                                                        | Objective                        | Output                                                                                                                                                                                                                                                                                                                                                      |
|-----------------|---------------------------------------------------------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| util-modulesync | /util-modulesync app1/context/CLAUDE.md app1/context/PRD.md app1/context/BUG.md | Module Structure Synchronization | Synchronize the module structure across CLAUDE.md, PRD.md and BUG.md to match the canonical structure defined in CLAUDE.md. This is to ensure the consistency of the module structure across all the artifacts. Example: If there is a new module added in CLAUDE.md, it should also be added in PRD.md and BUG.md with the same module name and structure. |
| util-usanalyzer | /util-usanalyzer app1/context/PRD.md                                            | Quality Check                    | Identify any quality issues in the PRD.md such as incomplete stories, bad references, contradictions and duplicates. Output a report with the identified issues and suggestions for improvement. Example: "User Story 1 is incomplete because it does not have the acceptance criteria."                                                                    |
| util-ustagger   | /util-ustagger app1/context/PRD.md                                              | Traceability                     | Each newly added point in PRD.md will be tagged with a unique 9-character ID code, which can be used for traceability in the later stages of the development. Example: [USL000009], [NFRL000009], [CONL000009], [REFL000009]                                                                                                                                |

## Data Model

_The skill goal is to generate the data model for each module for you to **review** before proceeding to next step._

Example skill invocation for data model generation:
~~~bash
/modelgen-relational <app_name>  ## For relational database model (SQL)
/modelgen-nosql <app_name>       ## For NoSQL document database model (MongoDB, Couchbase, DynamoDB, Firestore, CosmosDB)
~~~

- Input:
  - PRDd.md (User Stories, Non-Functional Requirements and Constraints)
- Output:
  - Model Markdown file containing:
    - Entity name
    - Attributes (name, type, description)
    - Relationships (type of relationship, related entity, description)
  - ERD Diagram in Mermaid syntax

Example of the Model Markdown file structure:
~~~markdown
- context
  - model
    - <module_1_folder>
      - model.md
      - erd.mermaid
    - <module_2_folder>
      - model.md
      - erd.mermaid
    MODEL.md ## A summary of the data model for all the modules
~~~

### HTML Mockup

_The skill goal is to generate the HTML mockup for each screen for you to **review** before proceeding to next step._

Example skill invocation for HTML mockup generation:
~~~bash
/mockgen-tailwind <app_name> ## For generating HTML mockup using Tailwind CSS
~~~

- Input:
  - PRD.md (User Stories, Non-Functional Requirements and Constraints)
  - Data Model (to understand the data structure and relationships)
- Output:
  - HTML mockup screens for the application, served via Node.js + Express + HTMX
  - **MOCKUP.html** — A landing page for all the mockup screens with links to each screen, and each screen will have a link back to the related User Stories, Non-Functional Requirements and Constraints in PRD.md for traceability.

Example of the HTML mockup output:
~~~markdown
- context
  - mockup
    - <role_1_folder>
      - screen1.html
      - screen2.html
    - <role_2_folder>
      - screen1.html
      - screen2.html
    MOCKUP.html ## A landing page for all the mockup screens
~~~

### Technical Specification

_The skill goal is to generate the technical specification for the application, which includes the architecture, components, APIs, etc. for you to **review** before proceeding to next step._

>⚠️ At this point, you should already **have determined the technology stack** as the technical specification generation will be based on the specific technology stack skill

Example skill invocation for technical specification generation:
~~~bash
/specgen-spring-jpa-jtehtmx <app_name> ## For generating Spring Boot 3 web application technical specification with JTE, Tailwind and htmx.
/specgen-spring-jpa-restapi <app_name> ## For generating Spring Boot 3 REST API technical specification.
/specgen-laravel-eloquent-bladehtmx <app_name> ## For generating Laravel 12 web application technical specification with Blade, Tailwind and htmx
~~~

- Input:
  - PRD.md (User Stories, Non-Functional Requirements and Constraints)
  - Data Model (to understand the data structure and relationships)
  - HTML Mockup (to understand the UI design and user interactions)
- Output:
  - Technical specification document containing:
    - **SPEC.md** — A per-module specification file containing:
      - Module overview (package, type, authentication)
      - Traceability table mapping back to User Story IDs, NFR IDs and Constraint IDs
      - Component design (components, responsibilities, interactions)
    - **SPECIFICATION.md** — A summary of the technical specification for all the modules

Example of the technical specification output:
~~~markdown- context
  - specification
    - <module_1_folder>
      - SPEC.md
    - <module_2_folder>
      - SPEC.md
    SPECIFICATION.md ## A summary of the technical specification for all the modules
~~~

### Test Specification Generation
_The skill goal is to generate the test specification for the application, which includes the test cases and scenarios to validate the functionality and performance of the application, for you to **review** before proceeding to next step._

Example skill invocation for test specification generation:
~~~bash
/testgen-functional <app_name> ## For generating Playwright E2E test plan and per-module test specifications as Markdown blueprints.
~~~

- Input:
  - PRD.md (User Stories, Non-Functional Requirements and Constraints)
  - Data Model (to understand the data structure and relationships)
  - HTML Mockup (to understand the UI design and user interactions)
  - Technical Specification (to understand the technical design and implementation)
- Output:
  - Test specification document containing:
    - **TEST_SPEC.md** — A per-module test specification file containing:
        - Module overview
        - Traceability table mapping back to User Story IDs, NFR IDs and Constraint IDs
        - Test cases and scenarios (test case ID, description, preconditions, steps, expected results)
    - **TEST_PLAN.mmd** - Summary of all the test cases and scenarios.

Example of the test specification output:
~~~markdown
- context
  - test
    - <module_1_folder>
      - TEST_SPEC.md
    - <module_2_folder>
      - TEST_SPEC.md
    TEST_PLAN.mmd ## A summary of all the test cases and scenarios in Mermaid syntax
~~~

### Application Development
_The skill goal is to develop the application based on all the context generated above, and you can track the development progress using the output from this skill._

Example skill invocation for application development:
~~~bash
/conductor-feature-develop <app_name> ## For orchestrating full-stack application development module-by-module using all generated artifacts.
~~~ 

- Input:
  - PRD.md (User Stories, Non-Functional Requirements and Constraints)
  - Data Model (to understand the data structure and relationships)
  - HTML Mockup (to understand the UI design and user interactions)
  - Technical Specification (to understand the technical design and implementation)
  - Test Specification (to understand the test cases and scenarios to validate the functionality and performance of the application)
- Output:
  - Source code files for the application, organized by module and component.
  - **IMPLEMENTATION_MASTER.md** — Master tracking file containing:
    - Application overview (name, source code path, context path)
    - Overall status (PENDING → IN PROGRESS → COMPLETED)
    - Execution order (copied from TEST_PLAN.md)
    - Pre-implementation checklist (project skeleton, build config, security, layouts, database, error handling, theming, pagination, Playwright setup)
    - Module implementation status table (module name, layer, status, start/completion dates)
    - Module details with dependencies and resource paths
  - **IMPLEMENTATION_MODULE.md** — Per-module tracking file containing:
    - Module overview (name, layer, status, dates)
    - Resources table (user story IDs, model/schema/specification/test spec/mockup paths)
    - Implementation checklist (entities, repositories, services, mappers, controllers, views, tests)
    - User Stories and NFR completion checklists for traceability
    - Visual consistency checklist (colors, spacing, layout, typography)
    - Source files created table
    - Implementation log with timestamped entries
  - **Playwright E2E test suite** — functional and visual consistency tests per module

# Bug Fixing Workflow

## Bug Reporting
Report bugs in `<app_folder>/context/BUG.md` following the module-based structure:
~~~markdown
# Business Module

## <Module Name>
[v1.0.4]
- Bug description here
  - Priority: High
  - Steps to Reproduce:
  - Expected Result:
~~~

## Bug Fixing

Example skill invocation for bug fixing:
~~~bash
/conductor-defect <app_name> ## For orchestrating bug fixing from BUG.md
~~~

- Input:
  - BUG.md (bug reports organized by module)
  - All context artifacts from the development phase (PRD.md, models, mockups, specifications, test specs)
- Flow:
  1. **Tag untagged bugs** — Assign sequential `[BUG-XXX]` tags to new bug entries in BUG.md
  2. **Create BUG_MASTER.md** — Master tracking checklist with all bugs, statuses and summary counts
  3. **For each bug** (processed sequentially):
     - **Reproduce** — Write and run a Playwright script to reproduce the bug, capture screenshot
     - **Write test spec** — Create `BUG_TEST_SPEC.md` with verification steps and expected results
     - **Analyze and plan** — Identify root cause, assess impact, create `BUG_FIX_PLAN.md` with fix checklist
     - **Apply fix** — Make code changes as planned, log progress in fix plan
     - **Verify fix** — Run Playwright verification script, capture post-fix screenshot
     - **Update artifacts** — Sync affected mockups, specifications, models and PRD.md with the fix
  4. **Completion** — Update BUG_MASTER.md status to COMPLETED when all bugs are resolved
- Output:
  - **BUG_MASTER.md** — Master tracking file with bug statuses (NEW, IN_PROGRESS, FIXED, CANNOT_REPRODUCE, HIGH_IMPACT)
  - **Per-bug folder** (`<app_folder>/context/bug/<module-slug>/<BUG-XXX>/`) containing:
    - `reproduce.spec.ts` — Playwright reproduction script
    - `screenshot_reproduce.png` — Pre-fix screenshot
    - `BUG_TEST_SPEC.md` — Test specification for verification
    - `BUG_FIX_PLAN.md` — Fix plan with checklist, files to modify, and timestamped fix log
    - `screenshot_fixed.png` — Post-fix screenshot

Example of the BUG_MASTER.md structure:
~~~markdown
- context
  - bug
    - <module_1_folder>
      - <BUG-001>
        - reproduce.spec.ts
        - screenshot_reproduce.png
        - BUG_TEST_SPEC.md
        - BUG_FIX_PLAN.md
        - screenshot_fixed.png
      - <BUG-002>
        ...
    - <module_2_folder>
      - <BUG-003>
        ...
    BUG_MASTER.md ## Master tracking file for all bugs in the application
~~~

# Directory Structure
- Each application will have its own root-level folder containing both context and source code.
- The context folder will contain all the artifacts generated in the Context Window Preparation Phase and Development Phase
- The source code files will be generated in the root-level folder of the application.
- The CLAUDE.md and SECRET.md will be in the root-level folder of the project, and they will be used as reference for all the applications in the project.
- Example of the directory structure:

~~~
- <project_folder>
  - CLAUDE.md
  - SECRET.md
  - <application_1_folder>
    - context
      - model
        - <module_1_folder>
          - model.md
          - erd.mermaid
        - <module_2_folder>
          - model.md
          - erd.mermaid
      - mockup
        - <role_1_folder>
          - screen1.html
          - screen2.html
        - <role_2_folder>
          - screen1.html
          - screen2.html
        MOCKUP.html
      - specification
        - <module_1_folder>
          - SPEC.md
        - <module_2_folder>
          - SPEC.md
        SPECIFICATION.md
      - test
        - <module_1_folder>
          - TEST_SPEC.md
        - <module_2_folder>
          - TEST_SPEC.md
        TEST_PLAN.mmd
      - develop
      - bug
      - PRD.md
      - BUG.md
    - (source code files)
~~~

# Disclaimer
- Currently only Claude Code Agent is supported, but we are planning to support more AI coding agents in the future.
- This workflow can consume a lot  of tokens. It is design to generate consistent and comprehensive context for the AI coding agent, which may require a large amount of tokens. Please be mindful of the token usage when using this workflow.
- This is workflow is currently designed for new application development, not maintenance of existing application. We are planning to support maintenance of existing application in the future, but for now please use this workflow for new application development only.