# AdventureCraft

<!-- TEASER IMAGE — generate with: replicate run black-forest-labs/flux-2-klein-9b-base \
     --input prompt="<see prompt below>" --input width=1280 --input height=640
     Prompt: "A vast medieval landscape seen from above, overlaid with glowing network graphs
     and data flows connecting castles, villages and trade routes, blending simulation
     aesthetics with historical realism, cinematic lighting, dark atmospheric"
     Replace this comment block with: ![AdventureCraft teaser](assets/teaser.png) -->

> A deterministic social simulation engine. Models historical dynamics from the ground up. Predicts emergent outcomes in real-world-analogous systems.

---

## What This Is

AdventureCraft is a simulation engine built around one thesis: **complex social, economic, and political outcomes emerge from simple, measurable local rules — no hardcoded narratives required.**

A medieval siege, a market collapse, a coup, a plague response, a supply chain failure. These are not scripted events. They are outputs. The engine runs the physics and reads the result.

The same model that simulates a 14th-century grain shortage can stress-test a modern logistics network. The same authority dynamics that produce feudalism can produce a fragile democracy. **The rules are the same. Only the parameters change.**

---

## Dual Use: History and Prediction

### As a History Simulator

History is not a sequence of special events. It is the aggregate of millions of small decisions made by people with partial information, competing drives, and constrained options.

AdventureCraft models this directly:

- **No scripted events.** Battles start because soldiers have orders, enemies are within range, and aggression drives exceed fear thresholds.
- **No handwaved economies.** Grain rots. Mills have throughput limits. Famine emerges from broken supply chains, not from a `trigger_famine()` call.
- **No abstract politics.** Authority is a derived quantity — a function of military presence, delegation chains, popular compliance, and accumulated tradition. Regimes are outputs, not inputs.
- **Historically-grounded rules.** Role and plan content is extracted from primary sources: occupation manuals, guild records, military doctrine, legal codes, folktale taxonomies. The behavioral library is bootstrapped from the actual historical record.

Run a simulation forward from a validated initial state and read what emerges. Compare against the historical record. Adjust parameters. Run again.

### As a Predictive Simulator

The engine is deliberately genre-agnostic. The same systems — trait-based entities, emergent authority, knowledge propagation, flow-based logistics, reactive and proactive agency — apply at any setting.

The structural questions are universal:

- What happens to supply chains when a critical node fails?
- How does information asymmetry affect coordination?
- When does delegated authority collapse under pressure?
- What conditions produce compliance versus resistance?
- How do sanctions propagate through relationship networks?

Swap the skin. The underlying dynamics are the same. A modern logistics firm, a political coalition, a military operation, a financial contagion — all are instances of the same simulation primitives.

---

## Architecture Summary

The engine is built on a single principle: **store only fundamental state; derive everything else.**

### One Node Type

Every entity — a person, a garrison, a faction, a grain pile, a rumor, a legal claim — is a `Node` with `Traits`. No inheritance hierarchy. No special cases.

```
Node { Id, Template, Weight, ContainerNode, ParentNode, Flags }
```

`Weight > 1` means a group. The same simulation logic runs on an individual and a 10,000-strong faction. Groups split when internal variance exceeds a threshold; they merge when they converge. Individuals are promoted out of groups only when story relevance demands it.

### Traits Are Everything

Traits are composable data components. A sword has `Weapon + Condition + Valuable + Heavy`. A legal authority claim has `Immaterial + AuthStrength + AuthScope`. A rumor has `Immaterial + StatCopy + Obscurity`. Trait composition replaces class hierarchies and special-case logic.

### Rules, Roles, Plans

Three distinct layers of behavior:

| Layer | What | When |
|-------|------|------|
| **Rules** | World mechanics — no agency | Always on, layered L0–L4 (physics → biology → items → social → economic) |
| **Roles** | Reactive agency — priority-sorted responses | ~95% of daily activity |
| **Plans** | Proactive agency — goal-directed sequences | Deliberate, multi-step objectives with HTN decomposition |

### Emergence, Not Scripting

Authority is computed from four measurable sources: force presence, delegation chains, popular approval, tradition weight. Compliance is computed per interaction. Tyranny, feudalism, democracy — these emerge from the balance of those sources. They are not selected from a dropdown.

Knowledge spreads through co-location, teaching, espionage, and research. Misinformation exists as false virtual items competing with true ones. Cover stories are structurally identical to real intelligence. The knowledge system has no special cases.

### Scale Without Special Cases

| Depth | Entity Type | Pathfinding | Time Step |
|-------|-------------|-------------|-----------|
| Tile | Individual | Hierarchical A* | 10 sec |
| Region | Cohort (5–200) | Region-level flow | Minutes |
| Province | Army (1,000+) | Zone-level flow | Days |
| Continent | Faction (10,000+) | Province graph | Weeks |

The same node structure operates at every scale. Groups flow through region graphs; individuals pathfind through tile graphs. No separate simulation modes.

### Determinism

Fixed Q16.16 arithmetic throughout — no floating point in the simulation. PRNG seeded per `(world_seed, tick, rule_id, subject_id)`. All mutations pass through a delta buffer resolved after each read phase. **Given the same seed and inputs, the simulation produces identical output every time.** This is a prerequisite for reproducible historical modeling and reliable prediction.

---

## Data Format: `.acf`

All content — rules, roles, plans — is defined in `.acf` files: TOML-compatible flat sections with `{}` nesting and bare expressions.

```acf
rule hunger_drain:
    layer: L1
    scope: VitalsTrait
    condition: Vitals.Satiation > 0
    effect: Accumulate Vitals.Satiation rate = -Vitals.MetabolicRate * dt

role farmer : laborer [SEASONAL] {
    harvest: when Spatial.Season == Harvest and Crop.Mature, do Transfer.Direct {
        source = self, target = storage, trait = Crop
    }, priority = 10
}
```

Content is validated at four enforcement points: LLM self-check during extraction, post-generation scripts, git pre-commit hooks, and CI pipeline. The `.acf` format is the interface between the historical/real-world source material and the simulation engine.

---

## Content Extraction Pipeline

The behavioral library is not handcrafted. It is extracted from source material:

- **Narrative:** Propp, Polti, ATU folktale indices, Campbell, Booker
- **Military:** Clausewitz, Sun Tzu, modern doctrine manuals
- **Historical:** Guild records, occupation manuals, legal codes, household management texts
- **Behavioral:** Maslow, BDI literature, game theory
- **Reference systems:** D&D SRD, GURPS, Rolemaster, Dwarf Fortress, CK3, RimWorld

A GitHub Actions pipeline runs LLM extraction daily, validates output, and opens PRs. Coverage gaps are reported weekly as issues. The goal: a universal behavioral library — ~50–60 roles, ~150 everyday plans, ~80–100 composable plan compounds — derived from the actual record of how humans have organized and behaved.

---

## Current Status

**Phase: Design complete. Implementation pending.**

All core systems are fully specified in `/spec_*.md` files. The architecture document (`architecture.md`) is the canonical technical reference. No Unity implementation exists yet.

Specification modules:
- `spec_core_stats.md` — Attributes, Skills, Drives, Relationships
- `spec_unified_tree.md` — Node structure, traits, split/merge/promote, shells/cores
- `spec_rules.md` — World mechanics, batch modes, adaptive timestep
- `spec_roles.md` — Reactive agency, inheritance, shift patterns
- `spec_plans.md` — Proactive agency, HTN decomposition, counter-plans
- `spec_contracts_authority.md` — Contracts, authority sources, compliance, regimes
- `spec_pathfinding.md` — Flow-based movement, supply chains, throughput
- `spec_knowledge.md` — Knowledge tiers, virtual items, obscurity, misinformation
- `spec_objects_and_space.md` — Spatial hierarchy, objects as nodes, vehicles
- `spec_infrastructure.md` — Scale strategy, history aging, bundles, profiles, tech stack
- `spec_expression_language.md` — Condition/effect language, elementary ops, built-in functions
- `spec_file_format.md` — `.acf` grammar, validation rules

---

## Design Constraints (Non-Negotiable)

1. **Store only fundamental state. Derive everything else.** Authority is never stored. Reputation is never stored. They are computed.
2. **All conditions reference concrete, measurable values.** No abstract morale. No handwaved loyalty. If it can't be expressed as a trait field comparison or a built-in function, it doesn't exist.
3. **One node type. Traits are everything.** No special cases for persons vs. groups vs. factions vs. items vs. concepts.
4. **Composition over inheritance.** A torch is `LightSource + Flammable + Heavy + Weapon`. Not a subclass of anything.
5. **Emergent complexity from simple rules.** No scripted events. No trigger conditions that bypass the simulation. Read the output.
6. **Determinism.** Fixed-point arithmetic. Seeded PRNG. Delta buffers. Read-write separation. The simulation is a function of its inputs.

---

## Tech Stack

- **Engine:** Unity + Burst compiler
- **Arithmetic:** Q16.16 Fixed-point (int32)
- **Collections:** NativeList, NativeHashMap, NativeParallelMultiHashMap
- **Time unit:** 1 tick = 10 seconds
- **Execution:** Two-tier — Burst/HLSL codegen for shipping, generic IR interpreter for mods/hot-reload
- **Content format:** `.acf` (TOML-compatible)
- **Extraction:** GitHub Actions + LLM pipeline

---

*AdventureCraft is a work in progress. The specification is the product until the implementation exists.*
