# Cost Optimization and Token Economics at Scale

**Section:** 04 Production AI Systems (Security, Eval, Scale) → Scaling, Cost & Deployment for AI Systems | **Est. time:** 3 hrs | **Interview relevance:** High — cost awareness is one of the clearest senior signals in an AI system-design interview; a candidate who can write a per-request cost model on the whiteboard and name the levers to pull is instantly differentiated from one who only talks about accuracy.

---

## TL;DR

Unlike classic software, where marginal cost per request rounds to zero, an LLM system's cost is *usage-proportional*: it is driven by **(input tokens + output tokens) × per-token price × number of model calls**, so every extra turn, every re-sent conversation history, and every added agent multiplies the bill. The senior move is to treat cost as a first-class metric — build a **cost-per-request / cost-per-conversation model**, find the dominant driver, then pull the big levers: **right-size and route the model** (small model for easy work, escalate only when needed), **reduce tokens** (trim context, cap `max_tokens`), **cache** repeat work (provider prompt caching + semantic caching), **reduce calls** (bound agent loops — the multi-agent token multiplier is a cost decision, not just an architecture one), and **batch** when latency allows. Wrap it all in **governance**: per-tenant budgets, quotas, spend alerts, and cheaper-model fallback under budget pressure — because uncontrolled inference is also a security risk (OWASP LLM10 Unbounded Consumption, a.k.a. "Denial of Wallet"). **The one thing to remember: LLM cost = tokens × price × calls, so you optimize by shrinking each of those three factors — and every optimization trades off against quality and latency, so you tune the triangle, you don't "solve" it.**

---

## ELI5 — Explain It Like I'm 5

Imagine you hire an assistant who charges you by the word — every word you say *to* them and every word they say *back*, on every single message. Now imagine that on each new message they insist on re-reading the entire conversation from the beginning out loud before answering, and that for hard questions they phone a whole panel of expert friends who each also charge by the word. If you just "always use the most expensive genius assistant and let them ramble," your bill explodes — not because any one answer is dear, but because you're paying by the word, many times over, on every turn. So you get smart: you send short trips to a cheap assistant and only escalate the genius for the truly hard ones, you keep your messages tight and tell them "answer in two sentences," you keep a notepad of answers they've already given so you don't pay to re-derive them (that's caching), and you put a monthly spending cap on the whole arrangement. The misconception to kill is that "the model call is basically free, just use the biggest model" — the call is *never* free, it scales with words and repetitions, and the biggest model is the priciest word-for-word.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain the token-based unit economics of LLM systems (cost = tokens × price × calls) and why it scales with usage in a way traditional software does not.
- [ ] Build a cost-per-request and cost-per-conversation model from token counts and per-token prices, and identify the dominant cost driver.
- [ ] Apply the major cost levers — model routing/cascades, token reduction, prompt & semantic caching, call reduction, and batching — and justify each against the cost/quality/latency trade-off.
- [ ] Design cost governance: per-tenant budgets and quotas, spend alerts, cost-as-a-metric, and cheaper-model fallback under budget pressure.
- [ ] Diagnose an unbounded-consumption cost risk (OWASP LLM10 / "Denial of Wallet") and bound it with iteration caps and per-request spend guards.

---

## Visual Overview

### The Cost Equation — Where Every Dollar Comes From

```
COST per model call = (input_tokens × input_price) + (output_tokens × output_price)

COST per request     = Σ (cost per call)  over all calls in the request
COST per conversation = Σ (cost per request) over all turns

        ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
        │   TOKENS     │ × │    PRICE     │ × │    CALLS     │
        │ in + out     │   │ per model    │   │ per request  │
        └──────────────┘   └──────────────┘   └──────────────┘
              │                   │                   │
        lever: trim/cap    lever: right-size    lever: bound loops
        context, shorten   / route to a         / fewer agents /
        prompts, cap out   cheaper model        cache to skip calls
```

### Model Cascade / Routing Decision

```
incoming request
      │
      ▼
 ┌─────────────────────────────┐
 │ classify difficulty / try   │
 │ CHEAP SMALL model first     │
 └─────────────────────────────┘
      │
 confident & good enough?
      ├── YES ──► return cheap answer            (most traffic, low $)
      │
      └── NO  ──► ESCALATE to LARGE model        (minority, high $)
                       │
                  still failing / high-stakes?
                       └── route to human / refuse
```

### Where Caching Cuts Cost in the Pipeline

```
WITHOUT caching                     WITH caching
────────────────                    ─────────────
request ─► [pay for full            request ─► semantic cache?
           system prompt +                     ├─ HIT ─► return stored answer  ($0 model)
           history every turn]                 └─ MISS ─► model call, BUT
        ─► model call ($$$)                              prompt-cache hits the
                                                          static prefix (system
each turn re-pays for the                                 prompt/tools) at the
same static prefix                                        discounted cached-input rate
```

### The Cost / Quality / Latency Trade-off Triangle

```
                 QUALITY
                   /\
                  /  \
                 /    \
                /      \   pick a point inside; you cannot
               /        \  max all three at once
              /          \
             /____________\
          COST          LATENCY

  bigger model  ──► +quality  −cost  ±latency
  smaller model ──► −cost     −quality
  caching       ──► −cost     −latency   (quality neutral on hits)
  more agents/loops ──► ±quality  −−cost  −−latency
  batching      ──► −cost     ++latency (async only)
```

---

## Key Concepts

### Token-Based Unit Economics & the Cost Model

**What it is.** The unit economics of an LLM system are the rules that map usage to dollars. The atomic billing unit is the **token** (a sub-word chunk), and the invoice is a function of input tokens, output tokens, the per-token price of the chosen model, and how many model calls you make.

**How it works mechanistically.** Providers bill per token in two buckets at *different* prices — input (prompt) tokens are cheaper, output (completion) tokens are typically several times more expensive — so a verbose answer costs more per token than the prompt that produced it. Crucially, LLM APIs are *stateless*: the model has no memory between calls, so to continue a conversation you must **re-send the entire history as input tokens on every turn**. That makes cost grow *super-linearly* with conversation length (turn 10 re-pays for turns 1–9), and it explains why the naive "just call the model" architecture is nothing like a classic web request whose marginal cost rounds to zero. To model it, you estimate average input/output tokens per call, multiply by the model's per-token prices, multiply by expected calls per request, and roll up to cost-per-conversation. *(Illustrative assumption for a whiteboard: if a model costs \$3 / 1M input tokens and \$15 / 1M output tokens — a labeled assumption, not a current quoted price — a call with 4,000 input + 500 output tokens costs 4000×3/1e6 + 500×15/1e6 ≈ \$0.012 + \$0.0075 ≈ \$0.02; ten such turns re-sending growing history can easily be 3–5× that.)*

**Where it appears in real systems.** Every provider response returns a `usage` object you log as a metric: OpenAI's Chat/Responses API returns `usage.prompt_tokens` / `completion_tokens` (with `prompt_tokens_details.cached_tokens` for cache reads); Anthropic returns `usage.input_tokens`, `output_tokens`, `cache_read_input_tokens`, and `cache_creation_input_tokens`. You can price a prompt *before* sending it with Anthropic's `messages/count_tokens` endpoint (free) or a local tokenizer, which is how routing and budget-guard decisions are made. **Note:** different model families tokenize differently — Anthropic documents that its Claude 4.7+ tokenizer produces ~30% more tokens for the same text than earlier models — so token counts (and therefore cost) must be re-measured per model, never copied across families.

### Model Right-Sizing, Routing & Cascades

**What it is.** Right-sizing is choosing the *smallest* model that meets the quality bar for a given request; **routing / cascading** is doing this dynamically — send easy requests to a cheap small model and **escalate** to a large expensive model only when the cheap one is insufficient.

**How it works mechanistically.** A router sits in front of the models. It either (a) classifies request difficulty up front (a tiny classifier or heuristic on the query) and picks a tier, or (b) runs a **cascade**: try the cheap model first, evaluate its answer with a confidence signal (self-reported confidence, a verifier model, a schema/validation check, or log-probs), and only fall through to the bigger model on low confidence. Because real traffic is dominated by easy requests, routing most of them to a model that can be an order of magnitude cheaper per token slashes blended cost while a minority of hard requests still get the frontier model. The trade-off: escalation adds a *second* call's tokens on the fraction that fails the cheap tier, so a cascade only wins when the cheap tier resolves a large majority of traffic.

**Where it appears in real systems.** Provider model line-ups are explicitly tiered for this (e.g. OpenAI's `nano`/`mini`/full tiers, Anthropic's Haiku/Sonnet/Opus) — the pricing pages show the smaller tiers costing a fraction of the flagship per token. In LangGraph you implement routing as a conditional edge that inspects a difficulty score and directs to different model nodes; a cascade is a node that calls the small model, checks a confidence field, and re-routes to the large-model node on failure.

### Token Reduction (Context Trimming, Prompt Compression, Output Caps)

**What it is.** Techniques that lower the token count on each call: trimming/summarizing conversation history, compressing or shortening the system prompt and few-shot examples, pruning retrieved RAG chunks, and capping the maximum output length.

**How it works mechanistically.** Because you re-send history every turn, the biggest lever is usually **context management**: keep a sliding window of recent turns plus a running summary of older ones (compaction) instead of the full transcript, so input tokens stay bounded as the conversation grows. On the input side, shorter system prompts and fewer/tighter few-shot examples directly cut the fixed per-call overhead. On the output side, setting `max_tokens` caps the *most expensive* token bucket and, just as importantly, defends against a runaway generation that bills a huge completion. Over-trimming is the risk: cut too much context and quality drops (the model loses the fact it needed), so token reduction is a quality trade-off, not free money.

**Where it appears in real systems.** `max_tokens` (Anthropic) / `max_output_tokens` / `max_completion_tokens` (OpenAI) is a required or recommended field on every request. History compaction appears as a summarization step in a LangGraph pipeline or an agent's context-management middleware (ties to Section 03's context compaction). RAG systems tune retrieval `top_k` and rerank-then-truncate specifically to keep the injected context — and its token cost — under control.

### Prompt Caching (Provider Caching)

**What it is.** Provider-side caching of a *repeated prompt prefix* so that on a cache hit you pay a steeply discounted "cached input" rate for the reused tokens instead of the full input rate.

**How it works mechanistically.** Caching works on **exact prefix matches**: the provider hashes the leading portion of your prompt and, if a recent request shares that prefix, reuses the already-computed key/value tensors, billing those tokens at the cached rate and skipping the recompute. This is why you structure prompts with **static content first** (system prompt, tool definitions, long shared context) and variable content (the user's latest message) last — any change before the cached breakpoint invalidates the cache. Anthropic requires an explicit `cache_control` breakpoint (5-minute default TTL, 1-hour at higher write cost) and bills cache *reads* at ~10% of base input and 5-min cache *writes* at ~1.25× base input; OpenAI applies caching automatically for prompts ≥1024 tokens and reports `cached_tokens`. The economic rule: caching pays off when the same long prefix is reused often within the TTL, and can *cost more* if you cache a prefix that changes every request (you pay write premiums and never get reads).

**Where it appears in real systems.** Anthropic's `cache_control: {"type": "ephemeral"}` on a system/tools block, surfaced in `usage.cache_read_input_tokens`; OpenAI's automatic prefix caching plus the optional `prompt_cache_key` to improve routing/hit-rate, surfaced in `usage.prompt_tokens_details.cached_tokens`. Both are the default cost optimization for multi-turn chat and agentic tool-use loops, where the same system prompt + tool schemas ride along on every call.

### Semantic Caching

**What it is.** An application-level cache keyed on the *meaning* of a query (via embedding similarity) rather than an exact string match, so that a *paraphrase* of a previously answered question returns the stored answer with **no model call at all**.

**How it works mechanistically.** You embed the incoming query, do a nearest-neighbor search against a vector store of prior (query embedding → answer) pairs, and if the closest match exceeds a similarity threshold you return the cached answer, skipping the LLM entirely (a full-cost avoidance, not just a discount like prompt caching). The threshold is the critical knob: too loose and you serve a stale/wrong answer to a subtly different question (a correctness bug); too tight and hit-rate collapses. Semantic caching suits high-repetition, stable-answer workloads (FAQs, docs Q&A) and is dangerous for personalized or time-sensitive answers.

**Where it appears in real systems.** Implemented in front of the model as a vector-DB lookup (e.g. an embeddings call + similarity search) with a TTL and per-tenant namespacing; it's the layer that turns "1,000 people asked the same onboarding question" into one model call plus 999 cache hits.

### Reducing Calls — the Agent-Loop & Multi-Agent Token Multiplier

**What it is.** The recognition that *number of model calls* is a first-class cost driver, and that agentic patterns multiply it: each reflection/replan step is another call, and each sub-agent in a multi-agent system runs its own loop with its own context.

**How it works mechanistically.** A single-agent ReAct loop already makes one model call per reasoning step; add self-reflection or replanning and you add calls per turn. Multi-agent systems multiply further — an orchestrator plus N sub-agents, each re-reading its own (often overlapping) context, can burn many times the tokens of a single agent for the same task. Anthropic's own reporting on multi-agent research systems notes agents can use on the order of *4×* the tokens of a chat interaction and multi-agent systems *~15×*, which is precisely why "should this be one agent or five?" is a **cost** question as much as an architecture one (ties directly to Section 03). The lever is to bound loops (max iterations), prefer a single agent when it suffices, and reserve multi-agent for tasks whose value justifies the token multiplier.

**Where it appears in real systems.** LangGraph's `recursion_limit` (and `RemainingSteps`) caps the loop; an explicit `MAX_ITERS` in a hand-rolled loop does the same. Architecturally, the decision surfaces as "single tool-calling agent vs. orchestrator-worker multi-agent," where the cheaper single agent is the default unless roles/context-isolation genuinely require the multiplier.

### Batching

**What it is.** Grouping many independent requests into one asynchronous job that the provider processes within a longer window (e.g. 24h) in exchange for a substantial per-token discount.

**How it works mechanistically.** You submit a file of requests to a batch endpoint; the provider runs them off-peak against spare capacity and returns results within the completion window, billing at a reduced rate (OpenAI documents a **50% discount** vs. synchronous for its Batch API, with a separate higher rate-limit pool; Anthropic offers a batch discount that stacks with caching). The trade-off is explicit in the triangle: you sacrifice latency (results are not immediate) for cost, so batching fits *offline* workloads — bulk classification, embedding a corpus, nightly evals, backfills — and never fits an interactive chat turn.

**Where it appears in real systems.** OpenAI's `/v1/batches` (JSONL of requests, `completion_window: "24h"`, results keyed by `custom_id`) and Anthropic's Message Batches API. It's the standard cost play for any large, non-interactive job.

### Cost Governance (Budgets, Quotas, Alerts, Cost-as-Metric, Fallback)

**What it is.** The operational controls that keep spend predictable and bounded: per-user/per-tenant budgets and quotas, spend alerts, treating cost as a first-class observable metric, and automatic fallback to a cheaper model under budget pressure.

**How it works mechanistically.** You attribute every model call's token cost to a tenant/user (from the `usage` object), accumulate it against a period budget, and enforce: soft **alerts** at thresholds (e.g. 80% of monthly budget), hard **quotas / rate limits** that reject or queue further requests, and **graceful degradation** — when a tenant is near budget you route them to a cheaper model or a cached/canned path rather than failing outright. Because cost data lives in your traces/metrics (ties to this section's observability chapter), you can alert on cost-per-request regressions the same way you alert on latency. This directly mitigates **OWASP LLM10 Unbounded Consumption** ("Denial of Wallet"), where an attacker (or a buggy loop) drives excessive inference to run up your bill.

**Where it appears in real systems.** A middleware that reads `usage`, increments a per-tenant counter in Redis/Postgres, and enforces limits; alerting rules in your observability stack on a `cost_usd` metric tagged by tenant and model; a routing rule that swaps the model based on remaining budget. OWASP's LLM10 prevention list explicitly names rate limiting, per-user quotas, resource-allocation management, and comprehensive monitoring as the controls.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| `max_tokens` / `max_output_tokens` | Hard cap on generated (most expensive) tokens per call | Set to the smallest value that fits a *complete* good answer for the task (e.g. 300–800 for chat replies); never leave it unbounded — it's both a cost cap and a runaway-generation guard. |
| Routing / escalation threshold | Confidence level below which a cascade escalates to the bigger model | Tune so the cheap tier resolves the large majority of traffic (target ~70–90% hit); if escalation fires on most requests you've mis-sized the cheap model and the cascade *adds* cost. |
| Prompt-cache breakpoint placement | Which prefix is cached (static prefix vs. varying suffix) | Put the breakpoint on the *last block that stays identical across requests* (system prompt/tools); never cache a prefix containing per-request data (timestamps, user msg) — you'd pay writes and never read. |
| Cache TTL (5m vs 1h) / hit-rate target | How long a cached prefix survives and your target reuse | Use the default short TTL for steady high-frequency traffic (refreshes free on hit); pay for the longer TTL only when reuse is less frequent than the short window but latency/cost still matter. |
| Semantic-cache similarity threshold | How close a query must be to reuse a cached answer | Set high (strict) for anything where a wrong-but-similar answer is harmful; only loosen for stable, generic FAQ-style content — measure served-answer correctness, not just hit-rate. |
| Max agent-loop iterations (`recursion_limit`) | Upper bound on model calls per request in an agent loop | Set to the smallest value covering your worst *legitimate* multi-tool path (often 6–15); this makes worst-case cost deterministic and is the primary Unbounded-Consumption guard. |
| Batch size / window | How many requests per batch job and the completion window | Batch only latency-tolerant work; size to the provider's per-batch limits and accept the (up to 24h) window in exchange for the ~50% discount. |
| Per-tenant budget / quota | Spend or request ceiling per user/tenant per period | Derive from the tenant's plan/price; set a soft alert well below the hard cap so you can intervene (throttle or downgrade model) before rejecting traffic. |

### Worked Example: Requirement → Decision

**Given:** You run a B2B support-assistant agent. Each customer conversation averages ~8 turns; every turn re-sends the growing history plus a 1,500-token system prompt and tool schemas, and the agent currently always calls your largest model with `max_tokens` left unset, sometimes reflecting 2–3 times per turn. Finance reports the blended **cost-per-conversation is ~\$0.90** (illustrative assumption) and it's wrecking gross margin on your cheaper plan tiers. Target: cut cost-per-conversation by ~60% without a noticeable quality drop, latency budget ~5 s per turn (interactive), multi-tenant.

- **Step 1 — Identify the goal:** Reduce cost-per-conversation ~60% while holding answer quality and staying interactive (so batching is out).
- **Step 2 — Define inputs:** Per-call token counts (log the `usage` object), per-token prices per candidate model (from the provider pricing page — treat all figures as assumptions), turns/conversation, reflection calls/turn, and the static-prefix size.
- **Step 3 — Define outputs:** A cost model attributing spend to each driver (price tier, re-sent history, reflection calls, uncapped output), and a ranked lever list.
- **Step 4 — Apply constraints:** Interactive latency (≤5 s → no batch, cascade adds at most one extra call), quality bar (can't over-trim context), multi-tenant (must attribute cost + enforce budgets), and cost must be *deterministically bounded* (no Denial-of-Wallet).
- **Step 5 — Select the approach:** The cost model shows the dominant drivers are (a) always-largest-model, (b) re-sent 1,500-token static prefix every call, (c) 2–3 uncapped reflection calls/turn. Pull, in order: **(1) prompt caching** on the static system-prompt+tools prefix (cache reads at ~10% of input on hits — near-free reuse), **(2) model routing/cascade** so easy turns use a small model and only hard turns escalate, **(3) cap `max_tokens`** and **bound reflection** to ≤1 replan with a `recursion_limit`, **(4) history compaction** (sliding window + summary) so input tokens stop growing, and **(5) per-tenant budgets + spend alerts** for governance. *Rationale vs alternatives:* batching is rejected (interactive); a blanket "downgrade everyone to the small model" is rejected because it fails the quality bar on hard turns — the cascade preserves quality where it matters while capturing the savings on the easy majority; caching + call-bounding attack the two multipliers (re-sent prefix, extra calls) that a model swap alone wouldn't fix.

---

## Implementation

```python
# Scenario: Finance needs a defensible cost-per-conversation number before we
# optimize. We compute per-call cost from the provider's returned usage object
# (input/output/cached token buckets priced separately) and roll it up to a
# conversation. Prices here are LABELED ASSUMPTIONS for illustration, not a
# current quote — always read live per-token prices off the provider pricing page.
from dataclasses import dataclass

# --- ASSUMED prices ($ per 1M tokens) for illustration only ---
PRICES = {
    "small": {"input": 0.25, "cached_input": 0.025, "output": 2.00},   # ASSUMPTION
    "large": {"input": 3.00, "cached_input": 0.30,  "output": 15.00},  # ASSUMPTION
}

@dataclass
class Usage:              # mirrors fields returned by provider `usage` objects
    input_tokens: int
    cached_input_tokens: int   # billed at the discounted cached rate
    output_tokens: int

def call_cost_usd(model: str, u: Usage) -> float:
    p = PRICES[model]
    fresh_input = max(u.input_tokens - u.cached_input_tokens, 0)
    return (
        fresh_input          * p["input"]        / 1_000_000
        + u.cached_input_tokens * p["cached_input"] / 1_000_000
        + u.output_tokens    * p["output"]       / 1_000_000
    )

def conversation_cost_usd(calls: list[tuple[str, Usage]]) -> float:
    # A conversation is many calls; cost is their sum. Re-sent history shows up
    # as growing input_tokens per turn — the super-linear growth to watch.
    return sum(call_cost_usd(model, u) for model, u in calls)

# Example rollup: 8 turns on the large model with an uncached 1,500-tok prefix
turns = [("large", Usage(input_tokens=4000, cached_input_tokens=0, output_tokens=500))
         for _ in range(8)]
print(round(conversation_cost_usd(turns), 4))  # the number you attack with levers
```

```python
# Anti-pattern: the "just use the biggest model" agent — always the flagship,
# NO output cap, re-sends full history every turn, and reflects in an UNBOUNDED
# loop. Cost balloons with conversation length AND a stuck loop is a literal
# Denial-of-Wallet (OWASP LLM10 Unbounded Consumption) — one runaway request can
# bill unboundedly.
def answer_turn(history, tools):                 # BROKEN: uncapped everything
    while True:                                  # unbounded reflection loop
        resp = large_model.invoke(               # always the most expensive model
            messages=history,                    # full transcript re-sent every call
            tools=tools,                         # tool schemas re-sent, never cached
            # max_tokens unset -> output can run away (priciest token bucket)
        )
        history.append(resp)
        if resp.satisfied:                       # may never be satisfied -> spins
            return resp

# Correct approach: cache the static prefix, route cheap-first with escalation,
# cap output, and BOUND the loop + guard per-request spend.
MAX_ITERS = 3
PER_REQUEST_BUDGET_USD = 0.05        # hard spend guard per request (governance)

def answer_turn(history, tools, tenant):
    # 1) System prompt + tool schemas are STATIC -> mark them cacheable so every
    #    call re-reads them at the discounted cached-input rate, not full price.
    system = cacheable(SYSTEM_PROMPT)            # e.g. cache_control ephemeral / stable prefix
    # 2) Trim history to a sliding window + running summary so input stops growing.
    ctx = compact(history, keep_last=6)
    spent = 0.0
    for i in range(MAX_ITERS):                   # bounded calls -> deterministic cost
        model = route(ctx)                       # cheap small model first...
        resp = model.invoke(
            messages=[system, *ctx],
            tools=tools,
            max_tokens=600,                      # cap the expensive output bucket
        )
        spent += call_cost_usd(model.name, resp.usage)
        if spent > PER_REQUEST_BUDGET_USD:       # Denial-of-Wallet guard
            return degrade_to_cached_or_refuse(tenant)
        if resp.satisfied or not resp.tool_calls:
            record_cost(tenant, spent)           # attribute spend per tenant (budgets)
            return resp
        ctx = ctx + [resp]                       # escalate() inside route() on low confidence
    record_cost(tenant, spent)
    return best_effort(ctx)                      # fallback when the bound is hit
# What breaks without the fix: cost grows super-linearly with turns (re-sent
# history + full-price prefix), the flagship is used even for trivial turns, and
# an unsatisfiable reflection loop bills without limit. The corrected version makes
# worst-case cost = MAX_ITERS calls, discounts the repeated prefix via caching,
# spends the flagship only where routing says it's needed, and hard-stops on budget.
```

---

## Common Pitfalls & Misconceptions

- **"The model call is basically free — just use the biggest model."** — Beginners carry over the classic-software intuition that marginal request cost rounds to zero. LLM cost is usage-proportional (tokens × price × calls) and the biggest model is the priciest per token; the correct mental model is a metered taxi that charges by the word, in *and* out, on *every* call.
- **Forgetting that history is re-sent every turn** — People think a "conversation" is one cheap ongoing session, but the API is stateless, so each turn re-bills the entire transcript as input tokens. The correct model is that cost grows super-linearly with conversation length, which is why context compaction is a cost lever, not just a context-window lever.
- **Ignoring the output-token multiplier / leaving `max_tokens` unset** — Output tokens are typically several times pricier than input, and an uncapped generation can run away. Always cap `max_tokens` to the smallest length that fits a complete answer — it's simultaneously your biggest per-call cost lever and a runaway-generation guard.
- **Caching a prefix that changes every request** — Teams enable prompt caching and put the breakpoint after per-request content (a timestamp or the user message), so the prefix hash never matches; they pay cache-*write* premiums and get zero reads. The correct model: cache only the *stable* prefix (system prompt/tools) and keep variable content strictly after the breakpoint.
- **Setting the semantic-cache threshold too loose** — To boost hit-rate people relax the similarity threshold, then serve a cached answer to a subtly *different* question — a silent correctness bug, not just a cost quirk. Treat the threshold as a correctness dial: measure served-answer accuracy, and keep it strict for anything personalized or time-sensitive.
- **Treating multi-agent as free architecture** — "Agentic" sounds like it should mean many agents, and cost is invisible at design time. Every extra agent and reflection step is more calls re-reading overlapping context (reported multipliers of ~4× for agents, ~15× for multi-agent), so agent count is a cost decision — default to a single bounded agent and justify the multiplier before paying it.
- **No per-request/per-tenant spend bound (Denial of Wallet)** — Beginners bound *correctness* but not *spend*, assuming loops terminate. An attacker or a stuck loop can drive unbounded inference (OWASP LLM10); always pair an iteration cap with a per-request spend guard and per-tenant budgets/quotas.

---

## Key Definitions

| Term | Definition |
|---|---|
| Token | The atomic sub-word unit LLM providers bill by; both input (prompt) and output (completion) tokens are counted, at different prices. |
| Cost model (per-request / per-conversation) | An estimate of dollar cost built as (input+output tokens) × per-token price × number of calls, rolled up across a request or a whole conversation. |
| Input vs. output token price | The separate per-token rates for prompt tokens vs. generated tokens; output is typically several times more expensive per token. |
| Model routing / cascade | Dynamically sending a request to a cheap small model first and escalating to a larger model only when the cheap one is insufficient. |
| Right-sizing | Choosing the smallest model that clears the quality bar for a given request or workload. |
| Context compaction / trimming | Reducing re-sent history via a sliding window and/or running summary so input tokens stay bounded as a conversation grows. |
| `max_tokens` | The per-call cap on generated tokens; a primary cost lever and a runaway-generation guard. |
| Prompt (provider) caching | Provider-side reuse of a repeated prompt *prefix*, billing the reused tokens at a discounted cached-input rate on a hit. |
| Semantic caching | Application-level reuse of a stored answer when an incoming query is *semantically* similar (by embedding) to a prior one, avoiding a model call entirely. |
| Token multiplier | The increase in total calls/tokens from agentic loops and multi-agent designs (each reflection step and each sub-agent adds calls). |
| Batching | Submitting many requests asynchronously for a per-token discount in exchange for a longer completion window; latency-tolerant workloads only. |
| Cost governance | Budgets, quotas, spend alerts, cost-as-a-metric, and cheaper-model fallback that keep spend predictable and bounded. |
| Unbounded Consumption (OWASP LLM10) | The risk that uncontrolled inference (attack or bug) drives excessive resource/cost use — "Denial of Wallet"; mitigated by rate limits, quotas, and monitoring. |

---

## Summary / Quick Recall

- LLM cost = **tokens (in+out) × per-token price × number of calls**; it scales with usage, unlike classic software whose marginal cost rounds to zero.
- The API is **stateless**, so history is re-sent every turn → cost grows **super-linearly** with conversation length; log the `usage` object and build a per-conversation cost model.
- Big levers: **right-size/route** (cheap-first, escalate), **reduce tokens** (compact context, cap `max_tokens`), **cache** (prompt caching discounts a repeated prefix; semantic caching skips the call entirely), **reduce calls** (bound agent loops; single-agent when it suffices), and **batch** offline work (~50% off, latency-tolerant only).
- The **multi-agent/agent-loop token multiplier** (~4× agent, ~15× multi-agent by reported figures) makes "how many agents / how many reflections" a **cost** decision — bound loops and default to a single agent.
- **Governance** makes cost first-class: per-tenant budgets/quotas, spend alerts, cost-as-a-metric (ties to observability), and cheaper-model fallback under budget pressure.
- Every optimization moves you around the **cost/quality/latency triangle** — you tune it, you don't max all three; over-trimming context or a too-loose semantic-cache threshold trades cost for *quality/correctness*.
- Unbounded inference is also a **security risk** (OWASP LLM10 "Denial of Wallet"): always pair an iteration cap with a per-request spend guard.

---

## Self-Check Questions

1. State the formula for the cost of a single LLM model call and explain why an LLM system's cost scales with usage in a way a classic CRUD web service's does not.

   <details><summary>Answer</summary>

   Cost per call = (input_tokens × input_price) + (output_tokens × output_price); a request/conversation is the sum over all calls. It scales with usage because you pay **per token** on every call, output tokens cost more than input, and — since the API is **stateless** — you re-send the whole conversation history as input tokens each turn, so cost grows super-linearly with length. A classic CRUD service's marginal cost per request rounds to zero (fixed servers, cheap DB reads), so its cost is dominated by capacity, not by per-request work. The tempting wrong answer is "cost = number of requests × a flat price" — that ignores that the same request can cost 10× more depending on token volume and how many model calls it triggers.

   </details>

2. Your interactive chat agent always calls the flagship model, re-sends a 2,000-token system prompt + tool schemas on every turn, and leaves `max_tokens` unset. Name three concrete levers you would pull *first* and why each attacks the cost equation.

   <details><summary>Answer</summary>

   (1) **Prompt caching** on the static system-prompt+tools prefix — it's re-sent identically every turn, so caching bills those tokens at the ~10%-of-input cached rate on hits, attacking the *tokens × price* term for the repeated prefix. (2) **Cap `max_tokens`** — output tokens are the most expensive bucket and an uncapped generation can run away, so a tight cap cuts per-call cost and guards against runaway generations. (3) **Model routing/cascade** — send easy turns to a cheap small model and escalate only when needed, attacking the *price* term for the majority of traffic. The tempting wrong answer is "just switch everyone to the small model," which fails the quality bar on hard turns; the cascade preserves quality where it matters while still capturing the savings.

   </details>

3. **Which TWO** of the following correctly describe the difference between provider **prompt caching** and application-level **semantic caching**?
   - A. Prompt caching discounts a repeated exact-prefix's tokens but you still make a model call; semantic caching can skip the model call entirely.
   - B. Semantic caching requires an exact string match, while prompt caching matches on meaning via embeddings.
   - C. Semantic caching keys on embedding similarity, so a paraphrased query can hit the cache; prompt caching requires an identical leading prefix.
   - D. Prompt caching guarantees a bit-for-bit identical response on a hit, so it improves answer determinism.
   - E. Both eliminate all model calls and therefore have no correctness risk.

   <details><summary>Answer</summary>

   **A and C.** A is correct: a prompt-cache hit still runs the model (it just bills the reused prefix at the discounted cached rate), whereas a semantic-cache hit returns a stored answer with *no* model call. C is correct: semantic caching does nearest-neighbor search on query embeddings (paraphrases can hit), while prompt caching requires an identical prompt prefix. B is backwards (it swaps the two mechanisms). D is wrong — providers document that prompt caching does **not** change output generation or guarantee identical responses. E is wrong — semantic caching in particular carries a correctness risk if the similarity threshold is too loose, serving a cached answer to a subtly different question.

   </details>

4. A teammate proposes moving a nightly job that classifies 500,000 support tickets from synchronous flagship calls to the provider's Batch API, and *also* wants to use the same Batch API for the live chat widget's replies. Evaluate both proposals against the cost/quality/latency trade-off.

   <details><summary>Answer</summary>

   The nightly classification job is an **ideal batching candidate**: it's offline and latency-tolerant, so trading immediacy for the provider's batch discount (OpenAI documents ~50% off, plus a separate higher rate-limit pool and up to a 24h window) is pure win with no quality change. The live chat widget is a **wrong** use of batching: batch results aren't returned immediately (up to a 24h window), which violates an interactive latency budget — you'd cut cost but destroy the user experience. This is the triangle in action: batching buys cost at the expense of latency, so it fits async workloads only. The tempting error is "batching is cheaper, so use it everywhere"; the constraint that gates it is whether the workload can tolerate the completion-window latency.

   </details>

5. Your single most expensive metric spike traces to a handful of conversations that ran a reflection loop dozens of times on the flagship model, and a security review flags "Denial of Wallet." Explain the root cause in cost-economics terms and the fix, and why simply switching to a cheaper model is insufficient.

   <details><summary>Answer</summary>

   Root cause: the agent loop is effectively **unbounded**, so *number of calls* — the third factor in tokens×price×calls — exploded on those conversations; each reflection re-read the growing context and made another flagship call, and an unsatisfiable loop kept going, which is exactly OWASP LLM10 **Unbounded Consumption / Denial of Wallet**. The fix is to **bound the loop** (a small `recursion_limit`/`MAX_ITERS`) *and* add a **per-request spend guard** plus per-tenant budgets/quotas and alerts, making worst-case cost deterministic. Switching to a cheaper model is insufficient because it only shrinks the *price* factor while leaving the *calls* factor unbounded — a cheap model called hundreds of times in a runaway loop still runs up an unbounded bill; you must cap the calls, not just the per-call price.

   </details>

---

## Further Reading

- [OpenAI — Pricing (per-token input/output/cached rates by model)](https://platform.openai.com/docs/pricing) — *verified 2026-07-29* — The authoritative source for current per-token prices, batch and cached-input rates, and tool pricing; cite this page rather than hardcoding numbers, since prices change.
- [OpenAI — Prompt Caching guide](https://platform.openai.com/docs/guides/prompt-caching) — *verified 2026-07-29* — How automatic prefix caching works, the ≥1024-token threshold, `prompt_cache_key`, breakpoints/TTL, and the `cached_tokens`/`cache_write_tokens` usage fields.
- [OpenAI — Batch API guide](https://platform.openai.com/docs/guides/batch) — *verified 2026-07-29* — The ~50% batch discount, separate rate-limit pool, 24-hour completion window, and the JSONL request/`custom_id` workflow for latency-tolerant jobs.
- [Anthropic — Prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) — *verified 2026-07-29* — `cache_control` breakpoints, 5-minute vs 1-hour TTL, cache-write (~1.25×) vs cache-read (~0.1×) pricing multipliers, and prefix-invalidation rules.
- [Anthropic — Token counting](https://docs.anthropic.com/en/docs/build-with-claude/token-counting) — *verified 2026-07-29* — The free `messages/count_tokens` endpoint for pricing prompts before sending, plus the ~30% tokenizer difference across model families that forces per-model cost re-measurement.
- [OWASP — LLM10:2025 Unbounded Consumption](https://genai.owasp.org/llmrisk/llm102025-unbounded-consumption/) — *verified 2026-07-29* — Defines the "Denial of Wallet" cost risk and prescribes rate limiting, per-user quotas, resource-allocation management, and monitoring as mitigations.
