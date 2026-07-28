# Golden Rulebook & Cheatsheet — Answering Any AI System Design Question

**Section:** 05 Interview Practice → Cross-Cutting Concerns & Trade-off Patterns | **Use:** last-minute skim + in-interview quick reference. Synthesises notes 01–09 of this chapter.

---

## How to Use This Sheet

This is the index / quick-reference for the nine pattern notes (`01`–`09`) and the delivery framework — not a new deep-dive. Drill the framework loop and the 9-Concern Sweep until they are reflexive, so under time pressure you name the right axis and its trade-off automatically. Every row links to the deep note; when a follow-up goes deeper than a cell, jump to that file.

---

## The 60-Second Framework

A repeatable loop for attacking any prompt. Maps onto the delivery-framework flow **clarify → scope → drive → communicate** (`01-delivery-framework-.../01-*.md`).

1. **Clarify & scope** — ask the questions that *change the design* (scale, latency, quality bar, cost, data sensitivity, tenancy, change cadence); assume-and-state the ones that only change a number. State a **business objective** and a measurable **AI objective**.
2. **Name constraints / SLOs** — put numbers on it: p95/p99 latency, peak concurrency, hallucination tolerance, monthly cost ceiling, region/deletion obligations.
3. **Sketch high-level architecture** — a rough block diagram, whole lifecycle (ingest → chunk → embed → index → retrieve → rerank → generate → cite / act). Do not beautify it.
4. **Walk the request path** — narrate where each hop spends time, tokens, and dollars; this exposes the component worth a deep dive.
5. **Sweep the 9 concerns** — run the master table below as follow-up-proofing; raise eval, cost, safety **unprompted** (that is the depth signal).
6. **State trade-offs out loud** — name the tension you are accepting ("X beats Y under this constraint"); the axes are coupled, so prove the others held.
7. **Call failure modes + rollout** — retry/circuit-breaker/fallback/DLQ, idempotency on side effects, and eval-gate → canary → rollback for any change.

**Drive by default; yield the instant the interviewer redirects.** Depth is scored, not diagram polish.

---

## Clarifying Questions Checklist

Ask up front; each group maps onto a concern axis so no axis is forgotten.

| Group                            | Ask…                                                                         | Maps to                |
| -------------------------------- | ----------------------------------------------------------------------------- | ---------------------- |
| **Scale / throughput**     | Peak concurrency? RPS? bursty (Black-Friday) or steady? corpus size?          | Scalability (`02`)   |
| **Latency budget**         | p95/p99 SLO? interactive chat or batch? is*silence* (TTFT) the complaint?   | Latency (`01`)       |
| **Accuracy / quality bar** | Hallucination tolerance — advisory or binding policy? abstain-or-answer?     | Evaluation (`03`)    |
| **Cost budget**            | Monthly token ceiling? cost-per-request / per-tenant target? agent fan-out?   | Cost (`04`)          |
| **Reliability**            | Which actions are irreversible? provider-outage expectation? at-least-once?   | Reliability (`05`)   |
| **Security / tenancy**     | Tool access? untrusted/retrieved content? who is the authenticated principal? | Security (`06`)      |
| **Data sensitivity**       | PII? multi-tenant isolation? region/residency? deletion / erasure duty?       | Privacy (`07`)       |
| **Change cadence**         | How often do model/prompt/index change? rollback needs? reproducibility?      | Versioning (`09`)    |
| **Debuggability**          | How do you find which step broke live? feedback signal available?             | Observability (`08`) |

---

## The 9-Concern Sweep — Master Table

The centrepiece. One row per concern; run it as a checklist after the happy path is drawn.

| Concern                              | Interviewer's trigger question                    | Go-to resolution options (terse)                                                                                                                                                                                                                                                 | Trade-off to say out loud                                                                                                 | Deep note                                            |
| ------------------------------------ | ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| **Latency**                    | "It feels slow / frozen — make it faster."       | Split**TTFT vs total**; stream first (`stream=True`); cap `max_tokens`; parallelize *independent* hops; semantic cache; route to small model                                                                                                                         | Perceived vs actual latency; streaming cuts feel not compute; smaller model risks quality                                 | [01](01-latency-and-response-time-patterns.md)        |
| **Scalability / Throughput**   | "How does it handle a 50× spike?"                | Classify **compute-bound vs queue-bound** (read utilization); replicas + autoscale; dynamic/continuous batching; bounded queue + backpressure + load-shed; per-tenant **token** rate limit; async queue + DLQ; stateless + shared state                             | Adding GPUs wastes money on a queue-bound spike; batch latency vs throughput; shed coverage for a bounded tail            | [02](02-scalability-and-throughput-patterns.md)       |
| **Evaluation / Quality**       | "How do you know it works?"                       | **Offline** golden set + reference (CI gate) vs **online** (no reference, drift); RAG split: context precision/recall + faithfulness/answer-relevancy; LLM-as-judge (bias-controlled); A/B on live                                                                   | Judge bias (position/verbosity/self-preference); offline can't cover live distribution; velocity vs a maintained eval set | [03](03-evaluation-and-quality-assurance-patterns.md) |
| **Cost / Token Efficiency**    | "Keep it affordable at scale."                    | **Output tokens ~4–5× input; fan-out is the silent multiplier**; cap `max_tokens`; route to smallest model that passes eval; prompt-cache stable prefix (~10%); semantic cache (avoids call); Batch API (−50%); tune `k`                                            | Every cut risks accuracy; must eval-gate; distillation trades upfront cost                                                | [04](04-cost-and-token-efficiency-patterns.md)        |
| **Reliability / Failure**      | "What when the model/tool/provider fails?"        | Classify**transient → retry+backoff+jitter** (honor `retry-after`), **persistent → fallback chain**, **poison → DLQ**; timeout; circuit breaker; **idempotency key**; schema validate+repair; bounded loop + HITL                                   | Retries add latency/cost; fallback lowers quality; step cap too low truncates a valid task                                | [05](05-reliability-and-failure-handling-patterns.md) |
| **Security / Safety**          | "Secure an agent with tool access."               | **LLM is never the security boundary**; input moderation/injection screen; isolate untrusted content in `tool_result`; treat output as untrusted (LLM05); least-privilege tools; **authz in code/DB (complete mediation, RLS)**; sandbox; HITL before irreversible | Guardrails add latency/friction; no fool-proof prompt fix — limit blast radius, not prevent                              | [06](06-security-and-safety-patterns.md)              |
| **Data Privacy / Governance**  | "Handle PII, isolation, deletion."                | Redact PII**before prompt AND before log**; tenant isolation via RLS / per-tenant namespaces at retrieval; region-pin; retention TTL / zero-retention; **right-to-erasure cascades** source → chunks → embeddings → caches                                        | Utility/simplicity vs exposure; "no training" ≠ "no retention"; app-code filters fail open                               | [07](07-data-privacy-and-governance-patterns.md)      |
| **Observability / Monitoring** | "It's live giving wrong answers — which step?"   | One**trace per request, one span per step** under a propagated trace ID; LLM signals: tokens/cost/cache-hit/tool-success/guardrail flags (OTel GenAI); alert p95/error/cost; online quality + drift; feedback → eval                                                      | Sampling vs full capture cost; PII in captured prompts; metric cardinality blow-up                                        | [08](08-observability-and-monitoring-patterns.md)     |
| **Versioning / Change-mgmt**   | "Ship a change safely / reproduce a past answer." | **Pin every version** (model/prompt/index/schema) — never `-latest`; **eval-gate → canary → rollback**; dual-index cutover on embedder change; log the `(model, prompt, index)` triple                                                                        | Velocity vs reproducibility; rollout cost vs blast radius; passing offline eval ≠ safe at 100%                           | [09](09-versioning-and-change-management-patterns.md) |

---

## Golden Rules

1. **Stream first, then optimise the tail** — TTFT beats total time for interactive UX.
2. **Classify before you fix** — compute-bound vs queue-bound; read utilization before adding GPUs.
3. **Every queue must be bounded** — shed load (429) or apply backpressure; an unbounded queue is an OOM waiting to happen.
4. **Parallelize only independent hops** — a dependent chain (retrieve → generate) stays serial.
5. **Cache semantically, not by exact string** — and cap `max_tokens`; output is the expensive token class.
6. **Route to the smallest model that passes eval** — escalate to the frontier model only for genuinely hard requests.
7. **Design eval before you build, not after** — the golden set and metrics come first; eval-gate every model/prompt/corpus change.
8. **Idempotency turns at-least-once into exactly-once** — never retry a side-effecting call without a stable key.
9. **A 200 isn't always success** — check `stop_reason`, validate the schema, repair (bounded) or fall back.
10. **Bound every agent loop** — set `recursion_limit`; degrade/escalate proactively before it throws.
11. **Never let the model be the security boundary** — enforce authz in code/DB (complete mediation, RLS) on the authenticated user.
12. **Untrusted content is untrusted everywhere** — isolate retrieved/tool content, and screen outputs, not just user input.
13. **Redact PII before the prompt AND before the log** — and make deletion cascade to embeddings, or the "forgotten" record still answers.
14. **You can't debug what you didn't trace** — one span per step under a shared trace ID.
15. **Pin versions; never float to `-latest` in prod** — log the `(model, prompt, index)` triple so behaviour is reproducible and reversible.
16. **You never optimise one axis blind** — a win on cost/latency is only real if you prove accuracy and safety held.

---

## Decision Cues — "If the interviewer says X, reach for Y"

| If the interviewer says…             | Reach for…                                                                                             |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| "feels frozen / hangs"                | Streaming (`stream=True`); TTFT ≠ total time                                                         |
| "handle a Black-Friday burst"         | Bounded queue + autoscale + backpressure/load-shed; classify queue-bound first                          |
| "tail latency explodes under load"    | Read utilization → batch/replicas (compute) vs shed/rate-limit/decouple (queue)                        |
| "how do you know it works?"           | Offline gold-set gate + online guardrail metrics + LLM-as-judge (with bias caveats)                     |
| "keep the bill down"                  | Model routing + prompt caching + semantic cache + cap`max_tokens`; Batch API for async                |
| "one tenant is starving the others"   | Per-tenant**token**-based rate limiting + decouple bulk work to a queue                           |
| "the provider had an outage"          | Retry+backoff+jitter → circuit breaker → fallback chain (model → provider → cache → degraded)      |
| "duplicate orders / double charges"   | Idempotency key derived from a stable business ID                                                       |
| "malformed JSON from the model"       | Schema validate + bounded repair; don't trust a 200                                                     |
| "prompt injection"                    | Treat output as untrusted; isolate content in`tool_result`; least-privilege tools; complete mediation |
| "cross-tenant data leak"              | RLS / per-tenant namespaces enforced at the engine, not app-code filters                                |
| "GDPR / right to erasure"             | Cascade delete → chunks → embeddings → caches, then re-index; audit trail                            |
| "which version produced this answer?" | Trace with`(model, prompt, index)` version triple logged per output                                   |
| "ship a new model safely"             | Eval gate → canary (small %, ramp) → instant rollback; keep prev version revertible                   |
| "swap the embedding model"            | Dual-index cutover; flip query embedder + reader in lockstep                                            |
| "green dashboard but wrong answers"   | Online quality + drift monitoring; per-step spans, not endpoint logs                                    |

---

## Trade-off One-Liners

- **Latency vs accuracy** — a smaller/faster model or tighter context cuts latency but risks quality; stream to fix *feel* before trading quality.
- **Cost vs quality** — output caps, small models, and context trimming cut spend but can lower goal-accuracy; gate every cut behind eval.
- **Velocity vs reproducibility** — floating to `-latest` ships faster but loses the ability to reproduce/revert; pin and eval-gate instead.
- **Recall vs cost/latency of large-k retrieval** — more chunks lift recall but inflate prefill tokens, latency, and bill; use the smallest `k` that holds recall.
- **Autonomy/agency vs safety** — more tool freedom is more useful but widens injection blast radius; least-privilege + HITL on irreversible actions.
- **Coverage vs bounded tail** — load-shedding refuses some requests to keep p99 bounded for those accepted.

---

## Red Flags to Avoid Saying

- **"Just make it faster."** — no p95/p99, no TTFT-vs-total split.
- **"Just use the biggest / a smaller model."** — the reflex answer to every axis; ignores TTFT, queueing, recall, fan-out.
- **"A strong system prompt handles security."** — the LLM is the manipulable component; authz belongs in code/DB.
- **"The LLM checks permissions."** — inverts complete mediation; injection authorizes the attacker.
- **"We'll add eval / monitoring later."** — eval and observability are the instruments you steer every other axis by.
- **"Just retry until it works."** — retrying a persistent 4xx wastes quota; retrying a side effect with no idempotency key double-fires.
- **"We redact the prompt."** (but log it raw) — confinement solved at the model, broken at the log; forgets the leftover embedding.
- **"Just add more GPUs."** — for a queue-bound spike, wastes budget on capacity that's already underused.
- **"Check the logs."** — when the log records only the final answer; you need per-step spans.

---

## Pre-Interview 2-Minute Scan

**Loop (one line):** Clarify & scope (state numbers) → name SLOs → sketch high-level → walk the request path → sweep the 9 concerns → state trade-offs → call failure modes + rollout. Drive; yield when redirected.

**The 9 concerns (mnemonic — "Let Scaling Engineers Cut Risk, Secure Private Observable Versions"):**
Latency · Scalability · Evaluation · Cost · Reliability · Security · Privacy · Observability · Versioning

**Top 5 golden rules:**

1. Stream first, then optimise the tail (TTFT beats total).
2. Classify compute-bound vs queue-bound before you scale.
3. Never let the model be the security boundary — authz in code/DB.
4. Design eval before you build; eval-gate every change, then canary.
5. Idempotency key on any side-effecting retry; pin every version, never `-latest`.
