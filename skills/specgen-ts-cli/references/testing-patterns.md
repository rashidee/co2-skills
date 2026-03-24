# Testing Patterns — Vitest for TypeScript CLI

This reference describes the complete testing strategy for CLI applications. Include
relevant content in Section 14 of the generated specification.

---

## Testing Architecture

```
test/
├── unit/
│   └── services/
│       ├── init.service.test.ts      ← Pure unit tests; no I/O, no process.exit
│       └── deploy.service.test.ts
├── integration/
│   └── commands/
│       ├── init.test.ts              ← Invoke Commander with controlled argv
│       └── deploy.test.ts
└── helpers/
    ├── fixtures.ts                   ← Shared test data factories
    ├── tmp.ts                        ← Temp directory utilities
    └── argv.ts                       ← argv builder helpers
```

---

## vitest.config.ts

```typescript
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    globals:      true,
    environment:  'node',
    include:      ['test/**/*.test.ts'],
    exclude:      ['node_modules', 'dist'],
    testTimeout:  10_000,
    hookTimeout:  10_000,
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov', 'html'],
      include:  ['src/**/*.ts'],
      exclude:  [
        'src/cli.ts',       // entry point — tested via integration
        'src/ui/**/*.ts',   // UI utilities — covered via integration tests
        'src/**/*.d.ts',
      ],
      thresholds: {
        lines:     80,
        functions: 80,
        branches:  70,
      },
    },
  },
})
```

---

## Service Unit Testing

Services have no terminal I/O, no `process.exit`, and no `console.log`. They are
pure TypeScript and testable in isolation.

### Constructor injection pattern (testable services)

```typescript
// src/services/deploy.service.ts
import type { Got } from 'got'
import { getHttpClient } from './http.client.js'
import { CliError } from '../errors.js'

export interface DeployOptions {
  environment: string
  dryRun:      boolean
}

export interface DeployResult {
  deploymentId: string
  environment:  string
  status:       'pending' | 'success' | 'failed'
  url:          string
}

export class DeployService {
  constructor(
    // Inject dependencies — default to real implementation; override in tests
    private readonly http: Got = getHttpClient(),
  ) {}

  async deploy(options: DeployOptions): Promise<DeployResult> {
    if (options.dryRun) {
      return {
        deploymentId: 'dry-run',
        environment:  options.environment,
        status:       'pending',
        url:          'https://staging.example.com',
      }
    }
    return this.http
      .post('deployments', { json: { environment: options.environment } })
      .json<DeployResult>()
  }
}
```

### Unit test

```typescript
// test/unit/services/deploy.service.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import type { Got } from 'got'
import { DeployService } from '../../../src/services/deploy.service.js'

// Minimal got mock — only mock what DeployService actually uses
function makeHttpMock(response: unknown): Got {
  const postMock = vi.fn().mockReturnValue({
    json: vi.fn().mockResolvedValue(response),
  })
  return { post: postMock } as unknown as Got
}

describe('DeployService', () => {
  describe('deploy', () => {
    it('returns dry-run result without calling HTTP', async () => {
      const http = makeHttpMock({})
      const service = new DeployService(http)

      const result = await service.deploy({ environment: 'staging', dryRun: true })

      expect(result.deploymentId).toBe('dry-run')
      expect(result.status).toBe('pending')
      expect(http.post).not.toHaveBeenCalled()
    })

    it('calls the deployments endpoint with correct payload', async () => {
      const mockResponse: import('../../../src/services/deploy.service.js').DeployResult = {
        deploymentId: 'dep-123',
        environment:  'production',
        status:       'success',
        url:          'https://app.example.com',
      }
      const http = makeHttpMock(mockResponse)
      const service = new DeployService(http)

      const result = await service.deploy({ environment: 'production', dryRun: false })

      expect(http.post).toHaveBeenCalledWith('deployments', {
        json: { environment: 'production' },
      })
      expect(result.deploymentId).toBe('dep-123')
    })

    it('propagates CliError from HTTP client', async () => {
      const { CliError } = await import('../../../src/errors.js')
      const http = {
        post: vi.fn().mockReturnValue({
          json: vi.fn().mockRejectedValue(CliError.networkError('Connection refused')),
        }),
      } as unknown as Got
      const service = new DeployService(http)

      await expect(service.deploy({ environment: 'staging', dryRun: false }))
        .rejects.toThrow('Connection refused')
    })
  })
})
```

---

## Command Integration Testing

Test commands by invoking Commander's `parseAsync` with a controlled `argv` array.
Mock the service layer to avoid real side effects.

### argv builder helper

```typescript
// test/helpers/argv.ts

/**
 * Build a process.argv-compatible array for Commander.
 * Prepend the required 'node' and 'script' args.
 */
export function buildArgv(...args: string[]): string[] {
  return ['node', 'cli.js', ...args]
}
```

### Capture output helper

```typescript
// test/helpers/capture.ts
import { vi } from 'vitest'

export interface CapturedOutput {
  stdout: string[]
  stderr: string[]
}

/**
 * Spy on process.stdout.write and process.stderr.write.
 * Call restore() in afterEach.
 */
export function captureOutput(): { captured: CapturedOutput; restore: () => void } {
  const captured: CapturedOutput = { stdout: [], stderr: [] }

  const stdoutSpy = vi.spyOn(process.stdout, 'write').mockImplementation((data) => {
    captured.stdout.push(String(data))
    return true
  })
  const stderrSpy = vi.spyOn(process.stderr, 'write').mockImplementation((data) => {
    captured.stderr.push(String(data))
    return true
  })

  return {
    captured,
    restore: () => {
      stdoutSpy.mockRestore()
      stderrSpy.mockRestore()
    },
  }
}
```

### Command integration test

```typescript
// test/integration/commands/deploy.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { Command } from 'commander'
import { registerDeployCommand } from '../../../src/commands/deploy/index.js'
import { DeployService }         from '../../../src/services/deploy.service.js'
import { buildArgv }             from '../../helpers/argv.js'
import { captureOutput }         from '../../helpers/capture.js'

// Mock the service module — isolate command from real HTTP
vi.mock('../../../src/services/deploy.service.js', () => ({
  DeployService: vi.fn(),
}))

describe('deploy command', () => {
  let program: Command
  let capture: ReturnType<typeof captureOutput>

  beforeEach(() => {
    program = new Command()
    // Add global options that commands expect
    program.option('--json',    '', false)
    program.option('-v, --verbose', '', false)
    registerDeployCommand(program)
    capture = captureOutput()
  })

  afterEach(() => {
    capture.restore()
    vi.clearAllMocks()
  })

  it('outputs JSON result with --json flag', async () => {
    const mockDeploy = vi.fn().mockResolvedValue({
      deploymentId: 'dep-42',
      environment:  'staging',
      status:       'success',
      url:          'https://staging.example.com',
    })
    vi.mocked(DeployService).mockImplementation(() => ({ deploy: mockDeploy }) as never)

    await program.parseAsync(buildArgv('deploy', '--json', '-e', 'staging'))

    const stdout = capture.captured.stdout.join('')
    const parsed = JSON.parse(stdout) as { success: boolean; data: { deploymentId: string } }
    expect(parsed.success).toBe(true)
    expect(parsed.data.deploymentId).toBe('dep-42')
  })

  it('exits with code 1 on network error', async () => {
    const { CliError } = await import('../../../src/errors.js')
    vi.mocked(DeployService).mockImplementation(() => ({
      deploy: vi.fn().mockRejectedValue(CliError.networkError('Timeout')),
    }) as never)

    const exitSpy = vi.spyOn(process, 'exit').mockImplementation(() => {
      throw new Error('process.exit called')
    })

    await expect(
      program.parseAsync(buildArgv('deploy', '-e', 'staging'))
    ).rejects.toThrow('process.exit called')

    expect(exitSpy).toHaveBeenCalledWith(5) // ExitCodes.NETWORK_ERROR
    exitSpy.mockRestore()
  })

  it('uses dry-run flag and does not call HTTP', async () => {
    const mockDeploy = vi.fn().mockResolvedValue({
      deploymentId: 'dry-run', environment: 'staging', status: 'pending', url: '',
    })
    vi.mocked(DeployService).mockImplementation(() => ({ deploy: mockDeploy }) as never)

    await program.parseAsync(buildArgv('deploy', '--dry-run', '-e', 'staging'))

    expect(mockDeploy).toHaveBeenCalledWith(
      expect.objectContaining({ dryRun: true })
    )
  })
})
```

---

## Configuration Testing

Use temp directories to avoid polluting the real user config directory.

### Temp directory helper

```typescript
// test/helpers/tmp.ts
import { mkdtemp, rm } from 'node:fs/promises'
import { tmpdir } from 'node:os'
import { join }   from 'node:path'
import { vi }     from 'vitest'

export interface TmpDirContext {
  dir:     string
  cleanup: () => Promise<void>
}

export async function createTmpDir(prefix = 'cli-test-'): Promise<TmpDirContext> {
  const dir = await mkdtemp(join(tmpdir(), prefix))
  return {
    dir,
    cleanup: async () => { await rm(dir, { recursive: true, force: true }) },
  }
}
```

### User config test

```typescript
// test/unit/config/user.config.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest'
import { createTmpDir, type TmpDirContext } from '../../helpers/tmp.js'

describe('getUserConfig', () => {
  let tmp: TmpDirContext
  let getUserConfig: () => ReturnType<typeof import('../../../src/config/user.config.js').getUserConfig>

  beforeEach(async () => {
    tmp = await createTmpDir()
    // Override XDG_CONFIG_HOME so conf writes to our tmp dir
    process.env['XDG_CONFIG_HOME'] = tmp.dir
    // Re-import to get a fresh conf instance pointing to tmp dir
    const mod = await import('../../../src/config/user.config.js?v=' + Date.now())
    getUserConfig = mod.getUserConfig
  })

  afterEach(async () => {
    delete process.env['XDG_CONFIG_HOME']
    await tmp.cleanup()
  })

  it('returns default value for unset key', () => {
    const config = getUserConfig()
    expect(config.get('defaultEnv')).toBe('staging')
  })

  it('persists a set value', () => {
    const config = getUserConfig()
    config.set('defaultEnv', 'production')
    expect(config.get('defaultEnv')).toBe('production')
  })

  it('resets to defaults on clear', () => {
    const config = getUserConfig()
    config.set('defaultEnv', 'production')
    config.clear()
    expect(config.get('defaultEnv')).toBe('staging')
  })
})
```

### Project config test

```typescript
// test/unit/config/project.config.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest'
import { writeFile }  from 'node:fs/promises'
import { join }       from 'node:path'
import { createTmpDir, type TmpDirContext } from '../../helpers/tmp.js'

describe('loadProjectConfig', () => {
  let tmp: TmpDirContext

  beforeEach(async () => {
    tmp = await createTmpDir()
  })

  afterEach(async () => {
    await tmp.cleanup()
  })

  it('returns defaults when no config file exists', async () => {
    const { loadProjectConfig, resetProjectConfigCache } = await import('../../../src/config/project.config.js')
    resetProjectConfigCache()
    const config = await loadProjectConfig(tmp.dir)
    expect(config.environment).toBe('staging')
  })

  it('loads and validates a .mytoolrc.json file', async () => {
    await writeFile(
      join(tmp.dir, '.{{BINARY_NAME}}rc.json'),
      JSON.stringify({ environment: 'production', region: 'eu-west-1' })
    )
    const { loadProjectConfig, resetProjectConfigCache } = await import('../../../src/config/project.config.js')
    resetProjectConfigCache()
    const config = await loadProjectConfig(tmp.dir)
    expect(config.environment).toBe('production')
    expect(config.region).toBe('eu-west-1')
  })

  it('throws CliError for invalid config', async () => {
    await writeFile(
      join(tmp.dir, '.{{BINARY_NAME}}rc.json'),
      JSON.stringify({ environment: 'invalid-env' })
    )
    const { loadProjectConfig, resetProjectConfigCache } = await import('../../../src/config/project.config.js')
    const { CliError } = await import('../../../src/errors.js')
    resetProjectConfigCache()
    await expect(loadProjectConfig(tmp.dir)).rejects.toThrow(CliError)
  })
})
```

---

## Testing with File System

For commands that read or write files, use the temp directory helper and assert on
filesystem state:

```typescript
import { readFile, access }      from 'node:fs/promises'
import { constants as fsConst }  from 'node:fs'
import { join }                  from 'node:path'

it('creates expected files in the target directory', async () => {
  const tmp = await createTmpDir()
  const service = new InitService()

  await service.execute('my-project', {
    template: 'default',
    yes: true,
    dryRun: false,
    verbose: false,
    json: false,
    targetDir: tmp.dir,  // use tmp instead of cwd
  })

  // Assert files were created
  await expect(access(join(tmp.dir, 'my-project', 'package.json'), fsConst.F_OK))
    .resolves.toBeUndefined()

  const pkgJson = JSON.parse(
    await readFile(join(tmp.dir, 'my-project', 'package.json'), 'utf-8')
  ) as { name: string }
  expect(pkgJson.name).toBe('my-project')

  await tmp.cleanup()
})
```

---

## Shell Command Testing (when Shell Execution = yes)

Mock `execa` to test commands that spawn child processes:

```typescript
import { vi } from 'vitest'

vi.mock('execa', () => ({
  execa: vi.fn().mockResolvedValue({ stdout: 'v22.0.0', stderr: '', exitCode: 0 }),
}))

it('calls git init in the new project directory', async () => {
  const { execa } = await import('execa')
  const service = new InitService()
  await service.execute('my-project', { ...defaultOpts, initGit: true })

  expect(vi.mocked(execa)).toHaveBeenCalledWith(
    'git', ['init'], expect.objectContaining({ cwd: expect.stringContaining('my-project') })
  )
})
```

---

## Test Coverage Strategy

| Layer                  | Test Type                  | Coverage Goal |
|------------------------|----------------------------|---------------|
| Services               | Unit                       | ≥ 90%         |
| Command handlers       | Integration                | ≥ 75%         |
| Config modules         | Unit                       | ≥ 85%         |
| Repositories           | Unit (in-memory DB)        | ≥ 90%         |
| `poll()` utility       | Unit (fake timers)         | ≥ 85%         |
| `runBatch()` utility   | Unit                       | ≥ 90%         |
| `DaemonManager`        | Unit (mocked process.kill) | ≥ 80%         |
| `runDaemon()` loop     | Skipped                    | N/A (event loop) |
| UI utilities           | Skipped                    | N/A (no logic)|
| Entry point `cli.ts`   | Skipped                    | N/A           |

Services are the highest-value coverage target — all business logic lives there.
Command handlers are tested at integration level — mock services, assert on output.
Repositories use an in-memory SQLite database via `createDbClient(':memory:')`.
Async utilities (`poll`, `runBatch`) are pure logic and unit-testable without CLI overhead.
`runDaemon()` is an infinite loop by design — it is excluded from coverage and verified
manually or via integration test of `daemon start` + `daemon stop`.

---

## Async Pattern Testing

### Testing `poll()` — fake timers

Avoid real `setTimeout` waits by using Vitest's fake timers. The key is advancing
time with `vi.advanceTimersByTimeAsync` inside the same tick as the `poll()` call.

```typescript
// test/unit/utils/poll.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { poll } from '../../../src/utils/poll.js'

// Suppress ora output in tests
vi.mock('ora', () => ({
  default: () => ({
    start:   () => ({ text: '', isSpinning: true }),
    succeed: vi.fn(),
    fail:    vi.fn(),
    stop:    vi.fn(),
    isSpinning: false,
  }),
}))

describe('poll()', () => {
  beforeEach(() => vi.useFakeTimers())
  afterEach(()  => vi.useRealTimers())

  it('resolves when check() returns non-null', async () => {
    let calls = 0

    const pollPromise = poll({
      label:      'Waiting for job',
      intervalMs: 1_000,
      check: async () => {
        calls++
        return calls >= 3 ? { status: 'success', url: 'https://example.com' } : null
      },
    })

    // Advance through 3 check intervals
    await vi.advanceTimersByTimeAsync(3_100)

    const result = await pollPromise
    expect(result.value).toEqual({ status: 'success', url: 'https://example.com' })
    expect(result.elapsedMs).toBeGreaterThanOrEqual(0)
    expect(calls).toBe(3)
  })

  it('throws CliError when timeoutMs is exceeded', async () => {
    const pollPromise = poll({
      label:     'Waiting',
      timeoutMs: 5_000,
      intervalMs: 1_000,
      check:     async () => null,
    })

    await vi.advanceTimersByTimeAsync(6_000)

    await expect(pollPromise).rejects.toMatchObject({
      code:    'POLL_TIMEOUT',
      message: expect.stringContaining('Timed out'),
    })
  })

  it('calls onTick with elapsed time on each interval', async () => {
    const ticks: number[] = []

    const pollPromise = poll({
      label:      'Waiting',
      intervalMs: 1_000,
      timeoutMs:  10_000,
      onTick:     (_spinner, elapsed) => ticks.push(elapsed),
      check:      async () => {
        if (ticks.length >= 2) return { done: true }
        return null
      },
    })

    await vi.advanceTimersByTimeAsync(3_000)
    await pollPromise

    expect(ticks.length).toBeGreaterThanOrEqual(2)
    // Each tick elapsed should be increasing
    expect(ticks[1]!).toBeGreaterThan(ticks[0]!)
  })
})
```

### Testing `runBatch()` — concurrency and error handling

`runBatch` is synchronous-feeling from the test's perspective — no timers needed.
Focus on concurrency behaviour, error accumulation, and bail semantics.

```typescript
// test/unit/utils/batch.test.ts
import { describe, it, expect, vi } from 'vitest'
import { runBatch } from '../../../src/utils/batch.js'

describe('runBatch()', () => {
  it('processes all items and returns results array', async () => {
    const summary = await runBatch({
      total:   5,
      items:   [1, 2, 3, 4, 5],
      process: async (n) => n * 2,
    })

    expect(summary.succeeded).toBe(5)
    expect(summary.failed).toBe(0)
    expect(summary.results).toEqual([2, 4, 6, 8, 10])
    expect(summary.durationMs).toBeGreaterThanOrEqual(0)
  })

  it('collects errors and continues when bail = false (default)', async () => {
    const summary = await runBatch({
      total: 3,
      items: [1, 2, 3],
      process: async (n) => {
        if (n === 2) throw new Error('item 2 failed')
        return n * 10
      },
    })

    expect(summary.succeeded).toBe(2)
    expect(summary.failed).toBe(1)
    expect(summary.results).toContain(10)
    expect(summary.results).toContain(30)
    expect(summary.errors[0]?.error.message).toBe('item 2 failed')
  })

  it('halts after first error when bail = true', async () => {
    const processed: number[] = []

    await runBatch({
      total: 10,
      items: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10],
      bail:  true,
      process: async (n) => {
        processed.push(n)
        if (n === 2) throw new Error('bail trigger')
        return n
      },
    })

    // With concurrency=1 (default), processing stops at or after item 2
    expect(processed.length).toBeLessThan(10)
    expect(processed).toContain(2)
    expect(processed).not.toContain(3)
  })

  it('reports progress via onProgress callback after each item', async () => {
    const snapshots: number[] = []

    await runBatch({
      total: 4,
      items: [1, 2, 3, 4],
      process: async (n) => n,
      onProgress: (p) => snapshots.push(p.processed),
    })

    expect(snapshots).toEqual([1, 2, 3, 4])
  })

  it('runs items in parallel up to concurrency limit', async () => {
    // Track the maximum number of concurrent executions
    let concurrent = 0
    let maxConcurrent = 0

    await runBatch({
      total:       6,
      items:       [1, 2, 3, 4, 5, 6],
      concurrency: 3,
      process: async (n) => {
        concurrent++
        maxConcurrent = Math.max(maxConcurrent, concurrent)
        await new Promise(r => setTimeout(r, 10))  // simulate async work
        concurrent--
        return n
      },
    })

    expect(maxConcurrent).toBeLessThanOrEqual(3)
    expect(maxConcurrent).toBeGreaterThan(1)  // actually ran in parallel
  })

  it('processes async iterables (not just arrays)', async () => {
    async function* generateItems() {
      yield 'a'
      yield 'b'
      yield 'c'
    }

    const summary = await runBatch({
      total:   3,
      items:   generateItems(),
      process: async (s) => s.toUpperCase(),
    })

    expect(summary.results).toEqual(['A', 'B', 'C'])
  })
})
```

### Testing `DaemonManager` — mocking process signals

Test `isRunning()`, `start()`, and `stop()` without spawning real processes.
Mock `process.kill` for signal-0 checks and `child_process.spawn` for start.

```typescript
// test/unit/services/daemon.service.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { DaemonManager } from '../../../src/services/daemon.service.js'

// ── conf mock — in-memory store ───────────────────────────────────────────────
const mockStore = new Map<string, unknown>()

vi.mock('../../../src/config/user.config.js', () => ({
  getUserConfig: () => ({
    get:    (k: string) => mockStore.get(k),
    set:    (k: string, v: unknown) => { mockStore.set(k, v) },
    delete: (k: string) => { mockStore.delete(k) },
  }),
}))

describe('DaemonManager', () => {
  beforeEach(() => mockStore.clear())

  describe('isRunning()', () => {
    it('returns false when no PID is stored', async () => {
      const manager = new DaemonManager()
      expect(await manager.isRunning()).toBe(false)
    })

    it('returns true when the stored PID is alive (signal-0 succeeds)', async () => {
      mockStore.set('daemonPid', 12345)
      vi.spyOn(process, 'kill').mockReturnValue(true)

      const manager = new DaemonManager()
      expect(await manager.isRunning()).toBe(true)

      vi.restoreAllMocks()
    })

    it('returns false and clears PID when signal-0 throws ESRCH (process gone)', async () => {
      mockStore.set('daemonPid', 99999)
      const killSpy = vi.spyOn(process, 'kill').mockImplementation(() => {
        const err = Object.assign(new Error('ESRCH'), { code: 'ESRCH' })
        throw err
      })

      const manager = new DaemonManager()
      expect(await manager.isRunning()).toBe(false)
      expect(mockStore.has('daemonPid')).toBe(false)  // PID was cleared

      killSpy.mockRestore()
    })
  })

  describe('stop()', () => {
    it('sends SIGTERM to the stored PID', async () => {
      mockStore.set('daemonPid', 12345)
      mockStore.set('daemonStartedAt', new Date().toISOString())

      const killSpy = vi.spyOn(process, 'kill')
        .mockReturnValueOnce(true)   // SIGTERM send
        .mockImplementation(() => {  // subsequent signal-0 checks → process gone
          const err = Object.assign(new Error('ESRCH'), { code: 'ESRCH' })
          throw err
        })

      const manager = new DaemonManager()
      await manager.stop(1_000)

      expect(killSpy).toHaveBeenCalledWith(12345, 'SIGTERM')
      expect(mockStore.has('daemonPid')).toBe(false)

      killSpy.mockRestore()
    })

    it('is a no-op when daemon is not running', async () => {
      const killSpy = vi.spyOn(process, 'kill')
      const manager = new DaemonManager()
      await manager.stop()

      expect(killSpy).not.toHaveBeenCalled()
      killSpy.mockRestore()
    })
  })

  describe('getStatus()', () => {
    it('returns running=false when no PID stored', async () => {
      vi.spyOn(process, 'kill').mockReturnValue(true)
      const manager = new DaemonManager()
      const status  = await manager.getStatus()
      expect(status.running).toBe(false)
      vi.restoreAllMocks()
    })

    it('returns uptime when daemon is running', async () => {
      const startedAt = new Date(Date.now() - 60_000)  // 1 minute ago
      mockStore.set('daemonPid', 12345)
      mockStore.set('daemonStartedAt', startedAt.toISOString())
      vi.spyOn(process, 'kill').mockReturnValue(true)

      const manager = new DaemonManager()
      const status  = await manager.getStatus()

      expect(status.running).toBe(true)
      expect(status.pid).toBe(12345)
      expect(status.uptimeMs).toBeGreaterThanOrEqual(60_000)

      vi.restoreAllMocks()
    })
  })
})
```

### Testing commands that use `poll()` or `runBatch()`

For command integration tests, mock the utility modules entirely. The utilities are
tested independently above — at the command level you only need to verify that the
correct utility is called with the right options.

```typescript
// test/integration/commands/deploy.test.ts  (polling command)
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { Command } from 'commander'
import { registerDeployCommand } from '../../../src/commands/deploy/index.js'
import { buildArgv }    from '../../helpers/argv.js'
import { captureOutput } from '../../helpers/capture.js'

// Mock poll utility — prevents real intervals in integration tests
vi.mock('../../../src/utils/poll.js', () => ({
  poll: vi.fn().mockResolvedValue({
    value:     { status: 'success', url: 'https://staging.example.com' },
    elapsedMs: 5_000,
  }),
}))

vi.mock('../../../src/services/deploy.service.js', () => ({
  DeployService: vi.fn().mockImplementation(() => ({
    trigger:   vi.fn().mockResolvedValue({ deploymentId: 'dep-abc' }),
    getStatus: vi.fn().mockResolvedValue({ status: 'success', url: 'https://staging.example.com' }),
  })),
}))

describe('deploy command — polling path', () => {
  let program: Command
  let capture: ReturnType<typeof captureOutput>

  beforeEach(() => {
    program = new Command()
    program.option('--json', '', false).option('-v, --verbose', '', false)
    registerDeployCommand(program)
    capture = captureOutput()
  })

  afterEach(() => {
    capture.restore()
    vi.clearAllMocks()
  })

  it('exits 0 and prints deployment ID without --watch', async () => {
    await program.parseAsync(buildArgv('deploy', 'staging'))
    const { poll } = await import('../../../src/utils/poll.js')
    expect(poll).not.toHaveBeenCalled()
    expect(capture.captured.stdout.join('')).toContain('dep-abc')
  })

  it('calls poll() when --watch flag is provided', async () => {
    await program.parseAsync(buildArgv('deploy', 'staging', '--watch'))
    const { poll } = await import('../../../src/utils/poll.js')
    expect(poll).toHaveBeenCalledOnce()
  })

  it('outputs JSON with --json --watch', async () => {
    await program.parseAsync(buildArgv('deploy', 'staging', '--watch', '--json'))
    const raw    = capture.captured.stdout.join('')
    const parsed = JSON.parse(raw) as { success: boolean }
    expect(parsed.success).toBe(true)
  })
})
```
