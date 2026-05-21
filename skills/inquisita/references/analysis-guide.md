# Analysis Job Guide

## Table of Contents
- How Analysis Jobs Work
- Writing Good Prompts
- Output Schema Patterns
- Intelligence Levels
- Chunk-Level vs Summary-Only
- Polling and Error Handling
- Scoping Jobs with SQL
- Prompt Templates

## How Analysis Jobs Work

An analysis job sends each document (or chunk) to an LLM with your prompt and output schema. The LLM returns structured JSON for each item. Jobs run asynchronously — submit with `inquisita_analyze`, then poll `inquisita_get_analysis_job` every 10-15 seconds until done.

The analysis LLM is a smaller, faster model. It sees ONLY the document content and your prompt. It has zero context about the matter, the parties, the legal theories, or anything else. Every piece of context the LLM needs must be in your prompt.

## Writing Good Prompts

### Include Full Context

The analysis LLM knows nothing about your matter. Include:
- Party names and roles
- Case type / subject matter
- What you're looking for and why
- Specific definitions if terms are ambiguous

**Bad:**
```
Is this document relevant?
```

**Good:**
```
In the construction defect case Weston Properties LLC v. Summit Ridge Builders Inc.,
Weston (property owner) alleges Summit Ridge (general contractor) performed defective
work on a commercial building at 123 Main St, Lakewood, CO. Key claims include
foundation cracking, waterproofing failure, and HVAC defects.

Classify this document's relevance to the construction defect claims. Consider whether
it discusses the property, the construction work, inspections, complaints, repairs,
or communications between the parties about these issues.
```

### Be Specific About Outputs

Constrain the LLM to specific values rather than free text when possible.

**Bad:**
```
What type of document is this?
```

**Good:**
```
Classify this document into exactly one category:
- contract: Agreements, amendments, change orders between parties
- correspondence: Emails, letters, memos between parties
- inspection: Inspection reports, engineering assessments, test results
- financial: Invoices, payment records, lien documents
- pleading: Court filings, motions, orders
- other: None of the above
```

### Use Examples for Complex Extraction

When extracting structured data, show the LLM what you want:

```
Extract all dated events from this document. For each event, provide:
- date: The date in YYYY-MM-DD format (use best judgment if only month/year given)
- description: One sentence describing what happened
- party: Which party is primarily involved (use exact names)

Example output for a construction context:
- date: "2024-03-15", description: "Summit Ridge submitted Change Order #3 for additional foundation work", party: "Summit Ridge Builders"
- date: "2024-03-22", description: "Weston Properties rejected Change Order #3 citing scope creep", party: "Weston Properties"
```

## Output Schema Patterns

The `output_schema` field defines the structure of each result. Three formats, mixable:

### Flat Fields
```json
{
  "vendor": "string",
  "amount": "number",
  "is_relevant": "boolean"
}
```

### Enum Fields (Structurally Enforced)
```json
{
  "relevance": ["high", "medium", "low", "none"],
  "document_type": ["contract", "correspondence", "inspection", "financial", "other"]
}
```

### Nested Arrays (for Multi-Value Extraction)
```json
{
  "events": {
    "type": "array",
    "items": {
      "type": "object",
      "properties": {
        "date": {"type": "string"},
        "description": {"type": "string"},
        "party": {"type": "string"}
      }
    }
  }
}
```

### Array Shorthand
```json
{
  "keywords": "string[]",
  "amounts": "number[]"
}
```

### Vector Similarity (Special Case)
For `vector_similarity` method, the output_schema is ignored. Results always contain:
- `similarity_score`: float (0-1)
- `is_match`: boolean (based on threshold)

Query with: `results->>'similarity_score'` and `(results->>'is_match')::boolean`

## Intelligence Levels

Set via `config.intelligence`:

| Level | Use For | Speed | Cost |
|-------|---------|-------|------|
| `low` (default) | Simple classification, yes/no, category assignment | Fast | Cheap |
| `medium` | Nuanced analysis, multi-factor assessment | Moderate | Moderate |
| `high` | Complex legal reasoning, synthesis across sections | Slow | Expensive |

Default to `low` for most classification tasks. Use `medium` when the prompt requires judgment or weighing multiple factors. Reserve `high` for genuinely complex analytical tasks.

## Chunk-Level vs Summary-Only

Two modes, controlled by the optional `config.summary_only` boolean. **Default is chunk-level** — the right choice for almost every legal workflow.

### Chunk-level (default — omit `summary_only`, or set `false`)

- One LLM call per chunk reading the chunk's full native content (page/segment/clip)
- Required for finding passages, page-level Q&A, citations, event extraction, or any task whose answer might live in a specific section
- One result row per `(source_file_id, chunk_index)` pair
- Source SQL must return `source_file_id` AND `chunk_index`:
  ```sql
  SELECT source_file_id, chunk_index FROM chunks WHERE media_type = 'document'
  ```
- Query results via `analysis_results_chunk` (includes `text_content`, `chunk_index`, `file_name`)
- Enriches chunk-level collections naturally; results aggregated into `chunk_enrichments`

### Summary-only (`config.summary_only: true` — explicit opt-in)

- One LLM call per file reading ONLY the document's `overall_summary`
- Much faster and cheaper, but BLIND to anything the summary omits — do **not** use for finding passages, citations, or detail-dependent questions
- Appropriate for: bulk classification, document-typing, screening questions answerable from a short summary
- Source SQL returns `source_file_id` only:
  ```sql
  SELECT source_file_id FROM documents WHERE file_name ILIKE '%contract%'
  ```
- Query results via `analysis_results_doc` (includes `file_name`, `media_type`, `overall_summary`)
- Enriches both grain types; on chunk-level collections, the file's summary-only result propagates to every chunk row via `document_enrichments`

### How this interacts with collections

| Job mode | Can enrich document-level collection? | Can enrich chunk-level collection? |
|----------|---------------------------------------|------------------------------------|
| Chunk-level | ❌ (would fan-out) | ✅ joins on `(source_file_id, chunk_index)` |
| Summary-only | ✅ joins on `source_file_id` | ✅ file's value applies to every chunk |

If you're enriching a document-level collection, you must use `summary_only: true`.

## Polling and Error Handling

After calling `inquisita_analyze`, poll `inquisita_get_analysis_job` every 10-15 seconds.

**Status values:**
- `pending` / `running`: Keep polling
- `complete`: All documents analyzed successfully
- `complete_with_errors`: Some documents failed. Check `progress.failed` count. Failed documents have no result row.
- `failed`: Entire job failed. Check error message.

For `complete_with_errors`: compare `progress.total` vs actual result count. If failures persist across retries, flag it — don't silently proceed with partial data.

## Scoping Jobs with SQL

The `sql` parameter selects which documents to analyze. Same syntax as the query tool.

```sql
-- All documents in the matter
SELECT source_file_id FROM documents

-- Only PDFs
SELECT source_file_id FROM documents WHERE file_type = 'pdf'

-- Specific category
SELECT source_file_id FROM documents WHERE category = 'correspondence'

-- Top semantic matches
SELECT source_file_id FROM documents
ORDER BY embedding::halfvec(3072) <=> :query_vector::halfvec(3072)
LIMIT 20

-- Exclude already-analyzed documents
SELECT source_file_id FROM documents
WHERE source_file_id NOT IN (
  SELECT source_file_id FROM analysis_results_doc WHERE job_name = 'my_job'
)
```

When using `:query_vector`, also provide the `semantic_query` parameter to `inquisita_analyze`.

## Prompt Templates

### Document Classification
```
Classify this document into one of these categories: [list categories].
If the document doesn't clearly fit any category, use "other".
Provide a one-sentence reasoning for your classification.
```
Schema: `{"category": ["cat1", "cat2", "other"], "reasoning": "string"}`

### Relevance Scoring
```
In [case name], [brief case description]. Rate this document's relevance to [specific claim/topic].
Consider: [list of relevance factors].
```
Schema: `{"relevance": ["high", "medium", "low", "none"], "reasoning": "string"}`

### Event/Date Extraction
```
Extract all dated events from this document. Include the date (YYYY-MM-DD format),
a one-sentence description, and which party or entity is primarily involved.
[Include party names and roles for context.]
```
Schema: `{"events": {"type": "array", "items": {"type": "object", "properties": {"date": {"type": "string"}, "description": {"type": "string"}, "party": {"type": "string"}}}}}`
Mode: chunk-level (default — needed to extract events from each page)

### Privilege Review
```
Determine if this document is potentially privileged. A document is privileged if it is:
- A communication between [client] and their attorney for the purpose of legal advice
- Attorney work product prepared in anticipation of litigation
- Contains mental impressions, conclusions, or legal theories of [attorney/firm name]
Consider: author, recipients, subject matter, and whether the communication was
made in confidence for legal advice purposes.
```
Schema: `{"privileged": "boolean", "privilege_type": ["attorney_client", "work_product", "none"], "reasoning": "string"}`
Intelligence: `medium`