# Input Payload Field Map

This document maps every significant field in the n8n merged payload to its data source and explains how the research service should use it.

See `input-from-n8n.json` for a complete sample payload.

---

## Data Sources in the Payload

The n8n `Merge1` node combines data from these sources:

| Source | Description | Reliability |
|---|---|---|
| **CRM WordPress** | Author profile from the WordPress-based CRM system. Fields like `basic_information`, `biography`, `academic_ids`, `affiliations_work_experience`, `education`, etc. These have been processed by earlier n8n LLM steps (06_BasicIdentityAndIdentifiers, 07_AcademicProfileAffiliationsBiography). | Trusted — accept as-is |
| **Legacy CRM** | Raw database dump from the old CRM system at `old_crm_data`. Massive flat object with 300+ fields. Contains reviewer data, author data, and manuscript metadata. | Trusted — raw source data, may have encoding artifacts |
| **Mautic (Marketing)** | Marketing platform data at `authorData`. Contains paper titles, university, country. | Trusted — factual database fields |
| **Journal API (GJ/LJP)** | Article details at `article_details[]`. Contains paper titles, abstracts, keywords, classifications. **Only present for authors who have submitted papers.** | Trusted — Gold standard for research classification |
| **Email threads** | Author correspondence at `emails[]`. Contains email text and HTML. | Trusted — but noisy; use as last resort for evidence |

---

## Top-Level Fields

| Field | Source | Used For | Notes |
|---|---|---|---|
| `user_id` | CRM WordPress | `author_details.user_id` | WordPress user ID. Always present. |
| `basic_information` | CRM WordPress (LLM-processed) | `author_details.basic_information` | Name, email, country, phone. Trust as-is. |
| `biography` | CRM WordPress (LLM-processed) | `author_details.biography` | May be empty. Trust as-is. |
| `academic_ids` | CRM WordPress | `author_details.academic_ids` | ORCID, ResearchGate, Google Scholar URLs. Often empty. |
| `social_academic_links` | CRM WordPress | `author_details.social_links` | LinkedIn, Twitter, etc. **Note**: Key names differ from output schema (e.g., `linkedin_url` → `linkedin`). |
| `affiliations_work_experience` | CRM WordPress (LLM-processed) | `author_details.affiliations` | May be empty or have placeholder objects. Trust as-is. |
| `education` | CRM WordPress (LLM-processed) | `author_details.education` | Usually empty. Trust as-is. |
| `editor_roles` | CRM WordPress | `author_details.editor_roles` | Usually empty. Trust as-is. |
| `memberships` | CRM WordPress | `author_details.memberships` | Usually empty. Trust as-is. |
| `advisors` | CRM WordPress | `author_details.advisors` | Usually empty. Trust as-is. |
| `advisees` | CRM WordPress | `author_details.advisees` | Usually empty. Trust as-is. |
| `grants_awards` | CRM WordPress | `author_details.grants_awards` | Usually empty. Trust as-is. |
| `research_areas` | CRM WordPress | Reference only | Array of strings. May be from prior LLM extraction. |
| `research_disciplines` | CRM WordPress | Early exit check | If already populated with valid data, may skip processing. |
| `research_specializations` | CRM WordPress | Early exit check | Same as above. |
| `research_keywords` | CRM WordPress | Early exit check | Same as above. |
| `patents` | CRM WordPress | `author_details.patents` | Usually empty. Trust as-is. |
| `interest` | CRM WordPress | Evidence extraction | Array of interest strings. |

---

## `old_crm_data` — Key Fields

The `old_crm_data` object contains 300+ fields from the legacy CRM. Most are irrelevant. These are the ones that matter:

### Author/Profile Identity
| Field | Description | Used For |
|---|---|---|
| `canonical_email` | Canonical email address | Email deduplication |
| `canonical_name` | Canonical display name | Verification |
| `source_roles` | Role in the system: `"reviewer"`, `"author"`, `"editor"`, `"fellow"` | Determines if author has papers |
| `source_tables` | Source database table | Diagnostic |
| `author_record_count` | Number of author records (papers submitted) | If `"0"`, this is a reviewer-only profile |
| `reviewer_record_count` | Number of reviewer records | Indicates reviewer activity |
| `university` | University from legacy CRM | Affiliation data |
| `country` | Country from legacy CRM | Location data |
| `qualification` | e.g., `"PhD student"`, `"Professor"` | Can inform `position_role` in affiliations |

### Research Evidence (Tier 2)
| Field | Description | Used For |
|---|---|---|
| `keywords` | Pipe-separated research keywords (e.g., `"Sustainable Tourism\|Hospitality..."`) | Research classification evidence. **Warning**: May have encoding artifacts (`Â` characters). Use LLM to clean. |
| `classification` | Classification codes (if author has papers in old system) | Classification hints for taxonomy boost |

### Reviewer-Specific Data
| Field | Path | Description | Used For |
|---|---|---|---|
| `phdstream` | `old_crm_data.reviewer[].phdstream` | PhD research stream | Research classification evidence (Tier 2) |
| `stream` | `old_crm_data.reviewer[].stream` | Research stream | Research classification evidence (Tier 2). **Warning**: Encoding artifacts. |
| `university` | `old_crm_data.reviewer[].university` | Reviewer's university | Affiliation data |
| `email2` | `old_crm_data.reviewer[].email2` | Secondary email | Additional email collection |
| `qualification` | `old_crm_data.reviewer[].qualification` | e.g., `"PhD student"` | Can inform `position_role` |
| `nofpaperspubint` | `old_crm_data.reviewer[].nofpaperspubint` | Papers published internationally | Author credibility |

---

## `authorData` (Mautic) — Key Fields

| Field | Description | Used For |
|---|---|---|
| `lastName` | Full name (confusingly named) | Verification |
| `university` | University from Mautic | Affiliation data |
| `country` | Country from Mautic | Location data |
| `titleOfPaper` | Paper title #1 | **Critical for Tier 2 evidence.** Use to search OpenAlex for the paper and its pre-classified topics. |
| `titleOfPaper2` through `titleOfPaper5` | Additional paper titles | Same as above. Up to 5 paper titles possible. |
| `validationStatus` | `"Valid"`, `"Already Exists"`, etc. | Diagnostic |

---

## `article_details[]` — Key Fields (When Present)

**Only present for authors who have submitted papers to the journals.** Reviewer-only profiles will NOT have this field.

| Field | Description | Used For |
|---|---|---|
| `primary_author_name` | Author's name as submitted | Verification |
| `primary_author_email` | Author's email | Email collection |
| `author_university` | University (may contain HTML: `"<p>Jiangsu University</p>\n"`) | Affiliation data. **Clean HTML via LLM.** |
| `author_country` | Country | Location data |
| `article_title` | Paper title | **Tier 1 evidence.** Use for OpenAlex paper search. |
| `article_abstract` | Full abstract (may contain HTML) | **Tier 1 evidence.** Primary signal for research classification. **Clean HTML via LLM.** |
| `article_keywords` | Comma-separated author keywords | **Tier 1 evidence.** Direct research signals. |
| `classification` | **JSON string** — Stringified array of classification objects. Must `json.loads()` before parsing. Example: `"[{\"symbol\":\"arXiv\",\"code\":\"cs.CV\"}]"` | **Tier 1 evidence.** Classification hints for taxonomy boost. |
| `journal_title` | Journal where submitted | Context |
| `journal_abbreviation` | e.g., `"GJCST"` | Context |

---

## `emails[]` — Key Fields

| Field | Description | Used For |
|---|---|---|
| `mainEmail` | Email address of the author | Email collection |
| `from` | Source: `"GJ"` or `"LJP"` | Identifies which journal system |
| `text` | Plain text of the email thread | **Tier 3 evidence.** Author may state research interests in emails. |
| `textAsHtml` | HTML version of email | Not used (prefer `text`). |

---

## `connectionSuccess` and `error`

| Field | Description | Used For |
|---|---|---|
| `connectionSuccess` | Whether the Mautic/GJ API connection succeeded | If `false`, `authorData` may be missing. Handle gracefully. |
| `foundInTheDatabase` | Whether the author was found in Mautic | If `false`, `authorData` will have limited data. |
| `error` | Error object from failed API calls (e.g., 404 from GJ API) | Diagnostic only. Does not affect processing — the service should proceed with whatever data is available. |

---

## Field Mapping: Input → `author_details` Output

> **Important:** The `author_details` output is built from **external API data** (OpenAlex, ORCID, Semantic Scholar) — NOT by copying fields from the n8n input. n8n already has the input data; our service adds externally verified information that n8n cannot get on its own. The input payload is used **only for lookup keys** to find the author on external platforms.

### Input fields used as search keys

| Input Field | Used To |
|---|---|
| `basic_information.first_name` + `last_name` | Search OpenAlex/ORCID/Semantic Scholar by author name |
| `basic_information.email` | Disambiguate author matches |
| `academic_ids.orcid` | Direct ORCID lookup (if already known) |
| `affiliations_work_experience[].institution` | Disambiguate OpenAlex author matches by institution |
| `authorData.university` / `old_crm_data.university` | Fallback institution for author search |

### Output fields and their external sources

| Output (`author_details`) | Primary Source | Secondary Source | Notes |
|---|---|---|---|
| `user_id` | Input `user_id` | — | **Only direct copy.** WordPress CRM ID, not available externally. |
| `basic_information.*` | **OpenAlex** display_name / **ORCID** given-name, family-name | — | Built from external; `user_id` is the only CRM-sourced field |
| `biography` | **ORCID** biography | — | Only if available in ORCID record |
| `academic_ids.orcid` | **OpenAlex** orcid field | **ORCID** (if looked up directly) | Discovered or confirmed externally |
| `academic_ids.research_gate_url` | — | — | Rarely available externally; return `null` |
| `academic_ids.google_scholar_url` | — | — | Rarely available externally; return `null` |
| `social_links.*` | — | — | Not available from academic APIs; return `null` |
| `affiliations` | **ORCID** employments | **OpenAlex** affiliations | Merged from both; includes `institution_open_alex_id`, `location`, `is_current` |
| `education` | **ORCID** educations | — | Institution, degree, department, dates |
| `research_areas` | **Generated** from classification results | — | Comma-separated string of selected keywords/specializations |
| `patents` | **ORCID** works (type=patent) | — | Only if available in ORCID |
| `editor_roles` | — | — | Not available from academic APIs; return `[]` |
| `memberships` | **ORCID** memberships/services | — | Only if available in ORCID |
| `advisors` | — | — | Not available from academic APIs; return `[]` |
| `advisees` | — | — | Not available from academic APIs; return `[]` |
| `grants_awards` | **ORCID** fundings | — | Only if available in ORCID |

### What happens when external APIs return nothing?

If the author is not found on any external platform, `author_details` should contain only `user_id` and empty/null fields for everything else. Do not fall back to copying input data — n8n already has it.

