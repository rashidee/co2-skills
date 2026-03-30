# Dockerfile Pattern — Node.js

Multi-stage Dockerfile for Node.js applications (Express, Fastify, NestJS, Next.js, etc.).

## Detection Inputs

| Input | Source | Used For |
|---|---|---|
| Node version | `package.json` → `engines.node` or `.nvmrc` | Base image tag |
| Package manager | `pnpm-lock.yaml` / `yarn.lock` / `package-lock.json` | Install command |
| Has build step | `package.json` → `scripts.build` present | Build stage needed |
| Start command | `package.json` → `scripts.start` | CMD instruction |
| Framework | `package.json` → dependencies (next, express, fastify, nestjs) | Optimizations |
| App port | Source code / `.env` → `PORT` or framework default | EXPOSE instruction |

## Template — With Build Step (TypeScript / Next.js / NestJS)

```dockerfile
# =============================================================================
# Stage 1: Dependencies
# =============================================================================
FROM node:22-alpine AS deps

WORKDIR /app

# Copy package files for dependency caching
COPY package.json package-lock.json ./
RUN npm ci

# =============================================================================
# Stage 2: Build
# =============================================================================
FROM node:22-alpine AS build

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN npm run build

# =============================================================================
# Stage 3: Runtime
# =============================================================================
FROM node:22-alpine AS runtime

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy only production dependencies
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Copy built output
COPY --from=build /app/dist ./dist

# Set ownership
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Expose application port
EXPOSE {app_port}

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:{app_port}/health || exit 1

# Production defaults
ENV NODE_ENV=production

CMD ["node", "dist/index.js"]
```

## Template — Without Build Step (Plain JavaScript)

```dockerfile
FROM node:22-alpine AS runtime

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY . .

RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE {app_port}

ENV NODE_ENV=production

CMD ["node", "src/index.js"]
```

## Template — pnpm Variant

Replace npm commands with pnpm:

```dockerfile
FROM node:22-alpine AS deps

RUN corepack enable && corepack prepare pnpm@latest --activate

WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

# ... build stage same but with pnpm ...

FROM node:22-alpine AS runtime
RUN corepack enable && corepack prepare pnpm@latest --activate
# ...
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile --prod
```

## Template — Next.js (Standalone)

Next.js has a special standalone output mode for minimal container size:

```dockerfile
FROM node:22-alpine AS build

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS runtime

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app

# Next.js standalone output
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public

RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 3000
ENV NODE_ENV=production
CMD ["node", "server.js"]
```

## Notes

- **Build output directory**: Varies by tool — `dist/` (tsup, tsc), `.next/` (Next.js),
  `build/` (common convention). Detect from `tsconfig.json` → `outDir` or build tool config.
- **Start command**: Detect from `package.json` → `scripts.start`. Could be
  `node dist/index.js`, `next start`, `nest start`, `fastify start`, etc.
- **Monorepo**: If the app is in a monorepo (detected by `workspaces` in root
  `package.json`), the Dockerfile needs to be at the monorepo root with build context
  scoped to the package directory.
