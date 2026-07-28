# Local/Self-Hosted vs Hosted (API) LLM Deployment Decisions

**Section:** LLM Serving & RAG Architecture → LLM Inference & Serving Economics | **Est. time:** 2 hrs | **Interview relevance:** High — "where does the model run and who pays for it?" is the first architecture fork in almost every agentic/RAG system design, and the one interviewers push hardest on trade-offs.

---

## TL;DR

Choosing between a managed API (OpenAI, Anthropic, Bedrock) and self-hosted open weights (Llama/Mistral on vLLM) is a trade-off across five axes: data residency/compliance, cost at scale, latency and control, vendor lock-in and rate limits, and the ops burden of running GPUs. Managed APIs win on time-to-market, frontier model quality, and zero GPU ops; self-hosting wins on data control, per-token cost at sustained high volume, and freedom from rate limits and vendor roadmaps. The mature answer is rarely all-or-nothing — hybrid routing sends easy queries to a cheap local model and hard ones to a hosted frontier model. **The one thing to remember: self-hosting only beats an API on cost once your GPUs stay busy enough to cross the break-even utilization point — an idle A100 is far more expensive per token than any API.**

---

## ELI5 — Explain It Like I'm 5

Deciding where your model runs is like deciding how to get to work every day. Taking a taxi (a hosted API) means you pay per trip, you never fix an engine, and you can leave this instant — but if you commute twice a day, every day, those fares add up and you're at the mercy of surge pricing and whether a cab is even free. Buying your own car (self-hosting on your own GPUs) has a big upfront cost and you become the mechanic, insurer, and driver — but once you're driving it enough, each trip gets cheap and nobody can raise your fare or refuse you a ride. The mistake people make is assuming the car is always cheaper: if it sits in the garage 90% of the day, you're paying for a garage, insurance, and depreciation to make two cheap trips — which is worse than a taxi. The smartest commuters do both: drive the car for the daily routine and grab a taxi for the rare airport run at 3 a.m.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Compare managed-API and self-hosted deployment on the five decision axes (residency, cost, latency/control, lock-in, ops burden) and defend a choice under given constraints.
- [ ] Estimate the cost break-even point between an API and a self-hosted GPU fleet using utilization and token volume.
- [ ] Design a hybrid routing layer that sends easy queries to a local model and hard queries to a hosted frontier model.
- [ ] Diagnose when a data-residency or compliance requirement forces self-hosting or a VPC-isolated managed option.
- [ ] Explain how rate limits, vendor lock-in, and fine-tuning ownership shift the decision for a production system.

---

## Visual Overview

### Deployment Decision Tree

```
Does raw data leave your trust boundary legally?
├── No (hard residency / regulated data) ──► Can a VPC-isolated managed
│                                             option (Bedrock + PrivateLink)
│                                             satisfy the auditor?
│                                             ├── Yes ──► Managed API in VPC
│                                             └── No  ──► Self-host open weights
│
└── Yes (data can go to a 3rd-party API)
        │
        └──► Is sustained volume high AND GPUs stay busy?
             ├── Yes ──► Self-host on vLLM (cost + control win)
             └── No  ──► Is frontier model quality required?
                         ├── Yes ──► Managed frontier API
                         └── No  ──► Managed API (small model) or hybrid
```

### Cost Break-Even Crossover

```
$/month
  │                                    ╱ Self-host (fixed GPU + ops)
  │                                  ╱
  │                                ╱
  │                              ╱ ──────────────  ← flat floor:
  │                            ╱                     you pay for the
  │                          ╱                       GPU whether idle
  │                        ╱                         or busy
  │      API (pay-per-token, linear)
  │    ╱╱
  │  ╱╱
  │╱╱________________________________________
  └──────────────┬──────────────────────► token volume / QPS
            break-even
   left of line: API cheaper   right of line: self-host cheaper
```

### Hybrid Routing Flow

```
Incoming query
      │
      ▼
 ┌─────────────┐   easy / high-confidence
 │  Router /   │──────────────────────────► Local small model (vLLM)
 │  classifier │                                    │
 └─────────────┘                                    ▼
      │  hard / low-confidence / escalation     Response
      ▼
 Hosted frontier API (Opus/GPT) ──► Response
```

---

## Key Concepts

### Managed API (Hosted) Deployment

**What it is.** A provider (OpenAI, Anthropic) or cloud reseller (Amazon Bedrock, Azure Foundry, Google Vertex) runs the model on their hardware and exposes it over an HTTP endpoint; you pay per input/output token and never touch a GPU.

**How it works under the hood.** You send a request to an endpoint (`client.messages.create(...)` or `client.chat.completions.create(...)`); the provider queues it against a shared, continuously-batched serving fleet, meters your usage, and bills per million tokens (MTok). Capacity is governed by rate limits (RPM/TPM) that graduate as your spend rises through usage tiers. On Bedrock, the model runs in a provider-owned "Model Deployment Account" inside your Region — the model provider has no access to your prompts or completions, and traffic can stay inside your VPC via PrivateLink.

**Where it appears in real systems.** The Anthropic Messages API, the OpenAI Chat Completions/Responses API, and Bedrock's `Converse`/`InvokeModel` operations. Rate limits surface as `x-ratelimit-remaining-tokens` response headers and `429` errors; billing surfaces as per-MTok line items (e.g. Claude Opus 4.8 at $5/MTok input, $25/MTok output).

### Self-Hosted (Open-Weights) Deployment

**What it is.** You download open-weight models (Llama, Mistral, Qwen, DeepSeek, GPT-OSS) and serve them yourself on GPUs you rent or own, typically with an inference engine like vLLM.

**How it works under the hood.** vLLM loads the weights onto GPU memory and serves requests through an OpenAI-compatible endpoint, using PagedAttention for KV-cache memory management and continuous batching to keep the GPU saturated. Cost is fixed per GPU-hour regardless of traffic, so economics depend entirely on utilization — a busy GPU amortizes its cost across many tokens; an idle one wastes it. You own scaling, quantization (FP8/INT4/AWQ/GPTQ), autoscaling, failover, and upgrades.

**Where it appears in real systems.** A `vllm serve <model>` process behind a load balancer on GPU nodes (A100/H100 or cloud equivalents), exposing an OpenAI-compatible API so client code can point at your endpoint with a one-line base-URL change. vLLM supports 200+ HuggingFace architectures and multi-LoRA serving for per-tenant fine-tunes.

### Cost Break-Even (QPS / Token Volume)

**What it is.** The traffic level at which the fixed monthly cost of a self-hosted GPU fleet equals the pay-per-token cost of an equivalent managed API.

**How it works under the hood.** API cost is linear in tokens: `tokens × price_per_token`. Self-host cost is a near-flat floor: `GPU_hours × GPU_rate + ops_overhead`, largely independent of how many tokens you actually push through. Below break-even, the flat GPU floor is being paid for mostly-idle hardware and the API is cheaper; above it, the fixed cost is amortized across enough tokens that self-hosting wins. The break-even point moves with GPU utilization, model size, and the API's per-token price — a smaller/cheaper API model pushes break-even to much higher volume.

**Where it appears in real systems.** A capacity-planning spreadsheet comparing, e.g., a reserved H100 (~fixed $/month) against Claude Haiku 4.5 ($1/MTok input, $5/MTok output) at your forecast token volume; the crossover QPS is the decision boundary.

### Compliance, Data Residency & Privacy

**What it is.** Legal and contractual constraints on where data may be processed/stored and who may access it (GDPR, HIPAA, data-residency laws, customer contracts).

**How it works under the hood.** A managed API means prompts leave your infrastructure and transit a third party — acceptable only if that path is contractually and technically compliant. Providers mitigate this with regional endpoints (guaranteed geographic routing at a pricing premium — e.g. Anthropic's `inference_geo: "us"` 1.1x multiplier, Bedrock regional endpoints), VPC isolation (Bedrock + PrivateLink keeps traffic off the public internet), no-training-on-your-data guarantees, and encryption. When even a compliant managed path is disallowed, self-hosting inside your own trust boundary is the only option.

**Where it appears in real systems.** Bedrock's VPC/PrivateLink configuration and per-Region Model Deployment Accounts; Anthropic's `inference_geo` request field and data-residency endpoints; a self-hosted vLLM cluster inside a private subnet with no egress for regulated data.

### Hybrid Routing (Local + Hosted)

**What it is.** An architecture that keeps both a cheap local model and a hosted frontier model, and routes each query to the cheaper one that can handle it.

**How it works under the hood.** A lightweight router (a classifier, a confidence threshold on the local model's output, or a rules layer on query type/length) inspects the incoming query. Easy, high-confidence, or non-sensitive queries go to the local model; hard, low-confidence, or escalated queries go to the hosted frontier API. This captures most of the volume at local cost while paying frontier prices only for the queries that need it, and can also route sensitive data locally while sending only sanitized/non-sensitive queries out.

**Where it appears in real systems.** A router node in a LangGraph graph or a FastAPI middleware that selects a backend; both backends expose OpenAI-compatible APIs (vLLM locally, the provider remotely) so the client interface is identical and only the base URL/model name changes.

### Key Parameters / Configuration Knobs

<!-- This topic is decision-driver-oriented; the "knobs" are the decision drivers themselves plus concrete config fields, per AGENTS.md Rule 3. -->

| Parameter | What it controls | Decision rule |
|---|---|---|
| Data sensitivity / residency requirement | Whether raw data may leave your trust boundary | If a compliant managed path (VPC/regional endpoint) fails audit, self-host; otherwise a managed API is on the table. |
| Sustained token volume / QPS | Which side of the cost break-even you sit on | Self-host only if forecast volume keeps GPUs busy past break-even; below it, use the API. |
| GPU utilization target | How well fixed GPU cost is amortized | Self-host only if you can keep utilization high (e.g. >50–70%); bursty/low-utilization traffic favors pay-per-token APIs. |
| Model quality gap tolerance | Whether an open-weight model is "good enough" | If the task needs frontier reasoning quality that open weights can't match, use a hosted frontier model or route hard queries to it. |
| Rate-limit headroom (RPM/TPM tier) | Managed-API throughput ceiling | If your peak QPS exceeds achievable tier limits (or spend-based tier graduation is too slow), self-host or negotiate custom limits. |
| Ops maturity (GPU/on-call) | Ability to run inference infra reliably | Self-host only with a team that can own autoscaling, failover, quantization, and upgrades; otherwise the API's zero-ops wins. |
| Fine-tuning ownership needs | Who owns the customized weights | If you must own/export tuned weights or serve many per-tenant adapters, self-host with LoRA; if provider-managed fine-tuning suffices, use the API. |
| Endpoint routing geography | Where inference physically runs | Use regional/data-residency endpoints (accepting the pricing premium, e.g. 1.1x) when law requires geographic guarantees. |

### Worked Example: Requirement → Decision

**Given:** A regulated European insurer wants an internal document-QA assistant over policy documents and customer correspondence. Traffic is ~5 QPS during business hours, near-zero overnight. Legal requires that customer PII be processed only within the EU and never used to train a third party's model. Answer quality must be high but not frontier-level; latency budget is a few seconds.

**Step 1 — Identify the goal.** Choose a deployment that satisfies EU data residency and no-training guarantees while keeping cost and ops burden reasonable for modest, bursty traffic.

**Step 2 — Define inputs.** Regulated documents containing PII; ~5 QPS peak with a deep overnight trough (low average utilization); an internal ops team without deep GPU-fleet experience.

**Step 3 — Define outputs.** Grounded answers with citations, delivered within a few-second budget, with an auditable guarantee that data stayed in-region and was not used for training.

**Step 4 — Apply constraints.** EU residency (hard), no third-party training on data (hard), bursty low-average utilization (cost constraint against self-hosting), limited GPU ops maturity (constraint against self-hosting), non-frontier quality acceptable (opens up mid-tier and open-weight models).

**Step 5 — Select the approach.** Use a **managed API on Bedrock with an EU regional endpoint, VPC isolation via PrivateLink, and a documented no-training guarantee** — rather than self-hosting. Rationale: the residency and no-training requirements are met by Bedrock's regional Model Deployment Accounts and VPC path, so self-hosting is not *forced*; and because traffic is bursty with low average utilization, a self-hosted GPU fleet would sit idle most of the day and lose the cost argument, while burdening an inexperienced team with GPU ops. Self-hosting would only win here if audit rejected every managed path or if volume grew enough to keep GPUs busy.

---

## Implementation

```python
# Scenario: The insurer's compliance requirement is "inference must stay in the EU
# and never train a third party." A managed API satisfies it IF we pin the region
# and keep traffic inside our VPC — so we call Bedrock through a regional,
# VPC-routed client rather than a public global endpoint.
import boto3

# Region pinned to eu-central-1 so inference runs in-region (data residency).
# PrivateLink/VPC endpoint keeps the call off the public internet (configured at
# the network layer, not in this snippet).
client = boto3.client("bedrock-runtime", region_name="eu-central-1")

resp = client.converse(
    modelId="anthropic.claude-sonnet-4-5",  # non-frontier tier: "good enough", cheaper
    messages=[{"role": "user", "content": [{"text": grounded_prompt}]}],
)
answer = resp["output"]["message"]["content"][0]["text"]
```

```python
# Anti-pattern: hardcoding one provider's SDK and model string throughout the app.
# This welds you to a single vendor — you cannot A/B a cheaper model, fail over to
# self-hosted vLLM during a rate-limit spike, or move to a hybrid router without a
# rewrite. Vendor lock-in becomes a code-change cost, not a config change.
from anthropic import Anthropic
client = Anthropic()
def answer(q):
    return client.messages.create(
        model="claude-opus-4-8", max_tokens=1024,
        messages=[{"role": "user", "content": q}],
    ).content[0].text

# Correct approach: program against an OpenAI-compatible interface and pick the
# backend by config. vLLM, OpenAI, and Bedrock (via gateway) all speak this API,
# so local, hosted, and hybrid become a base_url/model swap — not a rewrite.
from openai import OpenAI

BACKENDS = {
    "local":  {"base_url": "http://vllm.internal:8000/v1", "model": "mistral-7b-instruct"},
    "hosted": {"base_url": "https://api.openai.com/v1",     "model": "gpt-5.5"},
}

def make_client(name):
    cfg = BACKENDS[name]
    return OpenAI(base_url=cfg["base_url"]), cfg["model"]

def answer(q, difficulty):
    # Hybrid routing: easy queries -> cheap local model; hard -> hosted frontier.
    backend = "hosted" if difficulty == "hard" else "local"
    client, model = make_client(backend)
    return client.chat.completions.create(
        model=model, messages=[{"role": "user", "content": q}],
    ).choices[0].message.content
```

---

## Common Pitfalls & Misconceptions

- **"Self-hosting is always cheaper because there's no per-token fee."** — Beginners compare the API's per-token price against a GPU's marginal cost and forget the GPU's fixed cost is paid whether it's busy or idle. The correct mental model is a break-even curve: self-hosting only wins *right of the crossover*, where high, sustained utilization amortizes the fixed GPU floor; below it, a mostly-idle GPU is more expensive per token than any API.
- **"An open-weight model is a drop-in replacement for a frontier API model."** — People assume all models are interchangeable because the API shape is identical. There is a real quality gap on hard reasoning/agentic tasks; the correct model is to benchmark the open model on *your* task before assuming parity, and to route only the queries it handles well to it.
- **"A managed API means my data is exposed, so regulated workloads must self-host."** — Teams conflate "third-party API" with "non-compliant." Managed providers offer VPC isolation (Bedrock PrivateLink), regional endpoints, and no-training guarantees; the correct model is to check whether a compliant managed path passes audit *first*, and only self-host when it genuinely cannot.
- **"Rate limits are a minor detail I'll deal with later."** — Beginners treat 429s as a retry problem, not an architecture one. The correct model is that RPM/TPM tiers cap your peak throughput and graduate slowly with spend, so a launch spike or a high-QPS batch job can hit a ceiling you can't raise in time — plan headroom, batching, or a self-hosted fallback up front.

---

## Key Definitions

| Term | Definition |
|---|---|
| Managed/Hosted API | An LLM served on a provider's or cloud reseller's infrastructure, consumed over HTTP and billed per token (OpenAI, Anthropic, Bedrock). |
| Self-hosted / open-weights | Running downloadable open-weight models (Llama, Mistral, Qwen) on GPUs you control, typically via an engine like vLLM. |
| Cost break-even | The token-volume/QPS point where fixed self-hosting cost equals variable API cost; left of it the API is cheaper, right of it self-hosting is. |
| GPU utilization | The fraction of time a GPU is doing useful inference; the primary lever on self-hosted per-token cost. |
| Rate limit (RPM/TPM) | Provider-imposed caps on requests-per-minute and tokens-per-minute, tied to a spend-based usage tier. |
| Vendor lock-in | Dependence on one provider's API, model, or roadmap that makes switching costly. |
| Data residency | A requirement that data be processed/stored within a specific geography, met via regional endpoints or in-region self-hosting. |
| Hybrid routing | Directing each query to the cheapest capable backend — local model for easy queries, hosted frontier model for hard ones. |
| Model Deployment Account (Bedrock) | A provider-owned account in your Region that runs the model without the model provider seeing your prompts/completions. |

---

## Summary / Quick Recall

- Five decision axes: data residency/compliance, cost at scale, latency/control, lock-in & rate limits, ops burden.
- Managed APIs win on time-to-market, frontier quality, and zero GPU ops; self-hosting wins on data control, high-volume cost, and freedom from rate limits/roadmaps.
- Self-hosting is cheaper only *past* the break-even point where GPU utilization is high — an idle GPU is expensive per token.
- Compliance rarely *forces* self-hosting: check a VPC-isolated/regional managed path first (Bedrock PrivateLink, `inference_geo`).
- Program against an OpenAI-compatible interface so local/hosted/hybrid is a config swap, not a rewrite — this also defuses lock-in.
- Hybrid routing captures most volume cheaply locally and pays frontier prices only for hard queries.
- Fine-tuning ownership and per-tenant LoRA adapters push toward self-hosting; provider-managed fine-tuning keeps you on the API.

---

## Self-Check Questions

1. What does the cost "break-even point" between a managed API and a self-hosted GPU fleet represent?

   <details><summary>Answer</summary>

   It is the token-volume/QPS level at which the fixed monthly cost of the self-hosted GPU fleet equals the variable pay-per-token cost of the equivalent API. Left of it the API is cheaper (the GPU would sit underutilized); right of it self-hosting is cheaper (fixed cost amortized across many tokens). The tempting wrong answer — "the point where the per-token marginal cost is equal" — is wrong because it ignores the GPU's fixed cost, which is paid whether the GPU is busy or idle and is exactly what break-even analysis exists to account for.

   </details>

2. Your app runs at ~4 QPS during business hours and near-zero overnight, with no residency constraints and non-frontier quality requirements. Which deployment is most cost-appropriate and why?

   <details><summary>Answer</summary>

   A managed API (with a small/mid-tier model), because bursty low-average-utilization traffic would leave a self-hosted GPU idle most of the day, paying its fixed cost to serve few tokens — putting you left of break-even. Self-hosting would only make sense if utilization were consistently high. Choosing self-hosting "to avoid per-token fees" is the trap: the idle-GPU fixed cost per token exceeds the API price at this volume.

   </details>

3. A bank says customer PII may be processed only in-region and never used to train a third party's model, but does not forbid all managed services. What is the correct first move before deciding to self-host?

   <details><summary>Answer</summary>

   Check whether a compliant managed path passes audit first: a regional endpoint pinned to the required geography plus VPC isolation (e.g. Bedrock + PrivateLink, whose per-Region Model Deployment Accounts prevent the model provider from seeing prompts/completions) and a documented no-training guarantee. Only if that path fails audit do you self-host. Jumping straight to self-hosting is the misconception that "third-party API" always equals "non-compliant" — often a VPC-isolated regional managed option satisfies the auditor at far lower ops burden.

   </details>

4. **Which TWO** of the following are legitimate reasons to self-host open weights instead of using a managed API?
   - A. You want the highest possible frontier reasoning quality with zero benchmarking.
   - B. You have sustained high token volume that keeps GPUs busy past the cost break-even.
   - C. You must own and export fine-tuned weights or serve many per-tenant LoRA adapters.
   - D. You want the fastest possible time-to-market with no infrastructure.
   - E. You want automatic rate-limit tier graduation without capacity planning.

   <details><summary>Answer</summary>

   **B and C.** B is correct because high sustained utilization amortizes the fixed GPU cost and puts you right of break-even where self-hosting is cheaper. C is correct because owning/exporting tuned weights and serving many per-tenant adapters (vLLM multi-LoRA) is a self-hosting strength that provider-managed fine-tuning can't fully replicate. A is wrong — frontier quality and zero-benchmark convenience favor a hosted frontier model, not self-hosting (there's a real quality gap). D and E are wrong because zero-infra time-to-market and automatic rate-limit graduation are advantages of managed APIs, not reasons to self-host.

   </details>

5. You must design for a launch that may spike to 10x normal QPS on day one, on a managed API. Two engineers propose: (a) rely on exponential-backoff retries alone, or (b) architect a self-hosted vLLM fallback plus request batching. Which is the sounder trade-off and why?

   <details><summary>Answer</summary>

   Option (b) is sounder as an architecture decision, because RPM/TPM rate limits cap peak throughput and usage tiers graduate slowly with cumulative spend — a 10x launch spike can hit a ceiling you cannot raise in time, and retries alone just resubmit against the same exhausted limit (and count against it). Batching raises effective token throughput under a fixed RPM, and a self-hosted fallback absorbs overflow. Option (a) is not wrong to *include* (backoff with jitter is best practice), but relying on it alone treats a capacity/architecture problem as a mere retry problem, which is the classic rate-limit pitfall.

   </details>

---

## Further Reading

- [vLLM Documentation](https://docs.vllm.ai/en/latest/) — *verified 2026-07-29* — Official docs for self-hosting open-weight models with an OpenAI-compatible server, continuous batching, quantization, and multi-LoRA.
- [OpenAI API — Rate limits](https://platform.openai.com/docs/guides/rate-limits) — *verified 2026-07-29* — How RPM/TPM limits, usage tiers, and spend-based graduation work, with backoff and batching mitigations.
- [Anthropic — Pricing](https://docs.anthropic.com/en/docs/about-claude/pricing) — *verified 2026-07-29* — Per-MTok model pricing, batch/cache discounts, and the data-residency (`inference_geo`) pricing multiplier.
- [Amazon Bedrock — What is Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html) — *verified 2026-07-29* — Managed multi-provider model access, supported APIs, and Regions.
- [Amazon Bedrock — Data protection](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html) — *verified 2026-07-29* — VPC/PrivateLink isolation, per-Region Model Deployment Accounts, and the shared-responsibility model for regulated data.
