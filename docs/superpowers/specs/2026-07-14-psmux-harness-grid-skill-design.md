# psmux-harness-grid skill — design

**Date:** 2026-07-14 (rev 3.2 — post gpt-5.6-sol review rounds 1-2; rev 3.1: at-shell detection hardened after Task 4 surfaced `pane_current_command` flakiness; rev 3.2: teardown root-PID identity validation added after Task 6 review — manifest pane records carry `PanePidCreated`, live psmux pane PIDs preferred over manifest at snapshot time)
**Status:** approved for planning
**Origin:** 2026-07-13 persona-panel session — 4 heterogeneous harnesses (claude/codex/agy/omp) stood up manually in a psmux grid; this skill makes that flow deterministic and repeatable.

## Purpose

A thin, deterministic skill layer between an agent (Claude Code) and psmux: one call stands up a visible, maximized, tiled grid of panes, clears them, launches a named agent harness in each pane (with model/effort), verifies each is actually ready, and returns a machine-readable pane map. Companion helpers swap a pane's harness mid-session and tear the grid down cleanly.

The skill mimics-by-script what a human operator does in psmux — window, maximize, cls, launch, watch for the prompt — so the agent gets reproducible behavior instead of re-improvising send-keys choreography each session.

## Relationship to existing skills

**Sibling, compositional. Nothing is deleted.**

- Grid building delegates to `psmux-fanout`'s `fanout.ps1 -Provision` (visible/maximized/tiled/fresh-start/layouts all inherited via pass-through flags). No pane-building logic is reimplemented. Socket-burst retries reuse fanout's `Invoke-PsmuxWithRetry`.
- Teardown delegates to fanout's `cleanup.ps1 -NoReap` for session-kill + visible-window close, then does its own precise harness reaping (below).
- `psmux-connector` remains the injection/broadcast layer for driving work into a running grid. This skill deliberately owns **no** briefing/broadcast/extraction features. `connect-manifest.json` remains the connector's authoritative seam, untouched; this skill's `grid-manifest.json` is supplementary metadata that the connector never reads.
- This skill owns exactly one new concern: **what runs in each pane and whether it is actually ready.**
- Deprecation of fanout/connector as user-visible skills is deferred: after this skill proves itself in real sessions, fanout's SKILL.md may be demoted to "lower-layer engine" with a pointer here. Not part of this build.
- Launch injection is a deliberately simple sequence (`send-keys -l` → pause → lone Enter) because it targets a **bare shell prompt**, not a TUI input box; the connector's scaled-settle/resubmission machinery solves a TUI problem the launch path does not have. Quirk interactions (trust gates) that do talk to TUIs send single keys only.

## Pane identity and targeting

A pane's **identity** is its stable `%id` from fanout's manifest (creation order — never visual `pane_index`, which is a permutation and gets renumbered by layout changes). But psmux `%id`s are **not server-unique**: fanout gotcha #9 records an observed wrong-session resolution when two sessions both have a `%1`, so bare `%id` is not a safe **target** either.

**Targeting rule:** before each psmux operation, resolve the pane's current `pane_index` fresh via `list-panes -t <session>:<window> -F "#{pane_index} #{pane_id}"` (scoped to this grid's session/window), then target the fully qualified `<session>:<window>.<index>`. Identity by id, addressing by fresh id→index resolution — immune to both the cross-session `%id` collision and index renumbering. A pane id missing from the resolution is how `pane-dead` is detected on send.

At provision the engine also records each pane's `#{pane_pid}` (the pane's shell PID) — the root of that pane's process ownership for teardown and exit-detection.

## File layout

```
~\.claude\skills\psmux-harness-grid\
├── SKILL.md            # when-to-use, quick-start, parameters, gotchas
├── gridup.ps1          # engine: provision → cls → launch → verify → pane map
├── griddown.ps1        # teardown: pane-descendant snapshot → cleanup.ps1 → reap → RAM report
├── harnesses.json      # harness registry (structured launch specs, ready patterns, quirks)
├── _harness-lib.ps1    # Start-PaneHarness, Wait-HarnessReady, Swap-PaneHarness
└── tests\Grid.Tests.ps1
```

## Harness registry (`harnesses.json`)

Top level: `{ "SchemaVersion": 1, "Harnesses": { ... } }`. One entry per known harness; adding a CLI whose behavior fits existing quirk semantics = one JSON entry (a genuinely new quirk still needs engine code — that is expected and fine).

| Field | Meaning |
|---|---|
| `executable` | Program name, e.g. `"claude"`. |
| `baseArgs` | Fixed argument array, e.g. `["--dangerously-skip-permissions"]`. |
| `modelArgs` | Argument array emitted only when Model supplied; `{model}` placeholder in an element is replaced, e.g. `["--model", "{model}"]`. |
| `effortArgs` | Same for Effort, e.g. `["--effort", "{effort}"]`. |
| `readyPattern` | Regex matched against `capture-pane -p -J` output to declare the harness ready. |
| `exitCommand` | How to quit the harness cleanly (e.g. `/exit`). Fallback: Ctrl+C ×2. |
| `quirks` | Declarative flags the engine implements (below). **Unknown quirk = hard error at registry validation** (fail closed — a typo must not silently degrade into a timeout). |

No launch templates as strings — structured args sidestep quoting/injection questions. Model/effort values are validated against a safe charset (`[\w.:-]+`) before use. No `cls` in launch specs — clearing is the engine's dedicated pass. No `processNames` — teardown ownership comes from pane PIDs, not name matching.

Registry validation (schema version, required fields, regex compile, quirk names) runs before any pane is built; unknown harness name is a hard error.

Initial entries (validated 2026-07-12/13):

- **claude** — `claude --dangerously-skip-permissions [--model M] [--effort E]`; ready: `bypasspermissions|❯`; exit `/exit`; quirks: `trust-gate-enter`.
- **codex** — `codex --dangerously-bypass-approvals-and-sandbox [-m M] [-c model_reasoning_effort=E]`; ready: model+effort banner; quirks: `self-update-exit`, `esc-clears-stray-prompt`.
- **agy** — `agy`; ready: `\? for shortcuts`; quirks: `trust-gate-enter`. (Note for future drivers: agy renders `<<TAG>>` markup as `<>`.)
- **omp** — `omp --auto-approve`; ready: status bar / `Ready\.`; quirks: `self-update-exit`.
- **pi** — `pi-team`; quirks: `extended-keys`, `session-purpose-first`. Its `readyPattern` was never captured during the 2026-07 campaign: the registry ships with the four validated harnesses above, and pi's entry is added during implementation only if its pattern is captured in the live smoke test — otherwise pi is deferred to a follow-up registry edit.

### Quirk semantics

| Quirk | Engine behavior |
|---|---|
| `self-update-exit` | If, before readiness, the harness process exits back to the shell, relaunch **once**, keep polling. At-shell detection is **corroborated and debounced** (rev 3.1, from Task 4 field finding): `#{pane_current_command}` alone is unreliable on Windows/ConPTY (observed stuck at the shell name for ~⅓–½ of launches), so at-shell requires BOTH the shell name AND zero live child processes under the recorded pane PID, observed on **2 consecutive sweeps**, before any action (relaunch or `launch-failed`). A relaunch failure (pane vanished) yields `pane-dead`, never an aborted sweep. |
| `trust-gate-enter` | If a trust/folder prompt is detected, send a lone Enter, keep polling. |
| `esc-clears-stray-prompt` | After readiness, send Esc once to discard any buffered text the TUI may have captured as a prompt. |
| `extended-keys` | Session-level: if any requested pane's harness declares it, run `psmux set extended-keys on` once for the session, before any launches. |
| `session-purpose-first` | SKILL.md-documented manual step (engine cannot answer it); flagged in the returned object as `NeedsSetup`. |

## Engine call shape

```powershell
$grid = & gridup.ps1 -Panes @(
  @{ Harness='claude'; Model='claude-fable-5'; Effort='high' },
  @{ Harness='codex';  Model='gpt-5.6-sol';    Effort='high' },
  @{ Harness='agy' },
  @{ Harness='omp' }
) -Visible -FreshStart
```

- String shorthand: `-Spec 'claude:claude-fable-5:high, codex:gpt-5.6-sol:high, agy, omp'` → parsed to the same structure (`harness[:model[:effort]]`, comma-separated).
- Pass-through flags to `fanout.ps1 -Provision`: `-Visible`, `-FreshStart`, `-LayoutPath`, `-Session`, `-AgentCwd`, `-CanvasWidth`, `-CanvasHeight`. Note `-FreshStart` inherits fanout semantics: it tears down **every** psmux session, not just this grid's — documented in SKILL.md.
- Engine sets `PSMUX_ALLOW_ELEVATED=1` itself. (The guard it satisfies lives in the psmux server binary — elevated shells are refused without it; the skills never see it.)

## Engine flow

1. **Pre-flight:** registry + `-Panes` validation (before anything is built). Free-RAM check — warn < 2.5 GB, abort < 1 GB unless `-Force`.
2. **Provision:** `fanout.ps1 -Provision -PaneCount N` + pass-throughs → pane ids from `connect-manifest.json`, creation order; `-Panes[i]` maps to manifest index `i`. **Manifest liveness check** (fanout warns-but-continues on manifest write failure, so a stale file from a prior build can sit at the same path): every manifest pane id must exist in live `list-panes` output for the session and the count must equal N — mismatch is a hard failure. Record each pane's `#{pane_pid}`.
   **Early manifest:** immediately write `grid-manifest.json` with `State='building'` and the per-pane records (ids, PanePids, requested harness/model/effort) — so failure cleanup and a standalone `griddown` always have the ownership data, even if the run dies mid-launch.
3. **Session quirks:** apply `extended-keys` once if any pane needs it.
4. **CLS pass:** broadcast `cls` + Enter to every pane to flush stdio junk left from pane creation. Explicit step, before any harness launch.
5. **Launch (staggered ~2.5s/pane):** set pane title `<harness>-<model>`; send launch args as `send-keys -l <text>`, pause 600ms, lone Enter.
6. **Readiness loop:** one sequential poll loop over all pending panes, ~5s per sweep; each pane's deadline is its own launch time + budget (90s default). A failed `capture-pane`/psmux call is transient — retried next sweep until that pane's deadline (transient psmux socket refusals are already retried inside `Invoke-PsmuxWithRetry`; note that helper *returns* non-transient failures rather than throwing, so callers must check its exit status). Per sweep, per pane:
   - `readyPattern` match in `capture-pane -p -J` → ready
   - `#{pane_current_command}` back to shell before readiness: with `self-update-exit` quirk → relaunch once, continue; without it (or on second exit) → `launch-failed`
   - pane no longer exists → `pane-dead`
   - `trust-gate-enter` quirk + trust prompt visible → send Enter, continue
7. **Post-ready guard:** apply `esc-clears-stray-prompt` where declared.
8. **Return + manifest finalize:** one object per pane — `Index, PaneId, PanePid, Harness, Model, Effort, Status, Title, NeedsSetup` with `Status ∈ ready | relaunched-ready | timeout | launch-failed | pane-dead` — and update `grid-manifest.json` (written early in step 2) to `State='up'` with final per-pane statuses. The manifest lives beside fanout's `connect-manifest.json` and carries `SchemaVersion`, the fanout `BuildId` it extends, session name, and grid-up timestamp. **The connector's seam remains `connect-manifest.json`, unchanged**; `grid-manifest.json` is for humans, drivers, and `griddown`.

**Failure cleanup:** if provisioning or the launch phase throws (as opposed to individual panes reporting `timeout`/`launch-failed`, which return normally in the pane map), the engine tears down its own session via the griddown path before rethrowing, unless `-KeepPartial` is passed. The early `State='building'` manifest guarantees griddown has the pane-PID ownership data even for a mid-launch death. Beyond that, no transactional state machine — `-FreshStart` on the next build is the coarse backstop.

## Swap

`Swap-PaneHarness -PaneId %1 -Harness omp -Model glm-5.2 [-Effort high]`
→ send old harness's `exitCommand` (fallback Ctrl+C ×2) → confirm the pane returned to the shell via `#{pane_current_command}` (abort with a clear error if it never does; never launch into a still-occupied pane) → run flow steps 4–7 for that pane → update `grid-manifest.json`.

Concurrent swap/inject/teardown coordination is out of scope: this is a single-operator tool driven by one orchestrator at a time.

## Teardown (`griddown.ps1`)

Ordering matters — ownership info must be captured while the session is alive:

1. **Snapshot first:** determine the root pane PIDs — **live psmux is authoritative** (rev 3.2): while the session is alive, take pane PIDs from `list-panes -F "#{pane_pid}"` (no reuse possible on a live session); only if the session is already gone, fall back to the manifest's recorded `PanePid`s, each validated against the manifest's `PanePidCreated` (pane shell CreationDate captured at gridup — the manifest schema is 10 fields) and **skipped with a warning on mismatch or absence** (`root-pid-reused-skipped`). Then enumerate each root's living descendant processes (CIM parent-chain walk), recording for each: `(PID, CreationDate, executable path)`. This set — descendants of this grid's panes — is exactly what the grid owns. No name matching, no time-window heuristics; an unrelated `codex`/`omp` started elsewhere (even after grid-up) is untouchable by construction. Snapshot is taken immediately before the session kill to minimize the window for late-spawned descendants (a residual, milliseconds-wide gap — accepted).
2. `cleanup.ps1 -Session <s> -NoReap` — kills the session and closes the visible window; its broad claude-reap (which spans all psmux servers) is bypassed in favor of step 3.
3. **Reap the snapshot:** for each surviving snapshot PID, **revalidate identity before killing** — re-query the process and require CreationDate (and path, when readable) to match the snapshot; a mismatch means PID reuse, skip it. Report per-PID outcomes (`killed | already-exited | pid-reused-skipped | kill-failed`). `griddown` is idempotent — safe to re-run after a partial failure.
4. Report free RAM; warn if psmux.exe processes remain.

Known fanout papercuts observed during review, **not** fixed here (candidates for a fanout follow-up): `cleanup.ps1`'s own claude-reap spans every psmux server's descendants rather than one session's; `window.pid` close has a theoretical PID-reuse race (mitigated by its protected-ancestor-chain guard).

## Testing

Pester, same convention as fanout's tests, against an isolated psmux socket (`-L gridtest`); **no real LLM CLIs in CI**. A fake harness (plain PowerShell command simulating: prints-ready-prompt, self-update-then-exit, trust-prompt, never-ready) drives the contract tests:

- provision + CLS pass happens (panes exist, buffers cleared)
- `-Spec` string parsing ↔ `-Panes` structure equivalence
- registry validation: unknown harness, unknown quirk, bad regex, bad model/effort charset — all fail closed **before** provisioning
- readiness detection (pattern match)
- `self-update-exit` path: fake exits once → relaunched → `relaunched-ready`; exits twice → `launch-failed`
- timeout path: never-ready fake → `timeout`, other panes unaffected
- `pane-dead` detection
- swap: exit → confirm-shell → relaunch → manifest updated; swap aborts if pane never returns to shell
- teardown: snapshot-based reap kills the grid's fake-harness descendants and **spares a same-name process started after grid-up outside the grid** (the review's counterexample, as a test)
- teardown revalidation: a snapshot entry whose PID now belongs to a different process (simulated by CreationDate mismatch) is skipped as `pid-reused-skipped`
- griddown idempotency: re-running after a partial teardown completes cleanly
- targeting: with two sessions each owning a `%1`, operations resolve to THIS grid's pane (duplicate-`%id` collision test)
- stale-manifest detection: a leftover `connect-manifest.json` whose pane ids don't exist live → hard failure before any launch
- failure cleanup: induced provision failure tears down the session unless `-KeepPartial`; induced mid-launch death leaves an early `State='building'` manifest that griddown can consume

One opt-in live smoke test (`-Tag Live`) launches real harnesses for manual verification (and captures pi's readyPattern if pi is present).

## Out of scope (explicit)

- Persona briefing, broadcast Q&A, tagged answer extraction (future skill or connector feature).
- Headless/one-shot modes, assignments, grid recipes (remain in psmux-fanout).
- Injection of work into running harnesses (remains psmux-connector).
- Deleting or deprecating fanout/connector (revisit after this skill proves out).
- Transactional lifecycle state machines, interrupt recovery, concurrency locking (single-operator tool; `-FreshStart` is the recovery story).
- Fixing the noted fanout cleanup papercuts (separate follow-up).
