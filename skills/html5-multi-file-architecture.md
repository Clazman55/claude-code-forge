# HTML5 Multi-File Application Architecture — Skill Reference

Patterns for extracting single-file HTML5+JS applications into modular ES6 codebases with esbuild bundling.
Validated through a large incremental game project (~12,700-line monolith extracted to 22 ES6 modules, 363KB bundle).

Companion to `html5-single-file-architecture.md` (single-file patterns) and `javascript-vanilla-standards.md` (coding standards). Not a replacement for single-file — many projects should stay single-file. This skill covers when and how to migrate.

---

## Table of Contents

1. [When to Migrate](#1-when-to-migrate)
2. [Build Pipeline (esbuild, No npm)](#2-build-pipeline-esbuild-no-npm)
3. [Project Structure](#3-project-structure)
4. [Global State Namespace (G Pattern)](#4-global-state-namespace-g-pattern)
5. [Module Extraction Order](#5-module-extraction-order)
6. [Circular Dependency Resolution](#6-circular-dependency-resolution)
7. [Render Module Architecture](#7-render-module-architecture)
8. [Pre-Computed Cost Ownership](#8-pre-computed-cost-ownership)
9. [Dev vs Production Split](#9-dev-vs-production-split)
10. [CSS Extraction and Inlining](#10-css-extraction-and-inlining)
11. [UTF-8 Encoding (Windows)](#11-utf-8-encoding-windows)
12. [Archive and Migration](#12-archive-and-migration)
13. [Anti-Patterns](#13-anti-patterns)
14. [Gotchas](#14-gotchas)
15. [Extraction Checklist](#15-extraction-checklist)

---

## 1. When to Migrate

### Signals That Single-File Has Outgrown Itself

- **File exceeds ~5000 lines** — section navigation becomes painful even with numbered sections
- **Multiple developers** need to work on the codebase simultaneously
- **Subsystem boundaries are clear** — economy, military, combat, research are distinct modules
- **IDE performance degrades** — autocomplete, search, linting slow down in a giant file
- **Testing granularity needed** — you want to test modules in isolation

### When to Stay Single-File

- File under ~3000 lines with clear section organization
- Solo developer, no collaboration needed
- Distribution requirement is double-click portability (can still use build step)
- No build tooling available or desired

### Key Principle: Mechanical Extraction

The migration should be purely structural — no logic changes, no renaming, no "while we're at it" improvements. Extract sections verbatim, fix imports, verify the build compiles. Quality improvements come in a separate pass after the extraction is verified.

---

## 2. Build Pipeline (esbuild, No npm)

### Why esbuild Standalone

For projects that don't need npm's ecosystem, ship esbuild as a standalone binary. No node_modules, no package.json, no dependency management overhead.

```
project/
  tools/
    esbuild.exe      # Standalone binary (~10MB), gitignored
  serve.ps1          # Dev server wrapper
  build.ps1          # Production build wrapper
```

### Dev Server (serve.ps1)

```powershell
param([int]$Port = 8000)
$esbuild = Join-Path $PSScriptRoot 'tools\esbuild.exe'
& $esbuild src/main.js --bundle --outdir=src --servedir=src --serve="localhost:$Port" --format=iife
```

- `--outdir=src` places the bundled JS alongside source files (in-memory, not written to disk)
- `--servedir=src` serves all static files (HTML, CSS) from src/
- `--serve` auto-rebundles on every request — no watch mode needed
- `--format=iife` wraps output in IIFE (matches single-file behavior)

### Production Build (build.ps1)

```powershell
$esbuild = Join-Path $PSScriptRoot 'tools\esbuild.exe'
$tempJs = Join-Path $distDir 'game.js'

# 1. Bundle all JS into single IIFE
& $esbuild src/main.js --bundle --format=iife --outfile=$tempJs

# 2. Read source files (UTF-8 without BOM)
$utf8 = [System.Text.UTF8Encoding]::new($false)
$html = [System.IO.File]::ReadAllText('src\index.html', $utf8)
$css  = [System.IO.File]::ReadAllText('src\styles.css', $utf8)
$js   = [System.IO.File]::ReadAllText($tempJs, $utf8)

# 3. Replace dev tags with inlined content
$html = $html -replace '<link rel="stylesheet" href="styles\.css">', "<style>`n$css</style>"
$html = $html -replace '<script src="main\.js"></script>', "<script>`n$js</script>"

# 4. Write dist file
[System.IO.File]::WriteAllText($distFile, $html, $utf8)
Remove-Item $tempJs
```

This produces a single self-contained HTML file identical to the original monolith — works via `file://` protocol.

---

## 3. Project Structure

```
project/
  archive/
    App-v1-single-file.html    # Pre-refactor snapshot (playable, gitignored or committed)
  src/
    index.html                  # HTML shell with dev-mode <link> and <script src>
    styles.css                  # All CSS extracted from <style> block
    main.js                     # Entry point — imports, wires deps, calls init()
    lib/
      break-infinity.js         # Embedded libraries (extracted from inline)
    balance.js                  # Constants/config (Object.freeze)
    state.js                    # All mutable state on G namespace
    util.js                     # Shared utilities (formatting, conversion)
    save.js                     # Save/load, migration, export/import
    economy.js                  # Resource production, building purchase
    [subsystem].js              # One file per game subsystem
    engine.js                   # Game loop, input handling, init
    ui.js                       # Tab switching, modals, notifications
    perf.js                     # Performance overlay
    render/
      render-core.js            # Render dispatcher (routes to tab renderers)
      render-[tab].js           # One file per tab's render logic
  dist/
    App.html                    # Built output (gitignored)
  tools/
    esbuild.exe                 # Bundler binary (gitignored)
  build.ps1
  serve.ps1
```

### index.html (Dev Mode)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>App Name</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <!-- All game HTML -->
    <script src="main.js"></script>
</body>
</html>
```

The `<link>` and `<script src>` tags are replaced during production build with inline `<style>` and `<script>` blocks.

---

## 4. Global State Namespace (G Pattern)

### The Problem

ES6 modules forbid reassigning imported bindings:

```javascript
// FAILS — esbuild rejects this
import { gold } from './state.js';
gold = gold - cost;  // TypeError: Assignment to constant variable
```

### The Solution

All mutable state lives as properties on a single exported object:

```javascript
// state.js
export const G = {
    resources: { gold: D(0), food: D(0) },
    buildings: {},
    leaders: [],
    tier: 'Household',
    productionDirty: true,
    // ... all mutable state
};
```

```javascript
// economy.js — property mutation works fine
import { G } from './state.js';
G.resources.gold = G.resources.gold.minus(cost);  // OK
G.productionDirty = true;                          // OK
```

### What Goes on G

- **All mutable game state** — resources, buildings, counters, flags
- **Auxiliary data objects** — cooldowns, researched flags, equipment inventory
- **Cached DOM references** — `G.el.resourceBar`, `G.el.buildingList`
- **Render state shared across modules** — `G.lastRendered`, `G.milCardRefs`
- **Pre-computed cost caches** — `G.HIRE_COSTS`, `G.RESEARCH_COSTS` (populated by init functions)
- **Cross-module function pointers** — `G.notify`, `G.save` (wired at init time)
- **ID caches** — `G.BUILDING_IDS`, `G.SQUAD_IDS` (computed once from BALANCE config)

### What Stays Module-Local

- **Tab-specific render state** — `const lastHoldings = {}` in render-holdings.js
- **Module-private helpers** — utility functions not needed by other modules
- **Init-time computed constants** — strings, labels, cost info that never change

---

## 5. Module Extraction Order

Extract leaves first, dependents last:

```
Phase 1: Leaves (no game-code imports)
  lib/break-infinity.js   — embedded library, verbatim
  balance.js              — BALANCE config + TABS + constants
  util.js                 — formatNumber, D(), costToDecimals

Phase 2: State Foundation
  state.js                — G namespace, canAfford, spendResources, ID caches

Phase 3: Subsystems (import state + balance only)
  save.js, economy.js, military.js, combat.js,
  research.js, politics.js, tiers.js

Phase 4: UI + Render (import subsystems)
  ui.js, perf.js
  render/*.js             — one per tab + core dispatcher

Phase 5: Engine + Entry
  engine.js               — game loop, input, init (imports everything)
  main.js                 — entry point, wires deps, starts game
```

### Extraction Rules

1. Copy section verbatim — no logic changes
2. Add `'use strict';` and JSDoc `@file` header
3. Convert closure variables to `G.property` references
4. Add `import` statements for dependencies
5. Add `export` to public functions
6. Mark cross-module calls with `// G.fn() — wired in main.js` comments

---

## 6. Circular Dependency Resolution

Three strategies, used in combination:

### Strategy 1: Shared Functions in state.js

Functions needed by many modules that would create cycles live in state.js alongside the state they operate on:

```javascript
// state.js — canAfford/spendResources live here, not in economy.js
// because military.js, tiers.js, politics.js all need them
// and economy.js imports from military.js (for upkeep)
export function canAfford(costs) { /* ... */ }
export function spendResources(costs) { /* ... */ }
```

### Strategy 2: G.fn() Wiring in main.js

For functions where direct import would create a cycle, wire them onto G at init time:

```javascript
// main.js — runs once at startup
import { notify } from './ui.js';
import { getScholarTimeMult } from './tiers.js';
import { recomputeResearchEffects } from './research.js';

G.notify = notify;
G.getScholarTimeMult = getScholarTimeMult;
G.recomputeResearchEffects = recomputeResearchEffects;
```

```javascript
// research.js — calls G.notify() instead of importing notify
// This avoids: research.js → ui.js → ... → research.js cycle
G.notify(nodeName + ' research complete.', 'success');
const timeMult = G.getScholarTimeMult();
```

### Strategy 3: setXDeps() Injection for Render Modules

Render modules that need subsystem functions receive them via dependency injection:

```javascript
// render-overview.js
let getCursusRank;
let checkTransitionReqs;

export function setOverviewDeps(deps) {
    if (deps.getCursusRank) getCursusRank = deps.getCursusRank;
    if (deps.checkTransitionReqs) checkTransitionReqs = deps.checkTransitionReqs;
}
```

```javascript
// main.js — wires deps at init time
import { setOverviewDeps } from './render/render-overview.js';
import { getCursusRank } from './politics.js';
import { checkTransitionReqs } from './tiers.js';

setOverviewDeps({
    getCursusRank: getCursusRank,
    checkTransitionReqs: checkTransitionReqs
});
```

### When to Use Which

| Strategy | Use When |
|----------|----------|
| State.js shared | Function operates on core state, needed by 3+ modules |
| G.fn() wiring | One module needs another's function, direct import cycles |
| setXDeps() | Render modules need subsystem functions (clean separation) |

---

## 7. Render Module Architecture

### Dispatcher Pattern

A single `render()` function in render-core.js routes to tab-specific renderers:

```javascript
// render-core.js
import { renderOverviewTab } from './render-overview.js';
import { renderHoldingsTab } from './render-holdings.js';

export function render() {
    renderResourceBar();
    renderTabStates();
    renderActiveTab();
    renderNotifications();
}

function renderActiveTab() {
    switch (G.activeTab) {
        case 'overview':  renderOverviewTab(); break;
        case 'holdings':  renderHoldingsTab(); break;
        // ...
    }
}
```

### Local vs Shared Render State

Each render module owns its change guards. State referenced by other modules (e.g., hardReset) lives on G:

```javascript
// render-holdings.js — module-local, only this module reads/writes
const lastHoldings = {};
const lastInfra = {};

// state.js — shared, hardReset() clears these
G.lastRendered = {
    tier: '', playtime: '', buildings: '',
    civilPatronVis: null, engAfford: null  // also used by render-holdings
};
```

**Rule:** If hardReset needs to clear it, put it on G.lastRendered. Otherwise keep it module-local.

### Card Pool Pattern

Build DOM cards once, cache element refs, update via property mutation:

```javascript
// First render: create cards and cache refs
if (G.cardRefs.length === 0) {
    for (const id of G.BUILDING_IDS) {
        const card = document.createElement('div');
        card.innerHTML = '...';
        container.appendChild(card);
        G.cardRefs.push({
            card: card,
            owned: card.querySelector('.owned'),
            cost: card.querySelector('.cost'),
            btn: card.querySelector('button'),
            lastOwned: '', lastCost: '', lastDisabled: null
        });
    }
}

// Subsequent renders: update only changed values
for (let i = 0; i < G.cardRefs.length; i++) {
    const ref = G.cardRefs[i];
    const ownedStr = String(G.buildings[id]);
    if (ownedStr !== ref.lastOwned) {
        ref.owned.textContent = ownedStr;
        ref.lastOwned = ownedStr;
    }
}
```

### Render State Reset Pattern

When a render module owns state that hardReset must clear, but the state shouldn't live on G:

```javascript
// render-subsystem.js — module-local render state
let _cardRefs = [];
let _initialized = false;
let _lastRendered = {};

/** Reset render state — exported for hardReset access via G.fn() */
export function resetSubsystemRenderState() {
    _cardRefs = [];
    _initialized = false;
    _lastRendered = {};
}

// main.js — wire into G namespace
import { resetSubsystemRenderState } from './render/render-subsystem.js';
G.fn = () => {
    G.resetSubsystemRenderState = resetSubsystemRenderState;
    // ... other wiring
};

// save.js — hardReset calls it
G.resetSubsystemRenderState();
```

**When to use:** Module-local render state that isn't needed by other modules but must be clearable on hard reset. Alternative to putting everything on G namespace — keeps render internals encapsulated.

[verified in Phase 9f-1: render-legion.js uses this pattern for legion tab render state]

---

## 8. Pre-Computed Cost Ownership

Each subsystem module owns its cost initialization:

```javascript
// military.js — runs at module load time
(function initHireCosts() {
    for (const id in BALANCE.military.squads) {
        G.HIRE_COSTS[id] = costToDecimals(BALANCE.military.squads[id].hireCost);
    }
})();
```

Or via exported init function called from main.js:

```javascript
// research.js
export function initResearchCosts() {
    for (const id of G.RESEARCH_IDS) {
        G.RESEARCH_COSTS[id] = [];
        for (let lv = 0; lv < def.maxLevel; lv++) {
            G.RESEARCH_COSTS[id][lv] = costToDecimals(scaledCost);
        }
    }
}

// main.js
initResearchCosts();
```

**Rule:** Cost caches are declared as empty objects on G in state.js, populated by the owning module's init function. This keeps the dependency arrow pointing the right way (subsystem → state, not state → subsystem).

---

## 9. Dev vs Production Split

| Aspect | Dev (serve.ps1) | Prod (build.ps1) |
|--------|-----------------|-------------------|
| Entry | `http://localhost:8000` | `file://dist/App.html` |
| CSS | `<link rel="stylesheet" href="styles.css">` | Inlined `<style>` |
| JS | `<script src="main.js">` (esbuild bundles in-memory) | Inlined `<script>` |
| Modules | ES6 imports resolved by esbuild per-request | Bundled to single IIFE |
| Source maps | Available in browser DevTools | Not included |
| File:// | Not supported (needs HTTP for module resolution) | Fully supported |

### Dev Console API

Expose testing helpers via explicit `window` assignment (survives IIFE wrapping):

```javascript
// engine.js — not gameplay, testing aid only
window.PF = {
    setSpeed: function(n) { G.devSpeed = Math.max(1, Math.min(n, 100)); },
    grant: function(res, amount) { G.resources[res] = G.resources[res].plus(D(amount)); }
};
```

---

## 10. CSS Extraction and Inlining

Extract CSS from the `<style>` block into `src/styles.css`. Keep section headers for navigation:

```css
/* ============================================================
   DARK THEME
   ============================================================ */

/* ============================================================
   RESOURCE BAR
   ============================================================ */

/* ============================================================
   TAB CONTENT — HOLDINGS
   ============================================================ */
```

Build script replaces the dev `<link>` tag with inlined `<style>` content.

---

## 11. UTF-8 Encoding (Windows)

PowerShell's `Get-Content`/`Set-Content` can mangle Unicode (em dashes, accents). Always use .NET file I/O:

```powershell
$utf8 = [System.Text.UTF8Encoding]::new($false)  # false = no BOM
$content = [System.IO.File]::ReadAllText($path, $utf8)
[System.IO.File]::WriteAllText($path, $content, $utf8)
```

**Never use** `Get-Content | Set-Content` for build pipeline files that contain non-ASCII characters.

---

## 12. Archive and Migration

### Before Extraction

1. Copy the monolith to `archive/App-v1-single-file.html`
2. Verify the archive is playable (double-click, test core features)
3. Commit the archive — this is your rollback point

### Save Version

Mechanical extraction should NOT bump the save version. The data format hasn't changed — only the code structure. If you add new state fields during extraction, that's a logic change, not a structural one.

### Verification Strategy

After extraction:
1. `esbuild src/main.js --bundle --format=iife` must compile with zero errors
2. Production build must produce a working single-file HTML
3. Existing saves must load without migration
4. All prior test phases must still pass

---

## 13. Anti-Patterns

### Don't Reassign Imported Bindings

```javascript
// WRONG — esbuild rejects this
import { gold } from './state.js';
gold = 100;

// RIGHT — mutate properties on imported object
import { G } from './state.js';
G.resources.gold = D(100);
```

### Don't Mix Module-Local and Shared State for Reset

```javascript
// WRONG — hardReset clears G.lastRendered but module keeps stale local copy
const lastRendered = { civilPatronVis: null };  // local
// In hardReset: G.lastRendered.civilPatronVis = null;  // different object!

// RIGHT — use G.lastRendered for anything hardReset needs to clear
if (patronVis !== G.lastRendered.civilPatronVis) {
    G.el.section.style.display = patronVis ? '' : 'none';
    G.lastRendered.civilPatronVis = patronVis;
}
```

### Don't Allocate in Hot Paths

```javascript
// WRONG — new Decimal every tick
const cost = costToDecimals(BALANCE.military.squads[id].hireCost);

// RIGHT — pre-compute at init, read from cache
const cost = G.HIRE_COSTS[id];  // populated once at module load
```

### Don't Rebuild innerHTML for Interactive Elements

```javascript
// WRONG — kills event listeners, expensive
container.innerHTML = '';
for (const campaign of G.activeCampaigns) {
    container.innerHTML += '<div>...</div>';
}

// RIGHT — stable DOM refs, property mutation only
for (let i = 0; i < G.activeCampaignRefs.length; i++) {
    G.activeCampaignRefs[i].progress.textContent = formatTime(remaining);
}
```

### Don't Guess Import Locations

Functions may not live where you'd expect after extraction. Always `grep` for the actual `export` before writing an import:

```bash
# isScholarConcurrentUnlocked — lives in tiers.js, NOT research.js
# getLeaderRatioEfficiency — lives in combat.js, NOT military.js
grep -r "export function functionName" src/
```

---

### Eliminate Tier/Type Duplication with Descriptor Objects

When the same logic exists N times for different tiers or entity types (e.g., household research, patron research, senator research), extract the logic into parameterized internal functions and define a descriptor object per tier:

```javascript
const HOUSEHOLD = {
    config: BALANCE.research.nodes,
    researched: () => G.researched,
    costCache: researchLevelCostCache,
    onComplete: () => { researchedCount++; recomputeResearchEffects(); }
};

function _getLevel(desc, nodeId) { return desc.researched()[nodeId] || 0; }
function _startResearch(desc, nodeId) { /* shared logic */ }

// Thin public wrappers
export function getResearchLevel(nodeId) { return _getLevel(HOUSEHOLD, nodeId); }
export function getPatronResearchLevel(nodeId) { return _getLevel(PATRON, nodeId); }
```

New tier = add one descriptor + thin wrappers. Zero logic duplication. [verified in Phase R3]

---

## 14. Gotchas

1. **esbuild --serve writes bundles to memory, not disk.** The `--outdir=src` flag tells esbuild where to "place" the virtual bundle, but no file is written. This is correct for dev — the server intercepts requests for `main.js` and returns the bundled version.

2. **Module-load-time IIFEs run before main.js wiring.** Cost initialization IIFEs in subsystem modules execute when the module is first imported, before `main.js` wires G.fn() pointers. This is safe as long as the IIFE only depends on `G` and `BALANCE` (both available at import time), not on late-wired functions.

3. **PowerShell backtick in string interpolation.** In `build.ps1`, the newline in `"<style>`n$css</style>"` uses PowerShell's backtick-n for newline, not `\n`. This is a PowerShell-ism that looks wrong but is correct.

4. **Two renderNotifications functions is OK.** If `ui.js` and `render-core.js` both define a local `function renderNotifications()`, there's no conflict — ES6 modules have their own scope. Each is module-private.

5. **git warnings about LF/CRLF** are cosmetic when esbuild outputs LF on Windows. The built output works regardless. Don't chase these warnings.

6. **`window.PF` survives IIFE wrapping** because it's an explicit `window` property assignment, not a module export. esbuild's `--format=iife` doesn't strip `window.*` assignments.

---

## 15. Extraction Checklist

### Pre-Extraction

- [ ] Archive the monolith (`archive/App-v1-single-file.html`)
- [ ] Verify archive is playable
- [ ] Set up esbuild binary in `tools/`
- [ ] Create `serve.ps1` and `build.ps1`
- [ ] Extract CSS to `src/styles.css`
- [ ] Create `src/index.html` shell with `<link>` and `<script src>`
- [ ] Verify dev server starts and serves a test page

### Extraction

- [ ] Extract leaf modules first (library, config, utility)
- [ ] Extract state module with G namespace
- [ ] Extract subsystem modules (one per game system)
- [ ] Extract render modules (one per tab + core dispatcher)
- [ ] Extract engine module (game loop, input, init)
- [ ] Create `main.js` entry point with all wiring
- [ ] Wire G.fn() pointers for cross-module calls
- [ ] Wire setXDeps() for render module dependencies

### Verification

- [ ] `esbuild src/main.js --bundle --format=iife` compiles with zero errors
- [ ] `build.ps1` produces working `dist/App.html`
- [ ] Dev server serves working game at localhost
- [ ] Dist file works via `file://` protocol
- [ ] All prior test phases still pass
- [ ] Hard Reset clears all state (no stale render guards)
- [ ] Save/load round-trip works
- [ ] Export/import round-trip works
- [ ] No console errors in dev or prod
- [ ] Unicode characters render correctly (em dashes, accents)

---

*Provenance: Extracted from a production incremental game project. 22 modules, 363KB bundle, 310 tests passing. Companion to html5-single-file-architecture.md.*
