# Validation Notes

## mem0 branch

Branch:

- `oss-temporal-locomo`

Commits:

- `e8a16f67 Enable OSS temporal timestamp and reference date paths`
- `94288abb Normalize epoch timestamps for temporal OSS paths`
- `ef06337 Use additive temporal scoring for reference date search`
- `2c2fb3b0 Make temporal score adjustment conservative`

Validated locally:

- `python -m py_compile mem0/memory/main.py`
- timestamp normalization smoke test
- `Memory.add(timestamp=epoch)` smoke test
- `Memory.search(reference_date=...)` smoke test
- additive temporal scoring smoke test
- conservative temporal score smoke test

## memory-benchmarks branch

Branch:

- `oss-temporal-locomo-benchmark`

Commits:

- `61b55b3 Pass temporal parameters through LOCOMO OSS benchmark`
- `15e1982 Build LOCOMO OSS server from local patched mem0`

Validated locally:

- `python -m py_compile benchmarks/common/mem0_client.py`
- `python -m py_compile benchmarks/locomo/run.py`
- `python -m py_compile docker/mem0/main.py`

## Checked design points

- `Memory.search()` uses `top_k`, so the Docker wrapper correctly maps `limit` to `top_k`.
- `user_id`, `agent_id`, and `run_id` are passed through `filters`, matching current OSS search validation.
- Temporal scoring is additive and conservative:
  - original retrieval score remains dominant
  - temporal score is a small bias
  - plain non-temporal queries are unchanged
- Word-boundary matching avoids false positives such as `now` matching inside `known`.

## Remaining benchmark work

Full LOCOMO run is pending Docker/Qdrant setup and fixed model configuration.
