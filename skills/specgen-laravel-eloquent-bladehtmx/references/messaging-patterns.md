# Messaging Patterns — RabbitMQ Pub/Sub

This reference describes the standalone RabbitMQ pub/sub messaging layer for inter-system
communication. This is **independent** from Laravel's internal event system and from
queue batching — it provides explicit publisher/consumer services that modules call to
communicate with external systems.

---

## Architecture Overview

```
                        ┌─────────────────────────────────────────────┐
                        │              RabbitMQ Broker                │
                        │                                             │
  ┌──────────┐          │  ┌──────────────────┐   ┌───────────────┐  │          ┌──────────────┐
  │  Module   │ publish  │  │  app.events      │   │ app.commands  │  │ consume  │   External   │
  │           │────────►│  │  (TopicExchange)  │   │ (DirectExch.) │  │────────►│   System A   │
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

## Composer Dependencies

```json
{
    "require": {
        "vladimir-yuldashev/laravel-queue-rabbitmq": "^14.0",
        "php-amqplib/php-amqplib": "^3.7"
    }
}
```

> **Note:** `laravel-queue-rabbitmq` provides the queue driver for consuming messages
> as Laravel jobs. `php-amqplib` provides low-level AMQP access for advanced exchange
> patterns (topic/direct publishing with routing keys).

---

## Application Configuration

### .env

```env
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USER=guest
RABBITMQ_PASSWORD=guest
RABBITMQ_VHOST=/
RABBITMQ_QUEUE=default
```

### config/queue.php (rabbitmq connection)

```php
'rabbitmq' => [
    'driver' => 'rabbitmq',
    'queue' => env('RABBITMQ_QUEUE', 'default'),
    'connection' => PhpAmqpLib\Connection\AMQPLazyConnection::class,
    'hosts' => [
        [
            'host' => env('RABBITMQ_HOST', '127.0.0.1'),
            'port' => env('RABBITMQ_PORT', 5672),
            'user' => env('RABBITMQ_USER', 'guest'),
            'password' => env('RABBITMQ_PASSWORD', 'guest'),
            'vhost' => env('RABBITMQ_VHOST', '/'),
        ],
    ],
    'options' => [
        'ssl_options' => [
            'cafile' => env('RABBITMQ_SSL_CAFILE'),
            'local_cert' => env('RABBITMQ_SSL_LOCALCERT'),
            'local_key' => env('RABBITMQ_SSL_LOCALKEY'),
            'verify_peer' => env('RABBITMQ_SSL_VERIFY_PEER', true),
        ],
        'queue' => [
            'exchange' => env('RABBITMQ_EXCHANGE', ''),
            'exchange_type' => env('RABBITMQ_EXCHANGE_TYPE', 'direct'),
            'exchange_routing_key' => env('RABBITMQ_EXCHANGE_ROUTING_KEY', ''),
        ],
        'heartbeat' => 30,
    ],
],
```

### config/messaging.php (custom exchange configuration)

```php
return [
    'exchanges' => [
        'events' => env('MESSAGING_EVENTS_EXCHANGE', 'app.events.exchange'),
        'commands' => env('MESSAGING_COMMANDS_EXCHANGE', 'app.commands.exchange'),
        'dead_letter' => env('MESSAGING_DLX_EXCHANGE', 'app.dlx.exchange'),
    ],

    'queues' => [
        'order_events' => 'app.events.order',
        'inventory_sync' => 'app.commands.inventory-sync',
        'dead_letter' => 'app.dlq',
    ],
];
```

---

## Package Structure

```
app/
└── Services/
    └── Messaging/
        ├── RabbitMQInfrastructure.php     # Exchange, queue, binding declarations
        ├── MessagePublisher.php            # Generic publisher service
        └── Consumers/
            └── SampleEventConsumer.php     # Sample consumer job
```

---

## RabbitMQ Infrastructure Setup

Service to declare exchanges, queues, and bindings at application boot:

```php
namespace App\Services\Messaging;

use PhpAmqpLib\Channel\AMQPChannel;
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Wire\AMQPTable;

class RabbitMQInfrastructure
{
    private AMQPChannel $channel;

    public function __construct()
    {
        $connection = new AMQPStreamConnection(
            config('queue.connections.rabbitmq.hosts.0.host'),
            config('queue.connections.rabbitmq.hosts.0.port'),
            config('queue.connections.rabbitmq.hosts.0.user'),
            config('queue.connections.rabbitmq.hosts.0.password'),
            config('queue.connections.rabbitmq.hosts.0.vhost'),
        );
        $this->channel = $connection->channel();
    }

    public function declareInfrastructure(): void
    {
        $eventsExchange = config('messaging.exchanges.events');
        $commandsExchange = config('messaging.exchanges.commands');
        $dlxExchange = config('messaging.exchanges.dead_letter');

        // Declare exchanges
        $this->channel->exchange_declare($eventsExchange, 'topic', false, true, false);
        $this->channel->exchange_declare($commandsExchange, 'direct', false, true, false);
        $this->channel->exchange_declare($dlxExchange, 'fanout', false, true, false);

        // Dead letter queue
        $this->channel->queue_declare('app.dlq', false, true, false, false);
        $this->channel->queue_bind('app.dlq', $dlxExchange);

        // Sample event queue (Topic Exchange) — subscribe to order.* events
        $this->channel->queue_declare(
            config('messaging.queues.order_events'),
            false, true, false, false,
            false,
            new AMQPTable(['x-dead-letter-exchange' => $dlxExchange])
        );
        $this->channel->queue_bind(
            config('messaging.queues.order_events'),
            $eventsExchange,
            'order.*'
        );

        // Sample command queue (Direct Exchange) — inventory sync
        $this->channel->queue_declare(
            config('messaging.queues.inventory_sync'),
            false, true, false, false,
            false,
            new AMQPTable(['x-dead-letter-exchange' => $dlxExchange])
        );
        $this->channel->queue_bind(
            config('messaging.queues.inventory_sync'),
            $commandsExchange,
            'inventory.sync'
        );
    }
}
```

Register as an Artisan command for setup:
```php
namespace App\Console\Commands;

use App\Services\Messaging\RabbitMQInfrastructure;
use Illuminate\Console\Command;

class SetupRabbitMQCommand extends Command
{
    protected $signature = 'rabbitmq:setup';
    protected $description = 'Declare RabbitMQ exchanges, queues, and bindings';

    public function handle(RabbitMQInfrastructure $infra): int
    {
        $infra->declareInfrastructure();
        $this->info('RabbitMQ infrastructure declared successfully.');
        return self::SUCCESS;
    }
}
```

---

## Publisher Service

```php
namespace App\Services\Messaging;

use Illuminate\Support\Facades\Log;
use PhpAmqpLib\Channel\AMQPChannel;
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

class MessagePublisher
{
    private AMQPChannel $channel;

    public function __construct()
    {
        $connection = new AMQPStreamConnection(
            config('queue.connections.rabbitmq.hosts.0.host'),
            config('queue.connections.rabbitmq.hosts.0.port'),
            config('queue.connections.rabbitmq.hosts.0.user'),
            config('queue.connections.rabbitmq.hosts.0.password'),
            config('queue.connections.rabbitmq.hosts.0.vhost'),
        );
        $this->channel = $connection->channel();
    }

    /**
     * Publish a domain event to the topic exchange.
     * Consumers subscribe via routing key patterns (e.g., "order.*").
     */
    public function publishEvent(string $routingKey, array $event): void
    {
        Log::info("Publishing event [{$routingKey}]", $event);

        $message = new AMQPMessage(
            json_encode($event),
            ['content_type' => 'application/json', 'delivery_mode' => 2]
        );

        $this->channel->basic_publish(
            $message,
            config('messaging.exchanges.events'),
            $routingKey
        );
    }

    /**
     * Send a command to the direct exchange for point-to-point delivery.
     */
    public function sendCommand(string $routingKey, array $command): void
    {
        Log::info("Sending command [{$routingKey}]", $command);

        $message = new AMQPMessage(
            json_encode($command),
            ['content_type' => 'application/json', 'delivery_mode' => 2]
        );

        $this->channel->basic_publish(
            $message,
            config('messaging.exchanges.commands'),
            $routingKey
        );
    }
}
```

---

## Consumer Job

Consumers are implemented as Laravel queue jobs consumed via `laravel-queue-rabbitmq`:

```php
namespace App\Jobs\Consumers;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class OrderEventConsumer implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $backoff = 5;

    public function __construct(
        public readonly array $payload,
    ) {}

    public function handle(): void
    {
        Log::info('Received order event', $this->payload);
        // Process the event — e.g., notify downstream systems, update read models
    }

    public function failed(\Throwable $exception): void
    {
        Log::error('Failed to process order event', [
            'payload' => $this->payload,
            'error' => $exception->getMessage(),
        ]);
    }
}
```

For low-level AMQP consumption (bypassing Laravel queues):
```php
// Custom Artisan command for consuming from specific queues
namespace App\Console\Commands;

use Illuminate\Console\Command;
use PhpAmqpLib\Connection\AMQPStreamConnection;

class ConsumeOrderEventsCommand extends Command
{
    protected $signature = 'rabbitmq:consume-order-events';
    protected $description = 'Consume messages from the order events queue';

    public function handle(): int
    {
        $connection = new AMQPStreamConnection(
            config('queue.connections.rabbitmq.hosts.0.host'),
            config('queue.connections.rabbitmq.hosts.0.port'),
            config('queue.connections.rabbitmq.hosts.0.user'),
            config('queue.connections.rabbitmq.hosts.0.password'),
            config('queue.connections.rabbitmq.hosts.0.vhost'),
        );

        $channel = $connection->channel();
        $channel->basic_qos(0, 10, false);

        $channel->basic_consume(
            config('messaging.queues.order_events'),
            '',
            false, false, false, false,
            function ($msg) {
                $payload = json_decode($msg->body, true);
                $this->info("Received: " . json_encode($payload));

                // Process...

                $msg->ack();
            }
        );

        $this->info('Waiting for order events...');
        while ($channel->is_consuming()) {
            $channel->wait();
        }

        return self::SUCCESS;
    }
}
```

---

## Usage from Modules

Modules inject `MessagePublisher` to send messages to external systems.
This is **separate** from Laravel's `event()` helper, which handles in-process
inter-module events.

```php
namespace Modules\Order\Services;

use App\Services\Messaging\MessagePublisher;
use Modules\Order\Contracts\OrderServiceInterface;
use Modules\Order\Models\Order;

class OrderService implements OrderServiceInterface
{
    public function __construct(
        private readonly MessagePublisher $publisher,
    ) {}

    public function exportOrder(string $orderId): void
    {
        $order = Order::findOrFail($orderId);

        // ... export logic ...

        // Publish event to external systems via RabbitMQ
        $this->publisher->publishEvent('order.exported', [
            'order_id' => $order->id,
            'customer_id' => $order->customer_id,
            'exported_at' => now()->toIso8601String(),
        ]);
    }
}
```

---

## Error Handling

### Retry with Dead Letter Queue

Queues are configured with `x-dead-letter-exchange` pointing to `app.dlx.exchange`.
When a message fails after the configured retry attempts, it is routed to the dead
letter queue (`app.dlq`) for manual investigation.

### Failed Jobs Table

Laravel's `failed_jobs` table catches jobs that exhaust all retries:
```bash
php artisan queue:failed
php artisan queue:retry <id>
php artisan queue:flush
```

---

## Operational Notes

- **Connection pooling**: Use `AMQPLazyConnection` to establish connections on-demand
  rather than at boot time.
- **Heartbeat**: Set `heartbeat: 30` in queue config to detect dropped connections
  within 60 seconds (2x heartbeat interval).
- **Consumer prefetch**: Set `basic_qos(0, 10, false)` for slow consumers to avoid
  message starvation across instances.
- **Publisher confirms**: Enable `confirm_select()` on the channel in production to
  guarantee message delivery to the broker.
- **Monitoring**: Monitor the dead letter queue (`app.dlq`) for failed messages.
  Integrate with your alerting system to notify on DLQ accumulation.
- **Queue naming convention**: All messaging queues use the `app.*` prefix. Batch
  queues use the default Laravel queue naming. These namespaces must not overlap.
