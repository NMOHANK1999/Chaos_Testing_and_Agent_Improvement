# 10: Persona Simulator Conversation Loop

**Status:** ready-for-agent
**Blocked by:** 02 (a simulator-native TraceAdapter is built alongside this spec), 08 (Persona Generator), 09 (Target Agent Adapter)

## What it builds

The loop that takes a `PersonaSpec` and an `agentAdapter`, drives a multi-turn conversation between them, and emits the resulting transcript — which spec 02's simulator adapter then normalizes into canonical `Event`s like any other source.

## Problem

Running a simulated user against a target agent turn-by-turn, at volume and concurrency, with correct termination/turn-limit handling, is a mechanism this project needs but shouldn't build from first principles if something already fits.

## Open-Source Landscape

- **τ-bench / τ²-bench** (Sierra Research) — a benchmark and simulation framework purpose-built for exactly this shape of problem: a simulated user (backed by an LLM) driving a multi-turn, tool-using conversation against an agent under test, in realistic domains, with strategy-compliance grading. As of 2026, actively maintained (a 1.0.1 release shipped July 2026), open source.
- **LangGraph, CrewAI, AutoGen** — general-purpose multi-agent orchestration frameworks. 2026 benchmarks show meaningfully different tradeoffs (LangGraph: lowest latency/cost, strongest production/durability features; CrewAI: fastest to stand up, higher token overhead; AutoGen: strongest open-ended reasoning, highest cost). None of the three is purpose-built for "simulated user vs. agent under test" specifically — they're built for constructing multi-agent systems generally, and a user-simulator loop would be one more thing to build on top of any of them.
- A minimal custom loop (a few dozen lines: send, receive, check termination, repeat) — always available as a fallback.

## Build vs Buy Decision

**Prefer τ-bench's simulation harness over general-purpose orchestration frameworks**, since it's the closest purpose-built fit (a simulated user driving turn-based conversation against an agent under test) rather than repurposing a tool designed for a different job. The real open question is whether τ-bench's domain/tool-calling assumptions (built around task-oriented, API-tool-equipped agents in specific verticals) are flexible enough to swap in this project's own `PersonaGenerator` and `agentAdapter`, or whether that rigidity makes a general framework — or a plain custom loop — cheaper in practice. That's the experiment.

## Experiment Design

**Question:** which option requires the least integration effort to bolt on this project's own persona generation and target-agent adapter, while still supporting the concurrency this project needs for scale/chaos testing?

- **Candidates:** τ-bench/τ²-bench (adapted), LangGraph (a custom user-simulator node graph), a minimal custom loop (as a lower-bound baseline).
- **Method:** prototype the identical single scenario — one persona spec, one scripted fake target agent (from spec 09's test fixtures) — in each candidate.
- **Metrics:**
  - *Integration effort* — lines of glue code and time needed to wire in `PersonaSpec` generation and the `agentAdapter` interface.
  - *Concurrency support* — can it run N simulated conversations in parallel without custom scaffolding.
  - *Termination/turn-limit correctness* — does it handle conversation-end conditions (task complete, turn limit, agent stuck) out of the box or does that need custom logic regardless of candidate.
  - *Maintenance signal* — release cadence, license, community activity.
- **Decision rule:** lowest integration effort with adequate concurrency support wins. τ-bench is the leading hypothesis on fit; if its task/API-tool domain assumptions prove too rigid to repurpose cheaply, fall back to the plain custom loop over the general orchestration frameworks — a custom loop this project fully controls is a smaller liability than bending a framework built for a different purpose (multi-agent system construction, not user simulation) to this task.

## Test Datasets & Reference Implementations

- **τ-bench / τ²-bench** itself ships complete retail and airline domain environments, task sets, and a working user-simulator-vs-agent loop — the single richest open reference implementation available for this spec, usable both as the leading candidate under evaluation and, if adopted, as an initial scenario library so v0 doesn't need to author scenarios from scratch.
- **LMSYS-Chat-1M** / **WildChat** opening turns (the first user message of real conversations) are a ready source of realistic conversation-starter prompts to seed simulated scenarios beyond τ-bench's built-in task sets.

## Interface & Implementation Decisions

- Interface: `simulate(persona_spec, agent_adapter, scenario) -> native_transcript`, matching the parent spec's `PersonaSimulator` module signature exactly.
- Whichever candidate wins, it sits *behind* this interface — the rest of the codebase never calls τ-bench (or LangGraph, or the custom loop) directly, only this module's interface, so a future re-benchmark can swap the underlying mechanism without touching callers.
- Every simulated conversation's output is tagged with the persona spec and scenario that produced it (per the parent spec's user stories), so a downstream error mode can be traced back to its origin.
- The output transcript is in whatever native shape the chosen mechanism produces; spec 02's simulator `TraceAdapter` (built alongside this spec) is responsible for normalizing it — this spec does not emit canonical `Event`s directly.

## Testing Decisions

- Test the loop against a fake `agentAdapter` (in-memory scripted responses, from spec 09's test fixtures), never a live target agent — fast, deterministic, no API cost in the test suite.
- Test termination handling explicitly: a scripted agent that never stops, one that finishes early, and one that errors mid-conversation — asserting the loop terminates and tags the transcript correctly in each case.
- Concurrency behavior (the experiment's metric) is validated once via a load-test script, not on every CI run.

## Out of Scope

- Voice/multimodal simulation (τ-bench's 2026 roadmap includes this; not needed here).
- Adapting τ-bench's own strategy-compliance grading — this project's grading is the Trace Analytics pipeline (specs 04–07), not τ-bench's built-in scorer.
