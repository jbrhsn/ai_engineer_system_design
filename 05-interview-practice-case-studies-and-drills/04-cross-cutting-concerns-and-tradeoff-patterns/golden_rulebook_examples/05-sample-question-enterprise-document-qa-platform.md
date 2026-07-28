# Sample Question — Enterprise Document Q&A Platform (Golden Rulebook Worked Answer)

This worked answer is grounded strictly in the Golden Rulebook cheatsheet at `../00-golden-rulebook-cheatsheet.md`; every recommendation traces to a named row, golden rule, decision cue, trade-off one-liner, or red flag in that document.

---

## The Question

**System Design: Enterprise Document Q&A Platform**

**Context.** You're the senior LLM engineer at a company building an internal Q&A platform for a large enterprise (50,000 employees). The platform must let employees ask natural language questions and get accurate answers grounded in the company's internal documents — Confluence pages, Google Docs, Slack threads, PDFs, and Jira tickets. The corpus is ~2 million documents and grows by ~5,000 documents/day.

**Requirements:**
- p95 latency under 3 seconds for a response
- Must support at least 5,000 concurrent users during peak hours
- Answers must cite sources, and citations must be verifiable (no hallucinated references)
- Different documents have different access permissions per employee (e.g., HR docs, legal docs, exec-only strategy docs) — the system must never leak restricted content to unauthorized users
- Documents are updated/deleted frequently; stale answers are unacceptable beyond a ~15 minute staleness window
- Cost is a real constraint — leadership wants to know rough $/query economics
- Needs to support both "simple lookup" questions ("what's our PTO policy?") and "multi-hop reasoning" questions ("compare our Q2 and Q3 churn drivers and explain what changed")

**Your task:**
1. **Architecture** — Sketch the end-to-end system: ingestion pipeline, indexing/retrieval layer, orchestration, model serving, and response layer. Call out where you'd use RAG vs. fine-tuning vs. tool use, and why.
2. **Permissions** — Design how document-level access control is enforced at retrieval and generation time without leaking content through embeddings, retrieved chunks, or model reasoning traces.
3. **Freshness** — How do you keep the index fresh within the 15-minute SLA at this ingestion rate, without full reindexing?
4. **Multi-hop reasoning** — How does your system distinguish a simple lookup from a query that needs decomposition, retrieval across multiple sources, and synthesis? What's your orchestration strategy (single call vs. agentic loop), and how do you bound cost/latency/runaway loops?
5. **Evaluation** — How would you measure answer quality and catch regressions before they reach production? What's your approach to detecting hallucinated citations specifically?
6. **Cost model** — Give a rough back-of-envelope estimate of $/query and identify the top 2–3 levers you'd pull to cut cost by 50% without materially hurting quality.

---

## Worked Answer

Before designing, I'd clarify the scope dials that set everything — scale, latency, quality bar, data sensitivity, tenancy, and change cadence — and restate them as measurable SLOs. Scale: ~2M documents growing +5k/day, 5,000 concurrent users at peak. Latency: **p95 < 3s** end-to-end (which I'll immediately split into TTFT vs total). Quality bar: **verifiable citations, zero hallucinated references** — this is a grounding/faithfulness constraint, not a vibe. Data sensitivity: **per-document access control** where restricted content (HR/legal/exec) must never leak to an unauthorized user — the security and privacy concerns are load-bearing here, not afterthoughts. Change cadence: **≤15-minute staleness window** at +5k docs/day plus edits/deletes. Cost: leadership wants **rough $/query economics** and a path to cut it.

Those clarifiers map to the SLOs I'll defend throughout: p95 < 3s (with streamed TTFT well under that), 5k concurrent sustained, **no cross-permission leak** (a hard safety invariant, not a metric to average), staleness ≤15 min, verifiable-citation faithfulness held on every change, and a defensible blended **$/query** with a 50%-reduction plan. The framing rule I commit to up front is **Golden Rule 16 — never optimise one axis blind**: any cost or latency win in tasks 4 and 6 is real only if faithfulness and the permission invariant are proven held.

### Answer

#### 1. Architecture

This is a **RAG** problem, not a fine-tuning problem. The corpus is 2M documents changing thousands of times a day; fine-tuning bakes knowledge into weights that go stale the moment a document changes and cannot enforce per-user permissions — so knowledge stays in the retrieval layer. Fine-tuning, if used at all, is reserved for *behavioural* shaping (answer format, citation discipline), never for facts. **Tool use** appears where a question needs a live system-of-record lookup (e.g. a Jira status) rather than a document chunk — but tool calls are treated as untrusted content, covered in task 2. So: RAG for grounding, optional light fine-tune for style, tool use for live structured lookups.

I'd sketch the standard RAG spine — ingest → chunk → embed → index → retrieve → rerank → generate → cite — wrapped in a bounded orchestrator, and walk the request path noting time, tokens, and dollars per hop.

```text
INGESTION (async, per source connector)                    QUERY PATH (p95 < 3s)
Confluence/GDocs/Slack/PDF/Jira                            USER QUESTION
        │  change events / crawl                                  │
        ▼                                                         ▼
[Ingest queue]  ──poison──► [DLQ]                        [Router / classifier]
        │  (bounded, at-least-once)                          simple ── vs ── hard
        ▼                                                    │              │
[Parse → chunk → embed]                                      │ single call  │ bounded
        │  attach ACL metadata + version                     │              │ agentic loop
        ▼                                                     ▼              ▼
[Vector index + metadata store]  ◄───── retrieve (top-k, ACL-filtered at engine)
   per-doc ACL tags, version pins                            │
        │                                                    ▼
   incremental upsert / tombstone                     [Rerank → smallest-k that holds recall]
   (never full reindex)                                      │
                                                             ▼
                                                    [Generate (stream=True, cap max_tokens)]
                                                             │
                                                             ▼
                                                    [Cite / faithfulness-grounding check]
                                                             │
                                                             ▼
                                                    ANSWER + verifiable citations (or ABSTAIN)
```

**Ingestion pipeline.** Each source (Confluence, Google Docs, Slack, PDF, Jira) has a connector that emits change events onto a **bounded async queue with a DLQ** — transient failures retry with backoff+jitter, poison documents land in the DLQ rather than blocking the pipeline (Reliability row). Documents are parsed, chunked, embedded, and upserted with two critical pieces of metadata: the **per-document ACL tags** (task 2) and a **version pin** (task 3).

**Indexing/retrieval layer.** A vector index plus a metadata store. Retrieval pulls top-k with the ACL filter applied *at the engine* (task 2), then a reranker trims to the smallest k that still holds recall — large k inflates prefill on every generation call (recall vs cost/latency-of-large-k trade-off).

**Orchestration.** A router classifies simple-lookup vs multi-hop and dispatches to either a single generation call or a bounded agentic loop (task 4).

**Model serving.** Stateless generation replicas behind autoscaling with continuous/dynamic batching; classify **compute-bound vs queue-bound** before adding GPUs (Golden Rule 2) — 5k concurrent at peak wants a bounded queue with backpressure and load-shed, not a reflexive GPU pile-on. Route simple lookups to the smallest model that passes eval; escalate to a frontier model only for hard synthesis (Golden Rule 6).

**Response layer.** Stream first (`stream=True`) so perceived latency beats the 3s bar even when total compute is higher — TTFT ≠ total (Golden Rule 1). Attach verifiable citations after the grounding check.

#### 2. Permissions

The non-negotiable principle from the Security row is **the LLM is never the security boundary** (Golden Rule 11). No prompt instruction, no system message, no "the model checks permissions" — authorization lives in **code/DB with complete mediation**. Concretely, three layers:

- **Retrieval-time filter at the engine, not in app code.** Every chunk carries the ACL tags of its source document. The user's effective permission set is resolved from the identity/DB, and the vector query applies that filter **inside the retrieval engine** — the RLS / per-tenant-namespace pattern from the Privacy row. This matters because **app-code filters fail open**: if a post-retrieval filter throws or is skipped, restricted chunks leak; an engine-level filter that returns nothing on failure fails *closed*. The decision cue "cross-tenant leak → RLS/per-tenant namespaces at the engine, not app-code filters" is exactly this.
- **Don't leak via embeddings.** Restricted content must not be retrievable by an unauthorized user *even in similarity space*. That means the ACL filter is a hard pre-filter on the candidate set, not a re-rank afterthought — an unauthorized user's query never sees restricted vectors as candidates. Embeddings of restricted docs live behind the same mediation as the source.
- **Treat generation input and output as untrusted.** Retrieved chunks and any tool results are **isolated as untrusted content** (Golden Rule 12, LLM05) — the model only ever sees chunks the user is already authorized to read, so even a prompt-injection payload embedded in a document cannot make the model surface content the retrieval layer never handed it. Reasoning traces are constrained to authorized context for the same reason: the model can only reason over what it was given, and it was given only authorized chunks.

The permission invariant is a **safety property, not a quality metric** — you don't get to average it. A single cross-permission leak is a shutdown event, which is why enforcement is in code/DB and fails closed. The trade-off I'd state: guardrails and mediation add a little latency/friction, but there is **no fool-proof prompt fix** — you limit blast radius structurally.

#### 3. Freshness

The staleness SLA is ≤15 minutes against +5k docs/day plus edits and deletes. The Versioning row is explicit: **never full reindex when unnecessary** — full reindex of 2M docs cannot hit a 15-minute window and burns cost for nothing. Instead:

- **Incremental upsert.** Change events flow through the same bounded ingestion queue (Reliability: async queue + DLQ). A new or edited document is chunked, embedded, and **upserted** into the live index within minutes; the queue's throughput is sized so the tail of the backlog stays inside the 15-minute window even at peak ingest.
- **Deletes cascade and tombstone.** A deleted document must vanish from answers, so deletion **cascades source → chunks → embeddings → caches** (Privacy row, right-to-erasure) — a tombstone removes it from retrieval candidates immediately, and any semantic-cache entries grounded on it are invalidated. This also satisfies right-to-erasure if a document contained personal data.
- **Dual-index cutover only on embedder change.** Routine content updates are incremental. The only time I touch the whole index is when the **embedding model changes** — then I build a parallel index and **flip the query embedder and reader in lockstep** (dual-index cutover), never leaving the query encoder and the index on mismatched embedders. Every version is pinned — model, prompt, index, schema — never `-latest` (Golden Rule 15).
- **Pin the (model, prompt, index) triple per answer** so I can always answer "which version produced this answer?" and prove freshness after the fact.

The trade-off: incremental indexing means the index is eventually-consistent inside the 15-minute window rather than instantly global-consistent — acceptable given the stated SLA, and vastly cheaper than reindexing.

#### 4. Multi-hop reasoning

The system must serve both "what's our PTO policy?" (one lookup) and "compare Q2 vs Q3 churn drivers and explain what changed" (decompose → retrieve across sources → synthesise). The routing principle is **Golden Rule 6 — route to the smallest model / cheapest path that passes eval; escalate only for hard requests.**

- **Router classifies simple vs hard.** A lightweight classifier (small model or heuristic) decides: single retrieval + single generation call for a lookup, versus a decomposition path for a multi-hop question. Simple questions never pay for an agentic loop — that's the single-call-vs-agentic-loop decision.
- **Single call for lookups.** Retrieve, rerank, generate with citations, done. Cheapest path, lowest latency, easily inside p95 < 3s.
- **Bounded agentic loop for multi-hop.** Decompose into sub-questions, retrieve per sub-question, synthesise. Critically, **every agent loop is bounded** (Golden Rule 10) — a `recursion_limit` / step cap prevents runaway loops, and on hitting the cap the system **degrades or escalates** rather than spinning: return a best-effort grounded partial answer or abstain, never loop forever.
- **Parallelize only independent hops** (Golden Rule 4). "Q2 drivers" and "Q3 drivers" are independent retrievals — fetch them concurrently to protect latency. The final "explain what changed" synthesis depends on both, so it runs after — dependent hops stay sequential.
- **Bound cost via fan-out discipline.** The Cost row warns **fan-out is the silent multiplier** — each sub-question is a full paid hop. So the decomposer is capped (a small, bounded number of sub-questions), and I resist letting the loop widen fan-out unchecked. The trade-off is autonomy/agency vs safety-and-cost: more decomposition can answer harder questions but multiplies spend and latency, so agency is bounded and gated.

Latency for multi-hop is protected by streaming the synthesis (TTFT), parallelizing independent retrievals, and capping the loop; when a genuinely hard question can't be answered inside budget, degrade to a scoped answer plus "I couldn't fully compare X" rather than blow the SLA.

#### 5. Evaluation

**Golden Rule 7 — design eval before you build.** The first deliverable is a golden set, not a feature. I split evaluation two ways (Evaluation row).

**Offline (CI gate, with references).** A curated golden set of question → expected-answer pairs, run on every prompt/model/index/k change as a CI gate. Because this is RAG, I split the metrics so a failure localises:
- **Retrieval:** context precision + context recall — is the retriever pulling the right chunks, under the *authorized* ACL scope?
- **Generation:** faithfulness (answer supported by retrieved context) + answer-relevancy (does it address the question?).

**Online (no reference, watch drift).** On live traffic I run guardrail metrics and watch for drift, because **offline can't cover the live distribution**. "Green dashboard but wrong answers" is caught by online quality + drift with per-step spans, not endpoint logs.

**Hallucinated-citation detection is a faithfulness / grounding eval.** The verifiable-citation requirement is precisely a grounding check: for every citation the answer emits, I verify (a) the cited chunk exists in the index and was actually retrieved for this query, and (b) the claim it supports is entailed by that chunk. A citation that points to a nonexistent or non-retrieved source, or a claim not entailed by its cited chunk, is a **faithfulness failure** — flagged offline in the golden set and online as a guardrail metric. When the grounding check fails at request time, the system **abstains or drops the unsupported claim** rather than emitting a hallucinated reference.

**Scoring uses LLM-as-judge, bias-controlled.** For the generation metrics I use an LLM judge but control for position, verbosity, and self-preference bias — randomise candidate order, don't reward length, be wary of a judge from the generator's own model family. Judge scores are bias-caveated signal, never ground truth. This three-part posture — offline gold-set gate + online guardrails + bias-controlled judge — is the "how do you know it works?" decision cue.

**Rollout:** eval-gate → canary → rollback. No change reaches 100% without clearing the offline gate, proving out on a small canary slice, and passing online guardrails — because passing offline eval ≠ safe at 100%.

#### 6. Cost model

Let me do a rough $/query, **clearly labelling unit prices as ASSUMPTIONS** (I'd use our real contracted rates in the room):

```text
ASSUMED (label as assumptions):
  simple lookup  ≈ 1 generation call; multi-hop ≈ 3–5 hops (fan-out multiplier)
  input tokens   ≈ 3k (retrieved chunks + prompt)   output ≈ 0.5k
  assumed price  ≈ $3 / 1M input,  $15 / 1M output  (output ≈ 5× input — from cheatsheet)

Simple lookup, per call:
  input : 3k × $3/1M    = $0.009
  output: 0.5k × $15/1M = $0.0075
  ≈ $0.0165 / query

Multi-hop (× ~4 fan-out): ≈ $0.066 / query
Blended (mostly lookups): rough order of a few cents/query
```

The two structural facts driving the bill: **output tokens run ~4–5× input cost**, and in multi-hop **fan-out is the silent multiplier** — each sub-question is a full paid hop. Every lever below is drawn from the **Cost row**, and each is **eval-gated** (ship only if faithfulness/answer-relevancy hold — Golden Rule 16):

- **Semantic cache** — "what's our PTO policy?" is asked thousands of times; cache semantically (not by exact string, Golden Rule 5) so equivalent questions skip generation entirely. This is the single biggest lever in a corporate FAQ workload where questions cluster hard. Cache entries are invalidated by the freshness cascade (task 3) so a cached answer is never stale past the SLA.
- **Route to the smallest model that passes eval** (Golden Rule 6) — most queries are lookups that a small model answers correctly; reserve the frontier model for hard multi-hop synthesis. Routing is gated on the golden set proving the small model holds faithfulness for the lookup class.
- **Cap `max_tokens` + attack fan-out** (Golden Rule 5, Cost row) — output is the expensive class, so bound generation length; and cap the decomposer's fan-out so multi-hop doesn't multiply hops unchecked. **Prompt-cache the stable prefix** (~10%) since the large shared system prompt + tool defs repeat every call, and route bulk/async enrichment to the **Batch API (−50%)**.

Stacking **semantic cache + model routing + max_tokens/fan-out caps** comfortably clears the 50%-reduction ask on a lookup-heavy workload, each shipped only after the eval gate proves faithfulness held — never the reflex red flag "just use a smaller model," which ignores fan-out and the eval gate entirely.

---

### How the cheatsheet was used

- **Opener → 60-Second Framework steps 1–2 (clarify & scope; name SLOs)** → restated scale/latency/quality/sensitivity/cadence/cost and turned them into defendable SLOs; **Golden Rule 16** framed the whole answer (no blind single-axis wins).
- **Task 1 → Framework step 3 "sketch architecture" (ingest → chunk → embed → index → retrieve → rerank → generate → cite)** → the RAG spine. **Golden Rule 2 (compute-bound vs queue-bound)**, **Golden Rule 1 (stream first, TTFT≠total)**, **Golden Rule 6 (route to smallest model)**, and **Scalability row (bounded queue + backpressure + load-shed, stateless + autoscale, continuous batching)** → the serving/response layer and the RAG-vs-fine-tune-vs-tool-use call.
- **Task 2 → Security row + Golden Rule 11 "LLM is never the security boundary; authz in code/DB (complete mediation, RLS)"**, **Golden Rule 12 "untrusted content is untrusted everywhere / treat output as untrusted (LLM05)"**, **Privacy row "tenant isolation via RLS / per-tenant namespaces at retrieval; app-code filters fail open"**, and **Decision Cue "cross-tenant leak → RLS at the engine, not app-code filters"** → engine-level ACL filter, no-leak-via-embeddings, fail-closed mediation.
- **Task 3 → Versioning row "incremental / dual-index cutover on embedder change; pin every version; never -latest; log (model, prompt, index) triple"**, **Privacy row "right-to-erasure cascades source → chunks → embeddings → caches"**, **Reliability row "async queue + DLQ"**, **Golden Rule 15**, and **Decision Cues "swap embedding model → dual-index cutover" / "GDPR erasure → cascade delete"** → incremental upsert, tombstone-and-cascade deletes, dual-index only on embedder change.
- **Task 4 → Golden Rule 6 (route simple vs hard)**, **Golden Rule 10 "bound every agent loop (recursion_limit; degrade/escalate)"**, **Golden Rule 4 "parallelize only independent hops"**, **Cost row "fan-out is the silent multiplier"**, and **Trade-off One-Liner "autonomy/agency vs safety"** → the router, single-call-vs-bounded-loop strategy, independent-hop parallelism, and fan-out/loop bounding.
- **Task 5 → Evaluation row (offline gold-set CI gate vs online drift; RAG split context precision/recall + faithfulness/answer-relevancy; LLM-as-judge bias-controlled)**, **Golden Rule 7 "design eval before you build"**, **Framework failure-mode step "eval-gate → canary → rollback"**, **Decision Cue "how do you know it works?"**, and **Red Flag "green dashboard but wrong answers"** → hallucinated-citation detection framed as a faithfulness/grounding eval, plus the rollout gate.
- **Task 6 → Cost row (output ~4–5× input; fan-out multiplier; cap max_tokens; route to smallest model that passes eval; prompt-cache stable prefix ~10%; semantic cache; Batch API −50%; tune k)**, **Golden Rule 5**, **Golden Rule 6**, **Golden Rule 16 (each lever eval-gated)**, **Decision Cue "keep the bill down"**, and **Red Flag "just use biggest/smaller model"** → the $/query arithmetic (assumptions labelled) and the eval-gated levers.

---

## Cheatsheet elements referenced

- **60-Second Framework** — clarify & scope, name SLOs, sketch architecture, walk request path (time/tokens/dollars per hop), sweep 9 concerns, state trade-offs, failure modes + rollout (eval-gate → canary → rollback).
- **9-Concern Sweep → Latency row** — split TTFT vs total, stream first (`stream=True`), cap max_tokens, parallelize independent hops, semantic cache, route to small model.
- **9-Concern Sweep → Scalability/Throughput row** — compute-bound vs queue-bound, replicas + autoscale, continuous batching, bounded queue + backpressure + load-shed, stateless + shared state.
- **9-Concern Sweep → Evaluation/Quality row** — offline gold-set CI gate vs online drift, RAG split (context precision/recall + faithfulness/answer-relevancy), LLM-as-judge (bias-controlled), A/B on live.
- **9-Concern Sweep → Cost/Token Efficiency row** — output ~4–5× input, fan-out as silent multiplier, cap max_tokens, route to smallest model that passes eval, prompt-cache stable prefix (~10%), semantic cache, Batch API (−50%), tune k.
- **9-Concern Sweep → Reliability/Failure row** — transient→retry+backoff+jitter, poison→DLQ, async queue + DLQ, bounded loop.
- **9-Concern Sweep → Security/Safety row** — LLM is never the security boundary, isolate untrusted content, treat output as untrusted (LLM05), least-privilege, authz in code/DB (complete mediation, RLS).
- **9-Concern Sweep → Data Privacy/Governance row** — tenant isolation via RLS / per-tenant namespaces at retrieval, right-to-erasure cascades source → chunks → embeddings → caches, app-code filters fail open.
- **9-Concern Sweep → Versioning/Change-mgmt row** — pin every version (never -latest), dual-index cutover on embedder change, log the (model, prompt, index) triple, eval-gate → canary → rollback.
- **Golden Rule 1** — stream first then optimise the tail (TTFT beats total).
- **Golden Rule 2** — classify compute-bound vs queue-bound before scaling.
- **Golden Rule 4** — parallelize only independent hops.
- **Golden Rule 5** — cache semantically not exact string; cap max_tokens (output is the expensive class).
- **Golden Rule 6** — route to smallest model that passes eval; escalate to frontier only for hard requests.
- **Golden Rule 7** — design eval before you build; eval-gate every change.
- **Golden Rule 10** — bound every agent loop (recursion_limit; degrade/escalate).
- **Golden Rule 11** — never let the model be the security boundary — authz in code/DB (complete mediation, RLS).
- **Golden Rule 12** — untrusted content is untrusted everywhere; isolate retrieved/tool content, screen outputs.
- **Golden Rule 15** — pin versions; never -latest; log the (model, prompt, index) triple.
- **Golden Rule 16** — never optimise one axis blind; a cost/latency win is real only if accuracy and safety held.
- **Decision Cue: "how do you know it works?"** — offline gold-set gate + online guardrail metrics + LLM-as-judge (bias caveats).
- **Decision Cue: "keep the bill down."** — model routing + prompt caching + semantic cache + cap max_tokens; Batch API async.
- **Decision Cue: "cross-tenant leak."** — RLS/per-tenant namespaces at the engine, not app-code filters.
- **Decision Cue: "GDPR erasure."** — cascade delete → chunks → embeddings → caches, re-index.
- **Decision Cue: "swap embedding model."** — dual-index cutover; flip query embedder + reader in lockstep.
- **Decision Cue: "ship a new model safely."** — eval gate → canary → instant rollback.
- **Decision Cue: "green dashboard but wrong answers."** — online quality + drift; per-step spans not endpoint logs.
- **Trade-off One-Liner: cost vs quality** — gate every cut behind eval.
- **Trade-off One-Liner: recall vs cost/latency of large-k retrieval** — smallest k that holds recall.
- **Trade-off One-Liner: autonomy/agency vs safety** — least-privilege + bounded loops.
- **Trade-off One-Liner: coverage vs bounded tail** — load-shed to hold the tail.
- **Red Flag: "just use biggest/smaller model"** — reflex that ignores fan-out and the eval gate.
- **Red Flag: "the LLM checks permissions" / "strong system prompt handles security"** — the model is never the security boundary.
- **Red Flag: "we'll add eval/monitoring later"** — eval is the instrument you steer by.
