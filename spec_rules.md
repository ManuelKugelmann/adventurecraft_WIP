# Rules (World Mechanics)

Rules define state changes triggered by conditions. No agency. Evaluated every tick, scaled by dt.

---

## Structure

```acf
rule <id> [tags] {
    when <Expr>
    rate = <Expr>            # deterministic continuous
    prob = <Expr>            # stochastic event
    effect: <Effect>
}
```

`rate` and `prob` are mutually exclusive. Both scale by dt.

### Multi-Rule Shorthand

Multiple rules in one block when they share context:

```acf
rule fire [physics, L0] {
    spread:  when region.fire > 0 AND adjacent.has(Flammable),
             prob = region.fire * adjacent.flammability * 0.01,
             effect: adjacent.fire += 20

    consume: when region.fire > 0,
             rate = region.fire * 0.1,
             effect: region.fuel -= rate * dt

    quench:  when region.fire > 0 AND region.water > 0,
             rate = region.water * 0.5,
             effect: region.fire -= rate * dt

    starve:  when region.fire > 0 AND region.fuel <= 0,
             rate = 20,
             effect: region.fire -= rate * dt
}
```

---

## Layers (Dependency Order)

| Layer | Domain    | Agency | Depends On | Typical dt    |
|-------|-----------|--------|------------|---------------|
| L0    | Physics   | None   | Nothing    | minutes       |
| L1    | Biology   | None   | L0         | hours–days    |
| L2    | Items     | None   | L0         | days          |
| L3    | Social    | None*  | L0–L2      | days          |
| L4    | Economic  | None*  | L0–L3      | days–weeks    |

\*L3/L4 rules are agentless (things that happen TO actors). Rumor spreads whether anyone intends it. Prices drift without anyone deciding.

No circular dependencies between layers. Higher layers may fire less often.

---

## Batch Modes (Auto-Selected)

```
Accumulate(rate * dt)                  deterministic continuous
BernoulliOnce(1 - (1-p)^dt)           did event fire?
PoissonCount(p * dt)                   how many times?
NormalApprox(mean*dt, var*dt)          CLT for large groups
TimeToThreshold(remaining/rate)        skip ticks entirely
```

### Selection Logic

```
single entity + small dt   → exact roll or accumulate
single entity + large dt   → BernoulliOnce or GeometricTime
group + small dt            → PoissonCount
group + large dt            → NormalApprox
goal has threshold          → TimeToThreshold
```

---

## Adaptive Timestep

```
Combat / player-observed:   dt = minutes
Local settlement:           dt = 1 day
Off-screen peacetime:       dt = 30 days
World gen / history:        dt = 1 year
```

Same rules. Same evaluator. Different dt.

---

## Rules as Switches

```
rule.prob = 0 → rule never fires
            → every plan with wait on dependent condition: step.prob = 0
            → plan.confidence = 0
            → planner never selects it
            → cascades to compounds, counters, roles
```

Game profiles are rule value tables (see Profiles section in `spec_infrastructure.md`).

---

## L0 — Physics

```acf
rule temperature [physics, L0] {
    solar:    when region.exposed_sky > 0,
              rate = region.latitude_heat * world.season_modifier,
              effect: region.temperature += rate * dt

    conduct:  when edge(region, adj, adjacent),
              rate = (adj.temperature - region.temperature) * edge(region, adj, adjacent).conductivity,
              effect: region.temperature += rate * dt

    altitude: rate = region.altitude * 0.006,
              effect: region.temperature -= rate
}

rule water [physics, L0] {
    evaporate: when region.water > 0,
               rate = region.temperature * 0.05,
               effect: region.water -= rate * dt

    rain:      when region.humidity > 70,
               prob = region.humidity * 0.01,
               effect: region.water += 30

    flow:      when region.water > region.capacity AND edge(region, adj, adjacent),
               rate = (region.water - region.capacity) * edge(region, adj, adjacent).flow_rate,
               effect: region.water -= rate * dt,
               effect: adj.water += rate * dt

    freeze:    when region.temperature < 0 AND region.water > 0,
               rate = region.water * 0.1,
               effect: region.ice += rate * dt,
               effect: region.water -= rate * dt

    thaw:      when region.temperature > 0 AND region.ice > 0,
               rate = region.ice * 0.1,
               effect: region.water += rate * dt,
               effect: region.ice -= rate * dt
}

rule fire [physics, L0] {
    spread:  when region.fire > 0 AND adjacent.has(Flammable),
             prob = region.fire * adjacent.flammability * 0.01,
             effect: adjacent.fire += 20

    consume: when region.fire > 0,
             rate = region.fire * 0.1,
             effect: region.fuel -= rate * dt

    quench:  when region.fire > 0 AND region.water > 0,
             rate = region.water * 0.5,
             effect: region.fire -= rate * dt

    starve:  when region.fire > 0 AND region.fuel <= 0,
             rate = 20,
             effect: region.fire -= rate * dt
}

rule light [physics, L0] {
    surface: effect: region.light = sun_curve(world.time, region.latitude)
    underground: when region.depth > 0,
                 effect: region.light = 0
}
```

## L1 — Biology

```acf
rule crop_growth [biology, L1] {
    grow:    when field.planted AND region.temperature > 5 AND region.water > 20 AND region.light > 0,
             rate = field.fertility * 0.1,
             effect: field.growth += rate * dt

    wilt:    when field.planted AND region.water < 10,
             rate = 0.5,
             effect: field.health -= rate * dt

    die:     when field.health <= 0,
             effect: field.planted = false

    ready:   when field.growth >= field.crop.maturity,
             effect: field.crop.ready = true
}

rule metabolism [biology, L1] {
    hunger:  rate = entity.weight * 0.01,
             effect: entity.hunger += rate * dt

    thirst:  rate = entity.weight * 0.02 * (1 + region.temperature * 0.01),
             effect: entity.thirst += rate * dt

    fatigue: rate = entity.activity_level * 0.05,
             effect: entity.fatigue += rate * dt

    starve:  when entity.hunger > 90,
             rate = 0.5,
             effect: entity.health -= rate * dt

    dehydrate: when entity.thirst > 90,
               rate = 1.0,
               effect: entity.health -= rate * dt

    death:   when entity.health <= 0,
             effect: entity => destroy,
             effect: entity.location => create(corpse, { source = entity })
}

rule healing [biology, L1] {
    natural: when entity.health < entity.health_max AND entity.hunger < 50 AND entity.fatigue < 50,
             rate = entity.attributes.bod * 0.01,
             effect: entity.health += rate * dt
}

rule disease [biology, L1] {
    spread:  when entity.infected AND co_located(entity, target) AND NOT target.immune,
             prob = entity.contagion * region.population / region.capacity * 0.01,
             effect: target.infected = true

    progress: when entity.infected,
              rate = entity.disease.virulence * (1 - entity.attributes.bod * 0.005),
              effect: entity.health -= rate * dt

    recover: when entity.infected AND entity.health > 50,
             prob = entity.attributes.bod * 0.01,
             effect: entity.infected = false,
             effect: entity.immune = true
}

rule aging [biology, L1] {
    age:     rate = 1,
             effect: entity.age += rate * dt

    old_age: when entity.age > entity.lifespan * 0.8,
             prob = (entity.age - entity.lifespan * 0.8) / (entity.lifespan * 0.2) * 0.01,
             effect: entity.health -= 1
}
```

## L2 — Items

```acf
rule item_decay [items, L2] {
    spoil:     when item.has(Perishable) AND NOT item.location.has(cold_storage),
               rate = 0.02 * item.location.temperature / 20,
               effect: item.condition -= rate * dt

    rust:      when item.has(Material(iron)) AND item.location.humidity > 50,
               rate = item.location.humidity * 0.01,
               effect: item.condition -= rate * dt

    wear:      when item.equipped AND item.in_use,
               rate = item.use_rate * 0.01,
               effect: item.condition -= rate * dt

    break:     when item.condition <= 0,
               effect: item => destroy
}

rule structure_decay [items, L2] {
    weather:   when structure.exposed_sky > 0,
               rate = 0.01 * (1 + region.water * 0.01),
               effect: structure.condition -= rate * dt

    overload:  when structure.load > structure.capacity,
               rate = (structure.load - structure.capacity) * 0.1,
               effect: structure.condition -= rate * dt

    collapse:  when structure.condition <= 0,
               effect: structure => destroy
}

rule fuel [items, L2] {
    burn:      when fuel.in_use,
               rate = region.fire * 0.1,
               effect: fuel.amount -= rate * dt

    lamp:      when light_source.lit,
               rate = 1,
               effect: light_source.fuel -= rate * dt

    extinguish: when light_source.fuel <= 0,
                effect: light_source.lit = false
}
```

## L3 — Social

```acf
rule social_judgment [social, L3] {
    disapprove: when observer.knowledge.contains($expectation)
                AND observer.witnessed($actor, $action)
                AND $action != $expectation.expected_behavior,
                effect: edge(observer, $actor, social).reputation -= $expectation.weight

    approve:    when observer.knowledge.contains($expectation)
                AND observer.witnessed($actor, $action)
                AND $action == $expectation.expected_behavior,
                effect: edge(observer, $actor, social).reputation += $expectation.weight * 0.3
}

rule familiarity [social, L3] {
    grow:      when a.location == b.location,
               rate = 1,
               effect: edge(a, b, social).familiarity += rate * dt

    fade:      when a.location != b.location,
               rate = 1,
               effect: edge(a, b, social).familiarity -= rate * dt

    affection_fade: when edge(a, b, social).familiarity < 30,
                    rate = 0.01,
                    effect: edge(a, b, social).affection *= (1 - rate * dt)

    debt_resentment: when edge(a, b, social).debt > 50,
                     rate = 1,
                     effect: edge(a, b, social).affection -= rate * dt
}

rule knowledge_propagation [social, L3] {
    spread:    when vitem.obscurity < 0.3 AND co_located(vitem.holder, target),
               prob = (1 - vitem.obscurity) * edge(vitem.holder, target, social).familiarity * 0.01,
               effect: target => copy_vitem(vitem, confidence * 0.8)

    decay:     rate = 0.05,
               effect: vitem.freshness -= rate * dt

    normalize: when vitem.copy_count > 0,
               rate = vitem.copy_count * 0.02,
               effect: vitem.obscurity -= rate * dt
}

rule mood [social, L3] {
    hunger:    when entity.hunger > 50,
               effect: entity.mood -= 3 * dt

    comfort:   when entity.housing_quality > 50,
               effect: entity.mood += 1 * dt

    crowding:  when region.population > region.capacity,
               effect: entity.mood -= 2 * dt

    threat:    when region.threat_level > 50,
               effect: entity.mood -= region.threat_level * 0.05 * dt

    unfulfilled: when entity.drives[$drive] > 70 AND entity.fulfillment[$drive] < 30,
                 effect: entity.mood -= 5 * dt

    recovery:  when entity.mood < 50 AND entity.hunger < 30 AND entity.health > 70,
               effect: entity.mood += 2 * dt
}

rule unrest [social, L3] {
    build:     when region.avg_mood < 25 AND region.authority_strength < 40,
               rate = 5,
               effect: region.unrest += rate * dt

    revolt:    when region.unrest > 90,
               effect: region => create(rebel_faction)
}
```

## L4 — Economic

```acf
rule supply_demand [economic, L4] {
    price_up:   when region.demand[$good] > region.supply[$good],
                rate = (region.demand[$good] - region.supply[$good]) * 0.01,
                effect: region.price[$good] += rate * dt

    price_down: when region.supply[$good] > region.demand[$good],
                rate = (region.supply[$good] - region.demand[$good]) * 0.01,
                effect: region.price[$good] -= rate * dt

    floor:      when region.price[$good] < 1,
                effect: region.price[$good] = 1
}

rule supply_depletion [economic, L4] {
    consume:   when entity.weight > 0 AND entity.supplies > 0,
               rate = entity.weight * entity.consumption_rate,
               effect: entity.supplies -= rate * dt
}
```
