# mem0 OSS Temporal LOCOMO Notes

This repository documents an experimental patch set for evaluating temporal timestamp and reference-date paths in mem0 OSS against the official `mem0ai/memory-benchmarks` LOCOMO harness.

This work is for technical evaluation only.

It is not affiliated with, sponsored by, or endorsed by mem0.  
No claim is made that this reproduces mem0 Platform behavior, private implementation details, or official commercial benchmark scores.

## Current status

Completed engineering validation:

- Patched mem0 OSS accepts `timestamp` on `Memory.add(...)`.
- Timestamps are normalized and persisted as `created_at`, `updated_at`, and `timestamp` metadata.
- Patched mem0 OSS accepts `reference_date` on `Memory.search(...)`.
- Reference-date-aware temporal reranking is applied through a conservative additive score adjustment.
- `MEM0_DISABLE_TEMPORAL_RERANK=1` was added as a controlled baseline switch.
- Patched `memory-benchmarks` LOCOMO runner passes session timestamps into add and reference dates into search.
- Qdrant was validated locally through Docker.
- A local uvicorn fallback server was validated using the same patched mem0 checkout when Docker Hub image pull was blocked by auth/token network timeout.
- LOCOMO predict-only smoke passed on conversation 0 with 419 chunks ingested and 3 questions processed.
- Baseline vs temporal-enhanced retrieval behavior was compared on the same local model stack.

Current limitation:

- This repository validates the OSS temporal path and LOCOMO harness integration at retrieval level.
- It does not claim full LOCOMO score reproduction under the local-free Ollama setup.
- Full score reproduction requires a fixed, stronger extraction / answer / judge model stack.

## Scope

The patch set focuses on:

1. Passing observation timestamps through `Memory.add(timestamp=...)`.
2. Using timestamps as `created_at` metadata and as Observation Date for extraction prompts.
3. Passing reference dates through `Memory.search(reference_date=...)`.
4. Applying conservative additive temporal scoring for temporal queries.
5. Providing a baseline switch to disable temporal reranking without changing the rest of the runtime.
6. Wiring these paths into the official LOCOMO benchmark harness.

## Repositories patched

- `mem0ai/mem0`
- `mem0ai/memory-benchmarks`

## mem0 patch summary

Implemented:

- Enable OSS `add(timestamp=...)` path.
- Normalize ISO-like timestamps and Unix epoch timestamps.
- Store timestamp as `created_at` / `updated_at` / `timestamp`.
- Pass timestamp into additive extraction prompt as Observation Date.
- Enable OSS `search(reference_date=...)` path.
- Add conservative additive temporal scoring for:
  - latest / recent / current queries
  - earliest / first queries
  - before / after / when / how long / since / during / by the time temporal-context queries
- Patch both sync and async paths.
- Add `MEM0_DISABLE_TEMPORAL_RERANK=1` for controlled baseline evaluation.

## memory-benchmarks patch summary

Implemented:

- Pass LOCOMO session timestamp through `/memories` to `mem.add(timestamp=...)`.
- Pass LOCOMO reference date through `mem0.search(..., reference_date=...)`.
- Convert LOCOMO reference date to Unix epoch before search.
- Patch Docker FastAPI wrapper to forward `timestamp` and `reference_date`.
- Patch Docker build to install a local patched mem0 checkout instead of reinstalling a remote mem0 package.
- Add local Ollama config examples for smoke validation.
- Ignore local runtime artifacts such as `.env`, downloaded LOCOMO data, Qdrant history DBs, and local smoke outputs.

## Local validation

Completed:

- `py_compile` checks.
- Timestamp normalization smoke test:
  - epoch int
  - epoch string
  - timezone-aware ISO
  - date-only ISO
- `Memory.add(timestamp=epoch)` smoke test.
- `Memory.search(reference_date=...)` smoke test.
- Additive temporal scoring smoke test.
- Docker wrapper syntax check.
- LOCOMO search passthrough syntax check.
- Manual Alice / Google / Meta temporal smoke:
  - baseline mode ranked the stale Google memory above Meta for a current-state query
  - temporal-enhanced mode ranked the newer Meta memory first
- LOCOMO predict-only smoke:
  - conversation 0
  - 419 chunks ingested
  - 3 questions processed
  - per-question retrieval traces generated
- Relative-time extraction sanity check with local `llama3.2:latest`:
  - `yesterday` resolved correctly to May 7, 2023
  - `last year` remained ambiguous
  - `last week` was incorrectly resolved to July 2026

## Not included yet

Full LOCOMO score comparison is not included in this patch package.

A full benchmark run still requires:

1. A fixed model stack for extraction, embedding, answer generation, and judging.
2. Baseline OSS run with `MEM0_DISABLE_TEMPORAL_RERANK=1`.
3. Temporal-enhanced OSS run with temporal reranking enabled.
4. Overall and temporal-category score comparison.
5. Error analysis separating extraction errors, retrieval/rerank errors, answer generation errors, and judge errors.

## Docker status

Qdrant was validated through Docker locally.

The mem0 Docker image build path is prepared through `memory-benchmarks/docker-compose.yml`, but local validation was blocked by Docker Hub auth/token network timeout while pulling `python:3.12-slim`. The same patched mem0 server was therefore validated through a local uvicorn fallback runtime.

## Notes

This is an experimental patch set intended to make the benchmark path testable and reproducible. It does not include private mem0 Platform logic, memory decay, graph invalidation, or any non-public implementation details.
