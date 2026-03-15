# Scheduling Patterns — Quartz Scheduling + Optional Spring Batch Processing

This reference describes the scheduling and batch processing scaffolding for the spec.
Include this content in Section 18 of the generated specification. All code samples must
be reproduced in the generated spec with full import statements and constructor injection.

The scheduling feature has two tiers:
- **Scheduling = yes**: Quartz scheduler for cron-triggered jobs (always included)
- **Scheduling = yes AND Spring Batch = yes**: Adds Spring Batch reader/processor/writer
  pipeline for chunk-oriented data processing, triggered by Quartz

---

## Architecture Overview

### Quartz-Only Mode (Scheduling = yes, Spring Batch = no)

When Spring Batch is not selected, Quartz runs standalone. Jobs extend `QuartzJobBean`
and execute application logic directly (service calls, API calls, cleanup tasks, etc.)
without the Batch reader/processor/writer pipeline.

```
Quartz Scheduler (job store matches selected database)
    └── Trigger (cron: "0 0 2 * * ?")
        └── QuartzJobBean
            └── executes business logic directly (service calls, etc.)
```

### Quartz + Spring Batch Mode (Scheduling = yes, Spring Batch = yes)

When Spring Batch is selected, Quartz triggers fire Spring Batch jobs. This gives the
application cron-based scheduling with robust, restartable, chunk-oriented processing.

- **Spring Quartz**: Manages job scheduling — cron triggers, intervals, fire-and-forget.
  Quartz decides WHEN jobs run.
- **Spring Batch**: Manages job execution — reading, processing, writing data in chunks.
  Batch decides HOW jobs run.

```
Quartz Scheduler (job store matches selected database)
    └── Trigger (cron: "0 0 2 * * ?")
        └── QuartzJobBean
            └── launches Spring Batch Job
                └── Step 1: Reader → Processor → Writer
```

---

## Maven Dependency

Include in `pom.xml`:

**[If Database = MongoDB]:**
```xml
<!-- Quartz MongoDB Job Store -->
<dependency>
    <groupId>io.fluidsonic.mirror</groupId>
    <artifactId>fluidsonic-mirror-quartz</artifactId>
    <version>${fluidsonic-mirror.version}</version>
</dependency>
```

---

## Quartz Configuration

### application.yml Quartz section

```yaml
spring:
  quartz:
    job-store-type: mongodb
    properties:
      org.quartz.threadPool.threadCount: 5
      org.quartz.scheduler.instanceName: {{ARTIFACT_ID}}-scheduler
      org.quartz.scheduler.instanceId: AUTO
      org.quartz.jobStore.class: io.fluidsonic.mirror.quartz.MongoDBJobStore
      org.quartz.jobStore.mongoUri: mongodb://localhost:27017/{{DB_NAME}}
      org.quartz.jobStore.dbName: {{DB_NAME}}
      org.quartz.jobStore.collectionPrefix: qrtz_
```

The `fluidsonic-mirror` MongoDB job store persists job details, triggers, and scheduler
state directly in MongoDB collections prefixed with `qrtz_`. This eliminates the need
for a relational database and supports clustered Quartz instances with `instanceId: AUTO`.

Adapt the `job-store-type` and properties based on the selected database:
- **MongoDB**: Use `mongodb` job store type with `fluidsonic-mirror-quartz` (as shown above)
- **PostgreSQL/MySQL**: Use `jdbc` job store type with built-in Quartz JDBC support
- **none**: Use `memory` job store type (no persistence, lost on restart)

### QuartzConfig.java

```java
package {{BASE_PACKAGE}}.config;

import {{BASE_PACKAGE}}.scheduling.internal.job.SampleQuartzJob;
import org.quartz.CronScheduleBuilder;
import org.quartz.JobBuilder;
import org.quartz.JobDetail;
import org.quartz.Trigger;
import org.quartz.TriggerBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class QuartzConfig {

    @Bean
    public JobDetail sampleJobDetail() {
        return JobBuilder.newJob(SampleQuartzJob.class)
            .withIdentity("sampleJob", "default-group")
            .withDescription("Sample scheduled job that triggers a Spring Batch job")
            .storeDurably()
            .build();
    }

    @Bean
    public Trigger sampleJobTrigger(JobDetail sampleJobDetail) {
        return TriggerBuilder.newTrigger()
            .forJob(sampleJobDetail)
            .withIdentity("sampleTrigger", "default-group")
            .withDescription("Fires daily at 2 AM")
            .withSchedule(CronScheduleBuilder.cronSchedule("0 0 2 * * ?"))
            .build();
    }
}
```

---

## Quartz Job Class (Quartz-Only — Spring Batch = no)

When Spring Batch is not selected, Quartz jobs execute business logic directly via
service calls. This is the default when `Scheduling = yes` but `Spring Batch = no`.

### SampleQuartzJob.java (Quartz-only)

```java
package {{BASE_PACKAGE}}.scheduling.internal.job;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.springframework.scheduling.quartz.QuartzJobBean;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class SampleQuartzJob extends QuartzJobBean {

    // Inject your application services here
    // private final SomeService someService;

    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        try {
            log.info("Executing scheduled job: sampleJob");

            // Execute business logic directly — no Spring Batch pipeline
            // someService.performScheduledTask();

            log.info("Scheduled job sampleJob completed successfully");
        } catch (Exception e) {
            log.error("Failed to execute scheduled job: sampleJob", e);
            throw new JobExecutionException("Scheduled job execution failed", e);
        }
    }
}
```

---

## Quartz Job Class (Spring Batch = yes)

When Spring Batch is selected, the Quartz job launches a Spring Batch job instead of
executing logic directly. Include this version **only when Spring Batch = yes**.

### SampleQuartzJob.java (Batch launcher)

```java
package {{BASE_PACKAGE}}.scheduling.internal.job;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.scheduling.quartz.QuartzJobBean;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class SampleQuartzJob extends QuartzJobBean {

    private final JobLauncher jobLauncher;
    private final Job sampleBatchJob;

    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        try {
            JobParameters params = new JobParametersBuilder()
                .addLong("timestamp", System.currentTimeMillis())
                .toJobParameters();

            log.info("Launching batch job: sampleBatchJob at {}", params);
            jobLauncher.run(sampleBatchJob, params);
            log.info("Batch job sampleBatchJob completed successfully");
        } catch (Exception e) {
            log.error("Failed to launch batch job: sampleBatchJob", e);
            throw new JobExecutionException("Batch job launch failed", e);
        }
    }
}
```

The `timestamp` parameter ensures each execution is unique (Spring Batch requires
unique parameters per run by default).

---

## Spring Batch Configuration [Include only if Spring Batch = yes]

### SampleBatchConfig.java

```java
package {{BASE_PACKAGE}}.scheduling.internal.batch;

import lombok.RequiredArgsConstructor;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
@RequiredArgsConstructor
public class SampleBatchConfig {

    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;

    @Bean
    public Job sampleBatchJob(Step sampleStep) {
        return new JobBuilder("sampleBatchJob", jobRepository)
            .start(sampleStep)
            .build();
    }

    @Bean
    public Step sampleStep(
            ItemReader<SampleSourceDocument> reader,
            ItemProcessor<SampleSourceDocument, SampleTargetDocument> processor,
            ItemWriter<SampleTargetDocument> writer) {
        return new StepBuilder("sampleStep", jobRepository)
            .<SampleSourceDocument, SampleTargetDocument>chunk(100, transactionManager)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .faultTolerant()
            .skipLimit(10)
            .skip(Exception.class)
            .build();
    }
}
```

Key configuration points:
- **Chunk size**: 100 items per transaction commit (tunable)
- **Fault tolerance**: Skip up to 10 failures before aborting
- **Transaction manager**: Each chunk is a single transaction

### Batch YAML configuration [Include only if Spring Batch = yes]

```yaml
spring:
  batch:
    jdbc:
      initialize-schema: never   # MongoDB-backed app, no JDBC schema needed
    job:
      enabled: false             # Prevent auto-run on startup; Quartz triggers jobs
```

---

## ItemReader — MongoDB Source [Include only if Spring Batch = yes]

### SampleItemReader.java

```java
package {{BASE_PACKAGE}}.scheduling.internal.batch;

import lombok.RequiredArgsConstructor;
import org.springframework.batch.core.configuration.annotation.StepScope;
import org.springframework.batch.item.data.MongoPagingItemReader;
import org.springframework.batch.item.data.builder.MongoPagingItemReaderBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.Sort;
import org.springframework.data.mongodb.core.MongoTemplate;
import java.util.Map;

@Configuration
@RequiredArgsConstructor
public class SampleItemReaderConfig {

    private final MongoTemplate mongoTemplate;

    @StepScope
    @Bean
    public MongoPagingItemReader<SampleSourceDocument> sampleReader() {
        return new MongoPagingItemReaderBuilder<SampleSourceDocument>()
            .name("sampleReader")
            .template(mongoTemplate)
            .targetType(SampleSourceDocument.class)
            .jsonQuery("{}")
            .sorts(Map.of("_id", Sort.Direction.ASC))
            .pageSize(100)
            .build();
    }
}
```

`@StepScope` makes the reader step-scoped, allowing late binding of parameters
(e.g., date ranges passed from Quartz job parameters).

---

## ItemProcessor — Transformation [Include only if Spring Batch = yes]

### SampleItemProcessor.java

```java
package {{BASE_PACKAGE}}.scheduling.internal.batch;

import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.stereotype.Component;

@Component
@Slf4j
public class SampleItemProcessor implements ItemProcessor<SampleSourceDocument, SampleTargetDocument> {

    @Override
    public SampleTargetDocument process(SampleSourceDocument item) {
        log.debug("Processing item: {}", item.getId());

        // Return null to skip/filter this item
        if (item.getFieldOne() == null) {
            log.warn("Skipping item with null fieldOne: {}", item.getId());
            return null;
        }

        return SampleTargetDocument.builder()
            .sourceId(item.getId())
            .transformedField(item.getFieldOne().toUpperCase())
            .processedAt(java.time.Instant.now())
            .build();
    }
}
```

---

## ItemWriter — MongoDB Sink [Include only if Spring Batch = yes]

### SampleItemWriter.java

```java
package {{BASE_PACKAGE}}.scheduling.internal.batch;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.item.Chunk;
import org.springframework.batch.item.ItemWriter;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class SampleItemWriter implements ItemWriter<SampleTargetDocument> {

    private final MongoTemplate mongoTemplate;

    @Override
    public void write(Chunk<? extends SampleTargetDocument> chunk) {
        log.info("Writing batch of {} items", chunk.size());
        mongoTemplate.insertAll(chunk.getItems());
    }
}
```

For updates to existing documents, use `chunk.getItems().forEach(mongoTemplate::save)`.
For inserts of new documents, `mongoTemplate.insertAll()` is more efficient as it uses
a bulk insert operation.

---

## Sample Document Classes [Include only if Spring Batch = yes]

These placeholder document classes are used by the reader, processor, and writer.
Replace with actual module types in the real implementation.

```java
package {{BASE_PACKAGE}}.scheduling.internal.batch;

import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "sample_source")
public class SampleSourceDocument {

    @Id
    private String id;
    private String fieldOne;
    private String fieldTwo;
}
```

```java
package {{BASE_PACKAGE}}.scheduling.internal.batch;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import java.time.Instant;

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "sample_target")
public class SampleTargetDocument {

    @Id
    private String id;
    private String sourceId;
    private String transformedField;
    private Instant processedAt;
}
```

---

## Job Metadata Storage

Quartz job metadata (triggers, job details, scheduler state) is persisted in MongoDB
via `io.fluidsonic.mirror`. This is configured entirely through `application.yml` Quartz
properties — no additional Java configuration is needed.

Spring Batch's `JobRepository` (which tracks job execution history, step status, etc.)
uses an in-memory `MapJobRepositoryFactoryBean` by default in this skeleton. For
production use where job restart and execution history matter, replace with a MongoDB-backed
`JobRepository` implementation.

---

## Scheduling Module Package Structure

### Quartz-Only (Scheduling = yes, Spring Batch = no)

```
scheduling/
├── SchedulingService.java              # Public interface (if other modules need to trigger jobs)
└── internal/
    └── job/
        └── SampleQuartzJob.java        # Quartz job — executes logic directly
```

### Quartz + Spring Batch (Scheduling = yes, Spring Batch = yes)

```
scheduling/
├── SchedulingService.java              # Public interface (if other modules need to trigger jobs)
└── internal/
    ├── job/
    │   └── SampleQuartzJob.java        # Quartz job that launches batch
    └── batch/
        ├── SampleBatchConfig.java      # Job, Step, chunk config
        ├── SampleItemReader.java       # MongoDB reader
        ├── SampleItemProcessor.java    # Transformation logic
        └── SampleItemWriter.java       # MongoDB writer
```

This module follows the same public/internal encapsulation as other modules. The Quartz
jobs and Batch configuration are implementation details hidden in `internal/`. If other
modules need to trigger a job programmatically, they use the `SchedulingService` interface
in the module root. Otherwise, jobs are triggered purely by Quartz cron schedules and the
public interface can be omitted.

---

# Remote Partitioning with RabbitMQ

This section covers **Spring Batch remote partitioning** using **RabbitMQ** as the
messaging middleware. Remote partitioning enables horizontal scaling of batch jobs across
multiple worker nodes. Include this content in Section 18 of the generated specification
only when **Scheduling = yes AND Remote Partitioning = yes**. All code samples must be
reproduced in the generated spec with full import statements and constructor injection.

---

## Remote Partitioning Architecture

Remote partitioning splits a batch step's data across multiple worker processes. A
**manager** node partitions the input and sends partition metadata via RabbitMQ queues.
**Worker** nodes pick up partitions, execute the step locally, and report completion back
via a reply queue.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Manager Node                                │
│                                                                     │
│  Quartz Trigger ─► QuartzJobBean ─► Spring Batch Job                │
│                                         │                           │
│                                    Manager Step                     │
│                                    (Partitioner)                    │
│                                         │                           │
│                            ┌────────────┴────────────┐              │
│                            ▼                         ▼              │
│                   Partition 1 metadata      Partition N metadata     │
│                            │                         │              │
│                            ▼                         ▼              │
│               ┌─── Outbound Channel (request) ───┐                  │
│               │    AmqpOutboundChannelAdapter     │                  │
│               └──────────────┬───────────────────┘                  │
│                              │                                      │
└──────────────────────────────┼──────────────────────────────────────┘
                               │
                               ▼
                ┌──────────────────────────────┐
                │         RabbitMQ             │
                │                              │
                │  batch.partition.requests     │  (Direct Exchange)
                │  batch.partition.replies      │
                │  batch.partition.requests.dlq │  (Dead Letter Queue)
                └──────────┬───────────────────┘
                           │
              ┌────────────┼────────────────┐
              ▼            ▼                ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│   Worker Node 1  │ │   Worker Node 2  │ │   Worker Node N  │
│                  │ │                  │ │                  │
│  Inbound Channel │ │  Inbound Channel │ │  Inbound Channel │
│  (request)       │ │  (request)       │ │  (request)       │
│       │          │ │       │          │ │       │          │
│       ▼          │ │       ▼          │ │       ▼          │
│  Worker Step     │ │  Worker Step     │ │  Worker Step     │
│  Reader→         │ │  Reader→         │ │  Reader→         │
│  Processor→      │ │  Processor→      │ │  Processor→      │
│  Writer          │ │  Writer          │ │  Writer          │
│       │          │ │       │          │ │       │          │
│       ▼          │ │       ▼          │ │       ▼          │
│  Outbound Channel│ │  Outbound Channel│ │  Outbound Channel│
│  (reply)         │ │  (reply)         │ │  (reply)         │
└──────────────────┘ └──────────────────┘ └──────────────────┘
```

Key points:
- The **manager** only partitions and delegates — it does NOT read/process/write data itself
- Each **worker** receives a `StepExecution` with partition-specific `ExecutionContext`
  (e.g., `minId`, `maxId`) and runs a full Reader → Processor → Writer pipeline locally
- All nodes share the **same `JobRepository` database** so the manager can track partition
  completion; this is a hard requirement for remote partitioning
- Workers are deployed as **separate JVM instances** of the same application JAR, activated
  with a different Spring profile (e.g., `--spring.profiles.active=worker`)

---

## Maven Dependencies (Remote Partitioning)

Add these dependencies **in addition to** the base scheduling dependencies
(`spring-boot-starter-quartz`, `spring-boot-starter-batch`):

```xml
<!-- Spring Batch Integration — remote partitioning support -->
<dependency>
    <groupId>org.springframework.batch</groupId>
    <artifactId>spring-batch-integration</artifactId>
</dependency>

<!-- Spring Integration AMQP — RabbitMQ channel adapters -->
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-amqp</artifactId>
</dependency>

<!-- Spring Boot AMQP Starter — RabbitMQ auto-configuration -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

These are managed by the Spring Boot BOM — no explicit version needed.

---

## Application Configuration (Remote Partitioning)

### application.yml — RabbitMQ connection

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /

app:
  batch:
    partition:
      grid-size: 4
      request-queue: batch.partition.requests
      reply-queue: batch.partition.replies
      request-dlq: batch.partition.requests.dlq
      exchange: batch.partition.exchange
```

### application-worker.yml — Worker profile

Workers disable the Quartz scheduler and web server since they only process partitions:

```yaml
spring:
  quartz:
    auto-startup: false
  main:
    web-application-type: none

server:
  port: 0
```

Activate with `--spring.profiles.active=worker`.

---

## RabbitMQ Infrastructure Configuration

### RabbitMQPartitionConfig.java

```java
package {{BASE_PACKAGE}}.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.QueueBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQPartitionConfig {

    @Value("${app.batch.partition.exchange}")
    private String exchangeName;

    @Value("${app.batch.partition.request-queue}")
    private String requestQueueName;

    @Value("${app.batch.partition.reply-queue}")
    private String replyQueueName;

    @Value("${app.batch.partition.request-dlq}")
    private String requestDlqName;

    @Bean
    public DirectExchange partitionExchange() {
        return new DirectExchange(exchangeName);
    }

    @Bean
    public Queue partitionRequestQueue() {
        return QueueBuilder.durable(requestQueueName)
            .withArgument("x-dead-letter-exchange", "")
            .withArgument("x-dead-letter-routing-key", requestDlqName)
            .build();
    }

    @Bean
    public Queue partitionReplyQueue() {
        return QueueBuilder.durable(replyQueueName).build();
    }

    @Bean
    public Queue partitionRequestDlq() {
        return QueueBuilder.durable(requestDlqName).build();
    }

    @Bean
    public Binding requestBinding(Queue partitionRequestQueue, DirectExchange partitionExchange) {
        return BindingBuilder.bind(partitionRequestQueue)
            .to(partitionExchange)
            .with(requestQueueName);
    }

    @Bean
    public Binding replyBinding(Queue partitionReplyQueue, DirectExchange partitionExchange) {
        return BindingBuilder.bind(partitionReplyQueue)
            .to(partitionExchange)
            .with(replyQueueName);
    }
}
```

---

## Partitioner — Splitting Data into Ranges

### SamplePartitioner.java

The Partitioner creates a `Map<String, ExecutionContext>` where each entry represents one
partition. Workers receive these execution contexts with range parameters to scope their
data reading.

```java
package {{BASE_PACKAGE}}.scheduling.internal.batch.partition;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.core.partition.support.Partitioner;
import org.springframework.batch.item.ExecutionContext;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.stereotype.Component;
import java.util.HashMap;
import java.util.Map;

@Component
@RequiredArgsConstructor
@Slf4j
public class SamplePartitioner implements Partitioner {

    private final MongoTemplate mongoTemplate;

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        long totalCount = mongoTemplate.count(new Query(), "sample_source");
        long partitionSize = (totalCount + gridSize - 1) / gridSize;

        Map<String, ExecutionContext> partitions = new HashMap<>();
        for (int i = 0; i < gridSize; i++) {
            long minIndex = i * partitionSize;
            long maxIndex = Math.min((i + 1) * partitionSize - 1, totalCount - 1);

            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minIndex", minIndex);
            ctx.putLong("maxIndex", maxIndex);
            ctx.putString("partitionName", "partition-" + i);

            partitions.put("partition-" + i, ctx);
            log.info("Created partition-{}: minIndex={}, maxIndex={}", i, minIndex, maxIndex);
        }
        return partitions;
    }
}
```

For relational databases (PostgreSQL/MySQL), partition by primary key ID ranges instead:

```java
// Alternative for JPA-backed data
@Override
public Map<String, ExecutionContext> partition(int gridSize) {
    Long minId = jdbcTemplate.queryForObject("SELECT MIN(id) FROM sample_source", Long.class);
    Long maxId = jdbcTemplate.queryForObject("SELECT MAX(id) FROM sample_source", Long.class);
    long range = (maxId - minId + gridSize) / gridSize;

    Map<String, ExecutionContext> partitions = new HashMap<>();
    for (int i = 0; i < gridSize; i++) {
        ExecutionContext ctx = new ExecutionContext();
        ctx.putLong("minId", minId + (i * range));
        ctx.putLong("maxId", Math.min(minId + ((i + 1) * range) - 1, maxId));
        partitions.put("partition-" + i, ctx);
    }
    return partitions;
}
```

---

## Spring Integration Channels

### PartitionChannelConfig.java

Defines the `DirectChannel` beans and AMQP adapters that bridge Spring Batch remote
partitioning with RabbitMQ.

```java
package {{BASE_PACKAGE}}.scheduling.internal.batch.partition;

import lombok.RequiredArgsConstructor;
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.amqp.dsl.Amqp;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.messaging.MessageChannel;

@Configuration
@RequiredArgsConstructor
public class PartitionChannelConfig {

    private final ConnectionFactory connectionFactory;
    private final AmqpTemplate amqpTemplate;

    @Value("${app.batch.partition.request-queue}")
    private String requestQueueName;

    @Value("${app.batch.partition.reply-queue}")
    private String replyQueueName;

    @Value("${app.batch.partition.exchange}")
    private String exchangeName;

    // --- Channels ---

    @Bean
    public MessageChannel partitionRequestChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageChannel partitionReplyChannel() {
        return new DirectChannel();
    }

    // --- Outbound: Manager sends partitions to RabbitMQ request queue ---

    @Bean
    public IntegrationFlow outboundRequestFlow() {
        return IntegrationFlow.from(partitionRequestChannel())
            .handle(Amqp.outboundAdapter(amqpTemplate)
                .exchangeName(exchangeName)
                .routingKey(requestQueueName))
            .get();
    }

    // --- Inbound: Manager receives replies from RabbitMQ reply queue ---

    @Bean
    public IntegrationFlow inboundReplyFlow() {
        return IntegrationFlow.from(
                Amqp.inboundAdapter(connectionFactory, replyQueueName))
            .channel(partitionReplyChannel())
            .get();
    }

    // --- Inbound: Worker receives partitions from RabbitMQ request queue ---

    @Bean
    public IntegrationFlow inboundRequestFlow() {
        return IntegrationFlow.from(
                Amqp.inboundAdapter(connectionFactory, requestQueueName))
            .channel(partitionRequestChannel())
            .get();
    }

    // --- Outbound: Worker sends replies to RabbitMQ reply queue ---

    @Bean
    public IntegrationFlow outboundReplyFlow() {
        return IntegrationFlow.from(partitionReplyChannel())
            .handle(Amqp.outboundAdapter(amqpTemplate)
                .exchangeName(exchangeName)
                .routingKey(replyQueueName))
            .get();
    }
}
```

**Important:** In a real deployment, the manager and worker channel beans must be
separated using Spring profiles (`@Profile("manager")` / `@Profile("worker")`) so that
manager-only flows (outbound request, inbound reply) and worker-only flows (inbound
request, outbound reply) are activated on the correct node. The sample above shows all
flows in one class for clarity.

---

## Manager Step Configuration

### PartitionedBatchConfig.java (Manager)

The manager step uses `RemotePartitioningManagerStepBuilder` to send partition metadata
to workers via RabbitMQ and aggregate their results.

```java
package {{BASE_PACKAGE}}.scheduling.internal.batch.partition;

import lombok.RequiredArgsConstructor;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.partition.support.Partitioner;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.integration.partition.RemotePartitioningManagerStepBuilderFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.messaging.MessageChannel;

@Configuration
@Profile("!worker")
@RequiredArgsConstructor
public class PartitionedBatchConfig {

    private final JobRepository jobRepository;
    private final RemotePartitioningManagerStepBuilderFactory managerStepBuilderFactory;

    @Value("${app.batch.partition.grid-size:4}")
    private int gridSize;

    @Bean
    public Job partitionedBatchJob(Step managerStep) {
        return new JobBuilder("partitionedBatchJob", jobRepository)
            .start(managerStep)
            .build();
    }

    @Bean
    public Step managerStep(
            Partitioner samplePartitioner,
            MessageChannel partitionRequestChannel,
            MessageChannel partitionReplyChannel) {
        return managerStepBuilderFactory.get("managerStep")
            .partitioner("workerStep", samplePartitioner)
            .gridSize(gridSize)
            .outputChannel(partitionRequestChannel)
            .inputChannel(partitionReplyChannel)
            .build();
    }
}
```

**Note:** `@EnableBatchIntegration` must be placed on a `@Configuration` class (or the
main application class) to make `RemotePartitioningManagerStepBuilderFactory` and
`RemotePartitioningWorkerStepBuilderFactory` available for injection.

```java
package {{BASE_PACKAGE}}.config;

import org.springframework.batch.integration.config.annotation.EnableBatchIntegration;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableBatchIntegration
public class BatchIntegrationConfig {
}
```

---

## Worker Step Configuration

### WorkerBatchConfig.java

The worker step uses `RemotePartitioningWorkerStepBuilderFactory` to receive partitions
and execute the local Reader → Processor → Writer pipeline.

```java
package {{BASE_PACKAGE}}.scheduling.internal.batch.partition;

import lombok.RequiredArgsConstructor;
import org.springframework.batch.core.Step;
import org.springframework.batch.integration.partition.RemotePartitioningWorkerStepBuilderFactory;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.messaging.MessageChannel;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
@Profile("worker")
@RequiredArgsConstructor
public class WorkerBatchConfig {

    private final RemotePartitioningWorkerStepBuilderFactory workerStepBuilderFactory;
    private final PlatformTransactionManager transactionManager;

    @Bean
    public Step workerStep(
            MessageChannel partitionRequestChannel,
            MessageChannel partitionReplyChannel,
            ItemReader<SampleSourceDocument> partitionedReader,
            ItemProcessor<SampleSourceDocument, SampleTargetDocument> processor,
            ItemWriter<SampleTargetDocument> writer) {
        return workerStepBuilderFactory.get("workerStep")
            .inputChannel(partitionRequestChannel)
            .outputChannel(partitionReplyChannel)
            .<SampleSourceDocument, SampleTargetDocument>chunk(100, transactionManager)
            .reader(partitionedReader)
            .processor(processor)
            .writer(writer)
            .build();
    }
}
```

The `partitionedReader` is a `@StepScope` bean that reads `minIndex`/`maxIndex` (or
`minId`/`maxId`) from the step's `ExecutionContext` to scope its query:

```java
package {{BASE_PACKAGE}}.scheduling.internal.batch.partition;

import lombok.RequiredArgsConstructor;
import org.springframework.batch.core.configuration.annotation.StepScope;
import org.springframework.batch.item.data.MongoPagingItemReader;
import org.springframework.batch.item.data.builder.MongoPagingItemReaderBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.Sort;
import org.springframework.data.mongodb.core.MongoTemplate;
import java.util.Map;

@Configuration
@RequiredArgsConstructor
public class PartitionedReaderConfig {

    private final MongoTemplate mongoTemplate;

    @StepScope
    @Bean
    public MongoPagingItemReader<SampleSourceDocument> partitionedReader(
            @Value("#{stepExecutionContext['minIndex']}") Long minIndex,
            @Value("#{stepExecutionContext['maxIndex']}") Long maxIndex) {
        return new MongoPagingItemReaderBuilder<SampleSourceDocument>()
            .name("partitionedReader")
            .template(mongoTemplate)
            .targetType(SampleSourceDocument.class)
            .jsonQuery("{}")
            .sorts(Map.of("_id", Sort.Direction.ASC))
            .currentItemCount(minIndex.intValue())
            .maxItemCount(maxIndex.intValue() + 1)
            .pageSize(100)
            .build();
    }
}
```

---

## Quartz Integration with Partitioned Jobs

When remote partitioning is enabled, the Quartz job launches the **partitioned** batch
job instead of the simple single-step job. Update the Quartz job bean to inject the
partitioned job:

### PartitionedQuartzJob.java

```java
package {{BASE_PACKAGE}}.scheduling.internal.job;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.scheduling.quartz.QuartzJobBean;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class PartitionedQuartzJob extends QuartzJobBean {

    private final JobLauncher jobLauncher;

    @Qualifier("partitionedBatchJob")
    private final Job partitionedBatchJob;

    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        try {
            JobParameters params = new JobParametersBuilder()
                .addLong("timestamp", System.currentTimeMillis())
                .toJobParameters();

            log.info("Launching partitioned batch job at {}", params);
            jobLauncher.run(partitionedBatchJob, params);
            log.info("Partitioned batch job completed successfully");
        } catch (Exception e) {
            log.error("Failed to launch partitioned batch job", e);
            throw new JobExecutionException("Partitioned batch job launch failed", e);
        }
    }
}
```

The `QuartzConfig` registers this job with its trigger in the same manner as
`SampleQuartzJob` (see the single-process Quartz section above). Both the simple and
partitioned Quartz jobs can coexist — use separate job identities and triggers.

---

## Job Metadata Storage Note

Remote partitioning has a **hard requirement**: all manager and worker nodes must share
the same `JobRepository` database. The manager writes partition metadata (`StepExecution`
entries) and workers update them on completion. Without a shared database, the manager
cannot track partition status.

- **If Database = MongoDB**: Use a shared MongoDB instance. Spring Batch's `JobRepository`
  must be configured with a MongoDB-backed implementation or fall back to an embedded
  JDBC `JobRepository` (e.g., H2) accessible to all nodes via a shared JDBC datasource.
- **If Database = PostgreSQL/MySQL**: The shared relational database naturally satisfies
  this requirement. Use `spring.batch.jdbc.initialize-schema: always` on first run.
- **If Database = none**: Remote partitioning **cannot be used** without a shared database.
  A shared database is required. The spec must warn the user that selecting
  `Remote Partitioning = yes` with `Database = none` is an invalid combination.

---

## Updated Package Structure (with Remote Partitioning)

When Remote Partitioning is enabled, add a `partition/` subpackage:

```
scheduling/
├── SchedulingService.java
└── internal/
    ├── job/
    │   ├── SampleQuartzJob.java
    │   └── PartitionedQuartzJob.java
    └── batch/
        ├── SampleBatchConfig.java
        ├── SampleItemReader.java
        ├── SampleItemProcessor.java
        ├── SampleItemWriter.java
        └── partition/
            ├── SamplePartitioner.java
            ├── PartitionChannelConfig.java
            ├── PartitionedBatchConfig.java      # Manager step (profile: !worker)
            ├── WorkerBatchConfig.java            # Worker step (profile: worker)
            └── PartitionedReaderConfig.java      # @StepScope partitioned reader
```

---

## Operational Notes

### Worker Deployment

Workers are deployed using the **same application JAR** with a different Spring profile:

```bash
# Manager node (default — runs Quartz scheduler + web server)
java -jar app.jar --spring.profiles.active=default

# Worker node (no web server, no Quartz — only listens for partitions)
java -jar app.jar --spring.profiles.active=worker
```

Scale workers horizontally by running more instances. RabbitMQ distributes partitions
across all connected workers via competing consumers on the request queue.

### Failure and Retry Semantics

- If a worker fails mid-partition, its `StepExecution` is marked `FAILED` in the
  `JobRepository`. The manager step also fails, and the entire job can be **restarted** —
  Spring Batch only re-executes failed partitions on restart.
- Configure RabbitMQ message acknowledgment to `AUTO` (default) so messages are only
  acknowledged after the worker finishes processing.
- Set reasonable RabbitMQ consumer prefetch (`spring.rabbitmq.listener.simple.prefetch=1`)
  to prevent a single worker from hoarding all partitions.

### Dead Letter Queue (DLQ)

The request queue is configured with a DLQ (`batch.partition.requests.dlq`). Messages
that are rejected (e.g., deserialization failure, poison messages) are routed to the DLQ
for inspection rather than being lost or infinitely retried.

Monitor the DLQ in production and alert on message accumulation.

### Monitoring

- Use RabbitMQ Management UI (port `15672`) to inspect queue depths, consumer counts,
  and message rates.
- Use Spring Batch's `JobExplorer` to query partition execution status programmatically.
- Consider exposing a health indicator that checks RabbitMQ connectivity
  (`management.health.rabbit.enabled: true`).
