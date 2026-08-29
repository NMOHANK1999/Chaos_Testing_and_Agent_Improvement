# 02: Reference Trace Adapters

**Status:** ready-for-agent
**Blocked by:** 01 (Canonical Trace Schema & TraceAdapter Interface)

## What it builds

At least two concrete `TraceAdapter` implementations, proving the seam from spec 01 actually holds across genuinely different native formats: one for a real instrumentation source, one for the simulator's own transcript format (co-developed with spec 10).

## Problem

A seam with only one adapter is a hypothetical seam — it's unverified whether the interface is actually source-agnostic until a second, different source is plugged in. v0 needs to prove this without hand-instrumenting a bespoke agent from scratch.

## Open-Source Landscape

Rather than hand-writing instrumentation for a target agent, two OSS auto-instrumentation ecosystems already emit structured spans for LLM/agent calls that a `TraceAdapter` can consume as input:

- **OpenLLMetry** (Traceloop) — OTel-based auto-instrumentation for common LLM SDKs (Anthropic, OpenAI, LangChain, etc.), emits spans following (an earlier, more stable variant of) the GenAI conventions.
- **OpenInference** (Arize) — the semantic convention Phoenix is built on; also auto-instruments common LLM/agent SDKs.
- Raw **OTel GenAI semantic conventions** directly (spec 01's vocabulary source) — usable if the target agent already emits OTel spans natively, but still pre-1.0/"Development" stability as of 2026.

## Build vs Buy Decision

**Buy the instrumentation, build only the adapter's mapping logic.** Point one of OpenLLMetry or OpenInference at a real target agent to get spans "for free," and write a `TraceAdapter` that maps that export format into canonical `Event`s. This avoids hand-instrumenting a target agent while still producing a genuine, non-simulator trace source for the seam.

## Experiment Design

**Question:** which instrumentation source (OpenLLMetry vs OpenInference vs raw OTel GenAI conventions) requires the least custom mapping logic while covering the fields the canonical model needs (model, tokens, tool calls, actor/turn boundaries)?

- **Candidates:** OpenLLMetry, OpenInference, raw OTel GenAI conventions (if the target agent supports emitting them natively).
- **Method:** instrument one sample target agent with each candidate, capture its span export for an identical scripted conversation (same script across all three so outputs are comparable), and hand-write the `Event` mapping for each.
- **Metrics:**
  - *Field coverage* — % of required canonical fields (model, harness, tokens in/out, tool call args/results, turn/actor boundaries) present without heuristic guessing.
  - *Mapping effort* — lines of custom mapping code / number of special-cased branches needed.
  - *Maturity signal* — release cadence, stability badge, open issues touching breaking changes.
- **Decision rule:** pick the source with highest field coverage per unit of mapping effort; break ties toward the more stable/mature convention, since this adapter will need to survive upstream schema churn (OTel GenAI is explicitly still shifting as of 2026).

## Test Datasets & Reference Implementations

- **OpenLLMetry** and **OpenInference** both ship example/quickstart repos with sample instrumented applications (LangChain, raw Anthropic/OpenAI SDK calls) — running these directly produces real native-format captures to build and test an adapter against, without needing a bespoke target agent built first.
- **Arize Phoenix**'s own open-source test suite and example trace fixtures are a reference implementation of consuming OpenInference-format spans; the OpenInference-based adapter's output can be checked for parity against how Phoenix itself interprets the same spans.
- **LMSYS-Chat-1M** / **WildChat** conversations, replayed through a thin OpenLLMetry- or OpenInference-instrumented wrapper, produce large volumes of realistic native-format exports at replay cost only — useful for adapter load and edge-case testing beyond a small hand-written fixture set.

## Interface & Implementation Decisions

- Each adapter is a standalone module satisfying `normalize(raw_export) -> list[Event]` (spec 01); no shared logic between adapters beyond that interface.
- The simulator adapter (built with spec 10) normalizes the simulator's own transcript format the same way — it is not special-cased as "the synthetic one," proving real and synthetic traces flow through an identical seam.
- Unmapped/unrecognized fields in a native export are preserved into `attributes` under a source-prefixed key rather than silently dropped, so nothing is lost even if the analyzer never reads it.

## Testing Decisions

- Each adapter is tested against fixture exports of its own native format (captured during the experiment above), asserting correct normalization into canonical `Event`s. This is the only place native formats appear in any test in this repo.
- A shared "adapter conformance" test suite (parameterized over every registered adapter) asserts each one produces valid `Event`s per spec 01's schema tests — catching an adapter that technically compiles but emits malformed events.

## Out of Scope

- Building an adapter for every possible harness — just enough to prove the seam (two, per above).
- Live/streaming ingestion from these instrumentation sources — v0 works from exported batches.
