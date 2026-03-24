# Specification Template — TypeScript CLI Application

This is the authoritative template for the generated specification. The specification is
split into **two types of files**:

1. **`SPECIFICATION.md`** (root) — Table of Contents, shared infrastructure, and all
   application-level sections. Generated once per application.
2. **`<command-name>/SPEC.md`** (per-command) — Self-contained command blueprint.
   Generated once per command from PRD.md.

```
specification/
├── SPECIFICATION.md
├── init/
│   └── SPEC.md
├── deploy/
│   └── SPEC.md
└── ...
```

Placeholders use `{{VARIABLE}}` syntax and must be replaced with actual values.

---

# Part A: Root SPECIFICATION.md

---

## Table of Contents

Generate a TOC with anchor links. Include a **Commands** section linking to each
command's `SPEC.md`:

```markdown
## Table of Contents

### Shared Infrastructure
- [1. Project Overview](#1-project-overview)
- [2. package.json Configuration](#2-packagejson-configuration)
- ...

### Commands
- [init](init/SPEC.md)
- [deploy](deploy/SPEC.md)
- [config](config/SPEC.md)
```

---

## Section 1: Project Overview

```
# {{APPLICATION_NAME}} — Technical Specification

## 1. Project Overview

**Application Name**: {{APPLICATION_NAME}}
**Package Name**:     {{PACKAGE_NAME}}        (e.g., @scope/my-tool or my-tool)
**Binary Name**:      {{BINARY_NAME}}          (the executable command, e.g., my-tool)
**Node.js Version**:  22.x LTS (minimum)
**TypeScript**:       5.x
**Description**:      {{APP_DESCRIPTION}}
**Versions Covered**: v0.1.0 — v{{LATEST_VERSION}}

### Optional Components (Auto-Determined)
**Prompts**:           {{yes|no}}
**User Config**:       {{yes|no}}
**Project Config**:    {{yes|no}}
**Shell Execution**:   {{yes|no}}
**HTTP Client**:       {{yes|no}}
**Auto-update**:       {{yes|no}}
**Binary Packaging**:  {{yes|no}}
**Table Output**:      {{yes|no}}

### Technology Stack
(Render core stack table from SKILL.md, plus selected optional versions)

### Commands
| Command  | Description                | Aliases | Version |
|----------|----------------------------|---------|---------|
| init     | Initialize a new project   | i       | v0.1.0  |
| deploy   | Deploy to environment      | d       | v0.1.0  |
| config   | Manage user configuration  |         | v0.1.0  |

### Distribution Strategy
**npm package**: `npm install -g {{PACKAGE_NAME}}`
[If Binary Packaging = yes] **Standalone binaries**: GitHub Releases — Windows, macOS, Linux
```

---

## Section 2: package.json Configuration

Generate a complete `package.json` inside a fenced code block. Must be copy-pasteable.

```json
{
  "name": "{{PACKAGE_NAME}}",
  "version": "0.1.0",
  "description": "{{APP_DESCRIPTION}}",
  "type": "module",
  "license": "MIT",
  "engines": {
    "node": ">=22.0.0"
  },
  "bin": {
    "{{BINARY_NAME}}": "./dist/cli.js"
  },
  "files": [
    "dist",
    "README.md"
  ],
  "scripts": {
    "dev":        "tsx watch src/cli.ts",
    "build":      "tsup",
    "typecheck":  "tsc --noEmit",
    "test":       "vitest run",
    "test:watch": "vitest",
    "lint":       "eslint src --ext .ts",
    "prepublishOnly": "npm run build && npm run typecheck && npm run test"
  },
  "dependencies": {
    "commander": "^12.0.0",
    "chalk":     "^5.0.0",
    "ora":       "^8.0.0",
    "zod":       "^3.0.0"
    // [conditional deps listed per section below]
  },
  "devDependencies": {
    "typescript":                 "^5.0.0",
    "tsup":                       "^8.0.0",
    "vitest":                     "^2.0.0",
    "@types/node":                "^22.0.0",
    "tsx":                        "^4.0.0",
    "eslint":                     "^9.0.0",
    "@typescript-eslint/parser":  "^8.0.0",
    "@typescript-eslint/eslint-plugin": "^8.0.0"
  }
}
```

**[If Prompts = yes]** add to `dependencies`:
```json
"@inquirer/prompts": "^7.0.0"
```

**[If User Config = yes]** add to `dependencies`:
```json
"conf": "^13.0.0"
```

**[If Project Config = yes]** add to `dependencies`:
```json
"cosmiconfig": "^9.0.0"
```

**[If Shell Execution = yes]** add to `dependencies`:
```json
"execa": "^9.0.0"
```

**[If HTTP Client = yes]** add to `dependencies`:
```json
"got": "^14.0.0"
```

**[If Auto-update = yes]** add to `dependencies`:
```json
"update-notifier": "^7.0.0"
```

**[If Table Output = yes]** add to `dependencies`:
```json
"cli-table3": "^0.6.0",
"@types/cli-table3": "^0.6.0"
```

**[If Local Database = yes]** add to `dependencies` and `devDependencies`:
```json
// dependencies:
"better-sqlite3": "^11.0.0",
"drizzle-orm":    "^0.36.0",
"env-paths":      "^3.0.0",

// devDependencies:
"@types/better-sqlite3": "^7.0.0",
"drizzle-kit":           "^0.28.0"
```

And add scripts:
```json
"db:generate": "drizzle-kit generate",
"db:migrate":  "drizzle-kit migrate",
"db:studio":   "drizzle-kit studio"
```

**[If Binary Packaging = yes]** add to `devDependencies` and scripts:
```json
// devDependencies:
"@yao-pkg/pkg": "^5.0.0"

// scripts:
"pkg:linux":   "pkg . --target node22-linux-x64   --output dist/bin/{{BINARY_NAME}}-linux",
"pkg:macos":   "pkg . --target node22-macos-x64   --output dist/bin/{{BINARY_NAME}}-macos",
"pkg:windows": "pkg . --target node22-win-x64     --output dist/bin/{{BINARY_NAME}}.exe",
"pkg:all":     "npm run pkg:linux && npm run pkg:macos && npm run pkg:windows"
```

---

## Section 3: TypeScript & Build Configuration

### 3.1 tsconfig.json

```json
{
  "compilerOptions": {
    "target":           "ES2022",
    "module":           "NodeNext",
    "moduleResolution": "NodeNext",
    "lib":              ["ES2022"],
    "outDir":           "./dist",
    "rootDir":          "./src",
    "strict":           true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "declaration":      true,
    "declarationMap":   true,
    "sourceMap":        true,
    "skipLibCheck":     true,
    "esModuleInterop":  false,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### 3.2 tsup.config.ts

```typescript
import { defineConfig } from 'tsup'

export default defineConfig({
  entry: ['src/cli.ts'],
  format: ['esm'],
  target: 'node22',
  platform: 'node',
  clean: true,
  sourcemap: process.env.NODE_ENV !== 'production',
  minify: false,
  shims: false,
  dts: false,
  banner: {
    js: '#!/usr/bin/env node',
  },
  esbuildOptions(options) {
    options.mainFields = ['module', 'main']
  },
})
```

### 3.3 .eslintrc.json

```json
{
  "root": true,
  "parser": "@typescript-eslint/parser",
  "parserOptions": { "project": "./tsconfig.json" },
  "plugins": ["@typescript-eslint"],
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/strict-type-checked"
  ],
  "rules": {
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-floating-promises": "error",
    "no-console": ["error", { "allow": [] }]
  }
}
```

### 3.4 .gitignore

```gitignore
# Build output
dist/
*.d.ts

# Binaries
dist/bin/

# Dependencies
node_modules/

# Testing
coverage/

# Environment
.env
.env.*

# OS
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
*.iml

# Logs
*.log
logs/

# Local development database  [If Local Database = yes]
dev.db
dev.db-shm
dev.db-wal
.drizzle/
```

---

## Section 4: Application Entry Point

`src/cli.ts` is the Commander.js program root. It must:
1. Inject the shebang (`#!/usr/bin/env node`) via tsup banner
2. Optionally check for updates (non-blocking)
3. Create the root `Command` with name, description, version
4. Register global options shared by all commands
5. Import and register each command
6. Call `program.parse()`

```typescript
// src/cli.ts
import { Command } from 'commander'
import { createRequire } from 'node:module'
import { fileURLToPath } from 'node:url'
import { dirname, join } from 'node:path'
import { readFileSync } from 'node:fs'
import { registerInitCommand }   from './commands/init/index.js'
import { registerDeployCommand } from './commands/deploy/index.js'
import { registerConfigCommand } from './commands/config/index.js'
// [If Auto-update = yes]:
import { checkForUpdates } from './utils/updater.js'

const __filename = fileURLToPath(import.meta.url)
const __dirname  = dirname(__filename)

const pkg = JSON.parse(
  readFileSync(join(__dirname, '../package.json'), 'utf-8')
) as { name: string; version: string; description: string }

// [If Auto-update = yes]: Fire and forget — never blocks the command
void checkForUpdates(pkg.name, pkg.version)

const program = new Command()

program
  .name('{{BINARY_NAME}}')
  .description(pkg.description)
  .version(pkg.version, '-V, --version', 'output the current version')
  .option('-v, --verbose', 'enable verbose output', false)
  .option('--json',        'output results as JSON', false)
  .option('--no-color',    'disable color output')
  .addHelpCommand(true)

registerInitCommand(program)
registerDeployCommand(program)
registerConfigCommand(program)

program.parseAsync(process.argv).catch((err: unknown) => {
  // Top-level fallback — most errors are caught inside commands
  console.error(String(err))
  process.exit(1)
})
```

**[If Background Daemon = yes]** insert the daemon detection block immediately before
`const program = new Command()`:

```typescript
// Must appear BEFORE Commander initialises — invisible to --help output
if (process.env['{{BINARY_NAME_UPPER}}_DAEMON_MODE'] === '1') {
  const { runDaemon } = await import('./daemon/runner.js')
  await runDaemon()
  process.exit(0)
}
```

> See `references/command-patterns.md` for the complete Commander.js patterns: root
> program setup, global option propagation (`getGlobalOptions`), sub-command groups,
> `--dry-run`, confirmation prompts, tabular output, help text conventions, exit code
> table, and the `handleError` async wrapper used in every action handler.

---

## Section 5: Project Directory Structure

```
{{BINARY_NAME}}/
├── src/
│   ├── cli.ts                      ← Program entry point
│   ├── types/
│   │   └── index.ts                ← Shared TypeScript types
│   ├── errors.ts                   ← CliError class + handleError()
│   ├── commands/
│   │   ├── init/
│   │   │   ├── index.ts            ← registerInitCommand(program)
│   │   │   └── prompts.ts          ← [If Prompts] Inquirer prompt flows
│   │   ├── deploy/
│   │   │   ├── index.ts
│   │   │   └── prompts.ts
│   │   └── config/                 ← [If User Config] config sub-commands
│   │       └── index.ts
│   ├── services/
│   │   ├── init.service.ts         ← InitService (no CLI I/O)
│   │   ├── deploy.service.ts       ← DeployService
│   │   └── config.service.ts       ← [If User Config] ConfigService
│   ├── config/
│   │   ├── user.config.ts          ← [If User Config] conf setup
│   │   └── project.config.ts       ← [If Project Config] cosmiconfig loader
│   ├── services/
│   │   └── http.client.ts          ← [If HTTP Client] got instance
│   ├── db/                         ← [If Local Database]
│   │   ├── client.ts               ← better-sqlite3 singleton + migration runner
│   │   ├── path.ts                 ← OS data directory via env-paths
│   │   ├── schema.ts               ← Drizzle table definitions (all entities)
│   │   └── repositories/
│   │       └── <entity>.repository.ts  ← One file per entity type
│   ├── utils/
│   │   ├── signal.ts               ← [If any Async] SIGINT/SIGTERM handler
│   │   ├── poll.ts                 ← [If Polling] polling loop utility
│   │   ├── batch.ts                ← [If Inline Batch] chunk processor
│   │   ├── shell.ts                ← [If Shell Execution] execa wrapper
│   │   └── updater.ts              ← [If Auto-update] update-notifier
│   ├── daemon/
│   │   └── runner.ts               ← [If Background Daemon] daemon event loop
│   └── ui/
│       ├── logger.ts               ← chalk-based logger
│       ├── spinner.ts              ← ora wrapper
│       ├── progress.ts             ← [If Inline Batch] ProgressBar (ora + percentage)
│       ├── table.ts                ← [If Table Output] cli-table3 wrapper
│       └── output.ts               ← Unified output (human + --json)
├── test/
│   ├── commands/
│   │   ├── init.test.ts
│   │   └── deploy.test.ts
│   └── services/
│       ├── init.service.test.ts
│       └── deploy.service.test.ts
├── tsconfig.json
├── tsup.config.ts
├── vitest.config.ts
├── drizzle.config.ts               ← [If Local Database]
├── package.json
├── .eslintrc.json
├── .gitignore
└── README.md

drizzle/                            ← [If Local Database] Generated SQL migrations
├── 0000_initial_schema.sql
└── meta/
    └── _journal.json
```

Use actual command names from PRD.md throughout — never generic placeholders like
`command1`, `command2`.

---

## Section 6: Shared Types

```typescript
// src/types/index.ts

/** Global options available on every command */
export interface GlobalOptions {
  verbose: boolean
  json:    boolean
  color:   boolean
}

/** Standard success result shape for --json output */
export interface CliResult<T = unknown> {
  success: true
  data:    T
}

/** Standard error result shape for --json output */
export interface CliErrorResult {
  success: false
  error: {
    code:    string
    message: string
  }
}

export type CliOutput<T = unknown> = CliResult<T> | CliErrorResult

// Add application-specific shared types from MODEL.md here
// e.g.:
// export interface ProjectConfig { ... }
// export interface DeployTarget  { name: string; url: string }
```

---

## Section 7: Terminal UI Utilities

### 7.1 Logger

```typescript
// src/ui/logger.ts
import chalk from 'chalk'

let verboseEnabled = false

export const logger = {
  enableVerbose(): void { verboseEnabled = true },

  info(message: string): void {
    process.stdout.write(chalk.blue('ℹ') + ' ' + message + '\n')
  },

  success(message: string): void {
    process.stdout.write(chalk.green('✔') + ' ' + message + '\n')
  },

  warn(message: string): void {
    process.stderr.write(chalk.yellow('⚠') + ' ' + chalk.yellow(message) + '\n')
  },

  error(message: string): void {
    process.stderr.write(chalk.red('✖') + ' ' + chalk.red(message) + '\n')
  },

  debug(message: string): void {
    if (!verboseEnabled) return
    process.stderr.write(chalk.gray('[debug] ') + chalk.gray(message) + '\n')
  },

  /** Print a section header */
  heading(message: string): void {
    process.stdout.write('\n' + chalk.bold(message) + '\n')
  },
}
```

### 7.2 Spinner

```typescript
// src/ui/spinner.ts
import ora, { type Ora } from 'ora'

export function createSpinner(text: string): Ora {
  return ora({ text, color: 'cyan' })
}

export async function withSpinner<T>(
  text: string,
  fn: () => Promise<T>,
  successText?: string,
): Promise<T> {
  const spinner = createSpinner(text)
  spinner.start()
  try {
    const result = await fn()
    spinner.succeed(successText ?? text)
    return result
  } catch (error) {
    spinner.fail()
    throw error
  }
}
```

### 7.3 Output Handler

```typescript
// src/ui/output.ts
import type { CliOutput } from '../types/index.js'
import { logger } from './logger.js'

let jsonMode = false

export const output = {
  enableJson(): void { jsonMode = true },

  /** Print a success result — JSON if --json, formatted otherwise */
  success<T>(data: T, humanSummary: string): void {
    if (jsonMode) {
      process.stdout.write(JSON.stringify({ success: true, data } satisfies CliOutput<T>, null, 2) + '\n')
    } else {
      logger.success(humanSummary)
    }
  },

  /** Print an error result — JSON if --json, formatted otherwise */
  failure(code: string, message: string): void {
    if (jsonMode) {
      process.stdout.write(JSON.stringify({ success: false, error: { code, message } } satisfies CliOutput, null, 2) + '\n')
    } else {
      logger.error(message)
    }
  },

  /** Print raw human-readable text (not affected by --json) */
  raw(text: string): void {
    process.stdout.write(text + '\n')
  },
}
```

### 7.4 Table Output *(conditional — include only if Table Output = yes)*

```typescript
// src/ui/table.ts
import Table from 'cli-table3'

export interface TableOptions {
  head:    string[]
  rows:    string[][]
  compact?: boolean
}

export function printTable({ head, rows, compact = false }: TableOptions): void {
  const table = new Table({
    head: head.map(h => `\u001b[1m${h}\u001b[0m`), // bold headers
    style: compact ? { compact: true } : {},
  })
  for (const row of rows) {
    table.push(row)
  }
  process.stdout.write(table.toString() + '\n')
}
```

---

## Section 8: Error Handling

```typescript
// src/errors.ts
import { output } from './ui/output.js'
import { logger } from './ui/logger.js'

export const ExitCodes = {
  SUCCESS:        0,
  GENERAL_ERROR:  1,
  INVALID_INPUT:  2,
  NOT_FOUND:      3,
  UNAUTHORIZED:   4,
  NETWORK_ERROR:  5,
  CONFIG_ERROR:   6,
} as const

export type ExitCode = (typeof ExitCodes)[keyof typeof ExitCodes]

export class CliError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly exitCode: ExitCode = ExitCodes.GENERAL_ERROR,
  ) {
    super(message)
    this.name = 'CliError'
  }

  static invalidInput(message: string): CliError {
    return new CliError(message, 'INVALID_INPUT', ExitCodes.INVALID_INPUT)
  }

  static notFound(resource: string): CliError {
    return new CliError(`${resource} not found`, 'NOT_FOUND', ExitCodes.NOT_FOUND)
  }

  static unauthorized(message = 'Not authenticated. Run `{{BINARY_NAME}} config login`.'): CliError {
    return new CliError(message, 'UNAUTHORIZED', ExitCodes.UNAUTHORIZED)
  }

  static networkError(message: string): CliError {
    return new CliError(message, 'NETWORK_ERROR', ExitCodes.NETWORK_ERROR)
  }

  static configError(message: string): CliError {
    return new CliError(message, 'CONFIG_ERROR', ExitCodes.CONFIG_ERROR)
  }
}

/**
 * Wrap a command action handler to catch and format all errors.
 * Usage: program.command('deploy').action(handleError(async (opts) => { ... }))
 */
export function handleError<TArgs extends unknown[]>(
  fn: (...args: TArgs) => Promise<void>,
): (...args: TArgs) => Promise<void> {
  return async (...args: TArgs): Promise<void> => {
    try {
      await fn(...args)
    } catch (err: unknown) {
      if (err instanceof CliError) {
        output.failure(err.code, err.message)
        process.exit(err.exitCode)
      }
      const message = err instanceof Error ? err.message : String(err)
      output.failure('UNKNOWN_ERROR', message)
      logger.debug(err instanceof Error ? (err.stack ?? message) : message)
      process.exit(ExitCodes.GENERAL_ERROR)
    }
  }
}
```

---

## Section 9: User Configuration Management *(conditional — include only if User Config = yes)*

See `references/config-patterns.md` for the complete implementation. This section must
include:

- `src/config/user.config.ts` — `conf` instance with typed schema (Zod), typed getter/setter
  accessors, and key migration logic
- `src/commands/config/index.ts` — `config get`, `config set`, `config list`, `config reset`
  sub-commands using `commander`'s `.addCommand()` chain
- Complete code sample for all four sub-commands
- All config key names, types, and defaults extracted from MODEL.md

---

## Section 10: Project Configuration Management *(conditional — include only if Project Config = yes)*

See `references/config-patterns.md` for the complete implementation. This section must
include:

- `src/config/project.config.ts` — cosmiconfig loader with named search path
  (`{{BINARY_NAME}}rc`, `{{BINARY_NAME}}.config.js`, `package.json#{{BINARY_NAME}}`)
- Zod schema for project config validation
- Precedence merge function: CLI flags > env vars > project config > defaults
- `loadProjectConfig()` exported function used by commands that need project context

---

## Section 11: HTTP Client Setup *(conditional — include only if HTTP Client = yes)*

```typescript
// src/services/http.client.ts
import got, { type Got, HTTPError } from 'got'
import { getUserConfig } from '../config/user.config.js'
import { CliError } from '../errors.js'

export function createHttpClient(baseUrl?: string): Got {
  const config = getUserConfig()
  const token = config.get('apiToken')

  return got.extend({
    prefixUrl: baseUrl ?? config.get('apiBaseUrl'),
    headers: {
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      'User-Agent': '{{PACKAGE_NAME}}/{{VERSION}}',
      'Content-Type': 'application/json',
      Accept: 'application/json',
    },
    retry: {
      limit: 2,
      methods: ['GET'],
      statusCodes: [408, 429, 500, 502, 503, 504],
    },
    timeout: {
      request: 30_000,
    },
    hooks: {
      beforeError: [
        (error) => {
          if (error instanceof HTTPError) {
            const { statusCode, body } = error.response
            const message = typeof body === 'string'
              ? (JSON.parse(body) as { message?: string }).message ?? body
              : 'HTTP request failed'
            if (statusCode === 401) throw CliError.unauthorized()
            if (statusCode === 404) throw CliError.notFound('Resource')
            throw CliError.networkError(`HTTP ${statusCode.toString()}: ${message}`)
          }
          return error
        },
      ],
    },
  })
}

/** Singleton HTTP client — initialized on first call */
let _client: Got | null = null

export function getHttpClient(): Got {
  _client ??= createHttpClient()
  return _client
}
```

---

## Section 12: Shell Execution Utilities *(conditional — include only if Shell Execution = yes)*

```typescript
// src/utils/shell.ts
import { execa, type Options as ExecaOptions } from 'execa'
import { logger } from '../ui/logger.js'
import { CliError } from '../errors.js'

export interface RunOptions extends ExecaOptions {
  /** If true, errors are logged but not thrown */
  ignoreErrors?: boolean
  /** Descriptive label for logging */
  label?: string
}

export async function run(
  command: string,
  args: readonly string[] = [],
  options: RunOptions = {},
): Promise<string> {
  const { ignoreErrors = false, label, ...execaOptions } = options
  const displayCmd = [command, ...args].join(' ')

  logger.debug(`${label ?? 'run'}: ${displayCmd}`)

  try {
    const result = await execa(command, args, {
      ...execaOptions,
      all: true,
    })
    return result.stdout
  } catch (err: unknown) {
    if (ignoreErrors) {
      logger.debug(`Command failed (ignored): ${displayCmd}`)
      return ''
    }
    const message = err instanceof Error ? err.message : String(err)
    throw new CliError(
      `Command failed: ${displayCmd}\n${message}`,
      'SHELL_ERROR',
    )
  }
}
```

---

## Section 13: Auto-Update Notifier *(conditional — include only if Auto-update = yes)*

```typescript
// src/utils/updater.ts
import updateNotifier from 'update-notifier'

/**
 * Non-blocking update check. Shows a notification on the NEXT run after
 * an update is detected (update-notifier caches the check result for 24h).
 */
export function checkForUpdates(packageName: string, currentVersion: string): void {
  // update-notifier is CommonJS — use createRequire for ESM compat
  const notifier = updateNotifier({
    pkg: { name: packageName, version: currentVersion },
    updateCheckInterval: 1000 * 60 * 60 * 24, // 24 hours
  })
  notifier.notify({ isGlobal: true })
}
```

---

## Section 14: Testing Strategy

See `references/testing-patterns.md` for the full testing patterns.

### vitest.config.ts

```typescript
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    globals:     true,
    environment: 'node',
    include:     ['test/**/*.test.ts'],
    coverage: {
      provider: 'v8',
      include:  ['src/**/*.ts'],
      exclude:  ['src/cli.ts', 'src/ui/**', 'src/**/*.d.ts'],
    },
  },
})
```

### Testing Approach

- **Service tests**: Pure unit tests — no file system, no network, no process.exit
  Inject all dependencies via constructor; mock with `vi.fn()`
- **Command tests**: Integration-style — invoke Commander's `parseAsync` with
  controlled `argv`, assert on `output.success`/`output.failure` calls
- **Config tests**: Use `tmp` directories; reset conf store in `beforeEach`

---

## Section 15: Packaging & Distribution *(conditional — include only if Binary Packaging = yes)*

See `references/packaging-patterns.md` for the full packaging guide. This section must
include:

- `pkg` field in `package.json` with scripts, assets, and targets
- Platform-specific npm scripts (`pkg:linux`, `pkg:macos`, `pkg:windows`, `pkg:all`)
- GitHub Actions workflow for automated release (triggered on `v*` tag)
- Checksum generation for release assets
- macOS code signing note (ad-hoc signing with `codesign`)
- Windows note (Windows Defender SmartScreen bypass for unsigned binaries)

For npm-only distribution, include:
- `"prepublishOnly"` script sequence (typecheck → test → build)
- `.npmignore` listing excluded files (`src/`, `test/`, config files)
- `"engines"` field asserting Node.js version requirement

---

## Section 16: Local Database *(conditional — include only if Local Database = yes)*

See `references/database-patterns.md` for the complete implementation. This section must
include:

### 16.1 drizzle.config.ts

```typescript
import type { Config } from 'drizzle-kit'

export default {
  schema:  './src/db/schema.ts',
  out:     './drizzle',
  dialect: 'sqlite',
  dbCredentials: {
    url: './dev.db',   // local dev only — production uses OS data directory
  },
} satisfies Config
```

### 16.2 Database path resolution (`src/db/path.ts`)

`env-paths` wrapper that resolves the OS-appropriate data directory and ensures it
exists before returning the SQLite file path. Must produce:
- macOS: `~/Library/Application Support/{{BINARY_NAME}}/data.db`
- Linux: `~/.local/share/{{BINARY_NAME}}/data.db`
- Windows: `%APPDATA%\{{BINARY_NAME}}\data.db`

### 16.3 Schema (`src/db/schema.ts`)

Drizzle table definitions for **every entity type** derived from the model files in
`context/model/`. Use actual table names and columns from model definitions. Every
table must include:
- `id` — `text().primaryKey()` storing a UUIDv4 generated in the repository
- Timestamp columns as `integer('col', { mode: 'timestamp' })` mapping to `Date`
- JSON payload columns as `text()` — serialise/deserialise in the repository layer

### 16.4 Database client (`src/db/client.ts`)

Singleton `getDb()` function and `createDbClient(dbPath?)` factory:
- Opens `better-sqlite3` with `journal_mode = WAL`, `foreign_keys = ON`, `synchronous = NORMAL`
- Runs `migrate(db, { migrationsFolder })` on creation — idempotent, zero-penalty once migrations are current
- `closeDb()` for graceful shutdown and test cleanup
- `createDbClient(':memory:')` used in all tests

### 16.5 Repositories (`src/db/repositories/`)

One repository class per entity. Each class:
- Receives `DbClient` via constructor
- Exposes typed CRUD methods matching the user stories for that entity
- Never leaks Drizzle internals — public methods return plain typed objects
- Uses `db.transaction()` for multi-step writes

Include the complete repository code for every entity derived from the model files.

### 16.6 Migration files

List the initial migration SQL file (`drizzle/0000_initial_schema.sql`) that creates
all tables. The SQL must match the Drizzle schema exactly:
- Integer primary keys or `CHAR(36)` UUIDs
- `FOREIGN KEY` constraints matching the schema relationships
- Indexes matching the query patterns in the repositories

Remind the developer: run `npm run db:generate` after any schema change, commit the
generated file, and the CLI will apply it automatically on next run.

---

## Section 17: Async Patterns *(conditional — include only if any async pattern detected)*

See `references/async-patterns.md` for complete implementation. Include only the
sub-sections that apply. Multiple sub-sections may coexist in the same application.

### 17a. Shared Signal Handler *(include whenever any async pattern is yes)*

```typescript
// src/utils/signal.ts — generated whenever any async pattern is selected
export type CleanupFn = () => void | Promise<void>

/**
 * Register SIGINT (Ctrl+C) and SIGTERM handlers. Returns a dispose function
 * that removes the listeners when the loop exits normally.
 * Exit codes: 130 for SIGINT (POSIX convention), 0 for SIGTERM (clean shutdown).
 */
export function onSignal(cleanup: CleanupFn): () => void {
  let handled = false
  const handler = async (signal: NodeJS.Signals): Promise<void> => {
    if (handled) return
    handled = true
    process.stderr.write('\n')
    try { await cleanup() } finally {
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

### 17b. Polling *(conditional — include only if Polling = yes)*

The spec must include:

- **`src/utils/poll.ts`** — Complete `poll<T>(options)` with `check`, `intervalMs`,
  `timeoutMs`, `label`, and `onTick`. Spinner managed internally. `onSignal` registered
  and disposed within the function. Returns `{ value: T, elapsedMs: number }`.
- **Command integration** — For every user story with a `--watch`/`--follow`/`--wait`
  flag, show both code paths in the action handler:
  - Without flag → trigger, print result ID, `output.success()`, return
  - With flag → trigger → `poll()` → interpret terminal state → `output.success()` or exit 1
- **Output contracts** — human and `--json` form for both paths (immediate and polled)
- **Timeout** (exit 1, message "Polling timed out after Xs") and **Ctrl+C** (exit 130,
  message "Polling stopped — operation continues in the background")

### 17c. Inline Batch Processing *(conditional — include only if Inline Batch = yes)*

The spec must include:

- **`src/utils/batch.ts`** — Complete `runBatch<TInput, TOutput>(options)` with `total`,
  `items`, `process`, `concurrency`, `bail`, `onProgress`. Returns `BatchSummary`.
- **`src/ui/progress.ts`** — `ProgressBar` wrapping ora: `update(BatchProgress)` shows
  `[pct%] n/total — ETA Xs`, plus `succeed()` / `fail()` / `stop()` methods.
- **Service resume pattern** — when `Local Database = yes`, the service filters out
  already-processed IDs before returning the item list (checkpoint / resume support).
- **Command options** — `--concurrency <n>`, `--bail`, `--dry-run` on every command
  using `runBatch`, mapped to `BatchOptions` fields.
- **Exit code table**: all succeeded → 0; any failed → 1; Ctrl+C → 130.

### 17d. Background Daemon *(conditional — include only if Background Daemon = yes)*

The spec must include:

- **`src/cli.ts` modification** — daemon mode detection block checking
  `MY_TOOL_DAEMON_MODE === '1'` placed BEFORE `program.name(...)`. Show exact code.
- **`src/daemon/runner.ts`** — `runDaemon()` with event loop, `onSignal`, `tick()` stub
  annotated with which NFRs define the actual work, and `TICK_INTERVAL_MS` constant
  derived from the NFRs (e.g. "checks every 30 seconds" → `30_000`).
- **`src/services/daemon.service.ts`** — Complete `DaemonManager`:
  `start()`, `stop(timeoutMs)`, `isRunning()` (signal-0 check), `getStatus()`,
  `getRecentLogs(n)`. PID and start time stored in `conf`.
- **`src/commands/daemon/index.ts`** — `start`, `stop`, `status`, `logs` sub-commands.
- **`conf` schema additions** — `daemonPid: z.number().int().positive().optional()` and
  `daemonStartedAt: z.string().optional()` added to `UserConfigSchema`.


Each command from PRD.md gets its own `<command-name>/SPEC.md`.

---

## Command SPEC.md Structure

Each command SPEC.md follows the template below. See `references/command-patterns.md`
for the full Commander.js patterns — the generated spec must draw from those patterns
and substitute actual command names, flags, aliases, and option defaults from the
`model/<command>/model.md` context file. Generic placeholders (e.g. `<n>`, `<template>`)
must be replaced with the real argument names from the model.

```markdown
# `{{command}}` Command — Specification

> Part of [{{APPLICATION_NAME}} Technical Specification](../SPECIFICATION.md)

## Overview

**Command**:    `{{BINARY_NAME}} {{command}}`
**Aliases**:    `{{aliases}}`
**Description**: {{description}}
**Version**:    {{version}}

## Traceability

### User Stories
| ID          | Version | Description                         |
|-------------|---------|-------------------------------------|
| USCLI00101  | v0.1.0  | As a user, I want to...             |

### Non-Functional Requirements
| ID          | Version | Description                         |
|-------------|---------|-------------------------------------|
| NFRCLI0001  | v0.1.0  | Output must be filterable by --json |

### Constraints
| ID          | Version | Description                         |
|-------------|---------|-------------------------------------|
| CONSCLI001  | v0.1.0  | Must support --dry-run              |

### Removed / Replaced
| ID          | Type        | Removed In | Replaced By | Reason |
|-------------|-------------|------------|-------------|--------|
| _None._     |             |            |             |        |
```

## 2. Usage

```
{{BINARY_NAME}} {{command}} [arguments] [options]

Arguments:
  <name>           Project name (required)

Options:
  -t, --template   Template to use (default: "default")
  -y, --yes        Skip prompts and use defaults
  --dry-run        Print what would happen without executing
  -h, --help       Display help for command
```

## 3. Command Registration

```typescript
// src/commands/{{command}}/index.ts
import type { Command } from 'commander'
import { handleError } from '../../errors.js'
import { logger } from '../../ui/logger.js'
import { output } from '../../ui/output.js'
// [If Prompts = yes]:
import { prompt{{Command}}Options } from './prompts.js'

export function register{{Command}}Command(program: Command): void {
  program
    .command('{{command}}')
    .alias('{{alias}}')
    .description('{{description}}')
    .argument('<name>', 'project name')
    .option('-t, --template <template>', 'template to use', 'default')
    .option('-y, --yes', 'skip prompts and use defaults', false)
    .option('--dry-run', 'preview without executing', false)
    .action(handleError(async (name: string, opts: {{Command}}Options) => {
      // Setup
      if (opts.verbose) logger.enableVerbose()
      if (opts.json)    output.enableJson()

      // [If Prompts = yes] Collect missing options interactively
      const resolvedOpts = opts.yes ? opts : await prompt{{Command}}Options(name, opts)

      // Delegate to service
      const service = new {{Command}}Service()
      const result  = await service.execute(name, resolvedOpts)

      // Output
      output.success(result, `{{Command}} complete: ${name}`)
    }))
}
```

## 4. Argument & Option Contracts

| Flag               | Short | Type      | Default     | Required | Description              |
|--------------------|-------|-----------|-------------|----------|--------------------------|
| `<name>`           | —     | `string`  | —           | Yes      | Project name             |
| `--template`       | `-t`  | `string`  | `"default"` | No       | Template identifier      |
| `--yes`            | `-y`  | `boolean` | `false`     | No       | Skip all prompts         |
| `--dry-run`        | —     | `boolean` | `false`     | No       | Preview without changes  |

### TypeScript Interface

```typescript
export interface {{Command}}Options {
  template: string
  yes:      boolean
  dryRun:   boolean
  verbose:  boolean
  json:     boolean
}
```

### Zod Validation Schema

```typescript
import { z } from 'zod'

export const {{command}}OptionsSchema = z.object({
  template: z.string().min(1).default('default'),
  yes:      z.boolean().default(false),
  dryRun:   z.boolean().default(false),
  verbose:  z.boolean().default(false),
  json:     z.boolean().default(false),
})
```

## 5. Prompt Flow *(conditional — include only if Prompts = yes for this command)*

```typescript
// src/commands/{{command}}/prompts.ts
import { input, select, confirm } from '@inquirer/prompts'
import type { {{Command}}Options } from './index.js'

export async function prompt{{Command}}Options(
  name: string,
  partial: Partial<{{Command}}Options>,
): Promise<{{Command}}Options> {
  const template = partial.template ?? await select({
    message: 'Select a template:',
    choices: [
      { name: 'Default',  value: 'default',  description: 'Minimal starter' },
      { name: 'Full',     value: 'full',      description: 'All features enabled' },
    ],
  })

  const confirm_ = await confirm({
    message: `Create project "${name}" with template "${template}"?`,
    default: true,
  })

  if (!confirm_) {
    process.stdout.write('Aborted.\n')
    process.exit(0)
  }

  return {
    template,
    yes:     false,
    dryRun:  partial.dryRun  ?? false,
    verbose: partial.verbose ?? false,
    json:    partial.json    ?? false,
  }
}
```

## 6. Service Interface

```typescript
// src/services/{{command}}.service.ts (public interface)
export interface I{{Command}}Service {
  execute(name: string, options: {{Command}}Options): Promise<{{Command}}Result>
}

export interface {{Command}}Result {
  name:      string
  template:  string
  path:      string
  files:     string[]
  dryRun:    boolean
}
```

## 7. Service Implementation

```typescript
// src/services/{{command}}.service.ts
import { join } from 'node:path'
import { mkdir, writeFile } from 'node:fs/promises'
import { CliError } from '../errors.js'
import { logger } from '../ui/logger.js'
import type { {{Command}}Options, {{Command}}Result, I{{Command}}Service } from './{{command}}.service.js'

export class {{Command}}Service implements I{{Command}}Service {
  async execute(name: string, options: {{Command}}Options): Promise<{{Command}}Result> {
    logger.debug(`Executing {{command}} for "${name}" with template "${options.template}"`)

    const targetPath = join(process.cwd(), name)

    // Service has NO terminal output — returns data only
    if (options.dryRun) {
      return { name, template: options.template, path: targetPath, files: [], dryRun: true }
    }

    // Real implementation (derive from user stories)
    // ...

    return {
      name,
      template: options.template,
      path:     targetPath,
      files:    ['package.json', 'tsconfig.json', 'README.md'],
      dryRun:   false,
    }
  }
}
```

## 8. Output Contract

### Human-readable output

```
✔ {{Command}} complete: my-project

  Created 3 files in /Users/you/my-project:
  • package.json
  • tsconfig.json
  • README.md

  Run `cd my-project && npm install` to get started.
```

### JSON output (`--json`)

```json
{
  "success": true,
  "data": {
    "name":     "my-project",
    "template": "default",
    "path":     "/Users/you/my-project",
    "files":    ["package.json", "tsconfig.json", "README.md"],
    "dryRun":   false
  }
}
```

### JSON error output

```json
{
  "success": false,
  "error": {
    "code":    "INVALID_INPUT",
    "message": "Project name cannot contain spaces"
  }
}
```

### Async output contracts *(include only for commands with async patterns)*

**If this command uses `--watch` / `--follow` (Polling):**

| Scenario | Exit Code | Human output | JSON `success` |
|----------|-----------|--------------|----------------|
| Triggered, no --watch | 0 | `ℹ Job started: job-123` | `true` |
| --watch, succeeded | 0 | `✔ Done in 14.2s` | `true` |
| --watch, failed | 1 | `✖ Job failed: <reason>` | `false` |
| --watch, timed out | 1 | `✖ Timed out after 300s` | `false` |
| Ctrl+C during poll | 130 | `⚠ Polling stopped — continues in background` | N/A |

**If this command uses `runBatch` (Inline Batch):**

| Scenario | Exit Code | Human output |
|----------|-----------|--------------|
| All succeeded | 0 | `✔ Synced 432 resources in 24.1s` |
| Partial failure | 1 | `✖ Completed with 3 error(s)` |
| --bail triggered | 1 | `✖ Stopped after first error` |
| Ctrl+C | 130 | `⚠ Batch interrupted — stopping after current chunk` |

## 9. Error Cases

| Scenario                        | Error Code      | Exit Code | Message shown to user                         |
|---------------------------------|-----------------|-----------|-----------------------------------------------|
| Name contains invalid chars     | `INVALID_INPUT` | 2         | "Project name may only contain..."            |
| Target directory already exists | `CONFLICT`      | 1         | "Directory already exists: /path/to/dir"      |
| Template not found              | `NOT_FOUND`     | 3         | "Template 'xyz' not found"                    |
| File system write failure       | `IO_ERROR`      | 1         | "Failed to create project files: <reason>"    |

## 10. Tests

(See `references/testing-patterns.md` for full test patterns)

```typescript
// test/commands/{{command}}.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { {{Command}}Service } from '../../src/services/{{command}}.service.js'

describe('{{Command}}Service', () => {
  it('returns dryRun result without creating files', async () => {
    const service = new {{Command}}Service()
    const result = await service.execute('my-project', {
      template: 'default', yes: true, dryRun: true, verbose: false, json: false,
    })
    expect(result.dryRun).toBe(true)
    expect(result.files).toHaveLength(0)
  })
})
```
