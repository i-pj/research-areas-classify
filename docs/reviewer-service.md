# Reviewer Recommendation Service

This document defines the technical specification for the manuscript-to-reviewer matching pipeline. It operates as a distinct service endpoint (`POST /manuscripts/match-reviewers`) utilizing the shared infrastructure described in `architecture.md`.

---

## 1. Data Modeling

### 1.1 Qdrant `reviewer_profiles` Schema

The `reviewer_profiles` collection uses a 5-vector architecture to separate publication expertise from declared interests, plus sparse exact-match routing.

- **`pubs_dense_512` & `pubs_dense_4096`**: Dense representation of the concatenated abstracts of the reviewer's top 5 recent publications (sorted by citation count).
- **`interests_dense_512` & `interests_dense_4096`**: Dense representation of self-declared CRM keywords and PhD streams.
- **`bm25`**: Sparse IDF-weighted vector of both publications and interests.
- **`colbert_pubs`**: Token-level multi-vector representations (96-dim, `answerai-colbert-small-v1`) for precision rescoring.

**Payload Schema (with CoI Indexes):**
```json
{
  "user_id": 36234,  // Indexed (INTEGER)
  "openalex_id": "A5023888391",
  "display_name": "Dr. Anil Kushwah",
  "institution_ids": ["I4210139833"],  // Indexed (KEYWORD)
  "coauthor_openalex_ids": ["A5012345678", "A5098765432"], // Indexed (KEYWORD)
  "research_disciplines": ["Computer Science", "Engineering"],
  "publication_years": [2024, 2025, 2026],
  "has_publications": true
}
```

### 1.2 Endpoint Input Schema

```json
{
  "manuscript_id": "MS-2026-0142",
  "title": "NMLoNet: An End-to-End Intelligent Vehicle Localization Network",
  "abstract": "Accurate and reliable localization is crucial for advanced...",
  "keywords": ["autonomous vehicles", "localization", "bird's-eye-view"],
  "author_openalex_ids": ["A5023888391"],
  "author_institution_ids": ["I4210139833"],
  "max_reviewers": 10,
  "exclude_user_ids": [36234]
}
```

---

## 2. Ingestion Pipeline (`scripts/backfill_reviewers.py`)

Reviewers are vectorized into Qdrant either via a bulk backfill or event-driven hook from `/research-areas/classify`. 

**CoI Graph Construction (The OpenAlex `group_by` approach):**
To build the `coauthor_openalex_ids` array, the script executes a single server-side aggregation query to OpenAlex, isolating recent collaborations (last 5 years).

```http
GET https://api.openalex.org/works?filter=author.id:{id},publication_year:2021-2026&group_by=authorships.author.id
```
This returns all co-authors with frequency counts. The script extracts the IDs (excluding the target author) and embeds them into the Qdrant payload.

---

## 3. The 3-Stage Inference Pipeline

The core logic of `POST /manuscripts/match-reviewers` is a funnel that narrows ~4,000 candidates down to a highly relevant, justified shortlist.

### Stage 1: Hybrid Retrieval + CoI Exclusion
- **Action**: Query Qdrant with the manuscript abstract/keywords.
- **Vectors Used**: `pubs_dense_512`, `interests_dense_512`, `bm25`.
- **Fusion**: Reciprocal Rank Fusion (RRF).
- **Filter**: A strict `must_not` payload filter is applied during the prefetch to exclude conflicted reviewers.

**CoI Filter Rules:**
- Exclude reviewers matching manuscript `author_openalex_ids`
- Exclude reviewers possessing a matching ID in `coauthor_openalex_ids`
- Exclude reviewers at the same `institution_ids`
- Exclude specific `exclude_user_ids` (provided by CRM)

### Stage 2: ColBERT Late Interaction Rescoring
- **Action**: Rescore the top 50 candidates from Stage 1.
- **Vectors Used**: `colbert_pubs` (96-dim MaxSim evaluation).
- **Scoring**: `Final = (ColBERT * 0.6) + (Stage1_RRF * 0.4)`
- **Output**: Top 15 rescored candidates.

### Stage 3: LLM-as-a-Judge (`gemma4:12b`)
- **Action**: Pass the manuscript abstract and the top 15 reviewer profiles to the LLM via `instructor`.
- **Goal**: Generate structured, Editor-facing reasoning.

**Pydantic Output Schema:**
```python
class ReviewerJudgment(BaseModel):
    reasoning: str = Field(description="Step-by-step chain-of-thought reasoning.")
    expertise_overlap: str = Field(description="Specific aspects matching the manuscript.")
    methodological_fit: str = Field(description="Whether the reviewer has relevant methodological experience.")
    confidence_score: int = Field(ge=0, le=100, description="Raw confidence score (0-100).")
    justification_paragraph: str = Field(description="Concise paragraph explaining the recommendation for the editor.")
```

**Temporal Decay Post-Processing:**
Applied to the LLM's `confidence_score` based on the reviewer's most recent publication year.
`decay_weight = exp(-0.15 * (current_year - most_recent_pub_year))`
*Example: A reviewer whose last paper was 5 years ago receives a 0.47x multiplier to their confidence score.*
