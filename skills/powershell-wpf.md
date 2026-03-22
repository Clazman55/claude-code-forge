# WPF + PowerShell Skill Reference

## Purpose

Patterns and gotchas for building WPF GUI applications in PowerShell. Covers runtime C# type
compilation, XAML loading, MVVM data binding, and the critical scope issues that arise when
PowerShell scriptblocks are invoked as C# delegates. General-purpose — not tied to any
specific project.

## Target Environment

- **PowerShell 7+** (PS 5.1 works but has minor differences in `Add-Type` behavior)
- **Windows only** — WPF requires the Windows Desktop runtime
- **No NuGet/external packages** — everything uses built-in .NET assemblies

---

## 1. Assembly Loading

Load all required WPF assemblies once at startup. Use an idempotent check so
re-importing the module doesn't fail.

```powershell
function Initialize-GuiFramework {
    [CmdletBinding()]
    param()

    # Idempotent — skip if already compiled
    if ('MyApp.GUI.ViewModelBase' -as [type]) { return }

    Add-Type -AssemblyName PresentationFramework
    Add-Type -AssemblyName PresentationCore
    Add-Type -AssemblyName WindowsBase
    Add-Type -AssemblyName System.Windows.Forms   # Only if using FolderBrowserDialog
}
```

**Key points:**
- The `-as [type]` check is the idempotent gate — C# types persist in the AppDomain
- `System.Windows.Forms` is only needed for `FolderBrowserDialog` (WPF has no built-in folder picker)
- Load assemblies BEFORE `Add-Type -TypeDefinition` for the C# code that references them

---

## 2. C# Type Definitions

Define minimal C# types for WPF data binding. Compile via `Add-Type -TypeDefinition`.

### ViewModelBase (INotifyPropertyChanged)

```csharp
public class ViewModelBase : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;

    protected void OnPropertyChanged(string name)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
    }

    protected bool SetProperty<T>(ref T field, T value, string name)
    {
        if (EqualityComparer<T>.Default.Equals(field, value)) return false;
        field = value;
        OnPropertyChanged(name);
        return true;
    }
}
```

### RelayCommand (ICommand)

```csharp
public class RelayCommand : ICommand
{
    private readonly Action<object> _execute;
    private readonly Func<object, bool> _canExecute;
    public event EventHandler CanExecuteChanged;

    public RelayCommand(Action<object> execute, Func<object, bool> canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }

    public bool CanExecute(object parameter) => _canExecute?.Invoke(parameter) ?? true;
    public void Execute(object parameter) => _execute(parameter);
    public void RaiseCanExecuteChanged() => CanExecuteChanged?.Invoke(this, EventArgs.Empty);
}
```

### ViewModel Properties

Use `SetProperty<T>` for all bindable properties:

```csharp
public class MainViewModel : ViewModelBase
{
    private string _statusMessage = "Ready";
    private bool _isBusy;

    public string StatusMessage
    {
        get => _statusMessage;
        set => SetProperty(ref _statusMessage, value, nameof(StatusMessage));
    }
    public bool IsBusy
    {
        get => _isBusy;
        set => SetProperty(ref _isBusy, value, nameof(IsBusy));
    }

    // Commands — wired from PowerShell, not C#
    public ICommand SaveCommand { get; set; }
    public ICommand RunCommand { get; set; }
}
```

### Compilation

```powershell
$refs = @(
    'PresentationCore'
    'PresentationFramework'
    'WindowsBase'
    'System.ObjectModel'
    'System.Collections'
    'System.Runtime'
)

Add-Type -TypeDefinition $csharpSource -ReferencedAssemblies $refs
```

---

## 3. XAML Loading

Load XAML at runtime (not compiled BAML). Strip `x:Class` if present.

```powershell
$xamlPath = Join-Path -Path $PSScriptRoot -ChildPath 'XAML/MainWindow.xaml'
$xamlContent = Get-Content -LiteralPath $xamlPath -Raw -Encoding utf8

# Remove x:Class — only needed for compiled WPF projects
$xamlContent = $xamlContent -replace 'x:Class="[^"]*"\s*', ''

$window = [System.Windows.Markup.XamlReader]::Parse($xamlContent)
```

**Key points:**
- Use `XamlReader::Parse($string)` — it manages the internal XmlReader lifecycle automatically. The older `XmlReader::Create()` + `XamlReader::Load()` pattern leaks the reader (PowerShell has no `using` keyword for auto-dispose)
- `x:Class` must be removed or XAML loading fails with "type not found"
- Use `-Encoding utf8` on `Get-Content` to handle Unicode properly
- The returned object is a fully instantiated WPF `Window` (or whatever the root element is)
- Named elements can be found with `$window.FindName('ElementName')`

---

## 4. Data Binding

Set the ViewModel as the window's `DataContext`. All `{Binding}` expressions in XAML
resolve against it automatically.

```powershell
$vm = [MyApp.GUI.MainViewModel]::new()
$window.DataContext = $vm
```

### XAML binding examples

```xml
<!-- Text binding -->
<TextBox Text="{Binding SmtpServer, UpdateSourceTrigger=PropertyChanged}" />

<!-- Checkbox binding -->
<CheckBox IsChecked="{Binding EmailEnabled}" Content="Enable Email" />

<!-- Command binding -->
<Button Content="Save" Command="{Binding SaveCommand}" />

<!-- Visibility via BooleanToVisibilityConverter -->
<ProgressBar Visibility="{Binding IsBusy,
    Converter={StaticResource BoolToVis}}" />

<!-- List binding -->
<DataGrid ItemsSource="{Binding Jobs}"
          SelectedItem="{Binding SelectedJob}" />
```

**Key points:**
- `UpdateSourceTrigger=PropertyChanged` updates on every keystroke (default is `LostFocus`)
- The ViewModel must raise `PropertyChanged` for bindings to update the UI
- `ObservableCollection<T>` auto-notifies the UI when items are added/removed
- Two-way binding is the default for editable controls (`TextBox`, `CheckBox`)

---

## 5. Command Wiring — The $global: Context Pattern

**THIS IS THE MOST IMPORTANT SECTION.**

When PowerShell scriptblocks are cast to `[Action<object>]` and passed to a C# constructor
(like `RelayCommand`), they are invoked later from C# — in a completely different scope.

### What DOES NOT work

```powershell
# BAD — .GetNewClosure() is unreliable with C# delegates
$vm.SaveCommand = [MyApp.GUI.RelayCommand]::new(
    [Action[object]]({
        $vm.StatusMessage = 'Saving...'    # $vm may be $null at runtime
    }.GetNewClosure())
)
```

`.GetNewClosure()` captures local variables into a dynamic module scope. When C# invokes
the delegate, the captured variables may or may not be accessible depending on factors
that are not documented or predictable. Some closures work, others silently fail.
**You cannot rely on this.**

### What DOES work — $global: context hashtable

```powershell
# Store everything closures need in a single global hashtable
$global:AppCtx = @{
    VM         = $vm
    Window     = $window
    ModuleRoot = $script:ModuleRoot
    LogFile    = $script:LogFilePath
}

# Closures reference $global:AppCtx — always accessible from any scope
$vm.SaveCommand = [MyApp.GUI.RelayCommand]::new(
    [Action[object]]{
        $ctx = $global:AppCtx
        $ctx.VM.StatusMessage = 'Saving...'
        Export-Config -ViewModel $ctx.VM
    }
)

# Clean up when window closes
$window.Add_Closed({
    $global:AppCtx = $null
})
```

### Why this works

- `$global:` scope is always accessible from any invocation context, including C# delegates
- No `.GetNewClosure()` needed — the scriptblock resolves `$global:AppCtx` at runtime
- A single hashtable is a reference type — all closures see the same mutable state
- Cleanup on window close prevents leaking global state

### What goes in the context hashtable

| Key | Value | Purpose |
|-----|-------|---------|
| VM | The ViewModel instance | Property access from all commands |
| Window | The WPF Window | Dialog ownership, dispatcher access |
| ModuleRoot | `$script:ModuleRoot` value | Path resolution (`$script:` is inaccessible in delegates) |
| LogFile | Log file path | Any `$script:` value needed by closures |
| Timers | DispatcherTimer refs | So they can be stopped on cleanup |
| Async state | PowerShell/IAsyncResult | For async operation polling |

### Module-exported functions still work

Functions exported by `Export-ModuleMember` or listed in `.psd1` resolve via PowerShell's
normal command discovery. They do NOT need to be captured:

```powershell
$vm.SaveCommand = [MyApp.GUI.RelayCommand]::new(
    [Action[object]]{
        $ctx = $global:AppCtx
        Get-MyConfig           # Module-exported — works fine
        Set-MyConfig -Config $config  # Module-exported — works fine
    }
)
```

### Functions that DON'T resolve

Functions defined via dot-sourcing (`. file.ps1`) into a local scope will NOT be
found when invoked from a C# delegate. Options:
1. Export them from the module
2. Call them from the context setup code (before closures)
3. Store them as scriptblock references in the context hashtable

---

## 6. Crash Handling

WPF dispatcher exceptions silently kill the process. Add an unhandled exception handler:

```powershell
$crashLogPath = Join-Path -Path $PSScriptRoot -ChildPath '../../logs/gui_crash.log'
$window.Dispatcher.add_UnhandledException({
    param($s, $e)
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    # New-Item -Force is idempotent — no Test-Path guard needed
    $null = New-Item -Path (Split-Path -Path $crashLogPath -Parent) -ItemType Directory -Force -ErrorAction SilentlyContinue
    $msg = @(
        "[$timestamp] GUI CRASH"
        "Exception: $($e.Exception.GetType().FullName)"
        "Message: $($e.Exception.Message)"
        "StackTrace: $($e.Exception.StackTrace)"
    )
    if ($e.Exception.InnerException) {
        $msg += "InnerException: $($e.Exception.InnerException.Message)"
        $msg += "InnerStack: $($e.Exception.InnerException.StackTrace)"
    }
    $msg -join "`n" | Add-Content -LiteralPath $crashLogPath -Encoding utf8
    Write-Host "`nGUI CRASH — details in: $crashLogPath" -ForegroundColor Red
    $e.Handled = $true
}.GetNewClosure())
```

**Note:** `.GetNewClosure()` works here because the handler is invoked by the WPF
dispatcher on the same thread, not through a C# delegate constructor. The `$crashLogPath`
variable is a simple string captured by value.

---

## 7. Async Operations

WPF is single-threaded. Long operations must run off the UI thread.

### Pattern: Separate Runspace + DispatcherTimer

```powershell
$vm.RunCommand = [MyApp.GUI.RelayCommand]::new(
    [Action[object]]{
        $ctx = $global:AppCtx
        $ctx.VM.IsBusy = $true

        # Create a new PowerShell runspace for the work
        $ps = [System.Management.Automation.PowerShell]::Create()
        $null = $ps.AddScript({
            param($modulePath)
            Import-Module $modulePath -Force
            Invoke-LongOperation
        }).AddArgument($ctx.ModulePath)

        $asyncResult = $ps.BeginInvoke()

        # Store async state so the timer can access it
        $ctx.AsyncPS = $ps
        $ctx.AsyncResult = $asyncResult

        # Poll for completion on the UI thread
        $timer = [System.Windows.Threading.DispatcherTimer]::new()
        $timer.Interval = [TimeSpan]::FromMilliseconds(500)
        $ctx.PollTimer = $timer
        $timer.Add_Tick({
            $ctx = $global:AppCtx
            if (-not $ctx.AsyncResult.IsCompleted) { return }
            $ctx.PollTimer.Stop()
            try {
                $result = $ctx.AsyncPS.EndInvoke($ctx.AsyncResult)
                $ctx.AsyncPS.Dispose()
                # Process result...
                $ctx.VM.StatusMessage = 'Done'
            }
            catch {
                $ctx.VM.StatusMessage = "Error: $($_.Exception.Message)"
            }
            finally {
                $ctx.VM.IsBusy = $false
            }
        })
        $timer.Start()
    }
)
```

**Key points:**
- The separate runspace has its own module state — `Import-Module` again inside it
- `EndInvoke()` must be called on the UI thread (in the timer tick)
- Check `$ps.Streams.Error` for non-terminating errors from the runspace
- Store timer/async refs in the context hashtable for cleanup

### Progress Bar: Filtered Maximum

When processing a subset of items (e.g., user selected 2 of 7 folders, 5 skipped), the progress maximum must exclude non-actionable items. Otherwise the bar never reaches 100%:

```powershell
# WRONG: includes skipped items — bar reaches ~30% when done
$ctx.VM.ProgressMax = [double]$status.TotalSizeBytes

# RIGHT: only count items that will be processed
$actionableBytes = [long]($status.Items |
    Where-Object { $_.status -ne 'skipped' } |
    Measure-Object -Property sizeBytes -Sum).Sum
$ctx.VM.ProgressMax = [double]$actionableBytes
```

### Transfer Detail During Async Operations

Show what's currently being processed in a TextBlock below the progress bar. Determine the "in-progress" item by finding the first item still in its pre-operation state:

```powershell
# In the timer tick handler, after polling status:
$inProgressStatus = if ($op -eq 'Export') { 'pending' } else { 'exported' }
$completedCount = @($status.Items | Where-Object { $_.status -eq $doneStatus }).Count
$actionableCount = @($status.Items | Where-Object { $_.status -ne 'skipped' }).Count
$currentItem = $status.Items |
    Where-Object { $_.status -eq $inProgressStatus } | Select-Object -First 1

if ($currentItem) {
    $ctx.VM.TransferDetail = "${op}ing: $($currentItem.name) ($completedCount of $actionableCount)"
    # Optional: override DataGrid status for visual feedback
    $guiItem = $ctx.VM.Items | Where-Object { $_.Name -eq $currentItem.name }
    if ($guiItem) { $guiItem.Status = 'copying...' }
}
```

---

## 8. Dialogs

### Modal Child Window

```powershell
$dialog.Owner = $parentWindow    # Centers on parent
$dialog.DataContext = $dataItem  # Bind dialog fields to data
$result = $dialog.ShowDialog()   # Returns $true (OK), $false (Cancel), or $null
```

### File Dialogs (WPF built-in)

```powershell
# Open file
$dlg = [Microsoft.Win32.OpenFileDialog]::new()
$dlg.Title = 'Select File'
$dlg.Filter = 'All Files (*.*)|*.*'
if ($dlg.ShowDialog($ownerWindow)) { $path = $dlg.FileName }

# Save file
$dlg = [Microsoft.Win32.SaveFileDialog]::new()
if ($dlg.ShowDialog($ownerWindow)) { $path = $dlg.FileName }
```

### Folder Dialog (WinForms — WPF has no built-in folder picker)

```powershell
# Requires: Add-Type -AssemblyName System.Windows.Forms
$dlg = [System.Windows.Forms.FolderBrowserDialog]::new()
$dlg.Description = 'Select Folder'
try {
    if ($dlg.ShowDialog() -eq [System.Windows.Forms.DialogResult]::OK) {
        $path = $dlg.SelectedPath
    }
}
finally {
    $dlg.Dispose()   # Must be in finally — Dispose() is skipped if ShowDialog() throws
}
```

---

## 9. Gotchas Summary

| Issue | Symptom | Solution |
|-------|---------|----------|
| `.GetNewClosure()` + C# delegates | "Property not found on this object" at runtime | Use `$global:` context hashtable (Section 5) |
| `$script:` vars in command closures | Variables are `$null` when command executes | Copy to context hashtable |
| Dot-sourced functions in closures | "Term not recognized" when command executes | Export from module, or call directly (module-exported functions work) |
| Pester tests pass but GUI crashes | Tests run in same PS scope; C# delegates don't | Manual GUI testing required for command execution |
| `x:Class` in XAML | "Type not found" on XAML load | Strip with regex before loading |
| WPF single-threaded | UI freezes during long operations | Separate runspace + DispatcherTimer polling |
| Silent WPF crashes | Window closes with no error | Add `Dispatcher.UnhandledException` handler |
| `FolderBrowserDialog` not in WPF | No built-in folder picker | Load `System.Windows.Forms` assembly, use `FolderBrowserDialog` |
| `EndInvoke` result indexing | Single PSCustomObject treated as array | Use result directly, don't index with `[0]` |
| `ObservableCollection` not updating | Items added but UI doesn't refresh | Ensure property type is `ObservableCollection<T>`, not `List<T>` |
| Converter not found | `StaticResource` lookup fails at XAML load | Declare converter in `Window.Resources` before any usage; check `xmlns:local` namespace |
| `Closing` event can't cancel | Window closes despite `e.Cancel = $true` | Use `Closing` (not `Closed`) — only `Closing` has `CancelEventArgs` |
| Validation errors not visible | Red border doesn't appear | Add `ValidatesOnDataErrors=True` to binding; implement `IDataErrorInfo` on ViewModel |
| `$sender` in event handlers | PSScriptAnalyzer warning `PSAvoidAssignmentToAutomaticVariable` | Use `$s` instead: `param($s, $e)` in `add_PropertyChanged`, `add_Click`, etc. |
| Progress bar stuck at low % | ProgressMax includes skipped/excluded items | Filter to actionable items only when computing max (Section 7) |

---

## 10. ObservableCollection + DataGrid

Almost every WPF app needs to display and manage a list of items. `ObservableCollection<T>` automatically notifies the UI when items are added or removed — no manual refresh needed.

### C# Item Class

Define a class for each row in the grid. Use `ViewModelBase` so individual cell edits trigger UI updates.

```csharp
public class JobItem : ViewModelBase
{
    private string _source = "";
    private string _destination = "";
    private bool _enabled = true;

    public string Source
    {
        get => _source;
        set => SetProperty(ref _source, value, nameof(Source));
    }
    public string Destination
    {
        get => _destination;
        set => SetProperty(ref _destination, value, nameof(Destination));
    }
    public bool Enabled
    {
        get => _enabled;
        set => SetProperty(ref _enabled, value, nameof(Enabled));
    }
}
```

### ViewModel Property

```csharp
public class MainViewModel : ViewModelBase
{
    // ObservableCollection — UI auto-updates on Add/Remove
    public ObservableCollection<JobItem> Jobs { get; }
        = new ObservableCollection<JobItem>();

    // Selection tracking
    private JobItem _selectedJob;
    public JobItem SelectedJob
    {
        get => _selectedJob;
        set => SetProperty(ref _selectedJob, value, nameof(SelectedJob));
    }

    // Commands for add/remove
    public ICommand AddJobCommand { get; set; }
    public ICommand RemoveJobCommand { get; set; }
}
```

**Compilation note:** Add `System.ObjectModel` to `-ReferencedAssemblies` — that's where `ObservableCollection<T>` lives.

### XAML DataGrid

```xml
<DataGrid ItemsSource="{Binding Jobs}"
          SelectedItem="{Binding SelectedJob}"
          AutoGenerateColumns="False"
          CanUserAddRows="False"
          CanUserDeleteRows="False"
          SelectionMode="Single"
          Margin="0,0,0,8">
    <DataGrid.Columns>
        <DataGridCheckBoxColumn Header="Enabled"
            Binding="{Binding Enabled}" Width="60" />
        <DataGridTextColumn Header="Source"
            Binding="{Binding Source}" Width="*" />
        <DataGridTextColumn Header="Destination"
            Binding="{Binding Destination}" Width="*" />
    </DataGrid.Columns>
</DataGrid>
```

**Column types:**
- `DataGridTextColumn` — editable text
- `DataGridCheckBoxColumn` — boolean toggle
- `DataGridTemplateColumn` — custom content (buttons, dropdowns, etc.)

### PowerShell Manipulation

```powershell
# Add item
$job = [MyApp.GUI.JobItem]::new()
$job.Source = 'C:\Data'
$job.Destination = 'D:\Backup'
$vm.Jobs.Add($job)

# Remove selected
if ($vm.SelectedJob) {
    $vm.Jobs.Remove($vm.SelectedJob)
}

# Clear all
$vm.Jobs.Clear()

# Iterate
foreach ($job in $vm.Jobs) {
    Write-Host "$($job.Source) -> $($job.Destination)"
}
```

### Subscribing to PropertyChanged from PowerShell

When a DataGrid has editable cells (checkboxes, text), you often need to react when the user changes a value — e.g., recalculate a summary. Subscribe to `PropertyChanged` on each item after creation:

```powershell
$item = [MyApp.GUI.ScanItem]::new()
$item.Name = 'Documents'
$item.Included = $true

# React to checkbox toggle (or any property change)
$item.add_PropertyChanged({
    param($s, $e)  # NOTE: use $s, NOT $sender — $sender is a PS automatic variable
    if ($e.PropertyName -eq 'Included') {
        & $global:AppCtx.UpdateSummary
    }
})

$vm.Items.Add($item)
```

**Key points:**
- Use `$s` (not `$sender`) to avoid PSScriptAnalyzer `PSAvoidAssignmentToAutomaticVariable` warning
- Access shared state via `$global:AppCtx` — same pattern as command wiring (Section 5)
- If items are created in multiple places, put the subscription in a factory scriptblock on the context to ensure consistency

---

## 11. Value Converters

Converters transform data between the ViewModel and the UI. The most common: `bool → Visibility`.

### C# Converter Implementation

```csharp
public class BooleanToVisibilityConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter,
        System.Globalization.CultureInfo culture)
    {
        bool flag = (bool)value;
        // Optional: invert if parameter is "Invert"
        if (parameter is string s && s == "Invert") flag = !flag;
        return flag ? Visibility.Visible : Visibility.Collapsed;
    }

    public object ConvertBack(object value, Type targetType, object parameter,
        System.Globalization.CultureInfo culture)
    {
        return (Visibility)value == Visibility.Visible;
    }
}
```

**Compilation:** Include this in the same `Add-Type -TypeDefinition` block as your ViewModel classes. Add `System.Xaml` to `-ReferencedAssemblies` if needed.

### XAML Declaration and Usage

```xml
<Window xmlns:local="clr-namespace:MyApp.GUI"
        ... >
    <Window.Resources>
        <local:BooleanToVisibilityConverter x:Key="BoolToVis" />
    </Window.Resources>

    <!-- Usage: hide/show based on a bool property -->
    <ProgressBar Visibility="{Binding IsBusy,
        Converter={StaticResource BoolToVis}}" />

    <!-- Inverted: visible when NOT busy -->
    <Button Content="Run"
        Visibility="{Binding IsBusy,
        Converter={StaticResource BoolToVis},
        ConverterParameter=Invert}" />
</Window>
```

### xmlns Namespace for Runtime-Compiled Types

When types are compiled via `Add-Type` (no assembly name on disk), the `xmlns:local` declaration uses the default assembly:

```xml
<!-- For types compiled with Add-Type (no explicit assembly name): -->
xmlns:local="clr-namespace:MyApp.GUI"

<!-- If you specified -OutputAssembly during Add-Type: -->
xmlns:local="clr-namespace:MyApp.GUI;assembly=MyAppGui"
```

**Gotcha:** The `xmlns:local` namespace must be declared on the root element (`<Window>`) — not deeper in the tree.

---

## 12. Styles and Theming Basics

Styles reduce repetition by applying property values to all instances of a control type.

### Implicit Styles (Apply to All Instances)

```xml
<Window.Resources>
    <!-- Every Button in this window gets these properties -->
    <Style TargetType="Button">
        <Setter Property="Margin" Value="4" />
        <Setter Property="Padding" Value="8,4" />
        <Setter Property="MinWidth" Value="80" />
    </Style>

    <!-- Every TextBox -->
    <Style TargetType="TextBox">
        <Setter Property="Margin" Value="0,2" />
        <Setter Property="Padding" Value="2" />
    </Style>
</Window.Resources>
```

### Keyed Styles (Apply Selectively)

```xml
<Window.Resources>
    <Style x:Key="PrimaryButton" TargetType="Button">
        <Setter Property="Background" Value="#0078D7" />
        <Setter Property="Foreground" Value="White" />
        <Setter Property="FontWeight" Value="Bold" />
        <Setter Property="Padding" Value="12,6" />
    </Style>

    <Style x:Key="DangerButton" TargetType="Button">
        <Setter Property="Background" Value="#D32F2F" />
        <Setter Property="Foreground" Value="White" />
    </Style>
</Window.Resources>

<!-- Usage -->
<Button Content="Save" Style="{StaticResource PrimaryButton}" />
<Button Content="Delete" Style="{StaticResource DangerButton}" />
```

### Style Inheritance (BasedOn)

```xml
<Style x:Key="BaseButton" TargetType="Button">
    <Setter Property="Margin" Value="4" />
    <Setter Property="Padding" Value="8,4" />
</Style>

<Style x:Key="PrimaryButton" TargetType="Button" BasedOn="{StaticResource BaseButton}">
    <Setter Property="Background" Value="#0078D7" />
    <Setter Property="Foreground" Value="White" />
</Style>
```

### Dark Theme Example

```xml
<Window.Resources>
    <!-- Dark background for the whole window -->
    <SolidColorBrush x:Key="BackgroundBrush" Color="#1E1E1E" />
    <SolidColorBrush x:Key="ForegroundBrush" Color="#D4D4D4" />
    <SolidColorBrush x:Key="AccentBrush" Color="#569CD6" />

    <Style TargetType="Window">
        <Setter Property="Background" Value="{StaticResource BackgroundBrush}" />
    </Style>
    <Style TargetType="TextBlock">
        <Setter Property="Foreground" Value="{StaticResource ForegroundBrush}" />
    </Style>
</Window.Resources>
```

**Key points:**
- Implicit styles (no `x:Key`) apply to ALL instances of that type within the scope
- Keyed styles must be explicitly referenced with `Style="{StaticResource KeyName}"`
- `Window.Resources` applies to the whole window. `StackPanel.Resources` applies only within that panel.
- `BasedOn` lets you build style hierarchies without repeating common setters

---

## 13. Window Lifecycle Events

Understanding when each event fires prevents "why does this run at the wrong time" bugs.

### Event Order

```
Constructor → Loaded → ContentRendered → ... user interaction ... → Closing → Closed
```

### Loaded — Window Is Rendered

```powershell
$window.Add_Loaded({
    param($sender, $e)
    # Safe to:
    #   - Set initial focus
    #   - Start animations
    #   - Load initial data
    #   - Access control dimensions (now known)
    $window.FindName('SearchBox').Focus()
})
```

### ContentRendered — All Content Drawn

```powershell
$window.Add_ContentRendered({
    param($sender, $e)
    # All visual elements are rendered
    # Good for post-render tasks like splash screen dismissal
})
```

### Closing — Can Be Cancelled

```powershell
$window.Add_Closing({
    param($sender, $e)
    # $e is [System.ComponentModel.CancelEventArgs]
    # Set $e.Cancel = $true to PREVENT the window from closing

    $ctx = $global:AppCtx
    if ($ctx.VM.HasUnsavedChanges) {
        $result = [System.Windows.MessageBox]::Show(
            'You have unsaved changes. Close anyway?',
            'Confirm',
            [System.Windows.MessageBoxButton]::YesNo,
            [System.Windows.MessageBoxImage]::Warning
        )
        if ($result -ne [System.Windows.MessageBoxResult]::Yes) {
            $e.Cancel = $true   # Prevent close
        }
    }
})
```

### Closed — Cleanup (Cannot Cancel)

```powershell
$window.Add_Closed({
    param($sender, $e)
    # Window is closed — cannot cancel at this point
    # Use for cleanup:
    $ctx = $global:AppCtx
    if ($ctx.PollTimer) { $ctx.PollTimer.Stop() }
    if ($ctx.AsyncPS) { $ctx.AsyncPS.Dispose() }
    $global:AppCtx = $null   # Clean up global state
})
```

### Key Distinctions

| Event | CancelEventArgs | When | Use For |
|-------|----------------|------|---------|
| `Loaded` | No | After render, before user input | Initial data load, focus |
| `ContentRendered` | No | After all content drawn | Post-render tasks |
| `Closing` | Yes (`$e.Cancel`) | Before close starts | "Save changes?" prompt |
| `Closed` | No | After close completes | Timer/resource cleanup |

**Gotcha from PowerShell:** `.GetNewClosure()` works for lifecycle events because they're invoked by the WPF dispatcher on the same thread — not through C# delegate constructors. But for safety and consistency, use the `$global:` context pattern anyway.

---

## 14. Input Validation (IDataErrorInfo)

WPF has built-in support for showing validation errors (red borders, tooltips) tied to ViewModel properties.

### C# ViewModel with IDataErrorInfo

```csharp
public class MainViewModel : ViewModelBase, IDataErrorInfo
{
    private string _smtpServer = "";
    private int _smtpPort = 587;

    public string SmtpServer
    {
        get => _smtpServer;
        set => SetProperty(ref _smtpServer, value, nameof(SmtpServer));
    }
    public int SmtpPort
    {
        get => _smtpPort;
        set => SetProperty(ref _smtpPort, value, nameof(SmtpPort));
    }

    // IDataErrorInfo implementation
    public string Error => null;   // Not used by WPF binding

    public string this[string propertyName]
    {
        get
        {
            switch (propertyName)
            {
                case nameof(SmtpServer):
                    if (string.IsNullOrWhiteSpace(SmtpServer))
                        return "SMTP server is required";
                    break;
                case nameof(SmtpPort):
                    if (SmtpPort < 1 || SmtpPort > 65535)
                        return "Port must be between 1 and 65535";
                    break;
            }
            return null;   // No error
        }
    }
}
```

### XAML Binding with Validation

```xml
<!-- Enable validation on the binding -->
<TextBox Text="{Binding SmtpServer,
    UpdateSourceTrigger=PropertyChanged,
    ValidatesOnDataErrors=True,
    NotifyOnValidationError=True}" />
```

WPF automatically shows a **red border** around the TextBox when the indexer returns a non-null string. To add a tooltip:

```xml
<Style TargetType="TextBox">
    <Style.Triggers>
        <Trigger Property="Validation.HasError" Value="True">
            <Setter Property="ToolTip"
                Value="{Binding RelativeSource={RelativeSource Self},
                    Path=(Validation.Errors)[0].ErrorContent}" />
        </Trigger>
    </Style.Triggers>
</Style>
```

### Custom Error Template

Replace the default red border with a custom indicator:

```xml
<ControlTemplate x:Key="ValidationErrorTemplate">
    <StackPanel>
        <AdornedElementPlaceholder />
        <TextBlock Text="{Binding [0].ErrorContent}"
                   Foreground="Red" FontSize="10" Margin="2,0,0,0" />
    </StackPanel>
</ControlTemplate>

<TextBox Validation.ErrorTemplate="{StaticResource ValidationErrorTemplate}"
         Text="{Binding SmtpServer, ValidatesOnDataErrors=True}" />
```

### Checking Validity from PowerShell

```powershell
# Check if a specific property has errors
$error = $vm['SmtpServer']   # Calls the IDataErrorInfo indexer
if ($error) {
    Write-Host "Validation error: $error"
}

# Disable Save button when there are errors (via CanExecute)
$vm.SaveCommand = [MyApp.GUI.RelayCommand]::new(
    [Action[object]]{
        $ctx = $global:AppCtx
        # Save logic...
    },
    [Func[object, bool]]{
        $ctx = $global:AppCtx
        # Enable only when no validation errors
        return [string]::IsNullOrEmpty($ctx.VM['SmtpServer']) -and
               [string]::IsNullOrEmpty($ctx.VM['SmtpPort'])
    }
)
```

**Key points:**
- `IDataErrorInfo` is simpler than `INotifyDataErrorInfo` and sufficient for most forms
- WPF calls the indexer automatically when `ValidatesOnDataErrors=True` is set on the binding
- The red border is WPF's default `ErrorTemplate` — no extra XAML needed for basic validation
- For the `Func<object, bool>` CanExecute to re-evaluate, call `RaiseCanExecuteChanged()` after property changes

---

## 15. Verified Patterns from Implementation

### CLR namespace for Add-Type assemblies in XAML [verified in Phase 3]
`Add-Type` creates a dynamic assembly whose name is not predictable. To reference compiled types in XAML (e.g., for `StaticResource` converters), resolve the assembly name at runtime and inject it via placeholder replacement:
```powershell
$guiAssembly = ([MyApp.GUI.SomeType]).Assembly.GetName().Name
$xaml = (Get-XamlString).Replace('__GUI_ASSEMBLY__', $guiAssembly)
# In XAML: xmlns:local="clr-namespace:MyApp.GUI;assembly=__GUI_ASSEMBLY__"
```
Do NOT use `DynamicResource` for `Binding.Converter` — `Converter` is not a DependencyProperty. Do NOT add converters to `$window.Resources` from code-behind and expect `StaticResource` to resolve them — the XAML template is parsed before code-behind runs.

### DataGridCheckBoxColumn requires double-click [verified in Phase 3]
Default `DataGridCheckBoxColumn` requires click-to-select-row then click-to-toggle. Use `DataGridTemplateColumn` with a `CheckBox` for single-click behavior:
```xml
<DataGridTemplateColumn Header="" Width="35">
    <DataGridTemplateColumn.CellTemplate>
        <DataTemplate>
            <CheckBox IsChecked="{Binding MyProp, UpdateSourceTrigger=PropertyChanged}"
                      HorizontalAlignment="Center" VerticalAlignment="Center" />
        </DataTemplate>
    </DataGridTemplateColumn.CellTemplate>
</DataGridTemplateColumn>
```

### Deferred view refresh after property changes [verified in Phase 3]
Calling `CollectionView.Refresh()` inside a `PropertyChanged` handler that fires during DataGrid click processing causes visual glitches. Defer to next dispatcher frame:
```powershell
$ctx.Window.Dispatcher.BeginInvoke([Action]{
    $v = [System.Windows.Data.CollectionViewSource]::GetDefaultView($items)
    if ($null -ne $v) { $v.Refresh() }
})
```

### DataGrid text color on dark themes [verified in Phase 3]
Setting `Foreground` on the DataGrid does not propagate to cell content. Must set explicit `CellStyle` and `RowStyle` with `Foreground` setters, including overrides for `IsSelected` trigger to prevent WPF reverting to system theme colors.
