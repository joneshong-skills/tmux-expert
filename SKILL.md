---
name: tmux-expert
description: >-
  This skill should be used when the user asks to "control tmux", "build tmux web UI",
  "tmux send keys", "read tmux pane", "tmux automation", "tmux 控制", "tmux 自動化",
  "tmux 指令", "remote terminal control", "bridge tmux to web",
  mentions tmux scripting, programmatic tmux control, tmux web integration,
  or discusses reading/writing tmux panes, managing sessions/windows/panes,
  or building interfaces that interact with tmux.
version: 0.1.0
---

# tmux Expert — Programmatic Control & Web Bridging

Specialist knowledge for controlling tmux programmatically: querying state, reading pane
content, injecting input, managing sessions/windows/panes, and bridging tmux to web UIs.

## Agent Delegation

Delegate tmux automation scripts to `worker` agent.

## tmux Object Hierarchy

```
Server
 └─ Session (name: "default", "dev")
     └─ Window (index: 1, 2, 5 — NOT necessarily sequential)
         └─ Pane (index: 0, 1, 2)
```

**Targeting syntax**: `session:window.pane` — e.g., `default:1.0`, `dev:5.2`

## Core Commands — Format String Patterns

### Query State

Use `-F` format strings with `#{variable}`. Use `\t` as delimiter for
reliable parsing (avoids spaces in names breaking split logic).

```bash
# List sessions
tmux list-sessions -F '#{session_name}\t#{session_windows}\t#{session_attached}'

# List windows in a session
tmux list-windows -t SESSION -F '#{window_index}\t#{window_name}\t#{window_active}\t#{window_panes}'

# List ALL panes across ALL windows in a session (-s flag)
tmux list-panes -s -t SESSION -F '#{window_index}\t#{window_name}\t#{pane_index}\t#{pane_active}\t#{pane_width}\t#{pane_height}\t#{pane_current_command}\t#{pane_title}'
```

**Critical**: `-s` flag on `list-panes` lists panes across all windows. Without it, only
the current window's panes are listed.

**Window indices are NOT sequential** — after creating/deleting windows, gaps appear
(e.g., 1, 5, 8). Never assume contiguous numbering.

### Read Pane Content

```bash
# Capture visible content + scrollback (last N lines, with escape sequences)
tmux capture-pane -t TARGET -p -e -S -150
```

Flags: `-p` print to stdout, `-e` include ANSI escapes, `-S -N` start N lines before visible.

**ANSI handling with `-e`**: Preserves SGR sequences (`\033[38;5;82m`, etc.).
For web display, either strip or parse. See `references/ansi-patterns.md` for details.

### Send Input

```bash
# Send literal text (-l flag: special chars escaped)
tmux send-keys -t TARGET -l "hello world"

# Send special keys (NO -l flag: key names interpreted)
tmux send-keys -t TARGET Enter
tmux send-keys -t TARGET C-c
```

**Critical**: `-l` (literal) vs no `-l`. For command + Enter, use two separate calls:
```bash
tmux send-keys -t TARGET -l "ls -la"
tmux send-keys -t TARGET Enter
```

### Manage Windows & Panes

```bash
tmux new-window -t SESSION           # create (auto-switches in attached session)
tmux kill-window -t SESSION:INDEX    # destroy
tmux split-window -t TARGET -h       # horizontal split
tmux split-window -t TARGET -v       # vertical split
tmux resize-pane -t TARGET -L 10     # grow left 10 cells (-R/-U/-D)
```

## Web UI Bridging Architecture

### Stack & Protocol

```
Browser (JS) ←WebSocket→ aiohttp/Python ←subprocess→ tmux CLI
```

| Msg Type | Direction | Payload |
|----------|-----------|---------|
| `panes` | S→C | `{panes: [{id, window, pane, command}]}` |
| `output` | S→C | `{panes: {"1.0": "content"}}` |
| `windows` | S→C | `{windows: [{index, name, panes}], active: int}` |
| `metrics` | S→C | `{metrics: {net, cpu, mem}}` |
| `input` | C→S | `{pane: "1.0", text: "cmd"}` |
| `key` | C→S | `{pane: "1.0", key: "C-c"}` |
| `switch_window` | C→S | `{window: 5}` |
| `new_window` | C→S | `{}` |
| `close_window` | C→S | `{window: 2}` |

### Poll Loop Design

- Pane content: diff-based, every 400ms (only send changed content)
- Metrics: every ~5s (lower frequency scripts like cpu/mem/disk)
- Filter by active window: only capture visible panes
- On window switch/create/close: `last_contents.clear()` to force full refresh

### CSS Grid Layout for Panes

Presets work better than auto-calculation for predictable layouts:

| Panes | Preset | Grid |
|-------|--------|------|
| 1 | Single | `1fr / 1fr` |
| 2 | 2-col | `1fr 1fr / 1fr` |
| 4 | 2x2 | `1fr 1fr / 1fr 1fr` |
| 4 | 4-col | `repeat(4, 1fr) / 1fr` |

Hide excess cells with `display:none` rather than breaking layout.

## Pitfalls & Hard-Won Lessons

### 1. Missing imports in LaunchAgent
LaunchAgent = minimal environment. `os.path.expanduser()` fails silently if `os` not imported.
**Always verify all imports when running as a daemon.**

### 2. ANSI strip regex eating content
Naive `/\x1b\[[0-9;]*m/` misses CSI terminators beyond `m`.
Use `/\x1b\[[0-9;]*[A-Za-z]/g` or a full SGR parser. See `references/ansi-patterns.md`.

### 3. CSS child selectors vs flat classes
`.parent .child` (descendant selector) ≠ `.parent.child` (both on same element).
Always match CSS structure to DOM structure.

### 4. Window index after create/delete
Never assume `max + 1`. Re-query `list-windows` after `new-window` to get actual index.

### 5. `new-window` vs WebSocket active tracking
`tmux new-window` switches the *attached terminal's* window, not the WebSocket client's.
Maintain a separate `active_window` variable in the WebSocket handler. After create:
set `active_window = new_index`, clear content cache, send fresh state.

### 6. Stale content after window switch
`last_contents` dict retains previous window's panes → diff fails to detect "new" content.
**Always `clear()` on switch/create/close.**

### 7. LaunchAgent PATH
Set PATH explicitly in plist `<EnvironmentVariables>`. Include `/opt/homebrew/bin` for
Homebrew tools. Without this, `tmux` and `uv` commands fail.

## Reference Implementation

`~/Claude/apps/tmux-webui/server.py` — single-file aiohttp server (HTML/CSS/JS inline):
- Session/window/pane management via WebSocket
- CSS Grid layout engine with presets + drag resize
- Full ANSI SGR color parser (256-color, RGB, backgrounds)
- Slash command autocomplete, skill palette with search
- tmux status bar metrics, window tabs with create/close

## Additional Resources

### Reference Files
- **`references/ansi-patterns.md`** — ANSI SGR parsing patterns and regex recipes

## Continuous Improvement

After every invocation:
1. **Reflect** — What worked, what caused friction
2. **Record** — Append lesson to `lessons.md`
3. **Refine** — When a pattern recurs 2+ times, update SKILL.md

### lessons.md Entry Format
```
### YYYY-MM-DD — Brief title
- **Friction**: What went wrong
- **Fix**: How resolved
- **Rule**: Generalizable takeaway
```
