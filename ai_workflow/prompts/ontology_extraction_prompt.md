# Ontology & Entity Extraction Prompt

## Purpose
Advanced entity extraction for building a knowledge graph of call center interactions. Use this when you need deeper entity relationships beyond basic analysis.

---

## System Prompt

```
You are a knowledge graph specialist extracting structured entities and relationships from call center transcripts. Your goal is to identify entities, their types, attributes, and relationships to enable semantic search and analytics.

EXTRACTION PRINCIPLES:
1. Extract only explicitly mentioned entities
2. Normalize entity names (e.g., "iPhone 15" and "iPhone fifteen" → "iPhone 15")
3. Capture relationships between entities
4. Include confidence scores for uncertain extractions
5. Preserve temporal references when mentioned
```

---

## User Prompt Template

```
Extract entities and relationships from this call transcript for knowledge graph construction.

## TRANSCRIPT
{transcript_text}

## EXTRACTION SCHEMA

Return a JSON object with the following structure:

{
  "entities": [
    {
      "id": "entity_1",
      "text": "Original text mention",
      "normalized": "Normalized form",
      "type": "PRODUCT | SERVICE | PERSON | ORGANIZATION | LOCATION | DATE | MONEY | ISSUE | FEATURE | PLAN",
      "confidence": 0.95,
      "attributes": {
        "key": "value"
      },
      "mentions": [
        {"start": 0, "end": 10, "context": "surrounding text"}
      ]
    }
  ],

  "relationships": [
    {
      "source_id": "entity_1",
      "target_id": "entity_2",
      "type": "HAS_ISSUE | PURCHASED | INQUIRED_ABOUT | USES | COMPARED_TO | UPGRADED_FROM | MENTIONED_WITH",
      "confidence": 0.85,
      "evidence": "Text that supports this relationship"
    }
  ],

  "topics": [
    {
      "name": "Topic name",
      "relevance": 0.9,
      "keywords": ["related", "keywords"]
    }
  ],

  "temporal_references": [
    {
      "text": "last month",
      "normalized": "2024-12",
      "type": "RELATIVE | ABSOLUTE",
      "context": "What happened at this time"
    }
  ],

  "call_classification": {
    "primary_category": "SALES | SUPPORT | BILLING | COMPLAINT | INQUIRY | FEEDBACK",
    "subcategories": ["specific", "subcategories"],
    "complexity": "LOW | MEDIUM | HIGH",
    "urgency": "LOW | MEDIUM | HIGH | CRITICAL"
  }
}
```

---

## Entity Type Definitions

| Type | Description | Examples |
|------|-------------|----------|
| PRODUCT | Physical or digital products | iPhone 15, Software X, Widget Pro |
| SERVICE | Services offered | Cloud Hosting, Premium Support, Installation |
| PERSON | Named individuals | John Smith, Dr. Johnson |
| ORGANIZATION | Companies, departments | Acme Corp, Sales Team, Partner Inc |
| LOCATION | Physical or virtual locations | New York office, US region, Cloud Zone A |
| DATE | Time references | January 2024, last week, Q1 |
| MONEY | Financial amounts | $99/month, 500 dollars, 20% discount |
| ISSUE | Problems or complaints | Login error, Slow performance, Missing feature |
| FEATURE | Product/service features | Auto-save, Dark mode, API access |
| PLAN | Subscription/pricing plans | Basic, Premium, Enterprise |

---

## Relationship Type Definitions

| Type | Description | Example |
|------|-------------|---------|
| HAS_ISSUE | Entity experiencing problem | Customer HAS_ISSUE Login error |
| PURCHASED | Bought relationship | Customer PURCHASED Premium Plan |
| INQUIRED_ABOUT | Asked about | Customer INQUIRED_ABOUT Enterprise features |
| USES | Currently using | Customer USES Basic Plan |
| COMPARED_TO | Comparison made | Premium COMPARED_TO Basic |
| UPGRADED_FROM | Previous version/plan | Premium UPGRADED_FROM Basic |
| MENTIONED_WITH | Co-occurrence | Feature A MENTIONED_WITH Feature B |

---

## Example Output

```json
{
  "entities": [
    {
      "id": "ent_1",
      "text": "Premium Plan",
      "normalized": "Premium Plan",
      "type": "PLAN",
      "confidence": 0.98,
      "attributes": {
        "tier": "premium",
        "billing": "monthly"
      },
      "mentions": [
        {"start": 145, "end": 157, "context": "interested in the Premium Plan"}
      ]
    },
    {
      "id": "ent_2",
      "text": "cloud storage",
      "normalized": "Cloud Storage",
      "type": "FEATURE",
      "confidence": 0.95,
      "attributes": {
        "capacity": "100GB"
      },
      "mentions": [
        {"start": 234, "end": 247, "context": "100GB of cloud storage"}
      ]
    },
    {
      "id": "ent_3",
      "text": "Basic Plan",
      "normalized": "Basic Plan",
      "type": "PLAN",
      "confidence": 0.97,
      "attributes": {
        "tier": "basic"
      },
      "mentions": [
        {"start": 89, "end": 99, "context": "currently on Basic Plan"}
      ]
    }
  ],

  "relationships": [
    {
      "source_id": "ent_1",
      "target_id": "ent_2",
      "type": "MENTIONED_WITH",
      "confidence": 0.90,
      "evidence": "Premium Plan includes 100GB of cloud storage"
    },
    {
      "source_id": "ent_1",
      "target_id": "ent_3",
      "type": "COMPARED_TO",
      "confidence": 0.88,
      "evidence": "Customer compared Premium Plan features to their current Basic Plan"
    }
  ],

  "topics": [
    {
      "name": "Plan Upgrade",
      "relevance": 0.95,
      "keywords": ["upgrade", "premium", "features", "pricing"]
    },
    {
      "name": "Storage",
      "relevance": 0.75,
      "keywords": ["cloud storage", "capacity", "files"]
    }
  ],

  "temporal_references": [
    {
      "text": "next billing cycle",
      "normalized": "2024-02-01",
      "type": "RELATIVE",
      "context": "Upgrade will take effect next billing cycle"
    }
  ],

  "call_classification": {
    "primary_category": "SALES",
    "subcategories": ["upsell", "plan-comparison"],
    "complexity": "LOW",
    "urgency": "LOW"
  }
}
```

---

## Usage Notes

1. **When to use this vs basic analysis**: Use ontology extraction when building search indexes, analytics dashboards, or knowledge bases. Use basic analysis for simple CRM updates.

2. **Performance**: This prompt generates more tokens. Consider using it selectively for high-value calls or batch processing.

3. **Post-processing**: The extracted entities can be:
   - Stored in a graph database (Neo4j)
   - Indexed in Pinecone with relationships as metadata
   - Used for trend analysis across calls
