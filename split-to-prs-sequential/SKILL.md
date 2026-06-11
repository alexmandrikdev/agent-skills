---
name: split-to-prs-sequential
description: >-
  Split a branch or commit range into sequential reviewable PRs using
  cherry-picks off updated main, with commit-to-branch mapping, descriptive
  branch names, per-PR title/description approval, merge checkpoints between
  slices, letter-keyed action menus at each gate, and optional branch cleanup.
  Use when the user asks to split a branch into PRs, split commits into PRs,
  cherry-pick commits into PRs, or invokes split-to-prs-sequential.
disable-model-invocation: true
---

# Split to PRs (Sequential)

Turn one branch or commit pile into small, sequential PRs. Each slice is cherry-picked onto updated main, one at a time, with approval gates before and after each PR.

## Hard rules

- Do not create branches, commit, push, or open PRs until the user approves the split plan.
- Never discard user work. No destructive git commands (`reset --hard`, `clean -fdx`, branch deletion, force-push, history rewrite) without explicit approval.
- Always save a recoverable snapshot before moving work around.
- Stage only named files or hunks. No `git add .` / `git add -A`.
- Stacked PRs are **not supported** — always branch from updated default branch after prior PR merges.

## Workflow overview

Four approval gates:

1. **Plan** — commit mapping, branch names, execution order
2. **PR metadata** — title and body per slice (before `gh pr create`)
3. **Merge** — user confirms merge before starting the next slice
4. **Cleanup** — optional branch deletion after all slices are merged

At **every gate**, end with an **action menu** — one letter per option (see [Action menus](#action-menus)). Accept a letter alone (`A`), letter + note (`R combine PR 2 and 3`), or the full phrase. Match case-insensitively.

Parent agent orchestrates; each slice's git/gh work runs in a **shell subagent** with minimal context.

## Action menus

Assign a **single uppercase letter** to each option at every stop. Use mnemonic letters when possible. Letters are **scoped to the current gate** — the same letter may mean different things at different gates.

**Format** — always append this block when waiting for the user:

```markdown
**Actions**
- **A** — …
- **B** — …
```

Rules:

- List only actions valid **right now** (do not show merge options during plan review).
- Primary forward action first (**A** or **M**).
- Destructive or abort paths last, marked clearly.
- If the user sends only a letter, treat it as that action; if ambiguous, ask once.
- Synonyms still work: at the merge gate, **M**, **C**, **N**, or phrases like "merged", "continue", and "next" all proceed to the next slice.

### Gate 1 — Plan approval

```markdown
**Actions**
- **A** — Approve plan; start backup and first slice
- **R** — Revise plan (describe changes to slices, order, or branch names)
- **X** — Cancel split (no further git/gh work)
```

### Gate 2 — PR metadata approval

```markdown
**Actions**
- **A** — Approve title and body; create PR
- **E** — Edit metadata (provide new title, body, or edits)
- **X** — Cancel split
```

### Gate 3 — Merge checkpoint

After returning the PR URL:

```markdown
**Actions**
- **M** — Merged; proceed to next slice (or final report if this was the last)
- **V** — Verify merge status on GitHub
- **W** — Not merged yet; wait (no next slice)
- **X** — Cancel split (stop after current PR)
```

Do not start the next slice until **M** or an explicit equivalent ("merged", "continue", "next").

### Gate 4 — Cleanup approval

```markdown
**Actions**
- **Y** — Yes — delete all listed slice branches (local + remote)
- **S** — Slice branches only — do not delete source branch
- **R** — Revise list (say which branches to add/remove)
- **N** — No cleanup
```

When proposing source-branch deletion, add:

```markdown
- **D** — Also delete source branch `<name>`
```

When a local branch is not fully merged and `-d` fails:

```markdown
**Actions**
- **F** — Force delete local branch (`git branch -D`)
- **K** — Keep branch; skip local delete (still delete remote if approved)
```

### Gate — Cherry-pick conflict

When the subagent reports a conflict:

```markdown
**Actions**
- **F** — Fixed locally; retry this slice
- **R** — Revise plan (re-map commits or reorder slices)
- **X** — Cancel split
```

## Phase 1: Analyze and propose plan

Run from repo root **in parallel**:

```bash
git status --short
git branch --show-current
git branch -a
git log main..HEAD --oneline
git diff main...HEAD --stat
```

Detect default branch via `git symbolic-ref refs/remotes/origin/HEAD` when not `main`.

Check ownership signals (`CODEOWNERS`, nested ownership files, `tools/ownership/PRODUCTOWNERS`, or repo equivalents) for reviewer boundaries.

Use chat history to recover intent. Optimize for reviewer-aligned slices with minimal unrelated diff.

### Plan template

Every slice must list commits explicitly and propose a descriptive branch name:

```markdown
# Split plan: [source branch]

## Summary
[N commits → N PRs, sequential off main]

## Slices

### PR 1 — [title]
| Commit | Subject |
|--------|---------|
| `abc1234` | feat(seed): add foundation |
| `def5678` | feat(seed): add tags |

**Base:** `main` (updated before branch)
**Branch name:** `feat/seed-foundation-tags`

### PR 2 — [title]
| Commit | Subject |
|--------|---------|
| `4250d3d` | feat(photos): add collection |

**Branch name:** `feat/photos-collection-legacy-seed`

## Execution order
1. PR 1 → wait for merge → PR 2 → ...
```

Include a Mermaid diagram when dependencies help reviewers, even though execution is sequential off merged main.

Append the Gate 1 action menu after the plan.

**Stop and ask for plan approval** (Gate 1 action menu). Do not mutate git state until **A**.

### Branch naming

Each slice needs a **descriptive branch name** — not `split/pr1-...`, not numbered placeholders.

**Default pattern:** `<type>/<kebab-case-description>`

| Part | Rules |
|------|-------|
| `type` | Dominant change: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `ci`, `build` |
| `description` | 2–5 words; lowercase; hyphens only |

Derive from slice scope, proposed PR title, and cherry-picked commits. Branch name and PR title should tell the same story (branch is slightly shorter).

Sample existing branches (`git branch -a`) and follow repo conventions when present (ticket prefixes, username prefixes). Otherwise use `<type>/<description>`.

Constraints: ~60 chars max; no trailing or double hyphens; avoid generic names (`feat/update`). Check for collisions before push.

Branch names are part of plan approval — user can revise them with the rest of the plan.

## Phase 2: Backup (once, after plan approval)

```bash
SHA=$(git stash create "pre-split")
if [ -n "$SHA" ]; then
  git update-ref "refs/backup/pre-split-$(date +%s)" "$SHA"
fi
```

Record source branch tip SHA in the plan for recovery.

## Phase 3: Per-slice execution

For **each approved slice**, in order:

### 3a. Subagent — git work

Launch a Task subagent (`subagent_type: shell`) with **only**:

- Slice index and approved branch name
- Commit SHAs to cherry-pick (in order)
- Repo path
- Default branch name

Subagent steps:

1. `git fetch origin <default-branch>`
2. `git checkout <default-branch> && git pull origin <default-branch>`
3. `git checkout -b <branch-name>` (verify name not taken; append suffix if collision)
4. `git cherry-pick <sha>` for each commit — stop and report on conflict; do not force-resolve
5. `git push -u origin HEAD`

Return: branch name, pushed tip SHA, cherry-pick log, any conflicts.

Do **not** open the PR in this subagent.

### 3b. Parent — draft PR metadata

Draft title and body from cherry-picked commits and repo `git log` style:

```markdown
## Proposed PR

**Title:** feat(scope): ...

**Body:**
## Summary
- ...

## Test plan
- [ ] ...
```

Append the Gate 2 action menu after the draft.

**Stop for user approval** of title and body (Gate 2 action menu).

### 3c. Subagent — create PR

After metadata approval, launch a shell subagent:

```bash
gh pr create --title "..." --body "$(cat <<'EOF'
...
EOF
)"
```

Return the PR URL. Request `network`/`all` permissions for push and `gh`.

### 3d. Merge checkpoint

After returning the PR URL, **stop and wait** (Gate 3 action menu). Do **not** poll GitHub unless the user chooses **V**. Do **not** start the next slice until **M** or an explicit equivalent.

If the user asks, verify merge read-only:

```bash
gh pr view <number> --json state,mergedAt,url
```

## Phase 4: Final report

When all slices are merged:

- List PR titles and URLs in order
- Note anything left on the source branch or working tree
- List branches still present (slice branches, source branch, backup ref)

## Phase 5: Optional cleanup

After the final report, **ask whether to clean up branches**. Do not delete anything until the user explicitly approves.

### Propose cleanup

Show exactly what would be removed:

```markdown
## Cleanup proposal

**Slice branches (local + remote):**
- `feat/seed-foundation-tags`
- `feat/photos-legacy-seed`
- ...

**Also delete source branch?** `wip3` — yes / no (default: no)

**Keep:** backup ref `refs/backup/pre-split-*` unless user asks to remove it
```

Default cleanup scope: **slice branches only** (every branch name from the approved plan). Offer source-branch deletion as an explicit opt-in — never delete it by default.

Append the Gate 4 action menu (include **D** when offering source-branch deletion).

**Stop for cleanup approval** (Gate 4 action menu).

### Execute cleanup (subagent)

After approval, launch a shell subagent with the approved branch list and repo path.

For each branch:

1. Delete local (if exists): `git branch -d <branch>` — if not fully merged, stop and report; use `-D` only if user explicitly approves force delete
2. Delete remote (if exists): `git push origin --delete <branch>`

If user approved source branch deletion, include it in the same pass.

To remove backup ref (only when user explicitly asks):

```bash
git update-ref -d refs/backup/pre-split-<timestamp>
```

Return: branches deleted (local/remote), any skipped or failed deletions.

Checkout default branch when done if cleanup switched branches.

### Cleanup report

Keep it short: what was deleted, what was kept, any failures.

## Subagent contract

- Parent holds full plan, chat history, and user approvals
- Subagents receive only the current slice payload
- One subagent per slice for git push; a short second call for `gh pr create`; one cleanup subagent at the end if approved
- Subagent returns structured summary; parent does not re-read full diffs
- Use `subagent_type: shell` for all git/gh commands

## Edge cases

| Case | Behavior |
|------|----------|
| Cherry-pick conflict | Subagent stops; parent reports conflict files; show conflict action menu; retry on **F**, revise on **R** |
| Uncommitted work | Backup ref created; only listed commits cherry-picked; uncommitted work stays on source branch |
| Commit spans slices | Flag in plan; `git cherry-pick -n` + selective commit only if user approves |
| Default branch not `main` | Detect via `origin/HEAD` or repo convention |
| Branch name collision | Check `git branch -a` before push; append scope suffix or `-2` if taken |
| Branch already deleted remotely | Skip remote delete; report as already gone |
| Local branch not fully merged | Stop; ask user before `-D` |

## Examples

See [examples.md](examples.md) for a full plan and PR metadata sample.
