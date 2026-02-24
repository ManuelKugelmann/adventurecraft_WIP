# Knowledge System

## Three Tiers

| Tier | What                                 | Cost                          |
|------|--------------------------------------|-------------------------------|
| 1    | Local observation + role scope       | Free, always current          |
| 2    | Skill gates (on-demand checks)       | Free, part of stat block      |
| 3    | Virtual items (remote/hidden/unique) | Runtime cost, possibly stale  |

Tiers 1 and 2 store nothing — they're evaluated when needed. Only tier 3 has runtime cost.

---

## Tier 1: Local Observation + Role Scope

An entity in a region can directly query that region's terrain, structures, connections, and visible occupants from world state. No virtual item needed.

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
    Geology ≥ 1 → sees ore type
    Geology ≥ 3 → sees vein quality and estimated yield
    Perception ≥ 3 → notices concealed weapons

Factual knowledge (when relevant):
    History ≥ 2 → knows about the Old Kingdom's trade routes
    Medicine ≥ 3 → recognizes plague symptoms, knows quarantine protocols

Recipes / rules (attempting an action):
    Metallurgy ≥ 2 → knows bronze alloy ratio
    Metallurgy ≥ 4 → knows crucible steel technique

Artifact identification:
    Arcana ≥ 2 → recognizes enchanted items
    History ≥ 3 → recognizes Old Kingdom craftsmanship
```

---

## Tier 3: Virtual Items

Knowledge about a thing is a **partial, possibly stale, possibly wrong copy** of that thing. No separate knowledge graph. No belief system.

```
VirtualItem {
    mirrors: EntityRef,    // what real thing this represents
    stats: StatBlock,      // what the knower thinks the stats are (partial, maybe wrong)
    freshness: f32,        // age of information
    source: Source,        // observed, taught, inferred, stolen
    obscurity: f32,        // 0=common knowledge, 1=deeply hidden
}
```

### What Gets Mirrored

Any entity type:

- **Person/group**: species, approximate skills, equipment tier, last known location, disposition
- **Item**: type, quality tier, last known location
- **Location**: terrain, resources, connections, structures, occupants (a map = collection of location v-items)
- **Plans**: virtual items mirroring desired future states (see `spec_plans.md`)

### Meta-Knowledge (Recursive)

Knowledge about what others know = virtual copy of them that contains virtual copies in their knowledge slot:

```
My knowledge about Enemy Faction:
    VirtualItem(EnemyFaction) {
        stats: { strength: ~500, location: FORTRESS_NORTH }
        knowledge: [
            VirtualItem(OurSecretTunnel) { ... }   // I think they know about our tunnel
        ]
        plans: [
            VirtualItem(AttackPlan) { target: US }  // I think they plan to attack
        ]
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

```
obscurity_drift = -copy_count × EXPOSURE_RATE
                  -evidence_visibility × EVIDENCE_RATE
                  +containment_effort × SUPPRESSION_RATE
```

---

## Cover Stories

A cover story is a **false virtual item at lower obscurity** competing with the true one:

```
True:  VirtualItem(TUNNEL, destination=ENEMY_FORTRESS, obscurity=0.9)
Cover: VirtualItem(TUNNEL, destination=NEW_MINE, obscurity=0.3)
```

Observers encounter the cover first. They need to overcome 0.9 obscurity to find the truth. If busted (someone follows the tunnel), they gain the true v-item at Observed source, and the cover collapses.

---

## Propagation Mechanisms

### Passive Spread (Social)

Co-located groups exchange low-obscurity virtual items automatically. Rate proportional to group sizes and affinity. Items with obscurity > threshold don't spread.

### Teaching

Deliberate transfer. Teacher copies a virtual item to student at reduced confidence. Rate limited by teacher skill, student capacity, connection throughput.

### Observation

Within observation range (current + adjacent regions), entities query world state directly. Virtual items created only when leaving a region (snapshot) or actively investigating hidden things.

### Espionage

Solo leaf infiltrates target, copies virtual items from target's knowledge set. Only way to acquire items at obscurity > 0.8 without being original observer.

### Research

Produces new virtual items for things not previously known. Creates first virtual copy of an uncopied entity or stat.

### Forgetting

Virtual items with low freshness dropped during coarsening. Important knowledge stays; trivia fades.

---

## Skills Include Rule Knowledge

Simulation rules exist globally. An entity can only *use* a rule if their skill meets the minimum:

```
Rule: Smelt iron          domain: Metallurgy, min_skill: 1
Rule: Temper steel         domain: Metallurgy, min_skill: 3
Rule: Exploit wall weakness domain: Combat, min_skill: 2
```

### Secret Rules

Some rules gated by possessing a specific virtual item rather than skill:

```
Rule: Damascus steel technique
    conditions: [has(IRON_INGOT), has(SPECIAL_FLUX), has(VirtualItem(DAMASCUS_RECIPE))]
    domain: Metallurgy, min_skill: 2
```

The recipe is a v-item that can be copied (taught), stolen (espionage), or discovered (research). Rarity controlled by obscurity.

---

## Cost

| Operation | Cost |
|---|---|
| Base rules + local geography + role knowledge | Zero — queried from world state |
| Skills (including rule knowledge) | Part of stat block, free |
| Virtual items per node | O(0–20 typical) — only remote/hidden/unique |
| Passive propagation | Filtered by obscurity threshold |
| Plan validation | O(assumptions per plan), ~3–10 checks |
| Meta-knowledge depth | Rarely > 2 levels in practice |

Most entities (farmers, laborers, soldiers in garrison) carry zero virtual items — their world is fully covered by observation, role knowledge, and skills. Only scouts, traders, spies, and leaders accumulate significant v-item inventories.
