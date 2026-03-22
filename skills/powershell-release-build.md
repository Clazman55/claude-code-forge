# PowerShell Release Build Pattern

Canonical pattern for building a versioned, distributable ZIP package from a PowerShell module
project. Covers staging, verification, hashing, and cleanup. Based on production project builds.

---

## 1. Script Structure Overview

```
Step 1: Read version from module manifest
Step 2: Run quality gate (in-process)
Step 3: Stage files to temp directory (explicit include list)
Step 4: Verify staged content (all expected files present + no dev leaks)
Step 5: Verify module imports cleanly from staged location
Step 6: Create ZIP + SHA256 hash file
Step 7: Print summary
Step 8: Cleanup staging dir
Step 9: Return PSCustomObject with build metadata
```

---

## 2. Script Header

```powershell
#Requires -Version 7.0

<#
.SYNOPSIS
    Builds a MyModule release package.
.DESCRIPTION
    Stages runtime files, runs quality gate, creates versioned ZIP + SHA256 hash.
    Dev-only files (tests, docs, CLAUDE.md, build script) are excluded.
.PARAMETER OutputDirectory
    Where the ZIP and hash file are written. Defaults to <script root>/release.
.PARAMETER SkipQualityGate
    Skip running Invoke-QualityCheck.ps1 before packaging.
#>
[CmdletBinding()]
param(
    [Parameter()]
    [string]$OutputDirectory = (Join-Path -Path $PSScriptRoot -ChildPath 'release'),

    [Parameter()]
    [switch]$SkipQualityGate
)

$ErrorActionPreference = 'Stop'
```

**Key points:**
- `$OutputDirectory` defaults to `$PSScriptRoot/release` — always correct regardless of working directory. Never use `.\release` (relative to wherever pwsh was invoked from).
- `$ErrorActionPreference = 'Stop'` — any unexpected failure throws immediately.

---

## 3. Read Version from Manifest

```powershell
$manifestPath = Join-Path -Path $PSScriptRoot -ChildPath 'src/MyModule.psd1'
if (-not (Test-Path -LiteralPath $manifestPath)) {
    throw "Module manifest not found at '$manifestPath'"
}

$version = (Import-PowerShellDataFile -LiteralPath $manifestPath).ModuleVersion
Write-Host "MyModule Release Builder" -ForegroundColor Cyan
Write-Host "Version: $version" -ForegroundColor Cyan
Write-Host ""
```

- `Import-PowerShellDataFile` — reads the `.psd1` without executing it. Safe and correct.
- `-LiteralPath` throughout — never `-Path` for known paths (avoids wildcard expansion bugs).

---

## 4. Quality Gate

```powershell
if (-not $SkipQualityGate) {
    Write-Host "Running quality gate..." -ForegroundColor Yellow
    $qualityGatePath = Join-Path -Path $PSScriptRoot -ChildPath 'tests/Invoke-QualityCheck.ps1'
    if (-not (Test-Path -LiteralPath $qualityGatePath)) {
        throw "Quality gate script not found at '$qualityGatePath'"
    }

    & $qualityGatePath
    if ($LASTEXITCODE -ne 0) {
        throw "Quality gate failed (exit code $LASTEXITCODE). Fix issues before building a release."
    }
    Write-Host "Quality gate passed." -ForegroundColor Green
    Write-Host ""
}
```

- `& $qualityGatePath` — runs in-process (same pwsh session). Do NOT use `pwsh -File` subprocess; that spawns a child process that can't inherit module state and makes debugging harder.
- Check `$LASTEXITCODE` after the call — quality gate script sets it explicitly.

---

## 5. Staging — Explicit Include List

```powershell
$stagingRoot = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath "MyModule_Release_$(Get-Random)"
$stagingDir  = Join-Path -Path $stagingRoot -ChildPath 'MyModule'

Write-Host "Staging files..." -ForegroundColor Yellow

# Explicit include list — only runtime files ship
$includeFiles = @(
    'MyModule.ps1'                        # launcher
    'src/MyModule.psd1'
    'src/MyModule.psm1'
    'src/Core/Config.ps1'
    'src/Core/Logging.ps1'
    'src/Core/Helpers.ps1'
    'src/Core/Invoke-MyMain.ps1'
    # ... add all runtime source files ...
    'config/defaults.json'
)

$copiedCount = 0
foreach ($relPath in $includeFiles) {
    $sourcePath = Join-Path -Path $PSScriptRoot -ChildPath $relPath
    if (-not (Test-Path -LiteralPath $sourcePath)) {
        throw "Expected file not found: '$sourcePath'"
    }

    $destPath = Join-Path -Path $stagingDir -ChildPath $relPath
    $destDir  = Split-Path -Path $destPath -Parent
    $null = New-Item -Path $destDir -ItemType Directory -Force

    Copy-Item -LiteralPath $sourcePath -Destination $destPath -Force
    $copiedCount++
}

# Create placeholder directories as needed (e.g., logs/)
$null = New-Item -Path (Join-Path -Path $stagingDir -ChildPath 'logs') -ItemType Directory -Force
$null = New-Item -Path (Join-Path -Path $stagingDir -ChildPath 'logs/.gitkeep') -ItemType File -Force

Write-Host "  Staged $copiedCount files" -ForegroundColor Green
```

**Why explicit include list (not whole-directory copy):**
- Whole-directory copy leaks `tests/`, `docs/`, `CLAUDE.md`, `.claude/`, `Build-Release.ps1`, `.git/` into the package.
- Clients get dev reference material (PowerShell baseline docs, pester patterns, internal notes) — a security and professionalism problem.
- Explicit list forces conscious review of what ships.

**Staging structure:**
- Root: `$env:TEMP/MyModule_Release_<random>/` — isolated from project tree.
- Nested: `$stagingRoot/MyModule/` — so the ZIP extracts into a named folder, not bare files.
- `Get-Random` suffix — prevents collision if two builds run simultaneously.

---

## 6. Staged Content Verification

```powershell
Write-Host "Verifying staged content..." -ForegroundColor Yellow

# All expected files are present
foreach ($relPath in $includeFiles) {
    $checkPath = Join-Path -Path $stagingDir -ChildPath $relPath
    if (-not (Test-Path -LiteralPath $checkPath)) {
        throw "Staging verification failed: missing '$relPath'"
    }
}

# No dev files leaked in
$devPatterns = @('tests', 'docs', 'CLAUDE.md', '.claude', 'Build-Release.ps1', '.gitignore', '.git')
foreach ($pattern in $devPatterns) {
    $checkPath = Join-Path -Path $stagingDir -ChildPath $pattern
    if (Test-Path -LiteralPath $checkPath) {
        throw "Staging verification failed: dev file '$pattern' found in package"
    }
}

# Module imports cleanly from staged location
$importTestPsd = Join-Path -Path $stagingDir -ChildPath 'src/MyModule.psd1'
try {
    Import-Module $importTestPsd -Force -ErrorAction Stop
    Remove-Module 'MyModule' -Force -ErrorAction SilentlyContinue
}
catch {
    throw "Module import verification failed: $($_.Exception.Message)"
}

Write-Host "  Verification passed" -ForegroundColor Green
```

**Three verification checks:**
1. **All files present** — re-walks the include list, confirms each file exists in staging.
2. **No dev leaks** — checks known dev file/dir names are absent.
3. **Module imports** — imports the module from the staged copy, not the source tree. Catches missing dot-source entries in `.psm1` before the ZIP is created.

---

## 7. ZIP + SHA256 Hash

```powershell
Write-Host "Creating archive..." -ForegroundColor Yellow

$null = New-Item -Path $OutputDirectory -ItemType Directory -Force
$zipName = "MyModule-v$version.zip"
$zipPath = Join-Path -Path $OutputDirectory -ChildPath $zipName

if (Test-Path -LiteralPath $zipPath) {
    Remove-Item -LiteralPath $zipPath -Force
}

Compress-Archive -Path (Join-Path -Path $stagingDir -ChildPath '*') `
                 -DestinationPath $zipPath `
                 -CompressionLevel Optimal

$hash         = (Get-FileHash -LiteralPath $zipPath -Algorithm SHA256).Hash
$hashFileName = "MyModule-v$version.sha256"
$hashPath     = Join-Path -Path $OutputDirectory -ChildPath $hashFileName
"$hash  $zipName" | Set-Content -LiteralPath $hashPath -Encoding utf8

$zipSizeKB = [math]::Round((Get-Item -LiteralPath $zipPath).Length / 1024, 1)

Write-Host "  Created: $zipName ($zipSizeKB KB)" -ForegroundColor Green
```

- `Compress-Archive -Path "$stagingDir/*"` — zips the contents of the staging dir (not the staging dir itself).
- `Set-Content` for the hash file — not `Out-File`. `Set-Content` with `-Encoding utf8` produces clean UTF-8 without BOM; `Out-File` default on PS 5.1 is UTF-16 LE.
- Hash file format: `<SHA256>  <filename>` — two spaces (sha256sum convention).
- ZIP name includes version: `MyModule-v1.0.0.zip`.

---

## 8. Summary, Cleanup, Return

```powershell
# Summary
Write-Host ""
Write-Host "Build Summary" -ForegroundColor Cyan
Write-Host "  Version:  $version"
Write-Host "  Files:    $copiedCount"
Write-Host "  ZIP:      $zipPath"
Write-Host "  Size:     $zipSizeKB KB"
Write-Host "  SHA256:   $hash"
Write-Host ""
Write-Host "Release build complete." -ForegroundColor Green

# Cleanup — after summary so the user sees results even if cleanup has issues
if (Test-Path -LiteralPath $stagingRoot) {
    Remove-Item -LiteralPath $stagingRoot -Recurse -Force -ErrorAction SilentlyContinue
}

# Re-import from project source (restore after staged verification import)
Import-Module $manifestPath -Force -ErrorAction SilentlyContinue

# Return result object for scripted/pipeline use
[PSCustomObject]@{
    Success   = $true
    Version   = $version
    ZipPath   = $zipPath
    HashPath  = $hashPath
    FileCount = $copiedCount
    ZipSizeKB = $zipSizeKB
    SHA256    = $hash
}
```

**Key points:**
- Cleanup is AFTER the summary block — if cleanup fails, the user still sees the results.
- Do NOT put cleanup in `finally` — a `finally` block runs even on success, making it harder to skip cleanup intentionally (e.g., for debugging a staging issue).
- Re-import from project source after the staged import test — restores the module to the live dev copy for any subsequent interactive use.
- Always return a `PSCustomObject` — allows `$result = .\Build-Release.ps1` in scripts and pipelines.

---

## 9. Exclusion List — What Never Ships

| Category | Examples |
|----------|---------|
| Dev tooling | `tests/`, `Invoke-QualityCheck.ps1` |
| Dev reference | `docs/` (PowerShell skill docs, architecture notes) |
| Internal notes | `CLAUDE.md`, `.claude/` |
| Build tooling | `Build-Release.ps1` itself |
| Source control | `.git/`, `.gitignore` |

The build script itself never ships — it's a dev tool. Users get a folder of source + launcher, not the build infrastructure.

---

## 10. Gotchas

| Issue | Symptom | Fix |
|-------|---------|-----|
| Whole-directory staging | `docs/` and `tests/` end up in ZIP | Use explicit `$includeFiles` array |
| Relative `OutputDirectory` | ZIP written to wrong dir when pwsh invoked from elsewhere | `Join-Path -Path $PSScriptRoot -ChildPath 'release'` |
| `pwsh -File` for quality gate | Subprocess can't inherit module context; harder to debug | `& $qualityGatePath` in-process |
| `Out-File` for hash file | UTF-16 LE on PS 5.1, may show garbled in tools | `Set-Content -Encoding utf8` |
| Cleanup in `finally` | Can't skip for debugging; runs even mid-investigation | Cleanup after summary, not in `finally` |
| No module import test | ZIP created with broken module (missing dot-source entry) | Import from staged copy before zipping |
| Dev patterns check missing | `CLAUDE.md` ships to client | Explicit dev-pattern check in verification step |
| No return object | `$result = .\Build-Release.ps1` returns `$null` | Always return `[PSCustomObject]` |
| `-Path` instead of `-LiteralPath` | Paths with `[]` treated as wildcards | Use `-LiteralPath` for all known paths |

---

## 11. Quick Checklist

- [ ] `$OutputDirectory` defaults to `Join-Path $PSScriptRoot 'release'` (not `.\release`)
- [ ] Explicit `$includeFiles` list — no whole-directory copy
- [ ] Quality gate runs via `& $qualityGatePath` (not `pwsh -File`)
- [ ] `$LASTEXITCODE` checked after quality gate
- [ ] Staging in `$env:TEMP` with `Get-Random` suffix
- [ ] Nested staging: `$stagingRoot/ModuleName/` (so ZIP extracts to named folder)
- [ ] File presence verification (re-walk include list from staging dir)
- [ ] Dev-pattern leak check (tests, docs, CLAUDE.md, .git, etc.)
- [ ] Module import test from staged copy before zipping
- [ ] `Set-Content -Encoding utf8` for hash file (not `Out-File`)
- [ ] Hash file format: `<SHA256>  <filename>` (two spaces)
- [ ] Cleanup after summary (not in `finally`)
- [ ] Re-import from source after staged import test
- [ ] Return `[PSCustomObject]` with build metadata
