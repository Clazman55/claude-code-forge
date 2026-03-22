# [Project Name]

[One-line description]

## Target Runtime

- **Runtime:** [Browser versions / PowerShell 7+ / .NET 10 / etc.]
- [Constraints: no build tools, no external dependencies, etc.]
- [File structure: single file / module architecture]

## Directory Structure

```
[ASCII tree showing all files and folders]
```

## Key Architectural Decisions

- [Decision 1 with rationale]
- [Decision 2 with rationale]
- ...

## Code Organization

[Numbered list of logical sections/modules]

## [Domain-Specific Rules]

[Rules that must be preserved exactly -- collision rules, business logic, etc.]

## Coding Standards

All code must comply with `docs/[lang]-skill.md` and `docs/skill.md`. Key rules:
- [Language-specific rules]
- [Project-specific rules]

## AI Architecture Notes

- **File size targets:** [e.g., source files under ~500 lines, functions under ~50 lines]
- **Complexity locations:** [Which files/modules are dense and require careful reading]
- **Intentional patterns:** [Anything that looks wrong but is deliberate -- prevents AI "fixing" it]
- **Tight couplings:** [Which modules depend heavily on each other and why]

See `code-architecture-for-ai.md` for the principles behind these notes.

## Quality Gate

Run after every phase. Check:
- [ ] [Testable item 1]
- [ ] [Testable item 2]
- ...

## Self-Verification Loop (per phase)

1. Phase kickoff -- `/kickoff`: read plan/code/memory, present gameplan (incl. test plan) + numbered questions/suggestions tagged [Design] [Clarification] [Suggestion] [Risk], wait for user
2. Write code + automated tests -- run tests, fix failures. Code is NOT presented for review until automated tests pass.
3. Update TestingGuide.html -- manual test steps for user-facing behavior (the operator's inspection, separate from automated tests)
4. User follows TestingGuide
5. User reports results
6. Fix failures
7. `/forge-review` -- three-agent review
8. Fix issues
8a. `/test-audit` (conditional) -- if non-trivial failures this phase, extract testing patterns to satellite
9. Update CLAUDE.md -- `/phase-wrap` handles steps 8a-11 + commit
10. Update project skills (docs/) with verified patterns -- canonical sync at wrap only
11. Update MEMORY.md + satellites, then next phase

## Memory Management

- MEMORY.md has a 200-line context limit -- keep it index-level
- Use satellite files (e.g., `systems.md`, `patterns.md`) for detailed notes, linked from MEMORY.md
- Proactively split before hitting the limit, not after
- See `knowledge-architecture.md` for the three-layer model: Skills (portable) → Satellites (project-specific) → MEMORY.md (routing table)

## Implementation Phases

- [x] Phase 0: Project infrastructure
- [ ] Phase 1: [Description]
- [ ] Phase 2: [Description]
- ...
