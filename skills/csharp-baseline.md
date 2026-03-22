# C# Baseline

C# language fundamentals, coding standards, and patterns for .NET 10+ Windows desktop applications. Written for a developer transitioning from PowerShell -- maps PS patterns to C# idioms where helpful.

**Status: FOUNDATIONAL** -- core patterns from research. Will be expanded with project-specific lessons during implementation.

---

## 1. Target Environment
- .NET 10 (LTS, supported through November 2028)
- C# 13 (language version shipped with .NET 10)
- Windows-only (WPF, robocopy, Task Scheduler)
- `global.json` pins SDK version at project root

---

## 2. Naming Conventions

| Element | Style | Example |
|---------|-------|---------|
| Namespace | PascalCase | `MyApp.Core.Configuration` |
| Class / Record / Struct | PascalCase | `ConfigService`, `BackupJob` |
| Interface | IPascalCase | `IHashVerifier`, `ILogger` |
| Public method | PascalCase | `LoadConfig()`, `RunBackup()` |
| Private method | PascalCase | `ValidateSchema()` |
| Public property | PascalCase | `MaxRetries`, `IsEnabled` |
| Private field | _camelCase | `_config`, `_logger` |
| Local variable | camelCase | `filePath`, `exitCode` |
| Parameter | camelCase | `sourcePath`, `hashAlgorithm` |
| Constant | PascalCase | `DefaultTimeout`, `MaxRetries` |
| Enum type | PascalCase singular | `LogLevel`, `BackupStatus` |
| Enum value | PascalCase | `LogLevel.Warning`, `BackupStatus.Failed` |

PowerShell comparison: PS uses Verb-Noun for functions, C# uses PascalCase methods. PS `$camelCase` variables map to C# `camelCase` locals. PS `$script:Variable` maps to C# `private static` or `_field`.

---

## 3. Code Style

### Braces and Layout
- Allman style (opening brace on own line) -- C# default convention
- Always use braces for `if`/`else`/`for`/`while`, even single-line bodies
- One class per file (enforced by convention, not compiler)
- File name matches class name: `ConfigService.cs` contains `class ConfigService`

### Type Usage
- Use `var` when the type is obvious from the right side: `var list = new List<string>();`
- Use explicit types when clarity matters: `int exitCode = process.ExitCode;`
- Prefer `string` over `String`, `int` over `Int32` (C# aliases over .NET types)

### Nullable Reference Types
- Enabled project-wide via `<Nullable>enable</Nullable>` in .csproj
- `string?` means nullable, `string` means non-null
- Compiler warns on potential null dereference -- respect these warnings
- Use `!` (null-forgiving) sparingly and only when you know better than the compiler

### readonly and const
- `const` for compile-time constants: `const int MaxRetries = 3;`
- `static readonly` for runtime constants: `static readonly TimeSpan Timeout = TimeSpan.FromSeconds(30);`
- `readonly` on fields that shouldn't change after construction
- PowerShell comparison: no PS equivalent. PS variables are always mutable.

---

## 4. String Handling

### Interpolation (primary -- use for most cases)
```csharp
string message = $"Copying {source} to {destination}";
string path = $@"C:\Users\{username}\Documents";  // verbatim interpolated
```

### StringBuilder (loops and large construction)
```csharp
var sb = new StringBuilder();
foreach (var line in logEntries)
{
    sb.AppendLine($"[{line.Timestamp:HH:mm:ss}] {line.Message}");
}
string output = sb.ToString();
```

Rule of thumb: string interpolation for simple expressions. StringBuilder when appending in a loop or building more than ~10 segments. String concatenation with `+` is fine for 2-3 pieces.

### Raw String Literals (C# 11+)
```csharp
string json = """
    {
        "name": "test",
        "enabled": true
    }
    """;
```
Triple-quoted, whitespace-trimmed. Useful for embedded JSON, SQL, etc.

PowerShell comparison: PS `"$variable"` maps to C# `$"{variable}"`. PS here-strings (`@"..."@`) map to raw string literals.

---

## 5. Collections and LINQ

### Common Collections
| PS Type | C# Type | Notes |
|---------|---------|-------|
| `[System.Collections.Generic.List[string]]` | `List<string>` | Same type, cleaner syntax |
| `[hashtable]` | `Dictionary<string, T>` | Strongly typed in C# |
| `[System.Collections.Generic.HashSet[string]]` | `HashSet<string>` | Same |
| `@()` (array) | `string[]` or `List<string>` | Arrays are fixed-size; use List when adding |

### Essential LINQ Operations
```csharp
// Filtering (PS: Where-Object)
var errors = logEntries.Where(e => e.Level == LogLevel.Error);

// Projection (PS: Select-Object -Property)
var names = jobs.Select(j => j.Name);

// First match (PS: ... | Select-Object -First 1)
var first = items.FirstOrDefault(i => i.IsEnabled);
// Returns null (reference types) or default (value types) if no match -- no exception

// Any match exists? (PS: ... | Where-Object { $_ } | Measure | .Count -gt 0)
bool hasErrors = logEntries.Any(e => e.Level == LogLevel.Error);

// Aggregation
int total = items.Sum(i => i.Size);
int count = items.Count(i => i.IsEnabled);

// Ordering
var sorted = items.OrderBy(i => i.Name).ThenByDescending(i => i.Date);

// ToList / ToArray -- materializes the query
List<string> names = jobs.Select(j => j.Name).ToList();
```

Key difference from PS: LINQ is lazy (deferred execution). The query runs when you enumerate it, not when you define it. Call `.ToList()` or `.ToArray()` to materialize immediately.

### First() vs FirstOrDefault()
- `First()` -- throws `InvalidOperationException` if no match. Use when absence is a bug.
- `FirstOrDefault()` -- returns null/default if no match. Use when absence is valid.
- Same pattern: `Single()` (exactly one, throws if zero or multiple) vs `SingleOrDefault()`.

---

## 6. File I/O

```csharp
// Read all text (small files)
string content = File.ReadAllText(path);

// Read all lines
string[] lines = File.ReadAllLines(path);

// Write (overwrites)
File.WriteAllText(path, content);

// Append
File.AppendAllText(path, newContent);

// Check existence
if (File.Exists(path)) { ... }
if (Directory.Exists(dirPath)) { ... }

// Path manipulation (PS: [System.IO.Path] -- same class)
string combined = Path.Combine(baseDir, "config", "defaults.json");
string ext = Path.GetExtension(filePath);
string nameOnly = Path.GetFileNameWithoutExtension(filePath);
```

PowerShell comparison: `Get-Content` maps to `File.ReadAllLines()`. `Set-Content` maps to `File.WriteAllText()`. `Test-Path` maps to `File.Exists()` / `Directory.Exists()`. `Join-Path` maps to `Path.Combine()`.

---

## 7. Error Handling

### try/catch/finally
```csharp
try
{
    var config = ConfigService.Load(path);
}
catch (FileNotFoundException ex)
{
    Logger.Error($"Config not found: {ex.FileName}");
    // Handle specifically
}
catch (JsonException ex)
{
    Logger.Error($"Invalid JSON in config: {ex.Message}");
}
catch (Exception ex)
{
    Logger.Error($"Unexpected error: {ex.Message}");
    throw;  // Re-throw preserving stack trace
}
finally
{
    // Always runs -- cleanup
}
```

### Critical Rules
- **`throw;` not `throw ex;`** -- `throw ex;` resets the stack trace, losing the origin
- **Catch specific exceptions** before general `Exception`
- **Don't catch Exception just to log and swallow** -- catch what you can handle, let the rest propagate
- **Don't use exceptions for flow control** -- check conditions first (`File.Exists`, `TryParse`, etc.)

PowerShell comparison: PS `$ErrorActionPreference = 'Stop'` + `try/catch` maps to C# try/catch. PS `-ErrorAction SilentlyContinue` has no direct equivalent -- in C#, you catch and handle or you don't.

---

## 8. IDisposable and Resource Management

### using Statement (preferred)
```csharp
// Modern C# -- using declaration (no braces, disposes at end of scope)
using var stream = File.OpenRead(path);
using var reader = new StreamReader(stream);
string content = reader.ReadToEnd();
// stream and reader disposed automatically when method exits

// Classic form (explicit scope)
using (var connection = new SqlConnection(connString))
{
    // connection disposed when block exits
}
```

### When to Implement IDisposable
- Your class holds unmanaged resources (file handles, process handles, COM objects)
- Your class owns another IDisposable object as a field
- You do NOT need IDisposable just because you use a disposable object locally (that's what `using` is for)

### Pattern
```csharp
public class Logger : IDisposable
{
    private StreamWriter? _writer;
    private bool _disposed;

    public void Dispose()
    {
        if (!_disposed)
        {
            _writer?.Dispose();
            _disposed = true;
        }
    }
}
```

PowerShell comparison: no direct equivalent. PS relies on garbage collection. C# gives you explicit control via `using` and `IDisposable`.

---

## 9. System.Text.Json

### Deserialization
```csharp
var options = new JsonSerializerOptions
{
    PropertyNameCaseInsensitive = true,  // Match PS ConvertFrom-Json behavior
    WriteIndented = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
};

// From file
string json = File.ReadAllText(configPath);
AppConfig config = JsonSerializer.Deserialize<AppConfig>(json, options)
    ?? throw new InvalidOperationException("Config deserialized to null");

// From string
var data = JsonSerializer.Deserialize<Dictionary<string, object>>(jsonString, options);
```

### Serialization
```csharp
string json = JsonSerializer.Serialize(config, options);
File.WriteAllText(configPath, json);
```

### Config Model Pattern
```csharp
public class AppConfig
{
    public GeneralConfig General { get; set; } = new();
    public BackupConfig Backup { get; set; } = new();
    public EmailConfig? Email { get; set; }  // Nullable -- optional section
}

public class GeneralConfig
{
    public string LogDirectory { get; set; } = "";
    public int LogRetentionDays { get; set; } = 30;
}
```

### Key Options for Cross-Platform Compatibility
- `PropertyNameCaseInsensitive = true` -- PS `ConvertFrom-Json` is case-insensitive
- `PropertyNamingPolicy = JsonNamingPolicy.CamelCase` -- for serialization output matching PS convention
- `WriteIndented = true` -- human-readable output
- `[JsonPropertyName("fieldName")]` attribute to override individual property names if needed

PowerShell comparison: `ConvertFrom-Json` maps to `JsonSerializer.Deserialize<T>()`. `ConvertTo-Json -Depth 10` maps to `JsonSerializer.Serialize()` with appropriate options.

---

## 10. Process Execution (Robocopy, External Tools)

```csharp
var startInfo = new ProcessStartInfo
{
    FileName = "robocopy",
    Arguments = $"\"{source}\" \"{dest}\" /Z /NP /R:3 /W:5",
    UseShellExecute = false,
    RedirectStandardOutput = true,
    RedirectStandardError = true,
    CreateNoWindow = true
};

using var process = Process.Start(startInfo)
    ?? throw new InvalidOperationException("Failed to start robocopy");

string stdout = process.StandardOutput.ReadToEnd();
string stderr = process.StandardError.ReadToEnd();
process.WaitForExit();

int exitCode = process.ExitCode;
// Robocopy: 0-7 = success (varying levels), 8+ = errors
```

PowerShell comparison: `Start-Process -Wait -NoNewWindow` maps to `Process.Start()` with `UseShellExecute = false` + `WaitForExit()`. `$LASTEXITCODE` maps to `process.ExitCode`.

---

## 11. Common Mapping Table (PowerShell -> C#)

| PowerShell | C# | Notes |
|---|---|---|
| `$script:Variable` | `private static` field or singleton | Module-scoped state |
| `$PSScriptRoot` | `AppContext.BaseDirectory` | App's directory |
| `[CmdletBinding()]` params | Method parameters or System.CommandLine | |
| `Write-Host` / `Write-Output` | `Console.WriteLine()` | |
| `Write-Verbose` / `Write-Warning` | Logger methods | Custom logger |
| `$null` | `null` | |
| `-eq` / `-ne` / `-gt` / `-lt` | `==` / `!=` / `>` / `<` | |
| `-and` / `-or` / `-not` | `&&` / `||` / `!` | |
| `-match` (regex) | `Regex.IsMatch()` | `using System.Text.RegularExpressions;` |
| `-like` (wildcard) | No direct equivalent | Use `string.Contains()`, `StartsWith()`, `EndsWith()`, or Regex |
| `ForEach-Object { $_ }` | `.Select(x => ...)` or `foreach` | LINQ or loop |
| `Where-Object { $_ }` | `.Where(x => ...)` | LINQ |
| `Sort-Object` | `.OrderBy(x => ...)` | LINQ |
| `Measure-Object` | `.Count()`, `.Sum()`, etc. | LINQ |
| `[PSCustomObject]@{}` | Anonymous type or class | `new { Name = "test" }` |
| `Export-Clixml` (DPAPI) | `ProtectedData.Protect()` | Same DPAPI backend |
| `Get-FileHash` | `SHA256.HashData()` / `MD5.HashData()` | `System.Security.Cryptography` |
| `Test-Path` | `File.Exists()` / `Directory.Exists()` | |
| `Join-Path` | `Path.Combine()` | |
| `ConvertFrom-Json` | `JsonSerializer.Deserialize<T>()` | System.Text.Json |
| `ConvertTo-Json` | `JsonSerializer.Serialize()` | System.Text.Json |

---

## 12. Anti-Patterns

- **Don't catch bare `Exception` and swallow it** -- catch specific types, re-throw with `throw;` if needed
- **Don't use `async void`** except for event handlers -- use `async Task` for async methods
- **Don't use `throw ex;`** -- destroys stack trace. Use `throw;` to re-throw
- **Don't concatenate strings in tight loops** -- use `StringBuilder`
- **Don't use `Win32_Product` WMI class** -- same as PowerShell, triggers MSI reconfiguration
- **Don't ignore nullable warnings** -- they exist to prevent NullReferenceException
- **Don't implement `IDisposable` unless you hold resources** -- unnecessary complexity
- **Don't use `First()` when `FirstOrDefault()` is appropriate** -- know which exception behavior you want

---

## Sections to Expand During Project
- [ ] Async/await patterns (add when needed, not before)
- [ ] Dependency injection patterns (if complexity warrants it)
- [ ] WPF/MVVM patterns (Phase 6 -- compare with powershell-wpf skill)
- [ ] Additional patterns discovered during implementation
