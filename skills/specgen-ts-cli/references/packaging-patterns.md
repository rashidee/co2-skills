# Packaging & Distribution Patterns — tsup + @yao-pkg/pkg

This reference describes the build pipeline and cross-platform binary packaging strategy.
Include relevant content in Section 15 of the generated specification.

---

## Distribution Strategies Overview

```
┌──────────────────────────────────────────────────────────────────┐
│  Strategy A — npm distribution (default)                         │
│                                                                  │
│  TypeScript ──tsup──► dist/cli.js  ──npm publish──► npm registry│
│                                                                  │
│  Users: npm install -g my-tool                                   │
│  Requires: Node.js ≥ 22 on user's machine                       │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  Strategy B — Standalone binary (select when Binary = yes)       │
│                                                                  │
│  TypeScript ──tsup──► dist/cli.js ──pkg──► dist/bin/            │
│                        │                   ├── my-tool-linux     │
│                        │                   ├── my-tool-macos     │
│                        │                   └── my-tool.exe       │
│  Upload to GitHub Releases (or Homebrew tap, Chocolatey, etc.)  │
│  Users: download + chmod +x (no Node.js required)               │
└──────────────────────────────────────────────────────────────────┘
```

---

## tsup Build Configuration

### tsup.config.ts

```typescript
import { defineConfig } from 'tsup'

export default defineConfig({
  entry: ['src/cli.ts'],
  format: ['esm'],
  target: 'node22',
  platform: 'node',
  outDir: 'dist',
  clean: true,
  sourcemap: process.env['NODE_ENV'] !== 'production',
  minify: false,           // pkg does its own minification; keep readable for npm
  splitting: false,        // single-file output is essential for pkg compatibility
  bundle: true,
  dts: false,              // no type declarations needed for a CLI binary
  shims: false,            // we target Node.js natively — no CJS shims needed
  banner: {
    js: '#!/usr/bin/env node', // shebang injected at top of dist/cli.js
  },
  esbuildOptions(opts) {
    opts.mainFields = ['module', 'main']
    opts.conditions  = ['import', 'default']
  },
  external: [
    // These are Node.js builtins — never bundle them
    'node:fs', 'node:path', 'node:url', 'node:os', 'node:child_process',
    'node:crypto', 'node:stream', 'node:events',
  ],
  noExternal: [
    // Force-bundle ESM-only packages that pkg cannot resolve at runtime
    'chalk', 'ora', 'boxen',
  ],
})
```

> **Splitting must be `false`** — pkg requires a single entry-point file; code splitting
> produces multiple chunks that confuse the static analyser.

### Build Script in package.json

```json
{
  "scripts": {
    "build":     "tsup",
    "build:prod": "NODE_ENV=production tsup",
    "dev":       "tsup --watch",
    "typecheck": "tsc --noEmit"
  }
}
```

### Verifying the Build

```bash
node dist/cli.js --help          # smoke test
node dist/cli.js --version       # version output
```

---

## pkg Configuration (Binary Packaging)

`@yao-pkg/pkg` is the actively maintained fork of `vercel/pkg`. It compiles the bundled
JS file plus a Node.js runtime into a single executable.

### pkg field in package.json

```json
{
  "pkg": {
    "scripts": "dist/cli.js",
    "assets": [
      "dist/**/*"
    ],
    "targets": [
      "node22-linux-x64",
      "node22-macos-x64",
      "node22-macos-arm64",
      "node22-win-x64"
    ],
    "outputPath": "dist/bin"
  }
}
```

**Target string format:** `node{version}-{os}-{arch}`
- OS: `linux`, `macos`, `win`
- Arch: `x64`, `arm64`
- Version: `22` (match the project's `engines.node`)

### Binary Packaging Scripts

```json
{
  "scripts": {
    "pkg:linux":   "pkg . --target node22-linux-x64    --output dist/bin/{{BINARY_NAME}}-linux",
    "pkg:macos-x64": "pkg . --target node22-macos-x64  --output dist/bin/{{BINARY_NAME}}-macos-x64",
    "pkg:macos-arm64": "pkg . --target node22-macos-arm64 --output dist/bin/{{BINARY_NAME}}-macos-arm64",
    "pkg:windows": "pkg . --target node22-win-x64      --output dist/bin/{{BINARY_NAME}}.exe",
    "pkg:all":     "npm run pkg:linux && npm run pkg:macos-x64 && npm run pkg:macos-arm64 && npm run pkg:windows",
    "release":     "npm run build:prod && npm run pkg:all"
  }
}
```

### Asset Embedding

Files referenced at runtime that pkg cannot auto-detect must be declared in `assets`.
Common examples:

```json
{
  "pkg": {
    "assets": [
      "dist/**/*.json",
      "dist/templates/**/*",
      "node_modules/some-package/data/**/*"
    ]
  }
}
```

### ESM + pkg Compatibility Constraints

pkg has limited ESM support. These constraints apply when Binary Packaging = yes:

1. **Use `tsup` to produce a single CJS file for pkg.** Even if the project is ESM,
   add a separate pkg-specific tsup target:

```typescript
// tsup.config.ts — add CJS output for pkg
export default defineConfig([
  // Primary ESM build (for npm distribution)
  {
    entry: ['src/cli.ts'],
    format: ['esm'],
    outDir: 'dist',
    banner: { js: '#!/usr/bin/env node' },
    // ...
  },
  // CJS build for pkg binary packaging
  {
    entry: ['src/cli.ts'],
    format: ['cjs'],
    outDir: 'dist/cjs',
    noExternal: ['chalk', 'ora', 'boxen', 'update-notifier'],
    // ESM-only packages must be force-bundled
  },
])
```

Then point pkg to the CJS output:
```json
{
  "pkg": {
    "scripts": "dist/cjs/cli.cjs"
  }
}
```

2. **Use `fileURLToPath(import.meta.url)` — never `__dirname`.** pkg understands
   `import.meta.url` in ESM snapshots. For CJS bundles, tsup shims `__dirname`.

3. **Dynamic `import()` is not statically analysed by pkg.** If you use
   `import(someVariable)`, add the module to `pkg.assets` or `pkg.scripts`.

---

## GitHub Actions Release Workflow

This workflow triggers on `v*` tags (e.g., `v0.2.0`), builds all binaries, computes
SHA-256 checksums, and creates a GitHub Release.

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write

jobs:
  build-binaries:
    name: Build binaries
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - target: linux-x64
            os: ubuntu-latest
          - target: macos-x64
            os: macos-latest
          - target: macos-arm64
            os: macos-latest
          - target: windows-x64
            os: windows-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - run: npm ci

      - name: Build
        run: npm run build:prod

      - name: Package binary
        run: npx @yao-pkg/pkg . --target node22-${{ matrix.target }} --output dist/bin/{{BINARY_NAME}}-${{ matrix.target }}${{ matrix.target == 'windows-x64' && '.exe' || '' }}

      - name: Compute checksum
        run: |
          BINARY="dist/bin/{{BINARY_NAME}}-${{ matrix.target }}${{ matrix.target == 'windows-x64' && '.exe' || '' }}"
          sha256sum "$BINARY" > "${BINARY}.sha256"
        shell: bash

      - uses: actions/upload-artifact@v4
        with:
          name: binary-${{ matrix.target }}
          path: dist/bin/

  release:
    name: Create GitHub Release
    needs: build-binaries
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          path: artifacts
          merge-multiple: true

      - name: Create Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
          files: |
            artifacts/**/*
```

---

## npm-Only Distribution

When `Binary Packaging = no`, the CLI is published to npm. Checklist for the spec:

### package.json fields

```json
{
  "name":    "@scope/my-tool",
  "version": "0.1.0",
  "type":    "module",
  "bin":     { "my-tool": "./dist/cli.js" },
  "files":   ["dist", "README.md"],
  "engines": { "node": ">=22.0.0" },
  "publishConfig": {
    "access": "public"
  },
  "scripts": {
    "prepublishOnly": "npm run typecheck && npm run test && npm run build"
  }
}
```

### .npmignore

```
src/
test/
*.config.ts
*.config.js
.eslintrc.*
tsconfig.json
.github/
*.test.ts
coverage/
```

### Local Testing Before Publish

```bash
# Install locally from source
npm link

# Test the linked binary
my-tool --version
my-tool --help

# Unlink when done
npm unlink my-tool
```

### Semantic Versioning

| Change type                          | Version bump |
|--------------------------------------|--------------|
| Bug fix, minor improvement           | Patch `0.0.x` |
| New option or command, backward-compat| Minor `0.x.0` |
| Breaking flag rename, removed option | Major `x.0.0` |

---

## Platform-Specific Notes

### macOS Gatekeeper (Code Signing)

Unsigned binaries from GitHub Releases will be quarantined by macOS Gatekeeper.
Users will need to right-click → Open on first run, or run:
```bash
xattr -c ./my-tool-macos-arm64
```

For production tools distributed via Homebrew, add macOS ad-hoc signing to the
GitHub Actions workflow:
```yaml
- name: Sign binary (macOS)
  if: runner.os == 'macOS'
  run: codesign --force --sign - dist/bin/{{BINARY_NAME}}-${{ matrix.target }}
```

### Windows SmartScreen

Self-signed or unsigned `.exe` files trigger Windows Defender SmartScreen. Users can
click "More info" → "Run anyway". For wide distribution, consider code signing with
a purchased certificate or distributing via Chocolatey/Scoop which handle trust.

### Linux Permissions

The Linux binary needs `chmod +x` after download:
```bash
chmod +x ./my-tool-linux
./my-tool-linux --version
```

Document this in the project README under "Installation".

---

## Homebrew Tap (Optional)

For macOS/Linux distribution via Homebrew:

```ruby
# Formula/my-tool.rb
class MyTool < Formula
  desc "{{APP_DESCRIPTION}}"
  homepage "https://github.com/org/{{BINARY_NAME}}"
  version "{{VERSION}}"

  on_macos do
    on_arm do
      url "https://github.com/org/{{BINARY_NAME}}/releases/download/v#{version}/{{BINARY_NAME}}-macos-arm64"
      sha256 "REPLACE_WITH_ACTUAL_SHA256"
    end
    on_intel do
      url "https://github.com/org/{{BINARY_NAME}}/releases/download/v#{version}/{{BINARY_NAME}}-macos-x64"
      sha256 "REPLACE_WITH_ACTUAL_SHA256"
    end
  end

  on_linux do
    on_intel do
      url "https://github.com/org/{{BINARY_NAME}}/releases/download/v#{version}/{{BINARY_NAME}}-linux"
      sha256 "REPLACE_WITH_ACTUAL_SHA256"
    end
  end

  def install
    bin.install Dir["{{BINARY_NAME}}-*"].first => "{{BINARY_NAME}}"
  end

  test do
    system "#{bin}/{{BINARY_NAME}}", "--version"
  end
end
```

Host in a GitHub repo named `homebrew-tap`. Users install via:
```bash
brew tap org/tap
brew install my-tool
```
