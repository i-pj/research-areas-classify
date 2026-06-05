# Observability & Monitoring

The Research Taxonomy Service uses **Pydantic Logfire** (built on OpenTelemetry) to provide zero-config observability, distributed tracing, and real-time metric dashboards.

---

## 1. Bootstrapping Logfire

Ensure `LOGFIRE_TOKEN` is set in your environment variables. The service initializes Logfire centrally in the main application lifecycle:

```python
import logfire
from fastapi import FastAPI
from openai import AsyncOpenAI

# 1. Configure the central Logfire client
logfire.configure(pydantic_plugin=logfire.PydanticPlugin(record="all"))

app = FastAPI()

# 2. Instrument FastAPI (captures all HTTP requests, latencies, and 4xx/5xx errors)
logfire.instrument_fastapi(app)

# 3. Instrument the OpenAI/Ollama client
client = AsyncOpenAI(base_url="...", api_key="...")
logfire.instrument_openai(client)
```

---

## 2. Trace Propagation

Trace propagation happens automatically across the stack:
- **FastAPI**: Inbound HTTP requests from n8n automatically start a new trace span.
- **Pydantic**: Data validation errors are logged as span events.
- **Hishel/Httpx**: Outbound API calls (OpenAlex, Semantic Scholar, ORCID) are traced, allowing us to pinpoint slow third-party API dependencies.
- **Ollama/OpenRouter**: Every LLM generation request logs the prompt, response, tokens used, and latency.

---

## 3. Pipeline Metric Spans

For complex retrieval pipelines, we emit custom spans to benchmark each stage independently. This helps answer questions like, *"Is the ColBERT rescoring taking too long compared to the dense fetch?"*

```python
with logfire.span("stage_4_hybrid_search"):
    # Qdrant queries executed here
    pass

with logfire.span("stage_6_colbert_rescore"):
    # FastEmbed token-level late interaction scoring executed here
    pass

with logfire.span("stage_7_llm_selection", candidates_count=len(candidates)):
    # LiteLLM/Instructor LLM calls executed here
    pass
```

### Key Metrics to Monitor
1. **Cache Hit Ratios**: Monitor the frequency of Redis hits vs. live OpenAlex API calls.
2. **ColBERT Rescoring Latency**: Ensure the CPU overhead of evaluating multi-vector similarities locally remains within acceptable limits (< 300ms).
3. **Pydantic Validation Exceptions**: High rates of validation errors from Instructor mean the LLM is failing to adhere to the schema and wasting retry budgets.
