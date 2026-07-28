# Cost & Token-Efficiency Patterns for AI System Design

**Section:** 05 Interview Practice → Cross-Cutting Concerns & Trade-off Patterns | **Interview relevance:** High — "how do you keep this affordable at scale?" is the second-most-common follow-up after latency, and it forces you to reason about token economics (input vs output, cache reads, batch discounts), model tiering, and per-tenant cost observability under a stated budget.

---

## TL;DR

Cost in an LLM system is driven by tokens (input and output priced separately, output typically ~4–5× input), by the model tier you route to, and by how many model calls a single user request fans out into — agent loops and multi-source retrieval multiply spend silently. You reduce it two ways: make each call cheaper (prompt caching for stable prefixes, smaller/routed models, output-length caps, retrieval-k tuning) or avoid calls entirely (semantic caching on hits, the Batch API for async work at 50% off). Every lever trades against latency or quality, so you choose against an explicit cost-per-request or cost-per-tenant budget. **The one thing to remember: output tokens are the expensive tokens and agent fan-out is the silent multiplier — cap output, route to the smallest model that passes eval, and cache the stable prefix before you optimize anything else.**

---

## ELI5 — Explain It Like I'm 5

Imagine a taxi company where you pay by the word for every conversation with a driver — and talking (the answer) costs four times as much as listening (your question). A naive dispatcher sends the most expensive luxury car for every single trip, lets drivers ramble with no word limit, and for a big errand sends five cars circling the block re-asking each other the same directions. A smart dispatcher sends a cheap compact for easy trips and only calls the luxury car when the route is genuinely hard, tells drivers to answer in as few words as the job needs, keeps a notebook of common answers so it never dispatches a car for a question it already knows, and batches all the non-urgent errands into one overnight discount run. The common mistake is thinking the bill comes from *how many trips* — it actually comes from *how much talking* and from cars you dispatched that you never needed.

---

## Learning Objectives

By the end of this note you will be able to:
- [ ] Decompose an AI request's cost into input tokens, output tokens, cached-read tokens, model tier, and per-request call fan-out, and identify which dominates the bill.
- [ ] Select the right cost-reduction pattern (prompt caching, model routing/tiering, semantic caching, Batch API, retrieval-k tuning, output capping, distillation) for a stated spend problem.
- [ ] Justify each choice against its trade-off (latency, quality/eval regression, staleness, engineering complexity) under a cost-per-request or cost-per-tenant budget.
- [ ] Design cost observability that attributes token spend to a request and a tenant so runaway cost is caught before the invoice.

---

## Visual Overview

### Where the money goes (one agent request)

```
User request
   │
   ▼
Router (cheap model / classifier)  ~small input+output
   │
   ├──► Retrieval (k chunks) ─────► inflates INPUT tokens of every downstream call
   │
   ▼
Agent loop ── turn 1 ──► turn 2 ──► turn 3 ...   ◄── FAN-OUT: each turn = a full call
   │            (input reprocessed each turn unless cached)
   ▼
Final generation ──► OUTPUT tokens  ◄── priced ~4–5× input; scales with max_tokens
   │
   ▼
Bill = Σ over calls ( input×in_rate + cached×cache_rate + output×out_rate )
```

### Model-routing / tiering decision tree

```
Incoming request
├── Cheap classifier / heuristic: how hard is it?
│
├── Trivial / FAQ ──► semantic cache HIT? ──► return cached answer (≈ $0, no model call)
│                                     └─ MISS ─┐
│                                              ▼
├── Easy ─────────────► SMALL model (Haiku-tier)      ← default path, majority of traffic
│
├── Hard / reasoning ─► LARGE model (Opus-tier)       ← escalate only when needed
│
└── Async / no human waiting ─► Batch API queue (−50%) ← nightly evals, bulk classification
```

---

## The Core Problem

An LLM bills per token, and the two token classes are priced asymmetrically: output tokens cost roughly 4–5× input tokens on frontier models (e.g. an Opus-tier model at `$5 / input MTok` vs `$25 / output MTok`, per Anthropic's models overview). So the same request that "feels" cheap because the prompt is short can be expensive if it generates a long answer. On top of per-token price, three multipliers inflate the bill: the **model tier** you pick (a large model can be ~5–10× a small one for identical tokens), the **retrieved context** you stuff into the prompt (every chunk is input tokens paid on *every* call that carries it), and **call fan-out** — an agent loop or multi-tool plan turns one user request into N model calls, and a naive design reprocesses the whole prompt on every turn.

The interview question is rarely "is it expensive" — it is "where is the money going, and which reduction is worth its trade-off?" Three cost components must be separated, because different levers move them:

- **Input / prefill cost** — driven by prompt length: system prompt, retrieved-k, conversation history. Lowered by prompt caching (stable prefixes read at ~10% of input rate) and by retrieval-k / history trimming.
- **Output cost** — driven by how many tokens you let the model generate. Lowered by `max_tokens` caps and prompting for concision. This is usually the largest per-call line item.
- **Call-count cost** — driven by fan-out: agent turns, retries, multi-source calls. Lowered by bounding the loop, routing to a small model, or eliminating the call entirely via a semantic cache hit.

A design that trims the prompt but leaves `max_tokens` unbounded and routes everything to the largest model will still burn budget; a design that caps output, defaults to a small model, and caches stable prefixes can cut cost by an order of magnitude at similar quality.

---

## Resolution Options

| Option | How it works | Buys you | Costs you | Reach for it when… |
|---|---|---|---|---|
| **Prompt caching** | Provider caches a stable prompt prefix; reads bill at ~10% of input rate (Anthropic: `cache_control`; OpenAI: automatic + `prompt_cache_key`) | Large input-cost + TTFT cut when a long prefix repeats | Cache-write surcharge (Anthropic 5m write = 1.25× input); prefix must stay byte-identical; short TTL | A long, stable system prompt / doc / tool set is reused across calls |
| **Model routing / tiering** | Classify each request; send easy ones to a small model, escalate hard ones to a large one | Majority path runs at a fraction of large-model cost | Router-error risk; a misroute degrades quality; extra classifier call | Mixed workload where most requests are easy |
| **Semantic caching** | Embed the query; on a near-duplicate hit, return the stored answer with no model call | Near-zero cost *and* latency on hits | Staleness; false hits from a loose threshold; embedding cost per lookup | High query repetition (FAQs, common intents) |
| **Batch API** | Submit async requests as a file; results within 24h at 50% off (OpenAI & Anthropic) | Half price on all tokens for non-interactive work | Not for interactive use (up to 24h turnaround); no streaming | Offline evals, bulk classification, embeddings, backfills |
| **Retrieval-k tuning** | Reduce how many chunks enter the prompt | Lower input tokens on every downstream call | Recall loss if k is too low → quality drop | Retrieved context dominates input cost |
| **Output-length capping** | Set `max_tokens` to the shortest length the task needs; prompt for concision | Directly cuts the most expensive token class | Truncated answers if capped too tight | Answers run longer than the task requires |
| **Distillation / fine-tuning a small model** | Collect large-model outputs, fine-tune a small model to match on-task | Large-model-ish quality at small-model price & latency | Upfront labeling/training cost; narrower scope; retraining as data drifts | High-volume, narrow task where prompting a large model is the cost driver |

**Prompt caching** — providers detect a repeated prompt *prefix* and reuse the already-computed key/value tensors instead of re-processing those tokens, billing cache reads at a fraction of the input rate (Anthropic prices cache hits at ~10% of base input; OpenAI reports reads in `cached_tokens`). Mechanically, Anthropic uses explicit `cache_control` breakpoints (or automatic caching) on the stable prefix; OpenAI caches automatically for prompts ≥1024 tokens and lets you set `prompt_cache_key` to improve hit routing. It appears as a cache-control marker on the system prompt / tool definitions / long document, with the *variable* content (the user turn) placed last so the prefix stays identical. The hard constraint: any change before the breakpoint invalidates the cache, and Anthropic's 5-minute write costs 1.25× the input rate — so caching a prefix you only hit once is a net loss.

**Model routing / tiering** — a lightweight classifier or heuristic inspects each request and dispatches it to the smallest model that can handle it, escalating only hard cases (OpenAI's model-selection guide frames this as "optimize for accuracy first, then maintain that accuracy with the cheapest, fastest model"). Mechanically this is a routing node in front of generation; it appears as the `model` field being chosen per-request rather than hard-coded. The trade-off is router accuracy: a misroute sends a hard query to a weak model and degrades quality — so the router's decision must itself be evaluated (links to `03-evaluation-and-quality-assurance-patterns.md`).

**Semantic caching** — embed the incoming query, nearest-neighbour it against a store of prior `(query, response)` pairs, and return the cached response when similarity exceeds a threshold, skipping the model call entirely. Unlike prompt caching (which makes a call cheaper), semantic caching *avoids* the call — so it saves both input and output cost. It appears as a cache layer keyed on the query *embedding*; the controlling knobs are the similarity threshold and a TTL.

**Batch API** — for work where no human is waiting, both OpenAI and Anthropic accept a file of requests, process them asynchronously within a 24-hour window, and charge 50% of standard token prices. Mechanically you upload a `.jsonl` (OpenAI) or pass a `requests` list (Anthropic Message Batches), poll for completion, and match results by `custom_id`. It appears as a separate async pipeline for nightly evals, bulk classification, or embedding backfills — never on the interactive path, since results can take up to 24h.

**Retrieval-k tuning** — every retrieved chunk is input tokens paid on every call that carries it, so lowering `k` directly lowers input cost (and prefill latency, linking to `01-latency-and-response-time-patterns.md`). It appears as the `k` parameter on the retriever. The constraint: too-low k drops needed context and hurts answer recall, so k is tuned against a retrieval-quality metric, not lowered blindly.

**Output-length capping** — because output tokens are the expensive class, bounding `max_tokens` to the shortest length the task genuinely needs is the single highest-leverage per-call cost cut. It appears as `max_tokens` on the completion call plus a "be concise" instruction. The constraint: cap too tight and answers truncate mid-thought, so cap to observed p95 answer length, not to an arbitrary small number.

**Distillation / fine-tuning a small model** — for a high-volume, narrow task, collect the large model's outputs as labeled data and fine-tune a small model to match it on that task (OpenAI's model-selection guide shows a fine-tuned small model matching a large model's accuracy at <2% of the cost). It appears as a fine-tuning job plus a swapped `model` ID. The trade-off: upfront labeling/training cost and a model that only excels at the trained task — worth it only when volume is high enough to amortize.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| `max_tokens` | Upper bound on output length (the most expensive token class) | Cap to observed p95 answer length for the task; a huge default silently inflates cost and latency |
| Prompt-cache breakpoint placement (`cache_control` / `prompt_cache_key`) | Which prefix is cached and reused | Place after the last *stable* block (system prompt, tools, docs); never after a per-request timestamp or the user turn, or every request pays a write with no read |
| Cache TTL (Anthropic 5m default / 1h option; OpenAI ~5–10m in-memory) | How long a cached prefix survives | Use default 5m for steady traffic ≥ every few minutes; pay for 1h only when calls are spaced 5–60 min apart |
| Router escalation threshold | Confidence at which a request goes to the large model | Set so escalated share matches the measured fraction of genuinely-hard queries; audit misroutes against an eval set |
| Semantic-cache similarity threshold | How close a query must be to reuse a cached answer | Start ~0.95 cosine; lower only if a labelled set shows a low false-hit rate |
| Retrieval `k` | Chunks entering the prompt (input tokens per call) | Smallest `k` that keeps answer recall above target; each extra chunk is paid on every carrying call |
| Batch `completion_window` (OpenAI `24h`) | Async turnaround for the −50% price | Use only when no human waits; never on the interactive path |

### Worked Example: Requirement → Decision

**Given:** A document-QA assistant costs $0.42 per request and finance flags it as unaffordable at projected volume. Instrumentation shows each request sends a 6,000-token static policy document plus the user question to a large (Opus-tier) model, generates ~800-token answers with `max_tokens=2048`, and ~70% of questions are simple lookups already answered before.

- **Step 1 — Identify the goal:** Cut cost per request without regressing answer quality; the bill is dominated by a large input prefix, an expensive model, and generous output.
- **Step 2 — Define inputs:** 6,000-token repeated policy doc + user question; current model = large; `max_tokens=2048`.
- **Step 3 — Define outputs:** A cost-per-request low enough for the volume, with equivalent measured answer quality.
- **Step 4 — Apply constraints:** Interactive (Batch API is off the table); quality must hold on an eval set; the policy doc is *stable* and repeated on every call; ~70% of traffic repeats.
- **Step 5 — Select the approach:** Layer four levers in order of leverage. (1) **Prompt-cache** the 6,000-token policy doc — reads bill at ~10% of input rate, the biggest single win since it repeats on every call. (2) **Semantic-cache** the ~70% repeated questions to skip the model call entirely. (3) **Route** simple lookups to a small model, escalating only hard synthesis questions. (4) **Cap `max_tokens`** to the p95 observed answer length (~1000). Rationale vs alternatives: distillation would cut per-call price but needs a training investment and doesn't address the repeated-doc or repeat-question waste; the Batch API is invalid because the surface is interactive. Caching + routing + capping attack the actual cost drivers with no quality loss when thresholds are eval-verified.

---

## Implementation

```python
# Scenario: a doc-QA endpoint resends the same 6k-token policy doc on every
# call to a large model. The doc is stable and the user turn is tiny, so we
# cache the doc prefix (billed ~10% of input on hits) and cap output — the two
# highest-leverage cost cuts. Cache-control goes on the STABLE block, user last.
from anthropic import Anthropic

client = Anthropic()

POLICY_DOC = "<the full static policy document ...>"  # 6k tokens, unchanged per call

def answer(question: str):
    return client.messages.create(
        model="claude-haiku-4-5",          # small model default; escalate only if needed
        max_tokens=1024,                    # cap the expensive output class to p95 length
        system=[
            {
                "type": "text",
                "text": POLICY_DOC,
                "cache_control": {"type": "ephemeral"},  # cache the stable prefix
            }
        ],
        messages=[{"role": "user", "content": question}],  # variable content LAST
    )
```

```python
# Scenario: nightly regression eval over 5,000 stored cases. No human is
# waiting, so we submit them to the Batch API at 50% off instead of firing
# 5,000 synchronous calls. Results are matched by custom_id, order-independent.
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

def submit_eval(cases):
    return client.messages.batches.create(
        requests=[
            Request(
                custom_id=f"case-{c['id']}",           # map result back to input
                params=MessageCreateParamsNonStreaming(
                    model="claude-haiku-4-5",
                    max_tokens=512,
                    messages=[{"role": "user", "content": c["prompt"]}],
                ),
            )
            for c in cases
        ]
    )
    # Poll batch.processing_status until "ended"; stream results by custom_id.
```

```python
# Anti-pattern: always call the largest model AND leave max_tokens unbounded.
# Every request pays top-tier per-token price for the expensive output class,
# with no ceiling — a single rambling answer can 10× the cost of a request.
resp = client.messages.create(
    model="claude-opus-5",     # largest model for EVERY request, even trivial ones
    max_tokens=8192,           # no real cap → occasional long answers blow the budget
    messages=[{"role": "user", "content": q}],
)

# Correct approach: default to the smallest model that passes eval, escalate only
# hard requests, and cap output to the length the task actually needs.
def routed_answer(q: str, is_hard: bool):
    return client.messages.create(
        model="claude-opus-5" if is_hard else "claude-haiku-4-5",  # tier by difficulty
        max_tokens=1024,        # bound the expensive output class
        messages=[{"role": "user", "content": q}],
    )
# What breaks in the anti-pattern: cost scales with (top-tier rate × unbounded
# output) on 100% of traffic; the fix pays top-tier only on the hard minority
# and caps the output tokens that dominate the per-call bill.
```

---

## Common Pitfalls & Misconceptions

- **Treating input and output tokens as equally priced** — beginners sum "total tokens" because that is what a naive estimate suggests, but output tokens cost roughly 4–5× input on frontier models, so the correct mental model is to cap and minimize output first, since it is the dominant per-call line item.
- **Caching a prefix that changes every request** — teams add `cache_control` expecting savings but place the breakpoint after a timestamp or the user turn, so the prefix hash never matches; the correct mental model is that cache writes happen only at the breakpoint and reads require a byte-identical prefix, so the marker must sit on the last *stable* block.
- **Routing everything to the largest model "to be safe"** — the instinct is that the biggest model can't hurt quality, but it multiplies cost on the majority of easy traffic that a small model handles fine; the correct mental model (per OpenAI's guide) is to hit the accuracy target first, then serve that accuracy with the cheapest model that still passes eval.
- **Using the Batch API for interactive requests** — the 50% discount tempts teams to route user-facing calls through it, but batch turnaround can be up to 24 hours and can't stream; the correct mental model is that Batch is only for work where no human is waiting (evals, backfills, bulk jobs).

---

## Key Definitions

| Term | Definition |
|---|---|
| Input (prompt) tokens | Tokens sent to the model — system prompt, retrieved context, history, user turn; billed at the input rate. |
| Output (completion) tokens | Tokens the model generates; billed at the output rate, typically ~4–5× the input rate on frontier models. |
| Cached (read) tokens | Prompt-prefix tokens served from a provider cache, billed at a fraction of the input rate (Anthropic ~10%; OpenAI reported as `cached_tokens`). |
| Call fan-out | The number of model calls a single user request produces — agent turns, retries, multi-tool/multi-source calls — the silent cost multiplier. |
| Model tiering / routing | Dispatching each request to the smallest capable model and escalating only hard cases, to serve target accuracy at minimum cost. |
| Semantic cache | A cache keyed on query-embedding similarity that returns a stored answer above a threshold, avoiding the model call entirely. |
| Batch API | An asynchronous request pipeline (OpenAI Batch / Anthropic Message Batches) that processes within 24h at 50% of standard token prices. |
| Distillation | Fine-tuning a small model on a large model's outputs so it matches on-task quality at small-model cost and latency. |

---

## Summary / Quick Recall

- Cost = tokens × rate, summed over calls; output tokens are ~4–5× input, so cap and minimize output first.
- Call fan-out (agent loops, retries, multi-source) is the silent multiplier — bound the loop and count the calls.
- Prompt caching cuts *input* cost on repeated stable prefixes; semantic caching avoids the call entirely on hits.
- Default to the smallest model that passes eval; escalate to the large model only for genuinely hard requests.
- The Batch API is 50% off but async (≤24h) — use it for evals and bulk jobs, never the interactive path.
- Distillation trades an upfront training cost for large-model quality at small-model price on a narrow, high-volume task.
- Instrument cost per request and per tenant, or a runaway agent loop surfaces only on the invoice (links to `08-observability-and-monitoring-patterns.md`).

---

## Self-Check Questions

1. On a frontier model, which token class is the more expensive per token, and roughly by what factor?

   <details><summary>Answer</summary>

   Output (completion) tokens are the more expensive class, typically ~4–5× the input rate on frontier models (e.g. an Opus-tier model at $5/input MTok vs $25/output MTok). Answering "they cost the same" or "input, because prompts are long" is wrong: prompt length inflates input cost but per token, output is priced several times higher, which is why capping `max_tokens` is the highest-leverage per-call cut.

   </details>

2. A doc-QA service resends the same 6,000-token static document to a large model on every call, and finance flags the cost. What is the single highest-leverage first fix, and why?

   <details><summary>Answer</summary>

   Enable **prompt caching** on the static document prefix — because it repeats byte-identically on every call, cache reads bill at ~10% of the input rate, cutting the largest recurring input line item immediately with no quality change. Lowering retrieval-k is the tempting wrong answer here: the document is a fixed prefix, not tunable retrieved chunks, so there is no k to lower; the win is caching the stable prefix, not shrinking it.

   </details>

3. You have 5,000 stored test cases to run a nightly regression eval. Which cost pattern applies, and which one would be a mistake to use?

   <details><summary>Answer</summary>

   Use the **Batch API** (OpenAI Batch / Anthropic Message Batches): no human is waiting, so the 50% discount applies at a ≤24h turnaround, matching results by `custom_id`. Using semantic caching would be a mistake as the primary lever here: eval cases are typically distinct and you *want* fresh model outputs to measure quality, so cache hits would defeat the purpose — the async batch discount is the right fit for offline bulk work.

   </details>

4. **Which TWO** of the following reduce cost by *eliminating or discounting model calls* rather than by making a single synchronous call's tokens cheaper?
   - A. Prompt caching a stable prefix
   - B. Semantic caching on repeated queries
   - C. Capping `max_tokens`
   - D. Submitting async work through the Batch API
   - E. Lowering retrieval `k`

   <details><summary>Answer</summary>

   **B and D.** Semantic caching returns a stored answer and skips the model call entirely on a hit (no call), and the Batch API processes calls through a separate async pipeline at 50% off (discounted calls, not a synchronous per-token trim). A, C, and E all make a single *synchronous* call cheaper by lowering the tokens it processes (cached-rate input, fewer output tokens, fewer input chunks). The most tempting wrong pick is A: prompt caching clearly saves money, but it does so by discounting input tokens *within* a synchronous call, not by eliminating or batching the call.

   </details>

5. Two proposals to cut a high per-request cost dominated by a large model generating long answers on mostly-easy traffic: (a) fine-tune/distill a small model on the task, or (b) route easy requests to a small model and cap `max_tokens`. Which is the faster, lower-risk first move, and when does the other become worth it?

   <details><summary>Answer</summary>

   Option (b) is the faster, lower-risk first move: model routing plus an output cap requires no training data or fine-tuning job, ships immediately, and directly attacks both cost drivers (top-tier rate on easy traffic + unbounded output). Option (a) — distillation — is the tempting distractor because it yields the lowest per-call price, but it carries upfront labeling/training cost and a model scoped to one task, so it only becomes worth it once volume is high enough to amortize that investment and routing alone no longer meets the budget. This is why you sequence cheap, reversible levers (routing, capping, caching) before committing to a trained artifact.

   </details>

---

## Further Reading

- [OpenAI — Prompt Caching](https://platform.openai.com/docs/guides/prompt-caching) — *verified 2026-07-29* — automatic caching for prompts ≥1024 tokens, `prompt_cache_key` for hit routing, and the `cached_tokens` / `cache_write_tokens` usage fields for measuring cache savings.
- [Anthropic — Prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) — *verified 2026-07-29* — `cache_control` breakpoints, cache-read at ~10% of base input and 5m write at 1.25×, and the `cache_read_input_tokens` / `cache_creation_input_tokens` usage fields.
- [OpenAI — Batch API](https://platform.openai.com/docs/guides/batch) — *verified 2026-07-29* — async `.jsonl` request files at a 50% discount with a 24h completion window, `custom_id` result matching, and a separate rate-limit pool.
- [Anthropic — Batch processing (Message Batches API)](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing) — *verified 2026-07-29* — asynchronous Messages requests at 50% off, most batches under 1h (≤24h expiry), and stacking the batch discount with prompt caching.
- [OpenAI — Model selection](https://platform.openai.com/docs/guides/model-selection) — *verified 2026-07-29* — the "optimize for accuracy first, then cost and latency" framework, comparing a smaller model, and distilling a fine-tuned small model that matched accuracy at <2% of the cost.
- [Anthropic — Models overview](https://docs.anthropic.com/en/docs/about-claude/models/overview) — *verified 2026-07-29* — per-model input vs output pricing (e.g. Opus-tier $5/input vs $25/output MTok) and comparative latency/context tiers used to reason about routing.
