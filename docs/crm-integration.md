# CRM Integration Requirements: Reviewer System

This document outlines the Application Programming Interfaces (APIs) and data fields the AI Reviewer Recommendation System requires from the core CRM (WordPress/ResearchNodes) development team.

Because the AI system is stateless regarding user availability, the n8n orchestration layer must bridge the gap between the AI's semantic recommendations and the CRM's operational reality.

---

## 1. Initial Backfill Data Export

To initialize the Qdrant vector database, the AI system requires a one-time export of all active reviewers.

**Format**: JSON or CSV
**Required Fields**:
- `user_id` (Integer, Primary Key)
- `display_name` (String)
- `openalex_id` (String, if known)
- `institution_ids` (Array of Strings, OpenAlex format, if known)
- `declared_keywords` (Array of Strings)
- `phd_stream` (String)

---

## 2. The Availability API

After the AI generates a shortlist of top reviewers, the n8n workflow must filter out reviewers who are currently unavailable. The CRM team must expose a REST endpoint to query real-time availability.

**Endpoint Suggestion**: `GET /api/reviewers/availability`

**Query Parameters**:
- `user_ids`: Comma-separated list of reviewer IDs (e.g., `?user_ids=36234,38336,40001`)

**Response Schema**:
```json
{
  "reviewers": [
    {
      "user_id": 36234,
      "active_review_count": 2,
      "max_papers_limit": 3,
      "operational_status": "available", // Enum: available, on_leave, overloaded, declined_recently
      "last_review_date": "2026-05-15",
      "last_invited_date": "2026-06-01"
    },
    {
      "user_id": 38336,
      "active_review_count": 3,
      "max_papers_limit": 3,
      "operational_status": "overloaded"
    }
  ]
}
```

**n8n Workflow Logic**:
The n8n workflow will parse this response and drop any reviewer where:
1. `operational_status != "available"`
2. `active_review_count >= max_papers_limit`

---

## 3. Manual CoI Exclusion Data (Optional but Recommended)

The AI system automatically derives Conflict of Interest (CoI) from the OpenAlex co-authorship graph (last 5 years) and institutional affiliations. However, academic databases cannot capture personal conflicts, contractual exclusions, or editorial board overlaps.

If the CRM tracks this data, it should be provided either during the backfill or dynamically when a manuscript is submitted.

**Relevant Fields to track/expose**:
- `manual_coi_list`: Array of user IDs this reviewer cannot review.
- `editorial_board_journal_ids`: Array of journal IDs where this reviewer serves on the board.
