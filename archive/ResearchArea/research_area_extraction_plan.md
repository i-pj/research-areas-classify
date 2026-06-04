# Research Area Extraction Plan

## 1. Objective

Build a reliable research-area extraction system for the author profile workflow.

The system receives author evidence from:

- Global Journal (GJ) API data
- Journal Press (LJP) API data
- Mautic marketing profile data
- Gmail/email-thread extracted profile data
- Existing n8n merged profile output

It must infer and store three taxonomy-backed research-area groups:

1. **Primary Research Disciplines** from ResearchNodes `/taxonomy/fields`
2. **Research Specializations** from ResearchNodes `/taxonomy/subfields`
3. **Research Keywords & Themes** from ResearchNodes `/keywords`

The result must be usable inside the existing n8n author profile flow and safe enough for profile updates at scale.

Primary goal: maximize correctness. Wrong taxonomy assignments are worse than missing assignments.

---

## 2. Current Flow Context

Existing n8n workflow:

- File: `/Users/apple/Development/NewPaperMetaExtractor/000_CompleteAuthorProfile/authorProfileFlow.json`
- The flow currently:
  - receives author data through `AuthorDetailsFromFlow`
  - fetches Mautic, GJ, LJP, and Gmail data
  - parses email threads with `05_Email_parser`
  - merges author data in `MergeAllLoopResult`
  - extracts identity with `06_BasicIdentityAndIdentifiers`
  - extracts affiliations/biography with `07_AcademicProfileAffiliationsBiography`
  - verifies institutions through OpenAlex search and `04_UniversityVerifier`
  - assembles final payload in `AllLoopData`
  - updates the author through `UpdateAuthor`

Important local files:

- `/Users/apple/Development/NewPaperMetaExtractor/000_CompleteAuthorProfile/MergeAllLoopResult/code.js`
- `/Users/apple/Development/NewPaperMetaExtractor/000_CompleteAuthorProfile/AllLoopData/AllLoopData.js`
- `/Users/apple/Development/NewPaperMetaExtractor/000_CompleteAuthorProfile/ResearchArea/fields.json`
- `/Users/apple/Development/NewPaperMetaExtractor/000_CompleteAuthorProfile/ResearchArea/subfields.json`
- `/Users/apple/Development/NewPaperMetaExtractor/000_CompleteAuthorProfile/ResearchArea/keywords.json`
- `/Users/apple/Development/NewPaperMetaExtractor/000_CompleteAuthorProfile/ResearchArea/user_Gj_details.json`
- `/Users/apple/Development/NewPaperMetaExtractor/000_CompleteAuthorProfile/ResearchArea/mautic.json`

Recommended integration point:

- Only the new research-area flow should move into a service.
- The existing n8n author profile flow should continue as-is.
- n8n should call the research service after the merge node and merge the returned three research arrays into the existing `update-user` payload.

Service input samples:

- New-author input: `/Users/apple/Development/NewPaperMetaExtractor/000_CompleteAuthorProfile/ResearchArea/inputForService.json`
- Old-author input: `/Users/apple/Development/NewPaperMetaExtractor/000_CompleteAuthorProfile/ResearchArea/inputfor2nd.json`

---

## 3. Research Area Service APIs

Only the new research-area flow should move into a backend service.

The existing author profile flow remains in n8n:

- identity extraction remains in n8n
- email parsing remains in n8n
- affiliation/biography extraction remains in n8n
- institution verification remains in n8n
- final `UpdateAuthor` call remains in n8n

The research service is called from n8n after the merge node has produced enough author evidence. The service returns only these three arrays:

- `research_disciplines`
- `research_specializations`
- `research_keywords`

n8n then merges these three arrays into the existing final profile payload before calling `update-user`.

Important request rule:

- n8n must send the **complete input object** to the research service.
- Do not pre-filter, trim, reshape, or drop fields before the service call.
- Every field present in `/Users/apple/Development/NewPaperMetaExtractor/000_CompleteAuthorProfile/ResearchArea/inputForService.json` must be forwarded to `/research-area-for-new`.
- Every field present in `/Users/apple/Development/NewPaperMetaExtractor/000_CompleteAuthorProfile/ResearchArea/inputfor2nd.json` must be forwarded to `/research-area-for-old`.
- The service may internally choose which fields are useful for research evidence, but the request payload must preserve the full original input.
- This prevents losing hidden evidence in old CRM fields, reviewer fields, nested article data, or future fields that are not documented yet.

The service must expose two APIs:

1. `POST /research-area-for-new`
2. `POST /research-area-for-old`

Recommended responsibility split:

- n8n owns the complete author profile workflow.
- research service owns taxonomy retrieval, reranking, verification, custom candidate policy, and hierarchy-safe output.
- n8n sends the returned three arrays to `update-user` with the rest of the existing profile payload.

### API 1: `POST /research-area-for-new`

Purpose:

- Generate research taxonomy arrays for a new/current author flow item.
- Input comes from the existing n8n merge output.
- Example input file: `/Users/apple/Development/NewPaperMetaExtractor/000_CompleteAuthorProfile/ResearchArea/inputForService.json`

Observed input shape:

```json
[
  {
    "country": null,
    "posts": [],
    "article_details": [],
    "output": {
      "author": {
        "full_name": "Yicheng Li",
        "primary_email": "liyicheng070@ujs.edu.cn",
        "emails": ["liyicheng070@ujs.edu.cn"],
        "email": "liyicheng070@ujs.edu.cn"
      },
      "personal_info": {},
      "social_media": {},
      "affiliations_work_experience": [],
      "education": [],
      "research_areas": [],
      "memberships": []
    }
  }
]
```

Actual request body:

- Send the full array/object exactly as n8n receives/builds it.
- The shape above is only a shortened documentation example.
- Do not send only `article_details` or only selected `output` fields.

New-flow processing:

- accept array or single object
- build research evidence from `article_details`, author keywords, classifications, journal metadata, and existing merged `output.research_areas`
- retrieve/rerank/verify research taxonomy
- complete required field/subfield/keyword hierarchy
- return only research arrays plus diagnostics

Required response:

```json
{
  "success": true,
  "mode": "new",
  "research_disciplines": [],
  "research_specializations": [],
  "research_keywords": [],
  "diagnostics": {
    "status": "completed",
    "unresolved_custom_candidates": []
  }
}
```

Only these three fields should be merged into `update-user` payload:

- `research_disciplines`
- `research_specializations`
- `research_keywords`

Diagnostics should be logged or sent to review, not stored in the author profile unless the API explicitly supports it.

### API 2: `POST /research-area-for-old`

Purpose:

- Generate or repair research taxonomy arrays for an old/existing author flow item.
- Input contains existing author profile fields plus `old_crm_data`.
- Example input file: `/Users/apple/Development/NewPaperMetaExtractor/000_CompleteAuthorProfile/ResearchArea/inputfor2nd.json`

Observed input shape:

```json
[
  {
    "user_id": 36234,
    "basic_information": {
      "email": "100438330@alumnos.uc3m.es",
      "display_name": "Shutao Lan",
      "country": "Spain"
    },
    "research_areas": [],
    "research_disciplines": [],
    "research_specializations": [],
    "research_keywords": [],
    "old_crm_data": {
      "canonical_email": "100438330@alumnos.uc3m.es",
      "canonical_name": "Shutao Lan",
      "source_roles": "reviewer",
      "university": "Universidad Carlos III de Madrid",
      "country": "Spain",
      "keywords": "Sustainable Tourism|Hospitality, Sport and Tourism Management",
      "qualification": "PhD student",
      "reviewer": []
    }
  }
]
```

Actual request body:

- Send the full old-author object exactly as available in n8n.
- The shape above is only a shortened documentation example.
- Do not remove large `old_crm_data` fields, nested `reviewer` entries, existing profile fields, empty fields, or unknown future fields.

Old-flow processing:

- treat existing `research_disciplines`, `research_specializations`, and `research_keywords` as base data
- if existing research arrays are valid and hierarchy-complete, preserve them
- if existing research arrays are empty or invalid, generate/repair them from evidence
- use `old_crm_data.keywords`, reviewer stream fields, journal, classification, title, abstract, and existing research text as evidence
- complete missing parent field/subfield relationships
- return only research arrays plus diagnostics

Required response:

```json
{
  "success": true,
  "mode": "old",
  "user_id": 36234,
  "research_disciplines": [],
  "research_specializations": [],
  "research_keywords": [],
  "diagnostics": {
    "status": "completed",
    "preserved_existing_research": false,
    "repaired_hierarchy": false,
    "unresolved_custom_candidates": []
  }
}
```

Only these three fields should be merged into `update-user` payload:

- `research_disciplines`
- `research_specializations`
- `research_keywords`

### New vs Old Research Flow Policy

| Area                     | New Research API                                 | Old Research API                         |
| ------------------------ | ------------------------------------------------ | ---------------------------------------- |
| Input source             | n8n merge output                                 | existing profile +`old_crm_data`       |
| Existing research arrays | usually empty                                    | may already exist                        |
| Preservation rule        | not applicable                                   | preserve valid hierarchy-complete arrays |
| Missing data behavior    | classify from available article/profile evidence | fill/repair only when missing or invalid |
| Custom candidates        | review only in v1                                | review only in v1                        |
| Output                   | three research arrays only                       | three research arrays only               |

### Service-Level Internal Modules

The service should be organized around reusable modules:

- `ResearchInputNormalizer`
  - accepts array or single object
  - detects `new` vs `old` mode from endpoint
  - strips HTML and malformed encodings
  - keeps the original full request object available as `raw_input`
  - must not destructively remove unknown or currently unused fields
- `ResearchEvidenceBuilder`
  - extracts title, abstract, keywords, classifications, journal metadata, reviewer streams, and existing research text
  - reads from `raw_input` so future source fields can be used without changing the n8n request contract
- `ResearchTaxonomyClassifier`
  - Qdrant search + API search + reranker + verifier
- `HierarchyValidator`
  - enforces `domain -> field -> subfield -> keyword`
- `ResearchPayloadBuilder`
  - returns exact three-array `update-user` shape
- `DiagnosticsLogger`
  - stores candidate scores, model versions, rejected candidates, and unresolved custom candidates
  - logs top-level input keys and selected evidence paths used
  - never logs secrets or sensitive tokens if they appear in the raw input

### API Error Response

If processing fails, return safe empty research arrays and diagnostics.

```json
{
  "success": false,
  "mode": "new",
  "research_disciplines": [],
  "research_specializations": [],
  "research_keywords": [],
  "diagnostics": {
    "status": "failed",
    "reason": "qdrant_unavailable"
  }
}
```

The service must never return hallucinated taxonomy data on failure.

---

## 4. ResearchNodes API Facts

Documentation:

- ResearchNodes API docs: https://uni.researchnodes.com/api-docs/
- Swagger JSON: https://uni.researchnodes.com/swagger.json

Confirmed endpoints:

### Taxonomy Fields

- `GET /taxonomy/fields`
- `GET /taxonomy/fields/search?q=...`
- `GET /taxonomy/fields/{id}`

Fields are the source for **Primary Research Disciplines**.

Fields should be treated as fixed controlled taxonomy. Do not create custom fields.

### Taxonomy Subfields

- `GET /taxonomy/subfields?status=active`
- `GET /taxonomy/subfields/search?q=...`
- `GET /taxonomy/subfields/{id}`
- `PUT /taxonomy/subfields/{id}` with API secret
- `DELETE /taxonomy/subfields/{id}` with API secret, draft only

Subfields are the source for **Research Specializations**.

### Keywords

- `GET /keywords?page=...&limit=...&status=active`
- `GET /keywords/search?q=...`
- `GET /keywords/{id}`
- `POST /keywords` with API secret
- `PUT /keywords/{id}` with API secret
- `DELETE /keywords/{id}` with API secret, draft only

Keywords are the source for **Research Keywords & Themes**.

`POST /keywords` request body:

```json
{
  "keywordName": "string",
  "fieldId": "string",
  "subfieldId": "string",
  "subfieldName": "string",
  "description": "string"
}
```

Security:

- Write endpoints require header `x-api-secret`.
- Secrets must be stored in n8n credentials or environment variables, never hardcoded in workflow JSON.

---

## 5. Recommended Retrieval Stack

Use embeddings. Qdrant alone does not provide semantic matching; it only stores/searches vectors.

Recommended dense embedding model:

- `Qwen/Qwen3-Embedding-8B`
- Dimension: `4096`
- Context length: `32K`
- Supports instruction-aware retrieval
- Supports user-defined output dimensions from `32` to `4096`
- Use full `4096` dimensions for this quality-sensitive task

Recommended reranker:

- `Qwen/Qwen3-Reranker-8B`
- Context length: `32K`
- Instruction-aware
- Use for reranking retrieved candidates before verifier LLM selection

Fallback rerankers:

- `Qwen/Qwen3-Reranker-4B` if 8B latency/cost is too high
- `BAAI/bge-reranker-v2-m3` if a lightweight multilingual fallback is required

Sources:

- Qwen3 Embedding model card: https://huggingface.co/Qwen/Qwen3-Embedding-8B
- Qwen3 Reranker model card: https://huggingface.co/Qwen/Qwen3-Reranker-8B
- BGE reranker model card: https://huggingface.co/BAAI/bge-reranker-v2-m3
- Qdrant vectors/named vectors: https://qdrant.tech/documentation/manage-data/vectors/
- Qdrant hybrid queries: https://qdrant.tech/documentation/concepts/hybrid-queries/

---

## 6. Why 4096D Embeddings Make Sense Here

Use `4096` dimensions for v1 because:

- taxonomy matching is quality-sensitive
- terms can be narrow and technical
- evidence may be short, noisy, or incomplete
- author data may combine titles, abstracts, journal names, and email snippets
- Qwen3-Embedding-8B is the highest-capacity Qwen3 embedding model
- taxonomy size is manageable: about 26 fields, 252 subfields, and about 69k keywords in the local sample

Tradeoff:

- `4096` float32 vectors for about 70k records require roughly 1.1GB raw vector storage before index overhead.
- Float16 can reduce memory usage, but float32 is safer for first implementation.
- If infrastructure becomes expensive, Qwen3 MRL support allows later migration to lower dimensions such as `2048` or `1024`.

Recommendation:

- Start with `4096` dimensions and cosine distance.
- Measure retrieval quality and latency.
- Only reduce dimensions after an evaluation set shows no quality loss.

---

## 7. Qdrant Collection Design

Collection name:

```text
research_taxonomy
```

Dense vector:

```json
{
  "vectors": {
    "qwen3_8b_4096": {
      "size": 4096,
      "distance": "Cosine"
    }
  }
}
```

Optional future vectors:

- `qwen3_8b_2048` for migration testing
- sparse vector for BM25/SPLADE-style hybrid retrieval
- late-interaction vector if future infrastructure supports it

Qdrant supports named vectors, so future migration can add a new vector without immediately rebuilding the whole collection.

Payload schema per point:

```json
{
  "kind": "field | subfield | keyword",
  "_id": "database object id",
  "id": "OpenAlex or ResearchNodes id",
  "display_name": "taxonomy name",
  "description": "taxonomy description",
  "display_name_alternatives": ["aliases"],
  "domain": {
    "id": "domain id",
    "display_name": "domain name"
  },
  "field": {
    "id": "field id",
    "display_name": "field name"
  },
  "subfield": {
    "id": "subfield id",
    "display_name": "subfield name"
  },
  "type": "topic | concept",
  "level": 0,
  "status": "active | draft",
  "source_api": "/taxonomy/fields | /taxonomy/subfields | /keywords",
  "embedding_model": "Qwen/Qwen3-Embedding-8B",
  "embedding_dimension": 4096,
  "last_synced_at": "ISO timestamp"
}
```

Point ID:

- Use deterministic point IDs.
- Recommended format:
  - `field::<id-or-_id>`
  - `subfield::<id-or-_id>`
  - `keyword::<id-or-_id>`

---

## 8. Taxonomy Text to Embed

Each taxonomy record should be converted into rich text before embedding.

Template:

```text
Kind: keyword
Name: Autonomous Vehicle Technology and Safety
Aliases: autonomous vehicles, self-driving vehicles
Description: ...
Domain: Engineering
Field: Engineering
Subfield: Automotive Engineering
Type: topic
Level: 1
```

Rules:

- Include hierarchy because hierarchy improves disambiguation.
- Include aliases because authors may use alternate terms.
- Include descriptions because short display names alone are often ambiguous.
- Do not include unrelated metadata like `works_count` in embedding text.
- Keep `works_count` and `cited_by_count` in payload for tie-breaking only.

---

## 9. Author Evidence Text to Embed

Build a compact evidence object from the author profile.

Use these sources:

- article titles
- article abstracts
- author-provided article keywords
- classifications
- journal title and journal part
- Mautic title fields
- email-extracted research areas
- existing profile research area text
- biography if it contains explicit research focus

Do not use:

- institution name alone as research evidence
- country alone
- journal publisher boilerplate
- unrelated people from email threads

Recommended query text:

```text
Author: Yicheng Li
Paper titles:
- NMLoNet: An End-to-End Intelligent Vehicle Localization Network Using Navigation Maps
- Pedestrian Multi-Object Tracking Combining Appearance and Spatial Characteristics
- Robust Pedestrian Multi-Object Tracking in the Intelligent Bus Environment
Abstract evidence:
Accurate and reliable localization is crucial for advanced autonomous driving systems...
Author keywords:
bird’s-eye-view, high-precision localization, Intelligent Vehicles, navigation map, visual localization
Classifications:
arXiv cs.CV, arXiv cs.RO, ACM I.2.10, IEEE 8.8
Journal:
Global Journal of Computer Science and Technology - E: Network, Web & Security
```

Instruction for query embedding:

```text
Instruct: Retrieve research taxonomy records that best match an academic author's actual research discipline, specialization, and technical themes. Prefer precise existing taxonomy terms over broad or weakly related terms.
Query: {author evidence text}
```

---

## 10. Final `update-user` Payload Contract

The final data sent to `UpdateAuthor` / `update-user` must use these exact arrays:

- `research_disciplines`
- `research_specializations`
- `research_keywords`

If the existing database/API expects `research_keywords_data` for custom keyword storage, map the same keyword array to that field at the final adapter layer. The canonical internal output should remain `research_keywords` unless the API contract confirms otherwise.

### Required Item Shape

Every returned item must use this shape:

```json
{
  "id": "https://openalex.org/fields/17",
  "name": "Computer Science",
  "domain_id": "https://openalex.org/domains/3",
  "field_id": "https://openalex.org/fields/17",
  "subfield_id": "",
  "type": ""
}
```

Field rules:

- `id`: selected taxonomy ID or custom ID.
- `name`: selected taxonomy display name.
- `domain_id`: parent domain ID.
- `field_id`: parent field ID.
- `subfield_id`: parent subfield ID, empty string for primary disciplines.
- `type`: empty string for fields/subfields; keyword type for keywords; `custom` for created custom records.

### `research_disciplines`

Primary disciplines come only from `/taxonomy/fields`.

Example:

```json
{
  "research_disciplines": [
    {
      "id": "https://openalex.org/fields/17",
      "name": "Computer Science",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "",
      "type": ""
    }
  ]
}
```

Mapping:

- `id` = field `id`
- `name` = field `display_name`
- `domain_id` = field `domain.id`
- `field_id` = field `id`
- `subfield_id` = `""`
- `type` = `""`

### `research_specializations`

Specializations come from `/taxonomy/subfields`.

Example:

```json
{
  "research_specializations": [
    {
      "id": "https://openalex.org/subfields/1702",
      "name": "Artificial Intelligence",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1702",
      "type": ""
    }
  ]
}
```

Mapping:

- `id` = subfield `id`
- `name` = subfield `display_name`
- `domain_id` = subfield `domain.id`
- `field_id` = subfield `field.id`
- `subfield_id` = subfield `id`
- `type` = `""`

### `research_keywords`

Keywords/themes come from `/keywords`.

Example:

```json
{
  "research_keywords": [
    {
      "id": "https://openalex.org/T10054",
      "name": "Parallel Computing and Optimization Techniques",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1708",
      "type": "topic"
    }
  ]
}
```

Mapping:

- `id` = keyword `id`
- `name` = keyword `display_name`
- `domain_id` = keyword `domain.id`
- `field_id` = keyword `field.id`
- `subfield_id` = keyword `subfield.id`
- `type` = keyword `type`, usually `topic` or `concept`

### Custom Keyword Shape

Custom records are allowed only when no suitable existing match is found.

Example:

```json
{
  "research_keywords": [
    {
      "id": "custom/1780389689746",
      "name": "anolog",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1709",
      "type": "custom"
    }
  ]
}
```

Rules:

- `type` must be `custom` only for created custom records.
- Custom ID must be returned from the creation layer if the API returns one.
- If the API does not return a custom ID, generate deterministic temporary IDs only for review workflows, not permanent profile storage.
- Custom keyword must still have valid `domain_id`, `field_id`, and `subfield_id`.

### Complete Final Payload Example

```json
{
  "research_disciplines": [
    {
      "id": "https://openalex.org/fields/17",
      "name": "Computer Science",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "",
      "type": ""
    }
  ],
  "research_specializations": [
    {
      "id": "https://openalex.org/subfields/1708",
      "name": "Hardware and Architecture",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1708",
      "type": ""
    },
    {
      "id": "https://openalex.org/subfields/1702",
      "name": "Artificial Intelligence",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1702",
      "type": ""
    },
    {
      "id": "https://openalex.org/subfields/1705",
      "name": "Computer Networks and Communications",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1705",
      "type": ""
    },
    {
      "id": "https://openalex.org/subfields/1707",
      "name": "Computer Vision and Pattern Recognition",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1707",
      "type": ""
    }
  ],
  "research_keywords": [
    {
      "id": "https://openalex.org/T10054",
      "name": "Parallel Computing and Optimization Techniques",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1708",
      "type": "topic"
    },
    {
      "id": "https://openalex.org/T14067",
      "name": "Cloud Computing and Remote Desktop Technologies",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1708",
      "type": "topic"
    },
    {
      "id": "https://openalex.org/T11032",
      "name": "VLSI and Analog Circuit Testing",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1708",
      "type": "topic"
    }
  ]
}
```

---

## 11. Required Hierarchy and Relationship Rules

Relationships are mandatory. Do not return orphan records.

Required hierarchy:

```text
 field -> subfield -> keyword
```

Rules:

- Every `research_disciplines[]` item must be a field.
- Every `research_specializations[]` item must belong to a selected or implied field.
- Every `research_keywords[]` item must belong to a selected or implied subfield.
- Every keyword must have a non-empty `field_id` and `subfield_id`.
- Every subfield must have a non-empty `field_id`.
- A keyword cannot be returned if its parent subfield is missing from `research_specializations`, unless the final adapter also adds that parent subfield.
- A subfield cannot be returned if its parent field is missing from `research_disciplines`, unless the final adapter also adds that parent field.
- If a selected keyword points to a field/subfield not selected by the LLM, the hierarchy completion step must add the missing parent field/subfield automatically.
- If a selected custom keyword is created, it must be attached to a valid existing field and subfield.

Validation pseudocode:

```javascript
function completeHierarchy(result, taxonomyById) {
  for (const keyword of result.research_keywords) {
    if (!keyword.field_id || !keyword.subfield_id) {
      throw new Error(`Keyword ${keyword.name} is missing field_id or subfield_id`);
    }

    ensureFieldExists(result.research_disciplines, keyword.field_id, taxonomyById);
    ensureSubfieldExists(result.research_specializations, keyword.subfield_id, taxonomyById);
  }

  for (const subfield of result.research_specializations) {
    if (!subfield.field_id) {
      throw new Error(`Subfield ${subfield.name} is missing field_id`);
    }

    ensureFieldExists(result.research_disciplines, subfield.field_id, taxonomyById);
  }

  return result;
}
```

If hierarchy cannot be completed because parent metadata is unavailable:

- reject that child item
- add a diagnostic reason
- do not send invalid relationship data to `update-user`

---

## 12. Retrieval Pipeline

Use multi-stage retrieval, not raw vector search only.

### Stage 1: Evidence Builder

Create a code node or service function named `BuildResearchEvidence`.

Input:

- merged author record from `MergeAllLoopResult`
- current author record from `AuthorDetailsFromFlow`
- outputs from email parser and academic profile parser where available

Output:

```json
{
  "author_name": "Yicheng Li",
  "evidence_text": "...",
  "phrases": [
    "vehicle localization",
    "navigation maps",
    "computer vision",
    "pedestrian multi-object tracking"
  ],
  "classifications": [
    "arXiv cs.CV",
    "arXiv cs.RO",
    "ACM I.2.10"
  ],
  "source_fields_used": [
    "article_title",
    "article_abstract",
    "article_keywords",
    "classification",
    "mautic.titleOfPaper2"
  ]
}
```

### Stage 2: Dense Retrieval from Qdrant

Run separate searches by `kind`:

- fields: top `8`
- subfields: top `20`
- keywords: top `50`

Filters:

- field search: `kind = field`
- subfield search: `kind = subfield`
- keyword search: `kind = keyword`

If fields are already selected, filter subfields by selected field IDs where possible.

If subfields are already selected, boost or filter keywords by selected subfield/field IDs where possible.

### Stage 3: Exact API Search

Run exact/lexical API searches for extracted phrases:

- `/taxonomy/fields/search?q={phrase}`
- `/taxonomy/subfields/search?q={phrase}`
- `/keywords/search?q={phrase}`

Reason:

- embeddings may miss exact abbreviations
- API search can catch known taxonomy entries by name
- classifications such as `cs.CV` should map to known disciplines even if not semantically obvious

### Stage 4: Candidate Merge

Merge Qdrant and API candidates.

Dedupe keys:

- first by `id`
- fallback by `_id`
- fallback by normalized `kind + display_name`

Preserve provenance:

```json
{
  "retrieval_sources": ["qdrant_dense", "api_search"],
  "dense_score": 0.82,
  "api_match": true
}
```

### Stage 5: Reranking

Use `Qwen/Qwen3-Reranker-8B`.

Run reranking separately for:

- fields
- subfields
- keywords

Field reranker instruction:

```text
Classify whether this taxonomy field is a correct primary research discipline for the author, based only on the provided author evidence.
```

Subfield reranker instruction:

```text
Classify whether this taxonomy subfield is a precise research specialization for the author, based only on the provided author evidence.
```

Keyword reranker instruction:

```text
Classify whether this taxonomy keyword/theme is explicitly or strongly supported by the author's paper titles, abstracts, keywords, classifications, or profile evidence.
```

Rerank enough candidates:

- fields: rerank all retrieved candidates
- subfields: rerank top `20-30`
- keywords: rerank top `50-80`

### Stage 6: Verifier LLM

The verifier gets:

- author evidence
- reranked candidates
- candidate metadata
- hierarchy relationships
- existing profile research text

The verifier must:

- choose only supported records
- reject overly broad matches when precise matches exist
- reject unrelated matches caused by shared words
- preserve taxonomy IDs
- output confidence and evidence
- identify unresolved custom candidates but not create them

Verifier must not:

- invent IDs
- use external knowledge not present in candidates
- create custom fields
- create custom keywords directly

### Stage 7: Optional Custom Candidate Resolution

Only after verifier says no existing suitable match was found:

1. Search Qdrant again with the custom phrase.
2. Search live API again.
3. If no suitable existing result is found, mark the phrase as an `unresolved_custom_candidate`.
4. Only a separate controlled node should call `POST /keywords`.

Do not auto-create custom records in the main classification node.

---

## 13. Custom Creation Policy

Custom records are created only when existing data is not found.

Strict rules:

- Never create custom primary disciplines.
- Prefer existing subfields and keywords.
- Do not create a custom record if a close existing taxonomy match exists.
- Do not create a custom record from weak evidence.
- Do not create a custom record from a single ambiguous word.
- Do not create a custom record just because the author phrase is more specific than the taxonomy.

Allowed custom keyword case:

- evidence phrase is explicit in title, abstract, keywords, or repeated paper titles
- Qdrant retrieval fails to find a suitable record
- live API search fails to find a suitable record
- verifier confirms no existing candidate is good enough
- parent field is confidently known
- parent subfield is known or a genuinely missing specialization is proposed

Recommended custom write behavior:

- Create as draft/review if API supports status.
- If API does not expose draft creation in `POST /keywords`, keep as `unresolved_custom_candidates` and route to a manual review workflow.
- Store enough evidence for human approval.

Custom keyword payload:

```json
{
  "keywordName": "Navigation-map-based vehicle localization",
  "fieldId": "https://openalex.org/fields/17",
  "subfieldId": "https://openalex.org/subfields/1707",
  "description": "Theme inferred from author-provided paper title, abstract, and keywords about vehicle localization using navigation maps."
}
```

---

## 14. Supporting Service Operations

The service has exactly two research-area APIs for n8n:

1. `POST /research-area-for-new`
2. `POST /research-area-for-old`

Do not expose alternate research-area endpoint names as the main contract; n8n should use only the two endpoint names above.

The shared internal pipeline should be implemented once and reused by both APIs:

```text
Normalize service input
  -> Build research evidence
  -> Retrieve candidates from Qdrant
  -> Search live ResearchNodes API
  -> Merge and dedupe candidates
  -> Rerank candidates
  -> Verify selections
  -> Complete hierarchy
  -> Format the three update-user research arrays
```

### Internal Function: `classifyResearchAreas`

This should be a reusable internal function, not a separate public API in v1 unless debugging requires it.

Inputs:

- normalized author evidence
- retrieval options
- custom creation policy

Output:

```json
{
  "research_disciplines": [],
  "research_specializations": [],
  "research_keywords": [],
  "unresolved_custom_candidates": [],
  "diagnostics": {}
}
```

Both `/research-area-for-new` and `/research-area-for-old` must call this internal function.

### Admin Operation: Taxonomy Sync

Taxonomy sync is required, but it should be treated as an admin/background operation, not one of the two n8n research-area APIs.

Possible implementation options:

- scheduled background job
- CLI command
- protected admin endpoint such as `POST /admin/taxonomy/sync`

Purpose:

- keep Qdrant taxonomy vectors updated from ResearchNodes
- sync fields, subfields, keywords, and newly created custom records

Admin sync request shape, if implemented as an endpoint:

```json
{
  "scope": "all | fields | subfields | keywords",
  "status": "active",
  "force": false,
  "page_size": 500,
  "include_drafts": false
}
```

Admin sync response shape:

```json
{
  "success": true,
  "collection": "research_taxonomy",
  "embedding_model": "Qwen/Qwen3-Embedding-8B",
  "embedding_dimension": 4096,
  "synced": {
    "fields": 26,
    "subfields": 252,
    "keywords": 69542
  },
  "created": 0,
  "updated": 0,
  "skipped": 0,
  "last_synced_at": "ISO timestamp"
}
```

Rules:

- sync must be idempotent
- use deterministic Qdrant point IDs
- do not run full taxonomy sync for every author request
- run sync daily, before large batches, or after custom keyword creation
- if custom keyword creation happens, sync that new keyword immediately or upsert it directly into Qdrant

---

## 15. n8n vs Codebase: Pros and Cons

### Option A: Build Fully in n8n

Pros:

- fastest to connect with current workflow
- easy to inspect and modify visually
- low initial engineering overhead
- existing API calls and final profile update already live in n8n
- non-developers can trace the flow
- good for quick iteration and operational visibility

Cons:

- complex retrieval logic becomes hard to maintain in many nodes
- taxonomy sync, pagination, embedding batches, retries, and Qdrant upserts are awkward in n8n
- version control and code review are weaker than a codebase
- unit testing is limited
- error handling can become fragmented across nodes
- reranking and verifier logic may become expensive and hard to optimize
- large JSON payloads and loops can make workflow runs slow/noisy

Best use:

- orchestration
- calling external services
- triggering per-author jobs
- updating final user profile
- monitoring failures at workflow level

### Option B: Build as a Codebase Service

Pros:

- better maintainability for retrieval logic
- easier unit/integration testing
- easier to batch taxonomy sync
- better retry/backoff/idempotency control
- simpler Qdrant schema management
- cleaner model abstraction for embeddings/reranker/verifier
- easier benchmark/evaluation harness
- safer custom creation policy enforcement
- better observability with logs and metrics
- easier to reuse outside this one n8n flow

Cons:

- more upfront engineering work
- requires deployment, hosting, environment management
- adds another service dependency
- n8n users cannot visually edit internal logic
- needs API authentication between n8n and service

Best use:

- taxonomy syncing
- embedding generation
- Qdrant upsert/search
- reranking
- verifier prompt execution
- custom candidate policy
- evaluation and quality monitoring

### Option C: Research Service With n8n Caller

Build only the research-area taxonomy flow as a codebase service. Keep the existing complete author profile workflow in n8n.

Recommended.

n8n responsibilities:

- continue running the existing complete author profile flow
- call `/research-area-for-new` after the merge node for new/current merged author evidence
- call `/research-area-for-old` for old CRM/profile research updates
- merge returned `research_disciplines`, `research_specializations`, and `research_keywords` into the existing `UpdateAuthor` payload
- route unresolved custom candidates for review

Codebase service responsibilities:

- implement `/research-area-for-new`
- implement `/research-area-for-old`
- implement admin/background taxonomy sync
- normalize new-flow and old-flow research input shapes
- sync ResearchNodes taxonomy into Qdrant
- generate Qwen embeddings
- query Qdrant
- call live ResearchNodes search endpoints
- rerank candidates
- run verifier
- enforce custom creation policy
- return the three final research arrays plus diagnostics

Why research service is best:

- avoids putting complex research ranking logic into fragile workflow nodes
- gives enough testability for a quality-sensitive task
- supports future model changes without editing n8n
- can be used by other systems later

Public research-area APIs are defined in section 3.

---

## 16. Final Recommendation

Use a **research-service approach**:

- keep the existing complete author profile flow in n8n
- build only the research-area taxonomy logic inside one codebase service
- expose two n8n research-area APIs: `/research-area-for-new` and `/research-area-for-old`; taxonomy sync remains admin/background
- call the service from n8n after the merge node
- use Qdrant with `Qwen/Qwen3-Embedding-8B` at `4096` dimensions
- use `Qwen/Qwen3-Reranker-8B` for reranking
- use live ResearchNodes API search as a second retrieval signal
- use verifier LLM as final gate
- create custom records only through a controlled path when no existing match exists

Do not build research retrieval/ranking/custom-write logic directly in n8n.

If a short-term prototype is needed:

1. Build only temporary HTTP calls from n8n to `/research-area-for-new` and `/research-area-for-old`.
2. Do not implement custom creation yet.
3. Return `unresolved_custom_candidates`.
4. Keep the production research implementation in the service.

---

## 17. Decision Matrix

| Criterion                          | Fully n8n           | Research Service | Research Service + n8n |
| ---------------------------------- | ------------------- | ---------------- | ---------------------- |
| Initial speed                      | Best                | Slowest          | Medium                 |
| Long-term maintainability          | Weak                | Best             | Best                   |
| Testing                            | Weak                | Best             | Best                   |
| Workflow visibility                | Best                | Weak             | Good                   |
| Retrieval quality control          | Medium              | Best             | Best                   |
| Qdrant/index management            | Weak                | Best             | Best                   |
| Embedding/reranker model migration | Weak                | Best             | Best                   |
| Custom creation safety             | Medium              | Best             | Best                   |
| Operational simplicity             | Good at small scale | Medium           | Medium                 |
| Production suitability             | Prototype only      | Strong           | Strongest fit          |

Decision:

- Use **research service + n8n** for production.
- Keep n8n as the existing author-flow owner, but not as the owner of research ranking logic.
- Use **n8n-only** only for a short proof-of-concept.
- Avoid making n8n the owner of taxonomy sync, embedding batches, Qdrant schema, reranker orchestration, and custom write policy.

---

## 18. Additional Suggestions and Improvements

These are not required for v1, but they will improve quality and maintainability.

### Build an Evaluation Dataset

Create a small labeled dataset before production rollout.

Recommended shape:

```json
{
  "author_id": "123",
  "evidence": {},
  "expected_fields": ["https://openalex.org/fields/17"],
  "expected_subfields": ["https://openalex.org/subfields/1707"],
  "expected_keywords": ["https://openalex.org/C31972630"],
  "notes": "Ground truth manually reviewed"
}
```

Start with:

- 20 easy authors
- 20 ambiguous authors
- 20 authors with weak/missing abstracts
- 20 authors across non-computer-science journals

Track:

- field precision
- subfield precision
- keyword precision
- false positive rate
- unresolved custom candidate rate

### Add Classification-Aware Rules

Some article classifications should become deterministic hints.

Examples:

- `arXiv cs.CV` strongly boosts `Computer Vision and Pattern Recognition`
- `arXiv cs.RO` strongly boosts robotics/autonomous systems related candidates
- `ACM I.2.10` strongly boosts artificial intelligence related candidates

These rules should boost candidates, not directly bypass verifier checks.

### Add Abbreviation Expansion

Expand common technical abbreviations before retrieval.

Examples:

- `BEV` → `bird's-eye-view`
- `MOT` → `multi-object tracking`
- `SLAM` → `simultaneous localization and mapping`
- `CV` → `computer vision`
- `AI` → `artificial intelligence`

Keep expansions in diagnostics so the verifier can audit why candidates were retrieved.

### Use Hierarchical Consistency Checks

Selected records should be hierarchy-compatible.

Examples:

- If selected keyword has `field = Computer Science`, ensure `Computer Science` appears in primary disciplines unless another stronger parent field exists.
- If selected subfield belongs to `Computer Science`, boost that field.
- If a keyword belongs to a selected subfield, boost that keyword.

Do not force a parent only because one keyword exists; use hierarchy as a consistency signal.

### Add Human Review Queue

Create a review queue for:

- custom candidates
- confidence below threshold but above rejection
- conflicting field/subfield hierarchy
- high-impact profile updates

Suggested review item:

```json
{
  "author_id": "123",
  "candidate": "Cross-modal feature registration",
  "candidate_type": "custom_keyword",
  "evidence": ["paper abstract", "paper title"],
  "nearest_existing_matches": [],
  "verifier_reason": "No suitable existing keyword found"
}
```

### Store Diagnostics Separately

Do not overload the author profile with internal debugging data.

Recommended:

- profile stores final taxonomy result and readable summary
- diagnostics log stores candidates, scores, prompts, model names, and verifier reasons

### Add Model Versioning

Every response should include:

- embedding model
- embedding dimension
- reranker model
- verifier model
- taxonomy sync timestamp
- taxonomy source version if available

This is needed for reproducibility.

### Add Safe Degradation

If Qdrant or model service fails:

- do not hallucinate research areas
- return empty arrays with an error diagnostic
- let the profile update continue only if existing flow allows partial updates safely

Example:

```json
{
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

## 19. Quality Controls

Required controls:

- confidence scores on every selected record
- evidence list for every selected record
- model/version metadata in every response
- unresolved candidates separated from selected taxonomy records
- no custom primary discipline creation
- no custom keyword creation without live API search fallback
- idempotent custom candidate handling
- logging of candidate lists and verifier decisions

Recommended thresholds:

- fields:
  - accept if confidence `>= 0.75`
  - allow secondary broad field if confidence `>= 0.65` and evidence is strong
- subfields:
  - accept if confidence `>= 0.75`
  - reject broad/generic subfields if precise alternatives exist
- keywords:
  - accept if confidence `>= 0.70`
  - allow more keywords than fields/subfields, but cap final list to `5-12`
- custom candidates:
  - require confidence `>= 0.85`
  - require explicit source evidence
  - require no existing API match

These thresholds should be tuned with real evaluation data.

---

## 20. Example Output for Provided User

Evidence:

- Author: `Yicheng Li`
- Paper: `NMLoNet: An End-to-End Intelligent Vehicle Localization Network Using Navigation Maps`
- Keywords: `bird’s-eye-view`, `high-precision localization`, `Intelligent Vehicles`, `navigation map`, `visual localization`
- Classifications: `ACM I.2.10`, `arXiv cs.CV`, `arXiv cs.RO`
- Mautic titles:
  - `Pedestrian Multi-Object Tracking Combining Appearance and Spatial Characteristics`
  - `Robust Pedestrian Multi-Object Tracking in the Intelligent Bus Environment`

Expected result:

```json
{
  "research_disciplines": [
    {
      "id": "https://openalex.org/fields/17",
      "name": "Computer Science",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "",
      "type": ""
    },
    {
      "id": "https://openalex.org/fields/22",
      "name": "Engineering",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/22",
      "subfield_id": "",
      "type": ""
    }
  ],
  "research_specializations": [
    {
      "id": "https://openalex.org/subfields/1707",
      "name": "Computer Vision and Pattern Recognition",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1707",
      "type": ""
    },
    {
      "id": "https://openalex.org/subfields/1702",
      "name": "Artificial Intelligence",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1702",
      "type": ""
    },
    {
      "id": "https://openalex.org/subfields/2203",
      "name": "Automotive Engineering",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/22",
      "subfield_id": "https://openalex.org/subfields/2203",
      "type": ""
    }
  ],
  "research_keywords": [
    {
      "id": "https://openalex.org/T12549",
      "name": "Image and Object Detection Techniques",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1707",
      "type": "topic"
    },
    {
      "id": "https://openalex.org/T10320",
      "name": "Neural Networks and Applications",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1702",
      "type": "topic"
    },
    {
      "id": "https://openalex.org/T11099",
      "name": "Autonomous Vehicle Technology and Safety",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/22",
      "subfield_id": "https://openalex.org/subfields/2203",
      "type": "topic"
    },
    {
      "id": "https://openalex.org/T10586",
      "name": "Robotic Path Planning Algorithms",
      "domain_id": "https://openalex.org/domains/3",
      "field_id": "https://openalex.org/fields/17",
      "subfield_id": "https://openalex.org/subfields/1707",
      "type": "topic"
    }
  ],
  "unresolved_custom_candidates": [
    "Navigation-map-based vehicle localization",
    "Bird’s-eye-view perception",
    "Cross-modal feature registration",
    "Pedestrian multi-object tracking"
  ]
}
```

Note:

- The unresolved custom candidates are not automatically created.
- They are useful, but should only become custom keywords after Qdrant + live API search + verifier confirm no existing match.
- Concept keywords without `field_id` and `subfield_id` should not be sent to `update-user` unless the hierarchy completion step attaches a verified parent field and subfield.

---

## 21. Implementation Phases

### Phase 1: Documentation and Prototype

- Create this markdown context file.
- Build a local/offline prototype using `fields.json`, `subfields.json`, and `keywords.json`.
- Produce sample outputs for 5-10 known authors.
- Do not create custom records.

### Phase 2: Qdrant Taxonomy Index

- Create Qdrant collection.
- Build taxonomy sync script/service.
- Embed taxonomy records with Qwen3-Embedding-8B.
- Upsert fields, subfields, and keywords.
- Store sync metadata.

### Phase 3: ResearchAreaService

- Implement `/research-area-for-new`.
- Implement `/research-area-for-old`.
- Implement shared internal `classifyResearchAreas`.
- Build evidence extraction.
- Query Qdrant by taxonomy kind.
- Query live ResearchNodes API search.
- Rerank candidates.
- Run verifier.
- Return structured output.

### Phase 4: Service Integration

- Add optional n8n HTTP node after profile merge.
- Send merged evidence to `/research-area-for-new`.
- Send old CRM/profile research evidence to `/research-area-for-old`.
- Merge returned `research_disciplines`, `research_specializations`, and `research_keywords` into the existing final n8n author update payload.
- Keep unresolved custom candidates visible in diagnostics.

### Phase 5: Controlled Custom Creation

- Add separate review/approval path.
- Create custom keywords only after strict checks.
- Sync newly created custom records back into Qdrant.
- Log every custom write with evidence and model output.

### Phase 6: Evaluation and Monitoring

- Build evaluation set from real authors.
- Track precision by field/subfield/keyword.
- Track unresolved candidate rate.
- Track custom creation rate.
- Review low-confidence outputs manually.
- Tune thresholds and prompts.

---

## 22. Acceptance Criteria

The implementation is acceptable when:

- selected taxonomy IDs exist in ResearchNodes
- output includes all three groups
- output includes confidence and evidence
- n8n forwards the complete merge/old-author input object to the research service
- no input fields are dropped before the service call
- the service keeps the full raw request internally for evidence extraction and diagnostics
- no custom field is ever created
- custom keyword creation is blocked unless no existing match exists
- final author payload remains backward compatible
- the same input produces deterministic selected taxonomy records unless taxonomy/model versions change
- failures return safe empty arrays rather than hallucinated taxonomy

---

## 23. Open Questions for Implementation

These should be resolved before production deployment:

1. What exact database field names should `UpdateAuthor` receive for research taxonomy?
2. Should unresolved custom candidates be stored on the author profile or only sent to review?
3. Does `POST /keywords` support creating draft records, or are all created records active?
4. Which infrastructure will host Qwen3 embedding and reranker models?
5. Is latency more important than maximum quality for per-author runs?
6. Should taxonomy sync run hourly, daily, or only before profile batches?

Default assumptions:

- store final update payload under `research_disciplines`, `research_specializations`, and `research_keywords`
- store readable summary under `research_areas` only if the existing profile API still needs a legacy text summary
- do not auto-create custom records in v1
- run taxonomy sync daily or manually before large batches
- optimize for quality over latency
