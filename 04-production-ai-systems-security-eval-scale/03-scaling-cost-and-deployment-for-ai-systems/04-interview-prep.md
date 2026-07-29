# Scaling, Cost & Deployment for AI Systems — Interview Prep

**Section:** Production AI Systems — Security, Eval & Scale | **Role target:** Applied AI Engineer / AI Systems Engineer (agentic AI, RAG, multi-agent production systems)

---

## Core Conceptual Questions

These test whether you understand the fundamentals.

| Question | Key points to cover | Common weak-answer trap |
|---|---|---|
| Why can't you scale an LLM/agent service the way you scale a stateless CRUD API, and what makes the workload different? | LLM calls are **high-latency, I/O-bound** (seconds, not milliseconds), highly variable in duration (token count varies per request), and gated by an **external provider's rate limits (RPM and TPM)** you don't control. So the bottleneck is rarely local CPU — it's upstream throughput and long-lived in-flight requests. This favors **async concurrency** (many awaiting requests per worker), **queue-based decoupling** for long jobs, and **autoscaling on queue depth / in-flight requests** rather than CPU. | "Just add more replicas / more CPU." Adding replicas behind a shared provider quota doesn't raise throughput — it just hits the 429 wall faster and in parallel. Ignores that the constraint is external tokens-per-minute, not local compute. |
| How do provider rate limits work and how should a production client handle hitting them? | Providers meter **RPM (requests/min)** and **TPM (tokens/min)**; exceeding either returns HTTP **429**. Correct handling: **exponential backoff with jitter** on 429/503 (jitter prevents synchronized retry storms across workers), respect `Retry-After` when present, cap retries, and have a **fallback** (secondary model/provider or degraded path). Combine with a client-side concurrency limiter / token budget so you throttle *before* hitting the wall, and a queue to absorb bursts. | "Just retry immediately on 429" — immediate, un-jittered retries create a thundering-herd retry storm that makes the overload worse and can get you further throttled. No mention of jitter, backoff caps, or a fallback. |
| Why is a queue-based async architecture the standard pattern for agentic workloads, and what are its moving parts? | Long agent runs (multi-step, seconds-to-minutes) shouldn't block an HTTP request thread. Pattern: API **enqueues** a job and returns a job ID immediately; a pool of **workers** pulls from the queue, runs the agent loop, and writes results to external state; the client polls or gets a webhook/stream. Benefits: smooths bursty load, decouples ingestion rate from processing rate, lets you **autoscale workers on queue depth**, and isolates failures. Requires **stateless orchestration** so any worker can run any job (session state lives in Redis/Postgres, not in-process). | Describing a synchronous request/response for a 60-second agent job — ties up connections, times out at the load balancer, and can't absorb bursts. Or treating the queue as "just a buffer" without stateless workers + external state, so jobs can't be picked up by any worker. |
| Why does CI/CD for an AI service differ from ordinary software, and what belongs in the pipeline? | Behavior depends on **non-code artifacts** — prompts, the model/version, tool definitions, and the retrieval index — so a "code-only" green build says nothing about quality. You need an **eval gate in CI**: run a curated eval set on the candidate build and **block the deploy if quality/accuracy/safety metrics regress** past a threshold. You must **version and pin** prompt, model ID/version, tool schema, and index snapshot together so a build is reproducible. | "Prompts aren't code, they don't need CI / versioning." A prompt change or a silent provider model update can regress behavior with zero code diff and no test signal. Treating unit tests / a green build as sufficient for a probabilistic system. |
| What is progressive delivery for an AI service and how do you decide to roll back? | Instead of big-bang 100% cutover, expose the new prompt/model version gradually: **shadow** (mirror traffic, compare offline, no user impact), **canary** (small % of live traffic), **A/B** (measure metric deltas), **blue-green** (instant switch/rollback). Define **automatic rollback triggers** on quality/eval, error-rate, latency, and cost regressions. Decouple prompt/model from code (config/prompt registry) so you can roll back a prompt without redeploying the app. | "Ship straight to 100%." No canary/shadow, no rollback trigger, no way to compare new vs old behavior on real traffic — and rolling back requires a full code redeploy because the prompt is hardcoded. |
| Break down the unit economics of an LLM/agent call. What actually drives cost and what levers reduce it? | **Cost ≈ tokens × price-per-token × number of calls**, with input and output tokens often priced differently. In an agent, cost is **multiplied by the loop** (each iteration re-sends growing context) and by **multi-agent fan-out** (each sub-agent is its own call chain) — the *token multiplier*. Levers: **model right-sizing / routing / cascades** (cheap model first, escalate only when needed), **token reduction** (trim context, summarize, cap history), **prompt + semantic caching**, **batching**, and **bounding loop iterations / fan-out**. Govern with **budgets, quotas, alerts, and cost-as-a-first-class metric**. | "The model call is basically free" / "just use the biggest model for everything." Ignores that context re-sent per loop iteration and per sub-agent makes agent cost superlinear, and that a frontier model on every call is often 10–30× the cost of a routed cheaper model at similar quality for easy queries. |

---

## Applied / Scenario Questions

### Q1 — Your agent platform is throwing intermittent 429s under peak load; some user requests fail outright and support is complaining. Fix it without just "paying for a higher tier."

**Strong answer framework:**
- **Diagnose which limit you're hitting:** RPM vs TPM. A chatty agent with short prompts hits RPM; a RAG agent stuffing large context hits TPM first. Instrument request and token rates against the quota so you know the actual ceiling.
- **Throttle before the wall, not after:** add a **client-side concurrency limiter / token-budget rate limiter** so you stay under quota proactively, and put a **queue** in front so bursts are absorbed and drained at a safe rate rather than fired at the provider all at once.
- **Handle the 429s you do get correctly:** **exponential backoff with jitter** (jitter is essential — without it every throttled worker retries in lockstep and re-triggers the overload), respect `Retry-After`, cap total retries, then fall back to a **secondary model/provider or a degraded path** rather than hard-failing the user.
- **Reduce demand at the source:** **semantic + prompt caching** cuts repeat calls entirely; **token reduction** (trim history/context) lowers TPM pressure; **model routing** sends easy queries to a cheaper/faster model, freeing headroom on the constrained one.
- **Trade-off framing:** a queue adds tail latency (jobs wait) but protects **throughput and availability** under burst; backoff adds latency on the retried request but prevents a retry storm from taking the whole fleet down. The explicit trade is *worst-case latency up, success rate and stability up* — usually the right call for correctness over raw speed. Autoscale workers on **queue depth** to claw back latency when sustained load (not just a spike) is the cause.

### Q2 — Finance flags that your multi-agent research feature costs far more per session than projected. Cut cost-per-conversation without visibly degrading quality.

**Strong answer framework:**
- **Build the cost model first:** attribute `tokens × price × calls` per conversation and find where it concentrates — usually the **agent-loop token multiplier** (context re-sent every iteration) and **sub-agent fan-out** (N parallel sub-agents, each a full call chain). You can't optimize what you haven't measured; make **cost a first-class metric** per conversation, not a monthly aggregate.
- **Attack the multiplier:** **bound loop iterations and fan-out** (cap sub-agents; not every question needs 5 researchers), **trim/summarize context** so each iteration re-sends less, and **cache** (semantic cache for repeated sub-queries, prompt caching for the large stable system/context prefix).
- **Right-size the model per step:** use a **cascade / router** — cheap model for retrieval-planning, extraction, and easy sub-tasks; escalate to the frontier model only for the final synthesis or hard steps. Most tokens in a multi-agent flow are intermediate and don't need top-tier quality.
- **Batch** independent sub-tasks where the workload and provider support it to cut per-call overhead.
- **Trade-off framing:** this is the **cost / quality / latency triangle**. Aggressive context trimming or a smaller model can shave quality — so gate every change behind the **eval set**: only ship a cost cut if quality holds within threshold. Bounding fan-out trades a little breadth on rare hard queries for large, predictable savings on the common case. Add **budgets/quotas/alerts** so a runaway loop (**Unbounded Consumption**) can't produce a surprise bill — it's a cost *and* availability risk.

---

## System Design / Architecture Questions

### Q — Design a cost-efficient, horizontally scalable agent platform that can deploy prompt and model changes safely.

**Approach:**

1. **Clarify requirements.**
   - **Load & latency:** expected QPS / concurrent sessions, peak-to-average burst ratio, and the latency budget — interactive chat (target seconds) vs. long research jobs (async, minutes acceptable)? These decide sync vs. queue.
   - **Cost target:** is there a **cost-per-conversation** or monthly budget ceiling? That sets how aggressive routing/caching must be.
   - **Quality & safety:** accuracy/hallucination tolerance, whether there's an eval set, and any irreversible actions needing guardrails.
   - **Provider constraints:** which providers/models, and their RPM/TPM quotas — the real throughput ceiling.

2. **Architecture.**
   - **Ingestion:** stateless **FastAPI** front end enqueues jobs (returns a job ID/stream) for long agent runs; short interactive calls can go synchronous with strict timeouts.
   - **Queue + worker pool:** workers pull from the queue and run the agent loop. **Stateless orchestration** — session/graph state lives in **Postgres/Redis** (a LangGraph-style checkpointer), so any worker runs any job. **Autoscale workers on queue depth**, not CPU.
   - **Provider access layer:** a shared client that enforces a **client-side rate/token limiter** (stay under RPM/TPM), does **exponential backoff + jitter** on 429/503, and has **model fallback**. Front it with **prompt caching + a semantic cache** to kill repeat calls.
   - **Cost controls:** a **model router / cascade** (cheap model first, escalate on need), **bounded loop iterations and sub-agent fan-out**, and per-tenant **budgets/quotas/alerts** with **cost logged as a metric** per conversation.
   - **Deploy pipeline:** **version and pin** prompt + model ID + tool schema + index snapshot together in a **prompt/config registry** decoupled from code. CI runs an **eval gate** (block on quality/safety regression). Roll out via **shadow → canary → A/B**, with **automatic rollback triggers** on eval, error, latency, and cost deltas; **blue-green** for instant model swaps and prompt rollback without a code redeploy.
   - **Health & serving:** containerized stateless services with health checks; graceful handling of **model deprecations** (pin versions, test the successor in shadow before the provider sunsets the old one).

3. **Justify choices and name trade-offs.**
   - **Queue + async over synchronous:** trades added tail latency (queue wait) for burst absorption, independent scaling, and resilience — correct for long/bursty agent workloads; keep short interactive calls sync to protect their latency.
   - **Autoscale on queue depth over CPU:** CPU is near-idle on an I/O-bound LLM service, so CPU autoscaling never fires; queue depth is the true backpressure signal. Trade-off: needs a good queue-depth metric and scale-up lead time.
   - **Router/cascade + caching over one frontier model everywhere:** big cost win at some added routing complexity and a small risk of mis-routing a hard query to a weak model — mitigated by the eval gate and escalation.
   - **Eval-gated canary over big-bang:** trades slower rollout for the ability to catch a silent quality/cost regression on real traffic and roll back a prompt without redeploying. The core insight: for a probabilistic system, a green code build is not a quality signal — the **eval set is the gate**.
   - **Bounded loops/fan-out + budgets:** trades a little capability on rare hard cases for protection against **Unbounded Consumption** — a runaway agent is simultaneously a cost blow-up and an availability/DoS risk.

---

## Vocabulary That Signals Expertise

Use these terms naturally — don't force them:

- **RPM / TPM (requests- and tokens-per-minute limits)** — when explaining the *external* throughput ceiling that governs how an LLM service scales.
- **Exponential backoff with jitter** — when describing correct 429/503 handling and why jitter specifically prevents synchronized retry storms.
- **Model fallback / secondary provider** — when discussing graceful degradation instead of hard-failing on provider throttling or outage.
- **Queue-based async architecture / autoscaling on queue depth** — when decoupling ingestion from processing and scaling on the true backpressure signal (not CPU).
- **Stateless orchestration + external state (checkpointer)** — when explaining how any worker can pick up any job and why session state lives in Redis/Postgres.
- **Prompt caching / semantic caching** — when reducing repeat calls and cutting cost/latency on stable prefixes and near-duplicate queries.
- **Model routing / cascade / right-sizing** — when arguing cheap-model-first with escalation over a frontier model on every call.
- **Token multiplier (agent-loop + fan-out)** — when explaining why agent cost is superlinear as context re-sends per iteration and per sub-agent.
- **Cost-per-conversation / cost-as-a-metric** — when framing cost governance at the right unit of analysis, not a monthly aggregate.
- **Eval gate in CI** — when explaining why an AI deploy needs a quality/safety threshold check, not just green unit tests.
- **Version pinning (prompt + model + tools + index)** — when making a build reproducible and behavior explainable.
- **Progressive delivery: shadow / canary / A-B / blue-green + rollback triggers** — when describing safe, reversible rollout of prompt/model changes.
- **Prompt / config registry (decoupled from code)** — when explaining how you roll back a prompt without redeploying the app.
- **Unbounded Consumption** — when naming uncapped loops/fan-out as a combined cost and availability risk.
- **Cost / quality / latency triangle** — when framing any optimization as a three-way trade-off rather than a free lunch.

---

## Vocabulary That Signals Weakness

Avoid these — they signal outdated or shallow understanding:

- **"Just add more replicas / more CPU."** — Red flag: throughput is capped by the provider's RPM/TPM, not local compute; more replicas behind one quota just hit the 429 wall in parallel.
- **"Just use the biggest model for everything."** — Red flag: ignores routing/cascades and the token multiplier; a frontier model on trivial sub-steps is often an order of magnitude more expensive for no quality gain.
- **"Retry immediately on 429."** — Red flag: un-jittered immediate retries create a thundering-herd retry storm that worsens the overload; no backoff, jitter, `Retry-After`, or fallback.
- **"Prompts aren't code, so they don't need versioning / CI."** — Red flag: a prompt change (or a silent provider model update) can regress behavior with no code diff; shows no eval gate or version-pinning discipline.
- **"Ship it straight to 100%."** — Red flag: no shadow/canary, no rollback trigger, no way to compare new vs old on real traffic; and rollback needs a full redeploy because the prompt is hardcoded.
- **"The model call is basically free."** — Red flag: no cost model; ignores that context re-sent per loop iteration and per sub-agent makes agent cost superlinear.
- **"Just run the agent synchronously in the request."** — Red flag: a minute-long job blocks connections, times out at the load balancer, and can't absorb bursts; no queue/worker decoupling.
- **"We'll add cost monitoring later."** — Red flag: without budgets/quotas/alerts an uncapped loop is a live Unbounded-Consumption risk — a surprise bill *and* a DoS vector.

---

## STAR Answer Frame

**Situation:** A LangGraph multi-agent RAG feature on our FastAPI/PostgreSQL stack started throwing intermittent 429s at peak and, separately, finance flagged that its cost-per-conversation was several times the projection. Some user requests were hard-failing and long research runs were timing out at the load balancer because they ran synchronously in the request thread.

**Task:** Eliminate the 429 failures, make long agent runs reliable under burst, and cut cost-per-conversation — all without a measurable drop in answer quality.

**Action:** I (1) moved long agent runs onto a **queue with a stateless worker pool**, persisting graph state in Postgres so any worker could resume any job, and set the workers to **autoscale on queue depth**; (2) put a shared provider client in front with a **client-side token/RPM limiter**, **exponential backoff with jitter**, and a **secondary-model fallback** so we throttled *before* the wall and degraded gracefully after it; (3) added **prompt + semantic caching** and a **model cascade** (cheap model for retrieval-planning and extraction, frontier model only for final synthesis) and **capped loop iterations and sub-agent fan-out** to kill the token multiplier; (4) made **cost-per-conversation a logged metric** with per-tenant budgets and alerts; and (5) gated every prompt/model change behind an **eval set in CI** and rolled changes out via **canary with automatic rollback** on quality/cost regression, with prompts pinned in a registry decoupled from code.

**Result:** Peak 429-driven failures went to effectively zero and long-run timeouts disappeared; **cost-per-conversation dropped by roughly 55%** driven mainly by the cascade, caching, and fan-out cap; and eval-set quality held flat within noise across the rollout — better reliability and much lower cost with no measurable quality loss.

---

## Red Flags Interviewers Watch For

Specific to scaling, cost, and deployment of AI systems:

- **No backoff/queue for rate limits** — treating 429s as a rare error to retry immediately, with no jittered backoff, no proactive client-side limiter, and no queue to absorb bursts — the clearest sign of no production-scale LLM experience.
- **Synchronous long-running agent jobs** — running multi-step agent loops inside the HTTP request thread, ignoring load-balancer timeouts, connection exhaustion, and inability to absorb bursts or autoscale.
- **Scaling on CPU / "just add replicas"** — no grasp that throughput is provider-quota-bound and that queue depth (not CPU) is the real backpressure signal.
- **Unpinned prompts/models** — hardcoded prompts, no model-version pinning, no config/prompt registry — so behavior can silently change with a provider update and can't be rolled back without a redeploy.
- **Big-bang prompt/model deploys with no eval gate** — shipping straight to 100% with only unit tests green, no shadow/canary, and no automatic rollback trigger on quality/cost regression.
- **No cost model** — can't explain `tokens × price × calls`, the agent-loop/fan-out token multiplier, or cost-per-conversation; treats model calls as free.
- **Uncapped agent loops / fan-out** — no iteration cap or fan-out bound and no budgets/quotas/alerts, leaving an Unbounded-Consumption risk that's simultaneously a cost blow-up and an availability/DoS vector.
- **No caching or routing** — sending every call to a frontier model with no prompt/semantic cache and no cascade, then being surprised by the bill.
