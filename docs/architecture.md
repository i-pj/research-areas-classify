# Infrastructure & Architecture

## Deployment Overview

```
┌─────────────────────────────────────┐         ┌─────────────────────────────────────┐
│         Dokploy Server (VPS)        │         │      Apple M2 Mac (64GB RAM)        │
│                                     │         │                                     │
│  ┌───────────────────────┐          │         │  ┌───────────────────────────────┐  │
│  │  n8n Workflow Engine   │          │         │  │  Ollama                       │  │
│  │  (Merge1 → HTTP POST) │───────┐  │         │  ├─ qwen3-embedding:8         │  │
│  └───────────────────────┘       │  │         │  └─ gemma4:12b                │  │
│                                  │  │         │  └───────────────────────────────┘  │
│  ┌───────────────────────┐       │  │         │                                     │
│  │  FastAPI Service       │◄──────┘  │         │  ┌───────────────────────────────┐  │
│  │  (Granian, Hishel,     │─────────┼─Tailnet─┼─►│  Qdrant                       │  │
│  │   LiteLLM Router)      │         │         │  │  (research_taxonomy)          │  │
│  │  ├─ FastEmbed (ColBERT)│─────────┼─Tailnet─┼─►│  Port: 6333 (REST)            │  │
│  │  └─ FastEmbed (BM25)   │         │         │  └───────────────────────────────┘  │
│  └───────────────────────┘          │         │                                     │
│         │                           │         └─────────────────────────────────────┘
│         │ HTTP (outbound)           │
│         ▼                           │
│  ┌───────────────────────┐          │
│  │  External APIs         │          │
│  │  ├─ OpenAlex           │          │
│  │  ├─ Semantic Scholar   │          │
│  │  ├─ ORCID (v3.0)      │          │
│  │  └─ ResearchNodes      │          │
│  └───────────────────────┘          │
│                                     │
│  ┌───────────────────────┐          │
│  │  Redis                 │          │
│  │  (Cache & Locks)       │          │
│  └───────────────────────┘          │
└─────────────────────────────────────┘
```

## Request Flow

```
1. n8n Merge1 node produces merged author payload
2. n8n sends POST /research-areas/classify to FastAPI (on Dokploy)
3. FastAPI extracts evidence (Smart Tiering)
4. Hishel checks Redis cache for existing OpenAlex/S2 responses
5. FastAPI concurrently calls (via asyncio.gather):
   a. OpenAlex Paper Search (max 3 papers)
   b. Semantic Scholar Paper Search (max 3 papers)
   c. OpenAlex Author Search
   d. ORCID Lookup
6. Hishel stores responses in Redis
7. FastAPI builds author_details
8. FastAPI sends evidence to Ollama (qwen3-embedding:8) for dense embedding
9. FastEmbed generates BM25 sparse + ColBERT multivectors locally
10. FastAPI queries Qdrant with 3-stage hybrid: 
    prefetch(dense_512 + bm25) → RRF fusion → ColBERT MaxSim rescore
11. LiteLLM routes to Ollama (gemma4:12b) or OpenRouter for taxonomy selection
12. FastAPI assembles response and returns to n8n
13. n8n merges response into author profile and calls CRM UpdateAuthor
```

## Environment Variables

The FastAPI service uses `pydantic-settings`. Create a `.env` file referencing the MagicDNS hostnames assigned by Tailscale:

```bash
# --- ASGI Server ---
GRANIAN_WORKERS=4
GRANIAN_PORT=8000

# --- Ollama ---
OLLAMA_BASE_URL=http://<mac-hostname>.tail<netname>.ts.net:11434

# --- Qdrant ---
QDRANT_URL=http://<mac-hostname>.tail<netname>.ts.net:6333
QDRANT_API_KEY=<qdrant-api-key>
QDRANT_COLLECTION=research_taxonomy

# --- OpenAlex (REQUIRED - polite pool deprecated Feb 2026) ---
OPENALEX_API_KEY=<your-api-key-from-openalex.org/settings/api>

# --- ORCID ---
ORCID_CLIENT_ID=<orcid-client-id>
ORCID_CLIENT_SECRET=<orcid-client-secret>

# --- ResearchNodes ---
RESEARCHNODES_API_URL=https://uni.researchnodes.com
RESEARCHNODES_API_SECRET=<api-secret>

# --- LiteLLM (optional cloud fallback) ---
OPENROUTER_API_KEY=<openrouter-key>

# --- Redis ---
REDIS_URL=redis://localhost:6379/0

# --- Observability ---
LOGFIRE_TOKEN=<logfire-token>

# --- FastEmbed ---
FASTEMBED_CACHE_PATH=/app/models
```

## Model Details & Memory Tuning

### Ollama Models (on M2 Mac)

To fit efficiently within the 64GB unified memory of the M2 Mac, configure Ollama environments before launching it (e.g., via `launchctl` or `~/.zshrc`):

```bash
export OLLAMA_KEEP_ALIVE="-1"           # Keep both models loaded permanently (no cold-start latency)
export OLLAMA_MAX_LOADED_MODELS="2"     # Allow embedding + LLM to coexist in memory
export OLLAMA_NUM_PARALLEL="4"          # Allow parallel embedding requests during taxonomy sync
export OLLAMA_HOST="0.0.0.0:11434"      # Bind to Tailscale interface
```

| Model | Source | Purpose | VRAM |
|---|---|---|---|
| Qwen3-Embedding-8B | Ollama | Dense embedding (512 + 4096 dim via MRL) | ~8GB |
| Gemma4-12B | Ollama | LLM selection + data cleaning | ~12GB |

**Total estimated VRAM:** ~20GB (leaves ~44GB headroom for macOS, Qdrant, and Redis).

### FastEmbed Models (on Dokploy)

| Model | Source | Purpose |
|---|---|---|
| `Qdrant/bm25` | FastEmbed | BM25 sparse vectors (CPU only) |
| `colbert-ir/colbertv2.0` | FastEmbed | ColBERT late interaction (128-dim per-token, CPU only) |

## Qdrant Collection Config

The collection uses a **4-vector architecture** optimized for multivector ColBERT rescoring.

```python
client.create_collection(
    collection_name="research_taxonomy",
    vectors_config={
        "dense_512": models.VectorParams(size=512, distance=models.Distance.COSINE),
        "dense_4096": models.VectorParams(size=4096, distance=models.Distance.COSINE),
        "colbert": models.VectorParams(
            size=128,  # colbert-ir/colbertv2.0
            distance=models.Distance.COSINE,
            multivector_config=models.MultiVectorConfig(
                comparator=models.MultiVectorComparator.MAX_SIM
            ),
            hnsw_config=models.HnswConfigDiff(m=0),  # Disable HNSW — ColBERT only used for rescoring
        ),
    },
    sparse_vectors_config={
        "bm25": models.SparseVectorParams(modifier=models.Modifier.IDF)
    },
)
```
*Note: For 70K taxonomy records, this configuration requires ~3.3GB - 3.5GB of persistent storage, with ColBERT multivectors accounting for roughly 60% of that total.*

## Network & Security Overview

- **Tailscale**: Connects Dokploy and the Mac. Ollama MUST bind to `0.0.0.0` or the tailnet IP. Ollama lacks authentication, making Tailscale the required security boundary.
- **Latency**: Tailscale's WireGuard layer introduces ~1-3ms latency between nodes.
- **Data Locality**: FastEmbed executes directly on the Dokploy server because sending raw text over the network is significantly cheaper than sending large arrays of 96-dim token multivectors. NumPy handles these large arrays efficiently in memory without the need for Polars dataframes.
