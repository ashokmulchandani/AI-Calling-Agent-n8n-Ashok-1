# AI Call Centre Intelligence System

An AI-powered call centre intelligence pipeline built with n8n workflows, Azure OpenAI, AWS S3, HubSpot CRM, Pinecone vector search, and Telegram notifications.

## Architecture

```
Aircall (Phone) → n8n Workflows → Azure OpenAI (Whisper + GPT-4 + Embeddings)
                                 → AWS S3 (Storage)
                                 → HubSpot (CRM)
                                 → Pinecone (Vector Search)
                                 → Telegram (Notifications)
```

### Workflow Pipeline

```
┌──────────────┐    ┌──────────────────┐    ┌─────────────┐
│ 01 Call      │───▶│ 02 AI Processing │───▶│ 03 CRM Sync │
│ Ingest       │    │                  │    │ (HubSpot)   │
│ (Aircall→S3) │    │ Whisper→GPT-4→   │    └─────────────┘
└──────────────┘    │ Embeddings→S3    │───▶┌─────────────┐
                    └──────────────────┘    │ 05 Pinecone  │
                                           │ Vector Sync  │
                    ┌──────────────────┐    └─────────────┘
                    │ 06 Error Handler │
                    │ Retry→DLQ→Alert  │
                    └──────────────────┘
```

| Workflow | Purpose | Trigger |
|----------|---------|---------|
| **01 - Call Ingest** | Receives Aircall webhooks, downloads recordings, uploads to S3 | Aircall webhook (`call.ended`) |
| **02 - AI Processing** | Transcribes (Whisper), analyzes (GPT-4), generates embeddings | HTTP trigger from Workflow 01 |
| **03 - CRM Sync** | Creates HubSpot notes and follow-up tasks | HTTP trigger from Workflow 02 |
| **04 - Avoma Sync** | Skipped — Avoma auto-syncs via native Aircall integration | N/A |
| **05 - Pinecone Vector Sync** | Upserts call embeddings to Pinecone for semantic search | HTTP trigger from Workflow 02 |
| **06 - Error Handler** | Retries failed workflows (3x), saves to DLQ, alerts via Telegram | HTTP trigger from any workflow |

## Services & Infrastructure

| Service | Purpose | Details |
|---------|---------|---------|
| **n8n Cloud** | Workflow orchestration | `https://aiamfspr.app.n8n.cloud` |
| **Aircall** | Phone system + webhooks | Webhook on `call.ended` events |
| **AWS S3** | Audio + processed data storage | 2 buckets (raw audio, processed transcripts) |
| **Azure OpenAI** | AI models | Whisper (transcription), GPT-4o-mini (analysis), text-embedding-3-small (vectors) |
| **HubSpot** | CRM | Contact lookup, note creation, task creation |
| **Pinecone** | Vector database | 1536-dim cosine index, `call_transcripts` namespace |
| **Telegram** | Notifications | Pipeline status + error alerts |

### Azure OpenAI Deployments

| Model | Resource | Deployment Name |
|-------|----------|-----------------|
| Whisper | `ashok-mnuduagy-eastus2` | `whisper-1` |
| GPT-4o-mini | `testashok1` | `testashok1-gpt-4o-mini` |
| text-embedding-3-small | `testashok1` | `testashok1-noble-oak-text-embedding-3-small` |

### S3 Buckets

| Bucket | Purpose | Key Structure |
|--------|---------|---------------|
| `callcenter-raw-audio-prod-1-ashok` | Raw call recordings | `recordings/{call_id}.wav` + `metadata/{call_id}.json` |
| `callcenter-processed-transcripts-prod-1-ashok` | Processed analysis + DLQ | `processed/{call_id}/full_analysis.json`, `errors/dlq/{error_id}.json` |

### Pinecone Index

- **Index**: `call-transcripts`
- **Dimensions**: 1536 (text-embedding-3-small)
- **Metric**: Cosine
- **Namespace**: `call_transcripts`
- **Metadata**: Flat values only (string, number, boolean, list of strings)

## Setup

### Prerequisites

- n8n Cloud account (or self-hosted n8n)
- AWS account with S3 access
- Azure OpenAI resource with Whisper, GPT-4, and Embedding deployments
- HubSpot account with private app
- Pinecone account with index created
- Aircall account
- Telegram bot

### 1. Configure Credentials

Copy the example env file and fill in your values:

```bash
cp ai_workflow/config/.env.example ai_workflow/config/.env
```

> **Note**: n8n Cloud free plan does not support environment variables. All values are hardcoded in the workflow JSON files. The `.env` file serves as a credential reference.

### 2. Import Workflows to n8n

Import each JSON file from `ai_workflow/n8n/workflows/` into n8n:

1. Go to n8n → Workflows → Import from File
2. Import all 6 workflow files
3. Configure credentials (AWS, Telegram, HTTP Header Auth for Azure)
4. Activate all workflows

### 3. Register Aircall Webhook

```bash
curl -X POST https://api.aircall.io/v1/webhooks \
  -u "<api_id>:<api_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://<your-n8n>/webhook/aircall-webhook",
    "events": ["call.ended"]
  }'
```

## n8n Import Notes

When importing workflow JSONs into n8n Cloud, be aware of these known issues:

- **Code nodes** import with default template code (`myNewField = 1`). You must manually paste the correct code from the JSON's `jsCode` field into each Code node.
- **IF nodes** v1 format may fail on n8n Cloud. The JSONs use v2 format, but if issues occur, recreate the IF node manually.
- **HTTP Request nodes** may default to GET method on import. Verify POST method is set where needed.
- **Node names** may get numbers appended on import (e.g., `Process Transcript1`). Code nodes referencing other nodes by name (e.g., `$('Extract Body')`) must match exactly.
- **Webhook POST data** is nested under `$json.body` in n8n Cloud. Each workflow has an "Extract Body" Code node to handle this.
- **S3 expression mode** can embed a literal `=` prefix in field values. If S3 paths start with `=`, clear the field and retype.

## Data Flow

### Call Processing Pipeline

```
1. Aircall call ends → webhook fires
2. Workflow 01: Download recording → Upload to S3 (raw bucket)
3. Workflow 02: Download from S3 → Whisper transcription → GPT-4 analysis
   → Generate embedding → Save full analysis to S3 (processed bucket)
   → Trigger Workflow 03 + 05
4. Workflow 03: Download analysis from S3 → Search HubSpot contact
   → Create note → Create follow-up task (if needed) → Telegram notification
5. Workflow 05: Extract embedding + metadata → Upsert to Pinecone
```

### Error Handling

```
Any workflow failure → Workflow 06:
  → retry_count < 3? → Retry original workflow
  → retries exhausted? → Save to S3 DLQ → Telegram alert
  → severity = critical? → Additional "Page On-Call" Telegram alert
```

## Project Structure

```
ai_workflow/
├── architecture/
│   ├── ARCHITECTURE.md              # Initial design
│   ├── ARCHITECTURE_REVIEW.md       # Expert review
│   └── FINAL_ARCHITECTURE.md        # Final design document
├── config/
│   ├── .env                         # Credentials (gitignored)
│   └── .env.example                 # Template
├── n8n/
│   └── workflows/
│       ├── 01 - Call Ingest Workflow.json
│       ├── 02 - AI Processing Workflow.json
│       ├── 03 - CRM Sync Workflow.json
│       ├── 04 - Avoma Sync Workflow.json
│       ├── 05 - Pinecone Vector Sync Workflow.json
│       └── 06 - Error Handler Workflow.json
└── prompts/
    ├── call_analysis_prompt.md      # GPT-4 analysis prompt
    └── ontology_extraction_prompt.md # Entity extraction prompt
```

## Testing

### Simulate a call webhook

```bash
curl -X POST https://<your-n8n>/webhook/aircall-webhook \
  -H "Content-Type: application/json" \
  -d '{
    "event": "call.ended",
    "data": {
      "id": 99999,
      "direction": "inbound",
      "duration": 120,
      "raw_digits": "+61400000000",
      "recording": "https://example.com/recording.wav",
      "started_at": 1700000000,
      "ended_at": 1700000120,
      "user": { "name": "Test Agent" }
    }
  }'
```

### Test error handler

```bash
curl -X POST https://<your-n8n>/webhook/error-handler \
  -H "Content-Type: application/json" \
  -d '{
    "source_workflow": "02 - AI Processing Workflow",
    "source_node": "Azure Whisper",
    "error_message": "API timeout",
    "call_id": "99999",
    "severity": "high",
    "retry_eligible": true,
    "retry_count": 0
  }'
```

## License

Private — Noble Oak internal use.
