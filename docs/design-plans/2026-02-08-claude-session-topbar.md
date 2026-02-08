# Claude Code Session Monitor — Top Bar Extension Design

## Summary

This project provides a GNOME Shell top bar indicator that monitors all active Claude Code sessions and displays their current state, enabling the user to quickly identify which sessions need attention. A Python daemon polls running Claude Code processes every 2–3 seconds, reads each session's transcript file to determine whether Claude is actively working (WORKING), blocked on permission (IDLE), waiting for user input on a dialog (ASKING), or ready at the prompt (WAITING). This state is exposed via D-Bus, where a GNOME Shell extension subscribes to changes and renders numbered, colour-coded badges in the top bar — one per session. Clicking a badge opens a dropdown menu listing all sessions by project name; selecting an entry focuses the corresponding Ghostty terminal window.

The implementation splits across two independent components: the daemon runs as a systemd user service and has no UI dependency, while the extension integrates with GNOME Shell's panel and menu infrastructure. Communication happens entirely through D-Bus signals and method calls, allowing the daemon to restart or the shell to reload without requiring full system coordination beyond the D-Bus bus name.

## Definition of Done

1. A Python daemon detects all running Claude Code sessions and determines their state (WORKING, IDLE, ASKING, WAITING) by reading process info and transcript files
2. The daemon exposes session state via D-Bus with a signal on state changes
3. A GNOME Shell 49 extension shows per-session numbered badges in the top bar, each coloured by state
4. A dropdown menu lists sessions by project name and state; clicking an entry focuses the Ghostty window
5. The daemon runs as a systemd user service
6. The extension installs to `~/.local/share/gnome-shell/extensions/`

## Acceptance Criteria

### claude-session-topbar.AC1: Daemon detects Claude sessions and determines state
- **claude-session-topbar.AC1.1 Success:** Daemon detects a Claude process running on a known TTY and returns it via `GetSessions()`
- **claude-session-topbar.AC1.2 Success:** Session in WORKING state (transcript actively written, pending tool call) reported as WORKING
- **claude-session-topbar.AC1.3 Success:** Session in IDLE state (transcript stale >5s with pending tool call) reported as IDLE
- **claude-session-topbar.AC1.4 Success:** Session in ASKING state (last entry is AskUserQuestion tool_use) reported as ASKING
- **claude-session-topbar.AC1.5 Success:** Session in WAITING state (turn_duration system entry or text-only assistant) reported as WAITING
- **claude-session-topbar.AC1.6 Success:** Session that exits is removed from `GetSessions()` on next poll
- **claude-session-topbar.AC1.7 Failure:** Transcript file missing or unreadable — session skipped, daemon doesn't crash
- **claude-session-topbar.AC1.8 Failure:** Malformed JSON line in transcript — line skipped, state determined from remaining entries

### claude-session-topbar.AC2: D-Bus interface exposes session state
- **claude-session-topbar.AC2.1 Success:** `GetSessions()` returns correct project_name, cwd, state, age_secs, pid for each active session
- **claude-session-topbar.AC2.2 Success:** `SessionsChanged` signal fires when a session's state transitions (e.g. WORKING → IDLE)
- **claude-session-topbar.AC2.3 Success:** `SessionsChanged` signal fires when a new session appears or an existing one exits
- **claude-session-topbar.AC2.4 Edge:** Multiple state transitions within a single poll cycle — signal fires once with latest state

### claude-session-topbar.AC3: Extension shows per-session badges
- **claude-session-topbar.AC3.1 Success:** Each active session gets a numbered badge in the top bar
- **claude-session-topbar.AC3.2 Success:** Badge colour matches session state (dim=WORKING, red=IDLE, amber=ASKING, green=WAITING)
- **claude-session-topbar.AC3.3 Success:** Badges appear when sessions start and disappear when sessions exit
- **claude-session-topbar.AC3.4 Success:** Session numbers are stable — a session keeps its number until its process exits
- **claude-session-topbar.AC3.5 Edge:** Daemon not running — extension shows grey/inactive indicator with "Monitor not running" in menu

### claude-session-topbar.AC4: Dropdown menu and window focusing
- **claude-session-topbar.AC4.1 Success:** Menu lists all sessions as `N: project-name — STATE`
- **claude-session-topbar.AC4.2 Success:** Clicking a menu entry focuses the Ghostty window
- **claude-session-topbar.AC4.3 Failure:** Ghostty window not found — click is a no-op (no crash)

## Glossary

- **D-Bus**: Inter-process communication (IPC) system used by Linux desktop environments for service discovery and message passing between applications
- **GNOME Shell**: The user interface and window manager for the GNOME desktop environment
- **systemd user service**: A background process managed by systemd that runs in the user's session rather than system-wide
- **ESM module**: ECMAScript Module format — modern JavaScript module system using `import`/`export` rather than legacy CommonJS
- **GLib main loop**: Event loop from the GLib library that handles asynchronous events, timers, and I/O in GTK/GNOME applications
- **Meta.Window**: GNOME Shell's internal representation of a desktop window with focus, position, and workspace metadata
- **St.Label**: Clutter-based UI widget from GNOME Shell's toolkit for rendering styled text in the shell interface
- **PanelMenu.Button**: GNOME Shell component for creating clickable indicators in the top bar with associated dropdown menus
- **transcript file**: Claude Code's internal `.jsonl` log file recording each turn of conversation, tool use, and system events in a session
- **mtime**: Modification time — filesystem timestamp indicating when a file was last written to
- **TTY**: Teletypewriter — in Unix, the terminal device associated with a process
- **CWD**: Current Working Directory — the directory from which a process was started
- **turn_duration**: System event logged by Claude Code marking the end of an assistant turn and return to the user prompt
- **tool_use**: Claude API message type indicating the assistant wants to invoke a tool/function
- **AskUserQuestion**: Specific Claude Code tool that pauses execution to prompt the user for input or confirmation
- **busctl**: Command-line utility for inspecting and interacting with D-Bus services
- **dbus-monitor**: Command-line utility for watching D-Bus message traffic in real time

## Architecture

Two components connected via D-Bus: a Python daemon that monitors Claude Code session state, and a GNOME Shell extension that renders per-session indicators in the top bar.

### Python Daemon (`claude-session-monitor`)

Runs as a systemd user service. Polls every 2–3 seconds to detect running Claude Code sessions and determine their state.

**Detection chain:**
1. Find `claude` processes via `/proc` scan
2. Read `/proc/<PID>/cwd` to get each session's working directory
3. Map CWD to transcript directory: `~/.claude/projects/<encoded-path>/` where the encoding is `cwd.replace('/', '-').replace('.', '-')`
4. Find the most recently modified `.jsonl` transcript file
5. Read the last ~30 entries from the transcript
6. Determine state from entry types and transcript staleness

**State determination logic:**

| State | Condition |
|-------|-----------|
| WORKING | Last entry is `user`, `assistant:tool_use` (non-AskUserQuestion), or `progress:agent_progress`, AND transcript was modified within the last 5 seconds |
| IDLE | Last entry indicates pending work (tool_use, agent_progress, user input) BUT transcript mtime is stale (>5 seconds since last write) — Claude is blocked on permission or similar |
| ASKING | Last assistant entry contains `tool_use:AskUserQuestion` |
| WAITING | Last entry is `system:turn_duration` or assistant with text-only content — Claude finished its turn, at the prompt |

The 5-second staleness threshold distinguishes WORKING (transcript actively being written) from IDLE (transcript stopped updating mid-action, indicating a blocking prompt).

### D-Bus Interface

```
Bus name:    com.github.bballsun.ClaudeMonitor
Object path: /com/github/bballsun/ClaudeMonitor
Interface:   com.github.bballsun.ClaudeMonitor

Methods:
  GetSessions() -> Array of Dict{String, Variant}
    Returns: [{
      project_name: String,   # basename of CWD
      cwd: String,            # full working directory path
      state: String,          # WORKING | IDLE | ASKING | WAITING
      age_secs: UInt32,       # seconds since last transcript write
      pid: UInt32             # Claude process PID
    }, ...]

Signals:
  SessionsChanged()           # emitted when any session's state changes
```

### GNOME Shell Extension (`claude-attention-queue@bballsun`)

A GNOME Shell 49 extension using ESM module format. Renders per-session indicators in the top bar panel.

**Top bar display:** One numbered badge per active session (`1`, `2`, `3`, `4`, ...). Each badge is coloured according to that session's state:
- WORKING — neutral/dim (leave it alone)
- IDLE — red/urgent (blocked on permission, needs attention now)
- ASKING — amber/attention (dialog waiting for user choice)
- WAITING — green/ready (at prompt, user's turn when convenient)

Badges appear/disappear as Claude sessions start and stop. Each session keeps its assigned number as long as the process is alive — numbers are not reshuffled when sessions exit.

**Dropdown menu:** Clicking any badge or the container opens a popup menu listing all sessions: `1: milkdown-crdt-spike — IDLE`. Clicking an entry calls `Main.activateWindow()` on the Ghostty window (identified by `meta_window.get_wm_class() === 'ghostty'`).

**Data flow:**
1. On `enable()`: create D-Bus proxy for `com.github.bballsun.ClaudeMonitor`, subscribe to `SessionsChanged` signal
2. On signal: call `GetSessions()`, diff against previous state, update badge colours and menu items
3. On menu item click: find Ghostty `Meta.Window` via `global.get_window_actors()`, call `Main.activateWindow()`
4. On `disable()`: disconnect signal, destroy all UI elements

## Existing Patterns

No existing code in this repository (greenfield project, initial commit only). No patterns to follow or diverge from.

The project introduces two new component patterns:
- Python D-Bus daemon with GLib main loop — standard pattern for GNOME system services
- GNOME Shell panel extension with ESM modules — standard pattern for GNOME 45+ extensions

## Implementation Phases

<!-- START_PHASE_1 -->
### Phase 1: Project Setup and Daemon Skeleton

**Goal:** Python project structure with a daemon that starts, connects to D-Bus, and runs a GLib main loop.

**Components:**
- `pyproject.toml` — project metadata, dependencies (`dasbus`, `PyGObject`)
- `src/claude_session_monitor/__init__.py` — package init
- `src/claude_session_monitor/daemon.py` — D-Bus service registration, GLib main loop, signal emission
- `config/claude-session-monitor.service` — systemd user service unit file
- `config/com.github.bballsun.ClaudeMonitor.service` — D-Bus activation service file

**Dependencies:** None (first phase)

**Done when:** `uv run python -m claude_session_monitor.daemon` starts, claims the D-Bus bus name, and responds to `GetSessions()` returning an empty array. `busctl --user call` can invoke the method.
<!-- END_PHASE_1 -->

<!-- START_PHASE_2 -->
### Phase 2: Session Scanner

**Goal:** Detect running Claude Code sessions, read transcripts, and determine per-session state.

**Components:**
- `src/claude_session_monitor/scanner.py` — process detection (`/proc` scanning), CWD-to-project-dir mapping, transcript reading, state determination
- `src/claude_session_monitor/states.py` — state constants (WORKING, IDLE, ASKING, WAITING), state determination logic from transcript entries

**Dependencies:** Phase 1 (daemon skeleton)

**Done when:** Scanner correctly identifies running Claude sessions, maps them to transcript files, and returns the correct state for each. Tests verify state determination logic against sample transcript data for all four states.
<!-- END_PHASE_2 -->

<!-- START_PHASE_3 -->
### Phase 3: Daemon Integration

**Goal:** Wire scanner into daemon's poll loop and emit D-Bus signals on state changes.

**Components:**
- `src/claude_session_monitor/daemon.py` — integrate scanner polling via `GLib.timeout_add_seconds()`, diff session state between polls, emit `SessionsChanged` signal when state changes
- `src/claude_session_monitor/session_store.py` — in-memory store of current sessions, stable number assignment, diffing logic

**Dependencies:** Phase 1 (daemon), Phase 2 (scanner)

**Done when:** Daemon polls every 2–3 seconds, `GetSessions()` returns current sessions with correct states, `SessionsChanged` signal fires when a session's state transitions. `dbus-monitor` shows signals on state change.
<!-- END_PHASE_3 -->

<!-- START_PHASE_4 -->
### Phase 4: GNOME Shell Extension — Indicator and Menu

**Goal:** Top bar badges and dropdown menu, driven by D-Bus data.

**Components:**
- `extension/claude-attention-queue@bballsun/metadata.json` — extension metadata targeting GNOME Shell 49
- `extension/claude-attention-queue@bballsun/extension.js` — `PanelMenu.Button` with per-session `St.Label` badges, `PopupMenu` with session entries, D-Bus proxy and signal subscription
- `extension/claude-attention-queue@bballsun/stylesheet.css` — badge colours per state (WORKING: dim, IDLE: red, ASKING: amber, WAITING: green)

**Dependencies:** Phase 3 (working daemon with D-Bus interface)

**Done when:** Extension loads in GNOME Shell, shows numbered badges coloured by state, dropdown menu lists sessions with project names, badges update when `SessionsChanged` fires.
<!-- END_PHASE_4 -->

<!-- START_PHASE_5 -->
### Phase 5: Window Focusing

**Goal:** Clicking a menu entry focuses the Ghostty window.

**Components:**
- `extension/claude-attention-queue@bballsun/extension.js` — window lookup via `global.get_window_actors()` and `meta_window.get_wm_class()`, `Main.activateWindow()` call on menu item activation

**Dependencies:** Phase 4 (extension with menu)

**Done when:** Clicking a session entry in the dropdown brings the Ghostty window to the front.
<!-- END_PHASE_5 -->

<!-- START_PHASE_6 -->
### Phase 6: Daemon Not Running / Error Handling

**Goal:** Extension handles daemon unavailability gracefully. Daemon handles missing/locked transcripts.

**Components:**
- `extension/claude-attention-queue@bballsun/extension.js` — fallback UI when D-Bus proxy can't connect (grey indicator, "Monitor not running" menu item)
- `src/claude_session_monitor/scanner.py` — error handling for missing `/proc` entries, unreadable transcripts, malformed JSON lines

**Dependencies:** Phase 4 (extension), Phase 2 (scanner)

**Done when:** Extension shows a clear "not connected" state when daemon isn't running. Daemon skips sessions with inaccessible transcripts without crashing. Both components recover gracefully when conditions change.
<!-- END_PHASE_6 -->

<!-- START_PHASE_7 -->
### Phase 7: Install Script and systemd Integration

**Goal:** One-command installation.

**Components:**
- `install.sh` — installs Python package via `uv`, symlinks extension directory, copies systemd and D-Bus service files, enables and starts daemon, enables extension
- `config/claude-session-monitor.service` — systemd user service unit (refined from Phase 1)
- `config/com.github.bballsun.ClaudeMonitor.service` — D-Bus activation file

**Dependencies:** All prior phases

**Done when:** Running `./install.sh` on a fresh machine with GNOME 49 and Ghostty results in a working top bar indicator tracking Claude Code sessions.
<!-- END_PHASE_7 -->

## Additional Considerations

**Staleness threshold tuning:** The 5-second threshold for IDLE detection is a starting point. In practice, some tool calls (like large file reads) may take >5 seconds of legitimate processing without transcript writes. If false IDLE positives become a problem, the threshold can be increased or supplemented with CPU usage checks on the Claude process.

**Session number stability:** Numbers are assigned in discovery order and persist as long as the process is alive. If session 2 exits, sessions 1, 3, 4 remain — the gap is not filled. This prevents confusing renumbering mid-work.

**Badge density at high session counts:** At 6+ sessions, per-session badges may consume noticeable top bar space. If this becomes a problem, badges can be narrowed (smaller font, dots instead of numbers) rather than collapsed into a summary count — the per-session visibility is the point.

**Transcript file format dependency:** The state detection relies on Claude Code's `.jsonl` transcript format, which is an internal implementation detail and could change between versions. The scanner should handle format changes defensively (unknown entry types → treat as WORKING).
