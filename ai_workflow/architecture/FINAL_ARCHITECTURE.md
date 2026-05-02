# Final Architecture - Call Center AI Workflow

## Architecture Decisions (Confirmed)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Orchestration | n8n Cloud | Visual debugging, webhook support, no infra to manage |
| File Transfer | Direct API (Aircall → S3) | Greenfield setup, no legacy systems |
| Integrations | HubSpot + Pinecone + Telegram | Full AI-powered call intelligence |
| Avoma | Native Aircall integration | Auto-syncs calls, API is read-only |
| Notifications | Telegram | Simpler than Slack, instant mobile alerts |

---

## Final System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     CALL CENTER AI WORKFLOW - PRODUCTION                     │
│                           (n8n Cloud Orchestrated)                           │
└─────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────────┐
                              │   Customer Call  │
                              └────────┬────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  WORKFLOW 01 - INGEST LAYER                                                 │
│  Aircall webhook (call.ended) → Filter → Extract Metadata                   │
│  → Has Recording? → Download → Upload S3 (raw) → Save Metadata             │
│  → Trigger Workflow 02 → Telegram notification                              │
└────────────────────────────────────────────────────────┬────────────────────┘
                                                         │
                                                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  WORKFLOW 02 - AI PROCESSING LAYER                                          │
│  Download from S3 → Azure Whisper (transcription)                           │
│  → Process Transcript → GPT-4o-mini (analysis) → Combine Results            │
│  → text-embedding-3-small (1536-dim) → Save to S3 (processed)              │
│  → Trigger Workflow 03 + 05                                                 │
└──────────────────────┬──────────────────────────┬──────────────────────────┘
                       │                          │
                       ▼                          ▼
┌──────────────────────────────────┐  ┌──────────────────────────────────────┐
│  WORKFLOW 03 - CRM SYNC          │  │  WORKFLOW 05 - VECTOR SYNC           │
│  Download analysis from S3       │  │  Prepare flat metadata               │
│  → Search HubSpot contact        │  │  → Upsert to Pinecone               │
│  → Create note (HTTP API)        │  │  → call_transcripts namespace        │
│  → Needs follow-up? → Task       │  │                                      │
│  → Telegram notification         │  │                                      │
└──────────────────────────────────┘  └──────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  WORKFLOW 06 - ERROR HANDLER                                                │
│  Any workflow failure → Enrich Error → Should Retry?                        │
│  → YES (count < 3): Retry original workflow                                 │
│  → NO: Save to S3 DLQ → Telegram alert → Critical? → Page On-Call          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Service Dependencies

| Service | Purpose | Credentials |
|---------|---------|-------------|
| **n8n Cloud** | Workflow orchestration | Cloud account |
| **Aircall** | Call recording + webhooks | API ID + Token |
| **AWS S3** | Audio + data storage | IAM Access Key + Secret |
| **Azure OpenAI (Whisper)** | Transcription | API Key (separate resource) |
| **Azure OpenAI (GPT-4 + Embeddings)** | Analysis + vectors | API Key |
| **HubSpot** | CRM | Private App Token |
| **Pinecone** | Vector database | API Key |
| **Telegram** | Notifications + alerts | Bot Token + Chat ID |
| **Avoma** | AI meeting notes | Native Aircall sync (no API) |

---

## Data Models

### Call Record (S3: processed/{call_id}/full_analysis.json)

```json
{
  "call_id": "12345",
  "transcript": {
    "full_text": "Agent: Thank you for calling...",
    "segments": [
      {"id": 0, "start": 0.0, "end": 5.2, "text": "Thank you for calling...", "confidence": 0.95}
    ],
    "word_count": 1234,
    "duration": 120.5,
    "language": "en"
  },
  "analysis": {
    "summary": "Customer called to inquire about...",
    "customer_intent": "Product inquiry",
    "resolution_status": "resolved",
    "topics": ["product-info", "pricing"],
    "sentiment": {"label": "positive", "confidence": 85},
    "action_items": ["Send follow-up email"],
    "entities": ["Product X", "Premium Plan"]
  },
  "embedding": {
    "vector": [0.123, -0.456, ...],
    "model": "text-embedding-3-small",
    "dimensions": 1536,
    "generated_at": "2024-01-15T10:36:30Z"
  },
  "metadata": {
    "direction": "inbound",
    "duration": 120,
    "caller_number": "+61400000000",
    "agent_name": "Agent Name"
  },
  "processing": {
    "transcribed_at": "2024-01-15T10:36:00Z",
    "analyzed_at": "2024-01-15T10:36:30Z",
    "model_used": "gpt-4o",
    "whisper_model": "whisper-1",
    "workflow_version": "1.0.0"
  }
}
```

### Pinecone Vector Record

```json
{
  "id": "call_12345",
  "values": [0.123, -0.456, ...],
  "metadata": {
    "call_id": "12345",
    "summary": "Customer inquiry about Product X",
    "sentiment": "positive",
    "sentiment_score": 85,
    "resolution_status": "resolved",
    "topics": ["product-info", "pricing"],
    "duration": 120,
    "direction": "inbound",
    "agent_name": "Agent Name",
    "processed_at": "2024-01-15T10:36:30Z"
  }
}
```

Note: Pinecone metadata must be flat — no nested objects. Sentiment is split into `sentiment` (string) and `sentiment_score` (number).

---

## Error Handling

### DLQ Record (S3: errors/dlq/{error_id}.json)

```json
{
  "error_id": "ERR-1700000000000-abc123",
  "timestamp": "2024-01-15T10:40:00Z",
  "source_workflow": "02 - AI Processing Workflow",
  "source_node": "Azure Whisper",
  "error_message": "API timeout after 30s",
  "error_type": "timeout",
  "call_id": "12345",
  "severity": "high",
  "retry_count": 3,
  "max_retries": 3,
  "original_payload": {}
}
```

### Severity Defaults

| Source Workflow | Default Severity |
|----------------|-----------------|
| 02 - AI Processing | high |
| 03 - CRM Sync | medium |
| 05 - Pinecone Vector Sync | low |

---

## Known Limitations

- **n8n Cloud free plan**: No environment variables — all values hardcoded in workflow JSONs
- **n8n import quirks**: Code nodes import as templates, IF nodes may need v2 upgrade, HTTP methods may reset to GET
- **Avoma API**: Read-only (`/v1/meetings/` is GET). Relies on native Aircall integration for sync
- **HubSpot Engagement node**: Doesn't support NOTE type — uses direct HTTP API call instead
- **Pinecone metadata**: Must be flat values only (no nested objects)
- **Azure Whisper**: On separate resource from GPT-4/Embeddings — requires separate API key
