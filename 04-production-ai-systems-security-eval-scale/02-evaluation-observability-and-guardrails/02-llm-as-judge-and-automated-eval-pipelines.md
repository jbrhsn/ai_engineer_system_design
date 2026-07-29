# LLM-as-Judge and Automated Evaluation Pipelines

**Section:** 04 Production AI Systems (Security, Eval & Scale) → Evaluation, Observability & Guardrails | **Est. time:** 3 hrs | **Interview relevance:** High — "how do you know your agent got better?" is the follow-up to every design answer; you must be able to describe scoring an LLM's output with another LLM *and* how you keep that judge honest, then wire it into a deploy gate.

---

## TL;DR

**LLM-as-judge** uses a strong LLM to grade another model's outputs against a rubric — because human grading doesn't scale to thousands of daily traces. It comes in a few scoring modes (binary pass/fail, a numeric/Likert score, or pairwise "which of A/B is better"), each either **reference-based** (you have a gold answer) or **reference-free** (you don't). The catch is that the judge is *not* ground truth: it carries measurable **biases** — position, verbosity, self-preference, leniency — so you must validate it against a small human-labelled set and only trust it once judge-vs-human agreement is high. Once trusted, you wire it into an **automated eval pipeline**: an offline regression run over a golden dataset that gates deploys in CI, plus online judges sampled over live traffic to track quality over time. **The one thing to remember: an LLM judge is a cheap, scalable *approximation* of a human grader, not an objective oracle — it is only as trustworthy as its measured agreement with human labels, so calibrate it before you gate anything on it.**

---

## ELI5 — Explain It Like I'm 5

Imagine a giant school where ten thousand exams come in every day and there are only three human teachers — they can't possibly grade them all, so you hire one very smart teaching assistant and give them an **answer key and a grading rubric** ("full marks if it names the capital and spells it right"). The assistant can grade all ten thousand papers overnight, which is the whole point. But before you trust a single grade, you make the assistant re-grade fifty papers that the *real* teachers already graded, and you check: does the assistant agree with the teachers? If the assistant secretly gives higher marks to longer answers, or always favours whichever paper is on top of the pile, its grades are quietly wrong even though they *look* official. The mistake people make is thinking the assistant *is* the truth — it isn't; it's a fast stand-in whose grades you only trust *after* you've checked them against the teachers you already believe.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain the LLM-as-judge technique and choose among its scoring modes (binary, numeric/Likert, pairwise) and reference-based vs reference-free variants for a given task.
- [ ] Name the main LLM-judge biases (position, verbosity, self-preference, leniency, poor calibration) and apply the correct mitigation for each.
- [ ] Design a judge prompt/rubric (explicit criteria, few-shot anchors, chain-of-thought before the score, structured output) that produces stable, defensible scores.
- [ ] Validate a judge against a human-labelled calibration set and interpret judge-vs-human agreement as the gate on whether to trust it.
- [ ] Wire evaluation into a pipeline: an offline regression gate in CI over a versioned golden set, plus sampled online judges for production monitoring and A/B or canary comparison.

---

## Visual Overview

### LLM-as-Judge Scoring Flow

```
 model output ─┐
     input ────┼──► JUDGE PROMPT ──► JUDGE LLM ──► reasoning (CoT) ──► SCORE
 rubric ───────┤     (temp=0)          │                              (bool / 1-5
 [reference] ──┘   reference-based OR   │                               / A vs B)
                    reference-free   structured output
```

### Pointwise vs Pairwise Scoring

```
POINTWISE (grade one output)              PAIRWISE (compare two outputs)
──────────────────────────               ──────────────────────────────
  output ──► judge ──► 4/5                 output_A ─┐
                                                     ├─► judge ──► "A wins"
  absolute score against rubric            output_B ─┘   relative preference
  good for: regression thresholds          good for: model/prompt A-B choice
  weak: calibration drifts                 weak: POSITION BIAS ── fix: swap
                                                          order & average
```

### The CI/CD Offline Eval Gate

```
   git push / PR
        │
        ▼
  build candidate ──► run over GOLDEN SET ──► JUDGE scores each ──► aggregate
        (v_new)          (versioned)                                  │
                                                                      ▼
                                              mean faithfulness ≥ 0.85  AND
                                              no regression vs baseline?
                                              ├── YES ──► allow deploy
                                              └── NO  ──► block PR / fail CI
```

### Online Sampling + Async Judge + Calibration Loop

```
 live traffic ──► app answers user  (never blocked by eval)
      │
      └─(sample 10%)─► queue ──► JUDGE (async) ──► score ──► dashboard/alerts
                                    │                          │
                       low-scoring traces ──────────────┐      │ track over time
                                                         ▼      ▼
                          HUMAN reviews sample ──► measure judge-vs-human
                                                    agreement ──► trust? / retune
```

---

## Key Concepts

### LLM-as-Judge and Its Scoring Modes

**What it is.** LLM-as-judge is the technique of prompting a strong "grader" LLM to score another model's output against explicit criteria, as a scalable, cheaper substitute for human evaluation of open-ended text that has no single correct answer.

**How it works mechanistically.** You send the judge a prompt containing the task input, the candidate output, a rubric, and (optionally) a reference answer, and ask it to return a verdict. Three scoring modes dominate: **binary / pass-fail** (does the output meet a single criterion — e.g. Ragas `AspectCritic` returns 0/1 for "is this harmful?"); **numeric / Likert score** (grade on a scale, e.g. a 1–5 rubric where each level has a written description — Ragas `RubricsScore`); and **pairwise / preference** (given outputs A and B, say which is better — the MT-Bench "which assistant answered better" format). Cross-cutting these is **reference-based** (the judge compares against a gold answer, so it grades correctness) vs **reference-free** (no gold answer exists, so the judge grades intrinsic qualities like helpfulness, coherence, or — back-referencing chapter 01 — faithfulness of an answer to its retrieved context).

**Where it appears in real systems.** LangSmith exposes an "LLM-as-a-Judge Evaluator" whose feedback type you set to **Boolean**, **Categorical**, or **Continuous** (mapping exactly to binary/categorical/numeric), backed by structured output. Ragas ships `AspectCritic` (binary), `DiscreteMetric`/`RubricsScore` (numeric rubric), and `InstanceRubrics` (per-item rubric). Pairwise mode is LangSmith's `evaluate(..., randomize_order=True)` over two experiments.

### Judge Biases and Their Mitigations

**What it is.** LLM judges exhibit systematic, measurable biases that skew scores independent of true output quality — the reason a judge cannot be treated as ground truth.

**How it works mechanistically.** The "Judging LLM-as-a-Judge" (MT-Bench) paper documents the core failure modes: **position bias** (in pairwise mode the judge favours the answer shown first — or sometimes second — regardless of quality); **verbosity bias** (the judge prefers longer, more elaborate answers even when they add no correctness); and **self-enhancement / self-preference bias** (a judge tends to rate outputs from *its own* model family higher). Two more show up in practice: **leniency** (judges skew toward passing / high scores, compressing the usable range) and **poor calibration** (a "7/10" is not stably a 7 — the numeric scale is noisy run-to-run). Mitigations map one-to-one: for position bias, **randomize or swap the A/B order and average** (or count a win only if it holds both ways); for verbosity, instruct the judge explicitly to ignore length and score only substance; for self-preference, use a **different, strong judge model** than the one under test and/or an **ensemble of judges**; for leniency and calibration, use a **detailed rubric with anchored level descriptions**, **few-shot anchor examples**, and force **chain-of-thought reasoning before the score**; and above all, measure the residual error against a **human calibration set**.

**Where it appears in real systems.** LangSmith's pairwise `randomize_order` flag exists specifically as a position-bias mitigation (its docs note it "should mainly be addressed via prompt engineering" — the built-in pairwise prompt literally instructs "Avoid any position biases … Do not allow the length of the responses to influence your evaluation. Do not favor certain names"). Ragas `AspectCritic` collects **multiple LLM verdicts and takes a majority vote** — an ensemble mitigation against single-call noise.

### Judge Prompt and Rubric Design

**What it is.** The judge prompt is the interface that determines score quality: the rubric (criteria and, for numeric modes, a description of each score level), the reasoning instruction, and the required output format.

**How it works mechanistically.** A vague prompt ("rate 1–10") gives the model no anchor, so it falls back on its biases and produces noisy, uncalibrated scores. A good prompt does four things: (1) states **explicit criteria** and, for numeric scoring, a **written description per level** (Ragas rubrics define `score1_description` … `score5_description`) so the scale is grounded; (2) supplies **few-shot anchor examples** of already-graded outputs to pin the judge's scale to human preference; (3) requires **chain-of-thought before the score** ("first compare and explain, *then* output the verdict") because reasoning-first measurably improves judgment; and (4) demands **structured output** (a JSON field / tool call) so the score is parseable and can't hide inside prose. Setting **temperature = 0** removes sampling noise so the same input yields the same score.

**Where it appears in real systems.** LangSmith's evaluator config is exactly this: a **prompt** (rubric text) + **feedback configuration** (the scoring key and type) implemented as **structured output** on the judge model; it also supports inserting **human corrections as few-shot examples** automatically. The MT-Bench template shows the CoT-before-verdict pattern: "Begin your evaluation by … a short explanation … then output your final verdict."

### Offline Evaluation as a CI/CD Gate

**What it is.** An offline eval runs the candidate system over a fixed, curated **golden dataset** *before* release; wiring it into CI makes a quality threshold a hard gate on merging or deploying.

**How it works mechanistically.** You keep a versioned dataset of representative inputs (and, where possible, reference outputs). On each PR/build, CI runs the new version over the dataset, the judge scores every example, and the results are aggregated (mean score per metric, plus a **regression check** comparing against the current production baseline). A pass/fail rule — e.g. "mean faithfulness ≥ 0.85 **and** no metric regresses more than 2 points vs baseline" — decides whether the pipeline proceeds. This is a **regression test** for probabilistic systems: it catches the prompt tweak that silently degraded correctness the same way a unit test catches a broken function.

**Where it appears in real systems.** LangSmith frames offline evaluation as "test before you ship": create a dataset → define evaluators → run an **experiment** → analyze for **regression tests / benchmarking / unit tests**. Because `evaluate()` is a plain Python call returning per-example scores, you invoke it from a CI job (GitHub Actions, etc.) and fail the build when the aggregate falls below threshold.

### Online / Production Evaluation and Sampling

**What it is.** Online eval scores *real* production traffic in near-real-time to monitor live quality, detect regressions the golden set didn't cover, and surface bad traces for review — without a reference answer, so judges here are reference-free.

**How it works mechanistically.** The application answers the user first and is **never blocked** by evaluation; a **sampled** fraction of traces (to control cost) is sent to the judge **asynchronously** (e.g. via a queue or a background rule) and the resulting score is attached as feedback to the trace. You then track the metric over time on a dashboard, alert on drops, and feed low-scoring traces into a human-review / dataset-growth loop. Sampling rate is the main cost lever: 100% is expensive at scale; 5–10% usually gives a stable trend.

**Where it appears in real systems.** LangSmith online evaluators run on production traces with a configurable **sampling rate** (`0.1` = 10% of filtered runs), optional **filters** (only enterprise plans, only runs with a thumbs-down), and a **spend limit** cap. Its recommended feedback loop is explicit: add failing production traces to your dataset → build targeted evaluators → validate fixes offline → redeploy.

### Eval Dataset Versioning

**What it is.** The golden/eval dataset is a first-class, **versioned** artifact of test cases (inputs, optional reference outputs, metadata) that the pipeline grades against.

**How it works mechanistically.** Datasets are built from curated hand-written cases, replayed historical production traces, and synthetic generation. Because a metric number is only meaningful *relative to a fixed dataset*, the dataset must be versioned: if you silently add or change examples, a score change no longer tells you whether the *system* changed or the *test* changed. New failure modes found in production are promoted into the dataset (with a new version) so the same bug can never regress unnoticed again.

**Where it appears in real systems.** LangSmith datasets are composed of **examples** (inputs + reference outputs), can be built from production traces or synthetic data, and are the fixed target an experiment runs against; Ragas represents the same as an `EvaluationDataset` of `SingleTurnSample`/`MultiTurnSample` records.

### Judge–Human Agreement and Calibration

**What it is.** The single validity check on the whole approach: how often the judge's verdict matches a trusted **human** label on a held-out calibration set.

**How it works mechanistically.** You reserve a small set of examples graded by humans, run the judge over the *same* set, and compute an agreement metric (percentage agreement, or a correlation/Cohen's-κ style statistic for graded scores). If agreement is high, the judge is a trustworthy stand-in and you can scale it to thousands of ungraded traces; if it's low, the judge (or its rubric) is broken and any pipeline built on it is measuring noise. The MT-Bench result is the benchmark to cite: a strong judge (GPT-4 in the paper) reached **over 80% agreement with human preferences — the same level of agreement humans reach with each other** — which is exactly what *justifies* using a judge as a scalable approximation of human preference.

**Where it appears in real systems.** LangSmith supports collecting **human corrections** on evaluator scores and turning them into few-shot examples — a concrete calibration loop that both measures and improves agreement. In interviews, "how do you validate the judge?" is the question that separates people who treat the LLM as an oracle from those who treat it as an instrument that must be calibrated.

### A/B and Canary Comparison

**What it is.** Deciding whether a new system version is actually better than the incumbent, either offline (two experiments compared) or online (a fraction of live traffic routed to the candidate).

**How it works mechanistically.** **Pairwise/A-B offline:** run both versions over the golden set and use a *pairwise* judge to pick a winner per example, aggregating to a win-rate — more sensitive than comparing two absolute pointwise means because it sidesteps calibration drift. **Canary online:** route a small % of live traffic to the candidate, run reference-free judges on both arms, and promote only if the candidate's live score holds up. Position bias is the danger in the offline pairwise step, so you randomize A/B order.

**Where it appears in real systems.** LangSmith's `evaluate((exp1, exp2), evaluators=[...], randomize_order=True)` (and `evaluate_comparative` for >2) runs pairwise comparison over existing experiments and renders a "Pairwise Experiments" comparison view; the same sampled-online-judge machinery scores a canary arm.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Judge model strength | The grader LLM's quality/cost | Use a model **at least as strong** as (and ideally different-family from) the system under test; a weak judge grading a strong model produces unreliable, self-preference-tainted scores. Never let a model judge its own family in a high-stakes A/B. |
| Judge `temperature` | Sampling randomness of the judge | Set to **0** — you want the same input to yield the same score; any nonzero temperature adds run-to-run noise that masquerades as quality change. |
| Scoring mode (pointwise vs pairwise) | Absolute score vs relative preference | **Pointwise** when you need a fixed threshold to gate deploys or track over time; **pairwise** when choosing between two candidates (model/prompt A-B), because relative preference dodges calibration drift. |
| Reference-based vs reference-free | Whether a gold answer is provided | **Reference-based** for offline correctness on your golden set (you have expected answers); **reference-free** for online production traffic (no gold answer exists — grade faithfulness/helpfulness instead). |
| Number of judges / ensemble | How many judge calls per item | Single call for cheap, high-volume online monitoring; **ensemble + majority vote** (Ragas `AspectCritic` does 3) for high-stakes or borderline gating decisions to cut single-call variance. |
| CI pass threshold + regression margin | The deploy gate rule | Set an **absolute floor** (e.g. mean ≥ 0.85) **and** a **relative no-regression margin vs the production baseline**; a floor alone lets slow drift through, a relative check alone lets a low-but-stable score ship. |
| Online sampling rate | Fraction of live traces judged | Start at **5–10%** for a stable trend at bounded cost; raise for low-volume/high-risk flows, and filter to the segments you actually care about (e.g. only paid users) before sampling. |

### Worked Example: Requirement → Decision

**Given:** You run a customer-support bot on FastAPI. Product wants a guarantee that no prompt or model change ships if it makes answers less faithful to the retrieved help-centre docs, and they want to know if quality drifts in production. You have ~200 historical questions, and two support engineers can each spend two hours labelling.

- **Step 1 — Identify the goal:** Two things: (a) a **deploy gate** that blocks changes which reduce faithfulness on a fixed test set, and (b) **online monitoring** of live faithfulness with alerting on drops.
- **Step 2 — Define inputs:** A versioned **golden set** of the 200 questions with reference answers + the retrieved context per question; a judge model; the two engineers' labels on a ~50-item **calibration subset**.
- **Step 3 — Define outputs:** Offline: a per-example faithfulness score (numeric rubric) → aggregate → pass/fail. Online: a sampled faithfulness score attached to each judged production trace, plotted over time.
- **Step 4 — Apply constraints:** The judge must be **validated** (engineer time is scarce, so calibrate on 50 items, not all 200); offline eval must be **deterministic** enough to gate CI (temp=0); online eval must **not add latency** to user responses and must be **cost-bounded**.
- **Step 5 — Select the approach:** Offline, use a **reference-based, pointwise numeric-rubric judge** (temp=0, CoT-before-score, structured output) run in CI over the versioned golden set, gating on "mean faithfulness ≥ baseline − 2 pts." First **validate it** against the 50 human-labelled items and only enable the gate once judge-vs-human agreement is high. Online, use a **reference-free faithfulness judge sampled at 10%, run async**, with a dashboard + alert. *Rationale vs alternatives:* a *pairwise* judge is wrong for the gate because you need an absolute threshold to track over time, not just "is v2 > v1"; skipping calibration is the classic trap — an un-validated judge could be lenient or verbosity-biased and would wave through regressions while looking authoritative.

---

## Implementation

```python
# Scenario: Gate deploys in CI. Before every release we must confirm the support bot's
# answers stay faithful to retrieved docs on a versioned golden set. We use a numeric
# rubric judge (temp=0, anchored levels) so the score is stable and thresholdable.
# API verified against Ragas RubricsScore docs (docs.ragas.io).
from ragas import evaluate, EvaluationDataset
from ragas.metrics import RubricsScore
from ragas.llms import llm_factory
from openai import AsyncOpenAI

judge_llm = llm_factory("gpt-4o", client=AsyncOpenAI())  # strong, different family from bot

# Anchored rubric: every level has a written description -> calibrated, not a vague 1-10.
faithfulness_rubric = {
    "score1_description": "The answer makes claims that contradict or are absent from the retrieved context.",
    "score3_description": "The answer is mostly grounded but includes one unsupported detail.",
    "score5_description": "Every claim in the answer is directly supported by the retrieved context.",
}
scorer = RubricsScore(rubrics=faithfulness_rubric, llm=judge_llm)

golden = EvaluationDataset.from_list(load_golden_set("golden_v3.jsonl"))  # VERSIONED
result = evaluate(dataset=golden, metrics=[scorer], llm=judge_llm)

mean_score = result["rubrics_score"]
BASELINE, MARGIN = 4.30, 0.10   # current prod baseline on the same golden version
if mean_score < BASELINE - MARGIN:
    raise SystemExit(f"CI FAIL: faithfulness {mean_score:.2f} regressed vs {BASELINE}")
print(f"CI PASS: faithfulness {mean_score:.2f}")
```

```python
# Anti-pattern: a naive, high-temperature, rubric-less judge that is NEVER validated
# against humans. "Rate 1-10" gives the model no anchor, temp>0 adds run-to-run noise,
# and with no human check you have no idea if the number means anything -> you end up
# gating deploys on scores that are noisy, lenient, and verbosity-biased.
def naive_judge(answer: str) -> int:                       # BROKEN
    resp = client.chat.completions.create(
        model="gpt-4o", temperature=0.9,                   # noise: same input -> different score
        messages=[{"role": "user",
                   "content": f"Rate this answer 1-10:\n{answer}"}],  # no rubric, no CoT, no reference
    )
    return int(resp.choices[0].message.content)            # unparseable prose crashes often

# Correct approach: temp=0, explicit anchored rubric, CoT-before-score, structured output,
# AND a human-agreement check that must pass before this judge is allowed to gate anything.
import json
def judge(answer: str, context: str) -> dict:
    resp = client.chat.completions.create(
        model="gpt-4o", temperature=0,                     # deterministic
        messages=[{"role": "system", "content":
            "You grade answer faithfulness to CONTEXT. Ignore answer length. "
            "1=contradicts/unsupported, 3=one unsupported detail, 5=fully supported. "
            "First write a short reasoning, THEN the score."},                # rubric + CoT
            {"role": "user", "content": f"CONTEXT:\n{context}\n\nANSWER:\n{answer}"}],
        response_format={"type": "json_object"},           # structured -> parseable
    )
    return json.loads(resp.choices[0].message.content)     # {"reasoning": "...", "score": 4}

def validate_judge(human_labelled: list[dict]) -> float:
    # Judge-vs-human agreement is the gate on trusting the judge AT ALL.
    agree = sum(abs(judge(x["answer"], x["context"])["score"] - x["human_score"]) <= 1
                for x in human_labelled)
    return agree / len(human_labelled)                     # require e.g. >= 0.8 before use

# What breaks without this: the naive judge produces authoritative-looking but noisy,
# uncalibrated numbers; you ship regressions because the "score" never tracked real quality.
```

```python
# Scenario: Choose between prompt v1 and v2 offline with a PAIRWISE judge. Naive fixed
# A/B order makes the judge favour whichever answer is shown first (position bias),
# so a worse prompt can "win" just by being placed first.
from langsmith import evaluate

def ranked_preference(inputs: dict, outputs: list[dict]) -> dict:
    verdict = pairwise_chain.invoke({"question": inputs["question"],
                                     "answer_a": outputs[0]["answer"],
                                     "answer_b": outputs[1]["answer"]})
    scores = {0: [0, 0], 1: [1, 0], 2: [0, 1]}[verdict["Preference"]]
    return {"key": "ranked_preference", "scores": scores}

# Correct: randomize_order=True so A/B placement is shuffled per example and the
# position-bias contribution averages out. (LangSmith exposes this flag for exactly this.)
evaluate(("prompt-v1", "prompt-v2"),
         evaluators=[ranked_preference],
         randomize_order=True)                             # position-bias mitigation
# What breaks without it: with a fixed order the win-rate reflects placement, not quality.
```

---

## Common Pitfalls & Misconceptions

- **Treating the judge as ground truth** — the judge's output looks authoritative and comes from a "smarter" model, so people assume it's objective. It's a biased *approximation* of a human grader; its numbers mean nothing until you've measured agreement against human labels, so always calibrate before you gate.
- **Vague "rate 1–10" prompts** — beginners think a bigger model doesn't need a rubric. Without anchored level descriptions the model falls back on verbosity/leniency biases and its scale drifts run-to-run; a written description per score level plus few-shot anchors is what makes the scale mean something.
- **Fixed A/B order in pairwise mode** — it feels neutral to always show candidate A first. Position bias means the judge favours a slot regardless of quality, so a worse output wins by placement; randomize/swap the order and average (LangSmith's `randomize_order`).
- **Judge from the same model family as the system under test** — reusing the same model for generation and grading seems convenient and cheap. Self-enhancement bias inflates scores for its own family; use a different strong judge (or an ensemble) whenever the result influences a real decision.
- **Nonzero judge temperature** — people copy the app's `temperature` into the judge. Sampling noise then makes identical outputs get different scores, which reads as spurious "quality change"; judges run at `temperature=0` for reproducibility.
- **Blocking the user on evaluation** — treating the judge as an inline validation step adds an extra LLM call to every response's latency. Online eval must run **async on a sample** so the user is never slowed and cost stays bounded; the app answers first, the judge grades later.
- **Un-versioned eval datasets** — quietly editing the golden set to "add a case" breaks comparability. A score only means something relative to a fixed dataset version, so version datasets and promote new production failures as new versions.

---

## Key Definitions

| Term | Definition |
|---|---|
| LLM-as-judge | Using a strong LLM to score another model's output against a rubric, as a scalable substitute for human grading. |
| Pointwise scoring | Grading a single output on an absolute scale (binary or numeric) against a rubric. |
| Pairwise scoring | Presenting two outputs and having the judge pick the better one (relative preference). |
| Reference-based / reference-free | Whether the judge is given a gold answer to compare against (correctness) or must grade intrinsic qualities without one (e.g. faithfulness, helpfulness). |
| Position bias | A judge's tendency to favour an answer by its position (e.g. shown first) in pairwise comparison, independent of quality. |
| Verbosity bias | A judge's tendency to prefer longer/more elaborate answers even when they add no correctness. |
| Self-enhancement / self-preference bias | A judge's tendency to score outputs from its own model family more favourably. |
| Calibration | How well the judge's scores map to true quality on a stable scale; poor calibration means a "7" isn't reliably a 7. |
| Judge–human agreement | The rate at which judge verdicts match trusted human labels on a held-out set; the validity check on the whole approach. |
| Golden / eval dataset | A versioned set of test cases (inputs, optional reference outputs) the pipeline grades against. |
| Offline eval (regression gate) | Running the candidate over the golden set before release and blocking deploy if a threshold/regression check fails. |
| Online eval | Scoring sampled live production traffic asynchronously to monitor quality over time. |

---

## Summary / Quick Recall

- LLM-as-judge trades human grading for a scalable LLM grader; modes are **binary / numeric-rubric / pairwise**, each **reference-based** or **reference-free**.
- The judge is **not ground truth** — it has **position, verbosity, self-preference, leniency, and calibration** biases; each has a specific mitigation.
- Mitigations: explicit **anchored rubric**, **few-shot anchors**, **CoT before score**, **temp=0**, **randomize pairwise order**, **different/ensemble judges**, and — above all — **validate against a human set**.
- **Judge–human agreement** is the gate on trusting the judge at all; MT-Bench showed a strong judge can hit **~80%+ agreement, matching human–human agreement**.
- **Offline eval** runs over a **versioned golden set** in CI and **gates deploys** (absolute floor + no-regression margin); it's a regression test for probabilistic systems.
- **Online eval** runs **async on a sampled % (≈5–10%)** of live traffic (reference-free), never blocking the user, feeding a dashboard and a human-review loop.
- **A/B/canary**: pairwise judge (order randomized) offline, or a sampled canary arm online, to decide if a new version is genuinely better.

---

## Self-Check Questions

1. Name the three main LLM-as-judge scoring modes and state what distinguishes reference-based from reference-free scoring.

   <details><summary>Answer</summary>

   The three modes are **binary/pass-fail** (meets a criterion or not), **numeric/Likert** (a score on a scale against a rubric), and **pairwise/preference** (which of two outputs is better). **Reference-based** means the judge is given a gold/expected answer and grades correctness against it; **reference-free** means no gold answer exists, so the judge grades intrinsic qualities such as faithfulness or helpfulness. The tempting error is to call pairwise "reference-based" — pairwise is orthogonal to references (it compares two candidates and usually has no gold answer at all).

   </details>

2. You're setting up an *online* judge on live support-bot traffic and want it cheap and non-disruptive. What two configuration choices do you make and why?

   <details><summary>Answer</summary>

   Run the judge **asynchronously on a sampled fraction** of traffic (e.g. `sampling_rate=0.1` for 10%), and make it **reference-free** (production traffic has no gold answer). Async + sampling means the user's response is never blocked by an extra LLM call and cost stays bounded while still giving a stable quality trend. The tempting wrong answer is "judge 100% of traffic inline for full coverage" — that adds an LLM call to every user response's latency and multiplies cost, and full coverage is unnecessary when a 5–10% sample already tracks the trend.

   </details>

3. **Which TWO** of the following are correct about validating and de-biasing an LLM judge?
   - A. Judge–human agreement on a held-out labelled set is the check that determines whether the judge can be trusted to grade unlabelled data.
   - B. Randomizing the A/B order in pairwise comparison mitigates position bias.
   - C. Raising the judge's temperature improves score reliability by exploring more options.
   - D. A judge from the same model family as the system under test gives the most objective scores.
   - E. A vague "rate 1–10" prompt is sufficient as long as the judge model is strong.

   <details><summary>Answer</summary>

   **A and B.** A is correct because a judge is only a valid instrument once its verdicts demonstrably match trusted human labels — that agreement is the gate on scaling it to ungraded traces. B is correct because position bias attaches to a slot, so shuffling A/B placement per example averages it out (LangSmith's `randomize_order`). C is false — nonzero temperature *adds* run-to-run noise, so judges run at temp=0. D is the tempting distractor and is false: a same-family judge suffers **self-preference bias**, inflating its own family's scores; use a different/ensemble judge. E is false: without an anchored rubric even a strong model defaults to verbosity/leniency bias and drifts.

   </details>

4. Your CI faithfulness gate is set to "mean score ≥ 0.85." Over three months the mean slowly slides from 0.94 to 0.86 while always passing, and users complain quality dropped. What's wrong with the gate and how do you fix it?

   <details><summary>Answer</summary>

   An **absolute floor alone** permits slow drift: every release stayed above 0.85, so nothing was ever blocked even though quality fell 8 points. Add a **relative no-regression margin against the current production baseline** (e.g. "must not regress more than 2 points vs the deployed version"), so each incremental degradation is caught at merge time regardless of the absolute floor. The tempting wrong answer is "lower the threshold" or "the judge is broken" — the judge may be fine; the *gate rule* is missing the relative check that catches cumulative drift. (Assumes the golden-set version is held fixed, else the comparison is meaningless.)

   </details>

5. A colleague proposes using your production chat model (model X) to also judge model X's own answers in a numeric-rubric offline eval, arguing it's cheaper than adding a second model. Evaluate the trade-off.

   <details><summary>Answer</summary>

   It's a **poor choice for a decision-influencing eval**. Using model X to grade model X invites **self-enhancement/self-preference bias** — the judge systematically over-rates outputs from its own family — so the scores are inflated and can't be trusted to gate deploys or compare against a competing model. The cost saving is real but small next to the risk of shipping regressions on biased scores. The fix is a **different, at-least-as-strong judge model** (or an ensemble/majority vote), and regardless, **validate judge–human agreement** first. The tempting counter — "same model is consistent, so it's fine" — confuses *consistency* (low variance) with *validity* (matching truth); a consistently biased judge is consistently wrong.

   </details>

---

## Further Reading

- [Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena (Zheng et al., 2023)](https://arxiv.org/abs/2306.05685) — *verified 2026-07-29* — Primary source for LLM-judge position/verbosity/self-enhancement biases and the ~80% judge-vs-human agreement result that justifies the technique.
- [LangSmith — Define an LLM-as-a-judge evaluator](https://docs.langchain.com/langsmith/llm-as-judge) — *verified 2026-07-29* — Configuring a judge's prompt/rubric, Boolean/Categorical/Continuous feedback via structured output, and few-shot human corrections.
- [LangSmith — Evaluation overview (offline vs online)](https://docs.langchain.com/langsmith/evaluation) — *verified 2026-07-29* — Offline "test before you ship" regression gating vs online production monitoring, and the failing-trace feedback loop.
- [LangSmith — Set up LLM-as-a-judge online evaluators](https://docs.langchain.com/langsmith/online-evaluations-llm-as-judge) — *verified 2026-07-29* — Sampling rate, run filters, and spend limits for scoring live production traffic.
- [LangSmith — Run a pairwise evaluation](https://docs.langchain.com/langsmith/evaluate-pairwise) — *verified 2026-07-29* — `evaluate(..., randomize_order=True)` for A/B preference comparison and position-bias mitigation.
- [Ragas — General Purpose Metrics (AspectCritic, RubricsScore, InstanceRubrics)](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/general_purpose/) — *verified 2026-07-29* — Binary critic (majority-vote ensemble), numeric rubric scoring with anchored level descriptions, and per-instance rubrics.
