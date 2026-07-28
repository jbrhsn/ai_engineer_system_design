# Scalability, Load Balancing, and Caching

**Section:** Classical Systems Refresher — Scalability & Distributed Systems Primer | **Est. time:** 2.5 hrs | **Interview relevance:** High — every AI system-design interview eventually asks "how does this scale?", and inference is expensive enough that the answer is graded hard.

---

## TL;DR

Scalability is the ability to serve more load by adding resources; the two axes are **vertical** (a bigger box) and **horizontal** (more boxes behind a load balancer). Load balancers fan traffic across a pool of stateless replicas using an algorithm (round-robin, least-connections, etc.) and drop unhealthy members via health checks. Caching removes repeated work — for an AI system that means caching embeddings, LLM responses, and even semantically-similar prompts, which is the single biggest lever on both latency and GPU cost. **The one thing to remember: horizontal scaling only works if your replicas are stateless — the moment a replica holds state a client depends on, your load balancer becomes a correctness bug.**

---

## ELI5 — Explain It Like I'm 5

Imagine a busy coffee shop. When the line gets long you have two choices: hire one super-fast barista who can move quicker (vertical scaling — but there's a limit to how fast one person can go), or open more identical registers and put a host at the door who sends each new customer to whichever register is free (horizontal scaling with a load balancer). For the "more registers" plan to work, every register must be interchangeable — if register 3 is the only one holding your half-made latte, the host can't send your refill to register 1. That is why replicas must be *stateless*: the half-made latte (your session, your conversation history) has to live in a shared fridge everyone can reach, not on one counter. Caching is the pre-made batch of drips behind the counter: if someone orders plain coffee that's already brewed, you hand it over instantly instead of grinding beans again. The common misconception is that a cache "stores the answer forever" — really it stores a *copy that can go stale*, so the hard part isn't storing, it's deciding when to throw the copy out.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Compare vertical and horizontal scaling and choose the right axis for an LLM/RAG serving tier under a cost and latency constraint.
- [ ] Explain how a load balancer distributes requests, select a load-balancing algorithm, and describe how health checks remove failed GPU workers.
- [ ] Diagnose why a stateful replica breaks horizontal scaling and redesign it to externalize state.
- [ ] Design a multi-layer cache (exact-match, embedding, semantic) for a RAG pipeline and set a defensible TTL and similarity threshold.
- [ ] Identify the failure modes of naive caching (staleness, thundering herd, per-replica cache) and the correct mental model for each.

---

## Visual Overview

### Vertical vs Horizontal Scaling

```
VERTICAL (scale up)                 HORIZONTAL (scale out)

   ┌──────────────┐                   ┌────┐ ┌────┐ ┌────┐
   │  1 GPU node  │                   │rep1│ │rep2│ │rep3│
   │  A10  ──►    │                   └──┬─┘ └──┬─┘ └──┬─┘
   │  A100 ──►    │                      └──────┼──────┘
   │  H100        │                             │
   └──────────────┘                        ┌────┴─────┐
   hard ceiling: biggest              ──►  │ Load Bal │  ◄── add replicas,
   single box you can buy                  └────┬─────┘      near-linear
                                               clients
```

### Load-Balancer Fan-Out with Health Checks

```
                       ┌───────────────────────┐
   client ──► request  │     Load Balancer     │
                       │  algo: least_conn     │
                       │  health check /healthz│
                       └───┬──────┬──────┬─────┘
                           │      │      │
                        ┌──▼─┐ ┌──▼─┐ ┌──▼─┐
                        │GPU1│ │GPU2│ │GPU3│
                        │ ok │ │ ok │ │FAIL│ ◄── max_fails hit:
                        └────┘ └────┘ └────┘     removed from pool,
                                                  probed again later
```

### Cache-Aside Read Path (with miss)

```
                       ┌─────────┐  hit   ┌──────────────┐
   request ──► key ──► │  Cache  │ ─────► │ return copy  │ ──► fast
                       └────┬────┘        └──────────────┘
                            │ miss
                            ▼
                   ┌─────────────────┐   ┌──────────────┐
                   │ compute / LLM / │──►│ write to cache│──► return
                   │ DB / embed      │   │ with TTL      │
                   └─────────────────┘   └──────────────┘
```

### RAG Request: Where Each Cache Layer Sits

```
query ─► [1 semantic cache] ─miss─► [2 embed query]
              │ hit                       │ (embedding cache reuses vectors)
              ▼                           ▼
          answer                     retrieve ─► rerank ─► [3 LLM]
                                                              │
                                                     [response cache: exact prompt]
```

---

## Key Concepts

### Vertical vs Horizontal Scaling

**What is it?** Vertical scaling ("scale up") means giving one machine more resources — a bigger GPU, more VRAM, more CPU. Horizontal scaling ("scale out") means running more identical instances and spreading load across them.

**How does it work mechanistically?** Vertical scaling is bounded by the largest single node you can buy and usually requires a restart/redeploy, so it has a hard ceiling and no fault tolerance (one box, one failure domain). Horizontal scaling adds independent replicas behind a load balancer; throughput grows roughly linearly with replica count as long as the replicas share nothing and a shared bottleneck (a database, a single cache node) doesn't cap them. A control loop can add or remove replicas automatically in response to a measured signal.

**Where does it appear in real systems?** In Kubernetes the `HorizontalPodAutoscaler` watches a metric (e.g. average CPU utilization) and adjusts the replica count between `minReplicas` and `maxReplicas` to hold a target. For LLM inference, *within* one replica you scale vertically-in-spirit across GPUs using vLLM's `--tensor-parallel-size` (split one model across GPUs on a node) and `--pipeline-parallel-size` (split across nodes); *across* replicas you scale horizontally by running more vLLM servers behind a load balancer. vLLM's startup log even prints the `GPU KV cache size` and `Maximum concurrency` so you can tell whether to add GPUs (fit the model) or add replicas (serve more concurrent requests).

### Load Balancing and Algorithms

**What is it?** A load balancer is a component that accepts incoming requests and distributes them across a pool of backend targets, monitoring their health so traffic only goes to healthy ones.

**How does it work mechanistically?** The balancer keeps a list of upstream targets and picks one per request using an algorithm: *round-robin* (rotate evenly), *least-connections* (send to the target with the fewest in-flight requests — fairer when request durations vary), *least-time* (lowest average response time plus fewest connections), or *ip-hash* (hash the client IP so a client sticks to one target). Passive health checks mark a target failed after a number of consecutive errors and stop routing to it for a cool-down window, then probe it again with live traffic. Weights let you send proportionally more traffic to bigger targets.

**Where does it appear in real systems?** NGINX configures this in an `upstream` block with directives `least_conn`, `least_time header`, `ip_hash`, per-server `weight=`, and `max_fails` / `fail_timeout` for passive health checks. AWS Elastic Load Balancing does the same as a managed service, distributing across EC2 instances/containers/IPs in multiple Availability Zones, running health checks, and auto-registering/deregistering targets as EC2 Auto Scaling adds or removes them. For GPU inference, `least_conn` is usually the right default because LLM request durations vary wildly (a 20-token vs a 2,000-token completion), so pure round-robin overloads whichever worker drew the long jobs.

### Statelessness and Session Affinity

**What is it?** A stateless replica keeps no client-specific data between requests that another replica couldn't reconstruct; session affinity ("sticky sessions") is the fallback of pinning a client to one replica when you cannot make the replica stateless.

**How does it work mechanistically?** With round-robin or least-connections, consecutive requests from the same client can land on *different* replicas — there is no guarantee of the same server. If replica A holds the client's conversation history in local memory, replica B has no idea, and the answer is wrong or the session "resets." The fix is to push state into a shared store (Redis, a database, an object store) that every replica reads; the replicas then become interchangeable. Sticky sessions (ip-hash) route a client back to the same replica, but they break horizontal scaling's promise: load skews, and if that replica dies the state is gone.

**Where does it appear in real systems?** In agentic/RAG serving this is the conversation memory and any per-session scratchpad — put it in Redis or Postgres keyed by session ID, not in a Python global. vLLM's per-request KV cache lives on the GPU worker handling that request; if you route follow-up turns to a *different* replica you lose the prefix-cache benefit, which is a real reason teams use prefix-aware routing rather than plain round-robin. NGINX exposes stickiness via the `ip_hash` directive — treat it as a last resort, not a design default.

### Caching Patterns: Cache-Aside, TTL, and Invalidation

**What is it?** A cache is a fast store holding copies of expensive-to-produce results so repeated requests skip the expensive work. Cache-aside is the pattern where the application checks the cache first and populates it on a miss.

**How does it work mechanistically?** On a read the app asks the cache; a *hit* returns the copy immediately, a *miss* triggers the real computation (DB query, embedding call, LLM generation) whose result is then written back with a time-to-live (TTL). Correctness hinges on invalidation: the copy can go stale when the underlying data changes. Simple systems use a fixed TTL and accept bounded staleness; more advanced ones send explicit invalidation messages. Redis's client-side caching (Tracking) does exactly this — the server remembers which keys a client read and pushes an `INVALIDATE` message when another client modifies that key, so local copies are evicted instead of served stale.

**Where does it appear in real systems?** For AI systems the highest-value caches are: an **embedding cache** (the same document chunk or query text always produces the same vector, so cache `text ──► vector` keyed by a hash of the text) and an **exact-match response cache** (identical prompt + params ──► identical completion). Redis client-side caching recommends putting a max TTL on every key even if it "never" changes, to bound the blast radius of a stale copy or a lost invalidation connection — good discipline for an embedding cache whose model version might change under you.

### Semantic Caching for LLM/RAG

**What is it?** A semantic cache returns a stored answer when a *new* query is close enough in meaning to a previously-answered one, even if the wording differs — matching on embedding similarity rather than exact string equality.

**How does it work mechanistically?** Each incoming query is embedded into a vector; the cache does a nearest-neighbour lookup against stored query vectors. If the closest stored query is within a similarity threshold, its cached answer is returned (a "semantic hit"); otherwise it's a miss and the full RAG/LLM path runs, and the new query+answer are stored. The threshold is the safety dial: too loose and you return a confidently-wrong answer to a subtly-different question; too tight and hit rate collapses to that of an exact-match cache.

**Where does it appear in real systems?** It sits as a layer *in front of* retrieval and generation, typically backed by the same vector store you already run for RAG. It reuses the embedding cache from the previous concept (you already computed the query vector). It is the highest-leverage cost lever in production LLM serving because a semantic hit skips retrieval, reranking, *and* the GPU generation entirely — but it is also the easiest to get dangerously wrong, which is why it needs an aggressive TTL and human-auditable hit logging.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Load-balancing algorithm | How requests are assigned to backends | Use `least_conn` when request durations vary (LLM completions); use round-robin only for uniform, fast requests; use `ip_hash` only if you truly cannot externalize state. |
| `max_fails` / `fail_timeout` (health check) | Consecutive errors before a backend is dropped, and how long it stays out | Lower `max_fails` (1–2) for GPU workers where a crash means every routed request fails; raise `fail_timeout` only if restarts are slow, or you'll flap. |
| HPA `targetCPUUtilizationPercentage` (or custom metric) | Utilization the autoscaler holds by adding/removing replicas | Set below the point where p95 latency degrades (often 60–70%), not 90% — you need headroom to absorb a spike before new replicas are ready. |
| HPA `minReplicas` / `maxReplicas` | Floor and ceiling on replica count | Set `minReplicas ≥ 2` for availability (never a single point of failure); set `maxReplicas` at your GPU-budget ceiling so autoscaling can't bankrupt you. |
| Cache TTL | How long a cached copy is served before expiry | Set to the tolerable staleness window; for embeddings tie it to model-version changes, for LLM responses keep short if prompts include volatile context (dates, retrieved docs). |
| Semantic-cache similarity threshold | How close a new query must be to reuse an answer | Start strict (high cosine similarity, e.g. ≥ 0.95) and loosen only with offline eval showing acceptable answer reuse; log every hit for audit. |
| vLLM `--tensor-parallel-size` | Number of GPUs one model replica is split across on a node | Set to GPUs-per-node when the model doesn't fit on one GPU; keep at 1 if it fits (avoid needless cross-GPU communication overhead). |

---

### Worked Example: Requirement → Decision

**Given:** A customer-support RAG chatbot currently serves 500 QPS on 2 GPU replicas. Marketing is about to drive traffic to a projected 5,000 QPS peak. The product owner wants p95 latency ≤ 2 s and a hard monthly GPU budget that rules out simply running 20 replicas 24/7.

**Step 1 — Identify the goal:** Absorb a 10× traffic increase within a p95 latency budget *without* a 10× GPU bill.

**Step 2 — Define inputs:** Incoming user queries (highly repetitive — support questions cluster around a few dozen intents), a vector store of support docs, an LLM served by vLLM, and a Redis instance already in the stack.

**Step 3 — Define outputs:** A grounded answer per query, served under 2 s at p95, with GPU replica count that scales with *effective* (post-cache) load, not raw QPS.

**Step 4 — Apply constraints:** p95 ≤ 2 s; monthly GPU cost ceiling; answers must not be stale (support policies change) or wrong (a loose semantic cache returning the wrong policy is a liability); no single point of failure.

**Step 5 — Select the approach:** (a) Put a **semantic cache** in front of retrieval+generation with a strict similarity threshold and a short TTL — because support queries are repetitive, this deflects a large fraction of traffic from the GPUs entirely, cutting *effective* QPS well below 5,000. (b) Behind the cache, scale replicas **horizontally** with a Kubernetes HPA (`minReplicas: 2`, `maxReplicas` at the budget ceiling, target utilization ~65%) rather than vertically — a single bigger GPU has a hard ceiling and no fault tolerance. (c) Route with **`least_conn`**, not round-robin, because completion lengths vary. Rationale vs alternatives: vertical-only scaling can't reach 10× and adds a single failure domain; horizontal-only without the cache would need ~10× GPUs and blow the budget; the cache + autoscale combination scales cost with real work, and `least_conn` prevents one worker drowning in long generations.

---

## Implementation

```nginx
# Scenario: 500 QPS of LLM completions with wildly varying lengths (20 vs 2000 tokens)
# must be spread across 3 GPU workers so no single worker drowns in long jobs,
# and a crashed worker must be pulled from rotation fast (a crash fails every routed request).
http {
    upstream llm_workers {
        least_conn;                                  # fairer than round-robin for variable durations
        server gpu1.internal:8000 max_fails=2 fail_timeout=15s;
        server gpu2.internal:8000 max_fails=2 fail_timeout=15s;
        server gpu3.internal:8000 max_fails=2 fail_timeout=15s;
    }
    server {
        listen 80;
        location /v1/ {
            proxy_pass http://llm_workers;
        }
    }
}
```

```python
# Anti-pattern: cache LLM responses AND hold conversation state in a per-replica
# in-process dict, behind a round-robin load balancer. Two failures at once:
#   1. Each replica has its OWN cache -> hit rate is ~1/N of what it should be, and
#      a cached answer on replica A is invisible to replica B.
#   2. session_state lives on one replica; the next turn may route elsewhere -> lost context.
_response_cache = {}          # local to THIS process
_session_state = {}           # local to THIS process

def handle(session_id, prompt):
    if prompt in _response_cache:
        return _response_cache[prompt]
    history = _session_state.get(session_id, [])   # empty if the turn hit a different replica!
    answer = call_llm(history + [prompt])
    _response_cache[prompt] = answer
    _session_state[session_id] = history + [prompt, answer]
    return answer

# Correct approach: shared Redis for BOTH the response cache and session state, so every
# stateless replica sees the same data. Add a TTL to bound staleness, and key the cache on
# prompt + model + params so a model upgrade doesn't serve stale completions.
import hashlib, json, redis
r = redis.Redis(host="cache", port=6379)

def cache_key(prompt, model, params):
    raw = json.dumps({"p": prompt, "m": model, "x": params}, sort_keys=True)
    return "resp:" + hashlib.sha256(raw.encode()).hexdigest()

def handle(session_id, prompt, model="gpt-x", params=None):
    key = cache_key(prompt, model, params or {})
    hit = r.get(key)
    if hit:
        return hit.decode()
    history = json.loads(r.get(f"sess:{session_id}") or "[]")   # shared -> any replica sees it
    answer = call_llm(history + [prompt])
    r.set(key, answer, ex=3600)                                 # TTL bounds staleness
    r.set(f"sess:{session_id}", json.dumps(history + [prompt, answer]), ex=86400)
    return answer
```

---

## Common Pitfalls & Misconceptions

- **Treating "add a load balancer" as making the system scalable** — beginners assume the LB is the scaling mechanism, when it's only the traffic router. The correct mental model: the LB enables scaling *only if the replicas behind it are stateless*; a load balancer in front of stateful replicas just distributes a correctness bug across more machines.
- **Using round-robin for LLM inference** — it's the default and "looks fair," so people leave it. But completions vary from tens to thousands of tokens, so round-robin piles long jobs on unlucky workers; `least_conn` (or a queue-depth-aware router) matches assignment to actual in-flight work.
- **Per-replica in-process caches** — caching locally feels simplest and needs no extra infra, so beginners reach for a module-level dict. With N replicas your hit rate is roughly divided by N and copies never share; a *shared* cache (Redis) is what makes caching pay off at scale.
- **Forgetting cache invalidation / TTL** — a cache is mentally filed as "storage," so staleness is an afterthought. The correct model is that a cache holds a *copy that can diverge from the truth*; every entry needs a TTL or an invalidation path, and Redis's own guidance is to put a max TTL on every key even when it "never" changes.
- **Setting the semantic-cache threshold too loose to boost hit rate** — a higher hit rate looks like a win on a dashboard, so people relax the similarity bar. But a loose threshold returns a confident answer to a subtly-different question; treat the threshold as a correctness dial, tune it with offline eval, and log every semantic hit for audit.

---

## Key Definitions

| Term | Definition |
|---|---|
| Vertical scaling | Increasing the capacity of a single node (bigger GPU, more RAM); bounded by the largest available box and a single failure domain. |
| Horizontal scaling | Adding more identical replicas behind a load balancer; throughput grows near-linearly if replicas are stateless and share no bottleneck. |
| Load balancer | A component that distributes incoming requests across a pool of healthy backend targets according to an algorithm. |
| Health check | A probe (active or passive) that marks a backend failed and removes it from rotation until it recovers. |
| Statelessness | The property that a replica holds no client-specific data another replica couldn't reconstruct, making replicas interchangeable. |
| Session affinity (sticky sessions) | Pinning a client to one replica (e.g. by IP hash); a fallback for state, not a scaling strategy. |
| Cache-aside | Read pattern: check cache first; on a miss, compute the value and write it back with a TTL. |
| TTL (time-to-live) | The lifetime of a cached entry before it expires, bounding how stale a served copy can be. |
| Semantic cache | A cache that returns a stored answer when a new query is within a similarity threshold of a prior one, matching on meaning rather than exact string. |
| Tensor parallelism | Splitting a single model's weights across multiple GPUs on a node (vLLM `--tensor-parallel-size`) so a model too big for one GPU can run. |

---

## Summary / Quick Recall

- Two scaling axes: **vertical** (bigger box, hard ceiling, single failure domain) and **horizontal** (more replicas, near-linear, needs statelessness).
- A load balancer only *routes*; it makes you scalable **only if replicas are stateless** and share state via Redis/DB/object store.
- Pick **`least_conn`** (not round-robin) for LLM inference because completion lengths vary; use health checks to drop crashed GPU workers fast.
- Cache-aside + TTL is the workhorse; the hard part is **invalidation**, not storage — every entry needs a TTL or an invalidation path.
- AI-specific caches, highest leverage first: **semantic cache** (skips retrieval + generation), **embedding cache** (reuse vectors), **exact-match response cache** (identical prompt + model + params).
- A **semantic-cache similarity threshold** is a correctness dial: start strict, loosen only with eval, and log every hit.
- Autoscale (HPA) on a metric below the latency-degradation point, `minReplicas ≥ 2`, `maxReplicas` at the GPU-budget ceiling.

---

## Self-Check Questions

1. What is the defining difference between vertical and horizontal scaling, and why does horizontal scaling depend on statelessness?

   <details><summary>Answer</summary>

   Vertical scaling adds resources to a single node (a bigger GPU/box); horizontal scaling adds more identical replicas behind a load balancer. Horizontal scaling depends on statelessness because a load balancer may send consecutive requests from the same client to *different* replicas — if a replica holds client-specific state locally, another replica can't serve the follow-up correctly. The tempting wrong answer is "vertical scaling is just slower horizontal scaling"; it's not — vertical scaling has a hard ceiling (the biggest box you can buy) and a single failure domain, so it's qualitatively different, not merely slower.

   </details>

2. A vLLM serving tier uses round-robin load balancing and users report that latency is wildly inconsistent — some requests are fast, others crawl even when overall load is moderate. What is the likely cause and fix?

   <details><summary>Answer</summary>

   LLM completion lengths vary enormously (20 vs 2,000 tokens), so round-robin blindly assigns the next request to the next worker regardless of how busy it is. A worker that drew several long generations gets more piled on, spiking its queue and latency while others idle. The fix is `least_conn` (send to the worker with the fewest in-flight requests) or a queue-depth-aware router. Simply "adding more replicas" is the tempting wrong fix — it raises capacity but doesn't stop round-robin from unevenly loading whichever workers drew the long jobs.

   </details>

3. **Which TWO** of the following changes would meaningfully improve a RAG chatbot's GPU cost and latency under repetitive support-question traffic?
   - A. Switch the load balancer from `least_conn` to `ip_hash`.
   - B. Add a semantic cache in front of retrieval and generation with a strict similarity threshold.
   - C. Move the per-replica in-process response cache to a shared Redis cache.
   - D. Increase `--tensor-parallel-size` from 1 to 4 even though the model fits on one GPU.
   - E. Raise the HPA `targetCPUUtilizationPercentage` from 65% to 95%.

   <details><summary>Answer</summary>

   **B and C.** A semantic cache (B) deflects repetitive queries from the GPUs entirely — the biggest lever, since a hit skips retrieval, reranking, and generation. Moving to a shared cache (C) fixes the per-replica hit-rate-divided-by-N problem so caching actually pays off across replicas. A is wrong: `ip_hash` is a stickiness mechanism, not a cost/latency win, and it skews load. D is wrong: raising tensor-parallel size for a model that already fits one GPU just adds cross-GPU communication overhead. E is the most tempting distractor — higher utilization *looks* cheaper, but at 95% there's no headroom to absorb a spike before new replicas warm up, so p95 latency degrades.

   </details>

4. Your team is debating storing conversation history in a Python module-level dict on each replica "for speed." Analyze the trade-off and state whether this is defensible.

   <details><summary>Answer</summary>

   It is not defensible for a horizontally-scaled service. Local memory is faster per access, but because the load balancer can route a client's next turn to a different replica, that replica sees an empty history and the conversation "resets" — a correctness bug, not just a performance issue. It also makes replicas non-interchangeable, so you can't freely add/remove them or survive a replica crash (state is lost). The correct design externalizes session state to a shared store (Redis/Postgres) keyed by session ID; the small extra latency buys correctness and true statelessness. The only case where sticky sessions (ip_hash) partially rescue local state is when you cannot externalize it at all — treated as a last resort, since it reintroduces load skew and data loss on failure.

   </details>

5. A semantic cache boosts hit rate from 20% to 55% after the team loosens the similarity threshold — but support agents start reporting occasional wrong-policy answers. What went wrong, and how should the threshold be governed?

   <details><summary>Answer</summary>

   The loosened threshold now treats subtly-different questions as equivalent, so the cache returns a stored answer for a query it doesn't actually match — e.g. a refund-policy answer served for a slightly different exchange-policy question. Hit rate is a misleading success metric here because it rises precisely when the cache gets more permissive. The threshold is a *correctness* dial, not a performance dial: it should be set from offline evaluation that measures answer-reuse accuracy, kept strict by default (high cosine similarity), and paired with logging of every semantic hit so wrong reuses are auditable. The tempting wrong conclusion is "semantic caching is unsafe, remove it"; the real fix is to re-tighten the threshold and gate future changes on eval, keeping the deflection benefit without the liability.

   </details>

---

## Further Reading

- [vLLM — Parallelism and Scaling](https://docs.vllm.ai/en/latest/serving/parallelism_scaling.html) — *verified 2026-07-28* — Tensor/pipeline parallelism, single- vs multi-node deployment, and the KV-cache/concurrency signals for when to add GPUs vs replicas. (Fast-evolving: doc is marked "latest developer preview" — re-verify exact flag names before relying on them.)
- [Kubernetes — Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) — *verified 2026-07-28* — The autoscaling control loop, target-utilization mechanism, and min/max replica bounds for horizontal scaling.
- [AWS — What is Elastic Load Balancing?](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html) — *verified 2026-07-28* — Managed traffic distribution across targets and Availability Zones, health checks, and auto-registration with EC2 Auto Scaling.
- [NGINX — Using nginx as an HTTP load balancer](https://nginx.org/en/docs/http/load_balancing.html) — *verified 2026-07-28* — Load-balancing algorithms (round-robin, least_conn, least_time, ip_hash), server weights, and passive health checks via max_fails/fail_timeout.
- [Redis — Client-side caching reference](https://redis.io/docs/latest/develop/reference/client-side-caching/) — *verified 2026-07-28* — Cache invalidation via Tracking, TTL discipline, staleness trade-offs, and bounding cache memory.
