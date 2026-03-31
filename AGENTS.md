# OpenCode Repository Guide

This file provides essential context for agentic coding agents operating in this repository.

## Repository Structure

This is a Bun monorepo with multiple packages under `packages/`:

- `packages/opencode` — Core CLI/TUI application (main backend)
- `packages/app` — Web frontend (SolidJS + Vite)
- `packages/ui` — Shared UI components
- `packages/sdk/js` — JavaScript SDK
- `packages/console/app` — Console web app (SolidStart)
- `packages/desktop` — Tauri desktop app
- Other supporting packages: `util`, `plugin`, `script`, `function`, `enterprise`

## Build/Lint/Test Commands

### Root Level

```bash
bun install                    # Install dependencies
bun typecheck                  # Run typecheck across all packages (via turbo)
bun test                       # BLOCKED — do not run tests from root
```

### packages/opencode (Core Backend)

```bash
bun typecheck                  # Typecheck (uses tsgo, not tsc)
bun test                       # Run all tests
bun test --timeout 30000       # Run tests with custom timeout
bun test path/to/test.ts       # Run a single test file
bun test -g "test name"        # Run tests matching pattern
bun run db generate --name <slug>  # Generate Drizzle migration
bun run build                  # Build the CLI
bun run --conditions=browser ./src/index.ts serve --port 4096  # Dev server
```

### packages/app (Web Frontend)

```bash
bun typecheck                  # Typecheck (tsgo -b)
bun dev                        # Start Vite dev server
bun test:unit                  # Unit tests (happydom)
bun test:unit ./src/path.test.ts   # Single unit test
bun test:e2e                   # E2E tests (Playwright)
bun test:e2e -- app/home.spec.ts   # Single E2E test file
bun test:e2e -- -g "test name"     # Single E2E test by name
bun test:e2e:ui                # E2E with Playwright UI (debugging)
bun test:e2e:local             # E2E with full local server setup
```

### packages/ui (UI Components)

```bash
bun typecheck                  # Typecheck
bun dev                        # Start Vite dev server
```

### packages/sdk/js (SDK)

```bash
bun typecheck                  # Typecheck
bun build                      # Build SDK (runs ./script/build.ts)
```

## Key Rules

- **ALWAYS USE PARALLEL TOOLS WHEN APPLICABLE.**
- **The default branch is `dev`.** Local `main` may not exist; use `dev` or `origin/dev` for diffs.
- **Tests cannot run from repo root.** Run from package directories (e.g., `packages/opencode`).
- **Never restart the app or server process during debugging.**
- **Prefer automation**: execute requested actions without confirmation unless blocked.

## Style Guide

### General Principles

- Keep things in one function unless composable or reusable
- Avoid `try`/`catch` where possible — use `.catch(() => void)` for fire-and-forget
- Avoid using the `any` type
- Use Bun APIs when possible (e.g., `Bun.file()`, `Bun.write()`)
- Rely on type inference; avoid explicit type annotations unless necessary for exports
- Prefer functional array methods (flatMap, filter, map) over for loops
- Use type guards on filter to maintain type inference downstream

### Naming (MANDATORY FOR AGENT WRITTEN CODE)

Use single word names by default for locals, params, and helper functions:

- Good: `pid`, `cfg`, `err`, `opts`, `dir`, `root`, `child`, `state`, `timeout`
- Avoid unless required: `inputPID`, `existingClient`, `connectTimeout`, `workerPath`

Multi-word names allowed only when a single word would be unclear or ambiguous.

```ts
// Good
const foo = 1
function journal(dir: string) {}

// Bad
const fooBar = 1
function prepareJournal(dir: string) {}
```

Reduce total variable count by inlining when a value is only used once:

```ts
// Good
const journal = await Bun.file(path.join(dir, "journal.json")).json()

// Bad
const journalPath = path.join(dir, "journal.json")
const journal = await Bun.file(journalPath).json()
```

### Destructuring

Avoid unnecessary destructuring. Use dot notation to preserve context:

```ts
// Good
obj.a
obj.b

// Bad
const { a, b } = obj
```

### Variables

Prefer `const` over `let`. Use ternaries or early returns instead of reassignment:

```ts
// Good
const foo = condition ? 1 : 2

// Bad
let foo
if (condition) foo = 1
else foo = 2
```

### Control Flow

Avoid `else` statements. Prefer early returns:

```ts
// Good
function foo() {
  if (condition) return 1
  return 2
}

// Bad
function foo() {
  if (condition) return 1
  else return 2
}
```

### Imports

Group imports logically. Use `@/` alias for internal paths within packages:

```ts
import z from "zod" // External libraries
import { Hono } from "hono" // External libraries
import { Tool } from "./tool" // Same directory
import { Instance } from "@/project/instance" // Internal via alias
import { Log } from "@/util/log" // Internal via alias
```

### Schema Definitions (Drizzle)

Use snake_case for field names so column names don't need redefinition:

```ts
// Good
const table = sqliteTable("session", {
  id: text().primaryKey(),
  project_id: text().notNull(),
  created_at: integer().notNull(),
})

// Bad
const table = sqliteTable("session", {
  id: text("id").primaryKey(),
  projectID: text("project_id").notNull(),
  createdAt: integer("created_at").notNull(),
})
```

### Error Handling

- Prefer `Effect.catch` or `.catch(() => void)` over try/catch blocks
- For fire-and-forget operations: `appendFile(path, data).catch(() => {})`
- In Effect code: `yield* fs.remove(path).pipe(Effect.catch(() => Effect.void))`

## Testing

### Core Tests (packages/opencode)

- Avoid mocks as much as possible
- Test actual implementation, do not duplicate logic into tests
- Use `tmpdir` fixture from `test/fixture/fixture.ts` for temp directories:

```ts
import { tmpdir } from "./fixture/fixture"

test("example", async () => {
  await using tmp = await tmpdir({ git: true })
  // tmp.path is the temp directory path
  // Auto-cleaned when test ends via Symbol.asyncDispose
})
```

### E2E Tests (packages/app/e2e)

- Import from `../fixtures`, never from `@playwright/test`
- Use helper functions from `../actions` and `../selectors`
- Never use `waitForTimeout` — wait on observable state with `expect.poll`
- Use `data-component` or `data-action` selectors, not CSS classes
- Use `modKey` for cross-platform keyboard shortcuts

```ts
import { test, expect } from "../fixtures"
import { withSession, openSettings } from "../actions"
import { modKey } from "../utils"

test("settings can be opened", async ({ page, gotoSession }) => {
  await gotoSession()
  await page.keyboard.press(`${modKey}+Comma`)
  // ...
})
```

## Type Checking

Always run `bun typecheck` from package directories, never `tsc` directly.
Uses `tsgo` (TypeScript native preview) for performance.

## Effect Patterns (packages/opencode)

See `packages/opencode/AGENTS.md` and `specs/effect-migration.md` for detailed Effect patterns:

- Use `Effect.gen(function* () { ... })` for composition
- Use `Effect.fn("Domain.method")` for named/traced effects
- Use `Schema.Class` for multi-field data, `Schema.brand` for single values
- Use `Schema.TaggedErrorClass` for typed errors
- Use `InstanceState` for per-directory state that needs cleanup
- Use `makeRuntime` for shared services with layer deduplication

## Database Migrations

- Schema lives in `src/**/*.sql.ts` files
- Tables/columns use snake_case; join columns are `<entity>_id`
- Indexes are `<table>_<column>_idx`
- Generate migrations: `bun run db generate --name <slug>`
- Output: `migration/<timestamp>_<slug>/migration.sql`

## Local Development

### Backend + Web Frontend

For local UI changes, run separately:

- Backend: `bun run --conditions=browser ./src/index.ts serve --port 4096`
- App: `bun dev -- --port 4444`
- Open `http://localhost:4444` (targets backend at `localhost:4096`)

### Desktop App

```bash
bun run --cwd packages/desktop tauri dev
```

## Package-Specific Notes

- `packages/desktop`: Never call Tauri `invoke` manually. Use generated bindings in `src/bindings.ts`.
- `packages/app`: Prefer `createStore` over multiple `createSignal` calls.
- `packages/sdk/js`: Regenerate via `./packages/sdk/js/script/build.ts`.

## Formatting

Prettier config (from root package.json):

- `semi: false` — No semicolons
- `printWidth: 120`

## Cursor/Copilot Rules

No `.cursor/rules/`, `.cursorrules`, or `.github/copilot-instructions.md` files found.
