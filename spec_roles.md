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

### Resolution

An entity holds multiple roles. On each tick:
1. Collect all rules from all active roles
2. Sort by priority (lower = more urgent)
3. Evaluate conditions top-down
4. First rule whose condition is true → execute its action
5. If no rule fires → default idle behavior

Active plan steps (if any) override all role rules. See `spec_plans.md`.

---

## Role Activation

Roles gained/lost through world state, not assignment:

```
gain parent   { when edge(self, *, parent_of) exists }
gain farmer   { when edge(self, *, employed_by WHERE type == farm) }
lose farmer   { when NOT edge(self, *, employed_by WHERE type == farm) }
gain guard    { when edge(self, *, assigned_to WHERE tags.contains(military)) }
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
    rest:  when fatigue > 70, do rest {}, priority = 5
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

    craft_masterwork: when raw_materials.available AND self.skills.crafting > 80,
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

    repair_tools: when tools.condition < 0.1,
                  do Modify.Direct { target = $tools },
                  priority = 1

    mine:         when mine.has(ore) AND carrying < max,
                  do Transfer.Direct { source = $mine },
                  priority = 2

    haul:         when carrying(ore),
                  do Transfer.Direct { source = self, destination = $smelter },
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

    combat_stations: when alert_level >= CRITICAL,
                     do Defense.Indirect { position = $post },
                     priority = 1

    engage:          when threat_in(patrol_route),
                     do Attack.Direct { target = $threat },
                     priority = 2

    patrol:          when post_assigned,
                     do Move.Careful { route = $patrol_route },
                     priority = 3

    train:           when idle AND skills.attack < threshold,
                     do Modify.Direct { target = self, skill = attack },
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
                     do Sense.Careful { route = $perimeter },
                     priority = 4
}

role healer [support, medical] {
    shift = ALWAYS_ON

    critical:  when patient.health < 20,
               do Modify.Careful { target = $patient },
               priority = 1

    treat:     when patient.wounded,
               do Modify.Careful { target = $patient },
               priority = 2

    restock:   when supplies.low,
               do Modify.Indirect { recipe = $medicine },
               priority = 3

    train:     when idle,
               do Modify.Direct { target = self, skill = modify },
               priority = 4
}

role merchant [economic, trade] {
    shift = MARKET_HOURS

    negotiate: when trader_arrived,
               do Influence.Careful { target = $trader },
               priority = 1

    export:    when export_ready,
               do Transfer.Indirect { source = self, destination = $depot },
               priority = 2

    import:    when import_arrived,
               do Transfer.Direct { source = $depot, destination = $warehouse },
               priority = 3

    appraise:  when idle,
               do Sense.Careful { target = $goods },
               priority = 4
}
```

---

## Layer 3 Behavior Rules (Reactive, with agency)

Basic survival and social behavior — the first layer with agency. Single-step reactive:

```acf
role survival [basic, L3] {
    eat:       when hunger > 60,
               do Transfer.Direct { source = $food },
               priority = 1

    drink:     when thirst > 60,
               do Move.Direct { destination = $water },
               priority = 1

    rest:      when fatigue > 70,
               do Move.Direct { destination = $shelter },
               priority = 2

    flee:      when health < 30 OR region.threat > skills.defense * 2,
               do Move.Direct { destination = $safe_area },
               priority = 0

    warm:      when region.temperature < cold_tolerance,
               do Move.Direct { destination = $warm_region },
               priority = 3

    shelter:   when region.weather == storm,
               do Move.Direct { destination = $shelter },
               priority = 2
}

role social_basic [basic, L3] {
    join_group:  when drives.belonging > 60 AND NOT edge(self, *, member_of),
                 do Move.Direct { destination = $nearest_group },
                 priority = 5

    feed_child:  when edge(self, $child, parent_of) AND child.hunger > 50,
                 do Transfer.Direct { target = $child, item = food },
                 priority = 3

    warn:        when region.threat > 70,
                 do Influence.Direct { target = $group, info = $threat },
                 priority = 2
}
```

---

## Role → Job Instantiation

A role becomes a job instance when bound to a specific workplace and holder:

```
JobInstance {
    role: RoleId,
    workplace: RegionRef,
    holder: NodeRef,
    shift: ShiftSchedule,
    local_overrides: Rule[]?,       // player/leader can add/reorder rules
}
```

A group of 200 miners at Mine 3 shares one JobInstance. The rules evaluate against Mine 3's world state. A different group at Mine 7 has a different instance of the same role, evaluating against Mine 7.
