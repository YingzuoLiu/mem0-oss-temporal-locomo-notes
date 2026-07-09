# Controlled Single-Ingestion Search Comparison

## Scope

This note records a cleaner local comparison than the earlier smoke table.

The goal is to isolate the search-time effect of the temporal reranker:

1. Ingest LOCOMO conversation 0 once.
2. Keep the same Qdrant collection and the same `user_id`.
3. Run baseline search with `MEM0_DISABLE_TEMPORAL_RERANK=1`.
4. Run temporal-enhanced search with temporal reranking enabled.
5. Compare returned search results for the same 152 questions.

This is still a retrieval/search-level comparison. It is **not** an official LOCOMO final answer/judge score.

## Run metadata

- Dataset: LOCOMO `locomo10.json`, conversation 0 (`Caroline & Melanie`)
- Ingestion: 419/419 chunks completed
- Questions compared: 152/152
- User ID: `locomo_0_9d6ed635`
- Baseline directory: `results/locomo/controlled_single_ingest/predicted_controlled_single_ingest_baseline_searchonly`
- Temporal directory: `results/locomo/controlled_single_ingest/predicted_controlled_single_ingest_temporal_searchonly`
- Evidence summary files:
  - `results/locomo/controlled_single_ingest/evidence/controlled-search-compare-2026-07-09.json`
  - `results/locomo/controlled_single_ingest/evidence/controlled-search-compare-2026-07-09.txt`

## Summary

| Metric | Value |
|---|---:|
| Total compared | 152 |
| Top-1 changed | 0 |
| Rank order changed | 5 |
| Temporal questions | 37 |
| Temporal top-1 changed | 0 |
| Temporal rank order changed | 1 |

## By category

| Category | n | Top-1 changed | Rank order changed | Avg top-1 delta |
|---|---:|---:|---:|---:|
| multi-hop | 32 | 0 | 1 | +0.004063 |
| open-domain | 13 | 0 | 0 | +0.001923 |
| single-hop | 70 | 0 | 3 | +0.006143 |
| temporal | 37 | 0 | 1 | +0.024730 |

## Main interpretation

- The controlled comparison confirms that the temporal reranker is connected and affects returned scores on the same ingested memory pool.
- Temporal questions show the largest average top-1 returned-score delta: `+0.024730`.
- Top-1 did not change for any of the 152 questions. This means the current conservative reranker is not strong enough to change the first returned memory on this local LOCOMO run.
- Rank order changed in 5/152 questions, including 1 temporal question, with the same candidate set.
- This result should be presented as **engineering validation of timestamp/reference_date wiring and search-time rerank behavior**, not as final benchmark improvement.

## Temporal rank-order changed example

`conv0_q10`:

- Category: temporal
- Question: `How long has Caroline had her current group of friends for?`
- Ground truth: `4 years`
- Baseline top-1 score: `0.806057`
- Temporal top-1 score: `0.821057`
- Baseline top-1 memory: `Caroline and her friends and family have been together for 4 years since she moved from her home country.`
- Temporal top-1 memory: `Caroline and her friends and family have been together for 4 years since she moved from her home country.`
- Top-1 changed: `False`
- Rank order changed: `True`
- Same candidate set: `True`

## Evidence limitation

This controlled run is cleaner than the earlier two-ingestion smoke comparison, but it still has limitations:

1. It is retrieval/search-level only.
2. It does not include official LOCOMO answer-generation and judge scores.
3. It does not include explain-mode `score_details` for each candidate.
4. The conservative reranker mainly changes scores and some lower-rank ordering; it did not change top-1 on this local run.

## Recommended external wording

A safe summary is:

> We completed a controlled single-ingestion search-only comparison on LOCOMO conversation 0. Using the same ingested memory pool and the same user ID, temporal reranking produced the largest returned-score lift on temporal questions (+0.02473 average top-1 score delta). Top-1 did not change in this local run, while rank order changed in 5/152 questions, including 1 temporal question. This validates the timestamp/reference_date integration and search-time rerank path, but it is not an official LOCOMO final answer/judge score and should not be framed as final benchmark improvement.
