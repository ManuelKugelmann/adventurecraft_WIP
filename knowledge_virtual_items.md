# Template Library — Unified Rule System

## Expression Language

Two terminals read world state. Everything else is math and logic.

```
Expr =
    | STAT(entity, path)                              → value
    | EDGE(from, to, type, [path], [depth], [agg])    → value
    | CONST(literal)                                   → value
    | PARAM($name)                                     → value
    | PROB(Expr)                                        → 0-1
    | RANDOM(min, max)                                  → value
    | Expr {+, -, *, /} Expr
    | min(Expr, Expr) | max(Expr, Expr) | abs(Expr)
    | Expr {<, >, ==, !=} Expr                         → bool
    | Expr {AND, OR} Expr | NOT Expr                   → bool
```

### EDGE Query Modes

```
EDGE(from, to, type)                          // exists? (direct)
EDGE(from, to, type, path, depth)             // transitive up to N hops
EDGE(from, to, type, path, depth, agg)        // aggregate along path
```

| Agg | Use |
|---|---|
| `exists` | any path within depth (default) |
| `min_depth` | shortest hop count |
| `sum(field)` | total along path |
| `min(field)` | bottleneck |
| `max(field)` | peak |
| `product(field)` | decay chains (delegation strength) |

Multi-type paths composed via AND of single-type EDGE checks.

---

## Effects

```
Effect =
    | SET(entity, path, Expr)
    | ADD(entity, path, Expr)
    | REMOVE(entity, path, Expr)
    | CREATE(template_id, params)
    | DESTROY(entity)
    | ADD_EDGE(from, to, type, payload)
    | REMOVE_EDGE(from, to, type)
```

---

## Atomic Actions

Terminal — no further decomposition:

```
Movement:    MOVE_TO, FOLLOW, FLEE, HOLD_POSITION
Resource:    GATHER, CRAFT, CONSUME, STORE, HAUL, TRADE
Social:      COMMUNICATE, TEACH, RECRUIT, INTIMIDATE, NEGOTIATE, DECEIVE
Combat:      ATTACK, DEFEND, AMBUSH, RETREAT, SABOTAGE
Information: SCOUT, SPY, HIDE, REVEAL, RESEARCH
State:       REST, TRAIN, BUILD, REPAIR, WAIT
```

---

## Universal Template Envelope

Every template — role, plan, compound, rule, condition, quest, contract — uses the same outer structure:

```toml
[meta]
id = "unique_id"
category = "role | plan | compound | rule | condition | quest | contract"
extends = ""                          # optional parent template
tags = ["military", "offensive"]

[params]
param_name = { type = "type", default = "value" }

[requirements]
min_skill = { domain = "Domain", level = 0 }
structures = []
tools = []
requires_vitems = []                  # virtual item gates (secret recipes, etc.)
```

---

## Template Categories

### Roles (Perpetual Reactive)

```toml
[[rules]]
priority = 1
condition = "Expr"
action = "action_id or template_id"
scope = "local | regional | global"
```

Evaluated every tick. Never complete. Multiple roles per entity via non-overlapping shifts.

### Plans (Finite Goal-Directed)

```toml
[[steps]]
type = "action | expect"              # actor step or expected world rule
action = "action_id or template_id"   # for action steps
rule = "rule_id"                      # for expect steps
args = { target = "$param" }
preconditions = ["Expr"]
probability = "Expr"                  # skill check odds or rule probability
on_success = 2                        # next step index
on_failure = 3                        # fallback branch (optional)

[completion]
conditions = ["Expr"]

[failure]
conditions = ["Expr"]

[counters]
options = ["template_id", "..."]
```

Decompose lazily. Depth matches tree depth (faction→phases, cohort→tasks, leaf→atomics).

Plan confidence = product of step probabilities. Branching plans sum branch confidences.

### Compound Actions (Reusable Middle Layer)

Same as plans but designed for reuse as sub-steps. Referenced by ID from plan steps.

### World Rules (Tick-Evaluated State Transitions)

```toml
[conditions]
expr = "Expr"

[probability]
base = 0.5
modifiers = "Expr"                    # adjusts base probability

[frequency]
interval = "every_tick | daily | seasonal | on_event(EventType)"

[[effects]]
op = "SET | ADD | REMOVE | CREATE | DESTROY | ADD_EDGE | REMOVE_EDGE"
target = "$entity"
path = "stat.path"
value = "Expr"
```

Actors who know a rule (skill gate or virtual item) can include it as an `Expect` step in plans — predicting world state transitions.

### Compound Conditions (Named Expressions)

```toml
mode = "all | any"                    # AND / OR

[[conditions]]
expr = "Expr"
```

Just saved expressions. Resolved at load time like everything else.

### Contracts (Virtual Items)

```toml
[obligations]
owner_owes = { items = {}, actions = {} }
holder_owes = { items = {}, actions = {} }

[penalties]
owner_breach = { items = {}, actions = {} }
holder_breach = { items = {}, actions = {} }

[execution]
frequency = "daily | weekly | monthly | one_time"
duration_days = 365
completion_conditions = ["Expr"]
renewal_automatic = false
```

---

## File Layout

```
templates/
  roles/        miner.toml, farmer.toml, guard.toml, ...
  plans/        siege.toml, raid.toml, trade_route.toml, ...
  compounds/    assemble_force.toml, secure_supply.toml, ...
  rules/        crop_growth.toml, food_spoilage.toml, rain.toml, ...
  conditions/   vulnerable.toml, supply_secure.toml, ...
  quests/       skill_development.toml, heist.toml, ...
  contracts/    guard_employment.toml, food_supply.toml, ...
mods/
  culture_pack/
    templates/  # same structure, extends or overrides base
```

One file per template. Mods drop files into same structure.

---

## Load-Time Validation

Registry loads all files, resolves `extends` chains, then walks every reference:

- Every `action` in steps/rules resolves to an atomic action or another template ID
- Every condition expression resolves to `STAT`/`EDGE`/`CONST`/`PARAM` terminals
- Every template ID referenced exists in the registry
- No cycles in reference chains
- All `$param` references have matching param definitions

Fail loud at startup. No runtime "template not found."

---

## Planning Integration

Actors use rule knowledge for forward prediction:

```
PlanStep =
    | Action(action_id, args, probability)    // actor does something
    | Expect(rule_id, probability)            // world rule fires

step.probability =
    Action: sigmoid(STAT($actor, skills[domain]) - difficulty)
    Expect: rule.probability.base + rule.probability.modifiers

plan.confidence = product(step.probability per step)
    branching: sum of (branch_probability * branch_confidence)
```

Planner compares competing plans by `confidence × goal_value` — expected utility.

---

## Everything Is The Same

```
Roles:        Expr conditions → actions, repeating
Plans:        Expr conditions → actions, finite, probabilistic
World rules:  Expr conditions → effects, tick-evaluated
Contracts:    Expr conditions → obligations + penalties
Conditions:   named Expr compositions
```

One expression language. One evaluator. One validator. One file format.
