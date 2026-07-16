# psmux-panel skill — design

**Date:** 2026-07-14 (converged after a 7-round Claude ↔ Codex (gpt-5.6-sol) review loop; final Codex verdict: READY)
**Status:** approved for planning
**Origin:** the broadcast/Q&A layer hand-rolled three times during 2026-07-13/14 (persona panel, multi-model Q&A, engineering consult) — deliberately excluded from psmux-harness-grid's scope, now specified as its own sibling skill.

## Purpose

The conversation-orchestration layer on top of a psmux-harness-grid grid: brief each pane's harness (role/persona + protocol), broadcast tagged questions, collect every pane's answer robustly, and produce a durable transcript. Two entry points: `panel-brief.ps1` (role briefing + READY handshake) and `panel-ask.ps1` (one blocking Q&A round). Relay mode (answer-becomes-prompt chains) is deferred to a follow-up increment.

## Field failure modes this skill fixes (observed 2026-07-13/14)

1. agy's TUI renders `<<ANSWER>>`/`<<END>>` tags as `<>` — tag-scraping needs per-harness fallbacks.
2. codex (gpt-5.6-sol ultra) answers can take 7+ minutes, exceeding guessed poll deadlines.
3. Long answers scroll out of the default `capture-pane` window.
4. `capture-pane` flattens TUI spacing — prose extraction from captures is unreliable.
5. The poll/extract loop was re-improvised in inline PowerShell every session.

## Core decisions (user-ratified)

1. **Dual-channel collection.** The protocol instructs each agent to answer in-pane (the watchable show) AND write the same answer to a per-round file. Files are the sole source of truth the scripts read; pane text is best-effort display, never parsed for content and never diffed against the file.
2. **Scope v1: brief + ask.** Personas are just briefing text (optional reusable templates in `briefings\`). Relay is out of scope.
3. **Blocking with per-pane deadlines.** `panel-ask` blocks until every dispatched pane is terminal; each pane's deadline is its own (`DeadlineAt = DispatchedAt + PaneTimeoutSec`, per-harness overridable, e.g. `-PaneTimeout @{codex=900}`).
4. **Name:** psmux-panel (family convention).

## Relationship to existing skills

Sibling, compositional, read-only on both seams:
- **psmux-harness-grid** — `grid-manifest.json` supplies pane identity (pane ids, harnesses, statuses). Panel validates it (SchemaVersion, `State='up'`, session match, well-formed 10-field records), cross-checks `BuildId` and pane-id liveness against the live session, and rejects panes with `NeedsSetup=true` (v1 does not automate session-purpose steps).
- **psmux-connector** — remains the generic injection layer for arbitrary work; panel does NOT dot-source its `_drive-lib.ps1` (bare-`%id` targeting) nor `_inject-lib.ps1` directly (it depends on connector's gate lib and does not re-resolve targets between operations). Panel has its own **injection adapter**: harness-grid-style fresh `%id → index` resolution before EVERY target-sensitive operation, connector-style length-scaled settle, separate lone Enter.
- **psmux-fanout** — precedent only (result-file + `.done` sentinel pattern).

## File layout

```
~\.claude\skills\psmux-panel\
├── SKILL.md
├── panel-brief.ps1     # role briefing + protocol preamble + READY handshake
├── panel-ask.ps1       # one blocking round: claim → dispatch → sweep → transcript
├── _panel-lib.ps1      # manifest validation, injection adapter, state store, sweep
├── briefings\          # optional reusable briefing templates (plain text)
└── tests\Panel.Tests.ps1 (+ Live.Tests.ps1, -Tag Live)
```

**Work area:** `%TEMP%\psmux-panel\<session>\` — `panel-state.json`, `transcript.md`, `lock`, per-round directories `r<N>\` containing `pane<Index>.txt` (+ `.done` sentinels) and READY files. Session names are validated `^[A-Za-z0-9_-]+$` (deliberately tighter than psmux allows) BEFORE any path construction; all paths are built exclusively from validated components — no user-supplied path fragments.

## Call shapes

```powershell
& panel-brief.ps1 -Session fanout -Briefing @{ '%1'='You are MARCUS...'; '*'='(common rules)' }
# or -BriefingPath <folder>; returns {PaneId, Harness, Ready} per pane

$r = & panel-ask.ps1 -Session fanout -Question "..." [-TimeoutSec 480] [-PaneTimeout @{codex=900}] [-Targets @('%1','%5')]
# returns {Round, PaneId, Harness, Answer, Status, LatencySec, DispatchedAt}
```

Note `-Targets` values must be quoted (`@('%1','%5')` — bare `%` breaks PowerShell parsing). Duplicate explicit `-Targets` entries are REJECTED with a clear error pre-claim (fail-closed; dedup would silently mask a caller bug).

## Protocol

**panel-brief:** per-pane role text (`'*'` entry prepended to all). The script appends the protocol preamble: answer-file convention, refusal convention (first line `REFUSED:`), and the READY instruction. Each invocation generates a random `BriefingRunId` (GUID); READY files are `ready-<BriefingRunId>-pane<Index>.txt` and the invocation waits ONLY for its own run-id — identical-text re-briefs never collide, pane-index reuse cannot resurrect an old READY. The full briefing SHA256 fingerprint is recorded in panel-state as diagnostics only; re-briefing is a feature and is never gated. Blocks on READY files with per-pane deadlines; returns per-pane `{PaneId, Harness, Ready}`.

**panel-ask, round N:**
1. **Claim:** atomically write the round record with the FULL intended set — explicit `-Targets` verbatim (unavailable member = hard error pre-claim), or default = ALL manifest panes including unavailable ones — every member `Status='intended'`. Record `{Round, Question, AskedAt, CompletedAt: $null, Panes:[...]}`.
2. **Skip pass:** immediately after the claim (before any injection), unavailable default-set members transition `intended → skipped` atomically.
3. **Dispatch loop** (sequential, per remaining `intended` pane): atomically set `Status='dispatching'` → inject via the adapter (`[Q<N>] <question>` + the **per-round envelope** with fully-resolved absolute paths: "write your complete answer to `<work>\r<N>\pane<Index>.txt`, then create `<same>.done`; also show your answer in the pane") → atomically set `Status='dispatched'`, `DispatchedAt`, `DeadlineAt` (or `Status='dispatch-error'` on throw, continuing with the remaining panes). The envelope repeats the file contract EVERY round — no reliance on agents remembering a template from the briefing (context compaction is real).
4. **Collection sweep** (~5s cadence): one `SweepAt` timestamp and ONE observation set per sweep (a `.done` scan for all pending panes + one live id→index resolution — "snapshot" means captured observations, not an atomic filesystem snapshot). Classify each pending pane against that same observation set, first match wins:
   (a) `.done` exists → read answer: first line `REFUSED:` → `refused`; readable non-empty → `answered`; else defer with a `GraceUntil` marker and re-judge from the NEXT sweep's observations (never an inline sleep — one pane's grace cannot delay others) → still invalid → `protocol-error`.
   (b) pane absent from live resolution → `pane-dead`.
   (c) `SweepAt > DeadlineAt` → `timeout`.
   A `.done` observed on the deadline sweep wins ((a) before (c)). Statuses are immutable once assigned; later `.done` files are ignored by classification and noted in the transcript. Late round-N stragglers land in `r<N>\` and cannot contaminate round N+1.
5. **Complete:** `CompletedAt` is set only when BOTH the dispatch iteration finished for the whole intended set AND every intended pane — including `skipped` and `dispatch-error` — has a durable terminal record. Append the round (question, per-pane status/answer/latency) to `transcript.md`.

**Crash honesty:** every panel-state mutation uses atomic temp-write + `Move-Item` (single writer under the held lock). A crash leaves durable, honest state: `intended` = not attempted; `dispatching` = delivery unknown; `dispatched` = delivered, unclassified; `AskedAt` set with `CompletedAt=$null`. There is NO reclaim operation — a crashed round is never resumed or reinjected (reinjection could duplicate a delivered question); its record remains inspectable and the next `panel-ask` claims round N+1.

## State schema (consolidated)

| State | Terminal? | Delivery certainty | DispatchedAt | LatencySec |
|---|---|---|---|---|
| `intended` | no | not attempted | `$null` | `$null` |
| `dispatching` | no | **unknown** | `$null` | `$null` |
| `dispatched` | no | delivered — operationally: the injection adapter completed successfully (does not imply the agent consumed the prompt) | set | `$null` |
| `answered` | yes | delivered | set | set (DispatchedAt → classifying SweepAt) |
| `refused` | yes | delivered | set | set |
| `timeout` | yes | delivered | set | `$null` |
| `protocol-error` | yes | delivered | set | `$null` |
| `pane-dead` | yes | delivered | set | `$null` |
| `skipped` | yes | not attempted | `$null` | `$null` |
| `dispatch-error` | yes (for orchestration) | **unknown** — an exception in the two-step text→Enter sequence may fire after the prompt landed; the transcript states "delivery unknown" | `$null` | `$null` |

## Lock

One lock per session work area, held for the ENTIRE invocation: `[System.IO.File]::Open(path, CreateNew, ReadWrite, FileShare.None)`, disposed in `finally`. Identity metadata (PID, CreationDate) is written and flushed through the held stream as diagnostics — acquisition logic never reads it, so partial metadata after a crash is harmless (and unobservable by other openers while the owner holds the handle). Liveness is structural: while the owner lives, the file can be neither opened nor deleted; a dead owner releases the handle by definition.

- Collision detection: catch `IOException` and require `($_.Exception.HResult -band 0xFFFF) -in (80, 183)` (ERROR_FILE_EXISTS / ERROR_ALREADY_EXISTS); any other HResult or exception type = fail closed, no staleness probe.
- Staleness probe: attempt delete. Success → ONE `CreateNew` re-attempt (concurrent stealers are serialized by `CreateNew`; the loser errors loudly — no retry loop). ANY delete failure = "lock unavailable, fail closed" — deliberately NOT interpreted as proof the owner is alive (permissions/filters/read-only also fail deletes).

## Testing

Family conventions: Pester 5.7.1 with `-ExcludeTag Live`, fake node harnesses (the fake reads the injected envelope and writes the real answer file — proving the actual file contract), default-server skip-guard, no real LLMs outside `-Tag Live`.

- Preflights fail closed: manifest schema/State/session, BuildId + liveness cross-check, session-name charset, duplicate `-Targets`, `NeedsSetup` rejection
- Envelope path resolution; sentinel-gated reads (answer never read before `.done`)
- Classification: `REFUSED:` → refused (never answered); malformed/empty → `protocol-error` after deferred grace; `.done`-beats-deadline same-sweep; `.done > pane-dead` and `pane-dead > timeout` precedence; multi-pane sweep with one pane in grace (others unaffected, mock-driven); per-pane deadline independence
- Null-field assertions per the state table (`dispatch-error`/`skipped` → both `$null`)
- Dispatch: injection made to throw → `dispatch-error`, remaining panes still dispatched; mixed round (answered + skipped + dispatch-error) proves `CompletedAt` gating over the full intended set
- Crash persistence: simulated abort after claim → durable full intended set, `AskedAt` set, `CompletedAt=$null`; abort after injection but before post-injection persistence → durable `dispatching` (delivery-unknown)
- READY: stale ready-file with a different `BriefingRunId` (including identical briefing text) never satisfies a new invocation
- Lock: second `Open(CreateNew)` while held fails as collision; delete-while-held fails (ABA impossible); steal succeeds only after owner exit; path-style `IOException` (e.g. directory-not-found) fails closed without probing; read-only lock file → delete failure → "lock unavailable", not owner-alive
- Rounds: late round-N write cannot satisfy round N+1
- Opt-in `-Tag Live`: one brief + one ask against real claude+omp; transcript verified

## Out of scope (explicit)

- Relay mode (answer-becomes-prompt chains) — follow-up increment on this foundation.
- Automating `NeedsSetup` panes (pi's session-purpose step).
- Concurrent panels on one session beyond the lockfile's loud error; connector-grade runspace parallelism; any database.
- Harness-specific pane-text scraping (files are the only read channel).
- Reclaim/resume of crashed rounds.

## Review provenance

Rounds 1-7 with Codex gpt-5.6-sol (2026-07-14): round 1 reshaped the collection contract (per-round envelope, expanded statuses, injection adapter); rounds 2-3 hardened rounds/locking/READY identity; round 4 replaced content-based lock staleness with the held-handle design (Codex confirmed it eliminates, not relocates, the ABA/partial-metadata races); rounds 5-6 made crash states honest and complete; round 7 resolved `dispatch-error` delivery-unknown semantics. Final verdict: READY. Rejected along the way (with Codex's eventual concurrence): formal round lifecycle state machine, re-brief gating, connector-grade locking.
