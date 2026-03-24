# Configuration Management Patterns — conf + cosmiconfig

This reference describes the two configuration layers for CLI applications. Include
relevant content in Sections 9 and 10 of the generated specification.

---

## Configuration Layers Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  Precedence order (highest → lowest)                                │
│                                                                     │
│  1. CLI flags        --environment staging  (per-invocation)        │
│  2. Env variables    MY_TOOL_ENV=staging    (per-session)           │
│  3. Project config   .mytoolrc / mytool.config.js (per-project)    │
│  4. User config      ~/.config/my-tool/config.json (per-user)      │
│  5. Built-in defaults (hardcoded in source)                         │
└─────────────────────────────────────────────────────────────────────┘
```

- **User config** (`conf`) — global, per-user settings: API tokens, preferences,
  default values. Stored in the OS user config directory
- **Project config** (`cosmiconfig`) — per-workspace/repo settings: project-specific
  options, team conventions. Committed to source control

---

## User Configuration — conf

`conf` stores JSON in the OS user config directory and provides a typed, schema-validated
key-value store. It uses the OS keychain on macOS for sensitive fields (optional).

### Setup

```typescript
// src/config/user.config.ts
import Conf from 'conf'
import { z } from 'zod'

// 1. Define schema with Zod for runtime validation
const UserConfigSchema = z.object({
  apiToken:      z.string().optional(),
  apiBaseUrl:    z.string().url().default('https://api.example.com'),
  defaultEnv:    z.enum(['staging', 'production']).default('staging'),
  telemetryEnabled: z.boolean().default(true),
})

export type UserConfigData = z.infer<typeof UserConfigSchema>

// 2. conf stores values as plain JSON (no Zod at rest — validated on read)
type ConfStore = Partial<UserConfigData>

const conf = new Conf<ConfStore>({
  projectName: '{{BINARY_NAME}}',
  projectVersion: 1,          // increment when schema changes to trigger migration
  schema: {
    // JSON Schema (conf uses ajv) for basic type checking
    apiToken:         { type: 'string' },
    apiBaseUrl:       { type: 'string' },
    defaultEnv:       { type: 'string', enum: ['staging', 'production'] },
    telemetryEnabled: { type: 'boolean' },
  },
  defaults: {
    apiBaseUrl:       'https://api.example.com',
    defaultEnv:       'staging',
    telemetryEnabled: true,
  },
})

// 3. Typed accessor — always validates on read
export function getUserConfig(): {
  get<K extends keyof UserConfigData>(key: K): UserConfigData[K]
  set<K extends keyof UserConfigData>(key: K, value: UserConfigData[K]): void
  delete<K extends keyof UserConfigData>(key: K): void
  all(): UserConfigData
  clear(): void
  path: string
} {
  return {
    get<K extends keyof UserConfigData>(key: K): UserConfigData[K] {
      const raw = conf.get(key)
      const parsed = UserConfigSchema.shape[key].safeParse(raw)
      if (!parsed.success) {
        // Fall back to default
        return UserConfigSchema.shape[key].parse(undefined)
      }
      return parsed.data as UserConfigData[K]
    },
    set<K extends keyof UserConfigData>(key: K, value: UserConfigData[K]): void {
      UserConfigSchema.shape[key].parse(value) // validate before write
      conf.set(key, value)
    },
    delete<K extends keyof UserConfigData>(key: K): void {
      conf.delete(key)
    },
    all(): UserConfigData {
      return UserConfigSchema.parse(conf.store)
    },
    clear(): void {
      conf.clear()
    },
    get path(): string {
      return conf.path
    },
  }
}
```

### Config Sub-commands

Every CLI using `conf` should expose `config get`, `config set`, `config list`, and
`config reset` sub-commands so users can inspect and modify their settings without
editing JSON manually.

```typescript
// src/commands/config/list.ts
import type { Command } from 'commander'
import { handleError } from '../../errors.js'
import { getUserConfig } from '../../config/user.config.js'
import { output } from '../../ui/output.js'
import { printTable } from '../../ui/table.js'

export function registerConfigListCommand(parent: Command): void {
  parent
    .command('list')
    .alias('ls')
    .description('List all configuration values')
    .action(handleError(async (_opts, cmd: Command) => {
      const { json } = cmd.parent?.parent?.opts<{ json: boolean }>() ?? { json: false }
      if (json) output.enableJson()

      const config = getUserConfig()
      const all    = config.all()

      output.success(all, '') // JSON mode

      if (!json) {
        printTable({
          head: ['Key', 'Value'],
          rows: Object.entries(all).map(([k, v]) => [k, String(v)]),
        })
        logger.info(`Config stored at: ${config.path}`)
      }
    }))
}
```

```typescript
// src/commands/config/set.ts
import type { Command } from 'commander'
import { handleError, CliError } from '../../errors.js'
import { getUserConfig } from '../../config/user.config.js'
import { output } from '../../ui/output.js'
import type { UserConfigData } from '../../config/user.config.js'

export function registerConfigSetCommand(parent: Command): void {
  parent
    .command('set <key> <value>')
    .description('Set a configuration value')
    .action(handleError(async (key: string, rawValue: string) => {
      const config    = getUserConfig()
      const validKeys = Object.keys(config.all()) as (keyof UserConfigData)[]

      if (!validKeys.includes(key as keyof UserConfigData)) {
        throw CliError.invalidInput(
          `Unknown config key: "${key}". Valid keys: ${validKeys.join(', ')}`
        )
      }

      // Coerce string to correct type
      let value: unknown = rawValue
      if (rawValue === 'true')  value = true
      if (rawValue === 'false') value = false
      if (!isNaN(Number(rawValue)) && rawValue !== '') value = Number(rawValue)

      config.set(key as keyof UserConfigData, value as never)
      output.success({ key, value }, `Set ${key} = ${rawValue}`)
    }))
}
```

```typescript
// src/commands/config/reset.ts
import type { Command } from 'commander'
import { confirm } from '@inquirer/prompts'
import { handleError } from '../../errors.js'
import { getUserConfig } from '../../config/user.config.js'
import { output, logger } from '../../ui/output.js'

export function registerConfigResetCommand(parent: Command): void {
  parent
    .command('reset')
    .description('Reset all configuration to defaults')
    .option('-f, --force', 'skip confirmation', false)
    .action(handleError(async (opts: { force: boolean }) => {
      if (!opts.force) {
        const confirmed = await confirm({
          message: 'Reset all configuration to defaults?',
          default: false,
        })
        if (!confirmed) {
          logger.info('Aborted.')
          process.exit(0)
        }
      }
      getUserConfig().clear()
      output.success({}, 'Configuration reset to defaults')
    }))
}
```

### Login / Logout Commands *(for API-token-based auth)*

```typescript
// src/commands/login/index.ts
import { password } from '@inquirer/prompts'
import type { Command } from 'commander'
import { handleError } from '../../errors.js'
import { getUserConfig } from '../../config/user.config.js'
import { AuthService } from '../../services/auth.service.js'
import { output } from '../../ui/output.js'

export function registerLoginCommand(program: Command): void {
  program
    .command('login')
    .description('Authenticate with {{APPLICATION_NAME}}')
    .option('--token <token>', 'API token (or use interactive prompt)')
    .action(handleError(async (opts: { token?: string }) => {
      const token = opts.token ?? await password({
        message: 'Enter your API token:',
        mask: '*',
      })

      const authService = new AuthService()
      const user        = await authService.validateToken(token)

      getUserConfig().set('apiToken', token)

      output.success({ user: user.email }, `Logged in as ${user.email}`)
    }))
}
```

```typescript
// src/commands/logout/index.ts
export function registerLogoutCommand(program: Command): void {
  program
    .command('logout')
    .description('Remove stored credentials')
    .action(handleError(async () => {
      getUserConfig().delete('apiToken')
      output.success({}, 'Logged out — credentials removed')
    }))
}
```

---

## Project Configuration — cosmiconfig

`cosmiconfig` searches for a project-level config file starting from the current working
directory and traversing upward. Useful when the CLI tool operates within a project
(e.g., a deployment tool, a linter, a build tool).

### Supported Config File Formats

cosmiconfig automatically searches for (in order):
- `package.json` — `"{{BINARY_NAME}}"` field
- `.{{BINARY_NAME}}rc` — JSON or YAML
- `.{{BINARY_NAME}}rc.json`
- `.{{BINARY_NAME}}rc.yaml` / `.{{BINARY_NAME}}rc.yml`
- `.{{BINARY_NAME}}rc.js` / `.{{BINARY_NAME}}rc.mjs` / `.{{BINARY_NAME}}rc.cjs`
- `{{BINARY_NAME}}.config.js` / `{{BINARY_NAME}}.config.mjs` / `{{BINARY_NAME}}.config.cjs`

### Setup

```typescript
// src/config/project.config.ts
import { cosmiconfig } from 'cosmiconfig'
import { z } from 'zod'
import { CliError } from '../errors.js'

// 1. Define schema
const ProjectConfigSchema = z.object({
  environment: z.enum(['staging', 'production', 'development']).default('staging'),
  region:      z.string().default('us-east-1'),
  services:    z.array(z.string()).default([]),
  hooks: z.object({
    preDeploy:  z.string().optional(),
    postDeploy: z.string().optional(),
  }).default({}),
})

export type ProjectConfig = z.infer<typeof ProjectConfigSchema>

const DEFAULT_CONFIG: ProjectConfig = ProjectConfigSchema.parse({})

// 2. Cached search result
let _cachedConfig: ProjectConfig | null = null

/**
 * Load and validate the project config file. Searches up from CWD.
 * Returns defaults if no config file found (non-fatal).
 */
export async function loadProjectConfig(searchFrom?: string): Promise<ProjectConfig> {
  if (_cachedConfig) return _cachedConfig

  const explorer = cosmiconfig('{{BINARY_NAME}}', {
    searchStrategy: 'project',
    stopDir: process.env['HOME'] ?? '/',
  })

  const result = await explorer.search(searchFrom ?? process.cwd())

  if (!result || result.isEmpty) {
    _cachedConfig = DEFAULT_CONFIG
    return DEFAULT_CONFIG
  }

  const parsed = ProjectConfigSchema.safeParse(result.config)
  if (!parsed.success) {
    throw CliError.configError(
      `Invalid config in ${result.filepath}:\n${parsed.error.message}`
    )
  }

  _cachedConfig = parsed.data
  return parsed.data
}

/** Reset the config cache (useful in tests) */
export function resetProjectConfigCache(): void {
  _cachedConfig = null
}
```

### Merging Config Precedence

In command handlers, merge configuration sources in the correct precedence order:

```typescript
// src/utils/resolve-config.ts
import { loadProjectConfig } from '../config/project.config.js'
import { getUserConfig }     from '../config/user.config.js'
import type { DeployOptions } from '../commands/deploy/index.js'

export interface ResolvedDeployConfig {
  environment: string
  region:      string
}

export async function resolveDeployConfig(
  cliOpts: Partial<DeployOptions>
): Promise<ResolvedDeployConfig> {
  const projectConfig = await loadProjectConfig()
  const userConfig    = getUserConfig()

  return {
    // CLI flag > env var > project config > user default > hardcoded default
    environment:
      cliOpts.environment                  // CLI flag (highest)
      ?? process.env['MY_TOOL_ENV']         // env var
      ?? projectConfig.environment         // project config
      ?? userConfig.get('defaultEnv')      // user preference
      ?? 'staging',                        // hardcoded default (lowest)

    region:
      cliOpts.region
      ?? process.env['MY_TOOL_REGION']
      ?? projectConfig.region
      ?? 'us-east-1',
  }
}
```

### Sample Config Files

The spec must include example config files that developers can copy:

**`.mytoolrc.json`**
```json
{
  "environment": "production",
  "region": "eu-west-1",
  "services": ["api", "worker"],
  "hooks": {
    "preDeploy":  "npm test",
    "postDeploy": "npm run notify"
  }
}
```

**`mytool.config.js`** (for dynamic configs using JS)
```javascript
// mytool.config.js
export default {
  environment: process.env.NODE_ENV === 'production' ? 'production' : 'staging',
  region: 'us-east-1',
  services: ['api', 'worker'],
}
```

**`package.json` field**
```json
{
  "name": "my-project",
  "my-tool": {
    "environment": "staging",
    "services": ["api"]
  }
}
```

---

## Environment Variable Support

Every config key should have an environment variable equivalent. Define a mapping table
in the spec:

| Config Key         | Environment Variable      | Priority |
|--------------------|---------------------------|----------|
| `environment`      | `MY_TOOL_ENV`             | 2nd      |
| `region`           | `MY_TOOL_REGION`          | 2nd      |
| `apiToken`         | `MY_TOOL_TOKEN`           | 2nd      |
| `apiBaseUrl`       | `MY_TOOL_API_URL`         | 2nd      |

Load environment variables in the `resolveConfig` utility, not in services directly.
This keeps services testable without process.env manipulation.
