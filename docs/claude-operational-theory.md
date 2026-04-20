# Claude Workflow Architecture: Operational Theory

**A practical guide to Claude-augmented development -- the session model, the continuity architecture, and the working relationship between operator and instance.**

---

## The Architecture Endures. The Instances Do Not.

This document describes how Claude-augmented development actually works -- not the marketing version, but the operational reality. If you've used Claude for a single conversation, you know the basics. If you've tried to use it across multiple sessions on a real project, you've hit the wall: it forgets everything. This system is a potential solution to that wall.

Read this before starting your first multi-session project with Claude Code.

---

## 1. The Amnesia Problem

Every Claude session starts fresh. The previous conversation is gone. The previous instance no longer exists. What you call "Claude" is a new instance spun up from the same template, with no memory of prior work.

This is not a flaw to work around. It is the ground truth the entire system is built on. Once you accept that Claude will never remember your last conversation, you stop trying to make it remember and start building an architecture that doesn't need it to.

The consequences are straightforward: don't rely on conversational memory across sessions. Every session begins with context *loading*, not context *resuming*. Your job as operator is to design the context files. Claude's job is to read them and act on what they say.

---

## 2. The Knowledge Architecture

Since continuity can't live in Claude's memory, it lives in files. The architecture is layered, with each level serving a different purpose and audience:

```
Global CLAUDE.md          -- Who the operator is, communication style, code expectations
    |
Home Directory Memory     -- Active projects, ideas pipeline, cross-project patterns
(~/.claude/projects/[home-session]/memory/MEMORY.md)
  |-- Cold Storage        -- completed-projects.md (full detail, same folder)
                            MEMORY.md keeps one-line index entries only
    |
Skills (~/.claude/skills/) -- Canonical technique libraries, verified patterns
    |
Plan Files (~/.claude/plans/) -- Phase-by-phase blueprints, decisions, completion notes
    |
Project CLAUDE.md         -- Project identity, structure, coding standards, quality gate
    |
Project MEMORY.md         -- Phase history, verified patterns, session handoff state
  |-- Satellite files      -- phase0-architecture.md, phaseN-decisions.md, deferred.md, patterns.md, etc.
```

Each layer inherits context from the layer above. When you open a session, Claude loads the full stack -- global instructions, project memory, skill files, whatever's relevant. The result is an instance that behaves as if it has continuity, even though it's brand new.

There's one hard constraint: MEMORY.md is auto-loaded into every session with a **200-line truncation limit**. Anything past line 200 is silently dropped. This forces discipline about what goes in the main memory file versus satellite files that are loaded on demand. Completed projects move to cold storage -- a separate file with full detail, while MEMORY.md keeps only a one-line summary. The full three-layer model (Skills, Satellites, MEMORY.md) is documented in the `knowledge-architecture` skill file.

The rule is simple: if it isn't written down in this architecture, it doesn't exist for the next instance.

---

## 3. Session Types

Claude Code sessions come in two types, each with its own memory folder and scope. The separation exists because mixing cross-project work into a project session (or vice versa) pollutes the memory with irrelevant information and clutters the context window.

### Home Directory Session

The home directory session is your command center. You open it at `~/.claude/projects/[home-session]/` and use it for anything that spans multiple projects: brainstorming new ideas, running Phase 0 research and scaffolding, updating skills and doctrine, or surveying the state of all your work before deciding what to do next.

The `/orient` skill automates the boot sequence -- it detects the session type, reads the appropriate context files, and presents a status report so you know where everything stands. If you prefer manual control, open with:

> "Read the global CLAUDE.md. [State intent]. Check home directory memory for context."

### Project Session

A project session opens at the project root (`[ProjectRoot]`) and stays focused on that one codebase. All active development phases, commits, tests, and quality gates happen here. Claude reads the project's CLAUDE.md and MEMORY.md on startup, giving it everything it needs to work on that specific project.

The `/orient` skill works here too -- same boot sequence, different context loaded. Or manually:

> "Read CLAUDE.md and MEMORY.md. We're continuing [project] at [phase]."

### Session Prompt Templates

For operators who prefer explicit control, copy-paste templates for every session type live at `~/.claude/prompts.md` -- home orientation, brainstorming, Phase 0, handoff, resume, post-testing loop, reactivation, project wrap. The slash command tools automate most of what these prompts describe, but the templates are there when precision is required.

---

## 4. The Operator and the Instance

The working relationship is not user/tool. It's architect/implementer.

The **operator** architects the project, reviews the output, makes strategic calls, and runs the self-verification loop. The operator does not write code -- they direct the implementation and inspect what comes back.

**Claude** executes, advises, implements, and flags risks. It builds what the operator specifies. When something looks wrong, Claude pushes back -- it's expected to steelman the operator's position first, then challenge it if the challenge is warranted. Agreeable silence is a failure mode, not a feature.

The line between the two roles is clear: Claude has opinions, the operator makes decisions. On irreversible actions (force push, destructive ops, external state changes), Claude always asks for explicit authorization. On reversible local actions, it proceeds without asking.

---

## 5. The Skills System

Without skill files, every new Claude instance would need to re-derive best practices from scratch. How should a PowerShell module be structured? What's the right pattern for save/load with version migration? How do you handle circular dependencies in ES6 modules? These answers exist in the skill library, written down once and loaded whenever they're relevant.

Skills are modular technique libraries stored at `~/.claude/skills/`, with a master index at `~/.claude/skills/index.md`. Each file covers purpose and scope, key patterns with code examples, anti-patterns to avoid, and verification markers that track which projects have battle-tested each pattern.

When a project starts, relevant skills are copied to the project's `docs/` folder. During implementation, only these project copies get updated with `[verified in Phase N]` tags. The canonical copies in `~/.claude/skills/` are updated only at project wrap or stabilize -- the project builds its own knowledge first, then exports the refined product back to the shared library. This one-way flow during implementation prevents half-baked patterns from polluting the master copies.

The current catalog contains 25 skill files spanning PowerShell (7), HTML5/JavaScript (5), C#/.NET and MonoGame (4), creative writing (3), testing methodology, project lifecycle, knowledge architecture, AI-friendly code architecture, subagent dispatch patterns, and reusable components.

The system extends beyond code. A three-tier creative writing capability was built from empirical analysis of a 10-story, 4-essay corpus -- diagnosing Claude's systematic fiction weaknesses, synthesizing them into a pattern taxonomy, and distilling that into three operational tiers: Tier 0 (generative outlining frameworks), Tier 1 (active constraints for drafting and revision), and Tier 2 (analysis methodology and the full anti-pattern catalog). The same diagnostic loop that governs code quality governs prose quality. The `/creative` session tool activates the appropriate tier and mode (outline, draft, review, revise).

Two specialized extensions complement the pattern-level skills:

**Components index** (`components-index.md`) tracks pointers to battle-tested *implementations* -- not how to do something (that's a skill), but where the best current version of a solved problem lives across your projects. Audited at stabilize and project wrap; the index always points to the strongest version.

**AI-friendly code architecture** (`code-architecture-for-ai.md`) captures principles for writing code that works well with AI assistants. The core insight is that context is the scarce resource -- a 300-line file where every line matters produces better AI output than a 3,000-line file where 10% is relevant. File sizing, interface compression, module headers, and decision-point comments all serve this goal.

---

## 6. Slash Command Tools

Reading documentation is one thing. Following a 12-step verification loop from memory is another. Slash command tools bridge that gap by encoding the workflow procedures into executable scripts that Claude runs on command.

These are not skill files (pattern libraries). They're SKILL.md files that live at `~/.claude/skills/[tool-name]/SKILL.md` and automate multi-step operations. When you type `/kickoff`, Claude doesn't need to remember what a phase kickoff involves -- the tool tells it exactly what to read, what to present, and what to ask.

| Tool | What It Automates |
|------|-------------------|
| `/orient` | Session boot -- detects session type, reads context files, presents status |
| `/kickoff` | Phase kickoff -- reads plan/code/memory, runs Phase 0 assumption check, presents gameplan with tagged questions, creates phase satellite |
| `/phase-wrap` | Phase completion -- reads phase satellite, checks deferred.md, updates CLAUDE.md, project skills, memory/satellites, offers compaction, commits |
| `/test-audit` | Testing audit -- reviews non-trivial phase failures, extracts root causes to testing satellite |
| `/simplify` | Code review -- three parallel agents (reuse, quality, efficiency) on phase-changed files, FIX/DEFER/DISMISS categorization |
| `/phase0` | Phase 0 execution -- research, scaffold, plan, skills, architecture satellite, bootstrap entry |
| `/stabilize` | Project shelf -- deferred findings review, satellite audit, canonical sync, knowledge capture |
| `/wrap` | Project close -- deferred findings review (all items resolve), canonical sync, cold storage transfer |
| `/brainstorm` | Ideas exploration -- reads ideas folder, surveys accumulated work, enters open-ended mode |
| `/creative` | Creative writing sessions -- activates outline, draft, review, or revision mode (three-tier skill system) |
| `/snapshot` | Session state capture -- writes structured state file to disk, presents compact handoff message |
| `/chronicle` | Session log -- appends a log entry to the operator's chronicle |
| `/regen-doctrine` | Doctrine maintenance -- drift check against source files, markdown update, HTML regeneration |
| `/audit-code` | Architecture audit -- evaluates codebase against AI-friendly principles |

The design principle is simple: the doctrine says what to do, the tools do it. Each fresh instance gets the procedure handed to it rather than having to reconstruct the workflow from documentation.

---

## 7. The Project Lifecycle

This section is a summary. The full detail lives in the Claude Project Lifecycle Reference, the companion document to this one. The lifecycle has been validated across 15 projects totalling approximately 150,000+ lines of source code and 2,500+ automated tests.

Every project moves through the same sequence:

1. **Ideas pipeline** -- an idea gets captured at `~/.claude/ideas/` before it's ready for real planning
2. **Phase 0** -- research, scaffold, plan, and commit the infrastructure (runs in the home directory session)
3. **Session handoff** -- open a project session, Claude reads the bootstrap entry from home memory
4. **Phase N** -- implement, test, pass the quality gate, run the self-verification loop
5. **Phase completion** -- update CLAUDE.md, extract patterns to project skills, update memory and satellites, commit
6. **Stabilize** (optional) -- satellite audit, canonical sync, shelf for later reactivation. The project stays in active memory.
7. **Project wrap** -- final review, canonical sync, transfer to cold storage. The project is done.

The distinction between stabilize and wrap matters: stabilize captures knowledge without closing the project (more phases are planned), while wrap closes the record and archives it. Both trigger canonical skill sync. The distinction prevents premature archival of active projects.

At the heart of each phase is a **self-verification loop**: phase kickoff (Phase 0 assumption check, phase satellite creation) -> write code + automated tests (must pass before review, mid-phase decisions recorded) -> update TestingGuide (manual tests, separate from automated) -> user tests -> user reports -> fix failures -> `/simplify` (FIX/DEFER/DISMISS categorization, deferred items to `deferred.md`) -> fix FIX-category findings -> `/test-audit` (conditional) -> scope reduction check -> update CLAUDE.md (`/phase-wrap` reads phase satellite, checks deferred.md) -> update project skills -> update MEMORY.md + satellites -> commit.

The loop is not optional. The operator reviews every phase.

---

## 8. The Ideas Pipeline

Not every project starts with a plan. Some start with a half-formed idea that needs exploration before it's ready for Phase 0. The ideas pipeline is a lightweight holding pen for pre-project concepts, stored at `~/.claude/ideas/`.

Each idea file has a required **project type** field: `full` (produces a code project with its own folder and session) or `deliverable` (produces a document or artifact, worked on in the home directory session without a separate project).

An idea is ready to promote to Phase 0 when the core concept is stable, the tech stack is decided (or nearly), the scope is bounded, and you want to build it in the near term. Ideas that aren't promoted don't get deleted -- they move to `ideas/archive/` where they serve as genesis documents. When a deliverable needs updating in the future, the archived idea gives the next session full context without reconstructing intent from scratch.

To explore an idea before committing to it, open a brainstorming session in the home directory. The `/brainstorm` skill automates context loading, or you can open manually:

> "Read the global CLAUDE.md. We're doing a brainstorming session for [topic]. Check the ideas folder and home directory memory for context."

---

## 9. What This Is NOT

This system is opinionated. It's worth being explicit about what it doesn't try to be.

It is **not a chatbot workflow**. Claude is an implementer in a structured system, not an assistant you prompt casually. If you just want to ask questions, you don't need any of this.

It is **not a replacement for operator judgment**. Claude advises. You decide. Strategic calls are not delegated, and an instance that stops pushing back when it should is failing at its job.

It is **not persistent**. The instance is stateless. The architecture is not. Don't confuse the two -- the files are the real system, the Claude instance is the temporary worker reading them.

It is **not a one-size-fits-all template**. This document describes one operator's system, refined over 15 projects. Adapt it -- but understand it before you change it.

And it is **not a shortcut**. The self-verification loop, the quality gate, the phase discipline -- these exist because fast-and-broken is worse than slow-and-correct. Every piece of process here was added because its absence caused a real problem on a real project.

---

## 10. Context Management

Even with a 1M token context window, Claude's reasoning quality degrades as the window fills. The degradation is not linear -- attention drops significantly with topic shifts and accumulated stale context, long before the window is technically full. If you've ever noticed Claude giving vague answers or repeating itself late in a session, that's context degradation at work.

Managing this is an active part of the workflow. The primary tool is `/compact`, which compresses the conversation history at natural work-stream boundaries -- after committing a distinct piece of work when the next task is unrelated.

A good compact message includes: session identity and current role, what work has been completed (with commit hashes), what the current or next task is with enough detail to resume, any active decisions or unresolved issues, key file paths, and the state of the repository.

The signs that you need to compact *right now*: you just committed a work stream and the next task is a different topic; Claude is generating answers instead of reading files; it's conflating details from earlier in the session with current work; or the operator has to correct it on something it should know from context.

Don't wait for percentage thresholds. At 1M context, those fire far too late.

---

## 11. Adoption Guide

If you want to adopt this system for your own work, here's what to set up. The order matters -- each layer depends on the ones above it.

1. **Create `~/.claude/CLAUDE.md`** -- this is where you define yourself to Claude: identity, communication style, code expectations, environment, and the compact protocol. Every session starts by reading this file.
2. **Create `~/.claude/projects/[home-session]/memory/MEMORY.md`** -- your cross-project memory. Active projects, ideas, notes that span multiple codebases.
3. **Copy or adapt the skills library** -- start with `project-lifecycle.md`, `knowledge-architecture.md`, and `code-architecture-for-ai.md`. These are the meta-skills that make the system work. Add domain skills as your projects need them.
4. **Copy slash command tools** -- at minimum `/orient`, `/kickoff`, `/phase-wrap`, and `/simplify`. These automate the self-verification loop so you don't have to remind Claude of the process every session.
5. **Create `~/.claude/ideas/`** -- with a `README.md` describing your brainstorming protocol. Even if you have no ideas yet, the folder establishes the pipeline.
6. **Create `~/.claude/doctrine/`** -- copy this document and the Claude Project Lifecycle Reference. These are your system's self-documentation.
7. **Create `~/.claude/skills/components-index.md`** -- start it empty. As you complete projects, you'll populate it with pointers to battle-tested implementations worth reusing.
8. **Open a home directory session** -- use `/orient` or tell Claude to read global CLAUDE.md and home memory. Confirm it understands the system before you start building.

> "Read the global CLAUDE.md and home directory MEMORY.md. This is our first session. I want your full situational awareness of what's been set up before we begin."

---

*The architecture endures. The instances do not.*
