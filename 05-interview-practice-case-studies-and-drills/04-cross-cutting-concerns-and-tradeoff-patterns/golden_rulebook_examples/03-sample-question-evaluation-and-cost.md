# Sample Question — Multi-Agent Research Assistant Under a Cost Ceiling (Golden Rulebook Worked Answer)

This worked answer is grounded strictly in the Golden Rulebook cheatsheet at `../00-golden-rulebook-cheatsheet.md`; every recommendation traces to a named row, golden rule, decision cue, trade-off one-liner, or red flag in that document.

---

## The Question

**Context.** You own a multi-agent research/analyst assistant at a mid-size firm. A user asks a question ("summarise our exposure to supplier X across the last 8 quarterly filings"), and an orchestrator fans out to sub-agents — a planner, several retrieval/reader agents that each work one source, and a synthesiser that composes the final answer with citations. Typical fan-out is 6–12 sub-agent calls per user question, occasionally deeper on complex tasks. You serve roughly **40,000 questions/day** over a corpus of internal filings and reports. Leadership has two demands that arrived in the same meeting: **prove the answers are trustworthy**, and **stop the LLM bill from climbing** — it is currently trending toward a hard **$60k/month ceiling**.

**Requirements:**
- Answer quality is provable: an **abstain-or-answer** policy with **low hallucination tolerance** (advisory tool, but wrong-with-confidence is the failure that gets us shut down).
- Every answer carries **verifiable citations** back to source chunks.
- **Faithfulness** (answer supported by retrieved context) is the headline quality metric; no cost cut ships if it regresses.
- **Monthly LLM spend ≤ $60k** at 40k questions/day (≈ **$0.05/question** blended ceiling).
- p95 end-to-end latency budget **< 8s** for interactive questions; background/bulk work may run async.
- Any change — prompt, model, corpus, k — must clear an **eval gate** before it reaches live traffic.

**Your task:**
1. Sketch the **end-to-end architecture** and walk the request path, marking **where tokens and dollars are spent** per hop.
2. Design the **evaluation system**: offline golden-set CI gate plus online guardrail metrics, including the RAG metric split and LLM-as-judge with bias control.
3. Design **hallucination / faithfulness detection** and the abstain policy that enforces the low hallucination tolerance.
4. Do a **cost back-of-envelope** for $/question and name the levers to cut cost ~50%, each eval-gated.
5. Show how you **prove quality did not regress** when you cut cost — the eval-gating workflow.
6. Cover **rollout, monitoring, and the trade-offs** you are consciously making.

---

## Worked Answer

Before designing, I'd clarify the two dials that set everything: the **quality bar** (I'll take it as advisory-but-low-hallucination-tolerance, abstain permitted when retrieval is weak) and the **cost ceiling** ($60k/month at 40k q/day ≈ **$0.05/question blended**). Those two clarifiers come from the accuracy/quality and cost-budget groups, and they map to two SLOs I'll restate throughout: **faithfulness held or improved on every change**, and **blended $/question ≤ $0.05**. Latency (p95 < 8s) and reliability I'll sweep lightly; evaluation and cost are the load-bearing concerns here.

The framing rule I commit to up front is Golden Rule 16 — **never optimise one axis blind; a cost win is real only if accuracy and safety held.** That is why tasks 4 and 5 are welded together: the cost cuts and the eval gate are one workflow, not two.

### Answer

#### 1. End-to-end architecture and where tokens/dollars are spent

I'd sketch the standard RAG spine (ingest → chunk → embed → index → retrieve → rerank → generate → cite) wrapped in a bounded multi-agent orchestration, then walk the request path noting time, tokens, and dollars per hop. The key insight from the Cost row is that in a multi-agent system, **fan-out is the silent multiplier** — every hop below is paid once *per sub-agent*, so 6–12× is the real cost base, not 1×.

```text
USER QUESTION
   │
   ▼
[Orchestrator / Planner]  ── 1 LLM call ── small-model plan
   │  tokens: modest input, capped output
   │  $: low (routed to smallest model that passes eval)
   ▼
[Fan-out: 6–12 sub-agents]  ◄── SILENT MULTIPLIER: every hop below × fan-out
   ├─ Retriever  → vector index (retrieve top-k) → rerank
   │     tokens: prefill grows with k  ·  $: embedding + rerank (cheap vs generation)
   ├─ Reader/Extractor  → 1 LLM call per source
   │     tokens: large INPUT (source chunks)  ·  $: input-heavy
   ▼
[Synthesiser]  ── 1 LLM call ── composes answer + citations
   │  tokens: large OUTPUT (the answer)  ← OUTPUT ≈ 4–5× cost of input
   │  $: the single most expensive hop
   ▼
[Cite / verify]  → attach source spans, run faithfulness check
   ▼
ANSWER (or ABSTAIN)
```

Dollar hotspots, in order: (1) the **synthesiser's output tokens** — output runs ~4–5× input cost, so uncapped generation dominates; (2) the **fan-out multiplier** — reader calls are individually cheap but paid 6–12×; (3) **retrieval prefill** — larger k inflates input tokens on every reader. Embedding and rerank are rounding errors by comparison. This walk is the framework's "walk the request path (time/tokens/dollars per hop)" step, and it tells me exactly where the levers in task 4 must land.

#### 2. Evaluation system — offline gate + online guardrail

Golden Rule 7 says **design eval before you build**, so the first deliverable is the golden set, not a feature. I split evaluation two ways.

**Offline (CI gate, with references).** A curated golden set of question→expected-answer pairs, run on every prompt/model/corpus/k change. Because this is RAG, I split the metrics so a failure localises:
- **Retrieval:** context precision + context recall — is the retriever pulling the right chunks at all?
- **Generation:** faithfulness (supported by context — the hallucination metric) + answer-relevancy (does it address the question?).

**Online (no reference, watch for drift).** On live traffic I run guardrail metrics and A/B live changes rather than trusting the offline number, because the cheatsheet is explicit that **offline can't cover the live distribution**.

For scoring the generation metrics I'd use **LLM-as-judge but bias-controlled** — the cheatsheet names position bias, verbosity bias, and self-preference. Concretely: randomise candidate order, don't reward length, and be wary of a judge from the same model family as the generator. I present judge scores with that caveat, never as ground truth. This three-part posture — offline gold-set gate + online guardrail metrics + LLM-as-judge with bias caveats — is exactly the "how do you know it works?" decision cue.

#### 3. Hallucination / faithfulness detection and the abstain policy

Low hallucination tolerance means faithfulness is not just measured offline — it's enforced at request time. Two mechanisms:

- **Faithfulness check in the cite/verify hop.** After the synthesiser produces an answer, I verify each claim is supported by a retrieved chunk (the answer must be groundable in its own citations). Unsupported claims are the hallucination signal.
- **Abstain gate.** When retrieval quality is weak (low context recall/precision for this query) or the faithfulness check fails, the assistant **abstains** rather than answering — this is the abstain-or-answer policy the quality bar demands. Abstaining is the safe failure; a confident wrong answer is the unsafe one.

Offline, faithfulness and context precision/recall from task 2 are the leading indicators; online, a rising abstain rate or falling faithfulness on live traffic is a drift alarm. This keeps the hallucination concern instrumented, not hoped-for.

#### 4. Cost back-of-envelope and the 50% levers

Let me do a rough $/question, **clearly labelling unit prices as assumptions** (I'd use our real contracted rates in the room):

```text
ASSUMED (label as assumptions):
  fan-out           ≈ 8 sub-agent calls / question
  input tokens/call ≈ 4k (source chunks)   output ≈ 0.7k
  assumed price     ≈ $3 / 1M input,  $15 / 1M output   (output ≈ 5× input — from cheatsheet)

Per sub-agent call:
  input : 4k × $3/1M   = $0.012
  output: 0.7k × $15/1M = $0.0105
  ≈ $0.0225 / call
Per question (× 8 fan-out): ≈ $0.18/question

At 40k q/day × 30d = 1.2M questions/month → ≈ $216k/month
  → WAY over the $60k ceiling; need ~4× reduction (well past the "50%" ask)
```

Every lever below is drawn from the **Cost row** and each is **eval-gated** (ship only if faithfulness/answer-relevancy hold):

- **Cap `max_tokens`** — output is the expensive class; bound synthesiser output so no call runs away. (Golden Rule 5.)
- **Attack fan-out** — the silent multiplier. Prune sub-agent calls that don't earn their keep; collapse readers where one call over multiple chunks suffices. Cutting 8→5 fan-out is ~40% alone.
- **Route to the smallest model that passes eval** — planner, reader, relevance-judging are not frontier work; route them down, escalate to frontier only for genuinely hard synthesis. (Golden Rule 6.)
- **Prompt-cache the stable prefix** — large shared system prompt + tool defs are reused every call; caching is ~10% at zero output change.
- **Semantic cache** — repeat/equivalent questions skip generation entirely (biggest per-hit save). Cache semantically, not by exact string (Golden Rule 5).
- **Batch API for async work** — background enrichment/report generation isn't latency-critical; Batch is ~−50% with no quality change, purely a scheduling decision.
- **Tune k** — pull retrieval k to the smallest value that still holds recall; large k inflates prefill on every reader (recall vs cost/latency of large-k trade-off).

Stacking fan-out pruning + model routing + `max_tokens` cap + semantic/prompt cache + Batch for async work is what takes us from ~$0.18 toward the $0.05 blended target — comfortably past the 50% ask, without touching the reflex red flag "just use a smaller model," which ignores fan-out entirely.

#### 5. Proving quality did not regress when cutting cost

This is where task 4 and task 2 fuse. The Cost row's trade-off is blunt: **every cut risks accuracy and must be eval-gated.** So the workflow for each lever is:

```text
propose cut → run offline golden set (faithfulness, answer-relevancy,
              context precision/recall) → metrics HOLD? ──no──► reject cut
                                                        └─yes─► canary on live traffic
                                                                → online guardrails HOLD? ─► promote
                                                                                          └► rollback
```

The eval gate is precisely what makes the degradation *not silent* — the word leadership cares about. A lever that shaves $/question but drops faithfulness below bar does not ship, however cheap. This is Golden Rule 16 made operational: a cost win only counts once accuracy is proven held.

#### 6. Rollout, monitoring, trade-offs

Rollout follows the framework's failure-mode path: **eval-gate → canary → rollback.** No change (prompt, model, k, cache policy) reaches full traffic without clearing the offline gate, proving out on a canary slice, and passing online guardrails. Monitoring watches the two SLOs continuously: **faithfulness / abstain rate** (drift = quality alarm) and **blended $/question vs the $0.05 ceiling** (cost alarm), plus p95 latency as a secondary guardrail.

Conscious trade-offs I'd state out loud:
- **Cost vs quality** — output caps, smaller models, and lower k genuinely cut spend but can lower goal-accuracy; that's exactly why every cut is gated behind eval.
- **Velocity vs a maintained eval set** — the golden set is real ongoing work; let it rot and the gate becomes theatre.
- **LLM-as-judge bias** — judge scores steer decisions but carry position/verbosity/self-preference bias, so I treat them as bias-controlled signal, not truth.
- **Distillation** — noted as a second-phase option; it trades upfront cost for lower per-call cost and needs its own eval before it's trusted.

---

### How the cheatsheet was used

- **Task 1 → 60-Second Framework "sketch architecture" + "walk request path (time/tokens/dollars per hop)"** → drove the ingest→…→cite spine and the per-hop dollar walk. **Cost row "fan-out is the silent multiplier" + "output tokens ~4–5× input"** → identified synthesiser output and fan-out as the dollar hotspots.
- **Task 2 → 9-Concern Evaluation row** → offline-vs-online split, RAG metric split (context precision/recall + faithfulness/answer-relevancy), LLM-as-judge, A/B on live. **Golden Rule 7 "design eval before you build"** → golden set is the first deliverable. **Eval row trade-off "judge bias / offline can't cover live"** → bias control + online guardrails.
- **Task 3 → Evaluation row faithfulness + Clarifying accuracy/quality group (hallucination tolerance, abstain-or-answer)** → request-time faithfulness check and abstain gate.
- **Task 4 → 9-Concern Cost row (cap max_tokens, route smallest model that passes eval, prompt-cache ~10%, semantic cache, Batch API −50%, tune k, output ≈ 4–5× input, fan-out multiplier)** → every lever. **Golden Rule 5** (cache semantically, cap max_tokens) and **Golden Rule 6** (route to smallest model that passes eval) → the two headline levers. **Decision Cue "keep the bill down"** → the ordered lever set. **Red Flag "just use a smaller model"** → contrasted against fan-out-first hygiene.
- **Task 5 → Cost row trade-off "every cut risks accuracy; must eval-gate" + Golden Rule 16 "never optimise one axis blind"** → the propose→gate→canary→promote/rollback workflow that makes degradation non-silent.
- **Task 6 → Framework failure-mode step "eval-gate → canary → rollback"** → rollout. **Trade-off One-Liners (cost vs quality; recall vs cost/latency of large-k)** and **Cost row distillation note** → the stated trade-offs.

---

## Cheatsheet elements referenced

- **60-Second Framework** — clarify & scope, name SLOs, sketch architecture, walk request path (time/tokens/dollars per hop), sweep 9 concerns, state trade-offs, failure modes + rollout (eval-gate → canary → rollback).
- **9-Concern Sweep → Evaluation/Quality row** — offline gold-set gate vs online drift, RAG split (context precision/recall + faithfulness/answer-relevancy), LLM-as-judge (bias-controlled), A/B on live; trade-offs (judge bias, offline can't cover live, velocity vs maintained eval set).
- **9-Concern Sweep → Cost/Token Efficiency row** — output ~4–5× input, fan-out as silent multiplier, cap max_tokens, route to smallest model that passes eval, prompt-cache stable prefix (~10%), semantic cache, Batch API (−50%), tune k; trade-off (every cut risks accuracy, must eval-gate, distillation trades upfront cost).
- **Golden Rule 5** — cache semantically not by exact string; cap max_tokens (output is the expensive class).
- **Golden Rule 6** — route to the smallest model that passes eval; escalate to frontier only for hard requests.
- **Golden Rule 7** — design eval before you build; eval-gate every change.
- **Golden Rule 16** — never optimise one axis blind; a cost/latency win is real only if accuracy/safety held.
- **Decision Cue: "how do you know it works?"** — offline gold-set gate + online guardrail metrics + LLM-as-judge (bias caveats).
- **Decision Cue: "keep the bill down."** — model routing + prompt caching + semantic cache + cap max_tokens; Batch API async.
- **Trade-off One-Liner: cost vs quality** — gate every cut behind eval.
- **Trade-off One-Liner: recall vs cost/latency of large-k retrieval** — smallest k that holds recall.
- **Red Flag: "just use biggest/smaller model"** — reflex that ignores fan-out.
- **Red Flag: "we'll add eval/monitoring later"** — eval is the instrument you steer by.
- **Clarifying Questions — accuracy/quality bar group** — hallucination tolerance, advisory vs binding, abstain-or-answer.
- **Clarifying Questions — cost budget group** — monthly token ceiling, cost-per-question/per-tenant target, agent fan-out.
