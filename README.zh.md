[English](README.md) | [繁體中文](README.zh.md)

# tmux-expert

Specialist knowledge for controlling tmux programmatically and bridging it to web UIs.

## 說明

tmux Expert provides specialist knowledge for querying tmux state, reading pane content, injecting input, managing sessions/windows/panes, and building web interfaces that interact with tmux.

## 功能特色

- Query tmux session, window, and pane state programmatically
- Read pane output and inject keystrokes via `send-keys`
- Manage sessions, windows, and panes with shell scripts
- Bridge tmux to web UIs via HTTP API patterns
- Handle pane synchronization and multi-window layouts
- Covers both interactive and scripted tmux automation

## 使用方式

透過以下觸發語句呼叫 Claude Code 來使用此技能：

- "control tmux"
- "build tmux web UI"
- "tmux send keys"
- "tmux automation"
- "tmux 控制"
- "remote terminal control"

## 相關技能

- [`macos-ui-automation`](https://github.com/joneshong-skills/macos-ui-automation)
- [`scheduler`](https://github.com/joneshong-skills/scheduler)
- [`executor`](https://github.com/joneshong-skills/executor)

## 安裝

將技能目錄複製到 Claude Code 技能資料夾：

```
cp -r tmux-expert ~/.claude/skills/
```

放置在 `~/.claude/skills/` 的技能會被 Claude Code 自動發現，無需額外註冊。
