Status: ready-for-agent

# Trace Analytics & Persona-Based Usage Simulator

## Problem Statement

Whoever operates an agentic system (a user↔agent or agent↔agent product) currently has no structured way to see:

- what people are actually asking the agent to do (usage modes),
- what serving those requests costs, broken down by model, harness, and version,
- where the agent struggles or fails, including failures a user never files as a bug but signals implicitly (frustration, correction, giving up), and
- what any of that implies for the product roadmap.

Separately, there's no way to generate a realistic volume and variety of usage data on demand. Today the only usage data comes from whatever real traffic happens to occur, so personality diversity, edge cases, and scale characteristics of the agent go untested until they show up in production.

Both problems compound each other: without synthetic traffic, there's not enough trace volume to validate the analytics pipeline pre-launch; without the analytics pipeline, synthetic traffic has no way to prove it's realistic or to close the loop back into what personas should be simulated next.

## Solution

One canonical way to represent a trace, normalized from any source, feeding two systems built on top of it:

1. **Trace Analytics Pipeline** — ingests traces from any agent/harness (production or synthetic), normalizes them, and produces a report covering usage modes, cost analytics, error modes, and open-ended insights.
2. **Persona-Based Usage Simulator** — generates a wide variety of personas grounded in the target agent's actual use cases (not generic archetypes), drives simulated conversations against the agent, and produces traces through the same normalization path as real traffic — for chaos/stress-testing personality diversity and scale ahead of or alongside real usage.

The two systems never assume a fixed trace format. Every trace source (a given harness, an API integration, or the simulator itself) is normalized through its own adapter into a small canonical event stream; the analytics pipeline only ever reads that canonical stream, never a source's native format directly. This is what lets the same pipeline analyze both live production traces and simulator-generated ones, and lets a new agent/harness be onboarded by writing one adapter rather than touching the analyzer.

## User Stories

**Trace Analytics**

1. As a product manager, I want to see what categories of requests users actually send the agent, so that I can prioritize the roadmap around real usage instead of guesses.
2. As a product manager, I want usage-mode categories to emerge from the data rather than be predefined, so that unexpected use cases aren't hidden by a fixed taxonomy.
3. As an engineering lead, I want cost broken down by model, so that I can see which models are driving spend.
4. As an engineering lead, I want cost broken down by harness, so that I can compare the cost of serving the same agent through different integration surfaces.
5. As an engineering lead, I want cost broken down by version, so that I can see whether a new release changed cost per interaction.
6. As an engineering lead, I want cost broken down by any combination of model, harness, and version, so that I can isolate which variable is actually responsible for a cost change.
7. As an engineering lead, I want traces missing token/cost data to be reported as "unknown" rather than dropped or crashing the pipeline, so that partial data from an incomplete adapter doesn't silently corrupt cost totals.
8. As an agent designer, I want to see error modes — places where the agent struggled, looped, or failed a task — so that I know what to fix first.
9. As an agent designer, I want error modes sourced from implicit signals (the user correcting the agent, expressing frustration, abandoning the conversation), not just explicit bug reports, so that silent failures surface too.
10. As an agent designer, I want error modes sourced from trace-pattern analysis (repeated tool failures, backtracking, contradicting earlier turns), so that structural failure patterns are caught even without explicit user criticism.
11. As a product manager, I want a synthesized "insights" section that goes beyond raw counts (e.g. "personas doing X consistently hit error mode Y"), so that the report is directly actionable for roadmap decisions.
12. As a data scientist, I want to query the analysis report by any of its dimensions (usage mode, model, harness, version, error mode), so that I can slice the data for my own investigation without re-running the pipeline.
13. As a platform engineer, I want to add a new trace source by writing one adapter, so that onboarding a new harness doesn't require changes to the analyzer itself.
14. As a platform engineer, I want the analyzer to run against a fixture of canonical events with no dependency on any real trace source, so that analyzer correctness can be verified independent of adapters.
15. As an agent designer, I want traces from agent-to-agent conversations (not just user-to-agent) supported by the same canonical model, so that multi-agent systems get the same analytics without a separate pipeline.
16. As an engineering lead, I want the pricing table used for cost calculation to be edited without touching analyzer code, so that price changes don't require a code change and review of analysis logic.

**Persona Simulator**

17. As a product manager, I want to simulate technical personas (engineers, power users), so that I can stress-test the agent against sophisticated, high-context usage.
18. As a product manager, I want to simulate non-technical personas, so that I can validate the agent handles low-context, imprecise requests gracefully.
19. As a product manager, I want to simulate role-specific personas (PM, engineer, support agent, etc.), so that the simulated traffic matches the actual population of people who use the agent.
20. As an agent designer, I want personas grounded in the target agent's actual use cases rather than generic persona templates, so that simulated conversations produce realistic, useful traces instead of noise.
21. As an agent designer, I want persona generation to start as a single grounding prompt (v0), so that the system is useful immediately without upfront investment in retrieval or determinism engineering.
22. As an agent designer, I want persona generation to later support a more research-grounded or deterministic strategy without changing the simulation loop, so that improving persona quality doesn't require rewriting the simulator.
23. As an engineering lead, I want the simulator to drive an arbitrary target agent through a pluggable adapter, so that the same simulator can chaos-test multiple agents/harnesses, not just one hardcoded integration.
24. As an engineering lead, I want simulated conversations to run at volume and concurrency, so that I can find scalability issues (latency degradation, rate-limit failures, state bugs under concurrent sessions) before real users do.
25. As an agent designer, I want the simulator's output traces normalized through the same adapter mechanism as real traces, so that simulated and real traffic can be analyzed identically and compared directly.
26. As a product manager, I want the option (not the default) to have persona generation grounded in usage modes discovered by the analytics pipeline, so that synthetic traffic can evolve to mirror real usage as it's discovered — while keeping v0 simulation possible before any real traces exist.
27. As an agent designer, I want each simulated conversation tagged with which persona and scenario produced it, so that when analytics finds an error mode, I can trace it back to the specific persona/scenario that caused it.

## Implementation Decisions

**Canonical trace model**

- A normalized trace is an ordered list of `Event`s. Each `Event` has a minimal required envelope: `sequence`, `actor` (`user` / `agent` / `tool`), `timestamp`, `kind` (`message` / `tool_call` / `tool_result` / `handoff` / `error`).
- Everything else — token counts, model name, tool arguments, raw source payload — lives in an open `attributes` bag on the event or trace. Nothing downstream may assume a given `attributes` key is present.
- Trace-level segmentation fields (`model`, `harness`, `version`) are best-effort values an adapter populates into `attributes` when its source exposes them. They are not guaranteed top-level schema fields. Analysis resolves them per-adapter at query time, defaulting to `unknown` when absent.

**`TraceAdapter` (the real seam)**

- One adapter per trace source (a harness, an API integration, or the simulator itself). Each adapter's sole job: translate that source's native export format into the canonical `Event` stream.
- `TraceAnalyzer` never touches a native format directly — only the canonical stream. Adding a new agent/harness later means writing one new adapter, not modifying the analyzer.
- v0 ships with at least two adapters (e.g. one real-source adapter and the simulator's own adapter) specifically to prove the seam holds across genuinely different native formats, per the "one adapter is hypothetical, two is real" rule.

**`TraceAnalyzer` (deep module)**

- Interface: `analyze(traces: NormalizedTrace[]) -> AnalysisReport { usageModes, costBreakdown, errorModes, insights }`.
- Internally composed of (all private to the module, not part of its interface):
  - **Usage Mode Classifier** — LLM-based clustering over message content to surface emergent categories of request, not a fixed taxonomy.
  - **Cost Calculator** — looks up a separately maintained, versioned pricing table by model; computes cost breakdowns sliceable by model, harness, and version; reports missing token/cost data as `unknown` rather than dropping the trace or failing.
  - **Error Mode Detector** — combines implicit-criticism heuristics (correction language, frustration signals, abandonment) with trace-structure heuristics (repeated tool failures, backtracking, self-contradiction) and an LLM-judge pass to label struggle/failure points.
  - **Insight Synthesizer** — aggregates across the above to produce roadmap-facing narrative insights (e.g. cross-referencing usage mode against error mode), not just raw counts.
- The pricing table is a separate, versioned config artifact, not hardcoded into the analyzer, so price changes don't require touching or re-reviewing analysis logic.

**`PersonaSimulator` (deep module)**

- Interface: `simulate(personaSpec, agentAdapter, scenario) -> native transcript`, which is then normalized via the simulator's own `TraceAdapter` like any other source.
- `agentAdapter` is a pluggable interface for driving the actual target agent (its API/CLI), decoupled from persona logic, so the same simulator can target multiple agents/harnesses.
- Internal seam: `PersonaGenerator`.
  - v0 adapter: a single grounding prompt, fed a description of the target agent's actual use cases, that produces a persona spec (role, technical level, goals, communication style).
  - v1 adapter (future): retrieval- or determinism-based generation, optionally grounded in the `usageModes` output of `TraceAnalyzer` — closing the loop so synthetic personas evolve to mirror discovered real usage. This coupling is opt-in, not required for v0 to function, and v0 must work standalone with no real traces available yet.
- Each simulated conversation is tagged with the persona spec and scenario that produced it, so an error mode found downstream can be traced back to its origin.

## Testing Decisions

- Good tests here exercise external behavior at the seam, not internals: given inputs at the interface, assert on the interface's output, not on which private helper ran.
- `TraceAnalyzer` is tested purely against canonical `Event` stream fixtures — never against a specific source's native format. Fixtures must cover: complete data, missing cost/token attributes, missing model/harness/version attributes, and multi-turn traces containing tool calls, errors, and handoffs.
- Each `TraceAdapter` is tested against fixture exports of its own native format, asserting correct normalization into canonical `Event`s. This is the only place native formats appear in tests.
- `PersonaGenerator` adapters are tested against the shape of the persona spec they produce (plausibly tied to the declared use-case description), not against a full simulation run.
- `PersonaSimulator`'s loop is tested against a fake `agentAdapter` (in-memory scripted responses) so tests are fast and deterministic, never against a live target agent.
- No prior test conventions exist in this repo yet (greenfield); these conventions are the baseline for specs that follow.

## Out of Scope

- A dashboard/UI for viewing analysis reports — this spec produces structured report output only.
- Real-time/streaming trace analysis — v0 is batch.
- Automatic remediation of detected error modes — this surfaces insights, it doesn't act on them.
- Building out every harness's `TraceAdapter` — v0 builds only what's needed to prove the seam (at least two adapters, one being the simulator's).
- Deeper agent-to-agent-specific analysis beyond what the canonical model already accommodates via `kind: handoff` — treated as future work.
- `PersonaGenerator` v1 (retrieval/determinism engineering) — v0 is prompt-only.

## Further Notes

- The usage-modes → persona-grounding feedback loop was discussed and confirmed as a deliberate future direction, not a v0 requirement: v0 persona generation must stand alone via prompt grounding before any real traces exist to feed back in.
- No existing trace store/export mechanism has been identified yet; the first real `TraceAdapter` to build should be chosen once the integration target (which harness/agent to instrument first) is picked.
- Issue tracker is local markdown for now (`docs/agents/issue-tracker.md`); this spec can be promoted to a GitHub issue later once ready to break it into tracked tickets.
