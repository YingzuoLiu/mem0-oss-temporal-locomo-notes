# mem0 OSS Temporal LOCOMO Notes

This repository documents an experimental patch set for evaluating temporal timestamp and reference-date paths in mem0 OSS against the official `mem0ai/memory-benchmarks` LOCOMO harness.

This work is for technical evaluation only.

It is not affiliated with, sponsored by, or endorsed by mem0.  
No claim is made that this reproduces mem0 Platform behavior, private implementation details, or official commercial benchmark scores.

## Scope

The patch set focuses on:

1. Passing observation timestamps through `Memory.add(timestamp=...)`.
2. Using timestamps as `created_at` metadata and as Observation Date for extraction prompts.
3. Passing reference dates through `Memory.search(reference_date=...)`.
4. Applying conservative additive temporal scoring for temporal queries.
5. Wiring these paths into the official LOCOMO benchmark harness.

## Repositories patched

- `mem0ai/mem0`
- `mem0ai/memory-benchmarks`

## mem0 patch summary

Implemented:

- Enable OSS `add(timestamp=...)` path.
- Normalize ISO-like timestamps and Unix epoch timestamps.
- Store timestamp as `created_at` / `timestamp`.
- Pass timestamp into additive extraction prompt as Observation Date.
- Enable OSS `search(reference_date=...)` path.
- Add conservative additive temporal scoring for:
  - latest / recent / current queries
  - earliest / first queries
  - before / after / when / how long / since / during / by the time temporal-context queries
- Patch both sync and async paths.

## memory-benchmarks patch summary

Implemented:

- Pass LOCOMO session timestamp through `/memories` to `mem.add(timestamp=...)`.
- Pass LOCOMO reference date through `mem0.search(..., reference_date=...)`.
- Convert LOCOMO reference date to Unix epoch before search.
- Patch Docker FastAPI wrapper to forward `timestamp` and `reference_date`.
- Patch Docker build to install a local patched mem0 checkout instead of reinstalling a remote mem0 package.

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

## Not included yet

Full LOCOMO score comparison is not included in this patch package.

A full benchmark run still requires:

1. Docker/Qdrant setup.
2. Fixed LLM, embedding, and judge model configuration.
3. Baseline OSS run.
4. Temporal-enhanced OSS run.
5. Overall and temporal-category score comparison.

## Notes

This is an experimental patch set intended to make the benchmark path testable and reproducible. It does not include private mem0 Platform logic, memory decay, graph invalidation, or any non-public implementation details.
