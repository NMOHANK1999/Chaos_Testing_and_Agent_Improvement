# Chaos Testing and Agent Improvement

Two connected systems for understanding and stress-testing an agentic product:

1. **Trace Analytics** — ingests agent conversation traces from any source, normalizes them through pluggable per-source adapters into a canonical event stream, and produces a report covering usage modes (what people are actually asking for), cost analytics (split by model, harness, and version), error modes (where the agent struggles or fails, including implicit signals a user never files as a bug), and roadmap-facing insights.
2. **Persona-Based Usage Simulator** — generates a wide variety of personas grounded in the target agent's actual use cases, drives simulated conversations against the agent, and feeds the resulting traces through the same normalization path as real traffic, for chaos/stress-testing personality diversity and scale ahead of or alongside real usage.

Both systems read and write the same canonical trace representation, so the analytics pipeline works identically over live production traces and simulator-generated ones.

## Status

Design stage. No implementation yet. The parent design spec and the component-level engineering specs below are the current deliverables.

## Docs

- **Parent design spec**: [`.scratch/trace-analytics-persona-simulator/spec.md`](.scratch/trace-analytics-persona-simulator/spec.md) — problem statement, solution shape, user stories, and the architectural seams (canonical event model, `TraceAdapter`, `TraceAnalyzer`, `PersonaSimulator`).
- **Component engineering specs**: [`engineering-specs/`](engineering-specs/) — one spec per buildable component, each with the open-source landscape considered, a build-vs-buy call, and (where more than one viable option exists) a experiment design for choosing between them.
- **Agent skills config**: see `CLAUDE.md` and `docs/agents/*.md` for how specs/issues are tracked in this repo.
