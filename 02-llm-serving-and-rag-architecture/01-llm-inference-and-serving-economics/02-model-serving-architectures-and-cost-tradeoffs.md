# Model Serving Architectures and Cost Trade-offs

**Section:** LLM Serving & RAG Architecture → LLM Inference & Serving Economics | **Est. time:** 2.5 hrs | **Interview relevance:** High — "serve model X at Y QPS under a $Z/month GPU budget" is a canonical AI system design prompt, and the interviewer scores you on whether you reason about GPU utilization, batching, and autoscaling rather than just picking a framework.

---

## TL;DR

Model serving turns a set of model weights into an API that answers requests at a target latency and throughput, and the dominant cost line is GPU-hours — so the whole discipline is about keeping expensive accelerators busy. The main levers are the inference server (vLLM, TGI, or a generic runtime that does continuous batching and paged KV-cache), how many models share each GPU (single-model, multi-model, or multi-LoRA), how replicas autoscale (including scale-to-zero and cold starts), and how requests are routed and queued to GPU workers. Reserved and spot GPUs cut the hourly rate but trade away flexibility and availability. **The one thing to remember: an LLM server's cost is set by GPU-hours divided by useful tokens served, so every architecture decision is really a bet on raising GPU utilization without breaking your latency SLO.**

---

## ELI5 — Explain It Like I'm 5

Imagine a gourmet kitchen where the only expensive thing is the giant oven, and you pay for the oven by the minute whether it's baking or empty. Orders (requests) arrive at the front counter, a host (the router) writes them on tickets and lines them up, and the chef slides as many trays into the oven at once as will fit (batching) so the oven is never half-empty. If orders pour in you rent a second identical oven (add a replica); if the restaurant goes quiet at night you can switch an oven off entirely (scale-to-zero) — but a cold oven takes minutes to heat back up, which is the cold-start delay. The common mistake is thinking you save money by buying a bigger oven and leaving it on all day "just in case"; you actually save money by keeping a right-sized oven packed with trays and turning off the ones nobody is using.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Compare inference-server options (vLLM, TGI, generic runtimes) and justify a choice from continuous batching and KV-cache management, not brand familiarity.
- [ ] Design a request router → GPU-worker-pool → autoscaler topology and size replicas by KV-cache headroom rather than raw QPS.
- [ ] Diagnose and mitigate cold starts when using scale-to-zero, and decide when scale-to-zero is even appropriate.
- [ ] Compute a serving cost model in GPU-hours-per-million-tokens and reason about on-demand vs reserved vs spot GPU pricing.
- [ ] Decide when single-model, multi-model, or multi-LoRA (multi-adapter) serving is the cheaper architecture for a given fleet of fine-tunes.

---

## Visual Overview

### Request Router → GPU Worker Pool with Queue

```
                         ┌──────────────────────────────┐
 Clients ──► Router ──►  │  Queue (pending requests)     │
 (QPS)      (LB /        │  [r1][r2][r3][r4]...          │
             gateway)    └───────────────┬──────────────┘
                                         │ continuous batching
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                     ▼
              GPU Worker A        GPU Worker B          GPU Worker C
              (replica, 1 GPU)    (replica, 1 GPU)      (replica, 1 GPU)
              KV-cache: 70% full  KV-cache: 55% full    KV-cache: 90% full
                    │                    │                     │
                    └──────── metrics (queue depth, util) ─────┘
                                         │
                                         ▼
                                   Autoscaler
                            (add/remove replicas)
```

### Autoscaling Replicas Over a Traffic Day

```
requests/s
   ▲
   │            ┌────┐
   │        ┌───┘    └───┐            scale UP  (cold start on new replica)
   │    ┌───┘            └───┐
   │ ───┘                    └────────────  scale DOWN toward min_replicas
   └───────────────────────────────────────────► time
 replicas:  1     2     3     3     2     1  (or 0 if scale-to-zero)
```

### Single-Model vs Multi-Model vs Multi-LoRA Serving

```
Single-model per GPU              Multi-model per GPU            Multi-LoRA (one base + adapters)
┌───────────────┐                 ┌───────────────┐              ┌───────────────────────────┐
│ GPU           │                 │ GPU           │              │ GPU                       │
│  ┌─────────┐  │                 │ ┌────┐ ┌────┐ │              │  ┌─────────────────────┐  │
│  │ Model A │  │                 │ │ Mdl│ │ Mdl│ │              │  │ Base model (shared) │  │
│  │  (full) │  │                 │ │ A  │ │ B  │ │              │  └──────────┬──────────┘  │
│  └─────────┘  │                 │ └────┘ └────┘ │              │   adapterA adapterB ...   │
│ high isolation│                 │ shares VRAM   │              │  (small LoRA deltas)      │
└───────────────┘                 └───────────────┘              └───────────────────────────┘
  cost: 1 GPU / model               cost: N models / GPU           cost: 1 GPU / many fine-tunes
```

### Serving Cost Decision Tree

```
Predictable, high round-the-clock traffic?
├── Yes ──► Reserved / committed-use GPUs (lowest $/hr, always on)
└── No  ──► Bursty or spiky traffic?
            ├── Yes ──► Autoscaled on-demand GPUs (+ scale-to-zero if idle gaps are long
            │            and cold-start latency is tolerable)
            └── No  ──► Interruptible / batch / offline workload?
                        ├── Yes ──► Spot / preemptible GPUs (cheapest, can be reclaimed)
                        └── No  ──► On-demand, min_replicas ≥ 1 (warm baseline)
```

---

## Key Concepts

### Inference servers (vLLM, TGI, generic patterns)

**What it is.** An inference server is the process that loads model weights onto the GPU and exposes a request API (usually OpenAI-compatible), doing the scheduling, batching, and KV-cache management that a naive `model.generate()` loop does not.

**How it works under the hood.** The two mechanisms that make a purpose-built server cheap are **continuous batching** (a.k.a. iteration-level scheduling) and **paged KV-cache**. Instead of waiting to assemble a fixed batch, the scheduler admits and evicts requests every decoding step, so a finished sequence's slot is immediately handed to a waiting request — the GPU never idles between requests of different lengths. Paged KV-cache (vLLM's PagedAttention) stores each sequence's attention keys/values in non-contiguous fixed-size blocks like OS virtual memory, which slashes the memory fragmentation that otherwise wastes VRAM and caps concurrency. TGI ships equivalent continuous batching and paged attention.

**Where it appears in real systems.** In vLLM you start a server with `vllm serve <model>` and it exposes `/v1/completions`; the scheduler is governed by `--max-num-seqs` (max concurrent sequences) and `--max-num-batched-tokens`. TGI (Hugging Face Text Generation Inference) exposes the same style of endpoint and, per its docs, is now in **maintenance mode** — Hugging Face explicitly recommends vLLM or SGLang going forward, which matters when you justify a framework choice in an interview: prefer an actively developed engine unless you have a specific TGI dependency.

### Autoscaling inference, scale-to-zero, and cold starts

**What it is.** Autoscaling adds or removes model replicas in response to load; scale-to-zero is the special case of dropping to zero replicas when idle to stop paying for the GPU entirely.

**How it works under the hood.** An autoscaler watches a signal — queue depth, in-flight requests per replica, or GPU utilization — and compares it to a target. Ray Serve, for example, autoscales on `target_ongoing_requests` (the average concurrent requests per replica it tries to hold) between `min_replicas` and `max_replicas`. Kubernetes' Horizontal Pod Autoscaler (HPA) scales pod count on CPU/memory or custom/external metrics. The **cold start** is the latency penalty when a new (or the first, from zero) replica must be scheduled onto a node, pull a multi-GB container image, download or mount model weights, and load them into VRAM — for a large LLM this is tens of seconds to minutes, so the first request after scale-up waits.

**Where it appears in real systems.** `min_replicas: 0` in a Ray Serve `autoscaling_config` enables scale-to-zero; setting `min_replicas: 1` keeps a warm baseline that eliminates the from-zero cold start at the cost of one always-on GPU. On Kubernetes you typically pair the HPA (pod scaling) with a cluster/node autoscaler that provisions GPU nodes, and cold start includes node provisioning time on top of weight loading.

### GPU multi-tenancy: single-model, multi-model, and GPU sharing

**What it is.** Multi-tenancy is packing more than one workload onto a single GPU — either several distinct models, or several tenants' requests through one model — to raise utilization.

**How it works under the hood.** A single model that doesn't fill the GPU wastes VRAM and compute; you can co-locate a second model in the leftover memory (multi-model), or use NVIDIA MIG to hard-partition one physical GPU into isolated instances, or MPS to let processes share the SM scheduler. The trade-off is isolation vs efficiency: co-located models contend for memory bandwidth and can suffer noisy-neighbor latency, while MIG gives firm isolation but fixed partition sizes. In vLLM, `--gpu-memory-utilization` (default `0.92`) caps the fraction of VRAM one instance claims, so two instances on one GPU can each be set to `0.45` to coexist.

**Where it appears in real systems.** Multi-model is common for small models (embedders, rerankers, a small chat model) that each waste a full GPU alone. For many fine-tunes of the *same* base model, the better multi-tenancy pattern is multi-LoRA (below). MIG appears as `nvidia.com/mig-1g.5gb` resource requests in Kubernetes.

### The serving cost model: GPU-hours, tokens, and utilization

**What it is.** The cost model expresses spend as a rate (GPU $/hour) times time, normalized against useful work (tokens served), i.e. **$ per million tokens = (GPU $/hr × replica-hours) ÷ (millions of tokens generated)**.

**How it works under the hood.** GPU-hours accrue whenever a replica is running, batched or idle — the GPU bills the same whether it's 10% or 95% utilized. Throughput (tokens/sec) is bounded by how many sequences you can batch, which is bounded by KV-cache capacity, which is bounded by VRAM. So utilization is the multiplier that turns a fixed hourly cost into a low or high per-token cost. A server running at 20% batch occupancy costs ~5× per token what the same server at near-full occupancy costs, for identical hardware.

**Where it appears in real systems.** The observable levers are prometheus-style metrics: batch size / running sequences, KV-cache utilization, queue depth, and tokens/sec. You raise utilization by increasing `--max-num-seqs` up to KV-cache limits, batching aggressively, and right-sizing replicas so each is well-packed rather than mostly idle.

### On-demand vs reserved vs spot GPUs

**What it is.** These are three purchasing models for the same accelerator: on-demand (pay-as-you-go, highest $/hr, instant), reserved/committed-use (commit 1–3 years for a large discount), and spot/preemptible (spare capacity at a steep discount that the provider can reclaim with little notice).

**How it works under the hood.** Reserved pricing amortizes a discount over a commitment, so it only wins if your average utilization of the reservation is high — a reserved GPU idle half the month can cost more per useful hour than on-demand. Spot instances can be terminated mid-request, so they require checkpointing, request draining, or replay, and are unsafe for low-latency user-facing traffic without an on-demand fallback. On-demand is the flexible default that autoscaling relies on.

**Where it appears in real systems.** A common blended design: a **reserved baseline** sized to the p50 traffic floor, **on-demand autoscaling** for the daily peak, and **spot** for offline/batch jobs (nightly embedding refresh, eval runs). Cloud APIs surface this as instance purchase-type flags and node-pool taints in Kubernetes.

### LoRA / multi-adapter serving

**What it is.** LoRA serving loads one base model once and applies small low-rank adapter deltas per request, so many fine-tuned "models" share a single set of base weights on one GPU.

**How it works under the hood.** A LoRA adapter is a pair of small low-rank matrices added to specific layers; because the base weights are shared, dozens of adapters fit in the VRAM that a single full fine-tune would occupy, and the server can even batch requests targeting *different* adapters together. In vLLM you launch with `--enable-lora` and register adapters via `--lora-modules name=path`; requests select an adapter by putting its name in the `model` field. `--max-loras` (default `1`) sets how many distinct adapters may appear in one batch, `--max-lora-rank` (default `16`) bounds adapter rank, and `--max-cpu-loras` sets how many adapters are cached in host RAM for fast swap-in.

**Where it appears in real systems.** Multi-LoRA is the go-to when you serve many customer- or task-specific fine-tunes of one base model (e.g. a per-tenant SQL assistant). Instead of N GPUs for N fine-tunes, you run one GPU pool of the base model and hot-load adapters — turning a linear cost in fine-tunes into a near-flat one. vLLM also supports dynamic runtime loading via `/v1/load_lora_adapter` (guarded by `VLLM_ALLOW_RUNTIME_LORA_UPDATING`, and flagged as insecure outside trusted environments).

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| `min_replicas` (Ray Serve) / HPA `minReplicas` | Floor on running replicas; `0` enables scale-to-zero | Set `0` only if idle gaps are long AND cold-start latency (tens of s to minutes for LLMs) is acceptable; otherwise set `1` to keep a warm baseline. |
| `max_replicas` | Ceiling on scale-out | Set ~20% above your measured peak-traffic replica need so bursts don't get throttled while still capping runaway cost. |
| `target_ongoing_requests` (Ray Serve) | Concurrent requests per replica the autoscaler targets | Lower it for long generations / tight latency SLO; raise it for short requests where throughput matters more than tail latency. |
| `--gpu-memory-utilization` (vLLM, default `0.92`) | Fraction of VRAM one instance claims for weights + KV-cache | Keep high (~0.9) for a single model to maximize KV-cache/concurrency; drop to ~0.45 per instance when co-locating two models on one GPU. |
| `--max-num-seqs` (vLLM) | Max concurrent sequences (batch width) | Raise until KV-cache utilization approaches its limit under peak load; if you see preemptions/OOM, lower it. |
| `--max-num-batched-tokens` (vLLM) | Max tokens processed per scheduler iteration | Raise for throughput on long prompts; lower to protect inter-token latency for chat. |
| `--max-loras` (vLLM, default `1`) | Distinct LoRA adapters allowed in one batch | Set to the number of adapters you expect to be hot simultaneously; higher values cost VRAM and scheduling overhead. |
| `--max-lora-rank` (vLLM, default `16`) | Upper bound on adapter rank | Set to the *max* rank across your adapters (e.g. 64), never higher — over-provisioning wastes memory. |
| scale-to-zero idle timeout (platform-specific) | How long a replica stays warm with no traffic before termination | Set longer than your typical inter-request gap so you don't thrash (repeated cold starts); shorter if idle GPU cost dominates. |

### Worked Example: Requirement → Decision

**Given:** A B2B product needs to serve an 8B-parameter chat model to ~30 requests/second at peak, ~3 requests/second overnight, with a p95 first-token latency SLO of 2 seconds, under a target of roughly **$6,000/month** in GPU spend. There are also 12 customer-specific fine-tunes of the same 8B base model, each used lightly.

**Step 1 — Identify the goal.** Meet 30 QPS peak within the 2 s p95 SLO while staying near the monthly budget, and serve 12 fine-tunes without paying for 12 separate GPU fleets.

**Step 2 — Define inputs.** Traffic profile (30 QPS peak / 3 QPS trough, so ~10× diurnal swing), model size (8B fits comfortably on a single mid-tier GPU), 12 LoRA fine-tunes of one base, and a hard-ish budget.

**Step 3 — Define outputs.** A serving topology: router + GPU worker pool + autoscaler, plus the purchase mix (reserved/on-demand/spot) and the multi-tenancy pattern for the fine-tunes.

**Step 4 — Apply constraints.** The 2 s first-token SLO means a from-zero cold start (weights loading = tens of seconds) is *not* acceptable on the hot path, so a fully scale-to-zero design fails the SLO for the first user after a quiet period. The 10× swing means an always-on peak-sized fleet would sit ~90% idle overnight, blowing the budget on GPU-hours that serve nothing. The 12 lightly-used fine-tunes make 12 dedicated GPUs absurd.

**Step 5 — Select the approach.** Serve one base model with **vLLM + multi-LoRA** (`--enable-lora`, all 12 adapters registered, `--max-lora-rank` set to the fleet max) on a **reserved baseline of 1 warm replica** (`min_replicas: 1`) to cover the 3 QPS trough and kill the cold-start problem, with **on-demand autoscaling up to the peak** (`max_replicas` ~20% over the measured 30-QPS need) driven by `target_ongoing_requests` tuned to the latency SLO. Rationale vs alternatives: an always-on peak fleet is rejected because it wastes GPU-hours overnight; full scale-to-zero is rejected because it violates the first-token SLO; 12 separate model deployments are rejected because multi-LoRA collapses them onto the shared base for a fraction of the GPU-hours. Reserve only the baseline (not the peak) so the discount lands on the hours you'd pay for anyway.

---

## Implementation

```bash
# Scenario: serve one 8B base model plus 12 customer fine-tunes at low incremental
# cost. Constraint — buying a GPU per fine-tune is 12x the hardware; multi-LoRA
# shares the base weights so all adapters ride one GPU pool.
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --enable-lora \
  --max-loras 8 \
  --max-lora-rank 64 \
  --max-cpu-loras 16 \
  --gpu-memory-utilization 0.90 \
  --max-num-seqs 256 \
  --lora-modules \
    tenantA=/adapters/tenantA \
    tenantB=/adapters/tenantB
# A request routes to a fine-tune by naming it as the model:
#   {"model": "tenantA", "prompt": "...", "max_tokens": 256}
```

```yaml
# Anti-pattern: over-provisioned, always-on peak-sized GPU fleet "so we never
# queue." With a 10x diurnal swing this leaves ~90% of GPU-hours idle overnight,
# and idle GPU-hours cost exactly the same as busy ones — the budget is burned
# on utilization near zero.
deployments:
  - name: ChatModel
    num_replicas: 6          # fixed at peak size, 24/7
    ray_actor_options: {num_gpus: 1}

# Correct approach: a warm baseline + autoscaling on-demand replicas. min_replicas
# keeps 1 GPU hot (meets the first-token SLO, no from-zero cold start), and the
# autoscaler adds replicas only while traffic is high, so you pay for peak capacity
# only during peak.
deployments:
  - name: ChatModel
    max_ongoing_requests: 8
    autoscaling_config:
      target_ongoing_requests: 4   # tune to latency SLO: lower = snappier, more replicas
      min_replicas: 1              # warm baseline; set 0 only if cold start is tolerable
      max_replicas: 8             # ~20% over measured peak need
    ray_actor_options: {num_gpus: 1}
```

---

## Common Pitfalls & Misconceptions

- **Sizing replicas by raw QPS instead of KV-cache headroom** — Beginners divide "peak QPS" by "requests one GPU handled in a quick test," but that test used short outputs. Concurrency on an LLM is bounded by KV-cache VRAM, which grows with sequence length; size replicas by how many *concurrent tokens/sequences* fit (`--max-num-seqs` at KV-cache limit) under realistic output lengths, not by a QPS number from a toy benchmark.
- **Treating scale-to-zero as free savings** — It's tempting because idle cost drops to zero, but the correct mental model is that you've traded steady spend for a latency cliff: the next request pays a multi-minute cold start (image pull + weight load + maybe node provisioning). Use scale-to-zero only for tolerant/batch workloads or long idle windows, and keep `min_replicas: 1` on anything with a tight first-token SLO.
- **Assuming a bigger GPU is always cheaper per token** — People reason "one big GPU beats several small ones," but per-token cost is set by utilization, not raw size; a large GPU running one small model at 15% occupancy is *more* expensive per token than a right-sized GPU packed with batched requests. Match hardware to the model and then maximize batch occupancy.
- **Reserving capacity for peak traffic** — The reserved discount only pays off on hours you'd run anyway, so reserving your *peak* fleet size means most reserved GPU-hours serve nothing overnight. Reserve the baseline/floor, autoscale the peak on-demand, and push interruptible work to spot.
- **Picking a serving framework by familiarity** — Choosing a stack because you used it once ignores maintenance status and feature fit; TGI, for instance, is now in maintenance mode with vLLM/SGLang recommended, so defaulting to it for a new greenfield system is a weak interview answer. Justify the engine by continuous batching, paged KV-cache, adapter support, and active maintenance.

---

## Key Definitions

| Term | Definition |
|---|---|
| Continuous batching | Iteration-level scheduling where the server admits/evicts requests every decode step so finished slots are reused immediately, keeping the GPU busy across variable-length requests. |
| Paged KV-cache (PagedAttention) | Storing each sequence's attention keys/values in fixed-size non-contiguous blocks (like OS paging) to cut VRAM fragmentation and raise achievable concurrency. |
| KV-cache headroom | The remaining VRAM available for attention keys/values, which bounds how many concurrent sequences (batch width) a replica can serve. |
| Replica | One running instance of a model server (typically one or more GPUs) that the router load-balances across. |
| Scale-to-zero | Autoscaling down to zero replicas when idle to stop all GPU billing, at the cost of a cold start on the next request. |
| Cold start | The latency to bring a new replica into service: scheduling, container image pull, weight download/load into VRAM, and (on K8s) possibly node provisioning. |
| Multi-tenancy / GPU sharing | Running multiple models or tenants on one physical GPU (co-location, NVIDIA MIG partitions, or MPS) to raise utilization. |
| Multi-LoRA serving | Serving many fine-tunes as small adapters over one shared base model on the same GPU, optionally batching requests across adapters. |
| Spot / preemptible GPU | Discounted spare-capacity GPU the provider can reclaim with little notice; suitable for interruptible/batch work, not tight-SLO user traffic without fallback. |
| GPU-hours-per-million-tokens | The normalized serving cost metric: (GPU $/hr × replica-hours) ÷ (millions of tokens served), the target quantity to minimize. |

---

## Summary / Quick Recall

- Serving cost is GPU-hours ÷ useful tokens; every decision is really a utilization bet under a latency SLO.
- Purpose-built servers (vLLM; TGI now in maintenance mode) win via continuous batching + paged KV-cache — that's the reason, not the brand.
- Size replicas by KV-cache headroom (concurrent sequences at realistic output length), not by a toy-benchmark QPS.
- Warm baseline (`min_replicas ≥ 1`) protects the first-token SLO; scale-to-zero saves money only when cold starts are tolerable.
- Multi-LoRA serves many fine-tunes of one base on a single GPU pool — near-flat cost instead of one GPU per fine-tune.
- Purchase mix: reserve the baseline, autoscale the peak on-demand, run interruptible/batch on spot.

---

## Self-Check Questions

1. What two mechanisms make a dedicated inference server (like vLLM) cheaper to run than a naive per-request `model.generate()` loop, and why?

   <details><summary>Answer</summary>

   **Continuous batching** and **paged KV-cache**. Continuous batching schedules at the iteration level so a finished sequence's slot is immediately reused by a waiting request, keeping the GPU busy across variable-length requests instead of stalling until a fixed batch completes. Paged KV-cache stores attention keys/values in fixed-size non-contiguous blocks, cutting the VRAM fragmentation that otherwise caps concurrency. The tempting wrong answer — "it uses a faster model" — is wrong: the model weights are identical; the savings come entirely from higher GPU utilization per hour.

   </details>

2. You must serve a chat model with a p95 first-token latency SLO of 2 seconds, and traffic drops to near zero for several hours overnight. Should you enable scale-to-zero? Justify.

   <details><summary>Answer</summary>

   No — not on the hot path. Scale-to-zero means the first request after an idle period pays a cold start (image pull + weight load into VRAM, tens of seconds to minutes for an LLM), which blows the 2 s first-token SLO. Keep `min_replicas: 1` as a warm baseline to serve the overnight trickle and avoid the from-zero penalty; autoscale extra replicas on-demand for the daytime peak. Scale-to-zero would only be acceptable if the SLO were relaxed or the workload were batch/tolerant.

   </details>

3. **Which TWO** of the following correctly describe when multi-LoRA (multi-adapter) serving is the right architecture?
   - A. You have 12 fine-tunes of the *same* base model, each used lightly.
   - B. You have 12 completely different base architectures to serve.
   - C. You want many fine-tunes to share one GPU's base weights and be batchable together.
   - D. You need the strongest possible hardware isolation between tenants.
   - E. Each fine-tune has enormous traffic and needs its own dedicated GPU fleet.

   <details><summary>Answer</summary>

   **A and C.** Multi-LoRA shines exactly when many fine-tunes share one base model (A) and can ride one GPU pool with the base weights loaded once, even batching across adapters (C) — turning a linear cost in fine-tunes into a near-flat one. B is wrong because LoRA adapters attach to a single shared base; different architectures can't share it. D is wrong because co-locating adapters gives *less* isolation, not more (MIG is the isolation tool). E is the most tempting distractor: if a fine-tune genuinely saturates its own GPUs, you'd give it a dedicated deployment — multi-LoRA's whole value is amortizing *lightly*-used fine-tunes.

   </details>

4. Two teams each run an 8B model. Team X uses one large GPU per model at ~15% batch occupancy "for headroom"; Team Y right-sizes the GPU and runs at ~85% occupancy. Which has the lower cost per million tokens, and what is the underlying reason?

   <details><summary>Answer</summary>

   Team Y. GPU-hours bill the same whether the GPU is 15% or 85% utilized, so per-token cost is (fixed hourly rate) ÷ (tokens served that hour) — occupancy is the denominator. Team X pays roughly the same hourly rate but serves far fewer tokens, so its per-token cost is several times higher. The reason is *not* that Team X's GPU is "wasted on being large" per se; even the same GPU at 15% vs 85% occupancy differs sharply in $/token. The fix for Team X is to raise batch occupancy (increase `--max-num-seqs` toward the KV-cache limit) and/or right-size the hardware.

   </details>

5. You have predictable round-the-clock baseline traffic plus a large daily peak, and a nightly batch embedding job that can be retried. Design the GPU purchase mix and explain the trade-off in each choice.

   <details><summary>Answer</summary>

   Reserve GPUs for the **baseline** (committed-use discount pays off because those hours run 24/7 anyway — the trade is a 1–3 year commitment and lost flexibility). Use **on-demand autoscaling** for the **daily peak** (instant and flexible, highest $/hr, but you only pay for peak capacity during the peak rather than all day). Run the **nightly batch job on spot/preemptible** GPUs (cheapest, and the retry-tolerant job survives reclamation; unsafe for the user-facing tiers because a mid-request preemption would break the latency SLO). The classic mistake is reserving the *peak* size — that leaves most reserved hours idle overnight, so reserve only the floor.

   </details>

---

## Further Reading

- [vLLM Engine Arguments](https://docs.vllm.ai/en/latest/configuration/engine_args.html) — *verified 2026-07-29* — authoritative list of serving knobs including `--gpu-memory-utilization`, `--max-num-seqs`, `--max-num-batched-tokens`, `--max-loras`, and `--max-lora-rank` with their defaults.
- [vLLM LoRA Adapters](https://docs.vllm.ai/en/latest/features/lora.html) — *verified 2026-07-29* — how to serve many fine-tunes over one base model, register adapters, and dynamically load/unload them at runtime.
- [Hugging Face Text Generation Inference — Index](https://huggingface.co/docs/text-generation-inference/en/index) — *verified 2026-07-29* — TGI's feature set (continuous batching, paged attention, tensor parallelism) and the notice that it is now in maintenance mode with vLLM/SGLang recommended.
- [Ray Serve Autoscaling](https://docs.ray.io/en/latest/serve/autoscaling-guide.html) — *verified 2026-07-29* — replica autoscaling with `num_replicas="auto"`, `target_ongoing_requests`, `min_replicas` (including scale-to-zero via `0`), and `max_replicas`.
- [Kubernetes Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) — *verified 2026-07-29* — how HPA scales replica count on CPU/memory or custom/external metrics, the layer that provisions and removes GPU-backed pods.
