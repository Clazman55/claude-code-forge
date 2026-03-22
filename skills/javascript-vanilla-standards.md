# Vanilla JavaScript Coding Standards — Portable HTML5 Projects

Comprehensive coding standards and best practices for vanilla JavaScript projects with no framework, no build step, no bundler, and no transpiler. Designed for portable HTML5 Canvas games and single-file applications. Current as of 2025-2026.

Companion to: `html5-canvas-game-development.md` (game-specific patterns — game loop, canvas rendering, data structures, state management, mobile/touch, save data).

---

## 1. Code Style

### Naming Conventions

| Context | Convention | Example |
|---------|-----------|---------|
| Variables, functions | camelCase | `playerScore`, `getNextWave()` |
| Classes, constructors | PascalCase | `EnemySpawner`, `ParticlePool` |
| Constants (true compile-time) | UPPER_SNAKE_CASE | `MAX_ENEMIES`, `TILE_SIZE` |
| Private fields/methods | `#` prefix (ES2022) | `#health`, `#recalculate()` |
| Boolean variables | `is`/`has`/`can` prefix | `isAlive`, `hasShield`, `canFire` |
| Event handlers | `on` + event name | `onClick`, `onKeyDown` |
| Factory functions | `create` prefix | `createBullet()`, `createEnemy()` |
| Enum-like objects | PascalCase + UPPER values | `GameState.PLAYING` |

### Semicolons

Always use explicit semicolons. ASI (Automatic Semicolon Insertion) has well-documented edge cases that cause silent bugs, particularly with lines starting with `(`, `[`, or template literals. In a no-linter, no-build environment, explicit semicolons are non-negotiable.

```javascript
// CORRECT
const x = 5;
doSomething();

// DANGEROUS — ASI can misparse
const x = getValue()
[1, 2, 3].forEach(n => console.log(n)) // Parsed as getValue()[1,2,3].forEach(...)
```

### Bracing

Use K&R style (opening brace on the same line). Always use braces for control structures, even single-line bodies. This prevents bugs when adding lines later and makes diffs cleaner.

```javascript
// CORRECT
if (enemy.isAlive) {
    enemy.update(dt);
}

// NEVER — works today, breaks tomorrow
if (enemy.isAlive)
    enemy.update(dt);
```

### Indentation

Use 4 spaces. No tabs. This matches the PowerShell baseline skill and keeps alignment consistent across the project. Configure your editor; do not rely on defaults.

### Line Length

Aim for 100 characters. Hard limit at 120. Break long expressions at operators or after commas.

```javascript
// Break long conditions
if (player.x > bounds.left
    && player.x < bounds.right
    && player.y > bounds.top
    && player.y < bounds.bottom) {
    // ...
}

// Break long function calls
const result = calculateTrajectory(
    startPos.x, startPos.y,
    velocity, angle,
    gravity, windResistance
);
```

### Variable Declarations [verified in Phase 1]

- Use `const` by default. Switch to `let` only when reassignment is genuinely needed.
- Never use `var`. It has function scope, not block scope, and hoists in confusing ways.
- Declare variables as close to their first use as possible.
- One declaration per line.

```javascript
// CORRECT
const maxHP = 100;
const startX = canvas.width / 2;
let currentHP = maxHP; // Will be modified

// WRONG
var x = 1, y = 2, z = 3;
```

### Comparison Operators [verified in Phase 1]

Always use `===` and `!==`. Never use `==` or `!=` except for the specific `value == null` idiom (checks both `null` and `undefined` — this is the one legitimate use case).

```javascript
if (score === 0) { /* ... */ }      // Correct
if (input == null) { /* ... */ }     // Acceptable: checks null OR undefined
if (type == 'string') { /* ... */ }  // WRONG: use ===
```

---

## 2. Modern JavaScript Features — Browser Safety Guide

All features listed here are safe for use in all evergreen browsers (Chrome, Firefox, Safari, Edge) without transpilation as of 2025. Each feature includes the ES version and approximate global support percentage.

### ES2020 (>92% global support)

```javascript
// Optional chaining — safe access into nested structures
const name = player?.inventory?.weapon?.name;
const attack = player?.getWeapon?.();   // On methods too
const item = inventory?.[slotIndex];     // On bracket notation

// Nullish coalescing — default only for null/undefined (NOT 0, '', false)
const lives = savedData.lives ?? 3;
const name = config.playerName ?? 'Player 1';

// BigInt — arbitrary precision integers (rarely needed in games)
const huge = 9007199254740993n;

// Promise.allSettled — wait for all, regardless of rejection
const results = await Promise.allSettled([loadAudio(), loadSprites()]);

// globalThis — universal global reference
// Replaces window/self/global depending on context
const ctx = globalThis.AudioContext || globalThis.webkitAudioContext;

// Dynamic import()
const module = await import('./levels/level1.js');
```

### ES2021 (>92% global support)

```javascript
// String.prototype.replaceAll — no more regex for global replace
const clean = rawInput.replaceAll('&amp;', '&');

// Logical assignment operators
options.volume ??= 0.5;    // Assign only if null/undefined
options.music ||= true;     // Assign only if falsy
cache.hits &&= cache.hits + 1; // Assign only if truthy

// Numeric separators — readability for large numbers
const MAX_SCORE = 1_000_000;
const COLOR_MASK = 0xFF_00_FF;

// Promise.any — first successful result
const asset = await Promise.any([loadFromCache(), loadFromNetwork()]);

// WeakRef — weak references (advanced, use sparingly)
const ref = new WeakRef(heavyObject);
const obj = ref.deref(); // Returns undefined if GC'd
```

### ES2022 (~88% global support)

```javascript
// Private class fields and methods — true encapsulation
class Player {
    #health = 100;          // Private field
    #maxHealth = 100;

    get health() { return this.#health; }

    #clampHealth() {        // Private method
        this.#health = Math.min(this.#health, this.#maxHealth);
    }

    heal(amount) {
        this.#health += amount;
        this.#clampHealth();
    }

    static #instanceCount = 0; // Private static
}

// Top-level await (in modules only — requires type="module")
// Not relevant for single-file IIFE apps; listed for completeness

// at() — relative indexing from end
const last = enemies.at(-1);          // Last element
const secondLast = enemies.at(-2);

// Object.hasOwn() — replacement for hasOwnProperty
if (Object.hasOwn(config, 'difficulty')) { /* ... */ }

// Error cause — chain error context
try {
    loadLevel(id);
} catch (err) {
    throw new Error(`Failed to load level ${id}`, { cause: err });
}
```

### ES2023 (~88% global support)

```javascript
// Non-mutating array methods — critical for avoiding side effects
const sorted = scores.toSorted((a, b) => b - a);      // Returns new array
const reversed = items.toReversed();                     // Returns new array
const spliced = inventory.toSpliced(2, 1, newItem);     // Returns new array
const replaced = tiles.with(index, newTile);             // Returns new array

// findLast / findLastIndex — search from end
const lastAlive = enemies.findLast(e => e.isAlive);
const lastIdx = enemies.findLastIndex(e => e.isAlive);

// Hashbang grammar (#! for CLI scripts — not relevant for browser)
```

### ES2024 (~85% global support)

```javascript
// Object.groupBy — group array elements by key
const byType = Object.groupBy(enemies, e => e.type);
// { 'grunt': [...], 'boss': [...], 'minion': [...] }

// Map.groupBy — same but returns a Map
const byState = Map.groupBy(entities, e => e.state);

// Promise.withResolvers — externalize resolve/reject
const { promise, resolve, reject } = Promise.withResolvers();
// Useful for bridging callback-based APIs

// String isWellFormed / toWellFormed — Unicode safety
if (!input.isWellFormed()) {
    input = input.toWellFormed();
}
```

### structuredClone (available since March 2022, >93% support)

```javascript
// Deep clone objects without JSON.parse(JSON.stringify()) hacks
const snapshot = structuredClone(gameState);

// Handles: Date, RegExp, Map, Set, ArrayBuffer, typed arrays
// Does NOT handle: Functions, DOM nodes, Error objects, Symbols
```

### Features to AVOID Without Transpilation

| Feature | Reason |
|---------|--------|
| ES2025 Iterator helpers | Too new — limited browser support as of early 2026 |
| Decorators (Stage 3) | Not yet standardized; no native browser support |
| Pipeline operator | Stage 2 proposal; not shipping |
| Pattern matching | Stage 1; years away |
| `using` declarations | ES2025; still rolling out to browsers |

---

## 3. Strict Mode

### When 'use strict' Is Required

ES modules (`<script type="module">`) are automatically strict. For non-module scripts (classic `<script>` tags), you must add the directive manually.

**For a single-file canvas game loaded as a classic script, always include 'use strict'.**

```javascript
// At the top of your IIFE or script
'use strict';

// What strict mode prevents:
// - Assigning to undeclared variables (silent global creation)
// - Duplicate parameter names
// - Deleting variables/functions
// - Octal literals (0777)
// - with() statements
// - Writing to read-only/getter-only properties (silently ignored otherwise)
```

### Practical Impact for Games

Strict mode catches the most common vanilla JS bug: accidental globals. Without strict mode, a typo like `playrX = 50` silently creates a global variable instead of throwing a ReferenceError. In a game with dozens of position variables being updated 60 times per second, this category of bug is devastating and hard to trace.

```javascript
'use strict';

(function() {
    const playerX = 100;
    playrX = 200; // ReferenceError! Caught immediately.
})();
```

---

## 4. Module Patterns for Single-File Apps

### Recommended: IIFE with Namespace [verified in Phase 1]

For a portable, single-file game with no build step and loaded as a classic script, the IIFE (Immediately Invoked Function Expression) is the correct pattern. ES modules require a server (no `file://` protocol support) and add complexity for single-file distribution.

```javascript
'use strict';

/**
 * Main game namespace. Contains all game logic and state.
 * @namespace
 */
const Game = (function() {

    // === Private State ===
    let score = 0;
    let isRunning = false;

    // === Private Constants ===
    const TICK_RATE = 1000 / 60;

    // === Private Functions ===
    function updatePhysics(dt) {
        // ...
    }

    // === Public API (Revealing Module) ===
    return Object.freeze({
        start() {
            isRunning = true;
            requestAnimationFrame(mainLoop);
        },

        stop() {
            isRunning = false;
        },

        get score() { return score; },
        get isRunning() { return isRunning; }
    });

})();
```

### Why Object.freeze on the Public API

`Object.freeze` prevents accidental mutation of the module's public interface. Without it, external code (or a typo) can overwrite methods. The freeze is shallow — it does not affect objects returned by getters.

### Multiple Modules in One File

If the game grows, split logical concerns into separate IIFEs that communicate through a shared namespace:

```javascript
'use strict';

/** @namespace */
const App = {};

// --- Module: Input ---
App.Input = (function() {
    const keys = {};

    function onKeyDown(e) { keys[e.code] = true; }
    function onKeyUp(e) { keys[e.code] = false; }

    document.addEventListener('keydown', onKeyDown);
    document.addEventListener('keyup', onKeyUp);

    return Object.freeze({
        isPressed(code) { return keys[code] === true; }
    });
})();

// --- Module: Audio ---
App.Audio = (function() {
    // Can reference App.Input if needed
    // ...
    return Object.freeze({ /* ... */ });
})();
```

### When to Consider ES Modules Instead

Use `<script type="module">` if ALL of these are true:
- The game will always be served from HTTP(S), never `file://`
- You want to split across multiple files for development
- You accept the deferred-by-default loading behavior
- You want automatic strict mode

Even then, for distribution, consider bundling back to a single file.

---

## 5. Error Handling

### General Principles

- Catch errors at boundaries, not everywhere. The game loop, asset loading, and save/load are boundaries.
- Never catch and swallow silently. At minimum, log with context.
- Use `Error` cause chaining (ES2022) to preserve the original stack.
- Fail fast during development; fail gracefully in production.

### Game Loop Error Boundary

Wrap the game loop so a single bad frame does not crash the entire game:

```javascript
function mainLoop(timestamp) {
    frameId = requestAnimationFrame(mainLoop);

    try {
        // ... update and render ...
    } catch (err) {
        console.error('[Game Loop Error]', err);

        // Strategy: skip frame and continue
        // For critical errors, stop the loop:
        // cancelAnimationFrame(frameId);
        // showErrorScreen(err);
    }
}
```

### Asset Loading with Fallbacks

```javascript
/**
 * Load an image with timeout and fallback.
 * @param {string} src - Image URL
 * @param {number} [timeoutMs=5000] - Timeout in milliseconds
 * @returns {Promise<HTMLImageElement>}
 */
function loadImage(src, timeoutMs = 5000) {
    return new Promise((resolve, reject) => {
        const img = new Image();
        const timer = setTimeout(() => {
            reject(new Error(`Image load timed out: ${src}`));
        }, timeoutMs);

        img.onload = () => {
            clearTimeout(timer);
            resolve(img);
        };

        img.onerror = () => {
            clearTimeout(timer);
            reject(new Error(`Image failed to load: ${src}`));
        };

        img.src = src;
    });
}

// Usage with graceful degradation
async function loadAssets() {
    const results = await Promise.allSettled([
        loadImage('sprites/player.png'),
        loadImage('sprites/enemies.png'),
        loadImage('sprites/background.png')
    ]);

    const assets = {};
    const failures = [];

    results.forEach((result, i) => {
        if (result.status === 'fulfilled') {
            assets[assetNames[i]] = result.value;
        } else {
            failures.push(result.reason.message);
            assets[assetNames[i]] = createPlaceholder(); // Colored rectangle
        }
    });

    if (failures.length > 0) {
        console.warn('[Asset Load] Fallbacks used:', failures);
    }

    return assets;
}
```

### Custom Error Types

```javascript
class GameError extends Error {
    /** @param {string} message @param {Object} [options] */
    constructor(message, options) {
        super(message, options);
        this.name = 'GameError';
    }
}

class AssetLoadError extends GameError {
    /** @param {string} assetPath @param {Error} [cause] */
    constructor(assetPath, cause) {
        super(`Failed to load asset: ${assetPath}`, { cause });
        this.name = 'AssetLoadError';
        this.assetPath = assetPath;
    }
}
```

### Global Error Handlers

```javascript
// Catch unhandled errors — last resort safety net
window.addEventListener('error', (event) => {
    console.error('[Unhandled Error]', event.error);
    // Optionally: pause game, show error overlay
});

// Catch unhandled promise rejections
window.addEventListener('unhandledrejection', (event) => {
    console.error('[Unhandled Rejection]', event.reason);
    event.preventDefault(); // Suppress default browser logging
});
```

---

## 6. Performance

### Avoiding GC Pressure

Garbage collection pauses are the #1 source of frame hitches in JS games. The core rule: **allocate nothing in the hot path**.

#### Pre-allocate and Reuse Objects

```javascript
// BAD — creates new object every frame, 60 times per second
function getVelocity(speed, angle) {
    return { x: Math.cos(angle) * speed, y: Math.sin(angle) * speed };
}

// GOOD — reuse a shared result object
const _velocity = { x: 0, y: 0 };
function getVelocity(speed, angle, out = _velocity) {
    out.x = Math.cos(angle) * speed;
    out.y = Math.sin(angle) * speed;
    return out;
}
```

#### Object Pooling Pattern

```javascript
class ObjectPool {
    #pool = [];
    #factory;
    #reset;

    /**
     * @param {Function} factory - Creates a new instance
     * @param {Function} reset - Resets an instance for reuse
     * @param {number} [initialSize=0] - Pre-allocate this many
     */
    constructor(factory, reset, initialSize = 0) {
        this.#factory = factory;
        this.#reset = reset;
        for (let i = 0; i < initialSize; i++) {
            this.#pool.push(factory());
        }
    }

    /** @returns {*} A fresh or recycled instance */
    acquire() {
        const obj = this.#pool.length > 0
            ? this.#pool.pop()
            : this.#factory();
        return obj;
    }

    /** @param {*} obj - Instance to return to the pool */
    release(obj) {
        this.#reset(obj);
        this.#pool.push(obj);
    }

    get available() { return this.#pool.length; }
}

// Usage
const bulletPool = new ObjectPool(
    () => ({ x: 0, y: 0, vx: 0, vy: 0, active: false }),
    (b) => { b.x = 0; b.y = 0; b.vx = 0; b.vy = 0; b.active = false; },
    200 // Pre-allocate 200 bullets
);
```

#### Hot Path Allocation Checklist

| Pattern | Allocates? | Fix |
|---------|-----------|-----|
| `{ x, y }` object literal | Yes | Use pre-allocated output objects |
| `[a, b, c]` array literal | Yes | Use pre-allocated arrays or typed arrays |
| String concatenation | Yes | Avoid in render loop; cache strings |
| `Array.map/filter/reduce` | Yes (new array) | Use `for` loops with pre-allocated storage |
| `Function.bind()` | Yes (new function) | Bind once during init, store reference |
| Template literals | Yes (new string) | Cache if used repeatedly with same values |
| Spread `...args` | Yes | Use explicit parameters |
| `Array.from()` | Yes | Use typed arrays or pre-allocated arrays |

### Typed Arrays

Use typed arrays when working with large homogeneous numeric data. They provide contiguous memory layout, better cache coherence, and direct compatibility with Canvas/WebGL APIs.

```javascript
// Particle system with struct-of-arrays layout
const MAX_PARTICLES = 10000;
const particleX = new Float32Array(MAX_PARTICLES);
const particleY = new Float32Array(MAX_PARTICLES);
const particleVX = new Float32Array(MAX_PARTICLES);
const particleVY = new Float32Array(MAX_PARTICLES);
const particleLife = new Float32Array(MAX_PARTICLES);
let particleCount = 0;

// Update loop — cache-friendly sequential access
for (let i = 0; i < particleCount; i++) {
    particleX[i] += particleVX[i] * dt;
    particleY[i] += particleVY[i] * dt;
    particleLife[i] -= dt;
}

// Direct pixel manipulation
const imageData = ctx.getImageData(0, 0, width, height);
const pixels = imageData.data; // Uint8ClampedArray — 0-255 clamping built in
for (let i = 0; i < pixels.length; i += 4) {
    pixels[i]     = /* R */;
    pixels[i + 1] = /* G */;
    pixels[i + 2] = /* B */;
    pixels[i + 3] = /* A */;
}
ctx.putImageData(imageData, 0, 0);
```

### Typed Array Selection Guide

| Type | Bytes | Range | Use Case |
|------|-------|-------|----------|
| `Uint8ClampedArray` | 1 | 0-255 | Pixel data (canvas ImageData) |
| `Int16Array` | 2 | -32768 to 32767 | Tile map indices, audio samples |
| `Uint16Array` | 2 | 0-65535 | Color values (5-6-5 format) |
| `Float32Array` | 4 | ~1.2e-38 to ~3.4e38 | Positions, velocities, transforms |
| `Float64Array` | 8 | ~5e-324 to ~1.8e308 | When Float32 precision is insufficient |

### String Performance

```javascript
// BAD — concatenation in a loop creates O(n) intermediate strings
let log = '';
for (const entry of entries) {
    log += entry.toString() + '\n'; // New string each iteration
}

// GOOD — collect in array, join once
const parts = [];
for (const entry of entries) {
    parts.push(entry.toString());
}
const log = parts.join('\n');

// BEST for known, small sets — template literal (single allocation)
const status = `HP: ${player.hp}/${player.maxHp} | Score: ${score}`;
```

### Loop Optimization

```javascript
// Cache array length (minor, but consistent)
for (let i = 0, len = enemies.length; i < len; i++) {
    enemies[i].update(dt);
}

// Reverse loop for safe removal
for (let i = bullets.length - 1; i >= 0; i--) {
    if (!bullets[i].active) {
        bullets[i] = bullets[bullets.length - 1]; // Swap with last
        bullets.pop();                              // Remove last (O(1))
    }
}

// Avoid forEach in hot paths — for loop has less overhead
// forEach creates a closure per iteration and cannot break early
```

---

## 7. DOM Interaction

### querySelector Best Practices

```javascript
// Cache elements once at initialization — never query in the game loop
const canvas = document.querySelector('#game-canvas');
const ctx = canvas.getContext('2d');
const scoreDisplay = document.querySelector('#score');
const healthBar = document.querySelector('#health-bar');

// Scope queries to parent elements when possible
const ui = document.querySelector('#game-ui');
const pauseButton = ui.querySelector('.pause-btn');

// querySelectorAll returns a static NodeList — cache it
const menuItems = document.querySelectorAll('.menu-item');
```

### Event Delegation

Attach one listener to a parent instead of N listeners to N children. Mandatory for dynamic content (menus, inventories, etc.).

```javascript
// BAD — one listener per button
document.querySelectorAll('.menu-item').forEach(item => {
    item.addEventListener('click', handleClick);
});

// GOOD — one listener on the container
document.querySelector('#menu').addEventListener('click', (e) => {
    const item = e.target.closest('.menu-item');
    if (!item) return; // Click wasn't on a menu item

    const action = item.dataset.action;
    if (action === 'start') startGame();
    else if (action === 'options') showOptions();
    else if (action === 'quit') quitGame();
});
```

### Avoiding Layout Thrashing

Layout thrashing occurs when you interleave DOM reads and writes, forcing the browser to recalculate layout multiple times per frame.

```javascript
// BAD — read-write-read-write forces 2 reflows
const height1 = element1.offsetHeight; // Read (triggers layout)
element1.style.height = '100px';       // Write (invalidates layout)
const height2 = element2.offsetHeight; // Read (forces re-layout!)
element2.style.height = '100px';       // Write

// GOOD — batch reads, then batch writes
const height1 = element1.offsetHeight; // Read
const height2 = element2.offsetHeight; // Read (same layout, no reflow)
element1.style.height = '100px';       // Write
element2.style.height = '100px';       // Write

// Properties that trigger forced reflow when read:
// offsetTop/Left/Width/Height, clientTop/Left/Width/Height,
// scrollTop/Left/Width/Height, getComputedStyle(),
// getBoundingClientRect()
```

### Canvas-Specific DOM Guidance

For canvas games, DOM interaction should be minimal and confined to:
- Initialization (set up canvas, UI overlays)
- State transitions (menu screens, game over)
- HUD updates (batch to once per frame, not per entity)

Never update DOM elements from within the entity update loop. Accumulate changes and apply once per frame.

```javascript
// BAD — updates DOM for every enemy killed
function killEnemy(enemy) {
    score += enemy.points;
    scoreDisplay.textContent = `Score: ${score}`; // DOM write in hot path!
}

// GOOD — dirty flag, update DOM once per frame
let scoreDirty = false;
function killEnemy(enemy) {
    score += enemy.points;
    scoreDirty = true;
}

function updateHUD() {
    if (scoreDirty) {
        scoreDisplay.textContent = `Score: ${score}`;
        scoreDirty = false;
    }
}
// Call updateHUD() once per frame, after all game logic
```

---

## 8. Documentation — JSDoc

### Module-Level Headers

Every source file should have a brief header comment stating its purpose, dependencies, and consumers. This lets AI (and humans) understand the file's role without reading the implementation. Keep it to 3-6 lines.

```javascript
/**
 * resources.js -- Resource generation, storage, and consumption
 *
 * Depends on: state.js (G namespace), config.js (balance values)
 * Used by: buildings.js, economy.js, render.js
 *
 * Exports: initResources(), tickResources(dt), getResourceRate()
 */
```

Update the header when dependencies or exports change. See `code-architecture-for-ai.md` Section 3 for the rationale.

### Why JSDoc for Vanilla JS [verified in Phase 1]

Without TypeScript, JSDoc is the only way to get editor IntelliSense, type checking (via `// @ts-check`), and generated documentation. VS Code fully supports JSDoc annotations for autocompletion, hover info, and type error detection.

### Core Tags

```javascript
/**
 * Calculate damage after applying armor reduction.
 *
 * @param {number} rawDamage - Base damage before mitigation
 * @param {number} armor - Target's armor value (0-100)
 * @param {number} [penetration=0] - Armor penetration percentage
 * @returns {number} Final damage value, minimum 1
 *
 * @example
 * calculateDamage(50, 30);       // 35
 * calculateDamage(50, 30, 0.5);  // 42.5
 */
function calculateDamage(rawDamage, armor, penetration = 0) {
    const effectiveArmor = armor * (1 - penetration);
    const reduction = effectiveArmor / 100;
    return Math.max(1, rawDamage * (1 - reduction));
}
```

### Custom Type Definitions

```javascript
/**
 * @typedef {Object} Vector2
 * @property {number} x - Horizontal component
 * @property {number} y - Vertical component
 */

/**
 * @typedef {Object} Entity
 * @property {number} id - Unique identifier
 * @property {Vector2} position - World position
 * @property {Vector2} velocity - Current velocity
 * @property {boolean} active - Whether entity is in play
 * @property {string} type - Entity type identifier
 */

/**
 * @typedef {'idle'|'running'|'jumping'|'falling'|'dead'} PlayerState
 */

/**
 * @typedef {Object} GameConfig
 * @property {number} width - Canvas width in pixels
 * @property {number} height - Canvas height in pixels
 * @property {number} [targetFPS=60] - Target frame rate
 * @property {boolean} [debug=false] - Enable debug rendering
 */
```

### Enum Pattern with JSDoc

```javascript
/**
 * Game state enumeration.
 * @readonly
 * @enum {string}
 */
const GameState = Object.freeze({
    MENU: 'menu',
    PLAYING: 'playing',
    PAUSED: 'paused',
    GAME_OVER: 'gameOver'
});
```

### Documenting Classes

```javascript
/**
 * Manages a pool of reusable particle objects.
 *
 * @class
 * @param {number} maxParticles - Maximum pool capacity
 */
class ParticleEmitter {
    /** @type {Particle[]} */
    #particles;

    /** @type {number} */
    #activeCount = 0;

    /**
     * Emit particles from a point.
     *
     * @param {number} x - Emission origin X
     * @param {number} y - Emission origin Y
     * @param {number} count - Number of particles to emit
     * @param {Object} [options]
     * @param {number} [options.speed=100] - Initial speed
     * @param {number} [options.spread=Math.PI*2] - Angular spread in radians
     * @returns {number} Actual number emitted (may be less if pool exhausted)
     */
    emit(x, y, count, options = {}) {
        // ...
    }
}
```

### Enabling Type Checking Without TypeScript

Add this comment at the top of your JavaScript file to get TypeScript-style type checking in VS Code:

```javascript
// @ts-check
'use strict';
```

This catches type mismatches, missing properties, wrong argument types, and more — all from JSDoc annotations alone. No build step required.

---

## 9. Security

### innerHTML: Never Use with Untrusted Data

`innerHTML` parses and executes HTML, including inline event handlers. For a game, this is primarily a risk when displaying player names, high scores, or any user-supplied text.

```javascript
// DANGEROUS — XSS vector
scoreBoard.innerHTML = `<div>${playerName}</div>`; // If playerName contains <script>...

// SAFE — textContent treats everything as literal text
scoreBoard.textContent = playerName; // HTML entities are escaped

// When you need to build DOM structure safely
function createScoreEntry(name, score) {
    const div = document.createElement('div');
    div.className = 'score-entry';

    const nameSpan = document.createElement('span');
    nameSpan.textContent = name; // Safe — no HTML parsing

    const scoreSpan = document.createElement('span');
    scoreSpan.textContent = String(score);

    div.append(nameSpan, scoreSpan);
    return div;
}
```

### Text Sanitization (When innerHTML Is Unavoidable)

```javascript
/**
 * Escape HTML special characters for safe insertion.
 * @param {string} text - Untrusted text input
 * @returns {string} HTML-safe string
 */
function escapeHTML(text) {
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
}

// Or the manual approach (faster, no DOM allocation)
function escapeHTML(text) {
    return text
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;')
        .replace(/"/g, '&quot;')
        .replace(/'/g, '&#039;');
}
```

### localStorage Limitations and Safety [verified in Phase 3, Phase 5 — path cache ~1.2MB for Screensaver]

| Concern | Detail |
|---------|--------|
| Capacity | ~5MB per origin. Exceeding throws `QuotaExceededError`. |
| Data type | Strings only. Must `JSON.stringify`/`JSON.parse`. |
| Synchronous | Blocks the main thread. Never call in the game loop. |
| Persistence | Survives page refresh but not browser "clear data". |
| Security | Accessible to any JS on the same origin. No encryption. |
| XSS exposure | If your page has an XSS vulnerability, localStorage is fully readable. |

```javascript
/**
 * Safe localStorage wrapper with size and parse guards.
 */
const Storage = Object.freeze({
    /**
     * @param {string} key
     * @param {*} value - Will be JSON-serialized
     * @returns {boolean} True if save succeeded
     */
    save(key, value) {
        try {
            const data = JSON.stringify(value);
            localStorage.setItem(key, data);
            return true;
        } catch (err) {
            if (err.name === 'QuotaExceededError') {
                console.warn('[Storage] Quota exceeded for key:', key);
            } else {
                console.error('[Storage] Save failed:', err);
            }
            return false;
        }
    },

    /**
     * @param {string} key
     * @param {*} [fallback=null] - Returned if key is missing or parse fails
     * @returns {*} Parsed value or fallback
     */
    load(key, fallback = null) {
        try {
            const raw = localStorage.getItem(key);
            if (raw === null) return fallback;
            return JSON.parse(raw);
        } catch (err) {
            console.warn('[Storage] Parse failed for key:', key, err);
            return fallback;
        }
    },

    /** @param {string} key */
    remove(key) {
        localStorage.removeItem(key);
    }
});
```

### Content Security Policy (CSP) for Game Pages

If you serve the game from a web server, add a CSP header or meta tag to prevent script injection:

```html
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';">
```

For local `file://` usage, CSP is not applicable, but the attack surface is inherently smaller.

### Never Use eval() or new Function()

These execute arbitrary strings as code. There is no legitimate use case in a game.

---

## 10. Accessibility for Canvas Games

Canvas is inherently inaccessible — screen readers cannot see its contents. Accessibility for canvas games requires deliberate, additive work.

### Minimum Viable Accessibility

```html
<!-- Semantic role and label for the canvas -->
<canvas id="game-canvas"
        role="img"
        aria-label="Game: Space Defenders - Use arrow keys to move, space to shoot"
        tabindex="0">
    <!-- Fallback content for screen readers and no-JS -->
    <p>Space Defenders requires a modern browser with JavaScript enabled.</p>
</canvas>

<!-- Live region for dynamic game state announcements -->
<div id="game-announcements"
     role="status"
     aria-live="polite"
     aria-atomic="true"
     class="sr-only">
</div>
```

### Screen Reader Announcements

```javascript
const announcer = document.querySelector('#game-announcements');

/**
 * Announce game events to screen readers.
 * @param {string} message - Text to announce
 * @param {'polite'|'assertive'} [priority='polite']
 */
function announce(message, priority = 'polite') {
    announcer.setAttribute('aria-live', priority);
    announcer.textContent = message;
}

// Usage
announce('Wave 3 starting — 15 enemies');
announce('Game Over. Final score: 12500');
announce('Player hit! 2 lives remaining', 'assertive');
```

### Screen-Reader-Only CSS

```css
.sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
}
```

### Keyboard Navigation

```javascript
// All game controls must work with keyboard
// Provide visible focus indicators for menu navigation

canvas.addEventListener('keydown', (e) => {
    // Prevent browser scroll on arrow keys / space when canvas is focused
    if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight', 'Space'].includes(e.code)) {
        e.preventDefault();
    }
});

// Menu navigation pattern
function handleMenuKeyboard(e) {
    switch (e.code) {
        case 'ArrowUp':
            selectedIndex = Math.max(0, selectedIndex - 1);
            announce(menuItems[selectedIndex].label);
            break;
        case 'ArrowDown':
            selectedIndex = Math.min(menuItems.length - 1, selectedIndex + 1);
            announce(menuItems[selectedIndex].label);
            break;
        case 'Enter':
        case 'Space':
            menuItems[selectedIndex].action();
            break;
        case 'Escape':
            if (gameState === GameState.PAUSED) resumeGame();
            break;
    }
}
```

### Reduced Motion

```javascript
// Respect user's motion preferences
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

if (prefersReducedMotion) {
    // Disable screen shake, reduce particle effects, simplify animations
    config.screenShake = false;
    config.particleMultiplier = 0.1;
    config.flashEffects = false;
}
```

### Color Contrast and Colorblind Modes

```javascript
// Provide alternative color palettes
const COLOR_SCHEMES = Object.freeze({
    default: { enemy: '#FF0000', ally: '#00FF00', neutral: '#0000FF' },
    deuteranopia: { enemy: '#FF6600', ally: '#0066FF', neutral: '#FFFF00' },
    protanopia: { enemy: '#FFD700', ally: '#00BFFF', neutral: '#FF69B4' },
    highContrast: { enemy: '#FFFFFF', ally: '#00FF00', neutral: '#FFFF00' }
});
```

---

## 11. Testing Vanilla JS Without a Build Step

### Approach: Inline Test Runner

For a no-dependency, no-npm project, build a minimal test runner directly into the project. Run tests by opening a test HTML page or appending `?test=true` to the game URL.

```javascript
/**
 * Minimal test runner — no dependencies.
 * Runs in the browser console or a dedicated test page.
 */
const TestRunner = (function() {
    let passed = 0;
    let failed = 0;
    let currentSuite = '';
    const failures = [];

    /**
     * Define a test suite.
     * @param {string} name - Suite description
     * @param {Function} fn - Test definitions
     */
    function describe(name, fn) {
        currentSuite = name;
        console.group(`Suite: ${name}`);
        fn();
        console.groupEnd();
    }

    /**
     * Define a single test.
     * @param {string} name - Test description
     * @param {Function} fn - Test body — should throw on failure
     */
    function it(name, fn) {
        try {
            fn();
            passed++;
            console.log(`  PASS: ${name}`);
        } catch (err) {
            failed++;
            const detail = `[${currentSuite}] ${name}: ${err.message}`;
            failures.push(detail);
            console.error(`  FAIL: ${name}`, err);
        }
    }

    /**
     * Assertion library.
     * @param {*} actual
     * @returns {Object} Chainable assertion methods
     */
    function expect(actual) {
        return {
            toBe(expected) {
                if (actual !== expected) {
                    throw new Error(`Expected ${expected}, got ${actual}`);
                }
            },
            toEqual(expected) {
                const a = JSON.stringify(actual);
                const b = JSON.stringify(expected);
                if (a !== b) {
                    throw new Error(`Expected ${b}, got ${a}`);
                }
            },
            toBeCloseTo(expected, epsilon = 0.001) {
                if (Math.abs(actual - expected) > epsilon) {
                    throw new Error(
                        `Expected ~${expected} (within ${epsilon}), got ${actual}`
                    );
                }
            },
            toBeTrue() {
                if (actual !== true) {
                    throw new Error(`Expected true, got ${actual}`);
                }
            },
            toBeFalse() {
                if (actual !== false) {
                    throw new Error(`Expected false, got ${actual}`);
                }
            },
            toThrow() {
                if (typeof actual !== 'function') {
                    throw new Error('Expected a function');
                }
                let threw = false;
                try { actual(); } catch (e) { threw = true; }
                if (!threw) {
                    throw new Error('Expected function to throw');
                }
            },
            toBeInstanceOf(cls) {
                if (!(actual instanceof cls)) {
                    throw new Error(
                        `Expected instance of ${cls.name}, got ${actual?.constructor?.name}`
                    );
                }
            },
            toContain(item) {
                if (!actual.includes(item)) {
                    throw new Error(`Expected [${actual}] to contain ${item}`);
                }
            }
        };
    }

    /** Print summary and return exit-code-friendly boolean. */
    function summarize() {
        console.log('');
        console.log(`Results: ${passed} passed, ${failed} failed`);
        if (failures.length > 0) {
            console.group('Failures:');
            failures.forEach(f => console.error(f));
            console.groupEnd();
        }
        return failed === 0;
    }

    return Object.freeze({ describe, it, expect, summarize });
})();
```

### Test File Structure

```html
<!-- test.html — open in browser to run tests -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Game Tests</title>
</head>
<body>
    <h1>Test Results (check console)</h1>
    <pre id="output"></pre>

    <!-- Load the game source (exposes testable functions) -->
    <script src="game.js"></script>

    <!-- Load the test runner -->
    <script src="test-runner.js"></script>

    <!-- Load test suites -->
    <script src="tests/test-math.js"></script>
    <script src="tests/test-collision.js"></script>
    <script src="tests/test-pool.js"></script>

    <!-- Run and display -->
    <script>
        const allPassed = TestRunner.summarize();
        document.querySelector('#output').textContent =
            allPassed ? 'ALL TESTS PASSED' : 'SOME TESTS FAILED — see console';
    </script>
</body>
</html>
```

### Example Test Suite

```javascript
// tests/test-collision.js
const { describe, it, expect } = TestRunner;

describe('AABB Collision', () => {
    it('should detect overlapping rectangles', () => {
        const a = { x: 0, y: 0, w: 10, h: 10 };
        const b = { x: 5, y: 5, w: 10, h: 10 };
        expect(checkAABB(a, b)).toBeTrue();
    });

    it('should not detect separated rectangles', () => {
        const a = { x: 0, y: 0, w: 10, h: 10 };
        const b = { x: 20, y: 20, w: 10, h: 10 };
        expect(checkAABB(a, b)).toBeFalse();
    });

    it('should detect edge-touching as collision', () => {
        const a = { x: 0, y: 0, w: 10, h: 10 };
        const b = { x: 10, y: 0, w: 10, h: 10 };
        // This depends on your collision function's boundary handling
        expect(checkAABB(a, b)).toBeTrue();
    });
});
```

### Testing Strategies for Game Code

| What to Test | How |
|-------------|-----|
| Pure math (vectors, collision, interpolation) | Direct unit tests — deterministic, no DOM needed |
| Object pool (acquire/release/capacity) | Unit tests with count assertions |
| State machine transitions | Test each valid transition, verify invalid ones throw |
| Save/load serialization | Roundtrip test: save, load, deep compare |
| Game config validation | Test defaults, overrides, invalid values |
| Input mapping | Simulate keydown/keyup events, verify state |

### Conditional Test Loading

```javascript
// In game.js — expose internals for testing without polluting production
if (typeof window !== 'undefined' && new URLSearchParams(window.location.search).has('test')) {
    window.__TEST_INTERNALS__ = {
        checkAABB,
        ObjectPool,
        calculateDamage,
        // ... other functions to test
    };
}
```

---

## 12. Miscellaneous Best Practices

### Const Correctness for Config Objects

```javascript
// Object.freeze for shallow immutability of config
const CONFIG = Object.freeze({
    CANVAS_WIDTH: 800,
    CANVAS_HEIGHT: 600,
    TARGET_FPS: 60,
    GRAVITY: 980,
    MAX_ENTITIES: 500,
    DEBUG: false
});

// For nested config, use a deep freeze utility
function deepFreeze(obj) {
    Object.freeze(obj);
    for (const value of Object.values(obj)) {
        if (value && typeof value === 'object' && !Object.isFrozen(value)) {
            deepFreeze(value);
        }
    }
    return obj;
}
```

### Defensive Coding Patterns

```javascript
// Guard clauses — exit early, avoid deep nesting
function takeDamage(entity, amount) {
    if (!entity) return;
    if (entity.invulnerable) return;
    if (amount <= 0) return;

    entity.health -= amount;
    if (entity.health <= 0) {
        entity.health = 0;
        entity.alive = false;
    }
}

// Enum validation
function setState(newState) {
    if (!Object.values(GameState).includes(newState)) {
        throw new GameError(`Invalid state: ${newState}`);
    }
    state = newState;
}
```

### Temporal Patterns

```javascript
// Performance.now() for high-resolution timing (not Date.now())
const start = performance.now();
// ... work ...
const elapsed = performance.now() - start;
console.log(`Operation took ${elapsed.toFixed(2)}ms`);

// Debounce for resize handlers
function debounce(fn, ms) {
    let timer;
    return function(...args) {
        clearTimeout(timer);
        timer = setTimeout(() => fn.apply(this, args), ms);
    };
}

window.addEventListener('resize', debounce(() => {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
}, 150));
```

### Console Debugging Levels

```javascript
/**
 * Lightweight logger with level control.
 * Set LOG_LEVEL to filter output. Does not allocate if level is filtered.
 */
const LOG_LEVEL = Object.freeze({
    NONE: 0, ERROR: 1, WARN: 2, INFO: 3, DEBUG: 4
});

let currentLogLevel = LOG_LEVEL.INFO;

const Log = Object.freeze({
    error(...args) { if (currentLogLevel >= LOG_LEVEL.ERROR) console.error('[ERROR]', ...args); },
    warn(...args)  { if (currentLogLevel >= LOG_LEVEL.WARN)  console.warn('[WARN]', ...args); },
    info(...args)  { if (currentLogLevel >= LOG_LEVEL.INFO)  console.info('[INFO]', ...args); },
    debug(...args) { if (currentLogLevel >= LOG_LEVEL.DEBUG) console.log('[DEBUG]', ...args); },

    /** @param {number} level - Use LOG_LEVEL constants */
    setLevel(level) { currentLogLevel = level; }
});
```

---

## Quick Reference: Feature Support Baseline

Minimum browser versions for all features in this document:

| Browser | Minimum Version | Release Date |
|---------|----------------|--------------|
| Chrome | 98+ | Feb 2022 |
| Firefox | 94+ | Nov 2021 |
| Safari | 16.4+ | Mar 2023 |
| Edge | 98+ | Feb 2022 |

If you need to support Safari 15 or older, avoid: private class fields, `structuredClone`, `at()`, `Error.cause`, `Object.hasOwn()`. Optional chaining and nullish coalescing are safe back to Safari 13.1 (2020).

---

## Sources

- [JavaScript in 2025: Roadmap to Modern Features](https://dev.to/hreuven/javascript-in-2025-your-roadmap-to-modern-features-31ff)
- [ECMAScript Support — Browser Compatibility Table](https://gist.github.com/Julien-Marcou/156b19aea4704e1d2f48adafc6e2acbf)
- [Can I Use — ES Feature Support](https://caniuse.com/?search=es20)
- [ES2024 Features You Can Use Now](https://www.infoworld.com/article/2336858/ecmascript-2024-features-you-can-use-now.html)
- [ES2025 Features in Production](https://medium.com/@mernstackdevbykevin/es2025-javascript-features-you-can-start-using-right-now-fed3784298b4)
- [JavaScript Object Pooling — VNGRS](https://medium.com/vngrs/javascript-object-pooling-47d888e1e2bf)
- [Object Pooling in Games](https://dev.to/patrocinioluisf/maximizing-memory-management-object-pooling-in-games-6bg)
- [JavaScript Game Development Core Techniques 2025](https://playgama.com/blog/general/javascript-game-development-core-techniques-for-browser-based-games/)
- [JavaScript Performance Optimization 2026](https://www.landskill.com/blog/javascript-performance-optimization/)
- [MDN — Strict Mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode)
- [JSDoc Getting Started](https://jsdoc.app/about-getting-started)
- [JSDoc Comprehensive Guide](https://medium.com/@uomroshan/a-comprehensive-guide-to-jsdoc-comments-in-javascript-ed14df2351ef)
- [TypeScript JSDoc Reference](https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html)
- [WordPress JavaScript Documentation Standards](https://developer.wordpress.org/coding-standards/inline-documentation-standards/javascript/)
- [OWASP HTML5 Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html)
- [innerHTML XSS Security Risks](https://docs.bswen.com/blog/2026-02-28-innerhtml-xss-security-risks/)
- [Securing Web Storage Best Practices](https://dev.to/rigalpatel001/securing-web-storage-localstorage-and-sessionstorage-best-practices-f00)
- [Secure Coding in JavaScript — Stack Overflow](https://stackoverflow.blog/2025/10/15/secure-coding-in-javascript/)
- [MDN — ARIA](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA)
- [ARIA and HTML — web.dev](https://web.dev/learn/accessibility/aria-html)
- [Canvas Accessibility — W3C](https://www.w3.org/Talks/2014/0510-canvas-a11y/)
- [Patterns for Efficient DOM Manipulation](https://blog.logrocket.com/patterns-efficient-dom-manipulation-vanilla-javascript/)
- [Avoiding Layout Thrashing — DebugBear](https://www.debugbear.com/blog/forced-reflows)
- [Event Delegation Internals](https://blog.logrocket.com/deep-internals-event-delegation/)
- [Testing JavaScript Without a Framework](https://alexwlchan.net/2023/testing-javascript-without-a-framework/)
- [Unit Testing with Vanilla JavaScript](https://dev.to/aurelkurtula/unit-testing-with-vanilla-javascript-the-very-basics-7jm)
- [MDN — Typed Arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Typed_arrays)
- [Float32Array — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Float32Array)
- [Typed Arrays in High-Performance JavaScript](https://egghead.io/lessons/javascript-typed-arrays-in-high-performance-javascript)
- [Vanilla JS Comeback 2025](https://devtechinsights.com/vanilla-javascript-comeback-2025/)
- [Why Developers Are Ditching Frameworks](https://thenewstack.io/why-developers-are-ditching-frameworks-for-vanilla-javascript/)
