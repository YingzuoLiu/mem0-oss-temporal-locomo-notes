# Validation Notes

## mem0 branch

Branch:

- `oss-temporal-locomo`

Commits validated locally:

- `e8a16f67 Enable OSS temporal timestamp and reference date paths`
- `94288abb Normalize epoch timestamps for temporal OSS paths`
- `ef06337 Use additive temporal scoring for reference date search`
- `2c2fb3b0 Make temporal score adjustment conservative`
- `a0793317 Add switch to disable temporal rerank for baseline eval`

Validated locally:

- `python -m py_compile mem0/memory/main.py`
- timestamp normalization smoke test
- `Memory.add(timestamp=epoch)` smoke test
- `Memory.search(reference_date=...)` smoke test
- additive temporal scoring smoke test
- conservative temporal score smoke test
- controlled baseline switch with `MEM0_DISABLE_TEMPORAL_RERANK=1`

## memory-benchmarks branch

Branch:

- `oss-temporal-locomo-benchmark`

Commits validated locally:

- `61b55b3 Pass temporal parameters through LOCOMO OSS benchmark`
- `15e1982 Build LOCOMO OSS server from local patched mem0`
- `d76a2dc Add local Ollama config examples for LOCOMO smoke`

Validated locally:

- `python -m py_compile benchmarks/common/mem0_client.py`
- `python -m py_compile benchmarks/locomo/run.py`
- `python -m py_compile docker/mem0/main.py`
- Docker Qdrant container reachable at `http://localhost:6333`
- local uvicorn mem0 OSS server reachable at `http://localhost:8888`
- Ollama local LLM / embedder reachable at `http://localhost:11434`

## Checked design points

- `Memory.search()` uses `top_k`, so the Docker wrapper correctly maps `limit` to `top_k`.
- `user_id`, `agent_id`, and `run_id` are passed through `filters`, matching current OSS search validation.
- LOCOMO session dates are converted into Unix timestamps and passed into `mem0.add(..., timestamp=...)`.
- LOCOMO reference dates are converted into Unix timestamps and passed into `mem0.search(..., reference_date=...)`.
- Temporal scoring is additive and conservative:
  - original retrieval score remains dominant
  - temporal score is a small bias
  - plain non-temporal queries are unchanged
- Word-boundary matching avoids false positives such as `now` matching inside `known`.
- `MEM0_DISABLE_TEMPORAL_RERANK=1` allows baseline and temporal-enhanced runs to use the same code path and model stack, with only temporal reranking toggled.

## Manual temporal smoke

A manual Alice / Google / Meta smoke test was used to verify that the temporal path changes retrieval ordering rather than only accepting new request fields.

Input memories:

- `2023-05-08`: Alice started working at Google.
- `2024-08-09`: Alice left Google and joined Meta.

Observed behavior:

- Baseline mode with `MEM0_DISABLE_TEMPORAL_RERANK=1` kept the stale Google memory above Meta for the query `Where does Alice work now?`.
- Temporal-enhanced mode with temporal reranking enabled ranked the newer Meta memory first.
- Qdrant payload contained normalized `created_at`, `updated_at`, and `timestamp` fields.

This validates the end-to-end path:

`timestamp add -> normalization -> Qdrant payload -> reference_date search -> temporal rerank -> changed retrieval ordering`.

## LOCOMO predict-only smoke

Runtime setup:

- Mem0 OSS server: local uvicorn fallback on `http://localhost:8888`
- Vector store: Qdrant Docker container on `http://localhost:6333`
- Local LLM: Ollama `llama3.2:latest`
- Local embedder: Ollama `mxbai-embed-large:latest`
- Dataset: local `datasets/locomo/locomo10.json`

Command shape:

```bash
python -m benchmarks.locomo.run \
  --project-name local_ollama_temporal_smoke \
  --backend oss \
  --mem0-host http://localhost:8888 \
  --dataset-path datasets/locomo/locomo10.json \
  --conversations 0 \
  --max-questions 3 \
  --top-k 10 \
  --predict-only \
  --max-workers 1 \
  --rpm 30 \
  --output-dir results/locomo/local_smoke \
  --debug
```

Observed result:

- Conversation 0 ingested successfully.
- 419 chunks were processed.
- 3 questions were processed.
- Per-question prediction files were generated.
- Each question output included `reference_date` and retrieval traces.
- Retrieved memories included `created_at` / `updated_at` timestamps.

Example temporal question:

> When did Caroline go to the LGBTQ support group?

Reference date:

> 9:55 am on 22 October, 2023

The top retrieved memories included multiple similar memories with different dates, confirming that LOCOMO temporal questions surface the exact type of multi-date ambiguity that the timestamp/reference-date path is meant to support.

## Local before/after retrieval comparison

Two predict-only local runs were produced under the same local Ollama/Qdrant setup:

- Baseline: `local_ollama_baseline_smoke`
- Temporal-enhanced: `local_ollama_temporal_compare`

Observed examples:

- `conv0_q0` temporal question: temporal reranking increased the scores of relevant dated LGBTQ support group memories.
- `conv0_q1` temporal question: temporal reranking promoted another `last year` sunrise memory into the top ranks.
- `conv0_q2` non-temporal question: ranking was unchanged, which is expected because temporal intent was not triggered.

## Relative-time extraction sanity check

A targeted local extraction sanity check was run with observation timestamp `2023-05-08`.

Input text:

> Yesterday I went to the LGBTQ support group. Last year I painted a sunrise over a lake. Last week I started looking into counseling certification.

Observed extraction behavior with local `llama3.2:latest`:

- `yesterday` resolved correctly to May 7, 2023.
- `last year` remained ambiguous as around 2022 or 2023.
- `last week` was incorrectly resolved to July 2026.

Conclusion:

The local-free Ollama setup is sufficient for engineering smoke validation, but it is not reliable enough for score-level LOCOMO reproduction. Full score reproduction should use a stronger and fixed extraction / answer / judge model stack.

## Docker status

Qdrant Docker was validated locally.

The mem0 Docker compose path is prepared, but the mem0 image build was blocked during local validation by Docker Hub auth/token network timeout while pulling `python:3.12-slim`.

The same patched mem0 server was therefore validated through local uvicorn fallback. Docker compose should be retried when Docker Hub connectivity is stable.

## Remaining benchmark work

Full LOCOMO score reproduction is still pending.

Required before claiming score-level comparison:

1. Choose and freeze extraction, embedding, answer, and judge model stack.
2. Run baseline OSS with `MEM0_DISABLE_TEMPORAL_RERANK=1`.
3. Run temporal-enhanced OSS with temporal reranking enabled.
4. Compare overall and temporal-category scores.
5. Run error analysis separating:
   - extraction errors
   - retrieval/rerank errors
   - answer generation errors
   - judge errors
