---
name: clean-up-skill
description: >-
  Clean up existing Cursor Agent Skills: tighten SKILL.md structure,
  improve descriptions and triggers, remove redundancy, and apply progressive
  disclosure without changing skill intent. Use when the user asks to clean up,
  polish, tighten, or simplify a skill, SKILL.md file, or skill directory.
disable-model-invocation: true
---

# Clean Up Skills

Improve target skill files cosmetically and remove redundant instructions without changing what the skill tells the agent to do. Apply confident intent-preserving cleanups; report uncertain cases for user review. The user reviews edits manually.

For code cleanup, use `clean-up` instead. For authoring new skills, see `create-skill`.

## Scope

Determine what to clean up before editing:

| Input | Scope |
|-------|-------|
| **Skill directory specified** | Entire directory: `SKILL.md` plus supporting files (`reference.md`, `examples.md`, `scripts/`, etc.) |
| **SKILL.md or file specified** | That file only, unless the user names the whole skill |
| **No path specified** | Ask which skill or `SKILL.md` to clean up |

**Path references** include `@`-attached files, explicit paths (`~/.cursor/skills/foo/`, `.cursor/skills/foo/`), or a skill name the user names. Do not expand to unrelated skills unless the user asked.

## Hard rules

- **Preserve skill intent**: Same workflow outcomes, hard constraints, trigger semantics, and user-facing behavior the skill defines.
- **Preserve verbatim user text**: If the skill contains exact wording the user supplied, keep it word-for-word. Do not paraphrase, soften, or reorder that text.
- **Never create skills in `~/.cursor/skills-cursor/`** — that directory is reserved for Cursor built-ins.
- **Never stage or commit** unless the user explicitly asks in the same message.
- **Leave edits unstaged**: The user reviews agent changes and stages them manually.

## Quality bar

Read `create-skill` for full authoring rules. When cleaning, verify:

| Check | Target |
|-------|--------|
| `name` | Lowercase, hyphens, max 64 chars; matches directory name |
| `description` | Third person; includes WHAT and WHEN; specific trigger terms; max 1024 chars |
| `SKILL.md` length | Under 500 lines |
| Progressive disclosure | Essential workflow in `SKILL.md`; detail in sibling files |
| References | One level deep from `SKILL.md` only |
| Terminology | One term per concept throughout |
| Paths | Forward slashes, not Windows-style |

## Allowed cleanup

**Cosmetic:**

- Frontmatter formatting and field ordering
- Heading hierarchy and markdown structure
- Whitespace, list formatting, link style
- Consistent terminology (pick one term and apply it)
- Import/script path formatting in examples

**Content (intent-preserving only):**

- Explanations the agent already knows (generic programming concepts, tool basics)
- Duplicate instructions repeated across `SKILL.md` and a reference file
- Redundant examples that restate the same pattern
- Obsolete sections left behind by recent workflow changes
- Over-long descriptions that can be tightened without losing trigger terms
- Sections that belong in `reference.md` or `examples.md` per progressive disclosure
- Unused supporting files or scripts no longer referenced anywhere in the skill

## Not allowed unless the user explicitly asks

- Changing what the skill instructs the agent to do (workflow steps, hard rules, output format)
- Adding, removing, or reordering workflow steps that alter behavior
- Paraphrasing verbatim user-supplied text
- Changing `disable-model-invocation` (affects auto-discovery)
- Renaming the skill (`name` field or directory) without explicit request
- Merging or splitting skills

## Cleanup opportunities

Scan target content for these patterns. **Apply when confident; report when uncertain.**

| Category | Apply when confident | Report when uncertain |
|----------|---------------------|----------------------|
| Token waste | Generic explanations, duplicated paragraphs, three examples of the same pattern | Example might document a non-obvious edge case |
| Structure | Section belongs in `reference.md`; `SKILL.md` over 500 lines with movable detail | Section might be intentionally kept in main file for discoverability |
| Description | Vague triggers, missing WHEN clause, first-person wording | Tightening might drop a trigger term the user relies on |
| Redundant files | `reference.md` fully duplicated in `SKILL.md`; unreferenced script | File might be used externally or planned for future use |
| Conflicting rules | Two sections give contradictory instructions after a recent edit | Conflict might reflect intentional conditional logic |
| Stale content | Examples reference removed workflow steps | Example might still apply to a branch not visible in scope |

**Confidence bar:** apply only when simplification is obvious from the target skill plus `create-skill` rules. If intent is ambiguous, **report only** — do not edit.

## Workflow

### 1. Determine scope and inspect

Identify the skill directory and read all files in scope:

```bash
# If the skill is in a git repo, run in parallel for context:
git status --short
git diff -- <skill-paths>
```

Read `SKILL.md` in full. Read supporting files (`reference.md`, `examples.md`, scripts) when they exist or when `SKILL.md` links to them.

### 2. Clean up target content

**Pass A — Cosmetic:** frontmatter, headings, formatting, terminology, paths.

**Pass B — Content:** for each file in scope:

1. Compare against the quality bar and cleanup opportunities table.
2. Check for verbatim user text — skip those passages.
3. Apply confident intent-preserving cleanups.
4. Move detailed content to sibling files when `SKILL.md` is long and the move is clearly safe.
5. Collect uncertain items for the report; do not edit those.

### 3. Confirm state

If in a git repo, run `git status --short` and verify agent changes are unstaged. If anything was accidentally staged, run `git restore --staged <path>`.

## Output

When finished, summarize using this structure:

```markdown
## Scope
- [skill directory or files cleaned]

## Cosmetic cleanup
- [bullets]

## Content cleanup applied
- [file: what was removed/moved/simplified and why intent is preserved]

## Quality checks
- [pass/fail or fixes applied: description, line count, progressive disclosure, terminology]

## Cleanup opportunities (needs your review)
- [file: what looks redundant or ambiguous, why uncertain, suggested action]

## Status
- Files modified: ...
- All agent edits are **unstaged**
```

Omit empty sections.

## Examples

**User:** "Clean up @clean-up/SKILL.md"

→ Read the full skill directory. Tighten description triggers, fix heading hierarchy, remove a duplicated workflow paragraph also present in a reference file. Do not change hard rules or git workflow steps.

**User:** "Polish my commit skill — it's too verbose"

→ Read `commit/SKILL.md`. Remove generic git explanations the agent already knows. Keep commit-message format templates and the staged-vs-unstaged diff rule intact.

**User:** "Clean up this skill and make it auto-invoke"

→ Content cleanup only. Report that changing `disable-model-invocation` is a behavioral change requiring explicit confirmation; do not edit the field unless the user confirms.

**Uncertain case:** `SKILL.md` has two similar example sections — one minimal, one detailed.

→ Report both; suggest keeping the detailed one in `examples.md` and one concise example in `SKILL.md`. Do not delete either without confirmation.
