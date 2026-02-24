Relationship & Knowledge Data StructuresRelationship System StructureRelationship Data FormatRELATIONSHIP {
  target: entity_id (individual or group),
  debt: -100 to +100,
  reputation: -100 to +100, 
  affection: -100 to +100,
  familiarity: 0-100
}Relationship Matrix (Any Entity → Any Entity)Individual → Individual: Personal relationships between specific actorsIndividual → Group: How individual feels about/relates to entire groupsGroup → Individual: How group collectively regards specific individualsGroup → Group: Inter-group relationships (alliances, rivalries, trade relationships)Examples// Individual blacksmith's relationships
blacksmith.relationships = [
  {target: "village_guards", debt: 10, reputation: 60, affection: 40, familiarity: 70},
  {target: "merchant_bob", debt: -20, reputation: 80, affection: 60, familiarity: 85},
  {target: "bandit_gang", debt: 0, reputation: -70, affection: -80, familiarity: 30}
]

// Village guards' collective relationships  
village_guards.relationships = [
  {target: "blacksmith", debt: -10, reputation: 70, affection: 50, familiarity: 70},
  {target: "neighboring_village", debt: 0, reputation: 40, affection: 20, familiarity: 45}
]Knowledge System Structure (Analogous)Knowledge Data FormatKNOWLEDGE {
  target: topic/entity_id,
  generic: 0-100,
  specialized: {domain: 0-100},
  detailed: {specific_fact: {content, confidence, source}}
}Knowledge Matrix (Any Entity → Any Topic/Entity)Individual → Topic: Personal knowledge about subjectsIndividual → Other Individual: What person knows about specific peopleGroup → Topic: Collective group knowledgeGroup → Individual: What group knows about specific peopleExamples// Blacksmith's knowledge
blacksmith.knowledge = [
  {target: "metalworking", generic: 85, specialized: {weaponsmithing: 90}, detailed: {...}},
  {target: "merchant_bob", generic: 60, detailed: {"tax_evasion": {content: "...", confidence: 70}}},
  {target: "local_politics", generic: 30, specialized: {village_council: 50}}
]

// Village guards' collective knowledge
village_guards.knowledge = [
  {target: "combat_tactics", generic: 70, specialized: {village_defense: 85}},
  {target: "bandit_gang", generic: 80, detailed: {"hideout_location": {content: "...", confidence: 90}}}
]Key AdvantagesAsymmetric Relationships: Blacksmith likes merchant_bob (affection: 60), but merchant_bob might be neutral (affection: 20)Group-Individual Dynamics: Individual guard might personally like blacksmith, but village_guards as group might have official neutral stanceKnowledge Specialization: Different entities know different things about the same targets, creating information asymmetries and trading opportunitiesScalable Complexity: Can model everything from personal friendships to international diplomacy using the same 4-axis structureThis creates a unified social simulation framework where relationships and knowledge follow identical organizational patterns, making the system easier to understand and implement.