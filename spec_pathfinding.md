# Pathfinding (Flow-Based)

## The Problem

Individual A* costs O(N × map_size) per tick. With 10,000 entities that's catastrophic. The solution mirrors the rest of the system: pathfinding operates at whatever granularity the tree node exists at.

---

## The Region Graph

The spatial hierarchy defines a graph at every level:

```
Tile graph:      100,000 nodes, 4–8 edges each    ← individual pathfinding
Region graph:    ~200 nodes, 3–6 edges each        ← group pathfinding
Zone graph:      ~15 nodes, 2–4 edges each         ← strategic movement
Province graph:  ~5 nodes                           ← world map
```

Each edge is a **connection** with properties:

```
Connection {
    regions: (RegionId, RegionId),
    width: u16,              // tiles wide
    throughput: f32,         // entities per game-second
    terrain_cost: f32,       // speed modifier (road=0.5, swamp=3.0)
    current_flow: f32,       // traffic right now
    capacity_remaining: f32, // throughput - current_flow
}
```

Throughput is the key concept. A 2-tile doorway has much lower throughput than a 10-tile road. A group of 200 doesn't teleport through — it queues.

```
throughput = width_in_tiles × speed_modifier × terrain_factor
speed_modifier:   road=2.0, floor=1.0, rough=0.5, stairs=0.3
terrain_factor:   flat=1.0, uphill=0.6, downhill=1.2
```

---

## Movement by Tree Depth

### Leaf (weight=1): Tile-Level A*

Standard A* on tile graph. Used for player-directed movement, pursuit/flee, short-range within a region. Hierarchical A* finds region-level path first → tile A* runs only within each region along route. Search space: ~500 tiles.

### Compound (5–200): Region-Level Flow

The group **flows**. A* on region graph (trivial: 200 nodes) → flow rate determined by bottleneck connection → group transitions through intermediate regions.

```
t=0:    location = Region(MINE_3),      weight=200
t=10:   40 in CORRIDOR_7, 160 in MINE_3
t=30:   120 in HALL_2, 60 in CORRIDOR_7, 20 in MINE_3
t=55:   location = Region(BARRACKS_1),  weight=200
```

During transition, the node temporarily refines along the spatial axis into sub-groups per route segment. These coarsen back once the full group arrives.

### Large Compound (1000+): Zone-Level Strategic Flow

Movement is essentially a resource transfer: "500 soldiers flow from ZONE_MILITARY to ZONE_FRONTIER at rate 50/day, limited by road capacity."

### Contention

Multiple groups using the same connection compete for throughput. Capacity allocated proportionally or by priority. This naturally creates traffic jams without individual collision.

---

## Flow Fields

For repeated/concurrent movements toward the same destination, precompute a flow field on the region graph: for every region, store the next-hop toward the destination.

```
Flow field toward MARKET_SQUARE:
    FARM_1 → ROAD_NORTH → GATE_1 → MARKET_SQUARE
    MINE_3 → CORRIDOR_7 → HALL_2 → MARKET_SQUARE
    BARRACKS_1 → HALL_2 → MARKET_SQUARE
```

Cheap to compute (Dijkstra on ~200 nodes). Cached. Invalidated when topology changes (wall built, door locked, bridge destroyed).

Common persistent flow fields:
- Dining halls (daily meal flow)
- Workplaces (morning commute)
- Markets / warehouses (hauling)
- Barracks (military mustering)

These are the "desire paths" of the city — visualizable as traffic heat maps.

---

## Throughput Simulation

Each tick:

```
for each connection:
    current_flow = sum(group.flow_rate for group using connection)

    if current_flow > connection.throughput:
        congestion_factor = connection.throughput / current_flow  // <1
        for group in groups_using(connection):
            group.effective_flow_rate *= congestion_factor
    else:
        capacity_remaining = connection.throughput - current_flow
```

Handles: bottleneck slowdown, traffic jams, rush hour, siege chokepoints (defenders hold narrow pass because throughput limits attacker engagement).

---

## Shell Movement (Rendering)

Simulation says "200 miners transitioning from MINE_3 to BARRACKS_1 via CORRIDOR_7 at rate 4/sec." Renderer shows 200 sprites walking.

1. **Route geometry**: region-level route → coarse path through region centroids / connection midpoints (the "spine")
2. **Shell distribution**: shells spread along spine proportional to transition progress
3. **Lateral spread**: shells offset perpendicular based on connection width (narrow corridor = tight column, wide road = spread)
4. **Per-shell noise**: small random offsets + staggered animation
5. **Bottleneck queuing**: shells bunch at narrow connections, trickle through — derived from flow rate, not simulated per-shell

No per-shell pathfinding. No collision detection between shells.

---

## Groups Split During Movement

If a group encounters a fork (destination reachable via two paths), it can split to increase total throughput:

```
Cohort(200) heading to DINING_HALL
    Route A: MINE → CORRIDOR_NORTH → DINING_HALL  (throughput 3/sec)
    Route B: MINE → CORRIDOR_SOUTH → DINING_HALL  (throughput 5/sec)

    Split proportional to throughput:
    SubGroup_A(75)  takes Route A
    SubGroup_B(125) takes Route B

    Both arrive → coarsen back into one group at DINING_HALL
```

This is spatial-axis refinement triggered by pathfinding. Sub-groups exist only during transit.

---

## Hauling as Flow

Item transport = flow between inventory nodes.

```
Warehouse(MINE_3, iron_ore, 500) → Warehouse(SMELTER_2, iron_ore, 0)
Flow rate = hauler_throughput(connection) × hauler_count
```

Haulers aren't individually simulated. They amplify connection throughput. Visually, shells walk back and forth carrying resource sprites.

### Supply Chain as Flow Network

```
FARM →(grain)→ MILL →(flour)→ BAKERY →(bread)→ DINING_HALL
       20/day     15/day         12/day

Bottleneck: BAKERY output. Fix: build second bakery or upgrade road.
```

Each arrow is a flow edge with throughput = min(connection capacity, production rate).

---

## Flow Congestion Events

When congestion exceeds thresholds:
- **Mood penalty**: "crowded corridor" modifier
- **Lateness**: workers arriving late → reduced production
- **Stampede risk**: extreme congestion + low morale → stochastic event → refine entities to solo
- **Player signal**: traffic heat map highlights bottlenecks

---

## Connection to the Unified Tree

Flow state is a stat dimension on the node:

```
stats.movement:
    Stationary { region: RegionId }

    Flowing {
        route: RegionId[],
        progress: f32,         // 0.0=origin, 1.0=destination
        flow_rate: f32,        // effective after congestion
        bottleneck: ConnectionId  // what's limiting, for UI
    }
```

---

## Cost Summary

| Tree depth | Method | Graph size | Cost | When used |
|---|---|---|---|---|
| Leaf (1) | Hierarchical A* | ~500 tiles | O(500) | Player-directed, pursuit |
| Compound (5–200) | Region A* + flow | ~200 regions | O(200) | Normal group movement |
| Large (1000+) | Zone A* + flow | ~15 zones | O(15) | Army march, migration |
| Strategic | Direct edge | ~5 provinces | O(1) | World map transfer |

Total per tick: O(region_graph) for flow field + O(active_flows) for throughput. Both independent of population.
