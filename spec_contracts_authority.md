# Contracts, Orders & Authority

## Contracts as Virtual Items

Contracts are nodes with ImmaterialTrait + ContractTermsTrait. No separate contract system — contracts get forgery, staleness, theft, and propagation for free through the virtual item machinery.

```
ContractTermsTrait  { Owner, ContractType: int, Status: int, Duration: int, Frequency: int }
```

A contract node lives in the ContainerNode of its holder(s). Party relationships link the contract to involved entities.

### Contract Types

All encoded through ContractTermsTrait fields + expectation v-items:

```
Employment:  ContractTerms + EmployedBy relationship + work expectation v-items
Trade deal:  ContractTerms + obligation expectation v-items
Alliance:    ContractTerms + AlliedWith relationship + mutual defense expectations
Lease:       ContractTerms + OwnedBy (landlord) + ContainerNode occupancy
```

### Enforcement Through Expectations

Expectations are separate virtual item nodes (ImmaterialTrait + relevant traits) that define expected behavior. Two generic rules handle all social enforcement:

```acf
rule social_judgment:
    layer: L3_Social
    scope: Social
    # observers who hold expectation v-items judge actors
    # disapproval: Accumulate Social.Rep rate=-weight
    # approval:    Accumulate Social.Rep rate=+weight*0.3
```

### What Emerges

```
Custom:     expectation v-item + social_judgment rule
Law:        expectation v-item + authority role + punishment plan
Tradition:  expectation v-item, high familiarity propagation, slow decay
Taboo:      penalty-only expectation, no approval for compliance
Tyranny:    authority punishes without expectation v-items
Culture:    shared expectation v-items + local relationship patterns (no keyword, no system)
```

---

## Authority

Authority is always derived, never stored. Authority **claims** are virtual items (nodes with ImmaterialTrait + AuthStrengthTrait + AuthScope):

```
AuthStrengthTrait   { Owner, Force, Consensus, Tradition, Delegation }  // all Fixed
AuthScope           { Owner, Target }  // Multi: what the authority covers
DelegatedFrom       { Owner, Target }  // Multi: chain of delegation
```

### Authority Sources

```
Force:       military presence in the region
Delegation:  granted by someone with existing authority (chain via DelegatedFrom)
Consensus:   aggregate approval from the governed population
Tradition:   long-standing familiarity × compliance patterns
```

Each computed from existing traits:

```
Force      = count(military nodes in region) × average Attributes.Str
Consensus  = avg(Social.Aff where Target = holder, across governed population)
Tradition  = avg(Social.Fam where Target = holder) × compliance_history
Delegation = product(AuthStrength along DelegatedFrom chain)
```

### Effective Authority

```
authority_strength = weighted_sum(Force, Delegation, Consensus, Tradition)
```

Weights vary by region/culture (some cultures weight tradition heavily, others weight consensus). Weights are expectation v-items themselves — meta-norms about how authority works.

### Compliance Check

An order is obeyed when authority strength exceeds resistance:

```
resistance = target.Drives.Dominance × (1 - Social.Aff(target→authority) * 0.01)
compliance = authority_strength > resistance
```

Non-compliance triggers enforcement plans from the authority holder.

---

## Orders

Orders are virtual items (ImmaterialTrait) assigned across entity boundaries. An order is a plan step that the issuer delegates to a subordinate.

### Decomposition Cascade

```
King:           plan(CONQUER_PROVINCE) → steps: [raise_army, march, siege]
  ↓ order v-item: raise_army
General:        plan(RAISE_ARMY) → steps: [recruit, equip, train]
  ↓ order v-item: recruit
Captain:        plan(RECRUIT) → steps: [visit_villages, persuade, march_to_camp]
  ↓ order v-item: persuade
Sergeant:       do Influence.Direct { target = $villagers }
```

Each level decomposes one step further. The order v-item carries:
- The action to perform
- The authority chain backing it (DelegatedFrom relationships)
- The expected completion conditions
- Consequences for failure

### Order Compliance

An entity receiving an order evaluates:
1. Do I recognize this authority? (DelegatedFrom chain valid?)
2. Does the authority strength exceed my resistance?
3. Can I actually execute this? (skills, resources, location)
4. Does this conflict with higher-priority orders?

If all pass → the order becomes an active plan step (AgencyTrait updated).
If authority insufficient → non-compliance → enforcement.
If impossible → failure report up the chain.

---

## Priority Between Sources

When multiple authority sources conflict:

```
1. Direct military threat (Force, immediate)
2. Active plan step from recognized superior (Delegation)
3. Community expectations (Consensus + Tradition)
4. Role default behavior
```

A soldier obeys their commander (Delegation) over community norms (Consensus/Tradition) unless the order is extreme enough to trigger moral resistance (Drives.Moral > threshold).

---

## Regime Types Emerge

Different authority source weightings produce recognizable governance patterns:

```
Heavy Force + low Consensus      → tyranny / military occupation
High Delegation + Tradition      → feudalism / aristocracy
High Consensus + low Force       → democracy / republic
Heavy Tradition + low Force      → tribal / elder council
Balanced all four                → stable mixed government
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
