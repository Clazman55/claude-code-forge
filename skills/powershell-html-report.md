# PowerShell HTML Report Generation

Patterns for generating self-contained HTML reports from PowerShell data.
Validated against System Inventory Tool development on Windows 10 Pro / PS 7+.

The goal is a single portable HTML file with embedded CSS and JS — no external dependencies,
opens in any browser, files neatly on a client share or email. No `ConvertTo-Html` for final
output — direct string construction gives full control over layout and styling.

---

## Table of Contents

1. [Architecture: Build Then Write](#1-architecture-build-then-write)
2. [HTML Shell Pattern](#2-html-shell-pattern)
3. [CSS: Dark Theme Baseline](#3-css-dark-theme-baseline)
4. [Section Structure with Collapsible Panels](#4-section-structure-with-collapsible-panels)
5. [PS Object to HTML Table](#5-ps-object-to-html-table)
6. [Key-Value Detail Block](#6-key-value-detail-block)
7. [Status Badges](#7-status-badges)
8. [JavaScript: Collapsible Sections](#8-javascript-collapsible-sections)
9. [JavaScript: Copy to Clipboard](#9-javascript-copy-to-clipboard)
10. [Output Naming Convention](#10-output-naming-convention)
11. [HTML Injection Safety](#11-html-injection-safety)
12. [Embedding Data from PowerShell](#12-embedding-data-from-powershell)
13. [Report Assembly Pattern](#13-report-assembly-pattern)
14. [Gotchas](#14-gotchas)
15. [Checklist](#15-checklist)

---

## 1. Architecture: Build Then Write

Collect all data first. Build all HTML strings. Write file once at the end.

```
Collect-Data     → PS objects in memory
Build-Sections   → HTML fragment strings per section
Assemble-Report  → inject sections into shell
Write-Report     → single Out-File call
```

**Never** write partial HTML to disk and append. **Never** use `ConvertTo-Html` for the final
report — its output is inflexible and hard to style. Use it only for quick data previews.

```powershell
# Pattern
$data = Invoke-SIInventory          # returns hashtable of section data
$html = Build-SIReport -Data $data  # returns complete HTML string
$path = Get-SIOutputPath            # e.g. C:\Reports\HOSTNAME_2026-03-07_inventory.html
$html | Out-File -FilePath $path -Encoding UTF8 -NoNewline
Write-Host "Report written: $path"
```

---

## 2. HTML Shell Pattern

The outer shell is a here-string with placeholders for injected content.

```powershell
function New-SIReportShell {
    param(
        [string]$Hostname,
        [string]$Timestamp,
        [string]$CssContent,
        [string]$JsContent,
        [string]$BodyContent
    )
    @"
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>System Inventory — $Hostname — $Timestamp</title>
    <style>
$CssContent
    </style>
</head>
<body>
    <header class="report-header">
        <h1>System Inventory</h1>
        <div class="report-meta">
            <span class="hostname">$Hostname</span>
            <span class="timestamp">Generated: $Timestamp</span>
        </div>
    </header>
    <main class="report-body">
$BodyContent
    </main>
    <script>
$JsContent
    </script>
</body>
</html>
"@
}
```

**Why separate CSS/JS parameters:** Keeps the shell function clean. CSS and JS are stored as
module-level constants or loaded from embedded strings. This makes them unit-testable.

---

## 3. CSS: Dark Theme Baseline

```css
:root {
    --bg-primary:    #1a1a2e;
    --bg-secondary:  #16213e;
    --bg-card:       #0f3460;
    --bg-table-row:  #1a2a4a;
    --bg-table-alt:  #162040;
    --text-primary:  #e0e0e0;
    --text-muted:    #a0a0b0;
    --accent:        #e94560;
    --accent-dim:    #533343;
    --border:        #2a3a5a;
    --status-ok:     #4caf50;
    --status-warn:   #ff9800;
    --status-bad:    #f44336;
    --status-unknown:#9e9e9e;
}

* { box-sizing: border-box; margin: 0; padding: 0; }

body {
    font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
    background: var(--bg-primary);
    color: var(--text-primary);
    font-size: 14px;
    line-height: 1.5;
}

.report-header {
    background: var(--bg-secondary);
    border-bottom: 2px solid var(--accent);
    padding: 1.5rem 2rem;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.report-header h1 { font-size: 1.6rem; color: var(--accent); }
.hostname { font-size: 1.1rem; font-weight: 600; }
.timestamp { color: var(--text-muted); font-size: 0.85rem; display: block; }

.report-body { max-width: 1200px; margin: 0 auto; padding: 1.5rem; }

/* Section cards */
.section {
    background: var(--bg-secondary);
    border: 1px solid var(--border);
    border-radius: 6px;
    margin-bottom: 1rem;
    overflow: hidden;
}

.section-header {
    background: var(--bg-card);
    padding: 0.75rem 1rem;
    cursor: pointer;
    display: flex;
    justify-content: space-between;
    align-items: center;
    user-select: none;
}

.section-header:hover { background: var(--accent-dim); }
.section-title { font-size: 1rem; font-weight: 600; }
.section-toggle { color: var(--text-muted); font-size: 1.2rem; }
.section-body { padding: 1rem; }
.section-body.collapsed { display: none; }

/* Tables */
table { width: 100%; border-collapse: collapse; font-size: 0.9rem; }
th {
    background: var(--bg-card);
    padding: 0.5rem 0.75rem;
    text-align: left;
    color: var(--text-muted);
    font-weight: 600;
    text-transform: uppercase;
    font-size: 0.75rem;
    letter-spacing: 0.05em;
}
td { padding: 0.45rem 0.75rem; border-bottom: 1px solid var(--border); }
tr:nth-child(even) td { background: var(--bg-table-alt); }
tr:nth-child(odd)  td { background: var(--bg-table-row); }
tr:hover td { background: var(--accent-dim); }

/* Key-value detail blocks */
.detail-grid {
    display: grid;
    grid-template-columns: 200px 1fr;
    gap: 0.3rem 1rem;
}
.detail-key   { color: var(--text-muted); font-size: 0.85rem; }
.detail-value { font-weight: 500; word-break: break-all; }

/* Status badges */
.badge {
    display: inline-block;
    padding: 0.2em 0.55em;
    border-radius: 3px;
    font-size: 0.78rem;
    font-weight: 600;
    text-transform: uppercase;
}
.badge-ok      { background: var(--status-ok);      color: #000; }
.badge-warn    { background: var(--status-warn);     color: #000; }
.badge-bad     { background: var(--status-bad);      color: #fff; }
.badge-unknown { background: var(--status-unknown);  color: #fff; }

/* Subsection headings (within a section body, e.g. Hardware sub-blocks) */
.subsection-title {
    font-size: 0.85rem;
    font-weight: 600;
    color: var(--text-muted);
    text-transform: uppercase;
    letter-spacing: 0.06em;
    margin: 1rem 0 0.5rem;
}

/* Copy button */
.copy-btn {
    background: none;
    border: 1px solid var(--border);
    color: var(--text-muted);
    padding: 0.15em 0.5em;
    border-radius: 3px;
    cursor: pointer;
    font-size: 0.75rem;
    margin-left: 0.5rem;
}
.copy-btn:hover { border-color: var(--accent); color: var(--accent); }
```

---

## 4. Section Structure with Collapsible Panels

```powershell
function New-SISection {
    param(
        [string]$Id,
        [string]$Title,
        [string]$Content,
        [switch]$Collapsed
    )
    $collapsedClass = if ($Collapsed) { ' collapsed' } else { '' }
    @"
    <div class="section" id="section-$Id">
        <div class="section-header" onclick="toggleSection('$Id')">
            <span class="section-title">$Title</span>
            <span class="section-toggle" id="toggle-$Id">$(if ($Collapsed) { '+' } else { '−' })</span>
        </div>
        <div class="section-body$collapsedClass" id="body-$Id">
$Content
        </div>
    </div>
"@
}
```

---

## 5. PS Object to HTML Table

```powershell
function ConvertTo-HtmlTable {
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [object[]]$InputObject,
        [string[]]$Headers,      # override column header labels
        [string[]]$Properties    # which properties to include (ordered)
    )
    begin { $rows = [System.Collections.Generic.List[object]]::new() }
    process { $rows.Add($InputObject) }
    end {
        if (-not $rows.Count) { return '<p class="text-muted">No data available.</p>' }

        $props = if ($Properties) { $Properties } else {
            $rows[0].PSObject.Properties.Name
        }
        $hdrs = if ($Headers) { $Headers } else { $props }

        $sb = [System.Text.StringBuilder]::new()
        [void]$sb.Append('<table><thead><tr>')
        foreach ($h in $hdrs) {
            [void]$sb.Append("<th>$(ConvertTo-HtmlSafe $h)</th>")
        }
        [void]$sb.Append('</tr></thead><tbody>')

        foreach ($row in $rows) {
            [void]$sb.Append('<tr>')
            foreach ($prop in $props) {
                $val = $row.$prop
                $safe = ConvertTo-HtmlSafe ($val -as [string])
                [void]$sb.Append("<td>$safe</td>")
            }
            [void]$sb.Append('</tr>')
        }
        [void]$sb.Append('</tbody></table>')
        $sb.ToString()
    }
}
```

**Use `[System.Text.StringBuilder]`** for table construction — string concatenation in a loop
creates O(n²) allocations. For 200+ row software tables this matters.

---

## 6. Key-Value Detail Block

For hardware summary panels where layout is label + value, not tabular.

```powershell
function New-SIDetailGrid {
    param([hashtable]$Data)

    $sb = [System.Text.StringBuilder]::new()
    [void]$sb.Append('<div class="detail-grid">')

    foreach ($key in $Data.Keys) {
        $safeKey = ConvertTo-HtmlSafe $key
        $safeVal = ConvertTo-HtmlSafe ($Data[$key] -as [string])
        [void]$sb.Append(@"
            <span class="detail-key">$safeKey</span>
            <span class="detail-value">$safeVal</span>
"@)
    }
    [void]$sb.Append('</div>')
    $sb.ToString()
}

# Usage:
$cpuDetail = New-SIDetailGrid -Data ([ordered]@{
    'Model'             = $cpu.Name
    'Cores / Threads'   = "$($cpu.NumberOfCores) / $($cpu.NumberOfLogicalProcessors)"
    'Base Clock'        = ConvertTo-GHz $cpu.MaxClockSpeed
    'L3 Cache'          = "$($cpu.L3CacheSize) KB"
    'Architecture'      = 'x64'
    'Socket'            = $cpu.SocketDesignation
})
```

---

## 7. Status Badges

```powershell
function New-SIBadge {
    param([string]$Status)
    $class = switch ($Status) {
        { $_ -in 'OK','Healthy','Running','Licensed','Good' }  { 'ok' }
        { $_ -in 'Warning','Degraded','Notification' }         { 'warn' }
        { $_ -in 'Unhealthy','Failed','Error','Unlicensed' }   { 'bad' }
        default                                                 { 'unknown' }
    }
    "<span class=`"badge badge-$class`">$Status</span>"
}
```

---

## 8. JavaScript: Collapsible Sections

```javascript
function toggleSection(id) {
    const body   = document.getElementById('body-'   + id);
    const toggle = document.getElementById('toggle-' + id);
    const isCollapsed = body.classList.toggle('collapsed');
    toggle.textContent = isCollapsed ? '+' : '−';
}

// Expand/collapse all
function toggleAll(collapse) {
    document.querySelectorAll('.section-body').forEach(function(body) {
        const id = body.id.replace('body-', '');
        const toggle = document.getElementById('toggle-' + id);
        if (collapse) {
            body.classList.add('collapsed');
            toggle.textContent = '+';
        } else {
            body.classList.remove('collapsed');
            toggle.textContent = '−';
        }
    });
}
```

---

## 9. JavaScript: Copy to Clipboard

```javascript
// Button passes itself (this) — value read from data-value attribute, not a JS string arg.
// Never embed the value directly in an onclick string literal (XSS risk on unexpected chars).
function copyValue(btn) {
    const value = btn.dataset.value;
    if (navigator.clipboard && window.isSecureContext) {
        navigator.clipboard.writeText(value).then(function() {
            showCopyFeedback();
        });
    } else {
        // Fallback for file:// protocol
        const ta = document.createElement('textarea');
        ta.value = value;
        ta.style.position = 'fixed';
        ta.style.opacity = '0';
        document.body.appendChild(ta);
        ta.select();
        document.execCommand('copy');
        document.body.removeChild(ta);
        showCopyFeedback();
    }
}

function showCopyFeedback() {
    // Brief visual feedback — implement as a toast or status bar update
    const el = document.getElementById('copy-status');
    if (el) {
        el.textContent = 'Copied!';
        setTimeout(function() { el.textContent = ''; }, 1500);
    }
}
```

**Note:** `navigator.clipboard` requires HTTPS or localhost. For `file://` opened reports,
always include the `execCommand` fallback. It's deprecated but still works for offline files.

Add a copy button next to copyable values:

```powershell
function New-SICopyButton {
    param([string]$Value)
    # Store value in data-value attribute — NOT in an onclick JS string literal.
    # Avoids XSS and breaks from quote characters in software names/paths.
    $safe = ConvertTo-HtmlSafe $Value
    "<button class=`"copy-btn`" data-value=`"$safe`" onclick=`"copyValue(this)`">copy</button>"
}
```

---

## 10. Output Naming Convention

```powershell
function Get-SIOutputPath {
    param(
        [string]$OutputDir,   # from config
        [string]$Hostname = $env:COMPUTERNAME
    )
    $date = Get-Date -Format 'yyyy-MM-dd'
    $name = "${Hostname}_${date}_inventory.html"
    Join-Path $OutputDir $name
}
```

Output directory from config. If the directory doesn't exist, create it with a warning — never
silently fail to write the report.

```powershell
if (-not (Test-Path $outputDir)) {
    New-Item -ItemType Directory -Path $outputDir -Force | Out-Null
    Write-Warning "Output directory did not exist — created: $outputDir"
}
```

---

## 11. HTML Injection Safety

System data (computer names, software names, user names) can contain `<`, `>`, `&`, `"`.
Always escape before embedding in HTML.

```powershell
function ConvertTo-HtmlSafe {
    param([string]$Value)
    if (-not $Value) { return '' }
    $Value `
        -replace '&',  '&amp;'  `
        -replace '<',  '&lt;'   `
        -replace '>',  '&gt;'   `
        -replace '"',  '&quot;' `
        -replace "'",  '&#39;'
}
```

**When NOT to escape:** Pre-built HTML fragments that you construct and control (badges, buttons,
table wrappers). Only escape values that come from system queries.

**Never** put raw system data directly into JavaScript string literals. Use `data-*` attributes
and read them in JS with `element.dataset.value`.

---

## 12. Embedding Data from PowerShell

For values that need to be readable by JS (e.g., for dynamic filtering):

```powershell
# Embed as JSON in a data attribute on a container element
$softwareJson = $softwareList | ConvertTo-Json -Compress
# Escape for embedding in an HTML attribute value
$softwareJsonSafe = $softwareJson -replace '"', '&quot;'
"<div id='software-data' data-inventory='$softwareJsonSafe' hidden></div>"

# Then in JS:
# const data = JSON.parse(document.getElementById('software-data').dataset.inventory);
```

For large datasets (500+ software entries), embed JSON and render client-side rather than
building a 500-row HTML table server-side.

---

## 13. Report Assembly Pattern

```powershell
function Build-SIReport {
    param([hashtable]$Data)

    # Build each section
    $sections = [System.Text.StringBuilder]::new()

    # OS section
    $osContent = New-SIDetailGrid -Data $Data.OS
    [void]$sections.Append((New-SISection -Id 'os' -Title 'Operating System' -Content $osContent))

    # Hardware section
    $hwContent = (Build-SIHardwareContent -Data $Data.Hardware)
    [void]$sections.Append((New-SISection -Id 'hardware' -Title 'Hardware' -Content $hwContent))

    # Software section (collapsed by default — long list)
    $swContent = $Data.Software | ConvertTo-HtmlTable `
        -Properties Name, Version, Publisher, InstallDate
    [void]$sections.Append((New-SISection -Id 'software' -Title 'Installed Software' `
        -Content $swContent -Collapsed))

    # Assemble shell
    New-SIReportShell `
        -Hostname $env:COMPUTERNAME `
        -Timestamp (Get-Date -Format 'yyyy-MM-dd HH:mm:ss') `
        -CssContent $Script:SI_CSS `
        -JsContent  $Script:SI_JS  `
        -BodyContent $sections.ToString()
}
```

Store CSS and JS as module-level string constants (`$Script:SI_CSS`, `$Script:SI_JS`). Don't
re-declare them in every function call.

---

## 14. Gotchas

| Issue | Symptom | Fix |
|-------|---------|-----|
| String concatenation in loop | Report build takes 10+ sec for large software lists | Use `[StringBuilder]` |
| Raw data in HTML | `<` in software name breaks table structure | `ConvertTo-HtmlSafe` on all queried values |
| `clipboard.writeText` fails on `file://` | Copy button silently does nothing | Include `execCommand` fallback |
| `navigator.clipboard` undefined | Old Chromium-based browsers | Feature-detect before calling |
| Value in `onclick` string literal | Quotes/special chars in software names break JS | Use `data-value` attribute; read with `btn.dataset.value` in JS |
| Here-string indentation in function body | Indented `@"` causes leading whitespace in output | `@"` and `"@` must be at column 0 in the source file |
| Large software table slow render | 500+ `<tr>` in static HTML | Embed JSON, render client-side with filter box |
| `Out-File` default encoding | UTF-16 LE on PS 5.1; may show as garbled | Always specify `-Encoding UTF8` |
| `$null` values in detail grid | "null" appears in report | Check `if ($value) { $value } else { 'N/A' }` |
| Missing output directory | `Out-File` throws, no report produced | `New-Item -Force` before writing |
| CSS `@` in here-string | PS interprets `@{` as expression start | Escape as `@@{` or put CSS in a separate variable |

---

## 15. Checklist

- [ ] All system data passed through `ConvertTo-HtmlSafe` before embedding in HTML
- [ ] No raw data in JavaScript string literals — use `data-*` attributes
- [ ] `[StringBuilder]` used for table and section assembly (not `+=` on strings)
- [ ] `clipboard.writeText` has `execCommand` fallback for `file://` protocol
- [ ] `Out-File` uses `-Encoding UTF8 -NoNewline`
- [ ] CSS and JS stored as module-level constants, not redeclared per call
- [ ] Output directory existence checked before writing
- [ ] Output file named `HOSTNAME_YYYY-MM-DD_inventory.html`
- [ ] Large datasets (software, services) rendered collapsed by default
- [ ] `$null` values display as 'N/A', not 'null' or empty cells
- [ ] CSS `@` characters in here-strings escaped or factored out
