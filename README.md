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

# Workflow

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
    - SPEC.md containig the 

### Test Specification Generation

### Application Development

## Bug Fixing Phase

## Bug Reporting

## Bug Analysis and Bug Fixing Plan Generation

## Bug Fixing

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
      - specification
      - test
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