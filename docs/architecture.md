# Infrastructure & Architecture

## Deployment Overview

```
┌─────────────────────────────────────┐         ┌──────────────────────────────────┐
│         Dokploy Server              │         │     Apple M2 Mac (64GB RAM)      │
│                                     │         │                                  │
│  ┌───────────────────────┐          │         │  ┌────────────────────────────┐  │
│  │  n8n Workflow Engine   │          │         │  │  Ollama                    │  │
│  │  (Merge1 → HTTP POST) │──────────┼── HTTP ─┼──│  ├─ qwen3-embedding:8      │  │
│  └───────────────────────┘          │         │  │  ├─ dengcao/Qwen3-Reranker  │  │
│                                     │         │  │  └─ gemma4:12b              │  │
│  ┌───────────────────────┐          │         │  └────────────────────────────┘  │
│  │  FastAPI Service       │          │         │                                  │
│  │  (research-areas-      │──────────┼── HTTP ─┼──┌────────────────────────────┐  │
│  │   classify)            │          │         │  │  Qdrant                    │  │
│  └───────────────────────┘          │         │  │  (research_taxonomy)       │  │
│         │                           │         │  │  Port: 6333 (REST)         │  │
│         │ HTTP (outbound)           │         │  │  Port: 6334 (gRPC)         │  │
│         ▼                           │         │  └────────────────────────────┘  │
│  ┌───────────────────────┐          │         │                                  │
│  │  External APIs         │          │         └──────────────────────────────────┘
│  │  ├─ OpenAlex           │          │
│  │  ├─ Semantic Scholar   │          │
│  │  ├─ ORCID (v3.0)      │          │
│  │  └─ ResearchNodes      │          │
│  └───────────────────────┘          │
│                                     │
└─────────────────────────────────────┘
```

## Request Flow

```
1. n8n Merge1 node produces merged author payload
2. n8n sends POST /research-areas/classify to FastAPI (on Dokploy)
3. FastAPI extracts evidence from the payload
4. FastAPI concurrently calls:
   a. OpenAlex Paper Search (max 3 papers)
   b. Semantic Scholar Paper Search (max 3 papers)
   c. OpenAlex Author Search (by name/institution)
   d. ORCID Lookup (if ORCID is known or discovered)
5. FastAPI builds author_details from input + enrichment
6. FastAPI sends evidence text to Ollama (qwen3-embedding:8) for embedding
7. FastAPI queries Qdrant (on Mac) with hybrid vectors
8. FastAPI sends candidates to Ollama (Qwen3-Reranker-4B) for reranking
9. FastAPI sends top candidates + evidence to Ollama (gemma4:12b) for selection
10. FastAPI assembles response and returns to n8n
11. n8n merges response into author profile and calls CRM UpdateAuthor
```

## Environment Variables

The FastAPI service requires these environment variables:

```bash
# Ollama (on Mac M2)
OLLAMA_BASE_URL=http://<mac-ip>:11434

# Qdrant (on Mac M2)
QDRANT_URL=http://<mac-ip>:6333
QDRANT_COLLECTION=research_taxonomy

# ORCID API (Public API v3.0)
ORCID_CLIENT_ID=APP-2S1FCSACCQV9GFYK
ORCID_CLIENT_SECRET=31a25db3-af70-4d91-bc1a-3f7985c7581d

# ResearchNodes API
RESEARCHNODES_API_URL=https://uni.researchnodes.com
RESEARCHNODES_API_SECRET=<api-secret>

# OpenAlex (polite pool — optional but recommended)
OPENALEX_EMAIL=<your-email-for-polite-pool>

# Redis (optional, for caching)
REDIS_URL=redis://<redis-host>:6379/0
```

## Model Details

### Ollama Models

| Model | Ollama Tag | Purpose | VRAM |
|---|---|---|---|
| Qwen3-Embedding-8B | `qwen3-embedding:8` | Dense embedding (512 + 4096 dim via MRL) + sparse BM42 | ~8GB |
| Qwen3-Reranker-4B | `dengcao/Qwen3-Reranker-4B:Q5_K_M` | Cross-encoder reranking | ~4GB |
| Gemma4-12B | `gemma4:12b` | LLM selection + data cleaning | ~12GB |

**Total estimated VRAM:** ~24GB (well within 64GB unified memory on M2)

### Qdrant

- **Version:** Latest stable
- **Storage:** Persistent (disk-backed with in-memory index)
- **Collection:** `research_taxonomy` with 3 vector types (dense_512, dense_4096, bm42)
- **Expected size:** ~10K-50K taxonomy records (fields + subfields + keywords)

## Network Requirements

- The Mac M2 must be accessible from the Dokploy server over the network.
- Ollama default port: `11434`
- Qdrant REST port: `6333`, gRPC port: `6334`
- Both services must be bound to `0.0.0.0` (not `127.0.0.1`) to accept external connections.
- Consider using a VPN or SSH tunnel if the machines are on different networks.

## Scaling Notes

- The service processes **one author at a time** (single-threaded per request from n8n).
- Ollama handles concurrent requests via its built-in queue.
- Qdrant handles concurrent queries natively.
- The main bottleneck is the LLM inference time on Ollama (~2-5 seconds for gemma4:12b).
- External API calls (OpenAlex, Semantic Scholar, ORCID) are the second bottleneck, mitigated by `asyncio.gather()` concurrency.
