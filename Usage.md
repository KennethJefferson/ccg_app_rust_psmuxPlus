# Usage

End-to-end usage of the psmuxPlus orchestration stack. All commands assume the freshly built
fork binary (`psmux\target\release\psmux.exe`) and PowerShell 7+.

## 1. Event-driven agent orchestration (the core loop)

```powershell
$psmux = 'K:\...\psmux\target\release\psmux.exe'
$env:PSMUX_ALLOW_ELEVATED = '1'   # elevated shells only
$env:PSMUX_NO_WARM        = '1'   # first pane gets minted identity

# Session + agent pane
& $psmux new-session -d -s work -x 200 -y 50
& $psmux send-keys -t work.0 -l 'codex --dangerously-bypass-approvals-and-sandbox'
& $psmux send-keys -t work.0 Enter

# Install the done-signal hook (once per machine; codex will ask to re-trust on next launch)
& $psmux hooks install codex        # claude also supported: hooks install claude [--project-local]
& $psmux hooks status codex         # installed [trusted|trust-pending]

# Dispatch work, then block until the agent's turn actually completes
& $psmux send-keys -t work.0 -l 'summarize README.md' ; & $psmux send-keys -t work.0 Enter
& $psmux wait-event --name agent-done --pane %1 --timeout 300000   # ms! exit 0 = event, 2 = timeout

# Read the finished screen (settle closes the paint gap)
& $psmux capture-pane -t work.0 -p --settle 400

# Teardown — never kill-server
& $psmux kill-session -t work
```

Relay discipline: always pass `--timeout` (milliseconds) to `wait-event` and fall back to
`capture-pane --settle` on timeout — the bus is a latency optimization, never a hang point.

## 2. Multi-agent grids (skills)

```powershell
# Build a visible grid with named harnesses, verified ready per pane
& "$env:USERPROFILE\.claude\skills\psmuxplus-harness-grid\gridup.ps1" `
    -Spec 'claude:claude-fable-5:high, codex, agy, omp' -Visible -FreshStart

# Brief roles + protocol, then run Q&A rounds with file-based collection
& "$env:USERPROFILE\.claude\skills\psmuxplus-panel\panel-brief.ps1" -Session fanout `
    -Briefing @{ '*' = 'common rules'; '%1' = 'you are the skeptic' }
& "$env:USERPROFILE\.claude\skills\psmuxplus-panel\panel-ask.ps1" -Session fanout `
    -Question 'What is the biggest risk in this design?' -PaneTimeout @{ codex = 900 }

# Transcript: %TEMP%\psmux-panel\<session>\transcript.md

# Teardown exactly this grid's processes
& "$env:USERPROFILE\.claude\skills\psmuxplus-harness-grid\griddown.ps1" -Session fanout
```

## 3. Verifying the done-signal end-to-end

```powershell
# Installs the codex hook for real, runs a live turn, asserts wait-event wakes; restores everything.
& psmux\tests\e2e_done_signal_codex.ps1 -Visible
```

## Gotchas quick list

- codex re-trust: any change to `~/.codex/hooks.json` triggers a one-time trust prompt; until
  accepted the hook silently doesn't fire (`status codex` reports `trust-pending`).
- `wait-event --timeout` = milliseconds; codex `hooks.json` `timeout` = seconds.
- antigravity (`agy`) has no hook mechanism — agy panes are settle-polled, not event-driven.
- Uninstall cleanly with `psmux hooks uninstall codex|claude` (removes only psmux's own entries).
