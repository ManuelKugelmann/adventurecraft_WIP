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

Agents plan from their **worldmodel** — a **filtered view of simulation history** plus a sparse override layer. Ground truth is never duplicated; agents store only divergences from what they have actually perceived.

```
Worldmodel {
    base:      filtered sim history (what this agent has actually perceived)
    overrides: { path → (value, confidence, freshness) }
}
```

Because the simulation is deterministic, the base is implemented as **sparse keyframe replay**: an agent replays only ticks they perceived from the last keyframe, avoiding full snapshot storage.

Overrides hold deductions and reports — things the agent inferred or was told that differ from what they directly observed. Suspect plans write overrides; role behaviors read them.

**`self.knows(X)`** queries the worldmodel. Two query forms:

```acf
# Property query — does the agent know this trait value?
self.knows(target.location)
self.knows(properties_of($about))

# History query — does the agent know this action happened?
self.knows(performed($subject, Sense, target = $post))
self.knows(performed($subject, Transfer, concealed = true))
```

History queries filter sim history by what this agent actually perceived — they are not stored facts. An agent who wasn't there cannot perceive a past action.

A specific writeable worldmodel path:
```acf
self.worldmodel($subject).active_plan    # what plan the agent believes $subject is running
```

Mismatches between worldmodel and reality cause runtime plan failures — correct behavior for imperfect-information agents, not a bug.

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

## Composition Hierarchy

Plans are structured in three tiers. Templates call compounds; compounds call leaf plans; leaf plans call atomic engine actions.

```
Template      military.siege               — top-level goal (1 plan node)
Compounds     assemble_force, blockade,    — domain building blocks (~80–100 total)
              breach_walls, storm
Leaf plans    Move.Structured, Attack.Direct  — thin wrapper around one 7×3 step
Atomic        ActionCall                   — engine resolves skill + prob
```

**Composite plans** reference only sub-plan names. **Leaf plans** contain exactly one `do Action.Approach` step. Never mix: a composite plan that contains a bare `do` is a parse error.

Steps decompose lazily. A faction creates a plan with 4 high-level compound steps. When step 2 becomes active and is assigned to a cohort, the cohort decomposes it into sub-steps. Unused branches never decompose.

Plan granularity matches node granularity: no entity holds a plan more detailed than its own depth in the hierarchy.

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
| **RESOLUTION** (adversarial) | `detection_risk`, `combat_chance`, `persuasion_chance`, `deception_chance`, `intimidation_chance`, `lockpick_chance`, `trade_advantage`, `chase_chance`, `observation_chance` |
| **KNOWLEDGE** (belief state) | `self.knows`, `worldmodel`, `route_to`, `location_of`, `reputation_of`, `suspected`, `performed` |
| **DERIVED** (data only) | `accessible`, `affordable`, `pursued`, `hostile`, `allied`, `capable`, `authority_over`, `threat_level`, `travel_time` |
| **SUGAR** (shorthand) | `portable`, `locked`, `burning`, `concealed`, `container_has_space` |
| **PLAN** (feasibility) | `can_reach`, `can_acquire`, `can_learn`, `can_enter` |

### Two-Stage Detection

Detection is a world rule (see `spec_rules.md`), not a plan step. It produces two sequential results:

1. **Spot** — "is someone there?" Resolved by observer's Observation skill vs actor's Stealth + equipment + location lighting + action noise.
2. **Identify** — "who is it?" Resolved separately: familiarity edge + Observation skill vs disguise quality + distance. An actor can be spotted but not recognised.

```acf
# From detection.acf (L3 rule):
spot_by_guard: when observer.role == guard AND co_located(observer, actor.location) AND NOT concealed(actor),
    prob = sigmoid(observer.Skills.observation - actor.Skills.stealth
                   - actor.equipment.stealth_bonus + action.noise * 0.5
                   - location.Lighting.concealment * 0.3),
    effect: visible(actor, observer)

identify_known: when visible(actor, observer) AND edge(observer, actor, familiarity) > 0.3,
    prob = sigmoid(edge(observer, actor, familiarity) * 30 + observer.Skills.observation * 0.5
                   - actor.equipment.disguise_quality - distance(observer, actor) * 0.1),
    effect: observer.knows(identity_of(actor))
```

**Evidence** is created by physical actions as ordinary items — nodes with `Physical + Decayable` traits. They obey normal world rules: rain decays them, fire destroys them, the `cover_tracks` compound plan Destroys them. There is no separate forensics system; investigators find evidence via Sense checks on items.

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

Counter selection is a skill check (Tactics). A leader with Tactics ≥ 3 considers all counters; Tactics ≥ 1 may only see the obvious one.

**Drive-aligned selection**: when multiple counters are viable, the agent's drive profile selects among them:

```acf
counter threat.troops_massing {
    military.fortify      when Condition.Condition > 30   # cautious (high Survival)
    military.sortie       when Weight > force.Weight * 0.3 # aggressive (high Dominance)
    military.feint_flank  when Skills.Tactics >= 3         # cunning (high Wit)
    political.call_allies when AlliedWith exists            # diplomatic (high Belonging)
}
```

The agent runs `plan.utility = confidence × goal_value × drive_weight − cost` for each viable counter and picks the highest. A high-Survival agent rates `fortify` highest; a high-Dominance agent rates `sortie` highest. Same threat, different drives → different responses emerge naturally.

### Three-Tier Counter System

Increasing sophistication, decreasing performance:

| Tier | Mechanism | When Used |
|------|-----------|-----------|
| **Static counters** | Pre-cached adversary response blocks in plan templates | Default — fast, deterministic |
| **Sequential suspect plans** | `suspect.*` plan activation based on observed threat signatures | When threat patterns match multiple possible hostile plans |
| **Full adversary simulation** | Shallow simulation of adversary role + drives, capped at depth 2 | High-stakes decisions only |

**Suspect plans have exactly one job: write the worldmodel.** Running a `suspect.*` plan is the suspicion state; it writes `self.worldmodel($subject).active_plan` to the best-matching reconstructed plan. Escalation — alert, arrest, report, pursue — is not the suspect plan's job. It comes from the agent's role behaviors, which trigger on the worldmodel override.

```
# Separation of concerns:
suspect.*    → writes self.worldmodel($subject).active_plan
role rules   → read self.worldmodel($subject).active_plan, trigger escalation
```

```acf
# Suspect plan — write-only to worldmodel
plan suspect.theft [detection] {
    method pickpocket_pattern {
        needs {
            self.knows(performed($subject, Move, target = $mark, distance < 1))
            AND self.knows(performed($subject, Transfer, source = $mark))
        }
        assess: do Sense.Structured { target = $subject }
        outcomes { self.worldmodel($subject).active_plan = criminal.pickpocket }
    }
}

# Role behavior — reads worldmodel, handles escalation
role market_guard : guard {
    arrest: when self.worldmodel($subject).active_plan matches criminal.*
                AND self.skills.melee > $subject.skills.active_defense,
            do Attack.Indirect { target = $subject, intensity = 0.4 }, priority = 40
}
```

**Dismissal is free**: if no method's `needs` are satisfied, the plan cannot run and no worldmodel override is written. No false suspicion.

Guard roles are **specialised by suspect plan training**: a vault guard runs `suspect.heist`; a gate guard runs `suspect.smuggling`; a caravan guard runs `suspect.ambush`. The base guard role handles escalation uniformly across all of them, because it only reads the worldmodel.

The counter chain forms a bidirectional graph. Adversary response links are of two kinds:
- **Resolution links**: outcome determined by engine rules (skill contests, physical events)
- **Behavioral links**: adversary choices based on their roles and drives (agent decision)
