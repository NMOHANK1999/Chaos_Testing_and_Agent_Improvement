# 03: Trace Storage & Query Layer

**Status:** ready-for-agent
**Blocked by:** 01 (Canonical Trace Schema & TraceAdapter Interface)

## What it builds

Where normalized `Event`s land after an adapter produces them, and how the analytics components (04–07) query them back out sliced by any dimension (usage mode, model, harness, version, error mode).

## Problem

The analyzer needs to run aggregate queries (group-by model × harness × version, join usage-mode labels against cost, filter to traces with a given error mode) over a growing corpus of both real and simulator-generated traces. Storage needs to be cheap to run, fast enough for these queries, and not force the rest of the system to bend to a platform's opinions about ingestion or schema.

## Open-Source Landscape

Full LLM-observability platforms with built-in storage exist and are mature in 2026 — **Langfuse** (OSS, self-hostable, traces + evals + datasets in one product) and **Arize Phoenix** (OSS, self-hostable, tracing + eval without feature gates, OTel/OpenInference-native) are the leading options. Both would give ingestion, storage, and a UI in one package.

Underneath any of that, general-purpose embedded/analytical engines — **DuckDB** (embedded OLAP, reads/writes Parquet natively, zero server to run), **SQLite**, and **Postgres** — are the standard lightweight options for exactly this kind of local analytical query workload.

## Build vs Buy Decision

**Adopt an embedded query engine, not a full observability platform.** Langfuse/Phoenix bring their own ingestion model and UI, which would mean bending the adapter-per-source architecture (spec 01/02) to fit their schema, and duplicate a dashboard this project has explicitly deferred (see parent spec's Out of Scope). Adopting DuckDB or a similar embedded engine gets the "don't hand-roll a query engine" win from open-source without importing a platform's opinions about what a trace looks like.

## Experiment Design

**Question:** which storage engine gives the best query latency and lowest operational overhead for the canonical `AnalysisReport` queries, at the volumes this project actually expects?

- **Candidates:** DuckDB, SQLite (+ pandas for aggregation), Postgres, Arize Phoenix self-hosted (as a stretch candidate, to have one "buy the platform" data point for comparison).
- **Method:** generate synthetic canonical `Event` fixtures at three volumes (1K, 100K, 1M events), load into each candidate, run the representative query set (group-by model/harness/version; join usage-mode label; filter by error-mode flag), record p50/p95 latency per query per volume.
- **Metrics:** ingest throughput (events/sec), query p50/p95 latency at each volume, infra/ops overhead (does it need a running server, migrations, credentials), ease of ad hoc SQL for a data scientist not familiar with this codebase.
- **Decision rule:** pick the lowest-overhead candidate that clears a latency bar (e.g., all representative queries under 1s at the 1M-event volume). Expect DuckDB to win on overhead given it needs no server process and reads/writes Parquet directly, but the experiment should actually run rather than assume this.

## Interface & Implementation Decisions

- Storage sits behind its own small interface — `write(events: list[Event])` / `query(spec) -> rows` — so the chosen engine (whatever the experiment picks) can be swapped without touching spec 04–07's code, which only ever calls this interface.
- Schema in storage mirrors the canonical `Event` model directly: envelope fields as columns, `attributes` as a semi-structured column (e.g. JSON/struct type) so new attribute keys never require a migration.

## Testing Decisions

- Test the storage interface against fixtures of canonical `Event`s (from spec 01), asserting round-trip fidelity (write then query returns equivalent events) and correctness of the representative aggregate queries — never against a specific adapter's native format.
- Run the experiment's benchmark script as a repeatable, committed script (not a one-off), so a future volume increase or engine swap can be re-benchmarked rather than re-litigated from scratch.

## Out of Scope

- A UI or dashboard over this storage layer.
- Real-time/streaming ingestion — v0 is batch writes from adapters.
- Multi-tenant access control — single-operator use for now.
