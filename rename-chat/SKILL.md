---
name: rename-chat
description: >-
  Renames the active Cursor chat from prior user messages. Use when the user
  invokes /rename-chat, asks to rename or title the current chat, or wants a
  conversation name based on earlier messages.
disable-model-invocation: true
---

# Rename Chat

Rename the **active** chat tab from prior **user** messages. Apply the new title via `cursor-app-control.rename_chat` only — no git, no fork APIs, no other side effects.

## Hard rules

- **Active chat only** — the user must be in the tab being renamed.
- **User messages only** — do not title from assistant or tool output unless the user explicitly asks.
- **Exclude the current message** — the invoking message is not in the candidate pool.
- **Wait for selection** when no message reference was given — never auto-pick.
- **Do not pass `composerId`** — unsupported by the rename API.

## Workflow

### 1. Parse optional message reference

If the user provided a reference (e.g. `/rename-chat 2`, `/rename-chat 1,3`, `/rename-chat latest`, or a short text hint), resolve it to one or more prior user messages and skip to step 3.

| Form | Maps to |
|------|---------|
| `latest`, `last`, `1` | Most recent prior user message |
| `2`, `3`, … | Nth most recent prior user message (1 = newest) |
| `2,3` or `2 and 3` | Multiple indices |
| Free-text hint | Best-matching prior user message(s) by content |

Strip wrappers (`<user_query>`, `<timestamp>`, system tags) when displaying or summarizing message text.

**Message source:** Use conversation context the agent already has. Do not read agent transcript JSONL unless context alone is insufficient (rare).

### 2. Interactive selection (no reference)

When no reference is provided:

1. Collect up to **5** prior user messages, **newest first** (descending).
2. Show a numbered list:

```markdown
## Recent user messages (newest first)
1. [truncated preview ~120 chars]
2. ...
```

3. Call **AskQuestion** with `allow_multiple: true` — one option per listed message (same index + truncated preview as label).
4. **Stop and wait** for the answer before renaming.

If fewer than 5 prior user messages exist, list only what is available. If **zero** exist, say so and ask the user to provide a title or a message to base it on.

### 3. Derive title

From the selected message(s), produce a concise chat title:

- **Single message:** noun phrase or `topic: detail` from the main question or task.
- **Multiple messages:** one title covering the shared theme; if topics differ, use `primaryTopic + secondaryTopic` or `topic: detail` with the **most recent** selection as primary.
- Strip noise (timestamps, XML, markdown artifacts).
- Max 200 characters; prefer ≤80.
- No trailing period; avoid generics (`New chat`, `Conversation`).

### 4. Apply rename

1. Read the MCP schema at `mcps/cursor-app-control/tools/rename_chat.json` (required before call).
2. Call `CallMcpTool`:

```json
{
  "server": "cursor-app-control",
  "toolName": "rename_chat",
  "arguments": { "title": "<derived title>" }
}
```

3. Confirm the new title to the user in one sentence.

## Examples

**`/rename-chat` (no reference)**

List the 5 most recent prior user messages, run AskQuestion with multi-select, derive title from the pick, call `rename_chat`.

**`/rename-chat latest`**

Use the most recent prior user message only.

| Selected user message(s) | Title |
|--------------------------|-------|
| "Create a skill that renames the current chat…" | `Rename chat skill` |
| "Fix auth redirect loop on login" | `Auth: fix login redirect loop` |
| Messages about `.env` and sync scripts | `Sync scripts: require .env` |
| "How does busyness calc use occupancy vs daily peak?" | `Busyness calc: occupancy vs daily peak` |

**`/rename-chat 2,3`**

Use the 2nd and 3rd most recent prior user messages; synthesize one title (most recent selection is primary if topics differ).

**Zero prior user messages**

Tell the user there is no history to draw from; ask for a title or a message to base it on. Do not call `rename_chat` until you have input.
