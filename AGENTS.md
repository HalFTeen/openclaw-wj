# OpenClaw Agent Guidelines

## Quick Reference

| Task | Command |
|------|---------|
| Install deps | `pnpm install` |
| Build | `pnpm build` |
| Lint | `pnpm lint` |
| Lint + fix | `pnpm lint:fix` |
| Format check | `pnpm format` |
| Format fix | `pnpm format:fix` |
| Test all | `pnpm test` |
| Test watch | `pnpm test:watch` |
| Test coverage | `pnpm test:coverage` |
| Run CLI (dev) | `pnpm openclaw ...` |

## Running a Single Test

```bash
# Run a specific test file
pnpm test src/utils.test.ts

# Run tests matching a pattern
pnpm test -t "normalizeE164"

# Run a specific test file in watch mode
pnpm test:watch src/config/schema.test.ts

# Run with verbose output
pnpm test --reporter=verbose src/utils.test.ts
```

Test file naming:
- Unit tests: `*.test.ts` (colocated with source)
- E2E tests: `*.e2e.test.ts`
- Live tests (real APIs): `*.live.test.ts`

## Project Structure

```
src/                 # TypeScript source (ESM)
├── cli/             # CLI wiring, prompts, progress
├── commands/        # CLI command implementations
├── config/          # Configuration loading, schemas, validation
├── channels/        # Multi-channel abstraction
├── gateway/         # Gateway server, protocol
├── agents/          # AI agent logic, model selection
├── providers/       # LLM provider integrations
├── web/             # WhatsApp Web channel
├── telegram/        # Telegram channel
├── discord/         # Discord channel
├── slack/           # Slack channel
└── ...
extensions/          # Plugin workspace packages
apps/                # Native apps (ios, android, macos)
docs/                # Documentation (Mintlify)
dist/                # Build output
```

## Code Style Guidelines

### Language & Runtime
- **TypeScript** with ESM modules (`"type": "module"`)
- **Node 22+** required; Bun also supported for dev/scripts
- **Strict mode** enabled; avoid `any` types

### Imports
```typescript
// Node built-ins first (with node: prefix)
import fs from "node:fs";
import path from "node:path";

// External packages
import { Type } from "@sinclair/typebox";

// Internal imports with .js extension (ESM requirement)
import { loadConfig } from "./config/io.js";
import type { AgentConfig } from "./config/types.js";
```

- Use `node:` prefix for Node.js built-ins
- Always include `.js` extension for relative imports
- Use `type` keyword for type-only imports
- Group imports: node builtins → external → internal

### Formatting (Oxfmt)
- **2 spaces** indentation
- **100 characters** max line width
- Run `pnpm format:fix` before committing

### Linting (Oxlint)
- Plugins: `unicorn`, `typescript`, `oxc`
- Correctness rules are errors
- Run `pnpm lint` before committing

### Naming Conventions
- **camelCase**: functions, variables, parameters
- **PascalCase**: types, interfaces, classes
- **SCREAMING_SNAKE_CASE**: constants
- **kebab-case**: file names
- Product name: **OpenClaw** (docs/UI), `openclaw` (CLI/code)

### Functions & Error Handling
```typescript
// Explicit return types for exported functions
export function normalizeE164(number: string): string {
  const digits = number.replace(/[^\d+]/g, "");
  return digits.startsWith("+") ? digits : `+${digits}`;
}

// Prefer nullish coalescing and optional chaining
const value = opts?.timeout ?? 30_000;

// Throw descriptive errors; use try-catch for recoverable errors
if (!body) throw new Error("Message (--message) is required");
```

### File Organization
- Keep files under ~500-700 LOC; split when it improves clarity
- Colocate tests with source: `foo.ts` → `foo.test.ts`
- Use barrel exports for type modules
- Extract helpers instead of creating "V2" copies

### Dependency Injection
```typescript
// Use createDefaultDeps pattern for testability
export function myCommand(
  opts: CommandOpts,
  deps: CliDeps = createDefaultDeps(),
) {
  // Use deps.logger, deps.config, etc.
}
```

### CLI Progress & Output
- Use `src/cli/progress.ts` for spinners/progress bars
- Use `src/terminal/table.ts` for table output
- Use `src/terminal/palette.ts` for colors (no hardcoded ANSI)

## Testing Guidelines

- **Framework**: Vitest with V8 coverage (70% threshold)
- **Timeout**: 120 seconds default
- **Pool**: `forks` (parallel test execution)

```typescript
import { describe, it, expect, vi } from "vitest";

describe("normalizeE164", () => {
  it("adds plus prefix to digits", () => {
    expect(normalizeE164("1234567890")).toBe("+1234567890");
  });
});
```

## Commit Guidelines

- Use `scripts/committer "<msg>" <file...>` for scoped commits
- Follow Conventional Commits: `fix:`, `feat:`, `refactor:`, `docs:`, `test:`, `chore:`
- Keep commits atomic (one logical change per commit)
- Run full gate before pushing: `pnpm lint && pnpm build && pnpm test`

## Multi-Agent Safety

When multiple agents work concurrently:
- Do NOT create/drop `git stash` entries
- Do NOT switch branches without explicit request
- Scope commits to your changes only
- Auto-resolve formatting-only conflicts without asking
