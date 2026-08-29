# 07: Insight Synthesizer & Analysis Report

**Status:** ready-for-agent
**Blocked by:** 04 (Usage Mode Classifier), 05 (Cost Calculator), 06 (Error Mode Detector)

## What it builds

The component that assembles usage modes, cost breakdowns, and error modes into the final `AnalysisReport`, including narrative insights that cross-reference the three (e.g. "personas/usage modes doing X consistently hit error mode Y").

## Problem

The interesting output isn't the three raw sections individually, it's what they imply together — and that synthesis step is inherently a generative/narrative task, which means it needs an LLM call whose output has to reliably conform to the report's schema (structured sections a downstream consumer, or spec 10's feedback loop, can query), not free text that occasionally breaks parsing.

## Open-Source Landscape

Structured-output libraries solve exactly "get an LLM to reliably produce schema-conforming output" so this project doesn't have to hand-roll JSON-parsing-with-retries:

- **Instructor** — wraps any LLM client with Pydantic validation and automatic retry on schema mismatch; the most widely adopted option (11K+ GitHub stars, 3M+ monthly downloads as of 2026), works across providers, minimal learning curve.
- **PydanticAI** — from the Pydantic team, adds type-safe tool calling/dependency injection on top of structured output; aimed at full agent-building, more than this spec needs.
- **Outlines** — enforces schema at the token level via constrained decoding (an FSM masks invalid tokens during generation), rather than validating after the fact. Requires logit-level access to the model, which generally means a self-hosted open-weight model — not applicable when the underlying LLM is only reachable via a hosted API (the expected case here), which rules it out a priori rather than needing to be benchmarked.

## Build vs Buy Decision

**Adopt a structured-output library for the report-generation call; don't hand-roll JSON parsing/retry logic.** Outlines is excluded up front on a hard constraint (no logit access to hosted-API models), narrowing the real comparison to Instructor vs PydanticAI.

## Experiment Design

**Question:** does Instructor or PydanticAI give a better schema-adherence rate and lower retry/latency overhead for this project's specific report schema (nested sections, lists of usage modes/error modes)?

- **Candidates:** Instructor, PydanticAI.
- **Method:** run N generations (e.g. 100) of the report schema through each library against the same underlying model and input data, holding the prompt constant.
- **Metrics:** schema-validation pass rate (first attempt and eventual, after library-internal retries), number of retries needed, added latency versus a raw ungoverned call.
- **Decision rule:** pick whichever clears 100% eventual validity with fewer retries/less latency overhead; given Instructor's stated maturity and "safest default" positioning as of 2026, it's the leading hypothesis, but PydanticAI is a legitimate second candidate if this project later needs tool-calling in the same call (it currently doesn't).

## Interface & Implementation Decisions

- Interface: `synthesize(usage_modes, cost_breakdown, error_modes) -> AnalysisReport`.
- `AnalysisReport` is queryable by any of its dimensions (usage mode, model, harness, version, error mode) per the parent spec's user stories — implemented as a structured object (via whichever library the experiment picks), not a rendered document, so a consumer (or spec 10's optional feedback loop) can read fields directly rather than re-parsing prose.
- The narrative "insights" section is the one genuinely generative part; it's produced by a single LLM call given the three structured inputs, schema-validated on the way out.

## Testing Decisions

- Test the aggregation logic (assembling the three inputs into the report shape) with mocked inputs — this part is pure data transformation and needs no LLM call in tests.
- Test the narrative-synthesis call's schema conformance against fixture inputs, using the same structured-output library chosen above; assert valid output shape, not the semantic quality of the narrative (that's a qualitative review, not a unit test).

## Out of Scope

- A rendered/visual report (dashboard) — out of scope per the parent spec; this produces structured + narrative report data only.
- Historical report versioning/diffing across runs.
