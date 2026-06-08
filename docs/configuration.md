# Configuration Management

The service manages configuration via `pydantic-settings`, providing type safety, validation, and secure handling of credentials.

---

## 1. Pydantic Settings Class

The configuration is defined in a `Settings` class that inherits from `BaseSettings`. It automatically loads values from the `.env` file and environment variables.

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import SecretStr
from functools import lru_cache

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    # Granian
    granian_workers: int = 4
    granian_port: int = 8000

    # Ollama
    ollama_base_url: str = "http://localhost:11434"

    # Qdrant
    qdrant_url: str = "http://localhost:6333"
    qdrant_api_key: SecretStr | None = None
    qdrant_collection: str = "research_taxonomy"
    reviewer_collection: str = "reviewer_profiles"

    # OpenAlex (mandatory API key since Feb 2026)
    openalex_api_key: SecretStr

    # ORCID
    orcid_client_id: str
    orcid_client_secret: SecretStr

    # ResearchNodes
    researchnodes_api_url: str = "https://uni.researchnodes.com"
    researchnodes_api_secret: SecretStr

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # Retrieval Tuning Weights
    colbert_weight: float = 0.6
    composite_weight: float = 0.4
    
    # Reviewer Matching Weights & Limits
    colbert_weight_reviewer: float = 0.6
    composite_weight_reviewer: float = 0.4
    temporal_decay_lambda: float = 0.15
    stage1_candidate_limit: int = 50
    stage2_rescore_limit: int = 15

    # LiteLLM (optional)
    openrouter_api_key: SecretStr | None = None

    # Observability
    logfire_token: SecretStr | None = None
    
    # FastEmbed Caching
    fastembed_cache_path: str = "/tmp/fastembed_cache"

@lru_cache
def get_settings():
    return Settings()
```

---

## 2. Environment Variables Reference

| Variable | Type | Default / Example | Required | Purpose |
|---|---|---|---|---|
| `GRANIAN_WORKERS` | `int` | `4` | No | Number of ASGI worker processes. |
| `GRANIAN_PORT` | `int` | `8000` | No | Port for the ASGI server. |
| `OLLAMA_BASE_URL` | `str` | `http://localhost:11434` | No | URL to the Ollama instance (use Tailscale MagicDNS). |
| `QDRANT_URL` | `str` | `http://localhost:6333` | No | URL to the Qdrant instance. |
| `QDRANT_API_KEY` | `SecretStr` | `None` | No | Defense-in-depth API key for Qdrant. |
| `REVIEWER_COLLECTION` | `str` | `reviewer_profiles` | No | Qdrant collection name for reviewer profiles. |
| `OPENALEX_API_KEY` | `SecretStr` | — | **Yes** | Mandatory API key for OpenAlex (polite pool deprecated). |
| `ORCID_CLIENT_ID` | `str` | — | **Yes** | Public API client ID for ORCID. |
| `ORCID_CLIENT_SECRET` | `SecretStr`| — | **Yes** | Public API client secret for ORCID. |
| `RESEARCHNODES_API_SECRET` | `SecretStr`| — | **Yes** | Secret for writing to ResearchNodes endpoints. |
| `REDIS_URL` | `str` | `redis://localhost:6379/0` | No | URL for Hishel caching, semantic caching, and locks. |
| `FASTEMBED_CACHE_PATH` | `str` | `/tmp/fastembed_cache` | No | Persistent volume path for FastEmbed ONNX models. |

---

## 4. Redis Namespacing Strategy

The service uses Redis for three distinct concerns. To prevent key collision and simplify TTL management, data is logically separated using key prefixes (or logical database indices, depending on configuration):

| Concern | Key Prefix | Redis DB | TTL (Expiration) | Purpose |
|---|---|---|---|---|
| **Hishel HTTP Cache** | `hishel:` | `DB 0` | 24 Hours | Caches external API responses (OpenAlex, S2). |
| **Semantic Cache** | `semantic:` | `DB 0` | 7 Days | Caches LLM taxonomy outputs for identical queries. |
| **Distributed Locks** | `lock:` | `DB 0` | 30 Seconds | Prevents race conditions during custom keyword creation. |

---

## 5. FastEmbed Model Caching (Dokploy)

By default, FastEmbed downloads ONNX models to `/tmp` on first use. In a Dockerized Dokploy environment, these models are lost on container restart.

**Action:** Configure a persistent Docker volume and set the `FASTEMBED_CACHE_PATH` environment variable.

```yaml
# docker-compose.yml on Dokploy
services:
  fastapi:
    ...
    volumes:
      - fastembed_cache:/app/models
    environment:
      - FASTEMBED_CACHE_PATH=/app/models

volumes:
  fastembed_cache:
```

---

## 6. Cache TTL Configurations

- **Hishel HTTP Cache:** Configured per external API domain based on volatility (e.g., OpenAlex author data cached for 24 hours).
- **Redis Semantic Cache:** LLM taxonomy selection cached for 7 days.
- **Redis Distributed Locks:** Short-lived locks (e.g., 30s timeout, 5s blocking timeout) for concurrent custom keyword creation.
