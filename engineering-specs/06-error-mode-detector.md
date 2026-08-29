# 06: Error Mode Detector

**Status:** ready-for-agent
**Blocked by:** 01 (Canonical Trace Schema), 03 (Trace Storage & Query Layer)

## What it builds

The component that labels where the agent struggled or failed within a trace — both structural failures (loops, repeated tool errors, contradictions) and implicit signals a user never files as a bug (correction, frustration, abandonment).

## Problem

Most failures aren't reported; they're just abandoned or worked around by the user. Detecting them requires reading the shape of the conversation, not waiting for an explicit complaint. This splits into two genuinely different sub-problems that shouldn't be solved with the same tool: *structural* trajectory failures (a trace-shape problem) and *implicit user sentiment* (a text-classification problem).

## Open-Source Landscape

**Structural/trajectory failure detection:**
- **Arize Phoenix** — ships LLM-as-judge evaluator templates specifically for AI agent trajectories, hallucination, and relevance, built around OTel/OpenInference traces.
- **DeepEval** — broader agent/chatbot/safety-testing coverage than Phoenix or RAGAS, with its own judge-based metrics.
- **RAGAS** — purpose-built for RAG evaluation specifically; narrower fit here since this project isn't RAG-specific.
- Recent research (e.g. "TRAIL: Trace Reasoning and Agentic Issue Localization") is active in this exact space — trace-based failure localization is not a solved, one-obvious-winner problem yet.

**Implicit user-criticism/frustration detection:**
- Off-the-shelf sentiment/emotion classifiers (widely available, e.g. via Hugging Face) — a much smaller, well-solved NLP problem than agent-trajectory evaluation.
- An LLM-judge prompt purpose-written for "does this user turn imply the agent got something wrong," since no off-the-shelf tool targets this specific category.

## Build vs Buy Decision

**Adopt an existing eval framework's judge templates/rubrics for structural failure detection; use a lightweight off-the-shelf classifier for sentiment, reserving custom LLM-judge work only for the implicit-criticism category that has no existing tool.** Don't hand-write agent-trajectory judge prompts from scratch when Phoenix/DeepEval already have published, tested rubrics for exactly that. Do write a custom judge for implicit criticism, since it's a narrower, project-specific signal ("did the user just implicitly say this went wrong") that general sentiment classifiers only approximate.

## Experiment Design

**Question:** which combination of structural-failure detector and implicit-criticism detector yields the best precision/recall against real failure/criticism instances?

- **Ground truth (unavoidable groundwork, not skippable via "buy"):** hand-label a fixture set of 100–200 traces (drawn from real traces once available and/or simulator-generated traces from spec 10) for: contains-a-failure (yes/no + severity) and contains-implicit-criticism (yes/no). This gold set is itself a deliverable of this spec.
- **Candidates (structural):** Phoenix's agent-trajectory evaluators, DeepEval's agent metrics, a custom LLM-judge prompt (as a comparison baseline against the off-the-shelf options).
- **Candidates (implicit criticism):** an off-the-shelf sentiment/frustration classifier, a custom LLM-judge prompt.
- **Metrics:** precision, recall, F1 against the gold set for each candidate in each category; cost and latency per trace.
- **Decision rule:** highest F1 at acceptable cost/latency, chosen independently per category (the winning structural detector need not come from the same toolkit as the winning criticism detector).

## Interface & Implementation Decisions

- Interface: `detect(trace: NormalizedTrace) -> list[ErrorMode] { category, severity, evidence_span, source: structural | implicit_criticism }`.
- The two detection paths (structural, implicit-criticism) are separate internal adapters behind this one interface — not because the interface exposes them separately, but because the experiment may pick different underlying tools for each, and the seam should make that swap-in-place possible.
- Every detected error mode carries an `evidence_span` pointing back into the trace's events, so a downstream reader (or the Insight Synthesizer) can show *why* something was flagged, not just that it was.

## Testing Decisions

- Test against the gold-labeled fixture set built during the experiment — this doubles as both the experiment's dataset and the regression test suite going forward (a new candidate or prompt change gets checked against the same ground truth).
- Test the two detection paths independently with mocked inputs for the other, so a change to the sentiment classifier can't silently break structural-detection tests and vice versa.

## Out of Scope

- Automatic remediation of a detected error mode — this labels problems, it doesn't fix them (per the parent spec's Out of Scope).
- Real-time detection during a live conversation — v0 runs over completed, stored traces.
