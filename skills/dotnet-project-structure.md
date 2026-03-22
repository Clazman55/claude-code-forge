# .NET Project Structure

.NET solution and project layout, NuGet dependency management, build configuration, and publishing for Windows desktop applications.

**Status: FOUNDATIONAL** -- core patterns from research. Will be expanded with project-specific lessons during implementation.

---

## 1. Target Environment
- .NET 10 SDK (LTS)
- Windows 10/11
- VS Code with C# Dev Kit (or OmniSharp)

---

## 2. Solution Layout

### Standard Structure
```
ProjectName/
  global.json                     # Pins SDK version
  ProjectName.slnx                # Solution file (.NET 10 XML format)
  .gitignore                      # .NET-specific ignores
  .editorconfig                   # Code style rules (optional but recommended)
  src/
    ProjectName.Core/             # Class library -- business logic
      ProjectName.Core.csproj
      Configuration/
      Logging/
      Engine/
    ProjectName.Cli/              # Console app -- CLI entry point
      ProjectName.Cli.csproj
      Program.cs
    ProjectName.Gui/              # WPF app -- GUI entry point
      ProjectName.Gui.csproj
  tests/
    ProjectName.Tests/            # xUnit test project
      ProjectName.Tests.csproj
  config/                         # Runtime config files
```

### Dependency Direction
```
Cli --> Core <-- Tests
Gui --> Core
```
- Core has no project references (leaf dependency)
- Cli and Gui reference Core
- Tests reference Core (and only Core)
- No circular references allowed

### Namespace Convention
- Root namespace matches project name: `MyApp.Core`
- Subdirectories become namespace segments: `MyApp.Core.Configuration`
- File location matches namespace: `src/MyApp.Core/Configuration/ConfigService.cs` -> `namespace MyApp.Core.Configuration`

PowerShell comparison: PS module architecture has a flat function structure with dot-sourcing. C# uses namespaces and project references for the same organization.

---

## 3. dotnet CLI Commands

### Project Creation
```bash
dotnet new globaljson --sdk-version 10.0.201 --roll-forward latestFeature
dotnet new sln --name ProjectName
dotnet new classlib -n ProjectName.Core -o src/ProjectName.Core --framework net10.0
dotnet new console -n ProjectName.Cli -o src/ProjectName.Cli --framework net10.0
dotnet new wpf -n ProjectName.Gui -o src/ProjectName.Gui --framework net10.0-windows
dotnet new xunit -n ProjectName.Tests -o tests/ProjectName.Tests --framework net10.0
```

### Wiring Up the Solution
```bash
# Add projects to solution
dotnet sln add src/ProjectName.Core/ProjectName.Core.csproj
dotnet sln add src/ProjectName.Cli/ProjectName.Cli.csproj
dotnet sln add tests/ProjectName.Tests/ProjectName.Tests.csproj

# Add project references
dotnet add src/ProjectName.Cli reference src/ProjectName.Core
dotnet add tests/ProjectName.Tests reference src/ProjectName.Core
```

### Daily Workflow
```bash
dotnet build              # Compile all projects
dotnet test               # Run all tests
dotnet run --project src/ProjectName.Cli    # Run the CLI app
dotnet clean              # Remove build output
dotnet restore            # Restore NuGet packages (usually automatic)
```

### NuGet Package Management
```bash
dotnet add src/ProjectName.Cli package System.CommandLine --version 2.0.5
dotnet add tests/ProjectName.Tests package Moq
dotnet list package                          # Show all packages
dotnet list package --outdated               # Check for updates
```

PowerShell comparison: `dotnet build` is like running `Import-Module` to verify the code loads. `dotnet test` is like `Invoke-Pester`. `dotnet run` is like `& .\script.ps1`.

---

## 4. .csproj Configuration

### Class Library (Core)
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

### Console App (CLI)
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\MyApp.Core\MyApp.Core.csproj" />
    <PackageReference Include="System.CommandLine" Version="2.0.5" />
  </ItemGroup>
</Project>
```

### WPF App (GUI)
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net10.0-windows</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <UseWPF>true</UseWPF>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\MyApp.Core\MyApp.Core.csproj" />
  </ItemGroup>
</Project>
```

### xUnit Test Project
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
    <PackageReference Include="xunit" Version="2.*" />
    <PackageReference Include="xunit.runner.visualstudio" Version="3.*" />
    <PackageReference Include="coverlet.collector" Version="6.*" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\..\src\MyApp.Core\MyApp.Core.csproj" />
  </ItemGroup>
</Project>
```

### Key Properties
| Property | Purpose |
|----------|---------|
| `<TargetFramework>` | `net10.0` or `net10.0-windows` (WPF) |
| `<Nullable>enable</Nullable>` | Nullable reference type warnings |
| `<ImplicitUsings>enable</ImplicitUsings>` | Auto-imports common namespaces |
| `<OutputType>Exe</OutputType>` | Console app (omit for class library) |
| `<OutputType>WinExe</OutputType>` | GUI app (no console window) |
| `<UseWPF>true</UseWPF>` | Enable WPF support |
| `<IsPackable>false</IsPackable>` | Don't create NuGet package (test projects) |

---

## 5. global.json

Pins the SDK version to avoid PATH resolution issues (e.g., x86 3.1 vs x64 10.0):

```json
{
  "sdk": {
    "version": "10.0.201",
    "rollForward": "latestFeature"
  }
}
```

`rollForward: latestFeature` allows using newer patch versions (10.0.202, etc.) without updating the file.

---

## 6. NuGet Packages (Known Needed)

| Package | Purpose | Project |
|---------|---------|---------|
| `System.CommandLine` 2.0.5 | CLI parsing, subcommands, --help | Cli |
| `xunit` | Test framework | Tests |
| `xunit.runner.visualstudio` | VS Code test discovery | Tests |
| `Microsoft.NET.Test.Sdk` | Test runner infrastructure | Tests |
| `coverlet.collector` | Code coverage | Tests |
| `Moq` or `NSubstitute` | Mocking (add when needed) | Tests |

### NuGet Source Setup
Fresh .NET SDK installs may have no NuGet sources configured:
```bash
dotnet nuget list source
dotnet nuget add source https://api.nuget.org/v3/index.json -n nuget.org
```

---

## 7. Publishing

### Single-File Self-Contained (Primary Release Format)
```bash
dotnet publish src/ProjectName.Cli -c Release -r win-x64 --self-contained -p:PublishSingleFile=true
```

Output: single `.exe` in `src/ProjectName.Cli/bin/Release/net10.0/win-x64/publish/`

### Key Publish Options
| Option | Effect |
|--------|--------|
| `-c Release` | Release configuration (optimized) |
| `-r win-x64` | Runtime identifier -- Windows 64-bit |
| `--self-contained` | Bundle .NET runtime (no runtime install needed on target) |
| `-p:PublishSingleFile=true` | Single executable output |
| `-p:PublishTrimmed=true` | Remove unused code (reduces size, may break reflection) |
| `-p:IncludeNativeLibrariesForSelfExtract=true` | Include native libs in single file |

### Publishing Notes
- Self-contained single-file for Windows is the primary deployment target
- Trimming (`PublishTrimmed`) reduces file size but can break reflection-based features -- test thoroughly before enabling
- WPF apps use `OutputType=WinExe` and `net10.0-windows` TFM
- Published exe size: ~60-80MB self-contained without trimming, ~20-40MB with trimming

PowerShell comparison: `dotnet publish` is like a release build script -- creates a deployable artifact. The single-file publish means no runtime needed on target.

---

## 8. .gitignore for .NET Projects

```
# Build output
bin/
obj/

# NuGet
*.nupkg
packages/

# User config (runtime-generated)
config/*.json
!config/defaults.json

# Credentials
config/smtp_credential.dat

# Logs
logs/

# IDE
.vs/
.vscode/
*.code-workspace
*.user
*.suo

# Publish output
publish/

# Test results
TestResults/
coverage/

# Project docs (dev reference, not shipped)
docs/

# OS
Thumbs.db
Desktop.ini
```

---

## 9. System.CommandLine Pattern

### Root Command with Subcommands
```csharp
using System.CommandLine;

var rootCommand = new RootCommand("MyApp -- Description here");

// backup command
var backupCommand = new Command("backup", "Run backup");
var dryRunOption = new Option<bool>("--dry-run", "Show what would happen");
var configOption = new Option<string?>("--config", "Config file path");
backupCommand.AddOption(dryRunOption);
backupCommand.AddOption(configOption);
backupCommand.SetHandler((dryRun, configPath) =>
{
    // backup logic here
}, dryRunOption, configOption);
rootCommand.AddCommand(backupCommand);

// config subcommands
var configCommand = new Command("config", "Configuration management");
var validateCommand = new Command("validate", "Validate config file");
validateCommand.SetHandler(() => { /* validation logic */ });
var initCommand = new Command("init", "Create default config");
initCommand.SetHandler(() => { /* init logic */ });
configCommand.AddCommand(validateCommand);
configCommand.AddCommand(initCommand);
rootCommand.AddCommand(configCommand);

return await rootCommand.InvokeAsync(args);
```

This gives you `myapp backup --dry-run`, `myapp config validate`, `myapp config init`, and auto-generated `--help` for free.

---

## Sections to Expand During Project
- [ ] Directory.Build.props for shared properties across projects
- [ ] .editorconfig rules and enforcement
- [ ] Build configurations (Debug vs Release specifics)
- [ ] CI/CD considerations (if ever needed)
- [ ] WPF-specific project setup details (Phase 6)
