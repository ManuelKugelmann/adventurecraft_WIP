# AdventureCraft — Comprehensive System Specification
## Version 2026-02-24

---

## 1. Design Principles

- **Emergent > scripted**: Complex behavior from simple composable rules.
- **Derive > store**: Compute properties on demand; store only fundamental state.
- **Composition > inheritance**: Flat traits, not class hierarchies.
- **Data > code**: Templates, rules, plans are all data files (`.acf` format).
- **Unified > special-cased**: One node type, one expression language, one evaluator.
- **Concrete > abstract**: Every condition references measurable world state values.
- **Scale-independent**: Same rules at dt=minutes (combat) through dt=years (worldgen).

---

## 2. Core Stats

### 2.1 Attributes (9 — 3×3 Matrix)

|              | Power        | Coordination  | Endurance        |
|--------------|-------------|---------------|------------------|
| **Physical** | Strength    | Agility       | Body (HP)        |
| **Mental**   | Willpower   | Intelligence  | Spirit (mana)    |
| **Social**   | Authority*  | Charisma      | Reputation*      |

\* Authority & Reputation derived from relationship aggregates. Never stored.

### 2.2 Skills (7 Actions × 3 Approaches + 2 Meta = 23)

Universal action signature: `ACTION(target, method, [objects], intensity, secrecy)`

| Action    | Direct          | Careful      | Indirect        |
|-----------|----------------|-------------|-----------------|
| Move      | Athletics      | Riding      | Travel          |
| Modify    | Operate        | Equipment   | Crafting        |
| Attack    | Melee          | Ranged      | Traps           |
| Defense   | Active Defense | Armor       | Tactics         |
| Transfer  | Gathering      | Trade       | Administration  |
| Influence | Persuasion     | Deception   | Intrigue        |
| Sense     | Search         | Observation | Research        |

Each skill uses 2 attribute scores (e.g., Melee = STR+AGI, Research = INT+SPI).

**Meta-skills**: Stealth (reduces detection), Awareness (improves detection).

**Trade-offs**: intensity↔stealth, stealth↔effectiveness, awareness↔focus.

### 2.3 Drives (7)

```
survival:    0–100   safety, basic needs
luxury:      0–100   comfort, fine goods
dominance:   0–100   control, authority
belonging:   0–100   relationships, community
knowledge:   0–100   learning, discovery
lawful:      0–100   0=chaotic, 50=neutral, 100=lawful
moral:       0–100   0=evil, 50=neutral, 100=good
```

Character flaws emerge from drive imbalances. No explicit flaw system.

### 2.4 Relationships (4-axis, any→any, asymmetric)

```
debt:        -100..+100   obligation balance
reputation:  -100..+100   fear↔respect
affection:   -100..+100   hate↔love
familiarity: 0–100        interaction history
```

High familiarity → personal relationship dominates inherited group relationship.

---

## 3. Unified Tree Architecture

### 3.1 Node Structure

Every entity — individual, group, army, city, planet — is a node.

```
Node {
    weight: u32,                   // 1=individual, >1=group
    stats: StatBlock {
        location: LocationStat,
        health: f32,
        mood: f32,
        attributes: [f32; 9],
        skills: [f32; 23],
        drives: [f32; 7],
    },
    relationships: [Relationship],
    inventory: Inventory,
    knowledge: [VirtualItem],
    template: TemplateRef,
    children: [Node]?,
    home_group: NodeRef?,
}
```

### 3.2 Split / Merge / Promote

| Trigger                          | Action                                     |
|----------------------------------|--------------------------------------------|
| Stat variance > split_threshold  | Split into children, each inherits + delta |
| Child similarity < merge_threshold | Coarsen children back into parent        |
| Story relevance                  | Promote individual to leaf (weight=1)      |
| Relevance lost                   | Reintegrate if divergence low              |

### 3.3 Location Granularity

```
weight=5000: Zone(INDUSTRIAL)
weight=200:  Region(MINE_3)
weight=1:    Tile(347, 891)
```

### 3.4 Shells and Cores (Rendering)

- **Core**: simulation node, exists always, O(core_nodes)
- **Shell**: rendering proxy, exists only when visible, O(visible_shells)
- Shell derives position/animation/sprite from core stats
- Two independent tick loops: simulation (1–10 Hz) and render (30–60 Hz)
- Tree restructuring is invisible to renderer

---

## 4. Expression Language

### 4.1 Two Primitives

```
STAT(entity, path)                              → value
EDGE(from, to, type, [path], [depth], [agg])    → value
```

Everything else is sugar built from these two.

### 4.2 Full Grammar

```
Expr =
    | entity.path | edge(from, to, type, ...) | $param | Number | Bool
    | prob(Expr) | random(min, max) | sigmoid(Expr)
    | Expr { + - * / } Expr | min | max | abs
    | Expr { < > == != >= <= } Expr
    | Expr AND Expr | Expr OR Expr | NOT Expr
    | count(entities WHERE Expr) | any(...) | sum(...)
```

### 4.3 EDGE Aggregations

| Agg            | Returns                     |
|----------------|-----------------------------|
| exists         | bool: any path within depth |
| min_depth      | int: shortest hop count     |
| sum(field)     | float: total along path     |
| min(field)     | float: bottleneck           |
| max(field)     | float: peak                 |
| product(field) | float: decay chains         |

### 4.4 Effects

```
SET | ADD | REMOVE | CREATE | DESTROY | ADD_EDGE | REMOVE_EDGE
```

### 4.5 Edge Types

social, member_of, owns, holds, contains, located_in, connected_to, adjacent, parent_of, parent_template, hostile_to, allied_with, knows_about, guards, delegates_to, supplies

### 4.6 Sugar

| Sugar              | Resolves To                       |
|--------------------|-----------------------------------|
| HAS(e, item, n)   | e.inventory[item] >= n            |
| AT(e, loc)         | e.location == loc                 |
| ALIVE(e)           | e.health > 0                      |
| CONTROLS(f, r)     | edge(f, r, controls)              |
| CONNECTED(a, b)    | edge(a, b, connected_to, 99, exists) |

---

## 5. Object System

### 5.1 Single Struct + Composable Traits

```
Object {
    id, template, traits: [Trait], condition, quantity,
    owner: EntityRef?,    // legal claim
    holder: EntityRef?,   // physical possession
    location, secrecy, crafted: CraftedInfo?
}
```

### 5.2 Trait Examples

Perishable(rate), Flammable(ignition, fuel_value), Edible(nutrition, taste), Weapon(damage, speed, range), Armor(protection, encumbrance), Tool(domain, bonus), Container(capacity), Heavy(weight), Fragile, Valuable(base_price), Material(type), LightSource(brightness, fuel_rate), ...

### 5.3 Owner vs Holder

Owner = social/legal claim (backed by authority). Holder = physical fact.
Theft = changing holder without owner consent. Applies to spatial nodes too.

### 5.4 Bulk Groups & Materialization

Fungible items stored as `{template, quantity, avg_condition, avg_quality}`.
Materialize individual objects on demand (theft, inspection, story relevance).
Region resources are implicit stats → GATHER → explicit object.

---

## 6. Spatial Hierarchy

Single node type: World → Province → Zone → Region → TileGroup → Tile.

```
SpatialNode {
    children, owner, holder, tags: [ZoneTag],
    contents: [BulkGroup], special_items: [ObjectId]
}
```

- Tags drive action eligibility: `[cooking]` needed for CRAFT(food)
- Buildings = parent nodes of room nodes
- Vehicles = moving subtrees with shared velocity
- Containers (portable or spatial) = same structure
- Properties (enclosed, roofed, temperature, light) derived from child tiles

---

## 7. Knowledge System (3 Tiers)

| Tier | What                                 | Cost                          |
|------|--------------------------------------|-------------------------------|
| 1    | Local observation + role scope       | Free, always current          |
| 2    | Skill gates (on-demand checks)       | Free, part of stat block      |
| 3    | Virtual items (remote/hidden/unique) | Runtime cost, possibly stale  |

### 7.1 Virtual Items

```
VirtualItem {
    mirrors: EntityRef,    // what real thing this represents
    stats: StatBlock,      // partial, possibly wrong copy
    freshness: f32,        // age of information
    source: Source,        // observed, taught, inferred, stolen
    obscurity: f32,        // 0=common knowledge, 1=deeply hidden
}
```

- Knowledge about a person = virtual copy of them
- Meta-knowledge = virtual copy containing virtual copies in their knowledge slot
- Cover stories = false virtual items at lower obscurity
- Plans = virtual items mirroring desired future states
- Secret recipes = virtual items gating rule access
- Most entities carry zero virtual items

### 7.2 Obscurity Controls Propagation

```
≈0.0: freely propagates (public facts)
≈0.3: spreads within community
≈0.6: restricted (military, trade secrets)
≈0.9: deep secret (espionage only)
≈1.0: unknown unknown (discovery only)
```

### 7.3 Skills Include Rule Knowledge

Skill level gates which simulation rules an entity can use. No separate knowledge category for rule-gated information. Metallurgy:3 = knows how to temper steel.

---

## 8. File Format: `.acf`

TOML-compatible flat sections + `{ }` nesting + bare expressions. Keywords: rule, role, plan, method, do, wait, when, done, fail, require, counter, params, effect, bundle, template, profile. ALL_CAPS for jump targets only.

### 8.1 Three Data Types

```
rule    trigger → state change       (world mechanics, no agency)
role    repeating priority rules     (reactive agency)
plan    sequential do/wait steps     (proactive agency)
```

### 8.2 Two Step Keywords

```
do      actor acts        → Action.Approach { params }
wait    actor waits       → condition becomes true
```

---

## 9. Rules (World Mechanics)

### 9.1 Structure

```acf
rule <id> [tags] {
    when <Expr>
    rate = <Expr>            # deterministic continuous
    prob = <Expr>            # stochastic event
    effect: <Effect>
}
```

rate and prob are mutually exclusive. Both scale by dt.

### 9.2 Layers (Dependency Order)

| Layer | Domain    | Agency | Depends On | Typical dt    |
|-------|-----------|--------|------------|---------------|
| L0    | Physics   | None   | Nothing    | minutes       |
| L1    | Biology   | None   | L0         | hours–days    |
| L2    | Items     | None   | L0         | days          |
| L3    | Social    | None*  | L0–L2      | days          |
| L4    | Economic  | None*  | L0–L3      | days–weeks    |

\*L3/L4 rules are agentless (things that happen TO actors).

### 9.3 Batch Modes (Auto-Selected)

```
Accumulate(rate * dt)                  deterministic continuous
BernoulliOnce(1 - (1-p)^dt)           did event fire?
PoissonCount(p * dt)                   how many times?
NormalApprox(mean*dt, var*dt)          CLT for large groups
TimeToThreshold(remaining/rate)        skip ticks entirely
```

### 9.4 Adaptive Timestep

```
Combat / player-observed:   dt = minutes
Local settlement:           dt = 1 day
Off-screen peacetime:       dt = 30 days
World gen / history:        dt = 1 year
```

### 9.5 Rules as Switches

rule.prob=0 → never fires → plans depending on it get confidence=0 → pruned. Game profiles are rule value tables.

---

## 10. Roles (Reactive Agency)

Prioritized condition→action rules evaluated every tick. Never complete.

```acf
role <id> [tags] {
    <name>: when <Expr>, do <ActionRef> <Args>, priority = <Expr>
}
```

- Multiple roles per entity via non-overlapping shifts
- All rules from all roles → flat list → sort by priority → first matching fires
- Template inheritance: `role master_smith : smith`
- Role acquisition/loss via world state conditions, not assignment
- Max concurrent roles per entity: 5

### 10.1 Universal Role Categories

Survival (forager, hunter, herder), Economic (farmer, miner, merchant), Craft (smith, carpenter, weaver), Military (guard, soldier, scout), Governance (judge, administrator), Knowledge (scholar, healer), Religious (priest), Social (entertainer, innkeeper), Domestic (parent, caretaker), Criminal (thief, smuggler, spy)

---

## 11. Plans (Proactive Agency)

Finite sequences of `do` and `wait` steps with completion/failure conditions.

```acf
plan <id> [tags] {
    params { ... }
    require { ... }
    method <name> {
        when { ... }
        priority = <Expr>
        <name>: do <ActionRef> <Args>
            prob = <Expr>
            fail = <LABEL>
        <name>: wait <Expr>
    }
    done { <Expr> }
    fail { <Expr> }
    counter <threat_id> { ... }
}
```

### 11.1 Probability

```
step.prob:  sigmoid(actor.skills[skill] - difficulty)  // always an Expression
plan.confidence = product(step.prob for critical path)
plan.utility = confidence × goal_value - total_cost
```

### 11.2 Decomposition Depth Matches Tree Depth

```
Depth 0: faction plan (SIEGE)
Depth 1: phase (ASSEMBLE, BREACH, STORM)
Depth 2: task (recruit, march, build_siege_tower)
Depth 3: compound (gather_wood, craft_components, haul)
Depth 4: leaf (do Modify.Indirect { ... })
```

Lazy decomposition: only decompose the current step. Max depth: 6.

### 11.3 Group Execution

Groups split effort across concurrent plans via fractional allocation. Max 4 concurrent, min allocation 0.05, sum ≤ 1.0.

### 11.4 Counter-Plans

Trigger from **observable world state only** (position, weight, faction, garrison, walls, equipped, visible actions, buildings, terrain). Never reference drives, plans, knowledge, mood. Max counter chain depth: 4.

---

## 12. Composable Compounds (~80–100)

Reusable sub-plan building blocks. Any plot is a sequence of these:

```
Information:  recon, investigate, eavesdrop, intercept, surveillance
Deception:    false_identity, plant_evidence, frame_target, cover_tracks, misdirect
Social:       seduce, blackmail, bribe, turn_agent, recruit, gossip_campaign
Economic:     smuggle, embargo, price_manipulation, counterfeit, extort
Political:    coup, undermine_authority, forge_alliance, install_puppet, exile
Protection:   safehouse, dead_drop, escape_route, alibi
Violence:     ambush, assassination, sabotage, arson, poison, kidnap
```

---

## 13. Contracts, Orders, Authority

All are **virtual items** — get obscurity, staleness, forgery, verification mechanics.

### 13.1 Contracts

Formalized mutual obligations. Employment→role instances, quests→plan steps, trade→recurring transfers. Breach = failing obligation counts/durations.

### 13.2 Orders

One-sided contracts. Plan step assigned across entity boundaries. Decomposition cascades down hierarchy.

### 13.3 Authority

Authority = credible threat of sanctions. Sources stack:

| Source     | Derived From                                    |
|------------|------------------------------------------------|
| Force      | Military strength in region                     |
| Delegation | Delegator's authority × fraction               |
| Consensus  | Population affection/reputation aggregates      |
| Tradition  | Time held × familiarity                         |

Regionally constrained. Erodes naturally as underlying state changes.

### 13.4 Priority Resolution

```
1. Active plan step (if assigned and preconditions met)
2. Highest-priority rule across active role instances
3. Default idle behavior
```

### 13.5 No Contracts (Alternative Model)

Expectations are v-items in knowledge. Two generic rules handle all social enforcement:

```
social_judgment:  observer has expectation AND witnessed violation → reputation -= weight
social_approval:  observer has expectation AND witnessed compliance → reputation += weight × 0.3
```

Custom, Law, Tradition, Taboo, Tyranny all emerge from these two rules + expectation v-items.

---

## 14. Pathfinding (Flow-Based)

### 14.1 Region Graph

~200 nodes, 3–6 edges each. Each connection has width, throughput, terrain_cost, current_flow.

### 14.2 Movement by Tree Depth

| Depth      | Method                | Graph size | Cost          |
|------------|----------------------|------------|---------------|
| Leaf (w=1) | Hierarchical A*      | ~500 tiles | O(500)        |
| Compound   | A* on region graph   | ~200       | O(200) + flow |
| Large      | A* on zone graph     | ~15        | O(15)         |
| Strategic  | Direct edge          | ~5         | O(1)          |

Total cost independent of population.

### 14.3 Flow Fields

Cached Dijkstra on region graph for common destinations (dining, workplaces, markets, barracks). Invalidate on topology change.

### 14.4 Congestion

Multiple groups competing for throughput → proportional capacity allocation. Creates natural bottlenecks, traffic jams, siege chokepoints.

### 14.5 Hauling as Flow

Item transport = flow between inventory nodes. Haulers = throughput multiplier.

---

## 15. History (3-Tier Aging)

| Window       | Detail                              | Used For                    |
|-------------|-------------------------------------|-----------------------------|
| Recent ≤30d | Full action logs, RLE compressed    | Object vicinity, knowledge  |
| Medium 30–365d | Snapshots + deltas only          | Broad temporal knowledge    |
| Ancient 1+yr | Major milestone snapshots           | Long-term world state       |

---

## 16. Bundles & Profiles

### 16.1 Bundles

Compatible sets of rules + roles + plans + v-items + rule values. Local to spatial nodes. **Discovered by sim stability testing, not authored.**

### 16.2 Profiles

Rule value tables. Same plans and roles, different game feel.

```acf
profile political_sim { fire_spread = 0, rumor_spread = 0.5 }
profile survival      { fire_spread = 0.2, crop_growth = 0.05 }
```

---

## 17. Scale Strategy

- Groups handle ~90% of population (80 people → 1 entity)
- Max 4 concurrent fractional actions per group
- 4 internal subgroups for heterogeneity; split/merge by divergence
- Probabilistic location distributions for unsplit groups
- Virtual items only for remote/hidden/unique (~0–20 per node)
- Bulk object groups; materialize individuals on demand
- Simulation cost O(core_nodes), render cost O(visible_shells)

---

## 18. Validation Rules

1. Every Expression resolves to STAT/EDGE/CONST/PARAM terminals
2. Every `do` uses valid Action × Approach from 7×3 table
3. Every plan decomposes to leaf actions within depth 6
4. Every step has a name
5. Every prob Expression is bounded 0..1
6. No references to authority or reputation as stored stats
7. Counter observables reference only externally visible state
8. Counter chains terminate within depth 4
9. No circular references in plan decomposition
10. All $param references have matching param definitions
11. sum(group allocations) ≤ 1.0, max 4 concurrent, min 0.05
12. Rule layers respect dependency order (L0→L1→L2→L3→L4)
13. Every `wait` implies a rule or condition that can produce the awaited state

Enforced at: LLM self-check → post-generate script → git pre-commit → CI.

---

## 19. Technical Stack

- **Engine**: Unity + Burst compiler
- **Arithmetic**: Q16.16 fixed-point, deterministic execution
- **Data structures**: NativeList, NativeParallelMultiHashMap (cache-friendly, O(1) lookups)
- **Execution**: Read-write phase separation
- **Templates**: Precompiled rules for performance + bytecode-like templates for data-driven balancing
- **Distribution**: Spatial partitions on different machines, delta merging for cross-partition interactions

---

## 20. Extraction Pipeline

### 20.1 Sources

Narrative (TVTropes, Propp, Polti, ATU), Military (doctrine manuals, Clausewitz, Sun Tzu), Games (DF, RimWorld, CK3, D&D SRD, GURPS), Behavioral (Maslow, BDI, game theory), Historical (guild records, legal codes, ethnographic databases), Everyday (household management, etiquette manuals, ritual taxonomies)

### 20.2 Priority

1. Rules (shape + defaults)
2. 50–60 universal roles
3. 150 universal everyday plans
4. 80–100 composable compounds
5. Counter-plans and threat signatures
6. Bundles (discovered by sim)
7. Settlement templates (sim-tested)

### 20.3 GitHub Agent Loop

```
Cron daily     → extract.yml     → LLM batch → validates → PR
PR merged      → counters.yml    → generates counter-plans → PR
Cron weekly    → coverage.yml    → gap report as issue
Manual trigger → targeted extraction from specific source
```
