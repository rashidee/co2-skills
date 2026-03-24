# Async Patterns — Polling, Batch Processing, and Background Daemon

This reference covers three patterns for CLI commands that run for longer than a single
request-response cycle. Include the relevant sections in Section 17 of the generated
specification, based on which patterns are detected.

---

## Pattern Decision Guide

```
┌──────────────────────────────────────────────────────────────────────────┐
│  "How does this command work over time?"                                  │
│                                                                           │
│  Polling           — command waits for an external condition to change    │
│    e.g. deploy --watch, pipeline --wait, build --follow                  │
│    Lifecycle: start → loop(check → sleep) → done / timeout               │
│                                                                           │
│  Inline Batch      — command processes many items in one run             │
│    e.g. sync --all, import records.csv, migrate --run                    │
│    Lifecycle: load items → chunk → process → report                      │
│                                                                           │
│  Background Daemon — persistent process managed by the CLI               │
│    e.g. daemon start/stop/status, worker start, watcher start            │
│    Lifecycle: start (detach) → run loop → stop (signal)                  │
└──────────────────────────────────────────────────────────────────────────┘
```

All three patterns share two requirements:
1. **SIGINT handling** — Ctrl+C must produce a clean exit with a human-readable message,
   not a Node.js stack trace
2. **Progress visibility** — the user must see that something is happening during any
   wait longer than ~300ms

---

## Shared Utility: SIGINT Handler

Every command that runs a loop or long operation must register a SIGINT handler
before entering the loop. This is the canonical pattern used by all three async patterns.

```typescript
// src/utils/signal.ts

export type CleanupFn = () => void | Promise<void>

/**
 * Register a cleanup function to run on SIGINT (Ctrl+C) or SIGTERM.
 * Returns a disposal function that removes the listeners.
 *
 * Usage:
 *   const dispose = onSignal(async () => {
 *     spinner.stop()
 *     await db.close()
 *   })
 *   // ... do work ...
 *   dispose()  // remove listeners when done normally
 */
export function onSignal(cleanup: CleanupFn): () => void {
  let handled = false

  const handler = async (signal: NodeJS.Signals): Promise<void> => {
    if (handled) return
    handled = true

    process.stderr.write('\n') // move past the ^C characters
    try {
      await cleanup()
    } finally {
      // Use the signal's natural exit code convention
      process.exit(signal === 'SIGTERM' ? 0 : 130)
    }
  }

  process.once('SIGINT',  () => { void handler('SIGINT') })
  process.once('SIGTERM', () => { void handler('SIGTERM') })

  return () => {
    process.off('SIGINT',  handler as never)
    process.off('SIGTERM', handler as never)
  }
}
```

**Exit code 130** is the POSIX convention for "terminated by Ctrl+C" (`128 + SIGINT(2)`).
Shells and CI systems interpret this correctly as user-initiated cancellation, not a
program error.

---

## Pattern 1: Polling

### When to use

A command fires an operation (e.g. trigger a deployment, start a pipeline) and then
waits for the resulting status to reach a terminal state (`success`, `failed`, `cancelled`).
The user wants to see live status updates rather than polling manually with repeated invocations.

### Trigger flags

```
my-tool deploy --watch          # deploy and poll until done
my-tool pipeline run --wait     # start pipeline and follow it
my-tool build trigger --follow  # trigger and tail status
```

Non-watching behaviour (fire-and-forget) must still work when the flag is absent:
```
my-tool deploy                  # triggers deploy, prints job ID, exits 0 immediately
```

### `src/utils/poll.ts`

```typescript
import type { Ora } from 'ora'
import { createSpinner } from '../ui/spinner.js'
import { CliError, ExitCodes } from '../errors.js'
import { onSignal } from './signal.js'

export interface PollOptions<T> {
  /** Human-readable label shown in the spinner while polling */
  label: string
  /** Called repeatedly. Return non-null when the terminal state is reached. */
  check: () => Promise<T | null>
  /** Milliseconds between checks. Default: 3000 */
  intervalMs?: number
  /** Maximum total wait time in milliseconds. Default: 300_000 (5 min) */
  timeoutMs?: number
  /** Optional: update spinner text on each tick */
  onTick?: (spinner: Ora, elapsed: number) => void
}

export interface PollResult<T> {
  value:     T
  elapsedMs: number
}

/**
 * Poll a condition function at a fixed interval until it returns a non-null
 * value or the timeout expires. Shows an ora spinner throughout.
 *
 * Registers a SIGINT handler so Ctrl+C exits cleanly instead of throwing.
 */
export async function poll<T>(options: PollOptions<T>): Promise<PollResult<T>> {
  const {
    label,
    check,
    intervalMs = 3_000,
    timeoutMs  = 300_000,
    onTick,
  } = options

  const spinner   = createSpinner(label)
  const startedAt = Date.now()

  spinner.start()

  const disposeSignal = onSignal(() => {
    spinner.warn('Polling stopped — operation continues in the background')
  })

  try {
    while (true) {
      const elapsed = Date.now() - startedAt

      if (elapsed >= timeoutMs) {
        spinner.fail(`Timed out after ${Math.round(elapsed / 1000).toString()}s`)
        throw new CliError(
          `Polling timed out after ${Math.round(timeoutMs / 1000).toString()}s`,
          'POLL_TIMEOUT',
          ExitCodes.GENERAL_ERROR,
        )
      }

      const result = await check()

      if (result !== null) {
        spinner.succeed(`${label} — done in ${(elapsed / 1000).toFixed(1)}s`)
        return { value: result, elapsedMs: elapsed }
      }

      if (onTick) {
        onTick(spinner, elapsed)
      } else {
        spinner.text = `${label} (${Math.round(elapsed / 1000).toString()}s elapsed)`
      }

      await sleep(intervalMs)
    }
  } finally {
    disposeSignal()
    if (spinner.isSpinning) spinner.stop()
  }
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms))
}
```

### Command integration

```typescript
// src/commands/deploy/index.ts  (watch variant)
import type { Command } from 'commander'
import { handleError }  from '../../errors.js'
import { output }       from '../../ui/output.js'
import { logger }       from '../../ui/logger.js'
import { poll }         from '../../utils/poll.js'
import { DeployService } from '../../services/deploy.service.js'
import type { DeployResult } from '../../services/deploy.service.js'

export function registerDeployCommand(program: Command): void {
  program
    .command('deploy')
    .description('Trigger a deployment')
    .argument('<environment>', 'target environment')
    .option('-w, --watch',           'wait for deployment to complete', false)
    .option('--interval <seconds>',  'polling interval in seconds',     '3')
    .option('--timeout <seconds>',   'maximum wait time in seconds',    '300')
    .action(handleError(async (env: string, opts: DeployOptions) => {
      const service = new DeployService()

      // Step 1 — always: trigger the deployment
      const { deploymentId } = await service.trigger(env)
      logger.info(`Deployment started: ${deploymentId}`)

      if (!opts.watch) {
        // Fire-and-forget: print the ID and exit
        output.success({ deploymentId, environment: env }, `Deployment ${deploymentId} triggered`)
        return
      }

      // Step 2 — only when --watch: poll until terminal status
      const { value: finalStatus } = await poll<DeployResult>({
        label:      `Waiting for deployment ${deploymentId}`,
        intervalMs: Number(opts.interval) * 1000,
        timeoutMs:  Number(opts.timeout)  * 1000,
        check:      async () => {
          const status = await service.getStatus(deploymentId)
          // Return non-null only when a terminal state is reached
          if (status.status === 'success' || status.status === 'failed') {
            return status
          }
          return null  // keep polling
        },
        onTick: (spinner, elapsed) => {
          spinner.text = `Deploying to ${env} (${Math.round(elapsed / 1000).toString()}s)…`
        },
      })

      if (finalStatus.status === 'failed') {
        output.failure('DEPLOY_FAILED', `Deployment failed: ${finalStatus.error ?? 'unknown reason'}`)
        process.exit(1)
      }

      output.success(finalStatus, `Deployed to ${env}: ${finalStatus.url}`)
    }))
}

interface DeployOptions {
  watch:    boolean
  interval: string
  timeout:  string
  verbose:  boolean
  json:     boolean
}
```

### Output contract

**Human output (--watch, success):**
```
ℹ Deployment started: dep-abc123
⠸ Deploying to production (12s)…
✔ Waiting for deployment dep-abc123 — done in 14.2s
✔ Deployed to production: https://app.example.com
```

**--json output (--watch, success):**
```json
{
  "success": true,
  "data": {
    "deploymentId": "dep-abc123",
    "environment": "production",
    "status": "success",
    "url": "https://app.example.com",
    "durationMs": 14200
  }
}
```

**Ctrl+C during polling:**
```
⚠ Polling stopped — operation continues in the background
```
Exit code: `130`

**Timeout:**
```
✖ Timed out after 300s
✖ Polling timed out after 300s
```
Exit code: `1`

---

## Pattern 2: Inline Batch Processing

### When to use

A command must process a potentially large collection of items (records, files, API
resources) within a single invocation. Key requirements:

- Show live progress (items processed / total, percentage, ETA)
- Support configurable concurrency (`--concurrency N`)
- Support fail-fast vs skip-on-error mode (`--bail` vs `--continue-on-error`)
- Support resume from last checkpoint (when combined with Local Database)
- Exit code reflects partial failure (processed some, failed some)

### `src/utils/batch.ts`

```typescript
import { CliError, ExitCodes } from '../errors.js'
import { logger }               from '../ui/logger.js'
import { onSignal }             from './signal.js'

export interface BatchOptions<TInput, TOutput> {
  /** Total item count (for progress display) */
  total: number
  /** Source of items to process (async generator or array) */
  items: AsyncIterable<TInput> | TInput[]
  /** Process a single item. Throw to signal failure. */
  process: (item: TInput) => Promise<TOutput>
  /** Number of items to process concurrently. Default: 1 */
  concurrency?: number
  /** If true, stop immediately on first error. Default: false */
  bail?: boolean
  /** Called after each item completes (for progress updates) */
  onProgress?: (progress: BatchProgress) => void
}

export interface BatchProgress {
  processed: number
  succeeded: number
  failed:    number
  total:     number
  percent:   number
  etaMs:     number | null
  startedAt: Date
}

export interface BatchSummary<TOutput> {
  results:   TOutput[]
  errors:    Array<{ item: unknown; error: Error }>
  processed: number
  succeeded: number
  failed:    number
  durationMs: number
}

/**
 * Process items in configurable concurrency batches with progress tracking.
 * Registers SIGINT so Ctrl+C stops after the current chunk completes.
 */
export async function runBatch<TInput, TOutput>(
  options: BatchOptions<TInput, TOutput>,
): Promise<BatchSummary<TOutput>> {
  const { total, process: processFn, concurrency = 1, bail = false, onProgress } = options

  const results: TOutput[]                        = []
  const errors: BatchSummary<TOutput>['errors']   = []
  const startedAt = new Date()
  let processed = 0
  let aborted   = false

  const disposeSignal = onSignal(() => {
    aborted = true
    logger.warn('Batch interrupted — stopping after current chunk')
  })

  const itemsArray: TInput[] = []
  for await (const item of options.items) {
    itemsArray.push(item)
    if (aborted) break
  }

  // Process in chunks of `concurrency`
  for (let i = 0; i < itemsArray.length; i += concurrency) {
    if (aborted) break

    const chunk = itemsArray.slice(i, i + concurrency)

    const chunkResults = await Promise.allSettled(
      chunk.map(item => processFn(item))
    )

    for (let j = 0; j < chunkResults.length; j++) {
      processed++
      const result = chunkResults[j]!

      if (result.status === 'fulfilled') {
        results.push(result.value)
      } else {
        const error = result.reason instanceof Error
          ? result.reason
          : new Error(String(result.reason))
        errors.push({ item: chunk[j], error })
        logger.debug(`Item failed: ${error.message}`)

        if (bail) {
          aborted = true
          break
        }
      }

      if (onProgress) {
        const elapsed = Date.now() - startedAt.getTime()
        const rate    = processed / (elapsed / 1000)
        const etaMs   = rate > 0 ? ((total - processed) / rate) * 1000 : null

        onProgress({
          processed,
          succeeded: results.length,
          failed:    errors.length,
          total,
          percent:   Math.round((processed / total) * 100),
          etaMs,
          startedAt,
        })
      }
    }

    if (aborted && bail) break
  }

  disposeSignal()

  return {
    results,
    errors,
    processed,
    succeeded: results.length,
    failed:    errors.length,
    durationMs: Date.now() - startedAt.getTime(),
  }
}
```

### Progress bar helper

```typescript
// src/ui/progress.ts
import { createSpinner } from './spinner.js'
import type { BatchProgress } from '../utils/batch.js'
import type { Ora } from 'ora'

export class ProgressBar {
  private spinner: Ora

  constructor(label: string) {
    this.spinner = createSpinner(label)
    this.spinner.start()
  }

  update(progress: BatchProgress): void {
    const { processed, total, percent, failed, etaMs } = progress
    const etaText = etaMs !== null ? ` — ETA ${formatDuration(etaMs)}` : ''
    const failText = failed > 0 ? ` (${failed.toString()} failed)` : ''
    this.spinner.text =
      `[${percent.toString().padStart(3)}%] ${processed.toString()}/${total.toString()}${failText}${etaText}`
  }

  succeed(message: string): void { this.spinner.succeed(message) }
  fail(message: string):    void { this.spinner.fail(message) }
  stop():                   void { this.spinner.stop() }
}

function formatDuration(ms: number): string {
  if (ms < 60_000) return `${Math.round(ms / 1000).toString()}s`
  return `${Math.round(ms / 60_000).toString()}m`
}
```

### Command integration

```typescript
// src/commands/sync/index.ts
import type { Command }    from 'commander'
import { handleError }     from '../../errors.js'
import { output }          from '../../ui/output.js'
import { logger }          from '../../ui/logger.js'
import { ProgressBar }     from '../../ui/progress.js'
import { runBatch }        from '../../utils/batch.js'
import { SyncService }     from '../../services/sync.service.js'

export function registerSyncCommand(program: Command): void {
  program
    .command('sync')
    .description('Synchronise all remote resources to local cache')
    .option('-c, --concurrency <n>',  'parallel workers',         '5')
    .option('--bail',                 'stop on first error',      false)
    .option('--dry-run',              'preview without writing',  false)
    .action(handleError(async (opts: SyncOptions) => {
      const service = new SyncService()

      // Fetch the item list (IDs only — full fetch happens per-item in process())
      const items   = await service.listRemoteIds()
      const total   = items.length
      const bar     = new ProgressBar(`Syncing ${total.toString()} resources`)

      const summary = await runBatch({
        total,
        items,
        concurrency: Number(opts.concurrency),
        bail:        opts.bail,
        process: async (id) => {
          return service.syncOne(id, { dryRun: opts.dryRun })
        },
        onProgress: (p) => bar.update(p),
      })

      if (summary.failed === 0) {
        bar.succeed(`Synced ${summary.succeeded.toString()} resources in ${(summary.durationMs / 1000).toFixed(1)}s`)
      } else {
        bar.fail(`Completed with ${summary.failed.toString()} error(s)`)
      }

      output.success(
        {
          succeeded:  summary.succeeded,
          failed:     summary.failed,
          durationMs: summary.durationMs,
          errors:     summary.errors.map(e => ({ error: e.error.message })),
        },
        `Sync complete: ${summary.succeeded.toString()} succeeded, ${summary.failed.toString()} failed`,
      )

      // Non-zero exit if any items failed
      if (summary.failed > 0 && !opts.bail) {
        process.exit(1)
      }
    }))
}

interface SyncOptions {
  concurrency: string
  bail:        boolean
  dryRun:      boolean
  verbose:     boolean
  json:        boolean
}
```

### Resume support (with Local Database)

When `Local Database = yes`, the batch command should support resuming a previous
incomplete run. The checkpoint pattern stores processed item IDs in SQLite:

```typescript
// src/services/sync.service.ts  — resume-aware variant

export class SyncService {
  constructor(
    private readonly http    = getHttpClient(),
    private readonly syncRepo = new SyncRepository(getDb()),
  ) {}

  async listRemoteIds(): Promise<string[]> {
    const all       = await this.http.get('resources').json<string[]>()
    const processed = this.syncRepo.listProcessedIds()
    const processedSet = new Set(processed)

    // Filter out already-processed items to support resume
    return all.filter(id => !processedSet.has(id))
  }

  async syncOne(id: string, opts: { dryRun: boolean }): Promise<SyncRecord> {
    const resource = await this.http.get(`resources/${id}`).json<Resource>()

    if (!opts.dryRun) {
      this.syncRepo.markProcessed(id, resource)
    }

    return { id, synced: !opts.dryRun, name: resource.name }
  }
}
```

**Output contract (human):**
```
ℹ Fetching resource list…  432 items
[  0%]   0/432
[ 23%]  99/432 — ETA 18s
[100%] 432/432
✔ Synced 430 resources in 24.1s
✖ Completed with 2 error(s)
```

**--json output:**
```json
{
  "success": true,
  "data": {
    "succeeded":  430,
    "failed":     2,
    "durationMs": 24100,
    "errors": [
      { "error": "Resource res-007 not found" },
      { "error": "Timeout on res-099" }
    ]
  }
}
```

**Exit codes:**
| Scenario                  | Exit Code |
|---------------------------|-----------|
| All items succeeded       | `0`       |
| Some items failed (no --bail) | `1`   |
| --bail triggered          | `1`       |
| Ctrl+C                    | `130`     |

---

## Pattern 3: Background Daemon

### When to use

The CLI manages a persistent background process (watcher, sync agent, local proxy, job
queue worker). The daemon runs indefinitely and the CLI provides `start`/`stop`/`status`
commands to manage it.

**Architecture:**
```
my-tool daemon start   → spawns detached child (same binary, special entrypoint flag)
                          writes PID to conf store
                          returns immediately (exit 0)

my-tool daemon stop    → reads PID from conf, sends SIGTERM, waits for exit
                          removes PID from conf

my-tool daemon status  → reads PID, checks if process is alive
                          shows uptime if running
```

### Daemon entry point detection

The daemon process is the **same binary** launched with a special hidden flag. This avoids
shipping a separate binary and keeps the package surface small:

```typescript
// src/cli.ts — detect daemon mode before registering user-facing commands

if (process.env['MY_TOOL_DAEMON_MODE'] === '1') {
  // Import and run the daemon loop directly — bypass Commander entirely
  const { runDaemon } = await import('./daemon/runner.js')
  await runDaemon()
  process.exit(0)
}

// Normal mode: register user-facing commands
registerDaemonCommand(program)
// ...
```

### `src/commands/daemon/index.ts`

```typescript
import type { Command }    from 'commander'
import { handleError }     from '../../errors.js'
import { CliError }        from '../../errors.js'
import { output }          from '../../ui/output.js'
import { logger }          from '../../ui/logger.js'
import { DaemonManager }   from '../../services/daemon.service.js'

export function registerDaemonCommand(program: Command): void {
  const daemonCmd = program
    .command('daemon')
    .description('Manage the background daemon process')

  daemonCmd
    .command('start')
    .description('Start the background daemon')
    .option('--foreground', 'run in foreground (do not detach)', false)
    .action(handleError(async (opts: { foreground: boolean }) => {
      const manager = new DaemonManager()

      if (await manager.isRunning()) {
        logger.warn('Daemon is already running')
        output.success(await manager.getStatus(), 'Daemon already running')
        return
      }

      if (opts.foreground) {
        logger.info('Starting daemon in foreground (Ctrl+C to stop)')
        const { runDaemon } = await import('../../daemon/runner.js')
        await runDaemon()
        return
      }

      const pid = await manager.start()
      output.success({ pid, status: 'started' }, `Daemon started (PID ${pid.toString()})`)
    }))

  daemonCmd
    .command('stop')
    .description('Stop the background daemon')
    .option('--timeout <seconds>', 'wait timeout before force-kill', '10')
    .action(handleError(async (opts: { timeout: string }) => {
      const manager = new DaemonManager()

      if (!await manager.isRunning()) {
        logger.warn('Daemon is not running')
        return
      }

      await manager.stop(Number(opts.timeout) * 1000)
      output.success({ status: 'stopped' }, 'Daemon stopped')
    }))

  daemonCmd
    .command('status')
    .description('Show daemon status')
    .action(handleError(async () => {
      const manager = new DaemonManager()
      const status  = await manager.getStatus()
      output.success(status, status.running
        ? `Daemon running (PID ${status.pid?.toString() ?? '?'}, uptime ${formatUptime(status.uptimeMs ?? 0)})`
        : 'Daemon is not running',
      )
    }))

  daemonCmd
    .command('logs')
    .description('Show recent daemon logs')
    .option('-n, --lines <n>', 'number of lines to show', '50')
    .action(handleError(async (opts: { lines: string }) => {
      const manager = new DaemonManager()
      const lines   = await manager.getRecentLogs(Number(opts.lines))
      for (const line of lines) {
        output.raw(line)
      }
    }))
}

function formatUptime(ms: number): string {
  const s = Math.floor(ms / 1000)
  if (s < 60)   return `${s.toString()}s`
  if (s < 3600) return `${Math.floor(s / 60).toString()}m`
  return `${Math.floor(s / 3600).toString()}h ${Math.floor((s % 3600) / 60).toString()}m`
}
```

### `src/services/daemon.service.ts`

```typescript
import { spawn }          from 'node:child_process'
import { fileURLToPath }  from 'node:url'
import { dirname, join }  from 'node:path'
import { existsSync, readFileSync, writeFileSync, unlinkSync } from 'node:fs'
import { getUserConfig }  from '../config/user.config.js'
import { CliError }       from '../errors.js'

const __filename = fileURLToPath(import.meta.url)
const __dirname  = dirname(__filename)

export interface DaemonStatus {
  running:   boolean
  pid?:      number
  startedAt?: Date
  uptimeMs?:  number
}

export class DaemonManager {
  private readonly config = getUserConfig()

  async isRunning(): Promise<boolean> {
    const pid = this.config.get('daemonPid') as number | undefined
    if (!pid) return false
    try {
      // Signal 0 tests if the process exists without sending a real signal
      process.kill(pid, 0)
      return true
    } catch {
      // ESRCH = no such process; EPERM = process exists but we can't signal it
      this.clearPid()
      return false
    }
  }

  async start(): Promise<number> {
    // Spawn the same binary with the daemon mode env var set
    const binaryPath = join(__dirname, '../../dist/cli.js')

    const child = spawn(process.execPath, [binaryPath], {
      detached: true,
      stdio:    'ignore',     // detach stdio completely
      env:      {
        ...process.env,
        MY_TOOL_DAEMON_MODE: '1',
      },
    })

    child.unref()  // allow parent to exit without waiting for child

    const pid = child.pid
    if (!pid) throw new CliError('Failed to start daemon', 'DAEMON_START_FAILED')

    this.config.set('daemonPid',       pid as never)
    this.config.set('daemonStartedAt', new Date().toISOString() as never)

    return pid
  }

  async stop(timeoutMs = 10_000): Promise<void> {
    const pid = this.config.get('daemonPid') as number | undefined
    if (!pid) return

    process.kill(pid, 'SIGTERM')

    // Wait for the process to exit
    const deadline = Date.now() + timeoutMs
    while (Date.now() < deadline) {
      await sleep(200)
      if (!await this.isRunning()) {
        this.clearPid()
        return
      }
    }

    // Force-kill if SIGTERM didn't work in time
    try {
      process.kill(pid, 'SIGKILL')
    } catch {
      // Process already gone
    }
    this.clearPid()
  }

  async getStatus(): Promise<DaemonStatus> {
    const pid       = this.config.get('daemonPid') as number | undefined
    const startedAt = this.config.get('daemonStartedAt') as string | undefined
    const running   = await this.isRunning()

    if (!running || !pid) return { running: false }

    const startDate = startedAt ? new Date(startedAt) : undefined
    return {
      running:   true,
      pid,
      startedAt: startDate,
      uptimeMs:  startDate ? Date.now() - startDate.getTime() : undefined,
    }
  }

  async getRecentLogs(lines: number): Promise<string[]> {
    // Daemon writes to a log file in the OS data directory
    // (see env-paths in database-patterns.md for path resolution)
    const logPath = join(getDaemonLogPath(), 'daemon.log')
    if (!existsSync(logPath)) return []

    const content = readFileSync(logPath, 'utf-8')
    return content.split('\n').filter(Boolean).slice(-lines)
  }

  private clearPid(): void {
    this.config.delete('daemonPid' as never)
    this.config.delete('daemonStartedAt' as never)
  }
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms))
}

function getDaemonLogPath(): string {
  // Import and use the same env-paths helper from database-patterns.md
  // Placed here inline for clarity — reference src/db/path.ts in real impl
  import envPaths from 'env-paths'
  const paths = envPaths('{{BINARY_NAME}}', { suffix: '' })
  return paths.log
}
```

### `src/daemon/runner.ts` — the daemon event loop

```typescript
import { onSignal }    from '../utils/signal.js'
import { logger }      from '../ui/logger.js'

/**
 * The daemon's main loop. Called when the process is launched with
 * MY_TOOL_DAEMON_MODE=1. Runs until SIGTERM is received.
 */
export async function runDaemon(): Promise<void> {
  logger.info('Daemon starting…')

  let running = true

  const disposeSignal = onSignal(() => {
    logger.info('Daemon received stop signal, shutting down…')
    running = false
  })

  // Replace with the actual daemon work derived from NFRs
  while (running) {
    try {
      await tick()
    } catch (err: unknown) {
      logger.error(`Daemon tick error: ${err instanceof Error ? err.message : String(err)}`)
    }

    await sleep(TICK_INTERVAL_MS)
  }

  disposeSignal()
  logger.info('Daemon stopped')
}

/** One unit of daemon work — implement per NFRs */
async function tick(): Promise<void> {
  // e.g. poll an API, process a queue, sync data, run a scheduled task
  logger.debug('Daemon tick')
}

const TICK_INTERVAL_MS = 30_000

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms))
}
```

### User Config keys required for daemon

When `Background Daemon = yes`, add these keys to the `conf` schema in
`src/config/user.config.ts`:

```typescript
// Additional keys in UserConfigSchema
daemonPid:       z.number().int().positive().optional(),
daemonStartedAt: z.string().optional(),    // ISO 8601 string
```

---

## Testing Async Patterns

### Testing polling

Use `vi.useFakeTimers()` to control time without actual waits:

```typescript
// test/unit/utils/poll.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { poll } from '../../../src/utils/poll.js'

describe('poll', () => {
  beforeEach(() => { vi.useFakeTimers() })
  afterEach(()  => { vi.useRealTimers() })

  it('resolves when check returns non-null', async () => {
    let calls = 0
    const pollPromise = poll({
      label:      'Waiting',
      intervalMs: 1_000,
      check: async () => {
        calls++
        return calls >= 3 ? { status: 'success' } : null
      },
    })

    // Advance time through 3 intervals
    await vi.advanceTimersByTimeAsync(3_000)

    const result = await pollPromise
    expect(result.value).toEqual({ status: 'success' })
    expect(calls).toBe(3)
  })

  it('throws CliError on timeout', async () => {
    const pollPromise = poll({
      label:     'Waiting',
      timeoutMs: 5_000,
      check:     async () => null,
    })

    await vi.advanceTimersByTimeAsync(6_000)

    await expect(pollPromise).rejects.toThrow('Polling timed out')
  })
})
```

### Testing batch processing

```typescript
// test/unit/utils/batch.test.ts
import { describe, it, expect, vi } from 'vitest'
import { runBatch } from '../../../src/utils/batch.js'

describe('runBatch', () => {
  it('processes all items and returns results', async () => {
    const items   = [1, 2, 3, 4, 5]
    const summary = await runBatch({
      total:   items.length,
      items,
      process: async (n) => n * 2,
    })

    expect(summary.succeeded).toBe(5)
    expect(summary.failed).toBe(0)
    expect(summary.results).toEqual([2, 4, 6, 8, 10])
  })

  it('collects errors without stopping when bail = false', async () => {
    const items = [1, 2, 3]
    const summary = await runBatch({
      total: items.length,
      items,
      bail:  false,
      process: async (n) => {
        if (n === 2) throw new Error('item 2 failed')
        return n
      },
    })

    expect(summary.succeeded).toBe(2)
    expect(summary.failed).toBe(1)
    expect(summary.errors[0]?.error.message).toBe('item 2 failed')
  })

  it('stops after first error when bail = true', async () => {
    const processed: number[] = []
    await runBatch({
      total: 5,
      items: [1, 2, 3, 4, 5],
      bail:  true,
      process: async (n) => {
        processed.push(n)
        if (n === 2) throw new Error('bail!')
        return n
      },
    })

    // Should stop at or near item 2 (exact cutoff depends on concurrency chunk)
    expect(processed.length).toBeLessThan(5)
  })

  it('reports progress via onProgress callback', async () => {
    const progressCalls: number[] = []
    await runBatch({
      total: 3,
      items: [1, 2, 3],
      process: async (n) => n,
      onProgress: (p) => progressCalls.push(p.processed),
    })

    expect(progressCalls).toEqual([1, 2, 3])
  })
})
```

### Testing daemon commands (DaemonManager)

Mock `process.kill` and `getUserConfig` to avoid spawning real processes:

```typescript
// test/unit/services/daemon.service.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { DaemonManager } from '../../../src/services/daemon.service.js'

vi.mock('../../../src/config/user.config.js', () => {
  const store = new Map<string, unknown>()
  return {
    getUserConfig: () => ({
      get:    (k: string) => store.get(k),
      set:    (k: string, v: unknown) => store.set(k, v),
      delete: (k: string) => store.delete(k),
    }),
  }
})

describe('DaemonManager.isRunning', () => {
  it('returns false when no PID is stored', async () => {
    const manager = new DaemonManager()
    expect(await manager.isRunning()).toBe(false)
  })

  it('returns true when stored PID is a live process', async () => {
    const killSpy = vi.spyOn(process, 'kill').mockReturnValue(true)
    const config  = (await import('../../../src/config/user.config.js')).getUserConfig()
    config.set('daemonPid', process.pid) // use our own PID — definitely alive

    const manager = new DaemonManager()
    expect(await manager.isRunning()).toBe(true)
    killSpy.mockRestore()
  })
})
```
