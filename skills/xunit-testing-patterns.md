# xUnit Testing Patterns

xUnit testing conventions, test isolation, assertions, and patterns for .NET 10+ projects. Written for a developer transitioning from Pester -- maps Pester patterns to xUnit equivalents.

**Status: FOUNDATIONAL** -- core patterns from research. Will be expanded with project-specific lessons during implementation.

---

## 1. Target Environment
- xUnit 2.x (latest stable)
- .NET 10
- `dotnet test` runner
- VS Code Test Explorer (via C# Dev Kit)

---

## 2. Pester-to-xUnit Mapping

| Pester | xUnit | Notes |
|--------|-------|-------|
| `Describe "ConfigService"` | `public class ConfigServiceTests` | One test class per unit under test |
| `Context "when file exists"` | Region or nested class | Grouping is less formal in xUnit |
| `It "should load config"` | `[Fact] public void Load_ValidFile_ReturnsConfig()` | Individual test |
| `It -TestCases @(...)` | `[Theory] + [InlineData(...)]` | Parameterized tests |
| `BeforeAll { }` | `IClassFixture<T>` | Shared setup for all tests in class |
| `BeforeEach { }` | Constructor | New instance per test = fresh state |
| `AfterEach { }` | `IDisposable.Dispose()` | Cleanup after each test |
| `AfterAll { }` | `IClassFixture<T>` + `IDisposable` | Shared teardown |
| `Should -Be $expected` | `Assert.Equal(expected, actual)` | Note: expected FIRST |
| `Should -BeTrue` | `Assert.True(condition)` | |
| `Should -BeFalse` | `Assert.False(condition)` | |
| `Should -BeNullOrEmpty` | `Assert.Null(obj)` or `Assert.Empty(str)` | Separate assertions |
| `Should -Throw` | `Assert.Throws<T>(() => ...)` | Catches and returns the exception |
| `Should -Not -Throw` | (no assertion -- just call it) | If it throws, test fails |
| `Should -Contain` | `Assert.Contains(item, collection)` | |
| `Should -HaveCount 3` | `Assert.Equal(3, collection.Count)` | |
| `Mock -CommandName Get-Content` | Moq: `Mock<IFileSystem>()` | Interface-based mocking |
| `InModuleScope` | Direct construction (no scope boundary) | C# classes are directly testable |
| `New-Item -ItemType Directory` (temp dir) | `Path.GetTempPath()` + `Directory.CreateTempSubdirectory()` | |

---

## 3. Test Class Structure

### Basic Pattern
```csharp
using MyApp.Core.Configuration;

namespace MyApp.Tests.Configuration;

public class ConfigServiceTests : IDisposable
{
    private readonly string _tempDir;
    private readonly ConfigService _service;

    // Constructor = setup (runs before EACH test -- new instance per test)
    public ConfigServiceTests()
    {
        _tempDir = Path.Combine(Path.GetTempPath(), $"myapp-test-{Guid.NewGuid()}");
        Directory.CreateDirectory(_tempDir);
        _service = new ConfigService();
    }

    [Fact]
    public void Load_ValidFile_ReturnsConfig()
    {
        // Arrange
        string configPath = Path.Combine(_tempDir, "config.json");
        File.WriteAllText(configPath, """{"general": {"logRetentionDays": 30}}""");

        // Act
        var config = _service.Load(configPath);

        // Assert
        Assert.NotNull(config);
        Assert.Equal(30, config.General.LogRetentionDays);
    }

    [Fact]
    public void Load_MissingFile_ThrowsFileNotFound()
    {
        // Arrange
        string bogusPath = Path.Combine(_tempDir, "nope.json");

        // Act & Assert
        Assert.Throws<FileNotFoundException>(() => _service.Load(bogusPath));
    }

    // Dispose = teardown (runs after EACH test)
    public void Dispose()
    {
        if (Directory.Exists(_tempDir))
        {
            Directory.Delete(_tempDir, recursive: true);
        }
    }
}
```

### Key Insight: Isolation by Default
xUnit creates a **new class instance for every test method**. This means:
- Constructor runs fresh for each test -- no shared state leaks between tests
- No need for explicit `[SetUp]` or `[TearDown]` attributes (those are NUnit)
- Each test gets its own `_tempDir`, its own `_service`, etc.

This is fundamentally different from Pester, where `BeforeAll` runs once and `BeforeEach` runs per test. In xUnit, the constructor IS `BeforeEach`, automatically.

---

## 4. Test Naming Convention

Pattern: `Method_Scenario_ExpectedResult`

```csharp
[Fact]
public void Load_ValidFile_ReturnsConfig() { }

[Fact]
public void Load_MissingFile_ThrowsFileNotFound() { }

[Fact]
public void Validate_NullConfig_ReturnsFalse() { }

[Fact]
public void Save_ValidConfig_WritesJsonToDisk() { }
```

Alternative (descriptive sentence):
```csharp
[Fact]
public void Returns_config_when_file_is_valid() { }
```

Pick one style and use it consistently within the project.

---

## 5. Parameterized Tests

### [Theory] + [InlineData] (simple values)
```csharp
[Theory]
[InlineData(0, true)]    // 0 = success
[InlineData(1, true)]    // 1 = some files copied
[InlineData(7, true)]    // 7 = files mismatched
[InlineData(8, false)]   // 8 = errors
[InlineData(16, false)]  // 16 = fatal
public void IsSuccess_ReturnsCorrectly(int exitCode, bool expected)
{
    var result = RobocopyRunner.IsSuccess(exitCode);
    Assert.Equal(expected, result);
}
```

### [Theory] + [MemberData] (complex objects)
```csharp
public static IEnumerable<object[]> InvalidConfigs =>
    new List<object[]>
    {
        new object[] { "", "empty string" },
        new object[] { "{}", "empty object" },
        new object[] { "not json", "invalid JSON" },
    };

[Theory]
[MemberData(nameof(InvalidConfigs))]
public void Validate_InvalidInput_ReturnsFalse(string json, string description)
{
    var result = ConfigValidator.IsValid(json);
    Assert.False(result, $"Expected invalid for: {description}");
}
```

Pester comparison: `[Theory]` + `[InlineData]` maps directly to `It -TestCases @(...)`.

---

## 6. Common Assertions

```csharp
// Equality
Assert.Equal(expected, actual);           // expected FIRST (opposite of some frameworks)
Assert.NotEqual(unexpected, actual);

// Boolean
Assert.True(condition);
Assert.False(condition);

// Null
Assert.Null(obj);
Assert.NotNull(obj);

// String
Assert.Contains("substring", actualString);
Assert.StartsWith("prefix", actualString);
Assert.EndsWith("suffix", actualString);
Assert.Empty(actualString);

// Collections
Assert.Contains(item, collection);
Assert.DoesNotContain(item, collection);
Assert.Empty(collection);
Assert.Single(collection);               // Exactly one element
Assert.Equal(3, collection.Count);

// Exceptions
var ex = Assert.Throws<FileNotFoundException>(() => service.Load("nope"));
Assert.Contains("nope", ex.Message);      // Can inspect the exception

// Type
Assert.IsType<ConfigService>(obj);

// Range
Assert.InRange(value, low, high);
```

---

## 7. Test Isolation Patterns

### Temp Directory (file I/O tests)
```csharp
public class FileOperationTests : IDisposable
{
    private readonly string _tempDir;

    public FileOperationTests()
    {
        _tempDir = Path.Combine(Path.GetTempPath(), $"myapp-{Guid.NewGuid()}");
        Directory.CreateDirectory(_tempDir);
    }

    public void Dispose()
    {
        if (Directory.Exists(_tempDir))
            Directory.Delete(_tempDir, recursive: true);
    }
}
```

Pester comparison: same pattern as `$TestDrive` or `New-Item -ItemType Directory -Path $env:TEMP`.

### Shared Expensive Fixtures (IClassFixture)
```csharp
// Fixture -- created once, shared across all tests in the class
public class DatabaseFixture : IDisposable
{
    public string ConnectionString { get; }

    public DatabaseFixture()
    {
        // Expensive setup -- runs once
        ConnectionString = SetupTestDatabase();
    }

    public void Dispose() => TeardownTestDatabase();
}

// Test class uses the fixture
public class DatabaseTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;

    public DatabaseTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public void CanConnect()
    {
        // Use _fixture.ConnectionString
    }
}
```

Pester comparison: `IClassFixture<T>` maps to `BeforeAll` / `AfterAll`.

---

## 8. Mocking (Interface-Based)

Unlike Pester (which mocks functions by name), C# mocking requires **interfaces**. Design for testability by depending on interfaces, not concrete classes.

```csharp
// Interface
public interface IFileSystem
{
    bool FileExists(string path);
    string ReadAllText(string path);
}

// Production implementation
public class RealFileSystem : IFileSystem
{
    public bool FileExists(string path) => File.Exists(path);
    public string ReadAllText(string path) => File.ReadAllText(path);
}

// In tests (using Moq)
var mockFs = new Mock<IFileSystem>();
mockFs.Setup(fs => fs.FileExists("config.json")).Returns(true);
mockFs.Setup(fs => fs.ReadAllText("config.json")).Returns("{\"general\": {}}");

var service = new ConfigService(mockFs.Object);
var config = service.Load("config.json");

Assert.NotNull(config);
mockFs.Verify(fs => fs.ReadAllText("config.json"), Times.Once);
```

**When to mock vs when to use real files:**
- Mock for unit tests where you're testing logic, not I/O
- Use real temp files for integration tests where file I/O behavior matters
- Integration tests for external process wrappers will likely use real temp directories (testing the Process wrapper, not faking it)

---

## 9. Test Project Setup

### Required NuGet Packages
```xml
<PackageReference Include="xunit" Version="2.*" />
<PackageReference Include="xunit.runner.visualstudio" Version="3.*" />
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
<PackageReference Include="coverlet.collector" Version="6.*" />
<!-- Add when mocking is needed: -->
<!-- <PackageReference Include="Moq" Version="4.*" /> -->
```

### Running Tests
```bash
dotnet test                           # Run all tests
dotnet test --filter "ClassName"      # Run tests in a specific class
dotnet test --filter "Method_Name"    # Run a specific test
dotnet test --verbosity detailed      # Verbose output
dotnet test --collect:"XPlat Code Coverage"  # With coverage
```

### Test File Organization
Mirror the source structure:
```
src/MyApp.Core/Configuration/ConfigService.cs
tests/MyApp.Tests/Configuration/ConfigServiceTests.cs

src/MyApp.Core/Engine/RobocopyRunner.cs
tests/MyApp.Tests/Engine/RobocopyRunnerTests.cs
```

---

## 10. Integration Test Patterns

For tests that need real file I/O or process execution (robocopy, etc.):

```csharp
public class RobocopyIntegrationTests : IDisposable
{
    private readonly string _sourceDir;
    private readonly string _destDir;

    public RobocopyIntegrationTests()
    {
        var baseDir = Path.Combine(Path.GetTempPath(), $"myapp-integ-{Guid.NewGuid()}");
        _sourceDir = Path.Combine(baseDir, "source");
        _destDir = Path.Combine(baseDir, "dest");
        Directory.CreateDirectory(_sourceDir);
        Directory.CreateDirectory(_destDir);

        // Create test files
        File.WriteAllText(Path.Combine(_sourceDir, "test.txt"), "test content");
    }

    [Fact]
    public void CopyFile_RealRobocopy_CopiesSuccessfully()
    {
        var runner = new RobocopyRunner();
        var result = runner.CopyFolder(_sourceDir, _destDir);

        Assert.True(result.IsSuccess);
        Assert.True(File.Exists(Path.Combine(_destDir, "test.txt")));
    }

    public void Dispose()
    {
        var baseDir = Path.GetDirectoryName(_sourceDir)!;
        if (Directory.Exists(baseDir))
            Directory.Delete(baseDir, recursive: true);
    }
}
```

---

## Sections to Expand During Project
- [ ] Async test patterns (`[Fact] public async Task ...`)
- [ ] Custom assertions for common project patterns
- [ ] Test data builders for complex config objects
- [ ] Code coverage configuration and thresholds
- [ ] Collection fixtures for test ordering control
- [ ] Moq patterns discovered during implementation
