# [Project Name]

[One line: primary language, framework, key dependencies with versions.]

## Commands

- `[install command]`: Install dependencies
- `[dev command]`: Start dev server
- `[test command]`: Full test suite
- `[single-test command]`: Run one test file or case
- `[lint command]`: Lint (and fix if applicable)
<!-- optional: migrate, build, typecheck -->
- `[migrate command]`: Run migrations

## Project structure

<!-- Delete this section if layout is self-evident. Max 5–8 bullets. -->

- `[path]/` — [responsibility]
- `[path]/` — [responsibility]

## Code style

<!-- Only rules that differ from language/framework defaults. -->

- [Non-default convention]
- See `[path/to/canonical-file]` as the pattern for [topic].

## Testing

<!-- optional: omit if fully covered under Commands -->

- Runner: [pytest | vitest | jest | etc.]
- `[fast single-test command]`
- Mock: [what to mock and what not to mock]

## Boundaries

### Always

- [Action the agent must do every time — e.g. run lint before finishing]

### Ask first

- [Action requiring human approval — e.g. schema migrations, dependency upgrades]

### Never

- [Hard limits — e.g. commit secrets, modify generated dirs, touch legacy modules]

## Git / PR

<!-- optional: omit unless repo has a clear convention in git log or CONTRIBUTING -->

- Branch: `[pattern]`
- Commits: `[format — e.g. conventional commits]`
- Before PR: `[verify commands]`
