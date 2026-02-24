# Claude Memory — AdventureCraft Project

## Purpose & Context

Manu is developing AdventureCraft, a complex simulation-based strategy game focused on emergent gameplay from composable building blocks. The project aims to create a unified framework where individual entities and groups share the same node type and trait-based composition, enabling realistic AI behavior through condition-based planning rather than scripted sequences. The game spans multiple scales from individual tile-level interactions to stellar empire management, designed to work across genres (medieval fantasy, sci-fi, modern) without mechanical gaps.

Core design philosophy emphasizes emergent complexity from simple rules, favoring computed properties over stored state, composition over inheritance, and data-driven templates over hardcoded behavior. Manu consistently pushes for elegant simplifications, fewer abstractions, and unified approaches that avoid data duplication while maintaining computational efficiency for large-scale simulation with vast numbers of actors.

## Current State

The architecture is well-established around a single Node type (~20 bytes, blittable) with trait-based ECS composition. Two trees: ContainerNode (physical containment) and ParentNode (organizational hierarchy). All simulation state in Fixed Q16.16. Rules authored in `.acf`, compiled to structured IR (RuleCondition/RuleEffect), consumed by codegen Burst jobs (Tier 1) and generic interpreter (Tier 2).

7 stored attributes (Str, Agi, Bod, Will, Wit, Spi, Cha) + 2 derived (Authority, Reputation). 23 skills (7 actions × 3 approaches: Direct/Indirect/Structured + 2 meta: Stealth, Awareness). 7 drives. 4-axis relationships (Social trait: Debt, Rep, Aff, Fam).

The knowledge system uses virtual items — nodes with ImmaterialTrait + Mirrors + StatCopy + ObscurityTrait. Three tiers: (1) local observation (free), (2) skill-gated rule knowledge (free), (3) virtual items for remote/hidden/unique knowledge (runtime cost). Authority derives from four sources (Force, Delegation, Consensus, Tradition) via AuthStrengthTrait. Contracts are virtual items (ImmaterialTrait + ContractTermsTrait). Social enforcement through expectation v-items + social judgment rules.

Nine elementary ops (Accumulate, Decay, Set, Transfer, Spread, Create, Destroy, AddTrait, RemoveTrait). Five built-in functions (distance, contains, count, sigmoid, depth). Batch modes auto-derived. Delta buffer for all mutations. Deterministic execution via seeded PRNG and read-write phase separation.

## File Structure

```
architecture.md               Implementation architecture (canonical)
SPECIFICATION.md              Master overview with links to topic specs
MEMORY.md                     This file — project context

spec_core_stats.md            Attributes (7+2), Skills (23+2), Drives (7), Relationships (4-axis)
spec_unified_tree.md          Node structure, two trees, traits, split/merge, shells/cores
spec_expression_language.md   Conditions, effects, elementary ops, rule IR, built-in functions
spec_objects_and_space.md     Objects as nodes, traits, spatial hierarchy, containers
spec_knowledge.md             3 tiers, virtual items as nodes, obscurity, propagation
spec_file_format.md           .acf tokens, grammar, rule syntax, validation
spec_rules.md                 World rules L0–L4, elementary ops, batch modes, adaptive timestep
spec_roles.md                 Reactive agency, inheritance, shift patterns, examples
spec_plans.md                 Proactive agency, compounds, counter-plans, examples
spec_contracts_authority.md   Contracts as v-items, authority sources, orders, compliance
spec_pathfinding.md           Flow-based, region graph, throughput, hauling
spec_infrastructure.md        Scale, history, bundles, profiles, tech stack, pipeline

spec_discarded_alternatives.md  Rejected designs with reasons (13 entries)
reference_rpg_stats.md          RPG system comparison (AD&D, Rolemaster, DSA, GURPS, CoC)
```

## Key Learnings & Principles

- Authority stems from credible threats of sanctions and emerges from actual relationship dynamics (Force, Delegation, Consensus, Tradition) rather than being a stored stat. Implemented as AuthStrengthTrait on authority claim v-items.
- Knowledge uses virtual items — nodes with ImmaterialTrait that mirror real entities via StatCopy. Skills gate rule knowledge on demand. Most entities carry zero v-items.
- Contracts are virtual items (ImmaterialTrait + ContractTermsTrait), not a separate system. They get forgery, staleness, theft, and propagation for free.
- The distinction between ownership (OwnedBy relationship trait) and possession (ContainerNode) must be maintained throughout all systems.
- Roles function as perpetual reactive rulesets (ActiveRole trait) while plans represent finite goal-directed sequences (PlanMetaTrait + AgencyTrait), with orders serving as plan steps assigned across entity boundaries.
- Counter-planning triggers from observable world state conditions only. Never references drives, plans, knowledge, mood, skills, or contracts of the target.
- All game mechanics reference concrete, measurable trait fields rather than abstract concepts.
- Economic value and social consequences like reputation emerge naturally from system interactions rather than being directly encoded.
- Culture is not a system — it's shared expectation v-items + local relationship patterns propagating through familiarity.
- Objects are nodes with traits, not a separate type. A sword is Node + WeaponTrait + ConditionTrait. A grain pile is Node(Weight=500) + EdibleTrait.
- The free-form expression language was replaced by structured IR with elementary ops — simpler to parse, compile, and execute in Burst.

## Approach & Patterns

Manu works through systematic iterative refinement, consistently questioning redundancies and seeking natural patterns. Design sessions involve progressing from initial sketches through multiple architectural approaches before settling on unified solutions. Technical discussions focus on eliminating verbose wrapper functions in favor of clean syntax.

The development approach emphasizes creating comprehensive reference documentation through artifact creation, with multiple iterations to remove unnecessary complexity and maintain focus on conceptual design rather than implementation details. Manu prefers technical precision over prose descriptions, requesting structured identifiers throughout and removal of natural language from data fields.

Problem-solving follows a pattern of establishing foundational frameworks first, then systematically expanding to cover edge cases and integration points. Design decisions consistently favor solutions that work efficiently with Unity's Burst compiler while maintaining clean separation between simulation logic and rendering systems.

## Tools & Resources

The technical stack centers on Unity with Burst compiler optimization, using NativeList, NativeHashMap, and NativeParallelMultiHashMap for cache-friendly iteration and O(1) lookups. All simulation state in Fixed Q16.16 (int32). ToFloat() one-way gate for rendering only.

Trait-based ECS: SingleTraitTable<T> with NativeHashMap for 1:1 lookup, MultiTraitTable<T> with NativeParallelMultiHashMap for 1:n. All traits blittable structs with Owner (and Target for relationships).

Two-tier execution: Tier 1 codegen Burst/HLSL from .acf at build time (shipping), Tier 2 generic interpreter over IR (mods, prototyping).

The knowledge system uses three-tier aging (recent action logs, medium-term deltas, ancient snapshots) with run-length compression for efficiency.

The system references established games like RimWorld, Dwarf Fortress, and Baldur's Gate 3 for validation and comparison, while incorporating the .acf data format for executable rules and AI planning systems.
