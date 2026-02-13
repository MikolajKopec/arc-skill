# ARC Framework Skill

Claude Code skill providing knowledge base for the [ARC framework](https://github.com/nicolo-ribaudo/arc) — a full-stack TypeScript framework built on CQRS and Event Sourcing patterns. Used at [narzedziadlatworcow.pl](https://narzedziadlatworcow.pl).

## Installation

```bash
npx skills add MikolajKopec/arc-skill@arc-framework -g -y
```

Installs globally for Claude Code, Cursor, Codex, and other supported agents.

## What it covers

| Package | Description |
|---------|-------------|
| `@arcote.tech/arc` | Core — type system, model, events, commands, views, listeners, routes, data storage |
| `@arcote.tech/arc-react` | React — hooks (`useQuery`, `useCommands`, `useRevalidate`), Form system, Provider |
| `@arcote.tech/arc-host` | Host — Bun HTTP + WebSocket server, JWT auth, PostgreSQL/SQLite adapters |
| `@arcote.tech/arc-cli` | CLI — `arc dev` / `arc build`, multi-client compile-time constants |

## Files

```
skills/arc-framework/
├── SKILL.md                          # Main skill (~340 lines)
└── references/
    ├── type-system.md                # ArcElement types, validators, JSON Schema
    ├── commands-events.md            # Commands, Events, Listeners, Translator
    ├── views-queries.md              # Views, Queries, Subscriptions, Auth
    ├── data-storage.md               # DataStorage, DB adapters, transactions
    ├── react-integration.md          # React hooks, Form system, Provider
    ├── host-server.md                # Bun server, endpoints, JWT, WebSocket
    └── cli-builds.md                 # CLI, multi-client builds, tree shaking
```

## Validation

Skill content was validated across **4 rounds** against the actual ARC framework source code (`v0.1.12`). Each round used parallel exploration agents to cross-reference every API signature, code example, and architectural claim with the real implementation.
