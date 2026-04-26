# Context
- This document is for coding agent to understand the project as a whole, including the project detail, goals and terminology.
- This document will be used as reference in any of the tasks related to this project.

# Project Detail
- Project Code: C02SKILLS
- Project Name: Compound Context Skills
- Project Description:  
  - This is an Open Source software development workflow using AI coding agent.
  - This workflow is an OPINIONATED approach which utilizes User Stories, Non-Functional Requirements and Constraints as the main input.
  - This workflow supports versioning and updating of the context as the project progresses.
  - From the User Stories, Non-Functional Requirements and Constraints, AI coding agent with the use of specific skills will generate:
    - Data Models
    - HTML Mockups
    - Technical Specifications
    - Test Specifications
    - Development Plan
  - Finally, the AI coding agent will develop the application based on all the context generated above.
  - After development, the AI coding agent will generate deployment artifacts (Dockerfile and deployment specification) for the application.
  - The workflow also covers bug fixing phase where:
    - User will report the bug with as much detail as possible.
    - AI coding agent will analyze the bug report and generate the bug fixing plan.
    - AI coding agent will fix the bug based on the generated plan and context from the development phase.

# Goals
- To reduce AI hallucination and AI slop by providing comprehensive and structured context for the AI coding agent.

# Terminology
- User Stories: The description of the features and functionalities of the system from the perspective of the end-users.
- Non-Functional Requirements: The description of the system's performance, security, usability and other non-functional aspects.
- Constraints: The description of the limitations and restrictions that the system must adhere to, such as technology stack, budget, timeline, etc.
- Data Models: The structured representation of the data used in the system, including entities, attributes and relationships.
- HTML Mockups: The visual representation of the user interface design for the application.
- Technical Specifications: The detailed description of the technical implementation of the application, including architecture, components, APIs, etc.
- Test Specifications: The detailed description of the test cases and scenarios to validate the functionality and performance of the application.

# Supporting 3rd Party Applications
- Below are the list of 3rd party applications which will be used as part of this project:
**No Supporting 3td Party Applications for this project since this is not an application development project**

# Custom Applications
- Below are the list of applications which will be custom developed as part of this project.:
**No Custom Applications for this project since this is not an application development project**

# Modules

## System Module
**There is no System Module for this project since this is not an application development project**

## Business Module
**There is no Business Module for this project since this is not an application development project**

# Folder structure
- The project folder structure is as follows:
  - .claude: Folder containing all the skills and tools for the compound-context workflow
  - skills: Folder containing Claude skills for compound-context methodology
    - util-ustagger: Tags untagged User Stories, NFRs, Constraints and References in PRD.md with unique 9-character ID codes. (Prerequisite: PRD.md)
    - util-usanalyzer: Analyzes PRD.md for quality issues such as incomplete stories, bad references, contradictions and duplicates. (Prerequisite: PRD.md)
    - util-projectsync: Validates dependencies and deployment environments, then syncs project folder structure, PRD.md, and BUG.md files based on applications and modules defined in CLAUDE.md. (Prerequisite: CLAUDE.md)
    - util-preparek8senv: Generates K8s StatefulSet manifests for all 3rd party supporting applications for a single target environment. Creates/updates ENVIRONMENT.md with per-environment configs. Output directly in environment/ folder (no per-environment subfolders — gitignored, each machine maintains its own copy). (Prerequisite: CLAUDE.md, DEVTOOL.md)
    - modelgen-relational: Extracts relational (SQL) entity models from user stories using Domain-Driven Design principles. (Prerequisite: PRD.md)
    - modelgen-nosql: Extracts NoSQL document models (MongoDB, Couchbase, DynamoDB, Firestore, CosmosDB) from user stories. (Prerequisite: PRD.md)
    - mockgen-tailwind: Generates HTML mockup screens served via Node.js + Express + HTMX. (Prerequisite: PRD.md, modelgen-relational or modelgen-nosql)
    - mockgen-shadcn: Generates React + shadcn/ui mockup screens served via Vite dev server with React Router. (Prerequisite: PRD.md, modelgen-relational or modelgen-nosql)
    - specgen-laravel-eloquent-bladehtmx: Generates Laravel 12 web application technical specification with Blade, Tailwind and htmx. (Prerequisite: PRD.md, modelgen-*, mockgen-tailwind)
    - specgen-react-mui: Generates React 19 SPA technical specification with TypeScript 5, Vite 6, Material UI v6, React Router v7, TanStack Query v5, Zustand v5, React Hook Form v7, and Zod v3. (Prerequisite: PRD.md, modelgen-*, mockgen-tailwind)
    - specgen-spring-jpa-jtehtmx: Generates Spring Boot 3 web application technical specification with JTE, Tailwind and htmx. (Prerequisite: PRD.md, modelgen-*, mockgen-tailwind)
    - specgen-spring-jpa-restapi: Generates Spring Boot 3 REST API technical specification. (Prerequisite: PRD.md, modelgen-*)
    - specgen-ts-cli: Generates Node.js CLI application technical specification with TypeScript, Commander.js, tsup, and @yao-pkg/pkg for cross-platform binary packaging. (Prerequisite: PRD.md)
    - specgen-sdk-java: Generates Java SDK library technical specification packaged as a Multi-Release fat JAR via Maven, supporting JDK 8 baseline plus JDK 11+ overlay. Uses OkHttp as the sole runtime third-party dependency; JSON, retry, and builders are hand-coded against the JDK. Scans PRD.md for the upstream Swagger UI URL or OpenAPI spec to drive method names, models, and URL constants. (Prerequisite: PRD.md)
    - specgen-react-mui: Generates React 19 SPA technical specification with TypeScript 5, Vite 6, Material UI v6, React Router v7, TanStack Query v5, Zustand v5. (Prerequisite: PRD.md, modelgen-*, mockgen-tailwind)
    - specgen-ts-cli: Generates Node.js CLI application technical specification with TypeScript, Commander.js, tsup, and @yao-pkg/pkg. (Prerequisite: PRD.md, modelgen-*)
    - testgen-functional: Generates Playwright E2E test plan and per-module test specifications as Markdown blueprints. (Prerequisite: PRD.md, modelgen-*, specgen-*, mockgen-tailwind)
    - depgen-k8s: Generates Dockerfile and Kubernetes manifests directly in <app_folder>/k8s/ for a single target environment. No per-environment subfolders — the k8s/ folder is gitignored, each machine maintains its own copy. Supports Spring Boot, Laravel, and Node.js stacks. (Prerequisite: SPECIFICATION.md, CLAUDE.md)
    - conductor-feature-prepare: Orchestrates the full artifact preparation pipeline — runs util-ustagger, modelgen-*, mockgen-tailwind, specgen-* and testgen-functional in sequence. (Prerequisite: PRD.md, CLAUDE.md)
    - conductor-feature-develop: Orchestrates full-stack application development module-by-module using all generated artifacts. (Prerequisite: conductor-feature-prepare output — model/, mockup/, specification/, test/)
    - conductor-defect: Orchestrates bug fixing from BUG.md — reproduces, fixes and verifies bugs and updates affected artifacts. (Prerequisite: BUG.md, conductor-feature-prepare output)
    - conductor-upgrade-version: Orchestrates a full version upgrade by running conductor-defect (bug fixing) followed by conductor-feature-develop (feature development) in sequence. (Prerequisite: BUG.md, conductor-feature-prepare output)
  - <app_folder>/CHANGELOG.md: Per-application changelog placed at the root of each application folder (e.g., `1_hub_middleware/CHANGELOG.md`). Tracks all skill executions by version for that application and serves as its version gate — skills reject execution for versions lower than the highest recorded version for that application. Auto-populated by skills after successful completion. Each application maintains its own independent version history.

# Path and Credentials
**There is no path and credentials for this project since this is not an application development project.**

# Rules
**There are no specific rules for this project since this is not an application development project.**