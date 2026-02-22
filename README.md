[English](README.md) | [繁體中文](README.zh.md)

# tmux-expert

Specialist knowledge for controlling tmux programmatically and bridging it to web UIs.

## Description

tmux Expert provides specialist knowledge for querying tmux state, reading pane content, injecting input, managing sessions/windows/panes, and building web interfaces that interact with tmux.

## Features

- Query tmux session, window, and pane state programmatically
- Read pane output and inject keystrokes via `send-keys`
- Manage sessions, windows, and panes with shell scripts
- Bridge tmux to web UIs via HTTP API patterns
- Handle pane synchronization and multi-window layouts
- Covers both interactive and scripted tmux automation

## Usage

Invoke by asking Claude Code with trigger phrases such as:

- "control tmux"
- "build tmux web UI"
- "tmux send keys"
- "tmux automation"
- "tmux 控制"
- "remote terminal control"

## Related Skills

- [`macos-ui-automation`](https://github.com/joneshong-skills/macos-ui-automation)
- [`scheduler`](https://github.com/joneshong-skills/scheduler)
- [`executor`](https://github.com/joneshong-skills/executor)

## Install

Copy the skill directory into your Claude Code skills folder:

```
cp -r tmux-expert ~/.claude/skills/
```

Skills placed in `~/.claude/skills/` are auto-discovered by Claude Code. No additional registration is needed.
