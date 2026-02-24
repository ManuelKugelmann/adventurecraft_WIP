# AdventureCraft — High-Level System Summary

## Core Philosophy

Emergent gameplay from simple composable building blocks. Data-driven templates over code. Derive everything possible; store only fundamental state. One unified structure per domain — no special cases.

---

## Attributes (9 — 3×3 Matrix)

|              | Power        | Coordination  | Endurance        |
|--------------|-------------|---------------|------------------|
| **Physical** | Strength    | Agility       | Body (HP)        |
| **Mental**   | Willpower   | Intelligence  | Spirit (mana)    |
| **Social**   | Authority*  | Charisma      | Reputation*      |

*Derived from relationship patterns, not stored.

## Actions & Skills (7 × 3 + 2 meta)

Universal signature: `ACTION(target, method, [objects], intensity, secrecy)`

| Action    | Direct         | Leveraged   | Structured     |
|-----------|---------------|-------------|----------------|
| MOVE      | Athletics     | Riding      | Travel         |
| MODIFY    | Operate       | Equipment   | Crafting       |
| ATTACK    | Melee         | Ranged      | Traps          |
| DEFENSE   | Active Defense| Armor       | Tactics        |
| TRANSFER  | Gathering     | Trade       | Administration |
| INFLUENCE | Persuasion    | Authority   | Intrigue       |
| SENSE     | Search        | Observation | Research       |

Meta-skills: **Stealth** (reduces detection), **Awareness** (improves detection). Trade-offs: intensity↔stealth, stealth↔effectiveness, awareness↔focus.

## Drives (7)

survival, luxury, dominance, belonging, knowledge, lawful (0=chaotic, 100=lawful), moral (0=evil, 100=good). Flaws emerge from imbalances.

## Relationships (4-axis, any→any, asymmetric)

debt (−100..+100), reputation (−100..+100), affection (−100..+100), familiarity (0–100). High familiarity → personal relationship dominates inherited group relationship.

---

## Unified Tree

One node type at every depth. `weight=1` = individual, `weight>1` = group. Same stat block, different precision.

- **Split** when variance > threshold → children inherit parent stats + delta
- **Merge** when children similar < threshold → coarsen back
- **Promote** individual to leaf for story relevance
- **Shells** (render proxies) derive visuals from cores; exist only when visible

Two independent loops: simulation tick (1–10 Hz, O(core nodes)) and render tick (30–60 Hz, O(visible shells)).

---

## Knowledge (3 Tiers)

| Tier | What | Cost |
|------|------|------|
| Local observation + role scope | Query world state directly | Free, always current |
| Skill gates | On-demand checks reveal detail / gate rules | Free, part of stat block |
| Virtual items | Remote, hidden, unique knowledge | Runtime cost, possibly stale/wrong |

Virtual items = partial copies of real entities with `freshness`, `source`, `obscurity`. Obscurity controls passive propagation (0=common knowledge, 1=unknown unknown). Cover stories = false virtual items at lower obscurity.

Most entities carry zero virtual items — world covered by observation + skills.

---

## Objects

Single struct + composable traits (Perishable, Weapon, Flammable, Tool, Container, ...). No polymorphic hierarchy.

- **Owner** (legal claim) vs **Holder** (physical possession)
- **Bulk groups** for fungible items; materialize individual objects on demand
- **Implicit→explicit**: region resource stats → GATHER → explicit object
- Rules reference traits directly: `item.has(Perishable)`, `item.trait(Weapon).damage`

---

## Spatial Hierarchy

Single node type: World → Province → Zone → Region → TileGroup → Tile.

- Tags drive action eligibility (`[cooking]` needed for CRAFT(food))
- Buildings = parent nodes of room nodes
- Vehicles = moving subtrees (shared velocity)
- Containers (portable or spatial) = same structure
- Owner/Holder applies to spaces (renting, occupation, conquest)
- Properties (enclosed, roofed, temperature, light) derived from child tiles

---

## Templates (Universal TOML Envelope)

All categories share: `[meta]` (id, category, extends, tags), `[params]`, `[requirements]`.

### Expression Language — Two Primitives

```
STAT(entity, path)                              → value
EDGE(from, to, type, [path], [depth], [agg])    → value
```

EDGE aggregations: exists, min_depth, sum(field), min(field), max(field), product(field). Everything else (HAS, AT, ALIVE, CONTROLS) is sugar over these two.

Effects: SET, ADD, REMOVE, CREATE, DESTROY, ADD_EDGE, REMOVE_EDGE.

### Template Categories

| Category   | Nature                        | Completes? |
|------------|-------------------------------|------------|
| Role       | Perpetual priority rules      | Never      |
| Plan       | Goal-directed step sequence   | Yes        |
| Compound   | Reusable sub-step sequence    | Yes        |
| World Rule | Tick-evaluated state transition| Recurring |
| Condition  | Named expression composition  | N/A        |
| Contract   | Mutual obligations + penalties| On terms   |

---

## Roles, Plans, Contracts, Orders, Authority

All are **virtual items** — get obscurity, staleness, forgery, verification for free.

**Roles**: Prioritized condition→action rules per tick. Multiple per entity via shift patterns. Template inheritance (MasterSmith extends Smith).

**Plans**: Finite steps = `Action(id, args, probability)` | `Expect(rule_id, probability)`. Confidence = product of step probabilities. Lazy decomposition matches tree depth (faction→phases, cohort→tasks, leaf→atomics).

**Contracts**: Employment→role instances, quests→plan steps, trade→recurring transfers. Breach = failing obligation counts/durations.

**Orders**: One-sided contracts. Plan step assigned across entity boundaries. Decomposition cascades down hierarchy.

**Authority**: Sources stack — Force (military in region), Delegation (delegator's strength × fraction), Consensus (population affection aggregates), Tradition (time × familiarity). Regionally constrained. Erodes naturally as underlying state changes.

### Priority Resolution

```
1. Active plan step (if assigned and preconditions met)
2. Highest-priority rule across active role instances
3. Default idle behavior
```

### Counter-Planning

Triggers from **observable world state conditions**, not enemy plan knowledge:

```
1. Observable conditions match trigger sets → candidate counter-plans
2. Known enemy plan template (if available) → additional candidates  
3. Select by expected utility (confidence × goal_value)
```

Knowing the enemy's actual plan template is a bonus, not a requirement.

---

## World Rules (5 Layers)

| Layer | Domain | Agency | Depends On |
|-------|--------|--------|------------|
| L0 | Physics (temp, water, fire, terrain) | None | Nothing |
| L1 | Biology (plants, metabolism, disease) | None | L0 |
| L2 | Items (decay, structures, consumables) | None | L0 |
| L3 | Basic behavior (survival, possession, mood) | Reactive | L0–L2 + drives |
| L4 | Complex behavior (economic, military, political) | Goal-directed | All + knowledge + plans |

Time-scale independent: every rule is rate×dt. Adaptive timestep (minutes in combat → days peacetime → years worldgen). Batch modes: accumulate, Bernoulli, Poisson, time-to-threshold skip.

---

## History (3-Tier Aging)

| Window | Detail | Used For |
|--------|--------|----------|
| Recent (≤30 days) | Full action logs, run-length compressed | Object vicinity queries, knowledge capture |
| Medium (30–365 days) | Snapshots + deltas only | Broad temporal knowledge |
| Ancient (1+ years) | Major milestone snapshots | Long-term world state |

Automatic progressive aging. Supports public→secret knowledge capture transitions.

---

## Pathfinding (Flow-Based)

Region graph (~200 nodes) with throughput-limited connections. Groups **flow**, not pathfind.

- Route: A* on region graph (trivial)
- Rate: bottleneck connection throughput limits group transit speed
- Contention: competing groups share capacity proportionally
- Hauling: flow between inventory nodes, haulers = throughput multiplier
- Cost: O(region_graph + active_flows), independent of population

Flow fields cached for common destinations (dining, workplaces, markets). Shells derive movement visually from group flow state — no per-shell pathfinding.

---

## Scale Strategy

- Groups handle ~90% of population (80 people → 1 entity)
- Max 4 concurrent fractional actions per group
- 4 internal subgroups for heterogeneity; split/merge by divergence
- Probabilistic location distributions for unsplit groups
- Virtual items only for remote/hidden/unique knowledge (~0–20 per node)
- Bulk object groups; materialize individuals on demand
- Simulation cost O(core nodes), render cost O(visible shells) — both independent of total population
