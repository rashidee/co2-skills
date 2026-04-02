# Domain Model Extraction from Agile User Stories

> **Version:** 1.1.0
> **Purpose:** Reusable instruction set for AI coding agents to systematically extract domain models, entities, attributes, and relationships from Agile-format user stories.
> **Usage:** Reference this document from your PRD. The AI agent will produce a domain model section to be appended to the PRD and a companion Mermaid ERD file sharing the same name as the PRD.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [Phase 1: Linguistic Analysis](#3-phase-1-linguistic-analysis)
4. [Phase 2: Entity Identification and Classification](#4-phase-2-entity-identification-and-classification)
5. [Phase 3: Attribute Extraction](#5-phase-3-attribute-extraction)
6. [Phase 4: Relationship Mapping](#6-phase-4-relationship-mapping)
7. [Phase 5: Cross-Cutting Concern Extraction](#7-phase-5-cross-cutting-concern-extraction)
8. [Phase 6: Bounded Context Identification](#8-phase-6-bounded-context-identification)
9. [Phase 7: Output Specification](#9-phase-7-output-specification)
10. [Phase 8: Validation and Review](#10-phase-8-validation-and-review)
11. [Appendix A: Verb-to-Relationship Heuristics](#appendix-a-verb-to-relationship-heuristics)
12. [Appendix B: NFR-to-Model Impact Map](#appendix-b-nfr-to-model-impact-map)

---

## 1. Overview

This document defines a **repeatable, auditable methodology** for extracting domain models from Agile user stories. It is designed to be consumed by an AI coding agent as part of a Product Requirements Document (PRD) pipeline.

The methodology follows **Domain-Driven Design (DDD)** principles and produces traceable outputs where every entity, attribute, and relationship maps back to one or more source user stories.

### Core Principles

- **Traceability** — Every model element MUST reference its originating user story or NFR.
- **Explicitness over assumption** — When a user story is ambiguous, the agent MUST flag it and state its assumption separately from confirmed facts.
- **Completeness** — The agent MUST consider user stories, NFRs, module definitions, role definitions, and stakeholder descriptions as input sources.
- **DDD Alignment** — Output must be organized by Bounded Contexts and Aggregates, not by database tables.

---

## 2. Prerequisites

Before executing this methodology, the AI agent must have access to the following inputs from the PRD:

| Input Artifact                       | Description                                                                                   | Required |
|--------------------------------------|-----------------------------------------------------------------------------------------------|----------|
| User Stories                         | Agile-format stories: "As [role], I want to [action] [object] so that [purpose]"              | Yes      |
| Module List                          | High-level feature groupings that serve as bounded context candidates                         | Yes      |
| Roles and Stakeholders               | Actor definitions with role codes                                                             | Yes      |
| Non-Functional Requirements (NFRs)   | Security, performance, integration, and compliance constraints                                | Yes      |
| Glossary / Ubiquitous Language       | Domain-specific term definitions (if available)                                               | No       |
| Existing Data Model                  | Current-state model for migration or extension scenarios                                      | No       |

---

## 3. Phase 1: Linguistic Analysis

### 3.1 Instruction

Agile user stories follow a structured grammar: **"As [role], I want to [action] [object] so that [purpose]."** This structure maps directly onto domain modeling concepts.

For **each** user story in the PRD, parse the sentence and map grammar elements to domain concepts:

| Grammar Element             | Domain Mapping                  | Extraction Rule                                                                                                          |
|-----------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| **Subject** (As [ROLE])     | Actor / Role Entity             | Extract the role code. This confirms a `Role` or similar authorization entity exists. Record the role as an enum value.   |
| **Verb** (I want to...)     | Operation / Command / Event     | Identifies the operation type. CRUD verbs imply entity lifecycle. State-change verbs (activate, deactivate, approve, reject) imply a state machine with an enum field. |
| **Direct Object**           | Primary Entity / Aggregate Root | The main noun after the verb is the **candidate entity**. If plural, it suggests a collection relationship.              |
| **Indirect Object**         | Related Entity / Association    | Nouns connected by prepositions (e.g., "X **to** Y", "X **for** Z") indicate relationships between entities.             |
| **Purpose Clause** (so that...) | Business Rule / Invariant / Domain Event | Reveals the **why** — often defines business rules, access control policies, or domain events.                       |
| **Adjectives / Qualifiers** | Attribute, State, or Type       | Descriptive words ("active", "pending", "approved") suggest enum values, boolean flags, or validation rules.             |
| **Prepositional Phrases**   | Relationship / Bounded Context  | Phrases like "access to X", "logs for Y" clarify context boundaries and entity ownership.                                |

### 3.2 Compound Story Handling

When a single story contains multiple verbs (e.g., "create, read, update, and delete"), decompose into individual operations and infer the model implications of each:

| Operation  | Model Implication                                                              |
|------------|--------------------------------------------------------------------------------|
| CREATE     | Entity lifecycle, ID generation, `createdAt`, `createdBy`                      |
| READ       | Query model, searchable attributes, pagination support                         |
| UPDATE     | `updatedAt`, `updatedBy`, `version` (optimistic locking)                       |
| DELETE     | `deletedAt` (soft delete) or cascade rules (hard delete) — determine from context |

---

## 4. Phase 2: Entity Identification and Classification

### 4.1 Classification Rules

After extracting all nouns from Phase 1, classify each into one of the following DDD building blocks:

| Classification       | Criteria                                                                                       |
|----------------------|------------------------------------------------------------------------------------------------|
| **Entity**           | Has a unique identity, a lifecycle (created, updated, deleted), and is persisted independently  |
| **Value Object**     | Immutable, no independent identity, describes a characteristic of an entity                    |
| **Aggregate Root**   | An entity that serves as the entry point for a cluster of related objects; owns transactional consistency |
| **Enum / Type**      | A finite, closed set of known values derived from qualifiers, states, or role codes             |
| **Domain Event**     | An occurrence of significance derived from state transitions or purpose clauses                 |
| **Domain Service**   | An operation that doesn't naturally belong to any entity, derived from complex verb phrases     |

### 4.2 Decision Flowchart

Apply this decision tree to each extracted noun:

```
Is it a finite, known set of values?
├── YES → ENUM
└── NO
    Does it have a unique identity and lifecycle?
    ├── YES
    │   Is it the root of a consistency boundary?
    │   ├── YES → AGGREGATE ROOT
    │   └── NO  → ENTITY (within an aggregate)
    └── NO
        Is it immutable and describes another entity?
        ├── YES → VALUE OBJECT
        └── NO
            Is it a verb/action that happened?
            ├── YES → DOMAIN EVENT
            └── NO  → DOMAIN SERVICE or requires further analysis
```

### 4.3 Naming Conventions

| Type             | Convention                | Example                              |
|------------------|---------------------------|--------------------------------------|
| Entity           | PascalCase, singular noun | `User`, `Order`, `Invoice`           |
| Value Object     | PascalCase, descriptive   | `Address`, `ContactInfo`, `Money`    |
| Enum             | PascalCase + contextual suffix | `OrderStatus`, `RoleType`       |
| Domain Event     | PascalCase, past tense    | `OrderPlaced`, `UserRegistered`      |
| Join Entity      | PascalCase, compound noun | `UserRole`, `OrderItem`              |

---

## 5. Phase 3: Attribute Extraction

### 5.1 Attribute Sources

Attributes are derived from **four sources**, in order of priority:

| Priority | Source                     | Description                                                                                    |
|----------|----------------------------|------------------------------------------------------------------------------------------------|
| 1        | **Explicit Mention**       | Attributes directly stated in the user story text                                              |
| 2        | **Operation Inference**    | Attributes implied by the operations performed (see Section 3.2)                               |
| 3        | **NFR Requirements**       | Attributes required by non-functional requirements (see Appendix B)                            |
| 4        | **Domain Convention**      | Standard attributes expected by convention (audit fields, soft delete flag, optimistic locking) |

### 5.2 Attribute Specification

For each attribute, the agent MUST capture:

| Field           | Description                                                                            |
|-----------------|----------------------------------------------------------------------------------------|
| `name`          | camelCase field name                                                                   |
| `type`          | Data type (String, UUID, Instant, Boolean, Long, custom Enum, or Value Object). UUID primary keys MUST specify DB-generated constraint. |
| `nullable`      | Whether the field can be null                                                          |
| `defaultValue`  | Default value if applicable                                                            |
| `constraints`   | Validation constraints (not null, unique, max length, pattern, etc.)                   |
| `description`   | Brief description of the attribute's purpose                                           |
| `sourceStory`   | Traceability reference to the originating user story ID                                |
| `sourceType`    | One of: `EXPLICIT`, `OPERATION_INFERENCE`, `NFR`, `CONVENTION`                         |

### 5.3 Implicit Attribute Extraction Rules

When a user story implies operations but does not explicitly name attributes, apply these inference rules:

| Story Pattern                            | Implied Attributes                                                                      |
|------------------------------------------|-----------------------------------------------------------------------------------------|
| "create X"                               | `id` (UUID, DB-generated), `createdAt` (Instant), `createdBy` (String)                   |
| "update X"                               | `updatedAt` (Instant), `updatedBy` (String), `version` (Long)                           |
| "delete X"                               | `deletedAt` (Instant), `deletedBy` (String), `deleted` (Boolean) — for soft delete      |
| "deactivate / reactivate X"             | `status` (Enum), `statusChangedAt` (Instant), `statusChangedBy` (String)                 |
| "assign X to Y"                          | Join entity with `assignedAt` (Instant), `assignedBy` (String)                           |
| "view X logs / activity"                 | Audit entity with `action`, `timestamp`, `actorId`, `entityType`, `entityId`, `details`  |
| "authenticate / log in"                  | `lastLoginAt`, `failedLoginAttempts`, `lockedUntil`                                      |
| "reset password"                         | `passwordChangedAt`, `passwordResetToken`, `passwordResetExpiry`                         |
| "update my profile"                      | Profile value object or embedded fields (flag as ambiguous — see 5.4)                    |
| "approve / reject X"                     | `approvalStatus` (Enum), `approvedBy`, `approvedAt`, `rejectionReason`                   |
| "schedule X"                             | `scheduledAt`, `startDate`, `endDate`                                                    |
| "search / filter X"                      | Implies indexed or searchable attributes; note for query model design                    |
| "upload / attach X"                      | `fileName`, `contentType`, `storageKey`, `fileSize`                                      |
| "notify X about Y"                       | `Notification` entity with `recipientId`, `type`, `message`, `readAt`, `sentAt`          |

### 5.4 Ambiguity Handling

When a user story contains vague or underspecified terms, the agent MUST:

1. **Flag** the ambiguity explicitly in the output.
2. **Propose** a reasonable default set of attributes based on domain conventions.
3. **Mark** each proposed attribute with `sourceType: ASSUMPTION`.
4. **Recommend** clarification questions to ask the product owner.

Output format for ambiguities:

| Story ID | Ambiguous Term | Issue Description | Proposed Default | Clarification Needed |
|----------|----------------|-------------------|------------------|----------------------|
| (ref)    | (term)         | (what is unclear) | (proposed attrs) | (question to ask PO) |

---

## 6. Phase 4: Relationship Mapping

### 6.1 Relationship Detection from Linguistic Cues

The agent MUST scan for verb and preposition patterns to identify relationships between entities:

| Linguistic Pattern                  | Relationship Type     | Cardinality    |
|-------------------------------------|-----------------------|----------------|
| "assign X to Y"                    | Association            | Many-to-Many   |
| "X belongs to Y"                   | Composition/Ownership  | Many-to-One    |
| "X has Y" / "X contains Y"        | Aggregation            | One-to-Many    |
| "X's Y" (possessive)              | Composition            | One-to-One     |
| "create X for Y"                   | Ownership              | Many-to-One    |
| "view X of Y" / "view X for Y"    | Read relationship      | One-to-Many    |
| "X manages Y"                      | Hierarchical           | One-to-Many    |
| "X is classified by Y"            | Categorization         | Many-to-One    |
| "X references Y"                   | Weak association       | Many-to-One    |

### 6.2 Relationship Specification

For each identified relationship, the agent MUST capture:

| Field              | Description                                                        |
|--------------------|--------------------------------------------------------------------|
| `sourceEntity`     | The entity on the owning side                                      |
| `targetEntity`     | The entity on the referenced side                                  |
| `type`             | `ONE_TO_ONE`, `ONE_TO_MANY`, `MANY_TO_ONE`, `MANY_TO_MANY`        |
| `joinEntity`       | Name of join entity (for Many-to-Many relationships only)          |
| `ownerSide`        | Which entity owns the foreign key                                  |
| `cascadeBehavior`  | What happens on delete/update of the parent                        |
| `sourceStory`      | Traceability reference                                             |
| `businessRule`     | Plain-language description of the relationship constraint          |

### 6.3 Cardinality Decision Matrix

| Question                                                    | If YES                 | If NO              |
|-------------------------------------------------------------|------------------------|---------------------|
| Can one instance of A be associated with many B?            | A → B is One-to-Many   | Check reverse       |
| Can one instance of B be associated with many A?            | B → A is One-to-Many   | One-to-One          |
| Can both A and B have many of each other?                   | Many-to-Many           | —                   |
| Does A cease to exist without B?                            | Composition (strong)   | Association (weak)  |
| Does the relationship itself carry data (e.g., timestamps)? | Promote to join entity | Use simple FK/join  |

---

## 7. Phase 5: Cross-Cutting Concern Extraction

### 7.1 Instruction

The agent MUST scan the **Non-Functional Requirements** section of the PRD and map each requirement to its impact on the domain model. See **Appendix B** for a comprehensive mapping table.

### 7.2 Base Entity Pattern

If audit fields, soft delete, or optimistic locking are implied by NFRs or user stories, the agent SHOULD define a base entity specification that all domain entities inherit from. At minimum, this base entity should include:

| Field         | Type     | Purpose                                |
|---------------|----------|----------------------------------------|
| `id`          | UUID     | Primary key (MUST be DB-generated — see note below) |
| `version`     | Long     | Optimistic locking                     |
| `createdAt`   | Instant  | Record creation timestamp              |
| `createdBy`   | String   | Actor who created the record           |
| `updatedAt`   | Instant  | Last modification timestamp            |
| `updatedBy`   | String   | Actor who last modified the record     |
| `deleted`     | Boolean  | Soft delete flag (if applicable)       |
| `deletedAt`   | Instant  | Soft delete timestamp (if applicable)  |
| `deletedBy`   | String   | Actor who deleted (if applicable)      |

> **UUID Generation Rule:** The `id` primary key on every entity MUST use database-level UUID generation (e.g., `gen_random_uuid()` in PostgreSQL, `UUID()` in MySQL, `NEWSEQUENTIALID()` in SQL Server). Application-generated UUIDs are NOT permitted. Database-generated UUIDs avoid performance overhead from network round-trips, ensure consistent generation across application instances, and allow the database to use optimized sequential UUID variants for better B-tree index performance.

---

## 8. Phase 6: Bounded Context Identification

### 8.1 Context Boundary Rules

Group entities into Bounded Contexts based on these heuristics:

| Heuristic                          | Application                                                                            |
|------------------------------------|----------------------------------------------------------------------------------------|
| **Module headers**                 | Module names from the PRD are primary candidates for bounded contexts                  |
| **Shared actors**                  | Stories with the same actor role often belong to the same context                       |
| **Transactional cohesion**         | Entities that MUST be modified together in one transaction belong in the same aggregate |
| **Independent lifecycle**          | Entities that can be created/deleted independently MAY belong in different contexts     |
| **Ubiquitous language boundary**   | If the same word means different things in different modules, those are separate contexts |

### 8.2 Context Map

The agent MUST produce a context map showing each bounded context with its contained aggregates and the relationship type between contexts:

| Relationship Type          | Description                                                    |
|----------------------------|----------------------------------------------------------------|
| **Shared Kernel**          | Shared model between contexts                                  |
| **Customer-Supplier**      | Upstream/downstream dependency                                 |
| **Conformist**             | Downstream conforms to upstream's model                        |
| **Anti-Corruption Layer**  | Translation layer between contexts                             |
| **Published Language**     | Shared API contract                                            |

---

## 9. Phase 7: Output Specification

The agent MUST produce **two deliverables**:

### 9.1 Domain Model Section (appended to the PRD markdown)

The following subsections MUST be added to the PRD document:

#### 9.1.1 Entity Catalog

A markdown table listing all extracted entities:

| Column             | Description                                          |
|--------------------|------------------------------------------------------|
| Entity             | Entity name (PascalCase)                             |
| DDD Type           | Aggregate Root, Entity, Value Object, Enum, or Join Entity |
| Bounded Context    | Which context/module this entity belongs to          |
| Key Attributes     | Primary attributes (comma-separated summary)         |
| Relationships      | Summary of relationships with cardinality notation   |
| Source Stories      | List of user story IDs this entity was derived from  |

#### 9.1.2 Attribute Detail

For each entity in the catalog, a dedicated attribute table containing:

| Column        | Description                                                      |
|---------------|------------------------------------------------------------------|
| Attribute     | camelCase field name                                             |
| Type          | Data type                                                        |
| Nullable      | Yes / No                                                         |
| Constraints   | Validation rules                                                 |
| Source        | `EXPLICIT`, `OPERATION_INFERENCE`, `NFR`, `CONVENTION`, or `ASSUMPTION` |
| Source Story  | Traceability reference                                           |

#### 9.1.3 Relationship Catalog

A markdown table listing all relationships:

| Column           | Description                                               |
|------------------|-----------------------------------------------------------|
| Source Entity    | Owning-side entity                                         |
| Target Entity    | Referenced entity                                         |
| Cardinality      | `1:1`, `1:N`, `N:1`, `M:N`                               |
| Join Entity      | Name of join entity (M:N only)                            |
| Cascade Behavior | What happens on parent delete/update                      |
| Business Rule    | Plain-language constraint description                     |
| Source Story     | Traceability reference                                    |

#### 9.1.4 Enum Definitions

A table for each enum listing its values, descriptions, and source stories.

#### 9.1.5 Domain Events

| Column          | Description                                             |
|-----------------|---------------------------------------------------------|
| Event Name      | PascalCase, past tense                                  |
| Trigger Story   | User story that implies this event                      |
| Aggregate       | The aggregate root that emits the event                 |
| Payload Fields  | Key data carried by the event                           |

#### 9.1.6 Bounded Context Map

A summary table showing context names, contained aggregates, and inter-context relationship types.

#### 9.1.7 Assumptions and Ambiguities

| Column               | Description                                           |
|----------------------|-------------------------------------------------------|
| Story ID             | Reference to the ambiguous user story                 |
| Entity / Attribute   | The model element affected                            |
| Assumption Made      | What the agent assumed                                |
| Clarification Needed | Question to resolve with the product owner            |

### 9.2 ERD Mermaid File

The agent MUST produce a **Mermaid ER diagram file** with the **same base name as the PRD file** and the `.mermaid` extension.

For example, if the PRD is named `my-project-prd.md`, the ERD file MUST be named `my-project-prd.mermaid`.

The Mermaid file MUST:

- Use the `erDiagram` diagram type.
- Include all entities from the entity catalog.
- Show all attributes with their types for each entity.
- Show all relationships with cardinality notation and relationship labels.
- Use Mermaid's standard cardinality markers:

| Marker   | Meaning          |
|----------|------------------|
| `\|\|`   | Exactly one      |
| `o\|`    | Zero or one      |
| `\|{`    | One or more      |
| `o{`     | Zero or more     |

Example structure (for illustration only — do not copy literally into output):

```
erDiagram
    ENTITY_A {
        uuid id PK
        string name
        string status
        timestamp created_at
    }
    ENTITY_B {
        uuid id PK
        uuid entity_a_id FK
        string description
    }
    ENTITY_A ||--o{ ENTITY_B : "has"
```

---

## 10. Phase 8: Validation and Review

After generating the model, the agent MUST perform a self-review.

### 10.1 Completeness Checklist

- [ ] Every user story has at least one corresponding entity.
- [ ] Every entity has at least one source user story.
- [ ] Every entity has an `id` field and audit fields.
- [ ] Every state-change verb has a corresponding enum and transition rules.
- [ ] Every "assign X to Y" pattern has a join entity or join table.
- [ ] Every NFR has been analyzed for model impact.
- [ ] All ambiguities are flagged with proposed defaults.

### 10.2 Quality Checklist

- [ ] No aggregate contains more than 4–5 entities (split if larger).
- [ ] No entity has more than 15–20 attributes (decompose into value objects).
- [ ] All Many-to-Many relationships have clearly defined join entities.
- [ ] Enum values match the role codes and states defined in the PRD.
- [ ] Naming conventions are consistent across all entities and attributes.
- [ ] The ERD Mermaid diagram is valid and parseable.

### 10.3 Refinement Pass

After initial extraction, the agent SHOULD run a second pass reviewing for:

1. Missing inverse relationships.
2. Aggregates that are too large (more than 4 entities).
3. Attributes that should be Value Objects instead of primitives.
4. Missing domain events for state transitions.
5. Naming consistency across the entire model.
6. Redundant or duplicate entities across bounded contexts.

---

## Appendix A: Verb-to-Relationship Heuristics

| Verb Pattern                      | Relationship Type          | Model Output                                                |
|-----------------------------------|----------------------------|-------------------------------------------------------------|
| `assign X to Y`                  | Many-to-Many               | Join entity with audit fields                               |
| `create X`                       | Entity lifecycle            | Entity with full CRUD and audit fields                      |
| `view X logs`                    | One-to-Many                | Audit/log entity with FK to parent                          |
| `deactivate / reactivate X`      | State machine               | `status` enum + `statusChangedAt` + transition rules        |
| `reset X`                        | Command / Domain Event      | Event sourcing candidate                                    |
| `authenticate / log in`          | Identity boundary           | Credential value object or entity                           |
| `approve / reject X`             | Workflow / State machine    | `approvalStatus` enum + `approvedBy` + `approvedAt`         |
| `search / filter X`              | Query model                 | Consider read model; add indexes                            |
| `import / export X`              | Batch operation              | Batch job entity with status tracking                       |
| `configure / set up X`           | Configuration entity        | Configuration entity with key-value pairs                   |
| `classify / categorize X`        | Categorization              | Classification entity with Many-to-One from subject         |
| `schedule X`                     | Temporal entity              | `scheduledAt`, `startDate`, `endDate` fields                |
| `notify X about Y`              | Event-driven                | Notification entity + template entity                       |
| `upload / attach X`             | File association             | Stored file entity with metadata                            |
| `X manages Y`                   | Hierarchical                | One-to-Many parent-child with self-referencing FK possible  |

---

## Appendix B: NFR-to-Model Impact Map

| NFR Category               | Common NFR Statement                           | Entity / Attribute Impact                                                     |
|----------------------------|-------------------------------------------------|-------------------------------------------------------------------------------|
| **OAuth2 / OIDC**          | Integration with external identity provider     | `externalIdpId`, `tokenSubject`, `issuer` on the user entity                  |
| **MFA / 2FA**              | Multi-factor authentication requirement         | MFA configuration value object: `mfaEnabled`, `mfaType`, `mfaSecret`         |
| **Password Policy**        | Strong passwords, expiry, history               | Password policy entity/VO, `passwordChangedAt`, `passwordHistory`             |
| **Audit Trail**            | Audit logging, compliance requirements          | Audit log entity, auditable base entity                                       |
| **Soft Delete**            | Data retention, compliance                      | `deleted`, `deletedAt`, `deletedBy` fields on base entity                     |
| **Multi-Tenancy**          | Tenant isolation                                | `tenantId` field, tenant-aware interface, row-level security                  |
| **Localization / i18n**    | Multi-language support                          | `locale` field, translated text value object or translation table             |
| **Versioning**             | Optimistic locking                              | `version` field (Long) on base entity                                         |
| **Rate Limiting**          | API throttling                                  | Rate limit policy entity per API key or user                                  |
| **Data Encryption**        | PII encryption at rest                          | Attribute-level encryption on sensitive fields                                |
| **GDPR / Privacy**         | Right to erasure, consent tracking              | Consent record entity, `anonymized` flag, data retention policy               |
| **File Storage**           | Document or image uploads                       | Stored file entity: `fileName`, `contentType`, `storageKey`, `fileSize`       |
