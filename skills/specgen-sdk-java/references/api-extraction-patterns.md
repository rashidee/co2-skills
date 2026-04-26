# API Surface Extraction — Swagger UI / OpenAPI

This reference describes how the skill extracts the upstream API surface from
`PRD.md` and turns it into the SDK's public method list, model classes, and URL
constants.

The extraction is a **multi-step search**: the skill must look for the Swagger UI
URL or OpenAPI document FIRST, fall back to inline endpoint tables in the PRD if
not found, and only then resort to user-stories-only inference (with a `[TODO]`).

---

## Step 1: Scan PRD.md for Swagger / OpenAPI Pointers

In priority order, search the entire `PRD.md` for:

1. URLs ending in:
   - `/swagger-ui.html`
   - `/swagger-ui/index.html`
   - `/swagger-ui/`
   - `/swagger`
   - `/api-docs`
   - `/v3/api-docs` (Springdoc default)
   - `/openapi.json`
   - `/openapi.yaml`
   - `/swagger.json`
   - `/swagger.yaml`

2. Section headings:
   - `## Swagger UI`
   - `## OpenAPI Spec`
   - `## OpenAPI`
   - `## API Reference`
   - `## API Documentation`
   - `## API Surface`

3. Phrases inside `# Architecture Principle` such as:
   - "Swagger UI at <url>"
   - "OpenAPI spec is published at <url>"
   - "API documentation: <url>"

4. Relative paths to local files:
   - `openapi.yaml`, `openapi.json`, `swagger.yaml`, `swagger.json`
   - Often inside `<app_folder>/context/api/` or `<app_folder>/context/openapi/`

Record the FIRST match found (PRD authoring order is meaningful — the most prominent
pointer wins).

### Resolution Rules

| Found                                | `apiSource.kind`         | `apiSource.value`                                           |
|--------------------------------------|--------------------------|-------------------------------------------------------------|
| `https://api.example.com/swagger-ui.html` | `swagger-ui-url`     | the URL                                                     |
| `https://api.example.com/v3/api-docs`     | `openapi-url`        | the URL                                                     |
| `<app_folder>/context/api/openapi.yaml`   | `openapi-path`       | the absolute path                                           |
| Endpoint table in PRD with `Method | Path | Description` | `inline-table`       | the table content                                           |
| Nothing relevant                          | `none`               | n/a — emit `[TODO]` and use user stories                    |

---

## Step 2: Convert Swagger UI URL to OpenAPI URL

Swagger UI is a viewer; the underlying OpenAPI document is what the skill needs. For
each known framework, derive the spec URL from the Swagger UI URL:

| Swagger UI URL pattern                                | OpenAPI URL pattern                                |
|-------------------------------------------------------|----------------------------------------------------|
| `<host>/swagger-ui.html`                              | `<host>/v3/api-docs` (Springdoc) or `<host>/swagger.json` (Swashbuckle/.NET) |
| `<host>/swagger-ui/index.html`                        | `<host>/v3/api-docs`                               |
| `<host>/swagger`                                      | `<host>/swagger/v1/swagger.json` (.NET)            |
| `<host>/swagger/index.html`                           | `<host>/swagger/v1/swagger.json`                   |
| `<host>/api-docs`                                     | `<host>/v3/api-docs`                               |

If the underlying URL is reachable from the host running the skill (it usually is for
Springdoc on `localhost`), fetch it and parse it with a YAML/JSON parser. Otherwise,
emit a `[TODO]` reminding the team to provide the spec URL or commit `openapi.yaml`
into the application's context folder.

---

## Step 3: Parse the OpenAPI Spec

The skill needs to extract:

1. **`info.title`** → confirms the API name (cross-check against CLAUDE.md)
2. **`servers[0].url`** → default base URL for `Config`
3. **`securitySchemes`** → drives the Auth determination
4. **`paths`** → one operation per HTTP method per path. For each operation:
   - `operationId` → method name on the service (camelCase)
   - `tags[0]` → which module the operation belongs to (matches PRD module headings
     or MODEL.md folders)
   - `parameters` → query / path / header inputs of the SDK method
   - `requestBody.content["application/json"].schema` → request DTO
   - `responses["2xx"].content["application/json"].schema` → response DTO
   - `responses` for 4xx/5xx → error mapping table in the module SPEC.md
5. **`components.schemas`** → one immutable model class per schema. Resolve `$ref`s
   into typed Java references. Inline schemas become anonymous inner types ONLY if
   they are simple; otherwise, lift them to a named schema.

### Mapping OpenAPI → Java

| OpenAPI                              | Java type (JDK 8 baseline)                         |
|--------------------------------------|----------------------------------------------------|
| `string`                             | `String`                                           |
| `string` + `format: date`            | `LocalDate`                                        |
| `string` + `format: date-time`       | `Instant`                                          |
| `string` + `format: uuid`            | `String` (avoid `UUID` to keep wire-format simple) |
| `string` + `enum`                    | dedicated `enum` with `wireValue()` mapper         |
| `integer` (`int32`)                  | `Integer` / `int` (nullable vs not)                |
| `integer` (`int64`)                  | `Long` / `long`                                    |
| `number` (`double`)                  | `Double` / `double`                                |
| `boolean`                            | `Boolean` / `bool`                                 |
| `array`                              | `List<T>` (always `Collections.unmodifiableList`)  |
| `object`                             | dedicated DTO class                                |
| `oneOf` / `anyOf`                    | tagged-union class with explicit subtype methods   |
| `additionalProperties: true`         | `Map<String, Object>`                              |

> **Nullability note.** OpenAPI represents nullability via the absence of a property
> from `required`, not via `nullable: true`. The SDK distinguishes nullable fields by
> using boxed types (`Integer`) for nullable; primitives (`int`) for required.

---

## Step 4: Cross-Reference With PRD User Stories

Once the OpenAPI operations are extracted, MATCH each user story to one operation:

1. The user story's verb hints the HTTP method (`view`, `list` → GET; `create`,
   `submit` → POST; `update` → PUT/PATCH; `delete` → DELETE)
2. The user story's noun hints the path resource (`order` → `/orders`)
3. The user story's `[USSDKxxxxx]` tag is recorded next to the matched operation in
   the module SPEC.md's "OpenAPI Operation Mapping" table

**If a user story has no matching operation:**
   - Emit a `[TODO]` in the module SPEC.md: `[TODO] USSDK00xxx has no matching
     OpenAPI operation. Add the operation to the upstream API or remove the story.`

**If an operation has no matching user story:**
   - Still generate the SDK method (downstream consumers may need it) but flag it in
     the module SPEC.md "Operations Without User Story Coverage" subsection.

---

## Step 5: Module Inference from `tags`

OpenAPI operations are grouped by the first entry in `tags`. The skill maps:

```
operation.tags[0]  ↔  PRD module name (case-insensitive, kebab/snake/title equivalent)
```

If `tags` is missing on an operation, infer the module from the URL's first path
segment (e.g., `/orders/{id}` → `Orders` module). Document this fallback in the
SPECIFICATION.md "Module Inference" subsection.

---

## Step 6: Default Base URL

The default base URL on `{{ClientName}}.Builder` defaults to `servers[0].url`:

- If the URL contains template variables (e.g., `{environment}`), substitute the
  defaults documented in `servers[0].variables`.
- If the URL is relative (`/api/v1`), prefix with the host portion of the Swagger UI
  URL.
- If `servers` is absent or empty, fall back to `https://api.example.com` and emit a
  `[TODO]` reminding the team to set the real base URL.

---

## Step 7: Security Scheme → Auth Determination

| OpenAPI `securitySchemes` entry                          | SDK Auth                  |
|----------------------------------------------------------|---------------------------|
| `type: http, scheme: bearer`                             | Bearer token              |
| `type: apiKey, in: header, name: X-API-Key`              | API key (header)          |
| `type: apiKey, in: query, name: api_key`                 | API key (query)           |
| `type: http, scheme: basic`                              | HTTP Basic                |
| `type: oauth2, flows: clientCredentials`                 | Bearer token (with refresh — caller supplies the token; SDK does not implement OAuth2 client credentials flow itself) |
| `type: openIdConnect`                                    | Bearer token (caller supplies) |
| Empty `security: []` on every operation                  | None                      |
| `mutualTLS` mention in description                       | mTLS                      |

If multiple schemes are defined, the SDK supports ALL of them via a configurable
`AuthInterceptor` chain — the consumer picks one in the Builder.

---

## Step 8: Generate Per-Module Operation Mapping Table

For each module, the SPEC.md must contain a table like:

```markdown
| User Story | OpenAPI Op ID | Method | Path             | Request | Response |
|------------|---------------|--------|------------------|---------|----------|
| USSDK00012 | listOrders    | GET    | /orders          | (query) | OrdersPage |
| USSDK00013 | getOrderById  | GET    | /orders/{id}     | —       | Order      |
| USSDK00014 | createOrder   | POST   | /orders          | OrderCreateRequest | Order |
```

This table is the **single source of truth** for the module — every service method,
URL constant, request DTO, and response DTO traces back to one row.

---

## Fallback: No Spec, No Tables (User Stories Only)

If neither an OpenAPI spec nor an inline endpoint table is found in PRD.md:

1. Emit at the top of `SPECIFICATION.md`:

   ```markdown
   > **[TODO]** No Swagger UI URL or OpenAPI spec was found in PRD.md.
   > The SDK API surface below is inferred from user stories alone and may drift
   > from the real API. Provide an OpenAPI document at
   > `<app_folder>/context/api/openapi.yaml` (or a Swagger UI URL in PRD.md) and
   > re-run the skill to refresh paths, schemas, and operation IDs.
   ```

2. For every user story, infer:
   - HTTP method (verb-based, see Step 4)
   - Path (`/<plural-resource>` for list, `/<plural-resource>/{id}` for single)
   - DTO names from MODEL.md (if present) or from the story noun

3. Each per-module SPEC.md gets `[TODO]` markers next to each method declaring "path
   inferred — verify against real API".

This path is the LAST RESORT. The SDK is only useful if it talks to a real API; the
generated spec must make that gap loud and impossible to miss.

---

## Persisting the Source Decision

The `SPECIFICATION.md` "Project Overview" → "API Surface Source" subsection MUST
record:

| Field             | Value                                                          |
|-------------------|----------------------------------------------------------------|
| `apiSource.kind`  | one of: `swagger-ui-url`, `openapi-url`, `openapi-path`, `inline-table`, `none` |
| `apiSource.value` | the URL, path, or `n/a`                                        |
| `apiSource.fetchedAt` | ISO-8601 timestamp if fetched                              |
| `apiSource.operationCount` | number of operations parsed (or `unknown`)            |
| `apiSource.schemaCount`    | number of schemas parsed (or `unknown`)               |

This metadata makes it trivial to detect drift on the next regeneration and to
diagnose stale specs.
