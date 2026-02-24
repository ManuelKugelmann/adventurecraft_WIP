# Knowledge Representation Framework - System Summary

## Core Knowledge Structure

### Basic Knowledge Entry
```javascript
knowledge_entry = {
  location: "merchant_bob",        // source of knowledge (omit for self)
  topic: "trade_routes.silk_road", // hierarchical topic identifier
  level: 75,                       // 0-100 depth of knowledge
  confidence: 85,                  // -100 to +100 (negative = known false)
  secrecy: 60,                     // 0-100 sensitivity level
  timestamp: "2025_08_15"          // when knowledge was acquired
}
```

### Knowledge Categories

**Personal Knowledge** (location omitted = self)
- Auto-generated from skills: `{topic: "blacksmithing", level: 75}`
- Specialized sub-domains: `{topic: "blacksmithing.damascus_steel", level: 90}`
- Acquired knowledge: `{topic: "lord_blackstone.secrets", level: 60, confidence: 80}`
- Direct experiences: `{topic: "castle_layout.secret_passage", level: 90, confidence: 95}`
- Learned information: `{topic: "dragon_weaknesses.silver", level: 45, confidence: 60}`

**Source Knowledge** (knowledge about who/what/where has information)
- Person sources: `{location: "scholar_aldric", topic: "dragon_lore", level: 85, confidence: 90}`
- Group sources: `{location: "thieves_guild", topic: "city_secrets", level: 70, confidence: 80}`
- Written sources: `{location: "tome_of_dragons", topic: "fire_breath", level: 90, confidence: 95}`
- Location sources: `{location: "royal_archives", topic: "genealogy", level: 95, confidence: 85}`
- Physical evidence: `{location: "temple_wall", topic: "ancient_ritual", level: 60, confidence: 70}`

### Unified Source Knowledge Structure
```javascript
// All source knowledge follows same pattern: location + topic + assessment
examples = [
  {location: "merchant_bob", topic: "trade_routes", level: 80, confidence: 90},
  {location: "spy_network", topic: "enemy_plans", level: 75, confidence: 70},
  {location: "ancient_library", topic: "lost_magic", level: 95, confidence: 85},
  {location: "haunted_forest", topic: "spirit_locations", level: 40, confidence: 30},
  {location: "bloodstained_letter", topic: "assassination_plot", level: 85, confidence: 80}
]
```

## Temporal Knowledge Integration

### Time-Embedded Topics
```javascript
// Recent intelligence vs outdated information
{topic: "enemy_troop_movements.2025_08_17", level: 80, confidence: 90}
{topic: "enemy_troop_movements.2025_08_10", level: 60, confidence: 40}

// Seasonal/cyclical knowledge
{topic: "market_prices.harvest_season", level: 70}
{topic: "lord_blackstone.weekly_schedule", level: 85}
```

### Knowledge Decay and Updates
- Recent information has higher confidence
- Contradictory evidence updates existing knowledge
- Time-sensitive knowledge automatically marked as stale

## Knowledge Discovery and Acquisition

### Discovery Actions
```javascript
// SENSE actions generate knowledge entries
sense_result = {
  action: "SENSE(actor, 'lord_blackstone.weekly_schedule', method='observation')",
  generates: {
    topic: "lord_blackstone.weekly_schedule", 
    level: 75,
    confidence: 85,
    secrecy: 40,
    timestamp: current_time
  }
}

// Research generates meta-knowledge about sources
research_result = {
  action: "SENSE(actor, 'who_knows.dragon_lore', method='research')",
  generates: {
    topic: "who_knows.dragon_lore",
    level: 60,
    details: "Scholar_Aldric (level ~80), Ancient_Library (level ~95), Dragon_Cult (level ~100)"
  }
}
```

### Source Discovery and Research
```javascript
// SENSE actions can discover sources
research_action = {
  action: "SENSE(actor, 'who_knows.dragon_lore', method='research')",
  generates: [
    {location: "scholar_aldric", topic: "dragon_lore", level: 85, confidence: 90},
    {location: "ancient_library", topic: "dragon_lore", level: 95, confidence: 80},
    {location: "dragon_cult_elder", topic: "dragon_lore", level: 100, confidence: 60}
  ]
}

// Investigation reveals source accessibility
investigation_result = {
  action: "SENSE(actor, 'access_requirements.ancient_library', method='investigation')",
  generates: {
    topic: "ancient_library.access_requirements",
    level: 80,
    details: "Requires royal permission OR stealth entry after midnight"
  }
}
```d_techniques",
    from_level: 90,
    to_level: "min(student_aptitude, teacher_level * 0.8)"
  }
}
```

## Implementation Architecture

### Storage Efficiency
```javascript
// Hierarchical knowledge inheritance (groups → individuals)
// Lazy evaluation for detailed knowledge generation
// Sparse storage - only track non-default knowledge
// Topic indexing for fast retrieval
```

### Knowledge Processing
```javascript
// Automatic skill-to-knowledge mapping
// Time-based knowledge decay for temporal topics
// Confidence updating when contradictory evidence emerges
// Secrecy-based information spread modeling
```

### Integration with Core Framework
- **7 Universal Actions**: SENSE actions generate knowledge, others consume it
- **Relationship System**: Knowledge sharing affected by trust and familiarity
- **Drive System**: Knowledge acquisition driven by actor motivations and goals

## Key Advantages

1. **Unified Structure**: Same format for personal knowledge and source knowledge
2. **Temporal Awareness**: Knowledge naturally ages and becomes outdated
3. **Uncertainty Modeling**: Confidence levels reflect information quality
4. **Source Attribution**: Always know where information came from or could come from
5. **Information Economics**: Secrecy and access create natural information markets
6. **Scalable Detail**: Simple topics can expand to detailed knowledge when needed

## Usage Patterns

### For Decision Making
- Actors choose actions based on knowledge level and confidence
- Low confidence knowledge triggers information gathering
- Source knowledge guides where to seek additional information

### For Social Dynamics
- Source knowledge drives information trading and alliance building
- Knowledge gaps create dependency relationships
- Information asymmetries enable deception and advantage

### For World Building
- Source knowledge creates realistic information ecosystems
- Knowledge requirements drive NPC specialization and role definition
- Information scarcity creates natural quest objectives and exploration goals

This framework creates a **living information ecosystem** where knowledge has sources, ages naturally, spreads through social networks, and drives intelligent decision-making behavior.