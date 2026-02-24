# Contracts, Orders & Authority

## No Explicit Contract System

Expectations are virtual items in knowledge. Actors predict consequences; nothing is binding.

```
Custom:     expectation v-item + social_judgment rule
Law:        expectation v-item + social_judgment rule + authority role + punishment plan
Tradition:  expectation v-item, high familiarity propagation, slow decay
Taboo:      penalty-only expectation, no approval for compliance
Tyranny:    authority punishes without expectation v-items
```

Two generic rules handle all social enforcement:

```acf
rule social_judgment [social, L3] {
    disapprove: when observer.knowledge.contains($expectation)
                AND observer.witnessed($actor, $action)
                AND $action != $expectation.expected_behavior,
                effect: edge(observer, $actor, social).reputation -= $expectation.weight

    approve:    when observer.knowledge.contains($expectation)
                AND observer.witnessed($actor, $action)
                AND $action == $expectation.expected_behavior,
                effect: edge(observer, $actor, social).reputation += $expectation.weight * 0.3
}
```

### Expectation V-Items

```
VirtualItem(Expectation) {
    mirrors: BEHAVIOR_PATTERN,
    stats: {
        expected_behavior: ActionPattern,
        weight: f32,                      // severity of judgment
        context: Condition,               // when this expectation applies
        source: tradition | law | custom, // affects propagation rate
    },
    obscurity: f32,                       // how widely known
}
```

"Culture" is shared expectation v-items + local relationship patterns. Propagates through familiarity. Drifts naturally. No keyword, no special system.

---

## Authority

Authority is always derived, never stored. Four sources:

### Authority Sources

```
Force:       military presence in the region
Delegation:  granted by someone with existing authority (chain)
Consensus:   aggregate approval from the governed population
Tradition:   long-standing familiarity × compliance patterns
```

Each is computed from existing state:

```
authority.force      = holder.military_in(region)
authority.consensus  = avg(edge(holder, population, social).affection)
authority.tradition  = avg(edge(holder, population, social).familiarity) × compliance_history
authority.delegation = product(delegation_chain_strengths)
```

### Effective Authority

```
authority_strength = weighted_sum(force, delegation, consensus, tradition)
```

The weights themselves can vary by region/culture (some cultures weight tradition heavily, others weight consensus).

### Compliance Check

An order is obeyed when the authority strength behind it exceeds the resistance:

```
resistance = target.drives.dominance × (1 - edge(target, authority, social).affection * 0.01)
compliance = authority_strength > resistance
```

Non-compliance triggers enforcement plans from the authority holder.

---

## Orders

Orders are plan steps assigned across entity boundaries.

### Decomposition Cascade

```
King:           plan(CONQUER_PROVINCE) → steps: [raise_army, march, siege]
  ↓ order: raise_army
General:        plan(RAISE_ARMY) → steps: [recruit, equip, train]
  ↓ order: recruit
Captain:        plan(RECRUIT) → steps: [visit_villages, persuade, march_to_camp]
  ↓ order: persuade
Sergeant:       do Influence.Direct { target = $villagers }
```

Each level decomposes one step further. The order carries:
- The action to perform
- The authority chain backing it
- The expected completion conditions
- Consequences for failure

### Order Compliance

An entity receiving an order evaluates:
1. Do I recognize this authority? (delegation chain valid?)
2. Does the authority strength exceed my resistance?
3. Can I actually execute this? (skills, resources, location)
4. Does this conflict with higher-priority orders?

If all pass → the order becomes an active plan step for the recipient.
If authority insufficient → non-compliance → enforcement.
If impossible → failure report up the chain.

---

## Priority Between Sources

When multiple authority sources conflict:

```
1. Direct military threat (force, immediate)
2. Active plan step from recognized superior (delegation)
3. Community expectations (consensus + tradition)
4. Role default behavior
```

A soldier obeys their commander (delegation) over community norms (consensus/tradition) unless the order is extreme enough to trigger moral resistance (drives.moral > threshold).

---

## Regime Types Emerge

Different authority source weightings produce recognizable governance patterns:

```
Heavy force + low consensus     → tyranny / military occupation
High delegation + tradition     → feudalism / aristocracy
High consensus + low force      → democracy / republic
Heavy tradition + low force     → tribal / elder council
Balanced all four               → stable mixed government
```

No governance system is hardcoded. These emerge from the relative strengths of authority sources in a region.

---

## Bundles Encode Governance

Governance patterns are captured as bundles (see `spec_infrastructure.md`):

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
