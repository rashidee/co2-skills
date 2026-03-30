# Dockerfile Pattern — Spring Boot

Multi-stage Dockerfile for Spring Boot applications with optional frontend build.

## Detection Inputs

| Input | Source | Used For |
|---|---|---|
| Java version | `pom.xml` → `<java.version>` | Base image tag |
| Artifact ID | `pom.xml` → `<artifactId>` | JAR filename |
| App version | `pom.xml` → `<version>` | JAR filename |
| Has frontend | `pom.xml` → `frontend-maven-plugin` present | Include Node in build stage |
| Has JTE precompile | `pom.xml` → `jte-maven-plugin` present | Compile templates in build |
| Has actuator | `pom.xml` → `spring-boot-starter-actuator` present | HEALTHCHECK instruction |
| Server port | `application.yml` → `server.port` default | EXPOSE instruction |

## Template — With Frontend Build

```dockerfile
# =============================================================================
# Stage 1: Build
# =============================================================================
FROM azul/zulu-openjdk:{java_version} AS build

WORKDIR /app

# Copy Maven wrapper and pom.xml for dependency caching
COPY pom.xml ./
COPY .mvn .mvn
COPY mvnw ./
RUN chmod +x mvnw && ./mvnw dependency:go-offline -B

# Copy source code
COPY src ./src

# Build the application (includes frontend build via frontend-maven-plugin)
RUN ./mvnw clean package -B -DskipTests

# =============================================================================
# Stage 2: Runtime
# =============================================================================
FROM azul/zulu-openjdk-alpine:{java_version}-jre AS runtime

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy JAR from build stage
COPY --from=build /app/target/{artifact_id}-{version}.jar app.jar

# Set ownership
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Expose application port
EXPOSE {server_port}

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:{server_port}{health_endpoint} || exit 1

# JVM tuning for containers
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

# Production defaults
ENV JTE_DEV_MODE=false
ENV JTE_PRECOMPILED=true
ENV DEVTOOLS_RESTART=false
ENV DEVTOOLS_LIVERELOAD=false

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

## Template — Without Frontend Build

Same as above but the build stage does NOT include `frontend-maven-plugin` steps.
Use `-DskipFrontend=true` if the plugin is present but frontend assets are pre-built
or not applicable.

## Template — Without Maven Wrapper

If the project does not include `mvnw`, use a Maven base image instead:

```dockerfile
FROM maven:3.9-azul-{java_version} AS build
```

And replace `./mvnw` with `mvn` in the build commands.

## Notes

- **Spring DevTools** is automatically excluded from the production JAR by
  `spring-boot-maven-plugin` (scope `runtime`, `optional true`). No extra Dockerfile
  handling needed.
- **JTE precompiled templates**: The `jte-maven-plugin` runs during `mvn package`
  (at `process-classes` phase) and produces precompiled template classes. The plugin's
  `<targetDirectory>` must be set to `${project.build.directory}/classes` (i.e.,
  `target/classes/`) so that precompiled templates are included in the fat JAR by
  `spring-boot-maven-plugin`. Do NOT output to `target/jte-classes/` — that directory
  is NOT included in the fat JAR, causing `TemplateNotFoundException` at runtime.
  Setting `JTE_PRECOMPILED=true` in the container tells JTE to use these precompiled
  classes.
- **Tailwind CSS + JTE in Docker**: When using a separate Node stage for the frontend
  build (instead of `frontend-maven-plugin`), Tailwind CSS cannot scan JTE templates
  unless they are explicitly copied into the frontend stage. If `app.css` contains
  `@source` directives pointing to JTE template directories (e.g.,
  `@source "../../../jte"`), add `COPY src/main/jte /jte` to the Node stage. This is
  NOT needed when using `frontend-maven-plugin` because Node runs within the Maven
  build context where all source files are already present.
- **Layered JAR** (optional optimization): Spring Boot 3.x supports layered JARs for
  better Docker caching. Add `<layers><enabled>true</enabled></layers>` to
  `spring-boot-maven-plugin` configuration, then use `java -Djarmode=layertools -jar`
  to extract layers in the Dockerfile.

## Template — With Separate Frontend Stage (No frontend-maven-plugin)

If the project uses Vite/Tailwind as a separate build step (not via `frontend-maven-plugin`),
use a dedicated Node stage. **Critical**: If the CSS framework (e.g., Tailwind v4 with `@source`)
scans JTE templates for utility classes, the JTE templates must be copied into the frontend
build stage.

**Detection rule**: Use this variant when the project has a standalone `frontend/` or
`src/main/frontend/` directory with its own `package.json` (not managed by
`frontend-maven-plugin`), OR the user explicitly requests a separate Node stage.

```dockerfile
# =============================================================================
# Stage 1: Frontend build (Vite + Tailwind)
# =============================================================================
FROM node:22-alpine AS frontend-builder

WORKDIR /frontend

COPY {frontend_dir}/package.json {frontend_dir}/package-lock.json ./
RUN npm ci --ignore-scripts

COPY {frontend_dir}/ ./

# CRITICAL: If Tailwind CSS scans JTE templates (e.g., @source "../../../jte" in app.css),
# copy them so Tailwind can find utility classes. Without this, Tailwind generates CSS
# with missing utility classes because the JTE files are not in this stage's filesystem.
COPY src/main/jte /jte

RUN npm run build

# =============================================================================
# Stage 2: Maven build (Java + JTE precompile)
# =============================================================================
FROM {maven_or_jdk_image} AS build

WORKDIR /app

COPY pom.xml ./
COPY .mvn .mvn
COPY mvnw ./
RUN chmod +x mvnw && ./mvnw dependency:go-offline -B -DskipFrontend=true

COPY src ./src

# Copy frontend build output into static resources
COPY --from=frontend-builder {vite_output_dir} ./src/main/resources/static/assets/

# Build (skip frontend — already built in Stage 1)
RUN ./mvnw clean package -B -DskipTests -DskipFrontend=true

# =============================================================================
# Stage 3: Runtime
# =============================================================================
FROM azul/zulu-openjdk-alpine:{java_version}-jre AS runtime
# ... (same runtime stage as the standard template)
```
