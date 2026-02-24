# Expression Language

## Two Primitives

All world state access resolves to:

```
STAT(entity, path)                              → value
EDGE(from, to, type, [path], [depth], [agg])    → value
```

Everything else is sugar built from these two.

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

| Agg            | Returns                     |
|----------------|-----------------------------|
| exists         | bool: any path within depth |
| min_depth      | int: shortest hop count     |
| sum(field)     | float: total along path     |
| min(field)     | float: bottleneck           |
| max(field)     | float: peak                 |
| product(field) | float: decay chains         |

Multi-type paths: compose via AND of single-type EDGE checks with bound intermediate variable:

```
// Is bob in a faction allied with alice's faction?
edge(bob, $group, member_of) AND edge($group, $target, allied_with)
```

---

## Full Grammar

```
Expr =
    | entity.path | edge(from, to, type, ...) | $param | Number | Bool
    | prob(Expr) | random(min, max) | sigmoid(Expr)
    | Expr { + - * / } Expr | min | max | abs
    | Expr { < > == != >= <= } Expr
    | Expr AND Expr | Expr OR Expr | NOT Expr
    | count(entities WHERE Expr) | any(...) | sum(...)
```

---

## Edge Types

| Type | From → To | Payload |
|------|-----------|---------|
| social | entity → entity | debt, reputation, affection, familiarity |
| member_of | entity → group | rank, since |
| owns | entity → object/spatial | — |
| holds | entity → object/spatial | — |
| contains | container → object | — |
| located_in | entity → spatial_node | — |
| connected_to | spatial → spatial | throughput, terrain_cost |
| adjacent | spatial → spatial | — |
| employed_by | entity → entity/group | contract_id |
| parent_of | entity → entity | — |
| parent_template | entity → template | — |
| hostile_to | faction → faction | — |
| allied_with | faction → faction | contract_id |
| knows_about | entity → entity/fact | confidence, freshness |
| guards | entity → entity/spatial | — |
| delegates_to | entity → entity | scope, grants |
| supplies | spatial → spatial | resource_type, flow_rate |

---

## Effects

```
Effect =
    | entity.path += Expr          // ADD
    | entity.path -= Expr          // REMOVE
    | entity.path = Expr           // SET
    | entity => destroy            // DESTROY
    | entity => create(template_id, { params })  // CREATE
    | edge(a, b, type) => add({ payload })       // ADD_EDGE
    | edge(a, b, type) => remove                 // REMOVE_EDGE
```

---

## Sugar

| Sugar              | Resolves To                                  |
|--------------------|----------------------------------------------|
| HAS(e, item, n)   | e.inventory[item] >= n                       |
| AT(e, loc)         | e.location == loc                            |
| ALIVE(e)           | e.health > 0                                 |
| CONTROLS(f, r)     | edge(f, r, controls)                         |
| CONNECTED(a, b)    | edge(a, b, connected_to, 99, exists)         |

Named sugar defined as bare expressions in data files. Resolved at load time.
