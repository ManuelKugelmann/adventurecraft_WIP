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
Node struct:     ~20 bytes (Id, Template, Weight, ContainerNode, ParentNode, Flags)
VitalsTrait:     ~28 bytes (7 Fixed fields)
AttributesTrait: ~28 bytes (7 Fixed fields)
SkillsTrait:     ~92 bytes (23 Fixed fields)
DrivesTrait:     ~28 bytes (7 Fixed fields)
Social traits:   ~24 bytes × rel_count (typically 5–20 per node)
Inventory:       contained nodes via ContainerIndex
Knowledge:       v-item nodes (typically 0–20, most carry 0)
Template ref:    4 bytes (TemplateId)
```

Compound nodes store only delta from template. Leaf nodes have full trait sets.

### Computational Budget per Tick

```
Per compound node:  ~10 rule evaluations, ~3 plan checks
Per leaf node:      ~20 rule evaluations, ~5 plan checks, tile-level actions
Flow updates:       O(active_flows) ~50 per tick
Knowledge propagation: O(co-located pairs × low-obscurity items) ~100 per tick
```

Target: <2ms per simulation tick on a single core.

---

## Spatial Scale

```
enum SpatialScale : byte {
    Tile, Room, Building, Block, District, Settlement,
    Region, Province, Continent, Planet, System, Sector, Empire
}
```

Tile grid: Tile through Settlement (cells, physics, LOS).
Graph: Settlement and above (topology, aggregate stats).
Coordinates local to containing node. Stellar = travel-time edges.

### Adaptive Timestep by Scale

| Scale | dt | Layers |
|-------|-----|--------|
| Active (player) | 1–6 ticks (10s–1min) | L0–L4 |
| Settlement (off-screen) | 8,640 (1 day) | L0–L4 batched |
| Region | 60,480 (1 week) | L1–L4 |
| Province | 259,200 (1 month) | L2–L4 |
| Continent+ | 3,153,600+ (1 year+) | L3–L4 |

Tier transitions: one catch-up tick at dt=gap.
Worldgen: same sim at dt=1 year for N centuries.

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
- Knowledge decay (how stale is a v-item's StatCopy?)
- Reputation building (long-term Social.Rep patterns)
- Authority tradition (compliance history for AuthStrengthTrait.Tradition)
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
Arithmetic:    Q16.16 Fixed (int32), 0x00010000 = 1.0
               Range: ±32767, Precision: 1/65536
               mul/div via int64 intermediate
               Transcendentals: lookup tables (sigmoid, sqrt, Ziggurat normal)
Collections:   NativeList, NativeHashMap, NativeParallelMultiHashMap
Execution:     Read-write phase separation per layer
Templates:     .acf → IR → Tier 1 codegen Burst/HLSL (shipping) + Tier 2 interpreter (mods)
File format:   .acf (TOML-compatible)
Time:          1 tick = 10s, dt = int32 tick count, clock = int64
```

IDs, tick counters, quantities: plain integers. `ToFloat()` one-way gate for rendering only.

### Read-Write Phase Separation

```
Tick(world, dt: int32):
    apply PlayerCommands (sorted by player id)
    for layer in L0..L4:
        READ  (parallel): evaluate conditions, compute deltas. Pure.
        WRITE: resolve deltas, apply, clear.
```

No mid-tick state mutations. Deterministic regardless of evaluation order.

### Delta Buffer

All mutations via delta buffer:

```
Delta { Owner, Key, TraitKind, Field, Op (Add/Set), Value, Priority, RuleId }
```

Add: commutative sum. Set: highest priority, RuleId tiebreak.

### Determinism

- Fixed or int everywhere. No floats in sim.
- PRNG: `hash(world_seed, tick, rule_id, subject_id)` per roll.
- Mutations via delta buffer only.

### Parallelism

```
Read phase:  Burst IJobParallelFor over trait Data arrays
Write phase: single-threaded
Layers:      sequential L0→L4
Spatial partitions: parallel within layer, sync at boundary
```

### Multiplayer & Distribution

Authoritative server model. Single-player = local server + one client. No code path difference.

```
Clients:   send PlayerCommands, receive filtered deltas
Fog of war: knowledge system filters per player per trait kind
```

Distributed: each server owns a spatial partition. Cross-partition via boundary messages (entity transfer, remote deltas). Delta merge: Add=sum (commutative), Set=priority+RuleId (deterministic). Sync barrier per layer before write phase.

---

## Templates

```
NodeTemplate {
    Id: TemplateId
    Parent: TemplateId?             // template inheritance
    DefaultWeight: int
    DefaultTraits: TraitInstance[]   // stamped onto node on CREATE
}
```

CREATE = allocate node, walk template parent chain, copy default traits.

---

## Validation

```
Enforcement pipeline:
    1. LLM self-check (during extraction)
    2. Post-generate validation script
    3. Git pre-commit hooks
    4. CI pipeline validation
```

See `spec_file_format.md` for full validation rules list.

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
