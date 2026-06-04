Here is the dependency stack for this service, formatted for a modern `pyproject.toml` (PEP 621 standard, compatible with `uv`, `poetry`, or `hatch`).

This stack is carefully selected to support **Async processing**, **BM42 sparse vectors**, **Structured Outputs**, **Fuzzy Matching**, and **Ollama-hosted models** with minimal bloat.

### `pyproject.toml`
```toml
[project]
name = "research-taxonomy-service"
version = "0.1.0"
description = "FastAPI service for academic research area classification and author profile enrichment"
requires-python = ">=3.11"
dependencies = [
    # --- Core Web Framework ---
    "fastapi>=0.111.0",           # High-performance async web framework
    "uvicorn[standard]>=0.30.1",  # ASGI server for FastAPI
    "pydantic>=2.7.0",            # Core data validation & JSON schemas (v2 is written in Rust, extremely fast)

    # --- LLM & Structured Outputs ---
    "instructor>=1.3.0",          # SOTA for forcing LLMs to return 100% valid Pydantic JSON
    "openai>=1.30.0",             # Standard client (Instructor wraps this; Ollama exposes OpenAI-compatible API)

    # --- Vector DB & Retrieval ---
    "qdrant-client>=1.9.0",       # Native Qdrant client
    "fastembed>=0.3.1",           # SOTA lightweight embedding generation. Natively supports BM42 and BGE models without needing massive PyTorch installations.

    # --- Async Network & API Resilience ---
    "httpx>=0.27.0",              # SOTA async HTTP client (for concurrent OpenAlex/Semantic Scholar/ORCID requests)
    "tenacity>=8.3.0",            # SOTA retry logic (handles exponential backoff and circuit breaking for external APIs)

    # --- Caching & Utility ---
    "redis>=5.0.4",               # Async Redis client for Semantic Caching
    "rapidfuzz>=3.9.0",           # SOTA string matching (written in C++, 100x faster than FuzzyWuzzy, used for 0.85 OpenAlex title similarity)
    "python-dotenv>=1.0.1"        # Secure environment variable management
]

[project.optional-dependencies]
dev = [
    "ruff>=0.4.5",                # SOTA Linter/Formatter (replaces flake8, black, isort - written in Rust)
    "pytest>=8.2.0",
    "pytest-asyncio>=0.23.6"
]
```

---

### Why these specific packages? (The Rationale)

**1. `instructor` (The LLM Savior)**
You *do not* want to parse raw strings from an LLM. `instructor` acts as a wrapper around the OpenAI SDK. You simply pass it your `TaxonomyResponse` Pydantic model, and it guarantees that the LLM returns exactly those arrays, or it automatically retries and fixes the JSON for you. **Because Ollama exposes an OpenAI-compatible API**, `instructor` works seamlessly with it — just point the OpenAI client's `base_url` to `http://<mac-ip>:11434/v1`.

**2. `openai` (Ollama Compatibility Layer)**
We use the standard `openai` Python client not because we're calling OpenAI's cloud API, but because **Ollama provides an OpenAI-compatible `/v1/chat/completions` endpoint**. This means we can use `instructor` + `openai` to talk to `gemma4:12b` on Ollama as if it were GPT-4. No separate Ollama SDK needed.

**3. `fastembed` (The Infrastructure Cheat Code)**
This library is built and maintained by the Qdrant team. Instead of downloading heavy, 5GB PyTorch/Transformers dependencies to run your embedding models, `fastembed` runs highly optimized ONNX runtimes. More importantly, **it natively generates BM42 sparse vectors**. 

**Note on Qwen3-Embedding-8B:** The `qwen3-embedding:8` model is hosted on Ollama and accessed via the Ollama embeddings API (`/api/embeddings` or the OpenAI-compatible `/v1/embeddings`). `fastembed` may be used as a fallback for BM42 sparse vector generation if needed (using `BAAI/bge-m3`).

**4. `rapidfuzz` (The Title Matcher)**
You need to verify OpenAlex returned the correct paper using a `0.85` string similarity. `rapidfuzz` is the modern standard — written in C++, highly accurate, and resolves Levenshtein distances in microseconds.

**5. `tenacity` (The Circuit Breaker)**
Your spec requires retry logic with exponential backoff for external API calls (OpenAlex, Semantic Scholar, ORCID). `tenacity` lets you simply add `@retry(wait=wait_exponential(multiplier=1, min=2, max=8))` above your HTTP calls, and it handles the entire backoff, retry, and failure logging elegantly.

**6. `httpx` (Async HTTP)**
All external API calls (OpenAlex, Semantic Scholar, ORCID) run concurrently via `asyncio.gather()`. `httpx` is the standard async HTTP client for Python, replacing `requests` for async workloads.

**7. `ruff` (DevEx)**
If you are starting a new Python service today, use `ruff`. It replaces Black, Flake8, and isort, and runs in milliseconds. It enforces ultra-clean codebase standards.

---

### Packages NOT needed

| Package | Reason |
|---|---|
| `anthropic` | Not using Claude. All LLM calls go through Ollama's OpenAI-compatible API. |
| `transformers` / `torch` | Not needed. Embeddings are via Ollama (`qwen3-embedding:8`) and `fastembed`. Reranking is via Ollama (`Qwen3-Reranker-4B`). |
| `ollama` (pip package) | Not needed. We use the `openai` SDK pointed at Ollama's OpenAI-compatible endpoint. Simpler and works with `instructor`. |
| `langchain` | Unnecessary abstraction for this service. Direct API calls are simpler. |

---

### Ollama Setup on Mac M2

The following models must be available on the Ollama instance:

```bash
# Pull models
ollama pull qwen3-embedding:8
ollama pull dengcao/Qwen3-Reranker-4B:Q5_K_M
ollama pull gemma4:12b

# Verify models are loaded
ollama list

# Start Ollama server (if not already running as a service)
ollama serve
```

**Ollama API endpoints used by the service:**

| Purpose | Endpoint | Model |
|---|---|---|
| Text embedding | `POST /v1/embeddings` | `qwen3-embedding:8` |
| LLM chat (taxonomy selection, data cleaning) | `POST /v1/chat/completions` | `gemma4:12b` |
| Reranking | Custom endpoint or via chat with scoring prompt | `dengcao/Qwen3-Reranker-4B:Q5_K_M` |

**Note on reranking:** Ollama does not natively expose a cross-encoder reranking endpoint. The reranker model can be used via the chat completion endpoint with a specifically formatted prompt that asks for relevance scores. Alternatively, if `fastembed` supports the reranker model, it can be used directly.