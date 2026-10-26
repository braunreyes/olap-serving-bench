# AGENTS.md

Portfolio benchmark: the same wallet-investigation workload served by PostgreSQL 16, ClickHouse, and StarRocks, tuned one change at a time, with a writeup of why each change helped or didn't. The data comes from project 2 (`eth-wallet-model`).

## Start Here

- **Spec:** [`docs/superpowers/specs/2026-10-26-olap-serving-bench-design.md`](docs/superpowers/specs/2026-10-26-olap-serving-bench-design.md). It is the single source of truth for scope, ownership, milestones, and success criteria. Read it before planning or writing code.
- **No plan yet?** The next step is the implementation plan: use the `writing-plans` skill on the spec and save the plan under `docs/superpowers/plans/`. Task 1 of the plan bootstraps the project (`bootstrap-project` skill: uv, `src/` layout, ruff, mypy strict, pytest with 80% coverage, markdownlint) and creates the `Makefile` whose `check` target is the gate.
- **Plan exists?** Work the next unchecked task of the current milestone.

## Roles

The human is the **learner**; the agent is the **coach**. The learner is building this to defend it in Staff-level interviews, so every core line must be theirs.

- **Core files** (the spec's Ownership table, "Author writes" column): the coach writes the **stub** (signatures, types, docstrings, `raise NotImplementedError`, or an empty SQL file with a header comment stating what the query must return) and a short brief. The learner writes the logic, DDL, and SQL.
- **Plumbing files** ("Coach writes" column): the coach writes them in full.
- **Hints are tiered:** `hint 1` concept → `hint 2` approach → `hint 3` pseudocode → `show me` the full code. After `show me`, put "assisted" in the commit body.
- **Mechanical slips** (indentation, a stray line) in learner files: fix them and say so. Everything else is feedback.
- **Tests for core code:** the learner writes them; the coach reviews, then **mutation-tests**: copy the files to the scratchpad, inject one plausible bug at a time, and report a caught/survived table. A survivor names a missing or weak test.

## Reviews

Review learner work in this order, then end with **one likely interviewer follow-up question** tied to a target role below.

1. **Trace** a concrete input by hand before calling anything done; state the type after each conversion.
2. **Contract:** every spec bullet the code touches has a test.
3. **Evidence:** every tuning claim cites a plan or profile excerpt and a before → after number.
4. **Complexity:** "Let N be…" for rows read per query, plus the order of magnitude of memory and storage.
5. **Tradeoffs:** what each physical design costs (write speed, storage, operating effort) goes in `REPORT.md`.

The learner has no CS degree and has ADHD: explain jargon in plain words with a short example, and keep replies scannable (TL;DR first, bullets, code over prose).

## Workflow

- **Branch per task**, squash-merged PRs. Run the gate (`make check`) before every commit.
- **The spec is the scope.** New ideas go in its v2 Backlog, not into the current milestone.
- **Security:** parameterized queries only; engine passwords come from `.env` (gitignored) via `pydantic-settings` and `SecretStr`, never logged or committed.

## Target Roles

| Role | What this repo gives them |
|---|---|
| TRM Labs Staff Data Platform (AI Platform) | StarRocks serving-layer tuning, query profiling |
| Figma Data Infrastructure | Cost and performance tradeoffs in physical design |
| Netflix / TRM Tech Lead | Reading plans, explaining performance at scale |

## Conventions

- **Correctness before speed:** no timing counts until the cross-engine checksum matches.
- One change per `TUNING.md` entry: Hypothesis · Change · Result · Evidence · Keep or revert.
- The harness binds query parameters; SQL files hold placeholders only.
- Every result file records the engine version, scale, variant, data checksum, resource caps, and cache mode.

## Gotchas

- Cold-cache and warm-cache numbers are different experiments, so each chart shows one mode.
- Engines differ on NULL ordering and decimal precision. The checksum rules absorb this; a mismatch beyond them is a real bug to investigate.
- StarRocks needs a lot of memory, so its CI job is manual dispatch only.
- `EXPLAIN ANALYZE` executes the query. It's fine in this local benchmark; production keeps plain `EXPLAIN`.
