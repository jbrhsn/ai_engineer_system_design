# Scalability and Distributed Systems Primer — Interview Prep

**Section:** Classical Systems Refresher → Scalability & Distributed Systems Primer | **Role target:** Applied AI Engineer / AI Systems Engineer (agentic AI, RAG, multi-agent production systems)

---

## Core Conceptual Questions

These test whether you understand the fundamentals.

| Question | Key points to cover | Common weak-answer trap |
|---|---|---|
| What is the difference between vertical and horizontal scaling, and why does horizontal scaling for an LLM serving tier depend on statelessness? | Vertical = bigger box (hard ceiling, single failure domain, restart to grow); horizontal = more replicas behind a load balancer, near-linear if replicas share nothing. A load balancer may route consecutive requests from one client to *different* replicas, so any per-client state (conversation history, KV cache) must live in a shared store or the follow-up turn is served wrong. | "Just add more servers." Treating the load balancer as the scaling mechanism, and ignoring that a stateful replica behind an LB distributes a correctness bug across more machines. |
| Why is round-robin usually the wrong load-balancing algorithm for LLM inference, and what is the right default? | Completion lengths vary enormously (20 vs 2,000 tokens), so round-robin piles long jobs onto whichever worker drew them while others idle, spiking p95. `least_conn` (or a queue-depth/prefix-aware router) matches assignment to actual in-flight work. Use health checks (`max_fails`/`fail_timeout`) to pull crashed GPU workers fast. | "Round-robin is fair, so it's fine." Confusing *even distribution of request count* with *even distribution of work*. |
| State the CAP theorem precisely, and explain why "pick 2 of 3" is misleading for an AI engineer. | Under a network partition a replicated store can guarantee Consistency (every read sees latest write, or errors) OR Availability (every live node responds, maybe stale) — not both. P is not optional on a real network, so the only live choice is C-or-A *during a partition*; Brewer himself walked back "2 of 3". CAP applies to a replicated cluster under partition, not a single node or a product name. | "CAP means pick 2 of 3." Reading the triangle as an à la carte menu and claiming "we chose CA," which isn't a real running mode. |
| What is PACELC, and why is it the more useful framing day-to-day than CAP? | PACELC: **if P**artition → choose **A** or **C**; **E**lse (normal operation) → choose **L**atency or **C**onsistency. Even with no partition, a consistent read costs a quorum/leader round trip = latency. Partitions are rare; the L↔C dial is always on, so it's what you actually tune (e.g. Cosmos DB's five levels span EL↔EC). | "CAP is the trade-off I tune." Obsessing over the rare partition case and ignoring the ever-present latency-vs-consistency cost on normal reads. |
| Distinguish at-most-once, at-least-once, and exactly-once delivery. Which do SQS/RabbitMQ give by default, and what does that force on your consumer? | At-most-once acks *before* processing (may lose on crash); at-least-once acks *after* success (may duplicate on redelivery); exactly-once needs broker+side-effect transactions and is generally unachievable across an external LLM API. SQS standard and RabbitMQ default to at-least-once, so the consumer **must be idempotent** (dedupe/upsert on a stable key like `document_id`). | "Exactly-once is easy / the queue delivered it so it ran once." Assuming no duplicates and writing a non-idempotent `insert`, which double-writes vectors on the inevitable redelivery. |
| What does a message queue actually buy you, and what does it *not* buy you? | It decouples producer arrival rate from consumer processing rate — smoothing bursts, isolating failures, enabling retries and load-sharing. It does **not** add throughput: if sustained producer rate > sustained consumer rate, the backlog grows unbounded regardless of queue size. Fix by scaling consumers or applying backpressure. | "Add a queue to make it faster / bigger queue fixes the backlog." Confusing buffering with capacity; durability only delays a rate-mismatch failure. |
| When is eventual consistency safe for an AI component, and when is it a correctness bug? | Safe for staleness-tolerant reads (a RAG document index, like counts) where a slightly stale corpus still yields useful context. A bug when a step must observe its *own* prior write — agent conversation memory or a just-updated fraud feature — where you need at least read-your-writes/session consistency (`ConsistentRead=True`, `W + R > RF`). | "Eventually consistent = eventually correct, so it's fine everywhere." The word "eventually" hides that a read *right after* a write can return stale/nothing. |

---

## Applied / Scenario Questions

**Q1:** A customer-support RAG chatbot on 2 GPU replicas serves 500 QPS today. Marketing will drive a 5,000 QPS peak. You have a p95 ≤ 2 s budget and a hard monthly GPU ceiling that rules out running 20 replicas 24/7. How do you absorb 10× traffic without a 10× GPU bill?

**Strong answer framework:**
- Note that support traffic is highly repetitive (clusters around a few dozen intents), so the biggest lever is a **semantic cache** in front of retrieval + generation — a semantic hit skips retrieval, reranking, *and* GPU generation, cutting *effective* (post-cache) QPS well below 5,000.
- Behind the cache, scale **horizontally** with a Kubernetes HPA (`minReplicas ≥ 2`, `maxReplicas` at the budget ceiling, target utilization ~65% so there's headroom before new replicas warm up) rather than vertically, which has a hard ceiling and a single failure domain.
- Route with **`least_conn`**, not round-robin, because completion lengths vary; externalize conversation state to Redis so replicas stay stateless and interchangeable.
- **Tradeoff point (latency vs accuracy vs cost vs safety):** the semantic-cache similarity threshold is a *correctness* dial, not a hit-rate dial. Loosening it raises hit rate (lower cost/latency) but starts returning a confident wrong-policy answer to a subtly-different question (a safety/liability failure). Start strict (e.g. cosine ≥ 0.95), keep a short TTL because support policies change, tune only with offline answer-reuse eval, and log every hit for audit.

**Q2:** You must embed 2 million documents to build a RAG index. The embedding provider allows 3,000 requests/min and intermittently returns `429`/`503`. The ingestion API must not block, and a worker crash must not silently lose documents. Design the async pipeline.

**Strong answer framework:**
- Decouple the slow work: the FastAPI endpoint enqueues jobs and returns `202 Accepted` + a `job_id` immediately; a worker pool drains a **durable point-to-point queue** (SQS standard / RabbitMQ) — pub/sub is wrong since only one consumer type needs each job.
- Use **manual/late ack** (`acks_late=True`) so a crashed worker's message is redelivered (at-least-once), and make the consumer **idempotent** — upsert by `document_id`, never `insert` — so redeliveries can't create duplicate vectors.
- Apply **backpressure**: bound prefetch (~50/worker) plus a client-side token-bucket rate limiter so consumption stays under 3,000 req/min and the queue absorbs the burst; retry `429/503` with **exponential backoff + jitter**; quarantine genuine poison docs to a **DLQ** at `maxReceiveCount=5` and alarm on DLQ depth.
- **Tradeoff point (latency vs accuracy vs cost vs safety):** don't chase true exactly-once — it's unachievable across an external LLM/embedding API and would add cost/complexity. At-least-once + idempotent upsert is cheaper and sufficient; the residual cost is occasional duplicate embedding *calls* (paid twice), which a longer visibility timeout / heartbeat on slow jobs minimizes without risking silent data loss.

---

## System Design / Architecture Questions

**Q:** Design a scalable RAG serving layer that stays available under a vector-DB partition, holds a p95 latency budget, and controls GPU cost.

**Approach:**

1. **Clarify requirements (scale, latency budget, hallucination tolerance, data sensitivity).**
   - Target QPS and its repetitiveness (drives cache leverage); p95/p99 latency budget (e.g. ≤ 2 s); acceptable staleness of the corpus (minutes? hours?); is conversation memory multi-turn (needs read-your-writes)? Cost ceiling on GPUs. Consequence of a stale-but-served answer vs a hard error during a partition — for most RAG retrieval, a slightly stale corpus beats a downed retriever.

2. **Propose high-level architecture (agent topology, retrieval layer, guardrails).**
   - Edge: load balancer with `least_conn` + health checks over stateless API replicas (HPA-autoscaled, `minReplicas ≥ 2`).
   - Cache tiers, highest leverage first: **semantic cache** (front of everything) → **embedding cache** (reuse query/chunk vectors) → **exact-match response cache** keyed on prompt + model + params; all in shared Redis with TTLs, not per-replica dicts.
   - Retrieval: vector DB replicated RF=3, configured **AP/eventual** for the corpus read path so retrieval keeps serving during a partition from a reachable replica.
   - State: conversation memory in a **session / read-your-writes** store (Cosmos DB *Session*, or `LOCAL_QUORUM` with `W + R > RF`) so the agent never "forgets" the turn it just wrote.
   - Async spine: index builds / re-embedding run off a durable queue with idempotent, late-ack workers, DLQ, and backpressure against the embedding provider.

3. **Justify choices and name tradeoffs explicitly (cost, latency, complexity, security).**
   - **Availability under partition:** the corpus index is AP/eventual (a stale doc is tolerable), but conversation state is session-consistent (read-your-writes is a correctness requirement, not a preference) — two different CAP postures in one system, chosen per data class, not globally.
   - **Cost vs latency vs safety:** the semantic cache is the dominant cost lever but its threshold trades hit rate against wrong-answer risk — keep it strict and audited. Autoscaling target below the latency-degradation knee trades a little idle GPU for p95 headroom.
   - **Complexity:** at-least-once + idempotency instead of exactly-once keeps the async path simple and correct. Externalizing state adds a Redis/DB hop (small latency) to buy true statelessness and horizontal scalability.

---

## Vocabulary That Signals Expertise

Use these terms naturally — don't force them:

- **Horizontal scaling / statelessness** — say replicas must be stateless *before* claiming a system scales out; ties scaling to a correctness property, not just a server count.
- **`least_conn` (least-connections)** — cite it as the LLM-serving default over round-robin because completion durations vary.
- **Cache-aside + TTL** — the workhorse read pattern; mention TTL as bounding staleness, and that the hard part is invalidation, not storage.
- **Semantic cache / similarity threshold** — the highest-leverage cost lever in LLM serving; frame the threshold as a correctness dial.
- **PACELC (PA/EL, PC/EC)** — use to describe the always-on latency-vs-consistency trade; labeling a store PA/EL or PC/EC signals you think past CAP.
- **Read-your-writes / session consistency** — the right level for agent memory; shows you can pick the *weakest* sufficient guarantee.
- **Quorum rule `W + R > RF`** — concrete mechanism for read-your-writes without global linearizability.
- **At-least-once + idempotent consumer** — the practical delivery posture; pairs the guarantee with the required consumer property.
- **Visibility timeout / late ack (`acks_late`)** — shows you know *when* a message is deleted governs loss vs duplication.
- **Backpressure (prefetch/QoS + rate limiting)** — the correct response to a rate-limited downstream, vs pulling faster.
- **Dead-letter queue / `maxReceiveCount`** — quarantining poison messages so they don't starve healthy jobs.

---

## Vocabulary That Signals Weakness

Avoid these — they signal outdated or shallow understanding:

- **"Just add more servers."** — Ignores statelessness; adding replicas behind a load balancer does nothing (or breaks correctness) if replicas hold client state.
- **"CAP means pick 2 of 3."** — The classic tell of a menu-style misreading; P isn't optional, so the real choice is C-or-A only during a partition.
- **"We're a CA system."** — CA isn't an achievable running mode on a real network; it reveals the "2 of 3" misconception.
- **"Exactly-once is easy" / "the queue guarantees it ran once."** — Ignores redelivery on crash/timeout; exactly-once is generally unachievable across an external LLM API.
- **"A bigger queue will fix the backlog."** — Confuses buffering with capacity; a sustained rate mismatch grows unbounded regardless of queue size.
- **"Round-robin is fair enough."** — Fair by request *count*, not by *work*, which is what matters when completion lengths vary 100×.
- **"Just cache it forever."** — Treats a cache as storage; every entry needs a TTL or invalidation path or it serves stale/wrong data.
- **"Eventually consistent is basically consistent."** — A read right after a write can be stale; wrong for read-your-writes state like agent memory.

---

## STAR Answer Frame

**Situation:** Our multi-turn LangGraph support agent, running on a horizontally-scaled FastAPI + vLLM tier, began "forgetting" the message a user had just sent, and the accompanying document-ingestion pipeline was silently dropping embeddings during worker restarts — the RAG index had gaps nobody noticed until retrieval quality dropped.

**Task:** I owned making the serving tier correct and cost-stable under a projected traffic spike: fix the memory/statelessness bug, stop silent data loss in the async embedding pipeline, and keep p95 within budget without a linear GPU-cost blowup.

**Action:** (1) Diagnosed the "forgetting" as a statelessness violation — conversation history lived in a per-replica in-process dict behind round-robin, so a follow-up turn routing to a different replica saw empty history. Moved session state to shared Redis keyed by `session_id`, switched routing to `least_conn`, and set the conversation-state store to read-your-writes/session consistency so a turn's own write was always visible. (2) Diagnosed the ingestion loss as auto-ack fire-and-forget: reconfigured Celery workers to `acks_late=True` (at-least-once), made the consumer idempotent with `upsert` by `document_id`, added prefetch-based backpressure + backoff/jitter against the embedding provider's `429`s, and routed poison docs to a DLQ at `maxReceiveCount=5`. (3) Added a strict-threshold semantic cache (cosine ≥ 0.95, short TTL, every hit logged) in front of retrieval + generation and put the GPU tier on an HPA at ~65% target with `minReplicas: 2`.

**Result:** Conversation "forgetting" incidents went to zero; the index-gap bug disappeared (no silent losses across restarts). The semantic cache deflected ~45% of traffic from the GPUs, so the 10× traffic event was served on roughly 4× replicas instead of 10×, holding p95 at ~1.7 s (under the 2 s budget) and keeping GPU spend within the monthly ceiling.

---

## Red Flags Interviewers Watch For

- **Naming a load balancer as "the scaling solution" without mentioning statelessness** — reveals they think routing == scaling and would ship a stateful replica behind an LB.
- **Defaulting to round-robin for LLM inference** and not distinguishing request *count* from request *work*.
- **Reciting "pick 2 of 3"** or claiming a "CA" system — shows CAP was memorized, not understood; strong candidates reframe as CP/AP-under-partition and reach for PACELC.
- **Applying one global consistency level to the whole system** instead of choosing per data class (AP/eventual RAG corpus vs read-your-writes agent memory vs strong feature store).
- **Claiming or assuming exactly-once**, or writing a non-idempotent consumer, and being surprised by duplicates.
- **Acking before the work commits** (auto-ack) and not seeing that it converts a crash into silent data loss.
- **"Bigger queue" as the fix for an unbounded backlog** — mistaking buffering for capacity, missing the sustained-rate-mismatch diagnosis.
- **Tuning a semantic cache by hit rate alone** — treating a correctness/safety dial as a performance metric, with no eval or audit logging.
- **One giant visibility timeout** for long jobs — not knowing to use a modest timeout + heartbeat so genuinely failed jobs still retry promptly.
