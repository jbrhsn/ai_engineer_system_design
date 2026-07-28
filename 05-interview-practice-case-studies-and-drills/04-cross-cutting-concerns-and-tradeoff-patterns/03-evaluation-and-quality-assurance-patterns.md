# Evaluation & Quality-Assurance Patterns for AI System Design

**Section:** 05 Interview Practice → Cross-Cutting Concerns & Trade-off Patterns | **Interview relevance:** High — "how do you know it works?" is the reliability axis every senior interviewer probes after the happy-path design is drawn. It forces you to reason about offline vs online evaluation, reference datasets, LLM-as-judge and its biases, RAG-specific metrics, regression gates in CI, and — above all — that evaluation is designed *before* the system is built, not bolted on after an incident.

---

## TL;DR

You cannot ship an AI system you cannot measure. Evaluation splits into **offline** (curated datasets with reference answers, run pre-deployment for benchmarking and regression testing) and **online** (live traffic without reference answers, run post-deployment for monitoring and drift detection). For RAG you decompose quality into retrieval metrics (context precision/recall) and generation metrics (faithfulness, answer relevance) so a failure points at the guilty component; LLM-as-judge fills the gap where no exact reference exists, at the cost of known biases you must control for. The whole apparatus — the golden dataset, the metrics, the pass thresholds — is defined up front from your success criteria, then wired into CI as a regression gate. **The one thing to remember: define task-appropriate metrics and a golden dataset before you build, because an eval bolted on after launch measures a system you can no longer change cheaply.**

---

## ELI5 — Explain It Like I'm 5

Think of a school teacher who writes the exam *before* teaching the class, not after. Because she wrote the questions and the answer key first, she knows exactly what "learning it" means, and every week she can re-run the same quiz to check that nobody has forgotten last month's lesson. Some questions have one right answer she can mark instantly (offline test with a key); others — like "write a good essay" — need a rubric and a careful reader, because there is no single correct string. If she only started writing quizzes after report cards were mailed, she would have no way to prove any student actually improved. Evaluating an AI system works the same way: you write the answer key and the rubric first, keep re-running the quiz on every change, and only trust a human grader (or a second AI grader with a checked rubric) for the essay-style questions.

---

## Learning Objectives

By the end of this note you will be able to:
- [ ] Distinguish offline evaluation (dataset + reference outputs, pre-deployment) from online evaluation (live runs, no reference) and choose which each situation needs.
- [ ] Decompose RAG quality into retrieval metrics (context precision/recall) and generation metrics (faithfulness, answer relevance) using Ragas so a low score localizes the fault.
- [ ] Design an LLM-as-judge evaluator and name the biases (position, verbosity, self-preference) you must control for.
- [ ] Wire a golden-dataset regression gate into CI and justify a pass/fail threshold against a stated success criterion.

---

## Visual Overview

### Offline → online evaluation loop

```
DESIGN TIME                          BUILD/CI                      PRODUCTION
success criteria                     offline eval                  online eval
   │                                    │                             │
   ▼                                    ▼                             ▼
define metrics ──► build golden ──► run on dataset ──► gate ──► deploy ──► score live runs
(faithfulness,       dataset       (has REFERENCE)   pass/     (NO reference:
 relevance, k…)   (input,ref-out)                    fail      safety, drift, judge)
   ▲                                                                    │
   └──────────────── failing live runs become new golden examples ◄─────┘
```

### RAG metric map — which score blames which component

```
                 ┌──────────── RETRIEVAL ────────────┐   ┌──── GENERATION ────┐
query ──► embed ──► vector search ──► retrieved chunks ──► LLM ──► answer
                        │                    │                        │
                        ▼                    ▼                        ▼
                 context recall        context precision         faithfulness
                 (did we fetch          (are the fetched         (is the answer
                  everything            chunks relevant &         grounded in the
                  needed?)              ranked high?)             chunks — no
                                                                  hallucination?)
                                                                        │
                                                                        ▼
                                                                answer relevance
                                                                (does it address
                                                                 the question?)
   low recall  ──► fix chunking / k / retriever
   low precision ──► fix reranking / filtering
   low faithfulness ──► fix prompt grounding / add citations
   low answer-relevance ──► fix answer prompt / task framing
```

---

## The Core Problem

LLM outputs are non-deterministic and open-ended, so "does it work?" has no compile-time answer and no single correct string to diff against. A retrieval-augmented or agentic pipeline compounds this: a wrong final answer could be caused by bad retrieval, a hallucinating generator, a mis-selected tool, or a bad prompt — and an end-to-end score alone cannot tell you which. Worse, quality can silently regress when you swap a model, edit a prompt, re-chunk a corpus, or when the world drifts under a static index. The interview question is rarely "is it accurate" — it is "what does 'good' mean for *this* task, how do you measure it component by component, and how do you catch a regression before a user does?"

Two evaluation regimes must be separated, because they see different data and answer different questions (per LangSmith's offline/online split):

- **Offline evaluation** — runs pre-deployment against a curated **dataset** of `(input, reference output)` **examples**. Because a reference exists, it can measure *correctness*: benchmarking versions, unit-testing components, and regression-testing that a change did not degrade quality.
- **Online evaluation** — runs post-deployment against production **runs/threads** that have *no* reference output. It can only measure reference-free properties: safety, format validity, drift, and reference-free LLM-as-judge quality on live traffic.

A system with a great offline suite but no online eval flies blind in production; one with only online dashboards can never prove a change is safe *before* shipping it. Mature teams run both in a loop — failing live runs get promoted into the offline golden set (links to `08-observability-and-monitoring-patterns.md`).

---

## Resolution Options

| Option | How it works | Buys you | Costs you | Reach for it when… |
|---|---|---|---|---|
| **Golden / reference dataset + code grader** | Curated `(input, reference)` examples scored by deterministic rules (exact match, JSON schema, string check) | Cheap, fast, fully repeatable pass/fail; ideal CI gate | Only works for tasks with a checkable answer; brittle to phrasing | Classification, extraction, routing, structured output |
| **RAG metrics (Ragas)** | Decompose into context precision/recall (retrieval) + faithfulness/answer-relevance (generation) | Localizes the failing component, not just "bad answer" | Most metrics need an LLM judge + sometimes a reference; cost per eval | Any RAG pipeline you must debug or regression-test |
| **LLM-as-judge** | An LLM scores outputs against a rubric encoded in its prompt (reference-free or reference-based) | Scales subjective/open-ended grading no code can express | Position/verbosity/self-preference bias; needs calibration to humans | Open-ended answers, tone, helpfulness with no exact key |
| **Human-in-the-loop review** | Reviewers grade runs via annotation queues against a rubric | Highest-signal ground truth; seeds datasets & few-shot judges | Slow, expensive, doesn't scale to all traffic | Building the golden set, calibrating judges, high-stakes domains |
| **CI regression gate** | Re-run the offline suite on every change; fail the build if a metric drops below threshold | Blocks quality regressions before deploy | Requires a stable dataset + agreed thresholds; adds CI time/cost | Any system where a prompt/model/corpus change can silently regress |
| **A/B test + online guardrail metrics** | Split live traffic between versions; compare online metrics + safety guardrails | Real-world proof one version is better on real users | Slow to reach significance; needs live traffic & telemetry | Validating a candidate that already passed offline |

**Golden / reference dataset + code grader** — you hand-curate 10–20+ `(input, reference output)` examples covering common cases and edge cases, then grade with deterministic functions. Under the hood the grader is pure code — a string comparison, regex, JSON-schema check, or exact-label match — so it is free, instant, and 100% repeatable. It appears as OpenAI's `string_check` grader (`operation: "eq"`) over an uploaded JSONL dataset, or a LangSmith **Code evaluator**. Its limit: it only works where a correct answer is checkable, so it covers classification/extraction but not "write a helpful summary."

**RAG metrics (Ragas)** — Ragas splits RAG quality into retrieval and generation metrics so a low score names the guilty stage. *Faithfulness* = fraction of the response's claims that are supported by the retrieved context (0–1); *answer/response relevance* reverse-engineers questions from the answer and measures their cosine similarity to the real question; *context precision* measures whether relevant chunks are ranked at the top; *context recall* measures whether all reference claims are covered by the retrieved chunks. Mechanically most metrics prompt an evaluator LLM (`llm_factory`, or the legacy `evaluator_llm`) to decompose text into claims and check attribution. In code these are `Faithfulness`, `ContextPrecision`, `ContextRecall`, `ResponseRelevancy` classes scored over a `SingleTurnSample` (legacy) or via `evaluate(dataset, metrics=[...])`.

**LLM-as-judge** — an LLM grades an output against a rubric written into its prompt; it is *reference-free* (checks tone, safety, helpfulness against criteria) or *reference-based* (compares to a reference answer). Under the hood the rubric plus the output become a structured-output prompt returning a boolean/categorical/continuous feedback score. It appears as a LangSmith LLM-as-a-judge evaluator or an inline "rate 1–5" prompt. The catch: judges carry biases — favoring the first option shown (position), longer answers (verbosity), and outputs from the same model family (self-preference) — so you calibrate against human labels and often add few-shot corrections.

**Human-in-the-loop review** — reviewers grade runs against a rubric in an annotation queue, and their labels become the ground truth that seeds the golden dataset and the few-shot examples that align an LLM judge. It appears as LangSmith annotation queues (single-run or pairwise) exporting annotated runs into a dataset. It is the highest-signal source but the slowest and costliest, so you spend it where it compounds: dataset creation, judge calibration, and high-stakes review.

**CI regression gate** — the offline suite is re-run on every prompt/model/corpus change and the build fails if a metric falls below an agreed threshold, converting fuzzy evaluation *metrics* into hard *tests* (LangSmith's "evaluations vs testing" distinction). It appears as a pytest/Vitest suite pinned to a tagged dataset version, or OpenAI eval runs invoked from CI. It requires a stable dataset and negotiated thresholds, and it is the single most effective guard against silent regression.

**A/B test + online guardrail metrics** — traffic is split between the current and candidate version and compared on online metrics (task-completion, thumbs-up rate) plus safety guardrails (toxicity/PII rate). Because production has no reference outputs, these are reference-free signals evaluated on live runs. It appears as online evaluators/rules over a tracing project. Use it only to confirm a candidate that already cleared the offline gate — never as the first line of defense (links to `05-reliability-and-failure-handling-patterns.md`).

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Golden dataset size | How many `(input, reference)` examples the offline suite runs on | Start with 10–20 hand-curated examples covering common + edge cases; grow from failing production runs, not synthetic bulk |
| Faithfulness pass threshold | Minimum share of answer claims that must be grounded in context | Set from the task's tolerance for hallucination — high (≥0.9) for medical/legal, lower for brainstorming; make it a hard CI gate for RAG |
| Evaluator LLM choice | Which model acts as the LLM-as-judge | Use a *different, capable* model than the one under test to reduce self-preference bias; pin its version so scores are comparable over time |
| Judge feedback type | Boolean vs categorical vs continuous score | Prefer boolean/categorical for gating decisions (repeatable); reserve continuous (1–5) for ranking/trend, calibrated to human labels |
| Answer-relevancy `strictness` | Number of questions reverse-generated from the answer (default 3) | Leave at default 3 for cost; raise only if relevance scores are noisy on short answers |
| Online eval sampling rate | Fraction of live traffic scored by online evaluators | Sample (not 100%) to bound cost; raise sampling on high-risk intents and after a deploy, lower it once stable |
| CI regression tolerance | How much a metric may drop before the build fails | Fail on any statistically meaningful drop vs the baseline experiment; allow a small tolerance band to absorb judge noise |

### Worked Example: Requirement → Decision

**Given:** A RAG assistant over an internal policy corpus. Users report it "sometimes makes things up." The team wants to fix it *and* prove future changes don't reintroduce the problem. There is currently no eval — quality is judged by eyeballing a few queries after each deploy.

- **Step 1 — Identify the goal:** Detect and gate on hallucination (ungrounded claims), and localize whether the fault is retrieval or generation.
- **Step 2 — Define inputs:** A golden dataset of `(user_input, reference answer, reference contexts)` built from real questions; each production run's `user_input`, `retrieved_contexts`, and `response`.
- **Step 3 — Define outputs:** Per-example scores for **faithfulness** and **answer relevance** (generation) and **context precision/recall** (retrieval); an aggregate that gates CI.
- **Step 4 — Apply constraints:** Open-ended answers (no exact-match key possible); must run in CI on every prompt/model/corpus change; evaluator-LLM cost must stay bounded; high tolerance requirement — this is policy guidance, so hallucination is expensive.
- **Step 5 — Select the approach:** Use **Ragas** metrics — **faithfulness** as the primary hallucination gate (fraction of claims supported by context), plus **context recall** to check retrieval actually fetched the needed policy. Wire both into a **CI regression gate** with a high faithfulness threshold. Rationale vs alternatives: a code/exact-match grader can't score open-ended answers; a bare end-to-end LLM-as-judge would flag "bad" answers but wouldn't tell you retrieval vs generation is at fault — Ragas's component split does, and faithfulness is the metric defined precisely for hallucination.

---

## Implementation

```python
# Scenario: an internal-policy RAG assistant "makes things up." We must measure
# hallucination (faithfulness) AND check retrieval coverage (context recall) so a
# low score tells us WHICH component to fix, not just "the answer was bad".
# Ragas splits RAG quality into retrieval vs generation metrics for exactly this.
from ragas import SingleTurnSample
from ragas.metrics import Faithfulness, LLMContextRecall

# A DIFFERENT, capable model judges — reduces self-preference bias vs the
# generator model under test. Pin its version so scores stay comparable.
faithfulness = Faithfulness(llm=evaluator_llm)
context_recall = LLMContextRecall(llm=evaluator_llm)

sample = SingleTurnSample(
    user_input="How many vacation days do new hires get?",
    response="New hires receive 15 vacation days in their first year.",
    retrieved_contexts=[
        "New employees accrue 15 days of paid vacation during year one."
    ],
    reference="New hires get 15 vacation days in the first year.",  # for recall
)

# faithfulness → share of answer claims grounded in retrieved_contexts (0..1)
# context_recall → share of reference claims covered by what we retrieved (0..1)
faith = await faithfulness.single_turn_ascore(sample)   # e.g. 1.0 = fully grounded
recall = await context_recall.single_turn_ascore(sample)
print(f"faithfulness={faith:.2f}  context_recall={recall:.2f}")
```

```python
# Scenario: prevent silent regressions. On every prompt/model/corpus change, CI
# re-runs the golden dataset and FAILS the build if faithfulness drops below the
# task threshold — converting a fuzzy metric into a hard test (LangSmith's
# "evaluations vs testing" idea). Run with `pytest` in the pipeline.
import statistics
import pytest
from ragas import SingleTurnSample
from ragas.metrics import Faithfulness

FAITHFULNESS_GATE = 0.90            # high bar: policy guidance, hallucination is costly
GOLDEN = load_golden_dataset()      # pinned, versioned (input, contexts, reference)

@pytest.mark.asyncio
async def test_faithfulness_regression():
    metric = Faithfulness(llm=evaluator_llm)
    scores = []
    for ex in GOLDEN:
        out = run_rag_pipeline(ex["user_input"])   # the system under test
        scores.append(await metric.single_turn_ascore(SingleTurnSample(
            user_input=ex["user_input"],
            response=out["answer"],
            retrieved_contexts=out["contexts"],
        )))
    mean = statistics.mean(scores)
    # Hard gate: block the deploy if grounding regressed below the agreed bar.
    assert mean >= FAITHFULNESS_GATE, f"faithfulness {mean:.3f} < {FAITHFULNESS_GATE}"
```

```python
# Anti-pattern: LLM-as-judge with a raw "which is better, A or B?" pairwise prompt
# and no bias controls. The judge favors whichever answer is shown first (position
# bias) and longer answers (verbosity bias), so scores drift with ordering — not quality.
verdict = judge_llm(f"Which answer is better?\nA: {answer_a}\nB: {answer_b}")  # biased!

# Corrected version: control for position bias by scoring BOTH orderings and
# requiring agreement, ground the judge in an explicit rubric, and calibrate it
# against human labels (few-shot). A disagreement across orderings flags a tie,
# not a winner — this is what breaks the naive prompt above.
def judge_pairwise(rubric, a, b):
    fwd = judge_llm(f"{rubric}\nAnswer 1: {a}\nAnswer 2: {b}\nReturn 1 or 2.")
    rev = judge_llm(f"{rubric}\nAnswer 1: {b}\nAnswer 2: {a}\nReturn 1 or 2.")
    # consistent only if the winner survives swapping positions
    if fwd == "1" and rev == "2":
        return "a"
    if fwd == "2" and rev == "1":
        return "b"
    return "tie"   # order-dependent verdict → position bias, don't trust it
```

---

## Common Pitfalls & Misconceptions

- **Bolting eval on after launch** — beginners treat evaluation as a post-incident debugging tool because the happy path "just worked" in demos; the correct model is to define success criteria, metrics, and a golden dataset *before* building, so every change is measured against a fixed bar rather than eyeballed.
- **Trusting a single end-to-end score for RAG** — a lone "answer quality" number feels sufficient, but it can't say whether retrieval or generation failed; decompose into context precision/recall (retrieval) and faithfulness/answer-relevance (generation) so a low score points at the component to fix.
- **Using LLM-as-judge as unbiased ground truth** — because an LLM grader is fast and articulate, beginners treat its scores as objective; judges have position, verbosity, and self-preference biases, so you must calibrate against human labels, swap orderings, and prefer a different model family than the one under test.
- **Confusing faithfulness with answer relevance** — both sound like "is the answer good," so they get conflated; faithfulness asks *is every claim grounded in the retrieved context* (hallucination), while answer relevance asks *does the answer address the question* — an answer can be perfectly grounded yet off-topic, or on-topic yet hallucinated.

---

## Key Definitions

| Term | Definition |
|---|---|
| Offline evaluation | Pre-deployment evaluation on a curated dataset of `(input, reference output)` examples; can measure correctness because a reference exists. |
| Online evaluation | Post-deployment evaluation on live production runs/threads with no reference output; measures reference-free properties (safety, drift, judged quality). |
| Golden / reference dataset | A curated set of examples defining "good" for a task; the fixed bar an offline suite and CI regression gate run against. |
| Faithfulness | Fraction of a response's claims that are supported by the retrieved context (0–1); the primary RAG hallucination metric. |
| Answer / response relevance | How directly the response addresses the user's question, scored by reverse-generating questions from the answer and measuring similarity to the real one. |
| Context precision | Whether relevant retrieved chunks are ranked at the top of the results (retrieval ranking quality). |
| Context recall | Whether the retrieved context covers all the claims in the reference answer (retrieval completeness). |
| LLM-as-judge | Using an LLM to score outputs against a rubric in its prompt; reference-free or reference-based, and subject to position/verbosity/self-preference bias. |
| Regression gate | A CI check that re-runs the offline suite and fails the build if a metric drops below an agreed threshold. |

---

## Summary / Quick Recall

- Evaluation is designed *up front* from success criteria — the golden dataset and metrics come before the build, not after an incident.
- Offline eval (dataset + reference, pre-deploy) measures correctness; online eval (live runs, no reference) measures safety/drift/quality on real traffic.
- For RAG, split quality into retrieval (context precision/recall) and generation (faithfulness, answer relevance) so a low score names the guilty component.
- Faithfulness = claims grounded in context (hallucination); answer relevance = addresses the question — they are different failures.
- Code/exact-match graders are for checkable answers; LLM-as-judge covers open-ended ones but carries position/verbosity/self-preference bias to control for.
- A CI regression gate turns fuzzy metrics into hard pass/fail tests and is the best guard against silent regression from a model/prompt/corpus change.
- A/B tests and online guardrails validate a candidate that already passed offline — never the first line of defense.

---

## Self-Check Questions

1. What is the core difference between offline and online evaluation, and what can each measure that the other cannot?

   <details><summary>Answer</summary>

   Offline evaluation runs pre-deployment on a curated dataset of `(input, reference output)` examples, so it can measure *correctness* against a known answer (benchmarking, regression testing). Online evaluation runs post-deployment on live production runs that have *no* reference output, so it can only measure reference-free properties (safety, format validity, drift, judged quality) — but on real traffic. Saying "offline is testing, online is monitoring" is incomplete: the defining difference is the *presence of a reference output*, which is what enables correctness checks offline and forbids them online.

   </details>

2. A RAG assistant returns a fluent, on-topic answer that states a fact the source documents never contained. Which Ragas metric catches this, and which metric would *not*?

   <details><summary>Answer</summary>

   **Faithfulness** catches it — faithfulness is the fraction of the response's claims supported by the retrieved context, so an ungrounded fact drops the score. **Answer relevance** would *not* catch it: relevance only measures whether the answer addresses the question (it reverse-generates questions from the answer and compares them to the real one), and a fluent on-topic hallucination scores high on relevance while failing faithfulness. Choosing answer relevance is the tempting error because the answer "looks good" and is on-topic.

   </details>

3. You must add a CI gate for an open-ended summarization feature that has no single correct output. Why can't you use a plain exact-match code grader, and what do you use instead — with what safeguard?

   <details><summary>Answer</summary>

   Exact-match grading fails because summaries have no single correct string — infinitely many valid summaries exist, so a code grader would fail correct outputs on phrasing alone. You use an **LLM-as-judge** with a rubric (or a reference-based judge), gated on its score. The required safeguard: calibrate the judge against human labels and control for its biases (position/verbosity/self-preference) — e.g., use a different model family than the generator and score both orderings for pairwise checks. Just swapping in an uncalibrated judge is the tempting wrong answer, because its biases make the gate unreliable.

   </details>

4. **Which TWO** of the following are genuine risks of relying on LLM-as-judge as your ground truth?
   - A. Position bias — the judge favors whichever answer is shown first
   - B. It cannot score open-ended text, only exact matches
   - C. Self-preference bias — the judge favors outputs from its own model family
   - D. It requires a reference output for every example
   - E. It is slower and costlier than a human annotation queue

   <details><summary>Answer</summary>

   **A and C.** Position bias (favoring the first-shown answer) and self-preference bias (favoring same-family outputs) are documented LLM-as-judge failure modes you must control for by swapping orderings and using a different evaluator model. B is false — the whole point of LLM-as-judge is to score open-ended text that code graders can't. D is false — LLM-as-judge can be reference-*free* (grading against a rubric alone). E is the most tempting distractor, but it is backwards: LLM judges are *faster and cheaper* than human review, which is precisely why their bias risk matters — you're scaling a biased grader.

   </details>

5. Two proposals to prevent a prompt change from silently degrading a production RAG system: (a) add an A/B test comparing the new prompt against the old on live traffic, or (b) add a golden-dataset regression gate in CI that fails the build if faithfulness drops. Which addresses the root cause of *silent pre-deploy regression*, and why?

   <details><summary>Answer</summary>

   Option (b). The stated problem is a regression slipping through *before* deploy, which is exactly what an offline regression gate blocks — it re-runs the golden dataset on the changed prompt and fails the build below the faithfulness threshold, so the bad change never reaches users. Option (a) is the tempting distractor: an A/B test only detects the regression *after* it is already serving live traffic and takes time to reach significance, so users are exposed in the meantime. A/B testing validates a candidate that already passed the offline gate — it is complementary, not a substitute (links to `05-reliability-and-failure-handling-patterns.md`).

   </details>

---

## Further Reading

- [Ragas — Faithfulness](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/) — *verified 2026-07-29* — definition and claim-decomposition formula for the primary RAG hallucination metric, plus the `Faithfulness` scorer and HHEM-based variant.
- [Ragas — Context Precision](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/) — *verified 2026-07-29* — retrieval-ranking metric (precision@k), with reference/reference-free and ID-based variants.
- [Ragas — Context Recall](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/) — *verified 2026-07-29* — retrieval-completeness metric using the reference answer as a proxy for reference contexts.
- [Ragas — Response Relevancy](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/answer_relevance/) — *verified 2026-07-29* — answer-relevance via reverse-generated questions and cosine similarity, and the `strictness` parameter.
- [LangSmith — Evaluation concepts](https://docs.smith.langchain.com/evaluation) — *verified 2026-07-29* — offline vs online evaluation, datasets/examples/experiments, evaluator techniques (human/code/LLM-as-judge/pairwise), and evaluations-vs-testing.
- [LangSmith — Define an LLM-as-a-judge evaluator](https://docs.smith.langchain.com/evaluation/how_to_guides/llm_as_judge) — *verified 2026-07-29* — configuring rubric prompts, feedback types (boolean/categorical/continuous), and few-shot human corrections to align a judge.
- [OpenAI — Working with evals](https://platform.openai.com/docs/guides/evals) — *verified 2026-07-29* — the Evals API: `data_source_config` schemas, `string_check`/graders as `testing_criteria`, and JSONL golden datasets for regression testing.
- [Anthropic — Define success criteria and build evaluations](https://docs.claude.com/en/docs/test-and-evaluate/develop-tests) — *verified 2026-07-29* — writing specific/measurable success criteria first, eval design principles, and grading methods (exact match, cosine similarity, ROUGE-L, LLM-based Likert).
