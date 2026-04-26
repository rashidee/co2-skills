# Specification Template — Java SDK (Multi-Release Fat JAR)

This is the authoritative template for the generated specification. The specification is
split into **two types of files**:

1. **`SPECIFICATION.md`** (root) — Table of Contents, shared infrastructure, and all
   library-level sections. Generated once per SDK.
2. **`<module>/SPEC.md`** (per-module) — Self-contained module blueprint. Generated
   once per module from PRD.md.

```
specification/
├── SPECIFICATION.md
├── auth/
│   └── SPEC.md
├── orders/
│   └── SPEC.md
└── ...
```

Placeholders use `{{VARIABLE}}` syntax and must be replaced with values gathered from
context files.

Sections marked **[CONDITIONAL]** must only be included when the corresponding option
is selected.

**CRITICAL: Sample code requirement.** Every component in the spec MUST include a
complete, continuous code sample in a fenced code block (Java, XML, or JSON as
appropriate). The code must be self-explanatory and directly usable as a reference for
a coding agent. Do not describe components with bullet points alone — always accompany
descriptions with full code.

---

# Part A: Root SPECIFICATION.md

Everything from this point until "Part B" goes into the root `SPECIFICATION.md` file.
A coding agent implements these shared sections **first**, before any module work.

---

## Table of Contents

```markdown
## Table of Contents

### Shared Infrastructure
- [1. Project Overview](#1-project-overview)
- [2. Maven Configuration](#2-maven-configuration)
- [3. Source Layout](#3-source-layout)
- [4. OkHttpClient Setup](#4-okhttpclient-setup)
- [5. JSON Support](#5-json-support)
- [6. Error Model](#6-error-model)
- [7. Retry Strategy](#7-retry-strategy)
- [8. Authentication](#8-authentication)
- [9. Top-Level Client Facade](#9-top-level-client-facade)
- [10. Diagnostic Main](#10-diagnostic-main)
- [11. Logging Strategy](#11-logging-strategy)
- [12. Testing Strategy](#12-testing-strategy)
- [13. Packaging & Distribution](#13-packaging-distribution)
- [14. Public API Surface & Compatibility](#14-public-api-surface)
- [15. Fat JAR Verification](#15-fat-jar-verification)
- [16. Architecture Patterns](#16-architecture-patterns)

### Modules
- [Auth](auth/SPEC.md)
- [Orders](orders/SPEC.md)
- ...(one link per module)...
```

---

## Section 1: Project Overview

```markdown
# {{APPLICATION_NAME}} — SDK Technical Specification

**Application Name**: {{APPLICATION_NAME}}
**Artifact ID**: {{ARTIFACT_ID}}
**Group ID**: com.bestinet.urp
**Base Package**: {{BASE_PACKAGE}}
**JDK Baseline**: 8
**JDK Overlay**: 11 (via Multi-Release JAR)
**Build Tool**: Maven 3.9.x
**Description**: {{APP_DESCRIPTION}}
**Versions Covered**: v1.0.0 — v{{LATEST_VERSION}}

### API Surface Source

**Source**: {{Swagger UI URL | OpenAPI Spec URL | OpenAPI Spec Path | "Inferred from user stories — no OpenAPI spec available"}}
**URL/Path**: {{actual url or path or "n/a"}}
**Number of operations**: {{count from spec or "unknown"}}

### Optional Components (Auto-Determined)

**Protocols**: REST{{ + WebSocket | + SSE | + AMQP | + MQTT}}
**Authentication**: {{Bearer | API Key | mTLS | Basic | None}}
**Async API**: {{yes (all) | yes (long ops) | no}}
**Logging**: {{SLF4J facade | JUL}}

### Technology Stack

| Component                | Version       |
|--------------------------|---------------|
| Java baseline (`release`)| 8             |
| Java overlay (`release`) | 11            |
| Maven                    | 3.9.x         |
| OkHttp                   | 4.12.0        |

(Test-only and optional protocol dependencies omitted.)

### Target Consumers

(List each consumer of the SDK, with a one-line description.)

| Consumer Application | Use Case |
|----------------------|----------|
| ...                  | ...      |

### Modules

| Module | Stories | Versions | Spec |
|--------|---------|----------|------|
| Auth   | 4       | 1.0.0    | [SPEC](auth/SPEC.md) |
| Orders | 8       | 1.0.0    | [SPEC](orders/SPEC.md) |
| ...    | ...     | ...      | ... |

### Default Base URL

`{{DEFAULT_BASE_URL}}` (override at runtime via `Client.Builder#baseUrl`)

### Input Sources

- CLAUDE.md: {{path}}
- PRD.md: {{path}}
- Module Model: {{path to MODEL.md or "n/a"}}
- OpenAPI: {{url or path or "n/a"}}
```

---

## Section 2: Maven Configuration

Generate a complete `pom.xml` inside a code block. Key requirements:

- `<packaging>jar</packaging>`
- `<properties>` with `maven.compiler.release=8` baseline
- One runtime dependency: `com.squareup.okhttp3:okhttp:4.12.0`
- Test-scoped: JUnit 5, AssertJ, OkHttp MockWebServer
- `<scope>provided</scope>` for `org.slf4j:slf4j-api` (only if Logging = SLF4J facade)
- `maven-compiler-plugin` with two executions:
  - Default: compiles `src/main/java` to `target/classes` with `<release>8</release>`
  - `compile-java11`: compiles `src/main/java11` to `target/classes-java11` with
    `<release>11</release>`
- `maven-jar-plugin`: adds the JDK 11 overlay into `META-INF/versions/11/` and writes
  `Multi-Release: true` in the manifest
- `maven-shade-plugin`:
  - `<phase>package</phase>` execution
  - Relocates `okhttp3` → `{{BASE_PACKAGE}}.shaded.okhttp3`
  - Relocates `okio` → `{{BASE_PACKAGE}}.shaded.okio`
  - Relocates `kotlin` → `{{BASE_PACKAGE}}.shaded.kotlin`
  - `<ManifestResourceTransformer>` preserving `Multi-Release: true`
  - `<filters>` excluding `META-INF/*.SF`, `*.DSA`, `*.RSA`
- `maven-source-plugin`, `maven-javadoc-plugin`, `maven-surefire-plugin`

The complete `pom.xml` MUST be written out in this section. See
`references/packaging-patterns.md` for the canonical configuration to copy in.

---

## Section 3: Source Layout

Render the full directory tree (see SKILL.md "Source Layout"). Annotate each directory
with one line explaining its purpose. Highlight that:

- `src/main/java/` is the JDK 8 baseline that supplies the COMPLETE public API
- `src/main/java11/` contains overlay implementations only — every overlay class MUST
  also exist in `src/main/java/` with an identical signature
- `src/test/java/` contains MockWebServer-driven tests

---

## Section 4: OkHttpClient Setup

### `http/HttpClientFactory.java`

Sample code (mandatory):

```java
package {{BASE_PACKAGE}}.http;

import okhttp3.ConnectionPool;
import okhttp3.OkHttpClient;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

final class HttpClientFactory {

    private HttpClientFactory() {}

    static OkHttpClient build({{ClientName}}Config config) {
        OkHttpClient.Builder builder = new OkHttpClient.Builder()
            .connectTimeout(config.connectTimeout().toMillis(), TimeUnit.MILLISECONDS)
            .readTimeout(config.readTimeout().toMillis(), TimeUnit.MILLISECONDS)
            .writeTimeout(config.writeTimeout().toMillis(), TimeUnit.MILLISECONDS)
            .connectionPool(new ConnectionPool(
                config.maxIdleConnections(),
                config.keepAliveDuration().toMillis(),
                TimeUnit.MILLISECONDS));

        if (config.credentialProvider() != null) {
            builder.addInterceptor(new AuthInterceptor(config.credentialProvider()));
        }
        builder.addInterceptor(new CorrelationIdInterceptor());
        builder.addInterceptor(new RetryInterceptor(config.retryStrategy()));
        if (Slf4jPresence.AVAILABLE) {
            builder.addInterceptor(new LoggingInterceptor());
        }
        return builder.build();
    }
}
```

Document thread-safety rules:

- The returned `OkHttpClient` is built ONCE per `{{ClientName}}` instance.
- Consumers MUST reuse a single `{{ClientName}}` across the application lifetime.
- Closing: call `client.close()` on application shutdown to release the connection pool.

---

## Section 5: JSON Support

### `json/Json.java`

A minimal hand-written JSON read/write utility. Key requirements:

- No reflection
- Operates on `Object` for read (`Map<String, Object>`, `List<Object>`, `String`,
  `Number`, `Boolean`, `null`) and `JsonWriter`/`JsonReader` token streams for write.
- Supports the surface area the SDK actually needs — no full JSON Pointer, no
  schema validation, no streaming for arbitrary trees.

Sample skeleton:

```java
package {{BASE_PACKAGE}}.json;

import java.io.IOException;
import java.io.Reader;
import java.io.StringReader;
import java.io.Writer;

public final class Json {

    private Json() {}

    public static Object parse(String text) {
        return new JsonReader(new StringReader(text)).readValue();
    }

    public static String stringify(Object value) {
        StringBuilder sb = new StringBuilder();
        new JsonWriter(sb).writeValue(value);
        return sb.toString();
    }

    // JsonReader and JsonWriter implementations follow.
    // Each model class provides its own:
    //   public void toJson(JsonWriter w) { ... }
    //   public static Foo fromJson(JsonReader r) { ... }
}
```

Per-model serializer pattern (every request/response DTO follows this):

```java
public Map<String, Object> toJsonMap() {
    Map<String, Object> m = new LinkedHashMap<>();
    if (id != null)        m.put("id", id);
    if (name != null)      m.put("name", name);
    if (status != null)    m.put("status", status.wireValue());
    return m;
}

public static OrderDto fromJsonMap(Map<String, Object> m) {
    return OrderDto.builder()
        .id((String) m.get("id"))
        .name((String) m.get("name"))
        .status(OrderStatus.fromWireValue((String) m.get("status")))
        .build();
}
```

The full `JsonReader`/`JsonWriter` implementations are non-trivial — generate them in
this section, fully expanded, even if the spec is otherwise terse.

---

## Section 6: Error Model

```java
package {{BASE_PACKAGE}}.error;

public class SdkException extends RuntimeException {
    public SdkException(String message)                  { super(message); }
    public SdkException(String message, Throwable cause) { super(message, cause); }
}

public class SdkApiException extends SdkException {
    private final int statusCode;
    private final String body;
    private final String method;
    private final String url;

    public SdkApiException(int statusCode, String body, String method, String url) {
        super("HTTP " + statusCode + " " + method + " " + url);
        this.statusCode = statusCode;
        this.body       = body;
        this.method     = method;
        this.url        = url;
    }
    public int statusCode() { return statusCode; }
    public String body()    { return body; }
    public String method()  { return method; }
    public String url()     { return url; }
}

public class SdkAuthException        extends SdkApiException { /* 401 / 403 */ }
public class SdkClientException      extends SdkException    { /* local I/O */ }
public class SdkTimeoutException     extends SdkClientException { /* socket timeout */ }
public class SdkRetryExhaustedException extends SdkClientException { /* retry budget */ }
```

---

## Section 7: Retry Strategy

```java
public interface RetryStrategy {
    Decision evaluate(int attempt, Response response, IOException error);

    final class Decision {
        public final boolean retry;
        public final long delayMillis;
        Decision(boolean retry, long delayMillis) {
            this.retry = retry; this.delayMillis = delayMillis;
        }
        public static Decision stop()                  { return new Decision(false, 0L); }
        public static Decision retryIn(long delayMs)   { return new Decision(true, delayMs); }
    }
}
```

Default implementation `ExponentialBackoffRetryStrategy` retries on:

- `IOException` with cause `ConnectException` / `SocketTimeoutException`
- HTTP 502, 503, 504

Backoff: `min(maxBackoff, base * 2^attempt + jitter)`. Default base = 200ms,
maxBackoff = 5s, max attempts = 3.

Provide the full implementation in this section.

---

## Section 8: Authentication [CONDITIONAL]

Refer to the SKILL.md "Authentication" section. Generate the relevant code:

- `BearerCredentialProvider` interface + `StaticBearerCredentialProvider`
- `AuthInterceptor` that reads from the provider, sets `Authorization` header, and on 401
  invokes `provider.refresh()` and replays the request once.
- Redact the `Authorization` header in `LoggingInterceptor`.

For API key, mTLS, or Basic, follow the patterns documented in
`references/http-client-patterns.md`.

---

## Section 9: Top-Level Client Facade

```java
package {{BASE_PACKAGE}};

import okhttp3.OkHttpClient;
import {{BASE_PACKAGE}}.http.HttpClientFactory;
import {{BASE_PACKAGE}}.auth.AuthService;
import {{BASE_PACKAGE}}.orders.OrdersService;

import java.time.Duration;
import java.util.Objects;

public final class {{ClientName}} implements AutoCloseable {

    private final {{ClientName}}Config config;
    private final OkHttpClient http;
    private final AuthService authService;
    private final OrdersService ordersService;

    private {{ClientName}}({{ClientName}}Config config, OkHttpClient http) {
        this.config        = config;
        this.http          = http;
        this.authService   = new {{BASE_PACKAGE}}.auth.internal.AuthServiceImpl(http, config);
        this.ordersService = new {{BASE_PACKAGE}}.orders.internal.OrdersServiceImpl(http, config);
    }

    public static Builder builder() { return new Builder(); }

    public AuthService   auth()   { return authService; }
    public OrdersService orders() { return ordersService; }

    @Override
    public void close() {
        http.dispatcher().executorService().shutdown();
        http.connectionPool().evictAll();
    }

    public static final class Builder {
        private String baseUrl;
        private Duration connectTimeout = Duration.ofSeconds(30);
        private Duration readTimeout    = Duration.ofSeconds(30);
        private Duration writeTimeout   = Duration.ofSeconds(30);
        private RetryStrategy retryStrategy = RetryStrategy.defaultStrategy();
        private CredentialProvider credentialProvider;
        // ...one setter per Config field...

        public Builder baseUrl(String baseUrl)         { this.baseUrl = baseUrl; return this; }
        public Builder connectTimeout(Duration d)      { this.connectTimeout = d; return this; }
        // ...etc...

        public {{ClientName}} build() {
            Objects.requireNonNull(baseUrl, "baseUrl is required");
            {{ClientName}}Config cfg = new {{ClientName}}Config(
                baseUrl, connectTimeout, readTimeout, writeTimeout,
                retryStrategy, credentialProvider /* , ... */);
            return new {{ClientName}}(cfg, HttpClientFactory.build(cfg));
        }
    }
}
```

Generate `{{ClientName}}Config` as a final class with `private final` fields, all
accessors named without `get` prefix (e.g., `baseUrl()`), and a package-private
constructor. No setters.

---

## Section 10: Diagnostic Main [CONDITIONAL]

If included:

```java
package {{BASE_PACKAGE}}.cli;

import {{BASE_PACKAGE}}.{{ClientName}};

public final class Diagnose {
    public static void main(String[] args) throws Exception {
        String baseUrl = System.getenv("BASE_URL");
        String token   = System.getenv("API_TOKEN");
        if (baseUrl == null) {
            System.err.println("BASE_URL env var is required");
            System.exit(2);
        }
        try ({{ClientName}} client = {{ClientName}}.builder()
                .baseUrl(baseUrl)
                /* .credentials(BearerCredentialProvider.of(token)) */
                .build()) {
            System.out.println("SDK initialised against " + baseUrl);
            // Make a single smoke-test call here.
        }
    }
}
```

The fat-JAR manifest's `Main-Class` points to this class only if Diagnose is included.

---

## Section 11: Logging Strategy

If Logging = SLF4J facade:

- `org.slf4j:slf4j-api` is `<scope>provided</scope>` — NOT bundled.
- The SDK uses `LoggerFactory.getLogger(...)`. If the consumer has no SLF4J binding,
  no-op output is the standard SLF4J behaviour.
- The `LoggingInterceptor` is registered ONLY if `org.slf4j.Logger` is on the classpath
  (use a class-presence check):

```java
final class Slf4jPresence {
    static final boolean AVAILABLE = isAvailable();

    private static boolean isAvailable() {
        try {
            Class.forName("org.slf4j.Logger");
            return true;
        } catch (ClassNotFoundException e) {
            return false;
        }
    }
}
```

If Logging = JUL: every component uses `java.util.logging.Logger.getLogger(...)`.
No external dependency. No facade.

---

## Section 12: Testing Strategy

```java
package {{BASE_PACKAGE}}.orders;

import okhttp3.mockwebserver.MockResponse;
import okhttp3.mockwebserver.MockWebServer;
import {{BASE_PACKAGE}}.{{ClientName}};
import {{BASE_PACKAGE}}.orders.model.OrderDto;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class OrdersServiceTest {

    private MockWebServer server;
    private {{ClientName}} client;

    @BeforeEach
    void setUp() throws Exception {
        server = new MockWebServer();
        server.start();
        client = {{ClientName}}.builder()
            .baseUrl(server.url("/").toString())
            .build();
    }

    @AfterEach
    void tearDown() throws Exception {
        client.close();
        server.shutdown();
    }

    @Test
    void getById_returnsOrder() {
        server.enqueue(new MockResponse()
            .setResponseCode(200)
            .setHeader("Content-Type", "application/json")
            .setBody("{\"id\":\"ORD-123\",\"status\":\"NEW\"}"));

        OrderDto order = client.orders().getById("ORD-123");

        assertThat(order.id()).isEqualTo("ORD-123");
        assertThat(order.status()).isEqualTo(OrderStatus.NEW);
    }
}
```

Every module's `*ServiceTest` follows this pattern. Document one full class here; per-
module SPEC.md only lists the test scenarios.

---

## Section 13: Packaging & Distribution

Refer to `references/packaging-patterns.md`. Render the full `maven-shade-plugin` and
`maven-jar-plugin` configurations and the `mvn package` / `mvn deploy` invocations.

---

## Section 14: Public API Surface & Compatibility

- Every public type and method is javadoc'd.
- Every introduction is annotated with `@since` matching the introducing version.
- Removed APIs are `@Deprecated(forRemoval = true, since = "x.y.z")` for at least one
  minor version before removal.
- Optional: `revapi-maven-plugin` configured in the build to fail the build on
  binary-incompatible changes.

---

## Section 15: Fat JAR Verification

Manual verification commands:

```
mvn clean package
jar tf target/{{ARTIFACT_ID}}-{{VERSION}}.jar | head -50
unzip -p target/{{ARTIFACT_ID}}-{{VERSION}}.jar META-INF/MANIFEST.MF
jar tf target/{{ARTIFACT_ID}}-{{VERSION}}.jar | grep "META-INF/versions/11/"
jar tf target/{{ARTIFACT_ID}}-{{VERSION}}.jar | grep -E "^okhttp3/|^okio/|^kotlin/" || echo "ok: no unrelocated transitives"
```

Expected:

- `META-INF/MANIFEST.MF` contains `Multi-Release: true`
- `META-INF/versions/11/` is non-empty
- No `okhttp3/`, `okio/`, or `kotlin/` packages at root (they live under
  `{{BASE_PACKAGE}}/shaded/...`)

Functional verification:

```
JAVA_HOME=/path/to/jdk-8  java -jar target/{{ARTIFACT_ID}}-{{VERSION}}.jar
JAVA_HOME=/path/to/jdk-11 java -jar target/{{ARTIFACT_ID}}-{{VERSION}}.jar
JAVA_HOME=/path/to/jdk-17 java -jar target/{{ARTIFACT_ID}}-{{VERSION}}.jar
JAVA_HOME=/path/to/jdk-21 java -jar target/{{ARTIFACT_ID}}-{{VERSION}}.jar
```

(Only if Diagnose main is included.)

---

## Section 16: Architecture Patterns

Render the table extracted in the SKILL.md "Architecture Principle Extraction" step,
with one row per pattern and a one-paragraph explanation of where it appears in the
codebase. Include only patterns that the PRD's Architecture Principle section actually
mentions, plus the always-on defaults (Adapter, Builder, Strategy).

---

# Part B: Module SPEC.md Template

Everything below is the template for each `<module>/SPEC.md` file. Generate one file
per module from PRD.md.

Each module SPEC.md is **self-contained** — a coding agent picks it up and implements
the entire module independently (after the shared infrastructure from
`SPECIFICATION.md` is in place).

---

## Module SPEC.md Structure

```markdown
# {{ModuleName}} Module — SDK Specification

> Part of [{{APPLICATION_NAME}} SDK Technical Specification](../SPECIFICATION.md)

## Overview

**Module:** {{ModuleName}}
**Package:** {{BASE_PACKAGE}}.{{modulename}}
**Public Service:** {{Module}}Service
**Internal Implementation:** {{BASE_PACKAGE}}.{{modulename}}.internal.{{Module}}ServiceImpl
```

### Traceability

```markdown
## Traceability

### User Stories
| ID | Version | Description |
|---|---|---|
| USSDK00012 | v1.0.0 | List orders by status |

### Non-Functional Requirements
| ID | Version | Description |
|---|---|---|
| NFRSDK0006 | v1.0.0 | Default per-call timeout 30s |

### Constraints
| ID | Version | Description |
|---|---|---|
| CONSSDK003 | v1.0.0 | Backwards-compatible within major version |

### Tests
| ID | Version | Description |
|---|---|---|
| TSDK00009 | v1.0.0 | MockWebServer 200/empty array → empty list |

### References
| ID | Version | Description |
|---|---|---|
| REFSDK001 | v1.0.0 | Upstream API docs at https://api.example.com/docs |

### Removed / Replaced
| ID | Type | Removed In | Replaced By | Reason |
|---|---|---|---|---|
| _None._ | | | | |
```

### OpenAPI Operation Mapping

```markdown
## OpenAPI Operation Mapping

| User Story | Operation ID | Method | Path | Request Schema | Response Schema |
|---|---|---|---|---|---|
| USSDK00012 | listOrders | GET | /orders | (query) | OrdersPage |
| USSDK00013 | getOrderById | GET | /orders/{id} | — | Order |
| USSDK00014 | createOrder | POST | /orders | OrderCreateRequest | Order |
```

If no OpenAPI spec was available, replace the table with one derived from user stories
alone and add `[TODO]` notes where the upstream operation ID is unknown.

### Public API

For each component, render a complete code sample.

#### Service Interface

```java
package {{BASE_PACKAGE}}.{{modulename}};

import {{BASE_PACKAGE}}.{{modulename}}.model.OrderDto;
import {{BASE_PACKAGE}}.common.Page;

import java.util.concurrent.CompletableFuture;

public interface {{Module}}Service {

    /** USSDK00012 — list orders, paginated. @since 1.0.0 */
    Page<OrderDto> list(OrderListQuery query);

    /** USSDK00013 — get order by id. @since 1.0.0 */
    OrderDto getById(String id);

    /** USSDK00014 — create order. @since 1.0.0 */
    OrderDto create(OrderCreateRequest request);

    // Async overloads if Async API = yes
    // CompletableFuture<Page<OrderDto>> listAsync(OrderListQuery query);
}
```

#### Request DTOs

```java
package {{BASE_PACKAGE}}.{{modulename}}.model;

import java.util.Objects;

public final class OrderCreateRequest {

    private final String customerId;
    private final String currency;
    private final long amountCents;

    private OrderCreateRequest(Builder b) {
        this.customerId  = Objects.requireNonNull(b.customerId, "customerId");
        this.currency    = Objects.requireNonNull(b.currency, "currency");
        this.amountCents = b.amountCents;
    }

    public String customerId()  { return customerId; }
    public String currency()    { return currency; }
    public long amountCents()   { return amountCents; }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private String customerId;
        private String currency;
        private long amountCents;
        public Builder customerId(String s)   { this.customerId = s; return this; }
        public Builder currency(String s)     { this.currency = s; return this; }
        public Builder amountCents(long n)    { this.amountCents = n; return this; }
        public OrderCreateRequest build()     { return new OrderCreateRequest(this); }
    }
}
```

#### Response DTOs

```java
package {{BASE_PACKAGE}}.{{modulename}}.model;

public final class OrderDto {

    private final String id;
    private final String customerId;
    private final OrderStatus status;
    private final long amountCents;

    private OrderDto(Builder b) { /* ... */ }

    public String id()           { return id; }
    public String customerId()   { return customerId; }
    public OrderStatus status()  { return status; }
    public long amountCents()    { return amountCents; }

    public static Builder builder() { return new Builder(); }

    public static OrderDto fromJsonMap(java.util.Map<String, Object> m) { /* ... */ }
    public java.util.Map<String, Object> toJsonMap() { /* ... */ }

    public static final class Builder { /* same shape as above */ }
}
```

#### Module Exception

```java
package {{BASE_PACKAGE}}.{{modulename}};

import {{BASE_PACKAGE}}.error.SdkApiException;

public class {{Module}}Exception extends SdkApiException {
    public {{Module}}Exception(int statusCode, String body, String method, String url) {
        super(statusCode, body, method, url);
    }

    public static class NotFound extends {{Module}}Exception { /* 404 */ }
    public static class Conflict extends {{Module}}Exception { /* 409 */ }
}
```

### Internal Implementation

#### URL Constants

```java
package {{BASE_PACKAGE}}.{{modulename}}.internal;

final class OrderPaths {
    private OrderPaths() {}

    static final String LIST           = "/orders";
    static final String BY_ID          = "/orders/%s";

    static String byId(String id) { return String.format(BY_ID, id); }
}
```

#### Service Implementation

```java
package {{BASE_PACKAGE}}.{{modulename}}.internal;

import okhttp3.HttpUrl;
import okhttp3.MediaType;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.Response;

import {{BASE_PACKAGE}}.{{ClientName}}Config;
import {{BASE_PACKAGE}}.error.SdkClientException;
import {{BASE_PACKAGE}}.json.Json;
import {{BASE_PACKAGE}}.{{modulename}}.{{Module}}Service;
import {{BASE_PACKAGE}}.{{modulename}}.{{Module}}Exception;
import {{BASE_PACKAGE}}.{{modulename}}.model.OrderDto;

import java.io.IOException;
import java.util.Map;

public final class {{Module}}ServiceImpl implements {{Module}}Service {

    private static final MediaType JSON = MediaType.parse("application/json; charset=utf-8");

    private final OkHttpClient http;
    private final {{ClientName}}Config config;

    public {{Module}}ServiceImpl(OkHttpClient http, {{ClientName}}Config config) {
        this.http   = http;
        this.config = config;
    }

    @Override
    public OrderDto getById(String id) {
        HttpUrl url = HttpUrl.parse(config.baseUrl())
            .newBuilder()
            .addPathSegments("orders/" + id)
            .build();

        Request req = new Request.Builder()
            .url(url)
            .header("Accept", "application/json")
            .get()
            .build();

        try (Response resp = http.newCall(req).execute()) {
            if (resp.code() == 404) {
                throw new {{Module}}Exception.NotFound(404, body(resp), "GET", url.toString());
            }
            if (!resp.isSuccessful()) {
                throw new {{Module}}Exception(resp.code(), body(resp), "GET", url.toString());
            }
            @SuppressWarnings("unchecked")
            Map<String, Object> map = (Map<String, Object>) Json.parse(body(resp));
            return OrderDto.fromJsonMap(map);
        } catch (IOException e) {
            throw new SdkClientException("GET " + url + " failed", e);
        }
    }

    private static String body(Response resp) throws IOException {
        return resp.body() != null ? resp.body().string() : "";
    }

    // Implementations for list(...) and create(...) follow the same pattern.
}
```

### Error Mapping

```markdown
## Error Mapping

| HTTP Status | Exception                          | Notes                                |
|-------------|------------------------------------|--------------------------------------|
| 400         | {{Module}}Exception (root)         | Validation error from server         |
| 401         | SdkAuthException                   | Triggers 1× refresh-and-retry        |
| 403         | SdkAuthException                   | Surfaced to caller                   |
| 404         | {{Module}}Exception.NotFound       |                                      |
| 409         | {{Module}}Exception.Conflict       |                                      |
| 429         | retried by RetryInterceptor        | Exhaustion → SdkRetryExhaustedException |
| 5xx         | retried by RetryInterceptor        | Exhaustion → {{Module}}Exception     |
```

### Test Plan

Derive scenarios from the PRD `### Test` items for this module. Each row maps to one
JUnit `@Test`.

```markdown
## Test Plan

| Test ID    | Scenario                                        | Mock Response                          |
|------------|-------------------------------------------------|----------------------------------------|
| TSDK00009  | getById on existing order                       | 200 + `{"id":"ORD-1","status":"NEW"}`  |
| TSDK00010  | getById on missing order                        | 404                                    |
| TSDK00011  | list with status filter                         | 200 + array of 3 items                 |
| TSDK00012  | create with invalid amount                      | 400 + error body                       |
| TSDK00013  | retry on 503                                    | 503, 503, 200                          |
```

### External References

```markdown
## External References

| Ref ID    | Version | Title                            | URL                              |
|-----------|---------|----------------------------------|----------------------------------|
| REFSDK001 | v1.0.0  | Upstream Orders API docs         | https://api.example.com/orders   |
```

### Critical Rules for Module SPEC.md

1. Use actual field names from MODEL.md / OpenAPI schemas.
2. Use actual operation IDs and paths from the OpenAPI spec when available.
3. Define service methods mapping to actual user stories (by tag ID).
4. Map HTTP status codes to specific exception subclasses.
5. List user story IDs, NFR IDs, constraint IDs, test IDs, reference IDs with versions.
6. Include complete, continuous code samples — no `// ...` gaps.
7. Follow all SDK constraints (constructor injection, immutable models, no Lombok, no
   reflection-based JSON).
8. Include "Removed / Replaced" subsection.
9. Test plan rows MUST trace back to PRD `### Test` items where possible.
