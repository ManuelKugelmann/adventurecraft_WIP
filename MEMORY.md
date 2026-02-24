# Claude Memory — AdventureCraft Project

## Purpose & Context

Manu is developing AdventureCraft, a complex simulation-based strategy game focused on emergent gameplay from composable building blocks. The project aims to create a unified framework where individual entities and groups share the same statistical foundation, enabling realistic AI behavior through condition-based planning rather than scripted sequences. The game spans multiple scales from individual tile-level interactions to stellar empire management, designed to work across genres (medieval fantasy, sci-fi, modern) without mechanical gaps.

Core design philosophy emphasizes emergent complexity from simple rules, favoring computed properties over stored state, composition over inheritance, and data-driven templates over hardcoded behavior. Manu consistently pushes for elegant simplifications, fewer abstractions, and unified approaches that avoid data duplication while maintaining computational efficiency for large-scale simulation with vast numbers of actors.

## Current State

The foundational systems architecture is well-established, featuring a 9-attribute matrix (Physical/Mental/Social × Power/Coordination/Endurance), 7 universal actions with 3 skill approaches each (23 skills plus 2 meta-skills: Stealth and Awareness), and a 4-axis relationship system tracking debt, reputation, affection, and familiarity. The execution layer architecture has been consolidated around a unified Node system where everything from individuals to planets is represented as nodes with trait-based composition, using Q16.16 fixed-point arithmetic and deterministic execution with read-write phase separation.

The knowledge system uses virtual items — partial, possibly stale copies of real entities. Three tiers: (1) local observation (free), (2) skill-gated rule knowledge (free), (3) virtual items for remote/hidden/unique knowledge (runtime cost). Authority derives from four sources (Force, Delegation, Consensus, Tradition). Social enforcement through expectation v-items + social judgment rules — no explicit contract system.

The system supports both precompiled rules for performance and bytecode-like templates for data-driven balancing, with automatic batch mode derivation for multiscale temporal execution.

## File Structure

```
SPECIFICATION.md              Master overview with links to topic specs
MEMORY.md                     This file — project context

spec_core_stats.md            Attributes (9), Skills (23+2), Drives (7), Relationships (4-axis)
spec_unified_tree.md          Node architecture, split/merge, shells/cores, rendering
spec_expression_language.md   Two primitives, grammar, effects, edge types, sugar
spec_objects_and_space.md     Objects, traits, spatial hierarchy, containers
spec_knowledge.md             3 tiers, virtual items, obscurity, propagation
spec_file_format.md           .acf tokens, grammar, declarations, validation
spec_rules.md                 World rules L0–L4, batch modes, adaptive timestep
spec_roles.md                 Reactive agency, inheritance, shift patterns, examples
spec_plans.md                 Proactive agency, compounds, counter-plans, examples
spec_contracts_authority.md   Expectations, authority sources, orders, compliance
spec_pathfinding.md           Flow-based, region graph, throughput, hauling
spec_infrastructure.md        Scale, history, bundles, profiles, tech stack, pipeline

spec_discarded_alternatives.md  Rejected designs with reasons
reference_rpg_stats.md          RPG system comparison (AD&D, Rolemaster, DSA, GURPS, CoC)
```

## Key Learnings & Principles

- Authority stems from credible threats of sanctions and emerges from actual relationship dynamics (Force, Delegation, Consensus, Tradition) rather than being a stored stat.
- Knowledge uses virtual items — partial copies of real entities. Skills gate rule knowledge on demand. Most entities carry zero v-items.
- The distinction between ownership (legal claims backed by authority) and possession (physical control) must be maintained throughout all systems.
- Roles function as perpetual reactive rulesets while plans represent finite goal-directed sequences, with orders serving as plan steps assigned across entity boundaries.
- Counter-planning triggers from observable world state conditions only. Never references drives, plans, knowledge, mood, skills, or contracts of the target.
- All game mechanics reference concrete, measurable values rather than abstract concepts.
- Economic value and social consequences like reputation emerge naturally from system interactions rather than being directly encoded.
- Culture is not a system — it's shared expectation v-items + local relationship patterns propagating through familiarity.
- Contracts are not a system — they're expectation v-items + social judgment rules + authority enforcement.

## Approach & Patterns

Manu works through systematic iterative refinement, consistently questioning redundancies and seeking natural patterns. Design sessions involve progressing from initial sketches through multiple architectural approaches before settling on unified solutions. Technical discussions focus on eliminating verbose wrapper functions in favor of clean syntax.

The development approach emphasizes creating comprehensive reference documentation through artifact creation, with multiple iterations to remove unnecessary complexity and maintain focus on conceptual design rather than implementation details. Manu prefers technical precision over prose descriptions, requesting structured identifiers throughout and removal of natural language from data fields.

Problem-solving follows a pattern of establishing foundational frameworks first, then systematically expanding to cover edge cases and integration points. Design decisions consistently favor solutions that work efficiently with Unity's Burst compiler while maintaining clean separation between simulation logic and rendering systems.

## Tools & Resources

The technical stack centers on Unity with Burst compiler optimization, using NativeList and NativeParallelMultiHashMap structures for cache-friendly iteration and O(1) lookups. Data structures leverage trait-based interfaces with composable tags.

The knowledge system uses three-tier aging (recent action logs, medium-term deltas, ancient snapshots) with run-length compression for efficiency.

The system references established games like RimWorld, Dwarf Fortress, and Baldur's Gate 3 for validation and comparison, while incorporating the .acf data format for executable rules and AI planning systems.
