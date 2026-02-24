# Core Stats

## Attributes (7 Stored + 2 Derived)

7 stored attributes. Authority and Reputation always derived from relationships, never stored.

```
Str    Physical Power        strength, carrying, melee force
Agi    Physical Coordination agility, reflexes, balance
Bod    Physical Endurance    constitution, health pool, resistance
Will   Mental Power          willpower, resolve, mental force
Wit    Mental Coordination   intelligence, cunning, speed of thought
Spi    Mental Endurance      spirit, mana pool, mental resilience
Cha    Social Coordination   charisma, presence, social influence
```

**Derived** (see `spec_contracts_authority.md`):
- **Authority** (Social Power): credible threat of sanctions, from Force + Delegation + Consensus + Tradition
- **Reputation** (Social Endurance): aggregate of reputation axes across relationships

Implementation: `AttributesTrait { Owner, Str, Agi, Bod, Will, Wit, Spi, Cha }` — all Fixed Q16.16. See `architecture.md` §5.

---

## Skills (7 Actions × 3 Approaches + 2 Meta = 23)

Universal action signature: `ACTION(target, method, [objects], intensity, secrecy)`

| Action    | Direct          | Indirect     | Structured      |
|-----------|----------------|-------------|-----------------|
| Move      | Athletics      | Riding      | Travel          |
| Modify    | Operate        | Equipment   | Crafting        |
| Attack    | Melee          | Ranged      | Traps           |
| Defense   | Active Defense | Armor       | Tactics         |
| Transfer  | Gathering      | Trade       | Administration  |
| Influence | Persuasion     | Deception   | Intrigue        |
| Sense     | Search         | Observation | Research        |

### Attribute Pairings per Skill

Skill bonus = skill + attr1 + attr2. Pairs are data-driven.

```
Idx  Action     Approach     Name              Attr1+Attr2
0    Move       Direct       Athletics         Str+Agi
1    Move       Indirect     Riding            Agi+Wit
2    Move       Structured   Travel            Wit+Will
3    Modify     Direct       Operate           Str+Wit
4    Modify     Indirect     Equipment         Agi+Wit
5    Modify     Structured   Crafting          Wit+Will
6    Attack     Direct       Melee             Str+Agi
7    Attack     Indirect     Ranged            Agi+Wit
8    Attack     Structured   Traps             Wit+Will
9    Defense    Direct       Active Defense    Agi+Str
10   Defense    Indirect     Armor             Str+Bod
11   Defense    Structured   Tactics           Wit+Will
12   Transfer   Direct       Gathering         Str+Agi
13   Transfer   Indirect     Trade             Cha+Wit
14   Transfer   Structured   Administration    Wit+Will
15   Influence  Direct       Persuasion        Cha+Will
16   Influence  Indirect     Deception         Cha+Wit
17   Influence  Structured   Intrigue          Wit+Will
18   Sense      Direct       Search            Agi+Wit
19   Sense      Indirect     Observation       Wit+Will
20   Sense      Structured   Research          Wit+Spi
21   —          —            Stealth           (modifier)
22   —          —            Awareness         (modifier)
```

Implementation: `SkillsTrait { Owner, Skills[23]: Fixed }`. See `architecture.md` §7.

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
Survival:    safety, basic needs
Luxury:      comfort, fine goods
Dominance:   control, authority
Belonging:   relationships, community
Knowledge:   learning, discovery
Lawful:      chaos↔order spectrum
Moral:       evil↔good spectrum
```

Character flaws emerge from drive imbalances. No explicit flaw system. Example: high Survival + low Moral = cowardice.

Implementation: `DrivesTrait { Owner, Survival, Luxury, Dominance, Belonging, Knowledge, Lawful, Moral }` — all Fixed. See `architecture.md` §5.

---

## Relationships (4-axis, any→any, asymmetric)

```
Debt:        obligation balance
Reputation:  fear↔respect
Affection:   hate↔love
Familiarity: interaction history
```

Implementation: Multi relationship trait `Social { Owner, Target, Debt, Rep, Aff, Fam }`. See `architecture.md` §5.

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

Implementation: AuthStrengthTrait for authority claims. Social trait's Rep field for reputation. See `architecture.md` §5.
