---
name: manual-testing-plan
description: >-
  Creates or updates a manual testing plan from code under review. Defaults to
  staged changes (git diff --cached); also works on named files, pasted diffs,
  PRs, or branches. Reads surrounding code for context, then writes or updates
  a checklist covering happy paths, edge cases, regressions, and risk areas.
  Use whenever the user asks to create, write, generate, or update a manual
  test plan, testing plan, QA checklist, manual-test-plan.md, or what to test
  for the current changes; also when finishing an implementation and the user
  needs concrete steps to verify the change. Read this skill immediately and
  follow it before drafting any manual test plan.
---

# Manual Testing Plan

Produce a thorough manual QA checklist from code under review. Write the plan to a markdown file; keep the chat reply short.

## Scope

| Input | Scope |
|-------|-------|
| **No files specified** (default) | Staged hunks only — `git diff --cached` |
| **Files / attachments named** | Named or attached paths (full files or highlighted ranges) |
| **Unstaged / branch / PR asked** | `git diff`, branch vs base, or PR diff as requested |
| Pasted snippet or git diff | That content; expand surrounding files only as needed for context |

**Default scope:** When the user invokes `/manual-testing-plan`, attaches this skill, or asks for a manual test plan without naming files or a diff, use staged changes. Do not ask what to plan first — gather the staged diff and proceed.

If the staging area is empty on default scope, say so in one sentence and stop unless the user then asks to use unstaged changes, specific files, a branch, or something else. For any other unclear scope (not the default case), ask one short question before continuing.

## Hard rules

- **Read-only on product code** — do not edit application source while producing the plan.
- **Never stage, commit, or push** the plan file unless the user explicitly asks.
- **Manual only** — produce human QA steps. You may briefly note existing automated tests for awareness; do not write or run automated test suites unless the user asks separately.
- **Do not invent requirements** that are not implied by the code or commit notes. If intent is ambiguous, ask one short question before finalizing the file.

## Workflow

### 1. Intake

From the repository root, resolve scope and gather diffs. Run in parallel as applicable:

```bash
git status --short
git diff --cached
git diff
git log -5 --oneline
```

On **default scope**, use only `git diff --cached`. For other git-scoped requests, use the diff that matches what the user asked for.

### 1a. Check for optional commit notes

Inspect `git status --short` for a staged or untracked helper file with one of these names:

- `.commit-notes.md`
- `commit-notes.md`
- `COMMIT_NOTES.md`
- `.git-commit-notes.md`

If more than one exists, prefer the filename specified by any project rule. Otherwise prefer the first staged match at the repository root.

When a commit notes file exists:

1. Read it and treat its contents as authoritative context for intent, scope, breaking changes, migration or deploy steps, risks, and what to emphasize.
2. Cross-check claims against the diff — flag gaps (e.g. notes say migration required but diff has none).
3. Mention briefly in the chat summary when commit notes shaped the plan.

Do not unstage or modify commit notes files.

### 2. Build context

For each changed area, read enough surrounding code to know **what behavior changed**, not just the diff lines:

- Adjacent functions, classes, or types in the same file
- Callers, routes, UI entry points, or API handlers tied to the change
- Related permissions, feature flags, schema, or config
- Existing tests that document expected behavior (for awareness only)

Do **not** scan the whole repository unprompted.

### 3. Derive the test surface

Map the change to behaviors a human can verify:

- User-visible flows (UI, CLI, docs-facing behavior)
- APIs and contracts (request/response, status codes, errors)
- Permissions and roles
- Data / migrations / backfill
- Error paths and invalid input
- Adjacent regression risk from the same modules

Cover **everything the diff can affect** that is manually verifiable — not every theoretical edge in the product.

### 4. Write the plan file

**Default path:** `manual-test-plan.md` at the repository root.

If the user specifies another path, use that instead. Edit only that plan file.

**Updating an existing plan:** If the target file already exists, read it first and update in place:

1. Revise **Scope**, **Setup**, cases, and risk notes to match the current change.
2. Add new cases for newly affected behavior; remove or rewrite cases that no longer apply.
3. **Uncheck** (`[ ]`) every item that was modified, added, or is otherwise relevant to the new change — including cases whose steps or expected results changed. Leave unrelated checked items (`[x]`) checked so prior verification is preserved.
4. Do not re-check items for the user.

Use the template below for new files. Skip sections that do not apply rather than padding with N/A. Prefer concrete steps (navigate X, do Y, expect Z) over vague “test the feature.” Group by feature/area. Use checkboxes. Include regression only for **adjacent** behavior touched by the change.

```markdown
# Manual test plan

## Scope
- What changed (1–3 bullets from the diff)
- Environment / preconditions

## Setup
- Accounts, data, flags, or migrations needed before testing

## Test cases

### [Area or feature name]
- [ ] **Happy path** — steps → expected result
- [ ] **Edge / invalid input** — steps → expected result
- [ ] **Error / failure** — steps → expected result
- [ ] **Permissions / roles** — only if relevant
- [ ] **Regression** — nearby behavior that must still work

## Cross-cutting checks
- [ ] Only include items that apply (e.g. mobile/responsive, a11y, concurrency, idempotency, rollback)

## Out of scope
- Explicitly list what this plan does *not* cover (unchanged areas, deferred work)

## Risk notes
- Highest-risk paths and why (from the diff + context)
```

### 5. Summarize in chat

Reply briefly:

- Path to the written plan file
- Approximate number of test cases / areas
- Highest-risk paths to test first
- Note if commit notes informed the plan
- When updating an existing plan, note how many items were unchecked for re-test

Full detail stays in the file — do not dump the entire checklist into chat.

## Coverage rules

- Concrete, tickable steps with expected results
- One job per case; avoid mega-scenarios that mix unrelated flows
- Permissions, migrations, and failure modes only when the diff implies them
- Call out environment/setup blockers in **Setup** before the cases
- If the change is pure refactor with no behavior change, say so and list focused regression checks only
