# HTML5 Canvas Game Development — Best Practices Reference

Comprehensive patterns and practices for building performant Canvas 2D games in vanilla JavaScript. No frameworks or engines assumed. Current as of 2025-2026.

---

## 1. Game Loop Patterns

### The Core Loop: requestAnimationFrame + Fixed Timestep [verified in Phase 1]

`requestAnimationFrame` (rAF) is the only correct way to drive a game loop in the browser. It synchronizes with the display's refresh rate (~60Hz = ~16.67ms, ~120Hz = ~8.33ms), throttles inactive tabs, and yields smoother results than any timer-based approach.

**Schedule rAF early** — call it at the top of your loop function, before doing any work. This ensures the browser gets the request in time even if the current frame overruns.

```javascript
// === Fixed Timestep Game Loop with Interpolation ===

const TIMESTEP = 1000 / 60; // Fixed 60 updates/sec regardless of display Hz
let lastFrameTime = 0;
let accumulator = 0;
let frameId = 0;

function mainLoop(timestamp) {
    frameId = requestAnimationFrame(mainLoop); // Schedule early

    const frameTime = timestamp - lastFrameTime;
    lastFrameTime = timestamp;

    // Clamp to prevent spiral of death (e.g., returning from background tab)
    accumulator += Math.min(frameTime, 1000);

    // Fixed-step update: deterministic, frame-rate-independent
    let updateCount = 0;
    while (accumulator >= TIMESTEP) {
        update(TIMESTEP / 1000); // Pass dt in seconds
        accumulator -= TIMESTEP;
        if (++updateCount >= 240) {
            // Panic: discard remaining time rather than locking up
            accumulator = 0;
            break;
        }
    }

    // Interpolation factor for smooth rendering between updates
    const alpha = accumulator / TIMESTEP;
    render(alpha);
}

// Bootstrap — use a throwaway first frame to initialize timing
function start() {
    frameId = requestAnimationFrame((timestamp) => {
        lastFrameTime = timestamp;
        frameId = requestAnimationFrame(mainLoop);
    });
}

function stop() {
    cancelAnimationFrame(frameId);
}
```

### Why Fixed Timestep Matters

- **Deterministic physics**: Same inputs always produce same outputs. Enables replays.
- **Frame-rate independence**: A 120Hz display runs `update()` the same number of times per second as a 60Hz display; it just renders more interpolated frames.
- **Stability**: Variable timesteps cause physics explosions with large dt values.

### Interpolation for Smooth Rendering

Store the previous state alongside the current state. In `render()`, blend between them using the alpha (remainder) factor:

```javascript
function render(alpha) {
    for (const entity of entities) {
        const drawX = entity.prevX + (entity.x - entity.prevX) * alpha;
        const drawY = entity.prevY + (entity.y - entity.prevY) * alpha;
        ctx.drawImage(entity.sprite, drawX | 0, drawY | 0);
    }
}

function update(dt) {
    for (const entity of entities) {
        entity.prevX = entity.x;
        entity.prevY = entity.y;
        entity.x += entity.vx * dt;
        entity.y += entity.vy * dt;
    }
}
```

### FPS Monitoring (Exponential Moving Average) [verified in Phase 3]

```javascript
let fps = 60;
let framesThisSecond = 0;
let lastFpsUpdate = 0;

// Inside mainLoop:
if (timestamp > lastFpsUpdate + 1000) {
    fps = 0.25 * framesThisSecond + 0.75 * fps; // Smoothed
    lastFpsUpdate = timestamp;
    framesThisSecond = 0;
}
framesThisSecond++;
```

---

## 2. Canvas Rendering Performance

### Disable Alpha When Unused [verified in Phase 1, Phase 7 — alpha:false for opaque layers, alpha:true for sparse overlays]

If the canvas never needs transparency (most games with solid backgrounds):

```javascript
const ctx = canvas.getContext('2d', { alpha: false });
```

This tells the browser to skip compositing with the page background — measurable speedup.

### GPU Compositing Hint [verified in Phase 7 — will-change on canvas for large boards]

Use `will-change: transform` on the canvas element to promote it to its own GPU-backed compositing layer. Reduces browser compositing overhead on large canvases (e.g., 1920x1080):

```css
canvas { will-change: transform; }
```

Only use this when the canvas is large enough for compositing to be a bottleneck. On small canvases, the extra GPU memory allocation is wasted.

### Pixel-Aligned Rendering

Sub-pixel coordinates force anti-aliasing calculations. Always snap to integers:

```javascript
// Use bitwise OR for fast truncation (works for positive values)
ctx.drawImage(sprite, x | 0, y | 0);

// Or Math.floor for coordinates that may be negative
ctx.drawImage(sprite, Math.floor(x), Math.floor(y));

// Bitwise double-NOT is another fast truncation option
ctx.drawImage(sprite, ~~x, ~~y);
```

### Batch Drawing Operations

Reduce the number of draw calls by batching same-type operations:

```javascript
// BAD: separate path per line
for (let i = 0; i < points.length - 1; i++) {
    ctx.beginPath();
    ctx.moveTo(points[i].x, points[i].y);
    ctx.lineTo(points[i + 1].x, points[i + 1].y);
    ctx.stroke();
}

// GOOD: single path for all lines
ctx.beginPath();
for (let i = 0; i < points.length - 1; i++) {
    ctx.moveTo(points[i].x, points[i].y);
    ctx.lineTo(points[i + 1].x, points[i + 1].y);
}
ctx.stroke();
```

### Minimize State Changes

Group draw calls by shared state (fill color, stroke style, etc.):

```javascript
// BAD: alternating styles
for (let i = 0; i < STRIPES; i++) {
    ctx.fillStyle = (i % 2) ? COLOR_A : COLOR_B;
    ctx.fillRect(i * W, 0, W, H);
}

// GOOD: group by style
ctx.fillStyle = COLOR_A;
for (let i = 0; i < STRIPES; i += 2) {
    ctx.fillRect(i * W, 0, W, H);
}
ctx.fillStyle = COLOR_B;
for (let i = 1; i < STRIPES; i += 2) {
    ctx.fillRect(i * W, 0, W, H);
}
```

### Offscreen Canvas (Pre-rendering) [verified in Phase 1, Phase 7 — grid cache + food cache with dirty flags]

Render complex or repeated graphics once to a hidden canvas, then blit with `drawImage`:

```javascript
function createOffscreenSprite(drawFn, width, height) {
    const offscreen = document.createElement('canvas');
    offscreen.width = width;
    offscreen.height = height;
    const offCtx = offscreen.getContext('2d');
    drawFn(offCtx, width, height);
    return offscreen; // Use as drawImage source
}

// Pre-render a complex shape once
const cachedStar = createOffscreenSprite((ctx, w, h) => {
    // ... complex star drawing code ...
}, 64, 64);

// In render loop: fast blit
ctx.drawImage(cachedStar, x | 0, y | 0);
```

**Text rendering** is especially expensive. Pre-render text labels to offscreen canvases and blit them. This alone can drop text render times from ~10ms to ~1ms.

### Layered Canvases

Separate elements by update frequency. Stack multiple canvases with CSS positioning:

```html
<div id="game-container" style="position: relative;">
    <canvas id="bg-layer"   style="position: absolute; z-index: 0;"></canvas>
    <canvas id="game-layer" style="position: absolute; z-index: 1;"></canvas>
    <canvas id="ui-layer"   style="position: absolute; z-index: 2;"></canvas>
</div>
```

- **Background layer**: Drawn once or rarely (sky, terrain)
- **Game layer**: Redrawn every frame (entities, projectiles)
- **UI layer**: Redrawn on change only (score, health)

### Dirty Rectangles

Instead of `clearRect(0, 0, W, H)` every frame, track what changed and clear/redraw only those regions:

```javascript
function clearRegion(ctx, rect) {
    ctx.clearRect(rect.x, rect.y, rect.w, rect.h);
}

// Track each entity's previous bounding box
// Clear old position, draw at new position
for (const entity of entities) {
    clearRegion(gameCtx, entity.prevBounds);
    entity.draw(gameCtx);
    entity.prevBounds = entity.getBounds();
}
```

Best suited for games with relatively few moving objects against a static background.

### Things That Are Expensive — Avoid in Hot Loops

- `shadowBlur` / `shadowColor` — extremely expensive, pre-render shadows to offscreen canvas
- `ctx.fillText()` / `ctx.strokeText()` — pre-render to offscreen canvas or use sprite font
- `ctx.save()` / `ctx.restore()` — avoid when you can manually track/reset state
- `createLinearGradient()` / `createRadialGradient()` — cache gradient objects, or pre-render to image
- Scaling images in `drawImage()` — pre-cache scaled versions

---

## 3. Efficient Data Structures

### Object Pooling

Avoid allocations in the hot path. Pre-allocate objects and recycle them:

```javascript
class Pool {
    constructor(factory, initialSize = 100) {
        this._factory = factory;
        this._pool = [];
        for (let i = 0; i < initialSize; i++) {
            this._pool.push(factory());
        }
    }

    acquire() {
        return this._pool.length > 0 ? this._pool.pop() : this._factory();
    }

    release(obj) {
        this._pool.push(obj);
    }

    // Bulk release
    releaseAll(arr) {
        for (let i = arr.length - 1; i >= 0; i--) {
            this._pool.push(arr[i]);
        }
        arr.length = 0;
    }
}

// Usage
const bulletPool = new Pool(() => ({ x: 0, y: 0, vx: 0, vy: 0, active: false }), 200);

function spawnBullet(x, y, vx, vy) {
    const b = bulletPool.acquire();
    b.x = x; b.y = y; b.vx = vx; b.vy = vy; b.active = true;
    activeBullets.push(b);
}

function recycleBullet(b, index) {
    b.active = false;
    activeBullets.splice(index, 1);
    bulletPool.release(b);
}
```

**Pool candidates**: bullets, particles, enemies, damage numbers, sound effect instances.

### Typed Arrays

Use `Float32Array` / `Float64Array` for large numerical datasets. Better cache locality, no boxing overhead, predictable memory layout:

```javascript
// Structure-of-Arrays for particle system (better cache performance)
const MAX_PARTICLES = 10000;
const particles = {
    x:    new Float32Array(MAX_PARTICLES),
    y:    new Float32Array(MAX_PARTICLES),
    vx:   new Float32Array(MAX_PARTICLES),
    vy:   new Float32Array(MAX_PARTICLES),
    life: new Float32Array(MAX_PARTICLES),
    count: 0
};

function updateParticles(dt) {
    for (let i = 0; i < particles.count; i++) {
        particles.x[i] += particles.vx[i] * dt;
        particles.y[i] += particles.vy[i] * dt;
        particles.life[i] -= dt;
    }
    // Compact: swap dead particles with last active
    for (let i = particles.count - 1; i >= 0; i--) {
        if (particles.life[i] <= 0) {
            const last = --particles.count;
            particles.x[i] = particles.x[last];
            particles.y[i] = particles.y[last];
            particles.vx[i] = particles.vx[last];
            particles.vy[i] = particles.vy[last];
            particles.life[i] = particles.life[last];
        }
    }
}
```

### Spatial Hashing for Collision Detection [verified in Phase 1]

Divide the world into a grid. Only check collisions between entities in the same (or adjacent) cells:

```javascript
class SpatialHash {
    constructor(cellSize) {
        this.cellSize = cellSize;
        this.cells = new Map();
    }

    _key(cx, cy) {
        return `${cx},${cy}`;
    }

    clear() {
        this.cells.clear();
    }

    insert(entity) {
        const cs = this.cellSize;
        const minCX = Math.floor(entity.x / cs);
        const minCY = Math.floor(entity.y / cs);
        const maxCX = Math.floor((entity.x + entity.w) / cs);
        const maxCY = Math.floor((entity.y + entity.h) / cs);

        for (let cx = minCX; cx <= maxCX; cx++) {
            for (let cy = minCY; cy <= maxCY; cy++) {
                const key = this._key(cx, cy);
                if (!this.cells.has(key)) this.cells.set(key, []);
                this.cells.get(key).push(entity);
            }
        }
    }

    query(entity) {
        const cs = this.cellSize;
        const minCX = Math.floor(entity.x / cs);
        const minCY = Math.floor(entity.y / cs);
        const maxCX = Math.floor((entity.x + entity.w) / cs);
        const maxCY = Math.floor((entity.y + entity.h) / cs);

        const found = new Set();
        for (let cx = minCX; cx <= maxCX; cx++) {
            for (let cy = minCY; cy <= maxCY; cy++) {
                const cell = this.cells.get(this._key(cx, cy));
                if (cell) {
                    for (const other of cell) {
                        if (other !== entity) found.add(other);
                    }
                }
            }
        }
        return found;
    }
}

// Usage in update loop
const spatialHash = new SpatialHash(64);

function checkCollisions(entities) {
    spatialHash.clear();
    for (const e of entities) spatialHash.insert(e);

    for (const e of entities) {
        for (const other of spatialHash.query(e)) {
            if (aabbOverlap(e, other)) {
                handleCollision(e, other);
            }
        }
    }
}
```

**Cell size guideline**: roughly 2x the average entity size. Too small = too many cells. Too large = too many entities per cell.

---

## 4. State Management

### Game State Machine

Use a state stack (pushdown automaton) for game modes. Each state owns its own `update`, `render`, and `handleInput`:

```javascript
class GameStateManager {
    constructor() {
        this.stack = [];
    }

    get current() {
        return this.stack[this.stack.length - 1] || null;
    }

    push(state) {
        if (this.current) this.current.pause?.();
        this.stack.push(state);
        state.enter?.();
    }

    pop() {
        const old = this.stack.pop();
        old?.exit?.();
        if (this.current) this.current.resume?.();
        return old;
    }

    replace(state) {
        const old = this.stack.pop();
        old?.exit?.();
        this.stack.push(state);
        state.enter?.();
    }

    update(dt) { this.current?.update(dt); }
    render(ctx, alpha) { this.current?.render(ctx, alpha); }
    handleInput(input) { this.current?.handleInput(input); }
}

// Example states
const PlayState = {
    enter() { /* init level */ },
    exit() { /* cleanup */ },
    pause() { /* optional: mute audio */ },
    resume() { /* optional: unmute */ },
    update(dt) { /* game logic */ },
    render(ctx, alpha) { /* draw game */ },
    handleInput(input) {
        if (input.escape) gameStates.push(PauseState);
    }
};

const PauseState = {
    enter() { /* show pause menu */ },
    update(dt) { /* animate menu */ },
    render(ctx, alpha) {
        // Draw play state underneath (semi-transparent overlay)
        PlayState.render(ctx, alpha);
        ctx.fillStyle = 'rgba(0,0,0,0.5)';
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        // Draw pause menu
    },
    handleInput(input) {
        if (input.escape) gameStates.pop(); // Resume
    }
};
```

### Separation of Concerns

The update loop must be the single source of truth for game state. Never modify game state in:
- Input handlers (record input, process in update)
- Render functions (read-only access to state)
- Event callbacks (queue events, process in update)

```javascript
// Input system: records state, does not act
const input = {
    keys: {},
    mouseX: 0, mouseY: 0,
    mouseDown: false,

    init(canvas) {
        window.addEventListener('keydown', (e) => { this.keys[e.code] = true; });
        window.addEventListener('keyup', (e) => { this.keys[e.code] = false; });
        canvas.addEventListener('mousemove', (e) => {
            const rect = canvas.getBoundingClientRect();
            this.mouseX = e.clientX - rect.left;
            this.mouseY = e.clientY - rect.top;
        });
        canvas.addEventListener('mousedown', () => { this.mouseDown = true; });
        canvas.addEventListener('mouseup', () => { this.mouseDown = false; });
    },

    isDown(code) { return !!this.keys[code]; }
};
```

---

## 5. Code Organization

### Module Pattern for Single-File Games

Use an IIFE or ES module to avoid polluting the global namespace. Organize with clear sections:

```javascript
const Game = (() => {
    // === CONSTANTS ===
    const CANVAS_W = 800;
    const CANVAS_H = 600;
    const TIMESTEP = 1000 / 60;

    // === STATE ===
    let score = 0;
    let entities = [];

    // === SYSTEMS ===
    const Input = { /* ... */ };
    const Physics = { /* ... */ };
    const Renderer = { /* ... */ };
    const Audio = { /* ... */ };

    // === ENTITY FACTORIES ===
    function createPlayer(x, y) { /* ... */ }
    function createEnemy(x, y, type) { /* ... */ }

    // === GAME LOOP ===
    function update(dt) { /* ... */ }
    function render(ctx, alpha) { /* ... */ }
    function mainLoop(timestamp) { /* ... */ }

    // === PUBLIC API ===
    return {
        init() { /* setup canvas, load assets, start loop */ },
        start() { /* ... */ },
        stop() { /* ... */ }
    };
})();

Game.init();
```

### Class-Based Entity Pattern

```javascript
class Entity {
    constructor(x, y, w, h) {
        this.x = x; this.y = y;
        this.w = w; this.h = h;
        this.prevX = x; this.prevY = y;
        this.vx = 0; this.vy = 0;
        this.active = true;
    }

    update(dt) {
        this.prevX = this.x;
        this.prevY = this.y;
        this.x += this.vx * dt;
        this.y += this.vy * dt;
    }

    render(ctx, alpha) {
        const drawX = (this.prevX + (this.x - this.prevX) * alpha) | 0;
        const drawY = (this.prevY + (this.y - this.prevY) * alpha) | 0;
        // Override in subclass
    }

    getBounds() {
        return { x: this.x, y: this.y, w: this.w, h: this.h };
    }
}
```

### Functional Alternative (Data-Oriented)

For simpler games, plain objects + functions can be cleaner than classes:

```javascript
// Entity is just data
function createBullet(x, y, vx, vy) {
    return { type: 'bullet', x, y, vx, vy, prevX: x, prevY: y, active: true };
}

// Systems operate on data
function updateBullets(bullets, dt) {
    for (let i = 0; i < bullets.length; i++) {
        const b = bullets[i];
        b.prevX = b.x; b.prevY = b.y;
        b.x += b.vx * dt;
        b.y += b.vy * dt;
        if (b.x < 0 || b.x > CANVAS_W || b.y < 0 || b.y > CANVAS_H) {
            b.active = false;
        }
    }
}

function renderBullets(ctx, bullets, alpha) {
    ctx.fillStyle = '#ff0';
    for (let i = 0; i < bullets.length; i++) {
        const b = bullets[i];
        if (!b.active) continue;
        const dx = (b.prevX + (b.x - b.prevX) * alpha) | 0;
        const dy = (b.prevY + (b.y - b.prevY) * alpha) | 0;
        ctx.fillRect(dx - 2, dy - 2, 4, 4);
    }
}
```

---

## 6. Color and Graphics Optimization

### Pre-Computed Color Tables [verified in Phase 4, Phase 5, Phase 7 — fillStyle change-guard skips redundant assignments]

Building color strings (`rgb(...)`, `hsl(...)`) at runtime is allocation-heavy. Pre-compute them:

```javascript
// Pre-compute a palette
const HEALTH_COLORS = [];
for (let i = 0; i <= 100; i++) {
    const r = Math.floor(255 * (1 - i / 100));
    const g = Math.floor(255 * (i / 100));
    HEALTH_COLORS[i] = `rgb(${r},${g},0)`;
}

// In render loop: zero allocation
ctx.fillStyle = HEALTH_COLORS[entity.health | 0];
```

### Gradient Caching [verified in Phase 4]

Never create gradients inside the render loop. Create once, reuse:

```javascript
// At init time
const skyGradient = ctx.createLinearGradient(0, 0, 0, CANVAS_H);
skyGradient.addColorStop(0, '#001');
skyGradient.addColorStop(1, '#114');

// In render loop — reuse the object
ctx.fillStyle = skyGradient;
ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);
```

If gradients depend on position (e.g., following an entity), pre-render the gradient to an offscreen canvas and `drawImage` it, or accept the cost if it is only used once per frame.

### Pre-Rendered Graphics via Offscreen Canvas

Complex procedural graphics (glow effects, radial patterns, text labels) should be rendered once and cached:

```javascript
function createGlowSprite(color, radius) {
    const size = radius * 2;
    const off = document.createElement('canvas');
    off.width = size;
    off.height = size;
    const c = off.getContext('2d');
    const grad = c.createRadialGradient(radius, radius, 0, radius, radius, radius);
    grad.addColorStop(0, color);
    grad.addColorStop(1, 'transparent');
    c.fillStyle = grad;
    c.fillRect(0, 0, size, size);
    return off;
}

const playerGlow = createGlowSprite('rgba(0,150,255,0.6)', 32);
// In render: ctx.drawImage(playerGlow, x - 32, y - 32);
```

---

## 7. Mobile and Touch Support

### Device Pixel Ratio Handling

Without DPR correction, canvas appears blurry on high-DPI/Retina screens:

```javascript
function setupHiDPICanvas(canvas, width, height) {
    const dpr = window.devicePixelRatio || 1;

    // Set display size (CSS)
    canvas.style.width = width + 'px';
    canvas.style.height = height + 'px';

    // Set actual resolution
    canvas.width = width * dpr;
    canvas.height = height * dpr;

    // Scale context so drawing code uses CSS-pixel coordinates
    const ctx = canvas.getContext('2d');
    ctx.scale(dpr, dpr);

    return ctx;
}
```

**For games with a fixed internal resolution**, use CSS transforms for scaling instead of changing the canvas resolution:

```javascript
function scaleToFit(canvas, gameWidth, gameHeight) {
    const scaleX = window.innerWidth / gameWidth;
    const scaleY = window.innerHeight / gameHeight;
    const scale = Math.min(scaleX, scaleY);

    canvas.style.transformOrigin = '0 0';
    canvas.style.transform = `scale(${scale})`;
    canvas.style.imageRendering = 'pixelated'; // For pixel-art games
}

window.addEventListener('resize', () => scaleToFit(canvas, 800, 600));
```

### Touch Event Handling

Unified input handling for mouse and touch:

```javascript
function getEventPos(canvas, e) {
    const rect = canvas.getBoundingClientRect();
    const clientX = e.touches ? e.touches[0].clientX : e.clientX;
    const clientY = e.touches ? e.touches[0].clientY : e.clientY;
    return {
        x: clientX - rect.left,
        y: clientY - rect.top
    };
}

function initInput(canvas) {
    // Prevent default touch behaviors (scrolling, zooming)
    canvas.addEventListener('touchstart', (e) => {
        e.preventDefault();
        const pos = getEventPos(canvas, e);
        input.mouseX = pos.x;
        input.mouseY = pos.y;
        input.mouseDown = true;
    }, { passive: false });

    canvas.addEventListener('touchmove', (e) => {
        e.preventDefault();
        const pos = getEventPos(canvas, e);
        input.mouseX = pos.x;
        input.mouseY = pos.y;
    }, { passive: false });

    canvas.addEventListener('touchend', (e) => {
        e.preventDefault();
        input.mouseDown = false;
    }, { passive: false });

    // Mouse events for desktop (same handler pattern)
    canvas.addEventListener('mousedown', (e) => {
        const pos = getEventPos(canvas, e);
        input.mouseX = pos.x;
        input.mouseY = pos.y;
        input.mouseDown = true;
    });

    canvas.addEventListener('mousemove', (e) => {
        const pos = getEventPos(canvas, e);
        input.mouseX = pos.x;
        input.mouseY = pos.y;
    });

    canvas.addEventListener('mouseup', () => { input.mouseDown = false; });
}
```

### Preventing Mobile Browser Behaviors

```css
/* Prevent text selection, callout menus, etc. */
canvas {
    touch-action: none;          /* Disable browser touch gestures */
    -webkit-touch-callout: none; /* iOS: no callout */
    user-select: none;           /* No text selection */
}
```

---

## 8. localStorage for Save Data

### Core Patterns

```javascript
const SaveManager = {
    SAVE_KEY: 'myGame_save',
    SETTINGS_KEY: 'myGame_settings',

    // Always wrap in try/catch — localStorage may be disabled or full
    save(gameData) {
        try {
            const data = {
                version: 2,                // Schema version for migration
                timestamp: Date.now(),
                payload: gameData
            };
            localStorage.setItem(this.SAVE_KEY, JSON.stringify(data));
            return true;
        } catch (e) {
            console.warn('Save failed:', e.message);
            return false;
        }
    },

    load() {
        try {
            const raw = localStorage.getItem(this.SAVE_KEY);
            if (!raw) return null;

            const data = JSON.parse(raw);

            // Version migration
            if (data.version < 2) {
                data.payload = this._migrateV1toV2(data.payload);
                data.version = 2;
                this.save(data.payload); // Re-save migrated data
            }

            return data.payload;
        } catch (e) {
            console.warn('Load failed:', e.message);
            return null;
        }
    },

    deleteSave() {
        try { localStorage.removeItem(this.SAVE_KEY); } catch (e) { /* */ }
    },

    // Check if localStorage is available at all
    isAvailable() {
        try {
            const test = '__storage_test__';
            localStorage.setItem(test, test);
            localStorage.removeItem(test);
            return true;
        } catch (e) {
            return false;
        }
    },

    _migrateV1toV2(oldData) {
        // Transform old schema to new schema
        return { ...oldData, newField: 'default' };
    }
};
```

### Best Practices [verified in Phase 3]

- **Always include a schema version number.** You will change your save format; migrations are inevitable.
- **Wrap every access in try/catch.** Private browsing, disabled storage, and quota limits all throw.
- **Test with `isAvailable()` at startup.** Degrade gracefully if storage is unavailable.
- **JSON.stringify/parse handles type conversion.** Everything is stored as strings. Numbers, booleans, and objects all roundtrip correctly through JSON.
- **localStorage limit is ~5-10MB per origin.** More than enough for game saves, but don't store large assets.
- **Don't save every frame.** Save on meaningful events (level complete, checkpoint, settings change) or at intervals (auto-save every 30-60 seconds).
- **Namespace your keys** (`gameTitle_save`, `gameTitle_settings`) to avoid collisions if the same domain hosts multiple games.

### Settings Persistence [verified in Phase 3]

```javascript
const DEFAULT_SETTINGS = {
    musicVolume: 0.7,
    sfxVolume: 1.0,
    fullscreen: false,
    difficulty: 'normal'
};

function loadSettings() {
    try {
        const raw = localStorage.getItem('myGame_settings');
        if (!raw) return { ...DEFAULT_SETTINGS };
        return { ...DEFAULT_SETTINGS, ...JSON.parse(raw) }; // Merge with defaults
    } catch {
        return { ...DEFAULT_SETTINGS };
    }
}

function saveSettings(settings) {
    try {
        localStorage.setItem('myGame_settings', JSON.stringify(settings));
    } catch { /* */ }
}
```

The `{ ...DEFAULT_SETTINGS, ...saved }` pattern is critical — it ensures new settings added in future versions get their default values automatically.

---

## 9. Common Anti-Patterns

### Using setInterval/setTimeout for Game Loops

```javascript
// WRONG: not synchronized to display refresh, drifts, doesn't pause in background
setInterval(gameLoop, 1000 / 60);

// CORRECT:
requestAnimationFrame(mainLoop);
```

`setInterval` was never intended for animation. It does not sync with VSync, does not throttle in background tabs, and its timing drifts under load.

### Modifying Game State in Input Handlers

```javascript
// WRONG: directly mutating position in event handler
document.addEventListener('keydown', (e) => {
    if (e.code === 'ArrowRight') player.x += 5; // Race condition with update loop
});

// CORRECT: record input state, process in update()
document.addEventListener('keydown', (e) => { keys[e.code] = true; });
document.addEventListener('keyup', (e) => { keys[e.code] = false; });

function update(dt) {
    if (keys['ArrowRight']) player.x += player.speed * dt;
}
```

### Creating Objects in Hot Paths

```javascript
// WRONG: allocating every frame
function update() {
    const velocity = { x: player.vx, y: player.vy }; // GC pressure
    const color = `rgb(${r},${g},${b})`;               // String allocation
    const pos = new Vector2(x, y);                      // Object allocation
}

// CORRECT: reuse pre-allocated objects, pre-computed strings
const _tempVec = { x: 0, y: 0 };
function update() {
    _tempVec.x = player.vx;
    _tempVec.y = player.vy;
}
```

### Unnecessary save/restore

```javascript
// WRONG: save/restore when you only set fillStyle
ctx.save();
ctx.fillStyle = 'red';
ctx.fillRect(x, y, w, h);
ctx.restore();

// CORRECT: just reset what you changed (or don't, if the next draw sets its own style)
ctx.fillStyle = 'red';
ctx.fillRect(x, y, w, h);
```

Use `save()`/`restore()` only when applying transforms (translate, rotate, scale) or clip paths that you need to undo. For simple style changes, just set the property directly.

### DOM Manipulation in Render Loops [verified in Phase 5, Phase 5.5 — cached refs + change-guarded textContent writes]

```javascript
// WRONG: touching the DOM every frame for score display
function render() {
    document.getElementById('score').textContent = score; // Layout thrashing
    // ... canvas drawing ...
}

// CORRECT: update DOM only when value changes
function setScore(newScore) {
    if (newScore !== score) {
        score = newScore;
        scoreElement.textContent = score; // Only when changed
    }
}
```

### Variable Timestep Without Clamping [verified in Phase 1 — caught and fixed spiral-of-death bug]

```javascript
// WRONG: physics blows up when tab returns from background (dt = 30 seconds)
function update(dt) {
    entity.x += entity.vx * dt; // dt could be huge
}

// CORRECT: use fixed timestep (see Section 1), or at minimum clamp dt
const dt = Math.min(rawDt, 0.1); // Never simulate more than 100ms at once
```

### Drawing Full Canvas When Only Part Changed

```javascript
// WRONG: clearing and redrawing everything for a score change
function render() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    drawBackground();      // Hasn't changed
    drawAllEntities();     // Most haven't changed
    drawUI();              // Only score changed
}

// CORRECT: use layered canvases or dirty rectangles (see Section 2)
```

---

## 10. Testing Canvas Games

### Core Principle: Separate Logic from Rendering

The most testable canvas game is one where the game logic knows nothing about canvas. Testing canvas draw calls directly is painful and fragile. Instead, structure code so that all game logic is pure functions operating on data:

```javascript
// TESTABLE: pure logic, no canvas dependency
function applyGravity(entity, dt, gravity) {
    entity.vy += gravity * dt;
    entity.y += entity.vy * dt;
}

function checkBounds(entity, worldWidth, worldHeight) {
    return entity.x >= 0 && entity.x <= worldWidth &&
           entity.y >= 0 && entity.y <= worldHeight;
}

function resolveCollision(a, b) {
    // Returns collision result data, doesn't draw anything
    const overlapX = Math.min(a.x + a.w, b.x + b.w) - Math.max(a.x, b.x);
    const overlapY = Math.min(a.y + a.h, b.y + b.h) - Math.max(a.y, b.y);
    return { overlapX, overlapY, collided: overlapX > 0 && overlapY > 0 };
}
```

### Unit Testing Game Logic

Use any standard test runner (Jest, Vitest, etc.). No browser needed for logic tests:

```javascript
// Example test file: game-logic.test.js
describe('Physics', () => {
    test('gravity increases velocity over time', () => {
        const entity = { x: 0, y: 0, vx: 0, vy: 0 };
        applyGravity(entity, 1/60, 980);
        expect(entity.vy).toBeCloseTo(16.33, 1);
    });

    test('entity moves out of bounds', () => {
        const entity = { x: -1, y: 50, w: 10, h: 10 };
        expect(checkBounds(entity, 800, 600)).toBe(false);
    });
});

describe('Collision', () => {
    test('overlapping rectangles collide', () => {
        const a = { x: 0, y: 0, w: 10, h: 10 };
        const b = { x: 5, y: 5, w: 10, h: 10 };
        const result = resolveCollision(a, b);
        expect(result.collided).toBe(true);
        expect(result.overlapX).toBe(5);
    });

    test('separated rectangles do not collide', () => {
        const a = { x: 0, y: 0, w: 10, h: 10 };
        const b = { x: 20, y: 20, w: 10, h: 10 };
        expect(resolveCollision(a, b).collided).toBe(false);
    });
});
```

### Testing State Machines

```javascript
describe('GameStateManager', () => {
    test('push/pop lifecycle', () => {
        const manager = new GameStateManager();
        const enterSpy = jest.fn();
        const exitSpy = jest.fn();
        const state = { enter: enterSpy, exit: exitSpy, update() {}, render() {} };

        manager.push(state);
        expect(enterSpy).toHaveBeenCalledTimes(1);
        expect(manager.current).toBe(state);

        manager.pop();
        expect(exitSpy).toHaveBeenCalledTimes(1);
        expect(manager.current).toBeNull();
    });
});
```

### Snapshot Testing for Render Output (When Needed)

If you must verify rendering, `jest-canvas-mock` provides a mock Canvas context that records calls:

```javascript
// With jest-canvas-mock installed
test('player renders at correct position', () => {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    const player = { x: 100, y: 200, prevX: 100, prevY: 200 };

    renderPlayer(ctx, player, 1.0); // alpha = 1.0 (no interpolation)

    const events = ctx.__getDrawCalls();
    expect(events).toMatchSnapshot();
});
```

This is a last resort. Prefer testing logic separately.

### Integration / Visual Regression Testing

For verifying actual visual output, tools like Playwright or Cypress can screenshot the canvas and compare against baselines:

```javascript
// Playwright example
test('game renders correctly', async ({ page }) => {
    await page.goto('http://localhost:8080');
    await page.waitForTimeout(1000); // Let game initialize
    const screenshot = await page.locator('canvas').screenshot();
    expect(screenshot).toMatchSnapshot('game-start.png');
});
```

Use this sparingly — visual tests are slow and brittle. Reserve them for critical screens (title, game over, key UI states).

---

## Quick Reference: Performance Checklist

Before shipping, verify:

- [ ] Game loop uses `requestAnimationFrame`, not `setInterval`
- [ ] Fixed timestep with accumulator for deterministic updates
- [ ] Delta time clamped to prevent spiral of death
- [ ] `{ alpha: false }` on canvas context if no transparency needed
- [ ] All draw coordinates snapped to integers
- [ ] Draw calls batched by style/state
- [ ] Expensive graphics pre-rendered to offscreen canvases
- [ ] Text pre-rendered to cached canvases
- [ ] Gradients created once and reused (never in render loop)
- [ ] No object allocation in update/render hot paths
- [ ] Object pools for frequently created/destroyed entities
- [ ] `save()`/`restore()` used only when transforms or clips require it
- [ ] DOM updates only on value change, not every frame
- [ ] Device pixel ratio handled for sharp rendering on HiDPI
- [ ] Touch events handled with `{ passive: false }` and `preventDefault()`
- [ ] `touch-action: none` on canvas CSS
- [ ] localStorage wrapped in try/catch with availability check
- [ ] Save data includes schema version for future migration
- [ ] Game logic testable independently of canvas rendering

---

## Sources

- [MDN: Optimizing Canvas](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Optimizing_canvas)
- [MDN: Anatomy of a Video Game](https://developer.mozilla.org/en-US/docs/Games/Anatomy)
- [web.dev: Improving HTML5 Canvas Performance](https://web.dev/canvas-performance/)
- [web.dev: OffscreenCanvas](https://web.dev/articles/offscreen-canvas)
- [web.dev: High DPI Canvas](https://web.dev/articles/canvas-hidipi)
- [Isaac Sukin: Detailed Explanation of JavaScript Game Loops and Timing](https://isaacsukin.com/news/2015/01/detailed-explanation-javascript-game-loops-and-timing)
- [Aleksandr Hovhannisyan: Performant Game Loops in JavaScript](https://www.aleksandrhovhannisyan.com/blog/javascript-game-loop/)
- [Game Programming Patterns: State](https://gameprogrammingpatterns.com/state.html)
- [Jake Gordon: JavaScript Game Foundations](https://jakesgordon.com/writing/javascript-game-foundations-the-game-loop/)
- [SK Lambert: HTML5 Game Tutorial — Module Pattern](https://blog.sklambert.com/html5-game-tutorial-module-pattern/)
- [SK Lambert: JavaScript Object Pool](https://blog.sklambert.com/javascript-object-pool/)
- [Spicy Yoghurt: Create a Proper Game Loop](https://spicyyoghurt.com/tutorials/html5-javascript-game-development/create-a-proper-game-loop-with-requestanimationframe)
- [Spicy Yoghurt: Store Data with HTML5 Local Storage](https://spicyyoghurt.com/tutorials/javascript/store-data-html5-local-storage)
- [GameDev.net: Spatial Hashing](https://www.gamedev.net/articles/programming/general-and-gameplay-programming/spatial-hashing-r2697/)
- [Gamedev.js: Using Local Storage for High Scores](https://gamedevjs.com/articles/using-local-storage-for-high-scores-and-game-progress/)
- [AG Grid: Optimising HTML5 Canvas Rendering](https://blog.ag-grid.com/optimising-html5-canvas-rendering-best-practices-and-techniques/)
- [Nicola Hibbert: Optimising HTML5 Canvas Games](https://nicolahibbert.com/optimising-html5-canvas-games/)
- [David Matthew: Retina-Ready Responsive Canvas](https://davidmatthew.ie/the-canvas-api-part-3-a-retina-ready-responsive-canvas/)
- [Ben Centra: Using Touch Events with HTML5 Canvas](https://bencentra.com/code/2014/12/05/html5-canvas-touch-events.html)
- [Ibrahim Diallo: Creating a State Stack Engine](https://idiallo.com/blog/javascript-game-state-stack-engine)
- [Generalist Programmer: Game Optimization Complete Guide 2025](https://generalistprogrammer.com/tutorials/game-optimization-complete-performance-guide-2025)
