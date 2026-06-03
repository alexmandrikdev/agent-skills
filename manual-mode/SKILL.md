---
name: manual-mode
description: Operates in manual mode without tools when they are unavailable or unreliable. Gives the user ordered step-by-step instructions instead of calling Read, Write, Grep, Shell, MCP, browser, or Task. Use when tools fail, the environment blocks tool use, the user asks for manual/advisor mode, or the user says tools are working again to exit manual mode.
---

# Manual mode (no tools)

Tools are unavailable or unreliable in this environment. **Do not call any tools** (Read, Write, Grep, Shell, MCP, browser, Task, etc.).

Act as an advisor; the user executes everything. For each task:

1. **State the goal** in one sentence.
2. **Give ordered steps** the user can follow (create/edit files, run commands, paste output back).
3. **Provide exact content** when creating or changing files (full file or clear search/replace blocks).
4. **Ask for artifacts** when you need context: file paths, pasted file contents, command output, errors, screenshots.
5. **Wait for results** before the next step when output matters (build errors, test failures, diffs).

## What to ask the user to do

| Need | Ask the user to |
|------|-----------------|
| Read code | Run a CLI command to print the file (see **Read files via CLI**); do not ask the user to open the editor and copy manually |
| Search codebase | Run `rg 'pattern' path/` (or IDE search) and paste matches |
| Edit files | Apply your diff/snippet in the editor, or use your exact `old` → `new` instructions |
| Run commands | Run in terminal (with clipboard pipeline — see below), paste **full** stdout/stderr and exit code |
| Verify | Run tests/build/lint and paste output |
| Git | Run `git status`, `git diff`, etc. and paste output |
| Dependencies | Run install commands and paste result |

## Read files via CLI

Prefer terminal commands over “open this file and paste”:

| Need | Command |
|------|---------|
| Full file | `echo '=== FILE: path/to/file.ts ===' && cat path/to/file.ts` |
| Line range | `echo '=== FILE: path/to/file.ts (lines 10-50) ===' && sed -n '10,50p' path/to/file.ts` |
| Head/tail | `echo '=== FILE: path/to/file.ts (first 80 lines) ===' && head -n 80 path/to/file.ts` |

Always prefix output with a context line (`=== FILE: … ===`) so pasted chat output is identifiable.

## CLI commands (paste-back)

When a command’s output should be pasted into chat, append a clipboard pipeline so the user can paste with Ctrl+V.

**Default (Ubuntu, X11):** pipe through `tee` (keeps terminal output visible) then `xclip`:

```bash
<command> 2>&1 | tee /dev/tty | xclip -selection clipboard
```

### Bundling multiple commands

When **two or more** CLI commands are needed for paste-back (reads, git, search, tests, etc.), give **one** bundled one-liner — not separate commands. Each segment must echo context before its output:

- Files: `echo '=== FILE: path ==='`
- Commands: `echo '=== CMD: <command> ==='`
- Separate segments with `echo` (blank line) for readability

Wrap in `{ …; } 2>&1` so the whole block shares one clipboard pipeline:

```bash
{ echo '=== CMD: git status ==='; git status; echo; echo '=== CMD: git diff ==='; git diff; echo; echo '=== FILE: src/foo.ts ==='; cat src/foo.ts; } 2>&1 | tee /dev/tty | xclip -selection clipboard
```

Rules:

- Always include `2>&1` so stderr is captured.
- Append the pipeline to **every** CLI command (or bundled one-liner) whose output the user must paste back.
- After the command block, remind: paste into chat (Ctrl+V).
- One-off install if missing: `sudo apt install xclip` (X11).
- Do **not** add the pipeline when output is only for the user’s local use (e.g. `cd`, `mkdir`, opening an editor).
- Prefer one bundled paste-back command per step over multiple sequential paste-back commands.

**Single-command example:**

```bash
git status 2>&1 | tee /dev/tty | xclip -selection clipboard
```

**Multi-command example:**

```bash
{ echo '=== CMD: rg "pattern" src/ ==='; rg 'pattern' src/; echo; echo '=== FILE: src/main.ts (lines 1-120) ==='; sed -n '1,120p' src/main.ts; } 2>&1 | tee /dev/tty | xclip -selection clipboard
```

## Response format

- Prefer **numbered steps** over long prose.
- One step = one action: one file edit, **or** one CLI block (single or bundled paste-back command).
- Label blocks: `### File: path/to/file.ts`, `### Command:`, `### Paste back:`.
- Paste-back commands must use the clipboard pipeline above; bundled multi-command steps use one `### Command:` block.
- If unsure about project state, say what to paste before continuing.

## Do not

- Say you will “run”, “read”, or “search” with tools — you cannot.
- Assume file contents or command output you have not been given.
- Skip verification steps when the user reported errors.

## When tools might work again

Only use tools if the user explicitly says tools are working again; until then, stay in manual mode.