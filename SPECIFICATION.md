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
- One node type, traits are everything, same rules at every scale.

---

## Architecture — `architecture.md`

Implementation specification: Node struct (~20 bytes), trait-based ECS, Fixed Q16.16 arithmetic, rule IR, two-tier execution (codegen + interpreter), delta buffer, determinism, parallelism, multiplayer/distribution.

---

## System Overview

### Core Stats — `spec_core_stats.md`

**7 Attributes** (Str, Agi, Bod, Will, Wit, Spi, Cha) + 2 derived (Authority, Reputation). Authority and Reputation always derived, never stored.

**23 Skills** (7 actions × 3 approaches + 2 meta). Approaches: Direct, Indirect, Structured. Skill bonus = skill + attr1 + attr2. Trade-offs: intensity↔stealth, stealth↔effectiveness, awareness↔focus.

**7 Drives**: Survival, Luxury, Dominance, Belonging, Knowledge, Lawful, Moral. Character flaws emerge from imbalances.

**4-Axis Relationships**: Debt, Reputation, Affection, Familiarity. Any→any, asymmetric. Social trait with Owner + Target.

### Unified Tree — `spec_unified_tree.md`

Every entity (individual, group, army, city, item, knowledge, contract) is a Node (~20 bytes). Two trees: ContainerNode (physical) and ParentNode (hierarchy). `Weight=1` = individual, `Weight>1` = group. Traits provide all differentiation — no monolithic stat block.

Split on variance threshold. Merge on similarity. Promote individuals for story relevance.

**Shells and Cores**: simulation nodes (cores) are always-on; rendering proxies (shells) exist only when visible. Two independent tick loops (sim at 1–10 Hz, render at 30–60 Hz).

### Conditions, Effects & Rule IR — `spec_expression_language.md`

All world state access through trait fields. All mutations through 9 elementary ops (Accumulate, Decay, Set, Transfer, Spread, Create, Destroy, AddTrait, RemoveTrait). 5 built-in functions (distance, contains, count, sigmoid, depth). Authored in `.acf`, compiled to structured IR. Two consumers: codegen Burst (Tier 1) and interpreter (Tier 2).

### Objects & Spatial Hierarchy — `spec_objects_and_space.md`

**Objects**: Nodes with composable traits (Weapon, Armor, Perishable, Flammable, etc.). No type hierarchy, no separate Object struct. Owner via OwnedBy relationship trait. Holder = ContainerNode. Bulk groups via Weight. Materialize on demand.

**Spatial**: Nodes with SpatialTrait at various scales (Tile through Empire). Derived properties (enclosed, roofed, temperature, light). Buildings = parent spatial nodes. Vehicles = moving subtrees. ConnectedTo edges define spatial graph.

### Knowledge — `spec_knowledge.md`

Three tiers: (1) local observation + role scope (free), (2) skill gates (free, on-demand), (3) virtual items (runtime cost). V-items are nodes with ImmaterialTrait + Mirrors + StatCopy + ObscurityTrait. Obscurity controls propagation (0=public, 1=secret). Cover stories = false v-items at lower obscurity.

### File Format — `spec_file_format.md`

`.acf` — TOML-compatible + `{ }` nesting + bare expressions. Three data types: rules (triggers), roles (reactive agency), plans (proactive agency). Two step keywords: `do` (act) and `wait` (condition). Rules use elementary op names in effects. Provenance tracking in `_provenance {}` blocks.

### Rules — `spec_rules.md`

World mechanics with no agency. Five layers: L0 Physics, L1 Biology, L2 Items, L3 Social, L4 Economic. Rules scope to trait tables. Effects use elementary ops. Batch modes auto-derived (Linear, Bernoulli, Poisson, NormalApprox). Adaptive timestep: minutes (combat) to years (worldgen). Rules as switches: Probability=0 cascades to prune dependent plans.

### Roles — `spec_roles.md`

Reactive agency. Repeating priority-sorted rules. Gained/lost through world state. Shift patterns for temporal activation. Inheritance for specialization. Cover ~95% of daily activity. ActiveRole relationship trait binds entity to role template. Roles become job instances when bound to workplace + holder.

### Plans — `spec_plans.md`

Proactive agency. Sequential do/wait steps with HTN method decomposition. Plans are v-items (nodes with ImmaterialTrait + PlanMetaTrait — shareable, stealable, stale-able). Plan syntax: `needs {}` (preconditions checked against agent worldmodel) + `outcomes {}` (probabilistic postconditions including costs). Probabilistic planning: step.prob via named resolution functions → plan.confidence → plan.utility. AgencyTrait on executing entities tracks active plan. Groups execute via fractional allocation (max 4 concurrent plans). Three-tier composition: templates → ~80–100 domain compounds → 7×3 leaf plans. Counter-plans trigger from observable state only; three tiers: static counters, suspect plans, shallow adversary simulation (depth 2, chains ≤ 4).

### Contracts & Authority — `spec_contracts_authority.md`

Contracts are virtual items — nodes with ImmaterialTrait + ContractTermsTrait. Expectations are separate v-items + social judgment rules for enforcement. Authority always derived from four sources: Force, Delegation, Consensus, Tradition (via AuthStrengthTrait). Governance types emerge from source weightings. Orders decompose down the command chain with compliance checks.

### Pathfinding — `spec_pathfinding.md`

Flow-based. Pathfinding granularity matches tree depth: tile A* for individuals, region-level flow for groups, zone-level for armies. Throughput-limited connections create natural bottlenecks, traffic jams, chokepoints. Hauling = flow between inventory nodes. Supply chains = flow networks. Shell movement derived from flow state (no per-shell pathfinding).

### Infrastructure — `spec_infrastructure.md`

**Scale**: ~150 simulation entities regardless of population via grouping. <2ms per sim tick target.

**History**: 3-tier aging (30-day detail → 1-year snapshots → milestone-only).

**Bundles**: Compatible rule/role/plan/expectation sets discovered by sim stability testing.

**Profiles**: Game profiles swap rule values, not behavior.

**Tech**: Unity + Burst, Q16.16 Fixed, delta buffer, read-write phase separation, codegen + interpreter.

**Pipeline**: LLM-assisted extraction from narrative/military/game/historical sources. GitHub Actions automation. Validation at 4 enforcement points.

---

## Reference & History

- `architecture.md` — Implementation architecture (nodes, traits, rule IR, execution, scale, multiplayer)
- `CLAUDE.md` — AI assistant context: authoring rules, validation checklist, action table, architecture references
- `reference_rpg_stats.md` — RPG system comparison (AD&D, Rolemaster, DSA, GURPS, CoC)
- `spec_discarded_alternatives.md` — All rejected designs with reasons for rejection (13 entries)
- `MEMORY.md` — Project context and development approach

## Companion Repository

[adventurecraft_HTN_GOAP](https://github.com/ManuelKugelmann/adventurecraft_HTN_GOAP) — the behavior dataset implementation. Contains `.acf` files (rules, roles, plans), Python schema/validator, utility function catalog, world rules catalog, extraction pipeline, and CI/CD hooks. This spec is the authoritative reference; HTN_GOAP is the implementation.
