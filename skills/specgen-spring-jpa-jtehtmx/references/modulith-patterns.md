# Spring Modulith Patterns — Module Structure & Event Architecture

This reference describes the Spring Modulith conventions used in the spec. Include this
content in Sections 4 and 8 of the generated specification.

---

## Module Boundary Rules

Spring Modulith enforces architectural boundaries by treating each top-level package
under the base package as an independent application module. This spec uses the strict
encapsulation model: public API at the module root, all implementation in `internal/`.

### What constitutes a module

```
com.example.app/
├── config/         ← NOT a module (infrastructure config)
├── shared/         ← NOT a module (shared kernel, accessible by all)
├── order/          ← MODULE: order bounded context
├── inventory/      ← MODULE: inventory bounded context
├── scheduling/     ← MODULE: scheduling bounded context
└── Application.java
```

Spring Modulith scans `Application.java`'s package and treats each direct sub-package
as a module, EXCEPT for packages explicitly designated as shared infrastructure.

### Visibility rules — Public API vs Internal Implementation

Each module has a strict two-layer structure:

```
com.example.app.order/                         ← PUBLIC API (visible to other modules)
├── OrderService.java                          ← Service interface
├── OrderDTO.java                              ← Return type for inter-module communication
├── OrderException.java                        ← Module-specific exception
├── OrderCreatedEvent.java                     ← Domain event (visible to listeners)
│
└── internal/                                  ← INTERNAL (hidden from other modules)
    ├── OrderServiceImpl.java                  ← Implements OrderService
    ├── OrderRepository.java                   ← MongoRepository — data access
    ├── OrderDocument.java                     ← MongoDB document — data model
    ├── OrderMapper.java                       ← MapStruct mapper
    └── web/
        └── OrderController.java               ← REST controller (inbound adapter)
```

**Public API (module root)** — only these types are visible to other modules:
- `OrderService` — Interface. The sole programmatic entry point for other modules.
  Other modules inject this interface; Spring wires the internal `OrderServiceImpl`.
- `OrderDTO` — The data contract. When `OrderService` returns data, it returns this
  DTO, never the internal `OrderDocument`. This prevents leaking persistence details.
- `OrderException` — Callers can catch module-specific failures.
- `OrderCreatedEvent` — Domain events must be in the root package so listeners in
  other modules can reference the event type.

**Internal implementation (`internal/`)** — invisible to other modules:
- `OrderServiceImpl` — Contains all business logic, repository calls, event publishing.
  Spring Modulith treats everything in `internal/` as module-private.
- `OrderRepository` — Data access is an implementation detail. Other modules never
  query another module's database directly.
- `OrderDocument` — The MongoDB document model is internal. It may have fields,
  embedded objects, or annotations that are irrelevant to consumers.
- `OrderMapper` — DTO ↔ Document conversion is an internal concern.
- `web/OrderController` — The REST controller is an **inbound adapter**. It accepts
  HTTP requests and delegates to the service interface. It is NOT part of the
  inter-module API — other modules never call controllers. Placing it in `internal/web/`
  makes this architectural intent explicit and enforceable.
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
If tomorrow you add a gRPC adapter or a message consumer, they would also be internal
adapters delegating to the same public service interface. Keeping adapters internal
enforces this separation.

### Inter-module dependency direction

```
OrderController (internal) → OrderService (public interface)
                                    ↑
InventoryService (another module) → OrderService (public interface)
```

Both the internal controller and external modules depend on the same public interface.
The `OrderServiceImpl` is wired by Spring but never directly referenced.

### What modules must NOT do

- Never inject a ServiceImpl from another module — use the Service interface
- Never reference another module's `internal` package (enforced by Spring Modulith)
- Never reference another module's Repository or Document
- Never create circular dependencies between modules
- Never bypass the service interface by calling another module's controller

---

## Inter-Module Communication via Events

When one module needs to notify others about a domain event (e.g., an order was created
and inventory needs to adjust), use Spring Modulith application events.

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

Why records: they're immutable, generate equals/hashCode, and make the event's contract
explicit. Using `Instant occurredAt` establishes when the event happened.

### Publishing Events

The service implementation (internal) publishes events using Spring's `ApplicationEventPublisher`:

```java
// In com.example.app.order.internal (INTERNAL — hidden from other modules)
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
        OrderDocument saved = repository.save(mapper.toDocument(request));

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

Note that the class has package-private visibility (no `public` keyword). Spring still
discovers it via component scanning, but other modules cannot reference it directly.
They depend on the `OrderService` interface instead.

Events are published within the transaction boundary. If the transaction rolls back,
the event is not delivered.

### Listening to Events

Other modules listen using `@ApplicationModuleListener` in their internal implementation:

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

The listener references `OrderCreatedEvent` which is in the `order` module's root
package (public API), so this cross-module reference is valid. The implementation
class itself is internal to the `inventory` module.

`@ApplicationModuleListener` is a composed annotation that includes `@Async` and
`@TransactionalEventListener`. This means:
- The listener runs in a separate transaction
- It executes after the publishing transaction commits
- Failures in the listener do not roll back the publisher

### Event Publication Registry

Spring Modulith's event publication registry (backed by MongoDB via
`spring-modulith-starter-mongodb`) persists events. If a listener fails, the event
is marked as incomplete and can be retried.

Configuration in `application.yml`:
```yaml
spring:
  modulith:
    events:
      republish-outstanding-events-on-restart: true
```

This ensures at-least-once delivery semantics for application events.

---

## Module Verification

Spring Modulith provides a test that verifies all module boundary rules are respected:

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

Include this test in the Testing Strategy section of the spec. It should run as part
of the build to enforce architectural integrity continuously.

---

## Module Documentation

Spring Modulith can generate architecture documentation from the module structure:

```java
@Test
void createModuleDocumentation() {
    ApplicationModules modules = ApplicationModules.of(Application.class);
    new Documenter(modules).writeDocumentation();
}
```

This produces PlantUML diagrams and Asciidoc documentation of module interactions.
Mention this as an optional capability in the spec.

---

## Web Application Module Adaptations

For server-rendered web applications, the module structure is adapted to include `page`
and `fragment` subpackages within `internal/`. The controller is a `@Controller` (not
`@RestController`) that returns JTE view names.

### Web Module Structure

```
com.example.app.order/                         ← PUBLIC API (visible to other modules)
├── OrderService.java                          ← Service interface
├── OrderDTO.java                              ← Return type for inter-module communication
├── OrderException.java                        ← Module-specific exception
│
└── internal/                                  ← INTERNAL (hidden from other modules)
    ├── OrderServiceImpl.java                  ← Implements OrderService
    ├── OrderRepository.java                   ← MongoRepository — data access
    ├── OrderDocument.java                     ← MongoDB document — data model
    ├── OrderMapper.java                       ← MapStruct mapper
    ├── page/                                  ← Page controllers (full HTML pages)
    │   ├── OrderPageController.java           ← @Controller returning JTE view names
    │   ├── OrderListView.java                 ← View model for list page
    │   └── OrderDetailView.java               ← View model for detail page
    └── fragment/                              ← Fragment controllers (htmx partials)
        └── OrderFragmentController.java       ← Returns HTML fragments for htmx
```

### Key Differences from API Modules

1. **No REST controllers**: Web modules use `@Controller` instead of `@RestController`.
   Controllers return JTE template paths, not JSON.

2. **Direct data ownership**: Each module owns its MongoDB documents and repositories.
   The service layer queries MongoDB directly via Spring Data `MongoRepository`.

3. **View models**: Each page has a dedicated view model class that encapsulates all
   data needed by the JTE template. View models are internal to the module.

4. **Page vs Fragment separation**: `page/` subpackage holds controllers for full page
   renders; `fragment/` holds controllers that return partial HTML for htmx requests.

5. **Mixins**: Controllers implement `PaginationAware`, `ThemeAware`, and `AuthAware`
   interfaces to extract pagination, theme, and user info from the request.

### Web Page Controller Pattern

```java
@Controller
@RequestMapping("/orders")
@RequiredArgsConstructor
class OrderPageController implements PaginationAware, ThemeAware, AuthAware {

    private final OrderService service;
    private final MainLayout layout;

    @GetMapping
    String list(Model model, HttpServletRequest req) {
        var pageRequest = pagination(req, 20, 100);
        var result = service.list(pageRequest);
        var paginationVm = result.toPagination("/orders");
        var view = new OrderListView(result.data(), paginationVm, theme(req), currentUser(req));
        model.addAttribute("layout", layout.defaultPage("Orders", req));
        model.addAttribute("view", view);
        return "com/example/app/order/page/OrderListPage";
    }

    @GetMapping("/{id}")
    String detail(@PathVariable String id, Model model, HttpServletRequest req) {
        var dto = service.get(id);
        var view = new OrderDetailView(dto, theme(req), currentUser(req));
        model.addAttribute("layout", layout.defaultPage(dto.getFieldOne(), req));
        model.addAttribute("view", view);
        return "com/example/app/order/page/OrderDetailPage";
    }
}
```

### Web Fragment Controller Pattern

```java
@Controller
@RequestMapping("/orders/fragments")
@RequiredArgsConstructor
class OrderFragmentController implements PaginationAware, AuthAware {

    private final OrderService service;

    @GetMapping("/row/{id}")
    String row(@PathVariable String id, Model model) {
        var dto = service.get(id);
        model.addAttribute("item", dto);
        return "com/example/app/order/fragment/OrderRowFragment";
    }

    @DeleteMapping("/{id}")
    ResponseEntity<String> delete(@PathVariable String id) {
        service.delete(id);
        return ResponseEntity.ok("");
    }
}
```
