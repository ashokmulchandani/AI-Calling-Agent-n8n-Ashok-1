# Call Center AI Workflow Architecture

## Overview

End-to-end automated workflow for processing call center interactions, transcribing audio, generating AI summaries, and syncing with CRM.

## High-Level Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           CALL CENTER AI WORKFLOW                                │
└─────────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   INGEST     │───▶│   TRANSFER   │───▶│   PROCESS    │───▶│   OUTPUT     │
│   LAYER      │    │   LAYER      │    │   LAYER      │    │   LAYER      │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
     │                    │                    │                    │
     ▼                    ▼                    ▼                    ▼
  Aircall            Axway MFT           AWS Step          HubSpot CRM
  Contact Center     S3 Upload           Functions         Avoma Notes
  Call Recording                         Transcription     S3 Archive
                                         Summarization
```

## Detailed Component Architecture

### 1. INGEST LAYER (Call Capture)

```
┌─────────────────────────────────────────────────────────┐
│                    INGEST LAYER                          │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐         ┌──────────────┐              │
│  │   Customer   │────────▶│   Aircall    │              │
│  │   Call       │         │   Platform   │              │
│  └──────────────┘         └──────┬───────┘              │
│                                  │                       │
│                    ┌─────────────┼─────────────┐        │
│                    ▼             ▼             ▼        │
│              ┌─────────┐  ┌───────────┐  ┌─────────┐   │
│              │ Call    │  │ Recording │  │ Webhook │   │
│              │ Metadata│  │ .wav/.mp3 │  │ Trigger │   │
│              └─────────┘  └───────────┘  └─────────┘   │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Components:**
- **Aircall**: Cloud-based phone system capturing calls
- **Webhook Events**: `call.ended`, `call.recorded`
- **Metadata Captured**: caller_id, agent_id, duration, timestamp, direction

### 2. TRANSFER LAYER (Secure File Movement)

```
┌─────────────────────────────────────────────────────────┐
│                   TRANSFER LAYER                         │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐         ┌──────────────┐              │
│  │   Aircall    │────────▶│   Axway MFT  │              │
│  │   Recording  │  SFTP   │   Gateway    │              │
│  └──────────────┘         └──────┬───────┘              │
│                                  │                       │
│                           ┌──────▼───────┐              │
│                           │   AWS S3     │              │
│                           │   Raw Bucket │              │
│                           └──────┬───────┘              │
│                                  │                       │
│                           ┌──────▼───────┐              │
│                           │ S3 Event     │              │
│                           │ Notification │              │
│                           └──────────────┘              │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Components:**
- **Axway MFT**: Secure File Transfer gateway (compliance, encryption)
- **S3 Raw Bucket**: `s3://callcenter-raw-audio/`
- **S3 Event**: Triggers processing pipeline on new object

### 3. PROCESSING LAYER (AI Pipeline)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PROCESSING LAYER                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     AWS STEP FUNCTIONS                               │    │
│  │  ┌───────────────────────────────────────────────────────────────┐  │    │
│  │  │                                                                │  │    │
│  │  │   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐   │  │    │
│  │  │   │ Extract │───▶│Transcribe│───▶│ Analyze │───▶│Summarize│   │  │    │
│  │  │   │ Audio   │    │ (Whisper)│    │(Ontology)│    │ (GPT-4) │   │  │    │
│  │  │   └─────────┘    └─────────┘    └─────────┘    └─────────┘   │  │    │
│  │  │        │              │              │              │         │  │    │
│  │  │        ▼              ▼              ▼              ▼         │  │    │
│  │  │   Audio File     Transcript      Entities       Summary      │  │    │
│  │  │                                  Topics         Action Items │  │    │
│  │  │                                  Sentiment                    │  │    │
│  │  └───────────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     AZURE OPENAI SERVICES                            │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │    │
│  │  │ Whisper API │  │ GPT-4 Turbo │  │ Embeddings  │                  │    │
│  │  │ (STT)       │  │ (Summary)   │  │ (Search)    │                  │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Processing Steps:**

| Step | Service | Input | Output |
|------|---------|-------|--------|
| 1. Extract | Lambda | S3 audio file | Audio stream |
| 2. Transcribe | Azure Whisper / AWS Transcribe | Audio stream | Raw transcript |
| 3. Diarize | Speaker diarization | Transcript | Speaker-labeled text |
| 4. Analyze | GPT-4 + Ontology | Transcript | Entities, topics, sentiment |
| 5. Summarize | GPT-4 | Full transcript | Concise summary + action items |
| 6. Embed | Azure Embeddings | Transcript | Vector for semantic search |

### 4. OUTPUT LAYER (Storage & Integration)

```
┌─────────────────────────────────────────────────────────┐
│                    OUTPUT LAYER                          │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │              PROCESSED OUTPUT                     │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐       │   │
│  │  │Transcript│  │ Summary  │  │ Metadata │       │   │
│  │  │  .json   │  │  .json   │  │  .json   │       │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘       │   │
│  └───────┼─────────────┼─────────────┼──────────────┘   │
│          │             │             │                   │
│          ▼             ▼             ▼                   │
│  ┌──────────────────────────────────────────────────┐   │
│  │        S3 PROCESSED BUCKET                        │   │
│  │   s3://callcenter-processed/                      │   │
│  └──────────────────────────────────────────────────┘   │
│                         │                                │
│          ┌──────────────┼──────────────┐                │
│          ▼              ▼              ▼                │
│   ┌────────────┐ ┌────────────┐ ┌────────────┐         │
│   │  HubSpot   │ │   Avoma    │ │  Vector DB │         │
│   │  CRM Notes │ │  AI Notes  │ │  (Search)  │         │
│   └────────────┘ └────────────┘ └────────────┘         │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

## N8N Orchestration Architecture

N8N serves as the central orchestrator connecting all components:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          N8N WORKFLOW ORCHESTRATION                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   WORKFLOW 1: Call Ingest                                                    │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐                  │
│   │ Aircall │───▶│ Extract │───▶│ Upload  │───▶│ Trigger │                  │
│   │ Webhook │    │ Metadata│    │ to S3   │    │ Process │                  │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘                  │
│                                                                              │
│   WORKFLOW 2: AI Processing                                                  │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐                  │
│   │ S3      │───▶│ Azure   │───▶│ Azure   │───▶│ Store   │                  │
│   │ Trigger │    │ Whisper │    │ GPT-4   │    │ Results │                  │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘                  │
│                                                                              │
│   WORKFLOW 3: CRM Sync                                                       │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐                  │
│   │ Process │───▶│ Match   │───▶│ Update  │───▶│ Notify  │                  │
│   │Complete │    │ Contact │    │ HubSpot │    │ Agent   │                  │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Data Flow Sequence

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         END-TO-END DATA FLOW                                 │
└─────────────────────────────────────────────────────────────────────────────┘

  Customer          Aircall         Axway         AWS            Azure         HubSpot
     │                │               │            │               │              │
     │──Call──────────▶               │            │               │              │
     │                │               │            │               │              │
     │◀───────────────│               │            │               │              │
     │  (Call Ends)   │               │            │               │              │
     │                │               │            │               │              │
     │                │──Recording────▶            │               │              │
     │                │   (SFTP)      │            │               │              │
     │                │               │            │               │              │
     │                │               │──Upload────▶               │              │
     │                │               │  (S3)      │               │              │
     │                │               │            │               │              │
     │                │               │            │──Audio────────▶              │
     │                │               │            │  (Whisper)    │              │
     │                │               │            │               │              │
     │                │               │            │◀──Transcript──│              │
     │                │               │            │               │              │
     │                │               │            │──Transcript───▶              │
     │                │               │            │  (GPT-4)      │              │
     │                │               │            │               │              │
     │                │               │            │◀──Summary─────│              │
     │                │               │            │               │              │
     │                │               │            │──────────────────Update──────▶
     │                │               │            │               │   (CRM)      │
     │                │               │            │               │              │
```

## Technology Stack Summary

| Layer | Component | Technology | Purpose |
|-------|-----------|------------|---------|
| Ingest | Phone System | Aircall | Receive & record calls |
| Ingest | Webhook | N8N | Event-driven triggers |
| Transfer | MFT | Axway | Secure file transfer |
| Transfer | Storage | AWS S3 | Audio file storage |
| Process | Orchestration | AWS Step Functions | Pipeline coordination |
| Process | Transcription | Azure Whisper | Speech-to-text |
| Process | Analysis | Azure OpenAI GPT-4 | Summarization & NLU |
| Process | Embeddings | Azure OpenAI | Semantic search |
| Output | CRM | HubSpot | Contact management |
| Output | Notes | Avoma | AI meeting notes |
| Orchestrate | Workflow | N8N | Connect all systems |

## Security Considerations

1. **Data in Transit**: TLS 1.3 for all API calls, SFTP for file transfer
2. **Data at Rest**: S3 SSE-KMS encryption, Azure encryption
3. **Access Control**: IAM roles, API key rotation, OAuth 2.0
4. **Compliance**: GDPR data handling, call recording consent
5. **Audit**: CloudTrail logging, N8N execution logs
