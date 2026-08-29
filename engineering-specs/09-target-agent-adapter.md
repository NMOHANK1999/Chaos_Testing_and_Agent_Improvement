# 09: Target Agent Adapter

**Status:** ready-for-agent
**Blocked by:** 01 (Canonical Trace Schema)

## What it builds

The pluggable interface the simulator uses to actually drive a target agent — decoupled from persona logic, so the same simulator can chaos-test more than one agent/harness.

## Problem

Target agents aren't uniform: some are reachable via a standard chat-completions-style API, others (a CLI-driven harness, for instance) have no such interface at all. The simulator shouldn't need to know which kind it's talking to.

## Open-Source Landscape

**LiteLLM** — already adopted in spec 05 for pricing data — is also a universal client giving a single OpenAI-compatible interface across 100+ providers (OpenAI, Anthropic, Bedrock, Azure, VertexAI, and others), with built-in load balancing and logging. For any target agent reachable via a standard request/response API, this covers the transport without writing per-provider client code.

For agents that aren't API-shaped (a CLI/harness-driven target, for example), no general-purpose OSS client applies — that path is necessarily custom.

## Build vs Buy Decision

**Default to LiteLLM as the transport for any API-reachable target agent; write a bespoke adapter only for non-API-shaped targets.** This isn't a comparison between competing finished libraries — LiteLLM is the only serious general-purpose candidate for the API-shaped case, and the CLI/harness case has no general-purpose candidate at all. No experiment is warranted (see below); the decision is which category a given target agent falls into, not which library wins a benchmark.

## Experiment Design

Not applicable — see Build vs Buy Decision. This spec is an architecture/build-vs-buy call with no genuine multi-candidate comparison to run. (Contrast with spec 02, where multiple instrumentation libraries are genuinely comparable and get benchmarked.)

## Interface & Implementation Decisions

- Interface: `agentAdapter.send(conversation_so_far) -> agent_response`, with the response captured in whatever native shape the target agent produces (normalized later via that target's own `TraceAdapter`, spec 02, not here).
- **API-shaped targets:** implemented once, generically, via LiteLLM — a single adapter parameterized by provider/model config, not one adapter per provider.
- **Non-API-shaped targets** (CLI/harness-driven): a separate, target-specific adapter per such target, since there's no shared library to lean on. Building the first one of these (alongside the first API-shaped one) is itself the proof that this seam is real — two adapters, not one.

## Testing Decisions

- Test the LiteLLM-backed adapter against a mocked/stubbed LLM response (no real API calls in tests), asserting the adapter's interface contract (input conversation shape in, response out) holds regardless of which underlying provider is configured.
- Test any CLI/harness-shaped adapter against a scripted fake process, for the same reason — no live target agent in the test suite.

## Out of Scope

- Building an adapter for every possible target agent — just enough to prove both categories (API-shaped, non-API-shaped) exist and work.
- Rate limiting / retry policy beyond whatever LiteLLM already provides out of the box.
