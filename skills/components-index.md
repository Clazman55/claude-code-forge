# Components Index -- Reusable Implementations

A catalog of battle-tested implementations discovered across your projects. Not a library -- no runtime dependencies. Each project owns its copy. This index says "this problem has been solved; read this implementation before writing from scratch."

Use this file to track where your best version of each pattern lives. When you solve a problem well, add it here so future projects can reference the implementation instead of reinventing it.

---

## Maintenance Rule

The index always points to the **best current version** of each component. "Best" means: most general, most edge cases handled, least project-specific coupling.

**Promotion flow:** A project adapts an indexed component. If the adaptation is more general or more robust than the current pointer, the index is updated to point to the new version. The originating project's version is not changed.

**When to audit:** At stabilization and project wrap. The audit step asks: "Did this project adapt any indexed component? Is the adaptation an improvement? If yes, update the pointer."

**Direction is always upward.** Projects improve implementations. The index follows the improvements. No canonical copy is maintained -- the index points to real project files.

---

## PowerShell

### Core Infrastructure

| Component | Description | Best Version | Used In |
|-----------|-------------|-------------|---------|
| Self-elevation | `[Security.Principal.WindowsPrincipal]` check + `-Verb RunAs` relaunch | `<project-root>/MainScript.ps1` | Admin tool with UAC requirement |
| PS 5.1 bootstrap launcher | `.bat` sets execution policy + `Bootstrap.ps1` checks/installs PS7 via winget, relaunches under pwsh | `<project-root>/Run-Tool.bat` + `<project-root>/Bootstrap.ps1` | Deployment tool targeting mixed PS environments |
| JSON config merge | `defaults.json` + optional `user-overrides.json`, merged at runtime (overrides win) | `<project-root>/Functions/` (config loading module) | Config-driven automation tool |
| Logging infrastructure (dual-output) | In-memory JSON accumulator + text log, Desktop target, UI push via global context, flush-once design (avoids O(n^2) per-entry writes) | `<project-root>/Functions/Write-Log.ps1` | GUI automation tool |
| Release packaging | Explicit include list, staging verification, dev-leak check, module import test, ZIP + SHA256 checksums | `<project-root>/Build-Release.ps1` | Any project with a release deliverable. See `powershell-release-build.md`. |

### WPF / GUI Patterns

| Component | Description | Best Version | Used In |
|-----------|-------------|-------------|---------|
| WPF window hosting | XAML runtime load + global context pattern for C# delegate scope | `<project-root>/src/GUI/MainWindow.ps1` | WPF desktop tool. See `powershell-wpf.md`. |
| WPF async runspace | Background `PowerShell.Create()` + `DispatcherTimer` polling for UI-safe progress | `<project-root>/src/GUI/MainWindow.ps1` | Tool with long-running background operations |
| WPF tick-based pipeline | `DispatcherTimer` one-item-per-tick execution -- UI updates between ticks, no runspace needed for sequential work | `<project-root>/Functions/Show-UI.ps1` | Step-by-step pipeline with live UI feedback |
| WPF SortGroup system | Numeric-prefixed string (`1-Category`) for visual group ordering, C# `IValueConverter` strips prefix for display | `<project-root>/Functions/Show-UI.ps1` | Categorized list/grid UI with dynamic grouping |

### File Operations

| Component | Description | Best Version | Used In |
|-----------|-------------|-------------|---------|
| Robocopy wrapper | File + folder copy with `/Z` `/R:3` `/W:5`, exit code handling (0-7 ok, 8-15 warn, 16+ fail) | `<project-root>/src/Core/BackupEngine.ps1` | Backup/sync utility |
| File hash verification | `Get-FileHash` wrapper, source/dest comparison, MD5/SHA256 algorithm selection | `<project-root>/src/Core/Verification.ps1` | Backup utility with integrity checks |
| Timestamped version rotation | Create version copy before overwrite, prune by count, files + folders | `<project-root>/src/Core/Versioning.ps1` | Any tool with file versioning |

### Software / System Management

| Component | Description | Best Version | Used In |
|-----------|-------------|-------------|---------|
| Windows Update cycle | PSWindowsUpdate-based enumerate+install+reboot-detect, cycle limit via state file, TLS 1.2 enforcement, NuGet fallback bootstrap, dual reboot detection | `<project-root>/Functions/Invoke-UpdateCycle.ps1` | Automated patching tool |
| Reboot recovery (RunOnce) | `HKLM:\...\RunOnce` registry key -- fires after user logon in interactive session, 15s delay for desktop shell, auto-deletes after execution | `<project-root>/Functions/Register-ResumeTask.ps1` | Multi-reboot automation (GUI apps) |
| winget installation | `winget install --exact --silent`, path resolution (PATH, LocalAppData, ProgramFiles), exit -1978335189 = already installed, `--source winget` to skip msstore cert errors | `<project-root>/Functions/Install-Software.ps1` | Software deployment tool |
| Power config | powercfg commands, NIC power management, USB selective suspend, laptop/desktop detection via Win32_Battery | `<project-root>/Functions/Set-PowerConfig.ps1` | PC provisioning or setup tool |

### Testing

| Component | Description | Best Version | Used In |
|-----------|-------------|-------------|---------|
| Invoke-QualityCheck.ps1 | Automated quality gate -- PSScriptAnalyzer, baseline violations, function standards (param block, error handling, help), hardcoded path detection, Pester test execution, summary with pass/fail | `<project-root>/tests/Invoke-QualityCheck.ps1` | Any PowerShell project. Adapt checks to your standards. |
| Pester test scaffolding | Per-module `.Tests.ps1` files with `Describe`/`Context`/`It` structure, mock helpers, function-level coverage matching source modules | `<project-root>/tests/` | Any PowerShell project. See `pester-testing-patterns.md`. |
| TestingGuide.html template | Self-contained manual testing checklist -- collapsible sections, checkbox + `<label for>` pattern, localStorage persistence, Copy Report with `execCommand` fallback, Reset All with confirm, progress counters | `<project-root>/tests/TestingGuide.html` | Any project with manual test steps |

### Reporting

| Component | Description | Best Version | Used In |
|-----------|-------------|-------------|---------|
| HTML report generation | StringBuilder-based, dark theme CSS, collapsible sections, copy-to-clipboard, JSON data embedding | `<project-root>/src/Report/Build-Report.ps1` | Data collection tool with HTML output. See `powershell-html-report.md`. |
| JSON data export | Serializes collected data to JSON with `ConvertTo-Json -Depth 10`, consistent naming with HTML output | `<project-root>/src/Report/Export-Json.ps1` | Any tool that exports structured data |

---

## HTML5 / JavaScript

### Game / Application Infrastructure

| Component | Description | Best Version | Used In |
|-----------|-------------|-------------|---------|
| esbuild bundler | `build.ps1` -- standalone esbuild, ES6 modules to single IIFE bundle, CSS inlining, UTF-8 handling | `<project-root>/build.ps1` | Multi-file JS project with single-file dist output |
| Fixed-timestep game loop | rAF + accumulator + panic brake + fractional tick | `<project-root>/src/engine.js` | HTML5 Canvas game or simulation |
| Save/load with version migration | localStorage + JSON + version chain with per-version migration functions | `<project-root>/src/save.js` | Any browser app with persistent state |
| Tab-based UI with dirty flags | DOM tab switching, per-tab render functions, change-guarded writes | `<project-root>/src/render/render-core.js` | Multi-panel browser UI |
| Performance overlay | F3 toggle, EMA FPS, tick timing, render timing | `<project-root>/Game.html` (inline) | HTML5 game with debug tooling |

---

## Notes

- File paths point to the best current version. If a project adapts a component and the adaptation is better, the path gets updated at stabilize/wrap.
- "Used In" describes the project type where the component was verified. It does not mean the component is limited to that context.
- For pattern-level guidance (how vs where), see the corresponding skill file. The components index covers implementations; skills cover patterns.
- **Add your own:** When you build a reusable implementation, add it to the appropriate category. If no category fits, create a new one.
