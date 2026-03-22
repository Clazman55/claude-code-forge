# Incremental Game Patterns — Skill Reference

Patterns for browser-based idle/incremental games. Covers game loop architecture, resource
economics, big number handling, save systems, prestige mechanics, and balance configuration.
Compiled from established community patterns (Kongregate idle game math series, break_infinity.js,
Profectus/The Modding Tree, HipHopHuman game loop guide) and cross-referenced with existing
html5-single-file-architecture, html5-multi-file-architecture, and javascript-vanilla-standards skills.

---

## Table of Contents

1. [Game Loop Architecture](#1-game-loop-architecture)
2. [Resource System](#2-resource-system)
3. [Cost Scaling Formulas](#3-cost-scaling-formulas)
4. [Big Number Handling](#4-big-number-handling)
5. [Save and Load System](#5-save-and-load-system)
6. [Offline Progress](#6-offline-progress)
7. [Prestige and Tier Mechanics](#7-prestige-and-tier-mechanics)
8. [Balance Configuration](#8-balance-configuration)
9. [UI Update Patterns](#9-ui-update-patterns)
10. [Number Formatting](#10-number-formatting)
11. [Anti-Patterns](#11-anti-patterns)
12. [Gotchas](#12-gotchas)
13. [New Incremental Game Checklist](#13-new-incremental-game-checklist)

---

## 1. Game Loop Architecture

Incremental games use a **fixed-timestep loop** with `requestAnimationFrame`. Unlike action
games that need 60fps physics, incrementals typically tick at a lower rate (e.g. 10-20 ticks/sec)
with visual updates at display refresh rate.

### Core Pattern: Fixed Timestep with Accumulated Lag

```javascript
const TICK_MS = 50; // 20 ticks per second
let lastTime = null;
let accumulated = 0;

function gameLoop(currentTime) {
    if (lastTime === null) lastTime = currentTime;
    const delta = currentTime - lastTime;
    lastTime = currentTime;
    accumulated += delta;

    // Fixed-step updates — deterministic, no float drift
    let ticks = 0;
    while (accumulated >= TICK_MS) {
        accumulated -= TICK_MS;
        update(TICK_MS);
        if (++ticks >= 240) {  // panic brake — prevents spiral of death
            accumulated = 0;
            break;
        }
    }

    render();
    requestAnimationFrame(gameLoop);
}

requestAnimationFrame(gameLoop);
```

### Why Fixed Timestep

- **Deterministic**: Same tick duration every time — no float precision drift from variable delta
- **Offline catch-up**: Can replay N ticks to simulate offline time (with optimization)
- **Testable**: Feed a known tick count, get predictable results
- **Panic brake**: If the browser was backgrounded, `accumulated` could be huge — cap the
  catch-up loop to prevent freezing

### Update vs Render Separation

```javascript
function update(dt) {
    // All game logic — resource production, timers, cooldowns
    // dt is always TICK_MS here (fixed)
    state.gold += state.goldPerTick;
}

function render() {
    // DOM updates only — read from state, write to DOM
    // Runs at display refresh rate, NOT game tick rate
    elements.gold.textContent = formatNumber(state.gold);
}
```

### Tick Rate Selection

| Rate | Ms/Tick | Use Case |
|------|---------|----------|
| 4/sec | 250ms | Simple resource accumulation, low CPU |
| 10/sec | 100ms | Standard incremental — responsive enough for active play |
| 20/sec | 50ms | Combat resolution, real-time feedback needed |
| 60/sec | ~16ms | Only if you have canvas animation (overkill for most incrementals) |

For DOM-based incrementals, 10 ticks/sec is the sweet spot. Reserve higher rates for phases
with active combat or time-critical mechanics.

### Fractional Tick Accumulator

When a speed bonus produces sub-integer extra ticks per frame, do NOT use `Math.ceil`
on individual ticks — any fractional bonus rounds to 2x. Use a fractional accumulator:

```javascript
let bonusAccum = 0;

function tickWithSpeedBonus(dt, speedMult) {
    let effectiveDt = dt;
    if (speedMult > 1) {
        bonusAccum += dt * (speedMult - 1);
        const bonusTicks = Math.floor(bonusAccum);
        bonusAccum -= bonusTicks;
        effectiveDt = dt + bonusTicks;
    }
    timer -= effectiveDt;
    if (timer <= 0) {
        completeAction();
        bonusAccum = 0; // reset on completion
    }
}
```

With a 1.2x multiplier at dt=1: `Math.ceil(1 * 1.2) = 2` (doubles speed — wrong).
The accumulator correctly distributes 0.2 extra ticks over 5 frames, granting 1 bonus
tick every 5th frame for a true 20% speedup.

---

## 2. Resource System

### Resource Object Pattern

```javascript
const resources = {
    gold:      { amount: 0, perTick: 0, display: 'Gold' },
    food:      { amount: 0, perTick: 0, display: 'Food' },
    materials: { amount: 0, perTick: 0, display: 'Materials' },
    influence: { amount: 0, perTick: 0, display: 'Influence' }
};
```

### Production Calculation

Recalculate `perTick` whenever buildings/upgrades change — not every tick.

```javascript
function recalculateProduction() {
    resources.gold.perTick = 0;
    for (const b of buildings) {
        if (b.owned > 0) {
            resources.gold.perTick += b.baseProduction * b.owned * getMultiplier(b.id);
        }
    }
}

// In update():
function update(dt) {
    for (const key in resources) {
        resources[key].amount += resources[key].perTick;
        // Floor check for resources that can't go negative
        if (resources[key].amount < 0) resources[key].amount = 0;
    }
}
```

### Production Formula

```
production_total = (base_production * owned) * multipliers
```

Multipliers stack multiplicatively from upgrades, prestige bonuses, and research.

### Upkeep-Gated Effects

Buildings that consume a resource to provide a passive effect. When net production goes
negative, disable the effect — don't allow infinite debt.

```javascript
// In recalculateProduction():
let upkeep = 0;
for (const id in BALANCE.buildings) {
    if (def.upkeep && buildings[id] > 0) {
        upkeep += def.upkeep * buildings[id];
    }
}
state.perTick.resource = state.perTick.resource.minus(upkeep);

// Effect functions gate on net production:
function getEffectMult() {
    if (state.perTick.resource.lessThan(0)) return 1; // disabled
    return 1 + bonus * count;
}
```

Creates meaningful economic tension: stronger effects require more upkeep buildings to
sustain the resource drain. Binary on/off gate (not gradual degradation) keeps the
tradeoff sharp and readable. Show "[INACTIVE]" on building cards when gated off.

---

## 3. Cost Scaling Formulas

### Exponential Cost (Standard)

The bread and butter of incremental games. Costs grow exponentially, production grows
linearly/polynomially — this creates the "wall" that drives prestige.

```
cost_next = base_cost * growth_rate ^ owned
```

```javascript
function getCost(building) {
    return Math.floor(building.baseCost * Math.pow(building.growthRate, building.owned));
}
```

Typical growth rates: 1.07 (gentle), 1.15 (standard), 1.5 (aggressive).

### Bulk Purchase Cost

Cost to buy `n` units starting from `k` already owned:

```
cost_bulk = base * rate^k * (rate^n - 1) / (rate - 1)
```

### Maximum Affordable

Given currency `c`, how many can you buy?

```
max = floor( log( c * (rate - 1) / (base * rate^k) + 1 ) / log(rate) )
```

```javascript
function maxAffordable(currency, baseCost, growthRate, owned) {
    const numerator = currency * (growthRate - 1) / (baseCost * Math.pow(growthRate, owned)) + 1;
    if (numerator <= 0) return 0;
    return Math.floor(Math.log(numerator) / Math.log(growthRate));
}
```

### Flat Cost + Upkeep Drain (Alternative to Exponential)

Not all purchasable items need exponential scaling. Units with ongoing upkeep use flat
purchase costs — the economic constraint is the recurring drain, not the one-time price.

```javascript
// BALANCE config — flat purchase, recurring upkeep
squads: {
    custodes: {
        hireCost: { gold: 80 },           // flat, same every time
        upkeep: { gold: 0.05, food: 0.03 } // per tick, per unit
    }
}

// Upkeep subtracted in recalculateProduction(), scaled by policy:
function getMilitaryUpkeep(resource) {
    let total = 0;
    for (const id in BALANCE.military.squads) {
        const count = squads[id];
        if (count <= 0) continue;
        const upkeep = BALANCE.military.squads[id].upkeep[resource];
        if (upkeep) total += count * upkeep;
    }
    total *= policyMultiplier; // e.g., payLevel scales gold upkeep
    return total;
}
```

This creates a different tension: buying more units increases ongoing costs, which can
drive production negative. The player must balance unit count against economic output.

### Sub-Linear Bulk Time Scaling

When bulk-purchasing items that take time to build (construction, crafting), linear time
scaling feels punitive. Use a sub-linear exponent so buying 10x doesn't take 10x longer:

```javascript
// Bulk construction time: baseTicks * count^exponent
// exponent < 1 gives diminishing time per additional unit
const BULK_TIME_EXPONENT = 0.32; // ~cube root behavior

function getBulkConstructionTime(baseTicks, count) {
    return Math.ceil(baseTicks * Math.pow(count, BULK_TIME_EXPONENT));
}
// 1 unit: 100 ticks, 10 units: ~209 ticks, 100 units: ~437 ticks
```

The exponent is a tuning knob: 0.5 (sqrt) is moderate, 0.3 is generous, 1.0 is linear.
This rewards bulk purchases without making them feel free.

### Key Insight

Exponential growth (n^x) always overtakes polynomial growth (x^k) regardless of coefficients.
This is why cost scaling works — eventually costs outrun production, forcing the player to
seek multipliers or prestige. The ratio between cost growth rate and production growth rate
determines pacing.

---

## 4. Big Number Handling

### When You Need It

JavaScript `Number.MAX_SAFE_INTEGER` is 2^53 (~9e15). `Number.MAX_VALUE` is ~1.8e308.
If your game's numbers will exceed 1e308, you need a big number library.

For most incremental games: you WILL exceed this in late game. Plan for it from the start —
retrofitting is painful.

### break_infinity.js

The standard choice for incremental games. Handles numbers up to ~1e(9e15). Prioritizes
speed over precision (which is fine — incremental games don't need 50-digit accuracy).

**Performance vs decimal.js:**

| Operation | Speedup |
|-----------|---------|
| Constructor | 2.8x |
| Addition | 2.5x |
| Multiplication | 2.9x |
| log10() | 121x |
| pow() | 442x |

### Embedding in Single-File HTML

For the no-external-dependencies constraint, embed the minified source directly:

```html
<script>
// === break_infinity.js v2.x (minified) ===
// Paste minified source here (~15KB)
// Available from: https://cdn.jsdelivr.net/npm/break_infinity.js@2/dist/break_infinity.min.js
</script>
<script>
// Your game code — Decimal is now available globally
const gold = new Decimal(0);
const income = new Decimal(10);
gold = gold.plus(income);
</script>
```

Fetch the minified source once, paste it into the HTML file. No CDN dependency at runtime.

### API Essentials

```javascript
// Construction
const x = new Decimal(100);
const y = new Decimal('1e500');
const z = new Decimal(x);

// Arithmetic (returns new Decimal, immutable)
x.plus(y)          // addition
x.minus(y)         // subtraction
x.times(y)         // multiplication
x.dividedBy(y)     // division
x.pow(3)           // exponentiation
x.sqrt()           // square root
x.log10()          // log base 10
x.floor()          // floor
x.ceil()           // ceiling
x.abs()            // absolute value

// Comparison
x.equals(y)        // ==
x.greaterThan(y)   // >
x.greaterThanOrEqualTo(y) // >=
x.lessThan(y)      // <
Decimal.max(x, y)  // max of two
Decimal.min(x, y)  // min of two

// Serialization (for save data)
const str = x.toString();       // "1e500"
const back = new Decimal(str);  // round-trips cleanly

// Compact serialization (mantissa + exponent pair)
const compact = { m: x.mantissa, e: x.exponent };
const restored = Decimal.fromMantissaExponent(compact.m, compact.e);

// Tolerance comparisons (useful for float fuzz in game logic)
x.eq_tolerance(y, 1e-10)  // equal within tolerance
x.lt_tolerance(y, 1e-10)  // less-than within tolerance
```

### Decimal-Aware Cost Functions

```javascript
function getCost(building) {
    return Decimal.floor(
        new Decimal(building.baseCost).times(
            new Decimal(building.growthRate).pow(building.owned)
        )
    );
}

function canAfford(resource, cost) {
    return resource.greaterThanOrEqualTo(cost);
}
```

### Pre-Computed Decimal Costs (Hot-Path Optimization)

Static costs from frozen BALANCE config should be converted to Decimals once at module init,
not per-frame. Creating `new Decimal()` objects in the render loop violates the no-allocation
hot-path rule and generates garbage every tick.

```javascript
// BAD — allocates new objects every render frame
function hireCostDecimals(squadId) {
    const costs = {};
    for (const res in def.hireCost) {
        costs[res] = D(def.hireCost[res]);  // allocation in hot path!
    }
    return costs;
}

// GOOD — pre-compute once, reference forever
const HIRE_COSTS = {};
for (const id in BALANCE.military.squads) {
    HIRE_COSTS[id] = costToDecimals(BALANCE.military.squads[id].hireCost);
}

function costToDecimals(costDef) {
    const costs = {};
    for (const res in costDef) {
        costs[res] = D(costDef[res]);
    }
    return costs;
}

// In render: canAfford(HIRE_COSTS[id]) — zero allocation
```

This applies to any static cost: building base costs, equipment costs, recruit costs.
Only dynamic costs (bulk buy with exponential scaling) need per-call computation.

### Custom Lightweight Alternative

If break_infinity.js is too large (~15KB minified) and your numbers stay under ~1e1000,
a mantissa+exponent pair works:

```javascript
// { mantissa: 1.5, exponent: 308 } represents 1.5e308
// Arithmetic is straightforward but you lose library convenience
```

For most projects, just embed break_infinity.js — the 15KB cost is negligible in a
single-file game.

---

## 5. Save and Load System

### Save Data Structure

```javascript
function createSaveData() {
    return {
        version: SAVE_VERSION,           // integer, increment on schema change
        timestamp: Date.now(),           // for offline progress calc
        resources: serializeResources(), // current resource amounts
        buildings: serializeBuildings(), // owned counts, unlock states
        upgrades: serializeUpgrades(),   // purchased upgrade IDs
        prestige: serializePrestige(),   // prestige currency, tier data
        statistics: serializeStats()     // lifetime totals, play time
    };
}
```

### Save to localStorage

```javascript
const SAVE_KEY = 'myGame_save';

function save() {
    const data = createSaveData();
    const json = JSON.stringify(data);
    localStorage.setItem(SAVE_KEY, json);
}
```

### Load with Version Migration

```javascript
function load() {
    const raw = localStorage.getItem(SAVE_KEY);
    if (!raw) return null; // new game

    let data;
    try {
        data = JSON.parse(raw);
    } catch (e) {
        console.error('Save data corrupted');
        return null; // fall back to new game
    }

    // Version migration chain
    if (data.version < 2) data = migrateV1toV2(data);
    if (data.version < 3) data = migrateV2toV3(data);
    // Each migration bumps data.version

    return data;
}
```

### Migration Function Pattern

```javascript
function migrateV1toV2(data) {
    // V2 added the 'influence' resource
    if (!data.resources.influence) {
        data.resources.influence = { amount: 0, perTick: 0 };
    }
    data.version = 2;
    return data;
}
```

### Property-Existence Loading (Defensive)

When applying save data to game state, always check properties exist:

```javascript
function applySaveData(data) {
    if (data.resources && typeof data.resources.gold === 'number') {
        resources.gold.amount = data.resources.gold;
    }
    // Repeat for each field — missing fields get default values
}
```

### Export/Import String

For sharing saves or backup:

```javascript
function exportSave() {
    const data = createSaveData();
    return btoa(JSON.stringify(data)); // base64 encode
}

function importSave(str) {
    try {
        const data = JSON.parse(atob(str));
        // Validate and migrate
        applySaveData(data);
    } catch (e) {
        alert('Invalid save data');
    }
}
```

### Auto-Save

```javascript
const AUTO_SAVE_INTERVAL = 30000; // 30 seconds
let lastAutoSave = Date.now();

function update(dt) {
    // ... game logic ...
    if (Date.now() - lastAutoSave >= AUTO_SAVE_INTERVAL) {
        save();
        lastAutoSave = Date.now();
    }
}

// Also save on page unload
window.addEventListener('beforeunload', save);
```

### Decimal Serialization

When using break_infinity.js, Decimals must be converted for JSON:

```javascript
function serializeDecimal(d) {
    return d.toString(); // "1.5e308"
}

function deserializeDecimal(s) {
    return new Decimal(s || 0);
}
```

Store Decimals as strings in save data. Reconstruct on load.

---

## 6. Offline Progress

When the player returns after being away, calculate accumulated resources for the elapsed time.

### Simple Approach (Linear Production)

```javascript
function calculateOfflineProgress(saveTimestamp) {
    const elapsedMs = Date.now() - saveTimestamp;
    const elapsedTicks = Math.floor(elapsedMs / TICK_MS);
    const cappedTicks = Math.min(elapsedTicks, MAX_OFFLINE_TICKS); // cap at e.g. 24 hours

    for (const key in resources) {
        resources[key].amount += resources[key].perTick * cappedTicks;
    }

    return { elapsed: elapsedMs, ticks: cappedTicks };
}
```

### Why Cap Offline Time

- Prevents returning after months to infinite resources
- Gives players a reason to check in (daily cap = daily engagement)
- Typical cap: 8-24 hours of production

### Complex Offline (Non-Linear Systems)

If production changes during offline time (e.g., timers complete, thresholds crossed),
you have two options:

1. **Simulate ticks** — replay N ticks at accelerated speed. Accurate but slow for long
   absences. Cap the simulation or optimize (skip ticks where nothing changes).

2. **Approximate** — use time-averaged production rate. Less accurate but instant.
   Good enough for most games: `resources += avgRate * elapsed`.

For a game with tier transitions and combat, option 2 with a "you were away" summary
screen is the pragmatic choice.

### Parameterized Tick for Offline Subsystems

When subsystems have their own timers (research countdowns, construction timers, cooldowns),
parameterize their tick functions so they work for both real-time and offline catch-up:

```javascript
// Subsystem tick accepts a count — 1 for normal play, N for offline
function tickResearchAndConstruction(ticks) {
    if (activeResearch) {
        activeResearch.remaining -= ticks;
        if (activeResearch.remaining <= 0) {
            completeResearch(activeResearch.id);
        }
    }
    if (activeConstruction) {
        activeConstruction.remaining -= ticks;
        if (activeConstruction.remaining <= 0) {
            completeConstruction(activeConstruction);
        }
    }
}

// In game loop: tickResearchAndConstruction(1);
// In offline progress: tickResearchAndConstruction(cappedTicks);
```

This avoids duplicating tick logic between the game loop and offline progress calculation.
The offline path gets the same behavior (including completions and side effects) without
simulating individual ticks.

### Offline Progress Display

Show the player what they earned:

```javascript
function showOfflineReport(elapsed, gains) {
    // "Welcome back! You were away for 4h 23m."
    // "Your household earned: 1.5M Gold, 230K Food"
    // Display in a modal overlay
}
```

---

## 7. Prestige and Tier Mechanics

### Core Concept

Prestige is a "soft reset" — sacrifice current-tier progress for a persistent bonus
currency that accelerates future runs through the same content. The player replays the
same systems but faster, creating a satisfying power curve.

### Prestige Currency Formulas (Proven in Production)

**Square root of lifetime earnings (Cookie Clicker style):**
```
prestige_gain = cube_root(lifetime_earnings / 1e12)
```

**Square root of max single-run earnings (Realm Grinder style):**
```
prestige_gain = (sqrt(1 + 8 * max_earnings / 1e12) - 1) / 2
```

**Square root of earnings this run (AdVenture Capitalist style):**
```
prestige_gain = 150 * sqrt(run_earnings / 1e15)
```

### Scaling Properties

| Formula | Doubling Requirement | Character |
|---------|---------------------|-----------|
| sqrt(x) | 4x previous earnings | Moderate — steady prestige growth |
| cbrt(x) | 8x previous earnings | Slow — long runs between prestiges |
| x^0.14 | ~140x earnings | Very slow — many small resets viable |

Choose based on desired pacing. sqrt is the safe default.

### Prestige Bonus Application

```javascript
// Additive (safer, easier to balance)
effectiveProduction = baseProduction * (1 + prestigeCurrency * PRESTIGE_BONUS_PER_POINT);

// Multiplicative (explosive — use with caution)
effectiveProduction = baseProduction * Math.pow(1 + PRESTIGE_BONUS_PER_POINT, prestigeCurrency);
```

Additive is the standard. Multiplicative creates runaway growth — only use if you want
prestige to feel dramatically more powerful each reset.

### Implementation Pattern

```javascript
function calculatePrestigeGain() {
    const earnings = state.lifetimeGold; // or thisRunGold
    if (earnings.lessThan(PRESTIGE_THRESHOLD)) return new Decimal(0);
    return Decimal.floor(earnings.dividedBy(1e12).sqrt());
}

function prestige() {
    const gain = calculatePrestigeGain();
    if (gain.lessThanOrEqualTo(0)) return;

    state.prestigeCurrency = state.prestigeCurrency.plus(gain);
    state.prestigeCount += 1;

    // Reset current-tier state
    resetTierState();

    // Prestige bonuses apply immediately to next run
    recalculateProduction();
}
```

### What Resets vs What Persists

| Resets | Persists |
|--------|----------|
| Resources (gold, food, etc.) | Prestige currency |
| Buildings (owned counts) | Purchased prestige upgrades |
| Current-tier upgrades | Statistics (lifetime totals) |
| Research progress (if tier-scoped) | Unlocked tiers |

### Layered Prestige (Tier System)

For a game with multiple tiers (Household → Patron → Senator → Consul):

```javascript
function tierTransition(fromTier, toTier) {
    // Award tier transition currency
    const gain = calculateTierGain(fromTier);
    state.tierCurrency[toTier] = state.tierCurrency[toTier].plus(gain);

    // Previous tier becomes automated (delegation)
    state.tiers[fromTier].automated = true;
    state.tiers[fromTier].automationLevel = 1;

    // New tier starts with base resources + tier bonuses
    initializeTier(toTier);

    // Previous tier still produces, but at reduced attention cost
    // Player focuses on new tier's decisions
}
```

### The Delegation Pattern

In a tier-based game where you "zoom out," the previous tier isn't reset — it's
**delegated**. The player stops micro-managing it; it runs on autopilot with simplified
output. This is different from traditional prestige where progress is wiped.

```javascript
// Each tier has an automation mode
const tier = {
    level: 1,         // Household
    automated: false,  // Player is actively managing
    automationLevel: 0, // 0 = manual, 1+ = delegated (efficiency scales)
    // When automated, produce at (baseRate * automationLevel * 0.8)
    // 80% efficiency — delegation has a cost, incentivizing upgrades
};
```

### Tier-Separated Multiplier Chains

When content spans multiple tiers (Household buildings vs Patron buildings), each tier
may have different bonus sources. Define the production chain order explicitly and apply
multipliers conditionally based on tier:

```javascript
function recalculateProduction() {
    // 1. Base production from Tier 1 (household) buildings
    for (const id of HOUSEHOLD_IDS) {
        const prod = BALANCE.buildings[id].production;
        for (const res in prod) {
            let val = D(prod[res]).times(buildings[id]);

            // 2. Tier-1-only research multiplier (e.g. fundamina)
            if (tier1ProdMult !== 1) val = val.times(tier1ProdMult);

            // 3. Global research multiplier (applies to all tiers)
            if (researchMults[res] !== 1) val = val.times(researchMults[res]);

            // 4. Standing multiplier
            if (standingMult !== 1) val = val.times(standingMult);

            state.perTick[res] = state.perTick[res].plus(val);
        }
    }

    // 5. Delegation efficiency (Tier 1 production reduced when delegated)
    if (state.tier !== 'Household') {
        for (const res in state.perTick) {
            state.perTick[res] = state.perTick[res].times(state.delegationEfficiency);
        }
    }

    // 6. Tier 2 buildings — global research + standing, NO delegation, NO tier-1 research
    for (const id of TIER2_IDS) {
        const prod = BALANCE.buildings[id].production;
        for (const res in prod) {
            let val = D(prod[res]).times(buildings[id]);
            if (researchMults[res] !== 1) val = val.times(researchMults[res]);
            if (standingMult !== 1) val = val.times(standingMult);
            state.perTick[res] = state.perTick[res].plus(val);
        }
    }

    // 7. Sub-entity production (clients, vassals) — after delegation
    // 8. Penalty multipliers (crime, corruption) — on sub-entity production only
    // 9. Upkeep subtraction (military, maintenance)
}
```

**Key principle:** Tier-2 buildings should NOT benefit from tier-1-specific research,
delegation penalties, or tier-1 cost reductions. Similarly, tier-1 research that reduces
building costs should use a different multiplier path for each tier:

```javascript
// Cost calculation — different multiplier source per tier
const costMult = def.tier === 'tier2'
    ? getTier2CostMult()         // e.g. from a specialist leader
    : tier1ResearchCostMult;      // e.g. from research tree

const baseCost = D(def.cost[res]).times(Decimal.pow(def.costScale, owned));
const finalCost = baseCost.times(costMult);
```

This prevents a single research tree from double-dipping across tiers and keeps each
tier's bonuses distinct for balancing purposes.

### Sub-Entity Management (Families, Vassals, Subsidiaries)

In management/tycoon incrementals, entities can own sub-entities with their own
production, loyalty, and growth mechanics. Sub-entities share the same mechanical
framework as buildings but add a relationship dimension:

```javascript
const clientFamilies = []; // Array of family objects

function generateFamilies(count) {
    const usedNames = new Set();
    for (let i = 0; i < count; i++) {
        let name;
        do { name = NAME_POOL[Math.floor(Math.random() * NAME_POOL.length)]; }
        while (usedNames.has(name));
        usedNames.add(name);

        clientFamilies.push({
            id: 'family_' + i,
            name: name,
            specialization: SPEC_IDS[i],  // Each family has a unique role
            estates: 1,                    // Sub-entity count (like buildings)
            loyalty: BALANCE.loyalty.base  // Relationship value
        });
    }
}
```

Sub-entity production follows the same exponential cost scaling as buildings but adds
a loyalty multiplier from threshold lookup:

```javascript
const LOYALTY_THRESHOLDS = [
    { min: 80, mult: 1.25, label: 'Devoted' },
    { min: 50, mult: 1.00, label: 'Loyal' },
    { min: 30, mult: 0.60, label: 'Restless' },
    { min: 10, mult: 0.25, label: 'Disloyal' },
    { min: 0,  mult: 0.00, label: 'Defiant' }   // Zero production
];

function getLoyaltyThreshold(loyalty) {
    for (let i = 0; i < LOYALTY_THRESHOLDS.length; i++) {
        if (loyalty >= LOYALTY_THRESHOLDS[i].min) return LOYALTY_THRESHOLDS[i];
    }
    return LOYALTY_THRESHOLDS[LOYALTY_THRESHOLDS.length - 1];
}
```

**Threshold collapse:** When loyalty hits 0, production drops to zero AND a punitive
effect triggers (estate seizure, defection). This creates a "red zone" the player
must actively manage:

```javascript
function tickLoyalty(ticks) {
    for (let i = 0; i < families.length; i++) {
        const fam = families[i];
        // ... decay/growth logic ...

        // Collapse at zero loyalty — punitive, not just zero production
        if (fam.loyalty <= 0 && fam.estates > 1) {
            fam.estates = 1;  // Lose all but one
            notify(fam.name + ' in open defiance — estates seized.', 'warning');
            state.productionDirty = true;
        }
    }
}
```

Use `getThresholdIndex()` to change-detect loyalty tier shifts — only set
`productionDirty` when the threshold actually changes, not on every tick.

### Derived Composite Ratings

A single display value computed from multiple weighted factors across subsystems.
Used for prestige gates, UI flourish, or unlock conditions:

```javascript
function calculateRating() {
    let score = 0;
    // Weighted components from different subsystems
    score += buildings.palace * BALANCE.rating.perPalace;
    score += researchedCount * BALANCE.rating.perResearch;
    score += Math.floor(state.standing / BALANCE.rating.standingDivisor);
    score += specialLeader ? BALANCE.rating.leaderBonus : 0;
    return Math.min(BALANCE.rating.cap, score);
}
```

Cache the result in `recalculateProduction()` alongside other derived values. Display
with a qualitative label from a threshold lookup (same pattern as loyalty/standing).
Useful as a gate for tier-2 political titles or prestige triggers.

---

## 7b. Decay, Derived Stats, and Feedback Loops

### Decaying Currency

Some resources decay over time rather than accumulating indefinitely. Creates spend-or-lose-it tension.

```javascript
// Counter-based decay (avoids per-tick fractional arithmetic)
function tickDecay(dt) {
    G.decayCounter += dt;
    const intervals = Math.floor(G.decayCounter / BALANCE.decay.interval);
    if (intervals > 0) {
        G.decayCounter -= intervals * BALANCE.decay.interval;
        G.resource = Math.max(BALANCE.decay.floor,
            G.resource - intervals * BALANCE.decay.amount);
    }
}
```

Key design levers:
- **Decay rate** vs **earning rate** — if decay > earn, players must invest in reduction mechanics
- **Decay reduction** tied to other systems (e.g., province happiness reduces gloria decay) creates cross-system incentives
- **Decay floor** prevents total loss but keeps pressure on

[verified in Phase 9g-2: gloria decay creates province happiness investment incentive]

### Purely Derived Stats

Stats computed entirely from player investment (no static base) create clear cause-and-effect:

```javascript
function recalculateHappiness() {
    const buildingBonus = sumBuildingEffects();
    const researchBonus = G.researchHappinessBonus;
    const factionBonus = G.factionHappinessBonus;
    const crimePenalty = getCrimeThreshold(G.crimeRate).penalty;
    G.happiness = Math.min(100, Math.max(floor,
        buildingBonus + researchBonus + factionBonus - crimePenalty));
}
```

Tradeoff: player starts at 0 and must invest to earn positive values. Can feel punishing at tier transitions — consider a grace period or cheap early investments.

[verified in Phase 9g-1: province happiness purely derived, accepted after playtesting]

### Feedback Loops

Two stats that influence each other create emergent tension. Resolve in a single pass to avoid oscillation:

```javascript
// Single-pass resolution: raw crime → crime penalty on happiness → happiness → happiness reduction on crime → final
const rawCrime = basePerEstate * estates - patrolReduction - infraReduction;
const crimePenalty = getCrimeThreshold(rawCrime).happinessPenalty;
const happiness = buildingBonus + researchBonus - crimePenalty;
const happinessReduction = getHappinessThreshold(happiness).crimeReduction;
const finalCrime = Math.max(0, rawCrime - happinessReduction);
```

Key: compute in one direction, don't iterate. The feedback is implicit in the formula, not a convergence loop.

[verified in Phase 9g-1: crime↔happiness single-pass resolution]

### Faction / Sway Systems

Competing influence tracks with discrete threshold bonuses:

```javascript
const THRESHOLDS = [
    { min: 100, label: 'Won',     bonuses: { ... } },
    { min: 67,  label: 'Allied',  bonuses: { ... } },
    { min: 34,  label: 'Favored', bonuses: { ... } },
    { min: 0,   label: 'Neutral', bonuses: null }
];
```

Design patterns:
- **Sway cap** (e.g., max 2 factions at top tier) forces strategic choice
- **Sway decay** prevents passive accumulation — must actively invest
- **Threshold bonuses** (not linear scaling) create clear decision points
- **`recomputeFactionEffects()`** accumulator pattern: module-scope scratch object, iterate factions, apply threshold bonuses, store totals on game state

[verified in Phase 9e: 4 factions, sway 0-100, tiered bonuses, max 2 Won]

### Balance Pass Methodology

When tuning an established game with multiple interconnected systems:

1. **Audit** — enumerate every system, every multiplier, every integration point. Count items per tier.
2. **Identify bottlenecks** — where does progression stall? What resources gate advancement?
3. **Tune config values** — adjust BALANCE numbers only. No structural changes. Track before/after.
4. **Verify cross-system** — every multiplier change affects downstream systems. Check production chain end-to-end.
5. **Playtest at speed** — use dev tools (`setSpeed(100)`) for rapid iteration, then verify at 1×.

Tiered cost approach: utility upgrades cheaper than power upgrades. Same scaling formula, different base costs.

[verified in Phase 9g: 3-phase balance pass across 3 tiers]

---

## 8. Balance Configuration

### Externalized Data Pattern

All numerical values live in a config object, not hardcoded in logic:

```javascript
const BALANCE = Object.freeze({
    resources: {
        gold:      { startAmount: 100, display: 'Gold' },
        food:      { startAmount: 50,  display: 'Food' },
        materials: { startAmount: 0,   display: 'Materials' },
        influence: { startAmount: 0,   display: 'Influence' }
    },
    buildings: {
        farm: {
            baseCost: { gold: 10 },
            growthRate: 1.15,
            production: { food: 1 },
            unlockRequirement: null
        },
        workshop: {
            baseCost: { gold: 50, food: 10 },
            growthRate: 1.18,
            production: { materials: 0.5 },
            unlockRequirement: { buildings: { farm: 3 } }
        }
    },
    upgrades: {
        betterTools: {
            cost: { gold: 500 },
            effect: { buildingMultiplier: { workshop: 2 } },
            unlockRequirement: { buildings: { workshop: 5 } }
        }
    },
    tiers: {
        household: {
            prestigeThreshold: 1e6,     // gold earned to unlock next tier
            prestigeFormula: 'sqrt',     // sqrt, cbrt, or custom
            prestigeDivisor: 1e6
        }
    },
    timing: {
        tickMs: 50,
        autoSaveMs: 30000,
        maxOfflineTicks: 1728000  // 24 hours at 20 ticks/sec
    }
});
```

### Why This Matters

- **Balance tuning**: Change numbers without touching logic
- **Readability**: All progression values visible in one place
- **Testability**: Override BALANCE in test scenarios
- **Modding potential**: Players can tweak values (future feature, not Phase 1)

### Balance Tuning Heuristics

**Payback period** — how long a purchase takes to pay for itself:

```
PaybackTime = Cost(n) / IncomeGainFromPurchase
```

Should be roughly consistent within a tier. If tier 1 pays back in 30s and tier 5 takes
30 minutes with no prestige in between, the pacing is broken.

**The 10-minute rule** (Kongregate heuristic): Players should hit a meaningful decision
point or milestone roughly every 10 minutes of active play in the early game. This cadence
stretches in later phases.

**Multiplier stacking:**
- **Additive within category**: `totalMult = 1 + bonusA + bonusB + bonusC` — safe, linear
- **Multiplicative across categories**: `totalMult = baseMult * prestigeMult * researchMult`
- Most games use additive within a category and multiplicative across categories

**Typical Starting Values:**

| Parameter | Range | Notes |
|-----------|-------|-------|
| Cost growth rate | 1.07 - 1.15 | Lower = slower obsolescence |
| Tier cost ratio | 8x - 15x | Cost of tier N+1 vs tier N base |
| Tier production ratio | 5x - 10x | Production of tier N+1 vs tier N |
| Prestige exponent | 0.5 (sqrt) | Higher = more resets needed |
| First prestige timing | ~1 hour | Tune to early game length |
| Offline efficiency | 10% - 50% | Of active income rate |
| Offline cap | 8 - 24 hours | Prevents infinite accumulation |
| Target payback time | 20s - 120s early | Scales up with progression |

### Progressive Unlock Pattern

Don't show everything at once. Gate UI elements behind progression:

```javascript
function isUnlocked(item) {
    const req = item.unlockRequirement;
    if (!req) return true;

    if (req.buildings) {
        for (const [id, count] of Object.entries(req.buildings)) {
            if (getBuildingCount(id) < count) return false;
        }
    }
    if (req.resources) {
        for (const [id, amount] of Object.entries(req.resources)) {
            if (getResource(id).lessThan(amount)) return false;
        }
    }
    return true;
}
```

### Dynamic Tab/Feature Unlock

Static unlock checks (from config) work for buildings, but tabs and features that depend
on aggregate game state need function-based unlock logic:

```javascript
function isTabUnlocked(tabId) {
    const tabDef = TABS.find(function(t) { return t.id === tabId; });
    if (!tabDef) return false;
    if (tabDef.unlocked) return true;  // static config
    if (tabId === 'military') return state.totalBuildings >= 5;  // dynamic
    return false;
}
```

Use this function in both the tab click handler and the render loop (to toggle
locked/unlocked CSS classes). Tab buttons should show a locked visual and title
text explaining the unlock condition.

### Tradeoff / Policy Systems

A central design tension where the player adjusts policy dials that have both positive
and negative effects. Each dial is a discrete level (e.g., 1-5) affecting multiple systems:

```javascript
// BALANCE config for a tradeoff system (morale example):
morale: {
    baseMorale: 50,
    payWeight: 8,           // morale bonus per pay level above neutral
    rationWeight: 6,        // morale bonus per ration level above neutral
    trainingWeight: 5,      // morale penalty per training level above neutral
    leaderWeight: 2,        // morale bonus per avg leader loyalty point
    // Upkeep multipliers at levels 1-5 (higher = more expensive)
    payUpkeepMult: [0.5, 0.75, 1, 1.5, 2],
    rationUpkeepMult: [0.5, 0.75, 1, 1.5, 2],
    // Combat effectiveness multipliers at training levels 1-5
    trainingCombatMult: [0.7, 0.85, 1, 1.15, 1.3],
    // Named labels per level
    policyLabels: {
        pay: ['Miserly', 'Frugal', 'Fair', 'Generous', 'Lavish']
    }
}

// Formula: morale = clamp(base + payBonus + rationBonus - trainPenalty + leaderBonus, 0, 100)
// Higher pay/rations = better morale but higher upkeep costs
// Higher training = better combat effectiveness but lower morale
```

The key: every dial has both a benefit and a cost. No optimal setting — the player must
adapt to their current economic and military situation. Thresholds (e.g., morale < 30 =
refuse dangerous orders) create meaningful decision points.

### Slot-Based Configuration Pattern

When multiple functions iterate the same set of configuration-driven positions (formation
slots, equipment slots, party members), define slots as a constant array and provide a
resolver that hydrates them with live state:

```javascript
// Slot definitions — single source of truth
const FORMATION_SLOTS = [
    { key: 'commanderId', fraction: 1.0 },
    { key: 'captain1Id', fraction: BALANCE.captainStatFraction },
    { key: 'captain2Id', fraction: BALANCE.captainStatFraction }
];

// Resolver — hydrates slots with current state
function getFormationLeaders() {
    const result = [];
    for (let i = 0; i < FORMATION_SLOTS.length; i++) {
        const slot = FORMATION_SLOTS[i];
        const id = state[slot.key];
        result.push({ leader: id ? getLeaderById(id) : null, fraction: slot.fraction });
    }
    return result;
}

// All consumers iterate the same resolved array:
function calculatePower() {
    const fl = getFormationLeaders();
    for (let i = 0; i < fl.length; i++) {
        power = applyBonus(fl[i].leader, power, fl[i].fraction);
    }
}
```

Adding a 4th slot is a one-line change to the constant array instead of updating every consumer.

### Equipment Distribution Across Aggregate Units

When equipment/buffs apply to aggregate population counts (not named individuals), distribute
inventory best-first and return a weighted average modifier:

```javascript
function getFleetEquipMod(slot) {
    // Collect inventory items with their modifiers
    const items = [];
    for (let i = 0; i < ids.length; i++) {
        if (inventory[slot][ids[i]] > 0) {
            items.push({ count: inventory[slot][ids[i]], mod: defs[ids[i]].mod });
        }
    }
    items.sort(function(a, b) { return b.mod - a.mod; }); // best first

    // Distribute to population
    const cap = totalUnits;
    let totalMod = 0, assigned = 0;
    for (let i = 0; i < items.length && assigned < cap; i++) {
        const take = Math.min(items[i].count, cap - assigned);
        totalMod += take * items[i].mod;
        assigned += take;
    }
    totalMod += (cap - assigned) * 1.0; // unequipped units = base modifier
    return totalMod / cap;
}
```

Best-first distribution rewards players for having fewer high-tier items over many low-tier ones.

### Research Tree with Prerequisites

Research/tech trees are standard progression gates. Define nodes with prerequisite chains
and effect descriptors, organized into tiers:

```javascript
research: {
    nodes: {
        studia: { id: 'studia', name: 'Studia Domestica',
            cost: { gold: 200 }, timeTicks: 300, prereqs: [],
            effects: { productionMult: 1.15 }
        },
        agri: { id: 'agri', name: 'Ars Agraria',
            cost: { gold: 500, food: 100 }, timeTicks: 600, prereqs: ['studia'],
            effects: { foodMult: 1.25 }
        }
    }
}

// Check if node can be started
function canResearch(nodeId) {
    if (researched[nodeId]) return false;
    const node = BALANCE.research.nodes[nodeId];
    for (let i = 0; i < node.prereqs.length; i++) {
        if (!researched[node.prereqs[i]]) return false;
    }
    return canAfford(RESEARCH_COSTS[nodeId]);
}
```

### Cached Research Effect Variables

When multiple systems read research bonuses every tick, cache the derived multipliers in
module-scoped variables with a centralized recompute function:

```javascript
// Cached effect variables — read by production, combat, UI
let researchProductionMult = 1;
let researchCombatMult = 1;
let researchMoraleBonus = 0;

// Single recompute path — called from completeResearch(), applySaveData(), hardReset()
function recomputeResearchEffects() {
    researchProductionMult = 1;
    researchCombatMult = 1;
    researchMoraleBonus = 0;
    for (const id in researched) {
        const effects = BALANCE.research.nodes[id].effects;
        if (effects.productionMult) researchProductionMult *= effects.productionMult;
        if (effects.combatMult) researchCombatMult *= effects.combatMult;
        if (effects.moraleBonus) researchMoraleBonus += effects.moraleBonus;
    }
}
```

This avoids walking the research tree every tick. Consumers just read `researchProductionMult`
directly. The recompute function is the single source of truth — `hardReset()` calls it
instead of duplicating the reset logic.

### Shared Multi-Factor Computation Helper

When a formula's intermediate values are needed by both computation and display (e.g.,
morale factors shown in a breakdown table), extract a shared helper:

```javascript
// Single source of truth for morale factors
function getMoraleFactors() {
    return {
        base: BALANCE.morale.base,
        payBonus: payLevels[state.payLevel].bonus,
        rationBonus: rationLevels[state.rationLevel].bonus,
        trainPenalty: -trainLevels[state.trainingLevel].penalty,
        leaderBonus: getLeaderMoraleBonus(),
        researchBonus: researchMoraleBonus
    };
}

// Computation uses it:
function calculateMorale() {
    const f = getMoraleFactors();
    return clamp(f.base + f.payBonus + f.rationBonus + f.trainPenalty + f.leaderBonus + f.researchBonus, 0, 100);
}

// Display uses the same factors:
function buildMoraleBreakdown() {
    const f = getMoraleFactors();
    // Render each factor as a row in a table
}
```

Without this, the formula and its display drift apart as you add new factors.

### Auto-Resolve Combat with Power Ratio

For incremental games where combat is progression-gated (not real-time), auto-resolve
using aggregate stats and a power ratio:

```javascript
function resolveCombat(encounterId) {
    const playerPower = calculatePlayerPower();
    const enemyPower = encounter.combat * encounter.count;
    const powerRatio = playerPower / enemyPower;

    const victory = powerRatio >= 1.0;
    const isOvermatch = victory && powerRatio >= BALANCE.overmatchRatio; // e.g., 5:1

    // Interpolate casualty rate based on ratio
    let casualtyRate;
    if (isOvermatch) {
        casualtyRate = 0; // trivial fight, no losses
    } else if (victory) {
        const t = Math.min((powerRatio - 1.0) / 2.0, 1.0);
        casualtyRate = maxRate - t * (maxRate - minRate);
    } else {
        casualtyRate = maxRate; // defeat = heavy losses
    }
}
```

The overmatch threshold rewards investment with a qualitative improvement (zero casualties),
not just better numbers. This creates a natural "farm until overmatch" loop.

### Temporary Event Modifiers with Decay

Combat outcomes or random events apply temporary morale/stat modifiers that decay over time:

```javascript
let moraleEventModifier = 0;   // positive for victory, negative for defeat
let moraleDecayCounter = 0;

// In combat resolution:
moraleEventModifier += victory ? BALANCE.victoryBonus : BALANCE.defeatPenalty;
moraleEventModifier = clamp(moraleEventModifier, -20, 20);

// In game loop tick:
moraleDecayCounter++;
if (moraleDecayCounter >= BALANCE.moraleDecayInterval) {
    moraleDecayCounter = 0;
    if (moraleEventModifier > 0) moraleEventModifier--;
    else if (moraleEventModifier < 0) moraleEventModifier++;
}
```

This creates a feedback loop: winning builds momentum, losing creates a spiral the player
must recover from. The decay rate controls how long effects linger.

### Diminishing Returns (Soft Cap)

When stacking bonuses from repeated purchases, use a diminishing returns formula instead
of linear scaling or hard caps:

```
effective = 1 + (bonus_per_unit * count) / (1 + count * diminish_factor)
```

```javascript
// diminishFactor controls curve steepness. Approaches bonusPerUnit/diminishFactor at infinity.
// bonus=0.15, factor=0.5: 1 unit → 1.10, 5 → 1.21, 10 → 1.25, 50 → 1.29
function getDiminishingBonus(bonusPerUnit, count, diminishFactor) {
    return 1 + bonusPerUnit * count / (1 + count * diminishFactor);
}
```

Preferable to hard caps (`Math.min`) because additional purchases always help, just less
than the last. Put `bonusPerUnit` and `diminishFactor` in BALANCE config, not in code.

### Compound Unlock Prerequisites

Beyond simple counters (totalBuildings >= 10), gate unlocks on specific owned items:

```javascript
// BALANCE config:
schola: { unlockAt: { forumStall: 2 } }  // requires 2 Forum Stalls

function isBuildingUnlocked(id) {
    const reqs = def.unlockAt;
    if (reqs.totalBuildings && state.totalBuildings < reqs.totalBuildings) return false;
    for (const key in reqs) {
        if (key === 'totalBuildings') continue;
        if ((buildings[key] || 0) < reqs[key]) return false;
    }
    return true;
}
```

Creates meaningful progression choices: invest in specific prerequisites, not just any
quantity. Define requirements in BALANCE config so they're tunable without code changes.

### Multiplier Accumulation Pattern

When multiple sources contribute to the same multiplier, accumulate raw sum in a loop,
then clamp once after:

```javascript
// CORRECT — accumulate then clamp
let cooldownAccum = 0;
for (const id in researched) {
    if (epl.cooldownMult) cooldownAccum += epl.cooldownMult * level;
}
cooldownMult = Math.max(0.1, 1 + cooldownAccum);

// WRONG — assignment per node overwrites, last node wins
for (const id in researched) {
    if (epl.cooldownMult) cooldownMult = 1 + epl.cooldownMult * level; // BUG
}
```

This matters when a second source is added for the same effect. The `=` version silently
drops the first source's contribution. Apply the same pattern consistently across all
accumulated effects (reduction mults, bonus mults, additive bonuses).

### Threshold-Based Tier Effects

When a value (standing, reputation, honor) crosses thresholds that change gameplay multipliers,
cache the current tier's effects and only recompute when the tier actually changes:

```javascript
const STANDING_THRESHOLDS = [
    { min: 0,  label: 'Poor',      effects: { prodMult: 0.9, moraleBonus: -5 } },
    { min: 20, label: 'Fair',      effects: { prodMult: 1.0, moraleBonus: 0 } },
    { min: 50, label: 'Good',      effects: { prodMult: 1.1, moraleBonus: 5 } },
    { min: 80, label: 'Excellent', effects: { prodMult: 1.2, moraleBonus: 10 } }
];

function getStandingThreshold(value) {
    for (let i = STANDING_THRESHOLDS.length - 1; i >= 0; i--) {
        if (value >= STANDING_THRESHOLDS[i].min) return STANDING_THRESHOLDS[i];
    }
    return STANDING_THRESHOLDS[0];
}

// Cache current effects — only recompute on tier change
let cachedProdMult = 1;
let cachedMoraleBonus = 0;

function recomputeStandingEffects() {
    const thr = getStandingThreshold(state.standing);
    const changed = thr.effects.prodMult !== cachedProdMult
                 || thr.effects.moraleBonus !== cachedMoraleBonus;
    cachedProdMult = thr.effects.prodMult;
    cachedMoraleBonus = thr.effects.moraleBonus;
    return changed;  // Caller decides whether to set productionDirty
}
```

Put thresholds in BALANCE config. The reverse-order search returns the highest matching tier.
This pattern applies to any tiered reputation/rank/difficulty system.

### Per-Action Cooldown Registry

Track cooldowns per action ID in a keyed map. Tick function decrements all active cooldowns.
Gate the tick behind an unlock flag so iteration doesn't run before the subsystem activates:

```javascript
const cooldowns = {};  // { actionId: ticksRemaining }

function performAction(id) {
    // ... execute action ...
    cooldowns[id] = BALANCE.actions[id].cooldownTicks;
}

function tickCooldowns() {
    if (!state.subsystemUnlocked) return;  // No work before unlock
    for (const id in cooldowns) {
        if (cooldowns[id] > 0) cooldowns[id]--;
    }
}

function isOnCooldown(id) {
    return (cooldowns[id] || 0) > 0;
}
```

For offline progress, subtract elapsed ticks in bulk: `cooldowns[id] = Math.max(0, cooldowns[id] - elapsed)`.
Include cooldowns in save data and reset them in `hardReset()`.

### Lock Reason Display Function

Separate the human-readable lock reason from the boolean unlock check. The unlock check
returns true/false for logic gates; the reason function returns display text for the UI:

```javascript
function isActionUnlocked(id) {
    const def = BALANCE.actions[id];
    if (def.prereqAction && !actionsPerformed[def.prereqAction]) return false;
    if (def.minStanding && state.standing < def.minStanding) return false;
    return true;
}

function getActionLockReason(id) {
    const def = BALANCE.actions[id];
    if (def.prereqAction && !actionsPerformed[def.prereqAction]) {
        return 'Requires: ' + BALANCE.actions[def.prereqAction].name;
    }
    if (def.minStanding && state.standing < def.minStanding) {
        return 'Requires ' + def.minStanding + ' Standing';
    }
    return '';  // Empty = unlocked
}
```

This avoids duplicating prerequisite logic between action execution and UI display.
The reason function checks prerequisites in the same order as the unlock check.

### Decay-to-Floor Mechanics

A value decays periodically but never below a configurable floor. The floor can be
raised by buildings or upgrades. Both live ticks and offline progress must clamp to
the same floor:

```javascript
function getDecayFloor() {
    const base = BALANCE.decay.minThreshold;         // e.g. 10
    const bonus = BALANCE.buildings.temple.floorPerUnit * buildings.temple;
    const cap = BALANCE.buildings.temple.floorCap;
    return Math.min(cap, base + bonus);
}

function tickDecay() {
    if (!state.subsystemUnlocked) return;
    const floor = getDecayFloor();
    if (state.standing <= floor) return;              // Already at floor
    decayCounter++;
    if (decayCounter >= BALANCE.decay.interval) {
        decayCounter = 0;
        state.standing = Math.max(floor, state.standing - BALANCE.decay.amount);
        if (recomputeEffects()) state.productionDirty = true;
    }
}

// Offline — same floor, same clamp:
const floor = getDecayFloor();
state.standing = Math.max(floor, state.standing - decayPerTick * elapsedTicks);
```

**Critical:** Use `Math.max(floor, ...)` not `Math.max(0, ...)`. If live decay stops at 10,
offline decay must stop at 10 too. Mismatched floors between live and offline is a common bug.

### recomputeX() Returning Boolean for Conditional Dirty-Flagging

When a recomputation function caches derived values, return whether anything actually changed.
Callers use the boolean to conditionally set `productionDirty`, avoiding redundant full
recalculations when the underlying values didn't shift:

```javascript
function recomputeEffects() {
    const newMult = calculateMult();
    const newBonus = calculateBonus();
    const changed = newMult !== cachedMult || newBonus !== cachedBonus;
    cachedMult = newMult;
    cachedBonus = newBonus;
    return changed;
}

// In tick function — only dirty-flag on actual change:
if (recomputeEffects()) state.productionDirty = true;

// In action handler — always dirty-flag (action implies change):
recomputeEffects();
state.productionDirty = true;
```

This pattern is most valuable when the recomputation runs frequently (every decay tick)
but actual threshold changes are rare.

### Multi-Level Node Status States

Leveled progression nodes (skills, research, upgrades) have three meaningful states.
Centralizing the status check prevents scattered level/prereq logic:

```javascript
function getNodeStatus(nodeId) {
    const def = BALANCE.nodes[nodeId];
    const level = getLevel(nodeId);
    if (level >= def.maxLevel) return 'maxed';
    for (let i = 0; i < def.prereqs.length; i++) {
        if (getLevel(def.prereqs[i]) < 1) return 'locked';
    }
    return 'available';
}
```

Render maps status to visual treatment:
- **maxed**: grayed out, no button, shows "MAX" badge
- **available**: normal appearance, clickable, shows cost for next level
- **locked**: darkened, disabled, shows prerequisite names

For leveled nodes, cost and time scale per level: `baseCost × costScale^currentLevel`.
Cache the next-level cost keyed by `nodeId:level` to avoid Decimal allocation in the
render loop; invalidate on level-up and hard reset.

### Equations Reference Document

For any game with more than a handful of interacting bonuses, multipliers, or entity-specific effects, maintain a living `equations.md` satellite file that maps:

| Column | Purpose |
|--------|---------|
| Effect name | e.g., "Patrol Defense Bonus" |
| Owner | Which entity/unit/research grants it |
| Scope | What it applies to (e.g., "Custodes only" vs "all retinue") |
| Formula | How it's calculated |
| Consumers | Which systems/functions read this value |

**Why this matters for incremental games specifically:** Entity-specific bonuses are a core design pattern (unit X gets bonus Y for role Z). Without a reference doc, a session auditing equations may incorrectly generalize a scoped bonus to all entities -- homogenizing intentional specialization. The document is the ground truth that prevents this.

**When to create:** Once you have 3+ entity types with distinct bonus profiles, or any time a bonus is intentionally scoped to a subset of entities rather than applied globally. Create it as a project satellite file and update it during phase wraps.

---

## 9. UI Update Patterns

### DOM Update Throttling

Don't update every DOM element every tick. Use a render cycle that's decoupled from
the game tick and only updates visible/changed elements.

```javascript
let renderDirty = true; // flag when state changes

function update(dt) {
    // Game logic
    renderDirty = true;
}

function render() {
    if (!renderDirty) return;
    renderDirty = false;

    updateResourceDisplay();
    updateBuildingList();
    updateUpgradeList();
}
```

### Tab-Based UI

Only render the active tab:

```javascript
function render() {
    switch (state.activeTab) {
        case 'overview':  renderOverview();  break;
        case 'holdings':  renderHoldings();  break;
        case 'military':  renderMilitary();  break;
        case 'research':  renderResearch();  break;
        case 'politics':  renderPolitics();  break;
    }
    // Always render the resource bar (visible on all tabs)
    renderResourceBar();
}
```

### Stable DOM for Interactive Frequently-Updated Elements [verified in Phase 7d — campaign active display]

When a container has interactive elements (buttons, dropdowns) AND updates frequently (timers, progress bars),
**never** use `innerHTML` rebuilds. The browser requires `mousedown` and `mouseup` to hit the same DOM element
for a `click` event. Rebuilding innerHTML between the two events (which happens at 60fps) silently kills clicks.

**Anti-pattern:**
```javascript
// BAD — buttons are destroyed and recreated every frame
function renderActiveItems() {
    let html = '';
    for (const item of items) {
        html += '<div>' + item.timer + '<button data-id="' + item.id + '">Cancel</button></div>';
    }
    container.innerHTML = html; // kills pending click events
}
```

**Correct pattern:**
```javascript
// GOOD — create elements once, update text/style in place
const itemRefs = [];

// Sync DOM element count with data array
while (itemRefs.length > items.length) { itemRefs.pop().el.remove(); }
while (itemRefs.length < items.length) {
    const el = document.createElement('div');
    const timer = document.createElement('span');
    const btn = document.createElement('button');
    btn.textContent = 'Cancel';
    el.appendChild(timer);
    el.appendChild(btn);
    container.appendChild(el);
    itemRefs.push({ el, timer, btn, lastTimer: '' });
}

// Update only changed values
for (let i = 0; i < items.length; i++) {
    const timeStr = formatTime(items[i].remaining);
    if (timeStr !== itemRefs[i].lastTimer) {
        itemRefs[i].timer.textContent = timeStr;
        itemRefs[i].lastTimer = timeStr;
    }
}
```

Use event delegation on the stable parent container — the buttons are never destroyed, so clicks always land.

### Tooltip Pattern

Tooltips are genre convention — everything should have one:

```javascript
function createTooltip(element, contentFn) {
    // contentFn is called on hover — returns current values
    // Use a single shared tooltip element, reposition on hover
    element.addEventListener('mouseenter', function(e) {
        tooltipEl.innerHTML = contentFn();
        tooltipEl.style.display = 'block';
        positionTooltip(e);
    });
    element.addEventListener('mouseleave', function() {
        tooltipEl.style.display = 'none';
    });
}
```

### Notification System

Events, completions, warnings — queue them and display:

```javascript
const notifications = [];

function notify(message, type) {
    notifications.push({ message, type, time: Date.now() });
    if (notifications.length > 50) notifications.shift(); // cap
}

function renderNotifications() {
    // Show last N notifications, fade old ones
    // Types: 'info', 'warning', 'achievement', 'combat'
}
```

---

## 10. Number Formatting

### Tiered Formatting

```javascript
function formatNumber(n) {
    if (n instanceof Decimal) return formatDecimal(n);
    if (n < 1000) return Math.floor(n).toString();
    if (n < 1e6) return (n / 1e3).toFixed(1) + 'K';
    if (n < 1e9) return (n / 1e6).toFixed(1) + 'M';
    if (n < 1e12) return (n / 1e9).toFixed(1) + 'B';
    if (n < 1e15) return (n / 1e12).toFixed(1) + 'T';
    return n.toExponential(2);
}

function formatDecimal(d) {
    if (d.lessThan(1000)) return d.toFixed(0);
    if (d.exponent < 6) return (d.toNumber() / 1e3).toFixed(1) + 'K';
    if (d.exponent < 9) return (d.toNumber() / 1e6).toFixed(1) + 'M';
    // ... continue tiers ...
    return d.mantissa.toFixed(2) + 'e' + d.exponent;
}
```

### Time Formatting

```javascript
function formatTime(ms) {
    const seconds = Math.floor(ms / 1000);
    if (seconds < 60) return seconds + 's';
    if (seconds < 3600) return Math.floor(seconds / 60) + 'm ' + (seconds % 60) + 's';
    const hours = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    return hours + 'h ' + mins + 'm';
}
```

---

## 11. Anti-Patterns

### Hardcoded Balance Values

```javascript
// BAD — magic numbers scattered in logic
gold += 10 * buildings.farm * 1.5;

// GOOD — reference the balance config
gold += BALANCE.buildings.farm.production.gold * buildings.farm * getMultiplier('farm');
```

### Floating-Point Accumulation

```javascript
// BAD — float errors accumulate over millions of ticks
gold += 0.1;  // after 10M ticks: 999999.9999...

// GOOD — use integers or Decimal for accumulation
// Or accept small drift if using fixed timestep (it's bounded)
```

### setInterval for Game Loop

```javascript
// BAD — drifts, doesn't pause when tab is hidden, imprecise
setInterval(update, 100);

// GOOD — requestAnimationFrame with fixed timestep
requestAnimationFrame(gameLoop);
```

### Saving Full State Every Tick

```javascript
// BAD — localStorage.setItem is synchronous and slow
function update(dt) {
    // ... logic ...
    save(); // every 50ms = 20 writes/sec to localStorage
}

// GOOD — auto-save on a 30-second timer + beforeunload
```

### Allocating Decimals in Hot Paths

```javascript
// BAD — creates new objects every render frame (10+ times/sec)
function renderCost(squadId) {
    const costs = {};
    for (const res in def.hireCost) {
        costs[res] = new Decimal(def.hireCost[res]);  // allocation!
    }
    return canAfford(costs);
}

// GOOD — pre-compute static costs once at init
const HIRE_COSTS = {};
for (const id in BALANCE.squads) {
    HIRE_COSTS[id] = costToDecimals(BALANCE.squads[id].hireCost);
}
// In render: canAfford(HIRE_COSTS[id]) — zero allocation
```

### Mutating Decimal Values

```javascript
// BAD — Decimal operations return NEW values, they don't mutate
gold.plus(income); // return value discarded!

// GOOD
gold = gold.plus(income);
```

---

## 12. Gotchas

| Issue | Symptom | Fix |
|-------|---------|-----|
| Tab backgrounding | Browser throttles rAF to 1/sec or less in background tabs | Offline progress calc handles this — big delta on return |
| localStorage 5MB limit | Save fails silently or throws | Keep save data lean — don't store derived values, only source-of-truth state |
| JSON.stringify + Decimal | Decimals become `{}` in JSON | Serialize Decimals to strings before JSON.stringify |
| beforeunload unreliable | Mobile browsers may not fire it | Auto-save timer is the primary; beforeunload is backup |
| Number.toFixed rounding | `1.005.toFixed(2)` gives "1.00" not "1.01" | Use Math.round for display, or formatNumber helper |
| Exponential cost overflow | `Math.pow(1.15, 500)` = Infinity | Switch to Decimal before numbers get large — plan from start |
| Prestige too early | Player can prestige for 0 gain | Gate prestige behind minimum gain threshold |
| Production recalc cost | Recalculating every tick is wasteful | Dirty flag — only recalculate when buildings/upgrades change |
| CSS reflow in render loop | Reading offsetHeight then writing style causes layout thrashing | Batch reads, then batch writes (see html5-single-file-architecture) |
| Decimal allocation in render | GC pressure from per-frame `new Decimal()` | Pre-compute static costs at init; only compute dynamic costs on demand |
| Morale not updating on recruit | Visual lag until next policy change | Set `productionDirty = true` whenever any morale input changes (hire, recruit, policy) |
| querySelector in render loop | DOM queries (12+/frame) per equipment/leader display | Cache element refs at DOM creation time, use parallel ref arrays |
| Change-guard key incomplete | Guard key `a + '\|' + b` but HTML also renders `c` — changes to `c` don't trigger re-render | Guard key must include ALL values that appear in the rendered output |
| Research effect drift | Formula and display disagree after adding a new research bonus | Extract shared factors helper; both computation and breakdown consume same intermediate values |
| Change-guard key incomplete | Guard key `a + '\|' + b` but HTML also renders `c` — changes to `c` don't trigger re-render | Guard key must include ALL values that appear in the rendered output |

---

## 13. New Incremental Game Checklist

### Engine Foundation
- [ ] Fixed-timestep game loop with rAF
- [ ] Update/render separation
- [ ] Panic brake for catch-up spiral
- [ ] BALANCE config object with all numerical values
- [ ] Resource system with perTick recalculation

### Persistence
- [ ] Save/load with JSON + localStorage
- [ ] Save versioning with migration chain
- [ ] Auto-save timer (30s default)
- [ ] beforeunload save backup
- [ ] Export/import base64 string
- [ ] Offline progress calculation on load

### Numbers
- [ ] Big number strategy chosen (break_infinity.js or native-until-needed)
- [ ] Number formatting (K/M/B/T/scientific)
- [ ] Time formatting helper
- [ ] Decimal serialization for save data

### Economy
- [ ] Exponential cost scaling
- [ ] Bulk purchase formula
- [ ] Max affordable calculation
- [ ] Production multiplier stacking
- [ ] Unlock/progression gating
- [ ] Pre-computed Decimal costs for static BALANCE values

### Military / Tradeoffs (if applicable)
- [ ] Flat-cost units with upkeep drain
- [ ] Upkeep subtracted in recalculateProduction()
- [ ] Policy/tradeoff dials with named levels
- [ ] Morale or satisfaction formula (multi-input, clamped)
- [ ] Threshold-based behavior gates (e.g., morale < 30 = refuse orders)
- [ ] Dynamic tab unlock based on game state
- [ ] Slot-based formation/party config (constant array + resolver)
- [ ] Equipment distribution across aggregate units (weighted average)
- [ ] Auto-resolve combat (power ratio interpolation, overmatch threshold)
- [ ] Temporary event modifiers with decay

### Research / Tech Tree (if applicable)
- [ ] Research nodes with prerequisite chains
- [ ] Cached research effect variables with centralized recompute function
- [ ] Shared multi-factor computation helper (formula + display use same factors)
- [ ] Parameterized subsystem tick function (1 for real-time, N for offline)
- [ ] Sub-linear bulk time scaling for construction/crafting

### UI
- [ ] Tab-based navigation
- [ ] Dynamic tab unlock (function-based, not just config)
- [ ] Tooltip system
- [ ] Notification queue
- [ ] DOM update throttling (dirty flag or visible-tab-only)
- [ ] Offline progress summary on return
- [ ] Cached DOM refs per subsystem (parallel ref arrays)

### Prestige (if applicable)
- [ ] Prestige currency formula chosen
- [ ] Reset vs persist state separation
- [ ] Minimum gain threshold
- [ ] Prestige upgrade tree
- [ ] Delegation/automation for previous tier
