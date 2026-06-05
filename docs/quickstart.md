# Quickstart Guide

This guide will help you onboard and run the Research Taxonomy Service locally.

---

## 1. Prerequisites

- **Python 3.12+**
- **uv** (for fast dependency management)
- **Docker & Docker Compose** (for running local Redis and Qdrant)
- **Ollama** (installed on your local Mac)

---

## 2. Bootstrapping

1. **Clone the repository:**
   ```bash
   git clone <repo-url>
   cd research-areas-classify
   ```

2. **Sync Dependencies:**
   ```bash
   # Install production dependencies
   uv sync
   
   # Install dev dependencies
   uv sync --group dev
   ```

3. **Configure Environment:**
   ```bash
   cp .env.example .env
   ```
   *Edit `.env` and fill in your OpenAlex API key, ORCID credentials, and other secrets.*

---

## 3. Infrastructure Setup

1. **Start Qdrant and Redis via Docker Compose:**
   The project root contains a `docker-compose.yml` pre-configured with persistent volumes.
   ```bash
   docker compose up -d
   ```

2. **Pull Ollama Models:**
   ```bash
   ollama pull qwen3-embedding:8
   ollama pull gemma4:12b
   ```

3. **Configure Ollama Memory Limits (Mac):**
   ```bash
   # Add to ~/.zshrc or set via launchctl to keep models loaded
   export OLLAMA_KEEP_ALIVE="-1"
   export OLLAMA_MAX_LOADED_MODELS="2"
   export OLLAMA_NUM_PARALLEL="4"
   export OLLAMA_HOST="0.0.0.0:11434"
   ```

---

## 4. Running the Service

1. **Perform Initial Taxonomy Sync:**
   Before classifying authors, you must populate the local Qdrant instance.
   ```bash
   # Assuming the sync script is exposed via an admin endpoint
   curl -X POST http://localhost:8000/admin/taxonomy/sync -H "x-api-secret: your_secret"
   ```

2. **Start the ASGI Server (Granian):**
   Use `$(nproc)` to automatically bind workers to your available CPU cores. For IO-bound tasks, you might manually set this to `2 * CPU cores`. In the Dokploy environment, this is overridden via environment variables.
   ```bash
   granian --interface asgi --workers $(nproc) --port 8000 src.main:app
   ```

3. **Test with a Sample Payload:**
   ```bash
   curl -X POST http://localhost:8000/research-areas/classify \
     -H "Content-Type: application/json" \
     -d @docs/input-from-n8n.json
   ```

---

## 5. Development & Quality Checks

Run the following commands to ensure code quality:

```bash
# Linting & Formatting
ruff check src/

# Static Type Checking
pyright src/

# Unit Tests
pytest

# Test Coverage
coverage run -m pytest && coverage report
```
