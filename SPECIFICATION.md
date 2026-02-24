# Claude Memory — AdventureCraft Project

## Purpose & Context

Manu is developing AdventureCraft, a complex simulation-based strategy game focused on emergent gameplay from composable building blocks. The project aims to create a unified framework where individual entities and groups share the same statistical foundation, enabling realistic AI behavior through condition-based planning rather than scripted sequences. The game spans multiple scales from individual tile-level interactions to stellar empire management, designed to work across genres (medieval fantasy, sci-fi, modern) without mechanical gaps.

Core design philosophy emphasizes emergent complexity from simple rules, favoring computed properties over stored state, composition over inheritance, and data-driven templates over hardcoded behavior. Manu consistently pushes for elegant simplifications, fewer abstractions, and unified approaches that avoid data duplication while maintaining computational efficiency for large-scale simulation with vast numbers of actors.

## Current State

The foundational systems architecture is well-established, featuring a 9-attribute matrix (Physical/Mental/Social × Power/Coordination/Endurance), 7 universal actions with 3 skill approaches each (totaling 22 skills plus Stealth), and a 4-axis relationship system tracking debt, reputation, affection, and familiarity. The execution layer architecture has been consolidated around a unified Node system where everything from individuals to planets is represented as nodes with trait-based composition, using Q16.16 fixed-point arithmetic and deterministic execution with read-write phase separation.

Recent work has focused on refining the technical implementation, including trait-based object systems, virtual item knowledge frameworks, contract mechanics, and authority systems derived from measurable sources (Force, Delegation, Consensus, Tradition). The system supports both precompiled rules for performance and bytecode-like templates for data-driven balancing, with automatic batch mode derivation for multiscale temporal execution.

## Key Learnings & Principles

- Authority ultimately stems from credible threats of sanctions and emerges from actual relationship dynamics rather than being a separate stat.
- Knowledge systems require careful distinction between personal knowledge (what actors know directly) and source knowledge (knowledge about information sources), with both following consistent structural patterns including confidence, secrecy, and temporal metadata.
- The distinction between ownership (legal claims) and possession (physical control) must be maintained throughout all systems.
- Roles function as perpetual reactive rulesets while plans represent finite goal-directed sequences, with orders serving as plan steps assigned across entity boundaries.
- Counter-planning should trigger from observable world state conditions rather than requiring knowledge of specific enemy templates.
- All game mechanics must reference concrete, measurable values rather than abstract concepts.
- Economic value and social consequences like reputation should emerge naturally from system interactions rather than being directly encoded.
- The simulation benefits from treating influence and causality as history-based computations rather than stored data, with caching for frequently accessed relationships.

## Approach & Patterns

Manu works through systematic iterative refinement, consistently questioning redundancies and seeking natural patterns. Design sessions involve progressing from initial sketches through multiple architectural approaches before settling on unified solutions. Technical discussions focus on eliminating verbose wrapper functions in favor of clean syntax similar to TypeScript or C#.

The development approach emphasizes creating comprehensive reference documentation through artifact creation, with multiple iterations to remove unnecessary complexity and maintain focus on conceptual design rather than implementation details. Manu prefers technical precision over prose descriptions, requesting structured identifiers throughout and removal of natural language from data fields.

Problem-solving follows a pattern of establishing foundational frameworks first, then systematically expanding to cover edge cases and integration points. Design decisions consistently favor solutions that work efficiently with Unity's Burst compiler and potentially GPU compute shaders while maintaining clean separation between simulation logic and rendering systems.

## Tools & Resources

The technical stack centers on Unity with Burst compiler optimization, using NativeList and NativeParallelMultiHashMap structures for cache-friendly iteration and O(1) lookups. The architecture supports distributed server designs where spatial partitions can run on different machines with delta merging for cross-partition interactions.

Data structures leverage ECS-style component systems with trait-based interfaces, concrete archetype classes for common entity types, and hybrid approaches using bitmasks with precompiled trait combinations. The knowledge system uses three-tier aging (recent action logs, medium-term deltas, ancient snapshots) with run-length compression for efficiency.

The system references established games like RimWorld, Dwarf Fortress, and Baldur's Gate 3 for validation and comparison, while incorporating domain-specific languages that map to C# subsets for executable rules and AI planning systems.
