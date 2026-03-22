# PowerShell CIM/WMI Data Collection

Patterns for querying system information via CIM cmdlets in PowerShell 7+.
Validated against Windows 10 Pro during System Inventory Tool development.

CIM cmdlets (`Get-CimInstance`) are the modern standard. `Get-WmiObject` is deprecated in
PowerShell 7 and removed in future versions. All patterns here use CIM exclusively.

---

## Table of Contents

1. [CIM vs WMI — What Changed](#1-cim-vs-wmi--what-changed)
2. [Core Query Pattern](#2-core-query-pattern)
3. [Essential Hardware Classes](#3-essential-hardware-classes)
4. [OS and System Classes](#4-os-and-system-classes)
5. [Network Classes](#5-network-classes)
6. [Storage Classes](#6-storage-classes)
7. [Services](#7-services)
8. [Installed Software — The Right Way](#8-installed-software--the-right-way)
9. [Windows Update History](#9-windows-update-history)
10. [Wi-Fi Profiles](#10-wi-fi-profiles)
11. [SMART and Disk Health](#11-smart-and-disk-health)
12. [Data Type Conversions](#12-data-type-conversions)
13. [Error Handling for CIM Queries](#13-error-handling-for-cim-queries)
14. [Performance Patterns](#14-performance-patterns)
15. [Gotchas](#15-gotchas)
16. [Checklist](#16-checklist)

---

## 1. CIM vs WMI — What Changed

| Aspect | Old (WMI) | New (CIM) |
|--------|-----------|-----------|
| Cmdlet | `Get-WmiObject` | `Get-CimInstance` |
| Protocol | DCOM (multiple ports) | WSMan (port 5985) |
| PS 7 status | **Removed** | Supported |
| Performance | Slower | Faster |
| Session reuse | No | Yes (New-CimSession) |
| Remoting | `-ComputerName` | `New-CimSession` |

**Rule:** Never use `Get-WmiObject`. Never use `-ComputerName` on CIM cmdlets for local queries — it adds unnecessary overhead.

---

## 2. Core Query Pattern

```powershell
# Basic query
$cpu = Get-CimInstance -ClassName Win32_Processor

# Filtered query (filter at source, not pipeline — faster)
$fixedDisks = Get-CimInstance -ClassName Win32_LogicalDisk -Filter "DriveType=3"

# Select specific properties (reduces data transferred)
$os = Get-CimInstance -ClassName Win32_OperatingSystem |
    Select-Object Caption, Version, BuildNumber, LastBootUpTime

# Query non-default namespace
$reliability = Get-CimInstance -Namespace root\Microsoft\Windows\Storage `
    -ClassName MSFT_StorageReliabilityCounter

# Error-safe wrapper (see Section 13)
function Invoke-CimQuery {
    param([string]$ClassName, [string]$Filter, [string]$Namespace = 'root\cimv2')
    try {
        $params = @{ ClassName = $ClassName; Namespace = $Namespace }
        if ($Filter) { $params.Filter = $Filter }
        Get-CimInstance @params
    }
    catch {
        Write-Warning "CIM query failed for ${ClassName}: $($_.Exception.Message)"
        $null
    }
}
```

---

## 3. Essential Hardware Classes

### CPU — Win32_Processor

```powershell
$cpu = Get-CimInstance -ClassName Win32_Processor | Select-Object `
    Name,           # "Intel(R) Core(TM) i7-10750H CPU @ 2.60GHz"
    NumberOfCores,
    NumberOfLogicalProcessors,
    MaxClockSpeed,  # MHz
    CurrentClockSpeed,
    L2CacheSize,    # KB
    L3CacheSize,    # KB
    Manufacturer,
    Architecture,   # 9 = x64
    SocketDesignation
```

Architecture decode: `0`=x86, `5`=ARM, `9`=x64, `12`=ARM64.

### RAM — Win32_PhysicalMemory

```powershell
$ram = Get-CimInstance -ClassName Win32_PhysicalMemory | Select-Object `
    BankLabel,
    DeviceLocator,  # Slot identifier, e.g. "ChannelA-DIMM0"
    Capacity,       # Bytes — divide by 1GB for display
    Speed,          # MHz (configured speed)
    ConfiguredClockSpeed,
    Manufacturer,
    PartNumber,
    SMBIOSMemoryType  # More reliable than MemoryType for DDR generation

# Total RAM (faster than summing DIMM capacities)
$totalRAM = (Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory
```

**SMBIOSMemoryType decode:**

| Code | Type |
|------|------|
| 20   | DDR  |
| 21   | DDR2 |
| 24   | DDR3 |
| 26   | DDR4 |
| 34   | DDR5 |

```powershell
function ConvertTo-RAMType {
    param([int]$Code)
    @{ 20='DDR'; 21='DDR2'; 24='DDR3'; 26='DDR4'; 34='DDR5' }[$Code] ?? "Unknown ($Code)"
}
```

### Disk Drives — Win32_DiskDrive

```powershell
$disks = Get-CimInstance -ClassName Win32_DiskDrive | Select-Object `
    Model,
    Size,           # Bytes
    MediaType,      # "Fixed hard disk media" — unreliable for SSD detection
    InterfaceType,  # "SCSI", "IDE", "USB", "SATA"
    SerialNumber,
    FirmwareRevision,
    Partitions,
    Status          # "OK", "Degraded", "Unknown"

# Use Get-PhysicalDisk for SSD vs HDD detection (more reliable)
$physicalDisks = Get-PhysicalDisk | Select-Object `
    FriendlyName, MediaType, BusType, Size, HealthStatus, OperationalStatus
# MediaType: SSD, HDD, SCM, Unspecified
```

### GPU — Win32_VideoController

```powershell
$gpu = Get-CimInstance -ClassName Win32_VideoController | Select-Object `
    Name,
    AdapterRAM,         # Bytes — may report 0 for some modern GPUs via WDDM
    DriverVersion,
    DriverDate,
    VideoProcessor,
    CurrentHorizontalResolution,
    CurrentVerticalResolution,
    CurrentRefreshRate
```

**Note:** `AdapterRAM` is unreliable for discrete GPUs with >4GB VRAM due to 32-bit integer overflow. Report as-is or flag when value = 4294967295 (UINT32_MAX).

### Motherboard and BIOS

```powershell
# Motherboard
$board = Get-CimInstance -ClassName Win32_BaseBoard | Select-Object `
    Manufacturer, Product, Version, SerialNumber

# BIOS
$bios = Get-CimInstance -ClassName Win32_BIOS | Select-Object `
    Manufacturer, Name, Version, SMBIOSBIOSVersion, ReleaseDate

# System (make/model for laptops/branded desktops)
$system = Get-CimInstance -ClassName Win32_ComputerSystem | Select-Object `
    Manufacturer, Model, SystemType, TotalPhysicalMemory, NumberOfProcessors
```

---

## 4. OS and System Classes

```powershell
$os = Get-CimInstance -ClassName Win32_OperatingSystem | Select-Object `
    Caption,           # "Microsoft Windows 10 Pro"
    Version,           # "10.0.19045"
    BuildNumber,
    OSArchitecture,    # "64-bit"
    InstallDate,
    LastBootUpTime,
    LocalDateTime,
    RegisteredUser,
    SerialNumber,      # OS serial (not hardware)
    TotalVisibleMemorySize,   # KB
    FreePhysicalMemory        # KB

# Uptime calculation
$uptime = (Get-Date) - $os.LastBootUpTime
# $uptime is a TimeSpan — format as needed

# Windows license status
$license = Get-CimInstance -ClassName SoftwareLicensingProduct `
    -Filter "PartialProductKey IS NOT NULL AND ApplicationId = '55c92734-d682-4d71-983e-d6ec3f16059f'" |
    Select-Object Name, LicenseStatus

# LicenseStatus decode:
# 0 = Unlicensed
# 1 = Licensed (activated)
# 2 = OOBGrace    (out-of-box grace period)
# 3 = OOTGrace    (out-of-tolerance grace period)
# 4 = NonGenuineGrace (non-genuine grace period)
# 5 = Notification (activation required, limited functionality)
# 6 = ExtendedGrace
```

---

## 5. Network Classes

```powershell
# Physical adapters only (excludes virtual/loopback)
$adapters = Get-CimInstance -ClassName Win32_NetworkAdapter `
    -Filter "PhysicalAdapter=True" | Select-Object `
    Name, MACAddress, Speed, NetConnectionID, NetEnabled, AdapterType

# IP configuration — join on Index
$configs = Get-CimInstance -ClassName Win32_NetworkAdapterConfiguration `
    -Filter "IPEnabled=True" | Select-Object `
    Index, IPAddress, IPSubnet, DefaultIPGateway, DNSServerSearchOrder, DHCPEnabled, DHCPServer

# Join adapters to configs
$adapterIndex = @{}
$adapters | ForEach-Object { $adapterIndex[$_.DeviceID] = $_ }

$networkInfo = $configs | ForEach-Object {
    $adapter = $adapterIndex[$_.Index]
    [PSCustomObject]@{
        Name        = $adapter.Name
        MAC         = $adapter.MACAddress
        IPAddresses = $_.IPAddress -join ', '
        Gateway     = $_.DefaultIPGateway -join ', '
        DNS         = $_.DNSServerSearchOrder -join ', '
        DHCP        = $_.DHCPEnabled
    }
}
```

**Note:** `Win32_NetworkAdapter.Speed` is reported in bits/sec. Divide by 1e9 for Gbps. Value may be $null for disconnected adapters.

---

## 6. Storage Classes

```powershell
# Logical volumes (drive letters)
$volumes = Get-CimInstance -ClassName Win32_LogicalDisk -Filter "DriveType=3" |
    Select-Object DeviceID, VolumeName, FileSystem, Size, FreeSpace
# DriveType 3 = fixed local disk; 2 = removable; 4 = network; 5 = CD-ROM

# Alternatively: Get-Volume (Storage module, more properties)
$volumes = Get-Volume | Where-Object { $_.DriveType -eq 'Fixed' } |
    Select-Object DriveLetter, FileSystemLabel, FileSystem, Size, SizeRemaining, HealthStatus
```

---

## 7. Services

```powershell
# All services
$services = Get-CimInstance -ClassName Win32_Service | Select-Object `
    Name, DisplayName, State, StartMode, PathName, Description

# Running services only
$running = Get-CimInstance -ClassName Win32_Service -Filter "State='Running'"

# Services that should be running but are stopped (common audit pattern)
$stoppedAuto = Get-CimInstance -ClassName Win32_Service `
    -Filter "StartMode='Auto' AND State!='Running'" |
    Select-Object Name, DisplayName, State, StartMode

# StartMode values: Auto, Manual, Disabled
# State values: Running, Stopped, Paused, Start Pending, Stop Pending
```

---

## 8. Installed Software — The Right Way

**NEVER use `Win32_Product`.** It triggers MSI consistency checks on every installed application,
causing silent repairs and taking 60+ seconds on a typical machine.

```powershell
# Fast registry-based approach (~2 seconds)
function Get-InstalledSoftware {
    $paths = @(
        'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
        'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*'
    )
    $paths | ForEach-Object {
        Get-ItemProperty -Path $_ -ErrorAction SilentlyContinue |
            Where-Object { $_.DisplayName } |
            Select-Object `
                @{N='Name';        E={$_.DisplayName}},
                @{N='Version';     E={$_.DisplayVersion}},
                @{N='Publisher';   E={$_.Publisher}},
                @{N='InstallDate'; E={$_.InstallDate}},
                @{N='InstallLocation'; E={$_.InstallLocation}}
    } | Sort-Object Name -Unique
}
```

**Why two paths:** The `WOW6432Node` path contains 32-bit apps installed on 64-bit Windows. Both paths may contain the same app with different GUIDs — `Sort-Object Name -Unique` collapses duplicates.

---

## 9. Windows Update History

`Win32_QuickFixEngineering` only returns CBS-delivered patches — not MSI or Windows Update Agent
delivered patches. For full history, use the WUA COM object.

```powershell
# Quick — CBS patches only (KBs applied via DISM/CBS)
$hotfixes = Get-CimInstance -ClassName Win32_QuickFixEngineering |
    Select-Object HotFixID, Description, InstalledOn, InstalledBy |
    Sort-Object InstalledOn -Descending

# Full history — Windows Update Agent COM object (slower, comprehensive)
function Get-WindowsUpdateHistory {
    param([int]$Count = 20)
    $session   = New-Object -ComObject Microsoft.Update.Session
    $searcher  = $session.CreateUpdateSearcher()
    $totalCount = $searcher.GetTotalHistoryCount()
    $history   = $searcher.QueryHistory(0, [Math]::Min($Count, $totalCount))
    $history | Where-Object { $_.ResultCode -eq 2 } |  # 2 = Succeeded
        Select-Object `
            @{N='Title';       E={$_.Title}},
            @{N='Date';        E={$_.Date}},
            @{N='KB';          E={if ($_.Title -match 'KB\d+') { $Matches[0] } else { '' }}}
}
```

**Use WUA COM for the report** — it's the most complete and is what Windows Update itself uses.

---

## 10. Wi-Fi Profiles

CIM has no class for Wi-Fi profiles. Use `netsh` — it's available on all Windows systems and does
not expose passwords when run without admin (names only, which is all we want).

```powershell
function Get-WiFiProfiles {
    $output = netsh wlan show profiles 2>$null
    if ($LASTEXITCODE -ne 0 -or -not $output) { return @() }
    $output | Where-Object { $_ -match 'All User Profile\s*:\s*(.+)' } |
        ForEach-Object { $Matches[1].Trim() }
}
```

**Do NOT run `netsh wlan show profile name="X" key=clear`** — that exposes the password. Profile names only.

---

## 11. SMART and Disk Health

```powershell
# Requires Storage module (available on Windows 8+ / Server 2012+)
# Returns health counters per physical disk
$health = Get-PhysicalDisk | Get-StorageReliabilityCounter |
    Select-Object `
        DeviceId,
        Temperature,       # Celsius (0 if not reported)
        TemperatureMax,
        Wear,              # 0-100, SSD wear indicator (0 = unknown)
        ReadErrorsTotal,
        ReadErrorsCorrected,
        WriteErrorsTotal,
        WriteErrorsUncorrected,
        PowerOnHours

# Match to physical disks for friendly names
$diskHealth = Get-PhysicalDisk | ForEach-Object {
    $rc = $_ | Get-StorageReliabilityCounter
    [PSCustomObject]@{
        Name        = $_.FriendlyName
        Health      = $_.HealthStatus    # Healthy, Warning, Unhealthy
        MediaType   = $_.MediaType       # SSD, HDD
        Temp        = $rc.Temperature
        Wear        = $rc.Wear
        ReadErrors  = $rc.ReadErrorsTotal
        WriteErrors = $rc.WriteErrorsTotal
        PowerOnHours = $rc.PowerOnHours
    }
}
```

**Note:** Temperature and Wear are 0 if the drive doesn't report them (many consumer drives). Check before displaying.

---

## 12. Data Type Conversions

```powershell
# Bytes to human-readable
function ConvertTo-FriendlySize {
    param([long]$Bytes)
    switch ($Bytes) {
        { $_ -ge 1TB } { '{0:N2} TB' -f ($_ / 1TB); break }
        { $_ -ge 1GB } { '{0:N2} GB' -f ($_ / 1GB); break }
        { $_ -ge 1MB } { '{0:N2} MB' -f ($_ / 1MB); break }
        { $_ -ge 1KB } { '{0:N2} KB' -f ($_ / 1KB); break }
        default         { "$_ B" }
    }
}

# KB to friendly (Win32_OperatingSystem returns KB)
function ConvertTo-FriendlySizeKB {
    param([long]$KB)
    ConvertTo-FriendlySize -Bytes ($KB * 1KB)
}

# Uptime from LastBootUpTime
function ConvertTo-FriendlyUptime {
    param([datetime]$BootTime)
    $uptime = (Get-Date) - $BootTime
    '{0}d {1}h {2}m' -f $uptime.Days, $uptime.Hours, $uptime.Minutes
}

# MHz to GHz
function ConvertTo-GHz {
    param([int]$MHz)
    '{0:N2} GHz' -f ($MHz / 1000)
}

# Network speed (bits/sec to human-readable)
function ConvertTo-FriendlySpeed {
    param([long]$Bps)
    switch ($Bps) {
        { $_ -ge 1e9 }  { '{0:N0} Gbps' -f ($_ / 1e9); break }
        { $_ -ge 1e6 }  { '{0:N0} Mbps' -f ($_ / 1e6); break }
        { $_ -ge 1e3 }  { '{0:N0} Kbps' -f ($_ / 1e3); break }
        default          { "$_ bps" }
    }
}
```

---

## 13. Error Handling for CIM Queries

CIM queries can fail for several reasons: class not available on this Windows version, access
denied (requires elevation), namespace not found. Always wrap in try/catch and return $null or
an empty collection — never let a failed section crash the whole report.

```powershell
function Invoke-CimQuery {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string]$ClassName,
        [string]$Filter,
        [string]$Namespace = 'root\cimv2',
        [string[]]$Property
    )
    try {
        $params = @{
            ClassName = $ClassName
            Namespace = $Namespace
            ErrorAction = 'Stop'
        }
        if ($Filter)   { $params.Filter   = $Filter }
        if ($Property) { $params.Property = $Property }
        Get-CimInstance @params
    }
    catch [Microsoft.Management.Infrastructure.CimException] {
        Write-Warning "CIM class not available: ${ClassName} — $($_.Exception.Message)"
        $null
    }
    catch {
        Write-Warning "CIM query error for ${ClassName}: $($_.Exception.Message)"
        $null
    }
}

# Per-section try/catch in collector functions
function Get-SIHardware {
    $result = [ordered]@{}
    try {
        $result.CPU = Get-CimInstance Win32_Processor -ErrorAction Stop
    }
    catch {
        $result.CPU = $null
        Write-Warning "CPU query failed: $_"
    }
    # ... continue with other subsections
    $result
}
```

---

## 14. Performance Patterns

```powershell
# Filter at source — NOT in the pipeline
# SLOW:
Get-CimInstance Win32_Service | Where-Object { $_.State -eq 'Running' }
# FAST:
Get-CimInstance Win32_Service -Filter "State='Running'"

# Select only needed properties at source
Get-CimInstance Win32_Process -Property Name, ProcessId, WorkingSetSize

# Avoid Win32_Product — always use registry for installed software (Section 8)

# For multiple queries, reuse a CimSession (only needed for remote — local queries are fine without)
# Local queries don't need session reuse; each Get-CimInstance is fast

# Parallel collection for independent sections (PS 7+ ForEach-Object -Parallel)
$results = @{
    Hardware = $null
    OS       = $null
    Network  = $null
}
$results.Hardware, $results.OS, $results.Network = @(
    { Get-SIHardware },
    { Get-SIOS },
    { Get-SINetwork }
) | ForEach-Object -Parallel { & $_ } -ThrottleLimit 4
```

**Note:** `-Parallel` creates separate runspaces — module functions must be dot-sourced or re-imported inside the parallel block if using module-scoped state.

---

## 15. Gotchas

| Issue | Symptom | Fix |
|-------|---------|-----|
| `Win32_Product` triggered MSI repair | Silent, takes 60+ sec, may pop UAC | Never use it. Registry approach (Section 8). |
| `AdapterRAM` shows 4GB for 8GB GPU | UINT32 overflow on modern discrete GPUs | Report as-is; note limitation in output |
| `SMBIOSMemoryType` vs `MemoryType` | `MemoryType` often returns 0 on modern systems | Use `SMBIOSMemoryType` |
| `Win32_NetworkAdapterConfiguration` IPEnabled filter | Without filter, returns many null-IP entries | Always use `-Filter "IPEnabled=True"` |
| `Win32_QuickFixEngineering` misses patches | Only CBS patches; MSI/WU patches not included | Use WUA COM object for full history |
| `Temperature` is 0 in `Get-StorageReliabilityCounter` | Drive doesn't report SMART temperature | Check `$rc.Temperature -gt 0` before displaying |
| `Get-PhysicalDisk` / `Get-Volume` require Storage module | Not available on stripped-down Windows SKUs | Wrap in try/catch; fall back to `Win32_DiskDrive` |
| `PhysicalAdapter=True` filter | Some USB NICs may not set this flag | Test on target hardware; manual QA required |
| CIM filter uses WQL not PS syntax | `$_.State -eq 'Running'` fails in -Filter | Use WQL: `State='Running'` (no $) |
| `netsh wlan` fails on machines with no WLAN adapter | Returns non-zero exit code | Check `$LASTEXITCODE` after call |
| Parallel runspaces lose module context | Module functions undefined inside -Parallel block | Re-import module or use `$using:` scope |

---

## 16. Checklist

- [ ] Using `Get-CimInstance` everywhere — no `Get-WmiObject`
- [ ] Installed software uses registry approach — no `Win32_Product`
- [ ] All CIM queries wrapped in try/catch; failed sections return $null gracefully
- [ ] Filtering at source (`-Filter`) not in pipeline for performance-sensitive queries
- [ ] RAM type uses `SMBIOSMemoryType` not `MemoryType`
- [ ] GPU VRAM overflow condition (UINT32_MAX) handled
- [ ] Network adapter filter `PhysicalAdapter=True` applied
- [ ] Wi-Fi profile collection uses `netsh` (names only — no passwords)
- [ ] Windows Update history uses WUA COM object for full coverage
- [ ] SMART data temperature/wear checked before display (may be 0)
- [ ] All byte values converted to human-friendly units before output
- [ ] Uptime expressed as days/hours/minutes, not raw seconds

---

## Verified Patterns from Implementation

### Win32_Battery for laptop detection [verified in Phase 4]
`Get-CimInstance Win32_Battery` returns non-null when a battery exists. Reliable on Dell hardware for laptop/desktop detection. Some desktops with UPS may report a phantom battery — the original Disable-PowerSaving script prompted for confirmation, but for automated provisioning auto-detection is sufficient.

### NIC PnPCapabilities for power management [verified in Phase 4]
Registry path: `HKLM:\SYSTEM\CurrentControlSet\Enum\<InstanceId>`, value `PnPCapabilities` (DWord). Value 24 disables power management. Bit mask `0x10` (16) indicates already disabled. Resolve InstanceId via `Get-PnpDevice -Class Net` matched by `FriendlyName` to `Get-NetAdapter -Physical`'s `InterfaceDescription`.
