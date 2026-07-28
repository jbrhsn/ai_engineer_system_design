# LLM Inference Fundamentals — Latency and Throughput

**Section:** LLM Serving & RAG Architecture → LLM Inference & Serving Economics | **Est. time:** 2.5 hrs | **Interview relevance:** High — nearly every agentic/RAG system design interview eventually asks "how do you hit your latency SLO while keeping GPU cost sane?", and that question lives entirely in prefill/decode, batching, and KV-cache mechanics.

---

## TL;DR

LLM inference runs in two mechanically different phases: a compute-bound **prefill** that processes the whole prompt in parallel to produce the first token, and a memory-bandwidth-bound **decode** loop that emits one token at a time by streaming the growing KV cache through the GPU. This split is why you track three separate numbers — **TTFT** (time to first token, dominated by prefill), **TPOT/ITL** (inter-token latency, dominated by decode), and **throughput** (tokens/sec across all concurrent requests) — and why continuous batching lets you trade a little per-request latency for a large throughput (and cost) win. GPU memory, not FLOPs, is usually the binding constraint, because the KV cache grows linearly with batch size × sequence length and must sit in VRAM alongside the model weights. **The one thing to remember: prefill is compute-bound and parallel, decode is memory-bandwidth-bound and serial — you optimise and measure each phase with different levers and different metrics.**

---

## ELI5 — Explain It Like I'm 5

Imagine a chef reading a whole recipe card at once (fast — eyes scan it in one glance) and then cooking the dish one spoonful at a time, where every new spoonful requires re-reading every note they've scribbled so far. Reading the recipe is *prefill*: it happens in one big parallel burst and produces the first spoonful. Cooking spoonful-by-spoonful is *decode*: it's serial, and the pile of scribbled notes (the KV cache) grows with every spoonful, so each new one takes a fixed re-read of the whole pile. The common misconception is that a "faster GPU" (more raw cooking horsepower) fixes slow token generation — but during decode the chef spends almost all their time re-reading notes, not cooking, so what actually helps is a wider desk to read from faster (memory bandwidth) or cooking several diners' dishes on the same stove at once (batching), not a hotter burner.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain why prefill is compute-bound and decode is memory-bandwidth-bound, and what that implies for optimisation.
- [ ] Distinguish TTFT, TPOT/ITL, and throughput, and map each to the phase and system lever that controls it.
- [ ] Diagnose why adding GPUs or a faster chip fails to improve decode latency, and identify the real binding constraint (memory / bandwidth).
- [ ] Compare static vs continuous (in-flight) batching and articulate the latency-throughput trade-off each produces.
- [ ] Select engine parameters (`gpu_memory_utilization`, `max_num_seqs`, `max_num_batched_tokens`) to meet a stated p95 TTFT budget.

---

## Visual Overview

### Prefill → Decode Token-Generation Loop (with KV cache)

```
PROMPT: "Summarise this contract"
        │
        ▼
┌──────────────────────────┐
│  PREFILL (1 forward pass) │  all prompt tokens processed in PARALLEL
│  compute-bound            │  ── writes KV for every prompt token ──►┐
└──────────────────────────┘                                          │
        │ emits token #1 (TTFT stops here)                            ▼
        ▼                                                    ┌─────────────────┐
┌──────────────────────────┐   read whole KV cache          │    KV CACHE     │
│  DECODE step  (1 token)   │◄───────────────────────────────│ grows +1 token  │
│  memory-bandwidth-bound   │────────────────────────────────│ every step      │
└──────────────────────────┘   append new K,V                └─────────────────┘
        │ emits token #2 … N   (gap between tokens = TPOT/ITL)
        └──────────► loop until EOS or max_tokens
```

### TTFT vs TPOT on a Request Timeline

```
request
arrives    prefill done          steady-state decode
  │           │                        │
  ▼           ▼                        ▼
  ├───queue───┼────prefill────┬──tok──┬──tok──┬──tok──►  ... EOS
  │           │               │       │       │
  │◄── TTFT ──────────────────►       │◄─TPOT─►│
  │                                   (gap between successive tokens = ITL)
  │◄──────────────── end-to-end latency ──────────────────────────────►
```

### Latency vs Throughput as Batch Size Grows

```
tokens/sec (throughput)                 per-request TPOT (latency)
  ▲                                        ▲
  │           ___________ saturates        │                    ___/  climbs
  │        __/  (bandwidth/mem cap)        │              _____/
  │      _/                                │        _____/
  │    _/                                  │  ___/
  │  _/                                    │_/
  └────────────────────────► batch         └────────────────────────► batch
   small        large                       small        large
   ↑ under-utilised GPU                     ↑ best latency, worst $/token
   ↑ big batch = best $/token, worse latency per user
```

---

## Key Concepts

### Prefill vs Decode Phases

**What it is.** Autoregressive generation has two phases: *prefill* ingests the entire input prompt and produces the first output token; *decode* then generates each subsequent token one at a time, feeding each output back in as the next input.

**How it works under the hood.** In prefill, all N prompt tokens are pushed through the transformer in a single forward pass, so the matrix multiplies are large and dense — the GPU's compute units (tensor cores) are the bottleneck, making prefill **compute-bound**. In decode, each step processes exactly one new token but must attend over all previous tokens, so the arithmetic is tiny while the volume of weights + KV cache that must be *read* from VRAM per step is large — the GPU spends most of its time waiting on memory, making decode **memory-bandwidth-bound**. This is why decode barely benefits from more FLOPs but benefits enormously from batching (which amortises the weight reads across many concurrent requests).

**Where it appears in real systems.** vLLM exposes these as separate histograms: `vllm:request_prefill_time_seconds` and `vllm:request_decode_time_seconds`. Its **chunked prefill** scheduler deliberately mixes compute-bound prefill chunks and memory-bound decode steps in the same batch to keep the GPU busy on both dimensions at once.

### KV Cache

**What it is.** The KV cache stores the key and value tensors computed for every token already seen, so that each new decode step can attend to the full context without recomputing the past.

**How it works under the hood.** Attention needs the K and V projections of every prior token. Without a cache, generating token *t* would recompute K/V for all *t−1* prior tokens every step — quadratic waste. The cache trades compute for memory: it holds those K/V tensors in VRAM and grows by one token's worth of K/V per decode step. Its size scales as roughly `2 × num_layers × num_kv_heads × head_dim × seq_len × batch_size × dtype_bytes`, so it grows linearly with both context length and concurrency. vLLM's **PagedAttention** stores this cache in fixed-size non-contiguous *blocks* (like OS virtual-memory paging) so fragmentation doesn't strand memory.

**Where it appears in real systems.** vLLM reports `vllm:kv_cache_usage_perc` (0–1). When it approaches 1.0 the scheduler must **preempt** requests — evicting and later recomputing them — which shows up as latency spikes and the `total_cumulative_preemption_cnt` warning. Block size is visible in `vllm:cache_config_info{block_size="16",...}`.

### TTFT, TPOT/ITL, and Throughput Metrics

**What it is.** Three orthogonal performance numbers: **TTFT** (time to first token) — how long until the user sees *anything*; **TPOT / ITL** (time per output token / inter-token latency) — the gap between successive streamed tokens; **throughput** — total tokens/sec the server produces across all concurrent requests.

**How it works under the hood.** TTFT is dominated by queue wait + prefill time, so it scales with prompt length and how loaded the server is. TPOT is dominated by a single decode step's memory-bandwidth cost, so it's roughly constant per token but degrades as the batch grows and contends for bandwidth. Throughput is a *system* property: bigger batches raise aggregate tokens/sec (better $/token) even while each individual user's TPOT gets slightly worse — the two metrics move in opposite directions, which is the central serving trade-off.

**Where it appears in real systems.** vLLM emits `vllm:time_to_first_token_seconds` (histogram — take the p95 bucket), `vllm:inter_token_latency_seconds` (TPOT), `vllm:e2e_request_latency_seconds`, and counters `vllm:prompt_tokens_total` / `vllm:generation_tokens_total` from which you derive tokens/sec. SLOs are almost always set on the *percentiles* of TTFT and ITL, not the mean.

### Continuous (In-Flight) Batching

**What it is.** A scheduling strategy where the engine forms a new batch every iteration, admitting freshly arrived requests and evicting finished ones *between token steps*, rather than freezing a batch for the whole request as static batching does.

**How it works under the hood.** In **static batching** the whole batch waits for its slowest/longest member to finish before any slot frees up, so short requests are held hostage by long ones and the GPU idles on padding. **Continuous batching** re-evaluates the running set each decode step: a request that hits EOS leaves immediately and a queued request takes its slot on the very next step. Because decode is memory-bandwidth-bound and reads the *same* model weights regardless of batch size, packing more sequences into a step amortises those weight reads — throughput rises steeply for little extra per-step time until bandwidth saturates.

**Where it appears in real systems.** This is vLLM's default scheduler. `max_num_seqs` caps how many sequences run concurrently; `max_num_batched_tokens` caps total tokens per iteration. With **chunked prefill** (default in V1), the scheduler prioritises pending decodes then fills the remaining token budget with prefill chunks, balancing compute-bound and memory-bound work in one batch.

### Quantization's Effect on Latency

**What it is.** Storing weights (and optionally activations or the KV cache) in lower precision — FP8, INT8, INT4 — instead of FP16/BF16, to shrink memory footprint and the bytes moved per step.

**How it works under the hood.** Because decode is bandwidth-bound, the dominant cost is *reading weights and KV from VRAM* each step. Halving the bytes per weight (FP16→FP8) roughly halves that read volume, so decode latency drops and more of the batch fits in memory (higher achievable batch → higher throughput). It also frees VRAM that would have held model weights, leaving more room for KV cache and thus larger effective batch sizes. The trade-off is a small accuracy loss and hardware-dependent kernel support (e.g. FP8 needs Ada/Hopper).

**Where it appears in real systems.** vLLM's `quantization=` argument / `--quantization` flag (AWQ, GPTQ, FP8, INT8 via LLM Compressor). A separate **quantized KV cache** (`kv_cache_dtype`) shrinks the cache itself, directly raising the sequence-length × batch ceiling.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| `gpu_memory_utilization` | Fraction of VRAM vLLM pre-allocates (weights + KV cache pool). Default 0.9. | If you see frequent preemptions and no OOM headroom is needed elsewhere, raise toward 0.95 to grow the KV pool; lower to ~0.85 if co-tenant processes need VRAM or you hit OOM at load. |
| `max_num_seqs` | Max sequences running concurrently in a batch. | Lower it to protect TPOT/ITL under an SLO; raise it to push throughput when latency has slack and KV cache is available. |
| `max_num_batched_tokens` | Max tokens processed per engine iteration (prefill chunks + decode). | Set smaller (e.g. 2048) for better ITL (fewer prefills slowing decodes); set >8192 for max throughput, especially small models on big GPUs. |
| `max_model_len` / `max_tokens` | Max context (prompt+generation) length / per-request output cap. | Cap `max_model_len` to the smallest length your product needs — it directly bounds worst-case KV cache size and therefore concurrency. |
| `tensor_parallel_size` | Shards weights across N GPUs. | Increase when the model won't fit on one GPU, or to free per-GPU VRAM for more KV cache; beware added cross-GPU sync latency. |
| `quantization` / `kv_cache_dtype` | Weight / KV-cache precision. | Use FP8/INT8 weights when bandwidth-bound decode latency or VRAM is the constraint and a small accuracy hit is acceptable; quantize KV cache to extend context/concurrency. |

---

### Worked Example: Requirement → Decision

**Given:** You're serving a customer-support chat product on a single 80 GB H100 with an 8B model. Product requirement: **p95 TTFT ≤ 500 ms** for prompts up to ~1k tokens, responses stream at a comfortable reading pace, and you want to serve as many concurrent chats as possible without breaking that TTFT budget.

- **Step 1 — Identify the goal.** Keep p95 TTFT under 500 ms *while at load*, and maximise concurrent sessions (throughput) subject to that constraint. TTFT is the hard SLO; throughput is the thing to maximise second.
- **Step 2 — Define inputs.** Prompt length ~1k tokens (prefill cost), request arrival rate (bursty), single H100 80 GB, FP16 8B weights (~16 GB), leaving ~50–60 GB usable for KV cache after `gpu_memory_utilization=0.9`.
- **Step 3 — Define outputs.** The metrics that prove success: `vllm:time_to_first_token_seconds` p95 bucket ≤ 0.5, plus `vllm:num_requests_running` / `vllm:kv_cache_usage_perc` staying below saturation, and a healthy tokens/sec from `vllm:generation_tokens_total`.
- **Step 4 — Apply constraints.** TTFT degrades when (a) prompts wait in the queue because the GPU is busy, or (b) prefill chunks compete with decodes. Preemptions (KV cache full) cause the worst tail spikes. So concurrency must be capped *before* KV cache saturates.
- **Step 5 — Select the approach.** Keep continuous batching + chunked prefill (default). Set `max_num_batched_tokens=2048` to prioritise decodes and keep prefill from blowing the TTFT budget, cap `max_num_seqs` at the level where p95 TTFT stays under 500 ms in a load test (raise it while the SLO holds, stop when it breaks), and set `gpu_memory_utilization=0.9` to grow the KV pool so you rarely preempt. Rationale vs alternatives: a *bigger* `max_num_batched_tokens` would raise throughput but push prefill TTFT past 500 ms; **static batching** would blow the tail latency entirely by holding short chats behind long ones; buying a second GPU adds cost without being necessary until the single-GPU KV pool is genuinely saturated.

---

## Implementation

```python
# Scenario: A chat product must hold p95 TTFT under budget under bursty load.
# We prioritise decodes (smaller token budget) and cap concurrency so the KV
# cache never saturates into preemption, which is what wrecks tail latency.
from vllm import LLM, SamplingParams

llm = LLM(
    model="meta-llama/Llama-3.1-8B-Instruct",
    gpu_memory_utilization=0.90,      # grow the KV-cache pool to avoid preemption
    max_num_seqs=64,                   # cap concurrency to protect p95 TTFT
    max_num_batched_tokens=2048,       # smaller budget -> better ITL/TTFT balance
    max_model_len=4096,                # bounds worst-case KV cache size
)

# Stream so the user sees the first token as soon as prefill finishes (TTFT).
params = SamplingParams(temperature=0.7, max_tokens=512)
for output in llm.generate(["Summarise this support ticket…"], params):
    print(output.outputs[0].text)
```

```bash
# Anti-pattern: chasing throughput by maxing out concurrency and token budget
# on a latency-sensitive chat product. This inflates the KV cache past capacity,
# triggers RECOMPUTE preemptions, and blows the p95 TTFT/ITL SLO — throughput
# numbers look great in isolation while real users see stalls.
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --max-num-seqs 512 \
  --max-num-batched-tokens 65536 \
  --gpu-memory-utilization 0.98        # so tight that co-tenant/CUDA-graph overhead OOMs

# Correct approach: size concurrency and token budget to the latency SLO, and
# leave memory headroom so the KV pool absorbs bursts instead of preempting.
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --max-num-seqs 64 \
  --max-num-batched-tokens 2048 \
  --gpu-memory-utilization 0.90
# What breaks in the anti-pattern: decode is memory-bandwidth-bound, so a huge
# batch saturates bandwidth (ITL climbs) AND fills the KV cache (preemptions +
# recompute), so per-user latency collapses even though aggregate tokens/sec is high.
```

---

## Common Pitfalls & Misconceptions

- **"A faster/bigger GPU will speed up token generation."** — Beginners equate a model's speed with raw FLOPs because prefill *is* compute-bound and feels fast. But decode is memory-bandwidth-bound: the correct mental model is that steady-state token latency is set by how fast weights + KV cache stream out of VRAM, so bandwidth, batching, and quantization move TPOT, not peak TFLOPs.
- **"Optimise for one latency number."** — People report "latency" as a single figure because that's how classic web services are measured. In LLM serving TTFT and TPOT are governed by different phases and different levers; the correct model is to set and monitor *separate* SLOs — TTFT (prefill/queue) and ITL (decode) — because a change that helps one often hurts the other.
- **"Bigger batches are strictly better because throughput goes up."** — The intuition comes from throughput-oriented training, where you always want larger batches. In online serving, larger batches raise aggregate tokens/sec but degrade each user's TPOT and grow the KV cache toward preemption; the correct model is that batch size is a *dial* trading per-request latency against cost-per-token, tuned to the SLO — not a value to maximise.
- **"GPU memory only matters for fitting the model weights."** — Newcomers size the GPU to the weight count and stop there. In reality the KV cache grows with batch × sequence length and often dominates; the correct model is that KV cache is usually the binding constraint on concurrency, so context length and batch size are memory-budget decisions, not just latency ones.

---

## Key Definitions

| Term | Definition |
|---|---|
| Prefill | The initial forward pass that processes the entire prompt in parallel and produces the first output token; compute-bound. |
| Decode | The autoregressive loop generating one token per step, each attending over the full KV cache; memory-bandwidth-bound. |
| KV cache | The stored key/value tensors for all prior tokens, kept in VRAM so each decode step avoids recomputing the past; grows linearly with batch × sequence length. |
| TTFT (Time To First Token) | Elapsed time from request arrival to the first generated token; dominated by queue wait + prefill. |
| TPOT / ITL | Time Per Output Token / Inter-Token Latency — the gap between successive streamed tokens; dominated by a single decode step. |
| Throughput | Total output tokens per second produced by the server across all concurrent requests; a system-level property driven by batching. |
| Continuous (in-flight) batching | Scheduling that admits/evicts requests between token steps rather than freezing a batch per request, amortising weight reads for high throughput. |
| Memory-bandwidth-bound | A workload whose speed is limited by how fast data moves to/from VRAM rather than by compute; characterises LLM decode. |
| Preemption | Evicting a running request to free KV cache for others; the freed request is recomputed later, causing latency spikes. |

---

## Summary / Quick Recall

- Prefill = compute-bound, parallel, sets **TTFT**; decode = memory-bandwidth-bound, serial, sets **TPOT/ITL**.
- The KV cache grows linearly with batch × sequence length and usually is the **binding constraint** on concurrency, not FLOPs.
- Track **three** metrics — TTFT, TPOT/ITL, throughput — on percentiles; a lever that helps one often hurts another.
- **Continuous batching** trades a little per-request latency for a large throughput / cost-per-token win; static batching wrecks tail latency.
- Because decode is bandwidth-bound, **quantization** (FP8/INT8 weights or KV cache) cuts decode latency and raises the batch/context ceiling.
- Tune `max_num_seqs` / `max_num_batched_tokens` to the SLO, keep `gpu_memory_utilization` high enough to avoid **preemption** spikes.

---

## Self-Check Questions

1. Which inference phase is memory-bandwidth-bound, and which single latency metric does it primarily determine?

   <details><summary>Answer</summary>

   **Decode** is memory-bandwidth-bound and primarily determines **TPOT / ITL** (inter-token latency). During decode each step does tiny arithmetic but must read the model weights and the whole KV cache from VRAM, so speed is set by memory bandwidth. The tempting wrong answer is "prefill / TTFT" — prefill *is* the phase tied to TTFT, but prefill is *compute*-bound (large parallel matmuls), not bandwidth-bound, so it doesn't fit the "memory-bandwidth-bound" half of the question.

   </details>

2. A chat product streams tokens fine once they start, but users complain the assistant "takes forever to begin replying" when the service is busy. Which metric is failing and what is the most likely cause?

   <details><summary>Answer</summary>

   **TTFT** is failing (first-token latency), not TPOT — the smooth streaming afterward means decode is fine. Under load the most likely cause is queue wait plus prefill contention: requests sit in the queue while the GPU is saturated, and large prefill chunks compete with decodes. Reducing `max_num_batched_tokens` (prioritise decodes) and/or capping `max_num_seqs` addresses it. It is *not* a TPOT/bandwidth problem — that would show up as choppy, slow streaming *after* the first token.

   </details>

3. You switch an 8B FP16 model to FP8 weights on the same GPU. Give the two distinct effects on serving and why each occurs.

   <details><summary>Answer</summary>

   (1) **Lower decode latency (TPOT):** decode is bandwidth-bound, so halving bytes-per-weight roughly halves the volume read from VRAM each step. (2) **Higher achievable throughput / larger batch:** FP8 weights free VRAM that had held the model, enlarging the KV-cache pool so more sequences fit concurrently. The tempting wrong answer is "prefill gets much faster" — prefill is compute-bound, so it benefits far less from the reduced memory traffic; the win is concentrated in decode and in memory headroom, at the cost of a small accuracy loss.

   </details>

4. **Which TWO** of the following are direct consequences of choosing a very large batch size (high `max_num_seqs`) on a single GPU for an online chat workload?
   - A. Aggregate throughput (tokens/sec) increases until memory bandwidth saturates.
   - B. Per-request TTFT and TPOT improve for every user.
   - C. KV-cache usage rises, increasing the risk of preemption and latency spikes.
   - D. Prefill becomes memory-bandwidth-bound instead of compute-bound.
   - E. The model's weights no longer need to reside in VRAM.

   <details><summary>Answer</summary>

   **A and C.** A: larger batches amortise the shared weight reads across more sequences, so aggregate tokens/sec rises until bandwidth saturates. C: more concurrent sequences means more KV cache, pushing `vllm:kv_cache_usage_perc` toward 1.0 and risking RECOMPUTE preemption and tail spikes. B is wrong — larger batches *worsen* per-user TPOT (more contention) and can worsen TTFT via queueing; that's the core latency-throughput trade-off. D is wrong — batch size doesn't change prefill's compute-bound nature. E is wrong — weights must stay in VRAM regardless of batch size.

   </details>

5. You're told "just add a second identical GPU with tensor parallelism and our per-user token latency (ITL) will drop." Under what condition is this claim weak, and what is the more reliable reason to add the GPU?

   <details><summary>Answer</summary>

   The claim is weak when you are **not** memory/bandwidth-starved on one GPU: tensor parallelism shards weights and adds cross-GPU synchronisation each step, so ITL can even get *worse* for a model that already fits comfortably. The reliable reasons to add the GPU are (a) the model doesn't fit on one GPU, or (b) you need to free per-GPU VRAM for a larger KV-cache pool to raise concurrency/throughput or avoid preemption. So the honest framing is that TP is primarily a *capacity/memory* lever, not a per-token *latency* lever; expecting ITL to drop from sharding alone misreads decode as compute-bound.

   </details>

---

## Further Reading

- [Optimization and Tuning — vLLM](https://docs.vllm.ai/en/latest/configuration/optimization/) — *verified 2026-07-29* — Chunked prefill, preemption, and the `max_num_batched_tokens` / `max_num_seqs` / `gpu_memory_utilization` tuning levers behind the latency-throughput trade-off.
- [Paged Attention — vLLM Design](https://docs.vllm.ai/en/latest/design/paged_attention/) — *verified 2026-07-29* — How the KV cache is stored in fixed-size blocks and read during the single-query decode attention kernel.
- [Metrics — vLLM Design](https://docs.vllm.ai/en/latest/design/metrics/) — *verified 2026-07-29* — Definitions and interval calculations for `time_to_first_token_seconds` (TTFT), `inter_token_latency_seconds` (TPOT), prefill/decode time, and `kv_cache_usage_perc`.
- [Quantization — vLLM](https://docs.vllm.ai/en/latest/features/quantization/) — *verified 2026-07-29* — Supported FP8/INT8/INT4 formats and hardware compatibility for trading precision for memory footprint and decode bandwidth.
