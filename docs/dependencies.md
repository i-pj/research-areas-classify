Here is the dependency stack for this service, formatted for a modern `pyproject.toml` (PEP 621 standard, compatible with `uv`, `poetry`, or `hatch`).

This stack is carefully selected to support **Async processing**, **BM25 sparse vectors**, **ColBERT late interaction**, **Structured Outputs**, **Fast HTTP Caching**, and **Ollama-hosted models** with minimal bloat.

### `pyproject.toml`
```toml
[project]
name = "research-taxonomy-service"
version = "0.1.0"
description = "FastAPI service for academic research area classification and author profile enrichment"
readme = "README.md"
requires-python = ">=3.12"

dependencies = [
    # --- Core Web Framework ---
    "fastapi>=0.136.0",          # SOTA async web framework
    "granian>=1.6.0",            # Rust-based ASGI server (massive throughput lift over Uvicorn)
    "pydantic>=2.9.0",           # Core data validation (Rust-backed pydantic-core)
    "pydantic-settings>=2.5.0",  # Type-safe environment variable management
    "orjson>=3.10.0",            # Rust-based JSON serialization for custom high-perf tasks (e.g. numpy arrays)

    # --- LLM & Structured Outputs ---
    "instructor>=1.7.0",         # Forces LLMs to return valid Pydantic JSON (from_provider API)
    "openai>=1.50.0",            # Standard client (Instructor wraps this for Ollama compatibility)
    "litellm>=1.55.0",           # LLM Routing (Ollama local -> OpenRouter cloud fallback)

    # --- Vector DB & Retrieval ---
    "qdrant-client>=1.12.0",     # Native Qdrant client (supports multivector MAX_SIM)
    "fastembed>=0.4.1",          # lightweight embedding generation (Dense, BM25, and ColBERT)

    # --- Async Network & API Resilience ---
    "httpx>=0.28.0",             # async HTTP client
    "hishel>=0.0.30",            # RFC 9111 HTTP Cache (wraps httpx to cache OpenAlex/S2 calls)
    "tenacity>=9.0.0",           # retry logic (exponential backoff and circuit breaking)

    # --- Domain API Clients ---
    "pyalex>=0.14.0",            # Official OpenAlex client
    "semanticscholar>=0.8.4",    # Official Semantic Scholar client

    # --- Caching, Matching & Observability ---
    "redis[hiredis]>=5.2.0",     # Async Redis client for Hishel cache and distributed locks
    "rapidfuzz>=3.11.0",         # C++ string matching (used for 0.85 OpenAlex title similarity)
    "logfire>=4.34.0",           # OpenTelemetry observability (traces FastAPI, Instructor, LLM)
]

[project.optional-dependencies]
dev = [
    "ruff>=0.4.5",                # Linter/Formatter (replaces flake8, black, isort - written in Rust)
    "pyright>=1.1.365",           # Strict static type checking
    "pytest>=8.2.0",
    "pytest-asyncio>=0.23.6",
    "pytest-httpx>=0.30.0",       # Mocking httpx calls in tests
    "polyfactory>=2.16.0",        # Mock data generation
    "anyio[trio]>=4.4.0",         # Async concurrency testing
    "coverage[toml]>=7.5.0"
]
```

---

### Why these specific packages? (The Rationale)

**1. `granian` (The ASGI Server)**
`granian` is a Rust-based HTTP server for Python applications. It provides native HTTP/2 support, built-in process management, and significantly higher throughput than `uvicorn`. No `gunicorn` wrapper is needed. You simply run: `granian --interface asgi --workers 4 main:app`.

**2. `pydantic` and `orjson` Serialization**
FastAPI natively uses Pydantic v2's Rust-based `pydantic-core` for extremely fast JSON serialization. **We explicitly DO NOT use `ORJSONResponse` as the default FastAPI response class because it is deprecated.** However, the `orjson` library is retained in the stack because it provides unique, high-performance capabilities for custom serialization tasks, such as serializing/deserializing NumPy arrays output by embedding operations.

**3. NumPy vs. Polars (Data Manipulation)**
We **exclusively use NumPy** and **reject Polars**.
- **Why NumPy:** It is already a transitive dependency installed by `fastembed` and `onnxruntime`. We use it for lightweight multi-dimensional vector array operations and slicing (e.g., MRL projection).
- **Why not Polars:** The service processes single-author JSON payloads (online transactions). It performs zero tabular data manipulation. Including a heavy Rust-based dataframe library like Polars would needlessly increase the container size and memory footprint without providing any functional benefit.

**4. `fastembed` (The CPU Infrastructure Cheat Code)**
Built by the Qdrant team, `fastembed` uses ONNX runtimes to run embeddings on the Dokploy server CPU without the bloat of PyTorch. It supports three distinct vector types:
1. Dense Embeddings (can also use `qwen3-embedding:8` via Ollama)
2. Sparse BM25 via `SparseTextEmbedding("Qdrant/bm25")`
3. ColBERT Late Interaction via `LateInteractionTextEmbedding("answerdotai/answerai-colbert-small-v1")`

**5. `instructor` & `litellm` (The LLM Router Layer)**
The stack uses both libraries in tandem because they serve distinct architectural purposes:
- **`litellm` (Network/Routing):** Acts as a routing layer to intelligently direct complex tasks to cloud providers (e.g., Claude 3.5 Sonnet on OpenRouter) while sending simple tasks to local Ollama (`gemma4:12b`). It handles API standardization, retries, and fallbacks.
- **`instructor` (Parsing/Validation):** Sits on top of the routing layer. It forces the LLM to return exactly the Pydantic schema required (`from_provider("ollama/gemma4:12b")` or wrapped OpenAI client), guaranteeing 100% valid JSON responses and eliminating brittle regex parsing.

**6. `hishel` & `redis[hiredis]` (The Caching Layer)**
`hishel` implements RFC 9111 compliant HTTP caching. By wrapping our `httpx` client, it transparently caches identical OpenAlex or Semantic Scholar responses in Redis, eliminating redundant network calls and saving API credits. Redis also powers distributed locks (`redis.lock`) to prevent race conditions during custom keyword creation.

**7. `pyalex` (OpenAlex Client)**
The official client for OpenAlex. **Note:** The `mailto` polite pool was deprecated in Feb 2026. The client must be configured with an API key: `pyalex.config.api_key = "<your_key>"`.

**8. `logfire` (Observability)**
OpenTelemetry native instrumentation. By simply calling `logfire.instrument_fastapi(app)` and `logfire.instrument_openai(client)`, you gain zero-config dashboards showing exact latencies of Pydantic validations, HTTP requests, Qdrant queries, and LLM generations.

---

### Packages NOT needed

| Package                  | Reason                                                                                        |
| ------------------------ | --------------------------------------------------------------------------------------------- |
| `uvicorn`                | Replaced by the much faster Rust-based `granian`.                                             |
| `python-dotenv`          | Replaced by `pydantic-settings` which offers type safety and validation.                      |
| `transformers` / `torch` | Not needed. Embeddings run via `fastembed` (ONNX) and Ollama.                                 |
| `ollama` (pip package)   | Not needed. We use the standard `openai` SDK pointing to Ollama's OpenAI-compatible endpoint. |
| `langchain`              | Unnecessary abstraction. Direct API calls are simpler and more maintainable.                  |
| `beautifulsoup4`         | LLM-based data cleaning handles dirty text without brittle HTML parsing.                      |
| `polars`                 | DataFrames are useless for single-payload JSON API operations.                                |

---

### Tool Configurations (ruff, pyright, pytest)

These tools are configured in the `[tool.*]` sections of `pyproject.toml`:

- **ruff**: Replaces Black, Flake8, and isort. Enforces modern syntax.
- **pyright**: Configured for strict type checking mode.
- **pytest-asyncio**: Configured with `asyncio_mode = "auto"`.