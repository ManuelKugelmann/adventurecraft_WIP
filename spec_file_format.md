# File Format: `.acf`

TOML-compatible flat sections + `{ }` nesting + bare expressions.

---

## Tokens

```
Keyword     = rule | role | plan | method | do | wait | when | done | fail
            | require | counter | params | effect | bundle | template
            | profile
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
DeclType    = 'rule' | 'role' | 'plan' | 'bundle' | 'template' | 'profile'
Inheritance = ':' QualifiedId
Tags        = '[' Identifier (',' Identifier)* ']'
Body        = (Section | Entry | NamedBlock)*
Section     = Keyword '{' Body '}'
NamedBlock  = Identifier '{' Body '}'
Entry       = StepEntry | RuleEntry | Assignment | BareExpr
```

## Step Entries

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
            crew.weight >= 3
            any(edge(crew, *, member_of) WHERE skills.modify > 50)
        }
        priority = self.drives.luxury * 0.6

        case_joint:  do Sense.Indirect { target = $vault, secrecy = 0.8 }
        profile:     do Sense.Careful { target = $mark }
        recruit:     do Influence.Indirect { target = $specialists }
        disguises:   do Modify.Indirect { target = disguises }
        distract:    do Influence.Indirect { target = $guard, false = true }
        infiltrate:  do Move.Careful { target = $vault, secrecy = 0.9 }
        CRACK:       do Modify.Direct { target = $vault_door }
            prob = sigmoid(crew.skills.crafting - $vault.security)
            fail = ABORT
        grab:        do Transfer.Direct { source = $vault, secrecy = 0.9 }
        ABORT:       do Move.Indirect { destination = $safehouse, secrecy = 0.9 }
        cleanup:     do cover_tracks {}
    }

    done { self.inventory.value > $vault.former_value * 0.5 }
    fail { any(edge(self, *, known_by) WHERE faction == law_enforcement) }
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
1.  Every Expression resolves to STAT/EDGE/CONST/PARAM terminals
2.  Every do uses valid Action × Approach from 7×3 table
3.  Every plan decomposes to leaf actions within depth 6
4.  Every step has a name (bare do without name: is a parse error)
5.  Every prob Expression is bounded 0..1 (sigmoid/prob/min/max)
6.  No references to authority or reputation as stored stats
7.  Counter observables reference only externally visible state
    (NOT: drives, plans, knowledge, mood, skills, contracts)
8.  Counter chains terminate within depth 4
9.  No circular references in plan decomposition
10. All $param references have matching param definitions
11. sum(group allocations) <= 1.0, max 4 concurrent, min 0.05
12. Rule layers respect dependency order (L0→L1→L2→L3→L4)
13. Every wait in a plan implies a rule or condition that can produce the awaited state
```

Enforced at: LLM self-check → post-generate script → git pre-commit → CI.
