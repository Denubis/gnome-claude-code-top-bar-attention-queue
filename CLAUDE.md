# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Two-component system for monitoring Claude Code sessions from the GNOME desktop:

1. **Python daemon** (`claude-session-monitor`) — polls running Claude processes, reads `.jsonl` transcripts from `~/.claude/projects/`, determines per-session state (WORKING, IDLE, ASKING, WAITING), exposes state via D-Bus
2. **GNOME Shell extension** (`claude-attention-queue@bballsun`) — subscribes to daemon's D-Bus signals, renders per-session coloured badges in the top bar, dropdown menu focuses the Ghostty window on click

See `docs/design-plans/2026-02-08-claude-session-topbar.md` for full design.

## Architecture

- **Daemon** runs as a systemd user service, polls every 2–3s
- **D-Bus** bus name: `com.github.bballsun.ClaudeMonitor`
- **Extension** targets GNOME Shell 49 (ESM module format)
- **State detection** uses transcript mtime staleness (>5s = blocked) + last entry type

## Build & Run

```bash
# Daemon
uv run python -m claude_session_monitor.daemon

# Install everything
./install.sh
```

## Technology

- **Python 3.14+**, dependencies: `dasbus`, `PyGObject`
- **GJS** (GNOME JavaScript) for the Shell extension
- **Licence:** MIT

## Key Conventions

- CWD-to-project-dir encoding: `path.replace('/', '-').replace('.', '-')`
- Session states: `WORKING` (active), `IDLE` (blocked/permission), `ASKING` (AskUserQuestion dialog), `WAITING` (at prompt)
- Badge numbers are stable per-session (not reshuffled on exit)
