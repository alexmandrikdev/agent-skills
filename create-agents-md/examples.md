# AGENTS.md Examples

Lean examples and contrasts showing what to cut.

## Example 1: Node monorepo (~40 lines)

```markdown
# Design System

React 18, TypeScript 5, pnpm workspaces, Turborepo, Vitest, ESLint.

## Commands

- `pnpm install`: Install all workspace dependencies
- `pnpm --filter @acme/ui dev`: Dev server for the UI package
- `pnpm --filter @acme/ui test`: Full test suite for UI package
- `pnpm --filter @acme/ui vitest run src/Button.test.tsx`: Single test file
- `pnpm --filter @acme/ui lint`: ESLint with auto-fix
- `pnpm build`: Build all packages

## Project structure

- `packages/ui/` — Published component library
- `packages/tokens/` — Design tokens (source of truth for colors/spacing)
- `apps/storybook/` — Component docs and visual tests
- `packages/ui/src/primitives/` — Low-level building blocks; do not import from apps

## Code style

- Named exports only in `packages/ui/` — no default exports
- See `packages/ui/src/Button.tsx` as the canonical component pattern

## Testing

- Vitest + Testing Library
- `pnpm --filter @acme/ui vitest run -t "renders label"`: Single test by name
- Do not mock design tokens — use real token values

## Boundaries

### Always

- Run `pnpm --filter <package> lint` on changed packages before finishing

### Ask first

- Adding dependencies to `packages/ui/`
- Changing token values in `packages/tokens/`

### Never

- Commit `dist/` or `.turbo/` cache
- Import from `apps/` into `packages/`
- Modify `packages/ui/src/legacy/` — sync-only code, scheduled for removal
```

### What to cut (bloated vs lean)

| Bloated (remove) | Lean (keep) |
|------------------|-------------|
| "This is a monorepo managed with pnpm and Turborepo for efficient builds..." | Stack line with versions |
| "Run the tests to make sure everything passes" | `pnpm --filter @acme/ui vitest run src/Button.test.tsx` |
| "Use TypeScript for type safety" | Named exports only — non-default rule |
| Full directory tree of every package | 4 non-obvious paths with responsibilities |
| "Follow React best practices" | Pointer to `Button.tsx` as canonical pattern |

---

## Example 2: Python API (~35 lines)

```markdown
# Invoice API

FastAPI, Python 3.12, PostgreSQL, SQLAlchemy 2.0, Alembic.

## Commands

- `uv sync`: Install dependencies
- `uv run dev`: Dev server (port 8000)
- `uv run pytest tests/ -v`: Full test suite
- `uv run pytest tests/unit/test_handlers.py -v`: Single test file
- `uv run ruff check --fix .`: Lint and auto-fix
- `alembic upgrade head`: Run migrations

## Project structure

- `app/api/v1/` — Route handlers (thin, delegate to services)
- `app/services/` — Business logic
- `app/models/` — SQLAlchemy models
- `app/schemas/` — Pydantic v2 request/response schemas
- `app/repositories/` — Data access (no raw SQL in handlers)

## Code style

- Type hints on all function signatures
- Async handlers by default
- Pydantic v2 models for all request/response shapes

## Testing

- pytest-asyncio for async tests
- Factory Boy for test data — no hand-rolled fixtures
- Do not mock the database; use the test database

## Boundaries

### Always

- Handlers delegate to services — no business logic in route files

### Ask first

- Schema migrations affecting production tables
- New external service integrations

### Never

- Modify `app/legacy/` — sync code, intentional
- Commit `.env` or credential files
- Raw SQL in handlers or services (use repositories)
```

### What to cut (bloated vs lean)

| Bloated (remove) | Lean (keep) |
|------------------|-------------|
| "FastAPI is a modern Python web framework..." | Stack line with versions |
| "Write clean, readable code with proper error handling" | Handlers delegate to services |
| "Use pytest for testing" | pytest-asyncio + Factory Boy + no DB mocking |
| Every SQLAlchemy best practice | Pydantic v2 + async handlers — agent would get wrong |
| Copy of README architecture diagram | 5 directory bullets with responsibilities |

---

## Section omission guide

| Section | Include when | Skip when |
|---------|--------------|-----------|
| Commands | Almost always | Never — highest ROI section |
| Project structure | Non-obvious layout, monorepo, layered architecture | Flat single-package with standard layout |
| Code style | Deviations from defaults or canonical-file patterns | ESLint/ruff config covers everything |
| Testing | Mock boundaries or non-obvious test setup | Commands section already has single-test command |
| Boundaries | Real constraints exist | Greenfield with no sensitive areas yet |
| Git / PR | Clear convention in `git log` or CONTRIBUTING | No established team workflow |
