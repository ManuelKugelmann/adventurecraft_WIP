# AdventureCraft — System Specification

A simulation-based strategy game engine generating emergent gameplay through composable, data-driven systems. Works across medieval fantasy, sci-fi, and modern settings.

---

## Design Principles

- Store only fundamental state. Derive everything else on demand.
- Emergent complexity from simple composable building blocks.
- All conditions reference concrete, measurable values. No abstract concepts.
- Wealth = owned valuables. Reputation = relationship patterns. Authority = derived from sources.
- Static properties via template hierarchy, not stored per-entity.
- Composition over inheritance. Data over code.

---

## System Overview

### Core Stats — `spec_core_stats.md`

**9 Attributes** (3×3 matrix: Physical/Mental/Social × Power/Coordination/Endurance). Authority and Reputation always derived, never stored.

**23 Skills** (7 actions × 3 approaches + 2 meta). Universal action signature: `ACTION(target, method, [objects], intensity, secrecy)`. Trade-offs: intensity↔stealth, stealth↔effectiveness, awareness↔focus.

**7 Drives**: survival, luxury, dominance, belonging, knowledge, lawful, moral. Character flaws emerge from imbalances.

**4-Axis Relationships**: debt, reputation, affection, familiarity. Any→any, asymmetric.

### Unified Tree — `spec_unified_tree.md`

Every entity (individual, group, army, city) is a node in a single tree. Same struct at every depth. `weight=1` = individual, `weight>1` = group. Stats at variable granularity: location ranges from exact tile to zone distribution, equipment from specific item to category tier.

Split on variance threshold. Merge on similarity. Promote individuals for story relevance.

**Shells and Cores**: simulation nodes (cores) are always-on; rendering proxies (shells) exist only when visible. Two independent tick loops (sim at 1–10 Hz, render at 30–60 Hz).

### Expression Language — `spec_expression_language.md`

Two primitives: `STAT(entity, path)` and `EDGE(from, to, type, ...)`. Everything else is sugar. Supports transitive edge queries with aggregation (exists, sum, min, max, product). Full grammar with math, comparison, logical operators. Effects: SET, ADD, REMOVE, CREATE, DESTROY, ADD_EDGE, REMOVE_EDGE.

### Objects & Spatial Hierarchy — `spec_objects_and_space.md`

**Objects**: Single struct with composable traits (Weapon, Armor, Perishable, Flammable, etc.). No type hierarchy. Rules reference traits: `item.has(Trait)`, `item.trait(T).field`. Owner (legal) vs Holder (physical). Bulk groups for fungible items; materialize on demand.

**Spatial**: Unified tree of spatial nodes (World → Province → Zone → Region → Tile). Derived properties (enclosed, roofed, temperature, light). Buildings = parent nodes. Vehicles = moving subtrees. Tags drive action eligibility.

### Knowledge — `spec_knowledge.md`

Three tiers: (1) local observation + role scope (free), (2) skill gates (free, on-demand), (3) virtual items (runtime cost). V-items are partial, possibly stale copies of real entities. Obscurity controls propagation (0=public, 1=secret). Cover stories = false v-items at lower obscurity. Meta-knowledge = recursion depth.

### File Format — `spec_file_format.md`

`.acf` — TOML-compatible + `{ }` nesting + bare expressions. Three data types: rules (triggers), roles (reactive agency), plans (proactive agency). Two step keywords: `do` (act) and `wait` (condition). Provenance tracking in `_provenance {}` blocks.

### Rules — `spec_rules.md`

World mechanics with no agency. Five layers: L0 Physics, L1 Biology, L2 Items, L3 Social, L4 Economic. All rates/probabilities scale by dt. Batch modes auto-selected (Accumulate, Bernoulli, Poisson, Normal, TimeToThreshold). Adaptive timestep: minutes (combat) to years (worldgen). Rules as switches: prob=0 cascades to prune dependent plans.

### Roles — `spec_roles.md`

Reactive agency. Repeating priority-sorted rules. Gained/lost through world state. Shift patterns for temporal activation. Inheritance for specialization. Cover ~95% of daily activity. Roles become job instances when bound to workplace + holder.

### Plans — `spec_plans.md`

Proactive agency. Sequential do/wait steps with HTN method decomposition. Plans are v-items (shareable, stealable, stale-able). Probabilistic planning: step.prob → plan.confidence → plan.utility. Decomposition depth matches tree depth. Groups execute via fractional allocation (max 4 concurrent plans). ~80–100 composable compounds as building blocks. Counter-plans trigger from observable state only, max depth 4.

### Contracts & Authority — `spec_contracts_authority.md`

No explicit contract system. Expectations are v-items + social judgment rules. Authority always derived from four sources: Force, Delegation, Consensus, Tradition. Governance types emerge from source weightings. Orders decompose down the command chain with compliance checks.

### Pathfinding — `spec_pathfinding.md`

Flow-based. Pathfinding granularity matches tree depth: tile A* for individuals, region-level flow for groups, zone-level for armies. Throughput-limited connections create natural bottlenecks, traffic jams, chokepoints. Hauling = flow between inventory nodes. Supply chains = flow networks. Shell movement derived from flow state (no per-shell pathfinding).

### Infrastructure — `spec_infrastructure.md`

**Scale**: ~150 simulation entities regardless of population via grouping. <2ms per sim tick target.

**History**: 3-tier aging (30-day detail → 1-year snapshots → milestone-only).

**Bundles**: Compatible rule/role/plan/expectation sets discovered by sim stability testing.

**Profiles**: Game profiles swap rule values, not behavior.

**Tech**: Unity + Burst, Q16.16 fixed-point, read-write phase separation, precompiled templates.

**Pipeline**: LLM-assisted extraction from narrative/military/game/historical sources. GitHub Actions automation. Validation at 4 enforcement points.

---

## Reference & History

- `reference_rpg_stats.md` — RPG system comparison (AD&D, Rolemaster, DSA, GURPS, CoC)
- `spec_discarded_alternatives.md` — All rejected designs with reasons for rejection
- `MEMORY.md` — Project context and development approach
