# mem0 v2.0.2 Temporal Backport Summary

## Base

- Base tag: `v2.0.2`
- Base commit: `9043fbf6`

## Scope

Backported temporal memory support onto mem0 v2.0.2:

- `timestamp` / `expiration_date` support in sync and async `add`
- `reference_date` / `show_expired` / `explain` support in sync and async `search`
- Unix epoch timestamp normalization
- Reference-date temporal reranking
- Conservative temporal score adjustment
- `MEM0_DISABLE_TEMPORAL_RERANK=1` switch for baseline comparison
- v2.0.2-specific compatibility fixes:
  - sync temporal metadata initialization
  - `date` import for expiration handling
  - score detail `created_at` fix
  - OSS notice helper fallbacks required by the backport

## Verification Completed

- Import path verified against local v2.0.2 worktree
- `Memory.add`, `Memory.search`, `AsyncMemory.add`, `AsyncMemory.search` signatures verified
- `python -m py_compile mem0/memory/main.py` passes
- Temporal helper smoke passes:
  - epoch seconds normalization
  - epoch milliseconds normalization
  - ISO timestamp normalization
  - expiration date normalization
  - reference-date rerank with score details
- Mem0 OSS server startup passes with Qdrant + Ollama
- LOCOMO predict-only smoke passes:
  - conversation 0 fully ingested: 419 chunks
  - 1 question search/predict-only completed

## Runtime Note

Local OSS + Ollama ingestion is slow. Observed smoke speed:

- 419 chunks / ~39.5 minutes
- LOCOMO full ingestion is ~5,882 chunks
- Estimated single full ingest: ~9 hours
- Estimated baseline + temporal if both re-ingest: ~18–20 hours

For a more efficient full before/after, reuse the same ingestion and only switch `MEM0_DISABLE_TEMPORAL_RERANK` during search.
