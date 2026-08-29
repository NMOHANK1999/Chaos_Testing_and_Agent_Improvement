# 04: Usage Mode Classifier

**Status:** ready-for-agent
**Blocked by:** 01 (Canonical Trace Schema), 03 (Trace Storage & Query Layer)

## What it builds

The component that turns a corpus of traces into emergent usage-mode categories ("what people are actually asking for"), without assuming a fixed taxonomy up front, and does it fast enough to re-run routinely as the corpus grows.

## Problem

Usage categories aren't known in advance and shouldn't be hardcoded — a fixed taxonomy hides whatever the taxonomy's author didn't anticipate. The classifier needs to discover categories from the data and produce human-readable labels for them, and — this is the priority constraint for this component — do it at a speed that scales, not just a quality that scales. A clustering pipeline that's accurate but takes hours per run on a growing corpus is a pipeline nobody re-runs.

**Revision note:** the first pass of this spec led with BERTopic on quality grounds alone and didn't treat speed as a first-class metric. That was a real gap, not a preference call — see below for the corrected framing.

## Open-Source Landscape

The clustering *architecture* (embed → reduce dimensionality → density-cluster → label) is still sound and still the standard shape for unsupervised topic/intent discovery in 2026; what changes under a speed constraint is which implementation fills each stage, not whether to use this shape at all:

- **BERTopic** — the reference implementation of that architecture. Its own documentation is explicit that it's modular: the embedding model, dimensionality reducer, and clustering algorithm are each swappable independently. "BERTopic" names the pipeline, not a commitment to BERT-sized embeddings — this matters directly for the speed fix below.
- **Model2Vec** — distills a full sentence-transformer into a static embedding model: up to 50x smaller on disk and up to 500x faster CPU inference, with clustering explicitly listed as a supported downstream task, while retaining most of the semantic quality of the transformer it was distilled from. Actively maintained, already integrated into Sentence-Transformers and LangChain. Dropped in as BERTopic's embedding backbone, this removes the single most expensive step (transformer inference over every trace) without changing anything downstream.
- **cuML / NVIDIA CUDA-X** (formerly branded RAPIDS, renamed August 2026) — GPU acceleration for UMAP and HDBSCAN with a documented zero-code-change path (`cuml.accel`) over existing scikit-learn/UMAP/HDBSCAN code. Benchmarked at up to ~60x speedup for UMAP and 175-179x for HDBSCAN versus CPU. This is a deployment-environment option (needs a GPU), not a rewrite.
- **Top2Vec** — an earlier embedding-clustering approach; generally outperformed by BERTopic on clustering quality and offers no particular speed advantage, so it's retained here only as a comparison point, not a leading candidate.
- **Pure LLM approaches** (zero-shot classification against an iteratively-built taxonomy, or fully LLM-driven clustering such as **ClusterLLM**) — trade the embedding/clustering step for LLM calls per trace or per batch; typically higher per-trace latency and cost at any real volume than an embedding-based pipeline, since it's an LLM call in the hot path rather than a cheap vector operation.

## Build vs Buy Decision

**Keep the BERTopic-shaped pipeline (it's still the right architecture and still open-source, actively maintained), but make the swap the first pass missed: Model2Vec as the embedding backbone instead of a full sentence-transformer, with cuML GPU acceleration as an optional second lever where a GPU is available.** LLM labeling of discovered clusters is retained from the original design — labeling runs once per *cluster*, not per trace, so it doesn't sit in the per-trace hot path and doesn't reintroduce the cost/latency problem the embedding swap just fixed.

## Experiment Design

**Question:** which pipeline configuration clears an acceptable per-1,000-trace latency budget *first*, and among those that do, which has the best clustering quality? Speed is now a gate, not a tiebreaker.

- **Candidates:**
  (a) **BERTopic + Model2Vec embeddings + LLM cluster labeling** — the corrected leading hypothesis, CPU-only.
  (b) BERTopic + standard sentence-transformer embeddings + cuML GPU acceleration + LLM cluster labeling — the quality-first path, conditional on GPU availability.
  (c) BERTopic + standard sentence-transformer embeddings, CPU only, no GPU — the original baseline, kept specifically to quantify what (a) and (b) are improving on.
  (d) Pure LLM zero-shot classification (no embedding/clustering step) — kept as the non-embedding alternative.
- **Dataset:** the labeled intent datasets below (for a correctness read against a known answer key) plus a sample of 500–1,000 traces from the simulator's own output (spec 10) — the primary corpus this project is designed around, and the one that exists before any real traffic does — for the production-shaped read. Real traces, once a production integration exists, supplement this rather than replace it.
- **Metrics, in priority order:**
  1. *Latency per 1,000 traces processed* (wall clock, CPU-only environment as the baseline case — this project should not assume every deployment has a GPU).
  2. *Topic coherence* (c_v or NPMI) and *human-rated relevance* (2 raters, 1–5 Likert, inter-rater agreement reported) — computed only for candidates that clear the latency gate.
  3. *Cost per 1,000 traces* (LLM labeling calls only — the clustering step itself has no per-call cost).
- **Decision rule:** eliminate any candidate that fails the latency gate first, regardless of quality; among survivors, pick highest coherence + relevance. Expect (a) to clear the gate cheaply on commodity CPU and (c) to be the one eliminated by it, which is the whole point of re-running this experiment with speed as a gate instead of an afterthought; (b) is the fallback if (a)'s Model2Vec-distilled embeddings lose too much semantic quality on this project's specific request text, and a GPU is actually available in the deployment target.

## Test Datasets & Reference Implementations

- **CLINC150** (22.5K queries, 150 intents + out-of-domain), **Banking77** (13K queries, 77 fine-grained intents), and **MASSIVE** (utterances across 18 domains / 59 intents) — open, human-labeled intent datasets. Not this project's domain, but each gives a known-correct clustering answer key to sanity-check that a candidate recovers approximately the right cluster structure before ever pointing it at real or simulated traces.
- **LMSYS-Chat-1M** (`lmsys/lmsys-chat-1m` on Hugging Face, 1M real multi-turn LLM conversations) and **WildChat** (1M+ real ChatGPT conversations, Hugging Face) — the production-shaped experiment dataset: real request text at real length/language/style distribution, standing in for this project's own corpus before enough real or simulated traces exist.
- **ClusterLLM** and "Summaries as Centroids for Interpretable and Scalable Text Clustering" — open reference implementations of LLM-assisted clustering, worth reviewing as a fallback direction if candidate (d)'s pure-LLM approach turns out competitive on latency at the volumes actually seen in production (unlikely at scale, but a cheap thing to double check rather than assume).

## Interface & Implementation Decisions

- Interface: `classify(traces: list[NormalizedTrace]) -> UsageModes` where `UsageModes` is a list of `{label, description, member_trace_ids, keyword_terms}`.
- The embedding backbone is a configuration point, not a hardcoded choice — swapping Model2Vec for a different backbone later (or adding the cuML-accelerated path conditionally when a GPU is present) shouldn't require touching the calling code in spec 07.
- Runs as a batch job over the storage layer's query interface (spec 03), not per-trace inline — usage-mode discovery is a corpus-level operation by nature.
- Re-running the classifier as the corpus grows may reshuffle cluster boundaries; label stability across runs is a known limitation, not solved in v0 (see Out of Scope).

## Testing Decisions

- Test against fixture trace sets with known, engineered clusters (e.g., a fixture corpus hand-built from N obviously-distinct request types, or a slice of CLINC150/Banking77) and assert the classifier recovers approximately that many clusters — a coarse sanity check, not a precision benchmark (that's the experiment's job, run once, not on every CI run).
- Test the labeling step independently against fixed cluster keyword/document inputs, asserting it produces a non-empty, on-topic label — mock the clustering step here so labeling tests don't depend on clustering nondeterminism.
- The latency gate from the experiment becomes a standing regression check: re-run the chosen pipeline's throughput benchmark whenever the embedding backbone or clustering library version changes, not just once at selection time.

## Out of Scope

- Cross-run label stability / cluster tracking over time (a v1 concern).
- Real-time/incremental classification as new traces arrive — v0 re-runs in batch.
- GPU infrastructure provisioning itself — candidate (b) is evaluated if a GPU is available, not used to justify acquiring one.
