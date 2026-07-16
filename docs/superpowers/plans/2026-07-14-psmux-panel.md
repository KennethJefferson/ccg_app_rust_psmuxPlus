# psmux-panel Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The conversation layer for psmux grids: `panel-brief.ps1` (role briefing + READY handshake) and `panel-ask.ps1` (one blocking, crash-honest Q&A round with dual-channel file collection) — per the approved spec `K:\Downloads\__Projects.Mine\psmuxPlus\docs\superpowers\specs\2026-07-14-psmux-panel-skill-design.md` (converged through a 7-round Codex review loop).

**Architecture:** New sibling skill at `~\.claude\skills\psmux-panel\`, composing psmux-harness-grid read-only (its `grid-manifest.json` for pane identity; its `_target-lib.ps1`/`_grid-lib.ps1` primitives dot-sourced). Files (not pane captures) are the only content channel; every panel-state mutation is atomic; the invocation holds a `FileShare.None` lock handle end-to-end.

**Tech Stack:** PowerShell 7.1+, psmux 3.3.x (Windows/ConPTY), Pester 5.7.1, CIM, node (test fakes only).

## Global Constraints

- `#requires -Version 7.1` in every `.ps1`.
- Skill dir is its own git repo (family convention); commits happen inside `~\.claude\skills\psmux-panel\`.
- Sibling resolution: `$GridDir = Join-Path (Split-Path $PSScriptRoot -Parent) 'psmux-harness-grid'` — no hardcoded user paths.
- Session names must match `^[A-Za-z0-9_-]+$` BEFORE any path construction; all work paths built only from validated components under `%TEMP%\psmux-panel\<session>\`.
- Pane states (exact): non-terminal `intended | dispatching | dispatched`; terminal `answered | timeout | refused | protocol-error | pane-dead | skipped | dispatch-error`. Null-field schema per the spec's state table (`DispatchedAt`/`LatencySec`).
- Delivery certainty: `dispatching` and `dispatch-error` are delivery-UNKNOWN; `dispatched` means "the injection adapter completed successfully" only.
- Every panel-state write is atomic (temp file + `Move-Item -Force`); single writer under the held lock.
- Lock: `[System.IO.File]::Open(path, CreateNew, ReadWrite, FileShare.None)` held for the whole invocation, disposed in `finally`; collision = `IOException` with `(HResult -band 0xFFFF) -in (80,183)`; any delete failure = fail closed; ONE steal attempt, no retry loop.
- Classification precedence per sweep, one observation set + one `SweepAt`: `.done` (→ refused/answered/grace→protocol-error) before pane-dead before deadline; statuses immutable once set.
- Per-round envelope repeats fully-resolved answer/sentinel paths every round; READY files are `ready-<BriefingRunId>-pane<Index>.txt` (GUID per brief invocation).
- Duplicate or unavailable explicit `-Targets` = hard error pre-claim; `NeedsSetup=true` panes rejected (explicit) / pre-skipped (default set).
- Tests: Pester 5.7.1 (`Import-Module Pester -RequiredVersion 5.7.1 -Force`), run with `-ExcludeTag Live`; integration tests use the default psmux server ONLY when it has zero sessions; `$env:PSMUX_ALLOW_ELEVATED='1'`; no real LLM CLIs outside `-Tag Live`.
- send-keys submit rule: literal text (`send-keys -l`), settle pause, then a SEPARATE lone `Enter`, with fresh `%id → index` resolution before every target-sensitive psmux op.

**Consumed sibling interfaces (verbatim, from shipped psmux-harness-grid):**
- `_target-lib.ps1`: `Invoke-Psmux -Psmux <exe> -ArgList <string[]>` (throws on psmux failure) · `Get-GridPaneInfo -Psmux -Session -> hashtable(%id -> @{Index;PanePid;CurrentCommand})` · `Resolve-PaneTarget -Psmux -Session -PaneId -> '<session>:agents.<idx>' | $null` · `Send-PaneKey -Psmux -Session -PaneId -Key`
- `_grid-lib.ps1`: `Get-GridManifestPath -Session -> %TEMP%\psmux-fanout\<session>\grid-manifest.json` · `Read-GridManifest -Path -> object` · `Write-GridManifest -Path -State -Session -BuildId -Panes` (tests use it to fabricate manifests)

---

### Task 1: Scaffold, state store, lock (`_panel-lib.ps1` part 1)

**Files:**
- Create: `~\.claude\skills\psmux-panel\_panel-lib.ps1`
- Test: `~\.claude\skills\psmux-panel\tests\State.Tests.ps1`

**Interfaces:**
- Consumes: nothing (pure + filesystem).
- Produces: `Test-PanelSessionName([string]$Session)` (throws on bad charset). `Get-PanelWorkDir([string]$Session) -> string`. `Read-PanelState([string]$WorkDir) -> object|$null`. `Write-PanelState([string]$WorkDir, [object]$State)` (atomic). `New-PanelState([string]$Session, [string]$BuildId) -> object` (`{SchemaVersion=1; Session; BuildId; BriefingFingerprints=@{}; Rounds=@()}`). `Enter-PanelLock([string]$WorkDir) -> System.IO.FileStream`. `Exit-PanelLock([System.IO.FileStream]$Lock)`.

- [ ] **Step 1: git init + write failing tests**

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills\psmux-panel\tests" | Out-Null
git -C "$env:USERPROFILE\.claude\skills\psmux-panel" init
```

`tests\State.Tests.ps1`:

```powershell
#requires -Version 7.1
BeforeAll {
    $script:SkillDir = Split-Path $PSScriptRoot -Parent
    . (Join-Path $script:SkillDir '_panel-lib.ps1')
    $script:Wd = Join-Path $env:TEMP ("ptest_{0}" -f ([guid]::NewGuid().ToString('n').Substring(0,8)))
}
AfterAll { Remove-Item -Recurse -Force $script:Wd -ErrorAction SilentlyContinue }

Describe 'Test-PanelSessionName' {
    It 'accepts safe names'  { { Test-PanelSessionName 'fanout_2-a' } | Should -Not -Throw }
    It 'rejects path tricks' {
        foreach ($bad in 'a.b','a:b','..','a b','a\b','a/b','') {
            { Test-PanelSessionName $bad } | Should -Throw '*session name*'
        }
    }
}
Describe 'panel state store' {
    It 'round-trips atomically' {
        $s = New-PanelState -Session 's1' -BuildId 'b-1'
        Write-PanelState -WorkDir $script:Wd -State $s
        $r = Read-PanelState -WorkDir $script:Wd
        $r.SchemaVersion | Should -Be 1
        $r.BuildId | Should -Be 'b-1'
        @($r.Rounds).Count | Should -Be 0
        (Get-ChildItem $script:Wd -Filter '*.tmp').Count | Should -Be 0
    }
    It 'returns null when absent' {
        Read-PanelState -WorkDir (Join-Path $env:TEMP 'ptest_nope') | Should -BeNullOrEmpty
    }
}
Describe 'panel lock' {
    It 'holds exclusively: second CreateNew collides, delete-while-held fails' {
        $lock = Enter-PanelLock -WorkDir $script:Wd
        try {
            { Enter-PanelLock -WorkDir $script:Wd } | Should -Throw '*held by another*'
            { [System.IO.File]::Delete((Join-Path $script:Wd 'lock')) } | Should -Throw
        } finally { Exit-PanelLock -Lock $lock }
    }
    It 'steals a stale (unheld) lock exactly once' {
        Set-Content -LiteralPath (Join-Path $script:Wd 'lock') -Value 'stale'   # unheld file = stale
        $lock = Enter-PanelLock -WorkDir $script:Wd
        try { $lock | Should -Not -BeNullOrEmpty } finally { Exit-PanelLock -Lock $lock }
    }
    It 'fails closed on non-collision IO errors' {
        { Enter-PanelLock -WorkDir (Join-Path $script:Wd 'no\such\<>dir') } | Should -Throw '*fail*'
    }
    It 'fails closed when the stale delete fails (read-only file)' {
        $p = Join-Path $script:Wd 'lock'
        Set-Content -LiteralPath $p -Value 'stale'
        Set-ItemProperty -LiteralPath $p -Name IsReadOnly -Value $true
        try { { Enter-PanelLock -WorkDir $script:Wd } | Should -Throw '*unavailable*' }
        finally { Set-ItemProperty -LiteralPath $p -Name IsReadOnly -Value $false; Remove-Item $p -Force }
    }
}
```

- [ ] **Step 2: Run to verify failure** — `pwsh -NoProfile -Command "Import-Module Pester -RequiredVersion 5.7.1 -Force; Invoke-Pester '$env:USERPROFILE\.claude\skills\psmux-panel\tests\State.Tests.ps1' -Output Detailed"` → FAIL (`_panel-lib.ps1` missing).

- [ ] **Step 3: Implement (`_panel-lib.ps1`)**

```powershell
#requires -Version 7.1
# psmux-panel core: session/path validation, atomic state store, held-handle lock.

$script:GridDir = Join-Path (Split-Path $PSScriptRoot -Parent) 'psmux-harness-grid'

function Test-PanelSessionName {
    [CmdletBinding()]
    param([AllowEmptyString()][string] $Session)
    if ($Session -notmatch '^[A-Za-z0-9_-]+$') {
        throw "invalid session name '$Session' (allowed: ^[A-Za-z0-9_-]+$)"
    }
}

function Get-PanelWorkDir {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string] $Session)
    Test-PanelSessionName $Session
    Join-Path (Join-Path $env:TEMP 'psmux-panel') $Session
}

function New-PanelState {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Session, [Parameter(Mandatory)][string]$BuildId)
    [pscustomobject]@{ SchemaVersion = 1; Session = $Session; BuildId = $BuildId
                       BriefingFingerprints = @{}; Rounds = @() }
}

function Read-PanelState {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string] $WorkDir)
    $p = Join-Path $WorkDir 'panel-state.json'
    if (-not (Test-Path -LiteralPath $p)) { return $null }
    Get-Content -LiteralPath $p -Raw | ConvertFrom-Json
}

function Write-PanelState {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$WorkDir, [Parameter(Mandatory)][object]$State)
    New-Item -ItemType Directory -Force -Path $WorkDir | Out-Null
    $p = Join-Path $WorkDir 'panel-state.json'
    $tmp = "$p.tmp"
    $State | ConvertTo-Json -Depth 8 | Set-Content -LiteralPath $tmp -Encoding utf8
    Move-Item -LiteralPath $tmp -Destination $p -Force
}

function Enter-PanelLock {
    # Held-handle lock (spec "Lock"): correctness is structural (FileShare.None), metadata is
    # diagnostics only. Collision = ERROR_FILE_EXISTS/ALREADY_EXISTS; anything else fails closed.
    # Staleness probe = one delete + one CreateNew re-attempt; any delete failure = unavailable.
    [CmdletBinding()]
    param([Parameter(Mandatory)][string] $WorkDir)
    New-Item -ItemType Directory -Force -Path $WorkDir | Out-Null
    $path = Join-Path $WorkDir 'lock'
    for ($attempt = 1; $attempt -le 2; $attempt++) {
        try {
            $fs = [System.IO.File]::Open($path, [System.IO.FileMode]::CreateNew,
                    [System.IO.FileAccess]::ReadWrite, [System.IO.FileShare]::None)
            $cim = Get-CimInstance Win32_Process -Filter "ProcessId=$PID" -ErrorAction SilentlyContinue
            $meta = [System.Text.Encoding]::UTF8.GetBytes("$PID $("$($cim.CreationDate)")")
            $fs.Write($meta, 0, $meta.Length); $fs.Flush()
            return $fs
        } catch [System.IO.IOException] {
            $code = $_.Exception.HResult -band 0xFFFF
            if ($code -notin 80, 183) { throw "panel lock open failed (non-collision, fail closed): $($_.Exception.Message)" }
            if ($attempt -ge 2) { throw "panel lock for '$WorkDir' is held by another invocation" }
            try { [System.IO.File]::Delete($path) }
            catch { throw "panel lock unavailable (stale delete failed; NOT proof of a live owner): $($_.Exception.Message)" }
        } catch {
            throw "panel lock open failed (non-collision, fail closed): $($_.Exception.Message)"
        }
    }
}

function Exit-PanelLock {
    [CmdletBinding()]
    param([Parameter(Mandatory)][System.IO.FileStream] $Lock)
    $path = $Lock.Name
    $Lock.Dispose()
    Remove-Item -LiteralPath $path -Force -ErrorAction SilentlyContinue
}
```

- [ ] **Step 4: Run to verify pass** (same command) → PASS (9 tests).
- [ ] **Step 5: Commit** — `git -C "$HOME/.claude/skills/psmux-panel" add -A && git commit -m "feat: panel state store and held-handle lock"`

---

### Task 2: Preflight + intended-set resolution (`_panel-lib.ps1` part 2)

**Files:**
- Modify: `~\.claude\skills\psmux-panel\_panel-lib.ps1` (append)
- Test: `~\.claude\skills\psmux-panel\tests\Preflight.Tests.ps1`

**Interfaces:**
- Consumes: Task 1; harness-grid `_grid-lib.ps1` (`Get-GridManifestPath`, `Read-GridManifest`) — dot-source via `$script:GridDir`.
- Produces: `Test-PanelManifest([object]$Manifest, [string[]]$LivePaneIds, [string]$Session)` (throws on: wrong SchemaVersion, `State -ne 'up'`, session mismatch, empty/missing Panes, missing PaneId/PanePid/Harness/Status fields, any manifest pane id not in `$LivePaneIds`, empty BuildId). `Resolve-PanelTargets([object]$Manifest, [string[]]$Targets) -> @{ Intended = <pane records>; PreSkipped = <pane records> }` — availability = `Status -in 'ready','relaunched-ready'` AND `-not NeedsSetup`. Explicit `-Targets`: unknown id, duplicate id, or unavailable member = throw. Default (no targets): ALL manifest panes; unavailable → `PreSkipped`.

- [ ] **Step 1: Write failing tests** — `tests\Preflight.Tests.ps1`, pure (fabricated manifest objects, injected live-id arrays, no psmux):

```powershell
#requires -Version 7.1
BeforeAll {
    $script:SkillDir = Split-Path $PSScriptRoot -Parent
    . (Join-Path $script:SkillDir '_panel-lib.ps1')
    function New-FakeManifest {
        param([object[]]$Panes, [string]$State = 'up', [string]$Session = 's1', [string]$BuildId = 'b-1')
        [pscustomobject]@{ SchemaVersion=1; State=$State; Session=$Session; BuildId=$BuildId
                           Timestamp=(Get-Date).ToString('o'); Panes=@($Panes) }
    }
    function New-FakePane {
        param([int]$Index, [string]$Id, [string]$Harness='fake', [string]$Status='ready', [bool]$NeedsSetup=$false)
        [pscustomobject]@{ Index=$Index; PaneId=$Id; PanePid=1000+$Index; PanePidCreated='20260714'
                           Harness=$Harness; Model=$null; Effort=$null; Status=$Status
                           Title="$Harness-default"; NeedsSetup=$NeedsSetup }
    }
}
Describe 'Test-PanelManifest' {
    It 'passes a healthy manifest' {
        $m = New-FakeManifest -Panes @((New-FakePane 0 '%1'), (New-FakePane 1 '%4'))
        { Test-PanelManifest -Manifest $m -LivePaneIds @('%1','%4') -Session 's1' } | Should -Not -Throw
    }
    It 'rejects State building' {
        $m = New-FakeManifest -Panes @((New-FakePane 0 '%1')) -State 'building'
        { Test-PanelManifest -Manifest $m -LivePaneIds @('%1') -Session 's1' } | Should -Throw '*State*'
    }
    It 'rejects session mismatch' {
        $m = New-FakeManifest -Panes @((New-FakePane 0 '%1'))
        { Test-PanelManifest -Manifest $m -LivePaneIds @('%1') -Session 'other' } | Should -Throw '*session*'
    }
    It 'rejects a dead pane id' {
        $m = New-FakeManifest -Panes @((New-FakePane 0 '%1'), (New-FakePane 1 '%4'))
        { Test-PanelManifest -Manifest $m -LivePaneIds @('%1') -Session 's1' } | Should -Throw '*not alive*'
    }
    It 'rejects empty BuildId and malformed panes' {
        $m = New-FakeManifest -Panes @((New-FakePane 0 '%1')) -BuildId ''
        { Test-PanelManifest -Manifest $m -LivePaneIds @('%1') -Session 's1' } | Should -Throw '*BuildId*'
        $bad = New-FakeManifest -Panes @([pscustomobject]@{ Index=0 })
        { Test-PanelManifest -Manifest $bad -LivePaneIds @() -Session 's1' } | Should -Throw '*malformed*'
    }
}
Describe 'Resolve-PanelTargets' {
    BeforeAll {
        $script:M = New-FakeManifest -Panes @(
            (New-FakePane 0 '%1'), (New-FakePane 1 '%4' -Status 'timeout'),
            (New-FakePane 2 '%5' -NeedsSetup $true), (New-FakePane 3 '%6' -Status 'relaunched-ready'))
    }
    It 'default set: all panes, unavailable pre-skipped' {
        $r = Resolve-PanelTargets -Manifest $script:M -Targets @()
        @($r.Intended).PaneId   | Should -Be @('%1','%6')
        @($r.PreSkipped).PaneId | Should -Be @('%4','%5')
    }
    It 'explicit targets pass when available' {
        (Resolve-PanelTargets -Manifest $script:M -Targets @('%1','%6')).Intended.Count | Should -Be 2
    }
    It 'explicit unavailable = hard error' {
        { Resolve-PanelTargets -Manifest $script:M -Targets @('%4') } | Should -Throw '*unavailable*'
        { Resolve-PanelTargets -Manifest $script:M -Targets @('%5') } | Should -Throw '*unavailable*'
    }
    It 'explicit unknown and duplicates = hard error' {
        { Resolve-PanelTargets -Manifest $script:M -Targets @('%99') } | Should -Throw '*unknown*'
        { Resolve-PanelTargets -Manifest $script:M -Targets @('%1','%1') } | Should -Throw '*duplicate*'
    }
}
```

- [ ] **Step 2: Run to verify failure** → FAIL (functions missing).
- [ ] **Step 3: Implement** (append to `_panel-lib.ps1`):

```powershell
if (-not (Get-Command Read-GridManifest -ErrorAction SilentlyContinue)) {
    . (Join-Path $script:GridDir '_grid-lib.ps1')
}

function Test-PanelManifest {
    [CmdletBinding()]
    param([Parameter(Mandatory)][object]$Manifest,
          [Parameter(Mandatory)][AllowEmptyCollection()][string[]]$LivePaneIds,
          [Parameter(Mandatory)][string]$Session)
    if ($Manifest.SchemaVersion -ne 1) { throw "grid-manifest SchemaVersion must be 1" }
    if ($Manifest.State -ne 'up') { throw "grid-manifest State is '$($Manifest.State)', need 'up' (run gridup first)" }
    if ($Manifest.Session -ne $Session) { throw "grid-manifest session '$($Manifest.Session)' does not match '$Session'" }
    if ([string]::IsNullOrWhiteSpace($Manifest.BuildId)) { throw "grid-manifest BuildId is empty" }
    if (-not @($Manifest.Panes)) { throw "grid-manifest is malformed: no Panes" }
    foreach ($p in @($Manifest.Panes)) {
        foreach ($f in 'PaneId','PanePid','Harness','Status') {
            if ($null -eq $p.PSObject.Properties[$f] -or [string]::IsNullOrWhiteSpace("$($p.$f)")) {
                throw "grid-manifest is malformed: pane record missing '$f'"
            }
        }
        if ($LivePaneIds -notcontains $p.PaneId) { throw "pane $($p.PaneId) not alive in session '$Session' (stale manifest?)" }
    }
}

function Resolve-PanelTargets {
    [CmdletBinding()]
    param([Parameter(Mandatory)][object]$Manifest,
          [AllowEmptyCollection()][string[]]$Targets = @())
    $isAvailable = { param($p) ($p.Status -in 'ready','relaunched-ready') -and -not $p.NeedsSetup }
    $all = @($Manifest.Panes)
    if ($Targets.Count -gt 0) {
        $dupes = @($Targets | Group-Object | Where-Object Count -gt 1)
        if ($dupes) { throw "duplicate -Targets entries: $($dupes.Name -join ', ')" }
        $intended = foreach ($t in $Targets) {
            $p = $all | Where-Object PaneId -eq $t
            if (-not $p) { throw "unknown target pane '$t' (not in grid manifest)" }
            if (-not (& $isAvailable $p)) { throw "target pane '$t' is unavailable (Status=$($p.Status), NeedsSetup=$($p.NeedsSetup))" }
            $p
        }
        return @{ Intended = @($intended); PreSkipped = @() }
    }
    @{ Intended   = @($all | Where-Object { & $isAvailable $_ })
       PreSkipped = @($all | Where-Object { -not (& $isAvailable $_) }) }
}
```

- [ ] **Step 4: Run to verify pass** → PASS (10 tests). Also re-run Task 1 suite (no regression).
- [ ] **Step 5: Commit** — `git add -A && git commit -m "feat: manifest preflight and intended-set resolution"`

---

### Task 3: Injection adapter + protocol text (`_panel-lib.ps1` part 3)

**Files:**
- Modify: `~\.claude\skills\psmux-panel\_panel-lib.ps1` (append)
- Test: `~\.claude\skills\psmux-panel\tests\Inject.Tests.ps1`

**Interfaces:**
- Consumes: harness-grid `_target-lib.ps1` (`Invoke-Psmux`, `Resolve-PaneTarget`, `Get-GridPaneInfo`) — dot-source via `$script:GridDir`.
- Produces: `Send-PanelPrompt([string]$Psmux, [string]$Session, [string]$PaneId, [string]$Text)` — fresh resolve → `send-keys -l` → settle `[Math]::Max(600, 200 + 8 * $Text.Length)` ms (connector-style length scaling) → fresh re-resolve → lone `Enter`; throws with 'pane-dead' text when resolution fails at either point. `Get-PanelPreamble([string]$ReadyPath) -> string` and `Get-RoundEnvelope([int]$Round, [string]$Question, [string]$AnswerPath) -> string` — single-line strings (newlines would submit early). Envelope wording is a STABLE contract (tests and fakes parse it): `... [Q<N>] <question> ... write your complete answer to <AnswerPath> then create the file <AnswerPath>.done ...`; preamble wording: `... to confirm you are ready create the file <ReadyPath> ... if you must decline a question, make the FIRST line of your answer file REFUSED: <reason> ...`.

- [ ] **Step 1: Write failing tests** — `tests\Inject.Tests.ps1`: pure text-builder tests (envelope contains `[Q3]`, the exact answer path, the `.done` path, no newlines; preamble contains the ready path, the `REFUSED:` convention, no newlines) + integration (default-server skip-guard copied from harness-grid `tests\Target.Tests.ps1`, `PSMUX_ALLOW_ELEVATED=1`): build a 2-pane raw session (`new-session -d -s itest -n agents`, one split, 1000ms settle), `Send-PanelPrompt` a marker line to pane 2, capture shows the marker; `Send-PanelPrompt` to id `%999` throws `*pane*`; duplicate-`%id` guard test (second session `other`, marker lands only in `itest`, same shape as Target.Tests' collision test). Kill own sessions in `AfterEach`.
- [ ] **Step 2: Run to verify failure** → FAIL.
- [ ] **Step 3: Implement** (append):

```powershell
if (-not (Get-Command Resolve-PaneTarget -ErrorAction SilentlyContinue)) {
    . (Join-Path $script:GridDir '_target-lib.ps1')
}

function Send-PanelPrompt {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Psmux, [Parameter(Mandatory)][string]$Session,
          [Parameter(Mandatory)][string]$PaneId, [Parameter(Mandatory)][string]$Text)
    $t = Resolve-PaneTarget -Psmux $Psmux -Session $Session -PaneId $PaneId
    if (-not $t) { throw "pane $PaneId not found in session '$Session' (pane-dead)" }
    Invoke-Psmux -Psmux $Psmux -ArgList @('send-keys','-t',$t,'-l',$Text) | Out-Null
    Start-Sleep -Milliseconds ([Math]::Max(600, 200 + 8 * $Text.Length))
    $t = Resolve-PaneTarget -Psmux $Psmux -Session $Session -PaneId $PaneId
    if (-not $t) { throw "pane $PaneId vanished before Enter (pane-dead)" }
    Invoke-Psmux -Psmux $Psmux -ArgList @('send-keys','-t',$t,'Enter') | Out-Null
}

function Get-PanelPreamble {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string] $ReadyPath)
    "PANEL PROTOCOL: for every message from the moderator tagged like [Q1], [Q2], ... you will be told a file path to write your answer to - write your COMPLETE answer to that exact file, then create the sentinel file named the same plus .done, and ALSO show your answer here in the pane. If you must decline a question, make the FIRST line of your answer file exactly: REFUSED: <short reason>. Do not use tools other than file writes for this protocol. To confirm you are ready now, create the file $ReadyPath (any content)."
}

function Get-RoundEnvelope {
    [CmdletBinding()]
    param([Parameter(Mandatory)][int]$Round, [Parameter(Mandatory)][string]$Question,
          [Parameter(Mandatory)][string]$AnswerPath)
    "[Q$Round] $Question -- Reply per the panel protocol: write your complete answer to $AnswerPath then create the file $AnswerPath.done and also show your answer in the pane."
}
```

- [ ] **Step 4: Run to verify pass**, full suite for regressions.
- [ ] **Step 5: Commit** — `git add -A && git commit -m "feat: injection adapter with length-scaled settle and protocol text builders"`

---

### Task 4: Sweep classification engine (`_panel-lib.ps1` part 4)

**Files:**
- Modify: `~\.claude\skills\psmux-panel\_panel-lib.ps1` (append)
- Test: `~\.claude\skills\psmux-panel\tests\Sweep.Tests.ps1`

**Interfaces:**
- Consumes: Task 3 (`Get-GridPaneInfo` for liveness).
- Produces: `Get-PaneClassification([object]$Pane, [hashtable]$Obs) -> string|$null` — PURE. `$Obs = @{ DoneExists=[bool]; AnswerPath=[string]; PaneAlive=[bool]; SweepAt=[datetime] }`. Returns a terminal status, or `$null` (still pending), mutating only `$Pane.GraceUntil`. Precedence: (a) `DoneExists` → read `$Obs.AnswerPath`: first line `^REFUSED:` → `'refused'`; readable non-empty → `'answered'`; else if `$Pane.GraceUntil` unset → set `GraceUntil = SweepAt.AddSeconds(2)`, return `$null`; else if `SweepAt -lt GraceUntil` → `$null`; else → `'protocol-error'`. (b) `-not PaneAlive` → `'pane-dead'`. (c) `SweepAt -gt $Pane.DeadlineAt` → `'timeout'`. Else `$null`.
  `Invoke-PanelSweep([string]$Psmux, [string]$Session, [object[]]$Panes, [string]$RoundDir, [scriptblock]$OnStateChange, [int]$PollMs = 5000)` — loops until every pane in `$Panes` (records with `Status='dispatched'`) is terminal. Per sweep: ONE `$SweepAt = Get-Date`, ONE `.done` directory scan (`Get-ChildItem $RoundDir -Filter '*.done'`), ONE `Get-GridPaneInfo` call; classify each pending pane against that same observation set; on a terminal classification set `Status`, `LatencySec` (`answered`/`refused` only: `(SweepAt - DispatchedAt).TotalSeconds` rounded to 1 decimal; `$null` otherwise), and invoke `$OnStateChange` (the caller persists atomically). Statuses immutable once set.

- [ ] **Step 1: Write failing tests** — `tests\Sweep.Tests.ps1`, almost entirely pure/mock-driven (no psmux, no skip-guard except one integration case):
  - `Get-PaneClassification` unit matrix: refused wins over answered (`REFUSED: nope` file → `refused`, never `answered`); `.done` with valid answer → `answered`; `.done` with empty file → `$null` first call (GraceUntil set), `$null` within grace, `protocol-error` after; `.done` present AND deadline expired same call → `answered` (precedence (a) before (c)); no `.done` + `PaneAlive=$false` + deadline expired → `pane-dead` (precedence (b) before (c)); no `.done` + alive + expired → `timeout`; no `.done` + alive + not expired → `$null`.
  - `Invoke-PanelSweep` mock-driven (Pester `Mock Get-GridPaneInfo`, real temp round dirs, `-PollMs 50`): two panes, pane A's answer+`.done` appear before pane B's → A classified `answered` on an earlier sweep, B later — and while pane A is in a malformed-grace deferral, pane B is still classified normally on that same sweep (grace defers only its own pane); a pane whose id disappears from the mocked `Get-GridPaneInfo` → `pane-dead`; late `.done` written after a pane was already `timeout` does NOT change its status (immutability).
- [ ] **Step 2: Run to verify failure** → FAIL.
- [ ] **Step 3: Implement** (append to `_panel-lib.ps1` — the exact logic specified in Produces; ~60 lines; classification first-match-wins ordering exactly (a),(b),(c); `Invoke-PanelSweep` computes `$doneSet = @{}` from one directory scan and `$live = Get-GridPaneInfo ...` once per sweep, wraps `Get-GridPaneInfo` in try/catch treating failure as "no liveness info this sweep" — panes are NOT declared pane-dead on a failed capture, they simply stay pending until their deadline).
- [ ] **Step 4: Run to verify pass**, full suite.
- [ ] **Step 5: Commit** — `git add -A && git commit -m "feat: snapshot-based sweep classification with deferred grace"`

---

### Task 5: `panel-brief.ps1`

**Files:**
- Create: `~\.claude\skills\psmux-panel\panel-brief.ps1`
- Create: `~\.claude\skills\psmux-panel\briefings\.gitkeep`
- Test: `~\.claude\skills\psmux-panel\tests\Brief.Tests.ps1`

**Interfaces:**
- Consumes: Tasks 1–3; harness-grid `Get-GridManifestPath`/`Read-GridManifest`; `Get-GridPaneInfo` (live ids).
- Produces: `panel-brief.ps1` params: `-Session` (mandatory), `-Briefing [hashtable]` XOR `-BriefingPath [string]` (folder: `common.txt` = the `'*'` entry, `pane<Index>.txt` or `<PaneId without %>.txt` per pane), `-TimeoutSec 120`, `-Targets [string[]]` (optional, same semantics as ask). Returns per-pane `[pscustomobject]@{ PaneId; Harness; Ready }`.

Flow: validate session name → work dir → `Enter-PanelLock` (all under `try/finally` with `Exit-PanelLock`) → read + validate grid manifest (`Test-PanelManifest` with live ids from `Get-GridPaneInfo`) → `Resolve-PanelTargets` → `BriefingRunId = [guid]::NewGuid().ToString('n')` → per pane: compose text = `'*'`/common + per-pane role + `Get-PanelPreamble -ReadyPath "<WorkDir>\ready-<BriefingRunId>-pane<Index>.txt"` (single line; join briefing fragments with spaces) → `Send-PanelPrompt` (a throw records `Ready=$false` and continues) → wait loop (5s sweeps, per-pane deadline `TimeoutSec` from its send): `Ready = Test-Path <its own run-id ready file>` → record `SHA256(text)` fingerprint in panel-state `BriefingFingerprints[PaneId]` (create state with manifest BuildId if absent; atomic write) → return records.

**Fake agent for tests** (the honest fake — a node REPL stand-in that actually implements the protocol; used here and in Task 6). Launched into a raw pane BEFORE briefing (test builds session + panes itself, then hand-writes a valid grid-manifest with `Write-GridManifest` naming those panes `Status='ready'`, plus fabricated `PanePid` from `Get-GridPaneInfo`):

```powershell
# typed into the pane to start the fake agent (single line):
node -e "const fs=require('fs');require('readline').createInterface({input:process.stdin}).on('line',l=>{let m=l.match(/create the file (\S+ready\S+?\.txt)/i);if(m)fs.writeFileSync(m[1],'1');m=l.match(/answer to (\S+?\.txt) then create/i);if(m){fs.writeFileSync(m[1],'the answer');fs.writeFileSync(m[1]+'.done','1');}});setInterval(()=>{},1e9)"
```

- [ ] **Step 1: Write failing tests** — `tests\Brief.Tests.ps1` (skip-guard + `PSMUX_ALLOW_ELEVATED`): (a) happy path: 2 fake-agent panes, `-Briefing @{ '*'='common'; '%<id>'='role' }` → both `Ready=$true`, ready files carry the run-id, fingerprints recorded; (b) invocation-uniqueness: plant a stale `ready-<otherguid>-pane0.txt` before briefing → NOT satisfied by it (new run-id awaited; with the fake agent responding it still passes, so assert the SATISFYING file's name contains THIS invocation's run-id — expose the run-id by returning it: add `BriefingRunId` to each returned record); (c) identical-text re-brief: brief twice with the same text → second invocation waits for its own new run-id file (assert two distinct ready files exist); (d) dead pane → `Ready=$false`, other panes unaffected; (e) lock released after run (a second brief immediately after succeeds).
  (Note: (b) requires `BriefingRunId` on the return record — include it in Produces.)
- [ ] **Step 2: Run to verify failure** → FAIL.
- [ ] **Step 3: Implement `panel-brief.ps1`** per the flow above (`#requires -Version 7.1`; `$ErrorActionPreference='Stop'`; dot-source `_panel-lib.ps1`; `$env:PSMUX_ALLOW_ELEVATED='1'`; `$psmux = (Get-Command psmux -ErrorAction Stop).Source`; return records include `BriefingRunId`).
- [ ] **Step 4: Run to verify pass**, full suite, zero leaked sessions.
- [ ] **Step 5: Commit** — `git add -A && git commit -m "feat: panel-brief with invocation-unique READY handshake"`

---

### Task 6: `panel-ask.ps1`

**Files:**
- Create: `~\.claude\skills\psmux-panel\panel-ask.ps1`
- Test: `~\.claude\skills\psmux-panel\tests\Ask.Tests.ps1`

**Interfaces:**
- Consumes: everything above.
- Produces: `panel-ask.ps1` params: `-Session` (mandatory), `-Question` (mandatory), `-TimeoutSec 480`, `-PaneTimeout [hashtable]` (harness name → seconds, overrides `-TimeoutSec` per pane), `-Targets [string[]]`. Returns per-pane `[pscustomobject]@{ Round; PaneId; Harness; Answer; Status; LatencySec; DispatchedAt }` (`Answer` = file content for `answered`, refusal text for `refused`, `$null` otherwise).

Flow (spec "Protocol", exactly):
1. Session name → work dir → lock (`try/finally`) → manifest read + `Test-PanelManifest` → `Resolve-PanelTargets`.
2. **Claim:** state = read-or-new; `Round = @($state.Rounds).Count + 1`; round record `{Round; Question; AskedAt=(Get-Date).ToString('o'); CompletedAt=$null; Panes=@()}` where Panes = FULL intended set (Intended + PreSkipped members) each `{PaneId; Harness; Status='intended'; DispatchedAt=$null; LatencySec=$null}`; append round to state; atomic write. Create `r<Round>\` dir.
3. **Skip pass:** every PreSkipped member → `Status='skipped'`; atomic write; `Write-Warning` naming them.
4. **Dispatch loop** (Intended members, sequential): set `Status='dispatching'` + atomic write → `$env = Get-RoundEnvelope -Round $Round -Question $Question -AnswerPath "<WorkDir>\r<Round>\pane<Index>.txt"` → `Send-PanelPrompt` → on success set `Status='dispatched'`, `DispatchedAt=(Get-Date)`, `DeadlineAt = DispatchedAt + (PaneTimeout[$Harness] ?? TimeoutSec)` + atomic write; on throw set `Status='dispatch-error'` (DispatchedAt stays `$null`) + atomic write, continue.
5. **Sweep:** `Invoke-PanelSweep` over the `dispatched` records; `$OnStateChange` = atomic state persist of the round record.
6. **Complete:** when dispatch finished AND every intended-set member terminal → `CompletedAt` + atomic write → append round to `transcript.md` (`## Round N — <AskedAt>`, question, per-pane `### <Harness> (<PaneId>) — <status> [delivery unknown]?`, answer text, latency; `dispatch-error` entries explicitly annotated `delivery unknown`) → return records.

- [ ] **Step 1: Write failing tests** — `tests\Ask.Tests.ps1` (skip-guard; fake agents from Task 5, plus variants):
  - fake-refuser: same node one-liner but writing `REFUSED: not my dept` as the answer content.
  - fake-malformed: writes ONLY the `.done` (no answer file): `m=l.match(/answer to (\S+?\.txt) then create/i);if(m)fs.writeFileSync(m[1]+'.done','1');`
  - fake-silent: reads lines, never writes (timeout path; drive with `-TimeoutSec 15`).
  Tests: (a) happy 2-pane round → both `answered`, `Answer` populated, `LatencySec` non-null, transcript has Round 1 with both answers, state `CompletedAt` set; (b) mixed round — answered + refuser + malformed + silent → statuses `answered/refused/protocol-error/timeout`, null-field schema assertions (`LatencySec` null for `protocol-error`/`timeout`), `CompletedAt` set only at the end (assert transcript written once, after all terminal); (c) explicit-target unavailable → throws pre-claim, state has NO new round; (d) default-set skip: manifest with one `timeout`-status pane → that pane `skipped` with `DispatchedAt=$null`, round completes; (e) dispatch-error: make injection throw for one pane by naming a manifest pane id that does not exist live — no wait, preflight would catch it... instead: `Mock Send-PanelPrompt { throw 'boom' } -ParameterFilter { $PaneId -eq $target }` in a unit-style invocation of the dispatch phase? panel-ask is a script, not mockable — so induce it live: kill the target pane AFTER preflight but BEFORE dispatch is impossible to time... USE THIS INSTEAD: fake agent that dies on first input? Still delivers. DECISION: test `dispatch-error` at the lib level — add a `Deploy-PanelRound` internal function? NO — simplest honest route: panel-ask gains an internal, test-only hook `-InjectOverride [scriptblock]` (hidden param, default `$null`, used in place of `Send-PanelPrompt` when set; documented as test-only in a comment). Test passes `-InjectOverride { param($Psmux,$Session,$PaneId,$Text) if ($PaneId -eq $victim) { throw 'boom' } else { Send-PanelPrompt @PSBoundParameters } }` → victim `dispatch-error` (DispatchedAt `$null`), others dispatched and answered, transcript marks victim `delivery unknown`;
  (f) crash honesty: run panel-ask with `-InjectOverride { throw 'hard' }` for ALL panes and wrap in try/catch — actually a full-throw doesn't crash the script (dispatch-error continues) → completes with all `dispatch-error`. For the true crash test: start panel-ask in a background `pwsh` process against fake-silent panes with a long timeout, `Stop-Process` it mid-sweep, then read panel-state → round has `AskedAt` set, `CompletedAt=$null`, panes `dispatched` (durable non-terminal), and the lock file is deletable (owner dead) — a follow-up `panel-ask` claims Round N+1 (crashed round untouched);
  (g) round isolation: after (f)'s crashed round, run a normal round and hand-write a stray `.done` into the crashed round's dir mid-sweep → new round unaffected.
- [ ] **Step 2: Run to verify failure** → FAIL.
- [ ] **Step 3: Implement `panel-ask.ps1`** per the flow (with the documented test-only `-InjectOverride` hook).
- [ ] **Step 4: Run to verify pass**, full suite, zero leaked sessions/fakes.
- [ ] **Step 5: Commit** — `git add -A && git commit -m "feat: panel-ask - crash-honest blocking Q&A round with dual-channel collection"`

---

### Task 7: `SKILL.md` + live smoke + memory pointer

**Files:**
- Create: `~\.claude\skills\psmux-panel\SKILL.md`
- Create: `~\.claude\skills\psmux-panel\tests\Live.Tests.ps1` (`-Tag Live`)
- Modify: `C:\Users\Tony Baloney\.claude\projects\K--Downloads---Projects-Mine-psmuxPlus\memory\psmux-harness-injection-gotchas.md` (controller does this — implementer skips)

**Interfaces:** none new — docs + verification.

- [ ] **Step 1: Write `SKILL.md`** — frontmatter `name: psmux-panel`; description: "Brief the harnesses in a psmux-harness-grid grid (roles/personas + protocol), broadcast tagged questions to all panes, and collect every answer robustly via per-round files with per-pane deadlines and a durable transcript. Use for multi-model Q&A panels, persona panels, and consults across a running grid." Sections: when-to-use vs fanout/connector/harness-grid (table); Quick start (gridup → panel-brief → panel-ask → transcript path → griddown); Parameters (both scripts, incl. `-PaneTimeout @{codex=900}` example and the `-Targets @('%1','%5')` quoting note); the consolidated state table from the spec (state × terminal × delivery-certainty × null fields) verbatim; Running the tests (`-ExcludeTag Live` REQUIRED — same wording pattern as harness-grid's section, incl. dual-Pester note); Gotchas (files are the only read channel — capture flattening/tag-mangling/scrollback are why; envelope repeats paths every round — context compaction is real; `dispatch-error` = delivery unknown; crashed rounds are never reclaimed; slow ultra-effort models need `-PaneTimeout`; the lock fails closed on any delete failure).
- [ ] **Step 2: Write `Live.Tests.ps1`** — `-Tag Live`, skip-guard, RAM note: gridup `-Spec 'claude:claude-haiku-4-5, omp' -Visible -FreshStart -Session panellive` → `panel-brief` with a one-line role → both Ready → `panel-ask -Question 'In one sentence, what model are you?' -TimeoutSec 240` → both `answered`, transcript contains both, then `griddown -Session panellive` clean. Run ONCE.
- [ ] **Step 3: Full suite** `-ExcludeTag Live` → all green; then the live smoke ONCE (needs ~2.5 GB free RAM and an empty default server).
- [ ] **Step 4: Commit** — `git add -A && git commit -m "docs: SKILL.md + opt-in live smoke"`

---

## Self-Review (completed)

1. **Spec coverage:** session/path validation + atomic state + held-handle lock incl. HResult discrimination and fail-closed deletes → T1; manifest preflight (SchemaVersion/State/session/BuildId/liveness/malformed), intended-set + duplicates/unavailable/NeedsSetup rules → T2; injection adapter (fresh resolution both ops, length-scaled settle, separate Enter) + stable envelope/preamble wording incl. `REFUSED:` convention → T3; snapshot sweep, precedence (a)>(b)>(c), deferred grace, immutability, latency computation → T4; BriefingRunId READY uniqueness, fingerprints-as-diagnostics, dead-pane tolerance → T5; claim-with-full-intended-set, skip pass, `intended→dispatching→dispatched/dispatch-error` transitions with atomic writes, `CompletedAt` gate, transcript with delivery-unknown annotation, crash honesty (background-process kill test), round isolation → T6; state table + docs + live smoke → T7. Relay, reclaim, `NeedsSetup` automation: out of scope per spec — no task touches them.
2. **Placeholder scan:** clean — Tasks 4–6 describe tests by exact scenario with all fakes' code given and assertions named; the one open implementation question found mid-draft (how to induce `dispatch-error` live) was resolved in-plan with the documented test-only `-InjectOverride` hook rather than left as an exercise.
3. **Type consistency:** pane-record fields (`PaneId, Harness, Status, DispatchedAt, LatencySec, DeadlineAt, GraceUntil`) and state names match across T2/T4/T6 and the Global Constraints enum; `Get-PaneClassification`/`Invoke-PanelSweep`/`Send-PanelPrompt` signatures consistent between Produces blocks and consuming tasks; return-record shapes for both scripts match the spec's call shapes.
