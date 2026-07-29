# Evaluation, Observability & Guardrails — Interview Prep

**Section:** Production AI Systems — Security, Evaluation & Scale | **Role target:** Applied AI Engineer / AI Systems Engineer (agentic AI, RAG, multi-agent production systems)

---

## Core Conceptual Questions

These test whether you understand the fundamentals.

| Question | Key points to cover | Common weak-answer trap |
|---|---|---|
| What is the difference between retrieval metrics and generation metrics in a RAG evaluation, and why can't a single number replace both? | Retrieval metrics judge whether the right context was fetched: **context precision** (are the retrieved chunks relevant / ranked well) and **context recall** (did we retrieve all the context needed to answer, usually measured against ground-truth), plus ranking metrics like **MRR** and **NDCG**. Generation metrics judge the produced answer: **faithfulness** (is the answer grounded in the retrieved context, no hallucination), **answer relevancy** (does it address the question), and **answer correctness** (does it match a reference). They fail independently — perfect retrieval with a hallucinating generator, or a faithful generator starved of context — so you must measure both stages separately to localize failure. | Collapsing everything into one "accuracy" number, or only checking the final answer — which hides *where* the pipeline broke (bad retrieval vs. bad generation) and makes the system undebuggable. |
| Explain the RAG triad. Which corner catches hallucination? | The **RAG triad** is three pairwise relevance/groundedness checks: **context relevance** (retrieved context ↔ question), **groundedness/faithfulness** (answer ↔ retrieved context), and **answer relevance** (answer ↔ question). Groundedness is the anti-hallucination corner — it verifies every claim in the answer is supported by the retrieved context. The triad is powerful because it is **reference-free**: you don't need golden answers, so it works on live production traffic. | Saying the triad "measures accuracy" or naming only one corner; or claiming faithfulness needs a golden reference — it's measured against the *retrieved context*, not a labeled answer. |
| How does evaluating an agent differ from evaluating a single RAG answer? | Agents need **trajectory evaluation** (was the sequence of steps/tool calls sensible), **tool-call accuracy** (right tool, right arguments), and **task success / outcome eval** (did the end state satisfy the goal), in addition to final-output quality. You evaluate both the *path* and the *outcome* — a correct final answer reached via a broken/expensive trajectory is still a problem, and a good trajectory that fails the task is still a failure. Reference trajectories or expected tool sequences form the golden set. | Only checking the final answer ("did it get the right answer") and ignoring the process — misses wrong-tool loops, wasted calls, and non-reproducible successes; or conflating trajectory with outcome as if one implies the other. |
| What is LLM-as-judge, what scoring modes exist, and what biases must you control for? | An LLM scores/ranks outputs against a rubric instead of (or alongside) humans. Modes: **binary** (pass/fail), **Likert** (graded score, e.g. 1–5), and **pairwise** (which of A/B is better — usually the most reliable). Judging can be **reference-based** (compare to a golden answer) or **reference-free** (score against a rubric only). Known biases: **position bias** (favoring the first/second option — mitigate by swapping order and averaging), **verbosity bias** (favoring longer answers — mitigate with rubric + length controls), and **self-preference bias** (a judge favoring outputs from its own model family — mitigate with a different judge model). You must **calibrate the judge against human labels** (agreement rate) before trusting it. | Treating the judge as objective/ground-truth, using a Likert score with no rubric, or using the same model to generate *and* judge without noting self-preference bias and without any human calibration. |
| What should you log for observability, and what's the unit of an agent trace? | A **trace** is one end-to-end request; it decomposes into **spans** (one per LLM call, retrieval, tool call, guardrail check) forming a tree. Per span log: prompt/completion, **token counts, cost, latency**, model + prompt **versions**, tool name/args/result, and retrieval hits. **OpenTelemetry GenAI** semantic conventions standardize this; **LangSmith** is a purpose-built tracing/eval backend. Production signals to watch: latency percentiles, cost per request, error/guardrail-trip rates, and online eval scores on sampled traffic. | "We log the final answer" — no per-step spans means you can't localize a failing tool or a bad retrieval; or logging prompts but not tokens/cost/latency/versions, so you can't attribute regressions to a model or prompt change. |
| Where do input vs. output guardrails sit, and what is fail-open vs. fail-closed? | **Input guardrails** run before the model: **PII detection/redaction**, **prompt-injection** detection, **topic/off-scope** filtering. **Output guardrails** run after generation: **toxicity**, **PII leak** detection, **groundedness** check (does the answer stick to sources), and **schema/format validation**. **Fail-closed** blocks/degrades the request when a guardrail errors or trips (safe default for high-stakes); **fail-open** lets it through (availability over safety). The core trade-off is **latency + false-positive rate vs. safety** — every guardrail adds a hop and can wrongly block legitimate traffic. | "A good prompt means we don't need guardrails," or adding guardrails with no thought to placement, added latency, false-positive cost, or what happens when the guardrail *itself* fails (silent fail-open). |

---

## Applied / Scenario Questions

### Q1 — Your production RAG assistant's user-reported answer quality is dropping, but your offline eval scores look flat. How do you find and fix the regression?

**Strong answer framework:**
- **Split the metric first:** flat aggregate offline scores often hide a stage-specific regression. Break down into retrieval metrics (context precision/recall) vs. generation metrics (faithfulness, answer relevancy) to localize — is retrieval fetching worse context, or is the generator drifting?
- **Offline ≠ online:** offline eval runs on a fixed golden dataset that may not reflect the current traffic distribution (data/query drift). Stand up **online eval** on sampled production traces using reference-free metrics (the RAG triad — context relevance, groundedness, answer relevance) so you measure real traffic, not a stale test set.
- **Trace to the failing span:** use per-span traces to inspect real failing requests — check whether retrieval hits changed (new/re-embedded docs, index change) or whether a model/prompt **version** bump correlates with the drop. Version tags on spans make this attribution direct.
- **Trade-off framing:** online eval costs money and latency (judge calls on live traffic), so **sample** rather than scoring 100% — e.g. score 1–5% of traffic, weighted toward low-confidence or guardrail-adjacent requests. This trades complete coverage for bounded observability cost.
- **Close the loop:** add the newly discovered failure cases to the golden dataset (versioned) so the regression is caught offline next time — eval-driven development.

### Q2 — You want to replace expensive human labeling with an LLM-as-judge in your CI eval gate. How do you make it trustworthy, and what breaks if you don't?

**Strong answer framework:**
- **Calibrate before trusting:** measure **judge-vs-human agreement** on a held-out human-labeled set before wiring the judge into any gate. If agreement is low, the judge's "pass" is meaningless. Report agreement (e.g. Cohen's κ / % agreement), not just judge scores.
- **Control judge biases:** use **pairwise** comparison where possible (more reliable than absolute Likert), **swap positions and average** to cancel position bias, add rubric + length normalization for verbosity bias, and use a **different model family** as judge to avoid self-preference bias.
- **Design the rubric:** a scored rubric with explicit criteria and anchored examples beats a vague "rate 1–5"; it reduces variance and makes disagreements diagnosable.
- **Trade-off framing:** LLM-as-judge is cheap, fast, and scalable but noisier and biased vs. human labels which are the gold standard but slow/expensive. The pragmatic answer: judge for high-volume CI/online screening, keep a small periodic human-labeled audit set to re-validate calibration over time (calibration drifts as models change).
- **Wire it as a gate carefully:** the CI eval gate should block a merge on a **regression** against a baseline (delta on the golden set), not on an absolute score, and fail-closed on a clear drop — but budget for false positives so the judge's noise doesn't block every PR.

### Q3 — Guardrails on your customer-facing agent are blocking legitimate requests (false positives) and adding noticeable latency. How do you tune this?

**Strong answer framework:**
- **Quantify the two costs:** measure the guardrail **false-positive rate** (legit requests blocked) and its **added latency** (extra hop per input/output check) — you can't tune a trade-off you haven't measured.
- **Placement and selectivity:** run cheap/high-precision checks inline (schema validation, regex PII) and reserve expensive model-based checks (injection classifier, groundedness LLM) for where risk warrants it; consider running some output guardrails async/sampled if the action is reversible.
- **Fail-open vs. fail-closed per action:** irreversible or high-stakes paths (money movement, data deletion, medical/legal advice) should **fail-closed**; low-stakes informational replies can tolerate **fail-open** to protect availability. Never fail-open *silently* — log every guardrail error.
- **Trade-off framing:** tightening a guardrail's threshold cuts safety incidents but raises false positives and user friction; loosening does the reverse. Pick the operating point from the cost of a miss vs. the cost of a false block for *that* action class — not one global threshold.
- **Tune with data:** collect blocked-request samples, label them, and adjust thresholds/rules against a measured precision-recall curve rather than by feel; add the confirmed false positives as regression cases.

---

## System Design / Architecture Questions

### Q — Design an evaluation + observability + guardrail stack for a production RAG assistant that answers from internal policy documents, with near-zero tolerance for hallucinated answers.

**Approach:**

1. **Clarify requirements.**
   - Hallucination tolerance: near-zero — a wrong grounded-sounding answer is worse than a refusal, so groundedness is the top priority signal.
   - Latency budget (interactive, e.g. target < a few seconds) and QPS/concurrency — determines how much inline guardrailing and online eval sampling we can afford.
   - Data sensitivity: internal policy docs likely contain PII/confidential content → input PII handling and output PII-leak checks matter.
   - Do we have (or can we build) a **golden dataset** of policy questions with reference answers and reference contexts? What's the audit/compliance requirement (traceability)?
   - Reversibility: this is a read/answer assistant (low action risk) but high *information* risk — bias guardrails toward blocking ungrounded answers.

2. **High-level architecture.**
   - **Metrics layer.** Retrieval: context precision + context recall (+ MRR/NDCG for ranking) against the golden set. Generation: **faithfulness/groundedness** (primary), answer relevancy, answer correctness vs. reference. Use the **RAG triad** for reference-free scoring on live traffic. (Ragas-style metric definitions.)
   - **Offline eval pipeline / CI gate.** Versioned golden dataset (curated + synthetic-augmented for coverage). On every prompt/model/retriever change, run the eval suite; **CI eval gate** blocks a merge on a groundedness/faithfulness regression vs. baseline. LLM-as-judge for graded/reference-free criteria, **calibrated against a human-labeled subset**, biases controlled (pairwise where possible, position-swap, non-self judge).
   - **Observability / tracing.** Every request emits a **trace** of **spans** (retrieval, LLM call, each guardrail) with tokens, cost, latency, and model/prompt/retriever **versions** — via **OpenTelemetry GenAI** conventions into a backend like **LangSmith**. Dashboards on latency percentiles, cost/request, guardrail trip rate, and **online eval** groundedness on *sampled* traffic.
   - **Guardrails.** *Input:* PII redaction, prompt-injection detection, topic/off-scope filter (only policy questions). *Output:* a **groundedness guardrail** (verify every claim is supported by retrieved context — reject/route-to-fallback if not), PII-leak check, toxicity, and schema validation. Ungrounded answers **fail-closed** to a safe fallback ("I can't find that in the policy documents") rather than being emitted.

3. **Justify choices and name trade-offs.**
   - **Groundedness fail-closed is the core decision:** given near-zero hallucination tolerance, an ungrounded answer must be blocked/replaced even at the cost of some false refusals and added output-check latency — trading availability/recall for safety, the correct trade here.
   - **RAG triad + online eval on sampled traffic** because a fixed golden set can't cover live query drift; sampling (not 100%) bounds the extra judge cost/latency — trading full coverage for affordable production observability.
   - **LLM-as-judge over pure human labeling** for scale/speed in CI, accepting judge noise/bias — mitigated by calibration + bias controls + a periodic human audit set; the trade is cost/throughput vs. label reliability.
   - **Per-span tracing with versions** so a groundedness regression can be attributed to a retriever re-index vs. a prompt change — trades storage/instrumentation overhead for debuggability (non-negotiable at near-zero-hallucination bar).
   - **CI eval gate on deltas, not absolutes**, so noise doesn't block every deploy while real regressions do — trades occasional false blocks for regression safety.
   - **Guardrail latency budget:** cheap checks inline, expensive groundedness check on the output path accepted as latency cost because it's the primary safety control — but measured, not assumed.

---

## Vocabulary That Signals Expertise

Use these terms naturally — don't force them:

- **Faithfulness / groundedness** — when arguing the anti-hallucination signal: is every claim supported by the retrieved context (measured against context, not a golden answer).
- **Context precision / context recall** — when localizing a RAG failure to the retrieval stage (relevant + well-ranked vs. complete-enough context).
- **MRR / NDCG** — when the concern is retrieval *ranking* quality, not just hit/miss.
- **Answer relevancy / answer correctness** — when separating "addresses the question" from "matches the reference."
- **RAG triad (context relevance / groundedness / answer relevance)** — when you need reference-free evaluation you can run on live production traffic.
- **Trajectory evaluation / tool-call accuracy / task success** — when evaluating agents, to separate the *path* from the *outcome*.
- **Golden dataset / synthetic data / reference-based vs. reference-free** — when discussing what you measure against and how you get coverage.
- **Offline vs. online eval / eval-driven development** — when framing CI-gate evals vs. live-traffic scoring and adding failures back to the test set.
- **LLM-as-judge (binary / Likert / pairwise; reference-based vs. free)** — when scaling evaluation beyond human labels.
- **Position bias / verbosity bias / self-preference bias** — when showing you know judge failure modes and their mitigations (swap-and-average, rubric+length control, cross-model judge).
- **Judge-vs-human agreement / calibration** — when establishing whether a judge can be trusted before wiring it into a gate.
- **Trace / span** — when describing the unit of observability (request → tree of instrumented steps).
- **OpenTelemetry GenAI / LangSmith** — when naming the standard/tooling for standardized tracing and eval backends.
- **Input vs. output guardrails; fail-open vs. fail-closed** — when reasoning about guardrail placement and the safety-vs-availability default.
- **Guardrail false-positive rate vs. latency vs. safety** — when framing the guardrail tuning trade-off quantitatively.
- **Canary / A/B eval** — when rolling out a model/prompt change and comparing eval scores on live traffic slices.

---

## Vocabulary That Signals Weakness

Avoid these — they signal outdated or shallow understanding:

- **"We eyeball the outputs / spot-check a few answers."** — Red flag: no systematic golden set, no metrics, not reproducible; can't detect regressions or scale.
- **"Accuracy is one number."** — Red flag: collapses retrieval vs. generation (and trajectory vs. outcome) into a single figure that hides *where* the system fails and makes it undebuggable.
- **"The judge LLM is objective / it's ground truth."** — Red flag: ignores position/verbosity/self-preference bias and the need to calibrate against human labels before trusting it.
- **"We use the same model to generate and grade."** — Red flag: self-preference bias with no cross-model judge or calibration noted.
- **"A good prompt means we don't need guardrails."** — Red flag: conflates prompt quality with runtime safety; no defense against injection, PII leak, or ungrounded output at inference time.
- **"We log the final answer."** — Red flag: no per-span traces, so you can't localize a failing tool or bad retrieval, or attribute a regression to a version change.
- **"Guardrails just filter bad words."** — Red flag: reduces guardrails to a toxicity blocklist; misses injection, PII, groundedness, schema, placement, and fail-open/closed.
- **"We score 100% of traffic online."** — Red flag: ignores the cost/latency of judge calls and the standard practice of *sampling* production traffic.
- **"Faithfulness needs a golden answer."** — Red flag: misunderstands that groundedness is measured against the *retrieved context*, which is exactly why it works reference-free in production.

---

## STAR Answer Frame

**Situation:** On a production RAG assistant I owned (LangGraph + FastAPI/PostgreSQL), users started reporting confident but wrong answers after a document re-ingestion, yet our aggregate offline eval score looked unchanged — so the regression was invisible to the team.

**Task:** Find and stop the hallucination regression, and make this class of failure detectable *before* it reached users going forward, without an unaffordable eval bill on live traffic.

**Action:** I (1) split the flat aggregate metric into retrieval vs. generation — context precision/recall had dropped after the re-index while generation metrics held, localizing the fault to retrieval; (2) confirmed it in **per-span traces**, where version tags tied the drop to the re-embedding job; (3) added **online eval** using the reference-free **RAG triad** (groundedness + context relevance) on a **sampled** 3% of production traffic so we measured real queries at bounded cost; (4) added an **output groundedness guardrail** that **fails closed** to a "not found in the docs" fallback when claims aren't supported; (5) introduced a **CI eval gate** with an **LLM-as-judge calibrated against a human-labeled subset** (pairwise, position-swapped, cross-model to control bias) that blocks merges on a groundedness *delta* vs. baseline; and (6) folded the discovered failure cases into the versioned golden dataset.

**Result:** Hallucination user reports on that flow dropped to near zero after the fail-closed guardrail shipped; the CI gate caught two later retriever changes before release; online-eval cost stayed low by sampling (~3% of traffic) while giving us groundedness visibility we previously lacked — and the previously invisible regression class was now caught offline.

---

## Red Flags Interviewers Watch For

Specific to evaluation, observability & guardrails:

- **No golden dataset** — proposing to ship/tune a RAG or agent system with nothing to measure against; "we'll just look at the outputs" instead of a versioned reference set.
- **No retrieval/generation split** — reporting a single "accuracy" and being unable to say whether a failure is in retrieval or generation (or, for agents, trajectory vs. outcome).
- **Unvalidated LLM judge** — wiring an LLM-as-judge into a gate with no human calibration and no bias controls (position/verbosity/self-preference), treating its score as ground truth.
- **No per-step tracing** — logging only the final answer, with no spans for retrieval/tool/guardrail steps, tokens, cost, latency, or model/prompt versions — so failures can't be localized or attributed.
- **Guardrails that fail-open silently** — a guardrail that, on its own error, lets the request through with no log or alert, quietly disabling the safety control it was supposed to provide.
- **Ignoring guardrail latency / false positives** — bolting on multiple model-based checks with no measurement of added latency or the rate of legitimate requests wrongly blocked, and no per-action fail-open/closed reasoning.
- **Offline-only, never online** — trusting a fixed test set forever and never scoring sampled live traffic, so query/data drift goes undetected.
- **Scoring everything online** — the inverse: no sampling strategy, ignoring the cost/latency of judge calls on 100% of production traffic.
