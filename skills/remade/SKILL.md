---
description: Interact with RemADE terminals and notes — list sessions, read terminal output, search buffer content, send commands/keystrokes, create terminals, and manage per-project quick notes (create, read, search, update, delete). Use when the user asks you to run commands in the IDE, check terminal output, automate terminal workflows, manage IDE terminals, read or write project notes, search notes, or check reminders.
---

# RemADE API

Interact with terminals and notes in RemADE.

## remade CLI (Preferred)

The `remade` CLI is a thin passthrough to the ADE REST API that handles port resolution automatically. Installed at `~/.remade/bin/remade`. **Always prefer the CLI over raw curl calls.**

> **RULE: You MUST single-quote every URL path argument passed to `$remade`.**
> zsh treats `?` and `&` as glob characters. Without quotes, the command fails
> with `no matches found`. Wrong: `$remade /api/foo?x=1` — Right: `$remade '/api/foo?x=1'`

```bash
remade=/Users/$USER/.remade/bin/remade

# GET is the default method — URL path is ALWAYS single-quoted
$remade '/api/project/open'
$remade '/api/notes?root=/path/to/project'
$remade '/api/terminals/list?projectPath=/path/to/project'
$remade '/api/terminals/buffer?terminalId=ID&projectPath=/path&tail=50'

# Specify method for mutations
$remade POST '/api/notes' '{"root":"/path","note":{"mode":"todo","content":"Fix bug #urgent"}}'
$remade POST '/api/terminals/input' '{"terminalId":"ID","projectPath":"/path","input":"npm test\n"}'
$remade PUT '/api/notes/NOTE_ID' '{"root":"/path","note":{"completed":true}}'
$remade DELETE '/api/notes/NOTE_ID?root=/path'
```

If a JSON body is provided without an explicit method, POST is assumed.

## curl Fallback

If the CLI is unavailable, resolve port manually:

```bash
RM_PORT=$(cat ~/.remade/server-port 2>/dev/null || echo 4711) && RM=http://localhost:$RM_PORT && curl -s "$RM/api/notes?root=/path/to/project"
```

## Project Path

Most endpoints require a project path (`projectPath` or `root`). The CLI infers this from cwd. For curl, discover open projects:

```bash
curl -s "$RM/api/project/open"
```

Returns: `[{ "path": "/Users/.../project", "name": "project", "openedAt": "..." }]`

---

## Terminals API

### List Terminals

```bash
curl -s "$RM/api/terminals/list?projectPath=PATH"
```

Returns: `{ "sessions": [{ "zmxName": "cosmic-monkey", "cwd": "...", "connected": true, ... }] }`

The `zmxName` field is the terminal ID used in all other terminal endpoints.

### Read Terminal Buffer

```bash
# Default mode — regex-based ANSI stripping (good for simple shells)
curl -s "$RM/api/terminals/buffer?terminalId=ID&projectPath=PATH&tail=50"

# Rendered mode — headless terminal emulator, use for TUI apps (Claude Code, vim, etc.)
curl -s "$RM/api/terminals/buffer?terminalId=ID&projectPath=PATH&render=true&mode=screen"
```

Returns: `{ "content": "...", "totalLines": N, "truncated": bool }` — content is a single newline-delimited string.

Params: `terminalId` (required), `projectPath` (required), `tail` (lines, default 200), `render` (true for TUIs), `mode` (screen = visible viewport only), `cols`/`rows` (render dimensions).

### Search Terminal Buffer

```bash
curl -s "$RM/api/terminals/search?terminalId=ID&projectPath=PATH&query=error&context=3"
```

Params: `query` (required), `regex`, `caseSensitive`, `context` (lines around match), `maxResults`.

### Send Input

```bash
curl -s -X POST "$RM/api/terminals/input" \
  -H 'Content-Type: application/json' \
  -d '{"terminalId": "ID", "projectPath": "PATH", "input": "npm test\n"}'
```

Keystroke placeholders: `\n` (Enter), `{{ctrl+c}}` (interrupt), `{{ctrl+d}}` (EOF), `{{ctrl+z}}` (suspend), `{{ctrl+l}}` (clear), `{{ctrl+a}}`/`{{ctrl+e}}` (line start/end), `{{ctrl+u}}` (kill line), `{{ctrl+w}}` (kill word), `{{ctrl+r}}` (reverse search), `{{ctrl+k}}` (kill to end), `{{tab}}`, `{{escape}}`, `{{up}}`/`{{down}}`/`{{left}}`/`{{right}}`, `{{home}}`/`{{end}}`, `{{backspace}}`/`{{delete}}`.

### Create Terminal

```bash
curl -s -X POST "$RM/api/terminals/create" \
  -H 'Content-Type: application/json' \
  -d '{"projectPath": "PATH"}'
```

Optional: `cwd` (working directory), `title` (display name). Returns: `{ "terminalId": "cosmic-monkey" }`.

---

## Notes API

Per-project quick notes stored in SQLite. All endpoints use `root` for the project path.

### List Notes

```bash
curl -s "$RM/api/notes?root=PATH"

# Pagination: limit (default 100) and offset (default 0)
curl -s "$RM/api/notes?root=PATH&limit=5&offset=0"
```

Returns: `{ "notes": [...], "total": 72, "limit": 5, "offset": 0 }`

Query params: `root` (required), `q` (search), `mode` (todo/note/reminder), `tag`, `completed` (true/false), `limit`, `offset`.

### Create Note

```bash
curl -s -X POST "$RM/api/notes" \
  -H 'Content-Type: application/json' \
  -d '{"root": "PATH", "note": {"mode": "todo", "content": "Fix the bug #urgent"}}'
```

Modes: `note` (freeform), `todo` (actionable), `reminder` (add `reminderTime` as ISO 8601).

### Update Note

```bash
curl -s -X PUT "$RM/api/notes/NOTE_ID" \
  -H 'Content-Type: application/json' \
  -d '{"root": "PATH", "note": {"completed": true}}'
```

Updatable fields: `content`, `mode`, `completed`, `pinned`, `reminderTime`, `reminderFired`.

### Delete Note

```bash
curl -s -X DELETE "$RM/api/notes/NOTE_ID?root=PATH"
```

### Other Note Endpoints

- **Hashtags**: `GET $RM/api/notes/hashtags?root=PATH` → `{ "tags": [{ "tag": "urgent", "count": 3 }] }`
- **Overdue reminders**: `GET $RM/api/notes/reminders?root=PATH`

---

## Prompts API

Show an interactive HTML form overlay on a terminal. When the user submits, their input is written to the terminal's stdin as a single line of JSON.

### Create Prompt

When called from within an IDE terminal (via `$remade`), the terminal is auto-detected from `$ZMX_SESSION` — no need to pass `terminalId` or `projectPath`:

```bash
$remade POST '/api/prompts' '{
  "prompt": {
    "title": "Sheet title",
    "html": "<label>Name <input type=\"text\" name=\"myField\"></label>",
    "submitLabel": "Go"
  }
}'
```

Returns: `{ "promptId": "uuid" }`

You can also target a specific terminal explicitly with `terminalId` and `projectPath` in the body.

The sheet appears as an overlay on the terminal with an X (dismiss) button and a Submit button. The HTML is rendered in a sandboxed iframe with base dark-theme styles pre-applied.

### How form data is collected

On submit, all elements with a `name` attribute are collected into a flat JSON object and written to the terminal stdin followed by a newline:

- **Text inputs / textareas / selects** → `{ "name": "value" }`
- **Checkboxes sharing a name** → `{ "name": ["checked_value_1", "checked_value_2"] }`
- **Single checkbox** → `{ "name": true }` or `{ "name": false }`
- **Radio group** → `{ "name": "selected_value" }`

If the user dismisses (X or Escape), nothing is written to stdin.

### submitContext — include context for the receiving CLI

The `submitContext` object is merged into the output JSON alongside form data. **Always include it** so the receiving CLI has enough context to understand the selections without seeing the original form.

```json
"submitContext": {
  "_action": "fix-bugs",
  "_description": "User selected which bugs to fix from triage list",
  "_items": {
    "040e1bc2": "SQLite plugin broken in installed app",
    "8ab3792d": "Run fails in renamed npm projects"
  }
}
```

Keys starting with `_` are conventional for context fields (won't collide with form field names). The receiving CLI gets a single JSON line like:
```json
{"_action":"fix-bugs","_items":{...},"bugs":["040e1bc2"],"notes":"Fix ASAP"}
```

### HTML requirements

- Use standard form elements (`input`, `textarea`, `select`) with `name` attributes
- Base styles provide: dark theme, styled checkboxes/inputs/textareas/selects, `.group`, `.row`, `.description` utility classes
- `h3` renders as an uppercase section header — good for grouping
- `hr` renders as a subtle divider
- `label` elements are styled as interactive rows (hover highlight)

### Dismiss Prompt

```bash
$remade POST '/api/prompts/PROMPT_ID/dismiss'
```

### List Active Prompts

```bash
$remade '/api/prompts'
$remade '/api/prompts?terminalId=TERMINAL_ID'
```

### Example: Bug triage checklist

```bash
$remade POST '/api/prompts' '{
  "prompt": {
    "title": "Select bugs to fix",
    "submitContext": {
      "_action": "fix-bugs",
      "_items": {
        "040e1bc2": "SQLite plugin broken in installed app",
        "8ab3792d": "Run fails in renamed npm projects",
        "4ae82ec5": "Editor says File changed on disk immediately after editing"
      }
    },
    "html": "<div class=\"group\"><h3>Must-fix bugs</h3><label><input type=\"checkbox\" name=\"bugs\" value=\"040e1bc2\"> SQLite plugin broken in installed app</label><label><input type=\"checkbox\" name=\"bugs\" value=\"8ab3792d\"> Run fails in renamed npm projects</label><label><input type=\"checkbox\" name=\"bugs\" value=\"4ae82ec5\"> Editor says File changed on disk immediately after editing</label></div><div class=\"group\"><h3>Options</h3><label>Priority <select name=\"priority\"><option value=\"high\">High</option><option value=\"medium\">Medium</option><option value=\"low\">Low</option></select></label><label>Notes <textarea name=\"notes\" placeholder=\"Any additional context...\"></textarea></label></div>",
    "submitLabel": "Fix Selected"
  }
}'
```

Result written to stdin: `{"_action":"fix-bugs","_items":{"040e1bc2":"SQLite plugin broken...","8ab3792d":"Run fails...","4ae82ec5":"Editor says..."},"bugs":["040e1bc2","4ae82ec5"],"priority":"high","notes":"Fix these first"}`

### Example: Simple confirmation

```bash
$remade POST '/api/prompts' '{
  "prompt": {
    "title": "Commit these changes?",
    "submitContext": {"_action": "confirm-commit", "_files": ["src/app.ts", "src/utils.ts"]},
    "html": "<label>Commit message <input type=\"text\" name=\"message\" placeholder=\"Describe changes...\"></label><label><input type=\"checkbox\" name=\"push\" checked> Push after commit</label>"
  }
}'
```

Result: `{"_action":"confirm-commit","_files":["src/app.ts","src/utils.ts"],"message":"Fix auth bug","push":true}`

---

## Important Notes

- **Combine setup with first call**: always chain `RM_PORT=... && CM=... && curl ...` in one bash command
- **Project path**: use bare filesystem paths (e.g. `/Users/ones/Projects/myapp`); `file://` URLs also work
- **Terminal IDs**: always use the `zmxName` from `/api/terminals/list`
- **Buffer is point-in-time**: for long-running commands, read again after a delay
- **Hashtags**: any `#word` in note content is auto-extracted for filtering
