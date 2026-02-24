# Plans (Proactive Agency)

Plans handle non-routine activity: strategic decisions, novel situations, multi-step goals that cross role boundaries. Plans override role rules temporarily.

---

## Structure

A plan is a virtual item — a node with ImmaterialTrait + PlanMetaTrait:

```
Plan v-item = Node with:
    ImmaterialTrait   { Owner }
    PlanMetaTrait     { Owner, CurrentStep: int, TotalSteps: int, Confidence: Fixed }
```

Plan steps live in templates. Runtime tracks current step index only (AgencyTrait on the executing entity).

```
AgencyTrait { Owner, ActivePlan: TemplateId, ActivePlanStep: int }
```

Plans are knowledge items. They can be shared (taught), stolen (espionage), stale (assumptions diverged), or wrong (based on bad intelligence) — all for free because they're just nodes with traits.

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
            force.Weight > target.garrison * 2
            Skills.Attack > 40
        }
        priority = Drives.Dominance * 0.5

        assemble: do military.assemble_force { destination = $target.region }
        survey:   do Sense.Indirect { target = $target.walls }
        BREACH:   do military.breach_walls { walls = $target.walls }
            prob = sigmoid(Skills.Attack - target.walls.Condition * 0.5)
            fail = STARVE
        storm:    do military.storm { garrison = $target.garrison }
    }

    method starve_out {
        when {
            force.Weight > target.garrison * 1.2
            force.supplies > target.supplies * 1.5
        }
        priority = Drives.Survival * 0.7

        assemble: do military.assemble_force { destination = $target.region }
        STARVE:   do military.blockade { target = $target }
        exhaust:  wait target.supplies <= 0
        demand:   do Influence.Direct { target = $target.leader }
    }

    done { OwnedBy.Target == force.faction }
    fail { force.Weight < 10 }
}
```

---

## Probabilistic Planning

```
step.prob = Expression over traits (never a constant)
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
Cohort(Weight=200):
    Plan A (build wall):    fraction=0.5 → 100 workers equivalent
    Plan B (stockpile food): fraction=0.3 → 60 workers equivalent
    Role rules:              fraction=0.2 → 40 workers on routine work
```

---

## Priority Resolution

```
1. Active plan step (AgencyTrait.ActivePlan, if preconditions met)
2. Highest-priority rule across active role instances (ActiveRole traits)
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

---

## Strategic Templates

### Military

```acf
plan military.raid [military, offensive] {
    scout:    do Sense.Indirect { target = $target_region }
    assemble: do military.assemble_force { destination = $staging_area }
    approach: do Move.Indirect { destination = $target, secrecy = 0.7 }
    strike:   do Attack.Direct { target = $target }
    withdraw: do Move.Direct { destination = $home }
        when Vitals.Health < 70

    done { target.Vitals.Health < 30 }
}
```

### Economic

```acf
plan economic.trade_route [economic, diplomatic] {
    scout:     do Sense.Indirect { target = $destination }
    negotiate: do Influence.Indirect { target = $partner, terms = $deal }
    ship_out:  do Transfer.Structured { goods = $export, destination = $partner }
    receive:   do Transfer.Structured { goods = $import, source = $partner }

    done { contains(Owner, $import) }
    fail { Social.Aff(Owner, $partner) < -20 }
}
```

### Political

```acf
plan political.subversion [political, covert] {
    infiltrate: do Sense.Indirect { target = $rival_faction, secrecy = 0.9 }
    identify:   do Sense.Indirect { target = $discontented_groups }
    recruit:    do Influence.Indirect { target = $sympathizers, secrecy = 0.8 }
    exploit:    wait $rival_faction.unrest > 70
    strike:     do political.coup { target = $rival_faction }

    done { $rival_faction.leader == $puppet }
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
        military.fortify      when Condition.Condition > 30
        military.sortie       when Weight > force.Weight * 0.3
        political.call_allies when AlliedWith exists
    }
}
```

Counter selection is a skill check. A leader with Tactics ≥ 3 considers all counters. Tactics ≥ 1 might only see the obvious one.
