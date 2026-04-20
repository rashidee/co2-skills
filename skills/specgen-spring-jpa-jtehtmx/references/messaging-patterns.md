# Messaging Patterns — RabbitMQ Pub/Sub

This reference describes the standalone RabbitMQ pub/sub messaging layer for inter-system
communication. This is **independent** from Spring Modulith's internal application events
and from Spring Batch remote partitioning — it provides explicit publisher/consumer
services that modules call to communicate with external systems.

---

## Architecture Overview

```
                        ┌─────────────────────────────────────────────┐
                        │              RabbitMQ Broker                │
                        │                                             │
  ┌──────────┐          │  ┌──────────────────┐   ┌───────────────┐  │          ┌──────────────┐
  │           │ publish  │  │  app.events      │   │ app.commands  │  │ consume  │   External   │
  │  Module   │────────►│  │  (TopicExchange)  │   │ (DirectExch.) │  │────────►│   System A   │
  │  Service  │          │  └────────┬─────────┘   └───────┬───────┘  │          └──────────────┘
  └──────────┘          │           │                      │          │
                        │     ┌─────▼──────┐         ┌─────▼──────┐  │
                        │     │ Queue:     │         │ Queue:     │  │
                        │     │ order.*    │         │ inventory  │  │
                        │     └─────┬──────┘         │ .sync      │  │
                        │           │                └─────┬──────┘  │
  ┌──────────┐ consume  │           │                      │         │          ┌──────────────┐
  │   This   │◄─────────│───────────┘                      └─────────│────────►│   External   │
  │   App    │          │                                             │          │   System B   │
  └──────────┘          │  ┌──────────────────┐                      │          └──────────────┘
                        │  │  app.dlx          │                      │
                        │  │  (Dead Letter)    │                      │
                        │  └──────────────────┘                      │
                        └─────────────────────────────────────────────┘
```

**Key concepts:**
- **Topic Exchange** (`app.events.exchange`) — pub/sub event broadcasting with routing key
  patterns (e.g., `order.exported`, `order.cancelled`)
- **Direct Exchange** (`app.commands.exchange`) — point-to-point command delivery to a
  specific queue (e.g., `inventory.sync`)
- **Dead Letter Exchange** (`app.dlx.exchange`) — failed messages routed here after max
  retries for investigation and replay

---

## Maven Dependencies

```xml
<!-- RabbitMQ Messaging (included when Messaging = yes) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

> **Note:** If Remote Partitioning is also selected, `spring-boot-starter-amqp` is already
> present — do not duplicate. The same RabbitMQ broker connection is shared.

---

## Application Configuration

```yaml
# [If Messaging = yes] add to application.yml:
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /

app:
  messaging:
    events-exchange: app.events.exchange
    commands-exchange: app.commands.exchange
    dead-letter-exchange: app.dlx.exchange
```

> **Shared broker config:** If Remote Partitioning is also selected, the `spring.rabbitmq.*`
> connection block is shared — include it only once. The batch partition queues
> (`batch.partition.*`) and messaging queues (`app.*`) use different exchanges and routing
> keys, so they do not conflict.

### application-prod.yml overrides

```yaml
spring:
  rabbitmq:
    host: ${RABBITMQ_HOST}
    port: ${RABBITMQ_PORT:5672}
    username: ${RABBITMQ_USERNAME}
    password: ${RABBITMQ_PASSWORD}
    virtual-host: ${RABBITMQ_VHOST:/}
    connection-timeout: 10000
    requested-heartbeat: 30
    publisher-confirm-type: correlated
    publisher-returns: true
    listener:
      simple:
        prefetch: 10
        concurrency: 2
        max-concurrency: 5
        retry:
          enabled: true
          initial-interval: 1000
          max-attempts: 3
          max-interval: 10000
          multiplier: 2.0
```

---

## Package Structure

Messaging artifacts split across two locations to honor Spring Modulith boundaries:

```
shared/
└── messaging/                            # Shared infrastructure ONLY (no business logic)
    ├── RabbitMQMessagingConfig.java      # Exchange, queue, binding declarations
    ├── RabbitMQPublisher.java            # Generic publisher service
    ├── MessageConverterConfig.java       # Jackson2JsonMessageConverter bean
    ├── OrderExportedEvent.java           # Inter-system event/command DTOs
    └── InventorySyncCommand.java

{{module}}/                               # Module — PUBLIC API
└── internal/                             # INTERNAL — hidden from other modules
    ├── {{Module}}ServiceImpl.java
    ├── ...
    └── {{Module}}EventConsumer.java      # @RabbitListener — MUST live here, NOT in shared/
```

> **Convention — MQ listeners are module-internal.**
> Every `@RabbitListener` (and any other inbound MQ adapter) is a module-specific
> **inbound adapter** — it belongs inside the owning module's `internal/` package, never
> under `shared/messaging/consumer/` or any other shared location. This mirrors how
> page/fragment controllers are placed: inside `internal/`. The shared `messaging/`
> package holds infrastructure beans only (config, publisher, converter, cross-module
> DTOs) — it must contain **no** `@RabbitListener` classes.
>
> Reasoning:
> - A consumer that calls `OrderService` belongs to the `order` module's bounded context.
> - Placing it in `shared/` lets it bypass module boundaries and inject any `*ServiceImpl`,
>   silently breaking Spring Modulith encapsulation and making `ApplicationModules.verify()`
>   results misleading.
> - Module-internal placement keeps the listener, the service it drives, and the entity
>   it persists colocated and independently verifiable.

---

## RabbitMQ Infrastructure Configuration

```java
package {{BASE_PACKAGE}}.shared.messaging;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.FanoutExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.QueueBuilder;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQMessagingConfig {

    @Value("${app.messaging.events-exchange}")
    private String eventsExchangeName;

    @Value("${app.messaging.commands-exchange}")
    private String commandsExchangeName;

    @Value("${app.messaging.dead-letter-exchange}")
    private String dlxExchangeName;

    // ── Exchanges ────────────────────────────────────────────────────────

    @Bean
    TopicExchange eventsExchange() {
        return new TopicExchange(eventsExchangeName, true, false);
    }

    @Bean
    DirectExchange commandsExchange() {
        return new DirectExchange(commandsExchangeName, true, false);
    }

    @Bean
    FanoutExchange deadLetterExchange() {
        return new FanoutExchange(dlxExchangeName, true, false);
    }

    // ── Dead Letter Queue ────────────────────────────────────────────────

    @Bean
    Queue deadLetterQueue() {
        return QueueBuilder.durable("app.dlq").build();
    }

    @Bean
    Binding deadLetterBinding(Queue deadLetterQueue, FanoutExchange deadLetterExchange) {
        return BindingBuilder.bind(deadLetterQueue).to(deadLetterExchange);
    }

    // ── Sample Event Queue (Topic Exchange) ──────────────────────────────
    // Subscribe to all order-related events via routing key pattern "order.*"

    @Bean
    Queue orderEventsQueue() {
        return QueueBuilder.durable("app.events.order")
                .withArgument("x-dead-letter-exchange", dlxExchangeName)
                .build();
    }

    @Bean
    Binding orderEventsBinding(Queue orderEventsQueue, TopicExchange eventsExchange) {
        return BindingBuilder.bind(orderEventsQueue).to(eventsExchange).with("order.*");
    }

    // ── Sample Command Queue (Direct Exchange) ───────────────────────────
    // Point-to-point command delivery for inventory sync

    @Bean
    Queue inventorySyncQueue() {
        return QueueBuilder.durable("app.commands.inventory-sync")
                .withArgument("x-dead-letter-exchange", dlxExchangeName)
                .build();
    }

    @Bean
    Binding inventorySyncBinding(Queue inventorySyncQueue, DirectExchange commandsExchange) {
        return BindingBuilder.bind(inventorySyncQueue).to(commandsExchange).with("inventory.sync");
    }
}
```

---

## Message Converter

```java
package {{BASE_PACKAGE}}.shared.messaging;

import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MessageConverterConfig {

    @Bean
    MessageConverter jackson2JsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

---

## Publisher Service

```java
package {{BASE_PACKAGE}}.shared.messaging;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
@Slf4j
public class RabbitMQPublisher {

    private final RabbitTemplate rabbitTemplate;

    @Value("${app.messaging.events-exchange}")
    private String eventsExchange;

    @Value("${app.messaging.commands-exchange}")
    private String commandsExchange;

    /**
     * Publish a domain event to the topic exchange.
     * Consumers subscribe via routing key patterns (e.g., "order.*").
     *
     * @param routingKey the routing key (e.g., "order.exported", "order.cancelled")
     * @param event      the event payload — serialized to JSON automatically
     */
    public void publishEvent(String routingKey, Object event) {
        log.info("Publishing event [{}]: {}", routingKey, event.getClass().getSimpleName());
        rabbitTemplate.convertAndSend(eventsExchange, routingKey, event);
    }

    /**
     * Send a command to the direct exchange for point-to-point delivery.
     *
     * @param routingKey the routing key matching a specific queue (e.g., "inventory.sync")
     * @param command    the command payload — serialized to JSON automatically
     */
    public void sendCommand(String routingKey, Object command) {
        log.info("Sending command [{}]: {}", routingKey, command.getClass().getSimpleName());
        rabbitTemplate.convertAndSend(commandsExchange, routingKey, command);
    }
}
```

---

## Sample Event / Command DTOs

```java
package {{BASE_PACKAGE}}.shared.messaging;

import java.time.Instant;

/**
 * Sample domain event broadcast to external systems via the topic exchange.
 */
public record OrderExportedEvent(
        String orderId,
        String customerId,
        Instant exportedAt
) {}

/**
 * Sample point-to-point command sent via the direct exchange.
 */
public record InventorySyncCommand(
        String productId,
        int quantity,
        String warehouseCode
) {}
```

---

## Consumer Service (module-internal)

The consumer lives inside the owning module's `internal/` package — **never** in
`shared/messaging/`. Use package-private visibility (no `public` modifier) so Spring
Modulith treats it as internal-only.

```java
package {{BASE_PACKAGE}}.order.internal;

import {{BASE_PACKAGE}}.order.OrderService;
import {{BASE_PACKAGE}}.shared.messaging.OrderExportedEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
class OrderEventConsumer {

    private final OrderService orderService;

    /**
     * Listens to order events from the topic exchange.
     * The queue "app.events.order" is bound with routing key pattern "order.*".
     *
     * Inbound adapter — adapts a RabbitMQ message into a call against the module's
     * public service interface (OrderService). Same role as a page/fragment controller,
     * but for MQ instead of HTTP.
     */
    @RabbitListener(queues = "app.events.order")
    void handleOrderEvent(OrderExportedEvent event) {
        log.info("Received order exported event: orderId={}, customerId={}",
                event.orderId(), event.customerId());
        // Delegate to the module's public service interface.
        // Do NOT inject another module's *ServiceImpl or repository here.
    }
}
```

> **Placement rule** — for every `@RabbitListener` you generate:
> 1. Identify the **bounded context** it serves (which module's data/logic it drives).
> 2. Put the class in `{{BASE_PACKAGE}}.<that-module>.internal`.
> 3. Make it package-private (drop `public`) so other modules cannot reference it.
> 4. Inject only that module's own components or other modules' **public service interfaces** — never another module's `*ServiceImpl`, `*Repository`, or `*Entity`.
> 5. If a single message would fan out work across multiple modules, publish a Spring Modulith `ApplicationEvent` from the listener and let each module react via `@ApplicationModuleListener` in its own `internal/` package.

---

## Error Handling

### Retry with Dead Letter Queue

Queues are configured with `x-dead-letter-exchange` pointing to `app.dlx.exchange`.
When a message fails after the configured retry attempts, Spring AMQP routes it to the
dead letter queue (`app.dlq`) for manual investigation.

### Custom Error Handler (optional)

```java
package {{BASE_PACKAGE}}.shared.messaging;

import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.listener.api.RabbitListenerErrorHandler;
import org.springframework.amqp.rabbit.support.ListenerExecutionFailedException;
import org.springframework.stereotype.Component;

@Component("messagingErrorHandler")
@Slf4j
public class MessagingErrorHandler implements RabbitListenerErrorHandler {

    @Override
    public Object handleError(Message amqpMessage,
                              org.springframework.messaging.Message<?> message,
                              ListenerExecutionFailedException exception) {
        log.error("Failed to process message from queue [{}]: {}",
                amqpMessage.getMessageProperties().getConsumerQueue(),
                exception.getCause().getMessage(),
                exception);
        // Optionally: send alert, record metric, etc.
        // Re-throw to trigger retry / DLQ routing
        throw exception;
    }
}
```

Usage in a consumer:
```java
@RabbitListener(queues = "app.events.order", errorHandler = "messagingErrorHandler")
public void handleOrderEvent(OrderExportedEvent event) {
    // ...
}
```

### Retry Configuration (application.yml)

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        retry:
          enabled: true
          initial-interval: 1000
          max-attempts: 3
          max-interval: 10000
          multiplier: 2.0
```

---

## Usage from Modules

Modules inject `RabbitMQPublisher` to send messages to external systems.
This is **separate** from Spring Modulith's `ApplicationEventPublisher`, which handles
in-process inter-module events.

```java
package {{BASE_PACKAGE}}.order.internal;

import {{BASE_PACKAGE}}.shared.messaging.OrderExportedEvent;
import {{BASE_PACKAGE}}.shared.messaging.RabbitMQPublisher;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import java.time.Instant;

@Service
@RequiredArgsConstructor
@Slf4j
class OrderServiceImpl implements OrderService {

    private final OrderRepository repository;
    private final RabbitMQPublisher messagingPublisher;

    public void exportOrder(String orderId) {
        var order = repository.findById(orderId)
                .orElseThrow(() -> new OrderException.NotFound(orderId));

        // ... export logic ...

        // Publish event to external systems via RabbitMQ
        messagingPublisher.publishEvent("order.exported",
                new OrderExportedEvent(order.getId(), order.getCustomerId(), Instant.now()));
        log.info("Order {} exported and event published", orderId);
    }
}
```

---

## Operational Notes

- **Connection pooling**: Spring Boot auto-configures a `CachingConnectionFactory`.
  Default channel cache size is 25. Adjust `spring.rabbitmq.cache.channel.size` for
  high-throughput scenarios.
- **Heartbeat**: Set `spring.rabbitmq.requested-heartbeat: 30` in production to detect
  dropped connections within 60 seconds (2x heartbeat interval).
- **Consumer prefetch**: `spring.rabbitmq.listener.simple.prefetch` controls how many
  messages a consumer fetches at once. Default is 250; set to 1-10 for slow consumers
  to avoid message starvation across instances.
- **Publisher confirms**: Enable `spring.rabbitmq.publisher-confirm-type: correlated` in
  production to guarantee message delivery to the broker.
- **Monitoring**: Monitor the dead letter queue (`app.dlq`) for failed messages.
  Integrate with your alerting system to notify on DLQ message accumulation.
- **Queue naming convention**: All messaging queues use the `app.*` prefix. Batch
  partition queues use `batch.partition.*`. These namespaces must not overlap.
