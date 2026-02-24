# Narrative & Trajectory Readout

The simulation produces ground-truth history. This spec defines how that history is **read out** — as structured event logs, LLM-readable trajectory summaries, and narrative text. No separate storytelling engine. The narrative is a view over simulation state.

---

## Design Principle

The simulation already structures behavior into three layers (rules → roles → plans). These map directly to narrative granularity:

| Sim Layer | Narrative Unit | Example |
|-----------|---------------|---------|
| Rule firing | Atomic fact | "The granary caught fire" |
| Role activation | Character beat | "The guard raised the alarm" |
| Plan step | Scene beat | "The army breached the outer wall" |
| Plan completion | Episode | "The siege of Harren ended in surrender" |
| State transition (authority shift, faction split, equilibrium break) | Chapter break | "The northern provinces declared independence" |

**No pattern matching required.** Every rule firing, role activation, and plan step is already a structured event with known subject, action, object, instrument, location, and tick. The narrative system reads these; it does not infer them.

---

## Event Tuples

Every action the simulation executes emits an event tuple:

```
EventTuple {
    Tick:       int64
    Subject:    NodeId          // who did it
    Action:     ActionId        // what (role rule or plan step template)
    Object:     NodeId?         // acted upon (target node, if any)
    Instrument: NodeId?         // using what (tool, weapon, resource)
    Location:   NodeId          // where (spatial container)
    Layer:      byte            // L0–L4 / Role / Plan
    PlanCtx:    TemplateId?     // which plan this step belongs to (if any)
    RoleCtx:    TemplateId?     // which role fired (if any)
    Result:     EventResult     // success / fail / partial
}
```

Event tuples are the atoms. They are emitted as a side effect of the existing rule/role/plan execution — no additional simulation logic.

### Compression

Event tuples follow the same 3-tier aging as history (see `spec_infrastructure.md`):

| Age | Storage |
|-----|---------|
| Recent (≤30 days) | Full tuples, run-length compressed for repeated actions |
| Medium (30–365 days) | Plan-level summaries + state-change tuples only |
| Ancient (1+ years) | Milestone tuples only (births, deaths, battles, regime changes, founding/destruction) |

---

## Narrative Queries

A narrative is a **filtered, ordered projection** of event tuples around a focal node.

### Focal Modes

| Focus | Filter | Produces |
|-------|--------|----------|
| Person | `Subject == node OR Object == node` | Biography |
| Location | `Location == node` | Chronicle |
| Faction | `Subject.Faction == node OR Object.Faction == node` | Political history |
| Item | `Object == node OR Instrument == node` | Provenance chain |
| Relationship | `Subject == A AND Object == B` (or vice versa) | Relationship arc |

### Proximity Expansion

A focal query expands to include events from **nearby nodes** — spatially (same region, adjacent regions) and socially (same faction, direct relationships). This captures context: "While the siege continued, the merchant guild in the neighboring town cut off supply routes."

Expansion radius is a query parameter. Tighter = personal story. Wider = geopolitical overview.

---

## Salience

Not every event tuple matters for a readable narrative. The raw log for a farming village over one year is thousands of tuples. Most are routine.

### Automatic Salience Markers

Events are tagged with salience at emit time based on cheap heuristics:

| Marker | Trigger |
|--------|---------|
| `STATE_CHANGE` | Authority source changed, faction membership changed, node created/destroyed, trait added/removed |
| `PLAN_BOUNDARY` | Plan started, plan completed, plan failed, plan abandoned |
| `THRESHOLD_CROSS` | Any trait field crossed a rule threshold (hunger critical, loyalty flipped, etc.) |
| `PROMOTION` | Node promoted from group to individual (story relevance) |
| `SPLIT_MERGE` | Group split or merged |
| `FIRST_OCCURRENCE` | First time this subject has performed this action type |
| `CONFLICT` | Competing plans targeting same node, compliance check failed, combat |

Routine role activity (guard patrols, farmer harvests, merchant trades) gets no marker unless it crosses a threshold or is a first occurrence.

### Multi-Scale Relevance

The multi-scale system already solves the "who matters" problem:

- **Promoted individuals** (out of groups) = protagonists. They were promoted because they deviated from the average. Their actions are inherently non-routine.
- **Group splits** = factions diverging. Narrative conflict.
- **Group merges** = alliances forming. Narrative resolution.
- **Weight changes** = populations shifting. Setting-level change.

At higher spatial scales (region, province, continent), individual event tuples are replaced by aggregate summaries: "The eastern provinces saw three faction splits and a 20% population decline over the decade."

---

## LLM-Readable Trajectory Format

For autotuning and narrative generation, the event log must be serialized into a format an LLM can consume. This is a structured natural-language summary, not raw data.

### Trajectory Summary Structure

```
World Trajectory: [start_tick] to [end_tick]
Scale: [spatial scale]
Focus: [focal node, if any]

## Setting
[Spatial layout, resource distribution, population counts, active bundles/governance type]

## Period: [tick_range] — [derived label]

### Key Events
- [Tick] [Subject] [action verb] [object] [at location]. [consequence if state_change].
- ...

### State at End of Period
- Population: [counts by faction/settlement]
- Authority: [who controls what, by what source]
- Resources: [critical shortages or surpluses]
- Active Conflicts: [competing plans, failed compliance]
- Knowledge: [significant information asymmetries]

## Period: [next tick_range] — [derived label]
...
```

Period boundaries align with chapter-level salience markers (equilibrium disruptions, authority transitions, major plan completions).

### What Gets Included

Only events with salience markers. Routine activity is summarized as aggregate counts ("42 harvest cycles completed without incident"). The LLM sees the shape of the trajectory, not the tick-by-tick grind.

### Dual Consumer

The same trajectory format serves two consumers:

1. **Autotuner** — "Does this trajectory match the target description?" The LLM compares the summary against a natural-language specification (historical account, desired game-world properties) and scores the fit.
2. **Narrator** — "Turn this trajectory into readable prose." The LLM renders the structured summary into chronicles, biographies, or event logs appropriate to the focal mode.

One serialization format. Two readout modes.

---

## Narrative Rendering

The LLM's role in narrative generation is **prose rendering and editorial selection**, not story discovery. The simulation provides:

- **What happened** — event tuples (ground truth)
- **Who mattered** — promoted nodes, salience markers
- **Structure** — plan boundaries = scenes, state transitions = chapter breaks

The LLM provides:

- **Prose** — turning `(Baker, flee, market, —, East_Gate, tick_4820)` into "Old Marten abandoned his stall and ran for the east gate."
- **Editorial judgment** — selecting which salient events to foreground vs. background for a readable narrative at the requested length
- **Connective tissue** — bridging between episodes, providing temporal context ("Three weeks had passed since the garrison withdrew.")
- **Tone/register** — chronicle style, personal diary, omniscient narrator, factual report — same data, different voice

### No Hallucination Risk

The LLM never invents events. Every claim in the narrative maps to one or more event tuples. The trajectory summary is the **sole source of truth**. The LLM may omit events for brevity but may not fabricate them. This is enforced by providing only the trajectory summary as context — the LLM has nothing to hallucinate from.

---

## Integration with Existing Systems

### History Aging (spec_infrastructure.md)

The 3-tier history aging maps to narrative depth:

- **Recent** → fine-grained narrative ("minute by minute, the wall crumbled")
- **Medium** → episode-level summaries ("over the summer, three raids depleted the granary")
- **Ancient** → milestone chronicles ("in the third year of the drought, the kingdom split")

### Knowledge System (spec_knowledge.md)

A character's **subjective narrative** is filtered by their knowledge tier:

- **Tier 1** (local observation) → what they personally witnessed
- **Tier 2** (skill gates) → what they could interpret ("the blacksmith recognized the blade's origin")
- **Tier 3** (virtual items) → what they've been told, possibly wrong

An unreliable narrator is a narrative query filtered through a specific node's knowledge state. Misinformation in v-items produces misinformation in subjective narratives — for free.

### Plans (spec_plans.md)

Plan structure maps directly to narrative arc:

- **Plan initiation** → inciting incident
- **Method selection** → choice / turning point
- **Step execution** → rising action
- **Branching (fail → fallback method)** → complication / reversal
- **Plan completion / abandonment** → resolution

Competing plans targeting the same node produce natural narrative conflict without any storytelling logic.

### Multi-Scale (spec_infrastructure.md)

| Scale | Narrative Style |
|-------|----------------|
| Tile / individual | Personal narrative, moment-to-moment |
| Settlement | Town chronicle, seasonal rhythm |
| Region | Political history, campaign accounts |
| Province+ | Civilizational arcs, epoch summaries |

The adaptive timestep already determines granularity. The narrative readout inherits it.

---

## Non-Goals

- **No story grammar engine.** No Propp functions, no act structure enforcement, no plot templates. The simulation produces the structure; the LLM renders it.
- **No dialogue generation during simulation.** Dialogue is a rendering concern, generated at readout time from the event tuples and knowledge state.
- **No narrative-driven simulation.** The simulation never changes behavior to make a better story. Determinism and ground truth are inviolable. If the story is boring, the parameters are wrong — fix them via autotuning, not via narrative override.
