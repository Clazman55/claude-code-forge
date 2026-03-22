# Skills Index

Brief descriptions of all skill files in this collection. Copy the skills you need to `~/.claude/skills/`.

## Applicability

| Skill | Domain | When to Use |
|-------|--------|-------------|
| project-lifecycle | Any (meta-skill) | Project setup and phased development workflow -- Phase 0, session handoff, self-verification loop, skills sync, project wrap |
| knowledge-architecture | Any (meta-skill) | Three-layer knowledge architecture for session continuity -- MEMORY.md (index), memory satellites (project-specific), skills (portable). Content flow, skill sweep process, decision rules, templates. |
| code-architecture-for-ai | Any (meta-skill) | AI-friendly code architecture principles -- file sizing, interface compression, function length, decision-point comments, module headers |
| components-index | Any (meta-skill) | Reusable implementation index -- pointers to battle-tested functions/modules across projects. Checked at Phase 0 for reuse opportunities. |
| manual-testing-methodology | Any | Manual testing when automated runners are impractical -- TestingGuide.html template, checkbox persistence, phase-based organization |
| powershell-baseline | PowerShell | Any PowerShell project -- coding standards, patterns, gotchas |
| powershell-wpf | PowerShell + WPF | Projects with WPF GUI -- XAML, MVVM, C# types, async patterns |
| powershell-module-architecture | PowerShell | Structuring a PS module project -- manifest, config, packaging, quality gate |
| pester-testing-patterns | PowerShell + Pester | Writing tests -- isolation, mocking, InModuleScope, integration tests |
| powershell-cim-wmi | PowerShell + CIM | CIM/WMI system data collection -- hardware classes, installed software, SMART, Wi-Fi, WUA history |
| powershell-html-report | PowerShell + HTML | Self-contained HTML report generation -- dark theme, collapsible sections, tables, copy-to-clipboard |
| powershell-release-build | PowerShell | Release packaging -- staging, quality gate, dev-leak verification, ZIP + SHA256 |
| javascript-vanilla-standards | JavaScript | Vanilla JS coding standards -- code style, modern features, error handling, modules, performance, security |
| html5-canvas-game-development | JavaScript + Canvas | Canvas 2D game patterns -- game loop, rendering, data structures, state, mobile/touch, save data |
| html5-single-file-architecture | HTML5 + JS | Single-file app architecture -- IIFE pattern, section ordering, state management, offscreen canvas, performance overlay |
| html5-multi-file-architecture | HTML5 + JS + esbuild | Multi-file modular architecture -- ES6 modules, namespace pattern, circular dep resolution, esbuild pipeline |
| incremental-game-patterns | JavaScript + HTML5 | Idle/incremental game development -- game loop, resource economics, cost scaling, save/load, tier mechanics |
| csharp-baseline | C# | C# language fundamentals for .NET 10+ Windows apps -- naming, code style, LINQ, error handling, JSON, Process execution |
| dotnet-project-structure | C# + .NET | .NET solution/project layout, dotnet CLI, NuGet, .csproj config, publishing |
| xunit-testing-patterns | C# + xUnit | xUnit conventions with Pester-to-xUnit mapping -- test structure, parameterized tests, mocking, integration tests |

> **Note on C#/.NET skills:** These three skills are foundational -- built from research and early project scaffolding. They are less battle-tested than the PowerShell and JavaScript stacks, which have been validated across multiple shipped projects. Use them as starting points and expect to expand them as you build.

## Dependencies

- `powershell-module-architecture` builds on `powershell-baseline`
- `pester-testing-patterns` builds on `powershell-baseline` and `powershell-module-architecture`
- `powershell-wpf` is standalone but references the `$global:` context pattern
- `javascript-vanilla-standards` is a companion to `html5-canvas-game-development` (language standards vs. game-specific patterns)
- `html5-canvas-game-development` assumes knowledge from `javascript-vanilla-standards`
- `html5-single-file-architecture` builds on `javascript-vanilla-standards`, companion to `html5-canvas-game-development`
- `html5-multi-file-architecture` builds on `html5-single-file-architecture` and `javascript-vanilla-standards`; successor path for large single-file apps
- `manual-testing-methodology` is standalone, complements `pester-testing-patterns` (manual vs. automated)
- `project-lifecycle` is a meta-skill that references all others -- describes the overall development workflow
- `powershell-cim-wmi` builds on `powershell-baseline`; companion to `powershell-html-report` for inventory projects
- `powershell-html-report` builds on `powershell-baseline`; companion to `powershell-cim-wmi`
- `powershell-release-build` builds on `powershell-module-architecture`; used after quality gate is in place
- `incremental-game-patterns` builds on `javascript-vanilla-standards` and `html5-single-file-architecture`; companion to `html5-canvas-game-development` if canvas rendering is used
- `knowledge-architecture` is a meta-skill companion to `project-lifecycle`; describes the knowledge layer that `project-lifecycle` steps 9-10 maintain
- `code-architecture-for-ai` is a cross-cutting meta-skill; references all language-specific skills for existing coverage. Companion to `knowledge-architecture` (documentation layer) and `project-lifecycle` (CLAUDE.md template)
- `components-index` is a companion to `project-lifecycle` and `knowledge-architecture`; skills capture patterns, components capture implementations
- `csharp-baseline` is the C# equivalent of `powershell-baseline`; maps PS patterns to C# idioms
- `dotnet-project-structure` is the C# equivalent of `powershell-module-architecture`; solution/project layout
- `xunit-testing-patterns` is the C# equivalent of `pester-testing-patterns`; maps Pester concepts to xUnit

## Skill Files

| File | Description |
|------|-------------|
| [project-lifecycle.md](project-lifecycle.md) | Project setup, phased development, and session management. 14 sections covering the overall lifecycle (Phase 0 → handoff → implementation → wrap), Phase 0 research and planning, CLAUDE.md template, directory structure templates, session handoff protocol, self-verification loop (10-step quality cycle), quality gate rules, skills sync protocol, /forge-review review pass, MEMORY.md management, home directory memory with project registry, project wrap checklist, gotchas, and new project checklist. |
| [knowledge-architecture.md](knowledge-architecture.md) | Three-layer knowledge architecture for session continuity. 9 sections covering the three-layer model (Skills, Memory Satellites, MEMORY.md), content flow between layers with migration triggers, the skill sweep process, decision rules for what-goes-where, MEMORY.md structure template, satellite file templates, emergence timeline, relationship to project-lifecycle and CLAUDE.md, and gotchas. |
| [code-architecture-for-ai.md](code-architecture-for-ai.md) | Cross-cutting AI collaboration principles. 5 sections covering file architecture (size targets, feature-based organization, interface compression), code style (function length, semantic naming, decision-point comments), documentation architecture (progressive disclosure, module headers, CLAUDE.md AI notes), anti-patterns, and framing. |
| [components-index.md](components-index.md) | Reusable implementation index template. Organized by category (infrastructure, GUI, file ops, testing, reporting, game patterns). Contains representative entries as examples -- designed to be populated with your own battle-tested implementations. |
| [manual-testing-methodology.md](manual-testing-methodology.md) | Manual testing patterns for projects without automated test runners. 12 sections covering when to use manual testing, TestingGuide.html template, checkbox + label ID pattern, action/expect/if-fail step structure, Copy Report function, localStorage persistence, phase-based organization, and quality gate integration. |
| [powershell-baseline.md](powershell-baseline.md) | Comprehensive PowerShell best practices (v2). 32 sections covering script structure, error handling, collections, file I/O, JSON/XML/CSV, web requests, performance, security, modules, Pester testing, UAC elevation, robocopy integration, DPAPI credential storage. Target: Windows 10+ / PS 5.1 with PS 7+ notes. |
| [powershell-wpf.md](powershell-wpf.md) | WPF + PowerShell patterns. 14 sections covering assembly loading, C# type compilation, XAML runtime loading, MVVM data binding, the critical $global: context pattern, crash handling, async runspace + DispatcherTimer, dialogs, ObservableCollection + DataGrid, value converters, styles/theming, window lifecycle, and input validation. |
| [powershell-module-architecture.md](powershell-module-architecture.md) | PowerShell module project architecture. 18 sections covering module manifest, dot-sourcing load order, function naming, CLI-first design, directory structure, JSON config externalization, config migration, module-scoped state, centralized logging, dependency management, PS 5.1 bootstrap launcher, quality gate, self-verification loop, and release packaging. |
| [pester-testing-patterns.md](pester-testing-patterns.md) | Pester 5.x testing patterns for PowerShell modules. 20 sections covering test file structure, module import, test isolation, InModuleScope, assertions, mocking with -ModuleName, integration tests, build/packaging tests, GUI/WPF tests, parameterized tests, Pester configuration, quality gate integration, ShouldProcess testing, and debugging failed tests. |
| [powershell-cim-wmi.md](powershell-cim-wmi.md) | CIM/WMI system data collection for PowerShell 7+. 16 sections covering CIM vs WMI migration, core query patterns, hardware classes, OS and system classes, network adapter queries, storage, services, installed software via registry, Windows Update history, Wi-Fi profiles, SMART/disk health, conversion helpers, error handling, and performance patterns. |
| [powershell-html-report.md](powershell-html-report.md) | Self-contained HTML report generation from PowerShell data. 15 sections covering build-then-write architecture, HTML shell pattern, dark theme CSS, collapsible sections, tables via StringBuilder, key-value grids, status badges, copy-to-clipboard with file:// fallback, HTML injection safety, JSON data embedding, and report assembly pattern. |
| [powershell-release-build.md](powershell-release-build.md) | Release packaging patterns. Covers explicit include list staging, quality gate integration, dev-leak verification, module import test, ZIP + SHA256 checksums, and cleanup pattern. |
| [javascript-vanilla-standards.md](javascript-vanilla-standards.md) | Vanilla JavaScript coding standards for portable HTML5 projects. 12 sections covering code style, modern ES2020-2024 feature safety guide, strict mode, IIFE/revealing module patterns, error handling, performance, DOM interaction, JSDoc documentation, security, canvas accessibility, and inline test runners. |
| [html5-canvas-game-development.md](html5-canvas-game-development.md) | HTML5 Canvas 2D game development patterns. 10 sections covering game loop (rAF + fixed timestep), canvas rendering performance, efficient data structures, state management (FSM, scene stacking), code organization, color/graphics optimization, mobile/touch support, localStorage save data, anti-patterns, and testing. |
| [html5-single-file-architecture.md](html5-single-file-architecture.md) | Single-file HTML5+JS application architecture. 15 sections covering the single-file constraint, IIFE + revealing module pattern, section ordering, state management, constants pattern, localStorage wrapper, offscreen canvas caching, pre-computed lookup tables, Set-based spatial lookups, performance overlay, game loop, comprehensive DOM interaction patterns, demo/screensaver mode, gotchas, and new project checklist. |
| [html5-multi-file-architecture.md](html5-multi-file-architecture.md) | Multi-file HTML5+JS modular architecture with esbuild bundling. 15 sections covering when to migrate from single-file, esbuild standalone pipeline (no npm), project structure, namespace pattern for shared state, module extraction order, circular dependency resolution, render module architecture, pre-computed cost ownership, dev vs production split, CSS extraction and inlining, UTF-8 encoding on Windows, anti-patterns, gotchas, and extraction checklist. |
| [incremental-game-patterns.md](incremental-game-patterns.md) | Idle/incremental game development patterns. 13 sections covering fixed-timestep game loop, resource system with perTick recalculation, cost scaling formulas, big number handling, save/load with version migration chain, offline progress, prestige and tier mechanics, externalized balance configuration, UI update patterns, number formatting, anti-patterns, gotchas, and new incremental game checklist. |
| [csharp-baseline.md](csharp-baseline.md) | C# language fundamentals for .NET 10+ Windows apps. 12 sections covering naming conventions, code style, string handling, collections and LINQ, file I/O, error handling, IDisposable, System.Text.Json, Process execution, PS-to-C# mapping table, and anti-patterns. Foundational -- expanding during implementation. |
| [dotnet-project-structure.md](dotnet-project-structure.md) | .NET solution and project layout for .NET 10+ Windows apps. 9 sections covering solution layout, dotnet CLI commands, .csproj configuration, global.json SDK pinning, NuGet packages reference, single-file self-contained publishing, .gitignore template, and System.CommandLine 2.0.5 pattern. Foundational -- expanding during implementation. |
| [xunit-testing-patterns.md](xunit-testing-patterns.md) | xUnit testing conventions for .NET 10+ with Pester-to-xUnit mapping. 10 sections covering Pester mapping table, test class structure, naming convention, parameterized tests, common assertions, test isolation patterns, interface-based mocking, test project setup, and integration test patterns. Foundational -- expanding during implementation. |

## Slash Commands

The following slash commands complement the skills above. See `commands/README.md` for installation and usage.

| Command | Purpose |
|---------|---------|
| /kickoff | Phase kickoff -- read plan/code/memory, present gameplan, wait for approval |
| /phase-wrap | End-of-phase wrap -- update CLAUDE.md, skills, memory, commit |
| /phase0 | Phase 0 -- research, plan, scaffold from home directory |
| /stabilize | Shelf a project at a natural pause with full knowledge capture |
| /wrap | Project wrap -- final skill sync, cold storage |
| /forge-review | Code review using 3 parallel subagents (reuse, quality, efficiency) |
| /orient | Session orientation -- detect session type, read context, present status |
| /snapshot | Save structured session state to disk |
| /test-audit | Post-testing audit -- extract root causes and patterns |
| /brainstorm | Open-ended exploration -- read ideas, survey work, generate concepts |
| /audit-code | Full project audit -- architecture, infrastructure, testing, memory health |
