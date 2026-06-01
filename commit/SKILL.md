---
name: commit
description: >-
  Generate a proper git commit message from staged changes (or all unstaged
  changes if nothing is staged). Use when the user invokes /commit, asks for a
  commit message, or wants help describing changes before committing.
disable-model-invocation: true
---

# Commit message generation

Generate a commit message only. Do **not** run `git commit`, stage files, or push unless the user explicitly asks.

## Workflow

### 1. Capture user context

If the user provided extra context (intent, ticket ID, scope, breaking changes, what to emphasize), treat it as authoritative input when drafting the message.

If context is missing and the diff is ambiguous, ask one short clarifying question before finalizing.

### 2. Read the changes

Run these commands **in parallel** from the repository root:

```bash
git status --short
git diff --cached
git diff
git log -10 --oneline
```

**Which diff to use:**

| Situation | Analyze |
|-----------|---------|
| Staged changes exist (`git diff --cached` non-empty) | Staged diff only |
| Nothing staged | Full working tree: staged + unstaged (`git diff` and untracked via status) |

For untracked files, use `git status` and read new file contents only when needed to understand the change.

**Do not** explore the codebase beyond git output unless the diff alone is insufficient.

### 3. Draft the commit message

**Match repository style** using recent `git log` output (subject format, scope prefixes, imperative mood, line length).

**Default format** (when the repo has no clear convention):

```
<type>(<optional-scope>): <short imperative summary>

<optional body: 1–3 sentences on why, not a file list>
```

Types: `feat`, `fix`, `refactor`, `style`, `docs`, `test`, `chore`, `build`, `ci`.

**Quality rules:**

- Subject: imperative, ≤72 characters, no trailing period
- Focus on **why** and user-visible impact, not a list of touched files
- One logical change per message; if the diff mixes unrelated concerns, say so and suggest splitting
- Do not mention secrets, credentials, or `.env`-like files
- Accurate verbs: `add` = new capability, `fix` = bug, `update`/`refactor` = behavior-preserving change

## Output

Return:

1. **Suggested commit message** — ready to paste, in a single fenced block
2. **Brief rationale** — 1–2 sentences on type, scope, and what you emphasized from the user’s context (if any)

If the user asked to commit as well, follow their project or user rules for staging and `git commit` (HEREDOC message, hooks, no `--no-verify` unless requested).

## Examples

**User context:** “Fixes hero overlap on mobile, closes #142”

**Staged diff:** CSS media-query change in hero component

```
fix(hero): prevent content overlap on small viewports

Adjust breakpoint spacing so the hero stack clears the nav. Closes #142.
```

**No user context; unstaged only** — mixed rename + lint in one file

Suggest splitting or pick the dominant change; call out that both rename and lint are present if a single message would mislead.
