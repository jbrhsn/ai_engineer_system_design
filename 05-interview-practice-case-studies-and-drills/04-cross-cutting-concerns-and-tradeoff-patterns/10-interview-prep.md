# Cross-Cutting Concerns & Trade-off Patterns — Interview Prep

**Section:** 05 Interview Practice → Cross-Cutting Concerns & Trade-off Patterns | **Role target:** Applied AI Engineer / AI Systems Engineer (agentic AI, RAG, multi-agent production systems)

> **How to read this file.** This is the *synthesis* companion for the eight concern-axis notes in this chapter — latency (`01`), scalability (`02`), evaluation (`03`), cost (`04`), reliability (`05`), security (`06`), privacy (`07`), and observability (`08`). Where those notes each drill one axis in depth, this file drills the interview skill that actually decides loops: **recognizing which cross-cutting concern a prompt is really probing, and answering with the right trade-off instead of one-axis tunnel vision.** Almost every senior follow-up ("make it faster," "how does it scale," "how do you know it works," "keep it affordable," "what happens when it fails," "keep it secure," "handle the PII," "debug it live") is one of these eight axes in disguise — and the strongest answers name the tension *between* them out loud.

---

## Core Conceptual Questions

These test whether you can identify the axis a question probes and answer with its governing distinction — not a generic "it depends." One row per concern axis.

| Question | Key points to cover | Common weak-answer trap |
|---|---|---|
| **(Latency)** A chat assistant "feels slow" — how do you diagnose and fix it? | Split end-to-end latency into **time-to-first-token (TTFT)** and total completion time; for interactive UX, TTFT is what the user feels. Stream first (`stream=True` over SSE) — near-zero cost, no quality risk — then attack the tail (cap `max_tokens`, trim retrieval-`k`/prefill, parallelize *independent* hops, route easy queries to a small model). Always state a p95/p99 budget before proposing fixes. See `01-latency-and-response-time-patterns.md`. | "Use a smaller/faster model" as the reflex — swapping quality for speed before checking whether the complaint is *silence* (TTFT) that streaming fixes for free. |
| **(Scalability)** How does this handle a 50× traffic spike? | First classify: **compute-bound** (GPU saturated even at low concurrency → dynamic/continuous batching, add replicas) vs **queue-bound** (utilization has headroom, tail explodes as RPS climbs → add capacity, load-shed, decouple). Read utilization to tell them apart. Bounded queue + backpressure + load shedding (429), stateless replicas over shared state, per-tenant **token**-based rate limiting, async queue + DLQ for bursty/long jobs. See `02-scalability-and-throughput-patterns.md`. | Reflexively "add more GPUs" for every load problem — wasting money on a queue-bound spike batching or rate-limiting would fix. |
| **(Evaluation)** How do you know the system works before and after you ship? | **Offline** (curated golden dataset with reference outputs, pre-deploy, gates CI) vs **online** (live traffic, no reference, catches drift). For RAG decompose into retrieval metrics (context precision/recall) and generation metrics (faithfulness, answer relevancy) so a low score localizes the fault. Define metrics + golden set *before* building; wire a regression gate; feed failing live runs back into the golden set. See `03-evaluation-and-quality-assurance-patterns.md`. | "We'll eyeball a few answers" / "add eval later" — no regression gate means you can't safely change a prompt, model, or corpus, and can't prove a cost cut didn't degrade quality. |
| **(Cost)** How do you keep this affordable at scale without regressing quality? | **Output tokens are the expensive class (~4–5× input); agent fan-out is the silent multiplier.** Levers: cap `max_tokens`, route to the smallest model that passes eval, prompt-cache stable prefixes (~10% of input rate), semantic-cache repeated queries (avoids the call entirely), Batch API (−50%) for async work, tune retrieval-`k`, bound agent steps. Gate every cut behind the eval suite. Attribute cost per request and per tenant. See `04-cost-and-token-efficiency-patterns.md`. | "Just use a cheaper model" applied blind — cutting cost with no eval gate to catch the silent drop in goal-accuracy, and ignoring that fan-out and output length dominate the bill. |
| **(Reliability)** What happens when the model, tool, or provider fails? | Classify the failure: **transient** (429/529/timeout → retry with exponential backoff **+ jitter**, honor `retry-after`), **persistent** (400/404 → fall back, don't burn quota), **poison** (→ dead-letter queue). Layer retry → timeout → circuit breaker → fallback chain → DLQ. AI-specific twists: schema-validate + repair non-deterministic output; cap agent recursion with a human escape hatch. **Never retry a side-effecting call without an idempotency key.** See `05-reliability-and-failure-handling-patterns.md`. | "Just retry it" for everything — retrying a persistent error wastes quota, and retrying a write with no idempotency key double-charges the card / double-sends the message. |
| **(Security)** How do you secure an agent that has tool access? | **The LLM is untrusted and can never be the security boundary.** OWASP LLM01 injection (direct vs *indirect* via retrieved/tool content) triggers LLM06 excessive agency. Defense-in-depth: isolate untrusted content in `tool_result` blocks, treat model output as untrusted (LLM05), least-privilege tool scopes, enforce authz in code/DB (**complete mediation**, e.g. RLS on the authenticated user), sandbox execution, HITL before irreversible actions. See `06-security-and-safety-patterns.md`. | "A strong system prompt stops injection" — OWASP is explicit there's no fool-proof prompt-level fix; treating the model's own reasoning as the authz decision point is the core error. |
| **(Privacy)** A design touches customer records across tenants — how do you handle PII, isolation, and deletion? | Control **flow, scope, and lifetime** of sensitive data. Confinement: redact PII *before* the prompt **and** before the log (two enforcement points), tenant isolation via RLS / per-tenant namespaces enforced at retrieval, region pinning, zero-retention endpoints. Lifecycle: retention TTLs and **right-to-erasure** that cascades source → chunks → embeddings → caches. Distinct from attack-defense (`06`) — this is data handling/compliance. See `07-data-privacy-and-governance-patterns.md`. | "We redact the prompt" while logging the raw request — confinement solved at the model, broken at the log; and forgetting that a deleted document's embedding still answers queries. |
| **(Observability)** It's live and giving wrong answers — how do you find which step broke? | One **trace** per request, one **span** per step (retrieve → tool → LLM), tied by a propagated trace ID, so you can replay a bad answer step-by-step. Layer LLM-specific signals the three pillars miss: token counts, cost, cache-hit rate, tool-call success, guardrail/hallucination flags (OTel GenAI conventions: `gen_ai.usage.input_tokens`, etc.). Alert on p95/error/cost; monitor quality + drift online; capture thumbs up/down to feed offline eval. See `08-observability-and-monitoring-patterns.md`. | "Check the logs" where the log records only the final answer — you cannot debug an agent from the endpoint output; you need per-step spans under a shared trace ID. |

---

## Applied / Scenario Questions

These are the questions that *look* like one axis but are really about the tension between several. Each strong-answer framework names the explicit **latency vs. accuracy vs. cost vs. safety** trade-off.

**Q1:** Your production RAG agent's monthly bill jumped 40% with no traffic increase, *and* p99 latency doubled during business hours. Leadership wants both fixed without hurting answer quality. Diagnose across axes.

**Strong answer framework:**
- **Instrument before you touch anything (observability first).** This looks like two problems (cost + latency) but they usually share a root cause. Pull traces: per-request token spend, step count, per-hop latency, cache-hit rate. An unbounded agent loop that fans out redundant retrievals inflates *both* the bill (call-count cost) and the tail (extra sequential hops). You cannot pick the right lever until a trace tells you where tokens and milliseconds actually go.
- **Separate compute-bound from queue-bound for the latency half.** If GPU utilization is near 100% during business hours it's compute-bound — batching/replicas. If utilization has headroom and the tail only spikes as RPS climbs, it's queue-bound — bounded queue, load shedding, per-tenant token rate limiting. Don't add GPUs to a queueing problem.
- **Attack cost by class.** Output tokens are the expensive class — cap `max_tokens`; route intent classification/extraction to a small model and reserve the frontier model for synthesis; prompt-cache the stable system prompt/tool prefix; semantic-cache repeated queries to avoid the call entirely; bound agent steps to kill fan-out.
- **Trade-off (latency/accuracy/cost/safety):** Every cost lever (fewer steps, smaller models, aggressive context trimming) risks lowering agent-goal-accuracy, and step caps can truncate a legitimately long task. Resolve it by gating *every* cut behind the offline eval suite so you reduce cost and tail latency *without* silently regressing faithfulness or task success — the axes are coupled, so you prove the whole set moved the right way, not just the one you optimized.

---

**Q2:** Design the eval + guardrail + privacy strategy for a new customer-facing support agent that can look up account data and draft (but not send) emails, serving multiple enterprise tenants.

**Strong answer framework:**
- **Eval, split by pipeline stage (evaluation axis).** Retrieval: context precision + recall. Generation: faithfulness + answer relevancy. Offline golden-set gate in CI blocks releases; online reference-free monitoring catches drift the offline set never saw; failing live runs get promoted into the golden set.
- **Guardrails as defense-in-depth (security axis).** Input: moderation + injection screen. Retrieval channel: treat retrieved content as untrusted (indirect-injection surface), deliver it in `tool_result` blocks. Tool side: account lookup runs read-only under the *authenticated user's* security context (least privilege + complete mediation in the DB, not the LLM); email drafting is autonomous but **sending requires human approval** (HITL before the irreversible action). Output: validate/sanitize before render (LLM05), run a faithfulness gate before display.
- **Privacy and tenancy (privacy axis, distinct from security).** Redact PII before the prompt *and* before logs; isolate tenants at retrieval with per-tenant namespaces or RLS so tenant B never surfaces tenant A's chunks; region-pin regulated data; make deletion cascade to embeddings; consider a zero-retention model endpoint for sensitive fields.
- **Trade-off (latency/accuracy/cost/safety):** Each guardrail and redaction layer adds latency and engineering surface, HITL adds human cost, and tenant isolation adds index/partition management — so gate HITL only on the irreversible action (send) while read-only lookups stay autonomous, and accept some false abstentions to keep fabrication near zero. You trade a slightly less "magical" hands-off assistant for a bounded blast radius and a provable compliance story.

---

**Q3:** During a launch spike your agent starts returning malformed answers and timing out; the on-call channel is silent because "everything looks green." Walk through reliability + observability together.

**Strong answer framework:**
- **The silence is the first bug (observability axis).** "Green" with user-visible failures means you're monitoring the endpoint, not the pipeline. You need per-step spans under a shared trace ID plus alerts on the *right* signals: p95 latency > SLO, error rate, tool-call failure rate, guardrail-flag rate, and cost spikes — not just HTTP 200s. Malformed output and timeouts are span-level events the endpoint log hides.
- **Classify the failures (reliability axis).** Timeouts under a spike are likely queue-bound (a scalability signal) — bound the queue and load-shed rather than let it grow. Malformed JSON is LLM non-determinism — add schema validation + a repair step, don't treat a 200 as success. Provider 429/529 are transient — retry with backoff + jitter honoring `retry-after`, and trip a circuit breaker so you fail fast to a fallback (alternate model → cache → degraded response) instead of hammering a sick dependency.
- **Protect side effects.** If the agent performs actions (refund, send), every retry path must carry an **idempotency key** so a timeout-then-retry doesn't double-execute.
- **Trade-off (latency/accuracy/cost/safety):** Aggressive retries and fallbacks raise reliability but add latency and cost (more calls, slower model in the fallback), and load-shedding trades *coverage* (some users get 429) for a *bounded tail* for everyone accepted. Resolve it by shedding early and cheaply, capping retry budget, and reserving the expensive fallback for high-value requests — reliability that ignores the cost/latency cost of every retry is just a slower way to fail.

---

## System Design / Architecture Questions

**Q1:** Design a production-grade RAG agent for enterprise customer support: it answers policy/product questions, looks up account state, and can perform a small set of account actions. It's multi-tenant, has a p95 latency SLO and a monthly cost ceiling, and handles regulated PII. Defend it across all eight concern axes.

**Approach:**

1. **Clarify requirements (which axes are load-bearing here).** What's the p95/p99 latency SLO and peak concurrency (latency + scalability)? What's the hallucination tolerance — advisory answers or binding policy that demands near-zero fabrication and abstention (evaluation + safety)? What data sensitivity, tenancy isolation, region, and deletion obligations (privacy)? Is there a hard monthly token budget (cost)? What actions are irreversible (reliability + security)? These answers decide topology, guardrail strictness, and where HITL and abstention sit — you can't optimize all eight to the max, so surface which ones are binding constraints vs. best-effort.

2. **Propose the architecture, mapped axis by axis.**
   - *Topology + latency:* A **supervisor** routes intents to a Q&A path (RAG), a read-only account-lookup path, and an action path. Stream tokens for perceived latency; parallelize independent retrievals; bound every agent loop with a max-step limit.
   - *Scalability:* Stateless replicas over shared state (vector DB, Redis) with autoscaling; a **bounded queue with backpressure/load-shedding**; **continuous batching** at the inference tier; per-tenant **token**-based rate limiting; async queue + DLQ for bulk ingestion off the interactive path.
   - *Evaluation:* Ragas metrics (context precision/recall, faithfulness, answer relevancy) as an offline golden-set **regression gate** in CI, plus online reference-free monitoring; failing live runs feed back into the golden set.
   - *Cost:* Model routing (small for classification, frontier for synthesis), prompt caching on the stable prefix, semantic cache on repeated queries, `max_tokens` caps, retrieval-`k` tuning, per-tenant cost attribution; Batch API for nightly evals.
   - *Reliability:* retry+backoff+jitter, timeouts, circuit breaker, fallback chain, DLQ; schema-validate structured output; **idempotency keys** on the action tools; agent recursion cap with a human escape hatch.
   - *Security:* LLM is untrusted — input moderation/injection screen, retrieved content isolated in `tool_result` blocks, output treated as untrusted (LLM05), least-privilege tool scopes, **authz in the DB (complete mediation) on the authenticated user**, sandbox for any execution, **HITL before the irreversible action**, no secrets in the system prompt (LLM07).
   - *Privacy:* PII redaction before prompt *and* logs, per-tenant isolation (namespaces/RLS) enforced at retrieval, region pinning, retention TTLs, right-to-erasure cascading to embeddings.
   - *Observability:* one trace / request, one span / step under a propagated trace ID, carrying token/cost/latency/cache-hit/guardrail attributes; dashboards + alerts on p95, error rate, cost, guardrail-flag rate; feedback capture closing the loop into eval.

3. **Justify choices and name trade-offs explicitly.** Supervisor over a swarm because centralized routing is cheaper (multi-agent runs ~15× chat tokens) and easier to trace — pay for isolation only where a driver exists. Hybrid retrieval + rerank buys precision at a bounded latency cost. The abstention + HITL layers trade coverage and latency for safety on binding-policy answers and irreversible actions — a deliberate choice given data sensitivity. Per-tenant token rate limiting trades some legitimate bursts for fairness and a bounded bill. Tracing + the eval gate add engineering surface but are the *only* way to cut cost (routing, step caps) or scale (shedding) without silently regressing quality — which is the through-line across all eight axes: **you never optimize one axis blind; you gate every change on the metrics for the others.** Net: the design optimizes for *defensible* answers within budget and compliance, not maximal autonomy.

---

**Q2:** You're told to cut inference cost by 50% on an existing agent. Frame this as a cross-cutting design decision, not a single knob.

**Approach:**

1. **Clarify the constraint set.** What's the current cost breakdown (input vs output vs cached vs call fan-out)? What's the eval baseline, and what quality regression is acceptable (evaluation)? What's the latency SLO you must not break (latency)? Any actions where a cheaper/misrouted model raises safety risk (security)? A 50% cut is trivially achievable by degrading quality — the real question is 50% *at held quality*, which couples cost to eval.

2. **Propose the levers in priority order, by cost class.** Output cost: cap `max_tokens`, prompt for concision. Call-count cost: bound agent steps, add a semantic cache for repeated queries, route easy traffic to a small model. Input cost: prompt-cache the stable prefix, tune retrieval-`k`. Async work: move nightly evals/backfills to the Batch API (−50%). Sequence them by leverage-per-risk: output caps and caching first (low quality risk), model routing next (needs a router eval), distillation last (upfront cost, only if volume amortizes it).

3. **Justify and name trade-offs.** Each lever trades against another axis: `max_tokens` too tight truncates answers (accuracy); routing risks misroutes (accuracy) and adds a classifier call (latency/cost); semantic-cache thresholds too loose return stale/wrong answers (accuracy + safety); aggressive context trimming drops recall (accuracy). The discipline that makes a 50% cut *safe* is the **eval gate + cost observability**: attribute spend per request/tenant to know what you're cutting, and run every candidate config against the golden set so you prove faithfulness and answer-relevancy held while cost fell. Cost is not a standalone axis — it's the axis most likely to silently borrow from accuracy and safety, so it's the one you must instrument hardest.

---

## Vocabulary That Signals Expertise

Use these naturally to show you reason across all eight axes — don't force them:

- **Time-to-first-token (TTFT)** — invoke on any latency question; signals you separate *perceived* latency (what streaming fixes) from total completion time, instead of reflexively shrinking the model.
- **Compute-bound vs. queue-bound** — name when asked about scaling/spikes; signals you read utilization to pick the fix (batch/replicas vs. shed/decouple) rather than always "adding GPUs."
- **Backpressure / load shedding** — use for graceful degradation under load; signals you know an unbounded queue is a bug and that refusing 5% fast beats accepting 100% and failing all slowly.
- **Continuous batching** — invoke for inference-tier throughput under concurrency (vLLM iteration-level batching); signals you scale below the app layer, not just add replicas.
- **Faithfulness vs. answer relevancy** — pair them to show grounding-in-context is measurable and distinct from *truth* and from *intent*; signals you decompose RAG quality by component before blaming the generator.
- **Idempotency key** — name whenever retries touch side effects; signals you know a retry is only *safe* on a read or on a write the server dedupes — the single most-missed reliability detail.
- **Complete mediation** — use for authz on tool calls; signals OWASP fluency and that you enforce access in code/DB on the authenticated user, never letting the LLM be the security boundary.
- **Excessive agency (LLM06)** — cite its three roots (functionality, permissions, autonomy); signals you shrink the *blast radius* of injection you can't fully prevent, rather than trusting a system prompt.
- **Right-to-erasure / cascade delete** — invoke on privacy; signals you know deletion must reach embeddings and caches, not just the source row — governance you can *prove*.
- **Span / trace ID** — name distributed tracing across agent hops with token/cost attributes (OTel GenAI conventions); signals you can debug and cost-attribute a live multi-step system, not just read the endpoint log.

---

## Vocabulary That Signals Weakness

Avoid these — each reveals one-axis tunnel vision or a shallow model:

- **"Just use a bigger/smaller model"** — red flag: the reflex answer to every axis at once. It ignores that the complaint might be TTFT (streaming), queueing (scaling), recall (retrieval), or fan-out (cost) — none of which a model swap fixes, and all of which it can silently regress.
- **"We'll add eval/monitoring later"** — red flag: eval and observability are the *instruments you steer the other six axes by*. Without them you can't cut cost, shed load, or harden security and *prove* you didn't break quality — so "later" means "blind."
- **"A strong system prompt handles security"** — red flag: OWASP LLM01 is explicit there's no fool-proof prompt-level fix, and it treats the untrusted, manipulable LLM as the security boundary — the exact inversion of complete mediation.
- **"Redaction/PII is handled at the model"** — red flag: solves confinement at one enforcement point and forgets the log, the shared index, and the leftover embedding after a deletion — treating privacy as a single step instead of a flow-scope-lifetime problem.

---

## STAR Answer Frame

**Situation:** A multi-tenant production support agent was quietly hallucinating on binding-policy answers, its monthly inference bill had climbed ~40% quarter-over-quarter with flat traffic, and during business-hours spikes its p99 latency doubled while the on-call dashboard stayed green. Leadership wanted quality, cost, and latency fixed at once — without a compliance incident on tenant data.

**Task:** I owned hardening groundedness, bringing spend under the ceiling, and stabilizing tail latency, while *proving with data* that no fix regressed answer quality or leaked cross-tenant data.

**Action:** I started with observability because the dashboard's silence was itself a bug — I added distributed tracing (one span per hop with token/cost/latency/cache-hit attributes) and a Ragas eval harness (context precision/recall + faithfulness + answer relevancy) run offline on a golden set and reference-free on live traffic. Tracing showed one root cause behind *three* symptoms: an unbounded agent loop fanned out redundant retrievals, inflating tokens (cost), adding sequential hops (latency), and dragging in irrelevant context (hallucination). I capped agent steps, routed intent classification to a small model while reserving the frontier model for synthesis, prompt-cached the stable prefix, and added a semantic cache. For the tail I found the spike was queue-bound (GPU headroom), so I added a bounded queue with load shedding and per-tenant token rate limiting rather than more GPUs. For groundedness I pinned temperature 0, enforced citations, and routed low-confidence answers to abstention. For safety I confirmed account lookups ran read-only under the authenticated user with authz in the DB, and PII was redacted before both prompt and logs. Every change shipped behind the offline eval gate.

**Result:** Faithfulness recovered from 0.71 to 0.94, hallucinated policy answers dropped from ~15% to under 2%, monthly token spend fell ~35%, and business-hours p99 came back within SLO — with answer-relevancy and cross-tenant isolation holding flat, proving the cost and latency cuts didn't quietly borrow from quality or safety.

---

## Red Flags Interviewers Watch For

Specific to cross-cutting concerns — an interviewer downgrades a candidate who:

- **Optimizes one axis in isolation** — pushes latency or cost hard while ignoring what it borrows from accuracy or safety (e.g. caps `max_tokens` with no eval, sheds load with no fairness story). The whole skill this chapter drills is that the eight axes are coupled — a win on one is only real if you prove the others held.
- **Misreads the axis being probed** — hears "make it faster" and shrinks the model when the fix is streaming (TTFT); hears "handle the spike" and adds GPUs when it's queue-bound; hears "it's giving wrong answers in prod" and greps the endpoint log instead of tracing spans. Naming the right axis first is half the answer.
- **Treats the model as the security boundary** — designs tool access with authz "in the prompt," trusts a system prompt to stop injection, or never mentions indirect injection, least privilege, complete mediation, or HITL on irreversible actions.
- **Has no eval before ship** — proposes an architecture but can't say how they'd *measure* it or gate a regression, or names "vibes"/manual spot-checks instead of retriever/generator metrics and a golden-set gate.
- **Ignores tail latency and graceful degradation** — reasons about average latency and happy-path throughput but not p95/p99 under load, bounded queues, backpressure, or what the system does when a dependency is sick (retry/circuit-breaker/fallback/DLQ).
- **Retries without idempotency, or scales without statelessness** — adds retries to side-effecting calls with no idempotency key (double-effects), or "adds replicas" while holding session/cache state inside the replica (breaks horizontal scaling).
- **Conflates privacy with security, or forgets data lifecycle** — treats PII as an attack-surface afterthought, redacts the prompt but logs it raw, or has no answer for tenant isolation, retention, or a deletion request that must reach the embeddings.
- **Can't debug the system they designed** — no tracing across hops, no per-request cost attribution, no online quality/drift monitoring, no feedback loop back into eval — meaning they could ship it but never operate it.
