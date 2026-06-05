---
name: code-review
description: >-
  Reviews provided code for bugs, errors, security issues, best practices,
  code smells, duplication, and refactoring opportunities. Reads surrounding
  files and project conventions when needed. Use when the user invokes
  /code-review, attaches this skill, or asks for a structured code review.
disable-model-invocation: true
---

# Code Review

Structured review of provided code. Identify bugs, smells, duplication, and refactoring options. Return a severity-rated report.

## Workflow

### 1. Intake

Identify what was provided:

| Input type | How to treat it |
|------------|-----------------|
| Pasted snippet | Review the snippet; note missing imports/types |
| Single file | Review the file; read full file if only a selection was highlighted |
| Git diff | Review changed lines; use `git diff` and `git log` for intent |
| PR / branch | Review the full diff from base branch |
| Directory | Review listed files; do not scan the entire repo unprompted |

If scope is unclear, ask one short question before reviewing.

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
git diff --cached
git diff
git log -5 --oneline
```

Use staged diff when present; otherwise full working tree diff.

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
| Duplication | Copy-paste logic, parallel implementations, repeated conditionals |
| Testing | Missing cases, brittle tests, untested branches |
| Maintainability | Dead code, unclear dependencies, missing docs on non-obvious logic |

Full prompts per dimension: [checklist.md](checklist.md). Sample outputs: [examples.md](examples.md).

## Output

Use this template:

```markdown
# Code Review: [scope]

## Summary
[2–4 sentences: overall quality, main risks, merge recommendation]

## Findings

### Critical
- **[file:line]** — [issue]. [why it matters]. **Fix:** [concrete suggestion].

### Major
- ...

### Minor
- ...

### Suggestions
- ...

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
- Balance criticism with positive notes when the code is genuinely well-written
- Do not run `git commit`, push, or modify code unless the user explicitly asks

## Examples

See [examples.md](examples.md) for sample findings at Critical, Major, and Suggestion levels.
