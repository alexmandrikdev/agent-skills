---
name: clean-up-staged-changes
description: >-
  Clean up staged changes cosmetically and apply safe behavior-neutral logic
  simplifications. Use when the user asks to clean up staged changes, tidy the
  staging area, polish staged diffs, remove redundant steps, simplify logic,
  or improve what is staged before committing.
---

# Clean Up Staged Changes

Improve staged work cosmetically and simplify redundant logic without touching git staging or altering observable behavior. Apply confident behavior-neutral logic cleanups; report uncertain cases for user review. The user reviews edits and stages manually.

## Hard rules

- **Preserve observable behavior**: Same outputs, side effects, error handling semantics, and public API contracts.
- **Logic simplification allowed** only when provably behavior-neutral (redundant work removal, not bug fixes or feature changes).
- **Never stage**: Do not run `git add`, `git add -A`, `git add -u`, or any variant.
- **Never commit**: Do not run `git commit` unless the user explicitly asks in the same message.
- **Never push**: Do not push unless the user explicitly asks.

Leave all agent edits **unstaged** so the user can review and stage themselves.

## Allowed cleanup

**Cosmetic:**

- Formatting and whitespace
- Naming (variables, functions) when meaning is unchanged
- Comments and docstrings
- Import order and unused imports
- Small readability edits that do not alter what the code does at runtime

**Logic (behavior-neutral only):**

- Dead code removal when provably unreachable
- Redundant intermediate steps (duplicate validation, double fetch/compute, no-op re-assignment)
- Collapsed no-op sequences
- Obsolete branches, helpers, or workarounds left behind by the staged refactor
- Unused variables, imports, helpers, or parameters tied to removed logic

## Not allowed unless the user explicitly asks

- Bug fixes, feature changes, or refactors that alter behavior
- New branches, early returns, or reordered side effects
- Changed function signatures, defaults, or exported types

## Logic cleanup opportunities

Scan staged hunks for these patterns. **Apply when confident; report when uncertain.**

| Category | Apply when confident | Report when uncertain |
|----------|---------------------|----------------------|
| Redundant steps | Duplicate validation, double fetch/compute, re-assignment with no effect | Step might be intentional defense-in-depth |
| Dead paths | Unreachable branches after early return/guard; code after `return`/`throw` | Branch might handle rare edge case not visible in diff |
| Obsolete workarounds | Commented "temporary" shim superseded by staged change | Shim might still be needed for another code path |
| Unused artifacts | Variables, imports, helpers, params only used by removed logic | Name suggests future use or external contract |
| Redundant abstraction | Wrapper that only passes through after refactor | Wrapper might enforce policy/logging |

**Confidence bar:** apply only when removal or simplification is obvious from staged content plus minimal local context. If intent is ambiguous, **report only** — do not edit.

If staged code has a logic bug or needs a behavioral fix, report it instead of fixing it (unless the user explicitly asked for that fix).

## Workflow

### 1. Inspect what is staged

From the repository root, run in parallel:

```bash
git status --short
git diff --cached
git log -5 --oneline
```

If nothing is staged, say so and ask whether to work from unstaged changes or wait for the user to stage files.

### 2. Clean up staged content

Scope edits to files reflected in `git diff --cached`. Match existing project conventions.

**Pass A — Cosmetic:** apply formatting, naming, comments, imports, and readability edits within staged hunks.

**Pass B — Logic:** for each staged file:

1. Read the full file (or enough surrounding lines) to understand control flow around staged hunks. Exploration is limited to **staged files** — not the whole repo.
2. Optionally read direct callers of changed symbols only when needed to confirm a step is unused.
3. Apply confident behavior-neutral logic cleanups from the checklist above.
4. Collect uncertain items for the report; do not edit those.

### 3. Confirm unstaged state

After edits, run:

```bash
git status --short
```

Verify agent changes appear as unstaged modifications (`M` in the second column, not the first). If anything was accidentally staged, run `git restore --staged <path>` to unstage it.

## Output

When finished, summarize using this structure:

```markdown
## Cosmetic cleanup
- [bullets]

## Logic cleanup applied
- [file: what was removed/simplified and why it's behavior-neutral]

## Logic opportunities (needs your review)
- [file: what looks redundant, why uncertain, suggested action]

## Status
- Files modified: ...
- All agent edits are **unstaged**
```

Omit empty sections. If no logic changes were made or reported, say so briefly.

## Examples

**User:** "Clean up the staged changes."

→ Read `git diff --cached`. Apply cosmetic edits and confident logic simplifications (e.g. remove unreachable branch left by refactor). Report uncertain redundancies (e.g. old API call kept alongside new one). Do not `git add`.

**User:** "Clean up the staged changes and fix the null check bug."

→ Cosmetic and logic cleanup still apply. The explicit bug-fix request permits the logic change for that null check only.

**User:** "Polish what's staged and commit."

→ Follow this skill for cleanup (no staging). Commit only because the user explicitly asked; still prefer not staging unless their commit workflow requires it.

**Uncertain logic case:** staged change adds a new API call but keeps the old call "just in case."

→ Report the old call as a possible redundant step; do not remove it without confirmation.
