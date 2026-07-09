# Universal-Map Integration Guide

## Overview

Universal-Map is a cross-platform data and entity mapping tool designed to integrate with the ZQM-Computing ecosystem. This document outlines integration points and implementation strategies.

## Current Status

**Phase**: Placeholder/Design  
**Maturity**: Experimental  
**Priority**: P2 (Long-term)

## Integration Points

### 1. Hermes Agent Integration

**Purpose**: Provide entity mapping and relationship tracking for Hermes agent memory

**Implementation**:
```python
# hermes-config/skills/universal-map/
from universal_map import EntityMapper, RelationshipGraph

class HermesUniversalMapSkill:
    def __init__(self, config_path: str):
        self.mapper = EntityMapper(config_path)
        self.graph = RelationshipGraph()
    
    def map_entity(self, entity_data: dict) -> str:
        """Map an entity and return its universal ID"""
        return self.mapper.register(entity_data)
    
    def link_entities(self, source_id: str, target_id: str, relationship: str):
        """Create relationship between entities"""
        self.graph.add_edge(source_id, target_id, relationship)
```

**Use Cases**:
- Track people, organizations, and their relationships
- Map bug bounty targets across platforms
- Link findings across different security assessments
- Build knowledge graph from wiki entries

### 2. Wiki Integration

**Purpose**: Structured data backing for wiki documentation

**Schema**:
```yaml
# Universal-Map entity schema
entity:
  id: "uuid"
  type: [person, organization, project, vulnerability, tool]
  name: "Display name"
  aliases: ["Alternative names"]
  metadata:
    created: "2026-07-09"
    source: "zqm-auth"
    confidence: 0.95
  
  relationships:
    - type: "works_for"
      target: "org_id"
      since: "2024-01-01"
    - type: "contributed_to"
      target: "project_id"
      role: "maintainer"
```

**Wiki Sync**:
- Entities in Universal-Map auto-generate wiki pages
- Relationships create cross-references
- Updates propagate to wiki via webhook

### 3. ZQM-AI-Council Integration

**Purpose**: Provide structured knowledge base for council deliberations

**Data Flow**:
```
ZQM-AI-Council → Universal-Map → Structured Knowledge
     ↓
  Deliberation → Entity Extraction → Map Update
```

**Implementation**:
```python
# ZQM-AI-Council/utils/knowledge_base.py
from universal_map import KnowledgeGraph

class CouncilKnowledgeBase:
    def __init__(self):
        self.kg = KnowledgeGraph()
    
    def extract_entities(self, deliberation_text: str) -> List[Entity]:
        """Extract entities from council deliberation"""
        # Use LLM to extract entities
        entities = llm.extract_entities(deliberation_text)
        return [self.kg.map_entity(e) for e in entities]
    
    def get_related_entities(self, entity_id: str, depth: int = 2) -> dict:
        """Get entity relationships for context"""
        return self.kg.get_neighborhood(entity_id, depth)
```

### 4. ZQM Bounty Hub Integration

**Purpose**: Map bug bounty targets, programs, and findings

**Entity Types**:
```python
BOUNTY_ENTITIES = {
    "program": {
        "platform": "hackerone|bugcrowd|intigriti|gitlab",
        "handle": "program_slug",
        "scope": ["url", "domain", "ip_range"],
        "rewards": {"low": 50, "medium": 250, "high": 1000}
    },
    "target": {
        "type": "web_app|api|mobile|network",
        "url": "https://...",
        "program_id": "uuid",
        "status": "pending|testing|resolved"
    },
    "finding": {
        "title": "SQL Injection in login",
        "severity": "critical|high|medium|low",
        "target_id": "uuid",
        "reporter": "zqmco",
        "date": "2026-07-09"
    }
}
```

**Workflow**:
1. zqm-auth discovers program → Universal-Map creates entity
2. zqm-bounty-hub runs checks → Universal-Map links findings
3. Wiki auto-documents program details

### 5. Hermes-Config Integration

**Purpose**: Map agent configurations and their relationships

**Entities**:
- Agent instances (hermes-agent, ZQM-AI-Council)
- Skills and their dependencies
- Sessions and conversation threads
- Memories and knowledge fragments

## Data Model

### Core Entities

```python
class Entity:
    id: str                    # UUID
    type: EntityType           # person, org, project, etc.
    name: str                  # Primary name
    aliases: List[str]         # Alternative names
    properties: dict           # Type-specific data
    created_at: datetime
    updated_at: datetime
    source: str                # Which repo created this
    confidence: float          # 0.0-1.0
```

### Relationships

```python
class Relationship:
    id: str
    source_id: str
    target_id: str
    type: str                  # works_for, contributed_to, etc.
    properties: dict
    valid_from: datetime
    valid_to: Optional[datetime]
```

### Graph Operations

```python
class UniversalMap:
    def register_entity(self, entity: Entity) -> str
    def link_entities(self, source: str, target: str, rel: str) -> str
    def get_entity(self, id: str) -> Entity
    def find_entities(self, query: dict) -> List[Entity]
    def get_relationships(self, entity_id: str) -> List[Relationship]
    def get_neighborhood(self, entity_id: str, depth: int) -> dict
    def export_graph(self, format: str) -> str  # json, yaml, dot
```

## Implementation Roadmap

### Phase 1: Core (Current)
- [ ] Define entity schema
- [ ] Implement basic CRUD operations
- [ ] Add JSON/YAML persistence
- [ ] Create CLI interface

### Phase 2: Integration
- [ ] Hermes-agent skill wrapper
- [ ] Wiki sync webhook
- [ ] ZQM-AI-Council entity extraction
- [ ] Bounty hub target mapping

### Phase 3: Advanced
- [ ] Graph database backend (Neo4j)
- [ ] Query language (Cypher-like)
- [ ] Visualization API
- [ ] Real-time sync across repos

## API Design

### REST API

```
GET    /api/v1/entities/{id}
POST   /api/v1/entities
PUT    /api/v1/entities/{id}
DELETE /api/v1/entities/{id}

GET    /api/v1/relationships/{id}
POST   /api/v1/relationships
GET    /api/v1/graph/neighborhood/{entity_id}?depth=2

POST   /api/v1/search
{
  "type": "person",
  "properties": {"name": "John Doe"}
}
```

### Python SDK

```python
from universal_map import UniversalMap

um = UniversalMap("config.yaml")

# Register entity
person_id = um.register_entity({
    "type": "person",
    "name": "John Doe",
    "aliases": ["JD", "johndoe"],
    "properties": {"email": "john@example.com"}
})

# Create relationship
um.link_entities(
    source=person_id,
    target="org_zqm_computing",
    relationship="works_for"
)

# Query
results = um.find_entities({"type": "person", "name": "John"})
neighborhood = um.get_neighborhood(person_id, depth=2)
```

## Storage

### File-Based (Current)
```
~/.universal-map/
├── entities/
│   ├── {entity_id}.json
│   └── index.json
├── relationships/
│   ├── {rel_id}.json
│   └── index.json
└── config.yaml
```

### Database (Future)
- Neo4j for graph queries
- PostgreSQL for metadata
- Redis for caching

## Security

- Entity data is private by default
- Sensitive relationships encrypted
- Access control per entity type
- Audit log for all changes

## Next Steps

1. **Design Phase**: Finalize entity schema and relationship types
2. **Prototype**: Build basic CRUD with file storage
3. **Integration**: Connect to Hermes agent
4. **Testing**: Populate with real ZQM data
5. **Deployment**: Set up graph database

## Related Documents

- [wiki/MATRIX.md](../wiki/MATRIX.md) - System relationships
- [hermes-config/STABILITY.md](../hermes-config/STABILITY.md) - Config stability
- [ZQM-AI-Council/README.md](../ZQM-AI-Council/README.md) - Council system