# Split-to-PRs Sequential — Examples

## Example plan (commit mapping + branch names)

Source: `wip3` (15 commits, clean tree vs `main`)

```markdown
# Split plan: wip3

## Summary
15 commits → 7 PRs, sequential off main (merge each before starting the next)

## Slices

### PR 1 — Legacy seed foundation + Tags
| Commit | Subject |
|--------|---------|
| `ef2fbc6` | feat(seed): add shared field factories and runner |
| `c2349d6` | feat(seed): add tags collection and migration |
| `e37521a` | docs(seed): document legacy id mapping rule |

**Base:** `main` (updated before branch)
**Branch name:** `feat/seed-foundation-tags`

### PR 2 — Photos collection + legacy seed
| Commit | Subject |
|--------|---------|
| `4250d3d` | feat(photos): add Photos collection |
| `d39aaa1` | feat(photos): add seed:photos bin and download helpers |

**Branch name:** `feat/photos-legacy-seed`

### PR 3 — Facilities migration
| Commit | Subject |
|--------|---------|
| `7331129` | feat(facilities): migrate from legacy CMS |

**Branch name:** `feat/facilities-legacy-migration`

### PR 4 — Albums collection + legacy seed
| Commit | Subject |
|--------|---------|
| `6929caa` | feat(albums): add collection with legacy seed import |

**Branch name:** `feat/albums-legacy-seed`

### PR 5 — Videos, Audios, Documents
| Commit | Subject |
|--------|---------|
| `4c685bd` | feat(videos): add collection with YouTube admin |
| `b0c3bfb` | feat(audios): add collection with playback admin |
| `1497568` | feat(documents): add documents and categories |

**Branch name:** `feat/secondary-media-collections`

### PR 6 — Admin UX: slugs, join fields, album polish
| Commit | Subject |
|--------|---------|
| `205eb9e` | feat(admin): add auto slug field |
| `894f0ba` | feat(admin): add join fields on tags and categories |
| `42c4d31` | fix(albums): correct seed slug generation |
| `72e8935` | feat(albums): add cover thumbnail list cell |

**Branch name:** `feat/admin-slug-join-fields`

### PR 7 — Rich-text media blocks + shortcode seeding
| Commit | Subject |
|--------|---------|
| `6c56f8d` | feat(rich-text): add media blocks and legacy shortcode seeding |

**Branch name:** `feat/rich-text-media-blocks`

## Execution order
1. PR 1 → merge → PR 2 → merge → … → PR 7

**Actions**
- **A** — Approve plan; start backup and first slice
- **R** — Revise plan (describe changes to slices, order, or branch names)
- **X** — Cancel split (no further git/gh work)
```

## Example PR metadata draft (PR 1)

After subagent cherry-picks and pushes `feat/seed-foundation-tags`:

```markdown
## Proposed PR

**Title:** feat(seed): add legacy seed foundation and tags collection

**Body:**
## Summary
- Add shared field factories and seed runner used by all legacy imports
- Introduce Tags collection with first Payload migration
- Document legacy ID mapping in Cursor rules and README

## Test plan
- [ ] Run seed against empty database; verify tags import
- [ ] Confirm migration applies cleanly on fresh checkout
- [ ] Check `.env.example` documents required seed variables

**Actions**
- **A** — Approve title and body; create PR
- **E** — Edit metadata (provide new title, body, or edits)
- **X** — Cancel split
```

User replies **A** → subagent runs `gh pr create` → returns URL → agent shows merge checkpoint:

```markdown
PR: https://github.com/org/repo/pull/1234

**Actions**
- **M** — Merged; proceed to next slice
- **V** — Verify merge status on GitHub
- **W** — Not merged yet; wait
- **X** — Cancel split
```

User replies **M** before starting PR 2.

## Example cleanup proposal (after all PRs merged)

```markdown
## Cleanup proposal

**Slice branches (local + remote):**
- `feat/seed-foundation-tags`
- `feat/photos-legacy-seed`
- `feat/facilities-legacy-migration`
- `feat/albums-legacy-seed`
- `feat/secondary-media-collections`
- `feat/admin-slug-join-fields`
- `feat/rich-text-media-blocks`

**Also delete source branch?** `wip3` — no (default)

**Keep:** backup ref `refs/backup/pre-split-1717934400`

**Actions**
- **Y** — Yes — delete all listed slice branches (local + remote)
- **S** — Slice branches only — do not delete source branch
- **R** — Revise list (say which branches to add/remove)
- **N** — No cleanup
- **D** — Also delete source branch `wip3`
```

User replies **S** → subagent deletes local and remote slice branches → reports what was removed.

## Branch naming: bad vs good

| Slice | Bad | Good |
|-------|-----|------|
| Foundation + tags | `split/pr1-foundation-tags` | `feat/seed-foundation-tags` |
| Photos collection | `split/2-photos` | `feat/photos-legacy-seed` |
| Admin slug UX | `split/pr6-admin` | `feat/admin-slug-join-fields` |
