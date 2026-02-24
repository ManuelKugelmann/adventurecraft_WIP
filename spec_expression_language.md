# Conditions, Effects & Rule IR

All world state access through trait fields. All mutations through elementary ops. Rules authored in `.acf`, compiled to structured IR consumed by codegen (Tier 1) and interpreter (Tier 2).

---

## Authoring Syntax (.acf)

### Conditions

Conditions compare trait fields against values or other fields:

```
condition: Vitals.Hunger > 50
condition: contains(Owner, Edible)
condition: distance(Owner, Target) < Weapon.Range
```

All conditions AND within a rule. OR = separate rule.

### Effects

Effects name an elementary op, a target trait field, and parameters:

```
effect: Accumulate Vitals.Hunger rate=-0.001
effect: Decay Condition.Condition rate=0.0001 floor=0
effect: Transfer Edible.Nutrition into=Vitals.Hunger
effect: Set Vitals.Health value=0
effect: Spread Climate.Temperature conductivity=ConnectedTo.Conductivity
```

### SkillCheck (Probabilistic Gate)

SkillCheck is a **condition** (not an effect). Evaluates sigmoid(attacker_skill - defender_skill) as a probability gate for subsequent effects:

```
effect: SkillCheck Attack.Melee vs Defense.ActiveDefense
effect: Accumulate Vitals.Health value=-Weapon.Damage mitigated=Armor.Protection
```

The SkillCheck gates all effects that follow it. Effects are pure mutations; conditions (including SkillCheck) are pure reads.

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

## Built-in Functions

Used in both conditions and value sources:

```
distance(A, B)          container chain walk → tile distance
contains(node, kind)    ContainerIndex → any contained node with trait?
count(node, kind)       ContainerIndex → how many contained with trait
sigmoid(x)              lookup table, probability mapping
depth(node)             container chain length
```

List grows with the game. Both codegen and interpreter implement these.

---

## Compiled Representation (IR)

Parser expands `.acf` to generic IR. See `architecture.md` §8 for full struct definitions.

```
RuleCondition {
    TraitKind, Field, Op (GT/LT/EQ/NEQ/Exists/NotExists/SkillCheck), Value
}

RuleEffect {
    Op (Accumulate/Decay/Set/Transfer/Spread/Create/Destroy/AddTrait/RemoveTrait),
    Target (Owner/Target/Both), TraitKind, Field, Value, Mitigator, Probability
}

ValueSource {
    Kind (Literal/TraitField/NodeField/Function), Literal, TraitKind, Field, Function
}
```

### Two Consumers

- **Tier 1 (codegen)**: .acf → C# Burst jobs at build time. All shipping rules. SkillCheck, Spread, distance, contains are codegen targets.
- **Tier 2 (interpreter)**: Walks IR at runtime. Mods, prototyping, hot-reload. Correct, slower.

Promote workflow: prototype in Tier 2 → verify → build → Tier 1 takes over. No rule rewrite.

---

## Trait Field Access

All state access is through traits. No free-form property paths.

```
Vitals.Health           → VitalsTrait, field index for Health
Weapon.Damage           → WeaponTrait, field index for Damage
Social.Affection        → Social relationship trait, Aff field
ConnectedTo.Throughput  → ConnectedTo relationship trait
```

Fields are all Fixed Q16.16. Uniform layout enables both typed C# access and raw byte+offset for interpreter/GPU.

---

## Relationship-Scoped Rules

Rules scoped to relationship traits (Multi) iterate over all instances. Effects can target Owner, Target, or Both:

```
rule heat_spread:
    layer: L0_Physics
    scope: ConnectedTo
    effect: Spread Climate.Temperature conductivity=ConnectedTo.Conductivity
```

Spread emits two deltas per evaluation (one per side). Conservation enforced by symmetry.

```
rule melee_combat:
    layer: L4_Complex
    scope: HostileTo
    condition: distance(Owner, Target) < Weapon.Range
    effect: SkillCheck Attack.Melee vs Defense.ActiveDefense
    effect: Accumulate Vitals.Health value=-Weapon.Damage mitigated=Armor.Protection
```

The codegen resolves cross-node lookups: iterate HostileTo → get Target NodeId → lookup Target's Vitals → apply delta to Target.

---

## Sugar

Named conditions defined in data files, resolved at load time:

```
HAS(e, item, n)     → contains(e, item) AND count(e, item) >= n
AT(e, loc)           → e.ContainerNode == loc (chain walk)
ALIVE(e)             → Vitals.Health > 0
CONNECTED(a, b)      → ConnectedTo relationship exists
```
