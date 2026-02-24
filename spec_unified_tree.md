# Unified Tree Architecture

## Core Principle

Every entity — individual, group, army, city, planet — is a node in a single tree. Same struct at every depth. `weight=1` = individual (exact values), `weight>1` = group (aggregate values).

---

## Node Structure

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

---

## Stats at Variable Granularity

Location, equipment, and identity aren't separate hierarchies. They're dimensions in the same stat block, at whatever granularity the node exists at.

### Location

```
weight=5000: Zone(INDUSTRIAL)         // "somewhere in the industrial district"
weight=200:  Region(MINE_3)           // "in mine 3"
weight=1:    Tile(347, 891)           // "at this exact tile"
```

Unsplit groups use probabilistic location distributions:

```
stats.location = Distribution {
    "goldenhaven.homes": 0.6,
    "goldenhaven.fields": 0.25,
    "goldenhaven.market": 0.1,
    "travel.nearby": 0.05,
}
```

### Equipment

```
weight=5000: Category(ARMED)                          // "they have weapons"
weight=200:  Tier(IRON_WEAPONS)                       // "iron-tier weapons"
weight=20:   SubTier(IRON_SWORDS, quality=0.7)        // "iron swords, decent"
weight=1:    Specific(item_id=48832)                  // "this exact sword"
```

---

## Split / Merge / Promote

No fixed subgroup count. Tree splits into arbitrary children when variance exceeds threshold.

| Trigger                          | Action                                     |
|----------------------------------|--------------------------------------------|
| Stat variance > split_threshold  | Split into children, each inherits + delta |
| Child similarity < merge_threshold | Coarsen children back into parent        |
| Story relevance                  | Promote individual to leaf (weight=1)      |
| Relevance lost                   | Reintegrate if divergence low              |

Split children and promoted individuals keep `home_group` pointing to their origin for reintegration.

### Refinement Behavior

When a core node refines (gains children), shells continue deriving from the parent's stats (which is now the weighted combination of children). Over subsequent frames, shells lazily reassign to child cores. When children coarsen, shells re-attach to the parent. **Tree restructuring is invisible to the renderer.**

---

## Shells and Cores

### Architecture

- **Core**: simulation node, exists always, O(core_nodes)
- **Shell**: rendering proxy, exists only when visible, O(visible_shells)

```
Core (tree node):
    weight: 200
    stats.location: Region(MINE_3)
    stats.equipment: Tier(IRON)
    stats.action: Role(MINING)
    stats.health: 0.85
    stats.mood: 0.6

Shell (rendering proxy, one per visible individual):
    core_ref → points to the Core above
    pixel_position: Vec2          // derived: random point within MINE_3
    animation: AnimState          // derived: mining animation
    sprite_variant: u8            // derived: species + equipment sprite
    facing: Direction             // derived: toward nearest rock face
    wobble: f32                   // cosmetic noise
```

200 shells × ~16 bytes + 1 core × ~200 bytes = 3,400 bytes vs 200 × 2,000 = 400,000 bytes for full individual simulation.

### Shell Lifecycle

Shells exist only for visible entities. Off-screen nodes have zero shells.

```
Camera reveals MINE_3:
    Core(miners) has 200 weight, 0 shells
    → spawn 200 shells
    → each picks position within MINE_3 bounds
    → each picks sprite variant from species + equipment tier
    → shells animate independently (staggered phases)

Camera moves away:
    → despawn all 200 shells
    → Core unchanged, simulation continues
```

### Shell Derivation Rules

| Visible stat | Derived from | Method |
|---|---|---|
| Position | core.stats.location | Random point within region, biased toward workplace. Persistent per-shell, slowly drifts. |
| Animation | core.stats.action | Action role → animation set. Phase offset per shell. Speed scaled by skill. |
| Sprite | core.stats.appearance + equipment | Species template + equipment tier → sprite sheet. Variant from shell ID. |
| Facing | position + workplace geometry | Face toward work target (rock face, anvil, crop row). |
| Health overlay | core.stats.health | Fraction of shells get injury overlay = 1.0 - core.health. |
| Mood indicator | core.stats.mood | Thought bubble on fraction of shells near break threshold. |

Shells don't need to be accurate about *which* individual is wounded — just that the right *proportion* looks wounded.

---

## Two Tick Loops

### Simulation Tick (1–10 Hz, game-time)

Walks the core tree. At each node:
- Update background stats (production, consumption, decay, needs)
- Check variance against thresholds → refine or coarsen
- Propagate dirty summaries upward
- Generate stochastic events (scaled by weight)
- Process events → refine targeted subtrees

Cost: O(number of core nodes). Typically ~100–500 nodes regardless of population.

### Render Tick (30–60 Hz, real-time)

For each visible core:
- Ensure shells exist (spawn/despawn as visibility changes)
- Update shell positions (drift, animation advance)
- Derive visual state from core stats
- Draw shells

Cost: O(number of visible shells). Typically ~500–2000.

The two loops don't share mutable state. Render tick reads core stats (read-only). Simulation tick never touches shells. Trivially parallelizable.

---

## Edge Cases

### Player-Tracked Pawn

Player clicks a shell → core refines to leaf depth for full individual stats. On deselect, leaf stays refined with TTL before coarsening back.

### Visible Combat

Cores in combat zone refine to weight=5–10 sub-squads. Shells animate combat using sub-squad stats. Individual kill animations are cosmetic — a shell "dies" when its sub-squad's weight decrements. Only named heroes refine to leaf.

### Off-Screen Events

No shells. Simulation resolves entirely at compound granularity. When the player scrolls to the aftermath, shells spawn showing the new state.
