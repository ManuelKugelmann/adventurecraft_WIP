# Plans (Proactive Agency)

Plans handle non-routine activity: strategic decisions, novel situations, multi-step goals that cross role boundaries. Plans override role rules temporarily.

See also: [adventurecraft_HTN_GOAP](https://github.com/ManuelKugelmann/adventurecraft_HTN_GOAP) — the companion dataset repo implementing this spec.

---

## Structure

A plan is a virtual item — a node with ImmaterialTrait + PlanMetaTrait. Plan steps live in templates; runtime tracks only current step index via AgencyTrait on the executing entity.

Plans are knowledge items. They can be shared (taught), stolen (espionage), stale (assumptions diverged), or wrong (based on bad intelligence) — all for free because they're just nodes with traits.

---

## Plan Syntax

Plans use `needs {}` and `outcomes {}`:

- **`needs {}`** — checked against the agent's **worldmodel** (belief state, not ground truth). Eligibility gate and method selection traits.
- **`outcomes {}`** — probabilistic postconditions: goals, side effects, costs, and elapsed time.

```acf
plan military.siege [military, territorial] {

    params { target = EntityRef, force = EntityRef }

    needs {
        self.knows(target.location)
        self.knows(target.garrison)
        Skills.Tactics >= 2
    }

    method assault {
        when { force.Weight > target.garrison * 2 }
        priority = Drives.Dominance * 0.5

        assemble: do military.assemble_force { destination = $target.region }
        survey:   do Sense.Indirect { target = $target.walls }
        BREACH:   do military.breach_walls { walls = $target.walls }
            prob = combat_chance(force, target.walls)
            fail = STARVE
        storm:    do military.storm { garrison = $target.garrison }
    }

    method starve_out {
        when { force.supplies > target.supplies * 1.5 }
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

> `require {}` / `done {} fail {}` remain valid for simple single-outcome plans.

---

## Agent Worldmodel

Agents plan from their **worldmodel** — a filtered view of simulation history plus a sparse override layer. Ground truth is never duplicated; only divergences from what the agent has actually perceived are stored.

Because the simulation is deterministic, the base is a **sparse keyframe replay**: the agent replays only perceived ticks from the last keyframe.

`self.knows(X)` queries the worldmodel — including past actions of other agents the agent has actually witnessed. Mismatches between worldmodel and reality cause runtime plan failures. That is correct behavior, not a bug.

Suspect plans write a specific worldmodel path: `self.worldmodel($subject).active_plan`. Role behaviors read that path to trigger escalation.

---

## Knowledge as Planning Gate

If `self.knows(target.location)` is false, no plan requiring it can be selected. Knowledge acquisition plans run first; they unlock downstream plans by filling the worldmodel.

An army cannot siege what it hasn't scouted. A spy cannot blackmail without intelligence on the target.

---

## Drives as Global Needs

Drives (Survival, Luxury, Dominance, Belonging, Knowledge, Lawful, Moral) constrain all outcome evaluation. The planner weights outcomes by drive intensity and auto-inserts maintenance plans when a plan would violate a drive threshold. A starving soldier will not execute a method with no food acquisition step.

---

## Composition Hierarchy

Three tiers. Templates call compounds; compounds call leaf plans; leaf plans call engine actions.

```
Template    military.siege              — top-level goal
Compounds   assemble_force, blockade,   — domain building blocks (~80–100 total)
            breach_walls, storm
Leaf plans  Move.Structured, Attack.Direct  — one 7×3 step each
```

Composite plans reference only sub-plan names. Leaf plans contain exactly one `do Action.Approach` step. Never mix — a composite with a bare `do` is a parse error.

Decomposition is lazy: a faction decomposes to 4 high-level compound steps; when step 2 becomes active and is assigned to a cohort, that cohort decomposes it further. Unused branches never decompose. Plan granularity matches node granularity.

```
Depth 0 — Template:  SIEGE(fortress)
Depth 1 — Phase:     ASSEMBLE_FORCE, CUT_SUPPLY, BREACH_WALLS, OCCUPY
Depth 2 — Task:      RECRUIT(soldiers), MOVE_TO(staging), BUILD(siege_tower)
Depth 3 — Atomic:    GATHER(wood), CRAFT(siege_components), HAUL(to_site)
```

---

## Resolution Functions

All contested step probabilities use **named resolution functions**, not inline sigmoid expressions:

```acf
BREACH: do military.breach_walls { walls = $target.walls }
    prob = combat_chance(force, target.walls)   # named — readable, testable, replaceable
    # not: prob = sigmoid(Skills.Attack * 0.7 - Condition * 0.5 + 3.2)
```

Standard categories: ENGINE (world truth), RESOLUTION (adversarial contests), KNOWLEDGE (worldmodel queries), DERIVED (computed properties), SUGAR (shorthand predicates). Full catalog in `schema/utility_functions.acf`.

---

## Detection

Detection is a world rule (L3), not a plan step. It resolves in two sequential stages:

1. **Spot** — "is someone there?" Observer skill vs actor Stealth + equipment + lighting + action noise.
2. **Identify** — "who is it?" Separate resolution: familiarity edge + Observer skill vs disguise. An actor can be spotted but not recognised.

Evidence left by physical actions is modelled as ordinary items (with `Decayable` trait) — no special forensics system. Normal world rules decay and destroy them; the `cover_tracks` compound Destroys them explicitly.

---

## Probabilistic Planning

```
step.prob       = resolution_function(actor, context)
plan.confidence = product(step.prob across critical path)
plan.utility    = confidence × goal_value - total_cost
```

Bayesian posteriors from experience override dataset priors. Veterans pick better plans because their estimates are accurate.

---

## Group Plan Execution

Groups divide effort across concurrent plans via fractional allocation:

- Max 4 concurrent plans per group
- Each gets a fraction (min 0.05) of the group's weight
- Fractions sum ≤ 1.0

---

## Priority Resolution

```
1. Active plan step (if worldmodel preconditions met)
2. Highest-priority role rule across active roles
3. Default idle behavior
```

---

## Composable Compounds (~80–100)

Middle-layer building blocks between templates and leaf plans:

```
Information:  recon, investigate, eavesdrop, intercept, surveillance
Deception:    false_identity, plant_evidence, frame_target, cover_tracks, misdirect
Social:       blackmail, bribe, turn_agent, recruit, gossip_campaign
Economic:     smuggle, embargo, price_manipulation, extort, monopolize
Political:    coup, undermine_authority, forge_alliance, install_puppet
Protection:   safehouse, dead_drop, escape_route, alibi
Violence:     ambush, assassination, sabotage, arson, kidnap
Military:     assemble_force, blockade, raid, siege, patrol
Domestic:     secure_supply, establish_trade_route, harvest_season, build_structure
```

Any plot composes from these:
```
Ocean's Eleven  = investigate → recruit → false_identity → misdirect → escape
Game of Thrones = forge_alliance → turn_agent → undermine → frame → coup
```

---

## Counter-Plans

Counters trigger from **observable world state only** — externally visible actions, never internal plans, drives, knowledge, or mood.

```
Plan A acts → observable signature detected → Counter B selected
Counter B acts → new signature → Counter C selected
Max chain depth: 4
```

Counter selection is a Tactics skill check. **Drive-aligned selection**: when multiple counters are viable, each is evaluated against the agent's drive profile — a high-Survival agent rates `fortify` highest; a high-Dominance agent rates `sortie` highest. Same threat, different drives → different responses.

### Three-Tier Counter System

| Tier | Mechanism | When Used |
|------|-----------|-----------|
| **Static counters** | Pre-cached response blocks in plan templates | Default — fast |
| **Suspect plans** | `suspect.*` plan activation reconstructing the threat | When pattern matches multiple possible hostile plans |
| **Adversary simulation** | Shallow sim of adversary role + drives, depth ≤ 2 | High-stakes decisions |

### Suspect Plans

A `suspect.*` plan has exactly one job: **write the worldmodel**. It matches observed action patterns against known criminal or hostile plan signatures and writes `self.worldmodel($subject).active_plan` to the best match.

Escalation (arrest, alert, report, pursue) is the **role's job** — role behaviors trigger on the worldmodel override. This separation means the base guard role handles escalation uniformly regardless of what specialised suspect plan wrote the override.

```
suspect.*  →  writes self.worldmodel($subject).active_plan
role rules →  read worldmodel, trigger escalation
```

Dismissal is free: if no suspect method's `needs` are satisfied, no override is written — no false suspicion.

Guard roles specialise by suspect plan training: vault guards run `suspect.heist`, gate guards run `suspect.smuggling`, caravan guards run `suspect.ambush`. The base guard handles the rest.

### Counter Link Types

- **Resolution links** — outcome determined by engine rules (skill contests, physical events)
- **Behavioral links** — adversary choice based on roles and drives
