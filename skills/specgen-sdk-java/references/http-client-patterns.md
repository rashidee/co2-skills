# HTTP Client Patterns — OkHttp Wiring

This reference describes the canonical OkHttp wiring the generated spec must produce.
The SDK uses **one shared `OkHttpClient`** for the entire library lifetime; every
service routes through that client; every cross-cutting concern is implemented as an
`Interceptor`.

---

## Why a Single Shared `OkHttpClient`

OkHttp's `OkHttpClient` is **expensive to create** (it owns a connection pool, a
dispatcher, and a thread pool) and **cheap to share** (it is fully thread-safe). The
SDK MUST:

- Build exactly one `OkHttpClient` per `{{ClientName}}` instance
- Hand the same instance to every module's service
- Expose a `close()` method on the facade that shuts down the dispatcher and evicts the
  pool

> **Anti-pattern:** Creating a new `OkHttpClient.Builder().build()` per service or per
> request. This leaks file descriptors and hammers the remote API with new connections.

---

## Interceptor Chain

Every request flows through this ordered chain:

```
[application code]
   │
   ▼
[CorrelationIdInterceptor]  ← adds X-Correlation-Id if absent
   │
   ▼
[AuthInterceptor]           ← injects Authorization or X-API-Key (conditional)
   │
   ▼
[RetryInterceptor]          ← strategic retry on 502/503/504 + IOException
   │
   ▼
[LoggingInterceptor]        ← redacted req/res logging via SLF4J (conditional)
   │
   ▼
[OkHttp dispatcher → network]
```

The order matters:

1. **Correlation ID first** — so the same ID propagates through retries and is logged.
2. **Auth before retry** — if a refresh happens, retry sees the new token.
3. **Retry before logging** — so logging sees the final attempt's wire data, not every
   attempt.

---

## Interceptor Implementations

### CorrelationIdInterceptor

```java
package {{BASE_PACKAGE}}.http;

import okhttp3.Interceptor;
import okhttp3.Request;
import okhttp3.Response;

import java.io.IOException;
import java.util.UUID;

final class CorrelationIdInterceptor implements Interceptor {

    static final String HEADER = "X-Correlation-Id";

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request original = chain.request();
        if (original.header(HEADER) != null) {
            return chain.proceed(original);
        }
        Request withId = original.newBuilder()
            .header(HEADER, UUID.randomUUID().toString())
            .build();
        return chain.proceed(withId);
    }
}
```

### AuthInterceptor (Bearer)

```java
package {{BASE_PACKAGE}}.auth;

import okhttp3.Interceptor;
import okhttp3.Request;
import okhttp3.Response;

import java.io.IOException;

public final class AuthInterceptor implements Interceptor {

    private final BearerCredentialProvider provider;

    public AuthInterceptor(BearerCredentialProvider provider) {
        this.provider = provider;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = applyToken(chain.request());
        Response response = chain.proceed(request);
        if (response.code() != 401) {
            return response;
        }
        // 401 → refresh once, replay once.
        response.close();
        provider.refresh();
        Request retry = applyToken(chain.request());
        return chain.proceed(retry);
    }

    private Request applyToken(Request request) {
        String token = provider.token();
        if (token == null || token.isEmpty()) return request;
        return request.newBuilder()
            .header("Authorization", "Bearer " + token)
            .build();
    }
}
```

### AuthInterceptor (API Key)

```java
public final class ApiKeyInterceptor implements Interceptor {

    private final String headerName;
    private final String value;

    public ApiKeyInterceptor(String headerName, String value) {
        this.headerName = headerName;
        this.value      = value;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request req = chain.request().newBuilder()
            .header(headerName, value)
            .build();
        return chain.proceed(req);
    }
}
```

### AuthInterceptor (HTTP Basic)

```java
import okhttp3.Credentials;

public final class BasicAuthInterceptor implements Interceptor {

    private final String credential;

    public BasicAuthInterceptor(String username, String password) {
        this.credential = Credentials.basic(username, password);
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request req = chain.request().newBuilder()
            .header("Authorization", credential)
            .build();
        return chain.proceed(req);
    }
}
```

### mTLS — `SSLSocketFactory` Configuration

```java
package {{BASE_PACKAGE}}.auth;

import okhttp3.OkHttpClient;

import javax.net.ssl.KeyManagerFactory;
import javax.net.ssl.SSLContext;
import javax.net.ssl.TrustManagerFactory;
import javax.net.ssl.X509TrustManager;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyStore;

public final class MTlsConfigurer {

    private MTlsConfigurer() {}

    public static OkHttpClient.Builder configure(
            OkHttpClient.Builder builder,
            Path keystorePath, char[] keystorePassword,
            Path truststorePath, char[] truststorePassword) {

        try (InputStream ksIn = Files.newInputStream(keystorePath);
             InputStream tsIn = Files.newInputStream(truststorePath)) {

            KeyStore keyStore = KeyStore.getInstance("PKCS12");
            keyStore.load(ksIn, keystorePassword);
            KeyManagerFactory kmf = KeyManagerFactory.getInstance(
                KeyManagerFactory.getDefaultAlgorithm());
            kmf.init(keyStore, keystorePassword);

            KeyStore trustStore = KeyStore.getInstance("PKCS12");
            trustStore.load(tsIn, truststorePassword);
            TrustManagerFactory tmf = TrustManagerFactory.getInstance(
                TrustManagerFactory.getDefaultAlgorithm());
            tmf.init(trustStore);

            X509TrustManager tm = (X509TrustManager) tmf.getTrustManagers()[0];

            SSLContext ctx = SSLContext.getInstance("TLS");
            ctx.init(kmf.getKeyManagers(), new javax.net.ssl.TrustManager[]{tm}, null);

            return builder.sslSocketFactory(ctx.getSocketFactory(), tm);
        } catch (Exception e) {
            throw new IllegalStateException("mTLS configuration failed", e);
        }
    }
}
```

### RetryInterceptor

```java
package {{BASE_PACKAGE}}.http;

import okhttp3.Interceptor;
import okhttp3.Request;
import okhttp3.Response;

import java.io.IOException;

final class RetryInterceptor implements Interceptor {

    private final RetryStrategy strategy;

    RetryInterceptor(RetryStrategy strategy) {
        this.strategy = strategy;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        int attempt = 0;
        IOException lastError = null;

        while (true) {
            Response response = null;
            IOException error = null;
            try {
                response = chain.proceed(request);
            } catch (IOException e) {
                error = e;
            }

            RetryStrategy.Decision decision = strategy.evaluate(attempt, response, error);
            if (!decision.retry) {
                if (error != null) throw error;
                return response;
            }

            if (response != null) response.close();
            lastError = error;
            attempt++;

            try {
                Thread.sleep(decision.delayMillis);
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
                if (lastError != null) throw new IOException("Interrupted while retrying", lastError);
                throw new IOException("Interrupted while retrying");
            }
        }
    }
}
```

### LoggingInterceptor (SLF4J, with redaction)

```java
package {{BASE_PACKAGE}}.http;

import okhttp3.Interceptor;
import okhttp3.Request;
import okhttp3.Response;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;

final class LoggingInterceptor implements Interceptor {

    private static final Logger log = LoggerFactory.getLogger("{{BASE_PACKAGE}}.http");

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request req = chain.request();
        if (log.isDebugEnabled()) {
            log.debug("--> {} {} (auth={})",
                req.method(), req.url(),
                req.header("Authorization") == null ? "none" : "Bearer ***");
        }
        long t0 = System.nanoTime();
        Response resp = chain.proceed(req);
        if (log.isDebugEnabled()) {
            long ms = (System.nanoTime() - t0) / 1_000_000L;
            log.debug("<-- {} {} ({} ms)", resp.code(), req.url(), ms);
        }
        return resp;
    }
}
```

> The `LoggingInterceptor` is registered ONLY when SLF4J is on the consumer's
> classpath (see `Slf4jPresence` in `references/spec-template.md`). The interceptor
> never logs the request or response BODY — bodies often contain credentials or PII
> and OkHttp's `consume`-once contract makes safe body logging non-trivial.

---

## Per-Service Call Pattern

Every service method follows this template:

```java
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
            throw new OrdersException.NotFound(404, body(resp), "GET", url.toString());
        }
        if (!resp.isSuccessful()) {
            throw new OrdersException(resp.code(), body(resp), "GET", url.toString());
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
```

Notes:

- `try (Response resp = ...)` is mandatory — OkHttp `Response` holds connection
  resources that MUST be closed.
- `resp.body().string()` consumes the body and closes the stream — call it once.
- Path segments use `addPathSegments("orders/" + id)` so `id` is URL-encoded by
  OkHttp. NEVER concatenate raw IDs into the URL string.
- Query parameters use `addQueryParameter("status", status.wireValue())`.

---

## Async Surface (CompletableFuture)

If Async API = yes, every sync method has a `*Async` variant:

```java
public CompletableFuture<OrderDto> getByIdAsync(String id) {
    Request req = /* same builder as the sync version */;
    CompletableFuture<OrderDto> future = new CompletableFuture<>();
    http.newCall(req).enqueue(new Callback() {
        @Override
        public void onFailure(Call call, IOException e) {
            future.completeExceptionally(new SdkClientException("GET " + req.url() + " failed", e));
        }
        @Override
        public void onResponse(Call call, Response resp) {
            try (Response r = resp) {
                if (!r.isSuccessful()) {
                    future.completeExceptionally(new OrdersException(r.code(), body(r), "GET", req.url().toString()));
                    return;
                }
                @SuppressWarnings("unchecked")
                Map<String, Object> map = (Map<String, Object>) Json.parse(body(r));
                future.complete(OrderDto.fromJsonMap(map));
            } catch (Exception e) {
                future.completeExceptionally(e);
            }
        }
    });
    return future;
}
```

The sync wrapper is a one-liner over the async one when Async = all:

```java
public OrderDto getById(String id) {
    try {
        return getByIdAsync(id).join();
    } catch (CompletionException ce) {
        Throwable cause = ce.getCause();
        if (cause instanceof RuntimeException) throw (RuntimeException) cause;
        throw new SdkClientException("getById failed", cause);
    }
}
```

OkHttp's dispatcher already manages a thread pool — the SDK does NOT spin up its own.

---

## WebSocket [CONDITIONAL]

OkHttp on JDK 8 supports WebSocket via `OkHttpClient#newWebSocket`:

```java
public Subscription subscribe(String topic, EventListener listener) {
    Request req = new Request.Builder()
        .url(config.baseUrl() + "/ws")
        .header("Authorization", "Bearer " + token())
        .build();
    WebSocket ws = http.newWebSocket(req, new WebSocketListener() {
        @Override public void onMessage(WebSocket s, String text) { listener.onEvent(text); }
        @Override public void onFailure(WebSocket s, Throwable t, Response r) { listener.onError(t); }
        @Override public void onClosed(WebSocket s, int code, String reason) { listener.onClosed(code, reason); }
    });
    return () -> ws.close(1000, "client closed");
}
```

The public `Subscription` interface is a single-method functional interface:

```java
public interface Subscription extends AutoCloseable {
    @Override void close();
}
```

The JDK 11 overlay MAY substitute `java.net.http.HttpClient.newWebSocketBuilder()`
for performance — but the public surface is identical, so consumers see no difference.

---

## Server-Sent Events (SSE) [CONDITIONAL]

Requires `okhttp-sse` (also a Square artifact, same group). If SSE is selected, add:

```xml
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp-sse</artifactId>
    <version>${okhttp.version}</version>
</dependency>
```

…and relocate `okhttp3.sse` along with `okhttp3` in the Shade plugin (it's the same
package prefix, so no extra rule is needed).

```java
EventSources.createFactory(http).newEventSource(req, listener);
```

---

## Configuration Knobs Surface

The `Config` class exposes (immutable, all `Duration`-typed):

- `baseUrl` — required
- `connectTimeout`, `readTimeout`, `writeTimeout` — default 30s each
- `callTimeout` — default 0 (off; use the per-stage timeouts)
- `maxIdleConnections` — default 10
- `keepAliveDuration` — default 5 min
- `retryStrategy` — defaults to `ExponentialBackoffRetryStrategy.defaults()`
- `credentialProvider` — null if Auth = none

Every knob is a Builder method on `{{ClientName}}.Builder`. Validation
(e.g., `baseUrl != null`) happens in `build()`.
