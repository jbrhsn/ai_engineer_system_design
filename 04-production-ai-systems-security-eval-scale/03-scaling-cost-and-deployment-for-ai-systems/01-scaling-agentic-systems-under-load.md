# Scaling Agentic Systems Under Load

**Section:** 04 Production AI Systems (Security, Eval, Scale) → Scaling, Cost & Deployment for AI Systems | **Est. time:** 3 hrs | **Interview relevance:** High — "how would you scale this to 10× traffic?" is the follow-up on every agentic system-design question, and the differentiating answer names the model provider's rate limit — not your CPU — as the real bottleneck.

---

## TL;DR

Scaling an agentic/LLM system is not like scaling a classic web service: requests are long, variable-latency, expensive, often stream tokens, and a single agent run holds state across many model+tool calls. The work is dominated by *waiting* on the model provider and tools, so the levers that matter are **concurrency/async** (do other work while waiting), **respecting provider rate limits** (RPM/TPM/ITPM with exponential backoff + jitter, retry budgets, and fallback across models/regions), **cutting repeat work** (semantic caching + provider prompt caching), and **moving long jobs off the request path** into a **queue + worker** architecture whose workers autoscale on queue depth. The orchestration layer stays **stateless and scales horizontally** because run state lives in an external checkpointer/store (see section 03 on durable execution). **The one thing to remember: the bottleneck is almost always the model provider's rate limit and per-request latency, not your code — so you scale by queuing, caching, backing off, and falling back, not by blindly adding replicas.**

---

## ELI5 — Explain It Like I'm 5

Imagine a call center where the receptionists can pick up any phone instantly, but every real question has to be forwarded to a single brilliant consultant in a back room — and that consultant will only take a fixed number of calls per minute and hangs up on you (a "429") if you dial too fast. Hiring twenty more receptionists does nothing, because they all still wait in line for the same consultant; you just get a bigger pile of people waiting. So instead you write callers' questions on tickets and drop them in a queue, you let each receptionist handle other tickets while waiting on hold, you keep a folder of answers to questions people ask over and over so you never bother the consultant twice, and when the consultant says "too fast," you wait a polite, growing amount of time before trying again instead of redialing instantly and making the jam worse. The common misconception is "we're slow, so add more servers" — but your servers are mostly *idle, waiting*; the real fix is managing the line to the consultant, not adding more people to stand in it.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain how LLM/agent workloads differ from classic web requests (long, variable, expensive, stateful, streaming) and why that changes the scaling playbook.
- [ ] Diagnose why agent throughput is bounded by I/O waiting on the model provider, and apply async/concurrency instead of vertical CPU scaling.
- [ ] Design rate-limit handling — RPM/TPM/ITPM awareness, exponential backoff **with jitter**, a bounded retry budget, request queuing, and provider/model fallback.
- [ ] Cut repeat work with semantic caching and provider prompt caching, and reason about their effect on latency, cost, and rate-limit headroom.
- [ ] Architect long-running agent jobs as an async task queue with workers, status polling/webhooks, and worker autoscaling on queue depth, with a stateless orchestration layer backed by an external checkpointer/store.

---

## Visual Overview

### Synchronous Request/Response vs. Queue + Workers (long agent jobs)

```
SYNCHRONOUS (breaks for long agent runs)
client ──► HTTP handler ──► [ agent runs 45s: model+tool calls ] ──► response
              │
              └─ holds one connection/thread the whole time; gateway times out
                 at ~30–60s; a traffic spike ties up every worker at once

QUEUE + WORKERS (scales for long agent runs)
client ──► API: enqueue job ──► returns job_id (fast, ~ms)
                    │
                    ▼
              ┌───────────┐     pull      ┌──────────┐   model/tool calls
              │  QUEUE    │──────────────►│ WORKER N │──────────────►provider
              │ (depth D) │               └──────────┘
              └───────────┘                    │ writes state + result
   client polls GET /jobs/{id}  ◄──────────────┘  (or receives a webhook)
        status: queued → running → done/failed
```

### Rate-Limit Backoff + Fallback Flow

```
send request ──► 200 OK ──► done
      │
      └─ 429 / 529 (rate limited)? read Retry-After header if present
             │
             ▼
        retries left in budget?
        ├── No  ──► fall back: another model / region / provider key
        │            └── still failing? ──► load-shed (reject/degrade)
        └── Yes ──► wait = min(cap, base * 2^attempt) + random_jitter
                     └──► retry  (NEVER retry instantly in a tight loop)
```

### Stateless Orchestration Layer + External State

```
        load balancer
        ┌────┬────┬────┐
        ▼    ▼    ▼    ▼
     ┌────┐┌────┐┌────┐┌────┐   orchestration replicas
     │ o1 ││ o2 ││ o3 ││ oN │   STATELESS (hold no run state)
     └────┘└────┘└────┘└────┘   scale horizontally = add replicas
        │    │    │    │
        └────┴──┬─┴────┘
                ▼
     ┌───────────────────────┐   run state lives HERE, not in the replica:
     │ checkpointer + store  │   any replica can resume any thread_id
     │ (e.g. Postgres)       │   (section 03: durable execution)
     └───────────────────────┘
```

### Autoscaling Workers on Queue Depth

```
queue depth (backlog of jobs)
   ▲
   │            ┌───────── scale UP: depth > target ──► add workers
   │           /
target ───────/──────────── steady state (depth ≈ target)
   │         /
   │ ───────┘  scale DOWN: depth < target for cooldown ──► remove workers
   └───────────────────────────────────────────────► time
scale on QUEUE DEPTH / oldest-message-age, not CPU% (workers look idle while waiting on the model)
```

---

## Key Concepts

### LLM/Agent Workload Characteristics (why classic web-scaling intuitions break)

**What it is.** An agent request is a *long-lived, variable-duration, expensive, stateful* unit of work — often a multi-step loop of model calls and tool calls that can run for seconds to minutes and streams tokens back incrementally — as opposed to a classic web request that is short, cheap, uniform, and stateless.

**How it works mechanistically.** A single agent run may issue many sequential model calls (each ReAct step: model → tool → observation → model), and each model call's latency scales with the number of *output* tokens generated (tokens are produced one at a time), so duration is inherently variable and hard to predict per request. Because the run carries a growing message list as state and may pause for human input or tool round-trips, you cannot treat two "identical" requests as interchangeable the way you would two static-page fetches. This means classic capacity math (requests × fixed latency) fails, tail latency is enormous, and holding the run in a request handler ties up a connection for the entire duration.

**Where it appears in real systems.** OpenAI's latency guidance notes the bulk of latency comes from the *token generation* step (tokens generated one at a time) and recommends `stream: true` to improve time-to-first-token; this streaming + long-duration profile is exactly why agent endpoints use SSE/websockets for partial output and why long agent jobs get pushed off the synchronous request path (below). The stateful, resumable nature is why LangGraph runs are keyed by a `thread_id` whose state is snapshotted by a checkpointer.

### Concurrency & Async (the work is waiting, not computing)

**What it is.** Concurrency is running many agent requests *in flight* at once on a small number of threads/processes by overlapping their idle time; async I/O is the mechanism that lets a single worker start one model call and, while awaiting the response, service others.

**How it works mechanistically.** For an agent, the CPU is idle almost the entire request — it is blocked on network I/O waiting for the model provider and tools to respond. A synchronous blocking model wastes a whole thread per in-flight request during that wait; an async event loop (`await client.responses.create(...)`) parks the coroutine when it hits the network wait and runs other coroutines, so one process can hold hundreds of concurrent model calls with minimal CPU. This is why the right scaling axis is *concurrency* (how many simultaneous waits you can hold) rather than vertical CPU/GPU — you have no local model to compute; you are a client of a remote one.

**Where it appears in real systems.** FastAPI `async def` endpoints on an ASGI server (Uvicorn) plus the provider SDK's async client (`AsyncOpenAI`, `AsyncAnthropic`) let one process fan out concurrent calls; `asyncio.gather` with an `asyncio.Semaphore` bounds how many run at once. LangGraph exposes `ainvoke`/`astream` for the async path. The practical ceiling on useful concurrency is set not by your event loop but by the provider's rate limit (next).

### Provider Rate Limits (RPM / TPM / ITPM) — the hard ceiling

**What it is.** Providers cap how much you can call the API per unit time, measured on multiple axes simultaneously: requests per minute (RPM), tokens per minute (TPM) — Anthropic splits this into input tokens per minute (ITPM) and output tokens per minute (OTPM) — plus per-day variants; you hit the limit on whichever axis fills first.

**How it works mechanistically.** Limits are enforced at the organization (and project/workspace) level, per model, using a replenishing scheme (Anthropic documents a token-bucket algorithm, so capacity refills continuously rather than resetting on a fixed clock — which is why short bursts can trip a limit even under the per-minute average). Exceeding a limit returns HTTP **429** (Anthropic also returns 529 for overload and includes a `Retry-After` header telling you how long to wait; OpenAI returns rate-limit metadata in `x-ratelimit-*` response headers). A subtle trap: *failed requests still count against your per-minute limit*, so hammering retries makes the limit harder to clear, and a sudden traffic spike can trip separate "acceleration" limits — providers advise ramping gradually.

**Where it appears in real systems.** You read `x-ratelimit-remaining-requests` / `x-ratelimit-remaining-tokens` (OpenAI) or `anthropic-ratelimit-*` / `retry-after` (Anthropic) headers to know your headroom, and you gate outbound calls with a client-side limiter sized below the account's RPM/TPM. Tiers raise these limits as spend grows, but at any moment the limit is a fixed ceiling you must design under — no amount of your own replicas raises it.

### Backoff + Jitter, Retry Budgets, and Fallback

**What it is.** The resilience layer for the rate-limit ceiling: retry transient failures with exponentially increasing, *randomized* waits; cap total retries with a budget; and when retries are exhausted, fail over to an alternate model/region/provider.

**How it works mechanistically.** On a 429/5xx you wait `base * 2^attempt` seconds and add **random jitter** so that many clients that failed at the same instant don't all retry in lockstep — jitter breaks the synchronized "thundering herd" that would re-trip the limit. A **retry budget** (max attempts, or a max total time) guarantees termination so a persistently-limited request can't retry forever and inflate cost/latency. **Fallback/load-balancing** routes overflow to a second model, a second region, or a second API key/provider so a single provider's ceiling isn't the whole system's ceiling; when even fallback is saturated you **load-shed** (reject or degrade low-priority traffic) to protect the core.

**Where it appears in real systems.** OpenAI's docs explicitly recommend "retrying with exponential backoff" and adding "random jitter … to [keep] retries from all hitting at the same time," and show a manual decorator plus Tenacity/`backoff` examples. In practice you honor Anthropic's `retry-after` header when present, wrap calls with `tenacity.retry(wait=wait_random_exponential(...), stop=stop_after_attempt(n))`, and put a router (e.g. a proxy or your own dispatcher) in front to spill to a fallback model on repeated 429s.

### Caching — Semantic Cache + Provider Prompt Caching

**What it is.** Two distinct techniques to avoid paying (in latency, tokens, and rate-limit budget) for work you've effectively done before: a **semantic cache** short-circuits *whole responses* for equivalent queries; **provider prompt caching** reuses the compute for a repeated prompt *prefix* server-side.

**How it works mechanistically.** A semantic cache embeds the incoming query and, on a near-match (cosine similarity above a threshold) to a prior query, returns the stored answer without any model call — eliminating the request entirely (great for FAQ-like traffic, risky when a wrong hit returns a stale/incorrect answer). Provider prompt caching instead keeps your call but makes the repeated *prefix* (system prompt, tool definitions, long context) cheaper and faster: the provider matches an exact prefix and skips re-processing it. Critically, prompt-cache reads are billed at a large discount and — on most Anthropic models — **cached input tokens do not count toward your ITPM rate limit**, so caching directly buys back rate-limit headroom (OpenAI caching, by contrast, still counts toward TPM but cuts latency/cost).

**Where it appears in real systems.** Anthropic's `cache_control: {"type": "ephemeral"}` (automatic or explicit breakpoints, 5-min default / 1-hr TTL) caches the `tools → system → messages` prefix; you verify hits via `cache_read_input_tokens` in the response `usage`. OpenAI prompt caching is automatic for prompts ≥1024 tokens (reported in `cached_tokens`), improved by setting a stable `prompt_cache_key`. The universal rule for both: put *static* content (instructions, tools, context) first and *variable* content (the user's message) last, or the prefix hash changes every request and you never get a hit.

### Queue-Based Async Architecture + Workers

**What it is.** A pattern that moves long agent runs off the synchronous HTTP path: the API endpoint *enqueues* a job and returns a `job_id` immediately; separate worker processes pull jobs, run the agent, and persist the result, which the client retrieves by polling a status endpoint or via a webhook.

**How it works mechanistically.** Decoupling accept-from-execute means a traffic spike grows the *queue* (a cheap backlog) instead of exhausting web workers or blowing past gateway timeouts; the queue also acts as a natural rate buffer, letting a fixed pool of workers drain work at whatever pace the provider's rate limit allows. Workers are idempotent and pull one job at a time (bounded concurrency), update job state (`queued → running → done/failed`) in a store, and can retry a failed job from a checkpoint rather than from scratch. Providers offer an analogous *batch* path (OpenAI Batch API, Anthropic Message Batches API) for non-interactive bulk work that runs against separate, higher queue limits and does not consume your synchronous rate budget.

**Where it appears in real systems.** Celery/RQ/Arq workers backed by Redis or SQS, or a managed runner; the client hits `POST /jobs` → `202 Accepted {job_id}` then `GET /jobs/{id}`. LangGraph's Agent/Platform server implements exactly this — long runs are backgrounded, state persists, and you poll run status — which is why you rarely run a multi-minute agent inside a plain request handler.

### Autoscaling Workers on Queue Depth

**What it is.** Automatically adding/removing worker instances based on the *backlog* — queue depth or oldest-message age — rather than on CPU utilization.

**How it works mechanistically.** Because agent workers spend their time *waiting on the model*, CPU stays near-idle even when the system is badly overloaded, so a CPU-based autoscaler never fires and the backlog grows unbounded. Queue depth (or age of the oldest unprocessed job) is the true demand signal: you set a target (e.g. "≤ N pending jobs per worker") and the autoscaler scales the worker count to hold that target, with a cooldown to avoid flapping. Note this scales *your workers*, which lets you submit work faster — but you still hit the provider ceiling, so worker autoscaling must be paired with rate-limit handling or you just spread more 429s across more workers.

**Where it appears in real systems.** Kubernetes HPA with a custom/external metric (queue length from Redis/SQS via KEDA), or a cloud autoscaler on SQS `ApproximateNumberOfMessagesVisible` / `ApproximateAgeOfOldestMessage`. The autoscale *target queue depth* and cooldown are the tuning knobs.

### Stateless Scaling with External State

**What it is.** Keeping the orchestration layer (the process running the agent loop) stateless so any replica can serve any request, while the run's state (message history, checkpoints) lives in an external, shared store.

**How it works mechanistically.** If a replica held run state in memory, requests for a given conversation would be pinned to that replica (sticky sessions) and a crash would lose the run — neither scales nor survives failures. Instead the agent framework snapshots state to an external **checkpointer** (thread-scoped, e.g. Postgres) and a **store** (cross-thread memory); a request carries a `thread_id`, any stateless replica loads that thread's latest checkpoint, advances it, and writes it back. This gives horizontal scaling (add replicas behind a load balancer) *and* durable execution (resume after a crash) from the same mechanism — the deep dive is section 03.

**Where it appears in real systems.** LangGraph compiles a graph with `checkpointer=PostgresSaver(...)` and `store=...`; the `InMemorySaver` is dev-only because it's lost on restart and not shared across replicas. The LangGraph Agent Server handles this persistence for you so replicas are interchangeable — the same reason OpenAI's production-best-practices guide lists horizontal scaling + load balancing as the default architecture.

### Bounding Sub-Agent Fan-Out

**What it is.** Capping how many sub-agents (or parallel tool/model calls) a single run spawns concurrently when it fans out work (e.g. an orchestrator dispatching parallel research sub-agents).

**How it works mechanistically.** Fan-out multiplies *per-run* provider load: one user request that spawns 20 parallel sub-agents issues 20× the model calls in a burst, which can single-handedly trip your RPM/TPM limit and starve every other user. Bounding fan-out with a concurrency cap (a semaphore over sub-agent launches) trades a little wall-clock latency for predictable, fair load and protects your rate-limit budget; it also caps worst-case cost per request. This is the multi-agent analogue of the single-agent recursion limit from section 03's agent fundamentals.

**Where it appears in real systems.** An `asyncio.Semaphore(max_parallel_subagents)` wrapping the fan-out, or a framework concurrency limit on parallel branches; Anthropic's own multi-agent research write-up highlights that sub-agent parallelism drives token usage sharply, making a fan-out cap a first-class scaling and cost control.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Max client-side concurrency per provider key (semaphore size) | How many simultaneous in-flight model calls one key issues | Set *below* your account's effective RPM/TPM headroom (leave margin for retries + bursts); raise only after you've raised the provider limit or added another key/region. |
| Retry budget (`stop_after_attempt` / max total time) | Upper bound on retries per request | Keep small (e.g. 3–6 attempts) so a persistently-limited request fails fast instead of inflating latency/cost; a bigger budget just prolongs a doomed request. |
| Backoff base + cap + jitter | The wait schedule between retries | Use exponential (`base·2^attempt`) with a cap and **always add random jitter**; honor `Retry-After` when the provider sends it — fixed or no jitter recreates the thundering-herd retry storm. |
| Semantic cache similarity threshold + TTL | When a query counts as a "hit" and how long answers live | Set the threshold high (conservative) for factual/transactional answers to avoid wrong hits; shorten TTL for volatile data. Disable for anything personalized or time-sensitive. |
| Prompt-cache breakpoint placement | Which prefix the provider caches | Put the breakpoint at the end of the *static* prefix (tools/system/context), before any per-request/variable content — a breakpoint after a timestamp or the user message never hits. |
| Worker autoscale target (queue depth / oldest-age) + cooldown | When to add/remove workers | Scale on backlog, not CPU; set target = pending jobs a worker clears within your latency SLA, with a cooldown long enough to avoid flapping. |
| Sub-agent fan-out cap | Max concurrent sub-agents/parallel calls per run | Set to the largest fan-out your rate budget tolerates when several requests run at once — one runaway fan-out can consume the whole org limit. |
| Request/run timeout | Max wall-clock before a run is abandoned | Set a hard timeout on both the model call and the whole run so a stuck provider call can't pin a worker forever; on timeout, fail the job and let it retry from a checkpoint. |

### Worked Example: Requirement → Decision

**Given:** You run a customer-facing "research assistant" agent. Each request triggers a multi-step LangGraph run (retrieval + several model calls, 20–90 s, some runs fan out to 3–8 parallel sub-agents). It currently runs synchronously inside a FastAPI handler. At peak, users see gateway timeouts and the provider starts returning 429s; you're on a single OpenAI/Anthropic key. Product wants to survive a 5× traffic spike next month, keep the median interaction responsive, and cap cost blow-ups. A stakeholder says "just add more API replicas."

- **Step 1 — Identify the goal:** Absorb a 5× spike without dropping runs or hitting gateway timeouts, keep p50 responsive, and bound worst-case cost — knowing the provider rate limit is the hard ceiling.
- **Step 2 — Define inputs:** User requests (long, variable, stateful multi-step runs with fan-out); one provider key with fixed RPM/TPM; runs already modeled as a LangGraph graph with `thread_id`.
- **Step 3 — Define outputs:** A `job_id` accepted immediately, a pollable/webhook status (`queued→running→done`), and a grounded final answer — never a timeout and never an unbounded retry storm.
- **Step 4 — Apply constraints:** Gateway ~30–60 s timeout (runs exceed it); provider RPM/TPM ceiling unchanged by adding your own replicas; cost must be bounded per request; runs must survive a worker crash.
- **Step 5 — Select the approach:** Move runs to a **queue + worker** architecture (enqueue → `job_id` → poll/webhook); make the orchestration layer **stateless** with a **Postgres checkpointer** so any worker resumes any `thread_id`; **autoscale workers on queue depth**; wrap every model call in **exponential backoff + jitter with a small retry budget**, honoring `Retry-After`; add **provider/model fallback** for overflow 429s; enable **prompt caching** on the static prefix and a **semantic cache** for repeat questions to buy rate-limit headroom; and **cap sub-agent fan-out** with a semaphore. *Rationale vs. "just add replicas":* more replicas don't raise the provider ceiling — they'd just fan more 429s out in parallel and still time out synchronously; the queue turns a spike into a drainable backlog, caching/fallback expand effective capacity, and backoff+budget keep cost bounded. Replicas help *only* the stateless orchestration layer, and only once state is externalized.

---

## Implementation

```python
# Scenario: Under a traffic spike our agent hammers the provider, gets 429s, and
# our naive "retry immediately" logic turns a soft rate-limit into a hard outage
# (a retry storm). We need every model call to back off with jitter, respect a
# retry budget, and honor the provider's Retry-After — while a semaphore keeps our
# own concurrency under the account's RPM/TPM ceiling.
# Backoff/jitter guidance verified against OpenAI rate-limits docs (platform.openai.com)
# and Anthropic rate-limits docs (retry-after header, docs.anthropic.com).
import asyncio
from tenacity import (
    retry, wait_random_exponential, stop_after_attempt, retry_if_exception_type,
)
from openai import AsyncOpenAI, RateLimitError, APIStatusError

client = AsyncOpenAI()
# Cap concurrent in-flight calls BELOW the account's RPM/TPM headroom.
# Adding more workers does not raise this ceiling — the semaphore protects it.
provider_gate = asyncio.Semaphore(20)

@retry(
    wait=wait_random_exponential(min=1, max=60),   # exponential base + random JITTER
    stop=stop_after_attempt(5),                    # retry BUDGET: fail fast, don't loop forever
    retry=retry_if_exception_type((RateLimitError, APIStatusError)),
    reraise=True,
)
async def call_model(messages):
    async with provider_gate:                      # bound our own concurrency
        return await client.responses.create(model="gpt-5.6", input=messages)

async def handle_many(batch):
    # Concurrency (not CPU) is the scaling axis: overlap the network waits.
    return await asyncio.gather(*(call_model(m) for m in batch))
```

```python
# Anti-pattern: run a long multi-step agent synchronously in the request handler
# AND retry 429s instantly in a tight loop. The run exceeds the gateway timeout,
# every replica is pinned for the full run duration, and the instant-retry loop
# is a thundering herd that keeps re-tripping the rate limit (failed requests
# still count against your per-minute limit — per OpenAI's docs).
from fastapi import FastAPI
app = FastAPI()

@app.post("/ask")                                   # BROKEN: blocks + retry storm
async def ask(body: dict):
    while True:                                     # no backoff, no jitter, no budget
        try:
            return await run_agent(body["query"])   # 20–90s: times out; ties up a worker
        except RateLimitError:
            continue                                 # instant retry = makes 429s worse

# Correct approach: accept fast, run in the background, expose status; the agent
# loop uses the backoff+budget wrapper above and persists to a checkpointer so
# any worker can resume the run. Client polls (or gets a webhook) for the result.
import uuid
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string("postgresql://...")  # shared, durable
JOBS = {}   # illustrative; real systems use Redis/SQS + a jobs table

@app.post("/jobs", status_code=202)
async def enqueue(body: dict):
    job_id = str(uuid.uuid4())
    JOBS[job_id] = {"status": "queued"}
    await queue.put({"job_id": job_id, "query": body["query"]})  # workers pull this
    return {"job_id": job_id}                                    # returns in ~ms

@app.get("/jobs/{job_id}")
async def status(job_id: str):
    return JOBS[job_id]      # queued -> running -> done/failed; result when done

async def worker():                                 # autoscaled on QUEUE DEPTH, not CPU
    graph = build_graph().compile(checkpointer=checkpointer)     # stateless replica
    while True:
        job = await queue.get()
        JOBS[job["job_id"]] = {"status": "running"}
        # thread_id ties the run to durable state; a crash resumes from the checkpoint.
        result = await graph.ainvoke(
            {"messages": [{"role": "user", "content": job["query"]}]},
            config={"configurable": {"thread_id": job["job_id"]}},
        )
        JOBS[job["job_id"]] = {"status": "done", "result": result}
# What breaks without this: synchronous long runs time out and can't scale past
# your worker count; instant retries amplify rate-limiting into an outage. The fix
# decouples accept-from-execute (queue absorbs spikes), bounds retries with backoff
# +jitter+budget, and externalizes state so replicas are interchangeable & crash-safe.
```

---

## Common Pitfalls & Misconceptions

- **"We're slow — add more replicas."** — Replicas feel like the universal web-scaling fix, so teams reach for them first. But agent replicas are mostly *idle waiting on the provider*; adding more just fans more requests at the same fixed RPM/TPM ceiling (more 429s, not more throughput). Scale by queuing, caching, backing off, and falling back; replicas help only the *stateless orchestration layer* after state is externalized.
- **Retrying 429s instantly (or with fixed delay, no jitter).** — "The request failed, so try again now" feels obviously right. But instant/synchronized retries are a thundering herd that re-trips the limit, and failed requests still count against your per-minute budget — so you dig deeper. Use exponential backoff **with random jitter**, honor `Retry-After`, and cap attempts with a retry budget.
- **Running long agent jobs inside the request handler.** — It's the simplest code path and works in the demo. In production, multi-step runs blow past gateway timeouts and a spike pins every worker at once. Long runs belong on an async **queue + worker** path with status polling/webhooks.
- **Autoscaling workers on CPU.** — CPU is the default autoscale metric everywhere, so it's the reflex. But agent workers idle on network I/O — CPU stays low while the backlog explodes, so the scaler never fires. Scale on **queue depth / oldest-message age**, the real demand signal.
- **Holding run state in the replica's memory.** — In-memory state is fast and easy, so it sneaks in. It pins conversations to one replica (sticky sessions) and loses everything on a crash — no horizontal scale, no durability. Keep orchestration **stateless** and put state in an external **checkpointer/store** (section 03).
- **Unbounded sub-agent fan-out.** — More parallel sub-agents *feels* faster. But one request fanning out to dozens of concurrent calls can single-handedly exhaust the org rate limit and cost budget. Cap fan-out with a semaphore; trade a little latency for fairness and bounded cost.

---

## Key Definitions

| Term | Definition |
|---|---|
| RPM / TPM / ITPM / OTPM | Provider rate-limit axes: requests-per-minute, tokens-per-minute (Anthropic splits into input-tokens- and output-tokens-per-minute); you hit the limit on whichever fills first. |
| 429 (rate limited) | HTTP status returned when you exceed a rate limit; often carries a `Retry-After` header telling you how long to wait before retrying. |
| Exponential backoff + jitter | Retrying after `base·2^attempt` seconds plus a random offset; the randomness prevents many clients from retrying in lockstep (a thundering herd). |
| Retry budget | A hard cap on retry attempts (or total retry time) per request, guaranteeing termination so a doomed request can't inflate latency/cost. |
| Provider fallback / load-balancing | Routing overflow traffic to an alternate model, region, or provider/key so one provider's ceiling isn't the whole system's ceiling. |
| Load-shedding | Deliberately rejecting or degrading lower-priority requests under overload to protect core traffic. |
| Semantic cache | A cache keyed by query *embedding similarity* that returns a stored answer for equivalent queries, skipping the model call entirely. |
| Prompt caching | Provider-side reuse of a repeated prompt *prefix* (tools/system/context) to cut latency and cost; on most Anthropic models cache reads don't count toward ITPM. |
| Queue + worker architecture | Accepting a job and returning a `job_id` immediately, with separate workers pulling and executing it; client polls status or receives a webhook. |
| Batch API | A provider path for non-interactive bulk requests that runs against separate queue limits and doesn't consume your synchronous rate budget. |
| Checkpointer / store | LangGraph external state: a checkpointer snapshots thread-scoped run state; a store holds cross-thread memory — enabling stateless, crash-safe scaling. |
| Autoscaling on queue depth | Adding/removing workers based on backlog (pending jobs / oldest-message age) rather than CPU, because agent workers idle on I/O. |

---

## Summary / Quick Recall

- Agent workloads are long, variable, expensive, streaming, and stateful — classic "requests × fixed latency" capacity math and CPU-based scaling both break.
- The work is **waiting on the provider**, so scale with **concurrency/async**, not vertical CPU; the true ceiling is the provider's **RPM/TPM/ITPM**, which your own replicas cannot raise.
- Handle 429s with **exponential backoff + jitter**, a **retry budget**, `Retry-After`, and **fallback** across models/regions/keys; never retry instantly (thundering herd, and failed calls still count).
- **Cut repeat work:** semantic caching skips whole calls; provider **prompt caching** reuses the static prefix — and on most Anthropic models cache reads don't consume ITPM, buying rate-limit headroom.
- Run long agent jobs on a **queue + worker** path (enqueue → `job_id` → poll/webhook), **autoscale workers on queue depth**, and keep the orchestration layer **stateless** with state in an external **checkpointer/store** (section 03).
- **Bound sub-agent fan-out** — one unbounded fan-out can exhaust the whole org rate limit and cost budget.

---

## Self-Check Questions

1. Name the axes on which model providers enforce rate limits, and state what HTTP status you receive when you exceed one and what header may tell you how long to wait.

   <details><summary>Answer</summary>

   Providers enforce limits on **requests per minute (RPM)** and **tokens per minute (TPM)** — Anthropic splits tokens into **input tokens/min (ITPM)** and **output tokens/min (OTPM)** — plus per-day variants; you hit the limit on whichever axis fills first. Exceeding one returns **HTTP 429**, and the response may include a **`Retry-After`** header (Anthropic) indicating how long to wait; OpenAI also exposes remaining headroom in `x-ratelimit-*` headers. The tempting wrong answer is "you get a 500 error" — a 500 is a server error, whereas rate limiting is specifically 429 (Anthropic uses 529 for overload), and treating it as a generic 500 leads you to miss the `Retry-After` guidance.

   </details>

2. Your synchronous FastAPI endpoint runs a 60-second multi-step agent inline, and under load users get gateway timeouts while replicas sit at ~5% CPU. What architectural change do you make and why?

   <details><summary>Answer</summary>

   Move the run **off the request path into a queue + worker architecture**: `POST /jobs` enqueues and returns a `job_id` immediately (202), workers pull and execute the run, and the client polls `GET /jobs/{id}` or receives a webhook. This is right because the 60 s run exceeds the gateway timeout and pins a worker for its whole duration; a queue absorbs spikes as a backlog and lets a fixed worker pool drain at the rate the provider allows. The low CPU is the tell that this is I/O-bound waiting on the provider — so the tempting answer, "add replicas / bigger instances," is wrong: more CPU does nothing for a workload that's idle-waiting, and more replicas still time out synchronously and just fan more calls at the same rate ceiling.

   </details>

3. **Which TWO** of the following correctly describe handling provider 429s at scale?
   - A. On a 429, retry immediately in a tight loop until it succeeds.
   - B. Use exponential backoff with random jitter and honor the `Retry-After` header when present.
   - C. Enforce a bounded retry budget so a persistently rate-limited request fails fast instead of retrying forever.
   - D. Failed (429'd) requests don't count toward your per-minute limit, so aggressive retries are free.
   - E. Adding more of your own replicas raises the provider's RPM/TPM ceiling.

   <details><summary>Answer</summary>

   **B and C.** B is correct because exponential backoff *with jitter* prevents synchronized retries (a thundering herd) from re-tripping the limit, and `Retry-After` tells you exactly how long to wait. C is correct because a retry budget guarantees termination so a doomed request can't inflate cost/latency indefinitely. A is the most tempting distractor and is wrong — instant tight-loop retries are precisely the retry storm that worsens rate-limiting. D is false: OpenAI's docs state failed requests still count against your per-minute limit, so aggressive retries dig you deeper. E is false: provider limits are set at the org/project level and are unaffected by your replica count.

   </details>

4. You add a semantic cache and enable Anthropic prompt caching. A teammate says both do "the same caching thing." Explain how they differ mechanistically and their different effects on rate-limit headroom.

   <details><summary>Answer</summary>

   They operate at different layers. A **semantic cache** sits *in front of* the provider: it embeds the query and, on a similarity match, returns a stored full answer with **no model call at all** — eliminating the request (and all its tokens) entirely, at the risk of a wrong hit returning a stale answer. **Prompt caching** still makes the call but reuses the repeated *prefix* (tools/system/context) server-side, cutting latency and cost of processing that prefix. On rate-limit headroom they differ sharply: a semantic hit frees an entire request's RPM+TPM, while prompt caching on most Anthropic models means **cached input tokens don't count toward ITPM** (so you get more effective throughput) — whereas OpenAI's prompt caching still counts toward TPM and only cuts latency/cost. The tempting wrong answer, "they're interchangeable," misses that one skips the call and the other cheapens it, and that their rate-limit effects are not the same.

   </details>

5. A stakeholder wants to guarantee your agent product survives a 10× spike by "autoscaling on CPU and adding API keys." Critique this and give the trade-off-aware design.

   <details><summary>Answer</summary>

   CPU autoscaling is the wrong signal: agent workers idle on network I/O waiting for the model, so CPU stays low while the backlog explodes and the scaler never fires — you must **autoscale on queue depth / oldest-message age**. Adding API keys/regions is a legitimate part of the answer (it's provider **fallback/load-balancing**, which raises effective capacity beyond one key's ceiling), but it's insufficient alone. The trade-off-aware design: put long runs behind a **queue + workers** (spike becomes a drainable backlog), keep orchestration **stateless** with an external **checkpointer** (any worker resumes any run, crash-safe), autoscale **workers on queue depth**, wrap calls in **backoff+jitter+retry budget**, spread load across **multiple keys/models/regions** with fallback, **cache** (semantic + prompt) to reclaim rate-limit headroom, **bound sub-agent fan-out**, and **load-shed** low-priority traffic when even fallback saturates. This balances latency (streaming/caching keep p50 responsive), throughput (queue + fallback), and cost (retry budget + fan-out cap bound worst case) — whereas "CPU autoscale + more keys" alone still times out synchronously and mis-scales.

   </details>

---

## Further Reading

- [OpenAI — Rate limits](https://platform.openai.com/docs/guides/rate-limits) — *verified 2026-07-29* — RPM/RPD/TPM/TPD axes, org/project-level enforcement, `x-ratelimit-*` headers, and the canonical exponential-backoff-with-jitter and batching mitigations.
- [OpenAI — Production best practices](https://platform.openai.com/docs/guides/production-best-practices) — *verified 2026-07-29* — Horizontal/vertical scaling, load balancing, caching, and cost-as-tokens×price framing for moving an LLM app to production.
- [OpenAI — Prompt caching](https://platform.openai.com/docs/guides/prompt-caching) — *verified 2026-07-29* — Automatic prefix caching (≥1024 tokens), `prompt_cache_key`, `cached_tokens`, and the static-prefix/variable-suffix structuring rule.
- [Anthropic — Rate limits](https://docs.anthropic.com/en/api/rate-limits) — *verified 2026-07-29* — RPM/ITPM/OTPM per-model limits, the token-bucket algorithm, 429 + `retry-after`, acceleration limits, and how cached tokens affect ITPM.
- [Anthropic — Prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) — *verified 2026-07-29* — `cache_control` breakpoints, tools→system→messages prefix, 5-min/1-hr TTL, `cache_read_input_tokens`, and cache reads not counting toward ITPM on most models.
- [LangGraph — Persistence (checkpointers & stores)](https://docs.langchain.com/oss/python/langgraph/persistence) — *verified 2026-07-29* — The external-state model that lets the orchestration layer stay stateless and scale horizontally (`PostgresSaver` for production; `InMemorySaver` is dev-only).
