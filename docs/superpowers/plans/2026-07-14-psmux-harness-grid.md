# psmux-harness-grid Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A deterministic skill that stands up a visible psmux grid, launches a named agent harness (claude/codex/agy/omp) in each pane with model/effort, verifies readiness, and tears down precisely — per the approved spec `K:\Downloads\__Projects.Mine\psmuxPlus\docs\superpowers\specs\2026-07-14-psmux-harness-grid-skill-design.md` (rev 3).

**Architecture:** New sibling skill at `~\.claude\skills\psmux-harness-grid\` composing `psmux-fanout` (`fanout.ps1 -Provision` for panes, `cleanup.ps1 -NoReap` for session-kill, `Invoke-PsmuxWithRetry` for socket retries). Registry-driven launches (`harnesses.json`), pane identity by `%id` with per-operation fresh `id→pane_index` resolution, early `State='building'` grid-manifest, pane-PID-descendant teardown with pre-kill identity revalidation.

**Tech Stack:** PowerShell 7.1+, psmux 3.3.x (Windows/ConPTY), Pester 5, CIM (`Win32_Process`).

## Global Constraints

- `#requires -Version 7.1` in every `.ps1`.
- The skill dir is its own git repo (same convention as psmux-fanout / psmux-connector). All commits happen inside `~\.claude\skills\psmux-harness-grid\`.
- Fanout skill location resolved as `$FanoutDir = Join-Path (Split-Path $PSScriptRoot -Parent) 'psmux-fanout'` — sibling directory, no hardcoded user paths.
- Model/Effort values must match `^[\w.:-]+$`; violation is a hard error before any pane is built.
- Known quirks (exact set): `self-update-exit`, `trust-gate-enter`, `esc-clears-stray-prompt`, `extended-keys`, `session-purpose-first`. Unknown quirk in the registry = hard error (fail closed).
- Registry `SchemaVersion` must equal `1`.
- Pane statuses (exact set): `ready | relaunched-ready | timeout | launch-failed | pane-dead`.
- `grid-manifest.json` lives beside fanout's `connect-manifest.json`: `%TEMP%\psmux-fanout\<session>\grid-manifest.json`.
- Tests never launch real LLM CLIs. Integration tests run on the default psmux server ONLY when it has zero sessions (same skip-guard as fanout's `Provision.Tests.ps1`) and set `$env:PSMUX_ALLOW_ELEVATED='1'`.
- Every `Invoke-PsmuxWithRetry` call site checks `$LASTEXITCODE` afterward (the helper returns non-transient failures, it does not throw).
- Readiness capture always uses `capture-pane -p -J` (join wrapped lines).
- send-keys submit rule: literal text (`send-keys -l`), pause 600ms, then a **separate** lone `Enter`.

---

### Task 1: Scaffold + harness registry (`harnesses.json`, `_registry-lib.ps1`)

**Files:**
- Create: `~\.claude\skills\psmux-harness-grid\harnesses.json`
- Create: `~\.claude\skills\psmux-harness-grid\_registry-lib.ps1`
- Test: `~\.claude\skills\psmux-harness-grid\tests\Registry.Tests.ps1`

**Interfaces:**
- Consumes: nothing (pure).
- Produces: `Import-HarnessRegistry([string]$Path) -> hashtable` (harness name → definition object; throws on any validation failure). `Resolve-HarnessLaunch([object]$Def, [string]$Model, [string]$Effort) -> string` (the exact one-line shell command to type into a pane; throws on bad charset). `$script:KnownQuirks` (string[]).

- [ ] **Step 1: git init + write the registry**

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills\psmux-harness-grid\tests" | Out-Null
git -C "$env:USERPROFILE\.claude\skills\psmux-harness-grid" init
```

`harnesses.json` (exact content):

```json
{
  "SchemaVersion": 1,
  "Harnesses": {
    "claude": {
      "executable": "claude",
      "baseArgs": ["--dangerously-skip-permissions"],
      "modelArgs": ["--model", "{model}"],
      "effortArgs": ["--effort", "{effort}"],
      "readyPattern": "bypasspermissions|❯",
      "exitCommand": "/exit",
      "quirks": ["trust-gate-enter"]
    },
    "codex": {
      "executable": "codex",
      "baseArgs": ["--dangerously-bypass-approvals-and-sandbox"],
      "modelArgs": ["-m", "{model}"],
      "effortArgs": ["-c", "model_reasoning_effort={effort}"],
      "readyPattern": "(gpt|codex)[^\\n]*\\b(minimal|low|medium|high|ultra|xhigh)\\b",
      "exitCommand": "/exit",
      "quirks": ["self-update-exit", "esc-clears-stray-prompt"]
    },
    "agy": {
      "executable": "agy",
      "baseArgs": [],
      "modelArgs": [],
      "effortArgs": [],
      "readyPattern": "\\? for shortcuts",
      "exitCommand": "/exit",
      "quirks": ["trust-gate-enter"]
    },
    "omp": {
      "executable": "omp",
      "baseArgs": ["--auto-approve"],
      "modelArgs": [],
      "effortArgs": [],
      "readyPattern": "Ready\\. What would you like|glm-[\\w.:-]+ ",
      "exitCommand": "/exit",
      "quirks": ["self-update-exit"]
    }
  }
}
```

(pi is deliberately absent — spec: added later only if its readyPattern is captured live.)

- [ ] **Step 2: Write failing tests**

`tests\Registry.Tests.ps1`:

```powershell
#requires -Version 7.1
BeforeAll {
    $script:SkillDir = Split-Path $PSScriptRoot -Parent
    . (Join-Path $script:SkillDir '_registry-lib.ps1')
    $script:RegistryPath = Join-Path $script:SkillDir 'harnesses.json'
    function New-TempRegistry([hashtable]$Obj) {
        $p = Join-Path $env:TEMP ("hreg_{0}.json" -f ([guid]::NewGuid().ToString('n').Substring(0,8)))
        $Obj | ConvertTo-Json -Depth 10 | Set-Content -LiteralPath $p -Encoding utf8
        $p
    }
}
Describe 'Import-HarnessRegistry' {
    It 'loads the shipped registry with 4 harnesses' {
        $r = Import-HarnessRegistry -Path $script:RegistryPath
        @($r.Keys) | Sort-Object | Should -Be @('agy','claude','codex','omp')
    }
    It 'fails closed on unknown quirk' {
        $p = New-TempRegistry @{ SchemaVersion = 1; Harnesses = @{ x = @{
            executable='x'; baseArgs=@(); modelArgs=@(); effortArgs=@()
            readyPattern='ok'; exitCommand='/exit'; quirks=@('trust-gate-entre') } } }
        { Import-HarnessRegistry -Path $p } | Should -Throw '*unknown quirk*'
    }
    It 'rejects wrong SchemaVersion' {
        $p = New-TempRegistry @{ SchemaVersion = 2; Harnesses = @{} }
        { Import-HarnessRegistry -Path $p } | Should -Throw '*SchemaVersion*'
    }
    It 'rejects a non-compiling readyPattern' {
        $p = New-TempRegistry @{ SchemaVersion = 1; Harnesses = @{ x = @{
            executable='x'; baseArgs=@(); modelArgs=@(); effortArgs=@()
            readyPattern='('; exitCommand=''; quirks=@() } } }
        { Import-HarnessRegistry -Path $p } | Should -Throw '*readyPattern*'
    }
    It 'rejects a missing executable' {
        $p = New-TempRegistry @{ SchemaVersion = 1; Harnesses = @{ x = @{
            baseArgs=@(); modelArgs=@(); effortArgs=@(); readyPattern='ok'; exitCommand=''; quirks=@() } } }
        { Import-HarnessRegistry -Path $p } | Should -Throw '*executable*'
    }
}
Describe 'Resolve-HarnessLaunch' {
    BeforeAll { $script:Reg = Import-HarnessRegistry -Path $script:RegistryPath }
    It 'emits base command with no model/effort' {
        Resolve-HarnessLaunch -Def $script:Reg.agy | Should -Be 'agy'
    }
    It 'fills model and effort slots' {
        Resolve-HarnessLaunch -Def $script:Reg.claude -Model 'claude-fable-5' -Effort 'high' |
            Should -Be 'claude --dangerously-skip-permissions --model claude-fable-5 --effort high'
    }
    It 'fills codex -c effort form' {
        Resolve-HarnessLaunch -Def $script:Reg.codex -Model 'gpt-5.6-sol' -Effort 'high' |
            Should -Be 'codex --dangerously-bypass-approvals-and-sandbox -m gpt-5.6-sol -c model_reasoning_effort=high'
    }
    It 'omits model args when model not given' {
        Resolve-HarnessLaunch -Def $script:Reg.claude |
            Should -Be 'claude --dangerously-skip-permissions'
    }
    It 'rejects unsafe model charset' {
        { Resolve-HarnessLaunch -Def $script:Reg.claude -Model 'x; rm -rf' } | Should -Throw '*charset*'
    }
}
```

- [ ] **Step 3: Run tests to verify failure**

Run: `pwsh -NoProfile -Command "Invoke-Pester '$env:USERPROFILE\.claude\skills\psmux-harness-grid\tests\Registry.Tests.ps1' -Output Detailed"`
Expected: FAIL — `_registry-lib.ps1` not found.

- [ ] **Step 4: Implement `_registry-lib.ps1`**

```powershell
#requires -Version 7.1
# Harness registry: load + validate harnesses.json (fail closed), resolve launch command lines.

$script:KnownQuirks = @('self-update-exit','trust-gate-enter','esc-clears-stray-prompt',
                        'extended-keys','session-purpose-first')
$script:SafeValue = '^[\w.:-]+$'

function Import-HarnessRegistry {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string] $Path)
    if (-not (Test-Path -LiteralPath $Path)) { throw "harness registry not found: $Path" }
    $raw = Get-Content -LiteralPath $Path -Raw | ConvertFrom-Json
    if ($raw.SchemaVersion -ne 1) { throw "registry SchemaVersion must be 1, got '$($raw.SchemaVersion)'" }
    $out = @{}
    foreach ($prop in $raw.Harnesses.PSObject.Properties) {
        $name = $prop.Name; $d = $prop.Value
        if ([string]::IsNullOrWhiteSpace($d.executable)) { throw "harness '$name': executable is required" }
        foreach ($f in 'baseArgs','modelArgs','effortArgs') {
            if ($null -eq $d.$f) { throw "harness '$name': $f is required (use [] when empty)" }
        }
        if ([string]::IsNullOrWhiteSpace($d.readyPattern)) { throw "harness '$name': readyPattern is required" }
        try { [void][regex]::new($d.readyPattern) } catch { throw "harness '$name': readyPattern does not compile: $($_.Exception.Message)" }
        foreach ($q in @($d.quirks)) {
            if ($script:KnownQuirks -notcontains $q) { throw "harness '$name': unknown quirk '$q' (fail closed)" }
        }
        $out[$name] = $d
    }
    $out
}

function Resolve-HarnessLaunch {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][object] $Def,
        [string] $Model,
        [string] $Effort
    )
    foreach ($pair in @(@('Model',$Model), @('Effort',$Effort))) {
        if ($pair[1] -and $pair[1] -notmatch $script:SafeValue) {
            throw "$($pair[0]) value '$($pair[1])' fails safe charset $script:SafeValue"
        }
    }
    $parts = @($Def.executable) + @($Def.baseArgs)
    if ($Model)  { $parts += @($Def.modelArgs  | ForEach-Object { $_ -replace '\{model\}',  $Model  }) }
    if ($Effort) { $parts += @($Def.effortArgs | ForEach-Object { $_ -replace '\{effort\}', $Effort }) }
    ($parts | Where-Object { $_ -ne '' }) -join ' '
}
```

- [ ] **Step 5: Run tests to verify pass**

Run: same Pester command as Step 3.
Expected: PASS (10 tests).

- [ ] **Step 6: Commit**

```bash
cd "$HOME/.claude/skills/psmux-harness-grid"
git add harnesses.json _registry-lib.ps1 tests/Registry.Tests.ps1
git commit -m "feat: harness registry with fail-closed validation and launch resolution"
```

---

### Task 2: Spec parsing + grid manifest (`_grid-lib.ps1`, part 1 — pure functions)

**Files:**
- Create: `~\.claude\skills\psmux-harness-grid\_grid-lib.ps1`
- Test: `~\.claude\skills\psmux-harness-grid\tests\Grid.Tests.ps1`

**Interfaces:**
- Consumes: nothing (pure; live-pane data injected as parameters).
- Produces: `ConvertFrom-GridSpec([string]$Spec) -> hashtable[]` (`@{Harness;Model;Effort}` per pane). `Get-GridManifestPath([string]$Session) -> string`. `Write-GridManifest([string]$Path,[string]$State,[string]$Session,[string]$BuildId,[object[]]$Panes)` (atomic: temp file + Move-Item). `Read-GridManifest([string]$Path) -> object`. `Test-ConnectManifestLive([object]$Manifest,[string[]]$LivePaneIds,[int]$ExpectedCount)` (throws on stale/mismatch).

- [ ] **Step 1: Write failing tests**

`tests\Grid.Tests.ps1`:

```powershell
#requires -Version 7.1
BeforeAll {
    $script:SkillDir = Split-Path $PSScriptRoot -Parent
    . (Join-Path $script:SkillDir '_grid-lib.ps1')
}
Describe 'ConvertFrom-GridSpec' {
    It 'parses harness:model:effort list' {
        $p = ConvertFrom-GridSpec 'claude:claude-fable-5:high, codex:gpt-5.6-sol:high, agy, omp'
        $p.Count | Should -Be 4
        $p[0].Harness | Should -Be 'claude'; $p[0].Model | Should -Be 'claude-fable-5'; $p[0].Effort | Should -Be 'high'
        $p[2].Harness | Should -Be 'agy';    $p[2].Model | Should -BeNullOrEmpty
    }
    It 'parses harness:model with no effort' {
        (ConvertFrom-GridSpec 'omp:glm-5.2')[0].Model | Should -Be 'glm-5.2'
    }
    It 'rejects empty entries' {
        { ConvertFrom-GridSpec 'claude,,omp' } | Should -Throw '*empty*'
    }
}
Describe 'grid manifest' {
    It 'writes early building state and reads it back' {
        $p = Join-Path $env:TEMP ("gm_{0}\grid-manifest.json" -f ([guid]::NewGuid().ToString('n').Substring(0,8)))
        $panes = @([pscustomobject]@{ Index=0; PaneId='%1'; PanePid=1234; Harness='claude';
                    Model='claude-fable-5'; Effort='high'; Status='pending'; Title='claude-claude-fable-5'; NeedsSetup=$false })
        Write-GridManifest -Path $p -State 'building' -Session 'fanout' -BuildId 'b-1' -Panes $panes
        $m = Read-GridManifest -Path $p
        $m.SchemaVersion | Should -Be 1
        $m.State | Should -Be 'building'
        $m.BuildId | Should -Be 'b-1'
        @($m.Panes).Count | Should -Be 1
        $m.Panes[0].PanePid | Should -Be 1234
    }
}
Describe 'Test-ConnectManifestLive' {
    BeforeAll {
        $script:Manifest = [pscustomobject]@{ Session='s'; Panes=@(
            [pscustomobject]@{ Index=0; PaneId='%1' }, [pscustomobject]@{ Index=1; PaneId='%4' }) }
    }
    It 'passes when all manifest ids are live and count matches' {
        { Test-ConnectManifestLive -Manifest $script:Manifest -LivePaneIds @('%1','%4') -ExpectedCount 2 } | Should -Not -Throw
    }
    It 'throws on stale manifest (id not live)' {
        { Test-ConnectManifestLive -Manifest $script:Manifest -LivePaneIds @('%1','%9') -ExpectedCount 2 } | Should -Throw '*stale*'
    }
    It 'throws on count mismatch' {
        { Test-ConnectManifestLive -Manifest $script:Manifest -LivePaneIds @('%1','%4') -ExpectedCount 3 } | Should -Throw '*count*'
    }
}
```

- [ ] **Step 2: Run tests to verify failure**

Run: `pwsh -NoProfile -Command "Invoke-Pester '$env:USERPROFILE\.claude\skills\psmux-harness-grid\tests\Grid.Tests.ps1' -Output Detailed"`
Expected: FAIL — `_grid-lib.ps1` not found.

- [ ] **Step 3: Implement `_grid-lib.ps1`**

```powershell
#requires -Version 7.1
# Grid-level pure helpers: -Spec parsing, grid-manifest read/write, connect-manifest liveness.

function ConvertFrom-GridSpec {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string] $Spec)
    @($Spec -split ',') | ForEach-Object {
        $entry = $_.Trim()
        if (-not $entry) { throw "grid spec contains an empty entry: '$Spec'" }
        $bits = $entry -split ':'
        @{ Harness = $bits[0].Trim()
           Model   = if ($bits.Count -ge 2) { $bits[1].Trim() } else { $null }
           Effort  = if ($bits.Count -ge 3) { $bits[2].Trim() } else { $null } }
    }
}

function Get-GridManifestPath {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string] $Session)
    Join-Path (Join-Path $env:TEMP 'psmux-fanout') (Join-Path $Session 'grid-manifest.json')
}

function Write-GridManifest {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string] $Path,
        [Parameter(Mandatory)][ValidateSet('building','up','torn-down')][string] $State,
        [Parameter(Mandatory)][string] $Session,
        [string] $BuildId,
        [object[]] $Panes = @()
    )
    $dir = Split-Path -LiteralPath $Path
    New-Item -ItemType Directory -Force -Path $dir | Out-Null
    $doc = [pscustomobject]@{
        SchemaVersion = 1; State = $State; Session = $Session; BuildId = $BuildId
        Timestamp = (Get-Date).ToString('o'); Panes = @($Panes)
    }
    $tmp = "$Path.tmp"
    $doc | ConvertTo-Json -Depth 6 | Set-Content -LiteralPath $tmp -Encoding utf8
    Move-Item -LiteralPath $tmp -Destination $Path -Force
}

function Read-GridManifest {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string] $Path)
    if (-not (Test-Path -LiteralPath $Path)) { throw "grid manifest not found: $Path" }
    Get-Content -LiteralPath $Path -Raw | ConvertFrom-Json
}

function Test-ConnectManifestLive {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][object] $Manifest,
        [Parameter(Mandatory)][AllowEmptyCollection()][string[]] $LivePaneIds,
        [Parameter(Mandatory)][int] $ExpectedCount
    )
    $ids = @($Manifest.Panes | ForEach-Object { $_.PaneId })
    if ($ids.Count -ne $ExpectedCount) { throw "connect-manifest pane count $($ids.Count) != requested $ExpectedCount" }
    foreach ($id in $ids) {
        if ($LivePaneIds -notcontains $id) { throw "connect-manifest is stale: pane $id not alive in session" }
    }
}
```

- [ ] **Step 4: Run tests to verify pass**

Run: same as Step 2. Expected: PASS (8 tests).

- [ ] **Step 5: Commit**

```bash
cd "$HOME/.claude/skills/psmux-harness-grid"
git add _grid-lib.ps1 tests/Grid.Tests.ps1
git commit -m "feat: grid spec parsing, atomic grid-manifest, connect-manifest liveness check"
```

---

### Task 3: Pane targeting + send helpers (`_target-lib.ps1`)

**Files:**
- Create: `~\.claude\skills\psmux-harness-grid\_target-lib.ps1`
- Test: `~\.claude\skills\psmux-harness-grid\tests\Target.Tests.ps1`

**Interfaces:**
- Consumes: fanout's `Invoke-PsmuxWithRetry` (dot-sourced from `psmux-fanout\_layout-lib.ps1`).
- Produces: `Get-GridPaneInfo([string]$Psmux,[string]$Session) -> hashtable` (pane `%id` → `@{Index;PanePid;CurrentCommand}` from one `list-panes` call). `Resolve-PaneTarget([string]$Psmux,[string]$Session,[string]$PaneId) -> string` (qualified `<session>:agents.<index>`, or `$null` when the pane is gone). `Send-PaneCommand([string]$Psmux,[string]$Session,[string]$PaneId,[string]$Text)` (literal text → 600ms → lone Enter; throws if pane is gone or psmux fails). `Send-PaneKey([string]$Psmux,[string]$Session,[string]$PaneId,[string]$Key)` (single key e.g. `Enter`, `Escape`, `C-c`). `Get-PaneCapture([string]$Psmux,[string]$Session,[string]$PaneId) -> string` (`capture-pane -p -J` joined; `$null` when pane gone).

**Integration tests:** default psmux server, skipped unless empty (fanout `Provision.Tests.ps1` guard, copied verbatim); `$env:PSMUX_ALLOW_ELEVATED='1'` in `BeforeAll`.

- [ ] **Step 1: Write failing tests**

`tests\Target.Tests.ps1`:

```powershell
#requires -Version 7.1
$DiscoveryPsmux = (Get-Command psmux -ErrorAction SilentlyContinue)?.Source
$DiscoveryDefaultSessions = if ($DiscoveryPsmux) {
    @(& $DiscoveryPsmux ls 2>$null | ForEach-Object { ($_ -split ':')[0].Trim() } | Where-Object { $_ })
} else { @() }
$CanRun = [bool]($DiscoveryPsmux -and $DiscoveryDefaultSessions.Count -eq 0)

BeforeAll {
    $env:PSMUX_ALLOW_ELEVATED = '1'
    $script:SkillDir = Split-Path $PSScriptRoot -Parent
    . (Join-Path $script:SkillDir '_target-lib.ps1')
    $script:psmux = (Get-Command psmux -ErrorAction SilentlyContinue)?.Source
}
Describe 'pane targeting' -Skip:(-not $CanRun) {
    BeforeEach {
        & $script:psmux new-session -d -s ttest -n agents
        $script:p2 = (& $script:psmux split-window -t 'ttest:agents.0' -v -P -F '#{pane_id}').Trim()
    }
    AfterEach { & $script:psmux kill-session -t ttest 2>$null }

    It 'maps ids to index/pid/command in one call' {
        $info = Get-GridPaneInfo -Psmux $script:psmux -Session 'ttest'
        $info.Keys.Count | Should -Be 2
        $info[$script:p2].PanePid | Should -BeGreaterThan 0
        $info[$script:p2].CurrentCommand | Should -Match 'pwsh|powershell|cmd'
    }
    It 'resolves a qualified target and null for a dead pane' {
        Resolve-PaneTarget -Psmux $script:psmux -Session 'ttest' -PaneId $script:p2 |
            Should -Match '^ttest:agents\.\d+$'
        Resolve-PaneTarget -Psmux $script:psmux -Session 'ttest' -PaneId '%999' | Should -BeNullOrEmpty
    }
    It 'targets THIS session''s pane when another session shares the %id (duplicate-%id collision)' {
        # Build a second session; on psmux, ids are per-server-reused so both can own a %0/%1.
        & $script:psmux new-session -d -s other -n agents
        try {
            $mine = (& $script:psmux list-panes -t 'ttest:agents' -F '#{pane_id}')[0].Trim()
            Send-PaneCommand -Psmux $script:psmux -Session 'ttest' -PaneId $mine -Text "echo TT_$PID"
            Start-Sleep -Seconds 2
            (Get-PaneCapture -Psmux $script:psmux -Session 'ttest' -PaneId $mine) | Should -Match "TT_$PID"
            # the other session's same-index pane must NOT have received it
            $otherCap = & $script:psmux capture-pane -t 'other:agents.0' -p -J
            ($otherCap -join "`n") | Should -Not -Match "TT_$PID"
        } finally { & $script:psmux kill-session -t other 2>$null }
    }
    It 'send + capture round-trips' {
        Send-PaneCommand -Psmux $script:psmux -Session 'ttest' -PaneId $script:p2 -Text 'echo HELLO_GRID'
        Start-Sleep -Seconds 2
        (Get-PaneCapture -Psmux $script:psmux -Session 'ttest' -PaneId $script:p2) | Should -Match 'HELLO_GRID'
    }
    It 'Send-PaneCommand throws for a dead pane' {
        { Send-PaneCommand -Psmux $script:psmux -Session 'ttest' -PaneId '%999' -Text 'x' } | Should -Throw '*pane*'
    }
}
```

- [ ] **Step 2: Run tests to verify failure**

Run: `pwsh -NoProfile -Command "Invoke-Pester '$env:USERPROFILE\.claude\skills\psmux-harness-grid\tests\Target.Tests.ps1' -Output Detailed"`
Expected: FAIL — `_target-lib.ps1` not found. (SKIPPED if the default server has sessions — clear them first.)

- [ ] **Step 3: Implement `_target-lib.ps1`**

```powershell
#requires -Version 7.1
# Pane addressing: identity by %id, targeting by fresh id->index resolution scoped to
# <session>:agents. Immune to cross-session %id collisions (fanout gotcha #9) and to
# pane_index renumbering. All psmux calls go through fanout's Invoke-PsmuxWithRetry
# (transient 10061 backoff) and check $LASTEXITCODE — the helper returns, never throws.

$script:FanoutDir = Join-Path (Split-Path $PSScriptRoot -Parent) 'psmux-fanout'
if (-not (Get-Command Invoke-PsmuxWithRetry -ErrorAction SilentlyContinue)) {
    . (Join-Path $script:FanoutDir '_layout-lib.ps1')
}

function Invoke-Psmux {
    # Retry + exit-status check in one place. Returns stdout lines.
    param([Parameter(Mandatory)][string]$Psmux, [Parameter(Mandatory)][string[]]$ArgList)
    $out = Invoke-PsmuxWithRetry { param($a) & $Psmux @a } @(,$ArgList)
    if ($LASTEXITCODE -ne 0) { throw "psmux $($ArgList -join ' ') failed ($LASTEXITCODE): $out" }
    $out
}

function Get-GridPaneInfo {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Psmux, [Parameter(Mandatory)][string]$Session)
    $lines = Invoke-Psmux -Psmux $Psmux -ArgList @('list-panes','-t',"${Session}:agents",'-F','#{pane_index}|#{pane_id}|#{pane_pid}|#{pane_current_command}')
    $map = @{}
    foreach ($l in @($lines)) {
        $f = "$l".Trim() -split '\|'
        if ($f.Count -ge 4) { $map[$f[1]] = @{ Index=[int]$f[0]; PanePid=[int]$f[2]; CurrentCommand=$f[3] } }
    }
    $map
}

function Resolve-PaneTarget {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Psmux, [Parameter(Mandatory)][string]$Session,
          [Parameter(Mandatory)][string]$PaneId)
    $info = Get-GridPaneInfo -Psmux $Psmux -Session $Session
    if (-not $info.ContainsKey($PaneId)) { return $null }
    "${Session}:agents.$($info[$PaneId].Index)"
}

function Send-PaneCommand {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Psmux, [Parameter(Mandatory)][string]$Session,
          [Parameter(Mandatory)][string]$PaneId, [Parameter(Mandatory)][string]$Text)
    $t = Resolve-PaneTarget -Psmux $Psmux -Session $Session -PaneId $PaneId
    if (-not $t) { throw "pane $PaneId not found in session '$Session' (pane-dead)" }
    Invoke-Psmux -Psmux $Psmux -ArgList @('send-keys','-t',$t,'-l',$Text) | Out-Null
    Start-Sleep -Milliseconds 600
    # re-resolve: the 600ms window is small but a layout change would shift indexes
    $t = Resolve-PaneTarget -Psmux $Psmux -Session $Session -PaneId $PaneId
    if (-not $t) { throw "pane $PaneId vanished before Enter (pane-dead)" }
    Invoke-Psmux -Psmux $Psmux -ArgList @('send-keys','-t',$t,'Enter') | Out-Null
}

function Send-PaneKey {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Psmux, [Parameter(Mandatory)][string]$Session,
          [Parameter(Mandatory)][string]$PaneId, [Parameter(Mandatory)][string]$Key)
    $t = Resolve-PaneTarget -Psmux $Psmux -Session $Session -PaneId $PaneId
    if (-not $t) { throw "pane $PaneId not found in session '$Session' (pane-dead)" }
    Invoke-Psmux -Psmux $Psmux -ArgList @('send-keys','-t',$t,$Key) | Out-Null
}

function Get-PaneCapture {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Psmux, [Parameter(Mandatory)][string]$Session,
          [Parameter(Mandatory)][string]$PaneId)
    $t = Resolve-PaneTarget -Psmux $Psmux -Session $Session -PaneId $PaneId
    if (-not $t) { return $null }
    (Invoke-Psmux -Psmux $Psmux -ArgList @('capture-pane','-t',$t,'-p','-J')) -join "`n"
}
```

- [ ] **Step 4: Run tests to verify pass**

Run: same as Step 2 (default server must be empty). Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
cd "$HOME/.claude/skills/psmux-harness-grid"
git add _target-lib.ps1 tests/Target.Tests.ps1
git commit -m "feat: qualified pane targeting with fresh id->index resolution and safe send helpers"
```

---

### Task 4: Launch + readiness sweep (`_harness-lib.ps1`)

**Files:**
- Create: `~\.claude\skills\psmux-harness-grid\_harness-lib.ps1`
- Test: `~\.claude\skills\psmux-harness-grid\tests\Ready.Tests.ps1`

**Interfaces:**
- Consumes: Task 1 (`Resolve-HarnessLaunch`), Task 3 (all senders/capture/info).
- Produces: `Start-PaneHarness([string]$Psmux,[string]$Session,[object]$Pane,[object]$Def)` — sets pane title, sends the launch command, stamps `$Pane.LaunchedAt` (`[datetime]`). `Wait-HarnessReady([string]$Psmux,[string]$Session,[object[]]$Panes,[hashtable]$Registry,[int]$TimeoutSec=90,[int]$PollMs=5000)` — single sweep loop; mutates each `$Pane.Status` to a final value; honors `self-update-exit` (one relaunch via `$Pane.Relaunched` flag), `trust-gate-enter`, `esc-clears-stray-prompt`. Pane objects are `[pscustomobject]` with `Index,PaneId,PanePid,Harness,Model,Effort,Status,Title,NeedsSetup,LaunchedAt,Relaunched`.
- Shell detection: `CurrentCommand -match '^(pwsh|powershell|cmd)(\.exe)?$'` means "back at the shell".
- Trust-gate detection: capture matches `(?i)do you trust|trust the files|trust this folder`.

**Fake harness for tests** (no LLMs): launch scripts that are plain pwsh one-liners; the test builds a temp registry whose entries exercise each path:
- `fake-ready`: `executable='pwsh'`, `baseArgs=@('-NoProfile','-Command','Write-Host READY_FAKE; Start-Sleep 300')`, `readyPattern='READY_FAKE'`.
- `fake-update`: same shape but the command exits immediately unless a marker file exists (`if (Test-Path $env:TEMP\fake_upd.marker) { 'READY_FAKE'; sleep 300 } else { New-Item $env:TEMP\fake_upd.marker; exit }`), `quirks=@('self-update-exit')` — first launch exits (simulated self-update), relaunch is ready → expect `relaunched-ready`.
- `fake-never`: prints nothing ready-like, sleeps → expect `timeout` (test uses `-TimeoutSec 15`).
- `fake-exit`: exits immediately, no quirk → expect `launch-failed`.

- [ ] **Step 1: Write failing tests** — `tests\Ready.Tests.ps1` with the default-server skip guard from Task 3; `BeforeEach` provisions a plain `new-session -d -s rtest -n agents` + N split panes (the same primitive loop as Target tests, not fanout, to keep the unit small); one `It` per fake above asserting the final `Status`, plus: `It 'reports pane-dead when the pane is killed mid-wait'` (kill-pane during the sweep, expect `pane-dead`).

- [ ] **Step 2: Run to verify failure** — `Invoke-Pester ...\tests\Ready.Tests.ps1` → FAIL (`_harness-lib.ps1` missing).

- [ ] **Step 3: Implement `_harness-lib.ps1`**

```powershell
#requires -Version 7.1
# Launch a harness into a pane and drive the readiness sweep (statuses per spec rev 3).

. (Join-Path $PSScriptRoot '_registry-lib.ps1')
. (Join-Path $PSScriptRoot '_target-lib.ps1')

$script:ShellPattern = '^(pwsh|powershell|cmd)(\.exe)?$'
$script:TrustPattern = '(?i)do you trust|trust the files|trust this folder'

function Start-PaneHarness {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Psmux, [Parameter(Mandatory)][string]$Session,
          [Parameter(Mandatory)][object]$Pane, [Parameter(Mandatory)][object]$Def)
    $t = Resolve-PaneTarget -Psmux $Psmux -Session $Session -PaneId $Pane.PaneId
    if (-not $t) { $Pane.Status = 'pane-dead'; return }
    Invoke-Psmux -Psmux $Psmux -ArgList @('select-pane','-t',$t,'-T',$Pane.Title) | Out-Null
    $cmd = Resolve-HarnessLaunch -Def $Def -Model $Pane.Model -Effort $Pane.Effort
    Send-PaneCommand -Psmux $Psmux -Session $Session -PaneId $Pane.PaneId -Text $cmd
    $Pane.LaunchedAt = Get-Date
}

function Wait-HarnessReady {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Psmux, [Parameter(Mandatory)][string]$Session,
          [Parameter(Mandatory)][object[]]$Panes, [Parameter(Mandatory)][hashtable]$Registry,
          [int]$TimeoutSec = 90, [int]$PollMs = 5000)
    while ($true) {
        $pending = @($Panes | Where-Object { $_.Status -in 'pending','launched' })
        if (-not $pending) { break }
        $info = $null
        try { $info = Get-GridPaneInfo -Psmux $Psmux -Session $Session } catch { $info = $null }
        foreach ($pane in $pending) {
            $def = $Registry[$pane.Harness]
            $quirks = @($def.quirks)
            if ($info -and -not $info.ContainsKey($pane.PaneId)) { $pane.Status = 'pane-dead'; continue }
            $cap = $null
            try { $cap = Get-PaneCapture -Psmux $Psmux -Session $Session -PaneId $pane.PaneId } catch {}
            if ($cap -and $cap -match $def.readyPattern) {
                $pane.Status = if ($pane.Relaunched) { 'relaunched-ready' } else { 'ready' }
                if ($quirks -contains 'esc-clears-stray-prompt') {
                    Send-PaneKey -Psmux $Psmux -Session $Session -PaneId $pane.PaneId -Key 'Escape'
                }
                continue
            }
            $atShell = $info -and $info.ContainsKey($pane.PaneId) -and
                       ($info[$pane.PaneId].CurrentCommand -match $script:ShellPattern)
            if ($atShell) {
                if (($quirks -contains 'self-update-exit') -and -not $pane.Relaunched) {
                    $pane.Relaunched = $true
                    Start-PaneHarness -Psmux $Psmux -Session $Session -Pane $pane -Def $def
                } else { $pane.Status = 'launch-failed' }
                continue
            }
            if ($cap -and ($quirks -contains 'trust-gate-enter') -and $cap -match $script:TrustPattern) {
                Send-PaneKey -Psmux $Psmux -Session $Session -PaneId $pane.PaneId -Key 'Enter'
            }
            if (((Get-Date) - $pane.LaunchedAt).TotalSeconds -gt $TimeoutSec) { $pane.Status = 'timeout' }
        }
        if (@($Panes | Where-Object { $_.Status -in 'pending','launched' })) { Start-Sleep -Milliseconds $PollMs }
    }
}
```

- [ ] **Step 4: Run to verify pass** — expect PASS (5 tests: ready, relaunched-ready, timeout, launch-failed, pane-dead).

- [ ] **Step 5: Commit** — `git add _harness-lib.ps1 tests/Ready.Tests.ps1 && git commit -m "feat: harness launch + readiness sweep with quirk handling"`

---

### Task 5: `gridup.ps1` entry point

**Files:**
- Create: `~\.claude\skills\psmux-harness-grid\gridup.ps1`
- Test: `~\.claude\skills\psmux-harness-grid\tests\Gridup.Tests.ps1`

**Interfaces:**
- Consumes: Tasks 1–4; fanout's `fanout.ps1 -Provision` (returns `{Provisioned,Session,PaneCount,PaneIds,ManifestPath,...}`); `Get-FanoutManifestPath` (fanout `_manifest-lib.ps1`).
- Produces: the skill's primary entry. Params: `-Panes [hashtable[]]` XOR `-Spec [string]` (exactly one required), `-Session 'fanout'`, pass-throughs `-Visible,-FreshStart,-LayoutPath,-AgentCwd,-CanvasWidth,-CanvasHeight`, `-TimeoutSec 90`, `-StaggerMs 2500`, `-RegistryPath (Join-Path $PSScriptRoot 'harnesses.json')`, `-Force`, `-KeepPartial`. Returns the pane `[pscustomobject[]]` (spec step 8 shape).

Flow (exactly spec "Engine flow"): registry+panes validation → RAM pre-flight (`(Get-CimInstance Win32_OperatingSystem).FreePhysicalMemory/1MB` GB: warn < 2.5, throw < 1 unless `-Force`) → set `$env:PSMUX_ALLOW_ELEVATED='1'` → call fanout `-Provision -PaneCount N` (splat only the pass-throughs the caller set) → read connect-manifest via `Get-FanoutManifestPath`, `Test-ConnectManifestLive` against `Get-GridPaneInfo` keys → build pane records (manifest order i → `-Panes[i]`), fill `PanePid` from pane info → `Write-GridManifest -State building` → session quirk: if any pane's def has `extended-keys`, `Invoke-Psmux set-option -t <session> extended-keys on` once → CLS pass: `Send-PaneCommand ... -Text 'cls'` to every pane → launches: `Start-PaneHarness` per pane with `Start-Sleep -Milliseconds $StaggerMs` between → `Wait-HarnessReady` → `NeedsSetup = $true` where def quirks contain `session-purpose-first` → `Write-GridManifest -State up` (final statuses) → return records. Whole flow after provision wrapped in `try/catch`: on throw, unless `-KeepPartial`, invoke `griddown.ps1 -Session $Session` (Task 6) then rethrow.

- [ ] **Step 1: Write failing test** — `tests\Gridup.Tests.ps1` (default-server skip guard): one contract `It` — `& gridup.ps1 -Panes @(@{Harness='fake-ready'},@{Harness='fake-ready'},@{Harness='fake-update'})` with `-RegistryPath` pointing at a temp fake registry (fakes from Task 4) and `-Session gridtest`; assert: 3 records all `Status -in 'ready','relaunched-ready'`; `grid-manifest.json` exists with `State='up'` and 3 panes with `PanePid -gt 0`; every live pane's capture no longer shows provision stdio junk (CLS pass ran — capture does not match `Provisioned`); plus `It 'rejects unknown harness before provisioning'` (expect throw AND `psmux ls` still empty — nothing was built); plus `It 'stale connect-manifest is a hard failure'` (pre-write a bogus `connect-manifest.json` with dead pane ids at the fanout path, run with `-FreshStart:$false` against a hand-built empty session, expect `*stale*` throw).

- [ ] **Step 2: Run to verify failure** — FAIL (`gridup.ps1` missing).

- [ ] **Step 3: Implement `gridup.ps1`** — param block + flow above; key skeleton:

```powershell
#requires -Version 7.1
[CmdletBinding()]
param(
    [hashtable[]] $Panes,
    [string] $Spec,
    [string] $Session = 'fanout',
    [switch] $Visible, [switch] $FreshStart, [string] $LayoutPath,
    [string] $AgentCwd = (Get-Location).Path,
    [int] $CanvasWidth = 0, [int] $CanvasHeight = 0,
    [int] $TimeoutSec = 90, [int] $StaggerMs = 2500,
    [string] $RegistryPath = (Join-Path $PSScriptRoot 'harnesses.json'),
    [switch] $Force, [switch] $KeepPartial
)
$ErrorActionPreference = 'Stop'
. (Join-Path $PSScriptRoot '_grid-lib.ps1')
. (Join-Path $PSScriptRoot '_harness-lib.ps1')   # pulls registry+target libs
$FanoutDir = Join-Path (Split-Path $PSScriptRoot -Parent) 'psmux-fanout'
. (Join-Path $FanoutDir '_manifest-lib.ps1')

if (-not $Panes -and -not $Spec) { throw 'supply -Panes or -Spec' }
if ($Panes -and $Spec) { throw 'supply -Panes OR -Spec, not both' }
if ($Spec) { $Panes = @(ConvertFrom-GridSpec -Spec $Spec) }

$registry = Import-HarnessRegistry -Path $RegistryPath
foreach ($p in $Panes) {
    if (-not $registry.ContainsKey($p.Harness)) { throw "unknown harness '$($p.Harness)'" }
    # charset check now, before anything is built
    [void](Resolve-HarnessLaunch -Def $registry[$p.Harness] -Model $p.Model -Effort $p.Effort)
}

$freeGb = (Get-CimInstance Win32_OperatingSystem).FreePhysicalMemory / 1MB
if ($freeGb -lt 1 -and -not $Force) { throw ("free RAM {0:N2} GB < 1 GB floor (use -Force to override)" -f $freeGb) }
if ($freeGb -lt 2.5) { Write-Warning ("free RAM {0:N2} GB is below the 2.5 GB comfort line" -f $freeGb) }

$env:PSMUX_ALLOW_ELEVATED = '1'
$psmux = (Get-Command psmux -ErrorAction Stop).Source
$fanoutArgs = @{ Provision = $true; PaneCount = @($Panes).Count; Session = $Session; AgentCwd = $AgentCwd }
foreach ($k in 'Visible','FreshStart') { if ((Get-Variable $k).Value) { $fanoutArgs[$k] = $true } }
foreach ($k in 'LayoutPath') { if ((Get-Variable $k).Value) { $fanoutArgs[$k] = (Get-Variable $k).Value } }
foreach ($k in 'CanvasWidth','CanvasHeight') { if ((Get-Variable $k).Value -gt 0) { $fanoutArgs[$k] = (Get-Variable $k).Value } }
$prov = & (Join-Path $FanoutDir 'fanout.ps1') @fanoutArgs

try {
    $connect = Get-Content -LiteralPath $prov.ManifestPath -Raw | ConvertFrom-Json
    $live = Get-GridPaneInfo -Psmux $psmux -Session $Session
    Test-ConnectManifestLive -Manifest $connect -LivePaneIds @($live.Keys) -ExpectedCount @($Panes).Count

    $records = @(); $i = 0
    foreach ($mp in $connect.Panes) {
        $spec = $Panes[$i]
        $records += [pscustomobject]@{
            Index=$i; PaneId=$mp.PaneId; PanePid=$live[$mp.PaneId].PanePid
            Harness=$spec.Harness; Model=$spec.Model; Effort=$spec.Effort
            Status='pending'; Title=("{0}-{1}" -f $spec.Harness, ($spec.Model ?? 'default'))
            NeedsSetup=($registry[$spec.Harness].quirks -contains 'session-purpose-first')
            LaunchedAt=[datetime]::MinValue; Relaunched=$false
        }
        $i++
    }
    $gmPath = Get-GridManifestPath -Session $Session
    Write-GridManifest -Path $gmPath -State building -Session $Session -BuildId $connect.BuildId -Panes $records

    if (@($records | Where-Object { $registry[$_.Harness].quirks -contains 'extended-keys' })) {
        Invoke-Psmux -Psmux $psmux -ArgList @('set-option','-t',$Session,'extended-keys','on') | Out-Null
    }
    foreach ($r in $records) { Send-PaneCommand -Psmux $psmux -Session $Session -PaneId $r.PaneId -Text 'cls' }
    foreach ($r in $records) {
        Start-PaneHarness -Psmux $psmux -Session $Session -Pane $r -Def $registry[$r.Harness]
        Start-Sleep -Milliseconds $StaggerMs
    }
    Wait-HarnessReady -Psmux $psmux -Session $Session -Panes $records -Registry $registry -TimeoutSec $TimeoutSec
    Write-GridManifest -Path $gmPath -State up -Session $Session -BuildId $connect.BuildId -Panes $records
    return $records
} catch {
    if (-not $KeepPartial) {
        Write-Warning "gridup failed mid-flight: $($_.Exception.Message) — tearing down session '$Session'"
        & (Join-Path $PSScriptRoot 'griddown.ps1') -Session $Session | Out-Null
    }
    throw
}
```

(Note: `griddown.ps1` exists after Task 6 — until then the catch block is exercised only by Task 6's tests; Gridup tests in this task use happy paths and pre-provision validation failures, which throw before the try block.)

- [ ] **Step 4: Run to verify pass** — all 3 `It`s green.
- [ ] **Step 5: Commit** — `git add gridup.ps1 tests/Gridup.Tests.ps1 && git commit -m "feat: gridup entry - provision, cls pass, staggered launches, readiness, manifests"`

---

### Task 6: Teardown (`_teardown-lib.ps1` + `griddown.ps1`)

**Files:**
- Create: `~\.claude\skills\psmux-harness-grid\_teardown-lib.ps1`
- Create: `~\.claude\skills\psmux-harness-grid\griddown.ps1`
- Test: `~\.claude\skills\psmux-harness-grid\tests\Teardown.Tests.ps1`

**Interfaces:**
- Consumes: Task 2 (`Get-GridManifestPath`, `Read-GridManifest`); fanout `cleanup.ps1` (params: `-Session`, `-NoReap`, `-DryRun`).
- Produces: `Get-GridProcessSnapshot([int[]]$PanePids) -> object[]` — descendants of the pane PIDs via one `Get-CimInstance Win32_Process` sweep + parent-chain walk; each `@{ProcessId;CreationDate;ExecutablePath;Name}`. `Invoke-GridReap([object[]]$Snapshot) -> object[]` — per-PID outcome `@{ProcessId;Name;Outcome}` with `Outcome ∈ killed|already-exited|pid-reused-skipped|kill-failed`; revalidates `CreationDate` (and `ExecutablePath` when both readable) immediately before `Stop-Process`. `griddown.ps1` params: `-Session 'fanout'`, `-DryRun`.

griddown flow (exactly spec "Teardown"): read grid-manifest (missing manifest → warn, still run cleanup) → snapshot descendants of recorded PanePids → `cleanup.ps1 -Session <s> -NoReap` → `Invoke-GridReap` on surviving snapshot → `Write-GridManifest -State torn-down` → report free RAM + warn if `Get-Process psmux` non-empty. Idempotent: second run finds no manifest panes alive / empty snapshot and completes cleanly.

- [ ] **Step 1: Write failing tests** — `tests\Teardown.Tests.ps1`:
  - Unit (`Get-GridProcessSnapshot`): start a local `pwsh -NoProfile -Command Start-Sleep 300` child, snapshot on `$PID`... no — snapshot on a *sacrificial parent*: start `pwsh -Command "pwsh -NoProfile -Command Start-Sleep 300"`, snapshot the outer PID, assert the inner pwsh appears with CreationDate + path. Kill both in `AfterEach`.
  - Unit (`Invoke-GridReap` revalidation): snapshot the sleeper, `Stop-Process` it manually, hand-craft a snapshot entry reusing that (now dead) PID with a **wrong** CreationDate matching no live process → outcome `already-exited`; then craft an entry whose PID is a live *unrelated* process (`$PID` of the test run) but CreationDate deliberately wrong → outcome `pid-reused-skipped` and the test process survives.
  - Integration (the review counterexample): build a 2-pane fake-harness grid via `gridup.ps1` (fake registry), start an **outside** decoy `pwsh -NoProfile -Command "Start-Sleep 300"` *after* grid-up, run `griddown.ps1`, assert: grid's fake harness processes gone, decoy still alive (kill decoy in cleanup), session gone, `grid-manifest.json` `State='torn-down'`.
  - Idempotency: run `griddown.ps1` again immediately → completes, no throw.
  - Mid-launch death: run `gridup.ps1` with the `fake-never` registry and `-TimeoutSec 5 -KeepPartial`, then delete nothing — kill the session's window ourselves? No: simulate by calling `griddown.ps1` directly while manifest still says `building` → teardown consumes the early manifest and reaps.

- [ ] **Step 2: Run to verify failure** — FAIL (libs missing).

- [ ] **Step 3: Implement** — `_teardown-lib.ps1`:

```powershell
#requires -Version 7.1
. (Join-Path $PSScriptRoot '_grid-lib.ps1')

function Get-GridProcessSnapshot {
    [CmdletBinding()]
    param([Parameter(Mandatory)][AllowEmptyCollection()][int[]] $PanePids)
    $all = Get-CimInstance Win32_Process -ErrorAction SilentlyContinue |
           Select-Object ProcessId, ParentProcessId, CreationDate, ExecutablePath, Name
    $byParent = $all | Group-Object ParentProcessId -AsHashTable -AsString
    $seen = @{}; $queue = [System.Collections.Queue]::new()
    foreach ($p in $PanePids) { $queue.Enqueue([string]$p) }
    $snap = @()
    while ($queue.Count) {
        $pp = $queue.Dequeue()
        if ($seen.ContainsKey($pp)) { continue }; $seen[$pp] = $true
        foreach ($child in @($byParent[$pp])) {
            if ($null -eq $child) { continue }
            $snap += [pscustomobject]@{ ProcessId=[int]$child.ProcessId; CreationDate=$child.CreationDate
                                        ExecutablePath=$child.ExecutablePath; Name=$child.Name }
            $queue.Enqueue([string]$child.ProcessId)
        }
    }
    $snap
}

function Invoke-GridReap {
    [CmdletBinding()]
    param([AllowEmptyCollection()][object[]] $Snapshot = @(), [switch] $DryRun)
    $out = @()
    foreach ($s in $Snapshot) {
        $live = Get-CimInstance Win32_Process -Filter "ProcessId=$($s.ProcessId)" -ErrorAction SilentlyContinue
        $outcome =
            if (-not $live) { 'already-exited' }
            elseif ("$($live.CreationDate)" -ne "$($s.CreationDate)" -or
                    ($live.ExecutablePath -and $s.ExecutablePath -and $live.ExecutablePath -ne $s.ExecutablePath)) {
                'pid-reused-skipped'
            } elseif ($DryRun) { 'killed' }   # what WOULD happen
            else {
                try { Stop-Process -Id $s.ProcessId -Force -ErrorAction Stop; 'killed' }
                catch { 'kill-failed' }
            }
        $out += [pscustomobject]@{ ProcessId=$s.ProcessId; Name=$s.Name; Outcome=$outcome }
    }
    $out
}
```

`griddown.ps1`:

```powershell
#requires -Version 7.1
[CmdletBinding()]
param([string] $Session = 'fanout', [switch] $DryRun)
$ErrorActionPreference = 'Stop'
. (Join-Path $PSScriptRoot '_teardown-lib.ps1')
$FanoutDir = Join-Path (Split-Path $PSScriptRoot -Parent) 'psmux-fanout'
$env:PSMUX_ALLOW_ELEVATED = '1'

$gmPath = Get-GridManifestPath -Session $Session
$snapshot = @()
$manifest = $null
if (Test-Path -LiteralPath $gmPath) {
    $manifest = Read-GridManifest -Path $gmPath
    $panePids = @($manifest.Panes | ForEach-Object { [int]$_.PanePid } | Where-Object { $_ -gt 0 })
    $snapshot = @(Get-GridProcessSnapshot -PanePids $panePids)   # BEFORE the session dies
} else {
    Write-Warning "no grid-manifest for session '$Session' — running session cleanup only"
}

& (Join-Path $FanoutDir 'cleanup.ps1') -Session $Session -NoReap:$true -DryRun:$DryRun | Out-Null
$reap = Invoke-GridReap -Snapshot $snapshot -DryRun:$DryRun

if ($manifest -and -not $DryRun) {
    Write-GridManifest -Path $gmPath -State 'torn-down' -Session $Session -BuildId $manifest.BuildId -Panes @($manifest.Panes)
}
$freeGb = (Get-CimInstance Win32_OperatingSystem).FreePhysicalMemory / 1MB
$psmuxLeft = @(Get-Process -Name psmux -ErrorAction SilentlyContinue)
if ($psmuxLeft) { Write-Warning ("psmux.exe still running: {0}" -f ($psmuxLeft.Id -join ',')) }
[pscustomobject]@{ Session=$Session; Reaped=$reap; FreeGb=[math]::Round($freeGb,2)
                   PsmuxRemaining=@($psmuxLeft.Id); DryRun=[bool]$DryRun }
```

- [ ] **Step 4: Run to verify pass** — all Teardown tests green (decoy survives, grid procs die, idempotent).
- [ ] **Step 5: Re-run Task 5's Gridup tests** — the failure-cleanup catch block now resolves `griddown.ps1`; confirm no regression.
- [ ] **Step 6: Commit** — `git add _teardown-lib.ps1 griddown.ps1 tests/Teardown.Tests.ps1 && git commit -m "feat: snapshot-based teardown with identity revalidation, idempotent griddown"`

---

### Task 7: `Swap-PaneHarness`

**Files:**
- Modify: `~\.claude\skills\psmux-harness-grid\_harness-lib.ps1` (append function)
- Test: `~\.claude\skills\psmux-harness-grid\tests\Swap.Tests.ps1`

**Interfaces:**
- Consumes: Tasks 1–4, 2 (manifest).
- Produces: `Swap-PaneHarness([string]$Psmux,[string]$Session,[string]$PaneId,[string]$Harness,[string]$Model,[string]$Effort,[hashtable]$Registry,[int]$TimeoutSec=90) -> pscustomobject` (the updated pane record; also rewrites `grid-manifest.json`).

Flow (spec "Swap"): look up current pane record in grid-manifest (throw if absent) → send old def's `exitCommand` via `Send-PaneCommand`; if none, `Send-PaneKey ... 'C-c'` twice with 500ms gap → poll `Get-GridPaneInfo` up to 15s for `CurrentCommand -match ShellPattern`; **abort (throw) if never** — never launch into an occupied pane → `Send-PaneCommand 'cls'` → build fresh record (`Relaunched=$false`, new Harness/Model/Effort/Title, same PaneId/PanePid/Index) → `Start-PaneHarness` + single-pane `Wait-HarnessReady` → rewrite manifest with the replaced record → return it.

- [ ] **Step 1: Write failing tests** — `tests\Swap.Tests.ps1` (skip guard): build 1-pane fake grid via `gridup.ps1`; swap `fake-ready` → `fake-ready` (assert new Status `ready`, manifest shows swap); swap where old harness ignores exit (use `fake-never` as the OLD harness — its pwsh sleeps, `/exit` typed into `Start-Sleep` does nothing... use a fake whose exitCommand is empty so C-c C-c interrupts the sleep → shell returns — assert success); and an abort case: temporarily make exit undetectable by giving the old fake a nested `pwsh -Command pwsh` double layer with exitCommand='' and TimeoutSec tiny? Simplest deterministic abort: pass `-TimeoutSec` small and an old fake that traps C-c (`-Command "[console]::TreatControlCAsInput=$true; Start-Sleep 300"`) → expect throw `*still occupied*` and no launch.

- [ ] **Step 2: Run to verify failure** — FAIL (function missing).
- [ ] **Step 3: Implement** (append to `_harness-lib.ps1`):

```powershell
function Swap-PaneHarness {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Psmux, [Parameter(Mandatory)][string]$Session,
          [Parameter(Mandatory)][string]$PaneId, [Parameter(Mandatory)][string]$Harness,
          [string]$Model, [string]$Effort,
          [Parameter(Mandatory)][hashtable]$Registry, [int]$TimeoutSec = 90)
    if (-not $Registry.ContainsKey($Harness)) { throw "unknown harness '$Harness'" }
    $gmPath = Get-GridManifestPath -Session $Session
    $manifest = Read-GridManifest -Path $gmPath
    $old = @($manifest.Panes) | Where-Object { $_.PaneId -eq $PaneId }
    if (-not $old) { throw "pane $PaneId not in grid manifest for '$Session'" }
    $oldDef = $Registry[$old.Harness]

    if ($oldDef -and $oldDef.exitCommand) {
        Send-PaneCommand -Psmux $Psmux -Session $Session -PaneId $PaneId -Text $oldDef.exitCommand
    } else {
        Send-PaneKey -Psmux $Psmux -Session $Session -PaneId $PaneId -Key 'C-c'
        Start-Sleep -Milliseconds 500
        Send-PaneKey -Psmux $Psmux -Session $Session -PaneId $PaneId -Key 'C-c'
    }
    $deadline = (Get-Date).AddSeconds(15); $atShell = $false
    while ((Get-Date) -lt $deadline) {
        $info = Get-GridPaneInfo -Psmux $Psmux -Session $Session
        if ($info.ContainsKey($PaneId) -and $info[$PaneId].CurrentCommand -match $script:ShellPattern) { $atShell = $true; break }
        Start-Sleep -Milliseconds 1000
    }
    if (-not $atShell) { throw "pane $PaneId is still occupied after exit attempt — aborting swap (never launch into an occupied pane)" }

    Send-PaneCommand -Psmux $Psmux -Session $Session -PaneId $PaneId -Text 'cls'
    $rec = [pscustomobject]@{
        Index=$old.Index; PaneId=$old.PaneId; PanePid=$old.PanePid
        Harness=$Harness; Model=$Model; Effort=$Effort; Status='pending'
        Title=("{0}-{1}" -f $Harness, ($Model ?? 'default'))
        NeedsSetup=($Registry[$Harness].quirks -contains 'session-purpose-first')
        LaunchedAt=[datetime]::MinValue; Relaunched=$false
    }
    Start-PaneHarness -Psmux $Psmux -Session $Session -Pane $rec -Def $Registry[$Harness]
    Wait-HarnessReady -Psmux $Psmux -Session $Session -Panes @($rec) -Registry $Registry -TimeoutSec $TimeoutSec
    $newPanes = @($manifest.Panes | Where-Object { $_.PaneId -ne $PaneId }) + $rec | Sort-Object Index
    Write-GridManifest -Path $gmPath -State $manifest.State -Session $Session -BuildId $manifest.BuildId -Panes $newPanes
    $rec
}
```

- [ ] **Step 4: Run to verify pass**, then re-run ALL suites: `Invoke-Pester '$env:USERPROFILE\.claude\skills\psmux-harness-grid\tests' -Output Detailed` — everything green.
- [ ] **Step 5: Commit** — `git add _harness-lib.ps1 tests/Swap.Tests.ps1 && git commit -m "feat: Swap-PaneHarness with occupied-pane abort"`

---

### Task 8: `SKILL.md` + live smoke + memory pointer

**Files:**
- Create: `~\.claude\skills\psmux-harness-grid\SKILL.md`
- Create: `~\.claude\skills\psmux-harness-grid\tests\Live.Tests.ps1` (opt-in, `-Tag Live`)
- Modify: `C:\Users\Tony Baloney\.claude\projects\K--Downloads---Projects-Mine-psmuxPlus\memory\psmux-harness-injection-gotchas.md` (one line: pattern now productized in the skill)

**Interfaces:** none new — documentation + verification.

- [ ] **Step 1: Write `SKILL.md`** — frontmatter `name: psmux-harness-grid`, description: "Stand up a visible psmux grid with a named agent harness (claude/codex/agy/omp) launched, verified ready, in each pane; swap harnesses mid-session; tear down precisely. Composes psmux-fanout (build) and leaves injection to psmux-connector. Use when the user wants multiple different agent CLIs running in visible panes." Body sections: When to use vs fanout/connector (table); Quick start (the spec's call shape + `-Spec` shorthand + griddown example); Parameters; Adding a harness (registry fields + quirk table from spec); Gotchas (readiness patterns go stale when a harness updates its banner — how to recapture; `-FreshStart` kills ALL sessions; `NeedsSetup` panes like pi need manual steps; capture flattening; the agy `<<TAG>>`→`<>` note for downstream drivers). Every gotcha copied from spec, not paraphrased from memory.
- [ ] **Step 2: Write `Live.Tests.ps1`** — `-Tag Live`, skipped by default (`Invoke-Pester -Tag Live` to run): builds `gridup.ps1 -Spec 'claude:claude-haiku-4-5, omp' -Visible -FreshStart`, asserts both `ready`, prints pane captures for eyeballing, `griddown.ps1`, asserts zero stray claude/omp from the snapshot outcomes. (Cheap models only; 2 panes; this is the harness-reality check no fake can give.)
- [ ] **Step 3: Run the full suite once more** — `Invoke-Pester tests -Output Detailed` (Live excluded by default): all green.
- [ ] **Step 4: Run the live smoke ONCE manually** — `Invoke-Pester tests\Live.Tests.ps1 -Tag Live` with the visible window up; verify by eye; then confirm teardown left `Get-Process psmux` empty and free-RAM reported.
- [ ] **Step 5: Update the memory file** — append to `psmux-harness-injection-gotchas.md`: "The launch/readiness/teardown half of this pattern is now productized in the psmux-harness-grid skill (~\.claude\skills\psmux-harness-grid); briefing/broadcast remains manual/connector."
- [ ] **Step 6: Commit** — `git add SKILL.md tests/Live.Tests.ps1 && git commit -m "docs: SKILL.md + opt-in live smoke test"`

---

## Self-Review (completed)

1. **Spec coverage:** registry/validation → T1; spec parsing, manifests, liveness → T2; targeting/duplicate-%id → T3; launch, CLS, readiness, quirks, statuses → T4/T5; RAM pre-flight, extended-keys, early manifest, failure cleanup → T5; teardown snapshot/revalidation/idempotency/counterexample, mid-launch death → T6; swap + occupied-abort → T7; SKILL.md, live smoke (incl. pi-pattern capture opportunity), out-of-scope notes → T8. No gaps found.
2. **Placeholder scan:** clean — every step names files and shows code or exact assertions; Task 4 Step 1 and Task 5 Step 1 describe tests by exact scenario + expected status rather than full listings, with all referenced fakes fully specified in the task preamble.
3. **Type consistency:** pane record shape (`Index,PaneId,PanePid,Harness,Model,Effort,Status,Title,NeedsSetup,LaunchedAt,Relaunched`) identical across T4/T5/T7; statuses match the Global Constraints set; `Invoke-Psmux`/`Get-GridPaneInfo`/`Send-PaneCommand` signatures consistent across consumers.
