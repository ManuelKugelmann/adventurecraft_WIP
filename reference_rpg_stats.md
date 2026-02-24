# Expression Language

## Two Primitives

All conditions and values resolve to two world-state reads:

```
STAT(entity, path)                              // property of one entity
EDGE(from, to, type, [path], [depth], [agg])    // relationship between two entities
```

Everything else — HAS, ALIVE, AT, CONTROLS, VISIBLE — is sugar built from these two.

---

## Syntax

```
// Property access
e.health
e.skills.combat
e.inventory[food]
e.drives.dominance
region.temperature
item.trait(Weapon).damage

// Edge access
edge(a, b, social).affection
edge(a, b, member_of)                   // existence check
edge(a, b, connected_to).throughput

// Trait checks
item.has(Perishable)
item.trait(Flammable).ignition

// Mutations
e.health += 5
e.mood -= 3
e.infected = true
item => destroy
region => create(template, {params})
edge(a, b, social).affection += 10
```

---

## EDGE Query Modes

```
edge(a, b, type)                         // direct: single hop, existence or payload
edge(a, b, type, depth)                  // transitive: any path within N hops
edge(a, b, type, depth, agg)             // transitive + aggregate along path
```

| Agg | Use |
|---|---|
| `exists` | any path within depth (default) |
| `min_depth` | shortest hop count |
| `sum(field)` | total along path |
| `min(field)` | bottleneck |
| `max(field)` | peak |
| `product(field)` | decay chains (delegation strength) |

Multi-type paths: compose via AND of single-type EDGE checks with bound intermediate variable.

```
// Is bob in a faction allied with alice's faction?
edge(bob, $group, member_of) AND edge($group, $target, allied_with)
```

---

## Full Expression Grammar

```
Expr =
    // Terminals
    | entity.path                                → value
    | edge(from, to, type, [path], [depth], [agg]) → value
    | const(literal)                              → value
    | $param                                      → value
    | prob(Expr)                                   → 0-1
    | random(min, max)                             → value

    // Math
    | Expr { +, -, *, / } Expr
    | min(Expr, Expr) | max(Expr, Expr) | abs(Expr)

    // Comparison → bool
    | Expr { <, >, ==, != , >=, <= } Expr

    // Logical
    | Expr AND Expr | Expr OR Expr | NOT Expr
```

---

## Effects

```
Effect =
    | SET(entity, path, Expr)
    | ADD(entity, path, Expr)
    | REMOVE(entity, path, Expr)
    | CREATE(template_id, params)
    | DESTROY(entity)
    | ADD_EDGE(from, to, type, payload)
    | REMOVE_EDGE(from, to, type)
```

---

## Edge Types

| Type | From → To | Payload |
|---|---|---|
| `social` | entity → entity | debt, reputation, affection, familiarity |
| `member_of` | entity → group | rank, since |
| `owns` | entity → object/spatial | — |
| `holds` | entity → object/spatial | — |
| `contains` | container → object | — |
| `located_in` | entity → spatial_node | — |
| `connected_to` | spatial → spatial | throughput, terrain_cost |
| `adjacent` | spatial → spatial | — |
| `employed_by` | entity → entity/group | contract_id |
| `delegates_to` | entity → entity | scope, grants |
| `parent_template` | entity → template | — |
| `supplies` | spatial → spatial | resource_type, flow_rate |
| `hostile_to` | faction → faction | — |
| `allied_with` | faction → faction | contract_id |
| `guards` | entity → entity/spatial | — |
| `knows_about` | entity → entity/fact | confidence, freshness |
| `parent_of` | entity → entity | — |

---

## Sugar Examples

| Sugar | Resolves To |
|---|---|
| `HAS(e, iron, 5)` | `e.inventory[iron] >= 5` |
| `AT(e, mine_3)` | `e.location == mine_3` |
| `ALIVE(e)` | `e.health > 0` |
| `CONTROLS(f, r)` | `edge(f, r, controls)` |
| `CONNECTED(a, b)` | `edge(a, b, connected_to, 99, exists)` |

Sugar forms can exist as named compound conditions in the template library — validated at load time like everything else.
