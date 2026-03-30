---
name: util-preparek8senv
description: >
  Prepare Kubernetes environment infrastructure by generating K8s manifests for all 3rd party
  supporting applications across all environments defined in CLAUDE.md. Creates/updates
  ENVIRONMENT.md with per-environment configs and credentials, then generates persistent
  StatefulSet-based K8s manifests for each 3rd party application (databases, message queues,
  caches, SSO, API gateways, etc.) in environment/<env>/ folders. Ensures all services are
  remotely accessible using tools from DEVTOOL.md.
  Trigger on keywords: "prepare k8s environment", "prepare kubernetes", "setup k8s infra",
  "generate k8s manifests for 3rd party", "prepare environment", "setup infrastructure",
  "prepare k8s", "init k8s environment", "scaffold k8s environment".
  Accepts no arguments — reads all configuration from CLAUDE.md, ENVIRONMENT.md, and DEVTOOL.md.
---

# Util Prepare K8s Environment

Prepare Kubernetes infrastructure by generating K8s manifests for all 3rd party supporting
applications. One set of manifests per environment defined in CLAUDE.md. This skill creates
the infrastructure layer that custom applications depend on.

## Input Resolution

This skill requires no arguments. All configuration is read from the project root:

| File | Purpose | Required |
|------|---------|----------|
| `CLAUDE.md` | Project detail, environments, 3rd party applications, rules | Yes |
| `ENVIRONMENT.md` | Per-environment configs and credentials for 3rd party apps | Created if missing |
| `DEVTOOL.md` | Local development tools (CLI paths for kubectl, mysql, mongosh, etc.) | Yes |

## Workflow

### 1. Parse CLAUDE.md

Read CLAUDE.md from the project root and extract:

- **Project Code**: From `# Project Detail` → Project Code (e.g., `URP`). Used as K8s namespace (lowercase: `urp`).
- **Environment list**: Each `## <Environment Name>` heading under `# Environment` is an environment. Extract:
  - Environment name (e.g., "Localhost", "Home Server")
  - Snake_case identifier for folder name (e.g., `localhost`, `home_server`)
  - Domain name (e.g., `localhost`, `home.server`) — from the `- Domain:` field
  - Deployment Type (e.g., `Manual`, `Kubernetes`) — from the `- Deployment Type:` field
  - Operating system, Kubernetes distribution/version (for Kubernetes environments)
  - SSH configuration (IP, port) — for remote environments
  - Kube config reference — if mentioned
- **3rd Party Application list**: Each `## <Application Name>` heading under `# Supporting 3rd Party Applications`. Extract:
  - Application name (e.g., "Hub Cache", "Hub Core Database")
  - Technology and version (e.g., "Redis version 7.2.1", "MongoDB version 7.0.6")
  - Databases/database names if applicable
  - Dependencies (`- Depends on:` list)
- **Credential rules**: From `# Rules` section — standard username/password format for new accounts.

**Stop conditions:**
- If `# Environment` section is missing or has no environments → stop with error
- If `# Supporting 3rd Party Applications` section is missing or has no entries → stop with error
- If `DEVTOOL.md` does not exist → stop with error
- If no environment has `Deployment Type: Kubernetes` → stop with error: "No Kubernetes environments found in CLAUDE.md. This skill only generates manifests for environments with Deployment Type: Kubernetes."

### 1b. Environment Deployment Type Decision Tree

After parsing all environments, classify each by its `Deployment Type` field:

```
For each environment in CLAUDE.md:
  ├── Deployment Type = "Kubernetes"
  │     → INCLUDE: Generate K8s manifests (Step 3) and sync ENVIRONMENT.md (Step 2)
  │
  ├── Deployment Type = any other value (e.g., "Manual", "Docker Compose", etc.)
  │     → SKIP: Do not generate K8s manifests for this environment
  │       Log: "Skipping <Environment Name> — Deployment Type: <type> (not Kubernetes)"
  │
  └── Deployment Type field is missing
        → SKIP: Do not generate K8s manifests for this environment
          Log: "Skipping <Environment Name> — no Deployment Type specified"
```

Only environments marked `Deployment Type: Kubernetes` proceed to Step 2 (ENVIRONMENT.md sync) and Step 3 (K8s manifest generation). All other environments are skipped entirely and reported in the output summary (Step 4).

### 2. Sync ENVIRONMENT.md

#### 2a. ENVIRONMENT.md Does Not Exist

Create a new `ENVIRONMENT.md` at the project root using the template structure below.

#### 2b. ENVIRONMENT.md Already Exists

Treat as **append-only**:
1. Parse the existing ENVIRONMENT.md to identify which environment sections (`# <Environment Name>`) and which service subsections (`## <Service Name>`) exist.
2. Compare against CLAUDE.md:
   - **Missing environment section**: Append the entire environment section with all 3rd party app subsections.
   - **Missing service subsection** under an existing environment: Append the missing service subsection at the end of that environment section.
   - **Missing config keys** under an existing service: Append the missing key lines with `TODO` placeholder values.
   - **Existing config keys**: Do **nothing** — leave existing values as-is, even if they differ from defaults.
3. **Never remove, modify, or reorder** existing sections, headings, or values.

#### 2c. ENVIRONMENT.md Template

```markdown
# Context
- This document contains environment-specific variables and credentials for all 3rd party
  supporting applications used in this project.
- Organized by environment as defined in CLAUDE.md.
- **This file must NOT be committed to version control.**

---

# {{Environment Name}}
{{For localhost:}}
- Kube Config: `{{path to kube config or TODO}}`
{{For remote environments:}}
- SSH Credentials:
  - Username: `{{username or TODO}}`
  - Password: `{{password or TODO}}`
- Kube Config: `{{path to kube config or TODO}}`

## {{3rd Party Application Name}}
- {{Technology}} version {{version}}
{{Technology-specific fields — see Section 2d}}

---
```

#### 2d. Technology-Specific Config Fields

For each 3rd party application, include config fields based on its technology type.
Use credential rules from CLAUDE.md `# Rules` section for default username/password values.
Use `TODO` for any value not yet known.

| Technology | Config Fields |
|-----------|---------------|
| MongoDB | Host, Port, Username, Password, Databases (list) |
| MySQL | Host, Port, Username, Password, Databases (list) |
| Redis | Host, Port, Password, Database Index |
| RabbitMQ | Host, AMQP Port, Admin Port, Username, Password, Vhost |
| Keycloak | Host, HTTP Port, Admin Username, Admin Password, Realm, Client IDs |
| Meilisearch | Host, Port, Master Key |
| Kong | Proxy Host, Proxy Port, Admin Host, Admin Port |
| Mailcatcher | SMTP Host, SMTP Port, HTTP Port |

#### 2e. Custom Docker Image Override

Any service in ENVIRONMENT.md may optionally specify a custom Docker image source using one of
these two fields (mutually exclusive — if both are present, `Docker File` takes precedence):

- **`Docker File`**: A relative path (Markdown link) to a Dockerfile in the project repository.
  Format: `- Docker File: [Dockerfile](path/to/Dockerfile)`
  The image will be built locally and referenced by a generated name: `{project-code}-{service-kebab-case}:latest`
  (e.g., `urp-hub-single-sign-on:latest`).

- **`Docker Image`**: A fully qualified Docker image reference.
  Format: `- Docker Image: \`my-registry.io/my-org/my-image:1.0.0\``
  The image will be used as-is in the K8s manifest.

When neither field is present, the skill falls back to the **official Docker Hub image** with the
exact version from CLAUDE.md (default behavior).

These fields are **per-environment** — one environment may use a custom Dockerfile while another
uses the official image. The skill must check each environment's service section independently.

**Default values for all environments:**
- Host: use the **Domain** from the environment's `- Domain:` field in CLAUDE.md (e.g., `localhost`, `home.server`). If no Domain is specified, fall back to `localhost` for localhost environments or the SSH IP for remote environments. If neither is available, use `TODO`.
- Ports: technology defaults (MongoDB: `27017`, MySQL: `3306`, Redis: `6379`, RabbitMQ AMQP: `5672`, RabbitMQ Admin: `15672`, Keycloak: `8180`, Meilisearch: `7700`, Kong Proxy: `8000`, Kong Admin: `8001`, Mailcatcher SMTP: `1025`, Mailcatcher HTTP: `1080`)
- Username/Password: from CLAUDE.md rules section (default: `bestrnd` / `B3st1n3t@2025`), or technology defaults (e.g., RabbitMQ: `guest`/`guest`, MongoDB: no auth)

### 3. Generate K8s Manifests

#### 3a. Ensure Folder Structure

1. Check if `environment/` folder exists at the project root. If not, create it.
2. For each environment from CLAUDE.md, check if `environment/<env_snake_case>/` exists. If not, create it.

Resulting structure:
```
environment/
  localhost/
    namespace.yaml
    smtp_server.yaml
    hub_cache_pvc.yaml
    hub_cache.yaml
    hub_core_database_pvc.yaml
    hub_core_database.yaml
    ...
  home_server/
    namespace.yaml
    smtp_server.yaml
    hub_cache_pvc.yaml
    hub_cache.yaml
    hub_core_database_pvc.yaml
    hub_core_database.yaml
    ...
```

#### 3b. Generate Namespace Manifest

For each environment, create `environment/<env>/namespace.yaml`:
- Namespace name: project code in lowercase (e.g., `urp`)
- Identical across all environments

#### 3c. Generate PVC Manifests (Separate Files)

For each 3rd party application that requires data persistence (databases, message queues, caches),
generate a **separate** PVC file: `environment/<env>/<service_snake_case>_pvc.yaml`.

**Why separate files:** If the PVC is bundled inside the service manifest, running
`kubectl delete -f <service>.yaml` followed by `kubectl apply -f <service>.yaml` would delete
and recreate the PVC, destroying all persisted data. Keeping the PVC in its own file ensures
that `kubectl delete -f <service>.yaml` only removes the StatefulSet, ConfigMap, Secret, and
Services — the PVC (and its data) remains untouched.

The PVC file contains only the PersistentVolumeClaim resource:

```yaml
# ==============================================================================
# {{Application Name}} ({{Technology}}) — PersistentVolumeClaim
# ==============================================================================
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{service-name}}-pvc
  namespace: {{namespace}}
  labels:
    app: {{service-name}}
    project: {{project-code}}
    environment: {{environment}}
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: {{size — see Section 3d for defaults per technology}}
```

#### 3d-svc. Generate Per-Service Manifests

For each 3rd party application, generate `environment/<env>/<service_snake_case>.yaml` containing a multi-document YAML (separated by `---`) with:

1. **ConfigMap** — non-sensitive configuration (hostnames, ports, database names, vhosts)
2. **Secret** — sensitive configuration (passwords, admin credentials), base64-encoded. Read actual values from ENVIRONMENT.md for the matching environment.
3. **StatefulSet** (NOT Deployment) — ensures persistent identity, stable network IDs, and ordered scaling:
   - **Image resolution** (checked per-environment in ENVIRONMENT.md — see Section 2e):
     1. If `Docker File` is specified → use `{project-code}-{service-kebab-case}:latest` as image name.
        - For Docker Desktop K8s (localhost): set `imagePullPolicy: Never` (image is built locally and already available).
        - For Microk8s / remote K8s: set `imagePullPolicy: IfNotPresent` (image must be imported beforehand).
        - Do NOT include `args` or `command` — the Dockerfile's `ENTRYPOINT`/`CMD` handles startup.
        - Add a comment block at the top of the manifest with the build command:
          - Localhost: `docker build -t {image-name} {dockerfile-context}/`
          - Remote: `docker build -t {image-name} {dockerfile-context}/` + `docker save {image-name} | ssh {user}@{host} "microk8s ctr image import -"`
     2. If `Docker Image` is specified → use the exact image reference as-is. Set `imagePullPolicy: IfNotPresent`.
     3. Otherwise → use the official Docker Hub image with the exact version from CLAUDE.md (e.g., `mongo:7.0.6`, `mysql:8.4`). Include `args`/`command` as needed by the technology.
   - `volumeMounts` referencing the PVC (defined in the separate `_pvc.yaml` file) for data persistence
   - `env` or `envFrom` referencing ConfigMap and Secret
   - Resource requests/limits (sensible defaults per technology)
   - Readiness and liveness probes (technology-appropriate)
   - `serviceName` matching the headless Service name
4. **Service** — two services per StatefulSet:
   - **Headless Service** (`clusterIP: None`) — for StatefulSet stable DNS
   - **NodePort Service** — for remote access from developer machines and CLI tools.
     Use a deterministic NodePort assignment strategy to avoid conflicts:
     - Base port = `30000 + (index * 10)` where index is the service's position in alphabetical order
     - Map each technology's primary port to the NodePort
5. **Init containers** (if the service depends on another service):
   - E.g., Keycloak depends on Hub Support Database — add an init container that waits for MySQL to be ready before starting Keycloak

**Important:** The service manifest must NOT contain the PersistentVolumeClaim. It only references the PVC by `claimName` in the StatefulSet's `volumes` section.

#### 3d. Technology-Specific Manifest Details

**MongoDB:**
- Image: `mongo:{version}`
- PVC: 5Gi default
- Ports: 27017
- Env: `MONGO_INITDB_ROOT_USERNAME`, `MONGO_INITDB_ROOT_PASSWORD` (from Secret, or skip if no auth)
- Init script: create databases listed in CLAUDE.md via ConfigMap-mounted init script
- Probe: `mongosh --eval "db.adminCommand('ping')"`

**MySQL:**
- Image: `mysql:{version}`
- PVC: 5Gi default
- Ports: 3306
- Env: `MYSQL_ROOT_PASSWORD` (from Secret)
- Init script: create databases listed in CLAUDE.md via ConfigMap-mounted `/docker-entrypoint-initdb.d/` script
- Probe: `mysqladmin ping -h localhost`

**Redis:**
- Image: `redis:{version}`
- PVC: 1Gi default
- Ports: 6379
- Env: `REDIS_PASSWORD` (from Secret, optional — skip if no auth)
- Probe: `redis-cli ping`

**RabbitMQ:**
- Image: `rabbitmq:{version}-management` (include management plugin for admin UI)
- PVC: 2Gi default
- Ports: 5672 (AMQP), 15672 (Management UI)
- Env: `RABBITMQ_DEFAULT_USER`, `RABBITMQ_DEFAULT_PASS`, `RABBITMQ_DEFAULT_VHOST` (from Secret/ConfigMap)
- Probe: `rabbitmq-diagnostics -q ping`

**Keycloak:**
- Image resolution: check ENVIRONMENT.md for `Docker File` or `Docker Image` first (see Section 2e).
  - If custom image: use generated/specified image name; do NOT add `args: ["start-dev"]` (Dockerfile handles it). Use HTTP Port from ENVIRONMENT.md (typically `8180` for custom builds) for containerPort, probes, and services. Add `KC_HTTP_PORT` to ConfigMap.
  - If official image: `quay.io/keycloak/keycloak:{version}` with `args: ["start-dev"]`. Default port `8080`.
- PVC: none (stateless — state is in the database)
- Env: `KC_DB`, `KC_DB_URL`, `KC_DB_USERNAME`, `KC_DB_PASSWORD`, `KC_HEALTH_ENABLED`, `KC_HTTP_ENABLED`, `KC_HOSTNAME_STRICT`, `KC_HTTP_PORT`, `KEYCLOAK_ADMIN`, `KEYCLOAK_ADMIN_PASSWORD` (from Secret/ConfigMap)
- Init container: wait for MySQL (Hub Support Database) to be ready
- Probe: HTTP GET `/health/ready` on the configured HTTP port

**Kong API Gateway:**
- Image: `kong:{version}`
- PVC: none (stateless — DB-less mode or state in database)
- Ports: 8000 (Proxy), 8001 (Admin API)
- Env: `KONG_DATABASE=off`, `KONG_PROXY_LISTEN=0.0.0.0:8000`, `KONG_ADMIN_LISTEN=0.0.0.0:8001`
- Probe: HTTP GET `/status` on admin port

**Mailcatcher (SMTP):**
- Image: `schickling/mailcatcher:{version}` or `sj26/mailcatcher:latest` (if specific version unavailable)
- PVC: none (ephemeral — mail is not persisted)
- Ports: 1025 (SMTP), 1080 (HTTP UI)
- Probe: TCP check on port 1025

**Meilisearch:**
- Image: `getmeili/meilisearch:v{version}`
- PVC: 2Gi default
- Ports: 7700
- Env: `MEILI_MASTER_KEY` (from Secret)
- Probe: HTTP GET `/health`

#### 3e. Create or Update Logic

For each service in each environment, there are up to two files to manage:
- `<service_snake_case>_pvc.yaml` — PVC file (only for services that require persistence)
- `<service_snake_case>.yaml` — service manifest (ConfigMap, Secret, StatefulSet, Services)

For each file:

1. **If the file does not exist** (CREATE mode): Generate and write the manifest.
2. **If it already exists** (UPDATE mode):
   - Read the existing manifest
   - Compare against current CLAUDE.md/ENVIRONMENT.md values:
     - Version changed → update image tag
     - New databases added → update init scripts
     - Credentials changed in ENVIRONMENT.md → update Secret values
     - New dependencies → add init containers
     - Storage size changed → update PVC (in `_pvc.yaml` only)
   - Preserve any resources marked with `# CUSTOM:` comments
   - Update only the parts that changed
   - Log what was changed

#### 3f. Remote Accessibility

All NodePort Services must be accessible from the developer's machine using the CLI tools
specified in DEVTOOL.md. Include a comment block at the top of each service manifest with
connection instructions:

```yaml
# Service: {{Application Name}} ({{Technology}} {{version}})
# Environment: {{Environment Name}}
#
# Connect from local machine:
#   {{CLI tool from DEVTOOL.md}} {{connection command using NodePort}}
#
# Example:
#   mysql -h {{host}} -P {{nodeport}} -u {{username}} -p{{password}}
#   mongosh mongodb://{{host}}:{{nodeport}}/{{database}}
#   rdcli -h {{host}} -p {{nodeport}}
#   rabbitmqadmin -H {{host}} -P {{nodeport}} list queues
---
```

For **all** environments: use the **Domain** from the environment's `- Domain:` field in CLAUDE.md as host with NodePort (e.g., `localhost`, `home.server`). If no Domain is specified, fall back to `localhost` for localhost environments or the SSH IP for remote environments.

### 4. Output Summary

Print a summary of all actions taken:

```
## K8s Environment Preparation Summary

### Environments
| Environment | Deployment Type | Folder | Status |
|-------------|----------------|--------|--------|
| Localhost | Manual | — | Skipped (not Kubernetes) |
| Home Server | Kubernetes | environment/home_server | Created |

### ENVIRONMENT.md
| Action | Details |
|--------|---------|
| Created/Updated | Added 1 Kubernetes environment, 11 services |

### K8s Manifests Generated
| Environment | Service | File | Status |
|-------------|---------|------|--------|
| home_server | SMTP Server | environment/home_server/smtp_server.yaml | Created |
| home_server | Hub Cache (PVC) | environment/home_server/hub_cache_pvc.yaml | Created |
| home_server | Hub Cache | environment/home_server/hub_cache.yaml | Created |
| home_server | Hub Core Database (PVC) | environment/home_server/hub_core_database_pvc.yaml | Created |
| home_server | Hub Core Database | environment/home_server/hub_core_database.yaml | Created |
| ... | ... | ... | ... |

### NodePort Assignments
| Service | Primary Port | NodePort |
|---------|-------------|----------|
| HC Adapter Message Queue | 5672 | 30000 |
| HC Database | 3306 | 30010 |
| Hub Cache | 6379 | 30020 |
| Hub Core Database | 27017 | 30030 |
| Hub Single Sign On | 8180 (custom) / 8080 (official) | 30040 |
| Hub Support Database | 3306 | 30050 |
| SC API Gateway | 8000 | 30060 |
| SC Adapter Message Queue | 5672 | 30070 |
| SC Database | 3306 | 30080 |
| SMTP Server | 1025 | 30090 |
```

## Important Rules

- **NEVER remove, modify, or delete existing values** in ENVIRONMENT.md. Only add missing sections, services, and config keys.
- **StatefulSet over Deployment** — all 3rd party services use StatefulSet to ensure persistent identity, stable network IDs, and data persistence across pod restarts.
- **PersistentVolumeClaim in separate files** — PVCs MUST be generated in their own `<service>_pvc.yaml` files, never bundled inside the service manifest. This prevents accidental data loss when recreating services with `kubectl delete -f` / `kubectl apply -f`. The service manifest references the PVC by `claimName` only.
- **PersistentVolumeClaim** is mandatory for any service that stores data (databases, message queues, caches). Ephemeral services (Mailcatcher, Kong DB-less) are exempt.
- **NodePort for remote access** — every service must have a NodePort Service so developers can connect using local CLI tools from DEVTOOL.md.
- **Deterministic NodePort assignment** — use alphabetical ordering of service names to assign NodePorts starting from 30000 with interval of 10. This ensures consistent ports across environments and avoids conflicts.
- **Init containers for dependencies** — if a 3rd party app depends on another (e.g., Keycloak → MySQL), add an init container that waits for the dependency to be ready.
- **Image resolution precedence** — for each service in each environment, check ENVIRONMENT.md for `Docker File` or `Docker Image` fields first (see Section 2e). Only fall back to official Docker Hub images when neither is specified. For custom Dockerfile builds, use `{project-code}-{service-kebab-case}:latest` as the image name. For official images, pin exact versions from CLAUDE.md (never use `latest`).
- **Namespace isolation** — all manifests use the project namespace (project code lowercase). The namespace manifest is generated once per environment.
- **Credential rules from CLAUDE.md** — when generating default credentials for new services, follow the `# Rules` section in CLAUDE.md for standard username/password format.
- **Idempotent execution** — safe to re-run. Existing manifests are updated only where values changed. Manual customizations marked with `# CUSTOM:` are preserved.
- **ENVIRONMENT.md is the credential source** — K8s Secret values come from ENVIRONMENT.md, not hardcoded. If a value is `TODO` in ENVIRONMENT.md, use `TODO` (base64-encoded) in the Secret and log a warning.
- This skill only generates manifests for **3rd party supporting applications** — NOT for custom applications (those are handled by `depgen-k8s`).
