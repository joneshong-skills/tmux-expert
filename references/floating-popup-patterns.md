# Floating Popup Patterns (`tmux display-popup`)

Patterns for building persistent, resizable, keybinding-driven floating popups on
top of `tmux display-popup` / `tmux popup`. Cannibalized from
[`omerxx/tmux-floax`](https://github.com/omerxx/tmux-floax) on 2026-07-06.

Load this reference when the task involves a **stateful** floating popup — one that
survives close/reopen, rebinds keys while open, or needs to resize — rather than a
throwaway one-shot popup.

---

## Technique 1: Persistent floating pane (popup + scratch session, the three-piece set)

A popup that keeps its state across open/close is really a popup *attached to a
dedicated background session*:

```bash
tmux popup -E "tmux attach-session -t scratch"
```

Closing the popup only detaches the client; the `scratch` session keeps running, so
reopening lands back in the exact same state (same running program, same scrollback).

**All three pieces are mandatory — omitting any one breaks it:**

1. **Attach a dedicated session inside the popup.** The popup runs
   `tmux attach-session -t <scratch>` against a session that exists solely to back
   the popup (create it lazily if missing).
2. **`set-option -t <scratch> detach-on-destroy on`.** When the last window in the
   scratch session closes, this makes the client *detach* (which tears down the
   popup) instead of hopping to some other session. Without it, closing the popup's
   program silently drops the user into an unrelated session.
3. **Before moving a window out of the scratch session, if it is the last window,
   `tmux neww -d` first.** A tmux session with zero windows dies. If you move the
   only window away (e.g. to "promote" the popup content into the main session)
   without first creating a placeholder detached window, the scratch session is
   destroyed and the *next* pop fails.

Pieces (2) and (3) are two faces of the same invariant — **a session with no windows
dies** — handled at the two moments it can bite (last-window-close, last-window-move).

**Optional path-follow.** On toggle, compare the main pane's `pane_current_path`
against the scratch pane's; if they differ, `send-keys` a `cd` into the popup so it
tracks the caller's working directory:

```bash
main_path=$(tmux display-message -p -t "$caller" '#{pane_current_path}')
pop_path=$(tmux display-message -p -t scratch '#{pane_current_path}')
[ "$main_path" != "$pop_path" ] && tmux send-keys -t scratch " cd \"$main_path\"" Enter
```

Source: `omerxx/tmux-floax` `scripts/utils.sh:68-116`.

---

## Technique 2: Popup-scoped transient keybindings

Bind keys only while the popup is open; unbind on close. Add a lock/unlock escape
hatch so the program *inside* the popup can receive the same keys:

- On open → bind the popup's control keys.
- On close → unbind them.
- **Lock** → unbind everything except the single **unlock** key, so keystrokes fall
  through to the program running in the popup. Change the popup title to an unlock
  hint (e.g. `-- LOCKED (prefix U to unlock) --`) as immediate visual feedback.
- **Unlock** → restore the full popup key set.

### HARD RULE — do not copy floax's `bind -n` verbatim in multi-client setups

floax binds its control keys with `bind -n` (the **root** key table). Root-table
bindings are **server-global**: every client attached to the tmux server sees them.
In a multi-client / multi-host scenario (e.g. a `tmux-webui` federation where several
browsers or hosts attach concurrently), all clients receive the same hotkeys and
stomp on each other.

For multi-client scenarios, use a **client-scoped key table** instead:

```bash
# Enter a custom table for THIS client only
tmux switch-client -T floatpop

# Inside that table, each binding fires once then auto-returns to root
tmux bind -T floatpop q  detach-client
tmux bind -T floatpop U  switch-client -T floatpop   # (re-enter for chords)
```

`switch-client -T` changes the key table for the invoking client alone, so concurrent
clients are unaffected. `bind -n` is acceptable **only** in a single-user / single-client
environment — which is floax's original design target.

Source: `omerxx/tmux-floax` `scripts/utils.sh:27-45`, `scripts/zoom-options.sh:37-52`.

---

## Technique 3: Resize via detach-reopen

tmux has **no API to resize an already-open popup**. The equivalent is a
close-and-reopen cycle with externalized dimensions:

1. Write the new width/height into state (a tmux user option, see below).
2. `detach-client` to close the current popup.
3. Immediately reopen the popup with the new dimension arguments.

Add a **bounds check**: the new size must not exceed the origin session's actual
window width/height, or the reopen produces a clipped/misplaced popup:

```bash
win_w=$(tmux display-message -p '#{window_width}')
win_h=$(tmux display-message -p '#{window_height}')
new_w=$(( new_w > win_w ? win_w : new_w ))
new_h=$(( new_h > win_h ? win_h : new_h ))
```

**General principle:** when a lower-level API has no live-update call, simulate live
adjustment with **"close + reopen + externalized state."** The state survives the
teardown; the reopen reads it.

Source: `omerxx/tmux-floax` `scripts/zoom-options.sh`.

---

## Anti-patterns to avoid

### bash flat-namespace function shadowing across multiple sourced files

floax's `embed.sh` redefines a `pop()` function that shadows the same-named function
in `utils.sh`. Because sourced bash files share one flat namespace, a bare call to
`pop` inside `utils.sh` can resolve to the *wrong* implementation depending on source
order. When borrowing code from a multi-file shell plugin, **prefix every function**
(`floax_pop`, `floax_toggle`, …) so cross-file calls are unambiguous.

### Do not use `tmux setenv -g` to store plugin state

`setenv -g` pollutes the tmux environment namespace (which is meant for child-process
env inheritance). Use a tmux **custom user option** (`@option`) instead — it is
semantically equivalent for state storage and keeps the environment namespace clean:

```bash
tmux set-option -g @floatpop_width  80    # write
tmux show-option -gv @floatpop_width      # read
```

---

## Backfire exceptions

- **One-shot command popups don't need the three-piece set.** For a popup that shows
  something and exits (lazygit, a cheatsheet, `htop`), plain
  `tmux display-popup -E '<cmd>'` is the correct answer. Bolting a persistent scratch
  session onto a look-and-leave popup is over-engineering.
- **Technique 2's client-scoped requirement only holds in multi-client setups.** In a
  personal single-machine, single-client environment, copying floax's `bind -n`
  verbatim is fine.

---

## Alignment with existing conventions

- **Complements the one-way-nesting invariant (herdr lesson).** The scratch session
  is an *attach target*, not a mirror — attaching to it from a popup does not form the
  mutual-attach cycle that caused the herdr↔tmux mirror loop. Persistent popups here
  are safe precisely because nesting stays one-directional.
- **Complements the cheatsheet popup (`prefix + ?`).** That is the canonical example
  of a *one-shot* popup; the three-piece set here is the canonical example of a
  *persistent* popup. The two divide the labor by popup lifetime.
- **Provenance:** intelflow report `019f352d037a7b239a76015835918522`.
