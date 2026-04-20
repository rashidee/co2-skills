# Specification Template — REST API

This is the authoritative template for the generated specification. The specification is
split into **two types of files**:

1. **`SPECIFICATION.md`** (root) — Table of Contents, shared infrastructure, and all
   application-level sections. Generated once per application.
2. **`<module>/SPEC.md`** (per-module) — Self-contained module blueprint.
   Generated once per module from PRD.md.

```
spec/
├── SPECIFICATION.md
├── job-demand/
│   └── SPEC.md
├── candidate-registration/
│   └── SPEC.md
└── ...
```

Placeholders use `{{VARIABLE}}` syntax and must be replaced with actual values gathered
from context files.

Sections marked **[CONDITIONAL]** should only be included when the corresponding
integration is selected.

**CRITICAL: Sample code requirement.** Every component in the spec MUST include a complete,
continuous code sample in a fenced code block (Java, YAML, or JSON). The code must be
self-explanatory and directly usable as a reference for a coding agent. Do not describe
components with bullet points alone — always accompany descriptions with full code.

---

# Part A: Root SPECIFICATION.md

Everything from this point until "Part B" goes into the root `SPECIFICATION.md` file.
A coding agent implements these shared sections **first**, before any module work.

---

## Table of Contents

Generate a TOC with clickable Markdown anchor links to every H2 and H3 section in this
file. Additionally, include a **Modules** section in the TOC that links to each
module's `SPEC.md` file:

```markdown
## Table of Contents

### Shared Infrastructure
- [1. Project Overview](#1-project-overview)
- [2. Maven Configuration](#2-maven-configuration)
- ...all shared sections...

### Modules
- [Job Demand](job-demand/SPEC.md)
- [Candidate Registration](candidate-registration/SPEC.md)
- ...(one link per module)...
```

---

## Section 1: Project Overview

```
# {{APPLICATION_NAME}} — Technical Specification

## 1. Project Overview

**Application Name**: {{APPLICATION_NAME}}
**Artifact ID**: {{ARTIFACT_ID}}
**Group ID**: com.bestinet.urp
**Base Package**: {{BASE_PACKAGE}}
**Java Version**: 21
**Spring Boot Version**: 3.5.7
**Description**: {{APP_DESCRIPTION}}
**Versions Covered**: v1.0.0 — v{{LATEST_VERSION}}
**API Version Prefix**: /api/v1

### Optional Components (Auto-Determined)
**Database**: {{DATABASE}}
**Authentication**: {{AUTH}}
**Scheduling**: {{SCHEDULING}}
**Messaging**: {{MESSAGING}}

### Technology Stack
(Render the core stack version table from SKILL.md, plus selected optional integration versions)

### API Consumers / Roles
(List each role that consumes this API, with a role-to-module mapping table.)

| Role | Modules | Spring Security Role |
|------|---------|---------------------|
| HC Employer Portal | Job Demand, Employer, ... | Internal service (no role) |
| Hub Middleware | All | `HUB_SERVICE` |

### Modules
| Module | Type | Stories | Versions | Spec |
|--------|------|---------|----------|------|
| Job Demand | Business | 12 | 1.0.0 | [SPEC](job-demand/SPEC.md) |
| ... | ... | ... | ... | ... |

### Input Sources
- CLAUDE.md: {{path}}
- PRD.md: {{path}}
- Module Model: {{path to MODEL.md}}
```

---

## Section 2: Maven Configuration

Generate a complete `pom.xml` inside a code block. Include:
- Parent: `spring-boot-starter-parent` 3.5.7
- Properties: java version, mapstruct version, lombok-mapstruct-binding version, springdoc version
- Dependencies:
  - **Web**: `spring-boot-starter-web`
  - **[If Database = MongoDB]** `spring-boot-starter-data-mongodb`
  - **[If Database = PostgreSQL]** `spring-boot-starter-data-jpa`, `org.postgresql:postgresql`, `org.flywaydb:flyway-core`
  - **[If Database = MySQL]** `spring-boot-starter-data-jpa`, `com.mysql:mysql-connector-j`, `org.flywaydb:flyway-core`, `org.flywaydb:flyway-mysql`
  - **[If Auth = Keycloak]** `spring-boot-starter-security`, `spring-boot-starter-oauth2-resource-server`
  - **[If Auth = JWT]** `spring-boot-starter-security`, `io.jsonwebtoken:jjwt-api/impl/jackson`
  - **Modulith**: `spring-modulith-starter-core`, `spring-modulith-events-api`
  - **[If Database = MongoDB]** `spring-modulith-starter-mongodb`
  - **[If Database = PostgreSQL/MySQL]** `spring-modulith-starter-jpa`
  - **[If Scheduling = yes]** `spring-boot-starter-quartz`
  - **[If Messaging = yes]** `spring-boot-starter-amqp`
  - **API Documentation**: `org.springdoc:springdoc-openapi-starter-webmvc-ui`
  - **Utilities**: `lombok`, `mapstruct`, `mapstruct-processor`
  - **DevTools**: `spring-boot-devtools` (runtime scope)
  - **Testing**: `spring-boot-starter-test`, `spring-modulith-starter-test`
  - **[If Auth = Keycloak or JWT]** `spring-security-test`
- Build plugins:
  - `spring-boot-maven-plugin`
  - `maven-compiler-plugin` with annotation processor paths for Lombok + MapStruct

Specify the `dependencyManagement` section for Spring Modulith BOM.

---

## Section 3: Application Configuration

### application.yml (default profile)

**IMPORTANT: Environment variable externalization.** All configuration values that may
change between environments (ports, hostnames, credentials, URIs) MUST use Spring's
`${ENV_VAR:default}` syntax. This ensures the application works out-of-the-box in local
development while allowing production deployments to override via environment variables.

**Always include:**
```yaml
server:
  port: ${SERVER_PORT:{{SERVER_PORT}}}

spring:
  application:
    name: {{ARTIFACT_ID}}
  modulith:
    events:
      republish-outstanding-events-on-restart: true
  jackson:
    default-property-inclusion: non_null
    serialization:
      write-dates-as-timestamps: false

springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: method
    tags-sorter: alpha

app:
  cors:
    allowed-origins: ${APP_CORS_ALLOWED_ORIGINS:http://localhost:3000}

logging:
  level:
    root: ${LOG_LEVEL_ROOT:WARN}
    {{BASE_PACKAGE}}: ${LOG_LEVEL_APP:INFO}
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] [%X{correlationId}] %-5level %logger{36} - %msg%n"
```

**[If Database = MongoDB] add:**
```yaml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI:mongodb://localhost:27017/{{DB_NAME}}}
      auto-index-creation: true
```

**[If Database = MySQL] add:**
```yaml
spring:
  datasource:
    url: ${DB_URL:jdbc:mysql://localhost:3306/{{DB_NAME}}}
    username: ${DB_USERNAME:{{DB_USER}}}
    password: ${DB_PASSWORD:{{DB_PASSWORD}}}
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
  flyway:
    enabled: true
    locations: classpath:db/migration
```

**[If Auth = Keycloak] add:**
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${KEYCLOAK_ISSUER_URI:{{KEYCLOAK_ISSUER_URI}}}
          jwk-set-uri: ${KEYCLOAK_JWK_SET_URI:${KEYCLOAK_ISSUER_URI:{{KEYCLOAK_ISSUER_URI}}}/protocol/openid-connect/certs}

app:
  security:
    keycloak-client-id: ${KEYCLOAK_CLIENT_ID:{{KEYCLOAK_CLIENT_ID}}}
    public-paths:
      - /api-docs/**
      - /swagger-ui/**
      - /swagger-ui.html
      - /actuator/health
      - /error
```

**[If Messaging = yes] add:**
```yaml
spring:
  rabbitmq:
    host: ${RABBITMQ_HOST:localhost}
    port: ${RABBITMQ_PORT:5672}
    username: ${RABBITMQ_USERNAME:guest}
    password: ${RABBITMQ_PASSWORD:guest}
    virtual-host: ${RABBITMQ_VHOST:/}

app:
  messaging:
    events-exchange: app.events.exchange
    commands-exchange: app.commands.exchange
    dead-letter-exchange: app.dlx.exchange
```

### application-dev.yml
Debug log levels, relaxed CORS.

### application-prod.yml
Externalize all sensitive config via environment variables, set conservative log levels.

---

## Section 4: `.gitignore`

```gitignore
# Build output
target/

# IDE
.idea/
*.iml
.vscode/
.settings/
.project
.classpath
*.swp
*~

# OS
.DS_Store
Thumbs.db

# Spring Modulith generated
**/build/generated-spring-modulith/

# Logs
*.log
logs/

# Environment
.env
.env.*
```

---

## Section 5: Package Structure

```
src/
├── main/
│   ├── java/
│   │   └── {{BASE_PACKAGE}}/
│   │       ├── Application.java
│   │       ├── config/
│   │       │   ├── JacksonConfig.java
│   │       │   ├── CorsConfig.java
│   │       │   ├── OpenApiConfig.java
│   │       │   ├── [If Auth = Keycloak] SecurityConfig.java
│   │       │   ├── [If Auth = Keycloak] KeycloakJwtGrantedAuthoritiesConverter.java
│   │       │   ├── [If Database = MongoDB] MongoConfig.java
│   │       │   ├── [If Database = PostgreSQL/MySQL] JpaConfig.java
│   │       │   ├── [If Scheduling = yes] QuartzConfig.java
│   │       │   └── [If Messaging = yes] RabbitMQConfig.java
│   │       ├── shared/
│   │       │   ├── exception/
│   │       │   │   ├── GlobalExceptionHandler.java
│   │       │   │   ├── ApiError.java
│   │       │   │   ├── ResourceNotFoundException.java
│   │       │   │   └── ConflictException.java
│   │       │   ├── model/
│   │       │   │   ├── [If Database = MongoDB] BaseDocument.java
│   │       │   │   └── [If Database = PostgreSQL/MySQL] BaseEntity.java
│   │       │   ├── security/                             # [If Auth != none]
│   │       │   │   ├── SecurityContextUtil.java
│   │       │   │   └── Roles.java
│   │       │   ├── filter/
│   │       │   │   └── CorrelationIdFilter.java
│   │       │   ├── audit/                                # [If Database != none]
│   │       │   │   └── AuditAwareImpl.java
│   │       │   ├── dto/
│   │       │   │   └── PageResponse.java                 # Optional simplified page DTO
│   │       │   └── [If Messaging = yes] messaging/       # INFRASTRUCTURE ONLY — no @RabbitListener here
│   │       │       ├── RabbitMQMessagingConfig.java      #   Exchange/queue/binding declarations
│   │       │       ├── RabbitMQPublisher.java            #   Generic publisher service
│   │       │       ├── MessageConverterConfig.java       #   Jackson2JsonMessageConverter bean
│   │       │       └── (event/command DTOs shared across modules, e.g., OrderExportedEvent.java)
│   │       │
│   │       ├── {{module1}}/                              # Module — PUBLIC API
│   │       │   ├── {{Module1}}Service.java               #   Public service interface
│   │       │   ├── {{Module1}}DTO.java                   #   Response DTO (record)
│   │       │   ├── Create{{Module1}}Request.java         #   Request DTO (record + validation)
│   │       │   ├── Update{{Module1}}Request.java         #   Request DTO (record + validation)
│   │       │   ├── {{Module1}}Exception.java             #   Public exception class
│   │       │   └── internal/                             #   INTERNAL
│   │       │       ├── {{Module1}}ServiceImpl.java       #     Service implementation
│   │       │       ├── [If Database = MongoDB] {{Module1}}Repository.java
│   │       │       ├── [If Database = MongoDB] {{Module1}}Document.java
│   │       │       ├── [If Database = PostgreSQL/MySQL] {{Module1}}Repository.java
│   │       │       ├── [If Database = PostgreSQL/MySQL] {{Module1}}Entity.java
│   │       │       ├── {{Module1}}Mapper.java            #     MapStruct mapper
│   │       │       ├── {{Module1}}Controller.java        #     @RestController (HTTP inbound adapter)
│   │       │       └── [If Messaging = yes] {{Module1}}EventConsumer.java  #  @RabbitListener (MQ inbound adapter — ALWAYS in module's internal/)
│   │       │
│   │       ├── {{module2}}/
│   │       │   └── (same public/internal structure)
│   │       │
│   │       └── [If Scheduling = yes] scheduling/
│   │           ├── internal/
│   │           │   └── job/
│   │           │       └── SampleQuartzJob.java
│   │           └── SchedulingService.java
│   │
│   └── resources/
│       ├── application.yml
│       ├── application-dev.yml
│       ├── application-prod.yml
│       ├── logback-spring.xml
│       ├── [If Database = PostgreSQL/MySQL] db/migration/
│       │   └── V1__initial_schema.sql
│       └── static/                                       # (empty, for future static assets)
│
└── test/
    └── java/
        └── {{BASE_PACKAGE}}/
            └── (test classes)
```

---

## Section 6: Security Configuration [CONDITIONAL]

Refer to `references/security-patterns.md` for the complete content to include.

**[If Auth = Keycloak (Resource Server)]:**
- `SecurityConfig.java` — `SecurityFilterChain` with `oauth2ResourceServer(jwt -> ...)`
- `KeycloakJwtGrantedAuthoritiesConverter.java` — extracting roles from JWT claims
- `SecurityContextUtil.java` — extracting user info from JWT
- `Roles.java` — role constants
- Stateless session policy, CORS, CSRF disabled
- Public vs. secured endpoint rules

---

## Sections 7-9: REST API Conventions, Error Handling, Pagination

Refer to `references/restapi-patterns.md` for the complete content to include.

Must cover:
- REST controller pattern with OpenAPI annotations
- Request/Response DTO patterns (Java records)
- Standardized `ApiError` response envelope
- `GlobalExceptionHandler` with `@RestControllerAdvice`
- `ResourceNotFoundException`, `ConflictException`
- Pagination with `Pageable` and `Page<T>` response
- Correlation ID filter
- Jackson configuration
- CORS configuration
- OpenAPI/SpringDoc configuration

---

## Sections 10-17: Data Access, API Docs, Logging, Scheduling, Events, Messaging, MapStruct, Testing

These sections follow the same patterns as the JTE variant where applicable. The key
differences are:

- **No frontend build tooling** (no Vite, Tailwind, Alpine.js, htmx)
- **No JTE templates** (no layouts, fragments, or view models)
- **No UI components** (no buttons, cards, tables, etc.)
- **Testing uses MockMvc** with JSON assertions instead of HTML/view assertions
- **API documentation** via SpringDoc OpenAPI (replaces HTML mockup screens)

---

# Part B: Module SPEC.md Template

Everything below is the template for each **`<module>/SPEC.md`** file. Generate one
file per module from PRD.md and MODEL.md.

Each module SPEC.md is **self-contained** — a coding agent picks it up and implements
the entire module independently (after the shared infrastructure from
`SPECIFICATION.md` is in place).

---

## Module SPEC.md Structure

```markdown
# {{ModuleName}} Module — Specification

> Part of [{{APPLICATION_NAME}} Technical Specification](../SPECIFICATION.md)

## Overview

**Module:** {{ModuleName}}
**Package:** {{BASE_PACKAGE}}.{{modulename}}
**Type:** {{System Module | Business Module}}
**API Base Path:** /api/v1/{{resource-plural}}
```

### Traceability

List all tagged IDs this module implements:

```markdown
## Traceability

### User Stories
| ID | Version | Description |
|---|---|---|
| USHC00012 | v1.0.0 | View list of job demands |

### Non-Functional Requirements
| ID | Version | Description |
|---|---|---|
| NFRHC0006 | v1.0.0 | Store in staging database |

### Constraints
| ID | Version | Description |
|---|---|---|
| CONSHC003 | v1.0.0 | Only ISO3 country codes |

### Removed / Replaced
| ID | Type | Removed In | Replaced By | Reason |
|---|---|---|---|---|
| _None._ | | | | |

### Tables / Collections
(flat list of table or collection names used by this module)
```

### API Endpoints

List all REST endpoints for this module:

```markdown
## API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/v1/job-demands | List job demands (paginated) | Authenticated |
| GET | /api/v1/job-demands/{id} | Get job demand by ID | Authenticated |
| POST | /api/v1/job-demands | Create a new job demand | ADMIN |
| PUT | /api/v1/job-demands/{id} | Update a job demand | ADMIN |
| DELETE | /api/v1/job-demands/{id} | Delete a job demand | ADMIN |
```

### Public API

Include complete code samples for:

- **Service Interface** — `{{ModuleName}}Service.java`
  Methods derived from user stories
- **Response DTO** — `{{ModuleName}}DTO.java` (Java record)
  Fields from `model/<module>/model.md`
- **Request DTOs** — `Create{{ModuleName}}Request.java`, `Update{{ModuleName}}Request.java`
  Java records with Bean Validation
- **Exception** — `{{ModuleName}}Exception.java`
  Module-specific exception with NotFound inner class

### Internal Implementation

Include complete code samples for:

- **[If MongoDB] Document** — `{{ModuleName}}Document.java`
- **[If PostgreSQL/MySQL] Entity** — `{{ModuleName}}Entity.java`
- **Repository** — `{{ModuleName}}Repository.java` with query methods
- **Mapper** — `{{ModuleName}}Mapper.java` with `@Mapper(componentModel = "spring")`
- **Service Implementation** — `{{ModuleName}}ServiceImpl.java` with full CRUD logic
- **REST Controller** — `{{ModuleName}}Controller.java` with all endpoints

### Critical Rules for Module SPEC.md

1. Use actual field names from module model files
2. Use actual table/collection names
3. Define service methods mapping to actual user stories (by tag ID)
4. Define REST endpoints mapping to actual user stories
5. List user story IDs, NFR IDs, and constraint IDs with versions
6. Include complete, continuous code samples
7. Follow all constraints (constructor injection, `@Getter @Setter` not `@Data`, records for DTOs, MapStruct, paginated lists)
8. Include "Removed / Replaced" subsection

**REST API URI rules:**
- Resource names are plural and kebab-case: `/api/v1/job-demands`
- Controller: `@RequestMapping("/api/v1/job-demands")`
- Role-based access via `@PreAuthorize` on controller class or methods
- `201 Created` with `Location` header for POST
- `204 No Content` for DELETE

```java
@RestController
@RequestMapping("/api/v1/job-demands")
@RequiredArgsConstructor
@Tag(name = "Job Demands", description = "Job demand management operations")
@PreAuthorize("hasRole('OPERATOR')")
class JobDemandController {
    // ...
}
```

If a module is accessible by multiple roles:
```java
@PreAuthorize("hasAnyRole('ADMIN', 'OPERATOR')")
```
