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

- **Ground truth:** for the structural-failure category, adopt the **TRAIL** dataset (Patronus AI; see Test Datasets below) instead of hand-labeling from scratch — 148 real agent traces with 841 annotated errors across a reasoning/execution/planning taxonomy is a substantially larger and more rigorously labeled gold set than this project could produce from scratch as a side effect of one spec. Implicit-criticism has no equivalent open gold set (see below), so a smaller hand-labeled sample is still needed for that half specifically — drawn primarily from the simulator's own output (spec 10), supplemented by LMSYS-Chat-1M/WildChat real user turns for additional variety once that's useful.
- **Candidates (structural):** Phoenix's agent-trajectory evaluators, DeepEval's agent metrics, a custom LLM-judge prompt (as a comparison baseline against the off-the-shelf options).
- **Candidates (implicit criticism):** an off-the-shelf sentiment/frustration classifier, a custom LLM-judge prompt.
- **Metrics:** precision, recall, F1 against the gold set for each candidate in each category; cost and latency per trace.
- **Decision rule:** highest F1 at acceptable cost/latency, chosen independently per category (the winning structural detector need not come from the same toolkit as the winning criticism detector).

## Test Datasets & Reference Implementations

- **TRAIL** (Trace Reasoning and Agentic Issue Localization; Hugging Face `PatronusAI/TRAIL`, GitHub `patronus-ai/trail-benchmark`) — 148 real agent execution traces built from GAIA and SWE-Bench tasks, with 841 hand-annotated errors spanning reasoning errors (hallucinations), system execution errors (API issues), and planning/coordination errors, across single- and multi-agent systems. This is the primary gold-labeled dataset for the structural-failure half of the experiment above: normalize TRAIL's traces through a `TraceAdapter` (spec 02) and use its existing annotations directly rather than re-labeling. Note: published results report even strong models scoring as low as ~11% accuracy on TRAIL's localization task, so treat it as a genuinely hard benchmark, not a rubber stamp.
- Patronus AI's own `trail-benchmark` GitHub repo is a reference implementation of a trace-failure-localization scorer, worth reading before writing this project's own judge prompts for the structural category.
- No equivalent open, labeled dataset was found for "implicit user criticism/frustration" specifically. The practical path is a small hand-labeled sample drawn primarily from the simulator's own output (spec 10) — persona scenarios can be deliberately scripted to provoke correction/frustration, which is easier to label than mining for it — supplemented by **LMSYS-Chat-1M** and **WildChat** real user turns (which contain visible corrections and frustration, but no failure labels) for additional real-world variety.

## Interface & Implementation Decisions

- Interface: `detect(trace: NormalizedTrace) -> list[ErrorMode] { category, severity, evidence_span, source: structural | implicit_criticism }`.
- The two detection paths (structural, implicit-criticism) are separate internal adapters behind this one interface — not because the interface exposes them separately, but because the experiment may pick different underlying tools for each, and the seam should make that swap-in-place possible.
- Every detected error mode carries an `evidence_span` pointing back into the trace's events, so a downstream reader (or the Insight Synthesizer) can show *why* something was flagged, not just that it was.

## Testing Decisions

- Test the structural detector against TRAIL's labeled traces, and the criticism detector against the hand-labeled LMSYS-Chat-1M/WildChat sample — both double as the experiment's dataset and the regression test suite going forward (a new candidate or prompt change gets checked against the same ground truth).
- Test the two detection paths independently with mocked inputs for the other, so a change to the sentiment classifier can't silently break structural-detection tests and vice versa.

## Out of Scope

- Automatic remediation of a detected error mode — this labels problems, it doesn't fix them (per the parent spec's Out of Scope).
- Real-time detection during a live conversation — v0 runs over completed, stored traces.
