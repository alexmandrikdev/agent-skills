---
name: split-to-prs-sequential
description: >-
  Split a branch or commit range into sequential reviewable PRs using
  cherry-picks off updated main, with commit-to-branch mapping, descriptive
  branch names, per-PR title/description approval, merge checkpoints between
  slices, and optional branch cleanup. Use when the user asks to split a branch
  into PRs, split commits into PRs, cherry-pick commits into PRs, or invokes
  split-to-prs-sequential.
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

Parent agent orchestrates; each slice's git/gh work runs in a **shell subagent** with minimal context.

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

**Stop and ask for plan approval.** Do not mutate git state until approved.

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

**Stop for user approval** of title and body.

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

After returning the PR URL, **stop and wait** for explicit user confirmation (e.g. "merged", "continue", "next"). Do **not** poll GitHub. Do **not** start the next slice until confirmed.

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

**Stop for cleanup approval.** User can approve all, approve slice branches only, revise the list, or decline.

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
| Cherry-pick conflict | Subagent stops; parent reports conflict files; user resolves or revises plan |
| Uncommitted work | Backup ref created; only listed commits cherry-picked; uncommitted work stays on source branch |
| Commit spans slices | Flag in plan; `git cherry-pick -n` + selective commit only if user approves |
| Default branch not `main` | Detect via `origin/HEAD` or repo convention |
| Branch name collision | Check `git branch -a` before push; append scope suffix or `-2` if taken |
| Branch already deleted remotely | Skip remote delete; report as already gone |
| Local branch not fully merged | Stop; ask user before `-D` |

## Examples

See [examples.md](examples.md) for a full plan and PR metadata sample.
