# Code Architecture for AI Collaboration -- Skill Reference

Principles for writing code that works well with AI coding assistants. Not a replacement for language-specific skills -- this is the cross-cutting "why" that connects patterns already taught elsewhere.

Core insight: **context is the scarce resource.** LLM reasoning quality degrades as context fills, even within large windows. A 300-line file where every line matters produces better AI output than a 3,000-line file where 10% is relevant. These principles manage signal-to-noise, not just file size.

Companion to all language-specific skills. References them where they already cover a topic.

---

## 1. File Architecture

### Keep Files Focused

A file should contain one coherent responsibility. When a file serves multiple purposes, AI must load all of them to work on any one of them.

**Orientation markers** (not hard limits):
- **Prefer under ~500 lines** for working source files. Past this, ask whether the file has one responsibility or several.
- **~5000 lines is the split signal** for single-file applications (see `html5-multi-file-architecture.md` Section 1).
- **Cohesion beats arbitrary caps.** A 700-line file with one clear purpose is better than three 250-line files that constantly import each other. Split when you have multiple responsibilities, not when you hit a number.

### Organize by Feature, Not by Type

When AI works on "resources," it should be able to read one folder -- not hunt through controllers/, models/, views/ for the resource-related pieces.

```
# Prefer this (feature-based / vertical slice)
src/
  resources/
    resources.js
    resources.test.js
  buildings/
    buildings.js
    buildings.test.js

# Over this (type-based / horizontal layers)
src/
  controllers/
    resources-controller.js
    buildings-controller.js
  models/
    resources-model.js
    buildings-model.js
```

Shared utilities and cross-cutting concerns go in a `shared/` or `common/` folder. That's fine -- the goal is locality of reference for features, not elimination of all shared code.

**Already practiced:** a large incremental game (22 modules by system), PowerShell module architecture (function files grouped by purpose). See `html5-multi-file-architecture.md` Section 3, `powershell-module-architecture.md` Section 5.

### Interface Compression

An interface definition is a compression algorithm for behavior. Behind it: maybe 500 lines of implementation. At the interface: 10-20 lines that describe what you can do and what you can expect. When AI needs to *use* a module (not modify it), the interface is sufficient.

**What counts as an interface:**
- **PowerShell:** Exported function signatures with help comments (Synopsis, Parameter blocks). The module manifest's `FunctionsToExport` defines the public surface. See `powershell-module-architecture.md` Section 1, `powershell-baseline.md` Section 1.
- **JavaScript:** Module exports + JSDoc on exported functions. A brief comment block at the top of each module file stating purpose and dependencies (see Section 3 below).
- **C#:** Interface types are idiomatic. This layer is free in .NET.

**The principle:** AI reads the interface first. If the interface tells it enough, it doesn't need to load the implementation. Every module should be understandable from its surface without reading its internals.

---

## 2. Code Style

### Function Length

Prefer functions under ~50 lines. Not because 51 is wrong, but because longer functions have more state to hold in context and more branches to reason about.

- A 100-line function that's one coherent algorithm is fine.
- A 100-line function that does three things should be three functions.
- The test: can you describe what the function does in one sentence without using "and"? If not, it's probably doing too much.

### Semantic Naming

Names are AI's primary navigation. Generic names force implementation reads. Descriptive names let AI understand purpose without loading the code.

```
# Good -- purpose is clear from the name
calculateUpkeepCost()
Get-BackupTargets
reserveFundsForBuilding()

# Bad -- forces AI to read the implementation
process()
doWork()
handleIt()
```

**Already covered:** `powershell-baseline.md` Section 1 (Verb-Noun), `powershell-module-architecture.md` Section 3 (project prefix), `javascript-vanilla-standards.md` Section 1 (naming table).

### Explicit Over Clever

AI reasons about explicit code more accurately than clever code. Ternary chains, dense one-liners, and implicit behavior through inheritance or prototype chains force AI to simulate more execution paths mentally.

```javascript
// Explicit -- AI parses this correctly every time
let cost;
if (resource.isRare) {
    cost = basePrice * rarityMultiplier;
} else {
    cost = basePrice;
}

// Clever -- AI may misread precedence or miss edge cases
const cost = resource.isRare ? basePrice * rarityMultiplier : basePrice;
```

The simple ternary above is actually fine -- the principle applies to *chains* and *dense* constructions, not all shorthand. Use judgment: if a human would have to pause to parse it, AI will too.

### Decision-Point Comments

Comment the *why*, not the *what*. The code already says what it does. Comments should explain non-obvious choices so AI (and future sessions) don't undo them.

```javascript
// We use a Set here instead of an array because membership
// checks happen every tick and Set.has() is O(1)
const activeEffects = new Set();

// Robocopy used instead of Copy-Item because it handles
// long paths, retries on network failures, and preserves
// timestamps without additional parameters
```

```powershell
# Config loaded before module import because the module's
# initialization depends on config values being available
$config = Get-Content $configPath | ConvertFrom-Json
Import-Module .\MyModule.psm1
```

**Project-level decisions** go in the CLAUDE.md "Key Architectural Decisions" section (see `project-lifecycle.md` Section 3). **Code-level decisions** go inline, near the code they explain.

---

## 3. Documentation Architecture

### Progressive Disclosure

Don't front-load all information. Structure documentation so AI loads the minimum needed and can drill deeper on demand.

```
Layer 1: CLAUDE.md (~200 lines) -- entry point, commands, architecture overview
Layer 2: Skills / project docs -- loaded when relevant
Layer 3: Satellites / reference -- loaded when specifically needed
```

**Already the strongest pattern in our system.** See `knowledge-architecture.md` for the full three-layer model. Martin Fowler's team independently arrived at the same structure ("Instructions + Skills") and called context engineering "a bunch of markdown files with prompts."

### Module-Level Headers

Every source file should have a brief header that tells AI what it does, what it depends on, and what depends on it. This is cheaper than reading the whole file and lets AI decide whether deeper reading is needed.

**PowerShell** already has this idiomatically -- Synopsis/Description blocks and module manifests. See `powershell-baseline.md` Section 1.

**JavaScript** needs an explicit convention:

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

Keep it to 3-6 lines. Update it when dependencies change. It's a routing table for AI, not comprehensive documentation.

### CLAUDE.md AI Notes

Project CLAUDE.md files should include a brief section noting AI-relevant architecture details that aren't obvious from the code:

- Which files are large and why
- Which modules are tightly coupled (and whether that's intentional)
- Where the complexity lives (so AI reads those files carefully)
- Any patterns that look wrong but are intentional (prevents "helpful" refactoring)

This is a paragraph or a short list, not a section that rivals the rest of the CLAUDE.md. Guidance for adding this to the CLAUDE.md template is in `project-lifecycle.md` Section 3.

---

## 4. Anti-Patterns

### Context Flooding

Loading every file in a directory "just in case" degrades AI performance. Read the interface first. Read the implementation only when modifying it.

### Premature Splitting

Splitting a cohesive 600-line file into three files to hit a line count makes things worse, not better. Each split adds import overhead and forces AI to hold cross-file relationships in context. Split when responsibilities diverge, not when numbers climb.

### Knowledge Dumps in CLAUDE.md

A 500-line CLAUDE.md front-loads tokens before the conversation starts. Keep it under 200 lines. Move everything else to docs that load on demand. See `knowledge-architecture.md` Section 4 for decision rules on what goes where.

### Generic Module Names

`utils.js`, `helpers.ps1`, `common.js` -- these become dumping grounds for unrelated functions. AI can't determine whether it needs the file without reading it. Name modules by purpose: `formatting.js`, `validation.ps1`, `cost-calculations.js`.

### Undocumented Intentional Patterns

Code that looks like a bug but is intentional (a deliberate redundancy, a performance hack, a workaround for a framework limitation) will get "fixed" by AI unless documented. A one-line comment saves a multi-file debugging session.

---

## 5. What This Changes

This skill doesn't introduce new patterns -- it explains *why* existing patterns matter for AI collaboration. The language-specific skills already teach modularity, naming, documentation, and organization. This skill connects them to the central insight: every architectural decision either helps or hurts AI's ability to reason about your code within a finite context window.

When starting a new project, ask:
1. Can AI understand any single module without loading the others?
2. Can AI find what it needs without reading everything?
3. Can AI tell *why* a decision was made without re-deriving it?

If the answer to all three is yes, the architecture is AI-friendly. These principles get you there.

---

## References

- `powershell-module-architecture.md` -- strongest existing example of AI-friendly patterns (module boundaries, public API surface, config externalization)
- `html5-multi-file-architecture.md` -- migration guidance, G namespace, circular dep resolution
- `knowledge-architecture.md` -- progressive disclosure, three-layer model
- `project-lifecycle.md` -- CLAUDE.md template, decision records
- `javascript-vanilla-standards.md` -- naming, JSDoc, module patterns
- `powershell-baseline.md` -- script template, help comments, naming conventions
