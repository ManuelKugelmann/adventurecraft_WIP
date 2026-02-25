# CLAUDE.md — AdventureCraft Project Context

This file provides context for AI assistants working on the AdventureCraft specification.

## Repository Purpose

This is the **overarching specification** for the AdventureCraft simulation engine. It contains:
- `architecture.md` — canonical technical reference (nodes, traits, rules, execution)
- `spec_*.md` files — detailed subsystem specifications
- `SPECIFICATION.md` — index and design principles

The companion dataset repository is [adventurecraft_HTN_GOAP](https://github.com/ManuelKugelmann/adventurecraft_HTN_GOAP), which implements the behavior dataset (rules, roles, plans in `.acf` format) with a validation pipeline, schema, and extraction tooling.

---

## Three Core Data Types

All content — world mechanics, agent behavior, and goal-directed action — is expressed in three declaration types in `.acf` files:

1. **Rules** — physics/perception/social mechanics (fire spreads, detection resolves, prices drift). No agency. Evaluated every tick by the rule engine.
2. **Roles** — entity behavior templates with drives and priority-sorted action triggers. Cover ~95% of routine daily activity.
3. **Plans** — hierarchical task decomposition with `needs {}` preconditions and `outcomes {}` postconditions. For non-routine, multi-step goals.

---

## Key Design Principles

**Store only fundamental state. Derive everything else.**
- Authority is never stored — derived from Social traits (Force, Delegation, Consensus, Tradition)
- Reputation is never stored — derived from relationship patterns
- Worldmodel divergences only — agents store what differs from last observation, with confidence metadata

**One node type. Traits are everything.**
- Persons, groups, factions, items, knowledge, contracts, authority claims = all Nodes with Traits
- No class hierarchy, no special cases

**Composition over inheritance.**
- A torch = `LightSource + Flammable + Heavy + Weapon`
- A legal claim = `Immaterial + AuthStrength + AuthScope`

**Emergent complexity from simple rules.**
- No scripted events. Authority, famine, coups — outputs, not inputs.

---

## Plan Authoring Rules

When writing or editing plan files:

1. **`needs {}`** checks run against the agent's **worldmodel** (belief state), not ground truth. Use `self.knows(X)` for knowledge gates.
2. **`outcomes {}`** lists all significant postconditions with probabilities. Costs (time, resources) are outcomes too.
3. **Composite plans** reference only sub-plan names (`do military.assemble_force {}`). **Leaf plans** contain concrete `do Action.Approach` steps. Never mix.
4. **Resolution functions** replace inline sigmoid expressions: `prob = combat_chance(attacker, defender)` not `prob = sigmoid(att * 0.7 - def * 0.5)`.
5. **Counter observables** must reference only externally visible state: `pos`, `weight`, `faction`, `equipped`, `action`, `condition`, `visible`. Never: `drives`, `plans`, `knowledge`, `mood`, `skills`, `contracts`.
6. **Decomposition depth ≤ 6.** Counter chains ≤ 4 deep.
7. **Probabilities bounded 0–1.** Use `sigmoid()`, `prob()`, `min()`, or `max()` — bare numeric expressions get a warning.
8. **Named steps.** Every `do` line needs a step name prefix (`name: do ...`). Bare `do` is a parse error.

---

## The 7×3 Action Table

All behavior ultimately decomposes to one of 21 action-approach pairs + 2 meta-skills:

| # | Action | Approach | Skill | Attributes |
|---|--------|----------|-------|-----------|
| 0 | Move | Direct | Athletics | Str+Agi |
| 1 | Move | Indirect | Riding | Agi+Wit |
| 2 | Move | Structured | Travel | Wit+Will |
| 3 | Modify | Direct | Operate | Str+Wit |
| 4 | Modify | Indirect | Equipment | Agi+Wit |
| 5 | Modify | Structured | Crafting | Wit+Will |
| 6 | Attack | Direct | Melee | Str+Agi |
| 7 | Attack | Indirect | Ranged | Agi+Wit |
| 8 | Attack | Structured | Traps | Wit+Will |
| 9 | Defense | Direct | Active Defense | Agi+Str |
| 10 | Defense | Indirect | Armor | Str+Bod |
| 11 | Defense | Structured | Tactics | Wit+Will |
| 12 | Transfer | Direct | Gathering | Str+Agi |
| 13 | Transfer | Indirect | Trade | Cha+Wit |
| 14 | Transfer | Structured | Administration | Wit+Will |
| 15 | Influence | Direct | Persuasion | Cha+Will |
| 16 | Influence | Indirect | Deception | Cha+Wit |
| 17 | Influence | Structured | Intrigue | Wit+Will |
| 18 | Sense | Direct | Search | Agi+Wit |
| 19 | Sense | Indirect | Observation | Wit+Will |
| 20 | Sense | Structured | Research | Wit+Spi |
| 21 | — | — | Stealth | (modifier) |
| 22 | — | — | Awareness | (modifier) |

---

## Rule Layers

`L0 (Physics) → L1 (Biology) → L2 (Items) → L3 (Social) → L4 (Economic)`

Higher layers depend on lower. Rules within a layer execute in parallel (read phase), then commit (write phase). No circular dependencies.

---

## Validation Checklist

Before submitting any `.acf` content:

- [ ] All `do` statements use valid `Action.Approach` from the 7×3 table
- [ ] All `prob =` expressions use a named resolution function or a bounded expression (sigmoid/prob/min/max)
- [ ] Counter/observe blocks reference only observable state (not drives/plans/mood/skills/knowledge)
- [ ] Decomposition depth ≤ 6; counter chains ≤ 4
- [ ] All `$param` references have matching `params {}` definitions
- [ ] Group allocation fractions sum ≤ 1.0, max 4 concurrent plans, min 0.05 each
- [ ] No references to `authority` or `reputation` as stored fields (they are derived)
- [ ] `needs {}` uses `self.knows()` for any knowledge-dependent check
- [ ] Composite plans do not contain bare `do Action.Approach` steps (delegate to leaf plans)
- [ ] `_provenance {}` block present in every file

---

## Architecture References

- `architecture.md` — full technical spec (nodes, traits, rules IR, execution tiers, scale)
- `spec_plans.md` — plan structure, HTN decomposition, worldmodel, counter-plans
- `spec_rules.md` — rule layers, batch modes, ESTIMATE vs SIMULATE
- `spec_file_format.md` — `.acf` grammar, keywords, validation rules
- `spec_expression_language.md` — condition/effect language, built-in functions
- `spec_knowledge.md` — knowledge tiers, worldmodel, obscurity, misinformation
