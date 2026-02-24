# Infrastructure: Scale, History, Bundles, Tech Stack & Pipeline

---

## Scale Strategy

### Population Budget

~90% of population as groups (80 people = 1 entity). At 10,000 population:

```
~100 compound nodes (groups)
~50 leaf nodes (promoted individuals: leaders, heroes, player pawns)
= ~150 simulation entities regardless of population
```

### Data Budget per Node

```
StatBlock:       ~200 bytes (9 attrs + 23 skills + 7 drives + location + health + mood)
Relationships:   ~40 bytes × rel_count (typically 5–20 per node)
Inventory:       ~20 bytes × bulk_groups (typically 3–10)
Knowledge:       ~60 bytes × v_items (typically 0–20, most carry 0)
Template ref:    8 bytes
Children refs:   8 bytes × child_count (0 for leaves)
```

~2 KB per leaf, ~500 bytes per compound (delta from template).

### Computational Budget per Tick

```
Per compound node:  ~10 rule evaluations, ~3 plan checks
Per leaf node:      ~20 rule evaluations, ~5 plan checks, tile-level actions
Flow updates:       O(active_flows) ~50 per tick
Knowledge propagation: O(co-located pairs × low-obscurity items) ~100 per tick
```

Target: <2ms per simulation tick on a single core.

---

## History (3-Tier Aging)

```
Recent (last 30 days):   full detail, action logs, run-length compressed
Medium (30–365 days):    snapshots + deltas, action logs dropped
Ancient (1+ years):      milestone snapshots only
```

### Action Log Compression

Repeated actions compressed to duration entries:

```
[t=100] MINE (duration: 45 ticks)
[t=145] HAUL_ORE (duration: 12 ticks)
[t=157] REST (duration: 8 ticks)
```

### Milestone Snapshots

Major state changes only: births, deaths, faction changes, major battles, leadership transitions, settlement founding/destruction.

### History Queries

Used for:
- Knowledge decay (how stale is a v-item?)
- Reputation building (long-term relationship patterns)
- Authority tradition (compliance history)
- Player timeline / event log

---

## Bundles

Compatible sets of rules + roles + plans + expectation v-items + rule values. **Not authored — discovered by simulation stability testing.** Local to spatial nodes.

```acf
bundle governance.feudal [governance] {
    expectations {
        obey_lord        { weight = 0.8, context = same_region }
        pay_taxes        { weight = 0.7, context = same_faction }
        military_service { weight = 0.6, context = when_called }
    }
    roles  { lord, vassal, serf, knight, bailiff }
    plans  { political.swear_fealty, political.grant_fief, military.levy_troops }
    rules  { loyalty_decay = 0.05, social_mobility = 0.01 }
    compatible   { economy.agrarian, economy.manor }
    incompatible { governance.democracy }
}
```

### Discovery Process

```
1. Run sim with varied starting conditions
2. Observe which combinations stabilize together
3. Clusters that survive = bundles
4. Rule values that kept them stable = bundle defaults
```

---

## Profiles

Game profiles swap rule values, not behavior:

```acf
profile political_sim {
    fire_spread = 0               # off
    disease_spread = 0            # off
    crop_growth = 1.0             # food reliable
    rumor_spread = 0.5            # fast, intrigue heavy
}

profile survival {
    fire_spread = 0.2             # dangerous
    disease_spread = 0.1          # scary
    crop_growth = 0.05            # food scarce
    rumor_spread = 0.05           # isolated, slow news
}
```

---

## Settlement Templates

Spatial layout + resource requirements + stability criteria. Sim-tested.

```acf
template farming_village [settlement, rural] {
    layout {
        center:  { tags = [well, gathering_point] }
        fields:  { count = pop / 6, tags = [farmable], adjacent = water_source }
        housing: { count = pop / 4, tags = [shelter, enclosed] }
        storage: { count = 1, tags = [cold_storage, enclosed] }
    }
    nearby {
        water_source { distance <= 1 }
        forest       { distance <= 2 }
    }
    population {
        starting = 30
        roles = { farmer: pop * 0.4, gatherer: pop * 0.1, smith: 1 }
    }
    stable_when {
        food_supply > consumption * 1.2
        pop_after_20yr >= starting * 1.3
        survived_bad_harvest >= 1
    }
}
```

---

## LLM-Assisted Balancing

```
Designer intent (natural language)
  → LLM translates to rule constraints + acceptance metrics
  → sim runs headless
  → LLM interprets results vs intent
  → LLM adjusts values with explanation
  → repeat until stable + matches intent
```

Nudge strength controls emergent vs templated worldgen:

```
nudge = 0    freeform, anything goes
nudge = 0.5  attractors, civilizations tend toward coherent bundles
nudge = 1.0  snap to nearest template
```

---

## Technical Stack

```
Engine:        Unity + Burst compiler
Arithmetic:    Q16.16 fixed-point (deterministic)
Collections:   NativeList, NativeParallelMultiHashMap (cache-friendly)
Execution:     read-write phase separation
Templates:     precompiled → bytecode-like evaluation
File format:   .acf (TOML-compatible)
```

### Read-Write Phase Separation

Each simulation tick has two phases:
1. **Read phase**: all nodes read world state, evaluate conditions, select actions
2. **Write phase**: all selected actions applied to world state simultaneously

No mid-tick state mutations. Deterministic regardless of evaluation order.

---

## Validation

```
Enforcement pipeline:
    1. LLM self-check (during extraction)
    2. Post-generate validation script
    3. Git pre-commit hooks
    4. CI pipeline validation
```

Validation rules (see `spec_file_format.md` for full list):
- Expression resolution to terminals
- Valid Action × Approach from 7×3 table
- Decomposition depth ≤ 6
- Counter chains ≤ 4
- No circular plan references
- No references to authority/reputation as stored stats
- Counter observables reference only externally visible state

---

## Extraction Pipeline

### Sources

```
Narrative:   TVTropes, Propp, Polti, ATU folktales, Booker, Campbell
Military:    doctrine manuals, Clausewitz, Sun Tzu
Games:       Dwarf Fortress, RimWorld, CK3, D&D SRD, GURPS
Behavioral:  Maslow, BDI literature, game theory
Historical:  occupation manuals, guild records, legal codes, ethnographic databases
Everyday:    household management books, etiquette manuals, ritual taxonomies
```

### Extraction Priority

```
1. Rules (shape + defaults, implied by all other content)
2. 50–60 universal roles
3. 150 universal everyday plans
4. 80–100 composable compounds
5. Counter-plans and threat signatures
6. Bundles (discovered by sim, not authored)
7. Settlement templates (sim-tested layouts)
```

### GitHub Agent Loop

```
Cron daily     → extract.yml     → LLM extracts batch → validates → PR
PR merged      → counters.yml    → generates counter-plans → PR
Cron weekly    → coverage.yml    → gap report as issue
Manual trigger → targeted extraction from specific source
```

---

## Beyond Games

Same architecture works for any agent-based model. Different bundles, same engine. Historical simulation, supply chain modeling, epidemic modeling, urban planning. Separate project.
