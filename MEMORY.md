# AdventureCraft Simulation Values Summary

## Static Values (Game Definition Data)

### Core System Framework

#### Universal Attributes (3 total)
```javascript
ATTRIBUTES {
  physical: 50,    // strength + agility + health combined
  mental: 50,      // intelligence + willpower combined  
  social: 50       // charisma + authority combined
}
```

#### Universal Skills (7 total - one per action)
```javascript
SKILLS {
  move: 0,         // movement, travel, athletics
  modify: 0,       // crafting, building, operating
  attack: 0,       // combat offense
  defend: 0,       // combat defense, tactics
  transfer: 0,     // trading, gathering, logistics
  influence: 0,    // persuasion, authority, intrigue
  sense: 0         // observation, research, awareness
}
```

#### Core Drives (4 total)
```javascript
DRIVES {
  survival: 50,    // safety, security, basic needs
  comfort: 50,     // luxury, ease, quality of life
  power: 50,       // dominance, control, authority
  social: 50       // belonging, relationships, community
}
```

### Race Templates (Only Static Templates)

#### Base Human Template
```javascript
HUMAN {
  attributes: {physical: 50, mental: 50, social: 50},
  drives: {survival: 50, comfort: 50, power: 30, social: 60},
  lifespan: 80,
  size: "medium"
}
```

#### Variant Races (inherit from human)
```javascript
DWARF {
  PARENT: "human",
  attributes: {physical: 60, social: 40},  // only differences stored
  drives: {comfort: 60, power: 40},
  lifespan: 150
}

ELF {
  PARENT: "human", 
  attributes: {mental: 60, physical: 45},
  drives: {social: 70, power: 20},
  lifespan: 300
}
```

### Object Category System

#### Object Categories (Static Templates)
```javascript
OBJECT_CATEGORIES = {
  "furniture.common": {
    base_value: 20,
    weight: "medium", 
    examples: ["table", "chair", "stool", "shelf"],
    durability: 80
  },
  
  "furniture.luxury": {
    base_value: 200,
    weight: "heavy",
    examples: ["ornate_chair", "marble_table", "silk_bed"],
    durability: 90
  },
  
  "tools.farming": {
    base_value: 15,
    weight: "light",
    examples: ["hoe", "rake", "scythe", "bucket"],
    durability: 60
  },
  
  "weapons.basic": {
    base_value: 50,
    weight: "medium",
    examples: ["iron_sword", "wooden_spear", "leather_armor"],
    durability: 70
  },
  
  "food.grain": {
    base_value: 2,
    weight: "light",
    examples: ["wheat", "barley", "oats"],
    durability: 30,
    perishable: true
  },
  
  "coin.currency": {
    base_value: 1,
    weight: "negligible", 
    examples: ["copper_coin", "silver_coin", "gold_coin"],
    durability: 100
  }
}
```

---

## Dynamic Simulation Values (World State)

### Group Entities (Algorithmic Groupings)

#### Group Structure with Internal Subgroups
```javascript
GOLDENHAVEN_POOR_FARMERS {
  PARENT: "human",                 // only static template reference
  LOCATION: "goldenhaven",
  GROUP_TYPE: "economic_class",    // simulation-determined grouping
  size: 80,                        // represents 80 individuals
  
  // Simulated group characteristics (computed from members)
  avg_wealth: 10,
  avg_skills: {transfer: 40, modify: 20, defend: 10},
  avg_drives: {survival: 80, comfort: 20, power: 10, social: 60},
  
  // Internal heterogeneity via 4 subgroups
  subgroups: [
    {
      fraction: 0.4,                    // 32 people (40%)
      opinion_drift: {mayor: +10, nobles: -5},
      skill_drift: {transfer: +5, influence: -3},
      drive_drift: {survival: +10}
    },
    {
      fraction: 0.3,                    // 24 people (30%) 
      opinion_drift: {mayor: -15, guards: +8},
      skill_drift: {attack: +8, defend: +5},
      drive_drift: {power: +15}
    },
    {
      fraction: 0.2,                    // 16 people (20%)
      opinion_drift: {nobles: +20, poor: -10},
      skill_drift: {influence: +12, sense: +6},
      drive_drift: {comfort: +20, social: +10}
    },
    {
      fraction: 0.1,                    // 8 people (10%)
      opinion_drift: {everyone: -20},
      skill_drift: {sense: +15, move: +10},
      drive_drift: {survival: +30, social: -20}
    }
  ],
  
  // Split/merge thresholds
  max_opinion_divergence: 40,           // split if subgroups differ by >40
  merge_similarity_threshold: 15,      // merge groups if centers within 15
  
  // Group coherence via self-relationship
  relationships: {
    "self": {opinion: 60, familiarity: 100},  // group cohesion
    "goldenhaven_mayor": {opinion: 40, familiarity: 70},
    "goldenhaven_rich_farmers": {opinion: -20, familiarity: 80},
    "goldenhaven_guards": {opinion: 60, familiarity: 50}
  },
  
  // Location probability distribution
  location_distribution: {
    "goldenhaven.homes": 0.6,          // 60% at home
    "goldenhaven.fields": 0.25,        // 25% working fields
    "goldenhaven.market": 0.1,         // 10% at market
    "travel.nearby": 0.05              // 5% traveling
  },
  
  // Multiple action plan execution (fractional)
  current_actions: {
    "transfer": 0.6,               // resource gathering/trading
    "influence": 0.2,              // social coordination  
    "move": 0.15,                  // travel/positioning
    "sense": 0.05                  // observation/awareness
  },
  max_concurrent_actions: 4,       // compute limit
  
  // Group knowledge (hierarchical structure)
  knowledge: {
    "social.local_network": {level: 70, confidence: 80, secrecy: 20, timestamp: "day_45"},
    "craft.agriculture": {level: 80, confidence: 95, secrecy: 10, timestamp: "day_1"},
    "social.trust_map": {level: 60, confidence: 70, secrecy: 40, timestamp: "day_42"}
  },
  
  current_mood: 45,                // group morale
  wealth: 10,                      // basic wealth level
  recent_events: ["harvest.failure", "market.activity"]
}
```

### Individual Entities (Story-Relevant Only)

#### Individual Structure (Direct from Race Templates)
```javascript
MAYOR_ALDWIN {
  PARENT: "human",                 // inherits from race template only
  LOCATION: "goldenhaven",
  name: "Mayor Aldwin Goldwright",
  
  // Individual stat overrides from base human
  attributes: {social: 75, mental: 65},
  skills: {influence: 75, sense: 55},
  wealth: 80,
  drives: {power: 80, social: 60},
  
  // Individual relationships
  relationships: {
    "player": {opinion: 50, familiarity: 0},
    "goldenhaven_poor_farmers": {opinion: 30, familiarity: 60},
    "goldenhaven_crafters": {opinion: 70, familiarity: 80},
    "house_blackstone": {opinion: 80, familiarity: 90}
  },
  
  // Individual knowledge (hierarchical structure)
  knowledge: {
    "politics.local": {level: 90, confidence: 95, secrecy: 60, timestamp: "day_40"},
    "politics.regional": {level: 60, confidence: 80, secrecy: 70, timestamp: "day_35"}, 
    "secrets.town": {level: 80, confidence: 90, secrecy: 90, timestamp: "day_30"},
    "reputation.player": {level: 20, confidence: 60, secrecy: 30, timestamp: "day_47"}
  },
  
  current_goals: ["maintain_control", "satisfy_nobles"],
  mood: 55,
  inventory: ["seal.mayor", "ledger.town", "weapon.ceremonial"]
}

WILLEM_REBEL {
  PARENT: "human",                 // inherits from race template only
  LOCATION: "goldenhaven",
  name: "Willem the Angry",
  
  // Individual characteristics (no group template)
  attributes: {physical: 60, social: 55},
  drives: {power: 70, survival: 80},
  skills: {influence: 45, attack: 30},
  wealth: 5,
  
  relationships: {
    "goldenhaven_poor_farmers": {opinion: 90, familiarity: 95},
    "mayor_aldwin": {opinion: -60, familiarity: 70},
    "goldenhaven_rich_farmers": {opinion: -80, familiarity: 60}
  },
  
  knowledge: {
    "tactics.rebellion": {level: 40, confidence: 60, secrecy: 80, timestamp: "day_44"},
    "social.supporters": {level: 80, confidence: 85, secrecy: 70, timestamp: "day_46"},
    "weakness.mayor": {level: 60, confidence: 70, secrecy: 90, timestamp: "day_43"}
  },
  
  current_goals: ["organize_group", "confront_authority"],
  mood: 25  // angry and motivated
}
```

### Object System

#### Simple Ownership + Public Knowledge
```javascript
OBJECT {
  id: "sword.masterwork",
  owner: "gareth_blacksmith",
  location: "goldenhaven.blacksmith",
  secrecy: 20,                    // 0-100 (0=public, 100=top secret)
  
  // Only track special knowledge if secrecy > 0
  hidden_knowledge: {
    "player": {knows_owner: false, knows_location: false},
    "thieves_guild": {knows_owner: true, knows_location: true}
  }
}

OBJECT {
  id: "grain.sack",
  owner: "farmer_john", 
  location: "goldenhaven.granary",
  secrecy: 0                      // public knowledge - everyone knows
}
```

#### Container System for Efficiency
```javascript
CONTAINER {
  id: "peasant_house",
  owner: "farmer_john",
  location: "goldenhaven.cottage_row",
  secrecy: 30,                    // home privacy
  
  // Bulk category storage
  contents: {
    "furniture.common": {quantity: 8, condition: 60},      // table, chairs, bed, etc.
    "tools.farming": {quantity: 12, condition: 70},        // various farm tools
    "food.grain": {quantity: 50, condition: 90},           // sacks of grain
    "clothing.basic": {quantity: 6, condition: 50},        // family clothes
    "coin.currency": {quantity: 45, condition: 100}        // mixed coins
  },
  
  // Special individual items
  special_items: [
    {id: "sword.fathers", category: "weapons.basic", condition: 40, story: "inherited"},
    {id: "ring.wedding", category: "jewelry.personal", condition: 80, story: "sentimental"}
  ]
}

CONTAINER {
  id: "blacksmith_inventory", 
  owner: "gareth_blacksmith",
  location: "goldenhaven.blacksmith",
  secrecy: 20,
  
  contents: {
    "metal.raw": {quantity: 200, condition: 100},          // iron ingots, coal
    "tools.smithing": {quantity: 15, condition: 80},       // hammers, tongs, files
    "weapons.basic": {quantity: 25, condition: 90},        // finished weapons
    "weapons.quality": {quantity: 5, condition: 95},       // superior weapons
    "coin.currency": {quantity: 300, condition: 100}       // payment for goods
  },
  
  special_items: [
    {id: "hammer.masters", category: "tools.smithing", condition: 100, story: "masterwork_tool"},
    {id: "sword.commissioned", category: "weapons.quality", condition: 100, story: "special_order"}
  ]
}
```

### Relationship System (Universal)

#### Universal Relationship Structure
```javascript
RELATIONSHIP {
  opinion: 50,        // how much they like each other (-100 to +100)
  familiarity: 0      // how well they know each other (0-100)
}

// Applies to all relationships including self-relationship:
// - Individual to Individual
// - Individual to Group  
// - Group to Group
// - Group to Self (tracks internal coherence)
```

### Knowledge System (Hierarchical Structure)

#### Knowledge Entry Structure
```javascript
KNOWLEDGE_ENTRY {
  location: "source_entity",      // source of knowledge (omit for self-knowledge)
  topic: "domain.specific_topic", // hierarchical topic identifier
  level: 75,                      // 0-100 depth of knowledge
  confidence: 85,                 // -100 to +100 (negative = known false)
  secrecy: 60,                    // 0-100 sensitivity level
  timestamp: "day_42"             // when knowledge was acquired
}

// Examples:
"social.local_network": {level: 70, confidence: 80, secrecy: 20, timestamp: "day_45"},
"lore.ancient": {level: 30, confidence: 60, secrecy: 50, timestamp: "day_10"},
"tactics.combat": {level: 85, confidence: 95, secrecy: 80, timestamp: "day_32"},
"social.trust_map": {level: 60, confidence: 70, secrecy: 40, timestamp: "day_42"}
```

---

## Key Design Principles

### Compression Strategy
- **Groups handle 90% of population**: 80 individuals → 1 group entity
- **Individuals only for story characters**: mayors, rebels, key NPCs
- **Single static template**: only race templates (human, dwarf, elf)
- **Algorithmic grouping**: groups formed by simulation based on location, wealth, skills
- **Universal relationship system**: self-relationships track group coherence
- **Object containerization**: bulk objects stored as categories with quantities

### Computational Limits
- **Max 4 concurrent actions per group** (performance optimization)
- **Fractional action allocation** from multiple planning algorithm variants
- **4 internal subgroups per group** for heterogeneity simulation
- **Group splitting/merging** based on opinion divergence thresholds
- **Probabilistic location tracking** instead of exact positions

### Emergent Complexity
- **Simple rules create complex behavior**: self-relationships, fractional action plans, knowledge confidence
- **Groups can split** when internal subgroup divergence exceeds thresholds
- **Groups can merge** when similarity drops below threshold
- **Individual extraction** when group members become story-relevant
- **Knowledge decay and verification** through confidence and timestamp tracking
- **Algorithmic social stratification**: groups emerge from wealth, location, skill similarities

### Object & Knowledge Efficiency
- **Public knowledge default**: all ownership/location is public unless marked secret
- **Secrecy levels 0-100**: fine-grained privacy without complex tracking
- **Category-based storage**: furniture.common, tools.farming, weapons.basic, etc.
- **Individual item extraction**: create unique objects only when story-relevant
- **Hierarchical knowledge topics**: domain.specific_topic naming convention

### Algorithmic Group Formation
```javascript
// Groups formed by simulation algorithms, not templates
function createGroups(population, location) {
  let groups = [];
  
  // Economic stratification
  let poor = population.filter(p => p.wealth < 30);
  let middle = population.filter(p => p.wealth >= 30 && p.wealth < 70);
  let rich = population.filter(p => p.wealth >= 70);
  
  // Skill-based clustering within economic classes
  let farmers = poor.filter(p => p.skills.transfer > 30);
  let crafters = middle.filter(p => p.skills.modify > 40);
  let guards = population.filter(p => p.skills.attack > 40);
  
  // Create group entities with 4 subgroups each
  if (farmers.length > 10) groups.push(createGroupEntity(farmers, "economic_class"));
  if (crafters.length > 5) groups.push(createGroupEntity(crafters, "craft_guild"));
  if (guards.length > 3) groups.push(createGroupEntity(guards, "military_unit"));
  
  return groups;
}
```

### Fractional Actions from Planning
```javascript
Planning algorithm generates multiple action plans for any group:
Plan A: 100% transfer (resource gathering)
Plan B: 100% influence (social coordination)  
Plan C: 100% move (positioning/travel)

Group executes fractional combination:
- 60% of Plan A (transfer)
- 20% of Plan B (influence)
- 15% of Plan C (move)
- 5% sense/other
```

### Example Population Compression
```
Town of 150 people simulated as:
- All inherit from "human" template
- Algorithm creates 4 group entities (represent 120 people)
- 10 individuals extracted for story relevance (30 key characters)
= 14 total entities instead of 150
= 90% data reduction from algorithmic grouping
= 1 static template serves entire population
```