# 04: Usage Mode Classifier

**Status:** ready-for-agent
**Blocked by:** 01 (Canonical Trace Schema), 03 (Trace Storage & Query Layer)

## What it builds

The component that turns a corpus of traces into emergent usage-mode categories ("what people are actually asking for"), without assuming a fixed taxonomy up front.

## Problem

Usage categories aren't known in advance and shouldn't be hardcoded — a fixed taxonomy hides whatever the taxonomy's author didn't anticipate. The classifier needs to discover categories from the data and produce human-readable labels for them, at a cost/latency that scales to a growing trace corpus.

## Open-Source Landscape

- **BERTopic** — embedding + dimensionality reduction + HDBSCAN clustering + c-TF-IDF for topic representation; the current standard for unsupervised topic discovery. Benchmarked at roughly 34% better clustering quality than Top2Vec on comparable corpora.
- **Top2Vec** — an earlier embedding-based topic modeling approach; generally outperformed by BERTopic on clustering quality.
- **LLM-only approaches** — recent research prompts an LLM directly to build a topic taxonomy and classify each item against it (no embedding-clustering step at all).
- **Hybrid (BERTopic + LLM labeling)** — active research pattern: run BERTopic for the clustering, then pass each cluster's keywords/representative documents to an LLM to generate a human-readable label — combining unsupervised discovery with LLM-quality labels.

## Build vs Buy Decision

**Adopt BERTopic for clustering; use an LLM only to label the clusters it finds.** This is a published hybrid pattern, not a novel combination invented for this project, and it avoids two failure modes: pure LLM-only classification tends to need a taxonomy pre-built (defeating "emergent"), and pure BERTopic labels (c-TF-IDF keyword lists) are useful for the classifier itself but not human-readable as a report output.

## Experiment Design

**Question:** which combination produces the most coherent, human-recognizable usage-mode categories at acceptable cost/latency?

- **Candidates:**
  (a) BERTopic clustering + LLM labeling (the leading hypothesis above)
  (b) Top2Vec clustering + LLM labeling
  (c) Pure LLM zero-shot classification against an iteratively-built taxonomy
  (d) Plain embedding + HDBSCAN, hand-rolled (no BERTopic packaging) — a lower bound, to check whether BERTopic's extra machinery earns its keep over the raw technique
- **Dataset:** a sample of 500–1,000 traces drawn from whatever corpus exists at the time (real traces once available; simulator-generated traces from spec 10 otherwise — the analyzer must work on either).
- **Metrics:**
  - *Topic coherence* (c_v or NPMI score) — a standard, automatic topic-modeling quality metric.
  - *Human-rated relevance* — 2 raters independently score a sample of discovered usage-mode labels for "does this label meaningfully describe what these traces have in common" (1–5 Likert), report inter-rater agreement.
  - *Cost & latency per 1,000 traces processed.*
- **Decision rule:** highest coherence + human-rated relevance within an acceptable cost/latency budget; ties broken toward whichever option is more open-source/less bespoke (candidate (a) or (b) over (c) or (d)), consistent with this project's default posture.

## Interface & Implementation Decisions

- Interface: `classify(traces: list[NormalizedTrace]) -> UsageModes` where `UsageModes` is a list of `{label, description, member_trace_ids, keyword_terms}`.
- Runs as a batch job over the storage layer's query interface (spec 03), not per-trace inline — usage-mode discovery is a corpus-level operation by nature.
- Re-running the classifier as the corpus grows may reshuffle cluster boundaries; label stability across runs is a known limitation, not solved in v0 (see Out of Scope).

## Testing Decisions

- Test against fixture trace sets with known, engineered clusters (e.g., a fixture corpus hand-built from N obviously-distinct request types) and assert the classifier recovers approximately that many clusters — a coarse sanity check, not a precision benchmark (that's the experiment's job, run once, not on every CI run).
- Test the labeling step independently against fixed cluster keyword/document inputs, asserting it produces a non-empty, on-topic label — mock the clustering step here so labeling tests don't depend on clustering nondeterminism.

## Out of Scope

- Cross-run label stability / cluster tracking over time (a v1 concern).
- Real-time/incremental classification as new traces arrive — v0 re-runs in batch.
