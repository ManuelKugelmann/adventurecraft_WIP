# Core Stats

## Attributes (9 — 3×3 Matrix)

|              | Power        | Coordination  | Endurance        |
|--------------|-------------|---------------|------------------|
| **Physical** | Strength    | Agility       | Body (HP)        |
| **Mental**   | Willpower   | Intelligence  | Spirit (mana)    |
| **Social**   | Authority*  | Charisma      | Reputation*      |

\* Authority & Reputation derived from relationship aggregates. Never stored.

---

## Skills (7 Actions × 3 Approaches + 2 Meta = 23)

Universal action signature: `ACTION(target, method, [objects], intensity, secrecy)`

| Action    | Direct          | Careful      | Indirect        |
|-----------|----------------|-------------|-----------------|
| Move      | Athletics      | Riding      | Travel          |
| Modify    | Operate        | Equipment   | Crafting        |
| Attack    | Melee          | Ranged      | Traps           |
| Defense   | Active Defense | Armor       | Tactics         |
| Transfer  | Gathering      | Trade       | Administration  |
| Influence | Persuasion     | Deception   | Intrigue        |
| Sense     | Search         | Observation | Research        |

### Attribute Pairings per Skill

Each skill uses 2 attribute scores for resolution:

```
Action     Direct           Careful          Indirect
────────   ──────           ───────          ────────
Move       str+agi          agi+int          int+wil
Modify     str+int          agi+int          int+wil
Attack     str+agi          agi+int          int+wil
Defense    agi+str          str+bod          int+wil
Transfer   str+agi          cha+int          int+wil
Influence  cha+wil          cha+int          int+wil
Sense      agi+int          int+wil          int+spi
```

### Meta-Skills (2)

- **Stealth**: reduces detection across all actions
- **Awareness**: improves detection across all actions

### Trade-offs

- **intensity↔stealth**: high intensity penalizes stealth capability
- **stealth↔effectiveness**: high stealth reduces action effectiveness
- **awareness↔focus**: high awareness reduces task focus

---

## Drives (7)

```
survival:    0–100   safety, basic needs
luxury:      0–100   comfort, fine goods
dominance:   0–100   control, authority
belonging:   0–100   relationships, community
knowledge:   0–100   learning, discovery
lawful:      0–100   0=chaotic, 50=neutral, 100=lawful
moral:       0–100   0=evil, 50=neutral, 100=good
```

Character flaws emerge from drive imbalances. No explicit flaw system. Example: high survival + low moral = cowardice.

---

## Relationships (4-axis, any→any, asymmetric)

```
debt:        -100..+100   obligation balance
reputation:  -100..+100   fear↔respect
affection:   -100..+100   hate↔love
familiarity: 0–100        interaction history
```

### Scope

Any entity → any entity:
- Individual ↔ Individual
- Individual ↔ Group
- Group ↔ Group
- Group ↔ Self (tracks internal coherence)

### Familiarity Weighting

High familiarity → personal relationship dominates inherited group relationship. When an individual has low familiarity with another entity, they fall back to their group's relationship.

### Authority & Reputation Derivation

Both are **always derived**, never stored:

- **Authority** = credible threat of sanctions, derived from Force + Delegation + Consensus + Tradition sources (see `spec_contracts_authority.md`)
- **Reputation** = aggregate of reputation axes across all relationships pointing at the entity
