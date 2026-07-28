# Scalability & Throughput Patterns for AI System Design

**Section:** 05 Interview Practice → Cross-Cutting Concerns & Trade-off Patterns | **Interview relevance:** High — "how does this scale / handle a traffic spike?" is the second-most-common follow-up after latency, and it forces you to distinguish compute-bound from queue-bound behaviour, reason about replicas, autoscaling, batching, backpressure, and rate limiting, and defend tail latency under load rather than just at rest.

---

## TL;DR

Throughput (req/s, tokens/s) and tail latency under load are governed by a different set of levers than single-request latency: you scale a model-serving tier horizontally with replicas + autoscaling, absorb bursts with a bounded queue that applies backpressure and load-sheds rather than falling over, and raise GPU efficiency with dynamic/continuous batching. Pipelines that must survive spikes are decoupled with async queues (SQS/Kafka) plus a dead-letter queue, and every stateful thing (session, retrieval index, cache) is pushed out of the request path into shared stores so any replica can serve any request. **The one thing to remember: first classify the bottleneck as compute-bound (add replicas/batch) or queue-bound (the work arrives faster than you drain it — add capacity, shed load, or decouple), because the two demand opposite fixes.**

---

## ELI5 — Explain It Like I'm 5

Imagine a busy restaurant kitchen. One cook (a model replica) can only plate so many dishes a minute; when a bus of tourists arrives at once, orders pile up on the rail and everyone waits longer — that pile-up is a queue, and the wait it adds is tail latency under load. You fix it by hiring more cooks who share the same fridge and recipe cards (stateless replicas reading shared state), by having a cook fry six identical burgers on one griddle instead of one at a time (batching), and by the host politely capping how many tables sit at once so the kitchen never collapses (rate limiting and load shedding). The common mistake is thinking a faster single cook solves a rush — but if orders arrive faster than any one cook can plate them, you need more cooks and a sane waiting policy, not a speedier knife.

---

## Learning Objectives

By the end of this note you will be able to:
- [ ] Classify a load problem as compute-bound vs queue-bound and choose the matching remedy (batching/replicas vs capacity/shedding/decoupling).
- [ ] Design a horizontally scaled model-serving tier with autoscaling, a bounded request queue, backpressure, and load shedding.
- [ ] Apply dynamic/continuous batching and per-tenant rate limiting to raise throughput without destroying tail latency or fairness.
- [ ] Decouple a multi-stage AI pipeline with an async queue + DLQ and keep the serving tier stateless by externalizing state to a vector DB and cache.

---

## Visual Overview

### Bounded queue with backpressure and load shedding

```
                 replicas (stateless, autoscaled)
                 ┌───────────┐
requests ──► [ bounded queue ] ──► │ replica 1 │ ──► shared state
   │              │  depth=N        ├───────────┤      (vector DB,
   │              │                 │ replica 2 │       Redis cache)
   │        queue FULL?             ├───────────┤
   └── yes ──► 429 / shed ◄─────────│ replica k │
              (load shedding)       └───────────┘
   backpressure ◄── queue depth rising ── autoscaler adds replicas
```

### Compute-bound vs queue-bound decision

```
Latency/throughput degrades under load?
├── Degrades even at LOW concurrency ──► COMPUTE-bound
│      └──► batch requests (dynamic/continuous) + add GPU replicas
└── Fine at low load, tail explodes as RPS climbs ──► QUEUE-bound
       ├── queue keeps growing ──► add replicas / shed load / rate-limit
       └── one slow stage stalls the rest ──► decouple with async queue + DLQ
```

---

## The Core Problem

Under load, an AI system's *average* latency can look fine while its *tail* (p95/p99) blows up — and the reason is almost always that requests are waiting in a queue for a busy resource, not that any single request got slower. An LLM replica processing a request holds a GPU for the full generation; while it is busy, newly arriving requests wait. As arrival rate approaches service rate, queueing delay grows non-linearly (a queueing-theory fact: as utilization → 100%, wait time → ∞). The interview question "how does this handle a spike?" is really "what happens to your queue, and does the system degrade gracefully or collapse?"

Two failure modes must be separated, because they demand opposite fixes:

- **Compute-bound** — the resource itself is saturated even at low concurrency: each request is expensive (long generation, large model, big prefill) and the GPU is the ceiling. The remedy is to do the work more efficiently (batching many requests into one GPU pass) or add more GPU replicas.
- **Queue-bound** — the resource could keep up at steady state, but bursts arrive faster than you drain them, so the *waiting* dominates. The remedy is to add capacity fast (autoscaling), refuse or defer excess work gracefully (load shedding / rate limiting), or decouple the burst from the slow stage entirely with an async queue.

A design that only makes single requests faster (covered in `01-latency-and-response-time-patterns.md`) does nothing for a queue-bound spike; a design that only adds replicas wastes money on a compute-bound job that batching would fix. You must name which one you're facing first.

---

## Resolution Options

| Option | How it works | Buys you | Costs you | Reach for it when… |
|---|---|---|---|---|
| **Horizontal replicas + autoscaling** | Run N identical stateless model workers behind a load balancer; scale N on a signal (queue depth, ongoing requests) | Linear throughput scaling; elasticity for bursts | Cold-start / warm-up latency on scale-up; cost; needs stateless design | Traffic varies or exceeds one replica's capacity |
| **Bounded queue + backpressure + load shedding** | Buffer requests in a fixed-depth queue; when full, reject (429) or signal slow-down upstream | System degrades gracefully instead of OOM/collapse; protects tail | Some requests refused; requires client retry/backoff | Any tier facing bursty or adversarial load |
| **Dynamic / continuous batching** | Group concurrent requests into one GPU forward pass; continuous batching swaps finished sequences out and new ones in per decode step | Much higher tokens/s and GPU utilization | Adds up to `batch_wait_timeout` latency; batch-vs-latency trade-off | GPU-bound inference with concurrent requests |
| **Per-tenant rate limiting** | Cap req/s or tokens/s per API key/tenant (token bucket) | Fairness; one tenant can't starve others; cost control | Legitimate bursts throttled; needs quota design | Multi-tenant APIs, shared model pool |
| **Async queue decoupling (SQS/Kafka) + DLQ** | Producer writes a job to a durable queue; workers consume at their own pace; failures land in a dead-letter queue | Absorbs spikes; isolates slow/failing stages; retries | Eventual (not synchronous) results; more moving parts | Long-running or multi-stage pipelines (ingest, batch eval, agent jobs) |
| **Stateless serving + shared state** | Keep no per-request state in the replica; put session, retrieval index, cache in external stores | Any replica serves any request → free horizontal scaling | Network hop to state; shared store becomes the new bottleneck | Whenever you want to scale the serving tier at all |
| **Scale retrieval (shard / replica / index type)** | Shard the vector index across nodes for capacity, replicate for read QPS, pick ANN index (HNSW/IVFFlat) for speed-recall | Retrieval keeps up as corpus and QPS grow | Recall/latency trade-off; sharding complexity; rebuild cost | Vector search is the throughput or latency bottleneck |

**Horizontal replicas + autoscaling** — each replica is an identical worker process running the model; a load balancer spreads requests across them, and an autoscaler adjusts the replica count from a demand signal. Ray Serve, for example, autoscales on queue size: `num_replicas="auto"` targets an average `target_ongoing_requests` per replica (default 2, with `max_ongoing_requests` default 5) between `min_replicas` and `max_replicas` (default 100). It appears as `num_replicas` / `autoscaling_config` in a serving config, or an HPA in Kubernetes. The catch is scale-up is not instant — new GPU replicas take seconds-to-minutes to warm, so the queue must absorb the burst while capacity arrives.

**Bounded queue + backpressure + load shedding** — an unbounded queue is a bug: under sustained overload it grows until the process runs out of memory and every request times out. A *bounded* queue caps in-flight work; when it fills, the system pushes back — either by returning `429 Too Many Requests` (load shedding) or by signalling upstream to slow down (backpressure). This appears as a `max_ongoing_requests` / concurrency limit on a deployment and an explicit rejection path. The mental shift: refusing 5% of requests fast is better than accepting 100% and failing all of them slowly.

**Dynamic / continuous batching** — GPUs are throughput machines: running one request underutilizes the hardware, so a batcher collects concurrent requests and runs them in a single vectorized forward pass. Ray Serve's `@serve.batch(max_batch_size, batch_wait_timeout_s)` waits up to the timeout for a full batch, then flushes. For LLMs specifically, *continuous* (a.k.a. iteration-level) batching — as in vLLM — goes further: it schedules at the token step, evicting finished sequences and admitting new ones every decode iteration, which vLLM reports can raise throughput dramatically versus static batching while keeping p50 latency low. The knob is the batch-vs-latency trade-off: bigger batches raise tokens/s but add wait time to the first requests.

**Per-tenant rate limiting** — in a shared model pool, one tenant's spike can starve everyone else and blow your token budget. A token-bucket limiter grants each tenant a refill rate and a burst allowance, keyed on API key or tenant ID, rejecting or queuing over-limit requests. It appears as middleware (or an API-gateway rule) in front of the model. The design decision is the unit: rate-limit on **tokens/s**, not just req/s, because LLM cost and GPU time scale with tokens, not request count. Ties into `04-cost-and-token-efficiency-patterns.md`.

**Async queue decoupling (SQS/Kafka) + DLQ** — a synchronous request holds a connection for the whole job; for long or multi-stage work (document ingestion, batch evaluation, multi-step agent runs) that's fragile under bursts. Instead the producer writes a job to a durable queue and returns immediately; workers pull at their own sustainable rate. Kafka partitions a topic across brokers so many consumers read in parallel (throughput scales with partitions); SQS uses a redrive policy with `maxReceiveCount` to move poison messages to a **dead-letter queue** after N failed receives, so one bad job never blocks the pipeline. It appears as a queue between pipeline stages plus a DLQ + alarm. See `05-reliability-and-failure-handling-patterns.md`.

**Stateless serving + shared state** — you can only add replicas freely if any replica can serve any request; that requires the replica to hold *no* per-request state. Session/conversation state, the retrieval index, and caches move to external shared stores (Redis, a vector DB, Redis Streams for work handoff). It appears as "no sticky sessions" plus a cache/DB client in the worker. The trade-off: you've added a network hop and made the shared store the new scaling concern — which it, in turn, scales by sharding/replication.

**Scale retrieval (shard / replica / index type)** — vector search has three orthogonal scaling axes. *Sharding* splits the index across nodes so a corpus bigger than one machine's memory still fits and writes parallelize. *Replication* adds read copies to raise query QPS. *Index type* trades recall for speed: pgvector's HNSW gives better speed-recall than IVFFlat but slower builds and more memory; IVFFlat is cheaper to build. It appears as index-build parameters (`m`, `ef_construction`) and query knobs (`hnsw.ef_search`, `ivfflat.probes`). For multi-tenant isolation, pgvector recommends partitioning or separate tables so one tenant's vectors don't degrade another's recall.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| `max_ongoing_requests` (concurrency limit) | Max in-flight requests per replica before queueing/shedding | Set to the point where per-request latency stays within SLO; beyond it, tail degrades — shed instead of queueing forever |
| `target_ongoing_requests` (autoscale target) | Avg concurrent requests per replica the autoscaler holds | Lower it for long/expensive requests and tight latency SLOs; raise it to pack cheap requests and cut cost |
| `min_replicas` / `max_replicas` | Floor and ceiling of the replica pool | Set `min` to cover baseline traffic (avoid cold starts); set `max` ~20% above expected peak to cap cost |
| `max_batch_size` | Requests (or tokens) fused into one GPU pass | Prefer a power of 2 up to GPU memory limit; larger = higher tokens/s but higher first-request latency |
| `batch_wait_timeout_s` | How long the batcher waits to fill a batch | Set well below `SLO − batch_compute_time` (e.g. SLO 150 ms, compute 100 ms → keep timeout ≪ 50 ms) |
| Rate limit (tokens/s per tenant) | Per-tenant throughput ceiling | Base it on tokens/s (tracks GPU cost), sized to fair-share of pool capacity plus a burst allowance |
| SQS `maxReceiveCount` (redrive) | Retries before a message goes to the DLQ | Set high enough for transient errors (e.g. 3–5); too low DLQs on one blip, too high replays poison messages forever |
| HNSW `ef_search` / IVFFlat `probes` | Search-time recall vs speed for the vector index | Raise until recall hits target on a labelled set, then stop — each increment costs query latency |

### Worked Example: Requirement → Decision

**Given:** A RAG API for 3 enterprise tenants shares one GPU model pool. At steady state p95 is 900 ms, but during one tenant's nightly bulk-import burst the whole pool's p99 jumps to 14 s and unrelated tenants time out. GPUs are ~60% utilized even during the spike.

- **Step 1 — Identify the goal:** Protect tail latency and fairness under a bursty, single-tenant spike without over-provisioning GPUs.
- **Step 2 — Define inputs:** Concurrent RAG requests per tenant, current single shared queue, GPU utilization telemetry (~60% during spike ⇒ not compute-bound).
- **Step 3 — Define outputs:** Bounded, fair per-tenant throughput; graceful shedding/deferral of the burst; steady tail for the other tenants.
- **Step 4 — Apply constraints:** Multi-tenant fairness; fixed GPU budget; bulk import is not latency-sensitive but interactive queries are; ~60% GPU util means the ceiling is *queueing*, not compute.
- **Step 5 — Select the approach:** This is **queue-bound**, not compute-bound (GPUs have headroom). Add **per-tenant token-bucket rate limiting** so the importing tenant can't monopolize the pool, and **decouple the bulk import onto an async queue (SQS/Kafka) + DLQ** so it drains at a sustainable rate off the interactive path; keep a **bounded queue with load shedding** on the sync API as a safety valve. Rationale vs alternatives: adding GPU replicas would burn budget on a non-compute bottleneck; dynamic batching alone raises throughput but doesn't stop one tenant from starving others — only rate limiting restores fairness.

---

## Implementation

```python
# Scenario: a GPU-bound embedding/inference service is underutilized because
# each request runs alone. We must raise tokens/s and GPU utilization under
# concurrent load by fusing requests into one vectorized forward pass, while
# capping added latency so we don't blow the SLO. Ray Serve dynamic batching.
from typing import List
import numpy as np
from ray import serve


@serve.deployment(num_replicas="auto")  # autoscale on queue depth
class Embedder:
    # max_batch_size fuses up to 32 requests; batch_wait_timeout_s bounds the
    # extra latency the first request in a batch pays waiting for the batch.
    @serve.batch(max_batch_size=32, batch_wait_timeout_s=0.01)
    async def __call__(self, texts: List[str]) -> List[np.ndarray]:
        # One GPU forward pass over the whole batch instead of 32 passes.
        return self.model.encode(texts)  # returns a list, one per input
```

```python
# Anti-pattern: an UNBOUNDED in-memory queue "so we never drop a request."
# Under a sustained spike the queue grows without limit until the process OOMs
# and EVERY request times out — the system collapses instead of degrading.
import asyncio

work = asyncio.Queue()  # no maxsize → unbounded

async def enqueue(req):
    await work.put(req)  # always accepts, even at 100k backlog → memory blows up


# Correct approach: a BOUNDED queue that sheds load when full. Refusing a small
# fraction fast (429 + Retry-After) protects the tail for everyone else and lets
# the client back off — graceful degradation instead of total collapse.
work = asyncio.Queue(maxsize=1000)  # bounded: this is the backpressure signal

async def enqueue(req):
    try:
        work.put_nowait(req)              # accept only if capacity remains
    except asyncio.QueueFull:
        raise HTTP429(retry_after=1)      # load-shed; client retries with backoff
```

```python
# Scenario: a document-ingestion pipeline (embed -> upsert to vector DB) must
# survive a 50x nightly burst without stalling the interactive query path or
# losing jobs on a transient failure. Decouple with a durable queue; poison
# messages fall to a DLQ after maxReceiveCount so one bad doc can't block it.
import boto3

sqs = boto3.client("sqs")

# Producer returns immediately; workers drain at a sustainable rate.
def submit_ingest(doc_id: str):
    sqs.send_message(QueueUrl=INGEST_QUEUE_URL, MessageBody=doc_id)

# Worker: on repeated failure, SQS auto-moves the message to the DLQ configured
# via the source queue's redrive policy (maxReceiveCount=5) — no manual retry loop.
def worker_poll():
    msgs = sqs.receive_message(QueueUrl=INGEST_QUEUE_URL, MaxNumberOfMessages=10)
    for m in msgs.get("Messages", []):
        embed_and_upsert(m["Body"])       # if this raises, message becomes visible
        sqs.delete_message(               # delete only on success (at-least-once)
            QueueUrl=INGEST_QUEUE_URL, ReceiptHandle=m["ReceiptHandle"]
        )
```

---

## Common Pitfalls & Misconceptions

- **Confusing compute-bound with queue-bound** — beginners see high tail latency under load and reflexively add GPU replicas, because "slow = need more power." If utilization has headroom the bottleneck is queueing, and the real fixes are batching, rate limiting, or decoupling; always read utilization before scaling out.
- **Unbounded queues "so nothing is dropped"** — it feels safer to accept every request than to reject any, but an unbounded queue converts a survivable overload into a total OOM collapse where *all* requests fail; a bounded queue that sheds a small fraction fast keeps the system alive and its tail bounded.
- **Rate limiting on requests instead of tokens** — teams cap req/s because that's the classic web pattern, but LLM cost and GPU time scale with *tokens*, so one request generating 4k tokens is not equal to one generating 40; limit on tokens/s to actually bound load and spend.
- **Stateful replicas / sticky sessions** — storing conversation or cache state inside a replica feels efficient (no network hop), but it pins users to specific replicas and blocks free horizontal scaling and clean autoscaling; externalize all state to a shared store so any replica can serve any request.

---

## Key Definitions

| Term | Definition |
|---|---|
| Throughput | Work completed per unit time — requests/second or tokens/second — the primary scaling metric distinct from per-request latency. |
| Compute-bound | The serving resource (GPU/CPU) is saturated even at low concurrency; fixed by batching or adding replicas. |
| Queue-bound | Work arrives faster than it is drained, so waiting-in-queue dominates latency; fixed by capacity, shedding, or decoupling. |
| Backpressure | A signal from a saturated component telling upstream producers to slow down or stop, preventing unbounded buildup. |
| Load shedding | Deliberately rejecting excess requests (e.g. HTTP 429) when at capacity, to protect the latency and availability of accepted work. |
| Continuous batching | Iteration-level LLM batching that admits/evicts sequences each decode step, raising GPU throughput without waiting for a whole batch to finish. |
| Dead-letter queue (DLQ) | A queue that captures messages a consumer failed to process after `maxReceiveCount` attempts, isolating poison messages for debugging. |
| Sharding (vector search) | Splitting a vector index across nodes so a corpus larger than one machine fits and writes/queries parallelize. |

---

## Summary / Quick Recall

- Classify the bottleneck first: compute-bound (batch/add replicas) vs queue-bound (capacity/shed/decouple) — they need opposite fixes.
- Horizontal replicas + autoscaling scale throughput, but scale-up isn't instant; the queue must absorb the burst while capacity arrives.
- Every queue must be bounded; when full, shed load (429) or apply backpressure — never let it grow unbounded.
- Dynamic/continuous batching trades a little first-request latency for large GPU-throughput gains; tune `max_batch_size` vs `batch_wait_timeout_s` against the SLO.
- Rate-limit per tenant on tokens/s (not req/s) for fairness and cost control in a shared pool.
- Decouple long/bursty/multi-stage pipelines with an async queue (SQS/Kafka) + DLQ; keep the serving tier stateless with state in a vector DB / cache.
- Scale retrieval on three axes: shard for capacity, replicate for QPS, choose the index type (HNSW vs IVFFlat) for the speed-recall point you need.

---

## Self-Check Questions

1. What is the difference between a compute-bound and a queue-bound scalability problem, and what one measurement quickly tells them apart?

   <details><summary>Answer</summary>

   Compute-bound means the serving resource (e.g. GPU) is saturated even at low concurrency, so each request is genuinely expensive; queue-bound means the resource could keep up but bursts arrive faster than they're drained, so waiting-in-queue dominates. The quick discriminator is **resource utilization**: if the GPU is near 100% the problem is compute-bound (batch / add replicas); if utilization has headroom while tail latency spikes, it's queue-bound (add capacity, shed, or decouple). Answering "just add replicas" is incomplete because that only helps the compute-bound case and wastes money on a queue-bound one.

   </details>

2. A model-serving tier hits its concurrency limit during a spike. You must protect p99 for accepted requests. What should the tier do with the excess, and why not just queue it all?

   <details><summary>Answer</summary>

   It should load-shed the excess — return `429 Too Many Requests` (ideally with `Retry-After`) so clients back off — while keeping a *bounded* queue as a small buffer. Queueing everything is the tempting wrong answer: an unbounded queue grows until the process OOMs and *every* request times out, and even a large bounded queue inflates p99 because accepted requests wait behind the whole backlog. Shedding a small fraction fast keeps the tail bounded for the requests you do accept.

   </details>

3. You have a GPU-bound embedding service that's only 40% utilized under concurrent load, and per-request latency is fine. Which pattern raises throughput, and what parameter trade-off must you manage?

   <details><summary>Answer</summary>

   Dynamic batching — fuse concurrent requests into one vectorized GPU forward pass (e.g. Ray Serve's `@serve.batch`), which raises GPU utilization and tokens/s. The trade-off is `max_batch_size` vs `batch_wait_timeout_s`: a larger batch and longer wait raise throughput but add latency to the first requests in the batch, so keep the timeout well below `SLO − batch_compute_time`. Adding replicas is the tempting wrong pick — it costs more GPUs to fix a *utilization* problem that batching solves for free.

   </details>

4. **Which TWO** of the following are required to horizontally scale a model-serving tier with autoscaling *and* keep any replica able to serve any request?
   - A. Store conversation/session state inside each replica for speed
   - B. Externalize session, retrieval index, and cache to shared stores (Redis, vector DB)
   - C. Make replicas stateless so requests can land on any of them
   - D. Use sticky sessions to pin each user to one replica
   - E. Give each replica its own private in-memory cache as the source of truth

   <details><summary>Answer</summary>

   **B and C.** Statelessness (C) lets the load balancer send any request to any replica, which is what makes adding/removing replicas and autoscaling work; externalizing state to shared stores (B) is how you achieve that statelessness without losing session/retrieval data. A, D, and E all pin state to individual replicas: sticky sessions (D) and per-replica state (A/E) break free horizontal scaling and clean scale-down. D is the most tempting distractor because sticky sessions are a real load-balancer feature — but they defeat elasticity by tying users to specific replicas.

   </details>

5. Two proposals for a shared multi-tenant LLM pool where one tenant's bulk job starves the others (GPUs ~60% utilized during the incident): (a) add more GPU replicas, or (b) add per-tenant token-based rate limiting plus move the bulk job to an async queue. Which addresses the root cause, and why?

   <details><summary>Answer</summary>

   Option (b). At ~60% GPU utilization the pool is not compute-bound — the problem is *fairness and queueing*: one tenant floods the shared queue and starves the rest. Per-tenant token-bucket rate limiting bounds each tenant's share, and moving the non-interactive bulk job to an async queue drains it off the latency-sensitive path. Option (a) is the tempting distractor: adding GPUs spends budget on capacity that's already underused and still lets the greedy tenant monopolize whatever capacity exists, so the starvation persists. Rate limiting on *tokens* (not requests) is what actually bounds an LLM tenant's load, since cost scales with tokens.

   </details>

---

## Further Reading

- [Ray Serve — Dynamic Request Batching](https://docs.ray.io/en/latest/serve/advanced-guides/dyn-req-batch.html) — *verified 2026-07-29* — `@serve.batch(max_batch_size, batch_wait_timeout_s)` mechanics and the guidance to keep the wait timeout below `SLO − batch_compute_time`.
- [Ray Serve — Autoscaling](https://docs.ray.io/en/latest/serve/autoscaling-guide.html) — *verified 2026-07-29* — `num_replicas="auto"`, `target_ongoing_requests` / `max_ongoing_requests`, and `min/max_replicas` for queue-driven horizontal scaling.
- [vLLM — Documentation](https://docs.vllm.ai/en/latest/) — *verified 2026-07-29* — PagedAttention and continuous (iteration-level) batching for high-throughput, low-p50-latency LLM serving.
- [Amazon SQS — Using dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) — *verified 2026-07-29* — redrive policy, `maxReceiveCount`, and DLQ retention semantics for isolating poison messages in decoupled pipelines.
- [Apache Kafka — Introduction](https://kafka.apache.org/intro) — *verified 2026-07-29* — topics, partitions, and replication as the basis for parallel, decoupled, horizontally scalable event pipelines.
- [Redis — Streams](https://redis.io/docs/latest/develop/data-types/streams/) — *verified 2026-07-29* — consumer groups, `XREADGROUP`/`XACK`, pending-entries list, and `XCLAIM`/`XAUTOCLAIM` for scalable, at-least-once work handoff between stateless workers.
- [pgvector — README (indexing & scaling)](https://github.com/pgvector/pgvector) — *verified 2026-07-29* — HNSW vs IVFFlat index types, `ef_search`/`probes` speed-recall knobs, and sharding/partitioning guidance for scaling vector search.
