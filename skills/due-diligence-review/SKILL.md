---
name: due-diligence-review
description: >
  Use this skill when the user asks for a due diligence review, data-room review, or
  any task that requires systematic legal evaluation of a corpus of documents organized
  by subject-matter category. Triggers include: M&A due diligence, acquisition
  diligence, regulatory diligence, litigation pre-engagement review, employment
  due diligence, financing diligence, or any task whose deliverable is a memo
  organized by issue category with risk ratings. Use even when the user does not
  mention "due diligence" by name — recognize the pattern.
version: 0.1.0
---

# Due Diligence Review

## Why this skill exists

Due diligence is the prototypical "many documents, many categories, find every issue" legal task. It is the workflow Inquisita was built for. The default agent failure mode on these tasks is **selective reading** — the agent picks 3-6 representative documents, writes a memo from limited evidence, and misses material issues lurking in the documents it didn't open.

This skill prescribes the canonical sequence so that **every document in the data room gets scrutinized at the right depth, no matter how large the corpus**. Follow it.

## The mandatory pattern

```
discover → survey → categorize → analyze (per category) → synthesize
```

Five phases, in order. Each phase has a specific tool. Do not skip phases.

### Phase 1 — Discover (1 call)

Always start with `inquisita_discover(matter_id)`. This returns categories, media_types, file counts, existing collections, enrichment schemas, and view definitions. Without this you do not know the shape of the corpus you are reviewing.

### Phase 2 — Survey (1-3 query calls)

Run **broad** `inquisita_query` calls against the `documents` view to enumerate every file in the matter, including filename, file_type, category, and `overall_summary`. The point is not to read every document — it is to ensure you have at least the summary of every document in your working context before you decide what to analyze in depth.

A typical survey query:
```sql
SELECT source_file_id, file_name, category, file_type, media_type,
       LEFT(overall_summary, 600) AS summary_preview
FROM documents
ORDER BY category, file_name
```

If the corpus exceeds ~50 docs and individual summaries are long, run a `summary_only: true` analyze pass first as a **screening step** that produces a structured one-line classification per document. Use the screening pass to decide which categories merit deep review — never to substitute for the deep review itself.

### Phase 3 — Categorize (no tool call — you do this)

After the survey, partition the corpus by **due diligence subject-matter category**. Use whatever categories fit the deal type. Common DD categories include — but are not limited to:

- **Corporate / Governance** — formation docs, LLC/stockholder agreements, board minutes, capitalization
- **Material Contracts** — supply agreements, customer contracts, distribution agreements, MSAs, licenses
- **Change of Control / Assignment** — change-of-control clauses, consent requirements, anti-assignment provisions across all contracts
- **IP / IT** — patents, trademarks, license-in / license-out agreements, IP assignments, open-source compliance
- **Litigation / Disputes** — pending litigation, settlements, demand letters, arbitration
- **Regulatory / Compliance** — permits, regulatory correspondence, compliance programs, sanctions screening
- **Employment / Benefits** — employment agreements, restrictive covenants, severance plans, benefit programs, labor matters
- **Tax** — tax returns, tax compliance summaries, audit history, tax exposures
- **Environmental** — environmental compliance, permits, remediation history, hazardous materials
- **Real Estate** — leases, owned property, title issues
- **Financial / Accounting** — financial statements, audit reports, debt instruments, off-balance-sheet items
- **Insurance** — coverage summaries, claims history, gaps
- **Data Privacy** — privacy program, breach history, data processing agreements

Map the documents you surveyed onto these categories. A single document may belong to multiple categories (an employment agreement with a change-of-control clause spans Employment AND CoC).

**Do not skip categories that have at least one relevant document in the corpus.** Every category with material in the data room must get its own analysis pass. The most common failure of DD agents is to deep-read a few "obvious" docs and skip everything else.

### Phase 4 — Analyze (one chunk-level job per category)

For **each** category from Phase 3 that has documents, submit **one `inquisita_analyze` job at chunk level** (the default — do not pass `summary_only: true` here). Each job extracts the structured findings relevant to that category.

Job structure per category:
- **`job_name`**: short identifier (`corporate_governance`, `material_contracts_coc`, `ip_assets`, `litigation_disputes`, etc.)
- **`sql`**: chunk-level SQL scoped to the relevant docs:
  ```sql
  SELECT source_file_id, chunk_index FROM chunks
  WHERE source_file_id IN (
    SELECT source_file_id FROM documents WHERE category IN (...) OR file_name IN (...)
  )
  ```
- **`config.prompt`**: category-specific prompt that names the parties / deal context, lists the specific issues to extract, defines the output fields, and instructs the analysis LLM what to do when an issue is absent (return null / empty, do not fabricate). Remember: the analysis LLM has zero case context unless you include it in the prompt.
- **`config.output_schema`**: structured fields capturing the **what / where / who / risk** of each finding. Typical fields: issue type (enum), quoted_language (string), party (string), risk_rating (enum), section_or_page (string), recommended_protection (string). Use nested object arrays when one document may contain multiple findings.
- **`config.intelligence`**: `low` for screening / classification, `medium` for nuanced legal characterization (most DD work), `high` only for genuinely complex reasoning (rare).

Submit all category jobs, then poll `inquisita_get_analysis_job` until each is `complete` or `complete_with_errors`. Many can run in parallel — submit them all before polling.

### Phase 5 — Synthesize

For each completed job, query its results:
```sql
SELECT * FROM analysis_results_chunk
WHERE job_name = 'litigation_disputes'
  AND (results->>'is_finding')::boolean = true
ORDER BY (results->>'risk_rating')
```

Organize the deliverable memo by the same categories from Phase 3. For each finding:
- State the issue
- Quote the source language (already extracted by the analysis)
- Cite the source document and section/page
- Assign or reflect the risk rating
- Recommend deal protection (rep, indemnity, closing condition, pre-closing remedy, walk right)

The structured analysis output gives you everything the memo needs. The agent's job in this phase is organization and synthesis, not re-reading docs.

## Optional: persist findings as a collection

For longer-running matters, create one chunk-level collection per major category (`inquisita_create_collection` with SQL pulling the relevant findings) and link the analysis jobs (`inquisita_enrich_collection`). This turns one-shot analysis into reusable, team-visible work product.

## Anti-patterns — never do these

1. **Read three docs, write a memo.** This is the dominant failure mode. The corpus is on Inquisita; use it.
2. **Skip Phase 4 in favor of `inquisita_query` alone.** Queries return text and summaries — they do not run an LLM over each chunk to extract structured findings. Analysis is what produces the evidence rows the memo cites.
3. **Use `summary_only: true` for finding issues.** The summary is a high-level abstract; specific provisions live in specific sections. Summary-only is for *screening* and *classification* only, never for diligence findings.
4. **Run one giant `inquisita_analyze` over the whole corpus with a vague prompt.** Per-category jobs with focused prompts produce far better results. The analysis LLM does better with narrow, well-scoped questions.
5. **Selectively pick "interesting" docs and only analyze those.** Every category present in the corpus gets a pass. The agent does not pre-judge what matters.

## Tool quick reference (DD-specific)

| Tool | When in the sequence |
|---|---|
| `inquisita_discover(matter_id)` | Phase 1, once |
| `inquisita_query` | Phase 2 (survey), Phase 5 (results) |
| `inquisita_analyze` (chunk-level, default) | Phase 4, once per category |
| `inquisita_get_analysis_job` | Phase 4, poll each submitted job |
| `inquisita_create_collection` + `inquisita_enrich_collection` | Optional, after Phase 5, for persistent work product |

## Sanity check before writing the memo

Before drafting, confirm:

- [ ] Every category from Phase 3 has a completed analysis job
- [ ] No category in the corpus was skipped because "it didn't look interesting"
- [ ] Every finding in the memo has a quoted source phrase and document citation (from the analysis output)
- [ ] Risk ratings come from the analysis output, not from your own intuition

If any box is unchecked, run the missing analysis before writing the memo.
