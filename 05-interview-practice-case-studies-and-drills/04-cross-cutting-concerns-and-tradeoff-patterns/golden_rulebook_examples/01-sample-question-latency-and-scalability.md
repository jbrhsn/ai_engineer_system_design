# Sample Question — Real-Time Customer-Facing Shopping Assistant (Golden Rulebook Worked Answer)

This worked answer is grounded strictly in `../00-golden-rulebook-cheatsheet.md`; every recommendation traces back to a specific 9-Concern Sweep row, Golden Rule, Decision Cue, Trade-off One-Liner, Red Flag, or 60-Second Framework step in that cheatsheet.

---

## The Question

### System Design: Real-Time Customer-Facing Shopping Assistant

**Context:** You're the senior LLM engineer at a mid-size e-commerce company. You're building a customer-facing RAG shopping assistant embedded in the storefront: shoppers type natural-language questions ("does this jacket run small?", "what's a warmer alternative under $120?"), and the assistant retrieves over a product-and-reviews corpus (~3M product docs + review snippets, growing ~10k docs/day) and answers conversationally, with the option to call tools (inventory lookup, add-to-cart). Steady-state traffic is roughly 800 requests/second of interactive chat, but marketing runs flash-sale campaigns that drive a ~20× spike within minutes, and the traffic is bursty and multi-tenant across several brand storefronts sharing the same backend. The assistant must feel instant or shoppers abandon the session.

**Requirements:**

- **Perceived latency:** something must appear on screen in well under a second (TTFT); p95 total answer time < 3s, p99 < 6s for interactive turns.
- **Throughput / burst:** sustain ~800 rps steady, absorb a ~20× flash-sale spike (~16,000 rps peak) without cascading timeouts.
- **Concurrency:** target ~5,000 concurrent interactive sessions at peak.
- **Multi-tenant fairness:** no single brand storefront (or one runaway campaign) may starve the others.
- **Cost:** a per-query cost ceiling; flash sales must not blow the monthly budget on the biggest model.
- **Correctness/eval:** answers grounded in the product corpus (faithfulness), no hallucinated stock/price; changes gated by eval before they reach shoppers.
- **Reliability:** graceful degradation under provider hiccups; tool actions (add-to-cart) must never double-fire.

**Your task:**

1. **Architecture & request path** — Sketch the end-to-end system and walk one interactive turn hop-by-hop, naming where the seconds and tokens go.
2. **Perceived vs actual latency** — How do you hit sub-second TTFT and the p95 < 3s SLO? What do you optimise first, and what's the trade-off?
3. **Spike / throughput strategy** — When the 20× flash-sale spike hits and tail latency explodes, how do you decide what to scale, and how do you keep p99 bounded?
4. **Multi-tenant fairness** — How do you stop one brand's campaign from starving the other storefronts sharing the backend?
5. **Failure modes & rollout** — Provider outage, poisoned inputs, double add-to-cart: how do you degrade gracefully and ship changes safely?
6. **Trade-offs & what you'd measure** — State the trade-offs out loud and the signals you'd watch to know each lever actually worked.

---

## Worked Answer

Before designing anything I want to pin the SLOs, because "make the shopping assistant fast" is a red flag phrasing that hides two different problems. I'll restate the numbers I'm designing to: **TTFT under ~1s** (something on screen fast), **p95 total < 3s / p99 < 6s** for an interactive turn, **~800 rps steady with a ~20× (~16k rps) flash-sale burst**, **~5,000 concurrent sessions**, a **per-query cost ceiling**, and **fairness across several brand tenants**. I also want two clarifications up front: is the complaint (or fear) about the silence before the first token (TTFT) or total completion time — because those get different fixes — and is the flash-sale spike genuinely saturating the model, or are requests just queuing? I refuse to scale or "make it faster" until I know which. With those fixed, here's the design.

### Answer

#### 1. Architecture & request path

The system is the standard RAG-plus-tools pipeline, kept stateless behind a load balancer so replicas can be added freely, with shared state (session, cache, index) externalised:

```text
                       shopper query (interactive turn)
                                   │
                                   ▼
                         [API gateway / LB]
                     per-tenant token rate limit
                                   │
                        ┌──────────┴───────────┐
                 semantic cache HIT        cache MISS
                        │                       │
                   stream cached           [embed query]
                   answer (~0 cost)              │
                                          [retrieve top-k] ──► [rerank]
                                                 │                 │
                                        (independent metadata/     │
                                         inventory hops run        │
                                         in parallel)              ▼
                                                          [generate, stream=True]
                                                                   │
                                              tool call? ──► [inventory / add-to-cart]
                                                                   │  (idempotency key)
                                                                   ▼
                                                         stream tokens + citations
```

Walking one interactive turn hop-by-hop, naming time/tokens/dollars per hop (60-Second Framework step 4):

- **Gateway + rate limit:** negligible time; this is where per-tenant fairness and load-shedding live.
- **Semantic cache check:** near-zero on a hit — a hit skips retrieval and generation entirely, so this is both the biggest latency win and the biggest cost win for repeated flash-sale questions ("is this on sale?").
- **Embed query:** one small embedding call; cheap, but on the critical path so it counts toward TTFT.
- **Retrieve top-k + rerank:** dominated by `k` — more chunks lift recall but inflate prefill tokens, latency, and bill. Independent hops (e.g. inventory/metadata lookups) run in parallel with retrieval; retrieve→generate stays serial because it's a dependent chain.
- **Generate:** the dominant cost. Output tokens run ~4–5× the price of input tokens, so an uncapped generation is the silent budget and latency risk. TTFT is set by prefill (query + retrieved context); total time is set by output length.
- **Tool call (optional):** inventory lookup or add-to-cart; side-effecting actions carry an idempotency key so a retry never double-fires.

#### 2. Perceived vs actual latency

The first lever is Golden Rule 1: **stream first, then optimise the tail.** If the assistant buffers the whole completion server-side and returns it in one shot, TTFT is the full generation time and the UI "feels frozen" — the classic latency-row trigger. Turning on `stream=True` collapses *perceived* latency to first-token time without touching model or retrieval quality: the shopper sees words appear well under a second. That alone attacks the sub-second TTFT requirement.

Only after streaming do I optimise the tail toward p95 < 3s:

- **Cap `max_tokens`.** Output is the expensive token class; capping it bounds both the tail latency and the per-query cost. A shopping answer doesn't need 800 tokens.
- **Semantic cache.** Flash-sale traffic is highly repetitive ("is the sale live?", "does the code work?"). A semantic cache (matching meaning, not exact string) returns near-instantly and skips the whole retrieve→generate cost.
- **Parallelize only independent hops.** Inventory/metadata lookups run concurrently with retrieval; I do **not** claim to parallelize retrieve→generate — that's a dependent chain and stays serial.
- **Tune `k`.** Smallest `k` that still holds recall — fewer chunks means less prefill, lower TTFT, lower bill.
- **Route to the smallest model that passes eval.** A smaller model cuts generation latency, but it trades quality, so it comes *after* streaming, gated by eval — never instead of streaming.

**Trade-off stated out loud:** streaming cuts *feel*, not *compute* — total generation cost is unchanged; and a smaller model risks quality. I stream to fix the feel before I trade any accuracy away, and I never optimise this axis blind.

#### 3. Spike / throughput strategy

When the 20× flash-sale spike hits and tail latency explodes, Golden Rule 2 governs: **classify before you fix.** I will not "just add more GPUs" — that's a red flag until I've read utilization. The first move is to **read serving utilization to decide compute-bound vs queue-bound**, because the remedy is completely different:

```text
              ~20× flash-sale spike arrives
                          │
                          ▼
             read utilization / classify
              ┌───────────┴────────────┐
        compute-bound              queue-bound
     (GPUs saturated)         (GPUs idle, requests waiting)
              │                        │
     replicas + autoscale      bounded queue + backpressure
     dynamic/continuous        + load-shed (429)
     batching                  per-tenant token rate limit
                               async queue + DLQ for bulk
                               stateless + shared state
```

- **If compute-bound** (model/GPU genuinely saturated): **replicas + autoscale** for real capacity, plus **dynamic/continuous batching** to raise throughput by packing concurrent requests. Trade-off: batching improves throughput at the cost of a little per-request latency, so I tune batch size against the p95/p99 SLO rather than maximising it.
- **If queue-bound** (requests piling up but GPUs not maxed): adding GPUs does nothing. Instead **bound every queue** — an unbounded queue under a 20× spike is an OOM waiting to happen — and past the bound apply **backpressure** or **load-shed with 429s**. Trade-off: load-shedding refuses some requests to keep p99 bounded for those we accept; I trade coverage for a bounded tail. Non-interactive/bulk work (e.g. re-indexing new products) is decoupled onto an **async queue + DLQ** so it never sits in the hot interactive path.

Because the service is **stateless with shared state externalised**, autoscaled replicas can take traffic immediately.

#### 4. Multi-tenant fairness

A flash sale is exactly the "one tenant starving others" cue: one brand's campaign floods the shared backend and the other storefronts time out. The fix is **per-tenant token-based rate limiting** at the gateway — each brand gets a token budget, so a runaway campaign is throttled (429/backpressure) rather than consuming the whole cluster. I rate-limit on tokens, not just request count, because a few large-context requests can starve the pool as effectively as many small ones. Bulk per-tenant work (bulk catalog Q&A, re-embedding a brand's new SKUs) is **decoupled to the async queue** so it can't crowd out interactive turns from other tenants. This keeps the fairness requirement — no brand starves another — enforceable independent of how aggressive one campaign gets.

#### 5. Failure modes & rollout

Standard closing sweep (60-Second Framework step 7):

- **Transient provider errors** → retry with backoff + jitter.
- **Persistent provider outage** → circuit breaker, then a **fallback chain** (e.g. smaller/self-hosted model, or a cached/canned response) so shoppers still get *something* instead of a spinner.
- **Poisoned / malformed inputs or tool payloads** → schema validate + repair; route unrecoverable messages to a **DLQ**.
- **Timeouts** on every hop so one slow retrieval can't hang the turn.
- **Double add-to-cart:** side-effecting tool actions carry an **idempotency key** — exactly-once semantics so a retry during the spike never double-charges or double-adds. And I treat a 200 from a tool as *claimed* success, not proven success — I validate the effect.
- **Rollout:** every change (new prompt, new `k`, smaller model, new index) goes through **eval-gate → canary → rollback**. I design the eval before build, not after — "we'll add eval later" is a red flag — with an offline golden set as a CI gate and online drift monitoring, splitting RAG metrics into context precision/recall and faithfulness/answer-relevancy, judged with a bias-controlled LLM-as-judge.

#### 6. Trade-offs & what you'd measure

Trade-offs stated out loud, straight from the cheatsheet rows:

- **Latency vs accuracy** — streaming fixes *feel*, not compute; a smaller model is cheaper/faster but risks quality, so stream first and gate any model swap on eval.
- **Batch latency vs throughput** — continuous batching raises throughput but adds per-request latency; tune against p95/p99.
- **Coverage vs bounded tail** — load-shedding refuses some requests to keep p99 bounded for the accepted ones.
- **Recall vs cost/latency of large-k** — smallest `k` that holds recall.
- **GPUs waste money on a queue-bound spike** — classify before scaling.

What I'd measure to know each lever worked: **TTFT and total time split separately** (streaming should crater TTFT while total is roughly flat); **serving utilization** (to confirm the compute-vs-queue classification); **p95/p99 total under load** (to confirm load-shed/batching held the tail); **per-tenant token consumption and 429 rate** (fairness); **cache hit rate and per-query cost** (cost ceiling); and **faithfulness / answer-relevancy on the golden set** (that the latency and cost wins didn't quietly degrade quality). I never claim a latency, cost, or throughput win in isolation — Golden Rule 16, never optimise one axis blind: a win is only real if accuracy and safety held.

---

### How the cheatsheet was used

- **60-Second Framework step 1 (clarify & scope) + step 2 (name SLOs)** → the opener refuses to design until TTFT-vs-total and compute-vs-queue are clarified and the SLO numbers (TTFT <1s, p95 <3s, p99 <6s, 800→16k rps, 5k concurrent, cost ceiling) are restated.
- **60-Second Framework step 3 (sketch architecture)** → Task 1's ingest→retrieve→rerank→generate→cite/act diagram.
- **60-Second Framework step 4 (walk request path, time/tokens/dollars per hop)** → Task 1's hop-by-hop walk naming where seconds, tokens, and dollars go.
- **60-Second Framework step 5 (sweep 9 concerns, raise eval/cost/safety unprompted)** → eval, cost, and reliability raised without being asked, in Tasks 5–6.
- **60-Second Framework step 7 (failure modes + rollout)** → Task 5's retry/circuit-breaker/fallback/DLQ, idempotency, eval-gate→canary→rollback.
- **9-Concern Sweep → Latency row** → Task 2's split TTFT vs total, stream first, cap `max_tokens`, parallelize independent hops, semantic cache, route to small model.
- **9-Concern Sweep → Scalability row** → Task 3's classify compute vs queue-bound, replicas+autoscale, dynamic/continuous batching, bounded queue + backpressure + load-shed, per-tenant rate limit, async queue + DLQ, stateless + shared state.
- **9-Concern Sweep → Reliability row** → Task 5's transient→retry+backoff+jitter, persistent→fallback chain, poison→DLQ, timeout, circuit breaker, idempotency, schema validate+repair.
- **9-Concern Sweep → Eval row** → Task 5's offline golden set CI gate + online drift, context precision/recall + faithfulness/answer-relevancy, bias-controlled LLM-as-judge.
- **9-Concern Sweep → Cost row** → cost ceiling handling: output ~4–5× input, cap `max_tokens`, route smallest model, semantic cache, tune `k`.
- **Golden Rule 1 (stream first, then optimise tail)** → Task 2's ordering: `stream=True` before any tail work.
- **Golden Rule 2 (classify compute vs queue-bound before scaling)** → Task 3's read-utilization-first branch.
- **Golden Rule 3 (every queue bounded)** → Task 3's bounded queue + 429 load-shed + OOM warning.
- **Golden Rule 4 (parallelize only independent hops)** → Task 1/2 keeping retrieve→generate serial while parallelizing inventory/metadata.
- **Golden Rule 5 (cache semantically; cap `max_tokens`)** → Task 2's semantic cache and `max_tokens` cap.
- **Golden Rule 6 (route smallest model that passes eval)** → Task 2's model-routing option bound to an eval gate.
- **Golden Rule 7 (design eval before build; eval-gate every change)** → Task 5's eval-first rollout.
- **Golden Rule 8 (idempotency = exactly-once)** → Task 5's add-to-cart idempotency key.
- **Golden Rule 9 (200 isn't always success)** → Task 5's validate-the-effect on tool 200s.
- **Golden Rule 16 (never optimise one axis blind)** → Task 6's closing caveat.
- **Decision Cue "feels frozen → stream, TTFT≠total"** → Task 2's primary lever.
- **Decision Cue "Black-Friday burst → bounded queue + autoscale + backpressure/load-shed, classify queue-bound first"** → Task 3.
- **Decision Cue "tail latency explodes → read utilization → batch/replicas vs shed/rate-limit/decouple"** → Task 3's branch.
- **Decision Cue "one tenant starving others → per-tenant token rate limit + decouple bulk to queue"** → Task 4.
- **Decision Cue "provider outage → retry+backoff+jitter → circuit breaker → fallback chain"** → Task 5.
- **Trade-off One-Liner (latency vs accuracy)** → Task 2/6: stream to fix feel before trading quality.
- **Trade-off One-Liner (coverage vs bounded tail)** → Task 3/6: load-shed keeps p99 bounded for accepted requests.
- **Trade-off One-Liner (recall vs cost/latency of large-k)** → Task 2/6: smallest `k` that holds recall.
- **Red Flag "just make it faster (no p95/TTFT split)"** → avoided in the opener.
- **Red Flag "just add more GPUs (queue-bound)"** → avoided in Task 3.
- **Red Flag "we'll add eval/monitoring later"** → avoided in Task 5.

---

## Cheatsheet elements referenced

- 60-Second Framework — steps 1, 2, 3, 4, 5, 7
- 9-Concern Sweep — Latency row
- 9-Concern Sweep — Scalability/Throughput row
- 9-Concern Sweep — Reliability row
- 9-Concern Sweep — Eval row
- 9-Concern Sweep — Cost row
- Golden Rule 1 — Stream first, then optimise the tail
- Golden Rule 2 — Classify compute vs queue-bound before scaling
- Golden Rule 3 — Every queue bounded
- Golden Rule 4 — Parallelize only independent hops
- Golden Rule 5 — Cache semantically; cap max_tokens
- Golden Rule 6 — Route to the smallest model that passes eval
- Golden Rule 7 — Design eval before build; eval-gate every change
- Golden Rule 8 — Idempotency = exactly-once
- Golden Rule 9 — 200 isn't always success
- Golden Rule 16 — Never optimise one axis blind
- Decision Cue — "feels frozen"
- Decision Cue — "Black-Friday burst"
- Decision Cue — "tail latency explodes"
- Decision Cue — "one tenant starving others"
- Decision Cue — "provider outage"
- Trade-off One-Liner — Latency vs accuracy
- Trade-off One-Liner — Coverage vs bounded tail
- Trade-off One-Liner — Recall vs cost/latency of large-k
- Red Flag — "just make it faster"
- Red Flag — "just add more GPUs"
- Red Flag — "we'll add eval/monitoring later"
