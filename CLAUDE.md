# psmuxPlus — Claude Code working instructions

This is the workspace wrapper around the psmux Rust fork (`psmux/`, its own git repo) and the
psmuxplus orchestration skills (symlinked; own repos in `~/.claude/skills/`).

## Hard rules (each learned the expensive way)

- **Never `kill-server`** — it force-kills every `psmux.exe` by image name, including real
  sessions. Tear down with `kill-session -t <name>` only, and clean up spawned attach windows.
- **Elevated shell**: every live psmux run needs `PSMUX_ALLOW_ELEVATED=1` (a refused server
  surfaces only as "failed to create session").
- **Cargo**: only `cargo test --bin psmux` (full `cargo test` runs the ~2400-test suite three
  times). Run cargo FOREGROUND — backgrounded cargo can phantom-hang on this machine.
- **Warm-claim gap**: a session's very first pane lacks identity env by default — set
  `PSMUX_NO_WARM=1` (or use post-claim panes) before relying on `notify`/`hook-notify`.
- **Units**: psmux `wait-event --timeout` is **milliseconds**; the `timeout` field inside
  codex's `hooks.json` is **seconds**.
- **codex hooks** run via `powershell.exe -NoProfile -Command <string>` — hook commands must use
  the call-operator form `& '<exe>' args` (a bare `"<exe>" args` string is a parser error).
- **Cross-shell state does not persist** between separate PowerShell tool calls — do a whole
  live test (dispatch + assertion + teardown) in ONE script.

## Where things are

- Rust source: `psmux/src/` — event bus in `events.rs`, hook installers in `hooks_install.rs`,
  CLI arms in `main.rs`, help text in `cli.rs`.
- Specs/plans: `psmux/docs/superpowers/` (repo-level) and `docs/superpowers/` (workspace-level).
- Agent-events user docs: `psmux/docs/agent-events.md`.
- Live e2e for the codex done-signal: `psmux/tests/e2e_done_signal_codex.ps1` (`-Visible` switch).
- Skills: `psmuxplus-harness-grid` (grid up/swap/down) and `psmuxplus-panel` (brief/ask/collect).

## Conventions

- Feature work happens on branches in `psmux/` (remote `fork` = KennethJefferson/psmux;
  `origin` = upstream read-only). PR-review before merge; SDD ledger in `.superpowers/sdd/`.
- Live demos/tests: announce, run visibly when asked, verify checkpoints, and always tear down
  in a `finally` block.
