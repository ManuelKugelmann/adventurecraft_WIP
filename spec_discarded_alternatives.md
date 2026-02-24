# Discarded Alternatives

Design decisions that were explored and rejected, with reasons.

---

## 1. Separate Knowledge Graph / Belief System

**What it was**: A dedicated knowledge graph where entities held "beliefs" as nodes in a separate graph structure, with confidence values, source tracking, and inference chains. Knowledge was modeled as a layered graph: generic knowledge → specialized → detailed, with lazy evaluation generating details on demand.

**Source files**: `plans.md`, `knowledge_and_plans.md`

**Why discarded**: Introduced a parallel data structure alongside the entity graph. Required its own query language, propagation rules, and consistency checking. The belief/inference machinery added complexity without proportional gameplay benefit.

**Replaced by**: Virtual items — partial, possibly stale copies of real entities stored in the knower's inventory using the same StatBlock structure as real entities. No separate graph. Knowledge about a thing IS a copy of that thing. Meta-knowledge (knowing what others know) is just recursion depth in the v-item tree. See `spec_knowledge.md`.

---

## 2. Simplified Stat Model (3 Attributes, 7 Skills, 4 Drives)

**What it was**: An early draft with only 3 primary attributes (Physical, Mental, Social), 7 skills (one per action), and 4 drives (survival, dominance, belonging, knowledge).

**Source files**: `SUMMARY.md`, `Draft__simulation_stats_and_groups`

**Why discarded**: Too coarse. The 3-attribute model couldn't distinguish between strength and agility or between willpower and intelligence. One skill per action provided no strategic choice — there was only one way to do anything. Missing drives (luxury, lawful, moral) prevented generating meaningful personality variation and moral dilemmas.

**Replaced by**: 9 attributes (3×3 matrix: Physical/Mental/Social × Power/Coordination/Endurance), 23 skills (7 actions × 3 approaches + 2 meta), 7 drives. The 3-approach structure per action is the key design — it creates meaningful tactical choices (direct/careful/indirect) without excessive granularity. See `spec_core_stats.md`.

---

## 3. Explicit Contract System

**What it was**: A dedicated Contract data type with structured fields for parties, obligations, durations, breach conditions, and enforcement mechanisms. Contracts were first-class objects tracked by the simulation with explicit state machines (active/breached/fulfilled/expired).

Example contract types included: employment, trade agreements, service contracts, quest contracts, mercenary contracts, alliances. Each had specific templated fields.

**Source files**: `Summary`, `Contracts`, `Contracts__authority__plans` (partial)

**Why discarded**: Contracts are just expectations + consequences. A formal Contract struct duplicated information already representable as virtual items (expectations about behavior), social rules (judgment on compliance), and authority mechanics (enforcement). The dedicated machinery added implementation complexity without enabling behavior that the simpler system couldn't produce.

**Replaced by**: Expectation v-items + social_judgment rules + authority roles. "Law" = expectation v-item + authority role + punishment plan. "Tradition" = high-familiarity expectation with slow decay. "Custom" = expectation + social judgment. No dedicated contract type needed. See `spec_contracts_authority.md`.

---

## 4. Authority as Stored Formula

**What it was**: Authority computed as a stored derived stat: `Authority = (debt + abs(reputation) + affection) / 3`. A simple formula applied to the 4-axis relationship values.

**Source files**: `Draft__simulation_stats_and_groups`, `SUMMARY.md`

**Why discarded**: A single formula couldn't capture the distinct sources of authority (military coercion vs popular support vs inherited legitimacy vs delegated power). It treated all authority as equivalent regardless of source. A king who lost his army but kept tradition would have the same authority as a warlord who conquered but had no legitimacy — which is wrong.

**Replaced by**: Four-source authority model: Force (military presence), Delegation (chain from existing authority), Consensus (aggregate approval), Tradition (long-standing familiarity × compliance). Each source is computed from existing state. Different weightings produce different governance types (feudalism, democracy, tyranny, tribal council). See `spec_contracts_authority.md`.

---

## 5. Hierarchical Topic Knowledge

**What it was**: Knowledge modeled as hierarchical topic entries: `generic: {"metalworking": 85}, specialized: {weaponsmithing: 90}, detailed: {specific_recipe: {...}}`. Each entity had a knowledge tree organized by topic, with confidence and source per entry.

**Source files**: `knowledge_and_plans.md`, `Object_system` (knowledge tracking section)

**Why discarded**: This was a midpoint between the separate knowledge graph (alternative #1) and the final virtual items approach. Topic hierarchies still required a custom data structure. The generic/specialized/detailed split mapped poorly to the actual information needs of the simulation (which are about specific entities, not abstract topics).

**Replaced by**: The three-tier model: (1) local observation is free, (2) skills gate rule knowledge on demand, (3) virtual items for remote/hidden/unique knowledge. Most entities carry zero v-items. See `spec_knowledge.md`.

---

## 6. "Jobs" as Distinct from Roles

**What it was**: "Jobs" were separate from roles — a job was a template instantiated with a workplace binding, while roles were broader behavioral patterns. Jobs had shift patterns, required structures, required tools, and compatibility lists. A "JobInstance" bound a template to a specific workplace and holder.

**Source files**: `Contracts__authority__plans`

**Why discarded**: The distinction between "job" and "role" was artificial. A role IS a job when bound to a workplace. The separate JobTemplate and JobInstance types duplicated the role structure. The compatibility and shift mechanics work just as well inside the role system.

**Replaced by**: Roles with optional workplace binding and shift patterns. A role gains job-like behavior when bound: `JobInstance = role + workplace + holder + shift`. Template inheritance handles specialization (MasterSmith extends Smith). See `spec_roles.md`.

---

## 7. TOML-Only Template Format

**What it was**: Templates defined in pure TOML with `[meta]`, `[[contents]]` sections. The format was standard TOML without extensions.

**Source files**: `TemplateLib`

**Why discarded**: Pure TOML couldn't express inline conditions, step sequences, or method alternatives without verbose workarounds. Template definitions required embedded expressions that TOML's syntax couldn't accommodate naturally.

**Replaced by**: `.acf` format — TOML-compatible sections + `{ }` nesting + bare expressions. Maintains TOML readability for simple structures while supporting the expression language inline. See `spec_file_format.md`.

---

## 8. Separate Culture System

**What it was**: An explicit "culture" keyword or system that tagged entities with cultural affiliations, customs, and behavioral modifiers. Culture would have been a first-class concept with its own propagation and inheritance rules.

**Why discarded**: Culture is an emergent property, not a fundamental one. What looks like "culture" is actually shared expectation v-items + local relationship patterns + common role assignments. These propagate through familiarity naturally. A person moving to a new region absorbs local expectations through social contact, predicts consequences from local observers, and adapts behavior — which IS cultural assimilation, without any culture keyword.

**Replaced by**: No dedicated system. Culture emerges from expectation v-items (norms), familiarity-based propagation (transmission), and social judgment rules (enforcement). Drift happens naturally as expectations mutate during transmission. See `spec_contracts_authority.md`.

---

## 9. Object Categories / Type Hierarchy

**What it was**: Objects organized into category hierarchies: `weapons.melee.swords.longsword`, with behavior attached to categories. Different object types had different struct layouts.

**Source files**: `SUMMARY.md`

**Why discarded**: Category hierarchies create boundary problems (is a torch a weapon or a light source?). Different struct layouts per type require polymorphic dispatch. Categories are rigid — adding a new cross-cutting concern (e.g., "can be thrown") requires touching every category.

**Replaced by**: Single Object struct with composable traits. A torch has `[LightSource, Flammable, Weapon, Heavy]`. Rules reference traits directly: `item.has(Weapon)`, `item.trait(Flammable).ignition`. No categories, no hierarchy. See `spec_objects_and_space.md`.

---

## 10. Object Secrecy as Separate Knowledge Tracking System

**What it was**: An elaborate system tracking per-entity knowledge about per-object locations, using temporal/spatial/interactional vicinity calculations, a hybrid tracking model with public baselines and actor-specific knowledge entries, and integration with a 3-tier world history aging system.

**Source files**: `Object_system`

**Why discarded**: Over-engineered. The per-object knowledge tracking with vicinity calculations and history integration added significant complexity for a problem already solved by the virtual items system. Most object knowledge doesn't need explicit tracking — it falls under local observation (tier 1) or the object's secrecy stat filters propagation naturally.

**Replaced by**: Object secrecy is just the `obscurity` field on virtual items that represent objects. Low obscurity = common knowledge. High obscurity = requires espionage. The same propagation mechanics that handle all other knowledge handle object location knowledge. See `spec_knowledge.md`.

---

## 11. Stored Social Stats (Authority, Reputation, Wealth)

**What it was**: Authority, Reputation, and Wealth stored as explicit stats on entities, updated periodically.

**Why discarded**: Violates the core design principle of "store only fundamental state, derive everything else." A stored Authority stat can become stale or inconsistent with the actual relationships and military power it should reflect. Stored Wealth duplicates information already present in the inventory system.

**Replaced by**: All three are derived on demand. Authority = f(force, delegation, consensus, tradition). Reputation = aggregate of reputation axes across relationships. Wealth = sum of owned valuables. See `spec_core_stats.md`.
