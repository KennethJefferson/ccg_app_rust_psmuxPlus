# Changelog

All notable changes to the psmuxPlus workspace are documented in this file.
This file is append-only. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versioning follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.1] - 2026-07-16

### Added
- `psmux/` — the Rust fork is now included as a **git submodule** pointing at
  [KennethJefferson/psmux](https://github.com/KennethJefferson/psmux)
  (branch `feature/agent-events-done-signal`, pinned at `f64e15b`).

### Removed
- `__references/` no longer tracked (back in `.gitignore`; content remains local-only):
  the cmux submodule, `learning-cmux-with-agents-main/`, `advisor-orchestrator-worker/`,
  and the cmux video transcript.

## [1.0.0] - 2026-07-15

### Added
- **Agent-events increment 1** (psmux fork, merged as PR #1): pane/session identity model,
  in-memory event bus with replay ring, `cursor` / `events` / `wait-event` / `notify` /
  `hook-notify` CLI verbs, `capture-pane --settle`, Claude Code hook installer
  (`psmux hooks install claude [--project-local]`), unified shutdown, pre-auth hardening,
  elevated-shell refusal, registry sandboxing, warm-server retirement.
- **Settle pinned-capture fix** (PR #2): captures are pinned to the resolved pane id —
  wrong-pane capture under concurrency impossible by construction.
- **Event-driven done-signal for codex** (branch `feature/agent-events-done-signal`, live
  e2e verified): `psmux hooks install|uninstall|status codex` (global, `Stop` → `agent-done`),
  trust-pending detection against codex's trust-hash gate, loud re-trust warning, and the
  PowerShell call-operator hook command form required by codex's hook runner.
- **Hardened installer file spine** (shared by claude + codex): read-error-safe `load`
  (NotFound-only means empty), atomic rename-over writes, backup-on-change-only,
  exact-match idempotency, command-level uninstall that preserves sibling hooks.
- **Orchestration skills** (own repos, symlinked here): `psmuxplus-harness-grid`
  (grid up / harness launch+readiness / swap / precise teardown, 54 tests) and
  `psmuxplus-panel` (brief / broadcast / file-based answer collection, 52 tests).
- Workspace docs: `README.md`, `CLAUDE.md`, `Usage.md`, this changelog.

### Changed
- Skills renamed from `psmux-*` to `psmuxplus-*` prefix (`psmuxplus-harness-grid`,
  `psmuxplus-panel`).

### Fixed
- codex Stop-hook "exited with code 1": codex executes hook commands via
  `powershell.exe -NoProfile -Command`, so the installed command now uses the call-operator
  form `& '<exe>' hook-notify codex stop` instead of a bare quoted-exe string.
- `wait-event --timeout` documentation corrected to milliseconds (codex's own `hooks.json`
  `timeout` field is seconds).
- Warm-claimed-pane warning no longer fires in plain (non-psmux) tmux sessions — gated on
  `PSMUX_SESSION_UID`.
