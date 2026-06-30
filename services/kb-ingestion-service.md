# KB Ingestion Service

**Runtime:** AWS Lambda (Python) · SQS subscriber + S3 event trigger  
**Domain:** Knowledge Management

## Responsibility

Keeps the RAG knowledge base up to date. Chunks new documents, generates embeddings via Bedrock, and upserts them into the Supabase pgvector table.

## Trigger Sources

| Source | Trigger | Content |
|--------|---------|---------|
| S3 (`kb-documents/`) | S3 Event Notification → SQS | FAQs, policies, guides (manual upload) |
| Supabase `menu_items` | DB trigger / Edge Function → SQS | Menu updates (auto) |
| Supabase `restaurants` | DB trigger / Edge Function → SQS | Restaurant info updates (auto) |

## Ingestion Pipeline

```
Trigger (S3 / DB event)
  → SQS Standard (kb-ingestion-queue)
  → KB Ingestion Lambda
  1. Load source document
  2. Chunk text  —  500 tokens, 50-token overlap (sliding window)
  3. Batch embed  —  Bedrock Titan Embed Text v2  →  1536-dim vectors
  4. Upsert to Supabase pgvector
     ON CONFLICT (source_id) DO UPDATE  (replaces stale chunks)
```

## Knowledge Base Content Types

| `source_type` | Content | Update Mode |
|---------------|---------|-------------|
| `faq` | Delivery times, refund policy, payment Q&A | Manual upload |
| `menu` | All restaurant menus + item descriptions | Auto (menu update) |
| `policy` | Courier policy, operational procedures | Manual upload |
| `restaurant_info` | Restaurant descriptions, hours, cuisine | Auto (profile update) |
| `guide` | How-to guides for all user roles | Manual upload |

## Data Owned

| Store | Table | Access |
|-------|-------|--------|
| Supabase pgvector | `knowledge_chunks` | Write (upsert) |
| S3 | `kb-documents/` | Read (source documents) |

## Integrations

- **AWS Bedrock** — Titan Embed Text v2 for vector generation
- **SQS Standard** (`kb-ingestion-queue`) — event queue from S3 and DB triggers
- **Supabase pgvector** — HNSW-indexed `knowledge_chunks` table consumed by AI Chat Service

## Concurrency

| Baseline | Burst |
|----------|-------|
| 5 concurrent | 10 concurrent |

Low concurrency is fine — ingestion is not on the customer critical path.
