# Milestone backlog — `SDK Model & Cost Optimizer`

Ready-to-file issues for [PRD 0001](0001-model-cost-optimizer.md). Each issue
title is prefixed with its phase. Sizes: S ≤ 2 days, M ≤ 1 week, L ≤ 2 weeks.

Milestone description to paste into GitHub:

> OpenRouter-style local routing in the Python SDK. One client calls several
> providers with the customer's own keys, routes on quality, price, latency and
> data policy, falls back on failure, enforces budgets, and reports exact cost
> and provable savings to the registry. See docs/prd/0001-model-cost-optimizer.md.

## P0 — Foundations

| # | Title | Size | Acceptance criteria |
|---|---|---|---|
| 1 | `[P0] Price events at served_model, not requested_model` | S | `_build_event` (sync + async) prices at `served_model`; `model="auto"` no longer yields `cost=None` when the served model is known; regression test. |
| 2 | `[P0] Model & pricing catalog (LiteLLM-compatible schema)` | M | `prompt_shields/catalog.json` + `catalog.py` loader. It has cache-read, cache-write, reasoning, `above_200k`, batch and capability columns and `max_input_tokens`. `estimate_cost()` stays backward compatible. Unknown model → `None`. |
| 3 | `[P0] Catalog generator + gateway parity check` | M | `scripts/build_catalog.py` emits the SDK JSON and the gateway `defaultPolicy` pricing; a CI step fails on drift. `FORK_NOTICE.md` is updated if an upstream file is touched. |
| 4 | `[P0] Cached and reasoning token accounting` | S | OpenAI `cached_tokens` and `reasoning_tokens`, Anthropic `cache_read`/`cache_creation`, and Gemini `cachedContentTokenCount` are parsed into a `CostBreakdown`. Fixtures prove ≤2% delta. |
| 5 | `[P0] Streaming usage capture` | M | OpenAI streams inject `include_usage`; Anthropic `message_delta` usage is accumulated; the event is emitted on stream close; aborted streams are emitted with `cost_source="estimated"`. |
| 6 | `[P0] price_catalog_version + cost_source on events` | S | Both fields appear on every SDK event; the collector accepts them (nullable). |

## P1 — Multi-provider client

| # | Title | Size | Acceptance criteria |
|---|---|---|---|
| 7 | `[P1] ShieldsRouter / AsyncShieldsRouter skeleton` | M | `providers={...}` BYOK config, `vendor/model` IDs, lazy upstream construction, and the same discovery metadata as `ShieldsClient`. Existing clients are unchanged. |
| 8 | `[P1] OpenAI-shape normalisation: Anthropic` | L | Text, system prompt, tools/tool_use and streaming map to and from the chat-completions shape. `resp.raw` keeps the native object. |
| 9 | `[P1] Google Gemini provider adapter` | L | A `GoogleAdapter` extracts usage and tool calls, normalises responses and is added to the catalog. |
| 10 | `[P1] resp.ps metadata (route, cost, attempts)` | S | A typed `PSResult` is attached to every response from the router client. |

## P2 — Router, fallback, shared policy

| # | Title | Size | Acceptance criteria |
|---|---|---|---|
| 11 | `[P2] Route policy JSON Schema + YAML loader (SDK & gateway)` | M | `schemas/route-policy.v1.json`; `RoutePolicy.load()` in Python and `loadPolicy()` in the gateway validate against it. The default policy is generated from the catalog. |
| 12 | `[P2] Python RouterStrategy + Heuristic/Cost/Latency strategies` | M | A port of gateway semantics with the same precedence and engagement rules. Golden tests are shared with `gateway/.../router.test.ts`. p50 < 1 ms. |
| 13 | `[P2] ProviderPrefs (sort, only/ignore, max_price, require)` | M | OpenRouter-equivalent provider preferences, applied as filters before the strategy runs. |
| 14 | `[P2] Data-policy filter (classification → providers/regions/ZDR)` | M | `restricted` data is never auto-routed. Property tests show zero violations. `served_provider`/`served_region` are recorded. |
| 15 | `[P2] Fallback & retry engine + circuit breaker` | L | Implements the §6.4 matrix: no retry after a stream has started, content-filter fallback off by default, capability check before a fallback target is chosen, and `fallback_chain` on the event. |
| 16 | `[P2] Prompt-cache-aware stickiness` | S | Reuse the last provider for a `session_id` when its cost is within ε. |

## P3 — Budgets, savings, registry

| # | Title | Size | Acceptance criteria |
|---|---|---|---|
| 17 | `[P3] Budget API + in-process store` | M | Supports the `raise` / `downgrade` / `warn` modes, a pre-call worst-case check and post-call reconciliation. The per-process limitation is documented. |
| 18 | `[P3] Redis BudgetStore (optional extra)` | S | `[optimizer-redis]` extra; atomic increments; TTL per period. |
| 19 | `[P3] Collector + DB: routing/cost fields (Alembic 004)` | M | Nullable columns for the §6.7 fields; ingest validation; tests. CONTRIBUTING privacy note in the PR. |
| 20 | `[P3] counterfactual_cost + usage-summary savings` | M | `counterfactual_cost` on routed events; `GET /assets/{id}/usage-summary` returns `savings_usd`, `fallback_rate` and `cost_by_model`; OpenAPI + Mintlify docs updated. |
| 21 | `[P3] OpenTelemetry GenAI spans (optional extra)` | S | `gen_ai.*` attributes under a pinned semconv version, plus `prompt_shields.route.*` and `prompt_shields.cost.*`. |
| 22 | `[P3] Docs: SDK guide, README limits, demo` | M | Rewrite SDK Guide §6 and `docs/sdks/python.mdx`; add a limits section to the README ("What this does not do"); add a `demo/demo_router.py` that shows fallback, a budget downgrade and savings. |

## P4 — Learned routing (experimental)

| # | Title | Size | Acceptance criteria |
|---|---|---|---|
| 23 | `[P4] LearnedStrategy adapter (RouteLLM mf, optional extra)` | M | Behind `[optimizer-learned]`; threshold calibrated to "% strong"; falls back to the heuristic on error. |
| 24 | `[P4] Routed-call export for offline training` | S | Exports metadata-only training rows from the registry (no prompt text unless the tenant opted into `send_prompt_text`). |
| 25 | `[P4] Offline eval harness + published savings report` | M | Public benchmark plus an LLM-judge score; reports the savings vs. quality curve; the results confirm or revise the PRD §8 target. |

## Suggested labels

`area:sdk`, `area:gateway`, `area:collector`, `optimizer`, `phase:P0` … `phase:P4`, `size:S|M|L`.
