# Objects & Spatial Hierarchy

## Object System

### Objects Are Nodes with Traits

No separate Object struct. Objects are nodes with composable traits. A sword is a Node with WeaponTrait + ConditionTrait + HeavyTrait + MaterialTrait + ValuableTrait.

```
iron_sword:   Node(Weight=1) + Weapon(12, 1.0, 1) + Heavy(3) + Material(iron) + Condition(0.9, 0.7) + Valuable(50)
torch:        Node(Weight=1) + LightSource(0.7, 1.0) + Flammable(0.8, 20) + Heavy(1) + Weapon(3, 0.8, 1)
cooked_steak: Node(Weight=1) + Perishable(0.1) + Edible(40, 70) + Flammable(0.6, 5)
```

### Trait Examples (all Single, all Fixed Q16.16)

```
WeaponTrait         { Owner, Damage, Speed, Range }
ArmorTrait          { Owner, Protection, Encumbrance }
PerishableTrait     { Owner, Rate }
FlammableTrait      { Owner, Ignition, FuelValue }
EdibleTrait         { Owner, Nutrition, Taste }
ToolTrait           { Owner, Domain: int, Bonus }
ContainerTrait      { Owner, Capacity }
HeavyTrait          { Owner, Mass }
FragileTrait        { Owner, BreakThreshold }
ValuableTrait       { Owner, BasePrice }
StackableTrait      { Owner, Max: int }
MaterialTrait       { Owner, MaterialType: int }
LightSourceTrait    { Owner, Brightness, FuelRate }
ConcealedTrait      { Owner, Bonus }
ConditionTrait      { Owner, Condition, Quality }
ImmaterialTrait     { Owner }
```

See `architecture.md` §5 for full list.

### Rules Reference Traits Directly

```acf
rule item_spoil:
    layer: L2_Items
    scope: Perishable
    condition: NOT contains(Owner, cold_storage)
    effect: Accumulate Condition.Condition rate=-0.02

rule item_rust:
    layer: L2_Items
    scope: Material
    condition: Material.MaterialType == iron
    condition: Climate.Humidity > 50
    effect: Accumulate Condition.Condition rate=-0.01

rule item_ignite:
    layer: L2_Items
    scope: Flammable
    condition: Burning.Fire > Flammable.Ignition
    effect: AddTrait Burning
```

No type checks, no inheritance casts. Scope iterates the trait table directly.

### Material Composition Through Crafting

Material traits propagate from inputs. Recipe defines which traits carry forward:

```
CRAFT(iron_sword, materials: [iron_ingot, wood_handle])
=> node gets:
   Material(iron)          // from iron_ingot
   Flammable(0.1, 5)      // from wood_handle, reduced — mostly metal
```

Masterwork items: same traits but ConditionTrait.Quality scales relevant trait values.

---

## Owner vs Holder

**Owner** = legal claim via OwnedBy relationship trait (backed by authority).
**Holder** = physical fact via ContainerNode.

| Situation              | OwnedBy Target  | ContainerNode    |
|------------------------|------------------|------------------|
| Own hammer             | blacksmith       | blacksmith       |
| Stolen sword           | original_owner   | thief            |
| Rented workshop        | landlord         | tenant           |
| Lent horse             | lender           | borrower         |
| Abandoned loot         | (no OwnedBy)    | ground tile      |
| Carried message        | sender           | courier          |

Theft = changing ContainerNode without OwnedBy holder's consent. Applies to spatial nodes too.

---

## Bulk Groups & Materialization

### Bulk = Weight on Node

Fungible items stored as a single node with Weight > 1:

```
Grain pile:    Node(Weight=500) + Edible + Perishable + Condition
Arrow bundle:  Node(Weight=50) + Weapon
```

ConditionTrait tracks average condition/quality across the batch.

### Implicit → Explicit

Region resources are traits on spatial nodes → GATHER → explicit object node:

```
NaturalResource(Kind=trees, Amount=500) on Region node    // implicit
GATHER(trees):
    NaturalResource.Amount -= 1
    → Create node from log template                        // explicit
```

### Materialize on Demand

Individual objects extracted from bulk nodes (Weight decremented, new Weight=1 node created) when:
- Player inspects
- Theft/looting
- Item becomes story-relevant
- Individual condition drops to breakage
- Trade/transfer of specific items

---

## Spatial Hierarchy

Spatial nodes = any node with SpatialTrait. Scale tracked by `SpatialTrait.ScaleLevel`:

```
enum SpatialScale : byte {
    Tile, Room, Building, Block, District, Settlement,
    Region, Province, Continent, Planet, System, Sector, Empire
}
```

Tile grid: Tile through Settlement (cells, physics, LOS).
Graph: Settlement and above (topology, aggregate stats).
Coordinates local to containing node. Stellar = travel-time edges.

See `architecture.md` §10 for scale and adaptive dt.

### Spatial Traits

```
SpatialTrait        { Owner, ScaleLevel: int, Capacity }
ClimateTrait        { Owner, Temperature, Humidity }
HydrologyTrait      { Owner, Water }
LightingTrait       { Owner, Light }
BurningTrait        { Owner, Fire, Fuel }
SoilTrait           { Owner, Fertility }
PopulatedTrait      { Owner, Population: int, ThreatLevel }
VehicleTrait        { Owner, Speed }
```

### Derived Properties

Computed from contained child nodes, not stored:

```
enclosed     = all boundary tiles have walls
roofed       = all tiles have ceiling
area         = count of contained Tile nodes
capacity     = f(area, ceiling_height)
temperature  = ClimateTrait (propagated via Spread rules)
light        = LightingTrait
access       = boundary tiles with ConnectedTo edges
```

### Everything Is a Spatial Node

| Example          | Tags                        |
|------------------|-----------------------------|
| Kitchen          | `[cooking, storage.food]`   |
| Workshop         | `[smithing, storage.metal]` |
| Market square    | `[trading, gathering]`      |
| Farm field       | `[farming]`                 |
| Mine shaft       | `[mining, storage.ore]`     |
| Ship deck        | `[navigation, combat]`      |

### Tags Drive Action Eligibility

```
CRAFT(food_recipe)    needs tags: [cooking]
CRAFT(iron_sword)     needs tags: [smithing]
TRADE(goods)          needs tags: [trading]
REST                  needs tags: [sleeping]
RESEARCH              needs tags: [library] or [laboratory]
```

### Buildings = Parent Spatial Nodes

A building is a spatial node whose contained children are room nodes:

```
Tavern (SpatialTrait, ScaleLevel=Building)
├── Common room (SpatialTrait, ScaleLevel=Room, tags: [gathering, eating])
├── Kitchen (SpatialTrait, ScaleLevel=Room, tags: [cooking, storage.food])
├── Room 1 (SpatialTrait, ScaleLevel=Room, tags: [sleeping, rental])
└── Cellar (SpatialTrait, ScaleLevel=Room, tags: [storage.drink])
```

District → buildings. Settlement → districts. Same structure all the way up.

### Vehicles = Moving Spatial Subtrees

A vehicle is a spatial node with VehicleTrait whose contained tiles share velocity:

```
Merchant Ship (SpatialTrait + VehicleTrait)
├── Deck (tags: [navigation, combat])
├── Hold (tags: [storage.cargo])
├── Cabin (tags: [sleeping, command])
└── Galley (tags: [cooking])
```

### Containers = Just Nodes

Portable containers (backpack, chest) use ContainerTrait. Any node can contain other nodes via ContainerNode. Both hold items; both can have OwnedBy.

### Connections Between Spatial Nodes

ConnectedTo relationship trait defines edges in the spatial graph:

```
ConnectedTo { Owner, Target, Throughput, TerrainCost, Width, Conductivity }
Adjacent    { Owner, Target }
```

Used for pathfinding flow, heat spread, and connectivity queries. See `spec_pathfinding.md`.

### Object Concealment

ConcealedTrait provides stealth bonuses to actions. ObscurityTrait on virtual items controls knowledge propagation. Hidden dagger (ConcealedTrait) vs ornate greatsword (no ConcealedTrait).
