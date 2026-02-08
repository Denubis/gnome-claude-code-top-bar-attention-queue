# Dependency Rationale

Falsifiable justifications for every direct dependency. Each entry records why the package was added, what evidence supports its use, and who it serves.

Maintained by design plans (when adding deps) and controlled-dependency-upgrade (when auditing). Reviewed by restate-our-assumptions (periodic philosophical audit).

## dasbus

**Added:** 2026-02-08
**Design plan:** docs/design-plans/2026-02-08-claude-session-topbar.md
**Claim:** We use dasbus to expose the session monitor daemon as a D-Bus service, registering a bus name and emitting signals that the GNOME Shell extension subscribes to.
**Evidence:** `src/claude_session_monitor/daemon.py` will use dasbus for D-Bus service registration, method export, and signal emission.
**Serves:** Runtime (daemon process)

## PyGObject

**Added:** 2026-02-08
**Design plan:** docs/design-plans/2026-02-08-claude-session-topbar.md
**Claim:** We use PyGObject to access the GLib main loop (`GLib.MainLoop`, `GLib.timeout_add_seconds`) for the daemon's poll-based event loop.
**Evidence:** `src/claude_session_monitor/daemon.py` will use `gi.repository.GLib` for the main loop and timer-based polling.
**Serves:** Runtime (daemon process)
