# Knowledge System

## Three Tiers

| Tier | What                                 | Cost                          |
|------|--------------------------------------|-------------------------------|
| 1    | Local observation + role scope       | Free, always current          |
| 2    | Skill gates (on-demand checks)       | Free, part of SkillsTrait     |
| 3    | Virtual items (remote/hidden/unique) | Runtime cost, possibly stale  |

Tiers 1 and 2 store nothing — they're evaluated when needed. Only tier 3 has runtime cost.

---

## Tier 1: Local Observation + Role Scope

An entity in a region can directly query that region's traits from world state. No virtual item needed.

Beyond raw observation, a group's **role implies local knowledge**. Miners at Mine 3 know vein quality, capacity, and hazards — not because they carry virtual items, but because their role + location is a direct query key into world state.

```
Query: what does this group know about?
    1. World state within observation range        (everyone)
    2. Detailed world state for their role+region  (role-specific depth)
    3. Virtual items for everything else            (remote, hidden, unique)
```

This means:
- No "map" virtual items for regions you're standing in or adjacent to
- Miners don't carry virtual items about their own mine
- Guards don't carry virtual items about their own fortifications

---

## Tier 2: Skill Gates

Skills represent knowledge of the rules they unlock. Metallurgy:3 = "knows how to temper steel." The skill *is* the domain knowledge.

Skill-based knowledge is **checked on demand, never stored**:

```
Observation (local detail):
    Search ≥ 1 → sees ore type
    Search ≥ 3 → sees vein quality and estimated yield
    Observation ≥ 3 → notices concealed weapons

Factual knowledge (when relevant):
    Research ≥ 2 → knows about the Old Kingdom's trade routes
    Research ≥ 3 → recognizes plague symptoms, knows quarantine protocols

Recipes / rules (attempting an action):
    Crafting ≥ 2 → knows bronze alloy ratio
    Crafting ≥ 4 → knows crucible steel technique
```

---

## Tier 3: Virtual Items

Knowledge about a thing is a **node with ImmaterialTrait** — a partial, possibly stale, possibly wrong representation using the same trait system as real entities.

### Virtual Item as Node

```
Knowledge v-item = Node with:
    ImmaterialTrait  { Owner }
    Mirrors          { Owner, Target: NodeId }      // what real thing this represents
    StatCopy         { Owner, FieldPath, Value }     // Multi: partial stat snapshot
    ObscurityTrait   { Owner, Value }                // 0=public, 1=secret
```

The v-item node lives in its knower's ContainerNode. It mirrors a real entity via the Mirrors relationship trait. StatCopy entries hold what the knower thinks the stats are (partial, possibly wrong).

### What Gets Mirrored

Any entity type:

- **Person/group**: approximate attributes, equipment tier, last known location, disposition
- **Item**: type, quality tier, last known location
- **Location**: terrain, resources, connections, structures, occupants (a map = collection of location v-items)
- **Plans**: ImmaterialTrait + PlanMetaTrait (see `spec_plans.md`)
- **Contracts**: ImmaterialTrait + ContractTermsTrait (see `spec_contracts_authority.md`)
- **Authority claims**: ImmaterialTrait + AuthStrengthTrait + AuthScope

See `architecture.md` §12 for full virtual item table.

### Meta-Knowledge (Recursive)

Knowledge about what others know = a v-item that Mirrors another v-item:

```
My knowledge about Enemy Faction:
    v-item(EnemyFaction) {
        Mirrors → real EnemyFaction node
        StatCopy: { strength: ~500, location: FORTRESS_NORTH }
        contains:
            v-item(OurSecretTunnel) { ... }   // I think they know about our tunnel
            v-item(AttackPlan) { ... }         // I think they plan to attack
    }
```

Meta-knowledge depth rarely exceeds 2 levels — exponential cost self-limits.

---

## Obscurity Controls Propagation

```
≈0.0: freely propagates (public facts)
≈0.3: spreads within community
≈0.6: restricted (military, trade secrets)
≈0.9: deep secret (espionage only)
≈1.0: unknown unknown (discovery only)
```

Propagation chance modified by obscurity:

```
propagation_chance = base_social_spread × (1.0 - obscurity) × proximity × faction_affinity
```

### Obscurity Changes

**Decreases** (secret becomes more known):
- More entities acquire copies
- Physical evidence becomes visible
- Holders become careless over time
- A holder is captured or defects

**Increases** (harder to discover):
- Holders eliminated (fewer copies)
- Evidence destroyed
- Counter-intelligence succeeds
- Cover story established

---

## Cover Stories

A cover story is a **false virtual item at lower obscurity** competing with the true one:

```
True:  v-item(TUNNEL, StatCopy: destination=ENEMY_FORTRESS, Obscurity=0.9)
Cover: v-item(TUNNEL, StatCopy: destination=NEW_MINE, Obscurity=0.3)
```

Observers encounter the cover first. They need to overcome 0.9 obscurity to find the truth. If busted, they gain the true v-item at Observed source, and the cover collapses.

V-items can be forged, stolen, or become stale — all for free because they're just nodes with traits.

---

## Propagation Mechanisms

### Passive Spread (Social)

Co-located groups exchange low-obscurity v-items automatically. Rate proportional to Social.Fam and proximity. Items with ObscurityTrait.Value > threshold don't spread.

### Teaching

Deliberate transfer. Teacher copies a v-item to student with reduced StatCopy confidence. Rate limited by teacher skill, student capacity, connection throughput.

### Observation

Within observation range (current + adjacent regions), entities query world state directly. V-items created only when leaving a region (snapshot) or actively investigating hidden things.

### Espionage

Solo leaf infiltrates target, copies v-items from target's ContainerNode. Only way to acquire items at Obscurity > 0.8 without being original observer.

### Research

Produces new v-items for things not previously known. Creates first virtual copy of an uncopied entity or stat.

### Forgetting

V-items with low freshness (old StatCopy) dropped during coarsening. Important knowledge stays; trivia fades.

---

## Skills Include Rule Knowledge

Simulation rules exist globally. An entity can only *use* a rule if their skill meets the minimum:

```
Rule: smelt_iron              domain: Crafting, min_skill: 1
Rule: temper_steel            domain: Crafting, min_skill: 3
Rule: exploit_wall_weakness   domain: Tactics, min_skill: 2
```

### Secret Rules

Some rules gated by possessing a specific v-item rather than skill:

```acf
rule damascus_steel:
    layer: L2_Items
    scope: Condition
    condition: contains(Owner, Material) AND Material.MaterialType == iron
    condition: contains(Owner, Immaterial) AND Mirrors.Target == damascus_recipe
    # requires v-item(DAMASCUS_RECIPE) in knower's possession
```

The recipe is a v-item that can be copied (taught), stolen (espionage), or discovered (research). Rarity controlled by ObscurityTrait.Value.

---

## Cost

| Operation | Cost |
|---|---|
| Base rules + local geography + role knowledge | Zero — queried from world state |
| Skills (including rule knowledge) | Part of SkillsTrait, free |
| Virtual items per node | O(0–20 typical) — only remote/hidden/unique |
| Passive propagation | Filtered by obscurity threshold |
| Plan validation | O(assumptions per plan), ~3–10 checks |
| Meta-knowledge depth | Rarely > 2 levels in practice |

Most entities (farmers, laborers, soldiers in garrison) carry zero virtual items — their world is fully covered by observation, role knowledge, and skills. Only scouts, traders, spies, and leaders accumulate significant v-item inventories.
