# PRD 0001 — SDK Model & Cost Optimizer

| | |
|---|---|
| **Status** | Draft — for review |
| **Owner** | TBD |
| **Milestone** | `SDK Model & Cost Optimizer` |
| **Packages** | `packages/sdk` (primary), `gateway/src/middlewares/router` (policy parity), `packages/collector` + `packages/db` (new event fields) |
| **Last updated** | 2026-10-10 |

## 1. Summary

Give the Python SDK an OpenRouter-style **model & cost optimizer**. One client
should be able to call several providers with the developer's own keys, pick a
model from a declared policy (quality intent, price ceiling, latency, data
residency), fall back when a provider fails, enforce spend budgets, and report
exact cost and provable savings into the registry.

None of this needs a hosted hop. Today the same behaviour needs the gateway: the
SDK only emits `X-PS-*` hints, and the gateway's `HeuristicStrategy` decides.
This PRD adds a **local routing mode** to the SDK. It shares one policy and
pricing catalog with the gateway, so the two can never disagree about what a
model costs or which group it belongs to.

## 2. Background — what exists today

| Capability | Where | State |
|---|---|---|
| Route hints (`RouteHint` → `X-PS-Quality`, `X-PS-Max-Cost`, `X-PS-Route`, `X-PS-Cache`) | `packages/sdk/prompt_shields/types.py` | Shipped. The SDK never routes. |
| Cost-aware router (`HeuristicStrategy`, `RoutePolicy`, budget clamp) | `gateway/src/middlewares/router/` | Shipped behind `PS_ROUTER_ENABLED`. |
| `requested_model` / `served_model` on events | `client.py`, `ps-telemetry.ts` | Shipped. |
| Pricing table | `packages/sdk/prompt_shields/pricing.py` | 14 models, Q1-2026 snapshot, per-1k in/out only. |
| Gateway pricing | `gateway/src/middlewares/router/policy.ts` | **Hand-copied** from `pricing.py`, so the two drift. |

Gaps this milestone closes:

1. **One vendor per client.** A `ShieldsClient` wraps exactly one upstream
   SDK, so cross-provider routing or fallback is impossible without the gateway.
2. **Cost is computed from the *requested* model** (`client.py`,
   `_build_event`). A `model="auto"` call always records `cost=None`, and any
   downgrade is priced at the wrong model's rate.
3. **The pricing model is too coarse.** It has no cached-input, cache-write,
   reasoning-token, long-context tier or batch pricing. Most of the 2026
   savings levers live in exactly those columns.
4. **Streaming records no usage.** It does not request `stream_options.include_usage`
   and does not read Anthropic `message_delta` usage.
5. **No fallback, retry policy, budgets or price ceilings** on the SDK path.
6. **No savings evidence.** `requested_model` and `served_model` are recorded,
   but nothing computes "what this would have cost on the requested model".

## 3. Market research (condensed)

| Product | Runs where | Key ideas we should adopt | Ideas we should not copy |
|---|---|---|---|
| **OpenRouter** | Hosted SaaS | `provider: { order, only, ignore, sort: price/latency/throughput, max_price, zdr, data_collection, quantizations, allow_fallbacks }`; `models: [...]` ordered fallback; exact `usage.cost` on every response; `:floor` / `:nitro` shorthands; Auto Router with an allow-list of candidate models; sticky provider routing to keep prompt caches warm. | A hosted middleman that sees prompts. It also takes a 5–5.5% fee. |
| **LiteLLM** | SDK + proxy | `model_prices_and_context_window.json`, the de-facto price schema (about 4k entries, with cache, reasoning, tiered and batch columns); routing strategies (`simple-shuffle`, `latency-based`, `cost-based`, `least-busy`); separate `fallbacks` / `context_window_fallbacks` / `content_policy_fallbacks`; cooldowns; budgets per key, user, team and model. | Heavy proxy plus a Redis dependency for basic routing. |
| **Portkey** (our gateway's upstream) | Gateway | Config `strategy.mode: fallback / loadbalance / conditional`; `on_status_codes`; simple and semantic cache. | — (already inherited by `gateway/`). |
| **RouteLLM** (LMSYS, Apache-2.0) | Library | A strong-vs-weak learned router (`mf` matrix factorisation recommended) with a threshold calibrated to "% of calls sent to strong". Reported >2x cost reduction while keeping about 95% of GPT-4 quality on MT-Bench. | An embedding call on the hot path by default. |
| **NotDiamond / Martian / Unify** | Hosted APIs | Pareto selection by quality, cost and latency, and custom routers trained on your own eval data. | Sending prompts to yet another third party. |
| **Vercel / Cloudflare AI Gateway, Helicone** | Hosted / edge | Provider `order`/`only`/`sort`, dynamic routing splits, caching, OTel tracing. | — |

**Our differentiation.** We are the only optimizer whose routing decisions,
cost records and savings land in an **AI asset registry** that an auditor can read.
Each record says which use case, business unit, data classification and model
served it, and at what cost. Local mode keeps prompts inside the customer's
process: routing runs with their keys, and nothing passes through a third party.

The full research notes and sources are in [Appendix A](#appendix-a--sources).

## 4. Goals and non-goals

### Goals

- **G1.** Route, fall back and enforce budgets across OpenAI, Anthropic and Google
  Gemini from **one client**, with no gateway required.
- **G2.** Make cost **accurate**: within ±2% of the provider-reported cost for
  every model in the catalog, including cache, reasoning and streaming calls.
- **G3.** Have **one source of truth** for pricing, model capabilities and routing
  policy, used by both the SDK and the gateway.
- **G4.** Make savings **provable** in the registry: a counterfactual cost on
  every routed event, plus an aggregate savings report.
- **G5.** Respect governance: a routing decision may never send data to a
  provider, region or retention class that the call's `data_classification`
  policy forbids.
- **G6.** Stay drop-in. Existing `ShieldsOpenAI` / `ShieldsAnthropic` callers
  see **zero behaviour change** unless they opt in.

### Non-goals (this milestone)

- A hosted, OpenRouter-style service, credits or billing. Prompt Shields does not
  resell inference.
- Semantic response caching in the SDK. It stays in the gateway (`middlewares/cache`).
- Prompt compression (LLMLingua-style). It conflicts with prefix caching and is
  out of scope.
- A production learned router. Phase 4 ships only an **experimental** adapter
  and a data-export path.
- A TypeScript SDK. Only the Partner API client exists in TS today.
- Embeddings, images, audio and batch-API routing. Chat/messages only.

## 5. Users and use cases

| Persona | Job to be done |
|---|---|
| **App developer** | "Give me the cheapest model that is good enough for this call, and fail over if OpenAI is down, without me writing retry code." |
| **Platform / FinOps** | "Cap the HR screening use case at $500/month and downgrade instead of failing when it gets close." |
| **Security / governance** | "Confidential data may only go to providers with ZDR in the EU, and I need evidence that routing honoured it." |
| **Engineering leadership** | "Show me how much routing saved last quarter, per business unit." |

## 6. Product design

### 6.1 Developer experience

The existing clients are unchanged. The new entry point is a multi-provider
client that uses OpenRouter-style `vendor/model` identifiers:

```python
from prompt_shields import ShieldsRouter, RoutePolicy, Budget

client = ShieldsRouter(
    providers={                       # BYOK — keys never leave the process
        "openai":    {"api_key": os.environ["OPENAI_API_KEY"]},
        "anthropic": {"api_key": os.environ["ANTHROPIC_API_KEY"]},
        "google":    {"api_key": os.environ["GEMINI_API_KEY"]},
    },
    ps_api_key="ps-...",
    business_unit="HR", use_case="interview-screening",
    data_classification="confidential",
    policy=RoutePolicy.load("ps-routing.yaml"),   # optional; sane default bundled
    budget=Budget(usd=500, period="month", on_exceed="downgrade"),
)

# 1. Let the optimizer choose (OpenRouter "auto")
resp = client.chat.completions.create(
    model="auto",
    messages=[...],
    route=RouteHint(quality="draft", max_cost=0.005),
)

# 2. Ordered fallback list (OpenRouter `models: [...]`)
resp = client.chat.completions.create(
    models=["anthropic/claude-sonnet-4-5", "openai/gpt-4o", "google/gemini-2.5-pro"],
    messages=[...],
)

# 3. Provider preferences (OpenRouter `provider: {...}`)
resp = client.chat.completions.create(
    model="auto",
    messages=[...],
    provider=ProviderPrefs(sort="price", only=["openai", "anthropic"],
                           max_price={"input": 3.0, "output": 15.0},  # USD / 1M tok
                           require=["tools", "json_schema"]),
)

resp.ps.route        # RouteDecision(model=..., group=..., reason=..., est_cost=...)
resp.ps.cost         # CostBreakdown(input=..., cached_input=..., output=..., reasoning=..., total=..., source="reported"|"estimated")
resp.ps.attempts     # [Attempt(model="openai/gpt-4o", error="429"), Attempt(model="anthropic/...", ok=True)]
```

- **Response shape.** The response is always an **OpenAI chat-completions shape**,
  whichever provider served it, because that is the lingua franca OpenRouter
  established. A `raw` attribute keeps the native provider object.
  `ShieldsAnthropic` callers who want the native Anthropic shape keep using
  `ShieldsAnthropic`.
- **Gateway mode** stays available: `ShieldsRouter(mode="gateway", base_url=...)`
  sends the existing `X-PS-*` hints and lets the gateway decide. Local mode is the default.

### 6.2 Model & pricing catalog (`prompt_shields.catalog`)

- **Schema.** Field-compatible with LiteLLM's `model_prices_and_context_window.json`:
  - per-token `input`, `output`, `cache_read`, `cache_write_5m`, `cache_write_1h` and `reasoning` prices
  - `*_above_200k` long-context tiers
  - `batch` / `flex` / `priority` service tiers
  - `max_input_tokens`, `max_output_tokens`
  - capability flags: `tools`, `json_schema`, `vision`, `reasoning`, `prompt_caching`
  - `regions`, `zdr_available`
- **Ships as `catalog.json`**, a versioned and dated snapshot inside the wheel.
  Every event records `price_catalog_version`.
- **Optional refresh.** `catalog.refresh(url=..., ttl=24h)` fetches a signed
  catalog and pins it with an ETag. It is off by default so that air-gapped
  installs work.
- **One generator** (`scripts/build_catalog.py`) writes both the SDK
  `catalog.json` and the gateway's `defaultPolicy` pricing. CI fails if the two drift.
- An unknown model reports cost as **`None` ("unmetered")**, never `0`. This
  keeps today's contract.
- `pricing.estimate_cost()` stays as a thin, backward-compatible wrapper over the catalog.

### 6.3 Router (`prompt_shields.routing`)

A Python port of the gateway's `RouterStrategy` interface, so the two share semantics:

```
RouteRequest(messages, requested_model, est_input_tokens, max_output_tokens,
             quality, max_cost, explicit_group, provider_prefs, data_policy)
  -> filter   : capability (tools/json_schema/context window), data policy
                (region, ZDR, classification allow-list), provider only/ignore,
                max_price, circuit-breaker state
  -> strategy : HeuristicStrategy (port of gateway) | CostStrategy (cheapest) |
                LatencyStrategy (EWMA p50) | custom callable
  -> budget   : clamp to max_cost / remaining budget (downgrade across groups)
  -> RouteDecision(model, group, reason, est_cost, candidates=[...ordered fallbacks])
```

- **Precedence** is identical to the gateway:
  explicit `model_group` > `quality` / `max_cost` hints > policy default.
- **Engagement** is identical too. Routing only engages for `model="auto"`,
  `models=[...]` or a non-empty hint. A concrete model with no hint is sent as-is.
- **Token estimate.** Use `tiktoken` when it is installed (`[optimizer]` extra),
  otherwise ~4 chars/token. Provider-reported usage always overrides the estimate afterwards.
- **Overhead budget.** p50 < 1 ms and p99 < 5 ms for rule-based strategies, with
  no network I/O on the hot path.

### 6.4 Fallback & retry engine

| Condition | Default action |
|---|---|
| 429 / rate-limit | Respect `Retry-After` up to `max_wait`, then move to the next candidate |
| 5xx, timeout, connection error | Next candidate. Open a circuit breaker after N failures in a window (cooldown) |
| Context-length exceeded | Next candidate with a larger `max_input_tokens` |
| 400 / 401 / 403 (bad request, auth) | **Do not fall back.** Raise. |
| Provider content-policy refusal | **Do not fall back by default** (`fallback_on_content_filter=False`). Routing a refused prompt to a laxer model is a policy-evasion path. If enabled, it is recorded on the event. |
| Stream already emitting tokens | **Never retry.** Raise the mid-stream error so the caller doesn't get duplicated output or double billing. |

Before choosing a fallback target, the engine checks capability compatibility:
tools, `response_format` and vision must be supported by the target.

### 6.5 Budgets & guardrails on spend

- **Scopes.** `Budget(usd, period=day|week|month, scope=client|use_case|user_id)`.
- **Pre-call check.** Worst-case estimate = input estimate + `max_tokens` ×
  output price. After the call, reconcile against reported usage.
- **`on_exceed` options.**
  - `"raise"` raises `BudgetExceeded`.
  - `"downgrade"` re-routes to the cheapest group that fits.
  - `"warn"` proceeds and emits `budget_state="exceeded"`.
- **Default store** is in-process, which means one counter per process.
  `BudgetStore` is a protocol, and a Redis implementation ships behind an extra
  for multi-replica deployments. The docs must say plainly that the in-process
  store is per process. That is consistent with how the README states limits.
- **`max_price`** (per-token ceiling) and **`max_cost`** (per-call ceiling) are
  separate knobs, mirroring OpenRouter and `X-PS-Max-Cost`.

### 6.6 Accurate cost accounting

- Price the call at the **served** model. This fixes gap 2.
- Read cached and reasoning tokens:
  - OpenAI: `prompt_tokens_details.cached_tokens` and `completion_tokens_details.reasoning_tokens`
  - Anthropic: `cache_read_input_tokens`, `cache_creation_input_tokens`
  - Gemini: `cachedContentTokenCount`
- **Streaming.**
  - OpenAI: inject `stream_options={"include_usage": True}`.
  - Anthropic: accumulate `message_start` and `message_delta` usage.
  - Emit the event on stream close. Aborted streams are emitted with
    `cost_source="estimated"`.
- `CostBreakdown` is attached to the response (`resp.ps.cost`) and to the event.

### 6.7 Telemetry & savings

New **optional** event fields. All are metadata, with no prompt content, as
CONTRIBUTING requires.

| Field | Type | Purpose |
|---|---|---|
| `route_group`, `route_reason`, `route_est_cost` | str, str, float | Parity with gateway events |
| `route_mode` | `local` \| `gateway` \| `none` | Which component decided |
| `fallback_chain` | list[{model, error_class}] | Reliability evidence. Error class only, never the message body. |
| `cost_breakdown` | {input, cached_input, cache_write, output, reasoning} | Accurate cost |
| `cost_source` | `reported` \| `estimated` | Honesty about streaming and aborted calls |
| `counterfactual_cost` | float \| null | Cost had `requested_model` (or the policy's `baseline_model` for `auto`) served the call |
| `price_catalog_version` | str | Reproducibility |
| `served_provider`, `served_region` | str | Proof of data-policy compliance |
| `budget_state` | `ok` \| `near` \| `exceeded` \| `downgraded` | FinOps |

- **Collector and DB.** Add nullable columns via an Alembic migration (`004_routing_cost.py`).
- **Registry API.** Extend `GET /assets/{id}/usage-summary` with `savings_usd`,
  `fallback_rate` and `cost_by_model`.
- **OTel (optional extra).** Emit `gen_ai.*` semantic-convention spans
  (`gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.response.model`,
  `gen_ai.usage.*`) and put the routing and cost attributes under `prompt_shields.*`.
  Pin the semconv version, because the spec is still in Development status.

### 6.8 Policy file (shared SDK ⇄ gateway)

Model IDs below are illustrative; the bundled default is generated from the catalog.

```yaml
version: 1
baseline_model: openai/gpt-4o            # for counterfactual savings on "auto"
default_group: balanced
quality_to_group: { draft: cheap, balanced: balanced, critical: frontier }
groups:
  cheap:    [openai/gpt-4o-mini, google/gemini-2.5-flash, anthropic/claude-haiku-4-5]
  balanced: [openai/gpt-4o, anthropic/claude-sonnet-4-5]
  frontier: [anthropic/claude-opus-4-1, openai/gpt-5]
data_policy:
  restricted:   { allow_providers: [], }            # never routed; must pin explicitly
  confidential: { require_zdr: true, regions: [eu, us] }
fallback: { on: [rate_limit, server_error, timeout, context_length], max_attempts: 3 }
```

`RoutePolicy.load()` in Python and `loadPolicy()` in the gateway both
validate against one JSON Schema (`schemas/route-policy.v1.json`).

## 7. Phased delivery (milestone breakdown)

| Phase | Scope | Exit criteria |
|---|---|---|
| **P0 — Foundations** | Catalog schema + generator; price at served model; cached/reasoning token pricing; streaming usage; `price_catalog_version`; gateway parity CI check | Cost within ±2% of provider-reported cost on a recorded fixture set; `auto` calls no longer record `cost=None` |
| **P1 — Multi-provider client** | `ShieldsRouter` (sync + async); OpenAI-shape normalisation for Anthropic + Gemini (text, tools, streaming); `vendor/model` IDs | Same test suite passes against all three providers via `respx` fixtures |
| **P2 — Router + fallback** | Python `RouterStrategy` + Heuristic/Cost/Latency strategies; `ProviderPrefs`; capability + data-policy filters; fallback engine + circuit breaker; shared policy YAML + JSON Schema | Router overhead p50 < 1 ms; fallback matrix (§6.4) covered by tests; zero data-policy violations in property tests |
| **P3 — Budgets & savings** | `Budget` + in-process / Redis stores; new event fields; Alembic migration; usage-summary savings; OTel extra | Demo shows budget downgrade, and a savings number appears in the registry |
| **P4 — Learned routing (experimental)** | `LearnedStrategy` adapter (RouteLLM `mf`, optional extra); export of routed-call telemetry for offline training; offline eval harness | Offline eval report on a public benchmark: savings at a quality-retention target, published in docs |

## 8. Success metrics

- **Accuracy.** Cost delta to the provider-reported cost is ≤ 2% at p95 across
  catalog models (fixture suite plus a sample of live calls).
- **Savings.** On the reference workload (`demo/`), `model="auto"` with the
  default policy reaches ≥ 30% lower cost than the `baseline_model`, with
  ≤ 2 points of quality loss on an LLM-judge eval. The target is to be confirmed
  in P4.
- **Reliability.** With one provider forced to 100% failure, ≥ 99% of calls
  still succeed through fallback in the integration test.
- **Overhead.** Added p50 latency is < 1 ms (rule-based) and there is no network
  I/O on the hot path.
- **Adoption.** Number of tenants with `route_mode=local` events, and the share
  of SDK events with `cost_source=reported`.

## 9. Risks & mitigations

| Risk | Mitigation |
|---|---|
| **Design reversal.** "The SDK never picks a model" is stated in `types.py`, `PS_README.md` and the SDK guide | Local routing is opt-in (`ShieldsRouter`); existing clients are untouched; docs updated in the same PR as P1; gateway mode retained |
| Price data goes stale | Versioned catalog, a generator from one source, an optional signed refresh, and `price_catalog_version` on every event |
| Response normalisation across providers is a large surface | Scope P1 to text + tools + streaming; keep `raw` passthrough; only add multimodal when it is needed |
| Fallback weakens governance (laxer model, other region) | Data-policy filter runs **before** strategy; content-filter fallback is off by default; `fallback_chain` + `served_region` recorded |
| Switching providers breaks prompt caching | Sticky preference: keep the last-served provider for the same `session_id` when its cost is within ε |
| Budgets are only per-process by default | Document it plainly; ship a Redis `BudgetStore`; recommend the gateway for org-wide hard caps |
| New dependencies bloat the base install | Everything ships behind an `[optimizer]` extra (`tiktoken`, `pyyaml`), with `[optimizer-redis]` and `[optimizer-learned]` on top; the base install stays `httpx`-only |
| Licence mixing | The policy schema and catalog generator live in Apache-2.0 code; the gateway reads the generated JSON and no TS code is copied across (see CONTRIBUTING) |

## 10. Open questions

1. Should `ShieldsRouter` be a new class, or should `ShieldsClient(vendor="auto", providers=...)` gain the capability? The PRD assumes a new class for zero-risk adoption.
2. Should a hosted catalog refresh endpoint be part of Prompt Shields Cloud (paid) or a public static file (free)? The README's free/paid boundary suggests free.
3. Is Gemini in P1, or is Mistral or Bedrock more requested by design partners?
4. What is the baseline for savings on `model="auto"`: a policy-level `baseline_model`, or the most expensive candidate in the chosen group?
5. Should the Atlas sink (`AtlasTelemetrySender`) receive the new fields at the same time?

## Appendix A — Sources

- OpenRouter: provider routing, model fallbacks, `:floor`/`:nitro`, auto router, usage accounting — https://openrouter.ai/docs/features/provider-routing, https://openrouter.ai/docs/guides/routing/model-fallbacks, https://openrouter.ai/docs/guides/routing/routers/auto-router, https://openrouter.ai/docs/cookbook/administration/usage-accounting
- LiteLLM routing & budgets — https://docs.litellm.ai/docs/routing, https://docs.litellm.ai/docs/proxy/users; price map — https://github.com/BerriAI/litellm/blob/main/model_prices_and_context_window.json
- Portkey config object — https://portkey.ai/docs/api-reference/config-object
- RouteLLM — https://github.com/lm-sys/RouteLLM, https://arxiv.org/abs/2406.18665
- NotDiamond — https://docs.notdiamond.ai/docs/key-concepts
- Prompt caching — https://docs.claude.com/en/docs/build-with-claude/prompt-caching, https://developers.openai.com/api/docs/guides/prompt-caching
- Vercel AI Gateway provider options — https://vercel.com/docs/ai-gateway/models-and-providers/provider-options
- Cloudflare AI Gateway — https://developers.cloudflare.com/ai-gateway/features/
- OTel GenAI semantic conventions — https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans

Some vendor details were checked only against secondary sources: OpenRouter fee
percentages and the auto-router backend, RouteLLM per-benchmark splits, and
Portkey conditional-routing syntax. Re-check them before they are quoted externally.
