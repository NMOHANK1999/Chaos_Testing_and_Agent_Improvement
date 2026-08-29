# 05: Cost Calculator & Pricing Table

**Status:** ready-for-agent
**Blocked by:** 01 (Canonical Trace Schema), 03 (Trace Storage & Query Layer)

## What it builds

The component that turns token/model attributes on a trace into a cost figure, sliceable by model, harness, and version, degrading gracefully to "unknown" when a trace lacks the data to compute cost.

## Problem

Per-model pricing changes over time and varies by provider; hand-maintaining a pricing table is exactly the kind of bookkeeping that goes stale silently. Cost also has to be computable even when a given trace source doesn't cleanly report token counts — the calculator can't assume complete data.

## Open-Source Landscape

- **LiteLLM** — an open-source universal LLM client/gateway that ships a maintained pricing database (claimed 1,600+ models) and built-in cost-tracking utilities, updated as providers change pricing.
- **llm-tokencost** — a lightweight companion library built on LiteLLM's pricing data, adding budget alerts, streaming support, and thread-safe tracking via LiteLLM's callback system.
- **tokencost** — a separate library computing USD cost for 400+ models by counting tokens (Tiktoken for most providers, Anthropic's official token-counting API for Claude 3+) against its own pricing table.
- Hand-maintained custom table — the fallback if none of the above cover a given model/provider.

## Build vs Buy Decision

**Adopt an existing pricing database; do not hand-maintain one.** This is squarely the kind of problem — a large, frequently-changing lookup table with real accuracy stakes — that should never be reinvented per this project's stated priority. The open question is which library's table to adopt, not whether to build one from scratch.

## Experiment Design

**Question:** which pricing source gives the best coverage and accuracy for the models/harnesses actually appearing in this project's traces?

- **Candidates:** LiteLLM's pricing DB, tokencost, a hand-maintained table (as a floor/fallback candidate only, to quantify what adopting a library actually saves).
- **Method:** pull a sample of traces spanning every model/harness currently in use; compute cost via each candidate; where actual provider-invoiced cost is available for the same calls, compare against it directly.
- **Metrics:**
  - *Coverage* — % of models/harnesses seen in the sample that have a pricing entry.
  - *Accuracy* — deviation from known invoiced cost where ground truth exists.
  - *Staleness risk* — update cadence / commit activity of the upstream project (a table that hasn't been touched in months is a liability regardless of current coverage).
  - *Integration effort* — how much glue code is needed to call it from our `Event.attributes` shape.
- **Decision rule:** highest coverage + accuracy at acceptable integration effort. Given LiteLLM's stated 1,600+-model coverage and active maintenance (it's also a candidate transport layer in spec 09, so adopting its pricing DB avoids a second dependency for the same ecosystem), it's the leading hypothesis — but the experiment should confirm coverage against this project's actual model list rather than assume it.

## Interface & Implementation Decisions

- Interface: `compute_cost(trace: NormalizedTrace) -> CostResult { amount | unknown, currency, breakdown_by }`.
- The pricing table is a dependency injected into the calculator, not hardcoded — swapping the chosen library later, or layering a project-specific override table on top for a model the upstream table doesn't yet cover, doesn't require touching calculator logic.
- Missing token/model data on a trace produces `unknown`, never an exception and never a silently-dropped trace; the analysis report (spec 07) surfaces "% of traces with unknown cost" as its own metric rather than excluding them from totals unlabeled.

## Testing Decisions

- Test the calculator against canonical `Event` fixtures covering: complete token+model data, missing token data, missing model data, and an unrecognized model not in the pricing table — asserting the `unknown` path in the latter three, never an exception.
- Pricing-table accuracy itself (does the adopted library's number match reality) is validated once via the experiment above, not re-verified on every test run — that's an external dependency's correctness, not this component's.

## Out of Scope

- Real-time cost alerting/budgets — this produces a report figure, not a live monitor.
- Negotiated/custom enterprise pricing that differs from public list price.
