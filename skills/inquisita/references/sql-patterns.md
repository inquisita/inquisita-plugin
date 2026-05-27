# SQL Patterns for Inquisita Queries

## Table of Contents
- Available Views (via discover)
- Query Efficiency (the cap and the patterns that beat it)
- Basic Document Queries
- Semantic Search
- Keyword Search
- Analysis Results Queries
- Collection Queries
- Enrichment Queries
- Combined Patterns

## Available Views

Before writing SQL, call `inquisita_discover` with the `matter_id` to get the current list of available views and their columns. The schema may evolve as Inquisita adds features — `discover` always returns the live schema.

For collections, call `inquisita_discover` with `collection_id` to see enrichment sources, their field names and types, and any highlights.

## Query Efficiency

Inquisita refuses query results larger than ~50,000 characters (~12K tokens). The cap is a forcing function: compute answers in the database, don't dump raw text back to your context. The pattern below is the one to avoid:

```sql
-- ❌ Dumps the whole document, forces you to read/grep it yourself
SELECT text_content FROM chunks WHERE source_file_id = 'X'
```

When the cap fires you get a structured response with these four patterns. Reach for them *before* hitting the cap.

### Snippet extraction
Matched paragraph (±200 chars context), not the whole chunk. Use when you want to see the operative passage for a clause/term.

```sql
SELECT source_file_id, chunk_index,
       substring(text_content,
                 GREATEST(1, position('term' in lower(text_content)) - 200),
                 600) AS snippet
FROM chunks
WHERE text_content ILIKE '%term%'
LIMIT 50
```

### Multi-pattern presence matrix
One query in place of N greps. Counts clause hits per document. Use when triaging which documents contain which clauses.

```sql
SELECT documents.file_name,
       COUNT(*) FILTER (WHERE text_content ILIKE '%assign%') AS n_assign,
       COUNT(*) FILTER (WHERE text_content ILIKE '%change of control%') AS n_coc,
       COUNT(*) FILTER (WHERE text_content ILIKE '%governing law%') AS n_gov
FROM chunks
JOIN documents USING (source_file_id)
GROUP BY documents.file_name
```

### Files-with-matches
Drop the text entirely; return only WHICH docs contain a term. Use when scoping which docs are worth opening.

```sql
SELECT DISTINCT source_file_id, file_name
FROM chunks
WHERE text_content ILIKE '%term%'
```

### Range slice
Read part of one doc by chunk_index range. Use when drilling into a known section of a known doc.

```sql
SELECT chunk_index, text_content FROM chunks
WHERE source_file_id = 'X' AND chunk_index BETWEEN 5 AND 15
```

### What grep can't do but SQL can
- Joins across documents and analysis results
- Aggregations (`COUNT`, `GROUP BY`, percentiles, statistical functions)
- Window functions (`RANK`, `LAG`, `LEAD` over `chunk_index`)
- JSONB metadata filtering (`metadata->>'field'`, `chunk_enrichments->>'field'`)
- Regex extraction (`regexp_matches`, `substring(text from pattern)`)
- Vector similarity ordering (no grep equivalent)

If a query would still return >50K chars after applying these, narrow the WHERE further, add `LIMIT`, or call `inquisita_analyze` for structured extraction across many docs at once.

## Basic Document Queries

```sql
-- List all documents
SELECT source_file_id, file_name, media_type, processing_status FROM documents

-- Filter by file type
SELECT source_file_id, file_name FROM documents WHERE file_type = 'pdf'

-- Filter by category
SELECT source_file_id, file_name FROM documents WHERE category = 'pleadings'

-- Filter by tag
SELECT source_file_id, file_name FROM documents WHERE 'privileged' = ANY(tags)

-- Check processing status
SELECT file_name, processing_status FROM documents WHERE processing_status != 'complete'

-- Simple WHERE shorthand (no SELECT needed)
-- The tool auto-wraps as SELECT * FROM documents WHERE ...
file_type = 'pdf'
```

## Semantic Search

Provide `semantic_query` parameter alongside the SQL. Reference `:query_vector` in the SQL.

```sql
-- Document-level semantic search (recommended starting point)
SELECT source_file_id, file_name,
  embedding::halfvec(3072) <=> :query_vector::halfvec(3072) AS distance
FROM documents
WHERE embedding IS NOT NULL
ORDER BY distance
LIMIT 20

-- Chunk-level semantic search (more granular, finds specific passages)
SELECT source_file_id, chunk_index, text_content,
  embedding::halfvec(3072) <=> :query_vector::halfvec(3072) AS distance
FROM chunks
ORDER BY distance
LIMIT 20
```

IMPORTANT: Always cast to `halfvec(3072)` for vector operations. The index requires this cast.

## Keyword Search

Uses PostgreSQL full-text search via `websearch_to_tsquery`.

```sql
-- Simple keyword search
SELECT source_file_id, file_name FROM documents
WHERE search_text @@ websearch_to_tsquery('change order')

-- OR search
WHERE search_text @@ websearch_to_tsquery('change order OR modification')

-- Exact phrase
WHERE search_text @@ websearch_to_tsquery('"change order number 1"')

-- Exclude terms
WHERE search_text @@ websearch_to_tsquery('contract -amendment')
```

NOTE: Keyword search is literal. If terms don't appear in the document text, you get 0 results. For conceptual/fuzzy matching, use semantic search instead.

## Analysis Results Queries

```sql
-- All results from a specific job (by job_name, no need for UUID)
SELECT source_file_id, file_name, results
FROM analysis_results_doc
WHERE job_name = 'relevance_scoring'

-- Filter by result values (JSONB access)
SELECT source_file_id, file_name, results->>'relevance' as relevance
FROM analysis_results_doc
WHERE job_name = 'relevance_scoring'
  AND results->>'relevance' = 'high'

-- Numeric result filtering
SELECT source_file_id, results->>'confidence' as confidence
FROM analysis_results_doc
WHERE job_name = 'scoring'
  AND (results->>'confidence')::float > 0.8

-- Boolean result filtering
SELECT source_file_id, file_name
FROM analysis_results_doc
WHERE job_name = 'privilege_review'
  AND (results->>'privileged')::boolean = true

-- Chunk-level results with text content
SELECT source_file_id, chunk_index, text_content, results
FROM analysis_results_chunk
WHERE job_name = 'event_extraction'
  AND results->>'has_date' = 'true'

-- Array field access in results (for nested extraction)
SELECT source_file_id, results->'events' as events
FROM analysis_results_chunk
WHERE job_name = 'event_extraction'

-- Vector similarity results (special fields)
SELECT source_file_id, file_name,
  results->>'similarity_score' as score,
  (results->>'is_match')::boolean as matched
FROM analysis_results_doc
WHERE job_name = 'similarity_check'
ORDER BY (results->>'similarity_score')::float DESC
```

## Collection Queries

```sql
-- List all collections
SELECT * FROM all_collections

-- Documents in a specific collection
SELECT source_file_id, file_name
FROM collection_members
WHERE collection_name = 'Change Orders'

-- Documents NOT in a collection (exclusion pattern)
SELECT source_file_id, file_name FROM documents
WHERE source_file_id NOT IN (
  SELECT source_file_id FROM collection_members
  WHERE collection_name = 'Privileged'
)
```

## Enrichment Queries

After enriching a collection with `inquisita_enrich_collection`, query enrichment fields via JSONB:

```sql
-- Access enrichment fields (text)
SELECT source_file_id, file_name,
  enrichments->'scoring'->>'relevance' as relevance
FROM collection_members
WHERE collection_name = 'Discovery Set'

-- Filter by enrichment boolean
SELECT source_file_id, file_name
FROM collection_members
WHERE collection_name = 'All Documents'
  AND (enrichments->'privilege'->>'privileged')::boolean = false

-- Sort by enrichment score
SELECT source_file_id, file_name,
  enrichments->'scoring'->>'similarity_score' as score
FROM collection_members
WHERE collection_name = 'Relevant Docs'
ORDER BY (enrichments->'scoring'->>'similarity_score')::float DESC

-- Multiple enrichment filters
SELECT source_file_id, file_name
FROM collection_members
WHERE collection_name = 'Review Set'
  AND enrichments->'relevance'->>'level' = 'high'
  AND (enrichments->'privilege'->>'privileged')::boolean = false
```

NOTE: Call `inquisita_discover` with `collection_id` to see which job names and enrichment fields are available on a collection.

## Combined Patterns

```sql
-- Semantic search within a collection
SELECT cm.source_file_id, cm.file_name,
  d.embedding::halfvec(3072) <=> :query_vector::halfvec(3072) AS distance
FROM collection_members cm
JOIN documents d USING (source_file_id)
WHERE cm.collection_name = 'Case Documents'
ORDER BY distance
LIMIT 10

-- Keyword search + category filter
SELECT source_file_id, file_name FROM documents
WHERE category = 'correspondence'
  AND search_text @@ websearch_to_tsquery('inspection OR defect')

-- Analysis results joined with document metadata
SELECT d.file_name, d.media_type, ar.results->>'relevance' as relevance
FROM analysis_results_doc ar
JOIN documents d USING (source_file_id)
WHERE ar.job_name = 'relevance_check'
ORDER BY ar.results->>'relevance'
```