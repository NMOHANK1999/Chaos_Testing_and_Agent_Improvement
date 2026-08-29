# Engineering Specs

Component-level breakdown of [`.scratch/trace-analytics-persona-simulator/spec.md`](../.scratch/trace-analytics-persona-simulator/spec.md). Each file below is one buildable, independently testable component: what problem it solves, what open-source options were considered, the build-vs-buy call, and — wherever more than one viable candidate exists — an experiment design for choosing between them empirically instead of by opinion.

Every spec defaults to adopting an existing open-source implementation over writing custom code, reserving custom work for the glue/interface layer (the seams) that ties adopted components together.

## Reading order (dependency graph)

| # | Spec | Blocked by |
|---|------|-----------|
| 01 | [Canonical Trace Schema & TraceAdapter Interface](01-canonical-trace-schema-and-adapter-interface.md) | None |
| 02 | [Reference Trace Adapters](02-reference-trace-adapters.md) | 01 |
| 03 | [Trace Storage & Query Layer](03-trace-storage-and-query-layer.md) | 01 |
| 04 | [Usage Mode Classifier](04-usage-mode-classifier.md) | 01, 03 |
| 05 | [Cost Calculator & Pricing Table](05-cost-calculator-and-pricing.md) | 01, 03 |
| 06 | [Error Mode Detector](06-error-mode-detector.md) | 01, 03 |
| 07 | [Insight Synthesizer & Analysis Report](07-insight-synthesizer-and-report.md) | 04, 05, 06 |
| 08 | [Persona Generator](08-persona-generator.md) | 01 |
| 09 | [Target Agent Adapter](09-target-agent-adapter.md) | 01 |
| 10 | [Persona Simulator Conversation Loop](10-persona-simulator-loop.md) | 02, 08, 09 |

Specs 01–07 form the Trace Analytics pipeline; 08–10 form the Persona Simulator. 02 (a simulator-native `TraceAdapter`) is shared: it's written alongside 10 and consumed by both halves.

The frontier (can start immediately, no blockers): **01**.

**This table shows blocking edges, not build priority — don't read it as "01-07 then 08-10."** The actual operating pattern is sequential in the other direction: the Persona Simulator (08-10) generates the trace corpus; Trace Analytics (04-07) analyzes it. Analytics has nothing production-shaped to run against until the simulator produces it, so 08-10 should be built ahead of 04-06 even though the table shows them as parallel-eligible. See [`../SYSTEM_DESIGN.md`](../SYSTEM_DESIGN.md) §13 for the actual recommended rollout order.
