# LLM Inference and Serving Economics — Interview Prep

**Section:** LLM Serving & RAG Architecture → LLM Inference & Serving Economics | **Role target:** Applied AI Engineer / AI Systems Engineer (agentic AI, RAG, multi-agent production systems)

---

## Core Conceptual Questions

These test whether you understand the fundamentals.

| Question | Key points to cover | Common weak-answer trap |
|---|---|---|
| Explain the difference between prefill and decode, and why it matters for optimisation. | Prefill = one forward pass over the whole prompt, all tokens in parallel, **compute-bound**, produces the first token and sets TTFT. Decode = autoregressive one-token-per-step loop, each step attends over the full KV cache, **memory-bandwidth-bound**, sets TPOT/ITL. Different phases → different levers and different metrics. | Describing generation as one undifferentiated "the model runs and produces text," so they can't say *which* lever fixes *which* latency number. |
| Why is decode memory-bandwidth-bound rather than compute-bound, and what does that imply about buying a "faster" GPU? | Each decode step does tiny arithmetic (one new token) but must *read* the model weights + the entire KV cache from VRAM every step, so speed is set by memory bandwidth, not TFLOPs. Implication: more raw FLOPs barely move TPOT; batching (amortises weight reads), quantization (fewer bytes/read), and higher-bandwidth memory do. | "Just use a bigger/faster GPU" — equates token-generation speed with peak compute, which is only true for prefill. |
| Distinguish TTFT, TPOT/ITL, and throughput, and map each to its controlling phase and lever. | TTFT = queue wait + prefill (prompt length, load); TPOT/ITL = a single decode step (bandwidth, batch contention); throughput = system-level tokens/sec across all concurrent requests (batching). SLOs set on **percentiles**, not means. A lever that helps one often hurts another. | Reporting "latency" as a single number, the way a classic REST service is measured — misses that TTFT and TPOT move independently. |
| What is continuous (in-flight) batching and what trade-off does it make versus static batching? | Iteration-level scheduling: admit/evict requests *between* decode steps, so a finished sequence's slot is reused immediately instead of the whole batch waiting on its slowest member. Because decode re-reads the same weights regardless of batch size, packing more sequences amortises those reads → throughput rises steeply. Trades a little per-request TPOT for a large throughput / $-per-token win. | "Batching always adds latency," or conflating it with training-style static batching that holds short requests hostage to long ones. |
| Where is the cost break-even between a managed API and a self-hosted GPU fleet, and what moves it? | API cost is linear in tokens (`tokens × price`); self-host is a near-flat floor (`GPU-hours × rate + ops`) paid whether the GPU is busy or idle. Left of break-even the API wins; right of it self-host wins. Break-even moves with GPU **utilization**, model size, and the API's per-token price (a cheap API model pushes it to much higher volume). | "Self-hosting is always cheaper because there's no per-token fee" — ignores the fixed GPU floor paid on idle hardware. |
| What is the serving cost model, and why is utilization the multiplier? | `$ / million tokens = (GPU $/hr × replica-hours) ÷ (millions of tokens served)`. GPU-hours bill identically at 15% or 95% occupancy, so per-token cost is (fixed hourly rate) ÷ (tokens served that hour); occupancy is the denominator. A server at 20% batch occupancy costs ~5× per token vs the same server near full. | Sizing replicas by raw QPS from a toy short-output benchmark instead of by KV-cache headroom at realistic output lengths. |
| When is multi-LoRA (multi-adapter) serving the right architecture, and how does it save money? | One base model loaded once; many small low-rank adapter deltas share those base weights, and the server can batch requests targeting *different* adapters. Turns "one GPU per fine-tune" (linear cost) into "one GPU pool for many fine-tunes" (near-flat). Right for many lightly-used fine-tunes of the *same* base. | Suggesting LoRA for different base architectures (adapters attach to one shared base) or when a fine-tune genuinely saturates its own GPUs. |

---

## Applied / Scenario Questions

**Q1:** Your customer-support chat product is missing its **p95 TTFT ≤ 500 ms** budget under bursty afternoon load on a single 80 GB H100 serving an 8B model. Streaming is smooth once tokens start, but users complain the assistant "takes forever to begin replying." How do you diagnose and fix it?

**Strong answer framework:**
- **Isolate the failing metric first.** Smooth post-first-token streaming means TPOT/ITL is fine — the failure is **TTFT** (queue wait + prefill), not decode. Confirm with `vllm:time_to_first_token_seconds` p95 bucket vs `vllm:inter_token_latency_seconds`.
- **Find the mechanism.** Under load, requests sit queued while the GPU is saturated, and large prefill chunks compete with in-flight decodes. Check `vllm:num_requests_running`, queue depth, and `vllm:kv_cache_usage_perc` — if it nears 1.0 you're preempting (RECOMPUTE), which produces the worst tail spikes.
- **Apply the right levers.** Lower `max_num_batched_tokens` (e.g. 2048) to prioritise decodes and stop prefill from blowing the budget; cap `max_num_seqs` at the level where p95 TTFT holds in a load test (raise while the SLO holds, stop when it breaks); keep `gpu_memory_utilization` ~0.90 so the KV pool absorbs bursts instead of preempting.
- **Latency vs accuracy vs cost vs safety tradeoff:** capping concurrency to protect TTFT sacrifices some throughput (higher $/token), so it's a deliberate latency-over-cost choice for a user-facing chat product; you are *not* trading accuracy or safety here — the model and guardrails are unchanged, only the scheduler dials move. If cost then becomes unacceptable, the next move is quantization (FP8) to reclaim throughput at a small accuracy cost, not raising `max_num_seqs` back into SLO violation.

**Q2:** Finance says you must cut LLM inference spend by **40%** without hurting p95 latency or answer quality on a self-hosted 8B chat service that currently runs one large GPU per model at ~15% batch occupancy. What do you do?

**Strong answer framework:**
- **Attack utilization first — it's the denominator.** GPU-hours cost the same at 15% or 85% occupancy, so raising batch occupancy is free throughput. Increase `max_num_seqs` toward the KV-cache limit and confirm `vllm:kv_cache_usage_perc` stays below saturation so you don't trigger preemption.
- **Right-size hardware and the purchase mix.** A large GPU running one small model at 15% is more expensive per token than a right-sized GPU packed with batched requests. Reserve the p50 baseline (committed-use discount on hours you run anyway), autoscale the peak on-demand, and push nightly/offline work (eval runs, embedding refresh) to spot.
- **Kill idle-hour waste without a cold-start cliff.** Keep `min_replicas: 1` warm to protect the first-token SLO; only enable scale-to-zero on tolerant/batch tiers. Consolidate lightly-used fine-tunes onto one base via multi-LoRA instead of a GPU per fine-tune.
- **Latency vs accuracy vs cost vs safety tradeoff:** the safe 40% comes from utilization + purchase mix (no latency or quality cost). If that's not enough, FP8/INT8 quantization cuts bandwidth-bound decode latency *and* frees VRAM for bigger batches — but that's an explicit small-accuracy-for-cost trade you must validate on your own task before shipping, and you must confirm guardrails/eval scores hold on the quantized model rather than assuming parity.

---

## System Design / Architecture Questions

**Q:** Design a cost-efficient LLM serving layer for an 8B chat model at **50 QPS peak** (with a ~10× diurnal swing down to ~5 QPS overnight) under a **p95 TTFT ≤ 1 s** SLA and a target GPU budget, where you also have ~10 customer-specific fine-tunes of the same base model.

**Approach:**

1. **Clarify requirements (scale, latency budget, hallucination tolerance, data sensitivity).**
   - Confirm the SLA is on **percentile TTFT** (not mean) and whether a separate ITL/TPOT target exists.
   - Realistic prompt/output lengths (drives KV-cache size and therefore concurrency), not a toy benchmark.
   - Data residency / compliance: does raw data leave the trust boundary? If a VPC-isolated/regional managed path fails audit, that forces self-host; otherwise self-host is a *cost* decision.
   - Are the 10 fine-tunes lightly used (multi-LoRA candidate) or does any one saturate a GPU (dedicated deployment)?

2. **Propose high-level architecture.**
   ```
   clients ──► router/LB ──► queue ──► GPU worker pool (vLLM, continuous batching + paged KV-cache)
                                            │  --enable-lora, all 10 adapters registered
                                            ▼
                                        autoscaler (target_ongoing_requests, min/max replicas)
   ```
   - **Engine:** vLLM (actively maintained; continuous batching + PagedAttention), *not* TGI (maintenance mode) chosen by familiarity.
   - **Multi-tenancy:** one base model + multi-LoRA for the 10 fine-tunes — `--enable-lora`, `--max-lora-rank` set to the fleet max, requests select an adapter by name in the `model` field. Near-flat cost instead of 10 GPU pools.
   - **Autoscaling:** `min_replicas: 1` warm baseline (covers the ~5 QPS trough, kills the from-zero cold start that would blow the TTFT SLA), `max_replicas` ~20% over the measured 50-QPS need, driven by `target_ongoing_requests` tuned to the SLA.
   - **Scheduler tuning:** `max_num_batched_tokens` sized to protect TTFT, `max_num_seqs` capped at the KV-cache limit under peak, `gpu_memory_utilization` ~0.90.
   - **Purchase mix:** reserve the baseline, autoscale the peak on-demand, spot for offline/batch.

3. **Justify choices and name tradeoffs explicitly.**
   - **Cost vs latency:** an always-on peak-sized fleet meets latency trivially but wastes ~90% of overnight GPU-hours — rejected. Full scale-to-zero minimises idle cost but the cold start (weight load into VRAM, tens of seconds to minutes) violates the TTFT SLA — rejected. Warm baseline + on-demand peak is the balance.
   - **Cost vs isolation:** multi-LoRA maximises utilization but co-locates tenants (less isolation, noisy-neighbor risk); acceptable for lightly-used internal fine-tunes, but a tenant with strict isolation or huge traffic gets a dedicated deployment or MIG partition.
   - **Latency vs throughput:** bigger `max_num_seqs`/`max_num_batched_tokens` raise tokens/sec (lower $/token) but degrade per-user TTFT/ITL and push KV cache toward preemption — size to the SLA, not to max throughput.
   - **Complexity vs lock-in vs safety:** program against an OpenAI-compatible interface so backend swaps are config, not rewrites; if data sensitivity is in play, keep regulated traffic on the self-hosted path inside a private subnet and route only non-sensitive queries out.

---

## Vocabulary That Signals Expertise

Use these terms naturally — don't force them:
- **TTFT (time to first token)** — use when discussing perceived responsiveness and prefill/queue behaviour; anchor it to a p95 SLO, not a mean.
- **TPOT / ITL (inter-token latency)** — use for streaming smoothness and decode-step cost; signals you separate first-token from steady-state latency.
- **Prefill vs decode / compute-bound vs memory-bandwidth-bound** — use to explain *why* a given lever helps; the phrase "decode is memory-bandwidth-bound" instantly signals depth.
- **KV cache / PagedAttention / preemption** — use when reasoning about the real binding constraint on concurrency and the cause of tail-latency spikes.
- **Continuous (in-flight) batching** — use to explain the throughput-vs-latency trade and why a purpose-built server beats a naive `generate()` loop.
- **`gpu_memory_utilization` / `max_num_seqs` / `max_num_batched_tokens`** — cite the actual knobs when proposing a fix; shows hands-on tuning, not hand-waving.
- **Scale-to-zero / cold start / warm baseline (`min_replicas`)** — use when weighing idle cost against a first-token SLO.
- **Cost break-even / GPU-hours-per-million-tokens / utilization** — use to make the self-host-vs-API and right-sizing arguments quantitatively.
- **LoRA multi-adapter serving** — use for consolidating many fine-tunes of one base model onto a single GPU pool.
- **Reserved / on-demand / spot purchase mix** — use to show you optimise the *rate*, not just the architecture.

---

## Vocabulary That Signals Weakness

Avoid these — they signal outdated or shallow understanding:
- **"Just use a bigger/faster GPU."** — Equates token-generation speed with peak FLOPs; ignores that decode is memory-bandwidth-bound, so more compute barely moves TPOT.
- **"Self-hosting is always cheaper — no per-token fee."** — Ignores the fixed GPU floor paid on idle hardware; self-host only wins right of the utilization break-even.
- **"Batching always adds latency."** — Conflates static batching with continuous batching; continuous batching trades a *little* per-request latency for a large throughput/cost win and is the default for a reason.
- **"Latency is X ms" (one number).** — Collapses TTFT and TPOT, which live in different phases with different levers; a serious answer quotes percentiles of each.
- **"Size the fleet by QPS."** — LLM concurrency is bounded by KV-cache VRAM at realistic output lengths, not by a QPS number from a short-output toy test.
- **"A bigger GPU is cheaper per token."** — Per-token cost is set by utilization/occupancy, not raw GPU size; a big GPU at 15% occupancy is more expensive per token.
- **"We'll use TGI"** (as a default, unqualified) — TGI is in maintenance mode with vLLM/SGLang recommended; picking an engine by familiarity rather than continuous batching / paged KV-cache / active maintenance is a weak justification.
- **"Reserve GPUs for peak."** — The reserved discount only pays off on hours you'd run anyway; reserving peak leaves most reserved hours idle overnight.
- **"A managed API means my data is exposed, so we must self-host."** — Conflates "third-party API" with "non-compliant"; VPC isolation, regional endpoints, and no-training guarantees often pass audit.

---

## STAR Answer Frame

**Situation:** A customer-support chat product on a single 80 GB H100 (8B model) was breaching its p95 TTFT ≤ 500 ms SLO during afternoon traffic peaks; users reported the assistant "hung" before replying, though streaming was smooth once it started. A previous "make it faster" attempt had maxed out `max_num_seqs` and `max_num_batched_tokens` to chase throughput and made the tail latency worse.

**Task:** I owned bringing p95 TTFT back under 500 ms at peak *and* maximising concurrent chats within that budget, without buying a second GPU or degrading answer quality.

**Action:** I split the diagnosis by metric — smooth streaming meant ITL was fine, so this was a TTFT problem (queue wait + prefill), not decode. Dashboards showed `kv_cache_usage_perc` hitting ~1.0 and rising RECOMPUTE preemptions at peak: the throughput-maxed config was over-packing the KV cache, and preemptions were the tail spikes. I reverted to continuous batching + chunked prefill, set `max_num_batched_tokens=2048` to prioritise decodes over prefill chunks, capped `max_num_seqs` at the highest value where a load test held p95 TTFT under 500 ms, and set `gpu_memory_utilization=0.90` so the KV pool absorbed bursts instead of preempting. I explicitly accepted slightly lower peak throughput as the cost of protecting the latency SLO.

**Result:** p95 TTFT dropped from ~900 ms back under 500 ms at peak, preemptions went to near zero, and by tuning `max_num_seqs` to the SLO edge (rather than blindly high) we sustained ~30% more concurrent chats than the "throughput-maxed" config actually delivered once its preemption stalls were counted — all on the same single GPU, no quality change.

---

## Red Flags Interviewers Watch For

- **Reporting a single "latency" number** instead of separating p95 TTFT (prefill/queue) from p95 ITL/TPOT (decode) — shows they haven't internalised the two-phase model.
- **Reaching for a bigger/faster GPU to fix slow token generation** — misreads decode as compute-bound; the honest lever is bandwidth/batching/quantization.
- **Treating batch size as a value to maximise** rather than a dial traded against per-request latency and KV-cache saturation.
- **Sizing replicas by raw QPS** from a short-output benchmark rather than by KV-cache headroom at realistic output lengths.
- **Claiming self-hosting is always cheaper** with no break-even/utilization reasoning, or **reserving peak capacity** instead of the baseline.
- **Ignoring cold starts / recommending scale-to-zero** on a tight first-token SLO without a warm baseline.
- **Proposing a GPU per fine-tune** when the fine-tunes share one base and multi-LoRA applies.
- **Assuming an open-weight model is a drop-in for a frontier API model** with no task benchmarking, or assuming a quantized model keeps quality without validating it.
- **Conflating "managed API" with "non-compliant"** and jumping to self-host before checking a VPC-isolated/regional managed path.
- **Picking a serving framework by brand familiarity** (e.g. defaulting to TGI) rather than justifying it by continuous batching, paged KV-cache, adapter support, and active maintenance.
- **Treating rate limits (429s) as a retry problem** rather than a capacity/architecture problem needing headroom, batching, or a self-hosted fallback.
