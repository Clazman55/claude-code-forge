# [Example Project] -- PC Provisioning Tool

> **Note:** This is a redacted example from a real shipped project. It demonstrates the expected depth and detail of a project CLAUDE.md after 6 completed phases. Project-specific names and paths have been replaced, but the structure and technical content are authentic.

## Description
PowerShell 7 tool with WPF GUI for automated PC provisioning -- bloatware removal, software installation, power configuration, and Windows Update sequencing.

## Tech Stack
- PowerShell 7+ (primary runtime)
- PowerShell 5.1 (bootstrap layer only)
- WPF (XAML-based GUI)
- PSWindowsUpdate module (Windows Update sequencing)
- Office Deployment Tool (Office language pack removal)
- winget (software installation)
- Target OS: Windows 11 Home/Pro (Dell OEM)

## Phase Status

| Phase | Description | Status |
|-------|-------------|--------|
| 0 | Research, Plan, Scaffold | Complete |
| 1 | Bootstrap + Core Engine | Complete |
| 2 | WPF UI | Complete |
| 3 | Removal Pipelines | Complete |
| 4 | Install + Power Config | Complete |
| 5 | Windows Update Integration | Complete |
| 6 | Resume, Polish, Testing, Release | Complete |

## Key File Paths
```
Run-Provision.bat          # Execution policy bypass launcher
Bootstrap.ps1              # PS 5.1 -- install PS7 if missing, relaunch
Invoke-Provision.ps1       # PS 7 -- main entry point, config merge, orchestrator
Config\
  defaults.json            # Bloatware database, install list, power/update settings
  user-overrides.json      # Tech-editable overrides (optional, merged at runtime)
Functions\
  Write-ProvisionLog.ps1        # [P1] Logging -- in-memory JSON accumulator
  Import-ProvisionConfig.ps1    # [P1] Config loading, merge, validation
  Get-InstalledSoftware.ps1     # [P1] Win32 + AppX scanning + bloatware matching
  New-ProvisionManifest.ps1     # [P1] Manifest generation, persistence, resume detection
  Remove-Win32App.ps1           # [P3] MSI/EXE uninstall with silent flag cycling
  Remove-AppxApp.ps1            # [P3] AppX + provisioned package removal
  Remove-OfficeLangPack.ps1     # [P3] ODT-based Office language pack removal
  Invoke-ProvisionPipeline.ps1  # [P3] Tick-based pipeline orchestrator
  Install-Software.ps1          # [P4] winget installation pipeline + path resolution
  Set-PowerConfig.ps1           # [P4] Power plan, timeouts, hibernate, USB/NIC
  Invoke-UpdateCycle.ps1        # [P5] PSWindowsUpdate cycle, state file, reboot detection
  Register-ResumeTask.ps1       # [P5+P6] RunOnce registry key for reboot recovery
  Show-ProvisionUI.ps1          # [P2+P3+P6] WPF window, MVVM, SortGroup, pipeline
Manifests\                 # Runtime -- generated job manifests (JSON)
Logs\                      # Runtime -- per-run log files
Build-Release.ps1          # Release packaging script
tests\
  TestHelpers.ps1               # [P3] Shared test setup
  Invoke-QualityCheck.ps1       # Quality gate -- all static + Pester checks
  Write-ProvisionLog.Tests.ps1  # [P1] 14 tests
  Scan-InstalledSoftware.Tests.ps1  # [P1] 20 tests
  New-ProvisionManifest.Tests.ps1   # [P1] 20 tests
  Invoke-Provision.Tests.ps1    # [P1] 16 tests
  Show-ProvisionUI.Tests.ps1    # [P2] 34 tests
  Remove-Win32App.Tests.ps1     # [P3] 27 tests
  Remove-AppxApp.Tests.ps1      # [P3] 6 tests
  Remove-OfficeLangPacks.Tests.ps1  # [P3] 6 tests
  Invoke-ProvisionPipeline.Tests.ps1 # [P3-P5] 20 tests
  Install-Software.Tests.ps1    # [P4] 7 tests
  Set-PowerConfig.Tests.ps1     # [P4] 14 tests
  Invoke-UpdateCycle.Tests.ps1  # [P5] 19 tests
  Register-ResumeTask.Tests.ps1 # [P5] 9 tests
  TestingGuide.html             # Manual test checklist (user-facing verification)
```

## Skills in Use
- powershell-baseline
- powershell-wpf
- powershell-module-architecture (patterns, not formal module)
- powershell-cim-wmi (registry scanning, hardware detection, NIC queries)
- pester-testing-patterns
- manual-testing-methodology
- project-lifecycle

## Architecture Notes

### Run Order
Suspend WU -> Win32 removal -> AppX removal -> Office lang pack removal -> Software install (winget) -> Power config -> Re-enable WU -> Update pass (optional)

### Config Strategy
- `defaults.json`: Ships with tool. Bloatware database with wildcard MatchPatterns for scan matching. PreChecked flag controls UI default state. Schema documented in the file's `_comment` field.
- `user-overrides.json`: Optional tech-editable layer. Not auto-created. Array keys replace defaults; scalar keys override; object keys merge one level deep.
- All app definitions are data-driven from JSON. No hardcoded names in code.

### Logging
- Dual output: human-readable `.log` + structured `.json`
- JSON entries accumulated in memory (`$script:LogEntries` List), flushed to disk once at `Complete-ProvisionLog`. Avoids O(n^2) read-modify-write per entry.
- Human-readable log auto-opens on completion
- Levels: Info, Warning, Error, Success (color-coded on console)
- Also pushes to UI log pane when `$global:ProvCtx` is active

### WPF UI
- C# types compiled via `Add-Type` at runtime: ViewModelBase, RelayCommand, ProvisionItem, MainViewModel, SortGroupNameConverter
- XAML inline in `Get-ProvisionXaml` function (no external .xaml file); takes `$GuiAssemblyName` param for `xmlns:local` CLR namespace mapping
- `$global:ProvCtx` hashtable for C# delegate scope access (per powershell-wpf skill Section 5)
- Dot-sourced functions stored as scriptblock refs in context -- invoked via `& $ctx.WriteLog` from closures
- Lookup dictionaries in ProvCtx for O(1) manifest resolution
- SortGroup system: numeric-prefixed string (`1-Category`) controls visual group order. SortGroupNameConverter (C# IValueConverter) strips prefix for display.
- Dynamic grouping: checking an item moves it between groups; view refresh deferred via `Dispatcher.BeginInvoke`
- DispatcherTimer tick-based pipeline execution (one item per tick, UI updates between ticks)

### Manifest System
- JSON manifest generated from scan matches + config before any execution
- Per-item schema: Name, Type, Method, Identifier, SilentArgs, Category, Selected, Status, Error
- Persisted to `Manifests\` after creation and after each item status change
- Resume detection: on launch, checks for in-progress manifests and marks interrupted items as Failed

### Two-Layer Launch
- Layer 1 (PS 5.1): `.bat` -> `Bootstrap.ps1` (install PS7 if needed, relaunch)
- Layer 2 (PS 7): `Invoke-Provision.ps1` (main tool, WPF, all pipelines)

### Removal Pipelines
- **Win32 (known methods):** QuietUninstall uses QuietUninstallString directly; MSI uses `msiexec /x {GUID} /qn /norestart`; EXE uses SilentArgs from database
- **Win32 (unknown method):** Flag cycling -- framework detection (Inno→`/VERYSILENT`, NSIS→`/S`, InstallShield→`/s`), then common flags. Timeout = hung dialog = mark Manual.
- **AppX:** `Remove-AppxPackage -AllUsers` + `Remove-AppxProvisionedPackage -Online`. Partial success treated as success with warning.
- **Office:** ODT with generated XML `<Remove>` block. Downloads setup.exe if needed. Verifies removal via registry.
- **Pipeline orchestration:** Single-pass queue build into category buckets. try/catch per item prevents single failure from stalling pipeline.

### Resume After Reboot
- Three resume paths: (1) update state file + manifest → auto-resume UI with update cycle, (2) manifest only → fresh scan + UI for tech review, (3) no manifest → normal fresh run
- RunOnce registry key for reboot recovery, 15s delay for desktop shell, auto-deletes after execution
- State file + RunOnce key cleaned up on completion or cycle limit

### Release Packaging
- `Build-Release.ps1` copies runtime files to `Release\` -- no tests, docs, git
- SHA256 checksums for deployment integrity verification

## Self-Verification Loop (per phase)

1. Phase kickoff -- `/kickoff`: read plan/code/memory, present gameplan (incl. `### Test Plan`) + questions tagged [Design] [Clarification] [Suggestion] [Risk], wait for user
2. Write code + automated tests (Pester) -- run tests, fix failures. Code is NOT presented for review until automated tests pass.
3. Update TestingGuide.html -- manual test steps for user-facing behavior (the operator's inspection, separate from automated tests)
4. User follows TestingGuide
5. User reports results
6. Fix failures
7. `/forge-review` -- three-agent review
8. Fix issues
9. Update CLAUDE.md -- `/phase-wrap` handles steps 9-11 + commit
10. Update project skills (docs/) with verified patterns -- canonical sync at wrap only
11. Update MEMORY.md + satellites, then next phase

### Testing Distinction
- **Automated tests (Pester):** Verify code correctness. The session's responsibility. Written alongside code, run before presenting for review. Every `Functions/*.ps1` gets a corresponding `tests/*.Tests.ps1`.
- **Manual tests (TestingGuide.html):** Verify user-facing behavior. The operator's inspection. Written after automated tests pass. Separate activity, separate owner.
- **Quality gate (Invoke-QualityCheck.ps1):** PSScriptAnalyzer + baseline violations + function standards + hardcoded paths + Pester. Must pass before presenting code.
