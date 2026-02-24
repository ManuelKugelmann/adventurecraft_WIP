# Unified Tree Architecture

## Core Principle

Every entity — individual, group, army, city, planet, item, knowledge, contract — is a Node. Same struct at every depth. `Weight=1` = individual (exact values), `Weight>1` = group (aggregate values). Traits provide all differentiation.

---

## Node Structure

```
Node {
    Id: NodeId                  // 32-bit monotonic
    Template: TemplateId
    Weight: int32               // 1=individual, >1=group/batch
    ContainerNode: NodeId       // physical: where I am (null if top-level)
    ParentNode: NodeId          // hierarchy: group/org parent (null if root)
    Flags: byte
}   // ~20 bytes, fully blittable
```

Two trees, reverse-mapped via external indices:

```
ContainerIndex: NativeParallelMultiHashMap<NodeId, NodeId>   // what's inside me
HierarchyIndex: NativeParallelMultiHashMap<NodeId, NodeId>   // my group members
```

Updated when ContainerNode or ParentNode changes.

### Everything Is a Node

| Node | Weight | Key Traits |
|------|--------|-----------|
| Person | 1 | Vitals, Attributes, Skills, Drives, Agency |
| Horse | 1 | Vitals, Attributes |
| Garrison | 200 | Vitals, Attributes, Skills, Drives |
| Grain pile | 500 | Edible, Perishable, Condition |
| Sword | 1 | Weapon, Condition |
| Arrow bundle | 50 | Weapon |
| Ship | 1 | Spatial, Vehicle, Vitals, Condition |
| Tile | 1 | Spatial, Climate |
| Planet | 1 | Spatial, Climate, NaturalResource |
| Faction | 10000 | Vitals, Attributes, Skills, Drives |
| Knowledge | 1 | Immaterial, Mirrors, StatCopy, Obscurity |
| Contract | 1 | Immaterial, ContractTerms |
| Authority claim | 1 | Immaterial, AuthStrength, AuthScope |

See `architecture.md` §4 for full table and `architecture.md` §5 for all trait definitions.

---

## Two Trees

### Containment (physical tree)

```
sword.ContainerNode = alice
alice.ContainerNode = ship_cabin
ship_cabin.ContainerNode = ship
ship.ContainerNode = ocean_tile
```

A node is "a place" if it has SpatialTrait. Any node can contain other nodes. Held items: ContainerNode = holder. Position derived by walking chain.

### Hierarchy (organizational tree)

```
soldier.ParentNode = squad           Weight=1
squad.ParentNode = company           Weight=49
company.ParentNode = regiment        Weight=500
```

### Ownership

Legal claim, separate from containment. OwnedBy relationship trait. Alice owns sword even after dropping it (changing ContainerNode doesn't affect OwnedBy).

---

## Traits Replace StatBlocks

No monolithic StatBlock. Each aspect of an entity is a separate trait table:

- **Single traits** (max one per node): Vitals, Attributes, Skills, Drives, Agency, Weapon, Armor, Spatial, etc.
- **Multi traits** (many per node): Social, MemberOf, ConnectedTo, ActiveRole, etc.

Storage: `SingleTraitTable<T>` with `NativeHashMap<NodeId, int>` for O(1) lookup. `MultiTraitTable<T>` with `NativeParallelMultiHashMap` for 1:n. See `architecture.md` §6.

---

## Stats at Variable Granularity

Location, equipment, and identity aren't separate hierarchies. They're whatever granularity the node exists at.

### Location

```
Weight=5000: Spatial at Zone level        // "somewhere in the industrial district"
Weight=200:  Spatial at Region level      // "in mine 3"
Weight=1:    Exact tile position          // "at this exact tile"
```

### Equipment

```
Weight=5000: Category(ARMED)                          // "they have weapons"
Weight=200:  Tier(IRON_WEAPONS)                       // "iron-tier weapons"
Weight=1:    Specific item node                       // "this exact sword"
```

---

## Split / Merge / Promote

No fixed subgroup count. Tree splits into arbitrary children when variance exceeds threshold.

| Trigger                          | Action                                     |
|----------------------------------|--------------------------------------------|
| Stat variance > split_threshold  | Create child nodes, distribute stats, adjust Weight |
| Child similarity < merge_threshold | Collapse children back, bump parent Weight |
| Story relevance                  | Split off Weight=1 child (promote)         |
| Relevance lost                   | Reintegrate if divergence low              |

Split: variance exceeds threshold → create child nodes, distribute stats, adjust Weight.
Merge: children similar → collapse back, bump parent Weight.
Promote: split off Weight=1 child for story relevance. Remains child of group (ParentNode still points to group).

---

## Shells and Cores

### Architecture

- **Core**: simulation node, exists always, O(core_nodes)
- **Shell**: rendering proxy, exists only when visible, O(visible_shells)

```
Core (tree node):
    Weight: 200
    Vitals.Health: 0.85
    Vitals.Mood: 0.6
    Spatial at Region(MINE_3)
    ActiveRole: mining

Shell (rendering proxy, one per visible individual):
    core_ref → points to the Core above
    pixel_position: Vec2          // derived: random point within MINE_3
    animation: AnimState          // derived: mining animation
    sprite_variant: u8            // derived: species + equipment sprite
    facing: Direction             // derived: toward nearest rock face
    wobble: f32                   // cosmetic noise
```

200 shells × ~16 bytes + 1 core node + traits ≈ 3,400 bytes vs 200 × full individual simulation.

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
| Position | ContainerNode chain | Random point within region, biased toward workplace. Persistent per-shell, slowly drifts. |
| Animation | ActiveRole | Action role → animation set. Phase offset per shell. Speed scaled by skill. |
| Sprite | Template + equipment traits | Species template + equipment tier → sprite sheet. Variant from shell ID. |
| Health overlay | Vitals.Health | Fraction of shells get injury overlay = 1.0 - health. |
| Mood indicator | Vitals.Mood | Thought bubble on fraction of shells near break threshold. |

Shells don't need to be accurate about *which* individual is wounded — just that the right *proportion* looks wounded.

### Refinement Behavior

When a core node refines (gains children), shells continue deriving from the parent's stats (weighted combination of children). Over subsequent frames, shells lazily reassign to child cores. When children coarsen, shells re-attach to the parent. **Tree restructuring is invisible to the renderer.**

---

## Two Tick Loops

### Simulation Tick (1–10 Hz, game-time)

Walks the core tree. At each node:
- Update background stats (production, consumption, decay, needs)
- Check variance against thresholds → refine or coarsen
- Propagate dirty summaries upward
- Generate stochastic events (scaled by Weight)
- Process events → refine targeted subtrees

Cost: O(number of core nodes). Typically ~100–500 nodes regardless of population.

### Render Tick (30–60 Hz, real-time)

For each visible core:
- Ensure shells exist (spawn/despawn as visibility changes)
- Update shell positions (drift, animation advance)
- Derive visual state from core traits
- Draw shells

Cost: O(number of visible shells). Typically ~500–2000.

The two loops don't share mutable state. Render tick reads core traits via `ToFloat()` gate (read-only). Simulation tick never touches shells.

---

## Edge Cases

### Player-Tracked Pawn

Player clicks a shell → core refines to leaf depth for full individual traits. On deselect, leaf stays refined with TTL before coarsening back.

### Visible Combat

Cores in combat zone refine to Weight=5–10 sub-squads. Shells animate combat using sub-squad traits. Individual kill animations are cosmetic — a shell "dies" when its sub-squad's Weight decrements. Only named heroes refine to leaf.

### Off-Screen Events

No shells. Simulation resolves entirely at compound granularity. When the player scrolls to the aftermath, shells spawn showing the new state.
