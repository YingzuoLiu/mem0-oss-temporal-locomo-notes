# Score Comparison Notes

## Important scope note

The numbers below are retrieval-level scores from the local LOCOMO predict-only smoke run.

They are **not** official LOCOMO final answer/judge scores.

They are useful for validating that the patched OSS temporal path changes retrieval behavior in the expected direction under a controlled baseline switch.

The numbers were re-checked from the local returned retrieval JSON files, not from a manually typed summary table. The files are local smoke artifacts and are intentionally ignored by git because they include downloaded benchmark/runtime outputs.

## Runtime

- mem0 server: patched OSS mem0, local uvicorn fallback
- vector store: Qdrant
- LLM: Ollama `llama3.2:latest`
- embedder: Ollama `mxbai-embed-large:latest`
- benchmark harness: `mem0ai/memory-benchmarks` LOCOMO runner
- dataset: LOCOMO conversation 0 from `locomo10.json`
- mode: `--predict-only`
- questions: first 3 questions
- top_k: 10

## Source artifact paths

Baseline run:

```text
results/locomo/local_smoke/predicted_local_ollama_baseline_smoke/conv0_q0.json
results/locomo/local_smoke/predicted_local_ollama_baseline_smoke/conv0_q1.json
results/locomo/local_smoke/predicted_local_ollama_baseline_smoke/conv0_q2.json
```

Temporal-enhanced run:

```text
results/locomo/local_smoke/predicted_local_ollama_temporal_compare/conv0_q0.json
results/locomo/local_smoke/predicted_local_ollama_temporal_compare/conv0_q1.json
results/locomo/local_smoke/predicted_local_ollama_temporal_compare/conv0_q2.json
```

These files contain returned `retrieval.search_results[*].score` values. They do not include `score_details`; explain-mode temporal score breakdown was not captured in this smoke run.

## Baseline vs temporal-enhanced setup

Both runs used the same local code and model stack.

The only intended behavior switch was:

- Baseline: `MEM0_DISABLE_TEMPORAL_RERANK=1`
- Temporal-enhanced: temporal reranking enabled

This avoids mixing temporal changes with model, dependency, or runtime drift.

## Top-1 returned retrieval score comparison

| Scope | Baseline top-1 returned retrieval score | Temporal-enhanced top-1 returned retrieval score | Delta |
|---|---:|---:|---:|
| Temporal questions only (`q0` + `q1`) | 0.8700965 | 0.8950965 | +0.0250 |
| All 3 smoke questions | 0.85151972 | 0.8681863867 | +0.0167 |
| Non-temporal control (`q2`) | 0.81436616 | 0.81436616 | 0.0000 |

Per-question details:

| Question | Category | Baseline top-1 | Temporal-enhanced top-1 | Delta | score_details |
|---|---|---:|---:|---:|---|
| `conv0_q0`: When did Caroline go to the LGBTQ support group? | temporal | 0.9230882 | 0.9480882 | +0.0250 | not captured |
| `conv0_q1`: When did Melanie paint a sunrise? | temporal | 0.8171048 | 0.8421048 | +0.0250 | not captured |
| `conv0_q2`: What fields would Caroline be likely to pursue in her education? | open-domain control | 0.81436616 | 0.81436616 | 0.0000 | not captured |

Interpretation:

- Temporal questions showed a positive returned-score shift after enabling reference-date-aware reranking.
- The non-temporal control question was unchanged.
- This is consistent with the current conservative rerank implementation: `when`-style temporal-context queries receive a small fixed additive bias for memories dated before the reference date.
- Because `score_details` was not captured, this table should be treated as returned-score evidence, not as a full temporal-score breakdown.

## Example trace: Alice / Google / Meta

A separate manual smoke test was used to verify that the patched temporal path changes ranking in a clean and explainable case.

Input memories:

- `2023-05-08`: Alice started working at Google.
- `2024-08-09`: Alice left Google and joined Meta.

Query:

> Where does Alice work now?

Observed behavior:

- Baseline mode ranked the stale Google memory above Meta.
- Temporal-enhanced mode ranked the newer Meta memory first.

This validates the end-to-end path:

`timestamp add -> timestamp normalization -> Qdrant payload -> reference_date search -> temporal rerank -> changed retrieval ordering`

## Integration path

The patched integration is:

```text
LOCOMO dataset
  -> memory-benchmarks LOCOMO runner
  -> conversation/session timestamp -> mem0.add(timestamp=...)
  -> QA reference_date -> mem0.search(reference_date=...)
  -> patched mem0 OSS server
  -> timestamp normalized into created_at / updated_at / timestamp
  -> Qdrant payload persistence
  -> semantic search candidates
  -> reference-date-aware temporal rerank
  -> baseline vs temporal-enhanced retrieval comparison
```

Code-level changes:

### memory-benchmarks

- LOCOMO ingestion passes each session timestamp through `/memories`.
- LOCOMO retrieval passes each QA `reference_date` through `/search`.
- `benchmarks/common/mem0_client.py` supports `reference_date`.
- `docker/mem0/main.py` forwards `timestamp` and `reference_date` to mem0 OSS.

### mem0 OSS

- `Memory.add(...)` accepts `timestamp`.
- Timestamp inputs support ISO-like strings and Unix epoch seconds/milliseconds.
- Timestamp is normalized and stored as `created_at`, `updated_at`, and `timestamp` metadata.
- `Memory.search(...)` accepts `reference_date`.
- Search applies conservative additive temporal reranking when temporal intent is detected.
- `MEM0_DISABLE_TEMPORAL_RERANK=1` disables temporal reranking for controlled baseline runs.

## Remaining work for official-like LOCOMO scores

The current local-free run validates engineering behavior, but should not be treated as a final official LOCOMO score.

A score-level comparison still needs:

1. A fixed extraction model.
2. A fixed embedding model.
3. A fixed answer-generation model.
4. A fixed judge model.
5. Baseline and temporal-enhanced runs with the exact same stack.
6. Overall and temporal-category score comparison.
7. Error analysis separating extraction, retrieval/rerank, answer generation, and judge errors.

The local `llama3.2:latest` model is sufficient for smoke validation but not reliable enough for score-level reproduction: a targeted relative-time extraction check resolved `yesterday` correctly, left `last year` ambiguous, and incorrectly resolved `last week` to July 2026.
