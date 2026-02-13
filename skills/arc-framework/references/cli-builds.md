# CLI & Multi-Client Builds — Deep Dive

## Package: `@arcote.tech/arc-cli`

The CLI provides development and build tooling for ARC projects with multi-client support.

## Installation

```bash
bun add -D @arcote.tech/arc-cli
```

## Commands

### `arc dev`

Watch mode with auto-rebuild on file changes:

```bash
bunx arc dev
```

- Watches source files for changes
- Rebuilds all client targets on change

### `arc build`

Production build:

```bash
bunx arc build
```

- One-time build of all client targets
- Optimized output via Bun bundler

## Configuration: `arc.config.json`

Place in project root:

```json
{
  "file": "index.ts",
  "outDir": "dist",
  "clients": ["browser", "server", "api"]
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `file` | `string` | Entry point file |
| `outDir` | `string` | Output directory |
| `clients` | `string[]` | Build target clients |

### Monorepo Support

For monorepo projects, the CLI reads `workspaces` from `package.json`:

```json
// package.json
{
  "workspaces": ["packages/**/*"]
}
```

Each workspace with an `arc.config.json` gets built independently.

## Compile-Time Constants

Constants are generated **dynamically** from your `clients` config. The client name is normalized: uppercased and non-alphanumeric chars replaced with `_`.

### Generation Logic

```typescript
// For each client in config.clients:
const normalized = clientName.toUpperCase().replace(/[^A-Z0-9]/g, "_")

// Three constants generated per client:
defineValues[normalized] = isCurrentClient ? "true" : "false"
defineValues[`NOT_ON_${normalized}`] = isCurrentClient ? "false" : "true"
defineValues[`ONLY_${normalized}`] = isCurrentClient ? "true" : "false"
```

### Example: clients = ["browser", "server"]

When building for `browser`:

| Constant | Value |
|----------|-------|
| `BROWSER` | `true` |
| `NOT_ON_BROWSER` | `false` |
| `ONLY_BROWSER` | `true` |
| `SERVER` | `false` |
| `NOT_ON_SERVER` | `true` |
| `ONLY_SERVER` | `false` |

When building for `server`:

| Constant | Value |
|----------|-------|
| `BROWSER` | `false` |
| `NOT_ON_BROWSER` | `true` |
| `ONLY_BROWSER` | `false` |
| `SERVER` | `true` |
| `NOT_ON_SERVER` | `false` |
| `ONLY_SERVER` | `true` |

### Custom Client Names

For a client named `mobile-app`:
- Normalized to `MOBILE_APP`
- Constants: `MOBILE_APP`, `NOT_ON_MOBILE_APP`, `ONLY_MOBILE_APP`

### Usage in Code

```typescript
import { context } from "@arcote.tech/arc"

// Conditional elements based on build target
const appContext = context([
  // Always included
  userCreatedEvent,
  userProfileView,

  // Server-only elements (tree-shaken from browser build)
  NOT_ON_BROWSER && databaseMigrationCommand,
  NOT_ON_BROWSER && serverOnlyListener,

  // Browser-only elements (tree-shaken from server build)
  ONLY_BROWSER && clientSideCache,
  ONLY_BROWSER && browserAnalytics,
])
```

### How It Works

1. CLI reads `arc.config.json`
2. For each client in `clients`:
   a. Sets `define` constants via Bun bundler
   b. Builds the entry point with those constants
   c. Dead code elimination removes false branches
3. Output: `dist/browser/index.js`, `dist/server/index.js`, etc.

### TypeScript Declarations

Add to your `global.d.ts` (match your actual clients config):

```typescript
// For clients: ["browser", "server", "api"]
declare const BROWSER: boolean
declare const NOT_ON_BROWSER: boolean
declare const ONLY_BROWSER: boolean

declare const SERVER: boolean
declare const NOT_ON_SERVER: boolean
declare const ONLY_SERVER: boolean

declare const API: boolean
declare const NOT_ON_API: boolean
declare const ONLY_API: boolean
```

## Build Output

```
dist/
├── browser/
│   └── index.js        # Browser-optimized bundle
├── server/
│   └── index.js         # Server bundle (includes DB, listeners)
└── api/
    └── index.js         # API-only bundle
```

### Tree Shaking

Thanks to compile-time constants:

```typescript
// In source:
if (NOT_ON_BROWSER) {
  // This entire block is removed from browser build
  hostLiveModel(appContext, { db: "app.db", version: 1 })
}

// After browser build:
// (block completely removed — dead code elimination)
```

This means:
- Server-only code (DB adapters, listeners) doesn't ship to browser
- Browser-only code (React hooks, DOM utils) doesn't ship to server
- Smaller bundles, faster loads

## Context Pipe + Build Targets

The `context.pipe()` method works with conditional elements:

```typescript
const baseContext = context([
  userCreatedEvent,
  createUserCommand,
])

const fullContext = baseContext.pipe([
  // false values are filtered out by pipe()
  NOT_ON_BROWSER && serverMigration,
  ONLY_BROWSER && clientCache,

  // Lazy imports for code splitting
  () => import("./features/analytics"),
])
```

`pipe()` filters out `false` values, so when `NOT_ON_BROWSER` is `false` (browser build), `serverMigration` is excluded entirely.

## Development Workflow

```bash
# 1. Start dev mode (watches all clients)
bunx arc dev

# 2. In another terminal, run the server build
bun run dist/server/index.js

# 3. Browser build served by your frontend tool (Vite, etc.)
```

## Project Structure Recommendation

```
my-arc-project/
├── arc.config.json          # Build config
├── package.json
├── global.d.ts              # Compile-time constant types
├── src/
│   ├── index.ts             # Entry point (configured in arc.config.json)
│   ├── context.ts           # Main ArcContext composition
│   ├── events/
│   │   ├── user.ts
│   │   └── order.ts
│   ├── commands/
│   │   ├── user.ts
│   │   └── order.ts
│   ├── views/
│   │   ├── user-profile.ts
│   │   └── order-board.ts
│   ├── listeners/
│   │   ├── user-notifications.ts
│   │   └── order-processing.ts
│   ├── routes/
│   │   └── api.ts
│   └── server.ts            # Server setup (guarded by NOT_ON_BROWSER)
├── dist/                    # Build output
│   ├── browser/
│   ├── server/
│   └── api/
└── app.db                   # SQLite database (development)
```
