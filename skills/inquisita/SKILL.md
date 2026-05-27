---
name: inquisita
description: >
  Core skill for using Inquisita, a document intelligence platform. Use this skill
  whenever the user mentions "Inquisita", wants to create a matter, upload documents,
  search or query documents, run analysis jobs, create document collections, or work
  with any Inquisita MCP tools. Also trigger when the user asks to organize documents
  into a knowledge base, run document analysis or classification, build document
  collections, or enrich collections with structured data — even if they don't
  mention Inquisita by name. Trigger for any document management task that goes
  beyond simple file reading.
version: 0.1.0
---

# Inquisita

Inquisita is a document intelligence platform and system of record. Documents go in, structured knowledge comes out. It organizes documents into **matters**, processes them into searchable chunks with embeddings, and supports LLM analysis jobs that produce structured results you can query with SQL.

Everything you create in Inquisita — matters, collections, analysis results, enrichments — persists as shared organizational data. Other users, attorneys, paralegals, and their agents can access and build on the same work. When you create a well-organized matter with enriched collections, you're not just completing a task — you're building a durable knowledge asset for the firm. Think of collections especially as the firm's curated, structured understanding of a matter: a "Privileged Documents" collection or a "Key Evidence for Breach Claim" collection enriched with relevance scores is something the whole team can query and rely on going forward.

## Mental Model

```
Matter (container)
  ├── Categories (organizational buckets for uploads)
  ├── Documents (uploaded files, auto-processed into chunks + embeddings)
  ├── Analysis Jobs (async LLM or vector tasks that produce structured results)
  └── Collections (curated document sets, optionally enriched with analysis results)
```

Everything lives inside a matter. A matter has categories (like folders), documents belong to one category each, and collections are dynamic groupings built from queries.

## Core Workflow Patterns

### Pattern 1: Matter Setup and Document Upload

```
inquisita_create_matter → inquisita_get_upload_link → [upload via curl] → inquisita_discover (verify)
```

1. **Create the matter** with meaningful categories. Categories are mandatory single-select buckets, not tags. Define them upfront — every uploaded document must go into one.

   Example legal categories: `pleadings`, `correspondence`, `contracts`, `discovery`, `work_product`.
   Example business categories: `financials`, `contracts`, `communications`, `reports`.

2. **Get an upload link.** This returns a `session_token` (valid 5 minutes) and an `api_url`. Upload each file via curl or via a script.
   ```bash
   curl -H "Authorization: Bearer <session_token>" \
     -F "file=@document.pdf" \
     "<api_url>/api/v1/documents/matters/<matter_id>/upload?category=<category_key>"
   ```

  If your working environment doesn't support shell/python access, you can return the link with the user and ask them to use it to upload.

3. **Wait for processing.** After upload, Inquisita auto-extracts text, chunks the document, and generates embeddings. This takes 30–120 seconds per document. Poll with `inquisita_discover` to watch document counts increase, or query `processing_status` in the documents view.

4. **Archive File Extraction** Archive files like zip, mbox, pst, etc are unpacked in Inquisita and expanded into their child items, each processed separately.

### Pattern 2: Querying Documents

```
inquisita_discover → inquisita_query
```

Always call `inquisita_discover` first to see what views, columns, categories, and metadata are available.

**Critical: never ask the user to write SQL.** When a user asks a natural-language question about their documents ("who attended the September 21, 2021 Lookout Ridge meeting?", "what are the liability caps in our vendor contracts?"), it is your job to translate that into the appropriate `discover` + `query` calls. Do not respond by asking the user "what SQL query or WHERE clause would you like to use?" — the user is not expected to know SQL or the schema. Call `discover` to see the available views and columns, then write the SQL yourself, combining it with semantic and keyword search as needed. If you genuinely cannot answer after a reasonable search (data not ingested, ambiguous question), explain what you tried in plain language — do not punt by asking the user for a query.

**Three search modes, combinable in one query:**

- **SQL filtering:** Direct WHERE clauses on metadata, file type, category, tags
- **Keyword search:** Full-text via `search_text @@ websearch_to_tsquery('term1 OR "exact phrase"')`
- **Semantic search:** Provide `semantic_query` text, reference `:query_vector` in SQL for vector distance ordering

Semantic search is usually the best starting point. Keyword search is better for exact names, dates, or specific phrases.

Queries can also be run with the presigned_url=true field to get presigned urls with every result. While docs can also be seen in the UI, this can be useful to provide your user with direct doc viewing links, or to build your own generative UI (In general generative UI takes significant tokens and time and should be avoided unless user requests. If using presigned urls in generative UIs or scripts, its best to save the output json and import it in your scripts/UI so you don't need to hard code them into your UI.

**Reading whole documents is allowed — and often the right move.**

For most legal tasks, reading the relevant documents end-to-end is what produces accurate output. Don't try to derive answers from summaries or clever SQL aggregations alone. To pull a doc's full content:

```sql
-- Whole document, in chunk order. If large, the host will spill it to disk —
-- read the spill and proceed.
SELECT chunk_index, text_content FROM chunks
WHERE source_file_id = 'doc_X' ORDER BY chunk_index;

-- A section of one document
SELECT chunk_index, text_content FROM chunks
WHERE source_file_id = 'doc_X' AND chunk_index BETWEEN 5 AND 15;
```

After you've identified the relevant docs for your task, just read them. The database is not "expensive grep" — it's a workspace. Reading a contract end-to-end is exactly what an attorney would do.

**Triage helpers — use when picking which docs to read.**

For a matter with many docs of different shapes, you'll need a triage step before reading every doc. Use `overall_summary` and a few targeted queries to narrow down:

```sql
-- Which docs mention specific terms (proper nouns, decision numbers, dates) from the task prompt
SELECT documents.file_name,
       COUNT(*) FILTER (WHERE text_content ILIKE '%TERM_1%') AS hits_1,
       COUNT(*) FILTER (WHERE text_content ILIKE '%TERM_2%') AS hits_2
FROM chunks JOIN documents USING (source_file_id)
GROUP BY documents.file_name;

-- Or one full-text-search sweep across the matter
SELECT DISTINCT source_file_id, file_name FROM chunks
JOIN documents USING (source_file_id)
WHERE search_text @@ websearch_to_tsquery('"TERM_1" OR "TERM_2"');

-- Snippet extraction when you only want a quote, not the whole doc
SELECT source_file_id, chunk_index,
       substring(text_content, GREATEST(1, position('term' in lower(text_content)) - 200), 600) AS snippet
FROM chunks WHERE text_content ILIKE '%term%' LIMIT 50;
```

**Caveat on `overall_summary`:** the summary captures what a doc is "about" at a high level. It usually doesn't surface specific named witnesses, dollar amounts, or quoted phrases buried in the body. When the rubric expects named-entity findings (a particular witness's testimony, a specific dollar figure, an exact quote), you'll need to look at the chunk text — either by reading the doc or by ILIKE-sweeping for those specific terms.

**Before declaring done — VERIFY and REVISE (not theatre).**

Multiple studies show that a real *check → revise* loop materially improves deliverable accuracy on complex tasks. A superficial "I reviewed it and it looks good" does nothing; an actual revise-after-check loop catches the omissions that summary-triage and topical filtering miss. Before treating any deliverable as final:

1. **Build an explicit checklist from the task prompt and matter description.** List every:
   - Proper noun the prompt cites (party names, jurisdictions, named witnesses, named entities, named facilities or sites)
   - Unique identifier (decision/regulation numbers, case numbers, exhibit IDs, statute references, specific contract numbers)
   - Specific number or date the prompt anchors to (dollar amounts, dates of relevant events, document version numbers)
   - Named deliverable section the prompt explicitly asks for

2. **Audit your draft against the checklist.** For each item:
   - Does the deliverable mention this specific entity / identifier?
   - When the prompt cites a specific identifier (e.g., "Decision N", "Case No. X"), is your deliverable anchored to THAT identifier — not a similar-but-different one in the matter? Multi-entity matters often contain distractor identifiers; if your draft cites a different one than the prompt, that's a defect, not a stylistic choice.
   - For named-entity facts: did you actually QUERY the chunks for THIS entity, or did you let your queries find it incidentally through topical filters? If you didn't explicitly query, you probably didn't find it.

3. **REVISE — actually fix what the audit flagged.** This step has to produce concrete `inquisita_query` calls that surface the missed entity, then integrate the findings into the deliverable. The output of a real revise step is a longer, more specific deliverable — not the same draft with a "I have reviewed and confirm completeness" stamp.

4. **Audit your audit.** If your verification step did not produce at least one new query and one concrete revision, your verification was theatre. Run the loop again with sharper attention to identifiers.

The failure mode this exists to prevent: agent produces a topically-correct, well-structured deliverable that references the wrong specific identifier (because a similar distractor was more frequent in the chunks) or omits a specific named entity (because the entity name lives in a document the agent never queried). Both look fine on a skim. Both fail the rubric. Verify-and-revise catches both.

Read `references/sql-patterns.md` for common query patterns and syntax.

### Pattern 3: Analysis Jobs (Async — Must Poll)

```
inquisita_analyze → [poll inquisita_get_analysis_job every 10-15s] → inquisita_query on results
```

This is the most powerful pattern and the one most likely to trip you up. Analysis jobs are **asynchronous**. After submitting, you must poll until status is `complete`, `complete_with_errors`, or `failed`.

**Two methods:**
- `llm`: Send document content to an LLM with a prompt and output schema. Returns structured JSON per document.
- `vector_similarity`: Compute cosine similarity against a reference query. Returns `similarity_score` and `is_match`.

**Two modes:**

- **Chunk-level (default — what you almost always want):** one LLM call per chunk reading the chunk's full native content. Required for finding passages, page-level Q&A, citations, or any question whose answer might live in a specific section. Source SQL must return `source_file_id, chunk_index` (typically `SELECT source_file_id, chunk_index FROM chunks WHERE ...`). Results live in `analysis_results_chunk`.
- **Summary-only (`config.summary_only: true` — explicit opt-in):** one LLM call per file reading ONLY the document's overall summary. Much faster and cheaper, but BLIND to anything not surfaced in the summary. Use only for bulk classification, document-level labels, or screening questions answerable from a short summary — never for finding passages or evidence. Source SQL returns `source_file_id` only (`SELECT source_file_id FROM documents WHERE ...`). Results live in `analysis_results_doc`.

When in doubt, omit `summary_only` — chunk-level is the safer default for legal workflows.

**Analysis Can Be Pricey:**
- Running analysis jobs can be pricey, especially as intelligence goes up. If you're not sure whether your prompt or intelligence level is sufficient, test against a smaller subset of docs and analyze results before processing the whole set.

**The analysis LLM is a lower intelligence model.** It has NO context about the matter, parties, or case. It's ability to reason or make deductions from multi-step deductions is limits. Write explicit prompts that include all necessary context. Be specific about output constraints. Perhaps provide examples.

Good prompt:
```
In the construction defect case Weston Properties LLC v. Summit Ridge Builders Inc.,
Weston (property owner/plaintiff) alleges Summit Ridge (general contractor/defendant)
performed defective construction work on a commercial building at 123 Main St,
Lakewood, CO. Key claims include foundation cracking, waterproofing failure, and
HVAC system defects. Other involved parties include Pinnacle Engineering (Weston's
engineering consultant), the City of Lakewood (issued permits and inspections), and
various subcontractors.

Extract every dated event mentioned in this document. For each event, provide:
- date: in YYYY-MM-DD format. If only a month and year are given, use the 1st of
  the month. If only a year is given, use January 1st.
- description: a one-sentence description of what happened, with enough context to
  understand the event without reading the source document.
- party: the party primarily involved. Use one of: "Weston Properties", "Summit Ridge
  Builders", "Pinnacle Engineering", "City of Lakewood", "Both", or the name of
  the specific entity if it's someone else.
- source_type: the type of evidence this date comes from. Use one of: "contract",
  "email", "inspection_report", "invoice", "letter", "court_filing", "other".

If no dated events are found, return an empty array. Do not infer or fabricate dates.
Only extract dates that are explicitly stated or clearly implied in the text.
```

Bad prompt: "What dates are mentioned?"

**Output schemas** define the structure of what the analysis LLM returns. The schema goes in `config.output_schema` and supports several formats that can be mixed:

- **Flat fields:** `{"vendor": "string", "amount": "number", "is_relevant": "boolean"}`
- **Enums (structurally enforced):** `{"relevance": ["high", "medium", "low", "none"]}` — the LLM can only return one of these values.
- **Arrays:** `{"keywords": "string[]", "amounts": "number[]"}`
- **Nested object arrays** (for multi-value extraction like events, line items, parties):
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

Match the schema to your prompt. For classification tasks, use enums to constrain outputs. For extraction tasks that pull multiple items from a document (dates, parties, amounts), use nested object arrays. For simple yes/no questions, use booleans.

The good prompt example above would pair with this schema:
```json
{
  "events": {
    "type": "array",
    "items": {
      "type": "object",
      "properties": {
        "date": {"type": "string"},
        "description": {"type": "string"},
        "party": {"type": "string"},
        "source_type": {"type": "string"}
      }
    }
  }
}
```

Read `references/analysis-guide.md` for more prompt templates, output schema patterns, and intelligence level guidance.

**Querying results:** After completion, query `analysis_results_view` (both levels), `analysis_results_chunk` (chunk-level only), or `analysis_results_doc` (document-level only). Results are in a `results` JSONB column — access fields with `results->>'field_name'`.

### Pattern 4: Collections

```
inquisita_create_collection → [optionally] inquisita_enrich_collection → inquisita_query on collection_members
```

Collections are named, curated document sets built from SQL queries, optionally with a `semantic_query` for vector filtering. They serve as the organizational layer between raw documents and structured work product — think of them as saved searches that you can then layer analysis results onto.

Importantly, collections are **persistent and shared**. When you create a "Key Evidence — Breach of Contract" collection enriched with relevance scores, that collection is available to every user and agent with access to the matter. Build collections with descriptive names and rich enrichments — you're creating durable reference points that the team will use long after this conversation ends.

**Creating a collection:**

There are two ways to populate a collection:

- **SQL-only:** `sql="category = 'correspondence'"` grabs all documents matching the filter.
- **Semantic + SQL:** Add a `semantic_query` for conceptual matching. This is usually better for topic-based collections (e.g., "documents related to change orders") because it catches documents that discuss the concept without using the exact keywords.

**Enriching collections with analysis results:**

This is where collections become powerful. After running an analysis job, you can merge its results onto collection items via `inquisita_enrich_collection`. Each item gains structured data under a `job_name` namespace — and these become **queryable fields** on `collection_members`. The enrichment fields from each job are discoverable: call `inquisita_discover` with `collection_id` to see all enrichment sources, their job names, field schemas, and whether they're document-level or chunk-level. Then use those fields in SQL queries against `collection_members` to filter, sort, and build views.

Two paths to get there:
- Create the collection first, then enrich it: `inquisita_create_collection` → `inquisita_enrich_collection`
- Or do it in one step: pass `enrichment_job_id` when creating the collection

You can stack multiple enrichments on the same collection — run a relevance job and a privilege job, enrich with both, then query for documents that are high-relevance AND non-privileged. Each job's results are namespaced independently, so enrichments never collide.

**How enrichments surface on `collection_members`:**

Each row of `collection_members` exposes two enrichment columns, both materialized at read time by joining `analysis_results` through the collection's `linked_job_ids`:

- `chunk_enrichments` (JSONB) — populated from chunk-level analysis jobs, keyed on `(source_file_id, chunk_index)`. One value per chunk row in chunk-level collections.
- `document_enrichments` (JSONB) — populated from summary-only analysis jobs, keyed on `source_file_id`. The file's value applies to every row of that file (so on a chunk-level collection, all chunks of the same file share the same `document_enrichments`).

Both columns are flat JSONB maps; access fields with `chunk_enrichments->>'field_name'` or `document_enrichments->>'field_name'`. Multiple linked jobs are merged into the same map (field-name collisions are rejected at link time, so each field has exactly one source job).

**Grain compatibility rule:** chunk-level analysis jobs can be linked only to chunk-level collections (linking to a document-level collection would fan out one row per chunk — the API rejects it). Summary-only jobs can be linked to either grain.

**Practical guidance:**
- For passage-level filtering inside a chunk-level collection, query `chunk_enrichments`.
- For document-level filtering or sorting on either grain, query `document_enrichments`.
- For per-document aggregates of a chunk-level collection (one row per file with chunk count), use the `collection_documents` view.

**Collection types:** `

also have a type specified on creation, freetext

### Pattern 5: Combined Workflow (Analysis → Collection)

For the most powerful results, chain analysis into collections:

1. Run an analysis job with a descriptive `job_name` (e.g., `relevance_scoring`, `privilege_review`)
2. Create a collection with `enrichment_job_id` pointing at the completed job — or create the collection first and call `inquisita_enrich_collection` afterward
3. Query enriched collection members with SQL to filter by analysis results
4. Optionally run a second analysis job on the collection's documents and enrich again to layer additional structured data

This is how you build things like "all documents relevant to the breach of contract claim, scored by relevance, with privileged documents flagged" — run a relevance job and a privilege job, create a collection, enrich with both, then query:

```sql
SELECT source_file_id, file_name,
  enrichments->'relevance'->>'level' as relevance,
  (enrichments->'privilege'->>'privileged')::boolean as privileged
FROM collection_members
WHERE collection_name = 'Breach of Contract Discovery'
  AND enrichments->'relevance'->>'level' IN ('high', 'medium')
  AND (enrichments->'privilege'->>'privileged')::boolean = false
ORDER BY enrichments->'relevance'->>'level'
```

## Tool Quick Reference

| Tool | Purpose | Key params |
|------|---------|------------|
| `create_matter` | Create a new matter with categories | `name`, `categories[]` |
| `get_upload_link` | Get session token for uploading files | `matter_id` |
| `discover` | See what's in a matter or collection | `matter_id`, optional `collection_id` |
| `query` | SQL search across documents, chunks, results | `sql`, `matter_id`, optional `semantic_query` |
| `analyze` | Run async LLM or vector analysis job | `matter_id`, `method`, `config`, `sql` |
| `get_analysis_job` | Poll analysis job status | `job_id` |
| `create_collection` | Build a named document set from a query | `name`, `sql`, `matter_id` |
| `enrich_collection` | Merge analysis results into collection items | `collection_id`, `job_ids[]` |
| `modify_collection` | Add/remove documents from a collection | `collection_id`, `action`, `source_file_ids[]` |
| `update_matter` | Change matter name, description, categories | `matter_id` |
| `get_document` | Get document details + download URL | `source_file_id` |
| `delete_document` | Remove a document and all its data | `source_file_id` |

## Sharing Links with Users

Inquisita has a web interface where users can browse their matters, documents, and collections visually. When you create or modify resources, share the relevant link so the user can see their data:

- **Matter:** `https://inquisita.ai/app/matters/{matter_id}`
- **Collections:** `https://inquisita.ai/app/matters/{matter_id}/collections?collection={collection_id}`

After creating a matter, uploading documents, or building collections, include the appropriate link in your response. The web interface lets users browse documents, view collection contents, and explore analysis results without needing to write SQL — it's the human-friendly counterpart to the API work you're doing as an agent.

## Common Mistakes

1. **Not polling analysis jobs.** They're async. You must poll `get_analysis_job` until complete.
2. **Writing vague analysis prompts.** The analysis LLM has no case context. Include parties, claims, and specific instructions.
3. **Forgetting to discover first.** Always call `discover` before querying to see available views, columns, and enrichment fields.
4. **Not waiting for document processing.** Freshly uploaded documents need 30-120 seconds before they're fully indexed and searchable.
5. **Asking the user to write SQL.** SQL is your job. When a user asks a natural-language question, call `discover` and translate it into a query yourself. Never respond with "what WHERE clause would you like to use?" — the user does not know the schema, and from their perspective the agent is broken when you ask.

## Examples

Inquisita's tools provide powerful abstractions that let agents flexibly analyze documents at scale. The workflow patterns above can be combined to support a wide range of tasks. Here are some examples:

**Contract risk review.** Upload a set of vendor contracts to a matter. Run a chunk-level analysis job prompted to identify clauses related to liability caps, indemnification, termination rights, and auto-renewal. Create a collection of all contracts, enrich with the results, then query to surface contracts with unfavorable terms — like uncapped liability or auto-renewal without notice periods. Produce a summary report ranked by risk.

**Due diligence document room.** Create a matter for an acquisition target with categories for financials, IP, employment, and regulatory. Upload thousands of documents from a data room export. Run multiple analysis jobs — one for relevance classification, one for red-flag detection (litigation mentions, regulatory violations, undisclosed liabilities). Build collections per diligence workstream, enriched with both jobs, so each reviewer gets a filtered, scored set of documents relevant to their area.

**Insurance claims processing.** Upload a batch of claim files (police reports, medical records, photos, adjuster notes). Run analysis jobs to extract key fields: date of loss, claimed amount, injury type, and whether the claimant's account is consistent across documents. Create collections grouped by claim status (new, under review, flagged for investigation) and enrich with the extracted fields so adjusters can sort and prioritize by severity or inconsistency.

**Compliance audit.** Upload a company's policy documents alongside employee communications (emails, Slack exports). Run a chunk-level analysis job that checks whether specific compliance requirements (data handling, access controls, reporting obligations) are referenced or violated in the communications. Build a collection of flagged communications enriched with which policy they potentially violate, and produce an audit report.

**Research literature review.** Upload a corpus of academic papers or technical reports. Run an analysis job to extract key findings, methodology type, sample size, and whether the conclusions support or contradict a specific hypothesis. Create a collection enriched with these fields, then query to build a structured literature table — sorted by methodology, filtered to papers above a sample size threshold, grouped by whether they support or challenge the hypothesis.

**Real estate portfolio analysis.** Upload leases, inspection reports, and appraisals across a portfolio of properties. Run analysis jobs to extract lease terms (rent, expiration, renewal options), inspection findings (structural issues, code violations), and appraised values. Build collections per property, enriched with all three jobs, then query across the portfolio to identify leases expiring within 12 months, properties with unresolved code violations, or assets where appraised value has declined.
