# Claude Code Forge

**A tested development methodology for Claude Code with persistent knowledge across sessions. Not a prompt library.**

Built from 8 shipped projects (~105,000 lines of production code, 1,120+ automated tests) using the Claude Code VS Code extension. Every pattern here exists because something failed in a real project and the fix was extracted into a reusable skill.

---

## The Problem

Claude Code has session amnesia. Every new conversation starts from zero. Your AI assistant doesn't know what it built yesterday, what architectural decisions were made, or what patterns work in your codebase.

The common response is prompt libraries -- collections of generic instructions you paste into each session. These treat the symptom. The AI gets better instructions, but it still doesn't know your project. Session 5 has no idea what session 2 learned.

This repository treats the disease: a structured methodology where knowledge persists across sessions, accumulates over time, and flows from project-specific observations into portable, reusable patterns.

---

## What This Is / Is Not

**This is:**
- A development methodology -- how to run multi-phase projects with Claude Code
- A knowledge architecture -- how to make session continuity work with markdown files
- A collection of battle-tested skill files -- reusable patterns for specific tech stacks
- A set of slash commands -- automation for the workflow's repetitive steps

**This is not:**
- A prompt library (no generic "be a better coder" instructions)
- Infrastructure (no Docker, no vector databases, no self-hosted embeddings, no MCP servers)
- A framework (no code to install, no dependencies, no build steps)
- A living project (snapshot releases with version tags, not continuous maintenance)

Everything runs on markdown files and Claude Code's built-in features. If you have a text editor and Claude Code, you have everything you need.

---

## The Lifecycle

This is the centerpiece. Every project follows this flow:

| Phase | Where | What Happens |
|-------|-------|-------------|
| **Phase 0** | Home session (`~/`) | Research tech stack, create skill files, scaffold project directory, write CLAUDE.md, create plan file |
| **Handoff** | Open project session | Claude reads CLAUDE.md + MEMORY.md + skill files, reconstructs project state |
| **Phase 1..N** | Project session | Implement features using the self-verification loop (below) |
| **Stabilize** (optional) | Project session | Shelf at a natural pause -- full knowledge capture, skill sync, memory audit |
| **Project Wrap** | Home session | Final review, sync all knowledge layers, move to cold storage |

### The Self-Verification Loop

Every implementation phase runs this loop before moving on:

```
Write code
    → Run tests
        → Review (/forge-review)
            → Fix issues
                → Simplify
                    → Update CLAUDE.md
                        → Sync project skills
                            → Update memory/satellites
                                → Commit
```

This is not optional. The loop is what turns individual sessions into accumulated knowledge. Each step feeds the next session's context.

---

## Three-Layer Knowledge Architecture

Knowledge lives in three layers, each with a different scope and lifetime:

```
┌─────────────────────────────────────────────┐
│  MEMORY.md (routing table)                  │
│  Auto-loaded every session. 200-line limit. │
│  Points to everything else. Never detailed. │
├─────────────────────────────────────────────┤
│  Memory Satellites (project-specific)       │
│  Implementation details: function names,    │
│  file paths, subsystem architecture,        │
│  bug patterns, save version tables.         │
│  Live in ~/.claude/projects/[key]/memory/   │
├─────────────────────────────────────────────┤
│  Skills (portable patterns)                 │
│  Reusable across projects. No project       │
│  names. Technology patterns extracted from  │
│  real implementations. Live in              │
│  ~/.claude/skills/                          │
└─────────────────────────────────────────────┘
```

**Content flows upward as it matures:**
1. Project-specific observation → satellite entry
2. Satellite pattern repeated across phases → project skill annotation
3. Project skill validated across projects → canonical skill update (at project wrap only)

This one-way flow during implementation prevents half-baked patterns from contaminating the canonical skill library.

---

## What's Included

### Skills (20 files)

| Category | Skills | Description |
|----------|--------|-------------|
| **Meta** (5) | project-lifecycle, knowledge-architecture, code-architecture-for-ai, components-index, manual-testing-methodology | How to run projects, manage knowledge, structure code for AI consumption |
| **PowerShell** (7) | powershell-baseline, powershell-wpf, powershell-module-architecture, pester-testing-patterns, powershell-cim-wmi, powershell-html-report, powershell-release-build | Full stack for PowerShell module development with WPF GUIs |
| **JavaScript/HTML5** (5) | javascript-vanilla-standards, html5-canvas-game-development, html5-single-file-architecture, html5-multi-file-architecture, incremental-game-patterns | Vanilla JS/HTML5 Canvas development, single-file and multi-file architectures |
| **C#/.NET** (3) | csharp-baseline, dotnet-project-structure, xunit-testing-patterns | Foundational C#/.NET patterns (less battle-tested than PS/JS stacks -- see note in skills/index.md) |

### Slash Commands (11)

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

### Also Included

- **CLAUDE-TEMPLATE.md** -- blank CLAUDE.md template for new projects
- **CLAUDE-EXAMPLE.md** -- redacted real-world example showing expected depth
- **commands/README.md** -- how to install and use slash commands
- **docs/** -- working register explanation and example

**Stats from the projects that built this:**
- 8 shipped projects across PowerShell, JavaScript, C#, and HTML5
- ~105,000 lines of production code
- 1,120+ automated tests plus manual test suites
- Built entirely with the Claude Code VS Code extension

---

## Quick Start

1. **Fork or clone** this repository
2. **Copy the meta-skills** to your `~/.claude/skills/` directory:
   - `project-lifecycle.md` (the workflow)
   - `knowledge-architecture.md` (the memory system)
   - `code-architecture-for-ai.md` (structuring code for AI sessions)
3. **Copy one language stack** that matches your tech:
   - PowerShell: all 7 `powershell-*.md` + `pester-testing-patterns.md`
   - JavaScript/HTML5: all 5 JS/HTML5 skills
   - C#/.NET: all 3 C# skills (foundational)
4. **Copy slash commands** to `~/.claude/skills/` (see [commands/README.md](commands/README.md) for installation)
5. **Copy CLAUDE-TEMPLATE.md** to your project root as `CLAUDE.md` and fill it in
6. **Run `/orient`** in a new Claude Code session -- it will detect your setup and present status

Start with the meta-skills and one language stack. Add more as you need them. The system is modular by design.

---

## How Skills Work

Skills are not prompts. They are structured reference documents built from empirical data -- what actually worked and what actually broke across real projects.

Each skill follows a consistent format:
- **Intro paragraph** with provenance (where the patterns came from)
- **Table of contents** for navigation
- **Numbered sections** with code examples
- **Anti-pattern tables** -- what NOT to do, with explanations
- **Gotchas** -- non-obvious traps discovered during implementation
- **Checklist** -- verification items for the self-verification loop

Skills reference each other. `powershell-module-architecture` references `powershell-baseline` for coding standards and `pester-testing-patterns` for test structure. `project-lifecycle` references `knowledge-architecture` for the memory system. The cross-references are intentional -- they form a coherent system, not a loose collection.

Skills get `[verified in Phase N]` tags during implementation, marking which patterns have been battle-tested in the current project. These tags are project-specific annotations that accumulate as you work.

---

## Before / After

**Scenario:** Building a PowerShell module that collects hardware inventory and generates HTML reports.

### Without the methodology

Session 1 writes a flat script. Session 2 doesn't know session 1 exists -- rewrites half the logic. Session 3 adds a GUI but doesn't know about the error handling patterns from session 1. Session 5 introduces a bug that session 2 already fixed. Tests are ad hoc. No architectural documentation. Every session starts from "read the whole codebase and figure out what's going on."

### With the methodology

**Phase 0** (home session): Research CIM/WMI data collection patterns, create `powershell-cim-wmi.md` skill, scaffold project with CLAUDE.md documenting architecture, write plan with phase breakdown.

**Phase 1** (project session): `/kickoff` reads the plan, CLAUDE.md, and skills. Claude knows the architecture before writing a line of code. Implements core data collection. `/phase-wrap` updates CLAUDE.md with what was learned, syncs skills, commits.

**Phase 3** (project session): `/orient` reconstructs full project state. Claude reads CLAUDE.md (updated through phases 1-2), memory satellites (bug patterns, design decisions), and skills (annotated with `[verified]` tags). It knows what session 2 learned. It knows what broke and why. It builds on accumulated knowledge instead of starting over.

**Phase 5** (project session): The project has a complete architectural record, tested patterns, verified skills, and a knowledge trail. A new Claude session can orient in seconds and contribute immediately.

---

## Making It Yours

The methodology is technology-agnostic. The skill files are technology-specific. Take the methodology, swap in your own tech stack's skills, and the system works the same way.

One design choice worth explaining: this system uses a consistent communication voice -- a "working register" that shapes how the operator and Claude interact. Ours uses an Adeptus Mechanicus (Warhammer 40K) register. Yours doesn't have to. The point is that a deliberate communication style reduces ambiguity and creates a shared vocabulary for technical concepts. See [docs/working-register.md](docs/working-register.md) for why this matters and how to develop your own, with our register as a worked example documented in [docs/station-brief.md](docs/station-brief.md).

---

## FAQ

**Is this a prompt library?**
No. Prompt libraries are collections of instructions. This is a methodology -- a workflow with feedback loops, knowledge persistence, and accumulated learning across sessions.

**Do I need PowerShell?**
No. The meta-skills (project-lifecycle, knowledge-architecture, code-architecture-for-ai) are technology-agnostic. The PowerShell, JavaScript, and C# skills are worked examples. Create your own language stack skills following the same format.

**Can I add my own skills?**
Yes. That's the point. Start a project, discover patterns, extract them into skills. The methodology includes guidance on when and how to create new skills (see project-lifecycle.md, Section 8).

**How is this different from [other tool]?**
Most alternatives are either prompt collections (no feedback loop, no knowledge persistence) or heavy infrastructure (Docker, vector databases, self-hosted embeddings). This runs on markdown files and Claude Code's built-in features. The differentiator is the tested methodology, not the file format.

**Is this maintained?**
Snapshot releases. The methodology is stable and actively used, but this repo ships versioned snapshots rather than continuous updates. If something significant changes, a new version tag gets cut.

**Does this work with the CLI and VS Code extension?**
Yes. Both interfaces read the same skill files and CLAUDE.md. The methodology works identically in both -- the memory folder bridges session state across interfaces.

---

## Built From Failure

Every skill in this repository exists because something broke.

`powershell-baseline` exists because early sessions produced scripts with no error handling and hardcoded paths. `code-architecture-for-ai` exists because Claude kept "fixing" intentional patterns it didn't understand. `knowledge-architecture` exists because project knowledge kept dying between sessions until the three-layer model was built to stop it.

The anti-pattern tables aren't theoretical. They're postmortems. The gotchas aren't hypothetical. They're things that cost hours to debug.

This isn't a system designed from first principles. It's scar tissue from 8 projects, organized into something reusable. The methodology works because it was built iteratively by the same process it describes.

---

## License

MIT. See [LICENSE](LICENSE).

---

*Built with the Claude Code VS Code extension.*
