# Command Patterns — Commander.js Architecture

This reference describes the CLI command architecture patterns. Include relevant content
in Sections 4 and the per-command SPEC.md files.

---

## Architecture Overview

```
cli.ts (Program root)
    └── registerXxxCommand(program)          ← Each command in its own file
            ├── program.command('xxx')
            │       .argument(...)
            │       .option(...)
            │       .action(handleError(async (arg, opts) => {
            │           // 1. Setup (verbose/json modes)
            │           // 2. Collect missing inputs (prompts if not --yes)
            │           // 3. Validate inputs
            │           // 4. Call service
            │           // 5. Format and print result
            │       }))
            └── (sub-commands registered via .addCommand())
```

**Dependency direction rule:**
- `cli.ts` → imports command registrations
- command files → import services and UI utilities
- services → import config and HTTP client
- UI utilities → import chalk/ora/table (no service/config imports)
- No circular dependencies

---

## Commander.js Program Setup

### Root Program

```typescript
// src/cli.ts
import { Command } from 'commander'
import { fileURLToPath } from 'node:url'
import { dirname, join }   from 'node:path'
import { readFileSync }     from 'node:fs'

const __filename = fileURLToPath(import.meta.url)
const __dirname  = dirname(__filename)
const pkg = JSON.parse(readFileSync(join(__dirname, '../package.json'), 'utf-8')) as {
  name: string; version: string; description: string
}

const program = new Command()

program
  .name(pkg.name)
  .description(pkg.description)
  .version(pkg.version, '-V, --version')
  // Global options available on all sub-commands via parent resolution
  .option('-v, --verbose', 'enable verbose output', false)
  .option('--json',        'output as JSON',         false)
  .option('--no-color',    'disable ANSI colours')
  .addHelpCommand('help [command]', 'Display help for command')
  // Prevent Commander from catching process.exit(0) as an error
  .exitOverride()

// Register commands
registerInitCommand(program)
registerDeployCommand(program)

// Allow top-level await in the entry point
await program.parseAsync(process.argv)
```

### Reading Parent Options Inside a Command

Commander stores parent options separately. Access global options from inside a command:

```typescript
.action(handleError(async (arg: string, opts: LocalOptions, command: Command) => {
  // Walk up to program root for global options
  const globalOpts = command.parent?.opts<GlobalOptions>() ?? {
    verbose: false,
    json:    false,
    color:   true,
  }
  if (globalOpts.verbose) logger.enableVerbose()
  if (globalOpts.json)    output.enableJson()
}))
```

---

## Command Patterns by Type

### Simple Command (no sub-commands)

```typescript
// src/commands/status/index.ts
import type { Command } from 'commander'
import { handleError }  from '../../errors.js'
import { output }       from '../../ui/output.js'
import { StatusService } from '../../services/status.service.js'

export interface StatusOptions {
  environment: string
  verbose:     boolean
  json:        boolean
}

export function registerStatusCommand(program: Command): void {
  program
    .command('status')
    .description('Check the deployment status of an environment')
    .argument('[env]', 'target environment', 'production')
    .option('-e, --environment <env>', 'environment override')
    .action(handleError(async (env: string, opts: StatusOptions, cmd: Command) => {
      const { verbose, json } = cmd.parent?.opts<{ verbose: boolean; json: boolean }>() ?? {}
      if (verbose) logger.enableVerbose()
      if (json)    output.enableJson()

      const service = new StatusService()
      const result  = await service.getStatus(opts.environment ?? env)
      output.success(result, `Status: ${result.status}`)
    }))
}
```

### Command with Sub-commands

Use `.addCommand()` to nest sub-commands. Each sub-command is its own registered handler.

```typescript
// src/commands/config/index.ts
import type { Command } from 'commander'
import { registerConfigGetCommand }   from './get.js'
import { registerConfigSetCommand }   from './set.js'
import { registerConfigListCommand }  from './list.js'
import { registerConfigResetCommand } from './reset.js'

export function registerConfigCommand(program: Command): void {
  const configCmd = program
    .command('config')
    .description('Manage {{BINARY_NAME}} configuration')
    .addHelpCommand(false)

  registerConfigGetCommand(configCmd)
  registerConfigSetCommand(configCmd)
  registerConfigListCommand(configCmd)
  registerConfigResetCommand(configCmd)
}
```

```typescript
// src/commands/config/get.ts
import type { Command } from 'commander'
import { handleError }  from '../../errors.js'
import { getUserConfig } from '../../config/user.config.js'
import { output }        from '../../ui/output.js'

export function registerConfigGetCommand(parent: Command): void {
  parent
    .command('get <key>')
    .description('Print the value of a config key')
    .action(handleError(async (key: string) => {
      const config = getUserConfig()
      const value  = config.get(key as never)
      if (value === undefined) {
        output.failure('NOT_FOUND', `Config key not found: ${key}`)
        process.exit(3)
      }
      output.success({ key, value }, `${key} = ${String(value)}`)
    }))
}
```

### Command with Required Confirmation

For destructive operations, require explicit confirmation:

```typescript
// src/commands/destroy/index.ts
import { confirm } from '@inquirer/prompts'

export function registerDestroyCommand(program: Command): void {
  program
    .command('destroy')
    .description('Permanently delete a project (irreversible)')
    .argument('<project-id>', 'ID of the project to delete')
    .option('-f, --force', 'skip confirmation prompt', false)
    .action(handleError(async (projectId: string, opts: DestroyOptions) => {
      if (!opts.force) {
        const confirmed = await confirm({
          message: `Permanently delete project "${projectId}"? This cannot be undone.`,
          default: false,
        })
        if (!confirmed) {
          logger.info('Aborted.')
          process.exit(0)
        }
      }

      const service = new DestroyService()
      await service.deleteProject(projectId)
      output.success({ projectId, deleted: true }, `Project deleted: ${projectId}`)
    }))
}
```

### Command with `--dry-run`

Every mutating command should support `--dry-run`. The service receives `dryRun` in its
options and returns a preview without side effects.

```typescript
.option('--dry-run', 'preview changes without applying them', false)
.action(handleError(async (opts: DeployOptions) => {
  const service = new DeployService()
  const result  = await service.deploy({ ...opts })

  if (opts.dryRun) {
    output.success(result, 'Dry run complete — no changes applied')
  } else {
    output.success(result, `Deployed to ${result.environment}`)
  }
}))
```

### Command Producing Tabular Output

```typescript
import { printTable } from '../../ui/table.js'
import type { GlobalOptions } from '../../types/index.js'

.action(handleError(async (_opts: ListOptions, cmd: Command) => {
  const { json } = cmd.parent?.opts<GlobalOptions>() ?? { json: false }
  output.enableJson() // conditionally

  const service = new ListService()
  const items   = await service.list()

  if (json) {
    output.success(items, '')
    return
  }

  if (items.length === 0) {
    logger.info('No items found.')
    return
  }

  printTable({
    head: ['Name', 'Status', 'Created'],
    rows: items.map(i => [i.name, i.status, i.createdAt]),
  })
  logger.info(`${items.length.toString()} item(s) found.`)
}))
```

---

## Global Option Propagation Pattern

Commander does not automatically pass parent options to child commands. Adopt this
consistent pattern across all command action handlers:

```typescript
/** Extract global options from the root program */
function getGlobalOptions(cmd: Command): GlobalOptions {
  // Walk up to root
  let current: Command = cmd
  while (current.parent) current = current.parent
  return current.opts<GlobalOptions>()
}
```

Or as a shared utility:

```typescript
// src/utils/command.ts
import type { Command } from 'commander'
import type { GlobalOptions } from '../types/index.js'

export function setupCommandContext(cmd: Command): void {
  const { verbose, json } = getGlobalOptions(cmd)
  if (verbose) logger.enableVerbose()
  if (json)    output.enableJson()
}

export function getGlobalOptions(cmd: Command): GlobalOptions {
  let root: Command = cmd
  while (root.parent) root = root.parent
  return root.opts<GlobalOptions>()
}
```

Usage in every action handler:

```typescript
.action(handleError(async (arg: string, opts: MyOptions, cmd: Command) => {
  setupCommandContext(cmd)
  // rest of handler
}))
```

---

## Command Exit Code Conventions

| Scenario                         | Exit Code |
|----------------------------------|-----------|
| Success                          | `0`       |
| General / unexpected error       | `1`       |
| Invalid argument / option value  | `2`       |
| Resource not found               | `3`       |
| Unauthorized / unauthenticated   | `4`       |
| Network / connectivity error     | `5`       |
| Configuration error              | `6`       |
| User aborted (Ctrl+C, no confirm)| `0`       |

Always use `process.exit(code)` — never `throw` past the command action handler.

---

## Help Text Conventions

Every command and option must have descriptive help text. Commander auto-generates
`--help` from these strings.

**Descriptions should:**
- Start with a capital letter
- End without a period (Commander adds none)
- Be actionable: "Deploy the project to an environment" not "Deployment"
- Include the default value in option descriptions when non-obvious

```typescript
program
  .command('deploy')
  .description('Deploy the current project to a target environment')
  .option('-e, --environment <name>', 'target environment name', 'staging')
  .option('--strategy <type>',        'deployment strategy: rolling|blue-green', 'rolling')
  .option('--timeout <seconds>',      'deployment timeout in seconds', '300')
  .option('--dry-run',                'preview what would be deployed without deploying')
```

Resulting `--help`:
```
Usage: my-tool deploy [options]

Deploy the current project to a target environment

Options:
  -e, --environment <name>  target environment name (default: "staging")
  --strategy <type>          deployment strategy: rolling|blue-green (default: "rolling")
  --timeout <seconds>        deployment timeout in seconds (default: "300")
  --dry-run                  preview what would be deployed without deploying
  -h, --help                 display help for command
```

---

## Async Error Handling in Commander

Commander v12+ supports `parseAsync`. All commands that do async work must be wrapped
with `handleError` (from `src/errors.ts`) to ensure clean exit codes:

```typescript
// WRONG — unhandled rejection bubbles past Commander
.action(async (arg, opts) => {
  const result = await service.execute(arg)
  // ...
})

// CORRECT — handleError wraps and formats all thrown errors
.action(handleError(async (arg: string, opts: Opts) => {
  const result = await service.execute(arg)
  // ...
}))
```

The `handleError` wrapper catches:
- `CliError` instances → formats with code and exits with `exitCode`
- Any other `Error` → prints generic message, logs stack in verbose mode, exits `1`
- Non-Error throws → converts to string, exits `1`
