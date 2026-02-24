# Objects & Spatial Hierarchy

## Object System

### Single Struct + Composable Traits

```
Object {
    id, template, traits: [Trait], condition, quantity,
    owner: EntityRef?,    // legal claim
    holder: EntityRef?,   // physical possession
    location, secrecy, crafted: CraftedInfo?
}

CraftedInfo {
    creator: EntityRef,
    quality: f32,                  // skill at time of creation
    materials: [MaterialRef],      // inputs → affect traits
    recipe: TemplateId,
}
```

### Trait Examples

Flat composable tags with optional payload. No hierarchy.

```
Trait =
    | Perishable(rate)
    | Flammable(ignition, fuel_value)
    | Edible(nutrition, taste)
    | Weapon(damage, speed, range)
    | Armor(protection, encumbrance)
    | Tool(domain, bonus)
    | Container(capacity)
    | Liquid(viscosity)
    | Heavy(weight)
    | Fragile(break_threshold)
    | Valuable(base_price)
    | Degradable(rate)
    | Magical(effect, charge)
    | Concealed(bonus)
    | Stackable(max)
    | Material(type)
    | Poisonous(potency)
    | LightSource(brightness, fuel_rate)
    ...
```

Example objects:

```
iron_sword:   [Weapon(12, 1.0, 1), Heavy(3), Material(iron), Degradable(0.01), Valuable(50)]
torch:        [LightSource(0.7, 1.0), Flammable(0.8, 20), Heavy(1), Weapon(3, 0.8, 1)]
cooked_steak: [Perishable(0.1), Edible(40, 70), Flammable(0.6, 5)]
```

Masterwork items: same traits but `crafted.quality` scales relevant trait values.

### Rules Reference Traits Directly

```
item.condition -= item.trait(Perishable).rate * location.temperature * 0.05  | item.has(Perishable)
item.condition -= location.humidity * 0.01                                   | item.has(Material(iron))
item           => ignite                                                     | item.has(Flammable) AND region.fire > item.trait(Flammable).ignition
```

No type checks, no inheritance casts. Just `has(Trait)` and `trait(T).field`.

### Material Composition Through Crafting

Material traits propagate from inputs. Recipe defines which traits carry forward:

```
CRAFT(iron_sword, materials: [iron_ingot, wood_handle])
=> object gets:
   Material(iron)          // from iron_ingot
   Flammable(0.1, 5)      // from wood_handle, reduced — mostly metal
   crafted.materials = [iron, wood]
```

---

## Owner vs Holder

Owner = social/legal claim (backed by authority). Holder = physical fact.

| Situation              | Owner          | Holder          |
|------------------------|----------------|-----------------|
| Own hammer             | blacksmith     | blacksmith      |
| Stolen sword           | original_owner | thief           |
| Rented workshop        | landlord       | tenant          |
| Lent horse             | lender         | borrower        |
| Abandoned loot         | null           | null            |
| Carried message        | sender         | courier         |
| Commissioned WIP       | knight         | blacksmith      |

Theft = changing holder without owner consent. Applies to spatial nodes too.

---

## Bulk Groups & Materialization

### Bulk Storage

Fungible items stored as aggregate stats:

```
BulkGroup {
    template: ObjectTemplateId,
    quantity: u32,
    avg_condition: f32,
    avg_quality: f32,
}
```

### Implicit → Explicit

Region resources are implicit stats → GATHER → explicit object:

```
region.trees = 500          // implicit
GATHER(trees):
region.trees -= 1
=> create(object, {template: log, traits: [Flammable, Heavy, Material(wood)]})
```

### Materialize on Demand

Objects extracted from bulk groups when:
- Player inspects
- Theft/looting
- Item becomes story-relevant
- Individual condition drops to breakage
- Trade/transfer of specific items

---

## Spatial Hierarchy

Single node type: World → Province → Zone → Region → TileGroup → Tile.

```
SpatialNode {
    children: [SpatialNode],
    owner: EntityRef?,             // legal claim
    holder: EntityRef?,            // physical occupant/tenant
    tags: [ZoneTag],               // purpose designation
    contents: [BulkGroup],         // stored objects (bulk)
    special_items: [ObjectId],     // materialized individual objects
}
```

### Derived Properties

Computed from child tiles on demand, not stored:

```
node.enclosed     = all boundary tiles have walls
node.roofed       = all tiles have ceiling
node.area         = tiles.len()
node.capacity     = f(area, ceiling_height)
node.temperature  = f(enclosed, roofed, ambient, insulation, heat_sources)
node.light        = f(roofed, windows, light_sources, time_of_day)
node.access       = boundary tiles with openings
```

### Everything Is a Spatial Node

| Example          | Enclosed | Roofed    | Tags                        |
|------------------|----------|-----------|-----------------------------|
| Kitchen          | yes      | yes       | `[cooking, storage.food]`   |
| Workshop         | yes      | yes       | `[smithing, storage.metal]` |
| Market square    | no       | no        | `[trading, gathering]`      |
| Farm field       | no       | no        | `[farming]`                 |
| Mine shaft       | yes      | yes (rock)| `[mining, storage.ore]`     |
| Ship deck        | yes      | no        | `[navigation, combat]`      |
| Forest clearing  | no       | no        | `[foraging, camping]`       |

### Tags Drive Action Eligibility

```
CRAFT(food_recipe)    needs tags: [cooking]
CRAFT(iron_sword)     needs tags: [smithing]
TRADE(goods)          needs tags: [trading]
REST                  needs tags: [sleeping]
RESEARCH              needs tags: [library] or [laboratory]
```

### Buildings = Parent Nodes

A building is a spatial node whose children are room nodes:

```
Tavern (tags: [commercial, social])
├── Common room (tags: [gathering, eating])
├── Kitchen (tags: [cooking, storage.food])
├── Room 1 (tags: [sleeping, rental])
└── Cellar (tags: [storage.drink])
```

District → buildings. Settlement → districts. Same structure all the way up.

### Vehicles = Moving Subtrees

A vehicle is a spatial subtree whose tiles share velocity:

```
Merchant Ship (tags: [transport, naval])
├── Deck (tags: [navigation, combat])
├── Hold (tags: [storage.cargo])
├── Cabin (tags: [sleeping, command])
└── Galley (tags: [cooking])
```

### Containers = Just Nodes

Portable containers (backpack, chest) are the same as spatial containers — just smaller and moveable. Both hold BulkGroups and special_items. Both have owner/holder.

### Owner / Holder on Spatial Nodes

| Situation            | Owner            | Holder           |
|----------------------|------------------|------------------|
| Lord's castle room   | lord             | lord             |
| Rented market stall  | market_guild     | merchant         |
| Occupied enemy fort  | original_faction | invading_faction |
| Squatted building    | original_owner   | squatters        |
| Leased farmland      | landowner        | tenant_farmer    |

### Object Secrecy

Objects carry a single `secrecy` value (0–100):

- **secrecy ≤ 30**: Public knowledge — location/ownership determined by vicinity (temporal, spatial, interactional proximity)
- **secrecy > 30**: Private — requires explicit knowledge tracking

High-secrecy objects provide stealth bonuses to actions; low-secrecy objects create stealth penalties (hidden dagger vs ornate greatsword).

MODIFY actions change secrecy: hiding objects increases it, revealing decreases it.
