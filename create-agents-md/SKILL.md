---
name: create-agents-md
description: >-
  Creates a lean, tool-agnostic AGENTS.md at the project root by inspecting
  the repository for build/test commands, structure, and agent boundaries.
  Use when the user asks to create, scaffold, or generate AGENTS.md, or wants
  agent instructions for a project.
disable-model-invocation: true
---

# Create AGENTS.md

Scaffold a lean, tool-agnostic `AGENTS.md` at the project root by inspecting the repository. AGENTS.md is plain markdown — no YAML frontmatter — readable by any coding agent.

For tightening an existing file, use `clean-up-rule`. For Cursor-specific `.mdc` rules, use `create-rule`.

## Scope

| In scope | Out of scope |
|----------|--------------|
| Create new root `AGENTS.md` from repo inspection | Migrate or reorganize existing docs |
| Draft lean content from manifests, CI, README skim | Incremental updates when conventions change |
| Tool-agnostic agent guidance | `.cursor/rules/`, `CLAUDE.md`, `.cursorrules` |

**Existing file guard:** If `AGENTS.md` exists and is non-empty, stop. Tell the user to use `clean-up-rule` or ask explicitly to overwrite.

## Hard rules

- **Tool-agnostic**: no tool-specific config paths or activation modes
- **Lean by default**: drop lines the agent can infer from `package.json`, manifests, or linter config
- **Commands over prose**: `pnpm test -- src/foo.test.ts`, not "run the test suite"
- **Iterate mindset**: AGENTS.md grows when agents repeat mistakes — do not front-load hypothetical rules
- **Never stage or commit** unless the user explicitly asks in the same message
- **Plain markdown only**: no YAML frontmatter in `AGENTS.md`

## Workflow

### 1. Intake

Capture user-provided constraints: package manager preference, off-limits areas, commit format, stack emphasis.

If none provided, proceed without blocking questions.

### 2. Inspect repository

Read only what is needed to infer commands and conventions. Run discovery in parallel. Do not explore the whole codebase.

| Source | What to extract |
|--------|-----------------|
| `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `Makefile`, `justfile` | Install, build, test, lint commands |
| `.github/workflows/*`, `Makefile` targets | CI-verified commands |
| Top-level README (skim only) | Project purpose, stack versions — do not copy prose |
| `docker-compose.yml`, `.env.example` | Dev server / env setup |
| Linter/formatter config | Only non-default style rules |
| Directory layout | Brief structure map |

```bash
# Typical parallel discovery (adapt to stack)
ls -la
# read manifests, Makefile, CI workflows, README opening paragraphs
```

### 3. Extract agent-only facts

Apply the inclusion filter from [agents.md](https://agents.md/):

**Include:**

- Exact shell commands with flags
- Language, framework, and tool versions
- Directory responsibilities (non-obvious layout only)
- Rules that differ from language or framework defaults
- Three-tier boundaries (Always / Ask first / Never)
- Fast single-file or single-test commands

**Exclude:**

- README narrative or marketing copy
- Generic programming tutorials
- Linter rules duplicated from config files
- Tool-specific agent config
- Hypothetical rules for mistakes that have not happened

### 4. Draft

Use [template.md](template.md). For section order and examples, see [examples.md](examples.md).

| Target | Guideline |
|--------|-----------|
| Most projects | 30–80 lines |
| Hard cap | ~120 lines |
| Tone | Imperative bullets, copy-pasteable commands |

**Recommended sections** (omit empty ones):

1. Title + one-line stack (language, framework, versions)
2. **Commands** — install, dev, test (full + single-file), lint, migrate
3. **Project structure** — 5–8 bullets for non-obvious directories only
4. **Code style** — deviations from defaults; point to canonical files when possible
5. **Testing** — runner, fast single-test command, mock boundaries
6. **Boundaries** — Always / Ask first / Never
7. **Git / PR** — only when `git log` or CONTRIBUTING shows a clear convention

### 5. Verify and write

Run the quality checklist below. Write `AGENTS.md` at the repository root.

## Quality checklist

| Check | Target |
|-------|--------|
| Length | 30–80 lines typical; never exceed ~120 |
| Commands | Literal shell commands with flags, verified against manifests or CI |
| Redundancy | No README copy; no generic advice the agent already knows |
| Style rules | Only non-default conventions; reference canonical files when helpful |
| Boundaries | Three tiers present when the project has real constraints |
| Structure | Non-obvious dirs only; skip self-evident `src/`, `tests/` |
| Tool-agnostic | No `.cursor/`, `CLAUDE.md`, or proprietary rule formats |
| Format | Plain markdown, no YAML frontmatter |

## Output

After writing the file, summarize using this structure:

```markdown
## Created
- `AGENTS.md` (~N lines)

## Included
- [sections added and why]

## Omitted (agent can infer)
- [what was left out and source]

## Suggested next steps
- Run [fast verify command] to confirm commands work
- Add rules only when an agent repeats a mistake
```

Omit empty sections.

## Examples

**User:** "Create AGENTS.md for this repo"

→ Check for existing non-empty `AGENTS.md`. Inspect manifests and CI. Draft ~40 lines: commands, structure, boundaries. Skip README prose.

**User:** "Scaffold agent instructions — use pnpm, never touch /legacy/"

→ Include user constraints in Boundaries. Verify `pnpm` scripts in `package.json`.

**User:** "We already have AGENTS.md but it's messy"

→ Stop. Suggest `clean-up-rule` instead of overwriting.

**User:** "Add Cursor rules for TypeScript"

→ Out of scope. Suggest `create-rule`. This skill writes tool-agnostic `AGENTS.md` only.

## Additional resources

- Lean section skeleton: [template.md](template.md)
- Before/after examples: [examples.md](examples.md)
