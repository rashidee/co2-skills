# REST API Patterns — Controllers, DTOs, and Conventions

This reference describes the REST API-specific patterns for the Spring Boot application.
Include this content in Sections 7-9 of the generated specification.

---

## Architecture Overview

The REST API uses a layered composition model:

```
Request → SecurityFilter → CorrelationIdFilter → @RestController
    → Service (queries DB via Repository)
    → Response DTO (mapped via MapStruct)
    → JSON Response (serialized by Jackson)
```

---

## REST Controller Pattern

Every REST controller follows this structure:

```java
package {{BASE_PACKAGE}}.{{module}}.internal;

import {{BASE_PACKAGE}}.{{module}}.{{ModuleName}}Service;
import {{BASE_PACKAGE}}.{{module}}.{{ModuleName}}DTO;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.support.ServletUriComponentsBuilder;
import java.net.URI;

@RestController
@RequestMapping("/api/v1/{{resource-plural}}")
@RequiredArgsConstructor
@Tag(name = "{{ModuleName}}", description = "{{ModuleName}} management operations")
class {{ModuleName}}Controller {

    private final {{ModuleName}}Service service;

    @GetMapping
    @Operation(summary = "List all {{resource-plural}} with pagination")
    public ResponseEntity<Page<{{ModuleName}}DTO>> list(
            @PageableDefault(size = 20, sort = "createdAt") Pageable pageable,
            @RequestParam(required = false) String search) {
        Page<{{ModuleName}}DTO> results = service.findAll(search, pageable);
        return ResponseEntity.ok(results);
    }

    @GetMapping("/{id}")
    @Operation(summary = "Get {{resource}} by ID")
    @ApiResponse(responseCode = "200", description = "{{ModuleName}} found")
    @ApiResponse(responseCode = "404", description = "{{ModuleName}} not found")
    public ResponseEntity<{{ModuleName}}DTO> getById(@PathVariable String id) {
        {{ModuleName}}DTO result = service.findById(id);
        return ResponseEntity.ok(result);
    }

    @PostMapping
    @Operation(summary = "Create a new {{resource}}")
    @ApiResponse(responseCode = "201", description = "{{ModuleName}} created")
    @ApiResponse(responseCode = "400", description = "Validation failed")
    public ResponseEntity<{{ModuleName}}DTO> create(
            @Valid @RequestBody Create{{ModuleName}}Request request) {
        {{ModuleName}}DTO created = service.create(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(created.id())
            .toUri();
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")
    @Operation(summary = "Update an existing {{resource}}")
    @ApiResponse(responseCode = "200", description = "{{ModuleName}} updated")
    @ApiResponse(responseCode = "404", description = "{{ModuleName}} not found")
    public ResponseEntity<{{ModuleName}}DTO> update(
            @PathVariable String id,
            @Valid @RequestBody Update{{ModuleName}}Request request) {
        {{ModuleName}}DTO updated = service.update(id, request);
        return ResponseEntity.ok(updated);
    }

    @DeleteMapping("/{id}")
    @Operation(summary = "Delete a {{resource}}")
    @ApiResponse(responseCode = "204", description = "{{ModuleName}} deleted")
    @ApiResponse(responseCode = "404", description = "{{ModuleName}} not found")
    public ResponseEntity<Void> delete(@PathVariable String id) {
        service.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

---

## Request/Response DTO Pattern

### Response DTO (Java Record)

Response DTOs are immutable Java records that represent the API contract for responses:

```java
package {{BASE_PACKAGE}}.{{module}};

import java.time.Instant;

public record {{ModuleName}}DTO(
    String id,
    String name,
    String code,
    String description,
    String status,
    Instant createdAt,
    Instant updatedAt,
    String createdBy,
    String updatedBy
) {}
```

### Request DTO with Bean Validation

Request DTOs use Java records with Bean Validation annotations:

```java
package {{BASE_PACKAGE}}.{{module}};

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;
import jakarta.validation.constraints.Size;

public record Create{{ModuleName}}Request(
    @NotBlank(message = "Name is required")
    @Size(max = 255, message = "Name must not exceed 255 characters")
    String name,

    @NotBlank(message = "Code is required")
    @Size(max = 50, message = "Code must not exceed 50 characters")
    String code,

    @Size(max = 1000, message = "Description must not exceed 1000 characters")
    String description
) {}
```

```java
package {{BASE_PACKAGE}}.{{module}};

import jakarta.validation.constraints.Size;

public record Update{{ModuleName}}Request(
    @Size(max = 255, message = "Name must not exceed 255 characters")
    String name,

    @Size(max = 1000, message = "Description must not exceed 1000 characters")
    String description
) {}
```

---

## Standardized Error Response

### Error Envelope

All error responses use a consistent JSON structure:

```java
package {{BASE_PACKAGE}}.shared.exception;

import java.time.Instant;
import java.util.List;

public record ApiError(
    Instant timestamp,
    int status,
    String error,
    String message,
    String path,
    List<FieldError> fieldErrors
) {
    public record FieldError(String field, String message) {}

    public static ApiError of(int status, String error, String message, String path) {
        return new ApiError(Instant.now(), status, error, message, path, null);
    }

    public static ApiError withFieldErrors(int status, String error, String message,
            String path, List<FieldError> fieldErrors) {
        return new ApiError(Instant.now(), status, error, message, path, fieldErrors);
    }
}
```

### Example Error Responses

**404 Not Found:**
```json
{
  "timestamp": "2026-03-13T10:15:30Z",
  "status": 404,
  "error": "Not Found",
  "message": "Job Demand not found with id: 12345",
  "path": "/api/v1/job-demands/12345",
  "fieldErrors": null
}
```

**400 Validation Error:**
```json
{
  "timestamp": "2026-03-13T10:15:30Z",
  "status": 400,
  "error": "Validation Failed",
  "message": "Request body contains invalid fields",
  "path": "/api/v1/job-demands",
  "fieldErrors": [
    { "field": "name", "message": "Name is required" },
    { "field": "code", "message": "Code must not exceed 50 characters" }
  ]
}
```

---

## Global Exception Handler

```java
package {{BASE_PACKAGE}}.shared.exception;

import jakarta.servlet.http.HttpServletRequest;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.List;

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(
            ResourceNotFoundException ex, HttpServletRequest request) {
        ApiError error = ApiError.of(404, "Not Found",
            ex.getMessage(), request.getRequestURI());
        return ResponseEntity.status(404).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest request) {
        List<ApiError.FieldError> fieldErrors = ex.getBindingResult()
            .getFieldErrors().stream()
            .map(fe -> new ApiError.FieldError(fe.getField(), fe.getDefaultMessage()))
            .toList();

        ApiError error = ApiError.withFieldErrors(400, "Validation Failed",
            "Request body contains invalid fields",
            request.getRequestURI(), fieldErrors);
        return ResponseEntity.badRequest().body(error);
    }

    @ExceptionHandler(ConflictException.class)
    public ResponseEntity<ApiError> handleConflict(
            ConflictException ex, HttpServletRequest request) {
        ApiError error = ApiError.of(409, "Conflict",
            ex.getMessage(), request.getRequestURI());
        return ResponseEntity.status(409).body(error);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleGeneral(
            Exception ex, HttpServletRequest request) {
        log.error("Unhandled exception at {}", request.getRequestURI(), ex);
        ApiError error = ApiError.of(500, "Internal Server Error",
            "An unexpected error occurred", request.getRequestURI());
        return ResponseEntity.status(500).body(error);
    }
}
```

---

## Base Exception Classes

```java
package {{BASE_PACKAGE}}.shared.exception;

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }

    public ResourceNotFoundException(String resourceName, String fieldName, Object fieldValue) {
        super(String.format("%s not found with %s: %s", resourceName, fieldName, fieldValue));
    }
}
```

```java
package {{BASE_PACKAGE}}.shared.exception;

public class ConflictException extends RuntimeException {
    public ConflictException(String message) {
        super(message);
    }
}
```

---

## Pagination Response Structure

Spring Data `Page` is serialized by Jackson automatically. The response structure:

```json
{
  "content": [
    { "id": "1", "name": "Widget A", "code": "WA01" },
    { "id": "2", "name": "Widget B", "code": "WB01" }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 20,
    "sort": { "sorted": true, "unsorted": false, "empty": false },
    "offset": 0,
    "paged": true,
    "unpaged": false
  },
  "totalElements": 142,
  "totalPages": 8,
  "last": false,
  "first": true,
  "size": 20,
  "number": 0,
  "sort": { "sorted": true, "unsorted": false, "empty": false },
  "numberOfElements": 20,
  "empty": false
}
```

### Controller Pagination Pattern

```java
@GetMapping
public ResponseEntity<Page<{{ModuleName}}DTO>> list(
        @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC)
        Pageable pageable,
        @RequestParam(required = false) String search) {
    Page<{{ModuleName}}DTO> results = service.findAll(search, pageable);
    return ResponseEntity.ok(results);
}
```

### Custom Page Serialization (Optional)

If you need a simplified page response:

```java
package {{BASE_PACKAGE}}.shared.dto;

import org.springframework.data.domain.Page;
import java.util.List;

public record PageResponse<T>(
    List<T> content,
    int pageNumber,
    int pageSize,
    long totalElements,
    int totalPages,
    boolean first,
    boolean last
) {
    public static <T> PageResponse<T> from(Page<T> page) {
        return new PageResponse<>(
            page.getContent(),
            page.getNumber(),
            page.getSize(),
            page.getTotalElements(),
            page.getTotalPages(),
            page.isFirst(),
            page.isLast()
        );
    }
}
```

---

## Correlation ID Filter

```java
package {{BASE_PACKAGE}}.shared.filter;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;
import java.io.IOException;
import java.util.Optional;
import java.util.UUID;

@Component
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final String CORRELATION_HEADER = "X-Correlation-Id";
    private static final String MDC_KEY = "correlationId";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        String correlationId = Optional
            .ofNullable(request.getHeader(CORRELATION_HEADER))
            .orElse(UUID.randomUUID().toString());

        MDC.put(MDC_KEY, correlationId);
        response.setHeader(CORRELATION_HEADER, correlationId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove(MDC_KEY);
        }
    }
}
```

---

## Jackson Configuration

```java
package {{BASE_PACKAGE}}.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        mapper.setDefaultPropertyInclusion(
            com.fasterxml.jackson.annotation.JsonInclude.Include.NON_NULL);
        return mapper;
    }
}
```

---

## CORS Configuration

```java
package {{BASE_PACKAGE}}.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;
import java.util.List;

@Configuration
public class CorsConfig {

    @Value("${app.cors.allowed-origins:http://localhost:3000}")
    private List<String> allowedOrigins;

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(allowedOrigins);
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

---

## OpenAPI / Springdoc Configuration

### Maven Dependency

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.0</version>
</dependency>
```

### Application Configuration

```yaml
springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: method
    tags-sorter: alpha
  default-produces-media-type: application/json
  default-consumes-media-type: application/json
```

### OpenAPI Global Config

```java
package {{BASE_PACKAGE}}.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.servers.Server;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.List;

@Configuration
public class OpenApiConfig {

    @Value("${spring.application.name}")
    private String applicationName;

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title(applicationName + " API")
                .version("v1")
                .description("REST API for " + applicationName)
                .contact(new Contact()
                    .name("Bestinet")
                    .email("support@bestinet.com")))
            .servers(List.of(
                new Server().url("http://localhost:8080").description("Local development")
            ));
    }
}
```

### Controller Annotations

```java
@RestController
@RequestMapping("/api/v1/job-demands")
@Tag(name = "Job Demands", description = "Job demand management operations")
class JobDemandController {

    @Operation(
        summary = "Search job demands",
        description = "Search job demands with pagination and optional filters",
        responses = {
            @ApiResponse(responseCode = "200", description = "Job demands retrieved"),
            @ApiResponse(responseCode = "400", description = "Invalid search parameters")
        }
    )
    @GetMapping
    public ResponseEntity<Page<JobDemandDTO>> search(
            @Parameter(description = "Country ISO3 code") @RequestParam(required = false) String country,
            @Parameter(description = "Status filter") @RequestParam(required = false) String status,
            @PageableDefault(size = 20) Pageable pageable) {
        // ...
    }
}
```

---

## Service Interface Pattern

```java
package {{BASE_PACKAGE}}.{{module}};

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface {{ModuleName}}Service {

    Page<{{ModuleName}}DTO> findAll(String search, Pageable pageable);

    {{ModuleName}}DTO findById(String id);

    {{ModuleName}}DTO create(Create{{ModuleName}}Request request);

    {{ModuleName}}DTO update(String id, Update{{ModuleName}}Request request);

    void delete(String id);
}
```

---

## Service Implementation Pattern

```java
package {{BASE_PACKAGE}}.{{module}}.internal;

import {{BASE_PACKAGE}}.{{module}}.*;
import {{BASE_PACKAGE}}.shared.exception.ResourceNotFoundException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Slf4j
@Transactional(readOnly = true)
class {{ModuleName}}ServiceImpl implements {{ModuleName}}Service {

    private final {{ModuleName}}Repository repository;
    private final {{ModuleName}}Mapper mapper;

    @Override
    public Page<{{ModuleName}}DTO> findAll(String search, Pageable pageable) {
        if (search != null && !search.isBlank()) {
            return repository.findByNameContainingIgnoreCase(search, pageable)
                .map(mapper::toDTO);
        }
        return repository.findAll(pageable).map(mapper::toDTO);
    }

    @Override
    public {{ModuleName}}DTO findById(String id) {
        return repository.findById(id)
            .map(mapper::toDTO)
            .orElseThrow(() -> new ResourceNotFoundException("{{ModuleName}}", "id", id));
    }

    @Override
    @Transactional
    public {{ModuleName}}DTO create(Create{{ModuleName}}Request request) {
        var entity = mapper.toEntity(request);
        var saved = repository.save(entity);
        log.info("Created {{ModuleName}}: {}", saved.getId());
        return mapper.toDTO(saved);
    }

    @Override
    @Transactional
    public {{ModuleName}}DTO update(String id, Update{{ModuleName}}Request request) {
        var entity = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("{{ModuleName}}", "id", id));
        mapper.updateEntity(request, entity);
        var saved = repository.save(entity);
        log.info("Updated {{ModuleName}}: {}", saved.getId());
        return mapper.toDTO(saved);
    }

    @Override
    @Transactional
    public void delete(String id) {
        if (!repository.existsById(id)) {
            throw new ResourceNotFoundException("{{ModuleName}}", "id", id);
        }
        repository.deleteById(id);
        log.info("Deleted {{ModuleName}}: {}", id);
    }
}
```

---

## MapStruct Mapper Pattern

```java
package {{BASE_PACKAGE}}.{{module}}.internal;

import {{BASE_PACKAGE}}.{{module}}.*;
import org.mapstruct.Mapper;
import org.mapstruct.MappingTarget;

@Mapper(componentModel = "spring")
interface {{ModuleName}}Mapper {

    {{ModuleName}}DTO toDTO({{ModuleName}}Entity entity);

    {{ModuleName}}Entity toEntity(Create{{ModuleName}}Request request);

    void updateEntity(Update{{ModuleName}}Request request, @MappingTarget {{ModuleName}}Entity entity);
}
```
