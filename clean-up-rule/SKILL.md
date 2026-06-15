---
name: clean-up-rule
description: >-
  Clean up existing Cursor rules: tighten .mdc structure, improve descriptions
  and globs, remove redundancy, and apply progressive disclosure without changing
  rule intent. Use when the user asks to clean up, polish, tighten, or simplify
  a rule, .mdc file, AGENTS.md, .cursor/rules/, or user rules.
disable-model-invocation: true
---

# Clean Up Rules

Improve target rule files cosmetically and remove redundant instructions without changing what the rule tells the agent to do. Apply confident intent-preserving cleanups; report uncertain cases for user review. The user reviews edits manually.

For code cleanup, use `clean-up`. For skill cleanup, use `clean-up-skill`. For authoring new rules, see `create-rule`.

## Scope

Determine what to clean up before editing:

| Input | Scope |
|-------|-------|
| **Rule file specified** | That `.mdc` file, or `AGENTS.md` if named |
| **`.cursor/rules/` directory specified** | All `.mdc` files in that directory (and nested subdirs if the user names a tree) |
| **User rule text specified** | The provided rule content only — do not edit other user rules |
| **No path specified** | Ask which rule file, directory, or user rule to clean up |

**Path references** include `@`-attached files, explicit paths (`.cursor/rules/foo.mdc`, `AGENTS.md`), globs the user names, or a rule name they provide. Do not expand to unrelated rules unless the user asked.

Project rules live in `.cursor/rules/*.mdc`. `AGENTS.md` at the repo root is always-applied agent guidance — treat it like a rule when the user names it.

## Hard rules

- **Preserve rule intent**: Same enforcement outcomes, constraints, examples, and guidance the rule defines.
- **Preserve verbatim user text**: If the rule contains exact wording the user supplied, keep it word-for-word. Do not paraphrase, soften, or reorder that text.
- **Never stage or commit** unless the user explicitly asks in the same message.
- **Leave edits unstaged**: The user reviews agent changes and stages them manually.

## Quality bar

Read `create-rule` for full authoring rules. When cleaning, verify:

| Check | Target |
|-------|--------|
| `description` | Clear, specific; states what the rule enforces |
| `globs` | Present and accurate for file-specific rules; omitted only when `alwaysApply: true` |
| `alwaysApply` | `true` only for universal guidance; `false` when scoped by globs |
| Rule length | Under ~50 lines per rule when possible; split only with user approval |
| One concern | Each rule covers a single topic — flag multi-topic rules for splitting |
| Actionable | Imperative guidance, not generic programming tutorials |
| Examples | Concrete good/bad examples where the rule teaches a pattern |
| Terminology | One term per concept throughout |
| Paths | Forward slashes, not Windows-style |

## Allowed cleanup

**Cosmetic:**

- Frontmatter formatting and field ordering
- Heading hierarchy and markdown structure
- Whitespace, list formatting, table alignment, link style
- Consistent terminology (pick one term and apply it)
- Code fence language tags in examples

**Content (intent-preserving only):**

- Explanations the agent already knows (generic programming concepts, tool basics)
- Duplicate paragraphs repeated within the same rule or copied from `AGENTS.md`
- Redundant examples that restate the same pattern
- Obsolete sections left behind by recent project changes
- Over-long `description` fields that can be tightened without losing meaning
- Verbose always-applied rules where detail can move to a scoped `.mdc` file (report first if the move changes when the rule loads)
- Stale file paths, function names, or globs that no longer match the codebase (fix only when clearly outdated from inspection)

## Not allowed unless the user explicitly asks

- Changing what the rule enforces (requirements, constraints, workflow steps)
- Paraphrasing verbatim user-supplied text
- Changing `globs`, `alwaysApply`, or `description` when the change alters when or how the rule applies
- Splitting one rule into multiple files or merging rules
- Migrating rules to skills (use `migrate-to-skills` instead)
- Editing user rules in Cursor settings via `cursor_dialog` without the user naming the rule and approving changes

## Cleanup opportunities

Scan target content for these patterns. **Apply when confident; report when uncertain.**

| Category | Apply when confident | Report when uncertain |
|----------|---------------------|----------------------|
| Token waste | Generic explanations, duplicated paragraphs, three examples of the same pattern | Example might document a non-obvious edge case |
| Length | Rule well over 50 lines with a clearly separable second topic | Split might break intentional cross-references |
| Description | Vague or missing what/when; first-person wording | Tightening might drop nuance the team relies on |
| Globs | Obvious typo or path that no longer exists in the repo | Glob might be intentional for not-yet-added files |
| Cross-rule overlap | Same paragraph in two rules after a recent copy | Overlap might be intentional reinforcement |
| Stale references | Dead file paths, renamed helpers, removed workflow steps | Reference might apply on another branch |
| Conflicting rules | Two rules give contradictory instructions after a recent edit | Conflict might reflect intentional conditional logic |
| AGENTS.md bloat | Content duplicated in a scoped `.mdc` rule | AGENTS.md entry might be intentionally always-on |

**Confidence bar:** apply only when simplification is obvious from the target rule plus `create-rule` rules and minimal local context (e.g. confirm a path is gone). If intent is ambiguous, **report only** — do not edit.

## Workflow

### 1. Determine scope and inspect

Identify rule files in scope. Search nested `.cursor/rules/` when the user names a directory or project.

```bash
# If in a git repo, run in parallel for context:
git status --short
git diff -- <rule-paths>
```

Read each target file in full. When cleaning a directory, skim sibling rules for cross-rule duplication to report — do not edit siblings unless in scope.

For user rules, read the text the user provided or list rules via `cursor_dialog` only when the user asks to clean user rules and names which one.

### 2. Clean up target content

**Pass A — Cosmetic:** frontmatter, headings, formatting, terminology, paths.

**Pass B — Content:** for each file in scope:

1. Compare against the quality bar and cleanup opportunities table.
2. Check for verbatim user text — skip those passages.
3. Optionally verify stale paths or symbol names against the repo when the rule references concrete files.
4. Apply confident intent-preserving cleanups.
5. Collect uncertain items for the report; do not edit those.

### 3. Confirm state

If in a git repo, run `git status --short` and verify agent changes are unstaged. If anything was accidentally staged, run `git restore --staged <path>`.

## Output

When finished, summarize using this structure:

```markdown
## Scope
- [rule file(s) or directory cleaned]

## Cosmetic cleanup
- [bullets]

## Content cleanup applied
- [file: what was removed/simplified and why intent is preserved]

## Quality checks
- [pass/fail or fixes applied: description, globs, length, one-concern, terminology]

## Cleanup opportunities (needs your review)
- [file: what looks redundant, stale, or ambiguous; why uncertain; suggested action]

## Status
- Files modified: ...
- All agent edits are **unstaged**
```

Omit empty sections.

## Examples

**User:** "Clean up @.cursor/rules/seed-legacy-id-mapping.mdc"

→ Read the full file. Tighten tables, remove a duplicated helper list, fix a stale path. Do not change resolution-key semantics or glob scope.

**User:** "Polish all rules in `.cursor/rules/` — they're too verbose"

→ Clean every `.mdc` in scope. Remove generic Payload/git explanations the agent already knows. Report two rules that repeat the same bootstrap warning; do not merge without confirmation.

**User:** "Clean up this rule and make it always apply"

→ Content cleanup only. Report that changing `alwaysApply` or removing `globs` is a behavioral change requiring explicit confirmation; do not edit those fields unless the user confirms.

**User:** "Clean up my user rule about commit messages"

→ List user rules if needed, edit only the named rule after showing the user the proposed text. Do not touch project `.mdc` files.

**Uncertain case:** Rule is 120 lines covering seed conventions and API error handling.

→ Report as a split candidate (seed rule + API rule). Do not split without confirmation.
