# Scheduling Patterns — Laravel Task Scheduling + Queue Batching

This reference describes the scheduling and batch processing scaffolding for the spec.
Include this content in Section 18 of the generated specification. All code samples must
be reproduced in the generated spec with full use statements and constructor injection.

The scheduling feature has two tiers:
- **Scheduling = yes**: Laravel Task Scheduling for cron-triggered jobs (always included)
- **Scheduling = yes AND Batch Processing = yes**: Adds `Bus::batch()` for chunk-oriented
  data processing triggered by scheduled commands

---

## Architecture Overview

### Scheduling-Only Mode (Scheduling = yes, Batch Processing = no)

When batch processing is not selected, Laravel Task Scheduling runs standalone. Scheduled
commands execute application logic directly (service calls, API calls, cleanup tasks, etc.)
without the batch pipeline.

```
Laravel Scheduler (cron: * * * * * php artisan schedule:run)
    └── Scheduled Command (daily at 02:00)
        └── executes business logic directly (service calls, etc.)
```

### Scheduling + Batch Mode (Scheduling = yes, Batch Processing = yes)

When batch processing is selected, scheduled commands dispatch job batches. This gives
the application cron-based scheduling with robust, restartable, chunk-oriented processing.

- **Laravel Scheduler**: Manages job scheduling — cron triggers, intervals.
  The scheduler decides WHEN jobs run.
- **Bus::batch()**: Manages job execution — processing data in chunks with
  progress tracking, failure handling, and cancellation support.
  Batching decides HOW jobs run.

```
Laravel Scheduler (cron: * * * * *)
    └── Scheduled Command (daily at 02:00)
        └── Bus::batch([...chunk jobs...])
            ├── ProcessChunkJob (items 1-1000)
            ├── ProcessChunkJob (items 1001-2000)
            └── ProcessChunkJob (items 2001-3000)
```

---

## Composer Dependencies

No additional packages needed — scheduling and batching are built into Laravel:

```json
{
    "require": {
        "laravel/framework": "^12.0"
    }
}
```

For batch processing, ensure the batch tables migration exists:
```bash
php artisan make:queue-batches-table
php artisan migrate
```

---

## Application Configuration

### Scheduling Configuration

```php
// routes/console.php (Laravel 11+)
use Illuminate\Support\Facades\Schedule;

// Scheduling-only jobs
Schedule::command('app:cleanup-expired-records')
    ->dailyAt('02:00')
    ->withoutOverlapping()
    ->onOneServer()
    ->appendOutputTo(storage_path('logs/scheduler.log'));

// [If Batch Processing = yes] Batch-triggered jobs
Schedule::command('app:process-data-export')
    ->dailyAt('03:00')
    ->withoutOverlapping()
    ->onOneServer();
```

### Queue Configuration (for batching)

```php
// config/queue.php
'connections' => [
    'database' => [
        'driver' => 'database',
        'table' => 'jobs',
        'queue' => 'default',
        'retry_after' => 90,
        'after_commit' => true,
    ],

    // [If Messaging = yes] RabbitMQ connection
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
    ],
],
```

---

## Package Structure

```
app/
├── Console/
│   └── Commands/
│       ├── CleanupExpiredRecordsCommand.php    # Direct scheduled command
│       └── [If Batch] ProcessDataExportCommand.php  # Batch-dispatching command
├── Jobs/
│   ├── [If Batch] ProcessExportChunkJob.php    # Chunk processing job
│   └── [If Batch] ExportCompletionJob.php      # Post-batch cleanup
```

---

## Scheduling-Only Mode

### Scheduled Command (Direct Execution)

```php
namespace App\Console\Commands;

use Illuminate\Console\Command;
use Modules\Order\Contracts\OrderServiceInterface;

class CleanupExpiredRecordsCommand extends Command
{
    protected $signature = 'app:cleanup-expired-records';
    protected $description = 'Delete records older than the configured retention period';

    public function __construct(
        private readonly OrderServiceInterface $orderService,
    ) {
        parent::__construct();
    }

    public function handle(): int
    {
        $this->info('Starting cleanup of expired records...');

        $deleted = $this->orderService->deleteExpiredRecords(
            olderThanDays: config('app.retention_days', 30)
        );

        $this->info("Deleted {$deleted} expired records.");

        return self::SUCCESS;
    }
}
```

---

## Scheduling + Batch Mode

### Batch-Dispatching Command

```php
namespace App\Console\Commands;

use App\Jobs\ProcessExportChunkJob;
use Illuminate\Bus\Batch;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\Bus;
use Modules\Order\Contracts\OrderServiceInterface;
use Throwable;

class ProcessDataExportCommand extends Command
{
    protected $signature = 'app:process-data-export';
    protected $description = 'Export order data in batched chunks';

    public function __construct(
        private readonly OrderServiceInterface $orderService,
    ) {
        parent::__construct();
    }

    public function handle(): int
    {
        $this->info('Preparing data export batch...');

        $totalRecords = $this->orderService->countExportable();
        $chunkSize = 1000;
        $jobs = [];

        for ($offset = 0; $offset < $totalRecords; $offset += $chunkSize) {
            $jobs[] = new ProcessExportChunkJob(
                offset: $offset,
                limit: $chunkSize,
            );
        }

        if (empty($jobs)) {
            $this->info('No records to export.');
            return self::SUCCESS;
        }

        $batch = Bus::batch($jobs)
            ->then(function (Batch $batch) {
                logger()->info("Export batch {$batch->id} completed successfully.");
            })
            ->catch(function (Batch $batch, Throwable $e) {
                logger()->error("Export batch {$batch->id} failed: {$e->getMessage()}");
            })
            ->finally(function (Batch $batch) {
                logger()->info("Export batch {$batch->id} finished. " .
                    "Processed: {$batch->processedJobs()}, Failed: {$batch->failedJobs}");
            })
            ->name('data-export-' . now()->format('Y-m-d'))
            ->allowFailures()
            ->dispatch();

        $this->info("Dispatched batch {$batch->id} with " . count($jobs) . " jobs.");

        return self::SUCCESS;
    }
}
```

### Chunk Processing Job

```php
namespace App\Jobs;

use Illuminate\Bus\Batchable;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Modules\Order\Contracts\OrderServiceInterface;

class ProcessExportChunkJob implements ShouldQueue
{
    use Batchable, Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $backoff = 60;

    public function __construct(
        public readonly int $offset,
        public readonly int $limit,
    ) {}

    public function handle(OrderServiceInterface $orderService): void
    {
        if ($this->batch()?->cancelled()) {
            return;
        }

        $records = $orderService->getExportableChunk($this->offset, $this->limit);

        foreach ($records as $record) {
            // Process each record (transform, export, etc.)
            $orderService->markAsExported($record->id);
        }

        logger()->info("Processed export chunk: offset={$this->offset}, count=" . count($records));
    }
}
```

---

## Monitoring Batches

Laravel provides built-in batch monitoring:

```php
// Check batch progress
$batch = Bus::findBatch($batchId);
echo "Progress: {$batch->progress()}%";
echo "Pending: {$batch->pendingJobs}";
echo "Failed: {$batch->failedJobs}";
echo "Finished: " . ($batch->finished() ? 'yes' : 'no');

// Cancel a running batch
$batch->cancel();
```

For a dashboard view, query the `job_batches` table:
```php
$batches = DB::table('job_batches')
    ->orderByDesc('created_at')
    ->paginate(20);
```

---

## Retry and Error Handling

### Job-level retry configuration

```php
class ProcessExportChunkJob implements ShouldQueue
{
    public int $tries = 3;           // Max retry attempts
    public int $backoff = 60;        // Seconds between retries
    public int $timeout = 300;       // Max execution time in seconds
    public int $maxExceptions = 2;   // Max exceptions before marking failed

    public function failed(Throwable $exception): void
    {
        logger()->error("Export chunk failed permanently: {$exception->getMessage()}", [
            'offset' => $this->offset,
            'limit' => $this->limit,
        ]);
    }
}
```

### Failed jobs

Failed jobs are stored in the `failed_jobs` table. Retry or delete them:
```bash
php artisan queue:retry <job-id>
php artisan queue:retry --queue=default
php artisan queue:flush
```

---

## Cron Setup

The Laravel scheduler requires a single cron entry on the server:

```cron
* * * * * cd /path-to-project && php artisan schedule:run >> /dev/null 2>&1
```

In Docker/production, use `schedule:work` for foreground scheduling:
```bash
php artisan schedule:work
```
