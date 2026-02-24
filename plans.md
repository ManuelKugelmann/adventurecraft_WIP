# Knowledge, Meta-Knowledge, and Plans

## Why Knowledge Is Different From Other Stats

Health, mood, position â€” these are properties of the entity that owns them. Knowledge is different in three ways:

1. **Knowledge is shareable without loss.** Teaching someone doesn't reduce your own knowledge. This breaks the conservation invariant that other stats obey during split/merge.

2. **Knowledge has truth value.** An entity can "know" something that's wrong. The simulation needs to track what entities believe, not just what's true. Beliefs propagate, mutate, and conflict.

3. **Knowledge is structured.** "Iron smelting requires flux" is not a scalar â€” it's a relationship between concepts. Plans are even more structured: ordered sequences of actions with preconditions and goals.

These properties mean knowledge can't just be another float in the stat block. It needs its own representation that still plugs into the tree hierarchy.

---

## Knowledge as a Shared, Layered Graph

Knowledge lives in a separate structure â€” a **knowledge graph** â€” that tree nodes *reference into* rather than own directly.

```
KnowledgeGraph {
    facts: Map<FactId, Fact>
    // Every fact in the world, whether anyone knows it or not
}

Fact {
    id: FactId
    subject: EntityRef       // what the fact is about
    predicate: Predicate     // what kind of fact
    value: Value             // the content
    truth: bool              // what's actually true (ground truth)
    visibility: Visibility   // observable, hidden, secret
}
```

Examples of facts:
- (MINE_3, contains, IRON_ORE) â€” observable, true
- (FACTION_NORTH, plans, INVASION) â€” secret, true
- (HERB_7, cures, PLAGUE) â€” hidden, true (not yet discovered)
- (GOLD, transmutes_from, LEAD) â€” doesn't exist in fact graph (false belief has no ground truth)

Entities don't contain knowledge. They contain **references to beliefs about facts**, which may or may not correspond to actual facts.

---

## Belief: What an Entity Thinks It Knows

Each tree node has a belief set â€” a sparse overlay on the knowledge graph:

```
stats.beliefs: BeliefSet {
    known_facts: Map<FactId, Belief>
    // Only facts this entity has encountered â€” sparse, not exhaustive
}

Belief {
    fact_id: FactId
    believed_value: Value     // what the entity thinks is true (may differ from fact.value)
    confidence: f32           // 0.0 = rumor, 1.0 = witnessed firsthand
    freshness: f32            // decays over time â€” old knowledge becomes unreliable
    source: BeliefSource      // where the belief came from
}

enum BeliefSource {
    Observed,        // entity directly witnessed it
    Taught,          // received from another entity/group
    Inferred,        // derived from other beliefs
    Rumor,           // received through social propagation, low confidence
    Inherited,       // came with the group when it formed
}
```

### Beliefs in the Hierarchy

Beliefs follow the same grouped/individual pattern as everything else:

**Compound node (weight=200 miners):** BeliefSet represents what the group collectively knows. "These miners know where the iron veins are, know basic smelting, have heard rumors about gold deposits to the north." Individual variation is captured by confidence distributions, not by per-miner belief sets.

**Leaf node (weight=1, promoted scholar):** Full individual BeliefSet. Can hold unique beliefs no one else has. Can have contradictory beliefs that create internal conflict.

When a compound refines, children inherit the parent's beliefs with added noise (confidence jitter, potential for individual deviations). When children coarsen, beliefs merge: shared beliefs strengthen (confidence increases), contradictory beliefs resolve toward majority or highest-confidence.

---

## Knowledge Propagation

Knowledge flows between nodes through the same connection/flow infrastructure as physical movement:

### Direct Transfer (Teaching, Orders)

An entity with belief B at high confidence transfers it to another entity. The receiver gains B at reduced confidence (teaching is lossy).

```
Transfer:
    source has Belief(SMELTING_RECIPE, confidence=0.95)
    target receives Belief(SMELTING_RECIPE, confidence=0.95 Ã— teaching_efficiency)
    source retains original belief unchanged   â† non-conserving, unlike other stats
```

At group level: a "teacher" node transfers a belief to a "student" group. The group's collective knowledge updates. Rate limited by group size (teaching 200 people takes longer than teaching 5) and by connection throughput (teacher must be in the same or adjacent region).

### Social Propagation (Rumors, Culture)

Beliefs spread passively between co-located groups at a rate proportional to:
- Group sizes (more people = faster spread)
- Proximity (same region > adjacent > distant)
- Social compatibility (same faction > allied > neutral > hostile)
- Belief salience (dramatic facts spread faster than mundane ones)

```
Propagation per tick:
    for each pair of co-located groups (A, B):
        for each belief in A.beliefs not in B.beliefs:
            spread_chance = A.weight Ã— B.weight Ã— proximity Ã— salience
            if roll(spread_chance):
                B gains belief at confidence = A.confidence Ã— RUMOR_DECAY
                mark source = Rumor
```

At group level, this is O(group_pairs Ã— belief_count), not O(individual_pairs). A city of 10,000 in 50 cohorts has ~1,250 pair interactions, not 50,000,000.

### Observation (Discovery)

Entities gain beliefs by being present when something happens or by inspecting something:

```
Miners in Region(MINE_3) discover iron vein:
    Fact(MINE_3, contains, IRON_ORE) exists in knowledge graph
    Miner group gains Belief(that fact, confidence=1.0, source=Observed)
```

Observation range: a group "sees" facts associated with its current region and adjacent regions. Hidden facts require specific actions to reveal (prospecting, scouting, research). Secret facts require espionage or betrayal.

---

## Meta-Knowledge: Knowledge About Knowledge

Meta-knowledge is beliefs about other entities' belief sets. It's the same structure, one level up.

```
MetaBelief {
    about: NodeRef              // which entity/group
    belief_id: FactId           // which fact
    believed_state: MetaState   // what we think they know
    confidence: f32
}

enum MetaState {
    TheyKnow,          // we believe they know this fact
    TheyDontKnow,      // we believe they're ignorant of this
    TheyBelieveWrong,   // we believe they hold a false version
    Unknown,            // we don't know whether they know
}
```

Examples:
- "We know the enemy doesn't know about our secret tunnel" â†’ MetaBelief(ENEMY, SECRET_TUNNEL, TheyDontKnow, confidence=0.8)
- "We think the traders know where the gold deposits are" â†’ MetaBelief(TRADERS, GOLD_LOCATION, TheyKnow, confidence=0.6)
- "We suspect the spy has learned our battle plans" â†’ MetaBelief(SPY, BATTLE_PLAN, TheyKnow, confidence=0.4)

### Why Meta-Knowledge Matters

Meta-knowledge drives strategic behavior:

**Trade:** "They don't know we have surplus iron" â†’ we can charge more. "They know we're desperate for grain" â†’ they'll charge us more. Trade prices become a function of mutual knowledge about supply/demand, not just ground truth.

**Diplomacy:** "They don't know we know about their planned invasion" â†’ we can prepare without tipping them off. "They know we're weak" â†’ they're more likely to attack.

**Espionage:** The entire value of spies is manipulating meta-knowledge â€” either gaining knowledge of enemy beliefs or planting false beliefs.

**Teaching/Research:** "We know that neighboring faction knows advanced smelting but we don't" â†’ motivation to trade for knowledge, send scholars, or steal it.

### Meta-Knowledge in the Hierarchy

Meta-knowledge is expensive â€” tracking what every group thinks every other group knows is O(groupsÂ²Ã— facts). The system controls this the same way it controls everything: by granularity.

**High-level nodes** (factions): track meta-beliefs about other factions' knowledge of strategic facts only. "Do they know our military strength? Our alliance? Our weakness?" Maybe 10-20 meta-beliefs per faction pair.

**Mid-level nodes** (cohorts): track meta-beliefs about immediate neighbors and counterparts. "Do the miners know the bridge is broken?" Relevant for coordination. Very sparse.

**Leaf nodes** (individuals): only promoted leaders, diplomats, and spies carry individual meta-knowledge. Everyone else inherits their group's.

Meta-knowledge also decays. Beliefs about what others knew last month become unreliable. This naturally limits the meta-knowledge set size.

---

## Plans: Future-Directed Knowledge Structures

A plan is a belief about a sequence of future actions that will achieve a goal. It lives in the same belief system but has additional structure:

```
Plan {
    goal: GoalSpec               // desired world state
    steps: Step[]                // ordered actions
    preconditions: Belief[]      // what must be true for the plan to work
    assumptions: Belief[]        // what we believe is true but haven't verified
    contingencies: (Condition, Plan)[]  // fallbacks
    owner: NodeRef               // who holds this plan
    shared_with: NodeRef[]       // who else knows about it
    confidence: f32              // how likely we think this plan succeeds
}

Step {
    action: ActionType
    actor: NodeRef               // who performs this (can be a group)
    target: EntityRef
    prerequisites: StepIndex[]   // which steps must complete first
    estimated_duration: f32
    status: Pending | Active | Complete | Failed
}

GoalSpec {
    facts: (FactId, Value)[]     // world state we want to be true
    priority: f32
    deadline: GameTime?
}
```

### Plans at Different Tree Depths

Plans scale with the node that holds them:

**Leaf (individual):** "Walk to the workshop, pick up iron, forge a sword, deliver it to the armory." Concrete steps, specific tiles, specific items. This is a traditional AI action sequence.

**Compound (cohort of 200 miners):** "Extract 500 iron ore from MINE_3 this season, transport to SMELTER_2." Steps are aggregate flows, not individual actions. Duration estimated from group production rate and flow throughput. Individual miners don't have plans â€” the cohort plan decomposes into their daily work patterns automatically.

**High-level (faction):** "Secure grain supply before winter by conquering PROVINCE_SOUTH or establishing trade with FACTION_EAST." Steps are strategic operations. Sub-plans generated for subordinate nodes. This is grand strategy.

Each level's plan decomposes naturally into sub-plans at the next level down, but only when that level refines. A faction plan to "raise an army" doesn't immediately generate 3,000 individual marching orders. It generates a cohort-level plan to "recruit, equip, and muster." The cohort plan generates flow-level movements. Individual orders only materialize if a leaf node refines during execution.

### Plan as Belief

A plan's validity depends on its assumptions â€” which are beliefs that may be wrong:

```
Plan: Attack FORTRESS_NORTH from the west
    Assumption: Belief(WEST_WALL, structural_integrity, LOW, confidence=0.7)
    Assumption: Belief(ENEMY_GARRISON, strength, 200, confidence=0.5)
    
    If either assumption is wrong, the plan fails.
    Confidence of plan â‰ˆ product of assumption confidences = 0.35
```

**Scouting** is the process of converting plan assumptions from low-confidence beliefs to high-confidence ones. A scout refines to solo, observes the target, gains Observed beliefs, reports back, and the plan's confidence updates.

**Counter-intelligence** is about detecting enemy plans by observing their preparations (troop movements, resource stockpiling) and inferring their goals. This is meta-knowledge about plans: "We believe the enemy plans to attack from the west because we observed siege equipment moving westward."

### Plan Failure and Replanning

When a step fails or an assumption is invalidated:

1. **Local recovery:** if the plan has a contingency for this failure, switch to it. No replanning needed.

2. **Step replanning:** regenerate the failed step and its dependents. The rest of the plan survives.

3. **Full replan:** goal is still valid but the approach is broken. Discard the plan, generate a new one from current beliefs. This is expensive and happens at most once per node per significant event.

4. **Goal abandonment:** the goal itself is no longer achievable or relevant. Drop the plan entirely.

At group level, replanning is cheap because plans have few steps (strategic operations, not individual actions). A faction replans "conquer the south" into "negotiate for grain instead" in one evaluation. The expensive per-entity replanning never happens because entities don't have individual plans unless promoted to solo.

---

## Knowledge Economy

Knowledge has production, storage, and transfer costs, creating an economy:

**Research** produces new facts. A laboratory group generates Belief entries for previously unknown facts at a rate proportional to researcher count Ã— skill. The knowledge graph contains hidden facts waiting to be discovered; research is the process of a group gaining beliefs about them.

**Libraries** store beliefs persistently. Without a library, beliefs decay (freshness drops). A library region arrests decay for all beliefs held by groups in that zone â€” it's a "freshness refresh" applied at region level.

**Schools** transfer beliefs between groups at amplified rate. A school region with a high-skill teacher group and a student group produces belief transfers at higher throughput than passive social propagation.

**Espionage** transfers beliefs from hostile groups. A spy (solo leaf) infiltrates an enemy faction node, copies beliefs from it (with confidence penalty), and carries them back. The meta-knowledge cost: the target might gain MetaBelief(us, their_secrets, TheyKnow) if the spy is detected.

**Censorship/propaganda** is the deliberate injection of false beliefs or suppression of true ones within your own faction. Modeled as the faction leader node overwriting or blocking belief propagation for specific facts.

All of this operates on the same flow infrastructure. Knowledge transfer rate between regions is limited by connection throughput (a messenger must physically travel) and by teaching capacity (one teacher can only teach so many per tick). Knowledge is a resource that flows through the city's infrastructure just like grain or iron â€” it just doesn't get consumed.

---

## Integration With the Unified Tree

Knowledge adds three things to the node stat block:

```
stats.beliefs: BeliefSet           // what this node knows (sparse)
stats.meta_beliefs: MetaBeliefSet  // what this node thinks others know (very sparse)  
stats.plans: Plan[]                // active plans (0-3 per node typically)
```

These follow all the same rules:
- Compound nodes hold aggregate beliefs (collective knowledge)
- Refinement inherits beliefs to children with noise
- Coarsening merges beliefs (majority wins, confidence combines)
- Shells don't render beliefs directly â€” but a "confused" or "eureka" visual indicator can derive from belief change events

The key cost control: beliefs are sparse. A group doesn't have beliefs about every fact in the world â€” only facts it has encountered. Meta-beliefs are even sparser. Plans are capped at a small number per node. The total knowledge cost scales with O(nodes Ã— avg_beliefs_per_node), not with O(total_facts Ã— total_nodes).

---

## Summary

| Concept | Representation | Scales how |
|---|---|---|
| Fact | Entry in global knowledge graph | O(world complexity), fixed |
| Belief | Sparse reference from node to fact, with confidence | O(nodes Ã— encounters) |
| Meta-belief | Belief about another node's belief | O(faction_pairs Ã— strategic_facts) |
| Plan | Goal + steps + assumptions (which are beliefs) | O(nodes Ã— 1-3 plans each) |
| Teaching | Belief transfer between co-located nodes | Flow on region graph |
| Rumor spread | Passive belief propagation | O(co-located group pairs) |
| Research | Converting hidden facts to beliefs | Production rate of researcher groups |
| Espionage | Copying beliefs across hostile boundaries | Solo leaf operation |
| Plan validity | Product of assumption confidences | Updates when beliefs update |
| Scouting | Upgrading assumption confidence via observation | Solo leaf operation â†’ belief update â†’ plan confidence update |
