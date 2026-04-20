# Spring Modulith Patterns — Module Structure & Event Architecture

This reference describes the Spring Modulith conventions used in the spec. Include this
content in Sections 5 and 14 of the generated specification.

---

## Module Boundary Rules

Spring Modulith enforces architectural boundaries by treating each top-level package
under the base package as an independent application module. This spec uses the strict
encapsulation model: public API at the module root, all implementation in `internal/`.

### What constitutes a module

```
com.example.app/
├── config/         <- NOT a module (infrastructure config)
├── shared/         <- NOT a module (shared kernel, accessible by all)
├── order/          <- MODULE: order bounded context
├── inventory/      <- MODULE: inventory bounded context
├── scheduling/     <- MODULE: scheduling bounded context
└── Application.java
```

Spring Modulith scans `Application.java`'s package and treats each direct sub-package
as a module, EXCEPT for packages explicitly designated as shared infrastructure.

### Visibility rules — Public API vs Internal Implementation

Each module has a strict two-layer structure:

```
com.example.app.order/                         <- PUBLIC API (visible to other modules)
├── OrderService.java                          <- Service interface
├── OrderDTO.java                              <- Return type for inter-module communication
├── CreateOrderRequest.java                    <- Request DTO (with Bean Validation)
├── UpdateOrderRequest.java                    <- Request DTO (with Bean Validation)
├── OrderException.java                        <- Module-specific exception
├── OrderCreatedEvent.java                     <- Domain event (visible to listeners)
│
└── internal/                                  <- INTERNAL (hidden from other modules)
    ├── OrderServiceImpl.java                  <- Implements OrderService
    ├── OrderRepository.java                   <- Spring Data repository — data access
    ├── OrderEntity.java                       <- JPA entity or MongoDB document
    ├── OrderMapper.java                       <- MapStruct mapper
    └── OrderController.java                   <- REST controller (inbound adapter)
```

**Public API (module root)** — only these types are visible to other modules:
- `OrderService` — Interface. The sole programmatic entry point for other modules.
- `OrderDTO` — The response data contract (Java record).
- `CreateOrderRequest` / `UpdateOrderRequest` — Request DTOs (Java records with validation).
- `OrderException` — Callers can catch module-specific failures.
- `OrderCreatedEvent` — Domain events must be in the root package so listeners in
  other modules can reference the event type.

**Internal implementation (`internal/`)** — invisible to other modules:
- `OrderServiceImpl` — Contains all business logic, repository calls, event publishing.
- `OrderRepository` — Data access is an implementation detail.
- `OrderEntity` — The entity/document model is internal.
- `OrderMapper` — DTO <-> Entity conversion is an internal concern.
- `OrderController` — The REST controller is an **inbound adapter**. It accepts
  HTTP requests and delegates to the service interface. It is NOT part of the
  inter-module API.
- `OrderEventConsumer` *(if Messaging = yes)* — The `@RabbitListener` is also an
  **inbound adapter** (MQ instead of HTTP) and follows the same rule: it lives in
  `internal/`, is package-private, and delegates to the module's service interface.
  It must NEVER be placed under `shared/messaging/` — see `messaging-patterns.md`
  for the full convention.

### Why the controller is internal

This is a deliberate architectural choice. In hexagonal / ports-and-adapters terms:
- The **port** is `OrderService` (the interface) — this is the module's public API
- The **adapter** is `OrderController` — it adapts HTTP to the service interface

Other modules interact through the port (service interface), not the adapter (controller).

### Inter-module dependency direction

```
OrderController (internal) -> OrderService (public interface)
                                    ^
InventoryService (another module) -> OrderService (public interface)
```

### What modules must NOT do

- Never inject a ServiceImpl from another module — use the Service interface
- Never reference another module's `internal` package (enforced by Spring Modulith)
- Never reference another module's Repository or Entity/Document
- Never create circular dependencies between modules
- Never bypass the service interface by calling another module's controller

---

## Inter-Module Communication via Events

When one module needs to notify others about a domain event, use Spring Modulith
application events.

### Event Class Convention

Events are Java records, immutable, and carry only the data the listener needs.

Naming: `<AggregateRoot><PastTenseVerb>Event`

```java
// In com.example.app.order (module root — visible to other modules)
public record OrderCreatedEvent(
    String orderId,
    String customerId,
    Instant occurredAt
) {}
```

### Publishing Events

```java
// In com.example.app.order.internal (INTERNAL)
@Service
@RequiredArgsConstructor
@Slf4j
class OrderServiceImpl implements OrderService {
    private final OrderRepository repository;
    private final OrderMapper mapper;
    private final ApplicationEventPublisher eventPublisher;

    @Override
    @Transactional
    public OrderDTO create(CreateOrderRequest request) {
        var entity = mapper.toEntity(request);
        var saved = repository.save(entity);

        eventPublisher.publishEvent(new OrderCreatedEvent(
            saved.getId(),
            saved.getCustomerId(),
            Instant.now()
        ));

        log.info("Order created: {}", saved.getId());
        return mapper.toDTO(saved);
    }
}
```

### Listening to Events

```java
// In com.example.app.inventory.internal (INTERNAL)
@Service
@RequiredArgsConstructor
@Slf4j
class InventoryServiceImpl implements InventoryService {

    @ApplicationModuleListener
    void on(OrderCreatedEvent event) {
        log.info("Adjusting inventory for order: {}", event.orderId());
        // Business logic to reserve inventory
    }
}
```

`@ApplicationModuleListener` includes `@Async` and `@TransactionalEventListener`:
- The listener runs in a separate transaction
- It executes after the publishing transaction commits
- Failures in the listener do not roll back the publisher

### Event Publication Registry

Spring Modulith's event publication registry persists events. If a listener fails, the
event is marked as incomplete and can be retried.

```yaml
spring:
  modulith:
    events:
      republish-outstanding-events-on-restart: true
```

---

## Module Verification

```java
@Test
void verifyModularity() {
    ApplicationModules.of(Application.class).verify();
}
```

This test catches:
- Direct dependencies between modules that bypass events
- Circular module dependencies
- References to another module's internal types

---

## REST API Module Structure

For REST API applications, the module structure uses a `@RestController` in `internal/`
instead of `@Controller` with page/fragment subpackages.

### REST Module Structure

```
com.example.app.order/                         <- PUBLIC API
├── OrderService.java                          <- Service interface
├── OrderDTO.java                              <- Response DTO (Java record)
├── CreateOrderRequest.java                    <- Request DTO (Java record + validation)
├── UpdateOrderRequest.java                    <- Request DTO (Java record + validation)
├── OrderException.java                        <- Module-specific exception
│
└── internal/                                  <- INTERNAL
    ├── OrderServiceImpl.java                  <- Implements OrderService
    ├── OrderRepository.java                   <- Spring Data repository
    ├── OrderEntity.java                       <- JPA entity / MongoDB document
    ├── OrderMapper.java                       <- MapStruct mapper
    └── OrderController.java                   <- @RestController (JSON in/out)
```

### REST Controller Pattern

```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Tag(name = "Orders", description = "Order management operations")
class OrderController {

    private final OrderService service;

    @GetMapping
    public ResponseEntity<Page<OrderDTO>> list(
            @PageableDefault(size = 20) Pageable pageable,
            @RequestParam(required = false) String search) {
        return ResponseEntity.ok(service.findAll(search, pageable));
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderDTO> getById(@PathVariable String id) {
        return ResponseEntity.ok(service.findById(id));
    }

    @PostMapping
    public ResponseEntity<OrderDTO> create(
            @Valid @RequestBody CreateOrderRequest request) {
        OrderDTO created = service.create(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}").buildAndExpand(created.id()).toUri();
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")
    public ResponseEntity<OrderDTO> update(
            @PathVariable String id,
            @Valid @RequestBody UpdateOrderRequest request) {
        return ResponseEntity.ok(service.update(id, request));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable String id) {
        service.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Key Differences from Web (JTE) Modules

1. **`@RestController` instead of `@Controller`**: Returns JSON directly, not view names.
2. **No view models**: Response DTOs serve as the API contract. No JTE templates.
3. **No page/fragment subpackages**: Single `OrderController` handles all CRUD operations.
4. **`ResponseEntity` wrapping**: All responses wrapped in `ResponseEntity` for explicit
   status code and header control.
5. **`@Valid @RequestBody`**: Bean Validation at the controller boundary.
6. **`Pageable` parameter**: Spring Data `Pageable` injected directly from query params.
7. **OpenAPI annotations**: `@Tag`, `@Operation`, `@ApiResponse` for API documentation.
