# Pester 5.x Testing Patterns — Skill Reference

Patterns for writing Pester tests in PowerShell 7+ module projects.
Extracted from real-world experience testing a PowerShell backup module (239 tests across 11 test files, 33 exported functions).

Companion to `powershell-baseline.md` (coding standards) and `powershell-module-architecture.md` (module structure).

---

## Table of Contents

1. [Test File Structure](#1-test-file-structure)
2. [Module Import Pattern](#2-module-import-pattern)
3. [Test Isolation — Temp Directories](#3-test-isolation--temp-directories)
4. [Test Isolation — Config Redirection](#4-test-isolation--config-redirection)
5. [InModuleScope — Accessing Module Internals](#5-inmodulescope--accessing-module-internals)
6. [Assertion Patterns (Should)](#6-assertion-patterns-should)
7. [Mocking External Commands](#7-mocking-external-commands)
8. [Integration Test Patterns](#8-integration-test-patterns)
9. [Build/Packaging Tests](#9-buildpackaging-tests)
10. [GUI/WPF Tests](#10-guiwpf-tests)
11. [Test Data Patterns](#11-test-data-patterns)
12. [Pester 5.x Configuration](#12-pester-5x-configuration)
13. [Quality Gate Integration](#13-quality-gate-integration)
14. [Test Organization](#14-test-organization)
15. [Output Suppression](#15-output-suppression)
16. [ShouldProcess Testing](#16-shouldprocess-testing)
17. [Parameterized Tests (TestCases)](#17-parameterized-tests-testcases)
18. [Debugging Failed Tests](#18-debugging-failed-tests)
19. [Gotchas](#19-gotchas)
20. [Quick-Reference Checklist](#20-quick-reference-checklist)

---

## 1. Test File Structure

Every test file follows this skeleton:

```powershell
#Requires -Version 7.0
#Requires -Modules Pester

BeforeAll {
    # Module import + test setup (runs ONCE before all tests in file)
    $projectRoot = Split-Path -Path $PSScriptRoot -Parent
    $modulePath = Join-Path -Path $projectRoot -ChildPath 'src/MyModule.psd1'
    Import-Module $modulePath -Force
}

AfterAll {
    # Cleanup (runs ONCE after all tests in file)
}

Describe 'Get-XXConfig' {
    Context 'Default behavior' {
        It 'returns valid config object' {
            $config = Get-XXConfig
            $config | Should -Not -BeNullOrEmpty
        }
    }

    Context 'With custom path' {
        It 'loads from specified file' {
            $config = Get-XXConfig -ConfigPath $testPath
            $config.general | Should -Not -BeNullOrEmpty
        }
    }
}
```

### Hierarchy

- **`BeforeAll` / `AfterAll`** — file-level setup/teardown. Module import goes here.
- **`Describe`** — groups tests for one function or feature
- **`Context`** — sub-groups for scenarios within a Describe
- **`It`** — individual test case with assertion
- **`BeforeEach` / `AfterEach`** — per-test setup/teardown (use sparingly)

### Naming Convention

- Test files: `FunctionGroup.Tests.ps1` — matches the source file name
- `Describe` blocks: function name or feature name
- `It` blocks: describe the expected behavior in plain English

---

## 2. Module Import Pattern

Always import via the manifest (.psd1), not the root module (.psm1):

```powershell
BeforeAll {
    $projectRoot = Split-Path -Path $PSScriptRoot -Parent
    $modulePath = Join-Path -Path $projectRoot -ChildPath 'src/MyModule.psd1'
    Import-Module $modulePath -Force
}
```

### Why .psd1 Over .psm1

- `.psd1` validates `PowerShellVersion`, `RequiredModules`, and `FunctionsToExport`
- `.psd1` is what consumers use — test the real import path
- `-Force` ensures a clean reload even if module was already loaded

### Anti-Pattern: Relative Import

```powershell
# BAD — fragile, depends on working directory
Import-Module ./src/MyModule.psd1

# GOOD — always resolve from test file location
$projectRoot = Split-Path -Path $PSScriptRoot -Parent
$modulePath = Join-Path -Path $projectRoot -ChildPath 'src/MyModule.psd1'
Import-Module $modulePath -Force
```

---

## 3. Test Isolation — Temp Directories

Every test file creates its own unique temp directory. Never reuse across test files.

```powershell
BeforeAll {
    # Unique name prevents collisions with parallel runs
    $script:TestDir = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath "MyModule_Tests_$(Get-Random)"
    New-Item -Path $script:TestDir -ItemType Directory -Force | Out-Null

    # Sub-directories as needed
    $script:SrcDir = Join-Path -Path $script:TestDir -ChildPath 'source'
    $script:DstDir = Join-Path -Path $script:TestDir -ChildPath 'dest'
    $script:LogDir = Join-Path -Path $script:TestDir -ChildPath 'logs'
    New-Item -Path $script:SrcDir -ItemType Directory -Force | Out-Null
    New-Item -Path $script:DstDir -ItemType Directory -Force | Out-Null
    New-Item -Path $script:LogDir -ItemType Directory -Force | Out-Null
}

AfterAll {
    # Always clean up, even on failure
    if (Test-Path -LiteralPath $script:TestDir) {
        Remove-Item -LiteralPath $script:TestDir -Recurse -Force -ErrorAction SilentlyContinue
    }
}
```

### Rules

- **`Get-Random` in name** — prevents collisions across parallel runs
- **`| Out-Null`** — suppress directory creation output
- **`-ErrorAction SilentlyContinue` on cleanup** — don't fail if files are locked
- **Clean up in AfterAll** — always, no matter what

---

## 4. Test Isolation — Config Redirection

Tests must NEVER modify the user's real config file. Redirect module config paths to temp.

```powershell
BeforeAll {
    # ... module import and temp dir setup ...

    # Redirect module's config path to temp directory
    $testConfigFile = Join-Path -Path $script:TestDir -ChildPath 'MyModule.json'
    InModuleScope 'MyModule' -Parameters @{ path = $testConfigFile } {
        $script:OriginalDefaultConfigPath = $script:XXDefaultConfigPath
        $script:XXDefaultConfigPath = $path
    }
}

AfterAll {
    # Restore real config path before cleanup
    InModuleScope 'MyModule' {
        $script:XXDefaultConfigPath = $script:OriginalDefaultConfigPath
    }

    # ... temp dir cleanup ...
}
```

### Config Snapshot/Restore

For integration tests that modify the loaded config object:

```powershell
BeforeAll {
    # Snapshot original config
    $script:OriginalConfig = InModuleScope 'MyModule' {
        $script:XXConfig | ConvertTo-Json -Depth 10 | ConvertFrom-Json
    }
}

AfterAll {
    # Restore original config
    InModuleScope 'MyModule' -Parameters @{ json = ($script:OriginalConfig | ConvertTo-Json -Depth 10) } {
        param($json)
        $script:XXConfig = $json | ConvertFrom-Json
    }
}
```

### Why JSON Round-Trip?

The `ConvertTo-Json | ConvertFrom-Json` creates a deep copy. Without it, the snapshot and the live config reference the same object — mutations to one affect the other.

---

## 5. InModuleScope — Accessing Module Internals

`InModuleScope` executes code within the module's scope, giving access to `$script:` variables.

### Reading Module State

```powershell
$configValue = InModuleScope 'MyModule' {
    $script:XXConfig.email.smtpServer
}
$configValue | Should -Be 'smtp.test.com'
```

### Writing Module State

```powershell
InModuleScope 'MyModule' {
    $script:XXConfig.email.enabled = $false
}
```

### Passing Parameters

Variables from the test scope are NOT visible inside `InModuleScope`. Pass them explicitly:

```powershell
# BAD — $testPath is NOT accessible inside InModuleScope
$testPath = 'C:\temp\test.json'
InModuleScope 'MyModule' {
    $script:ConfigPath = $testPath  # ERROR: $testPath is $null
}

# GOOD — pass via -Parameters
$testPath = 'C:\temp\test.json'
InModuleScope 'MyModule' -Parameters @{ path = $testPath } {
    $script:ConfigPath = $path  # Works!
}
```

### Helper Function Pattern for Integration Tests

When many tests need to modify config, create a helper:

```powershell
BeforeAll {
    function global:Set-IntegrationConfig {
        [CmdletBinding()]
        param([Parameter(Mandatory)] [hashtable]$Settings)

        InModuleScope 'MyModule' -Parameters @{ s = $Settings } {
            param($s)
            $config = $script:XXConfig
            if ($s.ContainsKey('emailEnabled'))  { $config.email.enabled = $s.emailEnabled }
            if ($s.ContainsKey('smtpServer'))    { $config.email.smtpServer = $s.smtpServer }
            if ($s.ContainsKey('logDirectory'))  { $config.general.logDirectory = $s.logDirectory }
        }
    }
}

AfterAll {
    Remove-Item -Path 'Function:\Set-IntegrationConfig' -ErrorAction SilentlyContinue
}

# Usage in tests:
Set-IntegrationConfig -Settings @{
    emailEnabled = $true
    smtpServer   = 'smtp.test.com'
    logDirectory = $script:LogDir
}
```

---

## 6. Assertion Patterns (Should)

### Value Assertions

```powershell
$result | Should -Be 42                        # Equal (case-insensitive for strings)
$result | Should -BeExactly 'CaseSensitive'    # Equal (case-sensitive)
$result | Should -BeTrue                        # Boolean true
$result | Should -BeFalse                       # Boolean false
$result | Should -BeNullOrEmpty                 # Null or empty string/collection
$result | Should -Not -BeNullOrEmpty            # Has a value
$result | Should -BeGreaterThan 0              # Numeric comparison
$result | Should -BeLessOrEqual 100            # Numeric comparison
$result | Should -BeLike '*.zip'               # Wildcard match
$result | Should -Match '^[A-Fa-f0-9]{64}'    # Regex match
$result | Should -BeOfType [hashtable]         # Type check
```

### Collection Assertions

```powershell
$list.Count | Should -Be 3
$list | Should -Contain 'expected'              # At least one element matches
$list | Should -HaveCount 3                     # Exact count
@($result).Count | Should -BeGreaterThan 0     # Wrap cmdlet output in @()
```

### Exception Assertions

```powershell
{ Do-Something -Bad } | Should -Throw                    # Any exception
{ Do-Something -Bad } | Should -Throw '*specific*'       # Message wildcard
{ Do-Something -Bad } | Should -Throw -ExceptionType [System.IO.FileNotFoundException]
{ Do-Something -Good } | Should -Not -Throw              # No exception
```

### File/Path Assertions

```powershell
Test-Path -LiteralPath $filePath | Should -BeTrue         # File exists
Test-Path -LiteralPath $filePath | Should -BeFalse        # File does NOT exist
$file.Extension | Should -Be '.zip'
(Get-Item -LiteralPath $filePath).Length | Should -BeGreaterThan 0
```

### Mock Invocation Assertions

```powershell
Should -Invoke Send-MailMessage -Times 1 -Exactly -ModuleName MyModule
Should -Invoke Stop-Process -Times 0 -Exactly                    # Never called
Should -Invoke Copy-Item -Times 3 -ModuleName MyModule           # At least 3
Should -Invoke Write-XXLog -ParameterFilter { $Level -eq 'ERROR' }
```

---

## 7. Mocking External Commands

### Basic Mock

```powershell
Context 'Email sending' {
    BeforeAll {
        Mock Send-MailMessage { } -ModuleName MyModule
    }

    It 'sends email when enabled' {
        Send-XXReport -BackupSuccess $true
        Should -Invoke Send-MailMessage -Times 1 -ModuleName MyModule
    }
}
```

### Critical: `-ModuleName` Is Required

When the function being tested is inside a module, the mock must target that module:

```powershell
# BAD — mock applies to test scope, not module scope
Mock Send-MailMessage { }

# GOOD — mock applies inside the module where it's called
Mock Send-MailMessage { } -ModuleName MyModule
```

### Mock with Return Value

```powershell
Mock Get-Process {
    [PSCustomObject]@{ Name = 'notepad'; Id = 1234; HasExited = $false }
} -ModuleName MyModule
```

### Mock with Parameter Filter

```powershell
Mock Stop-Process { } -ParameterFilter { $Name -eq 'notepad' } -ModuleName MyModule
Mock Stop-Process { throw "Should not stop other processes" } -ModuleName MyModule
```

### Wrapping Untestable Cmdlets

Some cmdlets cannot be directly mocked by Pester because their parameter types reference
internal .NET types that break Pester's stub generator at parse time (a `ParseException`
before the mock is even registered). The symptom: Pester throws on `Mock Get-PhysicalDisk`
with a type error, and the real cmdlet runs instead.

**Pattern:** Wrap the problematic cmdlet in a private module-scope function. Mock the wrapper.

```powershell
# In the source file (Get-SIHardware.ps1):
function Get-SIPhysicalDiskMap {
    # Private wrapper — exists solely to give Pester a mockable surface.
    # Get-PhysicalDisk has a parameter type ([Get-PhysicalDisk.PhysicalDiskUsage])
    # that Pester's stub generator cannot parse.
    $map = @{}
    Get-PhysicalDisk -ErrorAction Stop | ForEach-Object { $map[$_.FriendlyName] = $_ }
    $map
}

# In Get-SIHardware, call the wrapper instead of Get-PhysicalDisk directly:
$physicalDiskMap = Get-SIPhysicalDiskMap
```

```powershell
# In the test file (Hardware.Tests.ps1):
BeforeAll {
    Mock Get-SIPhysicalDiskMap {
        @{
            'Samsung SSD 970 EVO' = [PSCustomObject]@{
                FriendlyName     = 'Samsung SSD 970 EVO'
                MediaType        = 'SSD'
                HealthStatus     = 'Healthy'
            }
        }
    } -ModuleName MyModule
}
```

**Known cmdlets with this problem:** `Get-PhysicalDisk` (parameter type `[Get-PhysicalDisk.PhysicalDiskUsage]`).

**Apply this pattern whenever:** Pester throws a `ParseException` or type resolution error when
registering a `Mock` for a system cmdlet. Wrap the call in a private `Get-SI*` helper and mock
that instead.

Other wrappers in this project using the same pattern:

| Wrapper | Wraps | Reason |
|---------|-------|--------|
| `Get-SIPhysicalDiskMap` | `Get-PhysicalDisk` | Parameter type not parseable by Pester |
| `Get-SIWiFiProfiles` | `netsh wlan show profiles` | External process, `$LASTEXITCODE` side effect |
| `Get-SIUpdateHistory` | WUA COM object creation | COM interop untestable in unit context |

---

### Mock Robocopy

Robocopy is an external executable. Mock it as a function that sets `$LASTEXITCODE`:

```powershell
Mock robocopy {
    $global:LASTEXITCODE = 1  # 1 = files copied successfully
} -ModuleName MyModule
```

### Mock with Multiple Behaviors

```powershell
# First call returns running process, second returns nothing (process stopped)
$script:callCount = 0
Mock Get-Process {
    $script:callCount++
    if ($script:callCount -eq 1) {
        [PSCustomObject]@{ Name = 'notepad'; Id = 1234 }
    }
    # Second call returns nothing (process gone)
} -ModuleName MyModule
```

---

## 8. Integration Test Patterns

Integration tests verify real interactions across modules — real file I/O, real config, real function chains.

### Create Real Test Files

```powershell
Describe 'Invoke-XXBackup' {
    Context 'File copy with verification' {
        BeforeAll {
            # Create source file with known content
            $sourceFile = Join-Path $script:SrcDir "testfile_$(Get-Random).txt"
            $destFile = Join-Path $script:DstDir "backup_$(Get-Random).txt"
            Set-Content -LiteralPath $sourceFile -Value "Test content $(Get-Random)" -Encoding utf8

            # Configure the module for this test
            Set-IntegrationConfig -Settings @{
                fileCopy     = $true
                verification = $true
                fileCopyJobs = @([PSCustomObject]@{
                    source      = $sourceFile
                    destination = $destFile
                    enabled     = $true
                })
            }
        }

        It 'copies file to destination' {
            Invoke-XXFileCopy -Source $sourceFile -Destination $destFile
            Test-Path -LiteralPath $destFile | Should -BeTrue
        }

        It 'destination matches source hash' {
            $srcHash = (Get-FileHash -LiteralPath $sourceFile -Algorithm MD5).Hash
            $dstHash = (Get-FileHash -LiteralPath $destFile -Algorithm MD5).Hash
            $dstHash | Should -Be $srcHash
        }
    }
}
```

### Directory Structure Tests

```powershell
Context 'Folder copy' {
    BeforeAll {
        # Create source directory tree
        $sourceDir = Join-Path $script:SrcDir "folder_$(Get-Random)"
        $destDir = Join-Path $script:DstDir "backup_$(Get-Random)"
        $null = New-Item -Path $sourceDir -ItemType Directory -Force
        $null = New-Item -Path (Join-Path $sourceDir 'sub') -ItemType Directory -Force

        Set-Content -LiteralPath (Join-Path $sourceDir 'file1.txt') -Value 'content1' -Encoding utf8
        Set-Content -LiteralPath (Join-Path $sourceDir 'sub/file2.txt') -Value 'content2' -Encoding utf8
    }

    It 'copies entire directory tree' {
        Invoke-XXFolderCopy -Source $sourceDir -Destination $destDir
        Test-Path -LiteralPath (Join-Path $destDir 'file1.txt') | Should -BeTrue
        Test-Path -LiteralPath (Join-Path $destDir 'sub/file2.txt') | Should -BeTrue
    }
}
```

### Disable Side Effects

```powershell
BeforeAll {
    # Disable email globally in integration tests
    InModuleScope 'MyModule' {
        $script:XXConfig.email.enabled = $false
    }
}
```

---

## 9. Build/Packaging Tests

Test the release build script by running it and verifying output.

```powershell
BeforeAll {
    $script:ProjectRoot = Split-Path -Path $PSScriptRoot -Parent
    $script:BuildScript = Join-Path -Path $script:ProjectRoot -ChildPath 'Build-Release.ps1'
    $script:OutputDir = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath "BuildTests_$(Get-Random)"
}

AfterAll {
    if (Test-Path -LiteralPath $script:OutputDir) {
        Remove-Item -LiteralPath $script:OutputDir -Recurse -Force -ErrorAction SilentlyContinue
    }
    # Re-import module to restore state after build script's import test
    $modulePath = Join-Path -Path $script:ProjectRoot -ChildPath 'src/MyModule.psd1'
    Import-Module $modulePath -Force -ErrorAction SilentlyContinue
}

Describe 'Build-Release' {
    Context 'Full build' {
        BeforeAll {
            $script:BuildResult = & $script:BuildScript -OutputDirectory $script:OutputDir -SkipQualityGate
        }

        It 'returns Success=$true' {
            $script:BuildResult.Success | Should -BeTrue
        }

        It 'creates ZIP file' {
            Test-Path -LiteralPath $script:BuildResult.ZipPath | Should -BeTrue
            $script:BuildResult.ZipPath | Should -BeLike '*.zip'
        }

        It 'creates SHA256 hash file' {
            Test-Path -LiteralPath $script:BuildResult.HashPath | Should -BeTrue
            $hashContent = Get-Content -LiteralPath $script:BuildResult.HashPath -Raw
            $hashContent | Should -Match '^[A-Fa-f0-9]{64}\s+'
        }
    }

    Context 'ZIP content verification' {
        BeforeAll {
            $script:ExtractDir = Join-Path $script:OutputDir 'extracted'
            Expand-Archive -LiteralPath $script:BuildResult.ZipPath -DestinationPath $script:ExtractDir -Force
            $script:PackageRoot = $script:ExtractDir
        }

        It 'contains module manifest' {
            $psd1 = Join-Path $script:PackageRoot 'src/MyModule.psd1'
            Test-Path -LiteralPath $psd1 | Should -BeTrue
        }

        It 'does NOT contain tests directory' {
            $tests = Join-Path $script:PackageRoot 'tests'
            Test-Path -LiteralPath $tests | Should -BeFalse
        }

        It 'does NOT contain CLAUDE.md' {
            $claude = Join-Path $script:PackageRoot 'CLAUDE.md'
            Test-Path -LiteralPath $claude | Should -BeFalse
        }

        It 'module imports successfully from extracted package' {
            $psd1 = Join-Path $script:PackageRoot 'src/MyModule.psd1'
            { Import-Module $psd1 -Force -ErrorAction Stop } | Should -Not -Throw
            Remove-Module 'MyModule' -Force -ErrorAction SilentlyContinue
        }
    }
}
```

### Key Points

- **`-SkipQualityGate`** — build tests shouldn't re-run the full quality gate
- **Re-import module in AfterAll** — build script may import from staging, restoring module state
- **Test both positive (contains) and negative (does NOT contain)** — ensure dev files are excluded

---

## 10. GUI/WPF Tests

Test WPF infrastructure without displaying windows. Verifies types, XAML, and wiring.

### WPF Assembly Loading

```powershell
It 'loads WPF assemblies without error' {
    { Add-Type -AssemblyName PresentationFramework } | Should -Not -Throw
    [System.Windows.Window] | Should -Not -BeNullOrEmpty
}
```

### C# Type Compilation

```powershell
It 'compiles ViewModel types' {
    Initialize-XXGuiFramework  # Your Add-Type wrapper function
    [XXNamespace.MainViewModel] | Should -Not -BeNullOrEmpty
    [XXNamespace.RelayCommand] | Should -Not -BeNullOrEmpty
    [XXNamespace.ViewModelBase] | Should -Not -BeNullOrEmpty
}
```

### XAML Parsing (No Window Display)

```powershell
It 'parses MainWindow XAML without errors' {
    $xamlPath = Join-Path $projectRoot 'src/GUI/XAML/MainWindow.xaml'
    $xamlContent = Get-Content -LiteralPath $xamlPath -Raw -Encoding utf8
    $reader = [System.Xml.XmlReader]::Create([System.IO.StringReader]::new($xamlContent))
    { [System.Windows.Markup.XamlReader]::Load($reader) } | Should -Not -Throw
}
```

### ViewModel Property Verification

```powershell
It 'ViewModel has required properties' {
    $vm = [XXNamespace.MainViewModel]::new()
    $props = $vm.GetType().GetProperties()
    $propNames = $props.Name
    $propNames | Should -Contain 'ConfigPath'
    $propNames | Should -Contain 'IsRunning'
}
```

### Important Limitation

> **Pester CANNOT test WPF delegate scope issues.** Tests run in the same session scope
> where variable resolution works fine. The `$global:` context pattern bug (where
> `.GetNewClosure()` fails with C# delegates) only manifests when the WPF dispatcher
> invokes scriptblocks through the C# delegate chain. This requires manual GUI testing.

---

## 11. Test Data Patterns

### Files with Known Content

```powershell
$testContent = "Test file content $(Get-Random)"
$testFile = Join-Path $script:TestDir "file_$(Get-Random).txt"
Set-Content -LiteralPath $testFile -Value $testContent -Encoding utf8
```

### Files with Known Hash

```powershell
$knownContent = 'ExactContentForHashTest'
Set-Content -LiteralPath $testFile -Value $knownContent -NoNewline -Encoding utf8
$expectedHash = (Get-FileHash -LiteralPath $testFile -Algorithm SHA256).Hash
```

### Config Objects for Testing

```powershell
$testConfig = @{
    general = @{ logDirectory = './logs'; logRetentionDays = 15 }
    backup = @{
        fileCopyJobs = @()
        folderCopyJobs = @()
        features = @{ fileCopy = $true; folderCopy = $true; verification = $true; hashAlgorithm = 'SHA256' }
        versioning = @{ enabled = $false; maxVersions = 3; timestampFormat = 'yyyyMMdd_HHmmss' }
    }
    email = @{ enabled = $false; smtpServer = ''; smtpPort = 587; useSsl = $true; from = ''; to = ''; username = ''; credentialTarget = '' }
}
$testConfig | ConvertTo-Json -Depth 5 | Set-Content -LiteralPath $configPath -Encoding utf8
```

### Random Names

Always use `Get-Random` in test file/directory names:

```powershell
# Prevents collisions across parallel or repeated runs
$fileName = "testfile_$(Get-Random).txt"
$dirName = "AM_Tests_$(Get-Random)"
```

---

## 12. Pester 5.x Configuration

### Programmatic Configuration

```powershell
$config = New-PesterConfiguration

# Run settings
$config.Run.Path = './tests'
$config.Run.PassThru = $true          # Return result object
$config.Run.Exit = $true              # Exit with test result code (for CI)

# Filter settings
$config.Filter.Tag = 'Unit'           # Only run tagged tests
$config.Filter.ExcludeTag = 'Slow'    # Skip slow tests

# Output settings
$config.Output.Verbosity = 'Detailed' # Normal, Detailed, Diagnostic

# Test result output (NUnit XML)
$config.TestResult.Enabled = $true
$config.TestResult.OutputPath = './test-results.xml'

# Code coverage
$config.CodeCoverage.Enabled = $true
$config.CodeCoverage.Path = './src'

$result = Invoke-Pester -Configuration $config
```

### Common Invocations

```powershell
# Quick run — all tests, detailed output
Invoke-Pester -Path ./tests -Output Detailed

# Specific file
Invoke-Pester -Path ./tests/Configuration.Tests.ps1 -Output Detailed

# CI mode — exit code reflects pass/fail
Invoke-Pester -Configuration @{ Run = @{ Path = './tests'; Exit = $true } }
```

---

## 13. Quality Gate Integration

Running Pester from inside the quality gate script:

```powershell
$testDir = Join-Path -Path $projectRoot -ChildPath 'tests'
$testFiles = @(Get-ChildItem -LiteralPath $testDir -Filter '*.Tests.ps1' -File)

if ($testFiles.Count -eq 0) {
    Write-Host 'No test files found' -ForegroundColor Yellow
}
else {
    Import-Module Pester -ErrorAction Stop
    $pesterConfig = New-PesterConfiguration
    $pesterConfig.Run.Path = $testDir
    $pesterConfig.Run.PassThru = $true
    $pesterConfig.Output.Verbosity = 'Detailed'

    $pesterResult = Invoke-Pester -Configuration $pesterConfig

    if ($pesterResult.FailedCount -eq 0) {
        Write-Host "PASS  All $($pesterResult.TotalCount) tests passed" -ForegroundColor Green
    }
    else {
        Write-Host "FAIL  $($pesterResult.FailedCount) of $($pesterResult.TotalCount) tests failed" -ForegroundColor Red
        exit 1
    }
}
```

### Notes

- **Don't use `-CI`** in quality gate — it exits the process, preventing the summary from printing
- **Use `-PassThru`** to get the result object for programmatic inspection
- **Check `FailedCount`** rather than exit code for more flexibility

---

## 14. Test Organization

### File Mapping

| Test File | Tests For | Type |
|-----------|-----------|------|
| `Configuration.Tests.ps1` | `Configuration.ps1` | Unit |
| `BackupEngine.Tests.ps1` | `BackupEngine.ps1` | Unit |
| `Versioning.Tests.ps1` | `Versioning.ps1` | Unit |
| `EmailReport.Tests.ps1` | `EmailReport.ps1` | Unit + Mock |
| `GUI.Tests.ps1` | GUI types, XAML, ViewModel | Unit |
| `Integration.Tests.ps1` | Cross-module workflows | Integration |
| `Build.Tests.ps1` | Release packaging | Build |
| `Migration.Tests.ps1` | Migration functions | Unit + Integration |

### One Test File Per Source Module

```
src/Core/Configuration.ps1    →  tests/Configuration.Tests.ps1
src/Core/Logging.ps1          →  tests/Logging.Tests.ps1 (or included in Integration)
src/Email/EmailReport.ps1     →  tests/EmailReport.Tests.ps1
```

### Integration Tests Are Separate

Integration tests verify real cross-module interactions. They use real file I/O, real config, and real function chains. Keep them separate from unit tests for faster development iteration.

### Quality Gate Is NOT a Test File

`Invoke-QualityCheck.ps1` is a runner, not a Pester test file. It calls `Invoke-Pester` among other checks. It should not be named `*.Tests.ps1`.

---

## 15. Output Suppression

Suppress noisy output from module functions during testing:

```powershell
# Suppress New-Item output
$null = New-Item -Path $dir -ItemType Directory -Force

# Suppress Import-Module output
Import-Module $modulePath -Force | Out-Null

# Suppress function output you don't need
$null = New-XXConfig -Force

# For Write-Host output (can't be suppressed with $null =):
# Use -InformationAction SilentlyContinue if function uses Write-Information
# Write-Host output will still appear — this is expected in Pester output
```

### Anti-Pattern: Capturing to Variable You Don't Use

```powershell
# BAD — creates unnecessary variable
$result = New-Item -Path $dir -ItemType Directory -Force

# GOOD — suppress explicitly
$null = New-Item -Path $dir -ItemType Directory -Force
```

---

## 16. ShouldProcess Testing

Test functions that use `SupportsShouldProcess`:

```powershell
Describe 'Set-XXConfig' {
    It 'supports -WhatIf' {
        $config = Get-XXConfig -Force
        $config.general.logRetentionDays = 999

        $savePath = Join-Path $script:TestDir 'whatif.json'
        Set-XXConfig -Config $config -ConfigPath $savePath -WhatIf

        # File should NOT be created
        Test-Path -LiteralPath $savePath | Should -BeFalse
    }

    It 'saves when confirmed' {
        $config = Get-XXConfig -Force
        $savePath = Join-Path $script:TestDir 'confirmed.json'
        Set-XXConfig -Config $config -ConfigPath $savePath -Confirm:$false

        Test-Path -LiteralPath $savePath | Should -BeTrue
    }
}
```

---

## 17. Parameterized Tests (TestCases)

Run the same test logic with different input data:

```powershell
Describe 'Test-XXConfig' {
    It 'catches invalid <Field>' -TestCases @(
        @{ Field = 'logRetentionDays'; Value = 0; Section = 'general' }
        @{ Field = 'logRetentionDays'; Value = -5; Section = 'general' }
        @{ Field = 'smtpPort'; Value = 99999; Section = 'email' }
        @{ Field = 'maxVersions'; Value = 0; Section = 'backup.versioning' }
    ) {
        param($Field, $Value, $Section)

        $config = New-XXConfig -Force
        # Navigate to the section and set the invalid value
        $target = $config
        foreach ($part in $Section.Split('.')) { $target = $target.$part }
        $target.$Field = $Value

        $result = Test-XXConfig -Config $config
        $result.IsValid | Should -BeFalse
        $result.Messages | Where-Object { $_ -match $Field } | Should -Not -BeNullOrEmpty
    }
}
```

### Simpler Parameterization

```powershell
It 'handles hash algorithm <Algorithm>' -TestCases @(
    @{ Algorithm = 'MD5' }
    @{ Algorithm = 'SHA256' }
) {
    param($Algorithm)
    $result = Test-XXFileHash -Path $testFile -Algorithm $Algorithm
    $result.Hash | Should -Not -BeNullOrEmpty
}
```

---

## 18. Debugging Failed Tests

### Increase Verbosity

```powershell
Invoke-Pester -Path ./tests/Failing.Tests.ps1 -Output Diagnostic
```

### Check `$Error` After Failures

```powershell
It 'debug test' {
    try {
        $result = Do-Something
    }
    catch {
        Write-Host "Error: $_"
        Write-Host "Stack: $($_.ScriptStackTrace)"
        throw
    }
}
```

### Common Failure Causes

| Symptom | Likely Cause |
|---------|-------------|
| "Command not found" | Module not imported in BeforeAll |
| Mock not working, real function runs | Missing `-ModuleName` on Mock |
| Config values wrong from previous test | No isolation — state leaked |
| Tests pass alone, fail together | Order dependency — each test must set up its own state |
| Temp dir collision (intermittent) | Missing `Get-Random` in directory name |
| "File in use" on cleanup | Previous test didn't release handle |
| GUI test hangs | Window opened — use XAML parsing only, not window display |
| `$variable` is `$null` in InModuleScope | Didn't pass via `-Parameters` |

### Run Single Test

```powershell
# Run one specific test file
Invoke-Pester -Path ./tests/Configuration.Tests.ps1 -Output Detailed

# Run tests matching a name filter
Invoke-Pester -Path ./tests -Output Detailed -Filter @{ FullName = '*Config*' }
```

---

## 19. Gotchas

| # | Gotcha | Symptom | Fix |
|---|--------|---------|-----|
| 1 | Missing `-ModuleName` on Mock | Mock not applied, real function runs | Always specify `-ModuleName` when mocking inside modules |
| 2 | Test modifies `$script:` module state | Next test fails with wrong state | Snapshot + restore in BeforeAll/AfterAll |
| 3 | Temp dir name collision | Tests fail intermittently | Include `Get-Random` in directory name |
| 4 | Config path not redirected | Test writes to production config | InModuleScope redirect + restore |
| 5 | File still locked in AfterAll | `Remove-Item` fails, temp files left | `-ErrorAction SilentlyContinue` on cleanup |
| 6 | `Should -Throw` without scriptblock | Assertion doesn't catch the throw | Wrap: `{ code } \| Should -Throw` |
| 7 | `Should -Invoke` without `-Exactly` | Passes even with extra calls | Use `-Times N -Exactly` for precise counts |
| 8 | Pester 4 syntax in Pester 5 | `Assert-MockCalled` not found | Use `Should -Invoke` (Pester 5 syntax) |
| 9 | `$script:` test scope vs module scope | Variable not accessible | `InModuleScope` for module's `$script:` variables |
| 10 | BeforeAll runs once per Describe | Setup not repeated per test | Use `BeforeEach` for per-test setup if needed |
| 11 | GUI tests display windows | Tests hang waiting for user input | Parse XAML, test types — never show windows |
| 12 | WPF delegate scope bugs pass | Tests work, GUI breaks | Pester can't catch this — manual GUI testing required |
| 13 | Order-dependent tests | Pass alone, fail in batch | Each test sets up its own preconditions |
| 14 | Output pollution from module | Test output is cluttered | `$null =` or `\| Out-Null` on noisy calls |
| 15 | `ConvertTo-Json` depth in test config | Nested objects become strings | Always use `-Depth 5` or higher |
| 16 | Global helper functions leak | Affect other test files | `Remove-Item Function:\HelperName` in AfterAll |
| 17 | `-Force` missing on config reload | Cached stale config used in test | `Get-XXConfig -Force` after config changes |
| 18 | `@()` missing on cmdlet output | `.Count` returns `$null` for single item | Wrap: `@(Get-ChildItem ...).Count` |
| 19 | Test creates process, doesn't stop it | Process orphaned after test | Clean up spawned processes in AfterAll |
| 20 | Module re-import after build test | Module loaded from staging, not source | Re-import from source in AfterAll |
| 21 | `Mock Get-PhysicalDisk` throws ParseException | Pester can't parse the cmdlet's internal parameter type | Wrap in a private `Get-SI*` helper function; mock the wrapper (see Section 7) |
| 22 | `[AllowEmptyCollection()]` missing on mandatory array | Empty array rejected before function body runs | Add `[AllowEmptyCollection()]` above the param declaration |
| 23 | `Invoke-Pester` returns `$null` | `.FailedCount` throws null reference | Set `$config.Run.PassThru = $true` before calling `Invoke-Pester` |

---

## 20. Quick-Reference Checklist

When writing a new test file:

- [ ] `#Requires -Version 7.0` and `#Requires -Modules Pester` at top
- [ ] Module imported via `.psd1` in `BeforeAll`
- [ ] Temp directory with `Get-Random` in name
- [ ] Config redirected via `InModuleScope` (if config-dependent tests)
- [ ] Original state snapshot for restoration
- [ ] All cleanup in `AfterAll` with `-ErrorAction SilentlyContinue`
- [ ] `Describe` blocks named after function/feature under test
- [ ] `Context` blocks for each scenario
- [ ] `It` blocks with descriptive behavior-focused names
- [ ] `-ModuleName` on all `Mock` calls targeting module functions
- [ ] `-Exactly` on `Should -Invoke` when count matters
- [ ] `$null =` on noisy function calls
- [ ] Global helper functions removed in `AfterAll`
- [ ] No hardcoded paths — all relative to `$PSScriptRoot` or temp dirs
- [ ] Tests are order-independent — each sets up its own state

---

## Verified Patterns from Implementation

### Shared test helpers [verified in Phase 3]
Extract repeated `BeforeAll` boilerplate (log session init, AppX cmdlet stubs) into a `TestHelpers.ps1` file. Dot-source in each test file's `BeforeAll`. Prevents drift across 5+ test files.

### BeforeEach for mutable shared state [verified in Phase 3]
When tests share a manifest or state object that gets mutated during test execution, use `BeforeEach` to reset it. Otherwise test 2 sees test 1's mutations.

### Mocking native executables (powercfg, winget) [verified in Phase 5]
Mocked native executables are PowerShell functions, not real .exe calls. They don't set `$LASTEXITCODE`. If the source code checks `$LASTEXITCODE`, tests must set `$global:LASTEXITCODE = 0` in `BeforeEach` to prevent stale values from prior native calls causing false failures.

### Stubbing unavailable modules (PSWindowsUpdate, ScheduledTasks) [verified in Phase 5]
When testing against cmdlets from modules that may not be installed (PSWindowsUpdate) or have CimInstance type validation (ScheduledTasks), define `global:` stub functions in `BeforeAll` before mocking. The stubs provide the function signature that Pester can mock without triggering parameter type validation from the real cmdlet.
