# Rules (World Mechanics)

Rules define state changes triggered by conditions. No agency. Evaluated every tick, scaled by dt. Authored in `.acf`, compiled to IR (see `spec_expression_language.md` and `architecture.md` §8).

---

## Structure

```acf
rule <id>:
    layer: <L0..L4>
    scope: <TraitKind>           # iterate this trait table
    condition: <Expr>            # optional, repeatable, all AND
    effect: <Op> <Trait.Field> <params>
```

Scope determines what the rule iterates. `scope: Vitals` iterates all nodes with VitalsTrait. `scope: ConnectedTo` iterates all ConnectedTo relationship instances.

### Rule Struct (registered at startup)

```
WorldRule {
    Id: int
    Layer: byte                     // L0..L4
    Scope: ushort                   // TraitKind to iterate
    MinScale: byte                  // skip nodes below this scale
    MaxScale: byte                  // skip nodes above this scale
    Probability: Fixed
    Conditions: RuleCondition[]
    Effects: RuleEffect[]

    // Auto-derived at registration:
    BatchModes: BatchMode[]
    MaxDt: int
    ThresholdPairs: ThresholdPair[]
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

No circular dependencies between layers. Higher layers may fire less often. Layers execute sequentially L0→L4. See `architecture.md` §9.

---

## Elementary Ops

```
Accumulate      field += rate * dt  (or field += value - mitigator)
Decay           field moves toward floor/ceiling at rate * dt
Set             field = value
Transfer        move value from source field to target field
Spread          propagate value along connections by conductivity
Create          spawn node from template
Destroy         remove node
AddTrait        attach trait to node
RemoveTrait     detach trait from node
```

---

## Batch Modes (Auto-Derived)

| Effect | Deterministic | Stochastic |
|--------|--------------|------------|
| Accumulate | Linear | NormalApprox |
| Decay | Linear | NormalApprox |
| Set | Immediate | Bernoulli |
| Transfer | Linear | NormalApprox |
| Create | (invalid) | Poisson |
| Destroy | Immediate | Bernoulli |
| AddTrait | Immediate | Bernoulli |
| RemoveTrait | Immediate | Bernoulli |

Auto-derived from effect type + probability at rule registration. See `architecture.md` §8.

---

## Adaptive Timestep

| Scale | dt | Layers |
|-------|-----|--------|
| Active (player) | 1–6 ticks (10s–1min) | L0–L4 |
| Settlement (off-screen) | 8,640 (1 day) | L0–L4 batched |
| Region | 60,480 (1 week) | L1–L4 |
| Province | 259,200 (1 month) | L2–L4 |
| Continent+ | 3,153,600+ (1 year+) | L3–L4 |

Same rules. Same evaluator. Different dt. Tier transitions: one catch-up tick at dt=gap. Worldgen: same sim at dt=1 year for N centuries.

---

## Rules as Switches

```
rule.Probability = 0 → rule never fires
                     → every plan with wait on dependent condition: step.prob = 0
                     → plan.confidence = 0
                     → planner never selects it
                     → cascades to compounds, counters, roles
```

Game profiles are rule value tables (see Profiles section in `spec_infrastructure.md`).

---

## Execution Model

### Read-Write Phase Separation

```
Tick(world, dt: int32):
    apply PlayerCommands (sorted by player id)
    for layer in L0..L4:
        for rule in layer:
            if partition.scale outside [rule.MinScale, rule.MaxScale]: skip
            READ  (parallel): evaluate conditions, compute deltas. Pure.
            WRITE: resolve deltas, apply, clear.
```

### Delta Buffer

All mutations via delta buffer. No mid-tick state changes.

```
Delta { Owner, Key, TraitKind, Field, Op (Add/Set), Value, Priority, RuleId }
```

Add: commutative sum. Set: highest priority, RuleId tiebreak. Deterministic regardless of evaluation order.

### Determinism

- Fixed Q16.16 or int everywhere. No floats in sim.
- PRNG: `hash(world_seed, tick, rule_id, subject_id)` per roll.
- Read-write separation per layer. All reads see pre-layer snapshot.

---

## L0 — Physics

```acf
rule heat_spread:
    layer: L0_Physics
    scope: ConnectedTo
    effect: Spread Climate.Temperature conductivity=ConnectedTo.Conductivity

rule solar_heating:
    layer: L0_Physics
    scope: Climate
    condition: Spatial.ScaleLevel == Tile
    effect: Accumulate Climate.Temperature rate=sun_curve(Owner)

rule water_evaporate:
    layer: L0_Physics
    scope: Hydrology
    condition: Hydrology.Water > 0
    effect: Decay Hydrology.Water rate=0.05 floor=0

rule fire_spread:
    layer: L0_Physics
    scope: Adjacent
    condition: Burning.Fire > 0
    condition: contains(Target, Flammable)
    effect: Accumulate Burning.Fire value=20
    # probability from Burning.Fire * Flammable.Ignition

rule fire_consume:
    layer: L0_Physics
    scope: Burning
    condition: Burning.Fire > 0
    effect: Accumulate Burning.Fuel rate=-0.1

rule fire_starve:
    layer: L0_Physics
    scope: Burning
    condition: Burning.Fuel <= 0
    effect: Set Burning.Fire value=0
```

## L1 — Biology

```acf
rule hunger_drain:
    layer: L1_Biology
    scope: Vitals
    effect: Accumulate Vitals.Hunger rate=-0.001

rule starvation:
    layer: L1_Biology
    scope: Vitals
    condition: Vitals.Hunger > 90
    effect: Accumulate Vitals.Health rate=-0.01

rule healing:
    layer: L1_Biology
    scope: Vitals
    condition: Vitals.Health < 100
    condition: Vitals.Hunger < 50
    condition: Vitals.Fatigue < 50
    effect: Accumulate Vitals.Health rate=0.005

rule aging:
    layer: L1_Biology
    scope: Vitals
    effect: Accumulate Vitals.Fatigue rate=0.001

rule death:
    layer: L1_Biology
    scope: Vitals
    condition: Vitals.Health <= 0
    effect: Destroy
    effect: Create corpse_template
```

## L2 — Items

```acf
rule sword_decay:
    layer: L2_Items
    scope: Condition
    effect: Decay Condition.Condition rate=0.0001 floor=0

rule perishable_spoil:
    layer: L2_Items
    scope: Perishable
    effect: Accumulate Condition.Condition rate=-Perishable.Rate

rule item_break:
    layer: L2_Items
    scope: Condition
    condition: Condition.Condition <= 0
    effect: Destroy

rule fuel_burn:
    layer: L2_Items
    scope: LightSource
    condition: LightSource.Brightness > 0
    effect: Accumulate LightSource.FuelRate rate=-1
```

## L3 — Social

```acf
rule familiarity_grow:
    layer: L3_Social
    scope: Social
    condition: distance(Owner, Target) < 10
    effect: Accumulate Social.Fam rate=0.01

rule familiarity_fade:
    layer: L3_Social
    scope: Social
    condition: distance(Owner, Target) > 100
    effect: Decay Social.Fam rate=0.001 floor=0

rule eat_food:
    layer: L3_Basic
    scope: Vitals
    condition: Vitals.Hunger > 50
    condition: contains(Owner, Edible)
    effect: Transfer Edible.Nutrition into=Vitals.Hunger

rule mood_hunger:
    layer: L3_Social
    scope: Vitals
    condition: Vitals.Hunger > 50
    effect: Accumulate Vitals.Mood rate=-0.01

rule knowledge_spread:
    layer: L3_Social
    scope: Social
    condition: distance(Owner, Target) < 10
    # low-obscurity v-items propagate between co-located entities
    # see spec_knowledge.md for propagation mechanics
```

## L4 — Economic

```acf
rule melee_combat:
    layer: L4_Complex
    scope: HostileTo
    condition: distance(Owner, Target) < Weapon.Range
    effect: SkillCheck Attack.Melee vs Defense.ActiveDefense
    effect: Accumulate Vitals.Health value=-Weapon.Damage mitigated=Armor.Protection

rule supply_consume:
    layer: L4_Economic
    scope: Vitals
    effect: Accumulate Edible.Nutrition rate=-0.01
    # groups consume supplies proportional to Weight
```
