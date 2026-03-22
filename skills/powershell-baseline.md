# PowerShell Skill — Comprehensive Baseline v2

## Purpose
This skill provides Claude with comprehensive PowerShell best practices, patterns, and gotchas to produce correct, robust, and idiomatic PowerShell code. It serves as a **general baseline** — project-specific requirements should be compared against this document to identify gaps that need session-scoped or permanent additions.

## Target Environment
- **Minimum OS:** Windows 10 / Windows Server 2016+ (older OS support only if user explicitly specifies)
- **Primary:** Windows PowerShell 5.1 (ships with Windows 10/11, no installation required)
- **Secondary:** PowerShell 7+ (cross-platform, separate install) — note differences where they matter
- Unless the user specifies otherwise, assume Windows PowerShell 5.1 and avoid 7+-only features

---

## 1. Script Structure and Organization

### Script Template (Advanced Functions)
```powershell
#Requires -Version 5.1
# Optional: #Requires -RunAsAdministrator
# Optional: #Requires -Modules ActiveDirectory

<#
.SYNOPSIS
    Brief one-line description.
.DESCRIPTION
    Detailed description of what the script does.
.PARAMETER ParamName
    Description of the parameter.
.EXAMPLE
    .\Script.ps1 -ParamName "value"
    Description of what this example does.
.NOTES
    Author: Name
    Version: 1.0
    Date: YYYY-MM-DD
#>

[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, ValueFromPipeline)]
    [ValidateNotNullOrEmpty()]
    [string]$InputPath,

    [Parameter()]
    [ValidateSet('Option1', 'Option2')]
    [string]$Mode = 'Option1'
)

begin {
    # One-time setup: import modules, initialize collections, validate environment
    Set-StrictMode -Version Latest
    $ErrorActionPreference = 'Stop'
}

process {
    # Per-pipeline-item logic goes here
}

end {
    # Cleanup, summary output
}
```

### Key Structural Rules
- **Always use `[CmdletBinding()]`** on anything beyond a throwaway snippet — it enables `-Verbose`, `-Debug`, `-ErrorAction`, `-WhatIf` support
- **Use `SupportsShouldProcess`** on any script that modifies state (files, registry, services, etc.)
- **Use `begin/process/end`** blocks when accepting pipeline input; otherwise a simple sequential body is fine
- **Use `Set-StrictMode -Version Latest`** during development to catch uninitialized variables and property typos — but be aware it can break some third-party modules
- **Comment-based help** (the `<# .SYNOPSIS #>` block) is not optional for anything meant to be reused
- **Every `function Verb-Noun` needs its own `.SYNOPSIS` block** — including private helpers defined mid-file. A file-level block only covers functions near the top of the file; quality gates scan backward from the `function` keyword and stop at unrelated `}` braces from earlier functions
- **`[SuppressMessageAttribute()]` goes inside the function body**, before `[CmdletBinding()]` — NOT before the `function` keyword. PSScriptAnalyzer ignores attributes placed outside the function block:
```powershell
# CORRECT — inside function body
function Get-FooItems {
    [Diagnostics.CodeAnalysis.SuppressMessageAttribute('PSUseSingularNouns', '')]
    [CmdletBinding()]
    param()
    # ...
}

# WRONG — outside function body, silently ignored by PSScriptAnalyzer
[Diagnostics.CodeAnalysis.SuppressMessageAttribute('PSUseSingularNouns', '')]
function Get-FooItems {
    [CmdletBinding()]
    param()
}
```

### Script Path Variables
```powershell
# $PSScriptRoot — directory containing the currently running script
# Essential for portable scripts that reference config files, sibling scripts, logs
$configPath = Join-Path -Path $PSScriptRoot -ChildPath 'config.json'
$logDir     = Join-Path -Path $PSScriptRoot -ChildPath 'logs'

# $PSCommandPath — full path to the currently running script file
# Useful for self-referencing (logging which script ran, relaunch scenarios)
```
**Rule:** Never hardcode paths like `C:\Scripts\config.json`. Use `$PSScriptRoot` for relative references.

### Script vs. Function vs. Module Decision
| Scope | Use |
|-------|-----|
| One-off task | Script (`.ps1`) |
| Reusable logic within a script | Function (defined in the same `.ps1`) |
| Shared across scripts/sessions | Module (`.psm1` + `.psd1` manifest) |
| User-facing tool with multiple commands | Module with exported functions |

---

## 2. Parameter Handling and Validation

### Validation Attributes (Use These — Don't Hand-Roll Validation)
```powershell
[ValidateNotNullOrEmpty()]          # Rejects $null and ''
[ValidateSet('A', 'B', 'C')]       # Restricts to specific values (tab-completable)
[ValidateRange(1, 100)]            # Numeric range
[ValidatePattern('^\d{3}-\d{4}$')] # Regex match
[ValidateScript({ Test-Path $_ })] # Arbitrary validation (return $true or throw)
[ValidateLength(1, 255)]           # String length
[ValidateCount(1, 10)]             # Array element count
[AllowNull()]                      # Explicitly permit $null (for optional typed params)
[AllowEmptyString()]               # Explicitly permit '' on [string] params
[AllowEmptyCollection()]           # Required on [Parameter(Mandatory)] [object[]] / [string[]] / List<T>
                                   # when @() is valid input — without it, PS 7 throws
                                   # "Cannot bind argument to parameter because it is an empty collection"
```

### Common Parameter Patterns
```powershell
# Path parameter that must exist
[ValidateScript({ Test-Path -LiteralPath $_ })]
[string]$Path

# Optional parameter with default
[string]$LogPath = (Join-Path $env:TEMP "script.log")

# Switch parameter (boolean flag)
[switch]$Force

# Credential parameter (secure by default — prompts if not supplied)
[System.Management.Automation.PSCredential]
[System.Management.Automation.Credential()]
$Credential = [System.Management.Automation.PSCredential]::Empty
```

### Parameter Sets
Use when a function has mutually exclusive operating modes:
```powershell
[CmdletBinding(DefaultParameterSetName = 'ByName')]
param(
    [Parameter(ParameterSetName = 'ByName', Mandatory)]
    [string]$Name,

    [Parameter(ParameterSetName = 'ById', Mandatory)]
    [int]$Id
)
# Check which set is active: $PSCmdlet.ParameterSetName
```

---

## 3. Splatting (Clean Parameter Passing)

Splatting is fundamental to writing readable PowerShell. Use it anywhere a cmdlet call exceeds ~80 characters or has conditional parameters.

```powershell
# Instead of long lines with backtick continuation:
$params = @{
    LiteralPath = $path
    Destination = $dest
    Recurse     = $true
    Force       = $true
    ErrorAction = 'Stop'
}
Copy-Item @params    # Note: @ not $ when splatting

# Conditional parameters — build the hashtable dynamically
$params = @{
    LiteralPath = $path
    Encoding    = 'UTF8'
}
if ($append) { $params['Append'] = $true }
Add-Content @params -Value $data

# Splatting for external programs — use an array
$processArgs = @('--required-arg', 'value')
if ($verbose) { $processArgs += '--verbose' }
if ($dryRun)  { $processArgs += '--dry-run' }
& "app.exe" @processArgs
```

---

## 4. Error Handling

### The Error Handling Model
PowerShell has **terminating** and **non-terminating** errors. This is the single most common source of bugs.

- **Non-terminating:** Most cmdlet errors (file not found, access denied). These write to `$Error` but **do not trigger `catch` blocks** by default.
- **Terminating:** `throw`, `ThrowTerminatingError()`, and cmdlet errors when `-ErrorAction Stop` is set.
- **`$ErrorActionPreference = 'Stop'`** makes all errors terminating — set this at the top of scripts that use try/catch.

### Patterns
```powershell
# CORRECT: Errors will be caught
$ErrorActionPreference = 'Stop'
try {
    Get-Content -LiteralPath $path
    Remove-Item -LiteralPath $target
} catch [System.IO.FileNotFoundException] {
    Write-Warning "File not found: $($_.Exception.Message)"
} catch [System.UnauthorizedAccessException] {
    Write-Warning "Access denied: $($_.TargetObject)"
} catch {
    Write-Warning "Unexpected error: $($_.Exception.Message)"
    # Re-throw if you can't handle it
    throw
} finally {
    # Always runs — cleanup open handles, connections, etc.
}

# WRONG: Non-terminating errors silently bypass catch
try {
    Get-Content -LiteralPath $path   # If file missing, error prints but catch never fires
} catch {
    # This block never executes for non-terminating errors
}
```

### Per-Cmdlet Error Action
When you don't want global `Stop` behavior:
```powershell
# Suppress errors for a single call
$result = Get-Item -LiteralPath $path -ErrorAction SilentlyContinue
if ($null -eq $result) {
    # Handle missing item
}

# Force terminating for a single call (even if global preference is Continue)
Remove-Item -LiteralPath $path -ErrorAction Stop
```

### Error Logging
```powershell
# $_ inside a catch block is the ErrorRecord
catch {
    $errorDetails = [PSCustomObject]@{
        Timestamp  = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
        Message    = $_.Exception.Message
        Category   = $_.CategoryInfo.Category
        Target     = $_.TargetObject
        ScriptLine = $_.InvocationInfo.ScriptLineNumber
        StackTrace = $_.ScriptStackTrace
    }
    $errorDetails | Export-Csv -LiteralPath $logPath -Append -NoTypeInformation -Encoding UTF8
}
```

---

## 5. Output and Logging

### Write-* Cmdlet Selection
| Cmdlet | Stream | Purpose | Visible by default? |
|--------|--------|---------|-------------------|
| `Write-Output` | Success (1) | Return data from functions | Yes |
| `Write-Host` | Information (6) | User-facing display only | Yes |
| `Write-Verbose` | Verbose (4) | Diagnostic detail | No (use `-Verbose`) |
| `Write-Debug` | Debug (5) | Developer diagnostics | No (use `-Debug`) |
| `Write-Warning` | Warning (3) | Non-fatal issues | Yes |
| `Write-Error` | Error (2) | Non-terminating errors | Yes |
| `Write-Information` | Information (6) | Structured info (5.0+) | Depends on preference |

### Critical Rule: Write-Host vs Write-Output
```powershell
# Write-Output goes to the SUCCESS stream — it's your function's RETURN VALUE
# Write-Host goes to the console ONLY — it's for human-readable display

function Get-Data {
    Write-Host "Processing..."         # Display only — not capturable
    Write-Output $results              # This is what the caller receives
}

# MISTAKE: Using Write-Host for data
function Get-Data {
    Write-Host $results   # Caller gets NOTHING — $data will be $null
}
$data = Get-Data  # $null!

# In interactive scripts (menus, prompts): Write-Host is fine
# In functions that return data: Use Write-Output (or just output the object)
```

### Logging to File
```powershell
# Transcript (captures console output — quick and dirty)
Start-Transcript -Path $logFile -Append
# ... script runs ...
Stop-Transcript

# Structured logging function
function Write-Log {
    param(
        [string]$Message,
        [ValidateSet('INFO', 'WARN', 'ERROR')]
        [string]$Level = 'INFO'
    )
    $entry = "{0} [{1}] {2}" -f (Get-Date -Format 'yyyy-MM-dd HH:mm:ss'), $Level, $Message
    Add-Content -LiteralPath $script:LogPath -Value $entry -Encoding UTF8
    if ($Level -eq 'ERROR') { Write-Warning $Message }
    elseif ($Level -eq 'WARN') { Write-Warning $Message }
    else { Write-Verbose $Message }
}
```

### Progress Bars
```powershell
# Write-Progress for long-running operations
$items = Get-ChildItem -LiteralPath $dir -Recurse -File
$total = $items.Count
$i = 0

foreach ($item in $items) {
    $i++
    $percentComplete = [math]::Round(($i / $total) * 100, 0)
    Write-Progress -Activity 'Processing files' -Status "$i of $total" `
        -PercentComplete $percentComplete -CurrentOperation $item.Name

    # ... do work ...
}
Write-Progress -Activity 'Processing files' -Completed   # Clear the progress bar
```

---

## 6. String Handling

### Operators
```powershell
# Case-insensitive by default (PowerShell convention)
'Hello' -eq 'hello'          # True
'Hello' -ceq 'hello'         # False (case-sensitive variant)
'Hello' -like '*ell*'        # Wildcard match
'Hello' -match '^H\w+'       # Regex match — populates $Matches
'Hello' -replace 'ello','i'  # Regex replace: "Hi"
'Hello' -creplace 'H','h'    # Case-sensitive replace

# -match populates the automatic $Matches variable
if ('S01xE06' -match 'S(\d+)xE(\d+)') {
    $season  = $Matches[1]   # "01"
    $episode = $Matches[2]   # "06"
}

# -split (regex by default)
'one, two, three' -split ',\s*'  # @('one', 'two', 'three')
```

### String Formatting
```powershell
# Format operator (printf-style)
'{0:N2} out of {1}' -f 3.14159, 10     # "3.14 out of 10"
'{0:yyyy-MM-dd}' -f (Get-Date)          # "2025-01-15"
'0x{0:X8}' -f 255                       # "0x000000FF"

# Here-strings (multi-line)
$block = @"
Line 1 with $variable interpolation
Line 2
"@

# Literal here-string (no interpolation)
$block = @'
Line 1 with $literal dollar sign
Line 2
'@

# String interpolation — subexpressions for anything beyond simple variables
"File count: $($files.Count)"
"Today is $(Get-Date -Format 'dddd')"
# WRONG: "File count: $files.Count"  — only interpolates $files, then appends ".Count" as literal
```

### .NET String Methods (When Operators Aren't Enough)
```powershell
$s = "  Hello, World!  "
$s.Trim()                          # "Hello, World!"
$s.Contains('World')               # True (CASE-SENSITIVE in .NET — unlike PS operators!)
$s.IndexOf('World')                # 9
$s.Substring(2, 5)                 # "Hello"
$s.Replace('World', 'PowerShell')  # Simple literal replace (no regex)
$s.Split(',')                      # @("  Hello", " World!  ")
$s.PadLeft(30, '-')                # Pad to width
[string]::IsNullOrWhiteSpace($s)   # False

# For case-insensitive .NET contains:
$s.IndexOf('world', [StringComparison]::OrdinalIgnoreCase) -ge 0
```

---

## 7. Collections and Pipeline

### Array Gotchas (The #1 Source of Subtle Bugs)
```powershell
# GOTCHA: A cmdlet returning 0 items returns $null
$files = Get-ChildItem -LiteralPath $dir -Filter "*.log"
$files.Count  # If 0 files: $files is $null — .Count gives 0 but is unreliable in strict mode
              # If 1 file: .Count returns 1 (intrinsic member, PS 3+)
              # But $null behavior is inconsistent — @() wrap prevents all ambiguity

# FIX: Force array context
$files = @(Get-ChildItem -LiteralPath $dir -Filter "*.log")
$files.Count  # Always works reliably: 0, 1, or N

# GOTCHA: += on arrays is O(n) — creates a new array every time
$results = @()
foreach ($item in $bigList) {
    $results += $item  # Catastrophically slow for large collections
}

# FIX: Use ArrayList, Generic List, or collect from foreach output
$results = [System.Collections.Generic.List[object]]::new()
foreach ($item in $bigList) {
    $results.Add($item)  # O(1) amortized
}

# BEST: Let PowerShell collect output naturally
$results = foreach ($item in $bigList) {
    $item  # Output is collected into an array automatically
}

# GOTCHA: $null in pipeline — nuanced behavior
# A bare $null variable piped produces 0 items (swallowed):
$nada = $null
$nada | ForEach-Object { "Got: $_" }  # Outputs nothing

# But $null inside an array is passed through as an empty value:
@(1, $null, 3) | ForEach-Object { "Item: [$_]" }
# Outputs: "Item: [1]", "Item: []", "Item: [3]" — $null becomes empty string

# GOTCHA: Measure-Object on empty collections returns $null, not 0
$emptyFiles = @()
($emptyFiles | Measure-Object -Property Length -Sum).Sum  # Returns $null, NOT 0

# FIX (PS 7+): null-coalescing
$total = [long](($items | Measure-Object -Property Length -Sum).Sum ?? 0)

# FIX (PS 5.1): explicit guard
$measure = ($items | Measure-Object -Property Length -Sum)
$total = if ($measure.Count -gt 0) { [long]$measure.Sum } else { [long]0 }
```

### Hashtables and Ordered Dictionaries
```powershell
# Standard hashtable (unordered)
$config = @{
    Server   = 'db01'
    Database = 'AppDB'
    Timeout  = 30
}

# Ordered dictionary (preserves insertion order)
$config = [ordered]@{
    Server   = 'db01'
    Database = 'AppDB'
    Timeout  = 30
}

# Case-insensitive by default (PowerShell hashtables)
$config['server']    # Works — same as $config['Server']

# Case-sensitive hashtable (rare, but available)
$config = New-Object System.Collections.Hashtable ([StringComparer]::Ordinal)
```

### Pipeline Best Practices
```powershell
# RULE: Filter LEFT, format RIGHT
# Do filtering and selection as early as possible in the pipeline

# GOOD: Filter at the source
Get-ChildItem -LiteralPath $dir -Filter "*.log" -Recurse |
    Where-Object { $_.Length -gt 1MB } |
    Select-Object Name, Length, LastWriteTime

# BAD: Get everything, then filter
Get-ChildItem -LiteralPath $dir -Recurse |
    Where-Object { $_.Extension -eq '.log' -and $_.Length -gt 1MB }

# -Filter is faster than Where-Object (processed by the provider, not PowerShell)

# CRITICAL: Format-* cmdlets DESTROY objects — they output formatting instructions, not data
# WRONG:
Get-Process | Format-Table | Export-Csv ...   # Exports garbage
# RIGHT:
Get-Process | Select-Object Name, CPU | Export-Csv ...
# RULE: Format-* cmdlets are TERMINAL — only use at the end of a pipeline for display
```

### Custom Objects
```powershell
# PSCustomObject — the idiomatic choice for most scripts
[PSCustomObject]@{
    Name   = 'Widget'
    Count  = 42
    Status = 'Active'
}

# Add members to existing objects
$obj | Add-Member -NotePropertyName 'NewProp' -NotePropertyValue 'value'

# PowerShell 5+ class keyword is available for complex scenarios
# but PSCustomObject is the default choice for script output
```

---

## 8. File System Operations

### Critical: Always Use `-LiteralPath` Instead of `-Path`
```powershell
# -Path interprets wildcards: brackets, asterisks, question marks
Get-Item -Path "C:\Files\report[1].txt"        # FAILS — treats [1] as wildcard
Get-Item -LiteralPath "C:\Files\report[1].txt"  # Works correctly

# Only use -Path when you INTENTIONALLY want wildcard expansion
Get-ChildItem -Path "C:\Logs\*.log"
```
**Rule:** Default to `-LiteralPath` for all file operations unless you specifically need globbing.

### Path Construction
```powershell
# CORRECT: Use Join-Path (handles trailing slashes, OS separators)
$fullPath = Join-Path -Path $baseDir -ChildPath $filename

# CORRECT: .NET methods for complex manipulation
$nameOnly = [System.IO.Path]::GetFileNameWithoutExtension($file)
$ext      = [System.IO.Path]::GetExtension($file)
$dir      = [System.IO.Path]::GetDirectoryName($file)
$combined = [System.IO.Path]::Combine($dir, $subdir, $filename)

# Use $PSScriptRoot for paths relative to the script's own location
$configPath = Join-Path -Path $PSScriptRoot -ChildPath 'config.json'

# WRONG: String concatenation (breaks on missing/double slashes)
$fullPath = $baseDir + "\" + $filename
```

### Script Source File Encoding (PS 5.1 Parse-Time Gotcha)

Windows PowerShell 5.1 reads `.ps1` files without a BOM as **Windows-1252 (CP1252)**,
not UTF-8. Non-ASCII characters in the script source — including in comments — are
misinterpreted at parse time and can cause cryptic errors like "string is missing the
terminator" before the script runs a single line.

The most common trigger: em dashes (U+2014) copied from documentation or editors that
save UTF-8 without BOM. The UTF-8 bytes for U+2014 are `E2 80 94`. In CP1252, `0x94`
maps to the right double quotation mark, which PowerShell treats as a string terminator.

**Fix:** Keep script source **pure ASCII**. Replace em dashes with `--`, smart quotes
with straight quotes, and any other non-ASCII with ASCII equivalents. Editors that
default to UTF-8 without BOM (VS Code, many others) will silently produce this issue.

To verify a script is clean before deploying to PS 5.1 machines:
```powershell
# Find non-ASCII characters in a script file
$bytes = [System.IO.File]::ReadAllBytes('.\script.ps1')
$nonAscii = $bytes | Where-Object { $_ -gt 127 }
if ($nonAscii) { Write-Warning "Non-ASCII bytes found: $($nonAscii.Count)" }
```

### File Encoding (Major Gotcha)
```powershell
# Windows PowerShell 5.1 defaults:
#   Out-File / > operator:  UTF-16 LE (BOM) -- breaks many Unix tools, git diffs
#   Set-Content:            system's active code page (usually Windows-1252)
#   Export-Csv:             ASCII (loses non-ASCII characters!)

# ALWAYS specify encoding explicitly:
$data | Out-File -LiteralPath $path -Encoding UTF8
$data | Set-Content -LiteralPath $path -Encoding UTF8
# -NoTypeInformation is required in 5.1 to suppress the #TYPE header line
# that Export-Csv prepends by default. Without it, most CSV consumers choke.
# (PS 7+ changed the default — the flag is no longer needed but doesn't hurt.)
$data | Export-Csv -LiteralPath $path -Encoding UTF8 -NoTypeInformation

# PowerShell 7+ defaults to UTF-8 NoBOM — much saner, but don't assume it

# For BOM-less UTF-8 in 5.1 (needed for some tools):
[System.IO.File]::WriteAllText($path, $content, [System.Text.UTF8Encoding]::new($false))
```

### Reading File Content
```powershell
# Get-Content returns an ARRAY OF LINES by default, not a single string
$lines = Get-Content -LiteralPath $path                  # Array of strings
$text  = Get-Content -LiteralPath $path -Raw              # Single string (entire file)
$text  = [System.IO.File]::ReadAllText($path)             # .NET alternative

# If you do string operations on Get-Content output without -Raw,
# you're operating on individual lines — which is often not what you want.
# Example: $content.IndexOf('search term') won't find terms that span lines
```

### Robust File Operations
```powershell
# Check before acting
if (Test-Path -LiteralPath $path -PathType Leaf) { ... }     # File exists
if (Test-Path -LiteralPath $path -PathType Container) { ... } # Directory exists

# Safe directory creation (no error if exists)
$null = New-Item -ItemType Directory -Path $dir -Force

# Safe file copy with overwrite protection
if (-not (Test-Path -LiteralPath $dest)) {
    Copy-Item -LiteralPath $source -Destination $dest
} else {
    Write-Warning "Destination already exists: $dest"
}

# Recursive operations — be explicit about depth when possible
Get-ChildItem -LiteralPath $dir -Recurse -Depth 3 -File
```

### File Locking and Concurrent Access
```powershell
# Retry pattern for locked files
$maxRetries = 3
$retryDelay = 1  # seconds
for ($i = 0; $i -lt $maxRetries; $i++) {
    try {
        $content = Get-Content -LiteralPath $path -ErrorAction Stop
        break
    } catch {
        if ($i -eq ($maxRetries - 1)) { throw }
        Start-Sleep -Seconds $retryDelay
    }
}

# Low-level file access when you need read/write control
$stream = [System.IO.File]::Open($path, [System.IO.FileMode]::Open, [System.IO.FileAccess]::Read, [System.IO.FileShare]::ReadWrite)
try {
    $reader = [System.IO.StreamReader]::new($stream)
    $content = $reader.ReadToEnd()
} finally {
    if ($reader) { $reader.Dispose() }
    if ($stream) { $stream.Dispose() }
}
```

---

## 9. Data Format Handling (JSON / XML / CSV)

### JSON
```powershell
# CRITICAL GOTCHA: ConvertTo-Json default depth is 2 in PS 5.1
# Nested objects beyond depth 2 are silently converted to type name strings
# e.g., "System.Collections.Hashtable" instead of the actual data
# ALWAYS specify -Depth explicitly
$json = $object | ConvertTo-Json -Depth 10

# Reading JSON
$data = Get-Content -LiteralPath $path -Raw | ConvertFrom-Json

# Writing JSON
$data | ConvertTo-Json -Depth 10 | Set-Content -LiteralPath $path -Encoding UTF8

# 5.1 JSON quirks:
# - Unicode characters may be escaped as \uXXXX sequences
# - Empty arrays may serialize inconsistently
# - Large integers can lose precision (JSON spec limitation)
# - Single-element arrays may deserialize as scalars — wrap in @() if needed
```

### XML
```powershell
# Quick parsing with [xml] type accelerator
[xml]$doc = Get-Content -LiteralPath $path -Raw
$doc.Root.ChildElement.InnerText

# XPath queries with Select-Xml
$results = Select-Xml -Path $path -XPath '//Element[@Attribute="value"]'
$results | ForEach-Object { $_.Node.InnerText }

# XPath with namespaces
$ns = @{ ns = 'http://schemas.example.com/2024' }
Select-Xml -Path $path -XPath '//ns:Element' -Namespace $ns

# Creating/modifying XML
$doc = [xml]'<Root/>'
$element = $doc.CreateElement('Child')
$element.InnerText = 'value'
$doc.Root.AppendChild($element)
$doc.Save($outputPath)
```

### CSV
```powershell
# Import — everything comes in as strings
$data = Import-Csv -LiteralPath $path
# Type coercion is required for math/comparisons:
$data | ForEach-Object { [int]$_.Count * [decimal]$_.Price }

# Export — always specify encoding and NoTypeInformation (5.1)
$data | Export-Csv -LiteralPath $path -Encoding UTF8 -NoTypeInformation

# Delimiter handling
$data = Import-Csv -LiteralPath $path -Delimiter ';'

# ConvertTo/From for in-memory operations (no file)
$csv = $objects | ConvertTo-Csv -NoTypeInformation
$objects = $csvString | ConvertFrom-Csv
```

### Serialization (Clixml)
```powershell
# Full-fidelity PowerShell object serialization (preserves types)
$data | Export-Clixml -LiteralPath $path
$data = Import-Clixml -LiteralPath $path
# Useful for caching complex objects between script runs
# WARNING: Clixml files are NOT portable between different machines/users
# unless the objects are simple types (strings, numbers, hashtables)
```

---

## 10. Web Requests and API Interaction

### TLS Configuration (Do This First — Every Time)
```powershell
# Windows PowerShell 5.1 defaults to TLS 1.0
# Most modern HTTPS endpoints reject TLS 1.0/1.1
# This MUST be set before any web request or you'll get cryptic connection errors
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
```

### Invoke-RestMethod vs Invoke-WebRequest
```powershell
# Invoke-RestMethod — clean path for JSON APIs (auto-parses response)
$result = Invoke-RestMethod -Uri 'https://api.example.com/data' -Method Get

# Invoke-WebRequest — when you need headers, status codes, or non-JSON responses
$response = Invoke-WebRequest -Uri 'https://example.com' -UseBasicParsing
$response.StatusCode
$response.Headers
$response.Content
```

### The -UseBasicParsing Requirement (5.1)
```powershell
# In 5.1, Invoke-WebRequest uses IE's HTML parser by default
# This FAILS on Server Core and systems where IE isn't configured
# ALWAYS use -UseBasicParsing in 5.1 scripts
$response = Invoke-WebRequest -Uri $url -UseBasicParsing

# In PS 7+, basic parsing is the default — the parameter is accepted but unnecessary
```

### Authentication Patterns
```powershell
# Bearer token
$headers = @{ Authorization = "Bearer $token" }
$result = Invoke-RestMethod -Uri $url -Headers $headers

# Basic auth
$pair = "${username}:${password}"
$bytes = [System.Text.Encoding]::ASCII.GetBytes($pair)
$base64 = [Convert]::ToBase64String($bytes)
$headers = @{ Authorization = "Basic $base64" }
$result = Invoke-RestMethod -Uri $url -Headers $headers

# Credential object (for NTLM/Kerberos endpoints)
$cred = Get-Credential
$result = Invoke-RestMethod -Uri $url -Credential $cred
```

### POST with JSON Body
```powershell
$body = @{
    name  = 'widget'
    count = 42
} | ConvertTo-Json -Depth 10

$result = Invoke-RestMethod -Uri $url -Method Post -Body $body `
    -ContentType 'application/json'
```

### Retry Pattern for Unreliable Endpoints
```powershell
$maxRetries = 3
$retryDelay = 2  # seconds
for ($i = 0; $i -lt $maxRetries; $i++) {
    try {
        $result = Invoke-RestMethod -Uri $url -ErrorAction Stop
        break
    } catch {
        if ($i -eq ($maxRetries - 1)) { throw }
        Write-Warning "Request failed (attempt $($i+1)/$maxRetries): $($_.Exception.Message)"
        Start-Sleep -Seconds ($retryDelay * [math]::Pow(2, $i))  # Exponential backoff
    }
}
```

### File Download
```powershell
# Simple download
Invoke-WebRequest -Uri $url -OutFile $destPath -UseBasicParsing

# With progress (Invoke-WebRequest shows progress by default but it's slow)
# For large files, disable the progress bar for significant speed improvement:
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest -Uri $url -OutFile $destPath -UseBasicParsing
$ProgressPreference = 'Continue'
```

### Pagination Handling
```powershell
$allResults = [System.Collections.Generic.List[object]]::new()
$url = 'https://api.example.com/items?page=1&pageSize=100'

do {
    $response = Invoke-RestMethod -Uri $url
    $allResults.AddRange($response.items)
    $url = $response.nextPageUrl  # API-specific — check documentation
} while ($url)
```

---

## 11. Performance Patterns

### Measure Before Optimizing
```powershell
Measure-Command {
    # Code block to time
}
```

### Common Performance Wins
```powershell
# 1. Use -Filter instead of Where-Object on provider cmdlets
Get-ChildItem -Filter "*.log"          # Fast (provider-level)
Get-ChildItem | Where-Object { ... }   # Slow (PowerShell-level)

# 2. Use hashtable lookups instead of repeated Where-Object
$lookup = @{}
$users | ForEach-Object { $lookup[$_.Id] = $_ }
# Then: $lookup[$targetId] instead of $users | Where-Object { $_.Id -eq $targetId }

# 3. Batch queries when the API/provider supports it
# Instead of N individual calls, build a single query with multiple conditions

# 4. Use StreamReader for large text files
$reader = [System.IO.StreamReader]::new($path)
try {
    while ($null -ne ($line = $reader.ReadLine())) {
        # Process line
    }
} finally {
    $reader.Dispose()
}
# vs. Get-Content (loads entire file or is slow with -ReadCount)

# 5. Use StringBuilder for building large strings
$sb = [System.Text.StringBuilder]::new()
foreach ($item in $largeCollection) {
    $null = $sb.AppendLine("Processed: $item")
}
$result = $sb.ToString()

# 6. Suppress unwanted output (avoid pipeline overhead)
$null = $list.Add($item)       # Fast
[void]$list.Add($item)         # Also fast
$list.Add($item) | Out-Null    # Slow (pipeline overhead)
```

### Parallel Execution
```powershell
# PowerShell 5.1: Jobs (each is a new process — significant overhead)
$jobs = foreach ($server in $servers) {
    Start-Job -ScriptBlock { param($s) Test-Connection $s -Count 1 } -ArgumentList $server
}
$results = $jobs | Wait-Job | Receive-Job
$jobs | Remove-Job

# IMPORTANT: Always clean up jobs — leaked jobs consume memory and process handles

# PowerShell 5.1: Runspace pool (lighter than jobs, more setup required)
$pool = [RunspaceFactory]::CreateRunspacePool(1, 10)  # Min 1, Max 10 threads
$pool.Open()

$runspaces = foreach ($server in $servers) {
    $ps = [PowerShell]::Create().AddScript({
        param($s) Test-Connection $s -Count 1
    }).AddArgument($server)
    $ps.RunspacePool = $pool
    [PSCustomObject]@{
        PowerShell = $ps
        Handle     = $ps.BeginInvoke()
    }
}

$results = foreach ($rs in $runspaces) {
    $rs.PowerShell.EndInvoke($rs.Handle)
    $rs.PowerShell.Dispose()
}
$pool.Close()
$pool.Dispose()

# PowerShell 7+: ForEach-Object -Parallel (simple and effective)
$servers | ForEach-Object -Parallel {
    Test-Connection $_ -Count 1
} -ThrottleLimit 10
```

---

## 12. Security Considerations

### Credential Handling
```powershell
# NEVER store passwords in plain text in scripts
# WRONG:
$password = "MyP@ssw0rd"

# Prompt user (interactive)
$cred = Get-Credential

# Store encrypted credential (tied to user + machine via DPAPI)
$cred | Export-Clixml -LiteralPath "$env:USERPROFILE\cred.xml"
$cred = Import-Clixml -LiteralPath "$env:USERPROFILE\cred.xml"
# CAVEATS:
#   - Only works for the same user on the same machine
#   - Does NOT survive OS reinstalls, profile recreations, or domain trust changes
#   - For service account automation (scheduled tasks), consider:
#     - Windows Credential Manager (cmdkey.exe)
#     - SecretManagement module (cross-platform vault abstraction)
#     - Managed Service Accounts (for domain environments)

# SecureString from existing string (when you must)
$securePass = ConvertTo-SecureString $plainText -AsPlainText -Force
```

### Execution Policy
```powershell
# Check current policy
Get-ExecutionPolicy -List

# Common policies:
#   Restricted    — No scripts (default on client OS)
#   RemoteSigned  — Local scripts run; downloaded scripts need signature
#   Bypass        — No restrictions (use for automation with caution)

# Set for current user only (doesn't require admin)
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser

# Bypass for a single session (no permanent change)
powershell.exe -ExecutionPolicy Bypass -File .\script.ps1
```

### Input Sanitization
```powershell
# Escape strings used in -Filter or WMI queries
$safeName = [System.Management.Automation.Language.CodeGeneration]::EscapeSingleQuotedStringContent($userInput)

# For regex patterns from user input
$safePattern = [regex]::Escape($userInput)

# For paths: validate, don't trust
$resolvedPath = Resolve-Path -LiteralPath $userInput -ErrorAction Stop
if (-not $resolvedPath.Path.StartsWith($allowedRoot)) {
    throw "Path traversal attempt blocked"
}
```

---

## 13. Environment Variables

```powershell
# Read/write process-scope variables (current session only)
$env:MY_VAR = 'value'
$currentPath = $env:PATH

# Read machine or user scope (persistent, stored in registry)
$machineVar = [Environment]::GetEnvironmentVariable('MY_VAR', 'Machine')
$userVar    = [Environment]::GetEnvironmentVariable('MY_VAR', 'User')

# Write persistent environment variables (survive session restarts)
[Environment]::SetEnvironmentVariable('MY_VAR', 'value', 'User')     # Current user
[Environment]::SetEnvironmentVariable('MY_VAR', 'value', 'Machine')  # System-wide (requires admin)

# CRITICAL: $env:PATH modifications only affect the CURRENT PROCESS
# Child processes inherit the modified value, but other running processes don't see it
# To permanently modify PATH:
$currentPath = [Environment]::GetEnvironmentVariable('PATH', 'User')
[Environment]::SetEnvironmentVariable('PATH', "$currentPath;C:\NewTool", 'User')

# Remove a persistent variable
[Environment]::SetEnvironmentVariable('MY_VAR', $null, 'User')
```

---

## 14. Registry Operations

```powershell
# Read
$value = Get-ItemPropertyValue -LiteralPath 'HKLM:\SOFTWARE\MyApp' -Name 'Setting'

# Write (creates key path if needed with -Force)
New-Item -Path 'HKLM:\SOFTWARE\MyApp' -Force
Set-ItemProperty -LiteralPath 'HKLM:\SOFTWARE\MyApp' -Name 'Setting' -Value 'NewValue' -Type String

# Registry types: String, ExpandString, Binary, DWord, QWord, MultiString

# Test if key/value exists
$exists = Get-ItemProperty -LiteralPath 'HKLM:\SOFTWARE\MyApp' -Name 'Setting' -ErrorAction SilentlyContinue
if ($null -ne $exists) { ... }

# Browse like a filesystem
Get-ChildItem -LiteralPath 'HKLM:\SOFTWARE\Microsoft' -Depth 1
```

---

## 15. Services and Processes

```powershell
# Services
Get-Service -Name 'wuauserv'
Start-Service -Name 'wuauserv'
Stop-Service -Name 'wuauserv' -Force
Set-Service -Name 'wuauserv' -StartupType Disabled

# Wait for service state change
$svc = Get-Service -Name 'wuauserv'
$svc.WaitForStatus('Running', [TimeSpan]::FromSeconds(30))

# Processes
Get-Process -Name 'notepad' -ErrorAction SilentlyContinue
Stop-Process -Name 'notepad' -Force
Start-Process -FilePath 'notepad.exe' -ArgumentList 'file.txt' -Wait
Start-Process -FilePath 'cmd.exe' -Verb RunAs   # Elevate (UAC prompt)

# IMPORTANT: Start-Process does NOT capture output to variables
# $result = Start-Process ... captures nothing useful
# For capturable output, use the call operator:
$output = & "app.exe" --arguments 2>&1
$exitCode = $LASTEXITCODE
# Or redirect to file:
Start-Process -FilePath 'app.exe' -Wait -NoNewWindow `
    -RedirectStandardOutput "$env:TEMP\stdout.txt" `
    -RedirectStandardError "$env:TEMP\stderr.txt"
```

---

## 16. WMI / CIM Queries

```powershell
# Prefer CIM over WMI (CIM uses WinRM, works remotely, returns proper types)
# WMI cmdlets (Get-WmiObject) are removed in PowerShell 7+

# System info
Get-CimInstance -ClassName Win32_OperatingSystem | Select-Object Caption, Version, LastBootUpTime
Get-CimInstance -ClassName Win32_ComputerSystem | Select-Object Manufacturer, Model, TotalPhysicalMemory
Get-CimInstance -ClassName Win32_LogicalDisk -Filter "DriveType=3" | Select-Object DeviceID, Size, FreeSpace

# Remote queries
$session = New-CimSession -ComputerName 'Server01'
Get-CimInstance -CimSession $session -ClassName Win32_Service -Filter "State='Running'"
Remove-CimSession $session

# Method invocation
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine = 'notepad.exe'}
```

---

## 17. Remote Operations

```powershell
# One-off command on remote machine(s)
Invoke-Command -ComputerName 'Server01', 'Server02' -ScriptBlock {
    Get-Service -Name 'w3svc'
}

# Persistent session (reuse for multiple commands)
$session = New-PSSession -ComputerName 'Server01' -Credential $cred
Invoke-Command -Session $session -ScriptBlock { Get-Process }
Invoke-Command -Session $session -ScriptBlock { Get-EventLog -LogName System -Newest 10 }
Remove-PSSession $session

# Copy files over a session
Copy-Item -Path "C:\local\file.txt" -Destination "C:\remote\" -ToSession $session

# Prerequisites: WinRM must be enabled on target
# Enable: Enable-PSRemoting -Force (on target, as admin)
# Troubleshoot: Test-WSMan -ComputerName 'Server01'
```

### The Double-Hop Problem
```powershell
# SCENARIO: You remote into Server A, then Server A tries to access Server B
# (e.g., accessing a file share or querying AD from a remote session)
# RESULT: Access denied — credentials don't propagate past the first hop

# Solutions (in order of preference):
# 1. Avoid the hop — run the command directly against Server B
Invoke-Command -ComputerName 'ServerB' -ScriptBlock { ... }

# 2. CredSSP (enables credential delegation — security risk, use cautiously)
# On client:  Enable-WSManCredSSP -Role Client -DelegateComputer 'ServerA'
# On ServerA: Enable-WSManCredSSP -Role Server
Invoke-Command -ComputerName 'ServerA' -Authentication CredSSP -Credential $cred -ScriptBlock { ... }

# 3. Kerberos constrained delegation (domain admin configures in AD — most secure)

# 4. Pass credentials explicitly inside the remote session
Invoke-Command -ComputerName 'ServerA' -ScriptBlock {
    param($cred)
    Invoke-Command -ComputerName 'ServerB' -Credential $cred -ScriptBlock { ... }
} -ArgumentList $cred
```

---

## 18. Active Directory Operations

**Note:** Requires the `ActiveDirectory` module (part of RSAT on workstations, installed by default on domain controllers). `Import-Module ActiveDirectory` if not auto-loaded.

### Basic Queries
```powershell
# User lookup
Get-ADUser -Identity 'jsmith' -Properties DisplayName, Mail, Department, LastLogonDate

# Search with -Filter (CAUTION: AD filter syntax is NOT PowerShell syntax)
Get-ADUser -Filter "Department -eq 'Engineering'" -Properties Department

# Search with -LDAPFilter (more reliable for complex queries)
Get-ADUser -LDAPFilter '(&(department=Engineering)(enabled=TRUE))' -Properties Department

# Limit scope with -SearchBase
Get-ADUser -Filter * -SearchBase 'OU=Sales,DC=contoso,DC=com'
```

### AD Filter Syntax Gotcha (Critical)
```powershell
# AD -Filter looks like PowerShell but has DIFFERENT RULES:
#   - Does NOT support: -in, -contains, -notcontains, array comparisons
#   - Does NOT support: script blocks, method calls, complex expressions
#   - Quoting rules are different from PowerShell
#   - Variable expansion requires specific syntax

# WRONG — -in operator doesn't exist in AD filter:
Get-ADUser -Filter "Name -in 'Alice','Bob'"  # Parse error

# CORRECT — use -LDAPFilter with OR for multi-value queries:
$ldapFilter = '(|(name=Alice)(name=Bob))'
Get-ADUser -LDAPFilter $ldapFilter

# For dynamic lists:
$names = @('Alice', 'Bob', 'Charlie')
$conditions = ($names | ForEach-Object { "(name=$_)" }) -join ''
$ldapFilter = "(|$conditions)"
Get-ADUser -LDAPFilter $ldapFilter

# RULE: For anything beyond simple equality checks, prefer -LDAPFilter over -Filter
# Test filters interactively before putting them in scripts
```

### Common Operations
```powershell
# Bulk modification
Get-ADUser -Filter "Department -eq 'OldName'" | Set-ADUser -Department 'NewName'

# Group membership
Get-ADGroupMember -Identity 'IT-Staff' -Recursive
Add-ADGroupMember -Identity 'IT-Staff' -Members 'jsmith'

# Computer objects
Get-ADComputer -Filter "OperatingSystem -like '*Server*'" -Properties OperatingSystem, LastLogonDate

# Performance: -Properties * pulls ALL attributes from AD (slow for large queries)
# Only request the specific properties you need
Get-ADUser -Filter * -Properties DisplayName, Mail  # GOOD
Get-ADUser -Filter * -Properties *                   # SLOW — pulls everything
```

---

## 19. Event Log Querying

### Get-WinEvent (Modern — Use This)
```powershell
# Query System log for errors in the last 24 hours
$yesterday = (Get-Date).AddDays(-1)
Get-WinEvent -FilterHashtable @{
    LogName   = 'System'
    Level     = 2        # 1=Critical, 2=Error, 3=Warning, 4=Information
    StartTime = $yesterday
} -MaxEvents 50

# Application crashes
Get-WinEvent -FilterHashtable @{
    LogName      = 'Application'
    ProviderName = 'Application Error'
    StartTime    = $yesterday
}

# Security log — login failures (Event ID 4625)
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id      = 4625
} -MaxEvents 20

# IMPORTANT: Always use -MaxEvents to avoid accidentally pulling hundreds of
# thousands of records. Query first with a small limit, then expand if needed.
```

### FilterHashtable vs FilterXml vs Where-Object
```powershell
# FilterHashtable — fastest, covers most common queries (shown above)
# FilterXml — for complex queries that FilterHashtable can't express
Get-WinEvent -FilterXml @'
<QueryList>
  <Query Id="0" Path="Security">
    <Select Path="Security">
      *[System[(EventID=4624 or EventID=4625) and TimeCreated[timediff(@SystemTime) &lt;= 86400000]]]
    </Select>
  </Query>
</QueryList>
'@

# Where-Object — slowest (pulls all events then filters in PowerShell)
# Avoid for large logs
```

### Get-EventLog (Legacy — Simpler but Limited)
```powershell
# Only works with classic logs (System, Application, Security)
# Doesn't work with newer logs (Microsoft-Windows-*)
Get-EventLog -LogName System -EntryType Error -Newest 50
Get-EventLog -LogName Application -Source 'MyApp' -After (Get-Date).AddHours(-1)
```

---

## 20. Archive / Compression Operations

```powershell
# Built-in cmdlets (PowerShell 5.0+)
Compress-Archive -LiteralPath $sourcePath -DestinationPath "$dest.zip"
Compress-Archive -Path "C:\Logs\*.log" -DestinationPath "$dest.zip"  # Wildcards OK here
Expand-Archive -LiteralPath "$archive.zip" -DestinationPath $extractDir

# LIMITATIONS of Compress-Archive:
#   - No password/encryption support
#   - 2GB file size limit in PS 5.1 (fixed in 7+)
#   - ZIP format only
#   - No compression level control in 5.1

# .NET ZipFile class for more control
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::CreateFromDirectory($sourceDir, $zipPath)
[System.IO.Compression.ZipFile]::ExtractToDirectory($zipPath, $extractDir)

# For anything beyond basic ZIP (passwords, 7z, tar.gz):
# Use 7-Zip CLI (must be installed separately)
& "C:\Program Files\7-Zip\7z.exe" a -p"password" "$dest.7z" $sourcePath
```

---

## 21. Scheduled Tasks and Automation

```powershell
# Create a scheduled task
$action  = New-ScheduledTaskAction -Execute 'powershell.exe' `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "C:\Scripts\backup.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At '2:00AM'
$settings = New-ScheduledTaskSettingsSet -StartWhenAvailable -DontStopOnIdleEnd
Register-ScheduledTask -TaskName 'DailyBackup' -Action $action -Trigger $trigger `
    -Settings $settings -User 'SYSTEM' -RunLevel Highest

# Modify existing
Set-ScheduledTask -TaskName 'DailyBackup' -Trigger (New-ScheduledTaskTrigger -Daily -At '3:00AM')

# Query / Remove
Get-ScheduledTask -TaskName 'DailyBackup'
Unregister-ScheduledTask -TaskName 'DailyBackup' -Confirm:$false
```

---

## 22. Working with External Programs

```powershell
# Capture stdout with the call operator
$output = & "C:\tools\app.exe" --argument value 2>&1

# Check exit code
& "robocopy.exe" $source $dest /MIR
$exitCode = $LASTEXITCODE
# Note: Robocopy uses non-standard exit codes (0-7 = success variants)

# Start-Process for more control (but NOT for capturing output — see Section 15)
$proc = Start-Process -FilePath 'app.exe' -ArgumentList '--flag', 'value' `
    -Wait -NoNewWindow -PassThru `
    -RedirectStandardOutput "$env:TEMP\stdout.txt" `
    -RedirectStandardError "$env:TEMP\stderr.txt"
$proc.ExitCode

# GOTCHA: Arguments with spaces — use array splatting
$processArgs = @('--path', "`"$pathWithSpaces`"", '--verbose')
& "app.exe" @processArgs
```

---

## 23. .NET Interop

```powershell
# Load an assembly
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName PresentationFramework

# Use .NET classes directly
$stopwatch = [System.Diagnostics.Stopwatch]::StartNew()
# ... work ...
$stopwatch.Stop()
Write-Host "Elapsed: $($stopwatch.Elapsed.TotalSeconds)s"

# Create instances
$list = [System.Collections.Generic.List[string]]::new()    # PowerShell 5+ syntax
$list = New-Object 'System.Collections.Generic.List[string]' # Legacy syntax

# Define custom C# inline (for complex operations)
Add-Type -TypeDefinition @"
using System;
public class MathHelper {
    public static double Hypotenuse(double a, double b) {
        return Math.Sqrt(a * a + b * b);
    }
}
"@
[MathHelper]::Hypotenuse(3, 4)  # 5.0

# IMPORTANT: Dispose IDisposable objects
$stream = [System.IO.FileStream]::new($path, [System.IO.FileMode]::Open)
try {
    # Use stream
} finally {
    $stream.Dispose()
}
```

---

## 24. Debugging

```powershell
# Built-in debugger
Set-PSBreakpoint -Script .\script.ps1 -Line 42
Set-PSBreakpoint -Script .\script.ps1 -Variable 'result' -Mode Write
# Commands in debugger: s (step), v (step over), o (step out), c (continue), q (quit)

# Quick inspection
$variable | Get-Member                    # See all properties/methods
$variable | Format-List *                 # See all property values
$variable.GetType().FullName             # Exact type

# Verbose tracing
$VerbosePreference = 'Continue'          # Show all Write-Verbose output
Set-PSDebug -Trace 1                     # Trace every line executed
Set-PSDebug -Trace 0                     # Turn off

# Scope inspection
Get-Variable -Scope Local
Get-Variable -Scope Script
Get-Variable -Scope Global
```

### Testing with Pester
Pester (ships with Windows, may need updating) provides a test framework for PowerShell. For scripts that will be reused or maintained, consider adding Pester tests. Basic pattern:

```powershell
# Install latest: Install-Module Pester -Force -SkipPublisherCheck
# File naming convention: MyFunction.Tests.ps1

Describe 'Get-Widget' {
    BeforeAll {
        . $PSScriptRoot\MyFunction.ps1
    }
    It 'returns widgets for valid input' {
        $result = Get-Widget -Name 'Test'
        $result | Should -Not -BeNullOrEmpty
    }
}
# Run: Invoke-Pester .\MyFunction.Tests.ps1 -Output Detailed
```

---

## 25. Version Differences Quick Reference

| Feature | 5.1 | 7+ |
|---------|-----|----|
| Default encoding | UTF-16 / ASCII | UTF-8 NoBOM |
| `ForEach-Object -Parallel` | No | Yes |
| Ternary operator `? :` | No | Yes |
| Null-coalescing `??` | No | Yes |
| Pipeline chain `&&` `||` | No | Yes |
| `Get-WmiObject` | Yes | Removed (use CIM) |
| `$null?.Method()` | No | Yes (null-conditional) |
| SSH remoting | No | Yes |
| Cross-platform | No | Yes |
| `Clean` block | No | Yes (7.3+) |
| `Export-Csv` NoTypeInfo default | No (need flag) | Yes |
| `Invoke-WebRequest` basic parsing | Requires `-UseBasicParsing` | Default |
| TLS default | 1.0 | 1.2+ |
| `ConvertTo-Json` depth default | 2 | 100 |
| `Compress-Archive` 2GB limit | Yes | Fixed |
| `$x = if () {} else {}` (if-as-expression) | Parse error | Works |

**Compatibility strategy:** Target environment is Windows 10+ with PS 5.1. Write for 5.1 unless the user specifies 7+. When a 7+ feature would significantly simplify code, note it as an alternative.

**PS 7+ `??` replaces verbose config-default patterns:**
```powershell
# VERBOSE (5.1 + 7+) — must wrap in $() for 5.1:
$hashAlgo = $(if ($config.backup.features.hashAlgorithm) {
    $config.backup.features.hashAlgorithm
} else { 'MD5' })

# CONCISE (7+ only):
$hashAlgo = $config.backup.features.hashAlgorithm ?? 'MD5'
```
**Critical distinction:** `??` tests for `$null` ONLY. It does NOT trigger on `0`, `""`, or `$false`. Safe for strings, paths, names. Not safe for numeric values where `0` should fall back to a default.

---

## 26. GUI Elements (When Appropriate)

### Simple Dialogs (No Framework Needed)

```powershell
# Simple message box
Add-Type -AssemblyName PresentationFramework
[System.Windows.MessageBox]::Show('Backup complete!', 'Status', 'OK', 'Information')

# File picker dialog
Add-Type -AssemblyName System.Windows.Forms
$dialog = [System.Windows.Forms.OpenFileDialog]@{
    Filter      = 'Video Files|*.mp4;*.mkv;*.avi|All Files|*.*'
    Multiselect = $true
    Title       = 'Select files to process'
}
if ($dialog.ShowDialog() -eq 'OK') {
    $selectedFiles = $dialog.FileNames
}

# Folder picker
$dialog = [System.Windows.Forms.FolderBrowserDialog]@{
    Description = 'Select the target folder'
}
if ($dialog.ShowDialog() -eq 'OK') {
    $selectedFolder = $dialog.SelectedPath
}
```

### WPF Applications

For full WPF applications with data binding, XAML, and MVVM patterns, see `docs/wpf-skill.md`.

**Critical warning:** PowerShell scriptblocks passed as C# delegates (e.g., `[Action[object]]`
for `ICommand.Execute`) run in an unpredictable scope. **Do NOT use `.GetNewClosure()`** —
it works inconsistently. Instead, use a `$global:` hashtable to share state between the
ViewModel setup and the command scriptblocks. See wpf-skill.md Section 5 for the full pattern.

---

## 27. Common Gotchas Reference

This table is the quick-reference safety net. Claude should scan this before delivering any script.

| # | Gotcha | What Happens | Fix |
|---|--------|-------------|-----|
| 1 | Cmdlet returns 0 items | Variable is `$null`; `.Count` behavior unreliable in strict mode | Wrap in `@()`: `$files = @(Get-ChildItem ...)` |
| 2 | `-Path` with brackets/wildcards | Path interpreted as glob pattern, file not found | Use `-LiteralPath` for all non-glob operations |
| 3 | `$null` comparison with collections | `$collection -eq $null` filters elements instead of boolean compare | Always put `$null` on the left: `$null -eq $var` |
| 4 | Non-terminating errors in try/catch | Catch block never fires | Set `-ErrorAction Stop` or `$ErrorActionPreference = 'Stop'` |
| 5 | `Out-File` encoding (5.1) | UTF-16 LE by default — breaks Unix tools, git | Always specify `-Encoding UTF8` |
| 6 | `Export-Csv` encoding (5.1) | ASCII by default — loses non-ASCII characters | Always specify `-Encoding UTF8` |
| 7 | `Export-Csv` #TYPE header (5.1) | Prepends `#TYPE` line that breaks CSV consumers | Always use `-NoTypeInformation` |
| 8 | String comparison case sensitivity | `-eq` is case-insensitive; `.Contains()` is case-sensitive | Use `-ceq` for case-sensitive; use `-match` or `StringComparison` for .NET |
| 9 | `+=` on arrays | Creates new array every append — O(n) per add | Use `[List[object]]` or pipeline output. **Note:** `@(@($x) + $y)` is the same antipattern in disguise — PSScriptAnalyzer won't catch it. Quality gates check `+=` syntactically, not contextually — it's flagged even outside loops. |
| 10 | Boolean params to external programs | `-Flag:$true` sends literal `:True` string | Build argument array conditionally (see Section 3) |
| 11 | `return` in pipeline | Doesn't stop pipeline processing | Use `break` or rethink control flow |
| 12 | Backtick line continuation | Invisible trailing whitespace breaks it | Use splatting or pipeline continuation with `\|` |
| 13 | `ConvertTo-Json` depth (5.1) | Default depth 2 — nested objects become type name strings silently | Always specify `-Depth` (e.g., `-Depth 10`) |
| 14 | TLS 1.0 default (5.1) | HTTPS requests fail against modern endpoints with vague errors | Set `[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12` |
| 15 | `Invoke-WebRequest` IE dependency (5.1) | Fails on Server Core / systems without IE | Always use `-UseBasicParsing` |
| 16 | Double-hop authentication | Credentials don't propagate past first remote hop | Use CredSSP, constrained delegation, or explicit credential pass (see Section 17) |
| 17 | `Format-*` cmdlets destroy objects | Output is formatting instructions, not data | Never use `Format-*` before `Export-*`, `Select-Object`, or variable assignment |
| 18 | Scope bleed in functions | Parent scope variables are readable; writes create local copies silently | Always declare/initialize variables in the scope that owns them |
| 19 | AD `-Filter` is not PowerShell syntax | Looks like PS but has different parsing rules, limited operators | Use `-LDAPFilter` for complex queries; test interactively first (see Section 18) |
| 20 | `Get-Content` returns array of lines | String operations on result operate per-line, not on full text | Use `-Raw` for single-string content |
| 21 | Bare `if`-as-expression assignment (5.1) | `$x = if (...) {} else {}` is a parse error in 5.1 -- works in 7+ | Wrap in subexpression: `$x = $(if (...) {} else {})` [verified in Phase 3+fix] |
| 22 | Non-ASCII chars in .ps1 source (5.1) | Em-dashes, smart quotes cause cryptic parse errors when file has no BOM | Keep .ps1 source pure ASCII; use `--` not em-dash. See Section 9 encoding notes. |
| 23 | `Start-Process` doesn't capture output | `$result = Start-Process ...` captures nothing | Use `& executable` for capturable output, or `-RedirectStandardOutput` to file |
| 24 | `$args` is an automatic variable | Assigning to `$args` shadows the built-in; causes confusion | Name variables `$processArgs`, `$appArgs`, etc. |
| 25 | `.GetNewClosure()` + C# delegates | Variables silently resolve to `$null` when scriptblock invoked from C# `Action`/`Func` delegate | Use `$global:` hashtable context instead of closure capture (see wpf-skill.md) |
| 26 | `$script:` vars in WPF command closures | Script-scoped variables inaccessible when invoked from C# delegate | Copy to `$global:` context hashtable before wiring commands |
| 27 | Pester can't test WPF command execution | Tests run in same scope -- closures work fine. Real GUI invokes from C# delegates -- closures break silently | Manual GUI testing required for command execution; Pester only validates wiring |
| 28 | `$sender` in .NET event handlers | PSScriptAnalyzer `PSAvoidAssignmentToAutomaticVariable` -- `$sender` is a PS automatic variable | Use `$s` instead: `param($s, $e)` in `add_PropertyChanged`, `add_Click`, etc. |
| 29 | `[Parameter(Mandatory)]` on collections | PS 7 rejects `@()` with "Cannot bind argument...empty collection" | Add `[AllowEmptyCollection()]` when empty is semantically valid |
| 30 | `Stop-Process -Name` after timeout | Kills ALL processes with that name, including ones started after the timeout began | Capture PIDs before graceful close, then `Stop-Process -Id $proc.Id -Force` on the original set |

---

## 28. Pester Testing (Deep-Dive)

The basic Pester pattern is in Section 24. This section covers the patterns that actually trip people up.

### Pester 5 vs Pester 4 — Critical Differences
Pester 5 (ships with PS 7, installable on 5.1) changed the scoping model. Code written for Pester 4 will break silently.

```powershell
# Pester 4 (LEGACY — do not use for new code)
$myVar = 'setup'           # Visible in It blocks
It 'test' { $myVar | Should -Be 'setup' }

# Pester 5 (CURRENT)
BeforeAll { $script:myVar = 'setup' }    # Must use BeforeAll
It 'test' { $script:myVar | Should -Be 'setup' }
```

**Rule:** In Pester 5, all setup code goes in `BeforeAll` / `BeforeEach` / `AfterAll` / `AfterEach` blocks. Code outside these blocks in a `Describe`/`Context` runs at **discovery time**, not test time.

### Block Scoping
```powershell
Describe 'MyModule' {
    BeforeAll {
        # Runs ONCE before all tests in this Describe
        Import-Module ./MyModule.psd1 -Force
        $script:TestDir = Join-Path ([IO.Path]::GetTempPath()) "Test_$(Get-Random)"
        New-Item $script:TestDir -ItemType Directory -Force | Out-Null
    }

    AfterAll {
        # Runs ONCE after all tests — cleanup
        Remove-Item $script:TestDir -Recurse -Force -ErrorAction SilentlyContinue
    }

    BeforeEach {
        # Runs before EACH It block — per-test isolation
        $script:TempFile = Join-Path $script:TestDir "test_$(Get-Random).txt"
    }

    AfterEach {
        # Runs after EACH It block
        if (Test-Path $script:TempFile) { Remove-Item $script:TempFile -Force }
    }

    It 'does something' {
        # $script:TempFile is unique per test
    }
}
```

### InModuleScope — Accessing Private State
`InModuleScope` lets tests access `$script:` variables and private (non-exported) functions inside a module.

```powershell
# Read a private module variable
$value = InModuleScope 'MyModule' { $script:InternalCache }

# Call a private function
$result = InModuleScope 'MyModule' { Get-InternalHelper -Name 'test' }

# Modify module state for testing (restore in AfterAll!)
InModuleScope 'MyModule' {
    $script:Config.email.enabled = $true
}

# Pass parameters into InModuleScope (Pester 5.4+)
$testPath = '/tmp/test.json'
InModuleScope 'MyModule' -Parameters @{ Path = $testPath } {
    param($Path)
    $script:ConfigPath = $Path
}
```

### Mocking
```powershell
Describe 'Send-Report' {
    BeforeAll {
        # Mock within the module's scope — critical for module functions
        Mock -ModuleName 'MyModule' -CommandName 'Send-MailMessage' -MockWith {
            return @{ Success = $true }
        }

        # Mock with parameter filter
        Mock -ModuleName 'MyModule' -CommandName 'Get-Item' -ParameterFilter {
            $LiteralPath -like '*.log'
        } -MockWith { [PSCustomObject]@{ FullName = 'fake.log' } }
    }

    It 'sends email when enabled' {
        Send-Report -To 'admin@test.com'

        # Verify mock was called
        Should -Invoke -ModuleName 'MyModule' -CommandName 'Send-MailMessage' -Times 1 -Exactly
    }

    It 'passes correct recipient' {
        Send-Report -To 'admin@test.com'

        Should -Invoke -ModuleName 'MyModule' -CommandName 'Send-MailMessage' -ParameterFilter {
            $To -eq 'admin@test.com'
        }
    }
}
```

**Key mock rules:**
- `-ModuleName` is required when mocking calls made *inside* a module — without it, the mock only applies in the test scope
- Mocks are scoped to their `Describe`/`Context` block and auto-removed after
- Use `Should -Invoke` (not `Assert-MockCalled` — that's Pester 4)

### Test Isolation Patterns

```powershell
# Pattern: Redirect module paths to temp locations
BeforeAll {
    $script:TestDir = Join-Path ([IO.Path]::GetTempPath()) "Test_$(Get-Random)"
    New-Item $script:TestDir -ItemType Directory -Force | Out-Null

    # Override the module's config path
    InModuleScope 'MyModule' -Parameters @{ Dir = $script:TestDir } {
        param($Dir)
        $script:ConfigPath = Join-Path $Dir 'test-config.json'
    }
}

AfterAll {
    # Restore module to real config
    $null = Get-MyConfig -Force   # -Force pattern: reload from disk, clear cache
    Remove-Item $script:TestDir -Recurse -Force -ErrorAction SilentlyContinue
}
```

### Common Assertions
```powershell
$result | Should -Be 'exact value'
$result | Should -BeLike '*partial*'          # Wildcard match
$result | Should -Match 'regex\d+'            # Regex match
$result | Should -BeNullOrEmpty
$result | Should -Not -BeNullOrEmpty
$result | Should -BeOfType [hashtable]
$result | Should -HaveCount 3
$result | Should -BeGreaterThan 0
@($result).Count | Should -Be 5              # @() wrap for single-item safety
{ Do-BadThing } | Should -Throw               # Expect terminating error
{ Do-BadThing } | Should -Throw '*specific*'  # Match error message
Test-Path $file | Should -BeTrue
```

### Running Tests
```powershell
# Run all tests
Invoke-Pester ./tests/ -Output Detailed

# Run specific file
Invoke-Pester ./tests/MyModule.Tests.ps1 -Output Detailed

# Run with code coverage
Invoke-Pester ./tests/ -CodeCoverage ./src/MyModule.ps1
```

---

## 29. Module Development Patterns

Expanding on the brief table in Section 1 — this covers patterns for building production modules.

### Directory Structure
```
MyModule/
├── MyModule.psd1          # Module manifest (metadata + exports)
├── MyModule.psm1          # Root module (dot-sources everything)
├── Core/
│   ├── Configuration.ps1  # Get-MyConfig, Set-MyConfig
│   ├── Logging.ps1        # Write-MyLog
│   └── Engine.ps1         # Core business logic
├── Helpers/
│   └── Internal.ps1       # Private helper functions (not exported)
└── config/
    └── defaults.json      # Shipped default configuration
```

### Module Manifest (.psd1)
```powershell
# Generate scaffold: New-ModuleManifest -Path MyModule.psd1
@{
    RootModule        = 'MyModule.psm1'
    ModuleVersion     = '1.0.0'
    GUID              = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'   # Generate once, never change
    Author            = 'Your Name'
    Description       = 'Brief description of what the module does'

    # CRITICAL: List every exported function explicitly. Never use '*'
    FunctionsToExport = @(
        'Get-MyConfig'
        'Set-MyConfig'
        'Invoke-MyOperation'
    )

    # Prevent accidental exports
    CmdletsToExport   = @()
    VariablesToExport  = @()
    AliasesToExport    = @()

    # Dependencies (optional)
    RequiredModules    = @()
    PowerShellVersion  = '5.1'
}
```

**Why explicit exports matter:** `FunctionsToExport = '*'` exports every function in every dot-sourced file — including internal helpers. Explicit listing acts as a public API contract.

### Private Helper Naming
- Private helpers that construct and return in-memory `[PSCustomObject]` should use the **`Format-`** verb, not `New-`. `New-` triggers `PSUseShouldProcessForStateChangingFunctions` in PSScriptAnalyzer because it implies system state creation. `Format-` is correct — the function is shaping output, not creating external state.
- When a private helper returns a collection and uses a plural noun (e.g., `Get-FooItems`), add `[SuppressMessageAttribute('PSUseSingularNouns', '')]` inside the function body. PSScriptAnalyzer enforces singular nouns, but collection-returning helpers are a justified exception.

### Root Module (.psm1) — Dot-Source Pattern
```powershell
# Dot-source all .ps1 files from organized subdirectories
$folders = @('Core', 'Helpers', 'Email', 'Scheduling')
foreach ($folder in $folders) {
    $folderPath = Join-Path -Path $PSScriptRoot -ChildPath $folder
    if (Test-Path -LiteralPath $folderPath) {
        Get-ChildItem -LiteralPath $folderPath -Filter '*.ps1' | ForEach-Object {
            . $_.FullName
        }
    }
}
```

### Module-Scoped State
```powershell
# In Configuration.ps1 (loaded by .psm1)
$script:ConfigPath = Join-Path -Path $PSScriptRoot -ChildPath '../config/myconfig.json'
$script:ConfigCache = $null    # Lazy-loaded cache

function Get-MyConfig {
    [CmdletBinding()]
    param([switch]$Force)

    if ($Force -or -not $script:ConfigCache) {
        if (Test-Path -LiteralPath $script:ConfigPath) {
            $script:ConfigCache = Get-Content -LiteralPath $script:ConfigPath -Raw |
                ConvertFrom-Json
        } else {
            # Create from defaults
            $defaults = Get-Content -LiteralPath "$PSScriptRoot/../config/defaults.json" -Raw |
                ConvertFrom-Json
            $script:ConfigCache = $defaults
        }
    }
    return $script:ConfigCache
}
```

**Key points:**
- `$script:` variables are module-scoped when defined in files dot-sourced by the `.psm1`
- They're shared across all functions in the module but invisible outside it
- The `-Force` switch pattern clears the cache — essential for test isolation

### Versioning
```powershell
# In .psd1: ModuleVersion = '1.2.3' (SemVer: Major.Minor.Patch)
# Major: breaking changes, Minor: new features, Patch: bug fixes

# Consumer can require minimum version:
#Requires -Modules @{ ModuleName = 'MyModule'; ModuleVersion = '1.0.0' }

# Or in manifest RequiredModules:
RequiredModules = @(
    @{ ModuleName = 'OtherModule'; ModuleVersion = '2.0.0' }
)
```

---

## 30. Self-Elevation (UAC)

Common pattern for tools that need admin privileges for some operations (services, scheduled tasks, registry HKLM).

### Detecting Admin Context
```powershell
function Test-IsAdmin {
    $principal = [Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()
    return $principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
}
```

### Self-Relaunch as Admin
```powershell
if (-not (Test-IsAdmin)) {
    $args = @(
        '-NoProfile'
        '-ExecutionPolicy', 'Bypass'
        '-File', "`"$PSCommandPath`""
        # Pass through original arguments
    )
    Start-Process pwsh -Verb RunAs -ArgumentList $args -Wait
    return   # Exit non-elevated instance
}
# Code below runs elevated
```

**PS 5.1 targets:** Use `powershell.exe` instead of `pwsh`. `pwsh` is PS 7+ and may not be
installed on the target machine. Pass `-Resume` or other switches through explicitly since
`$args` is not automatically forwarded:

```powershell
# PS 5.1 compatible self-elevation with switch passthrough
$isElevated = ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole(
    [Security.Principal.WindowsBuiltInRole]::Administrator
)
if (-not $isElevated) {
    $argList = "-ExecutionPolicy Bypass -File `"$PSCommandPath`""
    if ($Resume) { $argList += ' -Resume' }   # repeat for each [switch] param
    Start-Process powershell.exe -Verb RunAs -ArgumentList $argList
    exit   # use exit, not return -- param block has already run
}
```

Note: `Start-Process -Verb RunAs` opens a new window. The original (unelevated) process
exits immediately. Output from the elevated window is not captured by the caller.

### Elevate a Specific Action (Not the Whole Script)
```powershell
function Invoke-ElevatedAction {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)]
        [string]$ScriptBlock,
        [string]$ModulePath
    )

    if (-not (Test-IsAdmin)) {
        $tempScript = Join-Path ([IO.Path]::GetTempPath()) "elevated_$(Get-Random).ps1"
        try {
            # Write the action to a temp script
            @"
Import-Module '$ModulePath' -Force
$ScriptBlock
"@ | Set-Content -LiteralPath $tempScript -Encoding utf8

            if ($PSCmdlet.ShouldProcess('PowerShell', 'Run elevated')) {
                Start-Process pwsh -Verb RunAs -ArgumentList @(
                    '-NoProfile', '-ExecutionPolicy', 'Bypass',
                    '-File', $tempScript
                ) -Wait
            }
        }
        finally {
            Remove-Item -LiteralPath $tempScript -Force -ErrorAction SilentlyContinue
        }
    }
    else {
        # Already admin — run directly
        Invoke-Expression $ScriptBlock
    }
}
```

### Credential Passthrough During Elevation
When a GUI prompts for credentials that the elevated process needs:

```powershell
# In the non-elevated process (has GUI access)
$cred = Get-Credential -Message 'Enter service account'
$tempCred = Join-Path ([IO.Path]::GetTempPath()) "cred_$(Get-Random).xml"
$cred | Export-Clixml -LiteralPath $tempCred

# Launch elevated, passing the temp path
Start-Process pwsh -Verb RunAs -ArgumentList @(
    '-NoProfile', '-File', $scriptPath, '-CredentialPath', $tempCred
) -Wait

# Clean up (non-elevated process)
Remove-Item -LiteralPath $tempCred -Force -ErrorAction SilentlyContinue

# In the elevated script:
$cred = Import-Clixml -LiteralPath $CredentialPath
Remove-Item -LiteralPath $CredentialPath -Force
# Use $cred...
```

**Why this works:** `Export-Clixml` uses DPAPI encryption tied to the current user. The elevated process runs as the same user, so it can decrypt.

### Session 0 Gotcha
Processes started by Task Scheduler run in **Session 0** (non-interactive). They cannot:
- Show GUI windows (invisible to the user)
- Use `Get-Credential` (no UI to display the prompt)
- Start GUI applications that the user can see

**Solution:** Use a `-Headless` or `-RunBackup` switch for scheduled task mode. Skip process restarts and GUI operations. Exit with codes (0 = success, 1 = failure).

---

## 31. Robocopy Integration

Robocopy is the most reliable file copy tool on Windows, but its exit codes confuse everyone.

### Exit Code Semantics
```powershell
# Robocopy exit codes are BITMASKS, not error codes:
#   0 = No files copied, no errors
#   1 = Files copied successfully
#   2 = Extra files/dirs detected (destination has files not in source)
#   4 = Mismatched files detected
#   8 = FAILED — some files could not be copied
#  16 = FATAL — usage error or insufficient access

# The correct error check:
robocopy $source $dest /E /R:2 /W:3
if ($LASTEXITCODE -ge 8) {
    throw "Robocopy failed with exit code $LASTEXITCODE"
}
# Exit codes 0-7 are all degrees of SUCCESS
```

### Common Flag Combinations
```powershell
# Copy folder with subdirs, moderate retry
robocopy $source $dest /E /R:2 /W:3 /NP

# Mirror (make destination match source exactly — DESTRUCTIVE: deletes extra files)
robocopy $source $dest /MIR /R:2 /W:3 /NP

# Quiet output (no file/dir listing)
robocopy $source $dest /E /R:2 /W:3 /NP /NFL /NDL

# Log to file
robocopy $source $dest /E /R:2 /W:3 /LOG:"$logPath"

# Log to file AND console
robocopy $source $dest /E /R:2 /W:3 /LOG:"$logPath" /TEE
```

**Flags to avoid:**
- `/V` (verbose) — generates huge output, one line per file including skipped. Use only for debugging.
- `/MIR` without explicit intent — it deletes files from destination. Use `/E` for safe copy.

### $LASTEXITCODE Reset Pattern
Robocopy sets `$LASTEXITCODE` even on success (e.g., 1 = files copied). This can confuse subsequent tools or CI/CD pipelines that check exit codes.

```powershell
robocopy $source $dest /E /R:2 /W:3
$robocopyExit = $LASTEXITCODE

if ($robocopyExit -ge 8) {
    Write-Error "Robocopy failed (exit $robocopyExit)"
}

# Reset for downstream tools that expect 0 = success
$global:LASTEXITCODE = 0
```

### Capturing Output
```powershell
$output = robocopy $source $dest /E /R:2 /W:3 /NP 2>&1
$exitCode = $LASTEXITCODE

# Parse summary (last few lines contain stats)
$summary = ($output | Select-Object -Last 10) -join "`n"
```

---

## 32. DPAPI Credential Storage

Store credentials securely without external modules (no CredentialManager dependency).

### How It Works
`Export-Clixml` serializes objects to XML. For `[PSCredential]` objects, it automatically uses the **Windows Data Protection API (DPAPI)** to encrypt the password. The encrypted data is tied to:
- The **specific user account** that encrypted it
- The **specific machine** it was encrypted on

No other user or machine can decrypt it — even with access to the XML file.

### Store a Credential
```powershell
$credPath = Join-Path $PSScriptRoot 'config/smtp_credential.xml'

# Interactive — prompts user
$cred = Get-Credential -Message 'Enter SMTP credentials'
$cred | Export-Clixml -LiteralPath $credPath -Force

# Programmatic — from known values
$securePass = ConvertTo-SecureString 'P@ssw0rd' -AsPlainText -Force
$cred = [PSCredential]::new('user@example.com', $securePass)
$cred | Export-Clixml -LiteralPath $credPath -Force
```

### Load a Credential
```powershell
$credPath = Join-Path $PSScriptRoot 'config/smtp_credential.xml'

try {
    $cred = Import-Clixml -LiteralPath $credPath
    # Use it:
    $cred.UserName                              # Plain text username
    $cred.GetNetworkCredential().Password       # Decrypted password
}
catch [System.Management.Automation.ItemNotFoundException] {
    Write-Warning 'No stored credential found. Run Set-Credential first.'
}
catch {
    # Decryption failure (wrong user/machine)
    Write-Warning "Cannot decrypt credential: $($_.Exception.Message)"
}
```

### Pattern: Config Points to Credential File
```powershell
# In config JSON — store the concept, not the credential
{
    "email": {
        "smtpServer": "smtp.office365.com",
        "from": "backup@company.com",
        "credentialTarget": "MyApp_SMTP"   # Logical name, not a file path
    }
}

# Credential path derived at runtime
function Get-CredentialPath {
    $configDir = Join-Path $PSScriptRoot '../config'
    return Join-Path $configDir 'smtp_credential.xml'
}
```

### Important Limitations
| Constraint | Impact |
|-----------|--------|
| Same user only | Different user account cannot decrypt |
| Same machine only | Copying the XML to another PC won't work |
| Profile migration | Moving user profile to new PC breaks it — user must re-enter |
| Service accounts | Works if the service runs as a named user (not LocalSystem) |
| Automation | Store credential interactively once, automated scripts read it back |

**User documentation tip:** Always tell users that credentials are machine-specific and must be re-entered after OS reinstall, user profile migration, or moving to a new PC.

---

## Appendix: Claude-Specific Guidance

### When Generating PowerShell for Users
1. **Assume Windows 10+ and Windows PowerShell 5.1** unless told otherwise — no need to account for Win 7/8 or WMF prerequisites
2. **No external module dependencies** unless the user approves — stick to what ships with Windows
3. **Always include a preview/dry-run mode** for destructive operations (rename, delete, modify)
4. **Use `-WhatIf`** where supported, or build a manual preview step
5. **Handle the single-item array gotcha** — always `@()` wrap cmdlet output that will be counted or iterated
6. **Specify encoding explicitly** on every file write operation
7. **Use `-LiteralPath`** by default, `-Path` only when wildcard expansion is intended
8. **Provide clear console output** — color-coded status, progress indicators for long operations
9. **Validate user input** — use parameter validation attributes or explicit checks for interactive scripts
10. **Include error handling** — at minimum, `try/catch` around file and network operations

### Project-Specific Extension Process

When the user requests a new PowerShell project:

**Step 1: Read this baseline skill in full** before writing any code.

**Step 2: Classify the task** against these domain categories:
- File system operations
- Data format handling (JSON/XML/CSV/etc.)
- Web/API interaction
- Active Directory / identity management
- Remote operations
- System administration (services, registry, scheduled tasks)
- Event log / monitoring
- External tool integration

If the task touches a category NOT covered in this baseline, flag it.

**Step 3: Check for these specific risk factors:**
- Does the task involve destructive operations? → Ensure `-WhatIf`/preview mode
- Does the task run unattended (scheduled task, automation)? → Ensure robust error handling and logging
- Does the task interact with external systems (APIs, remote machines, AD)? → Check authentication and retry patterns
- Does the task process user-provided paths or input? → Check sanitization patterns
- Will the output be consumed by other tools? → Check encoding requirements

**Step 4: Propose the approach to the user before writing code:**
- What the script will do (brief)
- Which baseline patterns apply
- What's NOT covered by the baseline that this project needs
- For each gap: whether it should be a permanent baseline addition (useful across 3+ potential future projects) or session-scoped (one-time need)
- Any assumptions being made about the environment

**Step 5: Build the script** following baseline patterns.

**Step 6: Self-review against the Gotcha Table** (Section 27) before presenting the script. Explicitly check each applicable gotcha.

**Step 7: If the script doesn't work,** diagnose the failure, check whether the issue is covered by the baseline (and was missed) or represents a new gotcha, fix the script, and recommend whether the new issue should be added to the baseline.
