# PowerShell Module Architecture — Skill Reference

Patterns for building properly structured PowerShell 7+ module projects.
Extracted from real-world experience building a PowerShell backup utility (33 exported functions, 239 Pester tests, 10 sub-modules).

Companion to `powershell-baseline.md` (coding standards) and `powershell-wpf.md` (GUI patterns).

---

## Table of Contents

1. [Module Manifest (.psd1)](#1-module-manifest-psd1)
2. [Root Module (.psm1) — Dot-Sourcing](#2-root-module-psm1--dot-sourcing)
3. [Function Naming Conventions](#3-function-naming-conventions)
4. [CLI-First Architecture](#4-cli-first-architecture)
5. [Directory Structure](#5-directory-structure)
6. [JSON Configuration Externalization](#6-json-configuration-externalization)
7. [Config Getter/Setter Pattern](#7-config-gettersetter-pattern)
8. [Config Validation Pattern](#8-config-validation-pattern)
9. [Config Migration / Backwards Compatibility](#9-config-migration--backwards-compatibility)
10. [Module-Scoped State](#10-module-scoped-state)
11. [Centralized Logging Pattern](#11-centralized-logging-pattern)
12. [Dependency Management](#12-dependency-management)
13. [PS 5.1 Bootstrap Launcher](#13-ps-51-bootstrap-launcher)
14. [Quality Gate Pattern](#14-quality-gate-pattern)
15. [Self-Verification Loop](#15-self-verification-loop)
16. [Release Packaging](#16-release-packaging)
17. [Gotchas](#17-gotchas)
18. [New Module Checklist](#18-new-module-checklist)

---

## 1. Module Manifest (.psd1)

The manifest declares your module's metadata, version, and public API surface.

### Creating a Manifest

```powershell
New-ModuleManifest -Path src/MyModule.psd1 `
    -RootModule 'MyModule.psm1' `
    -ModuleVersion '0.1.0' `
    -Author 'YourName' `
    -Description 'Short module description' `
    -PowerShellVersion '7.0' `
    -FunctionsToExport @('Get-XXConfig', 'Set-XXConfig')
```

### Key Fields

```powershell
@{
    RootModule        = 'MyModule.psm1'
    ModuleVersion     = '1.0.0'
    GUID              = '7889293d-2d7c-4323-9157-14515235eaeb'
    Author            = 'YourName'
    CompanyName       = 'YourCompany'
    Description       = 'Module description'
    PowerShellVersion = '7.0'

    # CRITICAL: Explicit list — NEVER use '*'
    FunctionsToExport = 'Get-XXConfig', 'Set-XXConfig', 'New-XXConfig'

    # Always declare even if empty
    CmdletsToExport   = @()
    AliasesToExport    = @()
}
```

### Anti-Pattern: Wildcard Export

```powershell
# BAD — exposes all internal helpers, breaks encapsulation, hurts performance
FunctionsToExport = '*'

# GOOD — explicit list, only public API
FunctionsToExport = 'Get-XXConfig', 'Set-XXConfig', 'New-XXConfig'
```

PowerShell uses `FunctionsToExport` for command discovery. Wildcards force scanning every function on every tab-complete.

---

## 2. Root Module (.psm1) — Dot-Sourcing

The root module dot-sources sub-modules in dependency order. Keep it minimal — just the load sequence.

```powershell
#Requires -Version 7.0

# MyModule — Root Module
# Dot-sources all sub-modules in dependency order.

# 1. Foundation (no dependencies)
. "$PSScriptRoot/Core/Configuration.ps1"
. "$PSScriptRoot/Core/Logging.ps1"

# 2. Core operations (depend on Configuration + Logging)
. "$PSScriptRoot/Core/Verification.ps1"
. "$PSScriptRoot/Core/Versioning.ps1"
. "$PSScriptRoot/Core/ProcessManager.ps1"
. "$PSScriptRoot/Core/BackupEngine.ps1"

# 3. Supporting features (depend on Configuration + Logging)
. "$PSScriptRoot/Email/EmailReport.ps1"
. "$PSScriptRoot/Scheduling/TaskManager.ps1"
. "$PSScriptRoot/Migration/UserDataMigration.ps1"

# 4. GUI entry point (loads XAML + ViewModel on demand)
. "$PSScriptRoot/GUI/MainWindow.ps1"
```

### Rules

- **Foundation first** — Configuration and Logging have no dependencies, load them before anything else
- **Dependency order** — if BackupEngine calls Verification, load Verification first
- **`$PSScriptRoot` always** — never hardcode paths
- **No logic in .psm1** — just dot-source statements and maybe `#Requires`
- **Comment the layers** — makes the dependency graph visible at a glance

### Anti-Pattern: Glob-Based Loading

```powershell
# BAD — load order is undefined, dependencies may break
Get-ChildItem -Path "$PSScriptRoot/Core/*.ps1" | ForEach-Object { . $_.FullName }

# GOOD — explicit order, dependencies declared
. "$PSScriptRoot/Core/Configuration.ps1"  # must load before BackupEngine
. "$PSScriptRoot/Core/BackupEngine.ps1"   # depends on Configuration
```

---

## 3. Function Naming Conventions

### Project Prefix

Choose a 2-3 letter prefix for your project. ALL exported functions use `Verb-XXNoun`:

| Prefix | Project | Example |
|--------|---------|---------|
| AM | A Backup Module | `Get-AMConfig`, `Invoke-AMBackup` |
| BK | Backup Utils | `Start-BKSync`, `Test-BKConnection` |

### Rules

- **Approved verbs only** — run `Get-Verb` to check. Never invent verbs.
- **Prefix on all exports** — `Get-AMConfig`, not `Get-Config`
- **No prefix on internals** — helper functions that aren't exported don't need the prefix
- **Consistent noun casing** — `PascalCase` nouns always

### Common Verb Choices

| Action | Verb | Not |
|--------|------|-----|
| Read data | `Get-` | `Fetch-`, `Load-`, `Read-` |
| Write data | `Set-` | `Save-`, `Write-`, `Update-` |
| Create new | `New-` | `Create-`, `Make-`, `Add-` |
| Delete | `Remove-` | `Delete-`, `Destroy-`, `Clear-` |
| Run action | `Invoke-` | `Run-`, `Execute-`, `Do-` |
| Check validity | `Test-` | `Check-`, `Validate-`, `Verify-` |
| Install | `Install-` | `Setup-`, `Deploy-` |
| Start/Stop | `Start-`/`Stop-` | `Begin-`/`End-` |
| Send data | `Send-` | `Email-`, `Post-`, `Transmit-` |
| Display | `Show-` | `Display-`, `Open-`, `Launch-` |
| Format output | `Format-` | `Pretty-`, `Render-` |

---

## 4. CLI-First Architecture

Every feature works from the command line. The GUI is an optional convenience layer.

### Design Principle

```
CLI Layer (functions)          GUI Layer (optional)
┌──────────────────┐          ┌──────────────────┐
│ Get-AMConfig     │◄────────►│ Settings Tab     │
│ Invoke-AMBackup  │◄────────►│ Execute Button   │
│ Send-AMReport    │◄────────►│ Email Tab        │
│ Install-AMTask   │◄────────►│ Schedule Tab     │
│ New-AMMigration  │◄────────►│ Migration Tab    │
└──────────────────┘          └──────────────────┘
     Functions exist               GUI calls
     independently                 the functions
```

### Rules

- Design the function API first, then wire GUI to it
- Functions return objects, not formatted text
- GUI binds to function output via ViewModel
- Tests cover functions directly — no GUI required
- Headless/scheduled mode must work: `.\App.ps1 -RunBackup`

### Anti-Pattern: Logic in GUI

```powershell
# BAD — backup logic lives in the GUI button handler
$RunButton.Add_Click({
    robocopy $source $dest /MIR /R:3
    Send-MailMessage -To $admin ...
})

# GOOD — GUI calls CLI function
$RunButton.Add_Click({
    $result = Invoke-AMBackup
    if ($result.Success) { Send-AMReport -BackupSuccess $true }
})
```

---

## 5. Directory Structure

Standard layout for a PowerShell module project:

```
ProjectRoot/
├── MyModule.ps1                   # Entry point / PS 5.1 launcher
├── Build-Release.ps1              # Release packaging (dev-only)
├── CLAUDE.md                      # Project instructions
├── config/
│   ├── MyModule.json              # User config (created on first run)
│   └── MyModule.defaults.json     # Default config template (shipped)
├── src/
│   ├── MyModule.psd1              # Module manifest
│   ├── MyModule.psm1              # Root module (dot-sources sub-modules)
│   ├── Core/                      # Foundation + core operations
│   │   ├── Configuration.ps1
│   │   ├── Logging.ps1
│   │   └── ...
│   ├── Email/                     # Feature module
│   │   └── EmailReport.ps1
│   ├── Scheduling/                # Feature module
│   │   └── TaskManager.ps1
│   └── GUI/                       # WPF GUI (optional)
│       ├── MainWindow.ps1
│       ├── XAML/
│       ├── ViewModels/
│       └── Dialogs/
├── tests/
│   ├── Configuration.Tests.ps1    # One test file per source module
│   ├── Integration.Tests.ps1      # Cross-module integration tests
│   ├── Build.Tests.ps1            # Release packaging tests
│   └── Invoke-QualityCheck.ps1    # Automated quality gate
├── docs/                          # Documentation
├── logs/                          # Runtime log output (gitignored)
├── release/                       # Build output (gitignored)
└── resources/                     # Icons, images
```

### Conventions

- **src/** contains all runtime code
- **tests/** contains all test code + quality gate
- **config/** contains default config (shipped) + user config (created on first run)
- **docs/** contains documentation and skill files
- **One test file per source module** — `Configuration.Tests.ps1` tests `Configuration.ps1`
- **Separation of concerns** — Core/, Email/, GUI/ each have a single responsibility

---

## 6. JSON Configuration Externalization

Never store configuration in script variables. Externalize to JSON with a defaults + user override pattern.

### Defaults File (Shipped)

```json
{
  "general": {
    "logDirectory": "./logs",
    "logRetentionDays": 30
  },
  "backup": {
    "fileCopyJobs": [],
    "folderCopyJobs": [],
    "features": {
      "fileCopy": true,
      "folderCopy": true,
      "verification": true,
      "hashAlgorithm": "MD5"
    },
    "versioning": {
      "enabled": true,
      "maxVersions": 7,
      "timestampFormat": "yyyyMMdd_HHmmss"
    }
  },
  "email": {
    "enabled": false,
    "smtpServer": "",
    "smtpPort": 587,
    "useSsl": true,
    "from": "",
    "to": "",
    "username": "",
    "clientName": "",
    "credentialTarget": "MyModule_SMTP"
  }
}
```

### User Config (Created on First Run)

```powershell
function New-XXConfig {
    [CmdletBinding()]
    param([switch]$Force)

    $defaultsPath = Join-Path -Path $PSScriptRoot -ChildPath '../../config/MyModule.defaults.json'
    $configPath = Join-Path -Path $PSScriptRoot -ChildPath '../../config/MyModule.json'

    if ((Test-Path -LiteralPath $configPath) -and -not $Force) {
        Write-AMLog -Message "Config already exists at '$configPath'" -Level WARN
        return (Get-Content -LiteralPath $configPath -Raw -Encoding utf8 | ConvertFrom-Json)
    }

    $defaults = Get-Content -LiteralPath $defaultsPath -Raw -Encoding utf8
    Set-Content -LiteralPath $configPath -Value $defaults -Encoding utf8
    return ($defaults | ConvertFrom-Json)
}
```

### Anti-Pattern: Config in Script Headers

```powershell
# BAD — config values hardcoded in script
$smtpServer = "smtp.office365.com"
$emailTo = "admin@company.com"
$maxVersions = 7

# GOOD — externalized, one source of truth
$config = Get-XXConfig
$config.email.smtpServer    # from JSON file
```

---

## 7. Config Getter/Setter Pattern

### Getter with Caching

```powershell
$script:XXConfig = $null
$script:XXDefaultConfigPath = Join-Path -Path $PSScriptRoot -ChildPath '../../config/MyModule.json'

function Get-XXConfig {
    [CmdletBinding()]
    param(
        [Parameter()]
        [string]$ConfigPath,

        [Parameter()]
        [switch]$Force
    )

    $targetPath = if ($ConfigPath) { $ConfigPath } else { $script:XXDefaultConfigPath }

    if ($null -eq $script:XXConfig -or $Force -or $ConfigPath) {
        if (-not (Test-Path -LiteralPath $targetPath)) {
            throw "Config file not found: '$targetPath'. Run New-XXConfig first."
        }
        $loaded = Get-Content -LiteralPath $targetPath -Raw -Encoding utf8 | ConvertFrom-Json
        if (-not $ConfigPath) {
            $script:XXConfig = $loaded
        }
        return $loaded
    }
    $script:XXConfig
}
```

### Setter

```powershell
function Set-XXConfig {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)]
        [PSObject]$Config,

        [Parameter()]
        [string]$ConfigPath
    )

    $targetPath = if ($ConfigPath) { $ConfigPath } else { $script:XXDefaultConfigPath }

    if ($PSCmdlet.ShouldProcess($targetPath, 'Save configuration')) {
        $json = $Config | ConvertTo-Json -Depth 5
        Set-Content -LiteralPath $targetPath -Value $json -Encoding utf8
        if (-not $ConfigPath) {
            $script:XXConfig = $Config
        }
    }
}
```

### Key Points

- **Cache in `$script:`** — module-scoped, not global
- **`-Force` to reload** — bypasses cache for testing or after external edits
- **`-ConfigPath` for custom paths** — tests redirect to temp directory
- **Always `-Depth` on `ConvertTo-Json`** — PS 7 default is 2, nested objects become strings
- **Always `-Encoding utf8`** — explicit encoding on all file I/O

---

## 8. Config Validation Pattern

Return a structured result, not exceptions. Callers decide how to handle invalid configs.

```powershell
function Test-XXConfig {
    [CmdletBinding()]
    param(
        [Parameter()]
        [PSObject]$Config
    )

    $messages = [System.Collections.Generic.List[string]]::new()

    if ($null -eq $Config) {
        $messages.Add('No configuration loaded')
        return [PSCustomObject]@{ IsValid = $false; Messages = $messages }
    }

    # Validate general section
    if ($Config.general.logRetentionDays -lt 1) {
        $messages.Add('general.logRetentionDays must be >= 1')
    }

    # Validate backup features
    if ($Config.backup.features.hashAlgorithm -notin @('MD5', 'SHA256')) {
        $messages.Add("backup.features.hashAlgorithm must be 'MD5' or 'SHA256'")
    }

    # Validate email (only when enabled)
    if ($Config.email.enabled) {
        if ([string]::IsNullOrWhiteSpace($Config.email.smtpServer)) {
            $messages.Add('email.smtpServer is required when email is enabled')
        }
        if ([string]::IsNullOrWhiteSpace($Config.email.from)) {
            $messages.Add('email.from is required when email is enabled')
        }
        if ($Config.email.smtpPort -lt 1 -or $Config.email.smtpPort -gt 65535) {
            $messages.Add('email.smtpPort must be between 1 and 65535')
        }
    }

    # Validate job entries
    for ($i = 0; $i -lt @($Config.backup.fileCopyJobs).Count; $i++) {
        $job = $Config.backup.fileCopyJobs[$i]
        if ([string]::IsNullOrWhiteSpace($job.source)) {
            $messages.Add("backup.fileCopyJobs[$i].source is required")
        }
    }

    [PSCustomObject]@{
        IsValid  = ($messages.Count -eq 0)
        Messages = $messages
    }
}
```

### Usage

```powershell
$result = Test-XXConfig -Config $config
if (-not $result.IsValid) {
    $result.Messages | ForEach-Object { Write-AMLog -Message "Config: $_" -Level ERROR }
    throw "Invalid configuration: $($result.Messages.Count) issue(s) found"
}
```

---

## 9. Config Migration / Backwards Compatibility

When you add new config fields in later versions, existing user configs won't have them.

### Safe Field Access

```powershell
# Check if field exists before reading
$clientName = if ($config.email.PSObject.Properties['clientName']) {
    $config.email.clientName
} else {
    ''
}
```

### Merge Pattern (Preferred for Bulk Migration)

```powershell
function Merge-ConfigDefaults {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [PSObject]$UserConfig,
        [Parameter(Mandatory)] [PSObject]$Defaults
    )

    # Walk the defaults tree — add any missing fields to user config
    foreach ($prop in $Defaults.PSObject.Properties) {
        if (-not $UserConfig.PSObject.Properties[$prop.Name]) {
            $UserConfig | Add-Member -NotePropertyName $prop.Name -NotePropertyValue $prop.Value
        }
        elseif ($prop.Value -is [PSCustomObject] -and $UserConfig.($prop.Name) -is [PSCustomObject]) {
            Merge-ConfigDefaults -UserConfig $UserConfig.($prop.Name) -Defaults $prop.Value
        }
    }
}
```

### Rules

- **Never break existing configs** — additive changes only
- **Never remove fields** — deprecate and ignore instead
- **Load defaults, overlay user config** — missing fields get default values
- **Test with old config files** — verify migration works

---

## 10. Module-Scoped State

### Rules

- **`$script:` for all module state** — config cache, session data, temp paths
- **Never `$global:` for module state** — leaks across modules, breaks test isolation
- **Exception:** WPF C# delegate workaround requires `$global:` (see `powershell-wpf.md` Section 5)
- **Getter functions expose state** — consumers never touch `$script:` directly
- **Lazy initialization** — load on first access, not at module import

```powershell
# Module-scoped state
$script:Config = $null
$script:LogSession = $null

# Public getter — the only way consumers access state
function Get-XXConfig {
    if ($null -eq $script:Config) {
        $script:Config = Import-XXConfigFromDisk
    }
    $script:Config
}

# Test cleanup — InModuleScope resets state
# (see pester-testing-patterns.md for details)
```

### Anti-Pattern: Global State

```powershell
# BAD — pollutes session, breaks test isolation
$global:MyConfig = Get-Content config.json | ConvertFrom-Json

# GOOD — module-scoped, accessed via function
$script:Config = $null
function Get-XXConfig { ... }
```

---

## 11. Centralized Logging Pattern

One logging function used by every module. Dual output to file + console.

```powershell
function Write-XXLog {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string]$Message,

        [Parameter()]
        [ValidateSet('INFO', 'WARN', 'ERROR')]
        [string]$Level = 'INFO'
    )

    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logLine = "$timestamp`t[$Level]`t$Message"

    # Console output with color coding
    $color = switch ($Level) {
        'INFO'  { 'White' }
        'WARN'  { 'Yellow' }
        'ERROR' { 'Red' }
    }
    Write-Host $logLine -ForegroundColor $color

    # File output
    $config = Get-XXConfig
    $logDir = $config.general.logDirectory
    if (-not [System.IO.Path]::IsPathRooted($logDir)) {
        $logDir = Join-Path -Path $PSScriptRoot -ChildPath "../../$logDir"
    }
    $null = New-Item -Path $logDir -ItemType Directory -Force
    $logFile = Join-Path -Path $logDir -ChildPath "$(Get-Date -Format 'yyyy-MM-dd').log"
    Add-Content -LiteralPath $logFile -Value $logLine -Encoding utf8
}
```

### Log Session Initialization

```powershell
function Initialize-XXLogSession {
    [CmdletBinding()]
    param()

    Write-XXLog -Message ('=' * 60)
    Write-XXLog -Message "Session started: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')"
    Write-XXLog -Message "Module version: $((Get-Module MyModule).Version)"
    Write-XXLog -Message ('=' * 60)
}
```

### Log Rotation

```powershell
# Clean up logs older than retention period
$config = Get-XXConfig
$logDir = Resolve-Path $config.general.logDirectory
$cutoff = (Get-Date).AddDays(-$config.general.logRetentionDays)
Get-ChildItem -LiteralPath $logDir -Filter '*.log' |
    Where-Object { $_.LastWriteTime -lt $cutoff } |
    Remove-Item -Force -ErrorAction SilentlyContinue
```

---

## 12. Dependency Management

### Goal: Zero External Dependencies

Replace external modules with built-in alternatives:

| External Module | Built-In Replacement |
|----------------|---------------------|
| CredentialManager | DPAPI: `Export-Clixml` / `Import-Clixml` with `[PSCredential]` |
| PSScriptAnalyzer | Dev-only — check availability, skip gracefully if missing |
| Pester | Dev-only — test framework, not runtime dependency |

### DPAPI Credential Storage

```powershell
# Store credential (encrypted to current user on current machine)
function Set-XXEmailCredential {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [PSCredential]$Credential
    )

    $credPath = Join-Path $PSScriptRoot '../../config/smtp_credential.xml'
    $Credential | Export-Clixml -LiteralPath $credPath
}

# Retrieve credential
function Get-XXEmailCredential {
    $credPath = Join-Path $PSScriptRoot '../../config/smtp_credential.xml'
    if (Test-Path -LiteralPath $credPath) {
        Import-Clixml -LiteralPath $credPath
    }
}
```

### Portability Rules

- All paths relative to `$PSScriptRoot` — no `$env:USERPROFILE`, `$HOME`, or Desktop paths
- No `C:\Users\...` hardcoding
- Module works from any directory location (USB drive, network share, etc.)

---

## 13. PS 5.1 Bootstrap Launcher

Entry point that works in Windows PowerShell 5.1, detects PS 7, and relaunches under it.
Use `-SkipRelaunch` as an internal guard flag to prevent an infinite relaunch loop.

```powershell
<#
.SYNOPSIS
    PS 5.1 bootstrap launcher. Detects PowerShell 7 and relaunches under it.
    Run from any PowerShell version — it handles the rest.
.PARAMETER OutputPath
    Override the output directory. Passed through to the main function.
.PARAMETER SkipRelaunch
    Internal flag — set automatically when already running under pwsh. Do not pass manually.
#>
param(
    [string]$OutputPath,
    [switch]$SkipRelaunch
)

# Guard: already running under PS 7+ (or relaunched) — execute directly
if ($SkipRelaunch -or $PSVersionTable.PSVersion.Major -ge 7) {
    $manifest = Join-Path -Path $PSScriptRoot -ChildPath 'src\MyModule.psd1'
    Import-Module $manifest -Force

    $params = @{}
    if ($OutputPath) { $params.OutputPath = $OutputPath }

    $result = Invoke-XXMain @params

    Write-Host "Output: $($result.OutputPath)" -ForegroundColor Cyan
    Invoke-Item (Split-Path -Path $result.OutputPath -Parent)   # open folder in Explorer
    exit 0
}

# Running under PS 5.1 — find pwsh and relaunch
$pwsh = Get-Command pwsh -ErrorAction SilentlyContinue
if (-not $pwsh) {
    Write-Host 'PowerShell 7 is required but was not found.' -ForegroundColor Red
    Write-Host 'Install it from: https://aka.ms/powershell'  -ForegroundColor Yellow
    exit 1
}

$relaunchArgs = @('-NoProfile', '-ExecutionPolicy', 'Bypass', '-File',
                   $MyInvocation.MyCommand.Path, '-SkipRelaunch')
if ($OutputPath) { $relaunchArgs += '-OutputPath'; $relaunchArgs += $OutputPath }

& $pwsh.Source @relaunchArgs
exit $LASTEXITCODE
```

### Key Points

- **`$SkipRelaunch` guard at the top** — checked before any other logic. Prevents relaunch
  loop when PS 7 script is already running. Also triggers the direct path when the user runs
  the script directly under PS 7 without needing the 5.1 compatibility path.
- **`-ExecutionPolicy Bypass`** on relaunch — required on machines with a restrictive execution
  policy; the 5.1 launcher itself bypassed it by being called directly.
- **`Invoke-Item` on output folder** — opens Explorer to the result directory on completion.
  Eliminates the "where did the file go?" problem for end users.
- **Pass-through parameters** — add each launcher param explicitly to `$relaunchArgs`. Switches
  use the flag name only; string params need name + value as separate array elements.

### Headless Mode

For scheduled tasks, add a dedicated flag that skips interactive output:

```powershell
param(
    [string]$OutputPath,
    [switch]$SkipRelaunch,
    [switch]$Headless      # no Explorer open, no Write-Host color output
)

if ($SkipRelaunch -or $PSVersionTable.PSVersion.Major -ge 7) {
    Import-Module (Join-Path $PSScriptRoot 'src\MyModule.psd1') -Force
    $result = Invoke-XXMain -OutputPath:$OutputPath
    if (-not $Headless) { Invoke-Item (Split-Path $result.OutputPath -Parent) }
    exit ($result.Success ? 0 : 1)
}
```

---

## 14. Quality Gate Pattern

Single script that runs all automated checks. Located at `tests/Invoke-QualityCheck.ps1`.

### Structure

```
Quality Gate
├── CHECK 1: PSScriptAnalyzer (if available)
│   └── Severity: Warning + Error, exclude known false positives
├── CHECK 2: Pattern-based violation scan
│   └── -Path without -Literal, += on arrays, ConvertTo-Json without -Depth
├── CHECK 3: Function standards
│   └── All exported functions have [CmdletBinding()] and .SYNOPSIS help
├── CHECK 4: Hardcoded path detection
│   └── $env:USERPROFILE, \\Desktop\\, C:\\Users\\
├── CHECK 5: Error handling patterns
│   └── try/catch blocks have $ErrorActionPreference = 'Stop' in scope
└── CHECK 6: Pester tests
    └── All *.Tests.ps1 pass
```

### Results Tracking

```powershell
$script:TotalChecks = 0
$script:PassedChecks = 0
$script:FailedChecks = 0
$script:Failures = [System.Collections.Generic.List[string]]::new()

function Add-CheckResult {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string]$Check,
        [Parameter(Mandatory)] [bool]$Passed,
        [string]$Detail
    )
    $script:TotalChecks++
    if ($Passed) { $script:PassedChecks++ }
    else {
        $script:FailedChecks++
        $script:Failures.Add("$Check — $Detail")
    }
}
```

### Exit Code

```powershell
if ($script:FailedChecks -gt 0) {
    Write-Host "QUALITY GATE: FAILED" -ForegroundColor Red
    exit 1
}
else {
    Write-Host "QUALITY GATE: PASSED" -ForegroundColor Green
    exit 0
}
```

### PSScriptAnalyzer Rule Exclusions

Document why rules are excluded:

```powershell
# Rules to exclude (with justification)
$excludeRules = @(
    'PSUseBOMForUnicodeEncodedFile'   # PS 7 uses UTF8NoBOM by default
    'PSAvoidGlobalVars'               # $global:GuiCtx required for WPF delegates
    'PSReviewUnusedParameter'         # False positives on splatted params
)
```

### Code-Line Filtering

Quality gate should skip comments when scanning for violations:

```powershell
function Get-CodeLines {
    [CmdletBinding()]
    param([Parameter(Mandatory)] [string[]]$Lines)

    $inBlockComment = $false
    for ($i = 0; $i -lt $Lines.Count; $i++) {
        if ($Lines[$i] -match '<#') { $inBlockComment = $true }
        if ($Lines[$i] -match '#>') { $inBlockComment = $false; continue }
        if ($inBlockComment) { continue }
        if ($Lines[$i].TrimStart() -match '^#') { continue }
        [PSCustomObject]@{ LineNumber = $i + 1; Text = $Lines[$i] }
    }
}
```

---

## 15. Self-Verification Loop

The development workflow for each module:

```
┌─────────────────────────────────────────────┐
│  1. Write module code                       │
│  2. Write corresponding Pester tests        │
│  3. Run quality gate (Invoke-QualityCheck)  │
│  4. Fix any failures                        │
│  5. Run /forge-review (code review pass)     │
│  6. Fix review findings                     │
│  7. Update CLAUDE.md / project docs         │
│  8. Move to next module                     │
└─────────────────────────────────────────────┘
```

### Rules

- Never skip step 3 — quality gate catches regressions
- Step 5 (/forge-review) looks for: dead code, over-engineering, missed reuse, inconsistencies
- Step 7 keeps CLAUDE.md accurate as the project evolves
- Repeat the loop for each new module or significant feature

---

## 16. Release Packaging

Build script stages files, verifies, creates ZIP + hash. Dev-only files are excluded.

### Explicit Include List

```powershell
# Define exactly what ships — no globs, no surprises
$includeFiles = @(
    'MyModule.ps1'
    'src/MyModule.psd1'
    'src/MyModule.psm1'
    'src/Core/Configuration.ps1'
    'src/Core/Logging.ps1'
    'config/MyModule.defaults.json'
    'docs/UserGuide.html'
)
```

### Staging and Remapping

```powershell
# Files to place at package root instead of their repo path
$rootStagedFiles = @{
    'docs/UserGuide.html'         = 'UserGuide.html'
    'docs/TechnicalReference.html' = 'TechnicalReference.html'
}

foreach ($relPath in $includeFiles) {
    $sourcePath = Join-Path $PSScriptRoot $relPath
    $stagingRelPath = if ($rootStagedFiles.ContainsKey($relPath)) {
        $rootStagedFiles[$relPath]
    } else { $relPath }
    $destPath = Join-Path $stagingDir $stagingRelPath
    $null = New-Item (Split-Path $destPath -Parent) -ItemType Directory -Force
    Copy-Item -LiteralPath $sourcePath -Destination $destPath -Force
}
```

### Empty Directories

```powershell
# Compress-Archive drops empty directories — add placeholder files
$emptyDirs = @('logs', 'resources')
foreach ($dir in $emptyDirs) {
    $dirPath = Join-Path $stagingDir $dir
    $null = New-Item -Path $dirPath -ItemType Directory -Force
    $null = New-Item -Path (Join-Path $dirPath '.gitkeep') -ItemType File -Force
}
```

### ZIP Creation (Flat, No Wrapper)

```powershell
# Use wildcard path to avoid wrapper folder in ZIP
Compress-Archive -Path (Join-Path $stagingDir '*') -DestinationPath $zipPath -CompressionLevel Optimal

# SHA256 hash file
$hash = (Get-FileHash -LiteralPath $zipPath -Algorithm SHA256).Hash
"$hash  $zipName" | Set-Content -LiteralPath $hashPath -Encoding utf8
```

### Dev-File Exclusion Verification

```powershell
$devPatterns = @('tests', 'docs', 'CLAUDE.md', '.claude', 'Build-Release.ps1')
foreach ($pattern in $devPatterns) {
    if (Test-Path -LiteralPath (Join-Path $stagingDir $pattern)) {
        throw "Dev file '$pattern' found in package"
    }
}
```

### Module Import Verification

```powershell
# Verify the staged module actually imports
$psd1 = Join-Path $stagingDir 'src/MyModule.psd1'
try {
    Import-Module $psd1 -Force -ErrorAction Stop
    Remove-Module 'MyModule' -Force -ErrorAction SilentlyContinue
}
catch {
    throw "Module import verification failed: $($_.Exception.Message)"
}
```

### Return Object

```powershell
[PSCustomObject]@{
    Success   = $true
    Version   = $version
    ZipPath   = $zipPath
    HashPath  = $hashPath
    FileCount = $copiedCount
    ZipSizeKB = [math]::Round((Get-Item $zipPath).Length / 1024, 1)
    SHA256    = $hash
}
```

---

## 17. Gotchas

| # | Gotcha | Symptom | Fix |
|---|--------|---------|-----|
| 1 | `FunctionsToExport = '*'` | All internal functions exposed, slow tab-complete | Explicit list in .psd1 |
| 2 | Dot-source order wrong | "Command not found" on dependent functions | Map dependencies, load foundation first |
| 3 | `ConvertTo-Json` without `-Depth` | Nested objects become `System.Collections.Hashtable` string | Always specify `-Depth 5` or higher |
| 4 | `$global:` for module state | State leaks across modules, breaks test isolation | Use `$script:` exclusively |
| 5 | Hardcoded `C:\Users\...` paths | Breaks on other machines, other user profiles | `$PSScriptRoot` relative paths only |
| 6 | `Compress-Archive -Path $dir` | Wrapper folder in ZIP, double nesting on extract | Use `-Path "$dir\*"` for flat contents |
| 7 | Empty directories in ZIP | `Compress-Archive` silently drops them | Add `.gitkeep` placeholder files |
| 8 | Missing `-Encoding utf8` on writes | BOM issues, encoding inconsistencies | Always explicit `-Encoding utf8` |
| 9 | Config file in source tree | User edits overwritten on update | Separate defaults (shipped) + user file (first run) |
| 10 | `-Path` instead of `-LiteralPath` | Brackets/wildcards in filenames break | Use `-LiteralPath` unless intentional globbing |
| 11 | `+=` on arrays in loops | O(n^2) — new array allocated each iteration | `[System.Collections.Generic.List[object]]` |
| 12 | No quality gate | Regressions, style drift, broken tests ship unnoticed | Automate all checks in single script |
| 13 | Test modifies production config | User loses their settings | InModuleScope redirect to temp paths |
| 14 | Missing `[CmdletBinding()]` | No `-Verbose`, `-WhatIf`, `-ErrorAction` support | Add to every function |
| 15 | No headless mode | Can't run from scheduled task without GUI | Add `-RunBackup` parameter to launcher |
| 16 | Config field added, no migration | Existing user configs throw on missing field | Check `PSObject.Properties` before access |
| 17 | Glob-based module loading | Undefined load order, intermittent failures | Explicit dot-source in dependency order |
| 18 | Logic in .psm1 | Hard to test, unclear dependencies | .psm1 is just the load sequence |
| 19 | No module import verification in build | Ship broken package, discover in production | Import from staged copy before packaging |
| 20 | Non-standard verbs | Import warning on every module load | Use `Get-Verb` to check before naming |
| 21 | `[ordered]@{}` uses `.ContainsKey()` | `MethodNotFound` at runtime | `[ordered]@{}` is `OrderedDictionary`, not `Hashtable` — use `.Contains('Key')` |
| 22 | Log directory created on every `Write-XXLog` call | Redundant `New-Item` on every log line | Use a `$Script:LogDirReady` flag; create the dir once on first call |

---

## 18. New Module Checklist

When starting a new PowerShell module project:

- [ ] Create directory structure (src/, tests/, config/, docs/, logs/, resources/)
- [ ] Create module manifest (.psd1) with explicit `FunctionsToExport`
- [ ] Create root module (.psm1) with dependency-ordered dot-sourcing
- [ ] Choose 2-3 letter project prefix for function names
- [ ] Create defaults.json with full config schema
- [ ] Implement Configuration.ps1 (Get/Set/New/Test-XXConfig)
- [ ] Implement Logging.ps1 (Write-XXLog, Initialize-XXLogSession)
- [ ] Write CLAUDE.md with project structure, naming conventions, config schema
- [ ] Create Invoke-QualityCheck.ps1 with all 6 check categories
- [ ] Create PS 5.1 bootstrap launcher if targeting end-user deployment
- [ ] Implement features in dependency order (foundation → core → features → GUI)
- [ ] One test file per source module
- [ ] Run quality gate after every module
- [ ] Build script with explicit include list, staging, verification, ZIP + hash
