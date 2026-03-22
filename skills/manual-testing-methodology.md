# Manual Testing Methodology — Skill Reference

Patterns for manual testing when automated test runners are impractical.
Extracted from real-world experience testing an HTML5 Canvas game (`file://` protocol, TestingGuide.html) and a PowerShell WPF application (GUI, visual verification).

Companion to `pester-testing-patterns.md` (automated testing). See `project-lifecycle.md` for how testing fits into the overall development workflow.

---

## Table of Contents

1. [When to Use Manual Testing](#1-when-to-use-manual-testing)
2. [TestingGuide.html Template](#2-testingguidehtml-template)
3. [Checkbox + Label ID Pattern](#3-checkbox--label-id-pattern)
4. [Action / Expect / If-Fail Structure](#4-action--expect--if-fail-structure)
5. [Copy Report Function](#5-copy-report-function)
6. [localStorage Checkbox Persistence](#6-localstorage-checkbox-persistence)
7. [Phase Progress Counters](#7-phase-progress-counters)
8. [Phase-Based Test Organization](#8-phase-based-test-organization)
9. [Quality Gate Checklist Construction](#9-quality-gate-checklist-construction)
10. [Reset and Utility Buttons](#10-reset-and-utility-buttons)
11. [Gotchas](#11-gotchas)
12. [New TestingGuide Checklist](#12-new-testingguide-checklist)

---

## 1. When to Use Manual Testing

Manual testing is the primary quality gate when automated runners are impractical:

- **`file://` protocol** — Cross-origin restrictions block iframe/XHR between HTML files. Automated test harnesses that load the SUT in an iframe will fail silently.
- **GUI visual verification** — WPF windows, canvas rendering, CSS layout, color schemes. "Does it look right?" cannot be automated without screenshot comparison infrastructure.
- **Single-file apps** — No module system to import, no build tools to hook into, no test framework to load.
- **No build tools** — Projects that must run by double-clicking a file. Adding a test runner means adding a dependency.

Manual testing is NOT a replacement for automated tests. Use Pester (`pester-testing-patterns.md`) or equivalent when the tech stack supports it. Use manual testing for the gaps automated tests cannot cover.

---

## 2. TestingGuide.html Template

A self-contained HTML file with CSS + JS for phase-based manual testing. Copy this skeleton for new projects and customize.

### HTML Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>[Project Name] — Testing Guide</title>
    <style>
        /* See Section 2a: CSS below */
    </style>
</head>
<body>
    <h1>[Project Name] — Testing Guide</h1>
    <p class="subtitle">Manual verification checklist. Check off each step as you verify it.</p>

    <!-- Phase sections go here (see Section 8) -->

    <div class="btn-row">
        <button class="btn btn-primary" id="copyReport">Copy Report</button>
        <button class="btn btn-secondary" id="resetChecks">Reset All Checks</button>
    </div>

    <script>
        'use strict';
        /* See Section 2b: JavaScript below */
    </script>
</body>
</html>
```

### 2a. Core CSS

```css
* { margin: 0; padding: 0; box-sizing: border-box; }

body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background: #0f0f23;
    color: #ccc;
    padding: 20px;
    max-width: 900px;
    margin: 0 auto;
    line-height: 1.6;
}

h1 { color: #667eea; margin-bottom: 5px; }
.subtitle { color: #888; margin-bottom: 25px; font-size: 14px; }

/* Phase container */
.phase { border: 1px solid #333; border-radius: 8px; margin-bottom: 15px; overflow: hidden; }
.phase-header {
    background: #16213e; padding: 12px 15px; cursor: pointer;
    display: flex; justify-content: space-between; align-items: center; user-select: none;
}
.phase-header:hover { background: #1a2744; }
.phase-header .phase-title { font-weight: bold; font-size: 16px; color: #667eea; }
.phase-header .phase-progress { font-size: 13px; color: #888; }
.phase-body { padding: 15px; display: none; }
.phase.active .phase-body { display: block; }

/* Step */
.step {
    margin-bottom: 15px; padding: 10px 15px;
    background: rgba(255, 255, 255, 0.03); border-radius: 6px; border-left: 3px solid #333;
}
.step.done { border-left-color: #2ecc71; opacity: 0.7; }
.step-header { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; }
.step-header input[type="checkbox"] { transform: scale(1.3); cursor: pointer; }
.step-header label { font-weight: bold; cursor: pointer; font-size: 15px; }
.step-detail { margin-left: 30px; font-size: 14px; }
.step-detail .action { color: #667eea; font-weight: bold; }
.step-detail .expect { color: #2ecc71; }
.step-detail .if-fail { color: #e74c3c; }

/* Buttons */
.btn-row { display: flex; gap: 10px; margin-top: 25px; flex-wrap: wrap; }
.btn {
    padding: 10px 20px; border: none; border-radius: 6px;
    font-size: 14px; font-weight: bold; cursor: pointer; color: white;
}
.btn-primary { background: linear-gradient(45deg, #667eea, #764ba2); }
.btn-secondary { background: #333; }
.btn-success { background: #2ecc71; }
.btn:hover { opacity: 0.9; }
```

---

## 3. Checkbox + Label ID Pattern

**Critical:** The `id` and `for` attributes must be separate elements, not wrapped.

```html
<!-- CORRECT: separate input and label linked by id/for -->
<input type="checkbox" id="check-3-1" data-phase="3" data-step="1">
<label for="check-3-1">Settings persist across page reload</label>
```

```html
<!-- BAD: wrapping label around input -->
<label><input type="checkbox"> Settings persist across page reload</label>
```

**Why this matters:** The `generateReport()` function extracts step names using:

```javascript
const label = phase.querySelector(`label[for="${check.id}"]`);
```

If you wrap the input inside the label, there is no `for` attribute, and the report will show raw checkbox IDs instead of human-readable step names.

### ID Convention

Format: `check-{phase}-{step}` where phase can be a number or string (e.g., `check-5.5-3`).

Each checkbox also carries `data-phase` and `data-step` attributes for progress tracking:

```html
<input type="checkbox" id="check-5.5-3" data-phase="5.5" data-step="3">
<label for="check-5.5-3">Demo tail continues moving naturally</label>
```

---

## 4. Action / Expect / If-Fail Structure

Every test step has three parts with distinct CSS classes:

```html
<div class="step-detail">
    <p><span class="action">Do:</span> Open the game HTML file, set board to Screensaver,
       enable demo mode, press F3.</p>
    <p><span class="expect">Expect:</span> Performance overlay shows FPS > 30,
       frame time < 33ms.</p>
    <p><span class="if-fail">If FPS is low:</span> Open F12 → Console, check for
       [PERF] warnings. Screenshot the overlay numbers.</p>
</div>
```

- **Action** (blue `#667eea`): What the tester does. Be specific — name buttons, keys, settings values.
- **Expect** (green `#2ecc71`): What should happen. Observable outcomes, not implementation details.
- **If-Fail** (red `#e74c3c`): Diagnostic steps if the expectation is not met.

**Anti-pattern:** Vague steps like "make sure it works" or "verify the feature." Every step must have a concrete observable outcome.

---

## 5. Copy Report Function

Generates a pasteable text report from the current checkbox state:

```javascript
function generateReport() {
    const lines = ['[Project Name] — Testing Guide Report', ''];
    const phases = document.querySelectorAll('.phase');

    for (const phase of phases) {
        const title = phase.querySelector('.phase-title')?.textContent ?? '';
        const checks = phase.querySelectorAll('input[type="checkbox"]');
        if (checks.length === 0) {
            lines.push(`${title}: (pending)`);
            continue;
        }

        const done = Array.from(checks).filter(c => c.checked).length;
        lines.push(`${title}: ${done}/${checks.length}`);

        checks.forEach(check => {
            const label = phase.querySelector(`label[for="${check.id}"]`);
            const status = check.checked ? 'DONE' : 'TODO';
            lines.push(`  [${status}] ${label?.textContent ?? check.id}`);
        });

        lines.push('');
    }

    return lines.join('\n');
}
```

### Example Output

```
Project Name — Testing Guide Report

Phase 0: Project Infrastructure: 3/3
  [DONE] Open app in browser
  [DONE] Open test suite file in browser
  [DONE] Verify project file structure

Phase 1: Core Engine: 7/8
  [TODO] Run test suite (test suite file)
  [DONE] App renders correctly
  ...
```

This format is designed to be pasted directly into a Claude conversation for reporting test results.

---

## 6. localStorage Checkbox Persistence

Save and restore checkbox state so testers can close the browser and resume:

```javascript
const STORAGE_KEY = 'projectNameTestGuideState';

function loadState() {
    try {
        const saved = localStorage.getItem(STORAGE_KEY);
        return saved ? JSON.parse(saved) : {};
    } catch {
        return {};
    }
}

function saveState(state) {
    try {
        localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
    } catch {
        // localStorage unavailable — silently continue
    }
}
```

### Event Delegation for Checkbox Changes

Use a single delegated handler on `document` instead of per-checkbox listeners:

```javascript
document.addEventListener('change', (e) => {
    if (e.target.type !== 'checkbox') return;
    const currentState = loadState();
    currentState[e.target.id] = e.target.checked;
    saveState(currentState);

    const phase = e.target.dataset.phase;
    if (phase !== undefined) {
        updateProgress(phase);
    }
});
```

### Restore on Load

```javascript
window.addEventListener('load', () => {
    const state = loadState();
    document.querySelectorAll('input[type="checkbox"]').forEach(check => {
        if (state[check.id]) {
            check.checked = true;
        }
    });
    // Update all progress counters
    for (let i = 0; i <= MAX_PHASE; i++) {
        updateProgress(String(i));
    }
});
```

---

## 7. Phase Progress Counters

Each phase header shows a `done/total` counter that updates in real time:

```javascript
function updateProgress(phaseNum) {
    const checks = document.querySelectorAll(`input[data-phase="${phaseNum}"]`);
    if (checks.length === 0) return;

    const done = Array.from(checks).filter(c => c.checked).length;
    const progressEl = document.getElementById(`progress-${phaseNum}`);
    if (progressEl) {
        progressEl.textContent = `${done}/${checks.length}`;
    }

    // Update step styling
    checks.forEach(check => {
        const step = check.closest('.step');
        if (step) {
            step.classList.toggle('done', check.checked);
        }
    });
}
```

Phase headers are collapsible — click to toggle:

```javascript
document.querySelectorAll('.phase-header').forEach(header => {
    header.addEventListener('click', () => {
        header.closest('.phase').classList.toggle('active');
    });
});
```

---

## 8. Phase-Based Test Organization

One phase section per implementation phase. Steps numbered within their phase.

```html
<div class="phase" id="phase-3">
    <div class="phase-header" data-phase="3">
        <span class="phase-title">Phase 3: Settings, UI, Modals</span>
        <span class="phase-progress" id="progress-3">0/9</span>
    </div>
    <div class="phase-body">
        <div class="step" id="step-3-1">
            <div class="step-header">
                <input type="checkbox" id="check-3-1" data-phase="3" data-step="1">
                <label for="check-3-1">Settings persist across reload</label>
            </div>
            <div class="step-detail">
                <p><span class="action">Do:</span> Change board size, close browser, reopen.</p>
                <p><span class="expect">Expect:</span> Board size matches what you set.</p>
            </div>
        </div>
        <!-- More steps... -->
    </div>
</div>
```

### Placeholder Phases

For phases not yet implemented, use a pending message instead of test steps:

```html
<div class="phase-body">
    <p style="color: #888; font-style: italic;">Steps will be added after Phase N implementation.</p>
</div>
```

### Dropped Phases

Mark dropped phases clearly:

```html
<span class="phase-title">Phase 6: Mobile/Touch (DROPPED)</span>
<span class="phase-progress" id="progress-6">skipped</span>
```

---

## 9. Quality Gate Checklist Construction

Quality gate items in CLAUDE.md must be **testable** — each item has an observable action and expected result.

### Good Items (Testable)

```markdown
- [ ] All 18 color schemes render correctly
- [ ] No `var` declarations in codebase
- [ ] Settings persist across page reload
- [ ] Wall=stop, body=stop, tail(len>5)=death
- [ ] Performance overlay toggles on/off (F3 or button)
```

### Bad Items (Not Testable)

```markdown
- [ ] Code quality is good          ← subjective, no observable outcome
- [ ] Performance is acceptable     ← no threshold defined
- [ ] All features work             ← not specific enough
```

### Rules

- Every quality gate item should map to one or more TestingGuide steps
- Items should specify the exact behavior expected, not a general quality attribute
- Include the triggering action where possible (e.g., "F3 or button" for perf overlay)
- Negative tests matter: "Wall collision stops movement (NOT death)" specifies what should NOT happen

---

## 10. Reset and Utility Buttons

### Copy Report with Clipboard Fallback

```javascript
document.getElementById('copyReport').addEventListener('click', (e) => {
    const report = generateReport();
    navigator.clipboard.writeText(report).then(() => {
        e.target.textContent = 'Copied!';
        e.target.classList.add('btn-success');
        setTimeout(() => {
            e.target.textContent = 'Copy Report';
            e.target.classList.remove('btn-success');
        }, 2000);
    }).catch(() => {
        // Fallback for file:// protocol where clipboard API may be blocked
        const textarea = document.createElement('textarea');
        textarea.value = report;
        document.body.appendChild(textarea);
        textarea.select();
        document.execCommand('copy');
        document.body.removeChild(textarea);
        e.target.textContent = 'Copied!';
        e.target.classList.add('btn-success');
        setTimeout(() => {
            e.target.textContent = 'Copy Report';
            e.target.classList.remove('btn-success');
        }, 2000);
    });
});
```

### Reset All with Confirmation

```javascript
document.getElementById('resetChecks').addEventListener('click', () => {
    if (!confirm('Reset all checkmarks? This cannot be undone.')) return;
    document.querySelectorAll('input[type="checkbox"]').forEach(c => c.checked = false);
    saveState({});
    for (let i = 0; i <= MAX_PHASE; i++) {
        updateProgress(String(i));
    }
});
```

---

## 11. Gotchas

| Issue | Symptom | Fix |
|-------|---------|-----|
| Label wraps input instead of `for` attribute | Copy Report shows checkbox IDs instead of step names | Use separate `<input id="...">` + `<label for="...">` |
| Duplicate checkbox IDs across phases | Checkboxes toggle in pairs, state corrupted | Use `check-{phase}-{step}` convention with unique phase numbers |
| `navigator.clipboard` blocked on `file://` | Copy Report silently fails | Include `execCommand('copy')` fallback (Section 10) |
| localStorage full | Checkbox state lost on reload | Wrap in try/catch, continue without persistence (Section 6) |
| Phase numbering gaps (e.g., 5.5) | `updateProgress()` misses the phase | Use string-based phase IDs: `data-phase="5.5"`, call `updateProgress('5.5')` |
| `data-phase` attribute missing on checkbox | Progress counter never updates for that step | Always include `data-phase` on every checkbox |
| Steps too vague | Tester doesn't know what to look for | Every step needs Action + Expect + If-Fail (Section 4) |
| Phase marked "pending" has stale steps from earlier iteration | Tester runs outdated steps | Clear and rewrite phase steps when implementation changes |

---

## 12. New TestingGuide Checklist

When creating a TestingGuide for a new project:

- [ ] Copy the HTML template from Section 2
- [ ] Update `<title>` and `<h1>` with project name
- [ ] Update `STORAGE_KEY` constant with project-specific key
- [ ] Add Phase 0 steps (file opens, structure verified)
- [ ] Add placeholder sections for future phases
- [ ] Verify Copy Report generates correct output (step names, not IDs)
- [ ] Verify localStorage persistence (check boxes, close tab, reopen)
- [ ] Verify collapsible phases toggle correctly
- [ ] Verify Reset All clears all checkboxes and localStorage
- [ ] Confirm the guide works on `file://` protocol (no server needed)
