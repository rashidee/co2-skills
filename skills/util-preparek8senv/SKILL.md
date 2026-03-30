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
  - Operating system, Kubernetes distribution/version
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

**Default values for localhost:**
- Host: `localhost`
- Ports: technology defaults (MongoDB: `27017`, MySQL: `3306`, Redis: `6379`, RabbitMQ AMQP: `5672`, RabbitMQ Admin: `15672`, Keycloak: `8180`, Meilisearch: `7700`, Kong Proxy: `8000`, Kong Admin: `8001`, Mailcatcher SMTP: `1025`, Mailcatcher HTTP: `1080`)
- Username/Password: from CLAUDE.md rules section (default: `bestrnd` / `B3st1n3t@2025`), or technology defaults (e.g., RabbitMQ: `guest`/`guest`, MongoDB: no auth)

**Default values for remote environments:**
- Host: use IP from CLAUDE.md SSH configuration, or `TODO`
- Ports: same technology defaults (NodePort will be used in K8s manifests)
- Username/Password: from CLAUDE.md rules section, or `TODO`

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
    hub_cache.yaml
    hub_core_database.yaml
    ...
  home_server/
    namespace.yaml
    smtp_server.yaml
    hub_cache.yaml
    hub_core_database.yaml
    ...
```

#### 3b. Generate Namespace Manifest

For each environment, create `environment/<env>/namespace.yaml`:
- Namespace name: project code in lowercase (e.g., `urp`)
- Identical across all environments

#### 3c. Generate Per-Service Manifests

For each 3rd party application, generate `environment/<env>/<service_snake_case>.yaml` containing a multi-document YAML (separated by `---`) with:

1. **PersistentVolumeClaim** (if the service requires data persistence — databases, message queues, caches)
2. **ConfigMap** — non-sensitive configuration (hostnames, ports, database names, vhosts)
3. **Secret** — sensitive configuration (passwords, admin credentials), base64-encoded. Read actual values from ENVIRONMENT.md for the matching environment.
4. **StatefulSet** (NOT Deployment) — ensures persistent identity, stable network IDs, and ordered scaling:
   - Use official Docker Hub image with the **exact version** from CLAUDE.md (e.g., `mongo:7.0.6`, `mysql:8.4`, `redis:7.2.1`, `rabbitmq:3.11.9-management`, `quay.io/keycloak/keycloak:26.5.3`)
   - `volumeMounts` referencing the PVC for data persistence
   - `env` or `envFrom` referencing ConfigMap and Secret
   - Resource requests/limits (sensible defaults per technology)
   - Readiness and liveness probes (technology-appropriate)
   - `serviceName` matching the headless Service name
5. **Service** — two services per StatefulSet:
   - **Headless Service** (`clusterIP: None`) — for StatefulSet stable DNS
   - **NodePort Service** — for remote access from developer machines and CLI tools.
     Use a deterministic NodePort assignment strategy to avoid conflicts:
     - Base port = `30000 + (index * 10)` where index is the service's position in alphabetical order
     - Map each technology's primary port to the NodePort
6. **Init containers** (if the service depends on another service):
   - E.g., Keycloak depends on Hub Support Database — add an init container that waits for MySQL to be ready before starting Keycloak

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
- Image: `quay.io/keycloak/keycloak:{version}`
- Command: `start-dev` (for development environments)
- PVC: none (stateless — state is in the database)
- Ports: 8080 (HTTP)
- Env: `KC_DB`, `KC_DB_URL`, `KC_DB_USERNAME`, `KC_DB_PASSWORD`, `KEYCLOAK_ADMIN`, `KEYCLOAK_ADMIN_PASSWORD` (from Secret/ConfigMap)
- Init container: wait for MySQL (Hub Support Database) to be ready
- Probe: HTTP GET `/health/ready`

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

For each manifest file in each environment:

1. **If the file does not exist** (CREATE mode): Generate and write the full manifest.
2. **If it already exists** (UPDATE mode):
   - Read the existing manifest
   - Compare against current CLAUDE.md/ENVIRONMENT.md values:
     - Version changed → update image tag
     - New databases added → update init scripts
     - Credentials changed in ENVIRONMENT.md → update Secret values
     - New dependencies → add init containers
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

For **localhost** environments: use `localhost` as host with NodePort.
For **remote** environments: use the SSH IP from CLAUDE.md as host with NodePort.

### 4. Output Summary

Print a summary of all actions taken:

```
## K8s Environment Preparation Summary

### Environments
| Environment | Folder | Status |
|-------------|--------|--------|
| Localhost | environment/localhost | Created |
| Home Server | environment/home_server | Already exists |

### ENVIRONMENT.md
| Action | Details |
|--------|---------|
| Created/Updated | Added 2 environments, 11 services each |

### K8s Manifests Generated
| Environment | Service | File | Status |
|-------------|---------|------|--------|
| localhost | SMTP Server | environment/localhost/smtp_server.yaml | Created |
| localhost | Hub Cache | environment/localhost/hub_cache.yaml | Created |
| localhost | Hub Core Database | environment/localhost/hub_core_database.yaml | Created |
| ... | ... | ... | ... |
| home_server | SMTP Server | environment/home_server/smtp_server.yaml | Created |
| ... | ... | ... | ... |

### NodePort Assignments
| Service | Primary Port | NodePort |
|---------|-------------|----------|
| HC Adapter Message Queue | 5672 | 30000 |
| HC Database | 3306 | 30010 |
| Hub Cache | 6379 | 30020 |
| Hub Core Database | 27017 | 30030 |
| Hub Single Sign On | 8080 | 30040 |
| Hub Support Database | 3306 | 30050 |
| SC API Gateway | 8000 | 30060 |
| SC Adapter Message Queue | 5672 | 30070 |
| SC Database | 3306 | 30080 |
| SMTP Server | 1025 | 30090 |
```

## Important Rules

- **NEVER remove, modify, or delete existing values** in ENVIRONMENT.md. Only add missing sections, services, and config keys.
- **StatefulSet over Deployment** — all 3rd party services use StatefulSet to ensure persistent identity, stable network IDs, and data persistence across pod restarts.
- **PersistentVolumeClaim** is mandatory for any service that stores data (databases, message queues, caches). Ephemeral services (Mailcatcher, Kong DB-less) are exempt.
- **NodePort for remote access** — every service must have a NodePort Service so developers can connect using local CLI tools from DEVTOOL.md.
- **Deterministic NodePort assignment** — use alphabetical ordering of service names to assign NodePorts starting from 30000 with interval of 10. This ensures consistent ports across environments and avoids conflicts.
- **Init containers for dependencies** — if a 3rd party app depends on another (e.g., Keycloak → MySQL), add an init container that waits for the dependency to be ready.
- **Official Docker images only** — use official images from Docker Hub or vendor registries. Pin exact versions from CLAUDE.md (never use `latest`).
- **Namespace isolation** — all manifests use the project namespace (project code lowercase). The namespace manifest is generated once per environment.
- **Credential rules from CLAUDE.md** — when generating default credentials for new services, follow the `# Rules` section in CLAUDE.md for standard username/password format.
- **Idempotent execution** — safe to re-run. Existing manifests are updated only where values changed. Manual customizations marked with `# CUSTOM:` are preserved.
- **ENVIRONMENT.md is the credential source** — K8s Secret values come from ENVIRONMENT.md, not hardcoded. If a value is `TODO` in ENVIRONMENT.md, use `TODO` (base64-encoded) in the Secret and log a warning.
- This skill only generates manifests for **3rd party supporting applications** — NOT for custom applications (those are handled by `depgen-k8s`).
