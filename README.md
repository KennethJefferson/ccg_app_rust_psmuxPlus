# psmuxPlus

A Rust-based terminal-multiplexer workspace built around a fork of [psmux](https://github.com/psmux/psmux)
(native tmux for Windows), extended with **deterministic multi-agent orchestration**:

- **Agent-events bus** (in-memory, per-server): `cursor`, `events`, `wait-event`, `notify`,
  `hook-notify`, and `capture-pane --settle` — agents in panes publish lifecycle events instead
  of being scraped by polling.
- **Event-driven done-signal**: `psmux hooks install claude|codex` wires an agent's turn-end hook
  to publish `agent-done` onto the bus, so an orchestrator blocks on
  `wait-event --name agent-done --pane %N --timeout <ms>` instead of settle-polling.
- **PowerShell orchestration skills** (developed alongside, in their own repos):
  - `psmuxplus-harness-grid` — stand up a visible grid with a named agent harness
    (claude/codex/agy/omp) launched and verified ready per pane; swap mid-session; precise teardown.
  - `psmuxplus-panel` — brief the grid (roles/personas), broadcast questions, and collect every
    answer via per-round files with deadlines and a durable transcript.

## Repository layout

| Path | What it is |
|---|---|
| `psmux/` | The Rust fork (own git repo — [KennethJefferson/psmux](https://github.com/KennethJefferson/psmux)); not vendored here |
| `docs/superpowers/` | Workspace-level design specs and implementation plans |
| `psmuxplus-harness-grid`, `psmuxplus-panel` | Local symlinks to the skills (own repos); not vendored here |
| `CLAUDE.md` | Working instructions for Claude Code sessions in this workspace |
| `Usage.md` | End-to-end usage of the orchestration stack |
| `Changelog.md` | Workspace changelog (Keep a Changelog format) |

## Status

Increment 1 (identity model + event bus + Claude hooks) and the codex done-signal
(hooks installer, PowerShell call-operator hook command, live e2e verified) are complete.
See `Changelog.md` for details.
