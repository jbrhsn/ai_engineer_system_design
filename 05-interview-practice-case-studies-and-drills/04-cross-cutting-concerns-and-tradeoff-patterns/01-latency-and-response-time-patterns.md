# Latency & Response-Time Patterns for AI System Design

**Section:** 05 Interview Practice → Cross-Cutting Concerns & Trade-off Patterns | **Interview relevance:** High — latency is the single most common follow-up axis in any LLM/RAG/agent design ("how do you make it feel fast?"), and it forces you to reason about perceived vs actual latency, streaming, caching, and model/route selection under a stated budget.

---

## TL;DR

Latency in AI systems is dominated by the LLM generation step (tokens are produced serially) and by any sequential network hops before it — embedding, retrieval, reranking, tool calls, agent turns. You reduce it two ways: make the work smaller or parallel (smaller/routed model, cache, parallel retrieval, fewer agent hops), or make it *feel* faster (stream tokens, show intermediate progress). The right answer is almost always a combination chosen against an explicit p95/p99 budget. **The one thing to remember: for interactive LLM UX, time-to-first-token matters more than total completion time — stream first, then optimize the tail.**

---

## ELI5 — Explain It Like I'm 5

Imagine ordering at a coffee shop. The slow part isn't the barista taking your name — it's making the drink one step at a time. You can speed it up three ways: hire a faster barista (smaller/faster model), keep popular drinks pre-made on a shelf (caching), or have several baristas each make part of a big order at once (parallel calls). But even before any of that, the barista shouting "starting your latte!" the moment you order makes the wait *feel* half as long — that's streaming the first token. The common mistake is obsessing over total brew time when customers actually judge the shop by how long they stand there wondering if anyone heard them.

---

## Learning Objectives

By the end of this note you will be able to:
- [ ] Decompose an AI request's end-to-end latency into time-to-first-token (TTFT) and total completion time, and identify which the user actually feels.
- [ ] Locate where latency accrues across an embed → retrieve → rerank → generate (or agent-turn) pipeline.
- [ ] Select the right latency-reduction pattern (streaming, caching, model routing, parallelization, context reduction) for a stated complaint under a p95/p99 budget.
- [ ] Justify each choice against its trade-off (perceived vs actual latency, quality risk, staleness, complexity) in an interview.

---

## Visual Overview

### Where the time goes (RAG request)

```
User query
   │  (embed query        ~10–50 ms)
   ▼
Embed ──► Vector search ──► Rerank ──► LLM prefill ──► LLM decode (streamed)
 ~30 ms     ~20–100 ms      ~50–200 ms   TTFT ◄────┘   │  tokens/sec
                                                        └──► perceived done
   └──────────── often run in SERIES (the problem) ─────────────┘
```

### Latency-reduction decision tree

```
Is TTFT or total completion time the complaint?
├── TTFT high ──► Is the prompt huge?
│                 ├── Yes ──► trim context / prompt caching / lower retrieval-k
│                 └── No  ──► warm the model / drain the queue / smaller model
└── Total high ──► Is the output long?
                  ├── Yes ──► stream + cap max_tokens + prompt for concision
                  └── No  ──► parallelize independent hops / cache / route to small model
```

---

## The Core Problem

An LLM generates tokens autoregressively — each token conditions on the previous one — so generation latency scales with output length and cannot be parallelized inside a single completion. In agentic and RAG systems this base cost is multiplied by *sequential dependencies*: embed → retrieve → rerank → generate, or agent turn → tool call → agent turn. Each hop adds its own network and compute latency, and a naive design runs them all in series. The interview question is rarely "is it slow" — it is "where is the time going, and which reductions are worth their trade-off?"

Two latencies must be separated, because they are fixed by different levers:

- **Time-to-first-token (TTFT)** — how long until *something* appears. Dominated by prompt length (the prefill pass), upstream retrieval, and queueing delay.
- **Total completion time** — TTFT plus generation of all output tokens. Dominated by output length and the model's tokens-per-second.

A design that halves total time but leaves a 5-second silent TTFT will still feel broken; a design that streams at 400 ms TTFT feels responsive even at the same total time.

---

## Resolution Options

| Option | How it works | Buys you | Costs you | Reach for it when… |
|---|---|---|---|---|
| **Token streaming** | Emit tokens as generated (`stream=True` → SSE) | Much lower *perceived* latency; TTFT is all the user waits for | No reduction in total compute; needs streaming-capable transport/UI; harder to moderate partial output | Any interactive/chat surface — the default, not an optimization |
| **Semantic / exact caching** | Serve a stored response for repeated or near-duplicate queries | Near-zero latency and cost on hits | Staleness; cache-key design; embedding cost for semantic keys | High query repetition (FAQs, common intents) |
| **Smaller / routed model** | Route easy requests to a fast small model, hard ones to a large one | Lower TTFT and total on the majority path | Router-error risk; quality regression if misrouted | Mixed workload where most queries are easy |
| **Parallel retrieval / tool calls** | Fire independent hops concurrently instead of in series | Collapses N sequential hops into ~max(hop) | Complexity; only valid if hops are truly independent | Multi-source retrieval or independent tool calls |
| **Context / prompt reduction** | Trim retrieved-k, compress history, use prompt caching | Lower prefill = lower TTFT and cost | Risk of dropping needed context (recall loss) | Long prompts dominate TTFT |
| **Fewer / bounded agent hops** | Cap the graph; avoid free-running loops; merge steps | Removes whole round-trips | Less "agentic" flexibility | Latency-sensitive agent flows |

**Token streaming** — the LLM already produces tokens one at a time; streaming simply forwards each delta as it is decoded rather than buffering the full completion. In practice this is `stream=True` on the completion call, plumbed through Server-Sent Events to the client (OpenAI emits `response.output_text.delta` events; Anthropic emits `content_block_delta` with `text_delta`). It changes nothing about total compute but converts a multi-second frozen wait into sub-second TTFT with visible progress. Caveat from the docs: streamed partial output is harder to moderate, since moderation scores arrive only with the full completion.

**Semantic caching** — embed the incoming query, nearest-neighbour it against a store of prior `(query, response)` pairs, and return the cached response when similarity exceeds a threshold. It appears as a cache layer in front of the LLM keyed on the query *embedding* rather than an exact string. The controlling knob is the similarity threshold plus a TTL: too loose returns subtly wrong answers, too tight never hits.

**Smaller / routed model** — a lightweight classifier (or a cheap model) inspects each request and dispatches it to the smallest model that can handle it, escalating only hard cases. Mechanically this is a routing node in front of the generation step. It appears as the "model" field being chosen per-request rather than hard-coded. The trade-off is router accuracy: a misroute sends a hard query to a weak model and degrades quality.

**Parallel retrieval / tool calls** — when hops do not depend on each other's outputs (e.g. querying three data sources), issue them concurrently (`asyncio.gather`, parallel tool calls) and join. Total latency drops from the sum of hops to roughly the slowest one. It appears as concurrent async calls or a fan-out node in a LangGraph graph. The hard constraint: dependent steps (retrieve → generate) cannot be parallelized.

**Context / prompt reduction** — shorter prompts prefill faster, cutting TTFT and cost. Techniques: lower retrieval-k, summarize/compress conversation history, and use provider prompt caching so a stable prefix is not re-processed. It appears as the `k` parameter on the retriever, a history-summarization node, or cache-control markers on the prompt.

**Fewer / bounded agent hops** — each agent turn is a full LLM round-trip; unbounded ReAct loops multiply latency. Capping iterations and merging steps removes whole round-trips. It appears as a recursion/step limit on the agent graph and consolidated tool calls.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| `stream` (True/False) | Whether tokens are forwarded incrementally | Set `True` for any interactive surface; `False` only for batch/offline jobs where no human waits |
| `max_tokens` | Upper bound on output length (and thus total time) | Cap it to the shortest length the task genuinely needs; long default caps silently inflate p95 |
| Semantic-cache similarity threshold | How close a query must be to reuse a cached answer | Start ~0.95 cosine; lower only if you measure a low false-hit rate on a labelled set |
| Retrieval `k` | How many chunks enter the prompt (prefill size) | Use the smallest `k` that keeps answer recall above target; each extra chunk adds prefill latency |
| Agent max iterations / recursion limit | Hard stop on agent loop turns | Set to the worst-case turns a *correct* solution needs, plus a small margin — never "unlimited" |

### Worked Example: Requirement → Decision

**Given:** A support chatbot backed by RAG feels "frozen" — users wait ~5 s seeing nothing, then the full answer appears at once. p95 total time is 5.5 s; answers average ~500 tokens; a single large model is used with `stream=False`.

- **Step 1 — Identify the goal:** Make the assistant *feel* responsive; the complaint is silence, not total duration.
- **Step 2 — Define inputs:** Query embedding, retrieved chunks, a large-model completion call currently buffered.
- **Step 3 — Define outputs:** A streamed answer where the first tokens appear within a sub-second TTFT budget.
- **Step 4 — Apply constraints:** Interactive UX (TTFT is what's felt); moderate answer length; must not regress answer quality; SSE transport available.
- **Step 5 — Select the approach:** Enable **token streaming** first (largest perceived-latency win at near-zero cost/risk), then cap `max_tokens` and prompt for concision to trim the tail. Rationale vs alternatives: swapping to a smaller model risks quality and would not fix the "frozen" feel; semantic caching helps only repeated queries, not the first-time silence. Streaming directly targets the actual complaint.

---

## Implementation

```python
# Scenario: a chat endpoint feels frozen because the full answer is buffered
# before sending. We must minimize time-to-first-token on an interactive
# surface, so we forward each token delta over SSE as it is generated.
from anthropic import Anthropic

client = Anthropic()

def stream_answer(prompt: str):
    # stream=True → forward content_block_delta text as it arrives
    with client.messages.stream(
        model="claude-opus-4",
        max_tokens=512,               # cap the tail; don't let output run unbounded
        messages=[{"role": "user", "content": prompt}],
    ) as stream:
        for text in stream.text_stream:
            yield text                # push each delta to the client (SSE)
```

```python
# Anti-pattern: "parallelize everything" — firing retrieval and generation
# concurrently to save time. Generation DEPENDS on retrieval output, so this
# either races on missing context or silently generates without the documents.
import asyncio

async def broken():
    docs, answer = await asyncio.gather(
        retrieve(query),             # produces the context...
        generate(query),            # ...but this already ran without it!
    )
    return answer                    # answer never saw the retrieved docs

# Correct approach: parallelize only INDEPENDENT hops (multi-source retrieval),
# then run the dependent generation step after they join.
async def correct():
    # these three sources don't depend on each other → safe to parallelize
    web, db, kb = await asyncio.gather(
        search_web(query), query_db(query), search_kb(query)
    )
    context = merge(web, db, kb)
    return await generate(query, context)   # dependent step runs last
```

---

## Common Pitfalls & Misconceptions

- **Optimizing total time while ignoring TTFT** — beginners equate "faster" with "less total compute" because that is what benchmarks report; users judge interactive systems by time-to-first-token, so stream before you shrink.
- **Parallelizing dependent steps** — the instinct to "run everything at once" ignores data dependencies; only steps whose inputs do not depend on another step's output can go concurrent, so map the dependency graph first.
- **Caching raw LLM strings by exact match** — exact-match caches almost never hit on natural-language queries because phrasing varies endlessly; use semantic (embedding) caching with a tuned similarity threshold and TTL instead.
- **Leaving `max_tokens` at a huge default** — a generous cap feels safe, but total latency scales with tokens actually generated, so an unbounded cap lets occasional long answers blow the p95; cap to what the task needs.

---

## Key Definitions

| Term | Definition |
|---|---|
| Time-to-first-token (TTFT) | Latency from request submission until the first output token is emitted; dominated by prefill, retrieval, and queueing. |
| Total completion time | TTFT plus the time to generate all remaining output tokens; scales with output length and tokens-per-second. |
| Prefill | The forward pass over the input prompt before any output token is produced; longer prompts mean longer prefill and higher TTFT. |
| Perceived latency | How slow the system *feels* to a user, driven mainly by TTFT and visible progress rather than total time. |
| Semantic cache | A cache keyed on query embedding similarity (not exact string match) that returns a stored response above a similarity threshold. |

---

## Summary / Quick Recall

- Latency = TTFT + generation; optimize TTFT first for interactive UX.
- Streaming cuts *perceived* latency, not total compute or cost.
- Parallelize only independent hops; a dependent chain must stay serial.
- Semantic caching (not exact-match) is how you cache LLM responses.
- Model routing trades a small router-error risk for a large majority-path speedup.
- Always state an explicit p95/p99 budget and split TTFT vs total before proposing fixes.

---

## Self-Check Questions

1. What are the two components of end-to-end LLM latency, and which one dominates the *perceived* speed of an interactive chat?

   <details><summary>Answer</summary>

   The two components are time-to-first-token (TTFT) and total completion time. For an interactive chat, TTFT dominates perceived speed — users tolerate a long stream as long as tokens start appearing quickly. Answering only "total latency" is incomplete because it ignores that a fast total time with a long silent TTFT still feels frozen.

   </details>

2. A chatbot's total response time is acceptable, but users complain it "hangs." The system uses a single large model with `stream=False`. What is the cheapest first fix?

   <details><summary>Answer</summary>

   Enable token streaming (`stream=True`) so tokens appear as generated — this attacks the actual complaint (silent wait / high TTFT) at near-zero cost and no quality risk. Switching to a smaller model is the tempting wrong answer: it risks quality and does not address the "hang," because the hang is the buffered silence, not the total duration.

   </details>

3. You have a pipeline that queries three independent data sources and then generates an answer from all three. Where can you parallelize, and where can you not?

   <details><summary>Answer</summary>

   The three source queries are independent and can run concurrently (e.g. `asyncio.gather`), collapsing three serial hops into roughly one. The generation step cannot be parallelized with them because it depends on their merged output. The tempting error is parallelizing generation with retrieval, which produces an answer that never saw the retrieved context.

   </details>

4. **Which TWO** of the following reduce *perceived* latency without necessarily reducing total compute cost?
   - A. Token streaming
   - B. Routing easy queries to a smaller model
   - C. Showing an intermediate "thinking / retrieving…" progress indicator
   - D. Lowering retrieval `k`
   - E. Semantic caching

   <details><summary>Answer</summary>

   **A and C.** Streaming forwards tokens as they are generated and progress indicators fill the silence — both improve how fast the system *feels* while total compute is unchanged. B and D reduce *actual* compute/latency (smaller model, shorter prefill), and E reduces both latency and cost on cache hits — so none of B/D/E are "perceived-only." E is the most tempting wrong pick because it clearly lowers latency, but it does so by eliminating real work on hits, not by changing perception.

   </details>

5. Two proposals to fix a 12 s p99 spike under bursty load: (a) trim the prompt/context, or (b) add model replicas plus a request queue with a small-model fast lane. Which addresses the root cause, and why?

   <details><summary>Answer</summary>

   Option (b). A p99 spike that appears only under bursty load is a queueing/capacity problem — requests wait for a busy model pool — so adding concurrency and a fast lane addresses the root cause. Option (a) is the tempting distractor: trimming the prompt lowers per-request prefill latency but does nothing for time spent waiting in a queue, so the tail spike under load persists. This is why you first classify the latency as compute-bound vs queue-bound before choosing a fix (links to `02-scalability-and-throughput-patterns.md`).

   </details>

---

## Further Reading

- [OpenAI — Streaming API responses](https://platform.openai.com/docs/api-reference/streaming) — *verified 2026-07-29* — `stream=True` over SSE, semantic streaming events (`response.output_text.delta`), and the note that moderation scores arrive only with the full completion.
- [Anthropic — Streaming Messages](https://docs.anthropic.com/en/docs/build-with-claude/streaming) — *verified 2026-07-29* — SSE event flow (`message_start` → `content_block_delta` → `message_stop`) and text/tool/thinking deltas for incremental output.
