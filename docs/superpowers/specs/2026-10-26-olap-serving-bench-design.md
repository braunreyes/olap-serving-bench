# OLAP Serving Bench: v1 Design

**Date:** 2026-10-26 · **Status:** draft for review

## Goal

A portfolio benchmark that serves the same wallet-investigation workload from PostgreSQL 16, ClickHouse, and StarRocks. It tunes each engine one change at a time and writes up why each change helped or didn't. Built for TRM Labs' Staff Data Platform Engineer (StarRocks serving layer) and other data-platform roles.

What it shows: matching physical design to query shape, reading query plans and profiles, honest benchmarking (a correctness gate, equal resources, cold vs. warm), and clear writing about tradeoffs.

## Non-Goals (v1)

- Claiming which engine is "best". The writeup explains *why* each one behaves as it does on *this* workload
- Multi-node clusters, cloud deploys, or published leaderboard numbers (SSB, ClickBench)
- Ingestion speed or streaming loads. v1 bulk-loads from Parquet

## Success Criteria

1. `make up && make load SCALE=1d && make bench` on a clean laptop runs all 6 queries on all 3 engines and writes `results/<run_id>.json`.
2. The correctness gate passes: each query's result checksum matches across engines before any timing is recorded.
3. `TUNING.md` has at least 3 logged changes per engine, each with a hypothesis, the change, the result, and a plan or profile excerpt.
4. `REPORT.md` is generated from results: latency charts per query × engine × naive/tuned at both scales, plus written conclusions.
5. CI is green on every PR: lint, types, unit tests, and a smoke run on PG + ClickHouse at the 1-day scale.
6. The author can explain every DDL choice and every tuning result without notes.

## Data

Exported from project 2 (`eth-wallet-model`) marts as Parquet:

| Table | Grain | Approx. rows (30d) |
|---|---|---|
| `fct_token_transfer` | One row per (`transaction_hash`, `log_index`) | Tens of millions (seed tokens only) |
| `dim_wallet` | One row per wallet version (SCD2) | Millions |
| `fct_wallet_daily` | One row per wallet × token × day | Tens of millions |

Two scales: `1d` (CI and quick iterations) and `30d` (the reported numbers). `make export` runs project 2's build and copies the marts into `data/<scale>/`. The data's row counts and checksums are recorded so every run states exactly what it ran on.

## Engines

| Engine | Naive DDL | Tuning levers to explore |
|---|---|---|
| PostgreSQL 16 | Plain heap tables, primary keys only | Range partitioning by month, BRIN on `block_timestamp`, B-tree and covering indexes on `from`/`to`, `work_mem`, parallel workers |
| ClickHouse | `MergeTree ORDER BY tuple()` | `ORDER BY` choice, projections (a second sort order for `to_address`), skip indexes, `LowCardinality` |
| StarRocks | Duplicate key table, default buckets | Partition and bucket keys, sort key, synchronous and async materialized views, colocated joins, bitmap/bloom indexes |

All three run in `docker compose` with **equal CPU and memory caps** and pinned versions, recorded in each result file.

## Query Suite

| Q | Workload | Stresses |
|---|---|---|
| 1 | Every transfer for one wallet in the last 30 days, `from` **or** `to` | Sort keys: one ordering can't serve both columns |
| 2 | Top 10 counterparties for one wallet by amount | Filter, then aggregate |
| 3 | Daily volume per token over the full range | Full scan + group-by |
| 4 | Per wallet: amount received from wallets labeled sanctioned at transfer time | Join with a time-range condition (`valid_from <= t < valid_to`) |
| 5 | Wallets that received from a wallet that received from a sanctioned wallet (2 hops) | Multi-join, fan-out |
| 6 | Q1 with 50 concurrent clients over random wallets | Serving concurrency: p99 under load |

Each query has one SQL file per dialect, with parameters bound by the harness and never interpolated. A parameter set (wallets, tokens, dates) is drawn once with a fixed seed and saved, so every engine gets the same inputs.

## Harness (Author)

```bash
bench run --engine pg|clickhouse|starrocks|all --suite all --scale 1d --variant naive|tuned \
          --runs 5 --warmup 1 --concurrency 1 --cache cold|warm
```

- **Correctness gate first:** each query runs once per engine; a canonical checksum of the sorted result (with a fixed numeric precision) must match across engines. A mismatch stops the run with exit `1` and prints a diff.
- **Timing:** wall-clock latency per run from the client side; p50, p95, and p99 per query. For Q6: throughput and p99 at the given concurrency.
- **Engine-side evidence:** bytes and rows read from each engine's own records (for PG, `pg_stat_statements` block counts plus `EXPLAIN (ANALYZE, BUFFERS)` in a separate evidence pass, which is fine on a local benchmark and never done against production; `system.query_log` for ClickHouse; query profiles for StarRocks), saved next to the timings.
- **Cold cache:** restart the engine container and drop the OS page cache where the platform allows it; record which cold method was used.
- Results go to `results/<run_id>.json`: engine, version, scale, variant, data checksum, hardware, and per-query stats.
- Exit codes: `0` ok · `1` correctness mismatch · `2` setup or engine error.

The timing statistics, the checksum, and the parameter sampling are pure functions with unit tests. Engine access goes through one small client interface per engine, so tests use a fake.

## Writeups (Author)

- **`TUNING.md`:** one entry per change, always in this shape: **Hypothesis · Change · Result (before → after, p50/p99, bytes read) · Evidence (plan or profile excerpt) · Keep or revert**.
- **`REPORT.md`:** generated charts plus the author's conclusions per query: which physical design fits which access pattern, and what it costs to write and store.

## Ownership

| Author writes (core) | Coach writes (plumbing) |
|---|---|
| Naive and tuned DDL per engine | `docker-compose.yml`, resource caps, version pins |
| Query SQL per dialect | Loaders (Parquet → each engine), `make export` |
| Harness: gate, timing, stats, evidence capture | Chart generation for `REPORT.md` |
| `TUNING.md`, `REPORT.md` conclusions, README tradeoffs and 100× section | Makefile, CI workflows; reviews and mutation tests |

## Repo Layout

```text
olap-serving-bench/
├── docker-compose.yml
├── Makefile                  # up, down, export, load, bench, report, check
├── engines/{pg,clickhouse,starrocks}/{naive,tuned}.sql   # author
├── queries/{pg,clickhouse,starrocks}/q1.sql … q6.sql     # author
├── src/olap_serving_bench/   # harness (author), loaders + charts (coach)
├── tests/
├── results/                  # committed JSON for the reported runs
├── TUNING.md
├── REPORT.md
└── README.md
```

## CI

| Job | When | Runs |
|---|---|---|
| `check` | Every push | ruff, mypy strict, unit tests with 80% coverage, markdownlint |
| `smoke` | Every PR | PG + ClickHouse at `1d`: load, gate, one run each |
| `starrocks-smoke` | Manual dispatch | The same for StarRocks (too heavy for a default runner) |

## Milestones

| # | Deliverable | Done when |
|---|---|---|
| M1 | Stack + loaders + naive DDL | All 3 engines load `1d` and row counts match the export |
| M2 | Harness + correctness gate | Gate passes for Q1–Q5 on all engines at `1d` |
| M3 | Naive baseline at `30d` | Results committed; Q6 runs at concurrency 50 |
| M4 | Tuning passes | 3+ `TUNING.md` entries per engine; tuned results committed |
| M5 | Report + publish | Success criteria 1–6 met; repo public |

## v2 Backlog

- Trino on Iceberg (project 1's lake) as a fourth engine
- Ingest under load: query latency while streaming inserts run
- StarRocks async materialized-view refresh cost vs. query savings
- Cost per query on a cloud instance type

## Risks

| Risk | Mitigation |
|---|---|
| StarRocks needs more memory than a laptop or runner has | Use the all-in-one image with capped memory; StarRocks CI is manual only; document the hardware |
| Results differ across engines because of numeric or NULL semantics, not bugs | Canonical checksum rules (precision, NULL ordering) defined in M2 and unit tested |
| Benchmarking the cache instead of the engine | Cold and warm runs reported separately; warmup runs excluded |
| An unfair comparison through one engine's defaults | Equal caps; naive and tuned variants for every engine; every setting recorded in results |
| Project 2 isn't done when M1 starts | `make export` can also run project 2's extract on its fixture; M1 needs only `1d` |
