# HTML5 Single-File Application Architecture — Skill Reference

Patterns for building single-file HTML5+CSS+JS applications with no build tools, no server, and no external dependencies.
Extracted from real-world experience building an HTML5 Canvas game (~2550 lines, single HTML file, 7 board sizes, 18 color schemes, demo/screensaver mode).

Analogous to `powershell-module-architecture.md` for PowerShell projects. Companion to `javascript-vanilla-standards.md` (coding standards) and `html5-canvas-game-development.md` (canvas-specific patterns). For multi-file refactoring of existing single-file projects, see `html5-multi-file-architecture.md`.

---

## Table of Contents

1. [Single-File Constraint](#1-single-file-constraint)
2. [IIFE + Revealing Module Pattern](#2-iife--revealing-module-pattern)
3. [Section Ordering Convention](#3-section-ordering-convention)
4. [State Management](#4-state-management)
5. [Constants Pattern](#5-constants-pattern)
6. [localStorage Wrapper](#6-localstorage-wrapper)
7. [Offscreen Canvas Caching](#7-offscreen-canvas-caching)
8. [Pre-Computed Lookup Tables](#8-pre-computed-lookup-tables)
9. [Set-Based Spatial Lookups](#9-set-based-spatial-lookups)
10. [Performance Overlay Pattern](#10-performance-overlay-pattern)
11. [requestAnimationFrame + Fixed Timestep Loop](#11-requestanimationframe--fixed-timestep-loop)
12. [DOM Interaction Patterns](#12-dom-interaction-patterns)
13. [Demo / Screensaver Mode](#13-demo--screensaver-mode)
14. [Gotchas](#14-gotchas)
15. [New Single-File Project Checklist](#15-new-single-file-project-checklist)

---

## 1. Single-File Constraint

### Why Single File

- **Portable:** Double-click to run. Copy to USB, email, share. No server, no build step.
- **Zero dependencies:** No npm, no webpack, no framework. Everything in one `.html` file.
- **Instant deployment:** The file IS the deliverable. No build artifacts, no deployment pipeline.
- **Permanent:** Will run in any browser 10 years from now. No dependency rot.

### Trade-offs

- No code splitting or lazy loading (everything loads at once)
- No ES module `import`/`export` (requires server or build tool)
- No TypeScript, JSX, or other transpiled languages
- File can get large (2000+ lines) — mitigated by section organization
- Testing is manual (see `manual-testing-methodology.md`)

### Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>App Name</title>
    <style>
        /* All CSS here */
    </style>
</head>
<body>
    <!-- All HTML here -->

    <script>
    (function() {
        'use strict';
        // All JavaScript here, organized in numbered sections
    })();
    </script>
</body>
</html>
```

---

## 2. IIFE + Revealing Module Pattern

All JavaScript wrapped in an Immediately Invoked Function Expression. No globals leak.

```javascript
(function() {
    'use strict';

    // ========================================================
    // 1. CONSTANTS
    // ========================================================
    const MAX_SPEED = 1000;
    const BOARD_SIZES = Object.freeze({ /* ... */ });

    // ========================================================
    // 2. STATE
    // ========================================================
    const state = { score: 0, paused: false };
    const settings = { speed: 500, boardSize: 'medium' };

    // ... more sections ...

    // ========================================================
    // 13. INIT
    // ========================================================
    function init() {
        // Wire event listeners, load settings, start loop
    }

    init();
})();
```

### Anti-pattern: ES Modules in Single File

```javascript
// BAD: requires server or build tool
import { GameEngine } from './engine.js';
export class Snake { ... }
```

### Anti-pattern: Global Variables

```javascript
// BAD: pollutes global scope
var gameState = {};
function updateGame() { ... }
```

---

## 3. Section Ordering Convention

Numbered comment banners enforce dependency flow — later sections depend on earlier ones, never the reverse.

```
 1. CONSTANTS     — Compile-time values, lookup tables, configuration objects
 2. STATE         — All runtime state, settings, cached DOM refs, timing vars
 3. STORAGE       — localStorage wrapper (load/save settings, high scores, cache)
 4. INPUT         — Keyboard handlers, touch handlers, input buffering
 5. SPATIAL       — Collision detection, spatial data structures
 6. COLOR/DOMAIN  — Domain-specific logic (color tables, physics, AI, etc.)
 7. FOOD/ENTITIES — Entity management (spawning, removal, batch operations)
 8. GAME/LOGIC    — Core update logic, state transitions, game rules
 9. RENDER        — Canvas drawing, offscreen caches, visual output
10. UI            — DOM manipulation, modals, status bar, forms
11. DEMO/SPECIAL  — Screensaver mode, auto-play, tutorials, special modes
12. PERF          — Performance overlay, timing instrumentation, profiling
13. INIT          — Wire everything together, start the application
```

### Banner Format

```javascript
// ========================================================
// 7. FOOD
// ========================================================
```

### Rules

- Each section is a logical concern. Rendering never touches input. Storage never touches rendering.
- Section numbers appear in code comments and in CLAUDE.md's "Code Organization" section.
- New sections can be inserted (domain-specific) but the Init section is always last.

---

## 4. State Management

No globals. All state in IIFE-scoped variables.

### Core State Objects

```javascript
/** Core game/app state (changes every tick) */
const state = {
    score: 0,
    foodEaten: 0,
    paused: false,
    gameOver: false,
    running: false,
    currentSpeed: 500,
    snake: [],        // Array of {x, y}
    food: [],         // Array of {x, y}
    demoMode: false
};

/** User-configurable settings (persisted to localStorage) */
const settings = {
    speed: 500,
    boardSize: 'medium',
    colorScheme: 'rainbow',
    speedIncrease: true,
    foodCoverage: 40
};

/** Derived board configuration (recomputed on settings change) */
let boardConfig = { width: 40, height: 25, cellSize: 15 };
```

### Cached DOM Refs

Cache all DOM references at init time to avoid per-frame `getElementById` calls:

```javascript
/** Cached DOM references (populated in init) */
let elScore = null;
let elLength = null;
let elSpeed = null;
let canvas = null;
let ctx = null;
```

```javascript
// In init():
elScore = document.getElementById('score');
elLength = document.getElementById('length');
canvas = document.getElementById('gameCanvas');
ctx = canvas.getContext('2d', { alpha: false });
```

### Anti-pattern: DOM Lookup in Hot Path

```javascript
// BAD: getElementById on every frame
function updateStatusBar() {
    document.getElementById('score').textContent = state.score;
    document.getElementById('length').textContent = state.snake.length;
}
```

---

## 5. Constants Pattern

Use `UPPER_SNAKE_CASE` for compile-time constants. Freeze lookup tables.

```javascript
/** Board size presets */
const BOARD_SIZES = Object.freeze({
    small:  { width: 30, height: 20, cellSize: 15 },
    medium: { width: 40, height: 25, cellSize: 15 },
    large:  { width: 50, height: 30, cellSize: 15 }
});

/** Speed tiers in milliseconds */
const SPEED_TIERS = Object.freeze([1000, 750, 500, 300, 150]);

/** Score per food eaten */
const SCORE_PER_FOOD = 10;

/** Default color scheme name */
const DEFAULT_COLOR_SCHEME = 'rainbow';

/** Food rendering constants */
const FOOD_COLOR = '#ff0000';
const FOOD_PADDING_RATIO = 0.2;
```

### Set-Based Enums

When you need to check membership in a category:

```javascript
/** Bit-depth color schemes (use modulo cycling, not interpolation) */
const BIT_DEPTH_SCHEMES = new Set(['1bit', '2bit', '3bit', '4bit', '5bit', '6bit', '7bit', '8bit']);
```

### Anti-pattern: Magic Numbers

```javascript
// BAD: what does 0.1206 mean?
const cap = Math.floor(total * 0.1206);

// GOOD: named constant with comment
const FOOD_CAP_RATIO = 0.1206;  // ~12% of cells, gives max 10K on Screensaver
const cap = Math.floor(total * FOOD_CAP_RATIO);
```

---

## 6. localStorage Wrapper

Every storage call wrapped in try/catch. Private browsing, storage quotas, and disabled cookies can all cause silent failures.

```javascript
function saveSettings() {
    try {
        localStorage.setItem('appSettings', JSON.stringify(settings));
    } catch {
        // Storage unavailable — continue without persistence
    }
}

function loadSettings() {
    try {
        const saved = localStorage.getItem('appSettings');
        if (saved) {
            const parsed = JSON.parse(saved);
            Object.assign(settings, parsed);
        }
    } catch {
        // Corrupted or unavailable — use defaults
    }
}
```

### Defaults Merge Pattern

```javascript
// Merge saved settings over defaults (new fields get default values)
const DEFAULTS = { speed: 500, boardSize: 'medium', colorScheme: 'rainbow' };
Object.assign(settings, DEFAULTS, parsed);
```

### Large Data Caching

For large pre-computed data (e.g., Hamiltonian paths at ~1.2MB):

```javascript
function cachePath(key, path) {
    try {
        localStorage.setItem('pathCache_' + key, JSON.stringify(path));
    } catch {
        // Quota exceeded — path will be regenerated next time
    }
}

function getCachedPath(key) {
    try {
        const cached = localStorage.getItem('pathCache_' + key);
        return cached ? JSON.parse(cached) : null;
    } catch {
        return null;
    }
}
```

### User-Facing Cache Clear

Always provide a button to clear cached data:

```javascript
document.getElementById('btnClear').addEventListener('click', () => {
    if (!confirm('Clear all saved data?')) return;
    try { localStorage.clear(); } catch { /* ignore */ }
    // Reset in-memory state too
    Object.assign(settings, DEFAULTS);
});
```

---

## 7. Offscreen Canvas Caching

Pre-render expensive layers to offscreen canvases. Blit per frame with `drawImage()`.

### Pattern: Dirty Flag + Offscreen Canvas

```javascript
// State
let gridCanvas = null;
let gridCtx = null;
let gridDirty = true;

let foodCanvas = null;
let foodCtx = null;
let foodDirty = true;

// Build function (called when dirty)
function buildGridCache() {
    const w = canvas.width;
    const h = canvas.height;

    if (!gridCanvas) {
        gridCanvas = document.createElement('canvas');
        gridCtx = gridCanvas.getContext('2d', { alpha: false });
    }
    gridCanvas.width = w;
    gridCanvas.height = h;

    // Draw grid lines...
    gridDirty = false;
}

function buildFoodCache() {
    const w = canvas.width;
    const h = canvas.height;
    const cs = boardConfig.cellSize;

    if (!foodCanvas) {
        foodCanvas = document.createElement('canvas');
        foodCtx = foodCanvas.getContext('2d', { alpha: true });  // sparse overlay needs alpha
    }
    if (foodCanvas.width !== w || foodCanvas.height !== h) {
        foodCanvas.width = w;
        foodCanvas.height = h;
    }

    foodCtx.clearRect(0, 0, w, h);
    foodCtx.fillStyle = FOOD_COLOR;
    const pad = Math.max(1, Math.floor(cs * FOOD_PADDING_RATIO));
    const size = cs - pad * 2;
    for (let i = 0; i < state.food.length; i++) {
        const f = state.food[i];
        foodCtx.fillRect(f.x * cs + pad, f.y * cs + pad, size, size);
    }
    foodDirty = false;
}

// In render():
if (gridDirty) buildGridCache();
ctx.drawImage(gridCanvas, 0, 0);

if (foodDirty) buildFoodCache();
ctx.drawImage(foodCanvas, 0, 0);
```

### Alpha Context Selection

- **Opaque layers** (backgrounds, grids): `{ alpha: false }` — faster compositing
- **Sparse overlays** (food, particles): `{ alpha: true }` — transparent areas don't overwrite content below

### Invalidation Points

Set `foodDirty = true` in every function that modifies the food array:

```javascript
function spawnFood() {
    // ... add food ...
    foodDirty = true;
}

function removeFoodAt(index) {
    // ... remove food ...
    foodDirty = true;
}

function resetGame() {
    gridDirty = true;
    gridCanvas = null;   // Force re-creation on board size change
    foodDirty = true;
    foodCanvas = null;
}
```

---

## 8. Pre-Computed Lookup Tables

Build expensive computations once per configuration change, index at O(1) during render.

```javascript
let colorTable = [];
let colorTableScheme = '';
let colorTableLength = 0;

function buildColorTable(schemeName, scheme, length) {
    colorTable = new Array(length);
    colorTable[0] = scheme.head;
    // ... interpolation logic ...
    colorTableScheme = schemeName;
    colorTableLength = length;
}

function getSegmentColor(index, total) {
    // Rebuild only when configuration changes
    if (colorTableScheme !== settings.colorScheme || colorTableLength !== total) {
        buildColorTable(settings.colorScheme, scheme, total);
    }
    return colorTable[index];  // O(1) lookup
}
```

### fillStyle Change-Guard

Skip redundant `ctx.fillStyle` assignments when adjacent items share a color:

```javascript
let lastColor = '';
for (let i = len - 1; i >= 0; i--) {
    const color = getSegmentColor(i, len);
    if (color !== lastColor) {
        ctx.fillStyle = color;
        lastColor = color;
    }
    ctx.fillRect(seg.x * cs, seg.y * cs, cs, cs);
}
```

This is most effective on gradient schemes where neighboring segments often interpolate to the same hex value.

---

## 9. Set-Based Spatial Lookups

O(1) collision detection using `"x,y"` string keys in `Set` objects.

```javascript
const snakeSet = new Set();
const foodSet = new Set();

function coordKey(x, y) {
    return x + ',' + y;
}

function isSnakeAt(x, y) {
    return snakeSet.has(coordKey(x, y));
}

function isFoodAt(x, y) {
    return foodSet.has(coordKey(x, y));
}

function isOccupied(x, y) {
    return snakeSet.has(coordKey(x, y)) || foodSet.has(coordKey(x, y));
}
```

### Rebuild on State Change

```javascript
function rebuildSnakeSet() {
    snakeSet.clear();
    for (const seg of state.snake) {
        snakeSet.add(coordKey(seg.x, seg.y));
    }
}
```

### Anti-pattern: Linear Array Scan

```javascript
// BAD: O(n) per check, called thousands of times per frame
function isSnakeAt(x, y) {
    return state.snake.some(seg => seg.x === x && seg.y === y);
}
```

---

## 10. Performance Overlay Pattern

Toggleable real-time performance instrumentation for debugging.

### Structure

```javascript
let perfEnabled = false;
const PERF_BUFFER_SIZE = 300;
const perfBuffer = [];
let smoothedFPS = 60;

function recordPerf(frameTime, updateTime, renderTime) {
    const fps = frameTime > 0 ? 1000 / frameTime : 60;
    smoothedFPS = 0.2 * fps + 0.8 * smoothedFPS;  // EMA smoothing

    perfBuffer.push({ frameTime, updateTime, renderTime, fps: smoothedFPS, timestamp: performance.now() });
    if (perfBuffer.length > PERF_BUFFER_SIZE) {
        perfBuffer.shift();
    }

    // Threshold warnings (sustained, not single spikes)
    if (frameTime > 16.67) frameTimeOverCount++;
    if (updateTime > 8) console.warn('[PERF]', { metric: 'updateTime', value: updateTime });
    if (renderTime > 12) console.warn('[PERF]', { metric: 'renderTime', value: renderTime });
}

function renderPerfOverlay(drawCtx) {
    if (!perfEnabled) return;
    // Draw semi-transparent box with stats
    // Color threshold violations red
}
```

### External Access

```javascript
window.__APP_PERF__ = {
    get buffer() { return perfBuffer; },
    get fps() { return smoothedFPS; },
    get enabled() { return perfEnabled; }
};
```

### Toggle Hotkey

```javascript
// F3 toggles overlay (does not interfere with gameplay)
if (e.key === 'F3') {
    e.preventDefault();
    perfEnabled = !perfEnabled;
}
```

### Console Warning Format

Structured objects with `[PERF]` prefix for grep-ability:

```javascript
console.warn('[PERF]', { metric: 'fps', value: 28.5, threshold: 30, board: 'screensaver' });
```

---

## 11. requestAnimationFrame + Fixed Timestep Loop

The only correct way to drive a game loop in the browser.

```javascript
let animFrameId = 0;
let lastTimestamp = 0;
let accumulator = 0;

function gameLoop(timestamp) {
    animFrameId = requestAnimationFrame(gameLoop);  // Schedule next frame first

    if (lastTimestamp === 0) {
        lastTimestamp = timestamp;
        return;
    }

    // Delta cap prevents spiral-of-death on tab return
    const delta = Math.min(timestamp - lastTimestamp, 200);
    lastTimestamp = timestamp;

    if (state.paused || state.gameOver) {
        render();
        return;
    }

    // Fixed timestep with accumulator
    const updateStart = performance.now();
    accumulator += delta;
    let ticks = 0;
    while (accumulator >= state.currentSpeed) {
        updateGame();
        accumulator -= state.currentSpeed;
        if (++ticks >= 4 || state.gameOver) {  // Panic cap: max 4 ticks/frame
            accumulator = 0;
            break;
        }
    }
    const updateTime = performance.now() - updateStart;

    const renderStart = performance.now();
    render();
    const renderTime = performance.now() - renderStart;

    recordPerf(delta, updateTime, renderTime);
    if (perfEnabled) renderPerfOverlay(ctx);
}

// Start the loop
requestAnimationFrame(gameLoop);
```

### Anti-pattern: setInterval

```javascript
// BAD: not synced to display refresh, causes jitter
setInterval(gameLoop, 16);
```

---

## 12. DOM Interaction Patterns

### Cache Refs at Init

```javascript
function init() {
    elScore = document.getElementById('score');
    elLength = document.getElementById('length');
    elFoodCount = document.getElementById('foodCount');
    elSpeed = document.getElementById('speed');
    elHighScore = document.getElementById('highScore');
    canvas = document.getElementById('gameCanvas');
    ctx = canvas.getContext('2d', { alpha: false });
}
```

### Change-Guarded Writes

Avoid redundant DOM writes at high tick rates (60-100Hz):

```javascript
let lastRenderedScheme = '';

function updateStatusBar() {
    elScore.textContent = state.score;
    elLength.textContent = state.snake.length;

    // Only write when value actually changes
    if (settings.colorScheme !== lastRenderedScheme) {
        elSchemeLabel.textContent = settings.colorScheme;
        lastRenderedScheme = settings.colorScheme;
    }
}
```

**Guard key completeness:** When using compound guard keys (`a + '|' + b`), every value
that appears in the rendered output must be included in the key. If the HTML renders values
A, B, and C but the guard only checks A and B, changes to C produce stale displays.

### Event Delegation

One listener on a container instead of per-element listeners:

```javascript
document.addEventListener('change', (e) => {
    if (e.target.type !== 'checkbox') return;
    // Handle all checkbox changes here
});
```

### Data-Attribute Click Routing

For complex tabs with many interactive element types, use a single delegated click
handler with `closest()` and data attributes to route actions:

```javascript
el.militaryTab.addEventListener('click', function(e) {
    const hireBtn = e.target.closest('[data-hire]');
    if (hireBtn) {
        hireSquad(hireBtn.getAttribute('data-hire'));
        return;
    }
    const equipBtn = e.target.closest('[data-equip-arms]');
    if (equipBtn) {
        equipBest(getLeaderById(equipBtn.getAttribute('data-equip-arms')), 'arms');
        return;
    }
    // ... more action types
});
```

Each `if` block checks one action type and returns early. The `closest()` call handles
clicks on child elements inside the button. One listener replaces N per-element listeners
and automatically handles dynamically created elements.

### Multi-Subsystem DOM Pooling

When multiple tabs each have their own pooled DOM elements, use parallel ref arrays
per subsystem — each cleared independently on reset:

```javascript
const cardRefs = [];           // building cards (Holdings tab)
const milCardRefs = [];        // squad cards (Military tab)
const equipShopRefs = { arms: [], armor: [] };  // equipment shop items
const leaderEquipRefs = [];    // leader equip display elements

// Create elements once in init, cache querySelector results immediately:
const item = document.createElement('div');
item.innerHTML = '...';
container.appendChild(item);
equipShopRefs.arms.push({
    inv: item.querySelector('.equip-item-inv'),
    buy: item.querySelector('.equip-buy-btn')
});

// In render loop — zero querySelector calls:
for (let i = 0; i < ARMS_IDS.length; i++) {
    const refs = equipShopRefs.arms[i];
    refs.buy.disabled = !canAfford(EQUIP_COSTS.arms[ARMS_IDS[i]]);
}

// On hard reset — clear each subsystem independently:
cardRefs.length = 0;
milCardRefs.length = 0;
equipShopRefs.arms.length = 0;
leaderEquipRefs.length = 0;
```

Key pattern: cache refs at element creation time, never querySelector in the render loop.

### Subsystem Initialized Flag

When tabs build their DOM on first visit, use an initialized flag to prevent re-creating
elements on every tab switch:

```javascript
let milInitialized = false;

function renderMilitaryTab() {
    if (!milInitialized) {
        // Create squad cards, leader cards, policy buttons — one-time DOM construction
        buildSquadCards();
        buildLeaderCards();
        milInitialized = true;
    }
    // Update existing elements with current state
    updateSquadCounts();
    updateLeaderStats();
}
```

Each subsystem (military, combat, research) gets its own flag. On hard reset, clear all
flags and their associated DOM containers so the next tab visit rebuilds from scratch:

```javascript
function hardReset() {
    milInitialized = false;
    combatInitialized = false;
    researchInitialized = false;
    milContainer.innerHTML = '';
    milCardRefs.length = 0;
    // ... clear each subsystem's refs and containers
}
```

Missing a flag or container in `hardReset()` is a common bug source. When a subsystem has
a centralized `recomputeX()` function, call it from `hardReset()` instead of duplicating
the reset logic inline.

### Filtered Dropdown Management (Mutual Exclusion)

When multiple `<select>` elements draw from the same pool and each must exclude items
selected in other dropdowns:

```javascript
function populateSelect(select, currentVal, excludeIds) {
    select.innerHTML = '<option value="">None</option>';
    for (let i = 0; i < items.length; i++) {
        if (excludeIds.indexOf(items[i].id) >= 0 && items[i].id !== currentVal) continue;
        const opt = document.createElement('option');
        opt.value = items[i].id;
        opt.textContent = items[i].name;
        select.appendChild(opt);
    }
    if (currentVal) select.value = currentVal;
}

// Rebuild all dropdowns with cascading exclusion:
function rebuildDropdowns() {
    populateSelect(el.slot1, state.slot1Id, []);
    state.slot1Id = el.slot1.value;
    populateSelect(el.slot2, state.slot2Id, [state.slot1Id]);
    state.slot2Id = el.slot2.value;
    populateSelect(el.slot3, state.slot3Id, [state.slot1Id, state.slot2Id]);
    state.slot3Id = el.slot3.value;
}
```

All dropdowns share one `onchange` handler that reads all values and calls `rebuildDropdowns()`.

### Entity-Hash Change Detection

When change-guarding a collection where both the set of entities and their individual
properties can change, hash entity properties into a composite string:

```javascript
let leaderHash = leaders.length.toString();
for (let i = 0; i < leaders.length; i++) {
    const l = leaders[i];
    leaderHash += '|' + l.id + ':' + l.stat1 + ',' + l.stat2 + ',' + l.equip;
}
if (lastHash !== leaderHash) {
    rebuildUI();
    lastHash = leaderHash;
}
```

**Important:** The hash must include exactly the properties that appear in the rendered
output. Including extra properties causes unnecessary rebuilds. Missing properties causes
stale displays.

### CSS GPU Hint

For large canvases (1920x1080+), promote to GPU compositing layer:

```css
canvas { will-change: transform; }
```

Only use this when the canvas is large enough for compositing to be a measurable bottleneck.

### Static vs Dynamic Card Text

When a card's text is set at creation time (static) but a render loop also writes to the
same element (dynamic), the static text is dead code — it flashes for one frame then gets
overwritten. Skip static text generation for elements managed by the render loop:

```javascript
// At card creation:
if (!def.hasDynamicText) {
    prodEl.textContent = computeStaticText(def);
}
// Otherwise leave empty — render loop fills it

// In render loop:
if (refs.prod) {
    const text = computeDynamicText(def, owned, active);
    if (text !== prev.prodText) {
        refs.prod.textContent = text;
        prev.prodText = text;
    }
}
```

This avoids duplicating string-building logic between creation and render paths.

### Category Grouping with Conditional Visibility

When a tab displays many items (research nodes, skills, buildings), group them into
categories using a config array. Some categories can be hidden until unlocked:

```javascript
const CATEGORIES = [
    { label: 'Basic',    ids: ['woodcut', 'mining'] },
    { label: 'Advanced', ids: ['smelting', 'alchemy'] },       // hidden until unlocked
    { label: 'Mastery',  ids: ['transmutation', 'synthesis'] }  // hidden until unlocked
];

function initCategoryDom(container) {
    for (const cat of CATEGORIES) {
        const label = document.createElement('h3');
        label.textContent = cat.label;
        const row = document.createElement('div');
        row.className = 'category-row';

        // Conditionally hide at creation time
        if (cat.hidden) {
            label.style.display = 'none';
            row.style.display = 'none';
            hiddenCatRefs[cat.label] = { label, row };  // Cache for later reveal
        }

        container.appendChild(label);
        container.appendChild(row);

        for (const id of cat.ids) {
            row.appendChild(createCard(id));
        }
    }
}

// When unlock condition is met:
function revealCategory(catLabel) {
    const refs = hiddenCatRefs[catLabel];
    if (refs) {
        refs.label.style.display = '';
        refs.row.style.display = '';
    }
}
```

Clear cached refs in `hardReset()` so categories re-hide on fresh start.

### Overview / Summary Tab (Multi-Subsystem Aggregation)

A dashboard tab that pulls data from multiple subsystems. Each section is conditionally
visible based on unlock state, with visibility change-guarded via dedicated `*Vis` keys:

```javascript
function renderOverviewTab() {
    // Economy section — always visible
    updateEconomyOverview();

    // Military section — visible after unlock
    const milVis = isMilitaryUnlocked();
    if (milVis !== lastRendered.ovMilVis) {
        el.overviewMilitary.style.display = milVis ? '' : 'none';
        lastRendered.ovMilVis = milVis;
    }
    if (milVis) updateMilitaryOverview();

    // Research section — visible after first research
    const resVis = researchedCount > 0;
    if (resVis !== lastRendered.ovResVis) {
        el.overviewResearch.style.display = resVis ? '' : 'none';
        lastRendered.ovResVis = resVis;
    }
    if (resVis) updateResearchOverview();
}
```

**Critical:** Initialize `*Vis` keys to `null` in `hardReset()` (not `''` or `false`)
so the first render after reset always evaluates the condition fresh:

```javascript
function hardReset() {
    // ... reset all state ...
    lastRendered.ovMilVis = null;
    lastRendered.ovResVis = null;
}
```

This pattern scales cleanly — each new subsystem adds one visibility key and one
conditional block. The overview tab never renders content for locked subsystems.

### Two-Step Confirmation for Irreversible Actions

For actions that cannot be undone (tier transitions, prestige resets, permanent
purchases), use a state-based two-step confirmation instead of `window.confirm()`.
This keeps the UI in-page and lets you style the warning:

```javascript
let confirmingTransition = false;

function handleTransitionClick() {
    if (!confirmingTransition) {
        confirmingTransition = true;
        // Button text changes to "Are you sure? Click again to confirm"
        // Render loop picks up the flag and updates the button
        return;
    }
    // Second click — execute the irreversible action
    confirmingTransition = false;
    executeTransition();
}

// In render loop:
if (confirmingTransition) {
    btn.textContent = 'Confirm Transition (irreversible)';
    btn.classList.add('btn-confirm-warning');
} else {
    btn.textContent = 'Advance to Next Tier';
    btn.classList.remove('btn-confirm-warning');
}
```

Reset `confirmingTransition = false` on tab switch and hard reset so the player
doesn't accidentally confirm after navigating away and back.

### Conditional Render Guard (Upstream Change Flag)

When a render subsection is expensive but only needs rebuilding when a specific
upstream value changes (not every frame), hoist the change detection before the
render loop and pass a flag:

```javascript
function renderTab() {
    // Hoist upstream change detection once per frame
    const specialMult = getSpecialMultiplier();
    const specialChanged = specialMult !== lastRendered._specialMult;
    if (specialChanged) lastRendered._specialMult = specialMult;

    for (let i = 0; i < CARD_IDS.length; i++) {
        const refs = cardRefs[i];
        const prev = lastState[CARD_IDS[i]];

        // Cheap updates — always check
        updateCount(refs, prev);
        updateCost(refs, prev);

        // Expensive string rebuild — only when upstream changed
        if (specialChanged || prev.prodText === '') {
            const text = buildProductionText(CARD_IDS[i], specialMult);
            if (text !== prev.prodText) {
                refs.prod.textContent = text;
                prev.prodText = text;
            }
        }
    }
}
```

This is more targeted than a full dirty flag — the dirty flag triggers full
recalculation of all derived values, while this guards only the render path
for a specific subsection. Use when the upstream value changes rarely (e.g.,
a leader leveling up) but the render loop runs every frame.

### Tab Relabeling Based on Game State

When a tab's label changes based on progression (e.g., "Holdings" → "Civil" at
tier 2), use the same change-guard pattern as other DOM writes:

```javascript
// In render loop (runs every frame for the active tab):
const newLabel = state.tier === 'Patron' ? 'Civil' : 'Holdings';
if (newLabel !== lastRendered.tabLabel) {
    el.tabButton.textContent = newLabel;
    lastRendered.tabLabel = newLabel;
}
```

This is trivial but easy to forget — if you hardcode the tab label in HTML and
only change it in the transition function, hard reset won't restore the original
label (the render system never re-evaluates it). Always let the render loop own
the label text so it stays correct across save/load/reset cycles.

---

## 13. Demo / Screensaver Mode

Patterns for auto-play, screensaver, and demonstration modes.

### Pre-Computed Path

Generate a path that visits every cell (Hamiltonian path). Cache in localStorage.

```javascript
let demoPath = [];  // Array of {x, y}

async function generatePathAsync(width, height) {
    // Chunk generation to avoid blocking UI
    // Show progress bar during generation
    // Cache result in localStorage when done
}
```

### Index-Based Tracking (O(1) per tick)

At 78K+ cells, array manipulation (`unshift`/`splice`) is O(n). Use index math instead:

```javascript
let demoPathIndex = 0;   // Head position in path
let demoTailIdx = 0;     // Tail position in path
let demoSnakeLen = 5;    // Current length

function updateDemo() {
    // Advance head
    demoPathIndex = (demoPathIndex + 1) % demoPath.length;
    demoSnakeLen++;

    // Advance tail (unless growing)
    if (!ateFood) {
        demoTailIdx = (demoTailIdx + 1) % demoPath.length;
        demoSnakeLen--;
    }
}
```

### Pause-Aware Timer

Encapsulate timer logic so it can be paused/resumed without leaking into other modules:

```javascript
let demoElapsed = 0;   // Accumulated ms
let demoLastTick = 0;  // 0 when paused

function getDemoElapsed() {
    return demoElapsed + (demoLastTick > 0 ? Date.now() - demoLastTick : 0);
}

function demoPauseTimer() {
    if (demoLastTick > 0) {
        demoElapsed += Date.now() - demoLastTick;
        demoLastTick = 0;
    }
}

function demoResumeTimer() {
    demoLastTick = Date.now();
}
```

### Auto-Cycling on Death

```javascript
function demoDeathCycle() {
    schemeIndex = (schemeIndex + 1) % schemeNames.length;
    settings.colorScheme = schemeNames[schemeIndex];
    resetDemoRun();
}
```

---

## 14. Gotchas

| Issue | Symptom | Fix |
|-------|---------|-----|
| `alpha: false` on sparse overlay canvas | Transparent areas render as black, overwriting content below | Use `{ alpha: true }` for sparse layers (food, particles) |
| `will-change: transform` on small canvas | Wastes GPU memory with no benefit | Only use on large canvases (1920x1080+) |
| Redundant `ctx.fillStyle` in loop | Measurable perf cost at 78K+ iterations | Change-guard: skip if color === lastColor |
| Snake borders at small cell sizes | Visual noise, doubles draw calls | Disable borders when cellSize < 7px, or remove entirely for smooth gradients |
| `file://` protocol | Cross-origin blocks iframe/XHR, may block clipboard API | Use manual testing (TestingGuide), clipboard fallback |
| localStorage ~5MB limit | Large cached data (1.2MB path) may fail | Wrap in try/catch, regenerate on failure |
| `Array.unshift()` at 78K elements | O(n) per tick, 100ms+ per operation | Use index-based tracking (Section 13) |
| Bit-depth color + scaled table lookup | Adjacent segments map to same color (blocks instead of stripes) | Compute bit-depth colors directly via modulo, bypass table |
| Setting `canvas.width`/`height` | Clears canvas contents AND resets context state | Only set dimensions when they actually change |
| `setInterval` for game loop | Not synced to display refresh, causes jitter | Use `requestAnimationFrame` with fixed timestep (Section 11) |
| Demo path-index tracking with pause | Timer accumulates real time including paused time | Use accumulated elapsed timer with pause/resume helpers (Section 13) |

---

## 15. New Single-File Project Checklist

When starting a new single-file HTML5 application:

- [ ] Create `E:\ProjectName\ProjectName.html` with HTML + CSS + `<script>` structure
- [ ] Add IIFE wrapper with `'use strict'`
- [ ] Add numbered section comment banners (Constants through Init)
- [ ] Define Constants section (board sizes, speed tiers, named constants)
- [ ] Define State section (`state` object, `settings` object, cached DOM refs)
- [ ] Add localStorage wrapper (save/load settings with try/catch)
- [ ] Add init function with DOM ref caching
- [ ] Wire `requestAnimationFrame` game loop with fixed timestep
- [ ] Add performance overlay (F3 toggle, EMA FPS, threshold warnings)
- [ ] Add `window.__APP_PERF__` exposure for console debugging
- [ ] Add `will-change: transform` CSS on canvas if large boards are expected
- [ ] Test in Chrome, Firefox, and Edge on `file://` protocol
- [ ] Create `tests/TestingGuide.html` (see `manual-testing-methodology.md`)
- [ ] Create CLAUDE.md with quality gate and phase checklist
