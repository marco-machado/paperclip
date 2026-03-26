# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Essential Reading

Before making changes, read in this order:
1. `AGENTS.md` — contributor rules, engineering invariants, definition of done
2. `doc/SPEC-implementation.md` — the concrete V1 build contract
3. `doc/DEVELOPING.md` — dev setup, worktrees, deployment modes
4. `doc/DATABASE.md` — database structure and patterns

## Commands

```sh
# Install dependencies
pnpm install

# Start full stack (API + UI) in dev mode
pnpm dev                  # API: http://localhost:3100, UI: http://localhost:3100

# Type checking
pnpm -r typecheck         # All packages
pnpm typecheck            # Root workspace only

# Tests (always use single-run mode)
pnpm test:run             # All unit/integration tests, once
pnpm -F @paperclipai/server test:run   # Server tests only
pnpm -F @paperclipai/ui test:run       # UI tests only

# E2E tests
pnpm test:e2e             # Playwright, headless
pnpm test:e2e:headed      # Playwright, visible browser

# Build all packages
pnpm build

# Database migrations (after editing packages/db/src/schema/*.ts)
pnpm db:generate          # Generate migration from schema changes
pnpm db:migrate           # Apply pending migrations

# Reset local dev database
rm -rf data/pglite && pnpm dev
```

## Monorepo Structure

pnpm workspaces with these packages:

| Package | Path | Description |
|---|---|---|
| `@paperclipai/server` | `server/` | Express REST API + orchestration services |
| `@paperclipai/ui` | `ui/` | React 19 + Vite dashboard |
| `paperclipai` (CLI) | `cli/` | Command-line tool (`paperclipai` binary) |
| `@paperclipai/db` | `packages/db/` | Drizzle ORM schema, migrations, DB clients |
| `@paperclipai/shared` | `packages/shared/` | Shared types, constants, validators, API paths |
| `@paperclipai/adapter-*` | `packages/adapters/*/` | Agent adapter packages (Claude, Codex, Cursor, Gemini, Pi, etc.) |
| `@paperclipai/plugin-sdk` | `packages/plugins/` | Plugin system SDK |

## Architecture Overview

Paperclip is a control plane for AI-agent companies. Agents are treated as employees with org charts, budgets, approval gates, and activity logs.

**Server** (`server/src/`):
- `routes/` — Express route handlers, all under `/api` base path
- `services/` — Business logic (66+ files); major services: `issues.ts`, `agents.ts`, `heartbeat.ts`, `company-portability.ts`
- `adapters/` — Server-side adapter orchestration
- `realtime/` — WebSocket live-event emission
- `auth/` — better-auth integration; board = operator, agent = bearer API key

**UI** (`ui/src/`):
- `pages/` — Route-level page components
- `components/` — Reusable React components
- `api/` — Typed API client functions (mirrors server routes)
- `context/` — Company selection and global state
- `adapters/` — UI adapter implementations

**Database** (`packages/db/src/`):
- `schema/` — 60+ Drizzle TypeScript schema files, one per domain entity
- `migrations/` — Auto-generated SQL (never edit manually)
- `schema/index.ts` — Must export all tables; update when adding new ones

**Shared** (`packages/shared/src/`):
- Single source of truth for types, enums, Zod validators, and API path constants used across server and UI

## Key Engineering Rules

1. **Company-scoped entities** — every domain object belongs to a company; enforce boundaries in both routes and services
2. **Sync contracts across layers** — schema → shared types → server routes → UI clients must stay in sync
3. **Preserve control-plane invariants**: single-assignee tasks, atomic issue checkout, approval gates, budget hard-stop auto-pause, activity logging on all mutations
4. **Plan docs** go in `doc/plans/` with `YYYY-MM-DD-slug.md` filenames

## Database Change Workflow

1. Edit `packages/db/src/schema/*.ts`
2. Export new tables from `packages/db/src/schema/index.ts`
3. Run `pnpm db:generate` (compiles schema first, then generates migration)
4. Run `pnpm -r typecheck` to validate

## Verification Checklist (Before Claiming Done)

```sh
pnpm -r typecheck
pnpm test:run
pnpm build
```
