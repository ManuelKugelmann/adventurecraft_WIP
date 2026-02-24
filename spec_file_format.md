# File Format: `.acf`

TOML-compatible flat sections + `{ }` nesting + bare expressions. Parsed to IR consumed by codegen (Tier 1) and interpreter (Tier 2). See `architecture.md` §8–9.

---

## Tokens

```
Keyword     = rule | role | plan | method | do | wait | when | done | fail
            | require | counter | params | effect | condition | bundle
            | template | profile | layer | scope
Identifier  = [a-zA-Z_][a-zA-Z0-9_]*
QualifiedId = Identifier ('.' Identifier)*
ParamRef    = '$' QualifiedId
Number      = [0-9]+ ('.' [0-9]+)?
Bool        = true | false
String      = '"' [^"]* '"'              # ONLY for human text with spaces
Symbol      = { } [ ] ( ) = , : + - * / < > >= <= == != += -=
Comment     = '#' rest_of_line
Label       = [A-Z][A-Z0-9_]* ':'       # jump target
StepName    = [a-z][a-z0-9_]* ':'       # human-readable step name
```

## Quoting Rule

Quoted: only human-readable text containing spaces.
Unquoted: identifiers, tags, refs, expressions, keywords, numbers, bools.

---

## Declaration Grammar

```ebnf
File        = Declaration+
Declaration = DeclType QualifiedId Inheritance? Tags? '{' Body '}'
            | RuleDecl
DeclType    = 'role' | 'plan' | 'bundle' | 'template' | 'profile'
Inheritance = ':' QualifiedId
Tags        = '[' Identifier (',' Identifier)* ']'
Body        = (Section | Entry | NamedBlock)*
Section     = Keyword '{' Body '}'
NamedBlock  = Identifier '{' Body '}'
Entry       = StepEntry | RuleEntry | Assignment | BareExpr
```

## Rule Declarations

Rules use a colon-separated format:

```ebnf
RuleDecl    = 'rule' QualifiedId ':'
              ('layer:' LayerId)?
              ('scope:' TraitKind)?
              ('condition:' Expr)*
              ('effect:' EffectExpr)+

LayerId     = 'L0_Physics' | 'L1_Biology' | 'L2_Items' | 'L3_Social' | 'L4_Complex' | 'L4_Economic'
EffectExpr  = EffectOp TraitField Params
EffectOp    = 'Accumulate' | 'Decay' | 'Set' | 'Transfer' | 'Spread'
            | 'Create' | 'Destroy' | 'AddTrait' | 'RemoveTrait'
            | 'SkillCheck'
TraitField  = QualifiedId                # e.g. Vitals.Health, Weapon.Damage
```

### Example Rule

```acf
rule hunger_drain:
    layer: L1_Biology
    scope: Vitals
    effect: Accumulate Vitals.Hunger rate=-0.001

rule eat_food:
    layer: L3_Basic
    scope: Vitals
    condition: Vitals.Hunger > 50
    condition: contains(Owner, Edible)
    effect: Transfer Edible.Nutrition into=Vitals.Hunger
```

## Step Entries (plans and roles)

```ebnf
StepEntry   = StepName DoStep StepProps?
            | Label DoStep StepProps?
DoStep      = 'do' ActionRef ActionArgs?
            | 'wait' Expr
ActionRef   = Action '.' Approach           # e.g. Attack.Direct
            | QualifiedId                    # e.g. military.breach_walls
ActionArgs  = '{' (Identifier '=' Expr (',' Identifier '=' Expr)*)? '}'
StepProps   = (Indent (ProbLine | FailLine | WhenLine))*
ProbLine    = 'prob' '=' Expr
FailLine    = 'fail' '=' Label
WhenLine    = 'when' Expr
```

## Role Rule Entries

```ebnf
RuleEntry   = StepName 'when' Expr ',' DoStep (',' 'priority' '=' Expr)?
```

---

## Three Data Types

```
rule    trigger → state change       (world mechanics, no agency)
role    repeating priority rules     (reactive agency)
plan    sequential do/wait steps     (proactive agency)
```

### Two Step Keywords

```
do      actor acts        → Action.Approach { params }
wait    actor waits       → condition becomes true
```

---

## Example

```acf
plan criminal.heist [criminal, economic] {

    params {
        vault = SpatialRef
        crew = EntityRef
        mark = EntityRef
    }

    require { sense >= 40 }

    method classic {
        when {
            crew.Weight >= 3
            Skills[16] > 50
        }
        priority = Drives.Luxury * 0.6

        case_joint:  do Sense.Indirect { target = $vault, secrecy = 0.8 }
        profile:     do Sense.Indirect { target = $mark }
        recruit:     do Influence.Indirect { target = $specialists }
        disguises:   do Modify.Structured { target = disguises }
        distract:    do Influence.Indirect { target = $guard, false = true }
        infiltrate:  do Move.Indirect { target = $vault, secrecy = 0.9 }
        CRACK:       do Modify.Direct { target = $vault_door }
            prob = sigmoid(Skills[5] - $vault.security)
            fail = ABORT
        grab:        do Transfer.Direct { source = $vault, secrecy = 0.9 }
        ABORT:       do Move.Indirect { destination = $safehouse, secrecy = 0.9 }
        cleanup:     do cover_tracks {}
    }

    done { contains(Owner, Valuable) }
    fail { KnowsAbout(law_enforcement, Owner) exists }
}
```

---

## Provenance

`_provenance { }` section in every file. Stripped by loader: `if key.startsWith("_"): skip`.

```acf
_provenance {
    sources = [
        { type = military_doctrine, id = fm3-90_ch12, confidence = 0.95 }
        { type = tvtropes, id = trope:TheSiege, confidence = 0.7 }
    ]
    model = claude-sonnet-4-20250514
    prompt_version = extract_task_v3
    verified = true
    reviewer = manu
    notes = "Adjusted force ratio from 3x to 2x"
}
```

Stats in separate `.stats.acf` sidecars. Gitignored, runtime-generated.

---

## Validation Rules

```
1.  Every condition resolves to trait field comparisons or built-in functions
2.  Every effect uses a valid elementary op (Accumulate, Decay, Set, Transfer, etc.)
3.  Every do uses valid Action × Approach from 7×3 table
4.  Every plan decomposes to leaf actions within depth 6
5.  Every step has a name (bare do without name: is a parse error)
6.  Every prob Expression is bounded 0..1 (sigmoid/prob/min/max)
7.  No references to authority or reputation as stored stats
8.  Counter observables reference only externally visible state
    (NOT: drives, plans, knowledge, mood, skills, contracts)
9.  Counter chains terminate within depth 4
10. No circular references in plan decomposition
11. All $param references have matching param definitions
12. sum(group allocations) <= 1.0, max 4 concurrent, min 0.05
13. Rule layers respect dependency order (L0→L1→L2→L3→L4)
14. Every wait in a plan implies a rule or condition that can produce the awaited state
15. Rule scope references a valid trait kind
```

Enforced at: LLM self-check → post-generate script → git pre-commit → CI.
