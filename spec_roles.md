# Roles (Reactive Agency)

Roles are repeating priority-sorted rules. First layer with agency. Cover ~95% of daily activity.

---

## Structure

```acf
role <id> : <parent>? [tags] {
    <rule_name>: when <condition>, do <Action.Approach> { params }, priority = <Expr>
    ...

    gain { when <condition> }
    lose { when <condition> }
}
```

Implementation: `ActiveRole { Owner, Target: TemplateId }` — a Multi relationship trait. An entity can hold multiple active roles simultaneously.

### Resolution

An entity holds multiple roles (multiple ActiveRole trait instances). On each tick:
1. Collect all rules from all active roles
2. Sort by priority (lower = more urgent)
3. Evaluate conditions top-down
4. First rule whose condition is true → execute its action
5. If no rule fires → default idle behavior

Active plan steps (AgencyTrait.ActivePlan) override all role rules. See `spec_plans.md`.

---

## Role Activation

Roles gained/lost through world state, not assignment:

```
gain parent   { when Guards(Owner, child) exists }
gain farmer   { when EmployedBy(Owner, farm) exists }
lose farmer   { when EmployedBy(Owner, farm) not exists }
gain guard    { when MemberOf(Owner, military_unit) exists }
```

### Shift Patterns

Roles can have temporal activation:

```
role farmer [economic, rural] {
    shift = SEASONAL(grow_season=ACTIVE, off_season=REDUCED)
    ...
}

role guard [military] {
    shift = ROTATING_WATCH
    ...
}
```

When shifts don't overlap, no conflict resolution needed — a farmer-militia member runs farming rules during the day, guard rules during evening watch.

When shifts overlap (emergency), rule priority resolves it.

---

## Inheritance

Specialized roles extend base roles:

```acf
role laborer [economic] {
    work:  when workplace.has_task, do Modify.Direct { target = $task }, priority = 10
    haul:  when stockpile.needs_item, do Transfer.Direct { source = $item, destination = $stockpile }, priority = 15
    rest:  when Vitals.Fatigue > 70, do rest {}, priority = 5
}

role farmer : laborer [economic, rural] {
    plow:    when season == spring AND field.state == fallow,
             do Modify.Direct { target = $field, objects = [plow] },
             priority = 10

    harvest: when season == autumn AND field.crop.ready,
             do Transfer.Direct { source = $field },
             priority = 15

    pest:    when field.pests > 0,
             do Modify.Direct { target = $field.pests },
             priority = 12
}

role master_smith : smith [economic, craft] {
    teach:   when apprentice.present,
             do Influence.Direct { target = $apprentice },
             priority = 8

    craft_masterwork: when contains(Owner, Material) AND Skills[5] > 80,
             do Modify.Indirect { recipe = $masterwork_recipe },
             priority = 10

    min_skill = crafting >= 4
}
```

---

## Example Roles

```acf
role miner : laborer [economic, extraction] {
    shift = DAY_SHIFT

    repair_tools: when Condition.Condition < 10,
                  do Modify.Direct { target = $tools },
                  priority = 1

    mine:         when NaturalResource.Amount > 0,
                  do Transfer.Direct { source = $mine },
                  priority = 2

    haul:         when contains(Owner, ore),
                  do Transfer.Direct { source = Owner, destination = $smelter },
                  priority = 3

    clear:        when mine.blocked,
                  do Modify.Direct { target = $rubble },
                  priority = 4

    maintain:     when idle,
                  do Modify.Direct { target = $mine },
                  priority = 5
}

role guard [military] {
    shift = ROTATING_WATCH

    combat_stations: when PopulatedTrait.ThreatLevel >= 90,
                     do Defense.Indirect { position = $post },
                     priority = 1

    engage:          when HostileTo exists AND distance(Owner, Target) < 20,
                     do Attack.Direct { target = $threat },
                     priority = 2

    patrol:          when MemberOf exists,
                     do Move.Indirect { route = $patrol_route },
                     priority = 3

    train:           when idle AND Skills[6] < threshold,
                     do Modify.Direct { target = Owner, skill = attack },
                     priority = 4

    hold:            when idle,
                     do Defense.Direct { position = $post },
                     priority = 5
}

role scout [military, reconnaissance] {
    shift = MISSION_BASED

    execute_mission: when mission_assigned,
                     do Sense.Indirect { target = $target_region, secrecy = 0.8 },
                     priority = 1

    report:          when threat_detected,
                     do Influence.Direct { target = $leader, info = $threat },
                     priority = 2

    return:          when returning,
                     do Move.Indirect { destination = $home },
                     priority = 3

    patrol:          when idle,
                     do Sense.Indirect { route = $perimeter },
                     priority = 4
}

role healer [support, medical] {
    shift = ALWAYS_ON

    critical:  when Vitals.Health < 20,
               do Modify.Indirect { target = $patient },
               priority = 1

    treat:     when Vitals.Wounds > 0,
               do Modify.Indirect { target = $patient },
               priority = 2

    restock:   when supplies.low,
               do Modify.Structured { recipe = $medicine },
               priority = 3

    train:     when idle,
               do Modify.Direct { target = Owner, skill = modify },
               priority = 4
}

role merchant [economic, trade] {
    shift = MARKET_HOURS

    negotiate: when trader_arrived,
               do Influence.Indirect { target = $trader },
               priority = 1

    export:    when export_ready,
               do Transfer.Structured { source = Owner, destination = $depot },
               priority = 2

    import:    when import_arrived,
               do Transfer.Direct { source = $depot, destination = $warehouse },
               priority = 3

    appraise:  when idle,
               do Sense.Indirect { target = $goods },
               priority = 4
}
```

---

## Layer 3 Behavior Rules (Reactive, with agency)

Basic survival and social behavior — the first layer with agency. Single-step reactive:

```acf
role survival [basic, L3] {
    eat:       when Vitals.Hunger > 60,
               do Transfer.Direct { source = $food },
               priority = 1

    drink:     when Vitals.Thirst > 60,
               do Move.Direct { destination = $water },
               priority = 1

    rest:      when Vitals.Fatigue > 70,
               do Move.Direct { destination = $shelter },
               priority = 2

    flee:      when Vitals.Health < 30 OR PopulatedTrait.ThreatLevel > 80,
               do Move.Direct { destination = $safe_area },
               priority = 0

    shelter:   when Climate.Temperature < cold_tolerance,
               do Move.Direct { destination = $warm_region },
               priority = 3
}

role social_basic [basic, L3] {
    join_group:  when Drives.Belonging > 60 AND NOT MemberOf exists,
                 do Move.Direct { destination = $nearest_group },
                 priority = 5

    feed_child:  when Guards(Owner, child) AND child.Vitals.Hunger > 50,
                 do Transfer.Direct { target = $child, item = food },
                 priority = 3

    warn:        when PopulatedTrait.ThreatLevel > 70,
                 do Influence.Direct { target = $group, info = $threat },
                 priority = 2
}
```

---

## Role → Job Instantiation

A role becomes a job instance when bound to a specific workplace and holder. ActiveRole trait + workplace context = job.

```
JobInstance = ActiveRole(holder → role_template) + workplace node + shift schedule
```

A group of 200 miners at Mine 3 shares one role binding. The rules evaluate against Mine 3's world state. A different group at Mine 7 has a different instance of the same role, evaluating against Mine 7.
