---
name: code-review
description: >-
  Reviews provided code for bugs, errors, security issues, best practices,
  code smells, duplication, overengineering, and refactoring opportunities.
  Reads surrounding files and project conventions when needed. When no code or
  scope is provided, reviews staged changes (git diff --cached) by default. Use
  when the user invokes /code-review, attaches this skill, or asks for a
  structured code review.
disable-model-invocation: true
---

# Code Review

Structured review of provided code. Identify bugs, smells, duplication, overengineering, and refactoring options. Return a severity-rated report.

## Workflow

### 1. Intake

Identify what was provided:

| Input type | How to treat it |
|------------|-----------------|
| **No input (default)** | Review **staged changes** — run `git diff --cached` from the repo root |
| Pasted snippet | Review the snippet; note missing imports/types |
| Single file | Review the file; read full file if only a selection was highlighted |
| Git diff | Review changed lines; use `git diff` and `git log` for intent |
| PR / branch | Review the full diff from base branch |
| Directory | Review listed files; do not scan the entire repo unprompted |

**Default scope:** When the user invokes `/code-review`, attaches this skill, or asks for a review without pasting code, naming files, or specifying a diff, treat the review scope as staged changes. Do not ask what to review first — gather the staged diff and proceed.

If the staging area is empty, say so in one sentence and stop unless the user then asks to review unstaged changes, a branch, or something else. For any other unclear scope (not the default case), ask one short question before reviewing.

### 1a. Check for optional commit notes

The user may provide extra review context in a temporary notes file. After determining scope, inspect `git status --short` for a staged or untracked helper file with one of these names:

- `.commit-notes.md`
- `commit-notes.md`
- `COMMIT_NOTES.md`
- `.git-commit-notes.md`

If more than one exists, prefer the filename specified by any project rule. Otherwise prefer the first staged match at the repository root.

When a commit notes file exists:

1. Read it and treat its contents as authoritative context for intent, scope, breaking changes, migration or deploy steps, risks, and what to emphasize.
2. Cross-check claims against the diff — verify breaking changes are reflected in code and flag gaps (e.g. notes say migration required but diff has none).
3. Do not treat notes as proof that code is correct; still analyze the diff independently.
4. Mention briefly in the summary when commit notes shaped the review focus.

Do not unstage or modify commit notes files — that belongs to the commit workflow.

### 2. Define scope

Review the **provided code first**. Expand only when needed to judge correctness or conventions:

- Adjacent functions, classes, or types in the same file
- Callers or callees when behavior depends on them
- Tests covering the changed code
- Config that affects behavior (env, feature flags, schema)

Do **not** explore the whole codebase unless the user asks.

### 3. Gather context

When the snippet alone is insufficient:

1. Read imports, exports, and neighboring code in the same file
2. Check project conventions: `AGENTS.md`, `.cursor/rules/`, README, linter/formatter config
3. Find one similar implementation nearby and match its patterns
4. For git diffs, run in parallel from repo root:

```bash
git status --short
git diff --cached
git diff
git log -5 --oneline
```

When scope is the **default (no input)**, review only `git diff --cached`. For other git-scoped reviews, prefer staged diff when present; otherwise use the full working tree diff or branch diff as appropriate to the requested scope.

5. If step 1a found a commit notes file, read it before analyzing. Use its **Emphasize** and **Breaking Changes** sections to prioritize checklist dimensions (especially API & contracts, errors & edge cases, security).

### 4. Analyze

Walk every applicable dimension from [checklist.md](checklist.md). Skip dimensions that do not apply (e.g. SQL checks on a CSS file).

For each issue found, record:

- **Location** — file + line number or symbol name
- **Issue** — what is wrong
- **Why it matters** — impact on correctness, security, or maintainability
- **Suggested fix** — concrete, minimal change (diff snippet when helpful)
- **Confidence** — definite / likely / possible

### 5. Prioritize

Classify each finding:

| Severity | When to use |
|----------|-------------|
| **Critical** | Definite bug, security flaw, or data-loss risk; must fix before merge |
| **Major** | Likely bug or serious maintainability/security concern; should fix |
| **Minor** | Convention deviation or low-risk smell; fix when touching the code |
| **Suggestion** | Optional improvement; no correctness impact |

Mark lower-confidence items explicitly (e.g. "likely — needs runtime confirmation").

### 6. Report

Return the structured report below. Omit empty severity sections. Include at least one refactoring opportunity when duplication or smell exists.

**Letter each finding** so the user can reference it in follow-up messages (e.g. "fix B", "explain finding D"):

1. Assign letters **A**, **B**, **C**, … in report order: all **Critical** first, then **Major**, **Minor**, **Suggestions**
2. Prefix every finding bullet with its letter: `**A** — …`
3. Do not letter refactoring opportunities, positive notes, or open questions — only items under **Findings**
4. If there is only one finding, still use **A**

## Quick checklist

| Category | What to look for |
|----------|------------------|
| Correctness | Logic bugs, off-by-one, null/undefined, race conditions, wrong types |
| Errors & edge cases | Missing validation, swallowed errors, empty inputs, boundary values |
| Security | Injection, XSS, auth gaps, secrets in code, path traversal |
| Performance | N+1 queries, unnecessary re-renders, hot-path allocations, blocking I/O |
| API & contracts | Breaking changes, wrong signatures, inconsistent return shapes |
| Best practices | Framework idioms, clear naming, appropriate abstraction level |
| Code smells | Long functions, deep nesting, god objects, magic numbers |
| Overengineering | Unneeded abstractions, premature generalization, heavy deps for simple tasks |
| Duplication | Copy-paste logic, parallel implementations, repeated conditionals |
| Testing | Missing cases, brittle tests, untested branches |
| Maintainability | Dead code, unclear dependencies, missing docs on non-obvious logic |

Full prompts per dimension: [checklist.md](checklist.md). Sample outputs: [examples.md](examples.md).

## Output

Use this template:

```markdown
# Code Review: [scope]

## Summary
[2–4 sentences: overall quality, main risks, merge recommendation. Note if commit notes informed the focus.]

## Findings

### Critical
- **A** — **[file:line]** — [issue]. [why it matters]. **Fix:** [concrete suggestion].

### Major
- **B** — ...

### Minor
- **C** — ...

### Suggestions
- **D** — ...

## Refactoring opportunities

| Opportunity | Benefit | Effort |
|-------------|---------|--------|
| [description] | [why refactor] | low / medium / high |

## Positive notes
[What is done well]

## Open questions
[Assumptions or missing context that could change the verdict]
```

Omit sections with no findings (e.g. skip "Critical" if none). Keep "Positive notes" and "Open questions" when relevant.

## Guardrails

- Review only the provided code and context gathered in step 3 — do not invent issues elsewhere
- Prefer existing project patterns over generic best-practice advice
- Do not flag style that formatter/linter already enforces
- Separate definite bugs from hypotheses; label confidence
- Propose the **smallest correct fix**; suggest larger refactors only when smell or duplication warrants it
- Flag overengineering when complexity exceeds what the change needs; do not recommend ripping out abstractions the project already relies on elsewhere
- Balance criticism with positive notes when the code is genuinely well-written
- Do not run `git commit`, push, or modify code unless the user explicitly asks
- When the user later cites a finding letter (e.g. "fix B", "more on finding A"), resolve it against the lettered findings from the most recent review in the conversation

## Examples

See [examples.md](examples.md) for sample findings at Critical, Major, and Suggestion levels.
