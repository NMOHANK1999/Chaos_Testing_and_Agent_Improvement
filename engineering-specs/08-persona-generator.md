# 08: Persona Generator

**Status:** ready-for-agent
**Blocked by:** 01 (Canonical Trace Schema — for the eventual usage-modes feedback loop's shape; the v0 generator itself has no hard dependency beyond an LLM client)

## What it builds

The component that produces persona specs (role, technical level, goals, communication style) for the simulator to drive — grounded in the target agent's actual use cases, not generic archetypes, and diverse enough that simulated traffic isn't a handful of near-duplicate personas repeated at volume.

## Problem

A single "generate a persona" prompt run repeatedly risks mode collapse — an LLM asked for variety many times in a row tends to converge on a handful of stereotypical answers. The personas also need to be *grounded*: tied to what the target agent is actually for, not generic (a "technical persona" and "non-technical persona" that could belong to any product isn't useful for stress-testing this one).

## Open-Source Landscape

- **PersonaHub** (Tencent AI Lab) — an open dataset of ~1 billion diverse, attribute-sparse personas synthesized from web data, built specifically to inject diversity into synthetic-data generation at scale. Diverse but generic — not grounded in any particular product's use cases.
- Recent research (SCOPE, KoPersona, CulturalPersonas, PolyPersona, DeepPersona) is actively exploring *grounded*/deep persona construction — conditioning generation on richer structure (sociopsychological protocols, cultural value frameworks, detailed profiles) rather than shallow attribute lists — and finds more detailed, realistic persona profiles improve simulation fidelity ("scaling law" result).
- A single unconditioned grounding prompt (the v0 baseline already agreed in the parent spec) — the simplest possible approach, with no external diversity source.

## Build vs Buy Decision

**Adopt PersonaHub as a diversity seed, then condition/graft it onto this project's use-case grounding — don't rely on the LLM to invent diverse base personas from nothing on every call.** This directly answers the original design goal ("personas made up with the use case in mind, not just mindlessly being a persona"): PersonaHub supplies the diversity (background, demographic, communication style), the grounding step supplies the relevance (what this specific persona would actually want from this specific agent).

## Experiment Design

**Question:** does seeding from PersonaHub before grounding actually produce more diverse *and* more relevant personas than grounding alone, and does grounding actually matter (versus PersonaHub diversity by itself)?

- **Candidates:**
  (a) Pure LLM grounding prompt only (v0 baseline, no external diversity source)
  (b) PersonaHub-seeded + grounding prompt (the hybrid hypothesis)
  (c) PersonaHub-seeded, no agent-specific grounding (negative control — diverse but not tied to the target agent's use cases, to prove grounding matters rather than assuming it)
- **Method:** generate ~100 persona specs per candidate for the same target agent description.
- **Metrics:**
  - *Diversity* — pairwise embedding cosine distance across generated persona specs within a candidate (higher = more diverse); mode-collapse rate = % of near-duplicate pairs below a distance threshold.
  - *Grounding fidelity* — an LLM-judge (spot-checked against a handful of human ratings for calibration) scores, per persona, "do this persona's stated goals plausibly map to the target agent's real use cases" (1–5).
- **Decision rule:** the winning candidate needs both acceptable diversity and high fidelity — a candidate that wins one axis while failing the other is disqualified. Expect (a) to be diverse-limited (mode collapse), (c) to score high diversity but low fidelity (proving the negative-control point), and (b) to win both; if the experiment doesn't confirm that, don't ship (b) anyway on the strength of the hypothesis alone.
- Persona spec output is schema-validated using the structured-output library chosen in spec 07 (Instructor or PydanticAI) — reused here rather than re-decided, since it's the same underlying problem (reliable structured LLM output).

## Test Datasets & Reference Implementations

- **PersonaHub** (Tencent AI Lab, Hugging Face-hosted, ~1B personas) — already the chosen diversity seed (candidate (b)/(c) above); directly usable as the sampling pool for the experiment itself, no separate acquisition step needed.
- **LMSYS-Chat-1M** / **WildChat** — real user conversations double as an indirect realism check: once spec 04's classifier can categorize both real and simulator-generated traffic, compare the request-type distribution of generated personas' simulated conversations against the real distribution in these corpora as a signal beyond the LLM-judge fidelity score.
- The **SCOPE**, **KoPersona**, **PolyPersona**, and **DeepPersona** papers (2025-2026) each publish grounded-persona-construction code — worth reviewing when designing the grounding-prompt step, even though PersonaHub remains the diversity-seed dataset of choice for v0.

## Interface & Implementation Decisions

- Interface: `generate(target_agent_description, count) -> list[PersonaSpec] { role, technical_level, goals, communication_style, seed_source }`.
- `PersonaGenerator` has its own internal seam separating the diversity source (PersonaHub sampling, or whatever the experiment picks) from the grounding step (the LLM call conditioning it on the target agent) — two adapters against one interface, so a v1 swap (retrieval-based or deterministic generation, per the parent spec's future direction) replaces the diversity-source adapter without touching the grounding step or the simulator that consumes `PersonaSpec`s.
- The optional future coupling to `TraceAnalyzer`'s `usageModes` output (grounding personas in real discovered usage) is a second, alternate diversity-source adapter to add later — not required for v0, which must work standalone with no real traces yet available.

## Testing Decisions

- Test the grounding step against a fixed set of target-agent descriptions and fixed (mocked) diversity-source samples, asserting the output is schema-valid and non-empty — determinism of LLM output itself isn't asserted, only shape.
- Diversity/fidelity quality (the experiment's actual metrics) are evaluated once via the experiment, then spot-checked periodically as a qualitative review, not re-run as a per-commit test — it's a statistical property of a generation process, not a pass/fail unit behavior.

## Out of Scope

- v1 retrieval/determinism-based generation (per parent spec's Out of Scope) — v0 ships prompt/seed-based generation only.
- The usage-modes feedback loop itself — noted as a future adapter slot, not built here.
