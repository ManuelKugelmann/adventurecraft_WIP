# Contracts, Roles, Orders & Authority

## Virtual Item Foundation

Contracts, orders, and authority claims are all **virtual items** — they get obscurity, staleness, forgery, theft, copying, and propagation mechanics for free from the existing knowledge system.

---

## Roles (Perpetual Reactive Rulesets)

Roles are prioritized condition→action rules evaluated every tick against world state. They never complete — they run until removed.

```
Role {
    id: RoleId,
    domain: Domain,
    rules: PriorityRule[],              // ordered by urgency
    required_structures: [StructureType],
    required_tools: [ItemType]?,
    shift_pattern: ShiftPattern,
    compatible_with: [RoleId],
}

PriorityRule {
    priority: u8,                       // lower = more urgent
    condition: Condition,               // concrete world state check
    action: ActionId,
    scope: Local | Regional | Global,
}
```

**Role instances** bind a template to a specific workplace, holder, and shift:

```
RoleInstance {
    template: RoleId,
    workplace: RegionRef,
    holder: NodeRef,
    shift: ShiftSchedule,
    local_overrides: PriorityRule[]?,   // leader/player can modify
}
```

An entity holds **multiple role instances** with non-overlapping shifts. Active shifts determine which rules evaluate. Template inheritance allows specialization (`MasterSmith extends Smith`).

---

## Plans (Finite Goal-Directed Sequences)

Plans have completion conditions — they end. They are virtual items mirroring a desired future state.

```
VirtualItem(Plan) {
    mirrors: GOAL_STATE,
    stats: {
        template: TemplateId?,          // strategic template (SIEGE, RAID, etc.)
        goal: Condition[],
        steps: Step[],
        current_step: usize,
        assumptions: VirtualItem[],     // knowledge the plan depends on
        assigned_to: NodeRef[],
        priority: f32,
    },
    obscurity: f32,
}

Step {
    action: ActionId,                   // atomic or compound
    actor: NodeRef,
    preconditions: Condition[],
    status: Pending | Active | Complete | Failed | Skipped,
    substeps: Step[]?,                  // lazy decomposition
}
```

**Decomposition depth matches tree depth.** Faction → phases. Cohort → tasks. Individual → atomics. Unused branches never decompose.

**Plan validity** = product of assumption confidences. Stale assumptions → low confidence → triggers scouting or replanning.

---

## Contracts (Virtual Items)

Formalized mutual obligations between two parties.

```
VirtualItem(Contract) {
    mirrors: agreement,
    owner: EntityRef,                   // typically the authority/employer
    holder: EntityRef,                  // the obligated party
    stats: {
        type: employment | trade | service | quest,
        status: pending | active | completed | breached | cancelled,
        obligations: {
            owner_owes: { items: {type: qty}, actions: {type: {times_per_period, duration_minutes}} },
            holder_owes: { items: {type: qty}, actions: {type: {times_per_period, duration_minutes}} },
        },
        penalties: {
            owner_breach: { items: {}, actions: {} },
            holder_breach: { items: {}, actions: {} },
        },
        execution: {
            frequency: daily | weekly | monthly | one_time,
            duration_days: u32,
            completion_conditions: Condition[],
            renewal_automatic: bool,
        },
    },
    obscurity: f32,                     // secret contracts vs public charters
    freshness: f32,                     // stale = potentially dissolved
}
```

As virtual items, contracts can be:
- **Forged** (DECEIVE plants a false contract)
- **Stolen/copied** (espionage reveals someone's obligations)
- **Stale** (you think the contract holds but it was dissolved)
- **Verified** (SENSE check against authority chain)

---

## Orders (One-Sided Contracts)

An order is a plan step assigned across entity boundaries. It becomes a virtual item in the recipient's inventory.

```
VirtualItem(Order) {
    mirrors: required_action,
    owner: EntityRef,                   // who issued it
    holder: EntityRef,                  // who must execute it
    stats: {
        action: ActionId | Step[],      // what to do
        deadline: GameTime?,
        authority_source: AuthorityClaim,  // what underwrites this order
    },
    obscurity: f32,                     // secret orders vs public decrees
    freshness: f32,
}
```

Orders inherit virtual item mechanics: forgery (false orders from "the king"), verification (check authority chain), staleness (outdated orders from a dead commander).

**Decomposition cascade:**
```
Plan step (internal)
  → assigned to subordinate → becomes Order (virtual item)
  → subordinate decomposes it into sub-steps
  → sub-steps assigned further down → become Orders to their subordinates
```

---

## Authority (Virtual Item)

Authority = credible threat of sanctions. Always derived from concrete sources. Claimable, delegable, regionally constrained.

```
VirtualItem(AuthorityClaim) {
    mirrors: RegionRef,                 // where this applies
    owner: EntityRef,                   // delegator (or self if root claim)
    holder: EntityRef,                  // who wields this authority
    stats: {
        scope: [EntityClass],           // over whom
        grants: [ActionType],           // what actions can be ordered
        sources: [AuthoritySource],     // stacked, each with strength
        delegation_from: AuthorityClaim?,
    },
    obscurity: f32,
    freshness: f32,                     // stale claim = weakened legitimacy
}

AuthoritySource {
    type: Force | Delegation | Consensus | Tradition,
    strength: f32,
}
```

**Source strength is concretely measurable:**

| Source | Derived From |
|---|---|
| Force | Military strength in region (existing stat) |
| Delegation | Delegator's authority strength × fraction |
| Consensus | Population affection/reputation aggregates (existing relationship data) |
| Tradition | Time held × familiarity (existing stats) |

**Multiple sources stack.** Total authority = sum of source strengths. Competing claims in same region resolve by total strength.

**Erosion is natural:** lose army → Force drops. Population turns → Consensus drops. Each source changes independently from existing world state.

---

## Spectrum

```
Plan          → internal, private goal with steps
Order         → plan step assigned to another entity (one-sided)
Contract      → mutual obligations with penalties (two-sided)
AuthorityClaim → what underwrites orders, regionally bound
Role          → perpetual behavior template, may or may not have a contract behind it
```

All are virtual items. Same propagation, obscurity, forgery, staleness mechanics.

---

## Priority Resolution

```
1. Active plan step (if assigned and preconditions met)
2. Highest-priority rule across active role instances
3. Default idle behavior
```

**How contracts feed in:**
- **Employment contracts → role instances.** Contract binds a role to the holder. Breach = failing action counts/durations.
- **Quest/service contracts → plans.** Multi-step obligations generate plan steps.
- **Trade contracts → recurring transfer actions.** Periodic obligations slot into role rules.

---

## Compliance Check

When an entity receives an order:

```
1. Authority valid?
   - Holder has the claimed role?
   - Region matches claim?
   - I'm in scope?
   - Grants include this action?
2. Highest competing claim?
   - Compare total source strengths of all claims in this region
3. Chain recognized?
   - Do I hold virtual items confirming the delegation chain?
4. Drives:
   - lawful → comply with recognized authority
   - dominance/survival → may resist
5. Sanctions credible?
   - Does the authority have actual enforcement power here?
```

Authority without enforcement is just a claim. A guard ordering someone in lawless territory has the claim but no credible sanctions — concretely: faction's Force source strength in this region ≈ 0.
