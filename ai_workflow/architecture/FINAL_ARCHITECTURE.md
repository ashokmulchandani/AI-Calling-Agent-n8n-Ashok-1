# Final Architecture - Call Center AI Workflow

## Architecture Decisions (Confirmed)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Orchestration | N8N Only | Simpler, visual debugging, great for learning |
| File Transfer | Direct API (No Axway) | Greenfield setup, no legacy systems |
| Integrations | HubSpot + Avoma + Pinecone | Full AI-powered call intelligence |

---

## Final System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     CALL CENTER AI WORKFLOW - FINAL DESIGN                       │
│                              (N8N Orchestrated)                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────────┐
                              │    Customer     │
                              │     Call        │
                              └────────┬────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  INGEST LAYER                                                                    │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │
│  │   Aircall   │───▶│ N8N Webhook │───▶│  Validate   │───▶│  Upload to  │       │
│  │   Platform  │    │   Receive   │    │  & Enrich   │    │     S3      │       │
│  └─────────────┘    └─────────────┘    └─────────────┘    └──────┬──────┘       │
│                                                                   │              │
└───────────────────────────────────────────────────────────────────┼──────────────┘
                                                                    │
                                                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  AI PROCESSING LAYER (N8N Workflow)                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │
│  │   Azure     │───▶│  Speaker    │───▶│   Azure     │───▶│   Azure     │       │
│  │   Whisper   │    │  Diarize    │    │   GPT-4     │    │  Embeddings │       │
│  │ (Transcribe)│    │  (Label)    │    │ (Summarize) │    │ (ada-002)   │       │
│  └─────────────┘    └─────────────┘    └─────────────┘    └──────┬──────┘       │
│                                                                   │              │
└───────────────────────────────────────────────────────────────────┼──────────────┘
                                                                    │
                          ┌─────────────────────────────────────────┼───────┐
                          │                                         │       │
                          ▼                                         ▼       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  OUTPUT LAYER                                                                    │
│                                                                                  │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐             │
│  │    HubSpot      │    │     Avoma       │    │    Pinecone     │             │
│  │    CRM          │    │   AI Notes      │    │   Vector DB     │             │
│  │                 │    │                 │    │                 │             │
│  │ • Contact Link  │    │ • Call Summary  │    │ • Semantic      │             │
│  │ • Call Activity │    │ • Key Moments   │    │   Search        │             │
│  │ • Follow-up     │    │ • Action Items  │    │ • Similar Calls │             │
│  │   Tasks         │    │ • Coaching Tips │    │ • Analytics     │             │
│  │ • Timeline      │    │                 │    │                 │             │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘             │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  OBSERVABILITY LAYER                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │
│  │ N8N Logs    │    │ CloudWatch  │    │ Slack       │    │ Dashboard   │       │
│  │ (Execution) │    │ (AWS)       │    │ (Alerts)    │    │ (Metrics)   │       │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘       │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## N8N Workflow Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         N8N WORKFLOW ORCHESTRATION                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   WORKFLOW 1: Call Ingest                                                        │
│   ────────────────────────                                                       │
│   Trigger: Aircall Webhook (call.ended)                                         │
│   Steps: Validate → Download Recording → Upload S3 → Trigger Processing         │
│   Output: Audio + metadata in S3                                                 │
│                                                                                  │
│   WORKFLOW 2: AI Processing                                                      │
│   ─────────────────────────                                                      │
│   Trigger: S3 upload OR HTTP trigger                                            │
│   Steps: Transcribe → Analyze → Embed → Store                                   │
│   Output: Transcript, summary, embeddings in S3                                  │
│                                                                                  │
│   WORKFLOW 3: CRM Sync                                                           │
│   ────────────────────                                                           │
│   Trigger: Processing complete                                                   │
│   Steps: Match Contact → Create Note → Create Task → Notify                      │
│   Output: HubSpot updated with call data                                         │
│                                                                                  │
│   WORKFLOW 4: Avoma Sync                                                         │
│   ─────────────────────                                                          │
│   Trigger: Processing complete                                                   │
│   Steps: Format Notes → Push to Avoma API                                        │
│   Output: AI notes in Avoma                                                      │
│                                                                                  │
│   WORKFLOW 5: Vector Store                                                       │
│   ─────────────────────────                                                      │
│   Trigger: Embeddings generated                                                  │
│   Steps: Format → Upsert to Pinecone                                            │
│   Output: Searchable call transcript vectors                                     │
│                                                                                  │
│   WORKFLOW 6: Error Handler                                                      │
│   ─────────────────────────                                                      │
│   Trigger: Any workflow error                                                    │
│   Steps: Log → Retry (3x) → DLQ → Alert                                         │
│   Output: Error handled or escalated                                             │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Models

### Call Record (S3: processed/{call_id}/analysis.json)

```json
{
  "call_id": "abc123",
  "metadata": {
    "caller_number": "+1234567890",
    "caller_name": "John Doe",
    "agent_id": "agent_001",
    "agent_name": "Jane Smith",
    "direction": "inbound",
    "duration_seconds": 342,
    "started_at": "2024-01-15T10:30:00Z",
    "ended_at": "2024-01-15T10:35:42Z"
  },
  "transcript": {
    "full_text": "Agent: Thank you for calling...",
    "segments": [
      {"speaker": "agent", "start": 0.0, "end": 5.2, "text": "Thank you for calling..."},
      {"speaker": "customer", "start": 5.5, "end": 12.1, "text": "Hi, I'm calling about..."}
    ],
    "word_count": 1234,
    "language": "en"
  },
  "analysis": {
    "summary": "Customer called to inquire about...",
    "customer_intent": "Product inquiry",
    "resolution_status": "resolved",
    "topics": ["product-info", "pricing", "shipping"],
    "sentiment": {"label": "positive", "confidence": 85},
    "action_items": [],
    "entities": ["Product X", "Premium Plan"]
  },
  "integrations": {
    "hubspot_contact_id": "12345",
    "hubspot_engagement_id": "67890",
    "avoma_note_id": "avo_123",
    "pinecone_vector_id": "vec_abc123"
  },
  "processing": {
    "transcribed_at": "2024-01-15T10:36:00Z",
    "analyzed_at": "2024-01-15T10:36:30Z",
    "synced_at": "2024-01-15T10:37:00Z"
  }
}
```

### Pinecone Vector Record

```json
{
  "id": "call_abc123",
  "values": [0.123, -0.456, ...],  // 1536 dimensions
  "metadata": {
    "call_id": "abc123",
    "caller_name": "John Doe",
    "agent_name": "Jane Smith",
    "date": "2024-01-15",
    "summary": "Customer inquiry about Product X",
    "sentiment": "positive",
    "topics": ["product-info", "pricing"],
    "duration_seconds": 342
  }
}
```

---

## Service Dependencies

| Service | Purpose | Required Credentials |
|---------|---------|---------------------|
| **N8N** | Workflow orchestration | Self-hosted or n8n.cloud |
| **Aircall** | Call recording & webhooks | API Key + Webhook Secret |
| **AWS S3** | Audio & data storage | Access Key + Secret Key |
| **Azure OpenAI** | Whisper, GPT-4, Embeddings | API Key + Endpoint |
| **HubSpot** | CRM integration | OAuth or Private App Token |
| **Avoma** | AI meeting notes | API Key |
| **Pinecone** | Vector database | API Key + Environment |
| **Slack** | Notifications | Bot Token |

---

## Implementation Files

```
ai_workflow/
├── architecture/
│   ├── ARCHITECTURE.md           # Initial design
│   ├── ARCHITECTURE_REVIEW.md    # Expert review
│   └── FINAL_ARCHITECTURE.md     # This file
├── n8n/
│   ├── workflows/
│   │   ├── 01_call_ingest.json          # Aircall → S3
│   │   ├── 02_ai_processing.json        # Whisper → GPT-4 → Embed
│   │   ├── 03_hubspot_sync.json         # → HubSpot CRM
│   │   ├── 04_avoma_sync.json           # → Avoma Notes
│   │   ├── 05_pinecone_sync.json        # → Vector DB
│   │   └── 06_error_handler.json        # Error handling
│   ├── credentials/
│   │   └── credentials_template.json
│   └── docker-compose.yml
├── aws/
│   ├── s3_buckets.tf                    # Terraform for S3
│   ├── iam_policies.json
│   └── cloudformation.yaml
├── prompts/
│   ├── call_analysis.md                 # GPT-4 analysis prompt
│   └── ontology_extraction.md           # Entity extraction prompt
├── config/
│   └── .env.example
└── docs/
    ├── SETUP_GUIDE.md
    └── TROUBLESHOOTING.md
```
