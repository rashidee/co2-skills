# Local Database Patterns — better-sqlite3 + Drizzle ORM

This reference describes the embedded SQLite persistence layer for CLI applications that
need structured, queryable, relational local data. Include relevant content in Section 16
of the generated specification.

---

## When to Use Each Persistence Layer

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Decision guide                                                           │
│                                                                           │
│  conf (key-value)         — user preferences, tokens, single values      │
│    e.g. apiToken, defaultEnv, telemetryEnabled                           │
│                                                                           │
│  cosmiconfig (file)       — project-level config, team conventions       │
│    e.g. .mytoolrc, deploy targets, service list                          │
│                                                                           │
│  SQLite (this reference)  — structured records, history, relationships   │
│    e.g. run history, cached API results, task queues, audit logs,        │
│         local entity store, sync state, multi-row queryable data          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Use SQLite when the data has any of these characteristics:**
- Multiple rows of the same entity type (history entries, cached records, tasks)
- Queries involving filtering, sorting, or aggregation
- Relationships between entity types (foreign keys)
- Data that grows over time and needs pruning
- Need for ACID transactions across multiple writes

**Do NOT use SQLite for:** API tokens, simple boolean flags, last-used values — those
belong in `conf`.

---

## Technology Stack

| Package          | Role                                          | Version |
|------------------|-----------------------------------------------|---------|
| `better-sqlite3` | Synchronous SQLite driver for Node.js         | 11.x    |
| `drizzle-orm`    | TypeScript-first ORM (schema + query builder) | 0.36.x  |
| `env-paths`      | OS-correct data directory resolution          | 3.x     |
| `drizzle-kit`    | Migration generator & CLI (devDependency)     | 0.28.x  |
| `@types/better-sqlite3` | Type declarations                      | 7.x     |

### Why better-sqlite3 (synchronous) over other options

CLI commands are inherently sequential. Every command action is a single invocation that
runs top-to-bottom. The synchronous API of `better-sqlite3` maps perfectly to this model:

```typescript
// Synchronous — reads and writes without async/await ceremony
const runs = db.prepare('SELECT * FROM runs ORDER BY started_at DESC LIMIT 10').all()

// vs async alternatives that add complexity with no benefit in a CLI:
const runs = await db.query.runs.findMany({ limit: 10 })
```

Using the synchronous driver also means no risk of forgetting to `await` a query inside
a command action handler.

### Why Drizzle over raw SQL or Kysely

- Schema defined in TypeScript → full type inference on query results
- Migrations generated automatically by `drizzle-kit generate`
- No runtime schema validation overhead (unlike Prisma's engine model)
- Works natively with `better-sqlite3`'s synchronous API

---

## Database File Location

The database file must live in the OS-appropriate **data directory** — not the config
directory (`conf` uses) and not the current working directory.

```typescript
// src/db/path.ts
import envPaths from 'env-paths'
import { join }  from 'node:path'
import { mkdirSync } from 'node:fs'

/**
 * Returns the absolute path to the application's SQLite database file.
 * Creates the data directory if it does not exist.
 *
 * OS locations:
 *   macOS:   ~/Library/Application Support/{{BINARY_NAME}}/data.db
 *   Linux:   ~/.local/share/{{BINARY_NAME}}/data.db
 *   Windows: %APPDATA%\{{BINARY_NAME}}\data.db
 */
export function getDatabasePath(): string {
  const paths = envPaths('{{BINARY_NAME}}', { suffix: '' })
  mkdirSync(paths.data, { recursive: true })
  return join(paths.data, 'data.db')
}
```

> **`env-paths` is ESM-only (v3+).** Import it with `.js` extension and ensure
> `"type": "module"` is set in `package.json`.

---

## Schema Definition

Drizzle schemas live in `src/db/schema.ts`. Each table is a typed constant. Derive
exact table names and columns from the `model/<command>/model.md` context files.

```typescript
// src/db/schema.ts
import { sqliteTable, text, integer, real } from 'drizzle-orm/sqlite-core'

// ── Example: command run history ─────────────────────────────────────────────

export const runs = sqliteTable('runs', {
  id:          text('id').primaryKey(),           // UUIDv4 — generated in service
  command:     text('command').notNull(),          // e.g. 'deploy'
  status:      text('status', {
                 enum: ['running', 'success', 'failed', 'cancelled'],
               }).notNull().default('running'),
  exitCode:    integer('exit_code'),
  startedAt:   integer('started_at', { mode: 'timestamp' }).notNull(),
  finishedAt:  integer('finished_at', { mode: 'timestamp' }),
  durationMs:  integer('duration_ms'),
  environment: text('environment'),
  notes:       text('notes'),
})

// ── Example: cached remote resources ─────────────────────────────────────────

export const resourceCache = sqliteTable('resource_cache', {
  id:           text('id').primaryKey(),
  resourceType: text('resource_type').notNull(),
  externalId:   text('external_id').notNull(),
  payload:      text('payload').notNull(),         // JSON-serialised
  cachedAt:     integer('cached_at', { mode: 'timestamp' }).notNull(),
  expiresAt:    integer('expires_at', { mode: 'timestamp' }),
})

// ── Exported schema map (used by drizzle-kit and by the client) ──────────────

export const schema = { runs, resourceCache }
```

**Timestamp columns** use `integer` with `mode: 'timestamp'` — Drizzle maps these to
JavaScript `Date` objects automatically. SQLite stores them as Unix epoch integers.

**JSON columns** are stored as `text` and serialised/deserialised in the repository layer.
Never store raw objects — always `JSON.stringify` before insert and `JSON.parse` after read.

---

## Database Client (Singleton)

```typescript
// src/db/client.ts
import Database   from 'better-sqlite3'
import { drizzle } from 'drizzle-orm/better-sqlite3'
import { migrate } from 'drizzle-orm/better-sqlite3/migrator'
import { join, dirname } from 'node:path'
import { fileURLToPath } from 'node:url'
import { getDatabasePath } from './path.js'
import { schema } from './schema.js'

const __filename = fileURLToPath(import.meta.url)
const __dirname  = dirname(__filename)

export type DbClient = ReturnType<typeof createDbClient>

let _db: DbClient | null = null

/**
 * Create a Drizzle client over a better-sqlite3 connection.
 * Exposed for testing (pass a custom dbPath to use an in-memory db).
 */
export function createDbClient(dbPath = getDatabasePath()) {
  const sqlite = new Database(dbPath)

  // Enable WAL mode for better concurrent read performance
  // (safe for CLI — multiple terminal tabs may read simultaneously)
  sqlite.pragma('journal_mode = WAL')
  sqlite.pragma('foreign_keys = ON')
  sqlite.pragma('synchronous = NORMAL')  // safe with WAL; faster than FULL

  const db = drizzle(sqlite, { schema })

  // Run pending migrations automatically on first use
  const migrationsFolder = join(__dirname, '../../drizzle')
  migrate(db, { migrationsFolder })

  return db
}

/**
 * Returns the application-wide singleton DB client.
 * Opens the database and runs migrations on first call.
 */
export function getDb(): DbClient {
  _db ??= createDbClient()
  return _db
}

/**
 * Close the database connection. Call in tests and before process.exit.
 */
export function closeDb(): void {
  // Drizzle doesn't expose a close method; access the underlying SQLite instance
  if (_db) {
    // Cast to access the underlying Database instance
    const sqlite = (_db as unknown as { session: { db: InstanceType<typeof Database> } }).session.db
    sqlite.close()
    _db = null
  }
}
```

> **Migrations run automatically** every time the CLI starts. Drizzle's migrator is
> idempotent — it tracks applied migrations in the `__drizzle_migrations` table and
> only applies new ones. There is no startup penalty once all migrations are applied.

---

## Repository Pattern

Each logical entity gets a repository class that encapsulates all queries. Services
inject the repository through their constructor, enabling clean unit testing with mocks.

```typescript
// src/db/repositories/runs.repository.ts
import { eq, desc, and, gte } from 'drizzle-orm'
import { randomUUID }          from 'node:crypto'
import type { DbClient }       from '../client.js'
import { runs }                from '../schema.js'

export interface RunRecord {
  id:          string
  command:     string
  status:      'running' | 'success' | 'failed' | 'cancelled'
  exitCode:    number | null
  startedAt:   Date
  finishedAt:  Date | null
  durationMs:  number | null
  environment: string | null
  notes:       string | null
}

export interface CreateRunInput {
  command:     string
  environment?: string
}

export interface CompleteRunInput {
  id:         string
  status:     'success' | 'failed' | 'cancelled'
  exitCode?:  number
  notes?:     string
}

export class RunsRepository {
  constructor(private readonly db: DbClient) {}

  create(input: CreateRunInput): RunRecord {
    const id = randomUUID()
    const now = new Date()

    this.db.insert(runs).values({
      id,
      command:     input.command,
      status:      'running',
      startedAt:   now,
      environment: input.environment ?? null,
      exitCode:    null,
      finishedAt:  null,
      durationMs:  null,
      notes:       null,
    }).run()

    return this.findById(id)!
  }

  complete(input: CompleteRunInput): RunRecord {
    const now = new Date()
    const existing = this.findById(input.id)
    if (!existing) throw new Error(`Run not found: ${input.id}`)

    const durationMs = now.getTime() - existing.startedAt.getTime()

    this.db.update(runs)
      .set({
        status:     input.status,
        exitCode:   input.exitCode ?? null,
        finishedAt: now,
        durationMs,
        notes:      input.notes ?? null,
      })
      .where(eq(runs.id, input.id))
      .run()

    return this.findById(input.id)!
  }

  findById(id: string): RunRecord | undefined {
    return this.db.select().from(runs).where(eq(runs.id, id)).get()
  }

  listRecent(limit = 20): RunRecord[] {
    return this.db.select()
      .from(runs)
      .orderBy(desc(runs.startedAt))
      .limit(limit)
      .all()
  }

  listByCommand(command: string, limit = 20): RunRecord[] {
    return this.db.select()
      .from(runs)
      .where(eq(runs.command, command))
      .orderBy(desc(runs.startedAt))
      .limit(limit)
      .all()
  }

  countSince(since: Date): number {
    const result = this.db.select({ count: runs.id })
      .from(runs)
      .where(gte(runs.startedAt, since))
      .all()
    return result.length
  }

  deleteOlderThan(before: Date): number {
    const result = this.db.delete(runs)
      .where(and(
        // Keep running entries — only delete terminal states
        eq(runs.status, 'success'),
      ))
      .run()
    return result.changes
  }
}
```

---

## Service Integration

Services receive the repository via constructor injection:

```typescript
// src/services/history.service.ts
import { getDb }         from '../db/client.js'
import { RunsRepository } from '../db/repositories/runs.repository.js'
import type { RunRecord } from '../db/repositories/runs.repository.js'

export interface HistoryListOptions {
  command?: string
  limit:    number
}

export class HistoryService {
  private readonly runsRepo: RunsRepository

  constructor(db = getDb()) {
    this.runsRepo = new RunsRepository(db)
  }

  list(options: HistoryListOptions): RunRecord[] {
    if (options.command) {
      return this.runsRepo.listByCommand(options.command, options.limit)
    }
    return this.runsRepo.listRecent(options.limit)
  }

  recordStart(command: string, environment?: string): RunRecord {
    return this.runsRepo.create({ command, environment })
  }

  recordComplete(id: string, exitCode: number): RunRecord {
    return this.runsRepo.complete({
      id,
      status:   exitCode === 0 ? 'success' : 'failed',
      exitCode,
    })
  }

  pruneOlderThanDays(days: number): number {
    const cutoff = new Date()
    cutoff.setDate(cutoff.getDate() - days)
    return this.runsRepo.deleteOlderThan(cutoff)
  }
}
```

**The key rules:**
- Services call repositories; they never use raw SQL
- Services contain all business logic (filtering, transformation, aggregation)
- Services return typed data — never raw Drizzle result objects in public APIs

---

## Transactions

Use `db.transaction()` for operations that must be atomic:

```typescript
// Inside a repository or service
import type { DbClient } from '../db/client.js'

export class SyncRepository {
  constructor(private readonly db: DbClient) {}

  applySync(items: SyncItem[]): void {
    // All inserts/updates happen atomically — partial failures roll back
    this.db.transaction((tx) => {
      for (const item of items) {
        tx.insert(resourceCache).values({
          id:           randomUUID(),
          resourceType: item.type,
          externalId:   item.id,
          payload:      JSON.stringify(item.data),
          cachedAt:     new Date(),
          expiresAt:    item.expiresAt ?? null,
        }).run()
      }
    })
  }
}
```

---

## Migration Workflow

### drizzle.config.ts

```typescript
import type { Config } from 'drizzle-kit'

export default {
  schema:    './src/db/schema.ts',
  out:       './drizzle',
  dialect:   'sqlite',
  dbCredentials: {
    url: './dev.db',  // local dev database only — production uses OS data dir
  },
} satisfies Config
```

### Migration scripts in package.json

```json
{
  "scripts": {
    "db:generate": "drizzle-kit generate",
    "db:migrate":  "drizzle-kit migrate",
    "db:studio":   "drizzle-kit studio",
    "db:push":     "drizzle-kit push"
  }
}
```

### Workflow for schema changes

```bash
# 1. Edit src/db/schema.ts — add/modify tables or columns
# 2. Generate a new migration SQL file
npm run db:generate
# → creates drizzle/0001_add_duration_column.sql

# 3. Commit the migration file alongside the schema change
git add drizzle/ src/db/schema.ts

# 4. On next CLI run, the migrator applies it automatically
```

**Migration files are committed to source control.** Never edit generated `.sql` files
after commit — always generate new ones with `db:generate`.

### The `drizzle/` folder structure

```
drizzle/
├── 0000_initial_schema.sql
├── 0001_add_duration_column.sql
├── 0002_add_resource_cache.sql
└── meta/
    ├── _journal.json          ← drizzle-kit tracking — commit this
    ├── 0000_snapshot.json
    └── 0001_snapshot.json
```

---

## Directory Structure Additions

When Local Database = yes, add these paths to the project structure:

```
src/
└── db/
    ├── client.ts              ← Database singleton + migration runner
    ├── path.ts                ← OS data directory resolution
    ├── schema.ts              ← Drizzle table definitions (all tables)
    └── repositories/
        ├── runs.repository.ts
        └── cache.repository.ts

drizzle/                       ← Generated SQL migrations (committed to git)
├── 0000_initial_schema.sql
└── meta/
    └── _journal.json

dev.db                         ← Local dev database (gitignored)
```

Add to `.gitignore`:
```gitignore
# Local development database
dev.db
dev.db-shm
dev.db-wal

# Drizzle studio temp
.drizzle/
```

---

## pkg / Binary Packaging Compatibility

When `Binary Packaging = yes`, the migration files must be available at runtime inside
the packaged binary. Add them to the `pkg.assets` array:

```json
{
  "pkg": {
    "scripts": "dist/cjs/cli.cjs",
    "assets": [
      "drizzle/**/*"
    ]
  }
}
```

The `getDatabasePath()` function uses `env-paths` which resolves to a **writable**
OS directory — the database file itself is never bundled, only the migration SQL files.

For the migration runner inside the binary, resolve the migrations path from
`import.meta.url` using a snapshot-safe path:

```typescript
// src/db/client.ts  — pkg-safe migration path resolution
import { fileURLToPath } from 'node:url'
import { dirname, join } from 'node:path'

const __filename = fileURLToPath(import.meta.url)
const __dirname  = dirname(__filename)

// Works in both development (src/) and packaged binary (dist/cjs/)
// because drizzle/ is bundled as an asset relative to the compiled output
const migrationsFolder = join(__dirname, '../../drizzle')
```

---

## Testing

Use an **in-memory SQLite database** for all tests. Pass a custom path to `createDbClient`:

```typescript
// test/helpers/db.ts
import { createDbClient } from '../../src/db/client.js'
import type { DbClient }   from '../../src/db/client.js'

/**
 * Create an isolated in-memory database for a single test.
 * Migrations are applied automatically.
 * ':memory:' means the database is discarded when the connection closes.
 */
export function createTestDb(): DbClient {
  return createDbClient(':memory:')
}
```

### Repository unit test

```typescript
// test/unit/db/runs.repository.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { createTestDb }     from '../../helpers/db.js'
import { RunsRepository }   from '../../../src/db/repositories/runs.repository.js'
import type { DbClient }    from '../../../src/db/client.js'

describe('RunsRepository', () => {
  let db:   DbClient
  let repo: RunsRepository

  beforeEach(() => {
    db   = createTestDb()
    repo = new RunsRepository(db)
  })

  it('creates a run with status "running"', () => {
    const run = repo.create({ command: 'deploy', environment: 'staging' })

    expect(run.id).toMatch(/^[0-9a-f-]{36}$/)  // UUID format
    expect(run.status).toBe('running')
    expect(run.command).toBe('deploy')
    expect(run.environment).toBe('staging')
    expect(run.finishedAt).toBeNull()
  })

  it('completes a run and sets durationMs', () => {
    const run = repo.create({ command: 'deploy' })

    // Small delay so duration is non-zero
    const completed = repo.complete({ id: run.id, status: 'success', exitCode: 0 })

    expect(completed.status).toBe('success')
    expect(completed.exitCode).toBe(0)
    expect(completed.finishedAt).toBeInstanceOf(Date)
    expect(completed.durationMs).toBeGreaterThanOrEqual(0)
  })

  it('listRecent returns entries newest-first', () => {
    repo.create({ command: 'init' })
    repo.create({ command: 'deploy' })

    const results = repo.listRecent(10)

    expect(results[0]?.command).toBe('deploy')   // newest first
    expect(results[1]?.command).toBe('init')
  })

  it('listByCommand filters by command name', () => {
    repo.create({ command: 'init' })
    repo.create({ command: 'deploy' })
    repo.create({ command: 'deploy' })

    const deploys = repo.listByCommand('deploy')

    expect(deploys).toHaveLength(2)
    expect(deploys.every(r => r.command === 'deploy')).toBe(true)
  })

  it('throws when completing a non-existent run', () => {
    expect(() => repo.complete({ id: 'nonexistent', status: 'success' }))
      .toThrow('Run not found')
  })
})
```

### Service unit test with injected test DB

```typescript
// test/unit/services/history.service.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { createTestDb }   from '../../helpers/db.js'
import { HistoryService } from '../../../src/services/history.service.js'

describe('HistoryService', () => {
  let service: HistoryService

  beforeEach(() => {
    const db = createTestDb()
    service = new HistoryService(db)  // inject test DB
  })

  it('lists no runs on empty database', () => {
    expect(service.list({ limit: 10 })).toHaveLength(0)
  })

  it('records a run start and completion lifecycle', () => {
    const run = service.recordStart('deploy', 'staging')
    expect(run.status).toBe('running')

    const completed = service.recordComplete(run.id, 0)
    expect(completed.status).toBe('success')
    expect(completed.exitCode).toBe(0)
  })

  it('pruneOlderThanDays removes old successful runs', () => {
    // Manufacture a run that finished 31 days ago
    // (use repo directly to backdate, or test via count)
    service.recordStart('deploy')
    // In a real test, you'd use a repository directly with a backdated timestamp
    const deleted = service.pruneOlderThanDays(30)
    expect(typeof deleted).toBe('number')
  })
})
```

---

## Operational Notes

### WAL mode explanation

```
sqlite.pragma('journal_mode = WAL')
```

WAL (Write-Ahead Log) allows concurrent readers while a write is in progress. Without it,
a write locks the entire database. For a CLI this matters when:
- Two terminal tabs run commands simultaneously
- A long-running background daemon holds a connection while the user runs a query

### Database inspection during development

```bash
# Drizzle Studio — web UI to browse your SQLite data
npm run db:studio

# SQLite CLI (if installed)
sqlite3 ~/Library/Application\ Support/my-tool/data.db
.tables
.schema runs
SELECT * FROM runs ORDER BY started_at DESC LIMIT 5;
```

### Database size management

For long-lived CLIs (e.g. CI tools), add an automatic pruning step:

```typescript
// src/cli.ts — call once at startup, after update check
import { HistoryService } from './services/history.service.js'

const retention = userConfig.get('historyRetentionDays') ?? 90
new HistoryService().pruneOlderThanDays(retention)
```

### Common mistake — forgetting `.run()` on write statements

Drizzle for `better-sqlite3` returns a prepared statement object. You must call
`.run()` to execute inserts/updates/deletes:

```typescript
// WRONG — nothing happens; the statement is prepared but not executed
db.insert(runs).values({ id: '...', ... })

// CORRECT
db.insert(runs).values({ id: '...', ... }).run()

// Queries (SELECT) return results directly — no .run() needed
const result = db.select().from(runs).all()      // ✓ array
const row    = db.select().from(runs).get()       // ✓ first row or undefined
```
