# Research Area Extraction & Author Profile — Production Specification

## 1. Objective

Build a research-area extraction and author profile enrichment service for the author profile workflow.

The service receives a merged author profile payload containing various data sources (Journal API data, Mautic marketing data, legacy CRM data, email threads). It must:

1. **Enrich the author profile** by querying external academic APIs (OpenAlex, Semantic Scholar, ORCID) to fill in verified information.
2. **Extract research taxonomy** by opportunistically gathering the highest-quality academic evidence, resolving it against a Qdrant vector database, and inferring three taxonomy-backed research-area groups:
   - **Primary Research Disciplines** from ResearchNodes `/taxonomy/fields`
   - **Research Specializations** from ResearchNodes `/taxonomy/subfields`
   - **Research Keywords & Themes** from ResearchNodes `/keywords`

The service is a FastAPI application deployed on Dokploy, exposing a single endpoint. n8n sends one author payload at a time. The service returns the three taxonomy arrays plus a fully enriched `author_details` object. n8n then merges these into the existing profile payload and calls the CRM `UpdateAuthor` webhook.

**Primary goal:** Maximize correctness. Wrong taxonomy assignments are worse than missing assignments. Do not stuff the LLM with low-quality text if high-quality academic data is available.

**Data cleaning policy:** Use LLM-based approaches for cleaning messy data (HTML tags in university names, encoding artifacts like `Â` characters, name prefix extraction). Do not write complex regex or manual parsing logic for data normalization.

---

## 2. n8n Workflow Context

### Integration point

The n8n workflow fetches data from legacy CRM, Mautic, and Journal APIs, then merges all sources into a single JSON object (the `Merge1` node).

After merging, n8n calls the research service:

```http
POST /research-areas/classify
```

n8n forwards the complete merged output — unmodified — as a **single JSON object** (not an array). The service returns the taxonomy arrays plus the enriched author profile. n8n then merges the returned data into the existing profile payload and calls the CRM `UpdateAuthor` webhook.

### Important request rule

n8n must send the **complete merged object** to the service. Do not pre-filter, trim, reshape, or drop fields before the call. The service acts as a smart data vacuum, dynamically evaluating which fields to use based on data quality.

### Input payload reference

See `input-from-n8n.json` for a complete sample payload. See `input-field-map.md` for documentation of which fields come from which data source.

---

## 3. FastAPI Service API

### `POST /research-areas/classify`

**Purpose:** Generate research taxonomy arrays and build an enriched author profile for any author based on the provided payload.

**Input:** A single JSON object — the complete merged payload from n8n.

**Processing Rules:**

1. **Refined Early Exit (Preservation Rule):**
   - Does the payload already contain valid, hierarchy-complete `research_disciplines`, `specializations`, and `keywords`?
   - AND is the payload missing any new "Tier 1" evidence (like newly added `article_details`)?
   - If *both* are true, return the existing arrays immediately and skip extraction to save compute. Do not overwrite good data. If new papers exist, proceed to pipeline to append/update.
   - **Important:** Even if early exit triggers for research taxonomy, the author profile enrichment (ORCID, OpenAlex Author) should still run if the `author_details` has empty fields that could be filled.
2. **Smart Evidence Tiering:** Do not feed all payload text to the LLM. Stop gathering evidence as soon as high-quality data is found (see Section 8).
3. **Enrichment:** Extract a maximum of **3 paper titles** and enrich via OpenAlex and Semantic Scholar. Additionally, search for the author on OpenAlex and ORCID for profile enrichment.
4. **Classify:** Run the Qdrant hybrid retrieval and LLM selection pipeline.
5. **Build author_details:** Construct the enriched author profile using all available data.
6. **Return:** The three research arrays, the enriched `author_details`, and diagnostics.

### Success response

```json
{
  "success": true,
  "author_details": {
    "user_id": 36234,
    "basic_information": {
      "prefix": "",
      "first_name": "Shutao",
      "middle_name": "",
      "last_name": "Lan",
      "suffix": "",
      "display_name": "Shutao Lan",
      "email": "100438330@alumnos.uc3m.es",
      "city": "",
      "country": "Spain",
      "phone_number": "605344887"
    },
    "biography": "",
    "academic_ids": {
      "orcid": "0000-xxxx-xxxx-xxxx",
      "research_gate_url": null,
      "google_scholar_url": null
    },
    "social_links": {
      "linkedin": null,
      "twitter": null,
      "website": null,
      "facebook": null,
      "instagram": null
    },
    "affiliations": [
      {
        "institution": "Universidad Carlos III de Madrid",
        "location": "Madrid, Spain",
        "university": "",
        "department": "",
        "position_role": "PhD student",
        "start_year": "",
        "end_year": "present",
        "institution_open_alex_id": "https://openalex.org/I...",
        "university_open_alex_id": "",
        "is_current": true
      }
    ],
    "education": [],
    "research_areas": "Sustainable Tourism, Cultural Heritage, Culinary Tourism",
    "patents": [],
    "editor_roles": [],
    "memberships": [],
    "advisors": [],
    "advisees": [],
    "grants_awards": []
  },
  "research_disciplines": [
    {
      "id": "https://openalex.org/fields/36",
      "name": "Earth and Planetary Sciences",
      "domain_id": "https://openalex.org/domains/3"
    }
  ],
  "research_specializations": [],
  "research_keywords": [],
  "diagnostics": {
    "status": "completed",
    "preserved_existing_research": false,
    "evidence_tiers_used": ["tier_2_mautic_crm"],
    "openalex_found": true,
    "semantic_scholar_found": false,
    "orcid_found": true,
    "orcid_id": "0000-xxxx-xxxx-xxxx"
  }
}
```

### `author_details` schema

The `author_details` object uses the exact same schema defined in `author-profile.json`. See that file for the complete field reference. The service must populate each field with the best available data from **external APIs** (OpenAlex, ORCID, Semantic Scholar). 

**Important Omission Policy:**
- Only return fields that are **not empty and not null**.
- **Do not send back data that was only provided in the input** (e.g., if you only know `email` because it was in the input, omit it. Return it only if it was verified/found via external APIs).
- The only exception is `user_id` which must always be returned to identify the author.

### Error response

If processing fails, return safe empty arrays and an empty `author_details` object. Never return hallucinated taxonomy.

```json
{
  "success": false,
  "author_details": {},
  "research_disciplines": [],
  "research_specializations": [],
  "research_keywords": [],
  "diagnostics": {
    "status": "failed",
    "reason": "qdrant_unavailable"
  }
}
```

---

## 4. ResearchNodes API

Documentation: `https://uni.researchnodes.com/api-docs/`

### Taxonomy fields
- `GET /taxonomy/fields`

### Taxonomy subfields
- `GET /taxonomy/subfields?status=active`

### Keywords
- `GET /keywords?status=active`
- `POST /keywords` with API secret

`POST /keywords` body (for custom creation outside the main flow):
```json
{
  "keywordName": "string",
  "fieldId": "string",
  "subfieldId": "string",
  "subfieldName": "string",
  "description": "string"
}
```

**Security:** Write endpoints require header `x-api-secret`. Store secrets in environment variables.

---

## 5. Infrastructure & Models

### Deployment architecture

See `architecture.md` for the full deployment diagram.

- **FastAPI Service** — Deployed on Dokploy (no GPU).
- **Ollama** — Running on Apple M2 Mac (64GB RAM). Serves embedding, reranking, and LLM models.
- **Qdrant** — Running on Apple M2 Mac (64GB RAM). Stores taxonomy vectors.

The FastAPI service on Dokploy communicates with Ollama and Qdrant on the Mac via HTTP.

### Models hosted on Ollama

| Model | Purpose | Ollama Tag |
|---|---|---|
| Qwen3-Embedding-8B | Dense embedding generation (512 + 4096 dim via MRL) | `qwen3-embedding:8` |
| Gemma4-12B | LLM for taxonomy selection and data cleaning | `gemma4:12b` |

**Total estimated VRAM:** ~20GB (well within 64GB unified memory on M2)

### Models hosted via FastEmbed (Dokploy CPU)

| Model | Purpose |
|---|---|
| `Qdrant/bm25` | BM25 sparse vectors |
| `answerdotai/answerai-colbert-small-v1` | ColBERT late interaction (96-dim per-token) |

### Embedding model details

**`qwen3-embedding:8`** (via Ollama)
- Dimension: `4096` (full), `512` (MRL projection for first-pass retrieval)
- Matryoshka Representation Learning (MRL) — supports lower-dimension projections of the same embedding.
- Handles academic terminology and native abbreviation resolution without manual regex tables.

**FastEmbed Models** (BM25 + ColBERT)
- Executed locally on Dokploy to save network round-trips for token-level embeddings.
- Native NumPy array manipulation for token matrices (transitive via ONNX/FastEmbed).

### LLM details

**`gemma4:12b`** (via Ollama)
- Used for final taxonomy selection from ranked candidates.
- Used for data cleaning tasks (HTML stripping, encoding fix, name normalization).
- Used via `instructor` library with structured outputs (Pydantic models) to guarantee valid JSON responses.

---

## 6. Qdrant Collection Design

The Qdrant collection stores **taxonomy records only**. Author data is never stored in Qdrant.

### Collection name
`research_taxonomy`

### Vector configuration

```json
{
  "vectors": {
    "dense_512": { "size": 512, "distance": "Cosine" },
    "dense_4096": { "size": 4096, "distance": "Cosine" },
    "colbert": { "size": 96, "distance": "Cosine", "multivector_config": {"comparator": "MaxSim"} }
  },
  "sparse_vectors": {
    "bm25": {}
  }
}
```

**`dense_512`** — fast first-pass retrieval.
**`dense_4096`** — precise reranking of the shortlist.
**`bm25`** — Sparse neural retriever. Captures exact taxonomy term matches with IDF modifier.
**`colbert`** — Late-interaction multivector representation for highly precise token-level scoring (HNSW index disabled `m=0` since it's only used for rescoring).

### Hybrid query pattern (executed per kind: field, subfield, keyword)

results = client.query_points(
    collection_name="research_taxonomy",
    prefetch=[
        models.Prefetch(query=dense_512_vector, using="dense_512", limit=100),
        models.Prefetch(query=sparse_vector, using="bm25", limit=100)
    ],
    query=models.FusionQuery(fusion=models.Fusion.RRF),
    limit=50
)
# Note: Results are then rescored using the colbert vector.
```

### Point ID format
Deterministic string point IDs: `field::<id>`, `subfield::<id>`, `keyword::<id>`

---

## 7. Taxonomy Text to Embed

Each taxonomy record is converted into rich text before embedding.

**Template:**
```
Kind: keyword
Name: Autonomous Vehicle Technology and Safety
Aliases: autonomous vehicles, self-driving vehicles
Description: Research into the design, control, and safety verification of vehicles capable of navigating without human input.
Domain: Engineering
Field: Engineering
Subfield: Automotive Engineering
Type: topic
Level: 1
```

---

## 8. Author Evidence Building & Tiering

To prevent LLM context stuffing and hallucinations, the `ResearchEvidenceBuilder` evaluates the raw payload and extracts data based on **Quality Tiers**. As soon as high-signal data is found, lower tiers are ignored.

### Smart Evidence Tiering

**Tier 1: Gold Standard (Academic Truth)**
- `article_details` (Paper titles, abstracts, author keywords)
- Classifications (arXiv, ACM, IEEE codes)
- *Logic:* If the payload contains at least 1 valid paper abstract, extract Tier 1, skip Tier 2 and 3, and proceed to enrichment.
- **Note:** `article_details` is only present for authors who have submitted papers to the journals. Reviewer-only authors will not have this field. In that case, fall through to Tier 2.

**Tier 2: Silver Standard (Metadata & CRM)**
- Mautic fields (`authorData.titleOfPaper`, `authorData.titleOfPaper2`, through `titleOfPaper5`)
- Legacy CRM Reviewer streams (`old_crm_data.reviewer[].phdstream`, `old_crm_data.reviewer[].stream`)
- Legacy CRM keywords (`old_crm_data.keywords`)
- *Logic:* Use only if Tier 1 is empty. Mautic paper titles are especially valuable — they can be used to search OpenAlex for the actual paper and its pre-classified topics (see Section 9).

**Tier 3: Bronze (Fallback Noise)**
- Email thread text (`emails[].text`), scraped biography, generic notes.
- *Logic:* Use only if Tier 1 and Tier 2 are entirely empty.

### Handling profiles without `article_details`

This is **not limited to reviewers**. Any profile — author, reviewer, editor, fellow — may arrive without `article_details` if no papers have been submitted through the journal system. The fallback strategy applies universally whenever `article_details` is absent or empty.

When the payload has no `article_details`:

1. **Extract paper titles from Mautic**: `authorData.titleOfPaper` through `titleOfPaper5` (max 5, but cap at 3 for enrichment).
2. **Search OpenAlex by paper title**: If a match is found (≥ 0.85 title similarity), the paper's `topics[]` already contain pre-classified domain/field/subfield with confidence scores. These become strong evidence.
3. **Search for the author on OpenAlex**: Using the author name + institution, try to find the OpenAlex author profile. If found, their `topics[]` and `affiliations[]` provide verified data for both research classification and the `author_details`.
4. **Extract from CRM data**: `phdstream`, `stream`, and `keywords` contain self-declared research areas. Use LLM to clean encoding artifacts (e.g., `"Hospitality, Sport and Tourism ManagementÂ Â Â Â "` → `"Hospitality, Sport and Tourism Management"`).
5. **Fall back to email text**: If an author explicitly states their research area in email correspondence (e.g., *"My research area is sustainable tourism and its application in cultural heritage"*), use LLM to extract this.

### Classification hint rules
Apply deterministically before retrieval to boost specific candidates if their code is found in the evidence.

| Classification | Taxonomy boost target |
| --- | --- |
| arXiv cs.CV | Computer Vision and Pattern Recognition |
| arXiv cs.RO | Robotics |
| arXiv cs.AI | Artificial Intelligence |
| arXiv cs.LG | Machine Learning |
| arXiv cs.CL | Natural Language Processing |
| ACM I.2.10 | Artificial Intelligence |

### Evidence query text format

```
Author: Yicheng Li
Paper titles:
- NMLoNet: An End-to-End Intelligent Vehicle Localization Network Using Navigation Maps
Abstract evidence:
Accurate and reliable localization is crucial for advanced autonomous driving systems...
Author keywords:
bird's-eye-view, high-precision localization, Intelligent Vehicles
Classifications:
arXiv cs.CV, arXiv cs.RO
OpenAlex topics (pre-classified):
Computer Vision and Pattern Recognition (score: 0.99)
```

---

## 9. External Enrichment

The external enrichment stage serves two purposes:
1. **Research classification enrichment** — Finding pre-classified topics for papers.
2. **Author profile enrichment** — Finding verified author data for `author_details`.

All external API calls should be executed **concurrently** using `asyncio.gather()` to minimize total latency.

To prevent n8n timeouts, deduplicate the extracted paper titles and **take a maximum of the 3 most recent/relevant titles**.

### Step 1: OpenAlex Paper Search (Max 3 queries)

> **Important (API Key):** OpenAlex polite pool (mailto) was deprecated Feb 13, 2026. API key authentication is now mandatory for the high-performance tier.

```
GET https://api.openalex.org/works?filter=title.search:{url_encoded_title}&per_page=1&select=id,title,topics,keywords,authorships&api_key={openalex_api_key}
```
**If a match is found (≥ 0.85 title similarity):**
- Extract `topics[]` (pre-classified taxonomy records with confidence scores). Treat them as strong candidates with a score boost in the merge step.
- Extract `authorships[]` to identify the author's OpenAlex author ID for Step 3.

### Step 2: Semantic Scholar Paper Search (Max 3 queries)

```
GET https://api.semanticscholar.org/graph/v1/paper/search?query={url_encoded_title}&fields=fieldsOfStudy&limit=1
```
Extract `fieldsOfStudy` as domain-level hints.

### Step 3: OpenAlex Author Search

Search for the author on OpenAlex to get their full academic profile.

**Option A — If author ID was found via paper search (Step 1):**
```
GET https://api.openalex.org/authors/{author_id}?select=id,orcid,display_name,affiliations,topics,works_count,cited_by_count,counts_by_year&api_key={openalex_api_key}
```

**Option B — Search by name + institution:**
```
GET https://api.openalex.org/authors?filter=display_name.search:{author_name}&per_page=5&select=id,orcid,display_name,affiliations,topics,works_count&api_key={openalex_api_key}
```
Then match the correct author by comparing institution names.

**Data extracted for `author_details`:**
- `orcid` — If not already known, discover it here.
- `affiliations[]` — Verified institutions with OpenAlex IDs, country codes, and year ranges.
- `topics[]` — Research topics with work counts (strong signal for research classification).
- `works_count`, `cited_by_count` — Publication metrics.

### Step 4: ORCID Lookup

If an ORCID iD is available (from the input payload at `academic_ids.orcid`, or discovered via OpenAlex in Step 3), fetch the full ORCID record.

**API:** ORCID Public API v3.0 (authenticated with client credentials).

```
GET https://pub.orcid.org/v3.0/{orcid}/record
Accept: application/json
Authorization: Bearer {access_token}
```

**Data extracted for `author_details`:**
- `employments` → `affiliations[]` (organization name, city, country, ROR ID, role, department, start/end dates)
- `educations` → `education[]` (institution, degree/role, department, dates)
- `works` → Used for research evidence if other tiers are empty; also validates paper titles.
- `fundings` → `grants_awards[]`
- `qualifications` / `distinctions` → Additional metadata
- `memberships` → `memberships[]`

**ORCID data is self-reported by the author and considered highly trustworthy for profile fields.**

### Step 5: Merge enrichment results

When the same field (e.g., affiliations) is available from multiple sources, apply this priority:
1. **ORCID** — Highest trust (self-reported, verified)
2. **OpenAlex** — High trust (algorithmically derived from publications)
3. **Input payload** — Trusted (from prior n8n pipeline steps)

For research classification, apply additive boosts during candidate scoring:
- OpenAlex topic match: +0.15
- Classification hint match (arXiv/ACM codes): +0.10
- BM25 sparse match: +0.05

---

## 10. Author Profile Extraction (`author_details`)

The `author_details` object is **built from external API data** (OpenAlex, ORCID, Semantic Scholar). The n8n input payload is used **only for identifiers** (`user_id`, `basic_information.email`, author name) to look up the author on external platforms. n8n already has the input data — our job is to provide what the external APIs give us.

### Why not copy from input?

The n8n workflow already has access to all the CRM/Mautic/legacy data. Copying those fields into `author_details` would be redundant — n8n can do that itself. The value our service adds is **externally verified, API-sourced data** that n8n cannot get on its own.

### Processing approach

1. **Use input only for lookup:** Extract `user_id`, author name (`basic_information.first_name` + `last_name`), email, institution name, and `academic_ids.orcid` from the input payload. These are used as **search keys** to find the author on external platforms.

2. **Build from OpenAlex Author data:** If the author is found on OpenAlex:
   - `academic_ids.orcid` — Discovered ORCID iD.
   - `affiliations[]` — Verified institutions with OpenAlex IDs, country, year ranges.
   - `research_areas` — Derived from OpenAlex `topics[]` with work counts.
   - Publication metrics (`works_count`, `cited_by_count`).

3. **Build from ORCID data:** If an ORCID record is available:
   - `academic_ids.orcid` — Confirmed ORCID iD.
   - `affiliations[]` — From ORCID employments (organization, city, country, ROR ID, role, department, dates).
   - `education[]` — From ORCID educations (institution, degree, department, dates).
   - `grants_awards[]` — From ORCID fundings.
   - `memberships[]` — From ORCID memberships/services.
   - `biography` — From ORCID biography if available.

4. **Build from Semantic Scholar:** If the author is found:
   - `affiliations[]` — Current institution.
   - Additional `fieldsOfStudy` for research classification.

5. **Set `research_areas`:** The `research_areas` field should be a comma-separated string derived from the research classification results (the selected keywords and specializations).

6. **Set `user_id`:** Copy directly from the input payload — this is the only field that is a direct copy, since it's a WordPress CRM identifier that external APIs don't know about.

7. **Dynamic Data Cleaning:** The system handles messy data dynamically (e.g., encoding artifacts, HTML in Mautic data). It will use basic string extraction or the LLM for complex cleaning, completely avoiding strict scraping libraries like BeautifulSoup. If a field is too messy or returns a 404, it is simply omitted.

### What to return if nothing is found externally

If the author is not found on any external platform, `author_details` should contain only the `user_id` and empty/null fields. Do not fall back to copying input data — n8n already has it.

Use `response_model_exclude_none=True` on the FastAPI endpoint or `model_dump(exclude_none=True)` to dynamically filter out empty fields during serialization.

### Output schema

The `author_details` output must conform exactly to the schema defined in `author-profile.json`. See that file for the full field reference.

---

## 11. Final Payload Contract — Research Taxonomy

Only these three arrays are used for research taxonomy assignment.

### Schema per array

**1. `research_disciplines`**
```json
{
  "id": "https://openalex.org/fields/17",
  "name": "Computer Science",
  "domain_id": "https://openalex.org/domains/3"
}
```

**2. `research_specializations`**
```json
{
  "id": "https://openalex.org/subfields/1708",
  "name": "Hardware and Architecture",
  "domain_id": "https://openalex.org/domains/3",
  "field_id": "https://openalex.org/fields/17"
}
```

**3. `research_keywords`**
```json
{
  "id": "https://openalex.org/T10054",
  "name": "Parallel Computing and Optimization Techniques",
  "domain_id": "https://openalex.org/domains/3",
  "field_id": "https://openalex.org/fields/17",
  "subfield_id": "https://openalex.org/subfields/1708",
  "type": "topic"
}
```

| Field | Rule |
| --- | --- |
| `id` | Selected taxonomy ID. Custom keywords use `custom/{timestamp}`. |
| `name` | Selected taxonomy display name. |
| `domain_id` | Parent domain ID. |
| `field_id` | Parent field ID. Required for specializations and keywords. |
| `subfield_id` | Parent subfield ID. Required for keywords. |
| `type` | `topic` for standard keywords. `custom` for created custom records. |

---

## 12. Required Hierarchy and Relationship Rules

Relationships are mandatory. Do not return orphan records. Required chain: `domain → field → subfield → keyword`

- Every keyword must have a non-empty `field_id` and `subfield_id`.
- If a selected keyword's parent subfield is not in `research_specializations`, the hierarchy completion step adds it automatically.
- If a selected keyword's parent field is not in `research_disciplines`, it is added automatically.

---

## 13. Retrieval Pipeline

### Stage 1 — Smart Evidence Builder
Extract data according to the Quality Tiers (Section 8). Produce the formatted evidence query text. Handle reviewer-only authors by extracting Tier 2/3 evidence.

### Stage 2 — External Enrichment (Concurrent)
Run all external API calls concurrently using `asyncio.gather()`:
- OpenAlex Paper Search (max 3 titles)
- Semantic Scholar Paper Search (max 3 titles)
- OpenAlex Author Search (by name + institution, or by discovered author ID)
- ORCID Lookup (if ORCID is known or discovered via OpenAlex)

Append OpenAlex topics with score ≥ 0.85 to the evidence object.

### Stage 3 — Author Profile Assembly
Build the `author_details` object from input data + enrichment results (see Section 10).

### Stage 4 — Hybrid Qdrant Retrieval
Run three separate hybrid queries (`field`, `subfield`, `keyword`). 

**Advanced Embedding Specifications:**
1. **Matryoshka Representation Learning (MRL):** Use a two-stage Qdrant search. Qdrant quickly fetches the top 100 candidates using a smaller `512-dimension` vector (oversampling), and then automatically rescores them using the full `4096-dimension` vector. This maximizes both speed and accuracy.
2. **Reciprocal Rank Fusion (RRF):** For the hybrid search (Sparse BM25 + Dense 4096), use Qdrant's native RRF instead of raw score addition. This prevents exact-keyword matches (which have massive sparse scores) from completely overpowering deep semantic matches.
3. **Instruction Prefixes:** When talking to the Ollama embedding model, explicitly prefix the query evidence with: *"Retrieve corresponding research taxonomy categories for the following academic profile: "* to activate its instruction-aware capabilities.

### Stage 5 — Candidate Merge & Scoring
After Qdrant returns the RRF-fused results, assign additive boosts based on external evidence:
- OpenAlex topic match: +0.15
- Classification hint match (e.g., arXiv cs.CV): +0.10

### Stage 6 — ColBERT Late Interaction Rescoring
Pass the FastEmbed-generated query multivector to Qdrant's native query API to rescore the RRF-fused shortlist. **FastEmbed is only used to encode the query tensor on Dokploy**; the actual token-level MaxSim comparison is executed natively by Qdrant.
`final_score = (colbert_maxsim × 0.6) + (composite_score × 0.4)`

> **Note on Weights:** The scoring weights (`0.6` ColBERT / `0.4` Composite) are **initial estimates**. These values must be empirically tuned (e.g., evaluating NDCG@k on a held-out query dataset). They are configurable via `pydantic-settings` without code changes.

### Stage 7 — LLM Selection
Pass the tiered evidence text and top reranked candidates to the LLM (gemma4:12b via Ollama).

**System Prompt Instructions:**
```
You are a research taxonomy classifier. Given the author evidence and the ranked candidate taxonomy records, select the correct research disciplines, specializations, and keywords.

Rules:
- Select only taxonomy records that are directly supported by the evidence.
- Contextual Acronyms: Automatically resolve and expand academic acronyms (e.g., CV -> Computer Vision) based on the surrounding context.
- Reject candidates that are too broad when a more precise match exists.
- Preserve the exact IDs from the candidate list. Do not modify or invent IDs.
- For each selected record, state which evidence field supports the selection.
- If no candidate is suitable, return an empty array for that type.
- Mark highly specific phrases that lack a taxonomy match as unresolved_custom_candidates.
- Do not create custom fields or subfields.
```

**Use `instructor` library with Pydantic structured outputs** to guarantee valid JSON responses. Do not rely on parsing raw LLM text.

**Confidence thresholds for final acceptance:**
- Fields: ≥ 0.75
- Subfields: ≥ 0.75
- Keywords: ≥ 0.70

### Stage 8 — Hierarchy Completion
Auto-add required parent fields/subfields for any selected keywords.

### Stage 9 — Response Assembly
Combine:
- `author_details` from Stage 3
- `research_disciplines`, `research_specializations`, `research_keywords` from Stages 7-8
- Set `author_details.research_areas` as a comma-separated string from the selected keywords/specializations
- Build `diagnostics` object

---

## 14. Custom Keyword Creation Policy

When the Qdrant retrieval pipeline cannot find suitable existing matches for an author's research area, the service **automatically creates new custom keywords** via the ResearchNodes API. This ensures the taxonomy grows organically and future authors with similar research areas get matched immediately.

### Rules

- Never create custom primary disciplines (fields) or subfields — only keywords.
- Custom keywords are created only after the full pipeline (retrieval → reranking → LLM selection) confirms no suitable existing keyword match exists.
- The LLM must determine the correct parent `field_id` and `subfield_id` for the new keyword from the existing taxonomy.

### Auto-creation flow

1. **LLM identifies unresolved phrases**: During Stage 7 (LLM Selection), if the LLM determines that a research area phrase has no suitable match among the retrieved candidates, it marks it as an `unresolved_custom_candidate` with a suggested `field_id`, `subfield_id`, and a clean keyword name.

2. **Create via `POST /keywords`**: The service uses Redis distributed locks to prevent race conditions. To prevent duplicates from case or punctuation differences, the keyword is normalized before hashing (lowercase, strip punctuation, alphabetize words):
   ```python
   def normalize(text: str) -> str:
       # Lowercase, strip punctuation, split into words, sort, join
       pass
       
   hash = hashlib.md5(normalize(keyword_name).encode()).hexdigest()
   lock = redis_client.lock(f"lock:keyword:{hash}", timeout=30, blocking_timeout=5)
   
   async with lock:
       # Check existence again, then call ResearchNodes API
       # POST https://uni.researchnodes.com/keywords
   ```
   ```json
   {
     "keywordName": "Navigation-map-based Vehicle Localization",
     "fieldId": "https://openalex.org/fields/17",
     "subfieldId": "https://openalex.org/subfields/1707",
     "subfieldName": "Computer Vision and Pattern Recognition",
     "description": "Research into localization techniques for autonomous vehicles using navigation map data and bird's-eye-view representations."
   }
   ```

3. **Immediately embed in Qdrant**: After successful creation, the new keyword must be:
   - Converted to the taxonomy text format (Section 7).
   - Embedded using `qwen3-embedding:8` (dense_512 + dense_4096) and FastEmbed (BM25 + ColBERT).
   - Inserted into the `research_taxonomy` Qdrant collection with point ID `keyword::<new_id>`.
   - This ensures future requests can match against this keyword without waiting for a full taxonomy sync.

4. **Include in the current response**: The newly created keyword is added to the `research_keywords` array in the response (with `type: "custom"`) so the CRM profile is updated immediately.

5. **Log everything**: Every custom keyword creation must be logged with:
   - Timestamp
   - Author `user_id` that triggered the creation
   - The keyword name, description, parent field/subfield
   - The API response (including the new keyword ID)
   - The evidence phrase that led to the creation
   
   These logs should be stored persistently (e.g., to a `custom_keywords_log.jsonl` file or a structured log output) so there is a complete audit trail of what was added and why.

### File logging format

Custom keyword creations are **not reported in the API response**. They are logged to a persistent `custom_keywords_log.jsonl` file. Each line is one JSON object:

```json
{
  "timestamp": "2026-06-04T16:10:00Z",
  "user_id": 36234,
  "keyword_name": "Navigation-map-based Vehicle Localization",
  "keyword_id": "custom/1717500000",
  "field_id": "https://openalex.org/fields/17",
  "subfield_id": "https://openalex.org/subfields/1707",
  "subfield_name": "Computer Vision and Pattern Recognition",
  "description": "Research into localization techniques for autonomous vehicles using navigation map data.",
  "evidence_phrase": "Navigation-map-based vehicle localization",
  "api_response_status": 201,
  "embedded_in_qdrant": true
}

---

## 15. Admin Operation: Taxonomy Sync

Taxonomy sync keeps Qdrant vectors populated and updated directly from the ResearchNodes API.

**Implementation:** A scheduled background job or a protected admin endpoint:
`POST /admin/taxonomy/sync`

This endpoint must perform the following pipeline:
1. **Fetch Fields:** Call `GET /taxonomy/fields` to retrieve all primary disciplines.
2. **Fetch Subfields:** Call `GET /taxonomy/subfields?status=active` to retrieve all active specializations.
3. **Fetch Keywords (Paginated):** Call `GET /keywords?page={page}&limit=1000&status=active` in a loop to retrieve all active keywords.
4. **Embed & Upsert:** For each retrieved record:
   - Format the record into the standard rich text block (Section 7).
   - Generate the 4096-dim dense vector (`qwen3-embedding:8` via Ollama).
   - Generate the sparse vector (BM25) and multivector (ColBERT) via `fastembed`.
   - Upsert into the Qdrant `research_taxonomy` collection using a deterministic ID (e.g., `keyword::<id>`).

**Rules:**
- **Idempotency:** Because deterministic IDs are used, running the sync script multiple times safely updates existing records without creating duplicates.
- **Trigger:** This should be run by an admin manually or via a nightly cron job. It must **never** run automatically per-author request.

---

## 16. Quality Controls

- **No Overwriting:** Refined Early Exit prevents overwriting valid legacy profiles unless new paper evidence exists.
- **Speed Limits:** Max 3 OpenAlex/Semantic Scholar paper queries to prevent n8n timeouts.
- **Concurrency:** All external API calls run concurrently via `asyncio.gather()`.
- **Structured Outputs:** Use `instructor` + Pydantic for all LLM responses. Zero parsing errors.
- **Idempotency:** The same input produces identical selected taxonomy records.
- **Data Cleaning via LLM:** Do not write regex for messy data. Use the LLM for HTML stripping, encoding cleanup, and name normalization.

---

## 17. Example Output (Author with Papers)

```json
{
  "success": true,
  "author_details": {
    "user_id": null,
    "basic_information": {
      "prefix": "",
      "first_name": "Yicheng",
      "middle_name": "",
      "last_name": "Li",
      "suffix": "",
      "display_name": "Yicheng Li",
      "email": "liyicheng070@ujs.edu.cn",
      "city": "",
      "country": "China",
      "phone_number": ""
    },
    "biography": "",
    "academic_ids": {
      "orcid": null,
      "research_gate_url": null,
      "google_scholar_url": null
    },
    "social_links": {
      "linkedin": null,
      "twitter": null,
      "website": null,
      "facebook": null,
      "instagram": null
    },
    "affiliations": [
      {
        "institution": "Jiangsu University",
        "location": "Zhenjiang, China",
        "university": "",
        "department": "",
        "position_role": "",
        "start_year": "",
        "end_year": "present",
        "institution_open_alex_id": "",
        "university_open_alex_id": "",
        "is_current": true
      }
    ],
    "education": [],
    "research_areas": "Computer Vision and Pattern Recognition, Autonomous Vehicle Technology and Safety",
    "patents": [],
    "editor_roles": [],
    "memberships": [],
    "advisors": [],
    "advisees": [],
    "grants_awards": []
  },
  "research_disciplines": [
    {
      "id": "https://openalex.org/fields/17",
      "name": "Computer Science",
      "domain_id": "https://openalex.org/domains/3"
    }
  ],
  "research_specializations": [
    {
      "id": "https://openalex.org/subfields/1707",
      "name": "Computer Vision and Pattern Recognition",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17"
    }
  ],
  "research_keywords": [
    {
      "id": "https://openalex.org/T11099",
      "name": "Autonomous Vehicle Technology and Safety",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1707",
      "type": "topic"
    }
  ],
  "diagnostics": {
    "status": "completed",
    "preserved_existing_research": false,
    "evidence_tiers_used": ["tier_1_abstracts"],
    "openalex_found": true,
    "semantic_scholar_found": true,
    "orcid_found": false,
    "orcid_id": null
  }
}
```

---

## 18. Implementation Phases

**Phase 1 — Infrastructure & Sync Script**
- Set up Redis for distributed locks and semantic caching.
- Set up Qdrant on Mac M2 with 4-vector schema (`dense_512`, `dense_4096`, `bm25`, `colbert`).
- Set up Ollama models (verify embedding generation and LLM responses).
- Build the `POST /admin/taxonomy/sync` script to embed all taxonomy records using deterministic IDs.
- Set up FastAPI project on Dokploy with environment variables for Ollama/Qdrant endpoints.

**Phase 2 — External Enrichment & Author Profile**
- Implement `ExternalEnrichmentFetcher` with concurrent OpenAlex + Semantic Scholar + ORCID calls.
- Implement `AuthorProfileBuilder` that merges input data with enrichment results.
- Implement ORCID client with OAuth2 client credentials flow.
- Test with sample payloads (both author-with-papers and reviewer-only).

**Phase 3 — Smart Extraction & Evidence Building**
- Implement `ResearchEvidenceBuilder` with Smart Evidence Tiering (Tier 1/2/3).
- Handle reviewer-only fallback path (Mautic paper titles → OpenAlex search).
- Implement LLM-based data cleaning (via gemma4:12b).

**Phase 4 — Classification Pipeline**
- Implement hybrid Qdrant retrieval (dense + BM25).
- Implement ColBERT late interaction rescoring.
- Implement LLM selection with `instructor` + Pydantic structured outputs.
- Implement `HierarchyValidator`.

**Phase 5 — n8n Integration & Testing**
- Point n8n `Merge1` output to the single new endpoint.
- Merge the response (`author_details` + taxonomy arrays) into the profile and call CRM `UpdateAuthor`.
- End-to-end testing with real author payloads.

**Phase 6 — Custom Keyword Logging**
- Set up `custom_keywords_log.jsonl` file-based audit trail for all auto-created keywords.
- Monitor log for quality control of auto-created keywords.

**Phase 7 — Observability & Monitoring**
- Integrate Logfire instrumentation for FastAPI, OpenAI/Ollama clients, and custom pipeline spans.