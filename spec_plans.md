# Plans (Proactive Agency)

Plans handle non-routine activity: strategic decisions, novel situations, multi-step goals that cross role boundaries. Plans override role rules temporarily.

---

## Structure

A plan is a virtual item mirroring a desired future state:

```
VirtualItem(Plan) {
    mirrors: GOAL_STATE,
    stats: {
        template: TemplateId,
        goal: Condition[],
        steps: Step[],
        current_step: usize,
        assumptions: VirtualItem[],
        assigned_to: NodeRef[],
        priority: f32,
        obscurity: f32,
    }
}

Step {
    action: ActionId,
    actor: NodeRef,
    preconditions: Condition[],
    status: Pending | Active | Complete | Failed | Skipped,
    substeps: Step[]?,
}
```

Plans are knowledge items. They can be shared (taught), stolen (espionage), stale (assumptions diverged), or wrong (based on bad intelligence).

---

## Methods (HTN Decomposition)

Plans have multiple methods — alternative approaches to achieve the same goal. The planner selects the method whose preconditions are met and whose expected utility is highest:

```acf
plan military.siege [military, territorial] {

    params {
        target = EntityRef
        force = EntityRef
    }

    require { attack >= 30 }

    method assault {
        when {
            force.weight > target.garrison * 2
            force.skills.attack > 40
        }
        priority = force.drives.dominance * 0.5

        assemble: do military.assemble_force { destination = $target.region }
        survey:   do Sense.Careful { target = $target.walls }
        BREACH:   do military.breach_walls { walls = $target.walls }
            prob = sigmoid(force.skills.attack - target.walls.condition * 0.5)
            fail = STARVE
        storm:    do military.storm { garrison = $target.garrison }
    }

    method starve_out {
        when {
            force.weight > target.garrison * 1.2
            force.supplies > target.supplies * 1.5
        }
        priority = force.drives.survival * 0.7

        assemble: do military.assemble_force { destination = $target.region }
        STARVE:   do military.blockade { target = $target }
        exhaust:  wait target.supplies <= 0
            prob = prob(target.consumption_rate > target.supply_rate)
        demand:   do Influence.Direct { target = $target.leader }
    }

    done { target.owner == force.faction }
    fail { force.weight < 10 }
}
```

---

## Probabilistic Planning

```
step.prob = Expression over actor/world state (never a constant)
plan.confidence = product(step.prob for critical path)
plan.utility = confidence × goal_value - total_cost
```

Bayesian posteriors from experience override dataset priors. Veterans pick better plans.

---

## Decomposition Depth Matches Tree Depth

A faction-level node decomposes to depth 1–2. "Siege the fortress" → "assemble force, cut supply, breach, occupy." Those compound sub-actions are assigned to subordinate groups.

A cohort-level node decomposes to depth 2–3. "Assemble force" → "recruit, train, march."

Atomic actions only appear at leaf level, for promoted solo entities.

```
Depth 0 — Template:    SIEGE(fortress)
Depth 1 — Phase:       ASSEMBLE_FORCE, CUT_SUPPLY, BREACH_WALLS, OCCUPY
Depth 2 — Task:        RECRUIT(soldiers), MOVE_TO(staging), BUILD(siege_tower)
Depth 3 — Atomic:      GATHER(wood), CRAFT(siege_components), HAUL(to_site)
```

Plan granularity = node granularity. No entity holds a plan more detailed than its own depth.

Steps decompose lazily. A faction creates a plan with 4 high-level steps. When step 2 becomes Active and is assigned to a cohort, that cohort decomposes it into sub-steps. Unused branches never decompose.

---

## Group Plan Execution

Groups execute plans through fractional allocation:

- Max 4 concurrent plans per group
- Each gets a fraction (min 0.05) of the group's attention
- Fractions sum ≤ 1.0
- Fractional assignment → proportional progress rate

```
Cohort(200 workers):
    Plan A (build wall):    fraction=0.5 → 100 workers equivalent
    Plan B (stockpile food): fraction=0.3 → 60 workers equivalent
    Role rules:              fraction=0.2 → 40 workers on routine work
```

---

## Priority Resolution

```
1. Active plan step (if assigned and preconditions met)
2. Highest-priority rule across active role instances
3. Default idle behavior
```

When a plan completes or the entity's step is done, they fall back to role behavior automatically.

---

## Atomic Actions

The smallest units of behavior. Everything decomposes to these:

```
Movement:     MOVE_TO, FOLLOW, FLEE, HOLD_POSITION
Resource:     GATHER, CRAFT, CONSUME, STORE, HAUL, TRADE
Social:       COMMUNICATE, TEACH, RECRUIT, INTIMIDATE, NEGOTIATE, DECEIVE
Combat:       ATTACK, DEFEND, AMBUSH, RETREAT, SABOTAGE
Information:  SCOUT, SPY, HIDE, REVEAL, RESEARCH
State:        REST, TRAIN, BUILD, REPAIR, WAIT
```

Every `do` resolves to the 7×3 action table + stealth/awareness modifiers + intensity/secrecy tradeoffs. Atomic actions ARE Action.Approach invocations.

---

## Composable Compounds (~80–100)

Middle-layer building blocks between templates and atomics:

```
Information:  recon, investigate, eavesdrop, intercept, surveillance
Deception:    false_identity, plant_evidence, frame_target, cover_tracks, misdirect
Social:       seduce, blackmail, bribe, turn_agent, recruit, gossip_campaign
Economic:     smuggle, embargo, price_manipulation, counterfeit, extort, monopolize
Political:    coup, undermine_authority, forge_alliance, install_puppet, exile
Protection:   safehouse, dead_drop, escape_route, alibi
Violence:     ambush, assassination, sabotage, arson, poison, kidnap
Military:     assemble_force, establish_outpost, blockade, raid, siege, patrol
Domestic:     secure_supply, establish_trade_route, harvest_season, build_structure
```

Any plot composes from these:

```
Ocean's Eleven  = investigate → recruit → false_identity → smuggle → misdirect → escape
Game of Thrones = forge_alliance → turn_agent → undermine → frame → coup → blackmail
```

### Compound Structure

```
CompoundAction {
    preconditions: Condition[],
    decomposition: Step[],
    completion: Condition[],
    failure: Condition[],
}
```

Example:

```acf
plan military.assemble_force [military] {
    params { target_strength, location }

    method standard {
        when { available_military >= target_strength * 0.5 }

        recruit: do Influence.Direct { target = $recruits }
            when available_military < target_strength
        train:   do Modify.Direct { target = $recruits, skill = attack }
            when avg_skill < threshold
        march:   do Move.Direct { destination = $location }

        done { count(force AT $location) >= target_strength }
        fail { available_military < target_strength * 0.3 }
    }
}
```

---

## Strategic Templates

### Military

```acf
plan military.raid [military, offensive] {
    # goal: weaken target without holding territory
    scout:    do Sense.Indirect { target = $target_region }
    assemble: do military.assemble_force { destination = $staging_area }
    approach: do Move.Careful { destination = $target, secrecy = 0.7 }
    strike:   do Attack.Direct { target = $target }
    withdraw: do Move.Direct { destination = $home }
        when force.casualties > 0.3 OR objective.complete

    done { target.military < target.military_initial * 0.7 }
}

plan military.ambush_defense [military, defensive] {
    # goal: destroy attacking force using terrain
    scout:   do Sense.Careful { target = $approaches }
    deploy:  do Move.Careful { destination = $hidden_positions, secrecy = 0.8 }
    wait:    wait enemy.location == $kill_zone
    strike:  do Attack.Direct { target = $enemy }
    pursue:  do Move.Direct { destination = $enemy.retreat_route }
        when enemy.morale < 30
}
```

### Economic

```acf
plan economic.trade_route [economic, diplomatic] {
    # goal: establish recurring exchange
    scout:     do Sense.Indirect { target = $destination }
    negotiate: do Influence.Careful { target = $partner, terms = $deal }
    ship_out:  do Transfer.Indirect { goods = $export, destination = $partner }
    receive:   do Transfer.Indirect { goods = $import, source = $partner }

    done { self.inventory[$import] > $threshold }
    fail { edge(self, $partner, social).affection < -20 }
}

plan economic.resource_expansion [economic, territorial] {
    scout:   do Sense.Indirect { target = $candidate_regions }
    claim:   do Influence.Direct { target = $region }
    build:   do Modify.Indirect { blueprint = $extraction_structure, location = $region }
    staff:   do Influence.Direct { target = $workers }

    done { $region.production > 0 }
}
```

### Political

```acf
plan political.subversion [political, covert] {
    # goal: weaken rival from within
    infiltrate: do Sense.Indirect { target = $rival_faction, secrecy = 0.9 }
    identify:   do Sense.Careful { target = $discontented_groups }
    recruit:    do Influence.Indirect { target = $sympathizers, secrecy = 0.8 }
    misdirect:  do Influence.Indirect { target = $rival_leader, false = true }
    exploit:    wait $rival_faction.unrest > 70
    strike:     do political.coup { target = $rival_faction }

    done { $rival_faction.leader == $puppet }
}
```

### Survival

```acf
plan survival.famine_response [survival, emergency] {
    ration:   do Transfer.Indirect { policy = rationing }
    forage:   do Transfer.Direct { source = $wilderness }
    trade:    do Transfer.Indirect { goods = $luxury, receive = food }
    migrate:  do Move.Indirect { destination = $fertile_region }
        when food_supply < population * 10

    done { food_supply > population * 60 }
    # escalation: each step triggers only if prior insufficient
}
```

---

## Counter-Plans

Trigger from **observable world state only**. Never reference drives, plans, knowledge, mood, skills, or contracts.

```
Plan A: observable actions → ThreatSignature matches → Counter B selected
Counter B: observable actions → ThreatSignature matches → Counter C selected
Max 4 deep. Beyond that, knowledge confidence too low.
```

```acf
plan military.siege [military] {
    ...
    counter threat.troops_massing {
        military.fortify      when self.walls.condition > 30
        military.sortie       when self.garrison > force.weight * 0.3
        political.call_allies when any(edge(self, *, alliance))
    }
}
```

Counter selection is a skill check. A leader with Tactics ≥ 3 considers all counters. Tactics ≥ 1 might only see the obvious one.

---

## Conditions

Mirror the action hierarchy — can be high-level or atomic:

### Atomic

```
HAS(entity, item, amount)      AT(entity, region)
SKILL_GE(entity, domain, level) ALIVE(entity)
CONNECTED(region_a, region_b)   CONTROLS(faction, region)
```

### Compound

```
STRONGER_THAN(a, b): a.military > b.military × 1.2
SUPPLY_SECURE(f, r): HAS(f, r, ≥ demand×2) OR TRADE_ROUTE_ACTIVE(f, r)
VULNERABLE(t):       t.garrison < t.threat_level OR t.morale < 0.3
```

Compound conditions decompose like compound actions — shorthand evaluated against world state.
