# 01: Canonical Trace Schema & TraceAdapter Interface

**Status:** ready-for-agent
**Blocked by:** None (can start immediately)

## What it builds

The one seam everything else in this repo depends on: a minimal canonical `Event` model that any trace source can be normalized into, and the `TraceAdapter` interface each source implements to do that normalization.

## Problem

Traces arrive in whatever shape their producing harness/agent happens to emit, and that shape isn't fixed — a new agent or harness onboarded later may look nothing like the ones known today. A rigid, all-fields-required schema breaks the first time a source doesn't populate every field, and locks out sources that don't fit the shape at all. The canonical model needs to be small enough that nothing downstream over-assumes, while still carrying enough structure (ordering, actor, kind) for analysis to work.

## Open-Source Landscape

The obvious prior art is the [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/blog/2026/genai-observability/) — a standardized vocabulary (`gen_ai.*` attributes) for LLM/agent spans covering client calls, agent invocations, tool execution, and events, modeling a whole agent run as a span tree (`invoke_agent` → `chat` → `execute_tool`). As of mid-2026 every `gen_ai.*` attribute is still stability-badged "Development" (pre-1.0, names can still change between releases), and the `gen_ai.*` conventions were split into their own repo off the main OTel semconv repo in June 2026 specifically because they're moving fast. [OpenInference](https://github.com/Arize-ai/openinference) (Arize's own semantic conventions, used by Phoenix) is a more mature, narrower alternative covering similar ground.

Adopting either wholesale — i.e., building on OTel's span/trace-tree data model and its Collector/exporter infrastructure — would mean pulling in distributed-tracing infrastructure this project doesn't need for v0 batch analysis, and pinning to a vocabulary that's explicitly still shifting under us.

## Build vs Buy Decision

**Borrow the vocabulary, don't buy the infrastructure.** Use `gen_ai.*`-style attribute names as the naming convention for populating the canonical `Event`'s open `attributes` bag (e.g. `gen_ai.usage.input_tokens`, `gen_ai.request.model`), so that data produced by OTel-GenAI-instrumented sources (see spec 02) maps in with minimal translation and the project isn't inventing parallel naming for the same concepts. Do not adopt OTel's span-tree object model, Collector, or exporter pipeline — those solve distributed-tracing problems (sampling, propagation across services) this project doesn't have at v0 scale (batch analysis of exported traces).

This is an architecture decision, not a comparison between competing finished components — there's nothing to run a horse race between. No experiment is warranted here (see Experiment Design).

## Experiment Design

Not applicable. This spec defines the interface everything else measures itself against; it has no peer candidates to benchmark. (Specs 02–10 each run their own experiment where genuine alternatives exist.)

## Test Datasets & Reference Implementations

- **OpenInference** and **Arize Phoenix** ship real captured example traces (in their own docs/example notebooks) in OTel/OpenInference span format — usable as schema-conformance fixtures for validating that the canonical `Event` envelope can represent a real trace shape, without needing a live agent running.
- **LMSYS-Chat-1M** (`lmsys/lmsys-chat-1m` on Hugging Face, 1M real multi-turn LLM conversations) and **WildChat** (1M+ real ChatGPT conversations, Hugging Face) — neither is trace-instrumented, but their raw multi-turn JSON is a ready source of realistic conversation shapes (long turn counts, multilingual content, occasional tool-like structure) to stress-test the schema against edge cases a hand-written fixture set would likely miss.

## Interface & Implementation Decisions

- **`Event`** (the unit of the canonical stream): required envelope —
  - `sequence` (int, order within the trace)
  - `actor` (`user` | `agent` | `tool`)
  - `timestamp`
  - `kind` (`message` | `tool_call` | `tool_result` | `handoff` | `error`)
  - `attributes` (open map, `gen_ai.*`-style keys where applicable, no key is guaranteed present)
- **Trace-level fields** (`model`, `harness`, `version`) live in `attributes` at the trace's root, populated best-effort by the adapter. Never assume presence; the correct default is `unknown`, not an exception.
- **`TraceAdapter`** interface: a single method, `normalize(raw_source_export) -> list[Event]`. No shared base class beyond the interface — a deep module here means "small interface, whatever internal parsing a given source needs," not a shared inheritance hierarchy.
- Versioning: the `Event` schema itself is versioned (a top-level `schema_version` field) from day one, since it's the one piece of this project every other component depends on and will need to evolve without breaking every adapter and consumer simultaneously.

## Testing Decisions

- Test the `Event` schema itself only for shape/validation (required envelope fields present, `attributes` accepts arbitrary keys, unknown `kind` values rejected). No adapter-specific logic belongs in these tests.
- Every other spec's tests are written against fixtures of canonical `Event`s, never against a native source format — that's the whole point of the seam. Enforce this by keeping native-format fixtures physically colocated only with the adapter that owns them (spec 02).

## Out of Scope

- Any specific adapter implementation (spec 02).
- Storage or persistence of `Event`s (spec 03).
- Deciding which real-world harness to instrument first — that's an integration-target decision for spec 02, not a schema decision.
