# tmux-expert — Lessons Learned

### 2026-02-14 — Missing `import os` in LaunchAgent daemon
- **Friction**: tmux status metrics all returned empty. No visible error in web UI.
  `os.path.expanduser()` in `tmux_status_metrics()` raised `NameError` but was silently caught.
- **Fix**: Added `import os` to server.py imports. Found via debug logging in error log.
- **Rule**: LaunchAgent daemons have minimal environment. Always double-check ALL stdlib imports.
  Add temporary `logging` to diagnose silent failures, then remove after fix.

### 2026-02-14 — ANSI strip regex eating SGR sequences
- **Friction**: Terminal output lost content characters. Naive regex `/\x1b\[[0-9;]*m/g`
  didn't handle concatenated SGR sequences or non-`m` CSI terminators (cursor moves, etc.).
- **Fix**: Built a full SGR state-machine parser handling 256-color, RGB, bold, italic, etc.
- **Rule**: For web display of terminal content, either use a comprehensive regex
  `/\x1b\[[0-9;]*[A-Za-z]/g` or build a proper parser. Never use the naive `*m` pattern.

### 2026-02-14 — CSS descendant selector vs same-element classes
- **Friction**: Metrics badges appeared unstyled. CSS had `.ms-net .ms-label` (child)
  but JS put both classes on the same span element.
- **Fix**: Wrapped each metric in a container span: `<span class="ms-net"><span class="ms-label">`.
- **Rule**: Always verify CSS selector structure matches actual DOM nesting.

### 2026-02-14 — Window index non-sequential after delete
- **Friction**: After creating/deleting windows, assumed indices would be contiguous (1,2,3).
  Actually got gaps (1,5) because tmux preserves original indices.
- **Fix**: Always query `list-windows` for actual indices. Use `max(w["index"])` to find
  newly created window, not `len(windows)`.
- **Rule**: Never assume tmux window indices are sequential. Always query actual state.

### 2026-02-14 — `new-window` doesn't switch WebSocket view
- **Friction**: `tmux new-window` switches the attached terminal's active window, but the
  WebSocket handler maintains its own `active_window` variable. View stayed on old window.
- **Fix**: After `new-window`, explicitly set `active_window = new_index`,
  `last_contents.clear()`, and send fresh `windows` + `panes` messages.
- **Rule**: WebSocket clients track their own view state independently of tmux's attached
  terminal. Always update both tmux AND client state.

### 2026-02-14 — Stale diff cache after window switch
- **Friction**: After switching windows, new pane content didn't appear because `last_contents`
  still had entries from previous window → diff showed "no change".
- **Fix**: `last_contents.clear()` on every switch_window, new_window, close_window.
- **Rule**: Any operation that changes visible panes must clear the content diff cache.

### 2026-02-14 — LaunchAgent missing PATH for Homebrew
- **Friction**: Server couldn't find `tmux` binary when launched by launchd.
- **Fix**: Added explicit PATH in plist `<EnvironmentVariables>` including `/opt/homebrew/bin`.
- **Rule**: Always set PATH in LaunchAgent plists. launchd inherits almost nothing.

### 2026-02-14 — Interactive CLI context test via send-keys
- **Friction**: First `send-keys -l "text" + Enter` pair didn't trigger response in either
  Gemini or Codex CLI. Both showed text at prompt but didn't process it.
- **Fix**: Sent an additional `Enter` key. After that, both CLIs responded correctly.
- **Rule**: TUI apps (Gemini CLI, Codex CLI) may need an extra `Enter` after `send-keys -l`
  + `send-keys Enter`. Always verify via `capture-pane` and retry Enter if the response
  doesn't appear within ~10 seconds.
- **Insight**: Interactive mode (via tmux send-keys) maintains full conversation context
  across turns — fundamentally different from headless (`-p`) which is stateless.
  Gemini also auto-persists facts via `SaveMemory` to `GEMINI.md`; Codex uses only
  in-session context. Both successfully recalled facts from Round 1 in Round 2.

### 2026-02-14 — Initial metrics not sent on WebSocket connect
- **Friction**: Status bar was empty for first 5 seconds until poll cycle sent metrics.
- **Fix**: Added immediate `tmux_status_metrics()` call right after sending initial
  `windows` and `panes` messages on WebSocket connect.
- **Rule**: Send initial state eagerly on connect, don't wait for the first poll cycle.

### 2026-07-20 — Sidecar resume must preserve per-CLI argv policy
- **Friction**: A tmux pane showed `codex --yolo`, while the restore sidecar recorded only `codex resume <sid>`.
- **Fix**: Trace the pane shell pid and child agent pid, then compare live argv, lsof-derived sid, and the generated TSV command; the loss happened before tmux `send-keys`.
- **Rule**: For agent recovery, inspect shell pid and child agent pid separately, then compare live argv, snapshot row, and restore command.

### 2026-07-20 — WebUI metrics needs source field plus compact consumer
- **Friction**: tmux-webui received the verbose Claude extra-usage string but dropped the numeric current balance because its generic LLM parser accepted strings only.
- **Fix**: Preserve `llm_cc_ex_balance_usd` as a narrow numeric exception in the HTTP provider; let the existing pill renderer display only `EX $59.21` and hide the helper field.
- **Rule**: For statusline bridges, trace source JSON → typed snapshot → frontend segment; missing numeric identity must hide the active pill instead of falling back to a derived display string.

### 2026-07-24 — Validate fzf input in a real tmux PTY
- **Friction**: Unit inspection could prove generated bindings but not whether literal printable input reached fzf's query buffer.
- **Fix**: Launched an isolated named tmux session, captured ANSI output, sent literal `erq?`, then confirmed the query line and removed the temporary session.
- **Rule**: For terminal keymap regressions, pair static argv tests with one disposable PTY capture using literal and special-key sends correctly.

### 2026-07-24 — Check tmux server environment before theme recovery
- **Friction**: A restored server looked like a Catppuccin failure, but live style options and color capability were healthy.
- **Fix**: Compared `show-environment -g`, pane process environments, live style options, and a disposable `NO_COLOR`/unset server pair.
- **Rule**: tmux theme state and child-process color policy are separate surfaces; verify both before reloading TPM or rewriting status styles.
