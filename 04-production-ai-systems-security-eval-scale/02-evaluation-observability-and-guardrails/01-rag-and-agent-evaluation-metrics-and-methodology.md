# RAG and Agent Evaluation Metrics and Methodology

**Section:** 04 Production AI Systems: Security, Eval & Scale → Evaluation, Observability & Guardrails | **Est. time:** 3.5 hrs | **Interview relevance:** High — "how do you know your RAG/agent actually works?" is the core senior evaluation question; if you can't decompose quality into retrieval-vs-generation layers and name the metrics you'd gate a deploy on, the rest of your system design is unfalsifiable.

---

## TL;DR

Evaluating an LLM system is hard because the output is non-deterministic and there is rarely a single correct answer, so you can't score it with plain accuracy. The discipline is to split quality into two measurable layers — **retrieval** (did we fetch the right context? → context precision, context recall) and **generation** (did the answer use that context and address the question? → faithfulness, answer/response relevancy) — plus a set of **agent** metrics that ask whether the agent called the right tools (tool-call accuracy) and reached the user's goal (agent goal accuracy). You ground these in a **golden dataset** of test cases, choose **reference-based vs reference-free** metrics per what labels you have, and run them **offline** (before deploy, as a regression gate) and **online** (on live traffic). The whole practice is eval-driven development: baseline → change → measure → guard against regressions. **The one thing to remember: never report one number — decompose quality into retrieval and generation (and, for agents, trajectory and outcome) so a failing score tells you *which stage* to fix.**

---

## ELI5 — Explain It Like I'm 5

Imagine grading a student's open-book essay. A lazy grader just skims the essay and says "sounds smart, A-minus" — but that tells you nothing about *why* it's good or bad. A real grader checks two separate things. First: did the student open the *right pages* of the textbook — did they pull the sources that actually contain the answer, and put the useful ones on top? That's the retrieval check. Second: does the essay's argument actually *come from* those pages (not made up), and does it actually *answer the question that was asked*? That's the generation check. The common mistake people make is thinking evaluation is "eyeball a few answers, they look fine" or "accuracy is one number" — but a single glance can't tell you whether a wrong answer came from grabbing the wrong pages or from ignoring the right ones, and that's exactly the difference that tells you what to fix.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain why LLM-system evaluation is hard (non-determinism, no single ground truth) and why a single aggregate score is insufficient.
- [ ] Distinguish the retrieval layer (context precision, context recall, MRR/NDCG) from the generation layer (faithfulness, answer relevancy, answer correctness) and name the Ragas metric for each.
- [ ] Apply the RAG triad (context relevance, groundedness/faithfulness, answer relevance) to localise a failure to the retriever vs the generator.
- [ ] Select agent-specific metrics — tool-call accuracy for trajectory, agent goal accuracy for outcome — and decide when to evaluate single-step vs end-to-end.
- [ ] Design a golden evaluation dataset, choose reference-based vs reference-free metrics, and structure an eval-driven-development loop with an offline regression gate plus online monitoring.

---

## Visual Overview

### Two Layers of RAG Quality (metric per stage)

```
                RETRIEVAL LAYER                    GENERATION LAYER
        ┌──────────────────────────┐      ┌───────────────────────────────┐
query ─►│ retriever ──► retrieved   │─────►│ LLM ──► response              │
        │            contexts       │ctx   │                               │
        └──────────────────────────┘      └───────────────────────────────┘
             ▲ measure HERE:                    ▲ measure HERE:
             • context precision (ranking)      • faithfulness (grounded in ctx?)
             • context recall (nothing missed?) • answer/response relevancy
             • MRR / NDCG (rank quality)        • answer correctness (vs reference)

  If the final answer is wrong, the two-layer split tells you WHICH stage failed:
    low context recall ─────► retriever missed the evidence  (fix retrieval)
    high recall + low faithfulness ─► LLM ignored/hallucinated (fix generation)
```

### The RAG Triad

```
                         (1) Context Relevance
                       does retrieved context
                       actually match the query?
                              ▲
                              │
              QUERY ──────────┼────────── CONTEXT
                │                            │
   (3) Answer   │                            │  (2) Groundedness / Faithfulness
   Relevance    │                            │  is the answer supported by
   does answer  ▼                            ▼  the retrieved context?
   address the  └────────── ANSWER ──────────┘
   query?
   ── all three must hold; each edge is a separately-scored metric ──
```

### Agent Eval: Trajectory vs Outcome

```
user goal: "book a table at a Chinese restaurant for 8pm"

TRAJECTORY (did it act correctly?)          OUTCOME (did it succeed?)
step 1: restaurant_search(cuisine=Chinese)   final state: table booked, 8pm
step 2: restaurant_book(name=..., time=8pm)         │
        │                                           ▼
        ▼                                   Agent Goal Accuracy
  Tool-Call Accuracy / Tool-Call F1         (compare end state to the
  (right tools, right args, right order)     user's intended goal)

  Trajectory can be wrong while outcome is right (took a wasteful path),
  and trajectory can look right while outcome fails — measure BOTH.
```

### Eval-Driven Development Loop

```
  ┌─────────────┐   ┌────────────┐   ┌──────────────┐   ┌────────────────┐
  │ 1. BASELINE │──►│ 2. CHANGE  │──►│ 3. MEASURE   │──►│ 4. REGRESSION- │
  │ score golden│   │ prompt /   │   │ re-run same  │   │    GUARD       │
  │ set, record │   │ retriever /│   │ golden set,  │   │ block deploy   │
  │ per metric  │   │ model      │   │ compare      │   │ if metric drops│
  └─────────────┘   └────────────┘   └──────────────┘   └────────────────┘
        ▲                                                        │
        └────────────────────── iterate ────────────────────────┘
```

---

## Key Concepts

### Why LLM Evaluation Is Hard

**What it is.** The measurement problem unique to generative systems: outputs are non-deterministic (the same input can yield different valid text) and open-ended (many phrasings are equally correct), so classification metrics like accuracy or F1 on an exact string don't apply.

**How it works mechanistically.** A classifier maps input → one of a fixed label set, so "correct" is a string/label equality you can count. A RAG or agent system produces free-form text over an effectively infinite output space; two answers can be semantically identical yet lexically different, so exact-match under-counts correct answers, while "looks fluent" over-counts them (fluent hallucinations score high to a human skim). The workaround is to *decompose* quality into narrowly-scoped, independently-verifiable properties — "is every claim supported by the context?", "does the answer address the question?" — each of which can be scored by an LLM judge or a deterministic check even when the surface text varies. This is why modern eval frameworks report a *vector* of metrics, not a scalar.

**Where it appears in real systems.** In Ragas you never get "accuracy"; you get a per-sample score for each named metric (e.g. `Faithfulness`, `ContextRecall`) in the 0–1 range, aggregated across a dataset. The non-determinism also forces you to fix `temperature=0` for the *judge* and often to average over multiple generations of the *system under test* before trusting a number.

### Retrieval Metrics (Context Precision, Context Recall, MRR/NDCG)

**What it is.** Metrics that score only the retriever — how good the set of `retrieved_contexts` is for the query — independent of what the LLM later does with them.

**How it works mechanistically.** **Context Precision** asks "of the chunks we retrieved, are the *relevant* ones ranked at the top?" — it is the mean of precision@k over the retrieved list, so a relevant chunk buried at position 5 scores lower than the same chunk at position 1 (ranking-aware). **Context Recall** asks "did we retrieve *all* the evidence needed?" — it breaks the reference answer into claims and checks what fraction of those claims are supported by the retrieved context, so it requires a reference. Classic IR metrics **MRR** (Mean Reciprocal Rank: 1/rank of the first relevant hit) and **NDCG** (Normalized Discounted Cumulative Gain: graded relevance discounted by rank position) measure the same ranking quality when you have relevance labels. Precision and recall trade off: retrieving more chunks raises recall but usually lowers precision.

**Where it appears in real systems.** Ragas exposes `ContextPrecision` (reference-based) and `ContextUtilization` (compares each chunk to the *response* when you have no reference), plus `NonLLMContextPrecisionWithReference` and `IDBasedContextPrecision` for label/ID-based scoring; `ContextRecall` uses the `reference` as a proxy for reference contexts. MRR/NDCG are what you compute directly over a labelled retrieval test set (e.g. via `ranx` or your own harness) when tuning the vector-DB `top_k`, reranker, or chunking.

### Generation Metrics (Faithfulness, Answer/Response Relevancy, Answer Correctness)

**What it is.** Metrics that score the LLM's output given the retrieved context — whether the answer is grounded, on-topic, and (if you have a reference) factually correct.

**How it works mechanistically.** **Faithfulness** (a.k.a. groundedness) = fraction of the claims in the response that can be inferred from the retrieved context; it is computed by extracting the atomic claims from the answer and checking each against the context, so an answer with a fabricated detail scores below 1.0 *even if that detail happens to be true in the world* — faithfulness is about support-by-context, not real-world truth. **Answer Relevancy** (Ragas `AnswerRelevancy` / `ResponseRelevancy`) measures whether the answer addresses the question: it reverse-generates candidate questions from the answer and takes the mean cosine similarity to the original question, penalising incomplete or padded answers — it does *not* check factual accuracy. **Factual Correctness / answer correctness** compares the response to a ground-truth `reference` (claim overlap / semantic similarity) and is the metric closest to "is this the right answer."

**Where it appears in real systems.** Ragas `Faithfulness` (with a fast classifier variant `FaithfulnesswithHHEM`), `AnswerRelevancy`, and `FactualCorrectness`. Faithfulness is the metric you gate on for hallucination-critical apps (legal, medical, finance); answer relevancy catches the "technically grounded but dodged the question" failure; factual correctness needs labels and is used offline on the golden set.

### The RAG Triad

**What it is.** A three-metric framing of end-to-end RAG quality: **context relevance** (query ↔ context), **groundedness/faithfulness** (context ↔ answer), and **answer relevance** (query ↔ answer) — the three edges of the query/context/answer triangle.

**How it works mechanistically.** Each edge isolates one failure mode. Low **context relevance** means the retriever pulled off-topic chunks (a retrieval bug). High context relevance but low **groundedness** means the LLM had good evidence but hallucinated or ignored it (a generation bug). High groundedness but low **answer relevance** means the answer is faithfully derived from context yet doesn't actually answer what was asked (a prompting/routing bug). Because the three are orthogonal, the triad turns "the answer is bad" into a specific, actionable diagnosis. Ragas expresses these as `NvidiaMetrics.ContextRelevance`, `Faithfulness`/`ResponseGroundedness`, and `AnswerRelevancy`.

**Where it appears in real systems.** It is the default dashboard for a RAG service: three time-series you watch in observability and three gates you set on the golden set. When an on-call alert fires "answers degraded," you read the triad to route the fix to the retrieval team or the prompt/model owner.

### Agent Evaluation: Trajectory (Tool-Call Accuracy) and Outcome (Goal Accuracy)

**What it is.** Agent eval adds two axes beyond RAG: **trajectory/step** evaluation (did the agent call the *right tools, with the right arguments, in the right order*?) and **final-outcome/task-success** evaluation (did the end state achieve the user's goal, regardless of path?).

**How it works mechanistically.** **Tool-Call Accuracy** (Ragas `ToolCallAccuracy`) compares the agent's actual tool calls to `reference_tool_calls`, scoring both sequence alignment and per-argument correctness; in default strict-order mode a correct set of tools in the wrong order scores 0, and it has a `strict_order=False` flexible mode for parallel calls. **Tool-Call F1** (`ToolCallF1`) is the softer, order-independent precision/recall variant that credits partial matches and penalises over-/under-calling — better for iteration. **Agent Goal Accuracy** (`AgentGoalAccuracyWithReference` / `...WithoutReference`) is a binary "was the user's goal achieved?" that inspects the conversation's end state against a reference outcome (or infers the goal when no reference is given) — it deliberately ignores *which* tools were used. **Topic Adherence** checks the agent stayed within its allowed domain. Single-step eval scores one node/decision in isolation (cheap, pinpoints the broken step); end-to-end eval scores the whole run (realistic, but a failure doesn't tell you which step broke) — you want both.

**Where it appears in real systems.** For a booking or research agent you gate trajectory with `ToolCallAccuracy` (regression-test that a prompt change didn't break tool selection) and gate outcome with `AgentGoalAccuracyWithReference`. LangSmith's `evaluate()` runs these over a dataset and its trace view lets you inspect the actual tool trajectory per example.

### Golden Datasets, Synthetic Generation, and Reference vs Reference-Free

**What it is.** The evaluation **dataset** is a curated set of test cases (a "golden set") — each an input plus, ideally, a reference answer and/or reference contexts. **Reference-based** metrics need a ground-truth label; **reference-free** metrics score using only the query/context/response.

**How it works mechanistically.** A good golden set has high-quality samples, covers real-world scenario variety, has enough samples for statistical significance, and is kept fresh to fight drift. Hand-labelling is slow, so **synthetic test-set generation** uses an LLM over your own document corpus to produce (question, reference answer, reference context) triples across difficulty types — Ragas provides testset generators for both RAG and agent/tool workflows. The reference/reference-free choice is driven by what you have: `ContextPrecision` and `FactualCorrectness` and `ContextRecall` need a `reference`; `Faithfulness`, `AnswerRelevancy`, and `ContextUtilization` are reference-free, which is exactly what lets them run **online** on live traffic where no label exists.

**Where it appears in real systems.** Offline, you version a golden dataset (in LangSmith as a `Dataset` of `examples`, each with `inputs` and reference `outputs`) and run reference-based metrics against it in CI. Online, you sample production traces and run only reference-free metrics (faithfulness, relevancy) because there's no ground truth for a live user's question. Failing production traces get promoted back into the golden set, closing the loop.

### Offline vs Online Evaluation, and Eval-Driven Development

**What it is.** **Offline eval** runs before deploy on a fixed dataset to compare versions and catch regressions; **online eval** runs on live production traffic to monitor real quality. Eval-driven development is the workflow of using these as the loop that drives every change.

**How it works mechanistically.** Offline is a controlled experiment: same inputs, same metrics, two system versions → a diff you can attribute to your change; this is where regression tests and benchmarking live. Online is uncontrolled monitoring: you attach reference-free evaluators (and sampling/rate limits for cost) to live runs to detect drift, anomalies, and safety issues you never anticipated. The methodology is **baseline → change → measure → regression-guard**: record the golden-set scores per metric *before* touching anything, make one change (prompt, chunk size, model), re-run the *identical* dataset, compare per-metric, and block the deploy (or roll back) if any gated metric regresses beyond a threshold.

**Where it appears in real systems.** LangSmith frames exactly this split — offline `evaluate()` over datasets for benchmarking/regression/unit tests, and online evaluators on production runs with configurable sampling — and its feedback loop explicitly says: add failing production traces to your dataset, build a targeted evaluator, validate the fix offline, redeploy.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Which metric to **gate** on | The single (or few) metric whose regression blocks a deploy | For zero-hallucination-tolerance apps (legal/medical/finance) gate on **faithfulness** first, then **context recall**; for open-domain helpfulness gate on **answer relevancy** + **answer correctness**. |
| Golden test-set **size** | Statistical significance of the eval; cost/latency of each run | Start with the smallest set that covers your scenario taxonomy (~50–100 diverse cases) for CI speed; expand toward hundreds+ before you trust small (<2–3 pt) score deltas as real. |
| **Reference-based vs reference-free** | Whether a metric needs ground-truth labels | Use reference-based (`ContextPrecision`, `FactualCorrectness`, `ContextRecall`) **offline** where you have a golden set; use reference-free (`Faithfulness`, `AnswerRelevancy`, `ContextUtilization`) **online** where no label exists. |
| Judge **`temperature`** | Determinism of the LLM-judge scoring the metric | Set to `0` so re-running eval on unchanged outputs gives stable scores — a noisy judge makes regression detection impossible. |
| Pass/fail **threshold** per metric | The score below which a case (or the suite) counts as a failure | Derive from the baseline distribution, not a round number: set the gate a small margin below current production so you catch regressions without blocking normal judge noise. |
| `ToolCallAccuracy` **`strict_order`** | Whether tool-call *order* must match the reference | `True` (default) for sequential workflows where order is causal (search *before* filter); `False` for independent parallel calls (fetch weather for N cities). |
| Answer-relevancy **`strictness`** | Number of questions reverse-generated per answer (default 3) | Raise for higher-variance answers where a single reconstruction is noisy; the cost is more judge/embedding calls per sample. |

### Worked Example: Requirement → Decision

**Given:** You are building a customer-facing RAG assistant over a regulated insurance policy corpus. The business requirement is *zero tolerance for hallucinated coverage claims* — an answer that invents a policy detail is a compliance incident. Answers must also actually address the customer's question, latency budget is generous (async chat), and you have a subject-matter expert who can label a few hundred question/answer/source triples but not tens of thousands.

- **Step 1 — Identify the goal:** Ship a RAG assistant with a defensible eval suite that would *catch* a hallucinated coverage claim before deploy and monitor for it in production.
- **Step 2 — Define inputs:** Customer questions; the retriever's `retrieved_contexts`; the generated `response`; an offline golden set of ~300 SME-labelled `(user_input, reference, reference_contexts)` triples, augmented with Ragas synthetic generation over the policy docs for coverage of edge cases.
- **Step 3 — Define outputs:** A per-metric score vector per sample (retrieval + generation), an aggregate gate result (pass/fail) for CI, and a live online dashboard of the RAG triad.
- **Step 4 — Apply constraints:** Hallucination is the dominant risk → faithfulness must be near-perfect; "answer looks right" is insufficient → must measure the retrieval layer separately so we can tell a missed-source failure from an ignored-source failure; only reference-free metrics can run online (no labels on live questions); judge must be deterministic.
- **Step 5 — Select the approach:** **Gate offline on `Faithfulness` (primary) + `ContextRecall` (secondary), with `AnswerRelevancy` and `FactualCorrectness` as supporting metrics**, and run **`Faithfulness` + `AnswerRelevancy` reference-free online**. *Rationale vs alternatives:* faithfulness directly targets the compliance risk (unsupported claims drop the score), and context recall guards the upstream cause (if the retriever missed the clause, the model *can't* answer faithfully); gating on answer correctness *alone* is wrong because a fluent, on-reference-sounding answer can still contain an unsupported claim; gating on a single end-to-end "looks correct" judge is wrong because a failure wouldn't tell us whether to fix the retriever or the prompt — the two-layer split is exactly what makes the failure actionable.

---

## Implementation

```python
# Scenario: Localise a RAG failure to the RIGHT stage. When a customer answer is
# wrong we must know if the retriever missed the evidence (fix retrieval) or the
# LLM ignored/hallucinated it (fix generation). So we score BOTH layers per sample,
# not one end-to-end "looks right" number.
# Metric names/APIs verified against Ragas docs (docs.ragas.io).
from ragas import evaluate, EvaluationDataset
from ragas.metrics import (
    LLMContextPrecisionWithReference,   # retrieval: ranking of relevant chunks
    LLMContextRecall,                   # retrieval: did we miss required evidence?
    Faithfulness,                       # generation: answer grounded in context?
    ResponseRelevancy,                  # generation: answer addresses the question?
)

# Each golden-set row carries the reference so retrieval + correctness are scorable.
dataset = EvaluationDataset.from_list([
    {
        "user_input": "Does my policy cover flood damage?",
        "retrieved_contexts": retriever.get_relevant_texts("flood damage coverage"),
        "response": rag_pipeline("Does my policy cover flood damage?"),
        "reference": "Standard homeowner policies exclude flood damage; it requires a separate rider.",
    },
    # ... ~300 SME-labelled + synthetic cases ...
])

result = evaluate(
    dataset=dataset,
    metrics=[
        LLMContextPrecisionWithReference(),
        LLMContextRecall(),
        Faithfulness(),        # gate metric for a zero-hallucination app
        ResponseRelevancy(),
    ],
    # judge llm/embeddings configured with temperature=0 for stable, comparable runs
)
# Reading the vector localises the bug:
#   context_recall low  -> retriever missed the flood clause      (fix retrieval)
#   recall high + faithfulness low -> LLM invented the coverage   (fix generation)
print(result)  # {'context_precision': .., 'context_recall': .., 'faithfulness': .., 'answer_relevancy': ..}
```

```python
# Anti-pattern: judge ONLY the end-to-end final answer against a tiny, anecdotal
# test set. This conflates the two failure modes (you can't tell retrieval from
# generation) AND overfits to a handful of cherry-picked examples, so a "green"
# result means nothing statistically.
def evaluate_rag_BROKEN(pipeline):
    cases = ["What is my deductible?", "Is flood covered?"]   # 2 hand-picked Qs
    for q in cases:
        answer = pipeline(q)
        looks_good = "yes" in answer.lower() or "no" in answer.lower()  # eyeball heuristic
        print(q, "PASS" if looks_good else "FAIL")   # one opaque pass/fail, no layers

# Correct approach: a proper golden set + layered metrics + a regression gate.
# Measure retrieval and generation SEPARATELY, aggregate over MANY cases, and
# block the deploy when a gated metric regresses past a threshold.
BASELINE = {"faithfulness": 0.94, "context_recall": 0.91}   # recorded from prod
GATE_MARGIN = 0.03                                          # tolerate judge noise

def evaluate_rag(dataset, metrics):
    result = evaluate(dataset=dataset, metrics=metrics).to_pandas()
    scores = {m: result[m].mean() for m in ["faithfulness", "context_recall"]}
    regressions = {
        m: scores[m] for m in scores
        if scores[m] < BASELINE[m] - GATE_MARGIN     # significant drop only
    }
    if regressions:
        raise SystemExit(f"BLOCK DEPLOY — regressed: {regressions}")
    return scores
# What breaks without this: with 2 eyeballed cases you ship a retriever change that
# quietly drops recall on the long tail; the anecdotal test stays green because it
# never covered those cases, and you learn about it from a compliance incident.
# The fix makes failures (a) statistically trustworthy and (b) attributable to a stage.
```

```python
# Scenario: Regression-test an AGENT's behaviour after a prompt change. We must
# guard TWO axes: did it call the right tools (trajectory) AND did it achieve the
# user's goal (outcome)? A prompt tweak can keep the goal met while silently
# breaking tool selection (extra/wrong calls => cost + latency blowups).
# APIs verified against Ragas agents metrics docs.
import asyncio
from ragas.metrics.collections import ToolCallAccuracy, AgentGoalAccuracyWithReference
from ragas.messages import HumanMessage, AIMessage, ToolCall, ToolMessage

conversation = [
    HumanMessage(content="Book a table at a Chinese restaurant for 8pm"),
    AIMessage(content="Searching...", tool_calls=[
        ToolCall(name="restaurant_search", args={"cuisine": "Chinese", "time": "8:00pm"})]),
    ToolMessage(content="Found: Golden Dragon, Jade Palace"),
    AIMessage(content="Booking Golden Dragon.", tool_calls=[
        ToolCall(name="restaurant_book", args={"name": "Golden Dragon", "time": "8:00pm"})]),
    ToolMessage(content="Table booked at Golden Dragon for 8:00pm."),
    AIMessage(content="Your table is booked for 8pm. Enjoy!"),
]

async def score_agent(llm):
    # Trajectory: right tools, right args, right order (strict for causal search->book).
    trajectory = await ToolCallAccuracy().ascore(
        user_input=conversation,
        reference_tool_calls=[
            ToolCall(name="restaurant_search", args={"cuisine": "Chinese", "time": "8:00pm"}),
            ToolCall(name="restaurant_book", args={"name": "Golden Dragon", "time": "8:00pm"}),
        ],
    )
    # Outcome: did the END STATE meet the goal, regardless of the path taken?
    outcome = await AgentGoalAccuracyWithReference(llm=llm).ascore(
        user_input=conversation,
        reference="Table booked at a Chinese restaurant at 8pm",
    )
    return trajectory.value, outcome.value   # gate BOTH in CI
```

---

## Common Pitfalls & Misconceptions

- **Reporting one aggregate "quality" number** — beginners want a single score like accuracy because it's tidy and comparable. But a scalar can't distinguish a retrieval miss from a generation hallucination, so it tells you nothing about *what to fix*; report a metric vector split by stage (retrieval vs generation, trajectory vs outcome) so every failure is attributable.
- **Confusing faithfulness with correctness** — people assume a "faithful" answer is a "true" answer. Faithfulness only measures whether claims are supported *by the retrieved context* — an answer can be perfectly faithful to a wrong document, or unfaithful yet accidentally true; use `FactualCorrectness` (against a reference) when you mean real-world truth, and faithfulness when you mean "no ungrounded claims."
- **Eyeballing a handful of outputs** — a few examples "looking fine" feels like evidence, and fluent text is convincing. Small anecdotal sets don't cover the scenario space and fluent hallucinations pass the eye test; you need a statistically meaningful golden set and metrics that check grounding and relevance, not vibes.
- **Only doing end-to-end eval on agents** — teams score "did the final answer look right" and stop. That hides trajectory failures — an agent can reach the goal via a wasteful or unsafe tool path (extra calls, wrong order) that blows up cost/latency; score the trajectory (`ToolCallAccuracy`/`ToolCallF1`) *and* the outcome (`AgentGoalAccuracy`) as separate axes.
- **A non-deterministic judge** — new evaluators leave the judge LLM at default temperature, then can't tell a real regression from judge noise. Run-to-run score wobble on *unchanged* outputs destroys your regression gate; fix the judge at `temperature=0` and, for high-variance systems, average the system-under-test over multiple generations before comparing.
- **No offline/online split** — beginners run eval once before launch and never again, or try to compute reference-based metrics on live traffic that has no labels. Offline (labelled, controlled, regression gate) and online (reference-free, live, drift monitor) answer different questions; you need both, and you promote failing production traces back into the offline golden set.

---

## Key Definitions

| Term | Definition |
|---|---|
| Context Precision | Retrieval metric: mean precision@k over retrieved chunks — are the relevant chunks ranked at the top? (Ragas `ContextPrecision`.) |
| Context Recall | Retrieval metric: fraction of reference-answer claims supported by the retrieved context — did we miss required evidence? Needs a reference. (Ragas `ContextRecall`.) |
| MRR / NDCG | Classic IR ranking metrics — Mean Reciprocal Rank (1/rank of first relevant hit) and Normalized Discounted Cumulative Gain (rank-discounted graded relevance). |
| Faithfulness (groundedness) | Generation metric: fraction of the response's claims that can be inferred from the retrieved context — support-by-context, *not* real-world truth. (Ragas `Faithfulness`.) |
| Answer / Response Relevancy | Generation metric: how well the answer addresses the question (mean cosine similarity of reverse-generated questions to the original); ignores factual accuracy. (Ragas `AnswerRelevancy`.) |
| Factual Correctness | Generation metric comparing the response to a ground-truth reference (claim overlap/similarity) — the metric closest to "is this the right answer." (Ragas `FactualCorrectness`.) |
| RAG Triad | The three-edge framing: context relevance (query↔context), groundedness/faithfulness (context↔answer), answer relevance (query↔answer). |
| Tool-Call Accuracy | Agent trajectory metric: do the agent's tool calls match the reference in tool, arguments, and (in strict mode) order? (Ragas `ToolCallAccuracy`.) |
| Agent Goal Accuracy | Agent outcome metric: binary — was the user's goal achieved by the end state, regardless of path? (Ragas `AgentGoalAccuracyWithReference` / `WithoutReference`.) |
| Golden dataset / test set | A curated, versioned set of test cases (input + optional reference answer/contexts) used for offline evaluation and regression gating. |
| Reference-based vs reference-free | Whether a metric requires a ground-truth label (reference-based, e.g. context precision, factual correctness) or scores from query/context/response alone (reference-free, e.g. faithfulness, answer relevancy). |
| Offline vs online eval | Offline: controlled runs on a fixed dataset before deploy (regression gate); online: reference-free evaluators on live production traffic (drift/quality monitor). |
| Eval-driven development | The loop: baseline → change → measure on the identical dataset → regression-guard (block deploy / roll back on a gated-metric drop). |

---

## Summary / Quick Recall

- LLM eval is hard because outputs are non-deterministic with no single ground truth — never report one number; report a metric *vector* split by stage.
- **Retrieval layer:** context precision (ranking), context recall (completeness, needs reference), MRR/NDCG. **Generation layer:** faithfulness (grounded?), answer relevancy (on-question?), factual correctness (right? needs reference).
- The **RAG triad** (context relevance, groundedness/faithfulness, answer relevance) turns "the answer is bad" into "the *retriever* is bad" or "the *generator* is bad."
- Faithfulness ≠ correctness: faithfulness = supported by context; correctness = matches real-world reference.
- **Agents** add two axes: trajectory (`ToolCallAccuracy`/`ToolCallF1` — right tools/args/order) and outcome (`AgentGoalAccuracy` — goal achieved); measure both, single-step *and* end-to-end.
- Build a proper **golden set** (curate + synthetic generation); pick **reference-based** metrics offline and **reference-free** metrics online.
- **Eval-driven development:** baseline → change → measure on the same dataset → regression-guard; keep a deterministic (`temperature=0`) judge and promote failing prod traces back into the golden set.

---

## Self-Check Questions

1. In Ragas terms, which metric measures whether every claim in the generated answer is supported by the retrieved context, and why is a high score on it *not* the same as the answer being factually correct?

   <details><summary>Answer</summary>

   **Faithfulness.** It is computed as the fraction of the response's atomic claims that can be inferred from the `retrieved_contexts`. A high faithfulness score only means the answer is *grounded in the context it was given* — if that context is itself wrong or incomplete, the answer can be perfectly faithful yet factually false; conversely an answer can state a real-world truth that isn't in the context and score *low*. The tempting wrong answer is "answer correctness / factual correctness" — but that metric compares to a ground-truth *reference*, not to the retrieved context, so it answers a different question ("is it true?" vs "is it supported?").

   </details>

2. Your RAG assistant returns a confident but wrong answer about a policy exclusion. You have per-sample scores for context precision, context recall, faithfulness, and answer relevancy. The trace shows **context recall ≈ 0.3 but faithfulness ≈ 0.95**. Which stage is broken and what do you fix?

   <details><summary>Answer</summary>

   The **retrieval** stage is broken — fix the retriever (chunking, `top_k`, reranker, or the embedding/query). Low context recall means the required evidence (the exclusion clause) was never retrieved, so the model literally didn't have it. The high faithfulness is consistent with this: the answer *was* faithful to the (wrong/incomplete) context it received — the model didn't hallucinate, it just had nothing to ground the exclusion on. The tempting wrong answer is "fix the prompt/model because the answer is wrong," but faithfulness ≈ 0.95 shows the generator behaved correctly given its inputs; changing the prompt won't surface a clause the retriever never returned.

   </details>

3. **Which TWO** of the following metrics can be computed *reference-free* (using only query, retrieved context, and/or response) and are therefore usable for **online** evaluation on live production traffic?
   - A. `Faithfulness`
   - B. `LLMContextRecall`
   - C. `AnswerRelevancy`
   - D. `FactualCorrectness`
   - E. `LLMContextPrecisionWithReference`

   <details><summary>Answer</summary>

   **A and C.** `Faithfulness` scores claims-in-response against retrieved context (no label needed) and `AnswerRelevancy` scores answer-vs-question via reverse-generated questions (no label needed), so both run on live traffic where the user's question has no ground truth. B (`LLMContextRecall`) needs a `reference` answer to break into claims, D (`FactualCorrectness`) compares to a ground-truth `reference`, and E is *WithReference* by name — all three are reference-based and only work offline on a labelled golden set. The most tempting distractor is B: context recall *feels* like a pure-retrieval property, but Ragas computes it using the reference as a proxy, so it is reference-based, not reference-free.

   </details>

4. A teammate proposes evaluating your booking agent with only `AgentGoalAccuracyWithReference` ("all we care about is whether the booking happened"). Under what circumstances is this insufficient, and what would you add?

   <details><summary>Answer</summary>

   It's insufficient whenever the *path* to the goal matters — cost, latency, safety, or side-effects. Goal accuracy deliberately ignores which tools were called, so an agent can achieve the booking via a wasteful or dangerous trajectory (redundant searches, calling a write tool it shouldn't, wrong order) and still score 1.0. Add **trajectory** evaluation — `ToolCallAccuracy` (strict order for causal steps like search-before-book) or `ToolCallF1` (softer, catches over-/under-calling) — so a prompt change that keeps the outcome met but breaks tool selection is caught. The tempting position — "outcome is all that matters" — holds only for a stateless, side-effect-free, cost-insensitive agent, which production booking agents are not.

   </details>

5. You want to compare two retrievers (change: swap the reranker) and decide which to ship. Compare using a **2-example eyeball test** vs an **offline golden-set regression gate**, and justify which you'd defend in a design review.

   <details><summary>Answer</summary>

   Defend the **offline golden-set regression gate**. It runs both retriever versions over the *same* labelled dataset of dozens-to-hundreds of scenario-covering cases, scoring retrieval metrics (context precision/recall, or MRR/NDCG on labels), so the score delta is (a) attributable to the reranker change and (b) statistically meaningful across the long tail. The 2-example eyeball test fails on both counts: two hand-picked cases don't cover the scenario space, and "looks better" is judge-of-one noise — you can ship a reranker that improves your two examples while regressing recall on the tail, and the anecdotal test stays green until a production incident. The only thing the eyeball test is good for is a quick smoke check; it is not a shippable decision criterion, and in a review you'd be expected to show the gated per-metric diff plus the regression threshold you set relative to the baseline.

   </details>

---

## Further Reading

- [Ragas — List of available metrics](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/) — *verified 2026-07-29* — Canonical index of RAG (context precision/recall, faithfulness, response relevancy), agent/tool, and natural-language-comparison metrics.
- [Ragas — Faithfulness](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/) — *verified 2026-07-29* — Definition and claim-decomposition mechanism of the groundedness metric, plus the fast `FaithfulnesswithHHEM` variant.
- [Ragas — Context Precision](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/) — *verified 2026-07-29* — Precision@k ranking metric, plus reference-free `ContextUtilization` and ID-based variants.
- [Ragas — Context Recall](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/) — *verified 2026-07-29* — Reference-based recall metric and why it uses the reference as a proxy for reference contexts.
- [Ragas — Response Relevancy](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/answer_relevance/) — *verified 2026-07-29* — Answer-relevancy definition, reverse-question generation, and the `strictness` parameter.
- [Ragas — Agentic or Tool use metrics](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/agents/) — *verified 2026-07-29* — `ToolCallAccuracy`, `ToolCallF1`, `AgentGoalAccuracy` (with/without reference), and `TopicAdherence`.
- [Ragas — Testset Generation](https://docs.ragas.io/en/stable/concepts/test_data_generation/) — *verified 2026-07-29* — Characteristics of an ideal test dataset and synthetic test-set generation for RAG and agent workflows.
- [LangSmith — Evaluation (offline vs online)](https://docs.smith.langchain.com/evaluation/concepts) — *verified 2026-07-29* — Offline vs online evaluation lifecycle, datasets/examples/experiments, and the failing-trace feedback loop.
- [LangSmith — How to evaluate an application](https://docs.smith.langchain.com/evaluation/how_to_guides/evaluate_llm_application) — *verified 2026-07-29* — Running `evaluate()` over a dataset with code/LLM-as-judge evaluators for benchmarking and regression tests.
