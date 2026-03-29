# Specification Template — Web Monolith

This is the authoritative template for the generated specification. The specification is
split into **two types of files**:

1. **`SPECIFICATION.md`** (root) — Table of Contents, shared infrastructure, and all
   application-level sections. Generated once per application.
2. **`<module>/SPEC.md`** (per-module) — Self-contained module blueprint.
   Generated once per module from PRD.md.

```
spec/
├── SPECIFICATION.md
├── location-information/
│   └── SPEC.md
├── corridor/
│   └── SPEC.md
└── ...
```

Placeholders use `{{VARIABLE}}` syntax and must be replaced with actual values gathered
from context files.

Sections marked **[CONDITIONAL]** should only be included when the corresponding
integration is selected. Within unconditional sections, some content blocks are
conditional — follow the inline guidance.

**CRITICAL: Sample code requirement.** Every component in the spec MUST include a complete,
continuous code sample in a fenced code block (Java, JTE, JavaScript, YAML, or HTML).
The code must be self-explanatory and directly usable as a reference for a coding agent.
Do not describe components with bullet points alone — always accompany descriptions with
full code.

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
- [Location Information](location-information/SPEC.md)
- [Corridor](corridor/SPEC.md)
- [Employer](employer/SPEC.md)
- ...(one link per module)...
```

Only include conditional sections that apply based on user selections.

---

## Section 1: Project Overview

```
# {{APPLICATION_NAME}} — Technical Specification

## 1. Project Overview

**Application Name**: {{APPLICATION_NAME}}   (from CLAUDE.md Custom Applications section)
**Artifact ID**: {{ARTIFACT_ID}}             (kebab-case of application name)
**Group ID**: com.bestinet.urp
**Base Package**: {{BASE_PACKAGE}}           (com.bestinet.urp.<artifactid>)
**Java Version**: 21
**Spring Boot Version**: 3.5.7
**Description**: {{APP_DESCRIPTION}}         (from CLAUDE.md application description)
**Versions Covered**: v1.0.0 — v{{LATEST_VERSION}}

### Optional Components (Auto-Determined)
**Database**: {{DATABASE}}       (from CLAUDE.md dependencies)
**Authentication**: {{AUTH}}     (from CLAUDE.md dependencies + PRD.md NFRs)
**Scheduling**: {{SCHEDULING}}   (from PRD.md NFRs)
**Messaging**: {{MESSAGING}}     (from CLAUDE.md dependencies)

### Technology Stack
(Render the core stack version table from SKILL.md, plus selected optional integration versions)

### User Roles
(List each user role extracted from mockup sidebar files, e.g., Hub Administrator, Hub Operation Support.
Include a role-to-module mapping table showing which modules each role can access.
This mapping drives `@PreAuthorize` annotations on controllers — NOT URL path prefixes.)

| Role | Modules | Spring Security Role |
|------|---------|---------------------|
| Hub Administrator | Location Information, Corridor, Recruitment Step, ... | `HUB_ADMINISTRATOR` |
| Hub Operation Support | Employer, Quota, Quota Allocation, ... | `HUB_OPERATION_SUPPORT` |

### Sidebar Navigation Items
(For each role, list the navigation items with module-based hrefs — NOT role-prefixed hrefs.)

**Hub Administrator:**
| Label | Href | Section |
|-------|------|---------|
| Location Information | `/location-information` | Geography |
| Corridor | `/corridor` | Administration |
| ... | ... | ... |

**Hub Operation Support:**
| Label | Href | Section |
|-------|------|---------|
| Employer | `/employer` | Operations |
| Quota | `/quota` | Operations |
| ... | ... | ... |

### Modules
(List each module from PRD.md organized by System Module and Business Module,
with description from the module heading and user story count from MODEL.md summary table.
Each module links to its dedicated SPEC.md file.)

| Module | Type | Stories | Versions | Spec |
|--------|------|---------|----------|------|
| Location Information | Business | 12 | 1.0.0, 1.0.1, 1.0.3 | [SPEC](location-information/SPEC.md) |
| Corridor | Business | 8 | 1.0.0, 1.0.3 | [SPEC](corridor/SPEC.md) |
| ... | ... | ... | ... | ... |

### Input Sources
(List the context files read to produce this specification)
- CLAUDE.md: {{path}}
- PRD.md: {{path}}
- Module Model: {{path to MODEL.md}}
- Mockup Screens: {{path to MOCKUP.html}}
```

---

## Section 2: Maven Configuration

Generate a complete `pom.xml` inside a code block. This section is critical — it must be
copy-pasteable and immediately functional.

Include:
- Parent: `spring-boot-starter-parent` 3.5.7
- Properties: java version, jte version, mapstruct version, lombok-mapstruct-binding version, frontend-maven-plugin version, node version
- Dependencies (with comments grouping them):
  - **Web**: `spring-boot-starter-web`
  - **Templating**: `gg.jte:jte-spring-boot-starter-3`, `gg.jte:jte`
  - **[If Database = MongoDB]** `spring-boot-starter-data-mongodb`
  - **[If Database = PostgreSQL]** `spring-boot-starter-data-jpa`, `org.postgresql:postgresql`, `org.flywaydb:flyway-core`
  - **[If Database = MySQL]** `spring-boot-starter-data-jpa`, `com.mysql:mysql-connector-j`, `org.flywaydb:flyway-core`, `org.flywaydb:flyway-mysql`
  - **[If Auth = Keycloak]** `spring-boot-starter-security`, `spring-boot-starter-oauth2-client`
  - **[If Auth = form]** `spring-boot-starter-security`
  - **Modulith**: `spring-modulith-starter-core`, `spring-modulith-events-api`
  - **[If Database = MongoDB]** `spring-modulith-starter-mongodb`
  - **[If Database = PostgreSQL/MySQL]** `spring-modulith-starter-jpa`
  - **[If Scheduling = yes]** `spring-boot-starter-quartz`
  - **[If Scheduling = yes AND Database = MongoDB]** `io.fluidsonic.mirror:fluidsonic-mirror-quartz`
  - **[If Scheduling = yes AND Spring Batch = yes]** `spring-boot-starter-batch`
  - **[If Scheduling = yes AND Spring Batch = yes AND Remote Partitioning = yes]** `spring-batch-integration`, `spring-integration-amqp`, `spring-boot-starter-amqp`
  - **[If Messaging = yes]** `spring-boot-starter-amqp` *(shared with Remote Partitioning — if both selected, include once)*
  - **Utilities**: `lombok`, `mapstruct`, `mapstruct-processor`
  - **DevTools**: `spring-boot-devtools` (runtime scope)
  - **Testing**: `spring-boot-starter-test`, `spring-modulith-starter-test`
  - **[If Auth = Keycloak or form]** `spring-security-test`
- Build plugins:
  - `spring-boot-maven-plugin` (exclude lombok)
  - `maven-compiler-plugin` with annotation processor paths for Lombok + MapStruct, and
    `<compilerArgs>` to set `-Aspring.modulith.metadata.output.directory=${project.basedir}/build/generated-spring-modulith`
    so Spring Modulith writes its metadata inside the module directory instead of the repo root
  - `frontend-maven-plugin` to run `npm ci && npm run build` in `frontend/` before `process-resources`
  - `maven-clean-plugin` configured with a `<fileset>` to delete `${project.basedir}/jte-classes`
    on `mvn clean` (this removes stale on-demand compiled JTE classes created at runtime in dev mode)
  - `gg.jte:jte-maven-plugin` for template precompilation:
    - Goal: `precompile` (generates AND compiles `.class` files, not just `.java` sources)
    - Phase: `process-classes` (must run after `compile` so application classes are available)
    - `<sourceDirectory>`: `${project.basedir}/src/main/jte`
    - `<targetDirectory>`: `${project.build.directory}/jte-classes` (inside `target/` so `mvn clean` removes it)
    - `<contentType>`: `Html`
    - **CRITICAL**: Do NOT use `${project.basedir}/jte-classes` as targetDirectory — that path is
      outside `target/` and `mvn clean` will not remove it, causing stale precompiled templates.

Specify the `dependencyManagement` section for Spring Modulith BOM.

---

## Section 3: Application Configuration

Generate complete configuration files for `src/main/resources/`.

The YAML content varies based on selected integrations. Include only sections relevant
to the user's choices.

### application.yml (default profile)

**IMPORTANT: Environment variable externalization.** All configuration values that may
change between environments (ports, hostnames, credentials, URIs) MUST use Spring's
`${ENV_VAR:default}` syntax. This ensures the application works out-of-the-box in local
development while allowing production deployments to override via environment variables.

**Always include:**
```yaml
server:
  port: ${SERVER_PORT:{{SERVER_PORT}}}
  servlet:
    context-path: /

spring:
  application:
    name: {{ARTIFACT_ID}}
  modulith:
    events:
      republish-outstanding-events-on-restart: true

gg:
  jte:
    developmentMode: ${JTE_DEV_MODE:false}
    usePrecompiledTemplates: ${JTE_PRECOMPILED:true}
    templateSuffix: .jte
    contentType: Html

spring:
  jte:
    templateLocation: src/main/jte
    templateSuffix: .jte
    contentType: Html

app:
  cors:
    allowed-origins: ${APP_CORS_ALLOWED_ORIGINS:http://localhost:3000}
  theme:
    default: light
    cookie-name: theme
    cookie-max-age: 31536000

logging:
  level:
    root: ${LOG_LEVEL_ROOT:WARN}
    {{BASE_PACKAGE}}: ${LOG_LEVEL_APP:INFO}
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] [%X{userId}] %-5level %logger{36} - %msg%n"
```

**[If Database = MongoDB] add:**
```yaml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI:mongodb://localhost:27017/{{DB_NAME}}}
      auto-index-creation: true

logging:
  level:
    org.springframework.data.mongodb: WARN
```

**[If Database = PostgreSQL] add:**
```yaml
spring:
  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/{{DB_NAME}}}
    username: ${DB_USERNAME:{{DB_USER}}}
    password: ${DB_PASSWORD:{{DB_PASSWORD}}}
    driver-class-name: org.postgresql.Driver
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
  flyway:
    enabled: true
    locations: classpath:db/migration

logging:
  level:
    org.springframework.data.jpa: WARN
    org.hibernate.SQL: WARN
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

logging:
  level:
    org.springframework.data.jpa: WARN
    org.hibernate.SQL: WARN
```

**[If Auth = Keycloak] add:**
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: ${KEYCLOAK_CLIENT_ID:{{KEYCLOAK_CLIENT_ID}}}
            client-secret: ${KEYCLOAK_CLIENT_SECRET:{{KEYCLOAK_CLIENT_SECRET}}}
            scope: openid,profile,email
        provider:
          keycloak:
            issuer-uri: ${KEYCLOAK_ISSUER_URI:{{KEYCLOAK_ISSUER_URI}}}

app:
  security:
    keycloak-client-id: ${KEYCLOAK_CLIENT_ID:{{KEYCLOAK_CLIENT_ID}}}
    public-paths:
      - /static/**
      - /assets/**
      - /error

logging:
  level:
    org.springframework.security: WARN
```

**[If Auth = form] add:**
```yaml
app:
  security:
    public-paths:
      - /static/**
      - /assets/**
      - /login
      - /register
      - /error

logging:
  level:
    org.springframework.security: WARN
```

**[If Scheduling = yes AND Database = MongoDB] add:**
```yaml
spring:
  quartz:
    job-store-type: mongodb
    properties:
      org.quartz.threadPool.threadCount: 5
      org.quartz.jobStore.class: io.fluidsonic.mirror.quartz.MongoDBJobStore
      org.quartz.jobStore.mongoUri: ${MONGODB_URI:mongodb://localhost:27017/{{DB_NAME}}}
      org.quartz.jobStore.dbName: {{DB_NAME}}
      org.quartz.jobStore.collectionPrefix: qrtz_
```

**[If Scheduling = yes AND Database = PostgreSQL/MySQL] add:**
```yaml
spring:
  quartz:
    job-store-type: jdbc
    jdbc:
      initialize-schema: always
    properties:
      org.quartz.threadPool.threadCount: 5
      org.quartz.jobStore.class: org.quartz.impl.jdbcjobstore.JobStoreTX
      org.quartz.jobStore.driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
```

**[If Scheduling = yes AND Database = none] add:**
```yaml
spring:
  quartz:
    job-store-type: memory
    properties:
      org.quartz.threadPool.threadCount: 5
```

**[If Scheduling = yes AND Spring Batch = yes AND Database = MongoDB] add:**
```yaml
spring:
  batch:
    jdbc:
      initialize-schema: never
    job:
      enabled: false
```

**[If Scheduling = yes AND Spring Batch = yes AND Database = PostgreSQL/MySQL] add:**
```yaml
spring:
  batch:
    jdbc:
      initialize-schema: always
    job:
      enabled: false
```

**[If Scheduling = yes AND Spring Batch = yes AND Database = none] add:**
```yaml
spring:
  batch:
    jdbc:
      initialize-schema: never
    job:
      enabled: false
```

**[If Scheduling = yes AND Spring Batch = yes AND Remote Partitioning = yes] add:**
```yaml
spring:
  rabbitmq:
    host: ${RABBITMQ_HOST:localhost}
    port: ${RABBITMQ_PORT:5672}
    username: ${RABBITMQ_USERNAME:guest}
    password: ${RABBITMQ_PASSWORD:guest}
    virtual-host: ${RABBITMQ_VHOST:/}

app:
  batch:
    partition:
      grid-size: 4
      request-queue: batch.partition.requests
      reply-queue: batch.partition.replies
      request-dlq: batch.partition.requests.dlq
      exchange: batch.partition.exchange
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
> **Note:** If Remote Partitioning is also selected, the `spring.rabbitmq.*` connection
> block is shared — include it only once. The batch partition queues (`batch.partition.*`)
> and messaging queues (`app.*`) use different exchanges and do not conflict.

### .env File — JTE Configuration

The `.env` file for local development **MUST** include JTE environment variables:

```properties
# --- JTE ---
JTE_DEV_MODE=true
JTE_PRECOMPILED=false
```

- `JTE_DEV_MODE=true`: JTE compiles templates on-the-fly from `.jte` sources (hot-reload during development)
- `JTE_PRECOMPILED=false`: Disables precompiled template lookup so changes to `.jte` files are reflected immediately

**For production** (no `.env` file), the defaults in `application.yml` apply: `JTE_DEV_MODE=false`,
`JTE_PRECOMPILED=true` — the Maven `precompile` goal generates the precompiled classes in `target/jte-classes`.

**CRITICAL — Stale JTE cache warning:** When `JTE_DEV_MODE=true`, JTE creates on-demand compiled
`.class` files in a `jte-classes/` directory at the project root. These cached classes can become
stale after template edits. The `maven-clean-plugin` configuration (Section 2) ensures `mvn clean`
deletes this folder. Developers should always run `mvn clean` before `spring-boot:run` if they
suspect stale templates. The `jte-classes/` folder MUST be listed in `.gitignore`.

### No Profile-Specific YAML Files

Do NOT generate `application-dev.yml` or `application-prod.yml`. All environment differences
are handled via environment variables with `${ENV_VAR:default}` syntax in `application.yml`.
JTE mode, log levels, and all other settings are controlled by the `.env` file locally and
system environment variables in production.

---

## Section 4: Build & Tooling

### 4.1 Frontend Directory Structure
```
frontend/
├── package.json
├── vite.config.js
├── tailwind.config.js
├── postcss.config.js
└── src/
    ├── css/
    │   ├── app.css                  # Main entry — imports all partials
    │   ├── base/
    │   │   ├── reset.css            # Tailwind @layer base
    │   │   └── typography.css       # Font imports, text defaults
    │   ├── components/
    │   │   ├── buttons.css          # @layer components — button styles
    │   │   ├── forms.css            # Input, select, textarea styles
    │   │   ├── cards.css            # Card component styles
    │   │   ├── tables.css           # Table styles
    │   │   ├── modals.css           # Modal/dialog styles
    │   │   └── navigation.css       # Nav, breadcrumb, tabs styles
    │   ├── layouts/
    │   │   ├── sidebar.css          # Sidebar responsive behavior
    │   │   ├── navbar.css           # Header/navbar styles
    │   │   └── footer.css           # Footer styles
    │   └── themes/
    │       ├── light.css            # Light theme CSS custom properties
    │       └── dark.css             # Dark theme CSS custom properties
    └── js/
        ├── app.js                   # Entry — registers Alpine, htmx, global stores
        ├── alpine/
        │   ├── stores/
        │   │   ├── toast.js         # Toast notification store
        │   │   ├── modal.js         # Modal state store
        │   │   ├── drawer.js        # Sidebar drawer store
        │   │   └── theme.js         # Theme toggle store
        │   └── components/
        │       ├── dropdown.js      # Dropdown behavior
        │       ├── tabs.js          # Tab switching
        │       ├── collapse.js      # Accordion/collapse
        │       └── carousel.js      # Carousel component
        └── htmx/
            ├── index.js             # Global error/loading handlers
            └── extensions/
                ├── toast-errors.js  # Map htmx errors to toast notifications
                └── loading-states.js # Toggle loading classes during requests
```

### 4.2 Vite Configuration
```js
// frontend/vite.config.js
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  root: resolve(__dirname, 'src'),
  build: {
    outDir: resolve(__dirname, '../src/main/resources/static/dist'),
    emptyOutDir: true,
    manifest: true,
    rollupOptions: {
      input: {
        app: resolve(__dirname, 'src/js/app.js'),
        styles: resolve(__dirname, 'src/css/app.css'),
      },
    },
  },
});
```

### 4.3 Tailwind Configuration
```js
// frontend/tailwind.config.js
export default {
  content: [
    '../src/main/jte/**/*.jte',
    './src/js/**/*.js',
  ],
  darkMode: ['class', '[data-theme="dark"]'],
  theme: {
    extend: {
      colors: {
        primary: { 50: '#eff6ff', /* ... full scale */ 900: '#1e3a5f' },
        accent: { 50: '#f0fdf4', /* ... full scale */ 900: '#14532d' },
      },
    },
  },
  plugins: [],
};
```

### 4.4 ViteManifestService
```java
package {{BASE_PACKAGE}}.shared.vite;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.annotation.PostConstruct;
import lombok.Getter;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.ClassPathResource;
import org.springframework.stereotype.Service;
import java.io.IOException;
import java.util.Map;

@Service
@Slf4j
public class ViteManifestService {

    @Value("${gg.jte.development-mode:false}")
    private boolean devMode;

    private Map<String, ManifestEntry> manifest;

    @PostConstruct
    void loadManifest() {
        if (devMode) {
            log.info("Vite dev mode — manifest not loaded");
            return;
        }
        try {
            var resource = new ClassPathResource("static/dist/.vite/manifest.json");
            var mapper = new ObjectMapper();
            manifest = mapper.readValue(resource.getInputStream(),
                new TypeReference<Map<String, ManifestEntry>>() {});
            log.info("Vite manifest loaded with {} entries", manifest.size());
        } catch (IOException e) {
            throw new IllegalStateException("Failed to load Vite manifest", e);
        }
    }

    public String resolveAsset(String entryName) {
        if (devMode) {
            return "http://localhost:5173/src/" + entryName;
        }
        var entry = manifest.get(entryName);
        if (entry == null) {
            throw new IllegalArgumentException("No manifest entry for: " + entryName);
        }
        return "/static/dist/" + entry.getFile();
    }

    @Getter
    static class ManifestEntry {
        private String file;
        private String src;
        private boolean isEntry;
        private String[] css;
    }
}
```

### 4.5 Maven Frontend Plugin
```xml
<!-- In pom.xml build/plugins -->
<plugin>
    <groupId>com.github.eirslett</groupId>
    <artifactId>frontend-maven-plugin</artifactId>
    <version>${frontend-maven-plugin.version}</version>
    <configuration>
        <workingDirectory>frontend</workingDirectory>
        <nodeVersion>v${node.version}</nodeVersion>
    </configuration>
    <executions>
        <execution>
            <id>install-node-and-npm</id>
            <goals><goal>install-node-and-npm</goal></goals>
        </execution>
        <execution>
            <id>npm-install</id>
            <goals><goal>npm</goal></goals>
            <configuration><arguments>ci</arguments></configuration>
        </execution>
        <execution>
            <id>npm-build</id>
            <goals><goal>npm</goal></goals>
            <phase>process-resources</phase>
            <configuration><arguments>run build</arguments></configuration>
        </execution>
    </executions>
</plugin>
```

---

## Section 4b: `.gitignore`

Generate a `.gitignore` file at the project root to exclude all generated, downloaded,
and environment-specific files from version control.

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

# Frontend
frontend/node_modules/
frontend/.vite/
src/main/resources/static/dist/

# JTE generated
**/jte-classes/

# Spring Modulith generated
**/build/generated-spring-modulith/

# E2E
e2e/node_modules/
e2e/test-results/
e2e/playwright-report/
e2e/visual-baselines/

# Logs
*.log
logs/

# Environment
.env
.env.*

# [If Scheduling = yes AND Spring Batch = yes] Batch temp files
# *.tmp

# [If Reporting = yes] Compiled JasperReports templates (regenerated at build time)
# *.jasper
```

> Remove the comment markers (`#`) and include the conditional entries only when
> their integration is selected. Delete the conditional blocks entirely when not selected.

---

## Section 5: Package Structure

Generate the full directory tree. The structure must follow Spring Modulith conventions
adapted for server-rendered web applications.

Adapt the config and shared packages based on user selections. Only include files relevant
to the chosen integrations.

```
src/
├── main/
│   ├── java/
│   │   └── {{BASE_PACKAGE}}/
│   │       ├── Application.java                          # @SpringBootApplication
│   │       ├── config/                                   # Shared configuration
│   │       │   ├── WebMvcConfiguration.java
│   │       │   ├── [If Auth = Keycloak] SecurityConfiguration.java
│   │       │   ├── [If Auth = Keycloak] KeycloakGrantedAuthoritiesMapper.java
│   │       │   ├── [If Auth = form] SecurityConfiguration.java
│   │       │   ├── [If Database = MongoDB] MongoConfiguration.java
│   │       │   ├── [If Database = PostgreSQL/MySQL] JpaConfiguration.java
│   │       │   ├── [If Scheduling = yes] QuartzConfig.java
│   │       │   ├── [If Scheduling = yes AND Spring Batch = yes] BatchConfig.java
│   │       │   ├── [If Remote Partitioning = yes] RabbitMQPartitionConfig.java
│   │       │   └── [If Remote Partitioning = yes] BatchIntegrationConfig.java
│   │       ├── shared/                                   # Shared kernel (cross-cutting)
│   │       │   ├── exception/
│   │       │   │   ├── GlobalExceptionHandler.java
│   │       │   │   ├── WebApplicationException.java
│   │       │   │   └── ErrorResponse.java
│   │       │   ├── model/
│   │       │   │   ├── [If Database = MongoDB] BaseDocument.java
│   │       │   │   └── [If Database = PostgreSQL/MySQL] BaseEntity.java
│   │       │   ├── security/                             # [If Auth != none]
│   │       │   │   ├── CspNonceFilter.java
│   │       │   │   ├── AuthAware.java
│   │       │   │   ├── ThemeAware.java
│   │       │   │   └── PaginationAware.java
│   │       │   ├── vite/
│   │       │   │   └── ViteManifestService.java
│   │       │   ├── audit/                                # [If Database != none]
│   │       │   │   └── AuditAwareImpl.java
│   │       │   ├── layout/
│   │       │   │   ├── MainLayout.java
│   │       │   │   ├── [If Auth != none] AuthLayout.java
│   │       │   │   └── ErrorLayout.java
│   │       │   ├── fragment/
│   │       │   │   ├── NavItem.java
│   │       │   │   ├── UserProfile.java
│   │       │   │   ├── SidebarViewModel.java
│   │       │   │   └── FooterViewModel.java
│   │       │   ├── component/
│   │       │   │   ├── Alert.java
│   │       │   │   ├── Pagination.java
│   │       │   │   ├── PaginationRequest.java
│   │       │   │   ├── PaginatedResult.java
│   │       │   │   └── (other component view models)
│   │       │   ├── [If Messaging = yes] messaging/
│   │       │   │   ├── RabbitMQMessagingConfig.java
│   │       │   │   ├── RabbitMQPublisher.java
│   │       │   │   ├── MessageConverterConfig.java
│   │       │   │   └── consumer/
│   │       │   │       └── SampleEventConsumer.java
│   │       │   └── auth/                                 # [If Auth != none]
│   │       │       ├── [If Auth = Keycloak] AuthService.java
│   │       │       ├── [If Auth = Keycloak] LoginController.java
│   │       │       ├── [If Auth = Keycloak] KeycloakLoginRequest.java
│   │       │       ├── [If Auth = form] AuthService.java
│   │       │       ├── [If Auth = form] LoginController.java
│   │       │       ├── [If Auth = form] CustomUserDetailsService.java
│   │       │       ├── LoginRequest.java
│   │       │       └── LoginView.java
│   │       │
│   │       ├── {{module1}}/                              # Module — PUBLIC API
│   │       │   ├── {{Module1}}Service.java               #   Public service interface
│   │       │   ├── {{Module1}}DTO.java                   #   Public return type
│   │       │   ├── {{Module1}}Exception.java             #   Public exception class
│   │       │   └── internal/                             #   INTERNAL — hidden from other modules
│   │       │       ├── {{Module1}}ServiceImpl.java       #     Service implementation
│   │       │       ├── [If Database = MongoDB] {{Module1}}Repository.java  #  MongoRepository
│   │       │       ├── [If Database = MongoDB] {{Module1}}Document.java    #  MongoDB document
│   │       │       ├── [If Database = PostgreSQL/MySQL] {{Module1}}Repository.java  # JpaRepository
│   │       │       ├── [If Database = PostgreSQL/MySQL] {{Module1}}Entity.java      # JPA entity
│   │       │       ├── {{Module1}}Mapper.java            #     MapStruct mapper
│   │       │       ├── page/
│   │       │       │   ├── {{Module1}}PageController.java#     Page controller (@Controller)
│   │       │       │   ├── {{Module1}}ListView.java      #     List page view model
│   │       │       │   └── {{Module1}}DetailView.java    #     Detail page view model
│   │       │       └── fragment/
│   │       │           └── {{Module1}}FragmentController.java # Fragment controller (htmx)
│   │       │
│   │       ├── {{module2}}/                              # Another module
│   │       │   └── (same public/internal structure)
│   │       │
│   │       └── [If Scheduling = yes] scheduling/         # Scheduling module
│   │           ├── internal/
│   │           │   ├── job/
│   │           │   │   ├── SampleQuartzJob.java
│   │           │   │   └── [If Remote Partitioning = yes] PartitionedQuartzJob.java
│   │           │   └── [If Spring Batch = yes] batch/
│   │           │       ├── SampleBatchConfig.java
│   │           │       ├── SampleItemReader.java
│   │           │       ├── SampleItemProcessor.java
│   │           │       ├── SampleItemWriter.java
│   │           │       └── [If Remote Partitioning = yes] partition/
│   │           │           ├── SamplePartitioner.java
│   │           │           ├── PartitionChannelConfig.java
│   │           │           ├── PartitionedBatchConfig.java
│   │           │           ├── WorkerBatchConfig.java
│   │           │           └── PartitionedReaderConfig.java
│   │           └── SchedulingService.java
│   │
│   ├── jte/                                              # JTE templates
│   │   └── {{BASE_PACKAGE_PATH}}/
│   │       ├── shared/
│   │       │   ├── layout/
│   │       │   │   ├── MainLayout.jte
│   │       │   │   ├── [If Auth != none] AuthLayout.jte
│   │       │   │   └── ErrorLayout.jte
│   │       │   ├── fragment/
│   │       │   │   ├── HeaderFragment.jte
│   │       │   │   ├── SidebarFragment.jte
│   │       │   │   ├── FooterFragment.jte
│   │       │   │   └── ThemeSwitcherFragment.jte
│   │       │   └── component/
│   │       │       ├── alert/Alert.jte
│   │       │       ├── avatar/Avatar.jte
│   │       │       ├── badge/Badge.jte
│   │       │       ├── breadcrumb/Breadcrumb.jte
│   │       │       ├── button/Button.jte
│   │       │       ├── card/Card.jte
│   │       │       ├── form/FormControl.jte
│   │       │       ├── form/Input.jte
│   │       │       ├── form/Select.jte
│   │       │       ├── loading/Loading.jte
│   │       │       ├── modal/Modal.jte
│   │       │       ├── pagination/Pagination.jte
│   │       │       ├── table/Table.jte
│   │       │       ├── tabs/Tabs.jte
│   │       │       └── toast/ToastContainer.jte
│   │       ├── {{module1}}/
│   │       │   ├── page/
│   │       │   │   ├── {{Module1}}ListPage.jte
│   │       │   │   └── {{Module1}}DetailPage.jte
│   │       │   └── fragment/
│   │       │       └── {{Module1}}RowFragment.jte
│   │       └── {{module2}}/
│   │           └── (same page/fragment structure)
│   │
│   └── resources/
│       ├── application.yml
│       ├── application-dev.yml
│       ├── application-prod.yml
│       ├── logback-spring.xml
│       ├── [If Database = PostgreSQL/MySQL] db/migration/
│       │   └── V1__initial_schema.sql
│       └── static/
│           ├── assets/
│           │   └── logo.svg
│           └── dist/                                     # Vite build output (gitignored)
│
├── frontend/                                             # Frontend build source
│   └── (see Section 4)
│
└── test/
    └── java/
        └── {{BASE_PACKAGE}}/
            └── (test classes)
```

Explain the Spring Modulith encapsulation rules and how they apply to web modules.
Refer to `references/modulith-patterns.md` for the detailed explanation.

---

## Section 6: Security Configuration [CONDITIONAL — Include only if Auth != none]

**[If Auth = Keycloak]:**

This section describes the Spring Security setup for Keycloak JWT auth.
Refer to `references/security-patterns.md` for the complete content to include.

Must cover:
- `SecurityConfiguration.java` — full `SecurityFilterChain` bean definition
- `KeycloakGrantedAuthoritiesMapper.java` — extracting Keycloak roles from OIDC ID token claims
- `CspNonceFilter.java` — CSP nonce generation per request
- `AuthAware.java` — mixin to extract user info from JWT
- Public vs. secured endpoint rules
- Role hierarchy mapping from Keycloak realm_access and resource_access claims
- `SecurityContextUtil.java` — helper to extract current user info from JWT
- CSRF handling (disabled for stateless)
- CORS configuration
- Session creation policy: `STATELESS`

**[If Auth = form]:**

This section describes the Spring Security setup for form-based login.

Must cover:
- `SecurityConfiguration.java` — full `SecurityFilterChain` bean with `formLogin()`, `logout()`, `rememberMe()`
- `CustomUserDetailsService.java` — `UserDetailsService` implementation backed by the selected database
- `CspNonceFilter.java` — CSP nonce generation per request
- `AuthAware.java` — mixin to extract user info from `SecurityContextHolder`
- Public vs. secured endpoint rules
- Role-based access control using `hasRole()` / `hasAuthority()`
- CSRF enabled (standard for form login)
- CORS configuration
- Password encoding with `BCryptPasswordEncoder` bean

---

## Section 7: Layouts (JTE)

### 7.1 MainLayout
Purpose: authenticated pages with header, sidebar, footer; responsive.

**CRITICAL — JTE content injection pattern:** Layouts accept a `gg.jte.Content` parameter
(not `@insert`/`@endtemplate`). Pages pass their body via a content block `content = @`...``.
This is the only pattern that works reliably with JTE 3.x. Do NOT use `@insert`/`@endtemplate`.

```java
@Getter @Setter
public class MainLayout {
    private final String title;
    private final List<NavItem> navItems;
    private final UserProfile user;
    private final String theme;
    private final String cspNonce;
    private final ViteManifestService vite;

    public MainLayout defaultPage(String title, HttpServletRequest req) { /* factory */ }
}
```
```jte
@import com.bestinet.urp.techres.shared.layout.MainLayout
@import gg.jte.Content
@param MainLayout layout
@param Content content
<!DOCTYPE html>
<html lang="en" data-theme="${layout.getTheme()}">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>${layout.getTitle()} — {{APP_TITLE}}</title>
  <link rel="stylesheet" href="${layout.getVite().resolveAsset("css/app.css")}" />
  <script type="module" src="${layout.getVite().resolveAsset("js/app.js")}" nonce="${layout.getCspNonce()}"></script>
</head>
<body class="min-h-screen bg-gray-50 dark:bg-gray-900 font-body">
  @template.{{BASE_PACKAGE_PATH_DOT}}.shared.fragment.HeaderFragment(layout = layout)
  <div class="flex pt-16 min-h-[calc(100vh-4rem)]">
    @template.{{BASE_PACKAGE_PATH_DOT}}.shared.fragment.SidebarFragment(navItems = layout.getNavItems())
    <main class="flex-1 ml-64">
      <div id="content-area" class="p-6 space-y-6">
        ${content}
      </div>
      @template.{{BASE_PACKAGE_PATH_DOT}}.shared.fragment.FooterFragment()
    </main>
  </div>
  @template.{{BASE_PACKAGE_PATH_DOT}}.shared.component.toast.ToastContainer()
</body>
</html>
```

**How pages use MainLayout** — pages wrap their body in `content = @`...``:
```jte
@import {{BASE_PACKAGE}}.{{module}}.internal.page.{{Entity}}ListView
@param {{Entity}}ListView view
@template.{{BASE_PACKAGE_PATH_DOT}}.shared.layout.MainLayout(layout = view.getLayout(), content = @`
    <%-- Page content here --%>
    <h1>Page Title</h1>
    <div id="{{module}}-list">
        @template.{{BASE_PACKAGE_PATH_DOT}}.{{module}}.page.{{Entity}}ListFragment(items = view.getItems(), pagination = view.getPagination())
    </div>
`)
```

### 7.2 AuthLayout [CONDITIONAL — Include only if Auth != none]
Purpose: split-screen login page for unauthenticated users.
```jte
@import {{BASE_PACKAGE}}.shared.layout.AuthLayout
@import gg.jte.Content
@param AuthLayout layout
@param Content content
<!DOCTYPE html>
<html lang="en" data-theme="${layout.getTheme()}">
<body class="min-h-screen grid md:grid-cols-2">
  <section class="hidden md:flex items-center justify-center bg-gray-100 dark:bg-gray-800">
    <div class="max-w-md space-y-4">
      <img src="/static/assets/logo.svg" alt="Brand" class="h-10" />
      <h1 class="text-3xl font-bold text-gray-900 dark:text-white">Welcome back</h1>
      <p class="text-gray-500 dark:text-gray-400">Access your dashboard securely.</p>
    </div>
  </section>
  <section class="flex items-center justify-center p-8">
    ${content}
  </section>
</body>
</html>
```

### 7.3 ErrorLayout
Purpose: centered error display. ErrorLayout does NOT use the Content parameter pattern
since it renders a self-contained error page with no injected body.
```jte
@import {{BASE_PACKAGE}}.shared.layout.ErrorLayout
@param ErrorLayout layout
<!DOCTYPE html>
<html lang="en" data-theme="${layout.getTheme()}">
<body class="min-h-screen flex flex-col items-center justify-center bg-gray-50 dark:bg-gray-900">
  <div class="text-center space-y-3">
    <p class="text-6xl font-bold text-red-600">${layout.getStatus()}</p>
    <h1 class="text-2xl font-semibold text-gray-900 dark:text-white">${layout.getTitle()}</h1>
    <p class="text-gray-500 dark:text-gray-400">${layout.getMessage()}</p>
    <a class="inline-flex items-center px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors" href="/">Return Home</a>
  </div>
</body>
</html>
```

---

## Section 8: Fragments

**CRITICAL — Template invocation syntax:** All JTE templates MUST use the fully-qualified
`@template.` syntax to call other templates. Do NOT use `@include()`. The correct syntax is:
```
@template.{{BASE_PACKAGE_PATH_DOT}}.shared.fragment.HeaderFragment(layout = layout)
```

### 8.1 HeaderFragment
```jte
@import {{BASE_PACKAGE}}.shared.layout.MainLayout
@param MainLayout layout
<header class="fixed top-0 left-0 right-0 z-50 bg-white dark:bg-gray-800 shadow-sm border-b border-gray-200 dark:border-gray-700">
  <div class="px-4 sm:px-6 lg:px-8">
    <div class="flex items-center justify-between h-16">
      <div class="flex items-center">
        <button class="lg:hidden p-2 rounded-md text-gray-500 hover:text-gray-700 hover:bg-gray-100 dark:text-gray-400 dark:hover:text-gray-200 dark:hover:bg-gray-700" x-data @click="$store.drawer.toggle()">
          <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16"/></svg>
        </button>
        <a class="flex items-center gap-2 ml-2 lg:ml-0" href="/">
          <img src="/static/assets/logo.svg" class="h-8" alt="Logo" />
        </a>
      </div>
      <div class="flex items-center gap-3">
        @template.{{BASE_PACKAGE_PATH_DOT}}.shared.fragment.ThemeSwitcherFragment()
        @template.{{BASE_PACKAGE_PATH_DOT}}.shared.component.avatar.Avatar(user = layout.getUser())
      </div>
    </div>
  </div>
</header>
```

### 8.2 SidebarFragment
The sidebar receives `navItems` directly (not a separate ViewModel). It is a fixed left
panel (w-64) that works with the `ml-64` offset on `<main>` in MainLayout.
```jte
@import {{BASE_PACKAGE}}.shared.fragment.NavItem
@import java.util.List
@param List<NavItem> navItems
<aside class="fixed top-16 left-0 bottom-0 z-40 w-64 bg-white dark:bg-gray-800 border-r border-gray-200 dark:border-gray-700 overflow-y-auto"
       x-show="$store.drawer.open || window.innerWidth >= 1024" x-transition>
  <div class="py-4 px-3">
    <ul class="space-y-1">
      @for(var item : navItems)
        <li>
          <a href="${item.href()}" class="flex items-center px-3 py-2 rounded-lg text-sm font-medium ${item.active() ? "bg-blue-50 text-blue-700 dark:bg-blue-900/50 dark:text-blue-300" : "text-gray-700 hover:bg-gray-100 dark:text-gray-300 dark:hover:bg-gray-700"} transition-colors">
            ${item.label()}
          </a>
        </li>
      @endfor
    </ul>
  </div>
</aside>
```

### 8.3 FooterFragment
The footer takes no parameters. It is rendered inside `<main>` after the content area.
```jte
<footer class="bg-white dark:bg-gray-800 border-t border-gray-200 dark:border-gray-700 mt-auto">
  <div class="px-4 sm:px-6 lg:px-8 py-4">
    <div class="flex flex-col sm:flex-row items-center justify-between gap-2">
      <p class="text-sm text-gray-500 dark:text-gray-400">© {{CURRENT_YEAR}} {{COMPANY_NAME}}. All rights reserved.</p>
      <div class="flex gap-4">
        <a href="/privacy" class="text-sm text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200 transition-colors">Privacy</a>
        <a href="/terms" class="text-sm text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200 transition-colors">Terms</a>
      </div>
    </div>
  </div>
</footer>
```

### 8.4 ThemeSwitcherFragment
```jte
<button type="button"
        class="p-2 rounded-lg text-gray-500 hover:text-gray-700 hover:bg-gray-100 dark:text-gray-400 dark:hover:text-gray-200 dark:hover:bg-gray-700 transition-colors"
        x-data @click="$store.theme.toggle()"
        :aria-label="$store.theme.isDark ? 'Switch to light mode' : 'Switch to dark mode'">
  <svg x-show="!$store.theme.isDark" class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M20.354 15.354A9 9 0 018.646 3.646 9.003 9.003 0 0012 21a9.003 9.003 0 008.354-5.646z"/></svg>
  <svg x-show="$store.theme.isDark" class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364 6.364l-.707-.707M6.343 6.343l-.707-.707m12.728 0l-.707.707M6.343 17.657l-.707.707M16 12a4 4 0 11-8 0 4 4 0 018 0z"/></svg>
</button>
```

---

## Section 9: UI Components (Tailwind-based)

Pattern: each component has a Java view model (record or class) and a JTE template.
All styling uses pure Tailwind CSS utility classes. No DaisyUI.

### 9.1 Alert
```java
public record Alert(String type, String message) {}
// type: "success" | "info" | "warning" | "error"
```
```jte
@param Alert alert
!{var colors = switch(alert.type()) {
    case "success" -> "bg-green-50 text-green-800 border-green-200 dark:bg-green-900/20 dark:text-green-400 dark:border-green-800";
    case "info" -> "bg-blue-50 text-blue-800 border-blue-200 dark:bg-blue-900/20 dark:text-blue-400 dark:border-blue-800";
    case "warning" -> "bg-yellow-50 text-yellow-800 border-yellow-200 dark:bg-yellow-900/20 dark:text-yellow-400 dark:border-yellow-800";
    case "error" -> "bg-red-50 text-red-800 border-red-200 dark:bg-red-900/20 dark:text-red-400 dark:border-red-800";
    default -> "bg-gray-50 text-gray-800 border-gray-200";
};}
<div class="flex items-center p-4 rounded-lg border ${colors}" role="alert">
  <span class="text-sm font-medium">${alert.message()}</span>
</div>
```

### 9.2 Avatar
```java
public record Avatar(String imageUrl, String fallback, String size) {}
// size: "sm" | "md" | "lg"
```
```jte
@param Avatar avatar
!{var dim = switch(avatar.size()) { case "sm" -> "w-8 h-8"; case "lg" -> "w-12 h-12"; default -> "w-10 h-10"; };}
<div class="relative inline-flex items-center justify-center ${dim} rounded-full overflow-hidden bg-gray-200 dark:bg-gray-600">
  @if(avatar.imageUrl() != null)
    <img src="${avatar.imageUrl()}" alt="${avatar.fallback()}" class="w-full h-full object-cover" />
  @else
    <span class="text-sm font-medium text-gray-600 dark:text-gray-300">${avatar.fallback()}</span>
  @endif
</div>
```

### 9.3 Badge
```java
public record Badge(String text, String tone) {}
// tone: "primary" | "success" | "warning" | "error" | "neutral"
```
```jte
@param Badge badge
!{var colors = switch(badge.tone()) {
    case "primary" -> "bg-blue-100 text-blue-800 dark:bg-blue-900/30 dark:text-blue-400";
    case "success" -> "bg-green-100 text-green-800 dark:bg-green-900/30 dark:text-green-400";
    case "warning" -> "bg-yellow-100 text-yellow-800 dark:bg-yellow-900/30 dark:text-yellow-400";
    case "error" -> "bg-red-100 text-red-800 dark:bg-red-900/30 dark:text-red-400";
    default -> "bg-gray-100 text-gray-800 dark:bg-gray-700 dark:text-gray-300";
};}
<span class="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium ${colors}">${badge.text()}</span>
```

### 9.4 Breadcrumb
```java
public record BreadcrumbItem(String label, String href, boolean active) {}
public record Breadcrumb(List<BreadcrumbItem> items) {}
```
```jte
@param Breadcrumb breadcrumb
<nav class="flex" aria-label="Breadcrumb">
  <ol class="flex items-center space-x-2">
    @for(var item : breadcrumb.items())
      <li class="flex items-center">
        @if(!item.active())
          <a href="${item.href()}" class="text-sm text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200">${item.label()}</a>
          <svg class="ml-2 w-4 h-4 text-gray-400" fill="currentColor" viewBox="0 0 20 20"><path fill-rule="evenodd" d="M7.293 14.707a1 1 0 010-1.414L10.586 10 7.293 6.707a1 1 0 011.414-1.414l4 4a1 1 0 010 1.414l-4 4a1 1 0 01-1.414 0z" clip-rule="evenodd"/></svg>
        @else
          <span class="text-sm font-medium text-gray-900 dark:text-white">${item.label()}</span>
        @endif
      </li>
    @endfor
  </ol>
</nav>
```

### 9.5 Button
```java
public record Button(String label, String href, String tone, boolean outline) {}
```
```jte
@param Button button
!{var base = "inline-flex items-center px-4 py-2 text-sm font-medium rounded-lg transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2";}
!{var colors = button.outline()
    ? switch(button.tone()) {
        case "primary" -> "border border-blue-600 text-blue-600 hover:bg-blue-50 focus:ring-blue-500 dark:border-blue-400 dark:text-blue-400 dark:hover:bg-blue-900/20";
        case "danger" -> "border border-red-600 text-red-600 hover:bg-red-50 focus:ring-red-500 dark:border-red-400 dark:text-red-400 dark:hover:bg-red-900/20";
        default -> "border border-gray-300 text-gray-700 hover:bg-gray-50 focus:ring-gray-500 dark:border-gray-600 dark:text-gray-300 dark:hover:bg-gray-700";
      }
    : switch(button.tone()) {
        case "primary" -> "bg-blue-600 text-white hover:bg-blue-700 focus:ring-blue-500";
        case "danger" -> "bg-red-600 text-white hover:bg-red-700 focus:ring-red-500";
        default -> "bg-white text-gray-700 border border-gray-300 hover:bg-gray-50 focus:ring-gray-500 dark:bg-gray-700 dark:text-gray-300 dark:border-gray-600 dark:hover:bg-gray-600";
      };}
<a class="${base} ${colors}" href="${button.href()}">${button.label()}</a>
```

### 9.6 Card
```java
public record Card(String title, String subtitle, String content, List<Button> actions) {}
```
```jte
@param Card card
<div class="bg-white dark:bg-gray-800 rounded-lg shadow-sm border border-gray-200 dark:border-gray-700 overflow-hidden">
  <div class="p-6">
    <h3 class="text-lg font-semibold text-gray-900 dark:text-white">${card.title()}</h3>
    @if(card.subtitle() != null)
      <p class="mt-1 text-sm text-gray-500 dark:text-gray-400">${card.subtitle()}</p>
    @endif
    @if(card.content() != null)
      <p class="mt-3 text-gray-700 dark:text-gray-300">${card.content()}</p>
    @endif
    @if(card.actions() != null && !card.actions().isEmpty())
      <div class="mt-4 flex justify-end gap-2">
        @for(btn : card.actions())
          @template.{{BASE_PACKAGE_PATH_DOT}}.shared.component.button.Button(button = btn)
        @endfor
      </div>
    @endif
  </div>
</div>
```

### 9.7 FormControl & Input
```java
public record FormControl(String label, String name, String help, boolean required) {}
public record Input(String name, String type, String value, String placeholder) {}
public record Select(String name, List<Option> options) {}
public record Option(String value, String label, boolean selected) {}
public record Checkbox(String name, boolean checked, String label) {}
public record Radio(String name, String value, boolean checked, String label) {}
public record Toggle(String name, boolean enabled, String label) {}
```
```jte
@!-- FormControl.jte --@
@import gg.jte.Content
@param FormControl ctrl
@param Content content
<div class="space-y-1">
  <label for="${ctrl.name()}" class="block text-sm font-medium text-gray-700 dark:text-gray-300">
    ${ctrl.label()} @if(ctrl.required()) <span class="text-red-500">*</span> @endif
  </label>
  ${content}
  @if(ctrl.help() != null)
    <p class="text-xs text-gray-500 dark:text-gray-400">${ctrl.help()}</p>
  @endif
</div>
```
```jte
@!-- Input.jte --@
@param Input input
<input type="${input.type()}" name="${input.name()}" id="${input.name()}" value="${input.value()}"
       placeholder="${input.placeholder()}"
       class="block w-full rounded-lg border border-gray-300 bg-white px-3 py-2 text-sm text-gray-900 placeholder-gray-400 focus:border-blue-500 focus:ring-1 focus:ring-blue-500 dark:bg-gray-700 dark:border-gray-600 dark:text-white dark:placeholder-gray-400 dark:focus:border-blue-400 dark:focus:ring-blue-400" />
```
```jte
@!-- Select.jte --@
@param Select select
<select name="${select.name()}" id="${select.name()}"
        class="block w-full rounded-lg border border-gray-300 bg-white px-3 py-2 text-sm text-gray-900 focus:border-blue-500 focus:ring-1 focus:ring-blue-500 dark:bg-gray-700 dark:border-gray-600 dark:text-white dark:focus:border-blue-400">
  @for(opt : select.options())
    <option value="${opt.value()}" ${opt.selected() ? "selected" : ""}>${opt.label()}</option>
  @endfor
</select>
```
```jte
@!-- Checkbox.jte --@
@param Checkbox checkbox
<label class="flex items-center gap-3 cursor-pointer">
  <input type="checkbox" name="${checkbox.name()}" class="h-4 w-4 rounded border-gray-300 text-blue-600 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-700" ${checkbox.checked() ? "checked" : ""} />
  <span class="text-sm text-gray-700 dark:text-gray-300">${checkbox.label()}</span>
</label>
```

### 9.8 Loading & Skeleton
```java
public record Loading(boolean spinner) {}
public record Skeleton(String width, String height) {}
```
```jte
@param Loading loading
@if(loading.spinner())
  <div class="flex justify-center">
    <svg class="animate-spin h-5 w-5 text-blue-600" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
      <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
      <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"></path>
    </svg>
  </div>
@else
  <div class="flex space-x-1">
    <div class="w-2 h-2 bg-blue-600 rounded-full animate-bounce [animation-delay:-0.3s]"></div>
    <div class="w-2 h-2 bg-blue-600 rounded-full animate-bounce [animation-delay:-0.15s]"></div>
    <div class="w-2 h-2 bg-blue-600 rounded-full animate-bounce"></div>
  </div>
@endif
```

### 9.9 Modal
```java
public record ModalAction(String label, String tone, String js) {}
public record Modal(String id, String title, String body, List<ModalAction> actions) {}
```
```jte
@param Modal modal
<div id="${modal.id()}" class="fixed inset-0 z-50 hidden" x-data="{ open: false }" x-show="open" x-cloak>
  <div class="fixed inset-0 bg-black/50 transition-opacity" @click="open = false"></div>
  <div class="fixed inset-0 flex items-center justify-center p-4">
    <div class="bg-white dark:bg-gray-800 rounded-xl shadow-xl max-w-md w-full" @click.stop>
      <div class="px-6 py-4 border-b border-gray-200 dark:border-gray-700">
        <h3 class="text-lg font-semibold text-gray-900 dark:text-white">${modal.title()}</h3>
      </div>
      <div class="px-6 py-4">
        <p class="text-gray-600 dark:text-gray-300">${modal.body()}</p>
      </div>
      <div class="px-6 py-4 border-t border-gray-200 dark:border-gray-700 flex justify-end gap-2">
        @for(action : modal.actions())
          <button class="px-4 py-2 text-sm font-medium rounded-lg transition-colors" @click="${action.js()}">${action.label()}</button>
        @endfor
      </div>
    </div>
  </div>
</div>
```

### 9.10 Pagination
```java
public record Pagination(int page, int totalPages, String baseHref) {}
```
```jte
@param Pagination pagination
<nav class="flex items-center justify-center gap-1" aria-label="Pagination">
  @if(pagination.page() > 1)
    <a href="${pagination.baseHref()}?page=${pagination.page() - 1}" class="px-3 py-2 text-sm rounded-lg text-gray-700 hover:bg-gray-100 dark:text-gray-300 dark:hover:bg-gray-700 transition-colors">Previous</a>
  @endif
  @for(int i = 1; i <= pagination.totalPages(); i++)
    <a href="${pagination.baseHref()}?page=${i}"
       class="px-3 py-2 text-sm rounded-lg transition-colors ${i == pagination.page() ? "bg-blue-600 text-white font-medium" : "text-gray-700 hover:bg-gray-100 dark:text-gray-300 dark:hover:bg-gray-700"}">
      ${i}
    </a>
  @endfor
  @if(pagination.page() < pagination.totalPages())
    <a href="${pagination.baseHref()}?page=${pagination.page() + 1}" class="px-3 py-2 text-sm rounded-lg text-gray-700 hover:bg-gray-100 dark:text-gray-300 dark:hover:bg-gray-700 transition-colors">Next</a>
  @endif
</nav>
```

### 9.11 Table
```java
public record TableHead(List<String> columns) {}
public record TableRow(List<String> cells) {}
public record TableBody(List<TableRow> rows) {}
public record Table(TableHead head, TableBody body) {}
```
```jte
@param Table table
<div class="overflow-x-auto rounded-lg border border-gray-200 dark:border-gray-700">
  <table class="min-w-full divide-y divide-gray-200 dark:divide-gray-700">
    <thead class="bg-gray-50 dark:bg-gray-800">
      <tr>
        @for(col : table.head().columns())
          <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider">${col}</th>
        @endfor
      </tr>
    </thead>
    <tbody class="bg-white dark:bg-gray-900 divide-y divide-gray-200 dark:divide-gray-700">
      @for(row : table.body().rows())
        <tr class="hover:bg-gray-50 dark:hover:bg-gray-800/50 transition-colors">
          @for(cell : row.cells())
            <td class="px-6 py-4 text-sm text-gray-900 dark:text-gray-300 whitespace-nowrap">${cell}</td>
          @endfor
        </tr>
      @endfor
    </tbody>
  </table>
</div>
```

### 9.12 Tabs
```java
public record Tab(String id, String label, boolean active) {}
public record Tabs(List<Tab> items) {}
```
```jte
@param Tabs tabs
<div class="border-b border-gray-200 dark:border-gray-700">
  <nav class="flex space-x-4" role="tablist">
    @for(tab : tabs.items())
      <a role="tab" href="#${tab.id()}"
         class="px-3 py-2 text-sm font-medium border-b-2 transition-colors ${tab.active() ? "border-blue-600 text-blue-600 dark:border-blue-400 dark:text-blue-400" : "border-transparent text-gray-500 hover:text-gray-700 hover:border-gray-300 dark:text-gray-400 dark:hover:text-gray-200"}">
        ${tab.label()}
      </a>
    @endfor
  </nav>
</div>
```

### 9.13 Toast & ToastContainer
```java
public record Toast(String type, String message) {}
public record ToastContainer(List<Toast> toasts) {}
```
```jte
@param ToastContainer container
<div class="fixed bottom-4 right-4 z-50 space-y-2" x-data>
  <template x-for="(toast, i) in $store.toast.items" :key="i">
    <div class="flex items-center p-4 rounded-lg shadow-lg border max-w-sm"
         :class="toast.type === 'error' ? 'bg-red-50 border-red-200 text-red-800 dark:bg-red-900/20 dark:border-red-800 dark:text-red-400' :
                  toast.type === 'success' ? 'bg-green-50 border-green-200 text-green-800 dark:bg-green-900/20 dark:border-green-800 dark:text-green-400' :
                  'bg-blue-50 border-blue-200 text-blue-800 dark:bg-blue-900/20 dark:border-blue-800 dark:text-blue-400'">
      <span class="text-sm font-medium" x-text="toast.message"></span>
      <button @click="$store.toast.remove(i)" class="ml-3 text-current opacity-50 hover:opacity-100">
        <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 20 20"><path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z" clip-rule="evenodd"/></svg>
      </button>
    </div>
  </template>
</div>
```

### 9.14 Stats
```java
public record Stat(String label, String value, String desc) {}
public record Stats(List<Stat> items) {}
```
```jte
@param Stats stats
<div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-${stats.items().size()} gap-4">
  @for(stat : stats.items())
    <div class="bg-white dark:bg-gray-800 rounded-lg shadow-sm border border-gray-200 dark:border-gray-700 p-6">
      <p class="text-sm text-gray-500 dark:text-gray-400">${stat.label()}</p>
      <p class="mt-1 text-2xl font-semibold text-gray-900 dark:text-white">${stat.value()}</p>
      @if(stat.desc() != null)
        <p class="mt-1 text-xs text-gray-500 dark:text-gray-400">${stat.desc()}</p>
      @endif
    </div>
  @endfor
</div>
```

### 9.15 Steps
```java
public record Step(String label, boolean done) {}
public record Steps(List<Step> items) {}
```
```jte
@param Steps steps
<ol class="flex items-center">
  @for(var step : steps.items())
    <li class="flex items-center ${step.done() ? "text-blue-600 dark:text-blue-400" : "text-gray-500 dark:text-gray-400"}">
      <span class="flex items-center justify-center w-8 h-8 rounded-full border-2 text-sm font-medium ${step.done() ? "border-blue-600 bg-blue-600 text-white dark:border-blue-400 dark:bg-blue-400" : "border-gray-300 dark:border-gray-600"}">
        @if(step.done()) ✓ @endif
      </span>
      <span class="ml-2 text-sm font-medium">${step.label()}</span>
    </li>
  @endfor
</ol>
```

### 9.16 Tooltip
```java
public record Tooltip(String text) {}
```
```jte
@import gg.jte.Content
@param Tooltip tooltip
@param Content content
<div class="relative group inline-block">
  ${content}
  <div class="absolute bottom-full left-1/2 -translate-x-1/2 mb-2 px-3 py-1.5 bg-gray-900 text-white text-xs rounded-lg opacity-0 group-hover:opacity-100 transition-opacity pointer-events-none whitespace-nowrap dark:bg-gray-700">
    ${tooltip.text()}
    <div class="absolute top-full left-1/2 -translate-x-1/2 -mt-1 border-4 border-transparent border-t-gray-900 dark:border-t-gray-700"></div>
  </div>
</div>
```

### 9.17 Progress
```java
public record Progress(int value, int max) {}
```
```jte
@param Progress progress
!{var pct = (int) ((double) progress.value() / progress.max() * 100);}
<div class="w-full bg-gray-200 dark:bg-gray-700 rounded-full h-2.5">
  <div class="bg-blue-600 h-2.5 rounded-full transition-all" style="width: ${pct}%"></div>
</div>
```

---

*(Module Blueprints are generated as separate files — see Part B below.
Each module gets its own `<module>/SPEC.md` file. The Module Index in
Section 1 links to each module's spec.)*

---

## Section 10: Frontend JS Structure

### 10.1 App Entry
```js
// frontend/src/js/app.js
import Alpine from 'alpinejs';
import htmx from 'htmx.org';
import { toastStore } from './alpine/stores/toast.js';
import { themeStore } from './alpine/stores/theme.js';
import { drawerStore } from './alpine/stores/drawer.js';
import { modalStore } from './alpine/stores/modal.js';
import './htmx/index.js';

Alpine.store('toast', toastStore());
Alpine.store('theme', themeStore());
Alpine.store('drawer', drawerStore());
Alpine.store('modal', modalStore());
Alpine.start();
window.htmx = htmx;
```

### 10.2 Alpine Stores
```js
// toast.js
export function toastStore() {
  return {
    items: [],
    push(type, message) {
      this.items.push({ type, message });
      setTimeout(() => this.items.shift(), 5000);
    },
    remove(index) { this.items.splice(index, 1); }
  };
}

// theme.js
export function themeStore() {
  return {
    isDark: document.cookie.includes('theme=dark'),
    toggle() {
      this.isDark = !this.isDark;
      const theme = this.isDark ? 'dark' : 'light';
      document.documentElement.dataset.theme = theme;
      document.cookie = `theme=${theme};path=/;max-age=31536000;SameSite=Lax`;
    }
  };
}

// drawer.js
export function drawerStore() {
  return {
    open: false,
    toggle() { this.open = !this.open; },
    close() { this.open = false; }
  };
}
```

### 10.3 htmx Global Handlers
```js
// frontend/src/js/htmx/index.js
document.addEventListener('htmx:responseError', (event) => {
  const status = event.detail.xhr.status;
  const message = status >= 500 ? 'Server error. Please try again.' : 'Request failed.';
  Alpine.store('toast').push('error', message);
});

document.addEventListener('htmx:sendError', () => {
  Alpine.store('toast').push('error', 'Network error. Check your connection.');
});
```

---

## Section 11: Authentication Pages [CONDITIONAL — Include only if Auth != none]

**[If Auth = Keycloak]:**

Keycloak handles all login flows externally. The application redirects unauthenticated
users to Keycloak's login page via the OAuth2 authorization code flow. Keycloak manages:
- Login form (email/password)
- Social login providers (Google, etc.)
- User registration (if enabled in Keycloak realm)
- Password reset

The application only needs to:
- Configure the OAuth2 client for Keycloak
- Handle the JWT token returned after successful authentication
- Provide a logout endpoint that invalidates the Keycloak session

**[If Auth = form]:**

The application provides its own login page. Include:
- Login page with email/password form, CSRF token, remember-me checkbox
- Login controller handling GET (render form) and POST (Spring Security handles via `formLogin()`)
- Logout endpoint clearing the session
- Optionally: registration page if user self-registration is desired
- Error display for invalid credentials

---

## Section 12: Error Handling & Exceptions

### 13.1 Base Exception
```java
package {{BASE_PACKAGE}}.shared.exception;

import lombok.Getter;
import org.springframework.http.HttpStatus;

@Getter
public class WebApplicationException extends RuntimeException {
    private final HttpStatus status;
    private final String code;
    private final String userMessage;

    public WebApplicationException(HttpStatus status, String code, String userMessage) {
        super(userMessage);
        this.status = status;
        this.code = code;
        this.userMessage = userMessage;
    }
}
```

### 13.2 Global Exception Handler
```java
package {{BASE_PACKAGE}}.shared.exception;

import jakarta.servlet.http.HttpServletRequest;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.servlet.ModelAndView;
import java.util.Map;

@ControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(WebApplicationException.class)
    Object handleWeb(WebApplicationException ex, HttpServletRequest req) {
        log.warn("Web error [{}]: {}", ex.getCode(), ex.getUserMessage());
        if (isHtmxRequest(req)) {
            return ResponseEntity.status(ex.getStatus())
                .header("HX-Trigger", toastEvent("error", ex.getUserMessage()))
                .body("");
        }
        return errorView(ex.getStatus().value(), ex.getStatus().getReasonPhrase(), ex.getUserMessage());
    }

    @ExceptionHandler(Exception.class)
    Object handleUnknown(Exception ex, HttpServletRequest req) {
        log.error("Unhandled error", ex);
        if (isHtmxRequest(req)) {
            return ResponseEntity.internalServerError()
                .header("HX-Trigger", toastEvent("error", "An unexpected error occurred"))
                .body("");
        }
        return errorView(500, "Internal Server Error", "An unexpected error occurred");
    }

    private boolean isHtmxRequest(HttpServletRequest req) {
        return "true".equals(req.getHeader("HX-Request"));
    }

    private ModelAndView errorView(int status, String title, String message) {
        return new ModelAndView("shared/layout/ErrorLayout",
            Map.of("layout", ErrorLayout.of(status, title, message)));
    }

    private String toastEvent(String type, String message) {
        return "{\"showToast\":{\"type\":\"" + type + "\",\"message\":\"" + message + "\"}}";
    }
}
```

---

## Section 13: Data Access [CONDITIONAL content varies by database]

**[If Database = MongoDB]:**

Describe MongoDB repositories per module with Spring Data MongoRepository.
DTO mapping via MapStruct. Pagination helpers with `PaginationAware` mixin.
All list endpoints paginated. Include `BaseDocument` with audit fields.

**[If Database = PostgreSQL/MySQL]:**

Describe JPA repositories per module with Spring Data JpaRepository.
JPA entities with `@Entity`/`@Table` annotations extending `BaseEntity`.
Flyway migration scripts for schema management.
DTO mapping via MapStruct. Pagination helpers. All list endpoints paginated.
Include `BaseEntity` with audit fields and `@MappedSuperclass`.

Include `BaseEntity` definition using `@GeneratedValue(strategy = GenerationType.UUID)` with `@Column(name = "id", columnDefinition = "CHAR(36)", nullable = false, updatable = false)`.
All UUID/foreign-key columns in entities MUST use `columnDefinition = "CHAR(36)"` (not `length = 36`) to match the database `CHAR(36)` type and pass Hibernate schema validation.

Include a sample Flyway migration script:

**[If Database = PostgreSQL]:**
```sql
-- V1__initial_schema.sql
CREATE TABLE {{module_name}} (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_one VARCHAR(255) NOT NULL,
    field_two VARCHAR(255),
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by VARCHAR(255),
    updated_by VARCHAR(255)
);
```

**[If Database = MySQL]:**
```sql
-- V1__initial_schema.sql
CREATE TABLE {{module_name}} (
    id CHAR(36) NOT NULL PRIMARY KEY,
    field_one VARCHAR(255) NOT NULL,
    field_two VARCHAR(255),
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by VARCHAR(255) NOT NULL,
    updated_by VARCHAR(255) NOT NULL
);
```
**MySQL UUID rule:** All UUID primary keys and foreign keys MUST use `CHAR(36)` (not `VARCHAR(36)`). In JPA entities, map these with `columnDefinition = "CHAR(36)"` to ensure Hibernate `ddl-auto: validate` passes.

**[If Database = none]:**

Note that no database is configured. Modules should use in-memory collections
or stub implementations. The spec should indicate where database integration can be
added later.

---

## Section 14: Theming

Describe:
- Light/dark theme using `data-theme` attribute on `<html>`
- CSS custom properties in `themes/light.css` and `themes/dark.css`
- Tailwind `darkMode: ['class', '[data-theme="dark"]']` config
- Server reads `theme` cookie on request; if missing, defaults to `light`
- Layout view model carries theme string; template sets `data-theme="${layout.getTheme()}"`
- Alpine `themeStore` syncs toggle with cookie and `data-theme` attribute

```css
/* themes/light.css */
[data-theme="light"] {
  --color-bg-primary: 255 255 255;
  --color-bg-secondary: 249 250 251;
  --color-text-primary: 17 24 39;
  --color-text-secondary: 107 114 128;
  --color-border: 229 231 235;
}

/* themes/dark.css */
[data-theme="dark"] {
  --color-bg-primary: 17 24 39;
  --color-bg-secondary: 31 41 55;
  --color-text-primary: 243 244 246;
  --color-text-secondary: 156 163 175;
  --color-border: 55 65 81;
}
```

---

## Section 15: Pagination Support

```java
public record PaginationRequest(int page, int size) {}

public record PaginatedResult<T>(List<T> data, int page, int size, long totalItems) {
    public int totalPages() { return Math.max(1, (int) Math.ceil((double) totalItems / size)); }
    public Pagination toPagination(String baseHref) { return new Pagination(page, totalPages(), baseHref); }
}
```

```java
public interface PaginationAware {
    default PaginationRequest pagination(HttpServletRequest req, int defaultSize, int maxSize) {
        int page = Math.max(1, parseIntParam(req, "page", 1));
        int size = Math.min(maxSize, Math.max(1, parseIntParam(req, "size", defaultSize)));
        return new PaginationRequest(page, size);
    }
    private int parseIntParam(HttpServletRequest req, String name, int defaultValue) {
        String val = req.getParameter(name);
        if (val == null) return defaultValue;
        try { return Integer.parseInt(val); } catch (NumberFormatException e) { return defaultValue; }
    }
}
```

All list endpoints must call services with pagination and render the Pagination component
above and below listings.

---

## Section 16: Logging Strategy

Same structure as API spec — `logback-spring.xml` with Spring profiles, correlation ID
(from JWT if auth is enabled, or request-scoped UUID if no auth), MDC context filter.
Include full logback-spring.xml sample:

```xml
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    <property name="LOG_PATTERN"
              value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] [%X{userId}] %-5level %logger{36} - %msg%n"/>
    <springProfile name="dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder><pattern>${LOG_PATTERN}</pattern></encoder>
        </appender>
        <root level="INFO"><appender-ref ref="CONSOLE"/></root>
        <logger name="{{BASE_PACKAGE}}" level="DEBUG"/>
    </springProfile>
    <springProfile name="prod">
        <root level="WARN"/>
        <logger name="{{BASE_PACKAGE}}" level="INFO"/>
    </springProfile>
</configuration>
```

---

## Section 17: Scheduling and Batch Processing [CONDITIONAL — Include only if Scheduling = yes]

Refer to `references/batch-patterns.md` for the full content.

This section has three tiers of functionality:

### Tier 1 — Quartz Only (Scheduling = yes, Spring Batch = no)
Quartz scheduler manages cron-triggered jobs that execute business logic directly
(service calls, API calls, file operations). No reader/processor/writer pipeline.

Adapt the Quartz job store configuration based on the selected database:
- **MongoDB**: Use `fluidsonic-mirror-quartz` for MongoDB-backed job store
- **PostgreSQL/MySQL**: Use built-in Quartz JDBC job store
- **none**: Use in-memory job store

### Tier 2 — Quartz + Spring Batch (Scheduling = yes, Spring Batch = yes)
**[If Spring Batch = yes]:**

Quartz triggers Spring Batch jobs with full reader/processor/writer pipelines for
ETL-style processing. Include all Spring Batch configuration, item reader, item
processor, item writer, and batch YAML config from `references/batch-patterns.md`.

### Tier 3 — Remote Partitioning (Scheduling = yes, Spring Batch = yes, Remote Partitioning = yes)
**[If Spring Batch = yes AND Remote Partitioning = yes]:**

Also include the remote partitioning content from `references/batch-patterns.md`
(section "Remote Partitioning with RabbitMQ"). This adds:
- RabbitMQ infrastructure configuration (exchange, request queue, reply queue, DLQ)
- `SamplePartitioner` — splits data into ranges for parallel processing
- Manager step using `RemotePartitioningManagerStepBuilderFactory`
- Worker step using `RemotePartitioningWorkerStepBuilderFactory`
- Spring Integration channels and AMQP adapters
- `PartitionedQuartzJob` — Quartz job that launches the partitioned batch job
- `@EnableBatchIntegration` configuration
- Worker deployment notes (same JAR, `--spring.profiles.active=worker`)
- Failure/retry semantics and DLQ monitoring

**Note:** If Database = none and Remote Partitioning = yes, the spec must include a
prominent warning that this is an invalid combination. Remote partitioning requires a
shared `JobRepository` database accessible by all manager and worker nodes.

---

## Section 18: Event-Driven Architecture

Refer to `references/modulith-patterns.md` for the full content. Same patterns apply
regardless of database or auth selection.

---

## Section 19: Messaging (RabbitMQ Pub/Sub) [CONDITIONAL — Include only if Messaging = yes]

Refer to `references/messaging-patterns.md` for the full content.

Provides standalone RabbitMQ pub/sub messaging for inter-system communication:
- Topic exchange for broadcasting domain events to external systems
- Direct exchange for point-to-point command delivery
- JSON message serialization via Jackson2JsonMessageConverter
- Dead-letter exchange with DLQ for failed message handling
- RabbitMQPublisher service for sending messages with routing keys
- @RabbitListener consumers with error handling and retry
- Separate from Spring Modulith internal events (those remain in-process)

**Note:** If Remote Partitioning = yes is also selected, the `spring.rabbitmq.*`
connection config is shared. Only include the connection block once in the YAML.
The batch partition queues (`batch.partition.*`) and messaging queues (`app.*`)
use different exchanges and do not conflict.

---

## Section 20: MapStruct Usage

Describe per-module MapStruct mappers for transforming between data models and DTOs.
Mapping flows adapt based on database selection:

- **MongoDB**: `Document -> DTO` (via mapper) and `DTO -> ViewModel` (via constructor or mapper)
- **PostgreSQL/MySQL**: `Entity -> DTO` (via mapper) and `DTO -> ViewModel` (via constructor or mapper)
- **none**: Direct DTO construction

---

## Section 21: Testing Strategy

- **Unit tests**: Mockito for service layer
- **Module tests**: `@ApplicationModuleTest` for module boundary verification
- **View model tests**: Verify view model construction and pagination
- **[If Auth = Keycloak]** Security tests: Custom JWT test utilities for Keycloak token mocking
- **[If Auth = form]** Security tests: `@WithMockUser` for testing secured endpoints
- **Architecture tests**: `ApplicationModules.of(Application.class).verify()`

---

## Section 22: Reporting (Jasper Reports) [CONDITIONAL — Include only if Reporting = yes]

Refer to `references/jasper-patterns.md` for the full content.

Provides a JasperReports-based reporting infrastructure with DTO data sources:

### Architecture

- **`ReportDefinition` interface** — modules implement this to define their reports.
  Each implementation is a `@Component` auto-discovered by the `ReportRegistry`
- **`ReportService`** — compiles `.jrxml` templates via `JasperCompileManager`, fills with
  data via `JRBeanCollectionDataSource` from DTO collections, and exports to PDF/XLSX/CSV
- **`ReportRegistry`** — auto-discovers all `ReportDefinition` beans at startup and persists
  their metadata (name, description, module, template path) to the database
- **`ReportPageController`** — serves the report list page (`GET /reports`), parameter form
  (`GET /reports/{id}`), and generate/download endpoint (`POST /reports/{id}/generate`)
- **JTE templates** — report list page (grouped by module) and parameter form page with
  dynamic fields based on `ReportParameter` descriptors

### Data Flow

1. Module defines a `@Component` implementing `ReportDefinition`
2. At startup, `ReportRegistry` discovers it and persists to the `report` table
3. User selects report → parameter form rendered from `ReportParameter` descriptors
4. User submits → `ReportDefinition.generateData(params)` calls module services to produce `List<DTO>`
5. `ReportService` wraps DTOs in `JRBeanCollectionDataSource`, compiles + fills + exports
6. File returned as download (PDF, XLSX, or CSV)

### Key Constraint: DTO as Data Source (no SQL in templates)

Report definitions call **module services** (never repositories) to produce DTO collections.
`.jrxml` templates contain **no `<queryString>` SQL** — all data comes via
`JRBeanCollectionDataSource`. This preserves Spring Modulith module boundaries and enables
unit testing of report data generation independently from the Jasper engine.

DTO field names must use **camelCase** and match `.jrxml` `<field name="...">` declarations
exactly (JavaBean getter convention). Nested objects should be flattened into the report DTO
(e.g., `staff.department.name` → `departmentName`).

### Maven Dependencies

```xml
<!-- Reporting (included when Reporting = yes) -->
<dependency>
    <groupId>net.sf.jasperreports</groupId>
    <artifactId>jasperreports</artifactId>
    <version>7.0.3</version>
</dependency>
<dependency>
    <groupId>net.sf.jasperreports</groupId>
    <artifactId>jasperreports-fonts</artifactId>
    <version>7.0.3</version>
</dependency>
<dependency>
    <groupId>com.github.librepdf</groupId>
    <artifactId>openpdf</artifactId>
    <version>2.0.4</version>
</dependency>
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>5.4.1</version>
</dependency>
```

### Application Configuration

```yaml
app:
  reporting:
    template-path: classpath:reports/
    compiled-path: reports/compiled/
    temp-path: ${java.io.tmpdir}/reports
    default-format: PDF
```

### Package Structure

```
shared/
└── reporting/
    ├── ReportDefinition.java
    ├── ReportParameter.java
    ├── ReportFormat.java
    ├── ReportResult.java
    ├── ReportService.java
    ├── ReportRegistry.java
    ├── ReportConfig.java
    ├── ReportGenerationException.java
    ├── ReportEntity.java / ReportDocument.java
    ├── ReportRepository.java
    ├── page/ReportPageController.java
    ├── fragment/ReportFragmentController.java
    └── view/ReportListView.java, ReportGenerateView.java
```

### Database Table/Collection

**[If PostgreSQL/MySQL]:**
```sql
CREATE TABLE report (
    report_id   VARCHAR(100) PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    description VARCHAR(500),
    domain      VARCHAR(100) NOT NULL,
    template_path VARCHAR(500) NOT NULL,
    active      BOOLEAN NOT NULL DEFAULT TRUE
);
```

**[If MongoDB]:**
Collection `report` with fields: `reportId`, `name`, `description`, `module`,
`templatePath`, `active`.

### Module Integration Pattern

Each module registers reports by creating `@Component` classes implementing `ReportDefinition`:

```java
@Component
@RequiredArgsConstructor
public class StaffAllocationSummaryReport implements ReportDefinition {

    private final StaffService staffService;

    @Override public String getReportId() { return "staff-allocation-summary"; }
    @Override public String getName() { return "Staff Allocation Summary"; }
    @Override public String getDescription() { return "Lists all staff with allocation status."; }
    @Override public String getDomain() { return "Staff"; }

    @Override
    public List<ReportParameter> getParameters() {
        return List.of(
            ReportParameter.select("departmentId", "Department", false, List.of(/* options */)),
            ReportParameter.select("allocationStatus", "Status", false, List.of(/* options */))
        );
    }

    @Override public String getTemplatePath() { return "staff/staff-allocation-summary.jrxml"; }

    @Override
    public Collection<?> generateData(Map<String, Object> parameters) {
        return staffService.getStaffForReport(/* extract params */);
    }
}
```

The report DTO (e.g., `StaffReportDTO`) is a flat bean whose field names map 1:1 to the
`.jrxml` template's `<field name="...">` declarations. See `references/jasper-patterns.md`
for the complete `ReportDefinition` interface, `ReportService`, `ReportRegistry`,
`ReportPageController`, view models, JTE templates, and a sample `.jrxml` template.

---

# Part B: Module SPEC.md Template

Everything below is the template for each **`<module>/SPEC.md`** file. Generate one
file per module from PRD.md and MODEL.md. Each file is placed in its own
folder named after the module (kebab-case): `spec/<module-name>/SPEC.md`.

Each module SPEC.md is **self-contained** — a coding agent picks it up and implements
the entire module independently (after the shared infrastructure from
`SPECIFICATION.md` is in place).

**File:** `spec/{{module-name}}/SPEC.md`

---

## Module SPEC.md Structure

Generate each module SPEC.md with the following structure:

```markdown
# {{ModuleName}} Module — Module Specification

> Part of [{{APPLICATION_NAME}} Technical Specification](../SPECIFICATION.md)

## Overview

**Module:** {{ModuleName}}
**Package:** {{BASE_PACKAGE}}.{{modulename}}
**Type:** {{System Module | Business Module}}
```

### Traceability

List all tagged IDs this module implements, organized by type with version tags:

```markdown
## Traceability

### User Stories
| ID | Version | Description |
|---|---|---|
| USHM00012 | v1.0.0 | View list of countries |
| USHM00222 | v1.0.3 | Manage provinces/states |

### Non-Functional Requirements
| ID | Version | Description |
|---|---|---|
| NFRHM0006 | v1.0.0 | Country code is unique identifier |

### Constraints
| ID | Version | Description |
|---|---|---|
| CONSHM003 | v1.0.0 | Only supported country types |

### Removed / Replaced
| ID | Type | Removed In | Replaced By | Reason |
|---|---|---|---|---|
| USHM00015 | User Story | v1.0.3 | USHM00222 | Reimplemented with expanded scope |
| NFRHM0039 | NFR | v1.0.3 | NFRHM0198 | Source changed from JD to Quota messages |

### Collections
(flat list — unchanged)

### Mockup Screens
| Screen | Version |
|---|---|
| location_information.html | v1.0.0 |
| location_information_province_create.html | v1.0.3 |
```

### Public API

Include complete code samples for:

- **Service Interface** — `{{ModuleName}}Service.java`
  Methods derived from user stories (e.g., "view list" → `list(PaginationRequest)`)
- **DTOs** — `{{ModuleName}}DTO.java`, `{{ModuleName}}CreateDTO.java`, `{{ModuleName}}UpdateDTO.java`
  Fields from `model/<module>/model.md` and `model/<module>/schemas.json`
- **Exception** — `{{ModuleName}}Exception.java`
  Module-specific exception with NotFound inner class

### Internal Implementation

Include complete code samples for:

- **[If MongoDB] Document** — `{{ModuleName}}Document.java` with `@Document(collection = "{{actual_collection_name}}")`
- **[If PostgreSQL/MySQL] Entity** — `{{ModuleName}}Entity.java` with `@Entity @Table(name = "{{actual_table_name}}")`
- **Repository** — `{{ModuleName}}Repository.java` with query methods from user stories
- **Mapper** — `{{ModuleName}}Mapper.java` with `@Mapper(componentModel = "spring")`
- **Service Implementation** — `{{ModuleName}}ServiceImpl.java` with full CRUD logic
- **Page Controller** — `{{ModuleName}}PageController.java` with endpoints matching mockup screens
- **Fragment Controller** — `{{ModuleName}}FragmentController.java` for htmx partial updates
- **View Models** — `{{ModuleName}}ListView.java`, `{{ModuleName}}DetailView.java` matching mockup data

### JTE Templates

**CRITICAL — JTE syntax rules for all module templates:**
1. Use `@template.{{BASE_PACKAGE_PATH_DOT}}.path.to.Template(param = value)` to call other templates — NEVER `@include()`
2. Page templates wrap content in `content = @`...`)` when calling MainLayout — NEVER use `@insert`/`@endtemplate`
3. Every template must have `@import` statements for all referenced classes
4. Fragment templates do NOT wrap in MainLayout — they render partial HTML for htmx swaps
5. Use `@param` for all template parameters, including `@param Content content` when accepting child content (import `gg.jte.Content`)

Include complete JTE templates for:

- **`page/{{Entity}}ListPage.jte`** — List page wrapping MainLayout with `content = @`...``, containing a div with id for htmx target, calling the list fragment
- **`page/{{Entity}}DetailPage.jte`** — Detail view wrapping MainLayout
- **`page/{{Entity}}FormPage.jte`** — Create/edit form wrapping MainLayout (single template, mode-aware via `view.isEditMode()`)
- **`page/{{Entity}}ListFragment.jte`** — List table fragment (no layout wrapper) for htmx partial updates, includes pagination component
- Additional fragments as needed for tabs, sub-sections, or repeating rows

### Critical Rules for Module SPEC.md

**CRITICAL: Use real module data.** Every module SPEC.md must:
1. Use actual field names from `model/<module>/model.md` and `model/<module>/schemas.json`
2. Use actual collection/table names from the module model
3. Define service methods that map to the actual user stories (by tag ID)
4. Define page controllers that map to actual mockup screens
5. List the user story IDs, NFR IDs, and constraint IDs that this module implements
6. Include complete, continuous code samples — no "// ..." gaps
7. Follow all constraints from the root `SPECIFICATION.md` (constructor injection,
   `@Getter @Setter` not `@Data`, no DaisyUI, MapStruct for mapping, paginated lists)
8. Tag every user story ID, NFR ID, constraint ID, and mockup screen with its version
   from PRD.md (e.g., `USHM00228 [v1.0.3]`)
9. Include a "Removed / Replaced" subsection listing deprecated items from previous
   versions — showing the removed ID, the version that removed it, the replacement ID
   (if any), and a brief reason. This ensures the spec captures the full evolution of
   requirements and what was removed from the implementation.

**CRITICAL: URL routing must be module-based, NOT role-prefixed.** Controller URLs
use the module name only:
- Page controller: `@RequestMapping("/corridor")` — NOT `@RequestMapping("/hub_administrator/corridor")`
- Fragment controller: `@RequestMapping("/corridor/fragments")` — NOT `@RequestMapping("/api/content/hub_administrator/corridor")`
- Sidebar nav href: `/corridor` — NOT `/hub_administrator/corridor`
- Pagination base URL: `/corridor` — NOT `/hub_administrator/corridor`

The mockup role folder (e.g., `mockup/hub_administrator/content/corridor.html`) determines
which role can access the page, expressed as `@PreAuthorize` annotations on the controller
class or methods:
```java
@Controller
@RequestMapping("/corridor")
@PreAuthorize("hasRole('HUB_ADMINISTRATOR')")
@RequiredArgsConstructor
class CorridorPageController implements PaginationAware, ThemeAware, AuthAware { ... }
```

If a module is accessible by multiple roles, use `hasAnyRole()`:
```java
@PreAuthorize("hasAnyRole('HUB_ADMINISTRATOR', 'HUB_OPERATION_SUPPORT')")
```

The code sample templates for Service, DTOs, Document/Entity, Repository, Mapper,
ServiceImpl, PageController, View Models, and JTE templates follow the same patterns
shown in the original Section 10 code samples above (now moved to this Part B).
Substitute actual module data from context files for every placeholder.
