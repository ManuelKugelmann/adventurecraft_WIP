# Plans (Proactive Agency)

Plans handle non-routine activity: strategic decisions, novel situations, multi-step goals that cross role boundaries. Plans override role rules temporarily.

See also: [adventurecraft_HTN_GOAP](https://github.com/ManuelKugelmann/adventurecraft_HTN_GOAP) — the companion dataset repo implementing this spec.

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

## Plan Syntax: `needs` and `outcomes`

The canonical plan sections are `needs {}` and `outcomes {}`:

- **`needs {}`** — Boolean filters checked against the agent's **worldmodel** (belief state, not ground truth). Serves dual purpose: plan eligibility gate and method selection traits.
- **`outcomes {}`** — Probabilistic postconditions covering goals, side effects, and costs. Time elapsed is another outcome field, not a special field.

```acf
plan military.siege [military, territorial] {

    params {
        target = EntityRef
        force = EntityRef
    }

    needs {
        self.knows(target.location)
        self.knows(target.garrison)
        Skills.Tactics >= 2
    }

    method assault {
        when {
            force.Weight > target.garrison * 2
            Skills.Attack > 40
        }
        priority = Drives.Dominance * 0.5

        assemble: do military.assemble_force { destination = $target.region }
        survey:   do Sense.Indirect { target = $target.walls }
        BREACH:   do military.breach_walls { walls = $target.walls }
            prob = combat_chance(force, target.walls)
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

    outcomes {
        goal:    OwnedBy.Target == force.faction   prob = 0.7
        failure: force.Weight < 10                 prob = 0.15
        cost:    elapsed >= 30                     value = -force.Weight * 0.1
    }
}
```

> **Compatibility note:** `require {}` (single precondition block) and `done/fail {}` (bare postcondition lines) remain valid for simple plans. `needs {}` + `outcomes {}` is the preferred form for complex plans with multiple possible outcomes.

---

## Agent Worldmodel

Agents plan from their **worldmodel** — a filtered copy of world state plus a divergence layer with confidence metadata. Ground truth is never duplicated; only divergences from last observation are stored.

```
Worldmodel {
    base:      snapshot of observed world state
    overrides: {path → (value, confidence, freshness)}
}
```

`self.knows(X)` queries the worldmodel. Mismatches between worldmodel and reality cause runtime plan failures — this is correct behavior reflecting imperfect information, not a bug.

Knowledge gaps are planning gates: if `self.knows(target.location)` is false, any plan requiring it cannot be selected until the agent has observed or been told the target's location.

---

## Knowledge as Planning Gate

Almost every `needs` check implicitly requires knowledge. The planner cannot select a method unless all `self.knows()` checks pass in the worldmodel.

Knowledge acquisition plans run first:

```
acquire_information → fills worldmodel → unlocks downstream plans
```

This creates realistic dependency chains: an army cannot siege what it hasn't scouted; a spy cannot blackmail without intelligence on the target.

---

## Drives as Global Needs

Persistent drives (Survival, Luxury, Dominance, Belonging, Knowledge, Lawful, Moral) constrain all outcome evaluation. The planner:

1. Weights outcomes by drive intensity (`outcome.value *= drive.intensity`)
2. Auto-inserts maintenance plans when a plan would violate a drive threshold

```
if Drives.Survival < 30 and plan.outcomes has no survival maintenance:
    insert acquire_item { target = food } before next step
```

Drive thresholds act as hard stops. A starving soldier will not execute a siege method that has no food acquisition step.

---

## Universal Building Blocks

Eight primitive plans compose into all complex hierarchies. Every compound plan ultimately decomposes to some combination of these:

| Plan | Description |
|------|-------------|
| `acquire_access` | Universal dispatcher — routes to gain_entry, influence_person, or modify_node based on obstacle type |
| `acquire_item` | Obtain a physical object (buy, steal, craft, find) |
| `move_to` | Reach a location (walk, ride, sail, teleport) |
| `acquire_information` | Learn something (observe, ask, research, spy) |
| `gain_entry` | Pass a barrier (door, lock, guard, social gate) |
| `influence_person` | Change an agent's behavior or state (persuade, deceive, threaten, bribe) |
| `modify_node` | Change a node's traits (repair, build, destroy, configure) |
| `protect_node` | Prevent changes to a node (guard, hide, fortify) |

Composite plans reference only sub-plans. Leaf plans contain concrete `do Action.Approach` steps. This two-level structure keeps templates reusable and avoids monolithic plan bodies.

---

## Resolution Functions

All contested outcomes use **named resolution functions**, not inline sigmoid expressions. This makes plan templates readable, testable, and replaceable:

```acf
# Correct — named function, implementable by engine or approximation
BREACH: do military.breach_walls { walls = $target.walls }
    prob = combat_chance(force, target.walls)

# Avoid — opaque inline expression
BREACH: do military.breach_walls { walls = $target.walls }
    prob = sigmoid(Skills.Attack * 0.7 - target.walls.Condition * 0.5 + 3.2)
```

Standard resolution functions (defined in `schema/utility_functions.acf`):

| Category | Functions |
|----------|-----------|
| **ENGINE** (world truth) | `distance`, `reachable`, `visible`, `nearby`, `co_located` |
| **RESOLUTION** (adversarial) | `detection_risk`, `combat_chance`, `persuasion_chance`, `deception_chance`, `intimidation_chance`, `lockpick_chance`, `trade_advantage`, `chase_chance` |
| **KNOWLEDGE** (belief state) | `self.knows`, `worldmodel`, `route_to`, `location_of`, `reputation_of`, `suspected` |
| **DERIVED** (data only) | `accessible`, `affordable`, `pursued`, `hostile`, `allied`, `capable`, `authority_over`, `threat_level`, `travel_time` |
| **SUGAR** (shorthand) | `portable`, `locked`, `burning`, `concealed`, `container_has_space` |
| **PLAN** (feasibility) | `can_reach`, `can_acquire`, `can_learn`, `can_enter` |

---

## Probabilistic Planning

```
step.prob = resolution_function(actor, context)   # never a bare constant
plan.confidence = product(step.prob for critical path)
plan.utility = confidence × goal_value - total_cost
```

Bayesian posteriors from experience override dataset priors. Statistics accumulate per plan template and aggregate by timescale:

| Timescale | Aggregation |
|-----------|-------------|
| Minutes | Individual rolls (full resolution) |
| Days | Step counts (NormalApprox) |
| Weeks | Phase outcomes (summary statistics) |
| Months | Plan-level only |

Veterans pick better plans because their Bayesian priors are accurate.

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

Counters trigger from **observable world state only** — detected `ActionCall` observables (externally visible actions), never internal plans, drives, knowledge, mood, skills, or contracts.

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

### Three-Tier Counter System

Increasing sophistication, decreasing performance:

| Tier | Mechanism | When Used |
|------|-----------|-----------|
| **Static counters** | Pre-cached adversary response blocks in plan templates | Default — fast, deterministic |
| **Sequential suspect plans** | `suspect.*` plan activation based on observed threat signatures | When threat patterns match multiple possible hostile plans |
| **Full adversary simulation** | Shallow simulation of adversary role + drives, capped at depth 2 | High-stakes decisions only |

**Suspect plans**: Running a `suspect.*` plan *is* the suspicion state. Guards trigger them when observable threat signatures match. Plan recognition works through plan execution, not through a separate inference mechanism.

```acf
role guard.city_watch {
    suspect_smuggler: when detected_hidden_cargo(nearby) and night_time,
        do suspect.smuggling { target = $suspicious_entity }, priority = 80

    suspect_scout:    when observed_systematic_movement(nearby, pattern=recon),
        do suspect.hostile_reconnaissance { target = $movement_source }, priority = 70
}
```

The counter chain forms a bidirectional graph. Adversary response links are of two kinds:
- **Resolution links**: outcome determined by engine rules (skill contests, physical events)
- **Behavioral links**: adversary choices based on their roles and drives (agent decision)
