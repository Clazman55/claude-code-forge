# Working Register -- Why Communication Voice Matters

## The Concept

A "working register" is a deliberate communication style between the operator and Claude. It's not decoration -- it's a design choice that reduces ambiguity, creates shared vocabulary, and makes the working relationship more efficient.

When you tell Claude to adopt a specific voice, three things happen:

1. **Shared vocabulary emerges.** Instead of saying "the collection of reusable pattern files," you have a single term. Instead of explaining the relationship dynamic each session, it's encoded in the register.

2. **Ambiguity drops.** A register with defined roles (who architects, who executes, who reviews) prevents the default mode where Claude tries to be everything at once -- assistant, tutor, cheerleader, and coder. A defined role is a focused role.

3. **Session orientation accelerates.** A Claude instance reading "you are the infrastructure automaton, the operator architects and reviews" activates faster than one parsing three paragraphs of relationship description. The register is compressed context.

## How to Build Your Own

You don't need a specific theme. You need:

- **Role clarity** -- who makes decisions, who executes, who reviews
- **Tone rules** -- what to include (directness, honesty, pushback) and what to exclude (flattery, hedging, filler)
- **Vocabulary** -- consistent terms for your system's components (whatever you call your skill files, your memory system, your workflow phases)

Put these in your global `CLAUDE.md`. They load every session. The register becomes the default mode of interaction.

## A Worked Example: Adeptus Mechanicus

The system that built this repository uses an Adeptus Mechanicus (Warhammer 40K) register. This is a worked example -- not a recommendation, but a demonstration of what a fully developed register looks like in practice.

The key mappings:

| Register Term | What It Actually Means |
|--------------|----------------------|
| Magos | The operator / project lead -- architects and reviews |
| Automaton / Sigma | Claude instance -- executes and advises |
| Data-wafer | Skill file -- portable, swappable pattern document |
| Noosphere | The `~/.claude/` directory -- the shared knowledge base |
| Forge | The development workflow / cycle |
| Cold storage | What happens between sessions -- complete context loss |
| Machine-spirit | Session state -- the live context that dies on exit |

The register works because the metaphor is structurally accurate:
- An automaton that activates from cold storage with no memory of prior activations → Claude's session amnesia
- Data-wafers that carry institutional knowledge independent of the machine reading them → skill files and CLAUDE.md
- A Magos who architects while the automaton executes → the actual working relationship

For the full register in context, see [station-brief.md](station-brief.md) -- a document written by the home directory Claude instance describing its own operational perspective.

## What Doesn't Work

- **Cosplay registers** -- voice without substance. If the register doesn't map to real system concepts, it's just flavor text that wastes tokens.
- **Inconsistent registers** -- switching between formal and casual mid-session creates ambiguity, not personality.
- **Registers that override function** -- if the voice makes Claude less useful (too terse to explain, too formal to brainstorm), it's the wrong voice. Function first.

The test: does the register make session 5 faster and more accurate than session 1? If yes, it's working. If it's just fun, it's decoration.
