# CI/CD and Deployment Patterns for AI Services

**Section:** 04 Production AI Systems: Security, Eval & Scale → Scaling, Cost & Deployment for AI Systems | **Est. time:** 3 hrs | **Interview relevance:** Medium-High — you will be asked "how do you ship a prompt/model change safely?"; the strong answer treats deployment as an eval-gated, versioned, reversible process, not a code push where green tests mean it's safe.

---

## TL;DR

Shipping an LLM or agentic service is not like shipping normal software: the artifacts you deploy — prompts, model versions, tool schemas, retrieval/embedding configs — are **non-deterministic**, so "unit tests pass" says almost nothing about whether output *quality* held up. The production discipline is to treat every one of those artifacts as a **pinned, versioned** thing, put a **regression eval on a golden set** in the CI gate (a change can't promote unless quality clears a threshold — ties to the eval chapter), and roll out through **progressive delivery** (shadow → canary → full) with **automatic rollback** when quality, cost, or latency regress. Two decoupling moves make this fast and reversible: keep the service **stateless** (state lives in the checkpointer/store) so any version is safe to kill, and hot-swap prompts/model pins through a **registry or feature flag** so a prompt change doesn't require a code redeploy. **The one thing to remember: an AI deploy is an eval-gated, versioned, reversible change to prompts/models/tools/indexes — "the code compiles and tests pass" is necessary but nowhere near sufficient, because behavior lives in artifacts that pass no compiler.**

---

## ELI5 — Explain It Like I'm 5

Imagine a restaurant kitchen where the recipe card is the *prompt* and the chef is the *model*. Changing a single word on the recipe card — "a pinch of salt" to "a spoonful" — can change the whole dish, even though nobody touched the oven or the plates (the code). So before that new card reaches customers, you cook a small test batch and have a trusted taster (the eval gate) approve it against a tray of reference dishes you know should taste right; if the taste score drops, the card never leaves the kitchen. When it does pass, you don't put it on every table at once — you serve one table first (a canary), watch their faces, and keep the old recipe card clipped right next to the stove so you can switch back in seconds. The mistake beginners make is thinking "the plates didn't break, so the food is fine" — but a recipe (prompt) can pass every mechanical check and still taste wrong, which is exactly why you need a taster and a one-table trial, not just a working kitchen.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain what is *different* about deploying LLM/agentic services versus deterministic software, and why passing unit tests is insufficient.
- [ ] Design an **eval-in-CI gate** that runs a regression eval on a golden set and blocks promotion below a quality threshold.
- [ ] Pin and version every behavior-bearing artifact — prompt, model snapshot, tool schema, index/embedding version — and justify why floating aliases are dangerous.
- [ ] Compare progressive-delivery strategies (shadow, canary, blue-green, A/B) and define concrete rollback triggers on quality, cost, and latency.
- [ ] Decouple prompt/model changes from code deploys via a prompt registry, and design a stateless, containerized FastAPI agent service with health checks that survives model deprecations.

---

## Visual Overview

### The AI CI/CD Pipeline (eval-gated promotion)

```
 commit ──► lint + unit tests ──► build image ──► REGRESSION EVAL on golden set
   │            (deterministic)      (artifact)     │
   │                                                ▼
   │                                     score ≥ threshold?
   │                                     ├── NO  ──► BLOCK deploy, alert
   │                                     └── YES ──► deploy to STAGE
   │                                                     │
   │                                                     ▼
   │                                          SHADOW (mirror prod traffic,
   │                                          compare, 0% user impact)
   │                                                     │
   │                                                     ▼
   │                                          CANARY 5% ──► watch quality/cost/latency
   │                                                     ├── regress ──► ROLLBACK
   │                                                     └── healthy ──► promote 100%
```

### Shadow vs Canary Traffic

```
SHADOW (no user impact)                 CANARY (real users, small %)
──────────────────────                  ────────────────────────────
 user request                            user request
      │                                       │
      ├──────────► v1 (prod) ──► RESPONSE      ├─ 95% ─► v1 (prod) ──► RESPONSE
      │                to user                │              to user
      └··(mirror)··► v2 (new)  ─► logged,      └─  5% ─► v2 (new)  ──► RESPONSE
                                 compared,                    to user (watched)
                                 discarded            rollback if metrics regress
```

### Prompt-Registry Hot-Swap (decoupled from code deploy)

```
   CODE DEPLOY PATH                    PROMPT/CONFIG PATH (no redeploy)
   ────────────────                    ───────────────────────────────
   git push                            edit prompt in registry
      │                                     │  new commit hash: a1b2c3
      ▼                                     ▼
   build + eval gate                   eval gate on the prompt version
      │                                     │
      ▼                                     ▼
   deploy container                    flip pointer  prod ──► "a1b2c3"
   (minutes, restarts pods)            (service pulls new version, no rebuild)
```

### Version-Pinning Map (the moving parts)

```
                    ┌─────────────────────────────────────────┐
   one "release" =  │ prompt commit   : joke-gen:a1b2c3        │
   a pinned tuple   │ model snapshot  : claude-opus-4-8        │  (NOT an alias)
   of ALL of these  │ tool schema ver : tools_v3              │
                    │ index/embed ver : idx_2026_07 / emb-3   │
                    │ app image tag   : svc:1.4.2             │
                    └─────────────────────────────────────────┘
   change ANY one ──► new release ──► must clear the eval gate
```

---

## Key Concepts

### Why AI Deploys Differ (non-determinism breaks "tests pass = safe")

**What it is.** The recognition that an AI service's *behavior* is produced by artifacts a compiler never checks — a prompt string, a model version, a tool description, a retrieval config — and that identical inputs can yield different outputs, so traditional pass/fail unit testing under-measures the risk of a change.

**How it works mechanistically.** A normal deploy is safe-ish because the code is deterministic: given inputs, the transformation is fixed, so a passing test suite covers the behavior. In an LLM service the transformation is a sampled token distribution conditioned on a prompt and a model's weights; changing the prompt wording, swapping a model snapshot, or re-chunking the index shifts that distribution *without changing a line of application code*. You can therefore have 100% green unit tests (routing works, JSON parses, the API returns 200) while answer quality, faithfulness, or tool-selection accuracy silently drops. The only way to detect that is to *run the system on representative inputs and score the outputs* — i.e. an eval — which is why eval, not compilation, is the real gate.

**Where it appears in real systems.** Concretely: your FastAPI `/chat` endpoint's integration test asserts a 200 and a well-formed body — it will happily pass after you swap `claude-opus-4-8` for a newer snapshot that answers differently, or after a teammate "improves" a system prompt. The failure surfaces later as a spike in user thumbs-down or a faithfulness-metric drop in production observability. LangSmith names this exact category **regression testing** — comparing a new version against a baseline on a dataset to catch quality regressions before ship.

### The Eval-in-CI Gate (regression eval on a golden set)

**What it is.** A required CI stage that runs the candidate version (new prompt/model/config) over a curated **golden dataset** of representative inputs with reference outputs/criteria, scores it with evaluators, and **fails the pipeline** (blocking deploy) if the aggregate score falls below a threshold or regresses versus the current production baseline.

**How it works mechanistically.** After unit tests and image build, the pipeline invokes the app on each golden example and applies evaluators — code rules (exact match, JSON-valid, contains-citation), and/or LLM-as-judge (correctness, faithfulness) — producing per-example scores aggregated into an experiment. A gate step compares the experiment's headline metric to a configured floor (e.g. faithfulness ≥ 0.85) *and* to the last known-good baseline (no more than an N-point drop). If either check fails, the job exits non-zero and the deploy step never runs; if it passes, the artifact is promoted. This is the same offline-evaluation loop from the eval chapter, wired as a required status check on the PR/merge.

**Where it appears in real systems.** In LangSmith this is offline evaluation over a **dataset**, run with `client.evaluate(...)` in a CI job (GitHub Actions), producing an **experiment** you can compare against the baseline; failing traces get added back to the dataset to grow coverage (the online→offline feedback loop). In the pipeline it's a job that must pass before the `deploy` job runs — the AI-native equivalent of "tests must be green to merge."

### Versioning & Pinning Everything (prompts, model, tools, index/embeddings)

**What it is.** Treating every behavior-bearing artifact as an immutably versioned entity — a prompt commit hash, an explicit dated model snapshot, a tool-schema version, an index/embedding-model version — so a "release" is a pinned *tuple* of all of them and is exactly reproducible.

**How it works mechanistically.** Behavior = f(prompt, model, tools, retrieved context). If any input to that function floats, the output can change with no code change and you can't reproduce or bisect a regression. Pinning turns each into a named, stored version: prompts get commit hashes/tags in a registry; models get an explicit snapshot ID instead of a floating alias; tool schemas get a version number; the vector index and the embedding model that populated it are tagged together (you cannot query an index with a different embedding model than built it). The most dangerous anti-pattern is a **floating model alias** (`gpt-4o`, `claude-*-latest`): providers silently update what the alias points to, so the identical request returns different behavior overnight — a "silent deploy" you never triggered.

**Where it appears in real systems.** OpenAI documents dated snapshots (e.g. `gpt-4o-2024-05-13`) precisely so you can pin instead of tracking a floating name; Anthropic states that **every model ID is a pinned snapshot** and that convenience aliases resolve to a dated ID. LangSmith prompts are versioned by **commit hash** and pullable by hash or tag (`client.pull_prompt("joke-generator:12344e88")`). For retrieval, you tag the embedding model (`text-embedding-3-large`) and the index build together — re-embedding with a new model is itself a versioned, eval-gated migration.

### Progressive Delivery (shadow, canary, blue-green, A/B) + Rollback Triggers

**What it is.** Releasing a new version gradually and reversibly rather than all-at-once: **shadow** mirrors traffic to the new version with zero user impact; **canary** sends a small % of real traffic to the new version and watches metrics; **blue-green** keeps two full environments and flips 100% of traffic between them (instant rollback); **A/B** splits traffic to compare two versions on a business/quality metric.

**How it works mechanistically.** Progressive delivery limits the *blast radius* by exposing the new version to a subset first, observing correctness, then widening — coupling automation with metric analysis to drive automated promotion or rollback. Shadow is the safest first step for AI because you can compare v2's outputs against v1 on *live* inputs (and even score them offline) without a single user seeing v2. Canary then exposes, say, 5%, while a controller watches metrics; **rollback triggers** for AI extend the usual ops signals (latency, error rate) with **quality** (eval score / thumbs-down rate) and **cost** (tokens or $ per request) — any of the three regressing past a threshold aborts the rollout and shifts traffic back. Blue-green gives the fastest rollback (flip the pointer) but a "massive" blast radius if the new version is bad and you're at 100%; canary trades a more complex setup for a small blast radius.

**Where it appears in real systems.** Argo Rollouts (and cloud service meshes) implement canary/blue-green with **automated promotion or rollback based on metric analysis**; for AI you wire the analysis to include quality and cost metrics, not just HTTP error rate. In a FastAPI deployment this is two Kubernetes `Rollout`/Deployments behind a traffic split, or two versions behind an API gateway weighted 95/5, with an analysis step querying your metrics/eval store.

### Environment, Secrets & Prompt-Registry / Feature-Flag Management

**What it is.** The separation of code from environment-specific configuration: distinct dev/stage/prod environments, secrets (API keys) injected at runtime rather than baked into images, and prompt/model choices managed through a **prompt registry** or **feature flags** so they can change per-environment and at runtime.

**How it works mechanistically.** The 12-factor principle — config in the environment, not the code — matters more for AI because the "config" now includes behavior-defining prompts and model pins. Secrets (provider keys, DB URLs) come from a secrets manager or env vars so the same immutable image runs in every environment. Prompts live in a registry keyed by name+version+environment tag: stage points at the candidate prompt commit, prod at the known-good one. Feature flags let you gate a new prompt/model to a cohort and flip it off instantly without a deploy — the config-side analog of a rollback.

**Where it appears in real systems.** A FastAPI service reads `ANTHROPIC_API_KEY` and `DATABASE_URL` from the environment (never committed); it pulls its system prompt from LangSmith by an env-specific tag (`support-agent:prod`), so promoting a prompt is *moving a tag*, not editing code. Feature-flag systems (e.g. a flags service) gate `use_new_router_prompt` to 5% of traffic — the canary implemented at the config layer.

### Containerized, Stateless FastAPI Serving + Health Checks

**What it is.** Packaging the agent service as a container that is **stateless** — no conversation state, checkpoints, or memory held in process — so any instance is interchangeable, horizontally scalable, and safe to kill/replace during a rollout; plus **health checks** so the orchestrator only routes traffic to ready instances.

**How it works mechanistically.** Progressive delivery and autoscaling require that killing v1 pods and starting v2 pods lose nothing. That holds only if per-conversation state (message history, agent checkpoints) lives *outside* the process — in the LangGraph checkpointer/store backed by PostgreSQL, and long-term memory in the store — so a new pod resumes a thread by reading it from Postgres by `thread_id` (this is the durable-execution/statelessness point from section 03). Health checks split into **liveness** (is the process alive? restart if not) and **readiness** (are dependencies — DB, provider reachable — up? only then receive traffic), so a pod that can't reach Postgres is pulled from rotation instead of erroring users.

**Where it appears in real systems.** A `Dockerfile` builds the FastAPI app; Kubernetes uses a `readinessProbe` hitting `GET /healthz` (which checks the DB/checkpointer connection) and a `livenessProbe`; the app holds no session dict in memory — `graph = builder.compile(checkpointer=PostgresSaver(...))` and every request carries a `thread_id`. Because the image is stateless, rolling from `svc:1.4.1` to `svc:1.4.2` (or rolling back) is a pod swap with zero data loss.

### Decoupling Prompt/Model Changes from Code Deploys

**What it is.** An architecture where changing a prompt or model pin is a **data/config change** (flip a registry pointer or flag) rather than a **code change** (rebuild + redeploy the container), so you can iterate on behavior at registry speed while still eval-gating each change.

**How it works mechanistically.** The service reads its prompt and model pin at runtime from the registry/flag store instead of hardcoding them. To ship a new prompt you commit it to the registry, run the eval gate against that prompt version, and — if it passes — move the environment tag; the running service picks it up (immediately, or after a short cache TTL). This makes prompt iteration fast *and* reversible (move the tag back) without touching the deploy pipeline for code. The discipline: decoupling removes the redeploy friction, it does **not** remove the eval gate — an un-gated registry flip is just a faster way to ship a regression.

**Where it appears in real systems.** LangSmith's prompt cache pulls versioned prompts with a stale-while-revalidate TTL (default 300s), so a tag move propagates within minutes without a redeploy; you pin the model in the same registry entry (push a prompt-plus-model chain). The corresponding risk control is that the tag-move is itself triggered only after the eval job on that prompt commit passes.

### Handling Breaking Provider Changes (model deprecations)

**What it is.** The operational process for surviving a provider retiring or changing a model you depend on — a deprecation (announced, with a shutdown date) or a silent alias update — without an outage or a quality cliff.

**How it works mechanistically.** Providers give notice before retiring generally-available models (OpenAI: ≥6 months for GA models; Anthropic: ≥60 days for publicly released models) and publish recommended replacements. Because you *pinned* an explicit snapshot, a deprecation is a scheduled, testable migration, not a surprise: you point stage at the replacement snapshot, run the golden-set regression eval to quantify the behavior delta, tune the prompt if needed, then canary the new pin to prod before the shutdown date. If you had used a floating alias, the provider's swap would have hit prod with no eval and no warning. Preview models get much shorter notice, so you never pin those for business-critical paths.

**Where it appears in real systems.** OpenAI's deprecations page lists shutdown dates and replacement models; Anthropic's model-deprecations page lists retirement dates and lets you audit which API keys still call a deprecated model. Your migration is: swap the pinned model ID in the registry entry → eval gate → shadow/canary → promote — the same eval-gated, reversible flow you use for any change.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Canary traffic % (initial) | Fraction of real traffic sent to the new version first | Start at ~1–5% so a bad version harms few users; only widen after metrics hold for a soak window. Skip canary and use blue-green if versions can't run in parallel. |
| Eval pass threshold to promote | The golden-set score a candidate must clear to deploy | Set an absolute floor *and* a max regression vs baseline (e.g. faithfulness ≥ 0.85 AND no >2-point drop vs prod); tighten the floor for higher-stakes flows. |
| Rollback trigger — quality | Eval/feedback metric that auto-aborts a rollout | Roll back if the canary's quality metric (LLM-judge score or thumbs-down rate) drops beyond a set delta vs the control version over the soak window. |
| Rollback trigger — latency | p95/p99 latency ceiling for the new version | Roll back if canary p95 exceeds the SLO (or is materially worse than control); a "smarter" model that blows the latency budget is a failed rollout. |
| Rollback trigger — cost | Tokens or $ per request ceiling | Roll back if cost/request on the canary exceeds budget by a set margin — a reasoning-heavier model can pass quality but break unit economics. |
| Model-version pin policy | Whether you call a dated snapshot or a floating alias | Always pin an explicit dated snapshot in prod; treat alias/`latest` usage as an incident risk. Re-pin only via an eval-gated migration. |
| Prompt-registry cache TTL | How fast a promoted prompt tag propagates to running pods | Short TTL (minutes) for fast iteration/rollback; longer for stability. Use `skip_cache` when you must force the freshest version. |
| Prompt delivery: registry vs redeploy | Whether prompt changes ship as config or code | Registry/flag when you iterate on prompts frequently and want instant rollback; bake into the image only when prompts are stable and you want a single audited artifact. |

### Worked Example: Requirement → Decision

**Given:** Your production support agent (FastAPI + PostgreSQL checkpointer, currently pinned to `claude-opus-4-8` with system prompt `support-agent:prod`) needs to adopt a newly reworded system prompt *and* trial a newer model snapshot that a teammate claims answers better. It's live 24/7 for paying customers; a bad rollout means wrong billing answers. You have a 300-example golden set with reference answers and a faithfulness LLM-judge. Latency SLO is p95 ≤ 3 s and there's a per-conversation cost budget.

- **Step 1 — Identify the goal:** Ship the new prompt + model pin *only if* quality holds and cost/latency stay within budget, with the ability to revert in seconds.
- **Step 2 — Define inputs:** The candidate prompt commit (`support-agent:a1b2c3`), the candidate model snapshot, the 300-example golden dataset, evaluators (faithfulness judge + JSON/format checks), and live prod traffic to mirror.
- **Step 3 — Define outputs:** A promote/block decision from CI; if promoted, a canary metric verdict; the final state is either "prod tag + model pin moved to the new versions" or "unchanged, alerted."
- **Step 4 — Apply constraints:** Non-deterministic outputs (unit tests insufficient → must eval); paying users (small blast radius, fast rollback); p95 ≤ 3 s and cost budget (latency + cost rollback triggers, not just quality); stateless service so a version swap loses no conversation.
- **Step 5 — Select the approach:** Run the **regression eval on the golden set in CI** for the new prompt+model tuple; if faithfulness clears the floor and doesn't regress vs the `prod` baseline, deploy to stage, then **shadow** live traffic (compare v2 vs v1 outputs, 0% user impact) to catch real-input surprises, then **canary at 5%** watching quality + p95 + cost/request, auto-rolling-back on any regression, and only then promote by moving the `prod` tag and model pin. *Rationale vs alternatives:* a big-bang deploy to 100% is wrong because a non-deterministic quality regression that unit tests can't catch would hit every paying customer at once; shadow-then-canary bounds the blast radius and the eval gate front-loads the cheapest failure detection, while the stateless design + tag-based pin make rollback a pointer flip.

---

## Implementation

```yaml
# Scenario: A prompt/model change to a live agent must NOT deploy just because unit
# tests are green — non-deterministic quality can regress with no code change. This
# CI workflow makes a golden-set regression eval a REQUIRED gate: the deploy job runs
# only if eval clears the threshold. (GitHub Actions + LangSmith offline evaluation.)
name: ai-service-ci
on: [pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt
      - run: pytest tests/            # deterministic checks: routing, JSON schema, 200s

  eval-gate:                          # the AI-native gate — must pass to deploy
    needs: unit-tests
    runs-on: ubuntu-latest
    env:
      LANGSMITH_API_KEY: ${{ secrets.LANGSMITH_API_KEY }}
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt
      # Runs the candidate over the golden dataset, scores it, and exits non-zero
      # if faithfulness < floor OR it regresses vs the prod baseline experiment.
      - run: python ci/run_regression_eval.py --dataset support-golden-v3 --min-faithfulness 0.85

  deploy-stage:
    needs: eval-gate                  # <-- deploy is BLOCKED unless the eval gate passed
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/deploy.sh stage
```

```python
# Scenario: the eval-gate script above. It runs the current app version on a curated
# golden set and blocks promotion on a quality regression — the thing unit tests miss.
# API verified against LangSmith evaluation docs (docs.langchain.com/langsmith).
import sys, argparse
from langsmith import Client

def faithfulness(outputs: dict, reference_outputs: dict) -> bool:
    # your LLM-as-judge / code evaluator; returns pass/fail or a score
    return judge_faithful(outputs["answer"], reference_outputs["answer"])

def target(inputs: dict) -> dict:
    # invokes the app with the CANDIDATE prompt/model pin under test
    return {"answer": run_support_agent(inputs["question"])}

if __name__ == "__main__":
    p = argparse.ArgumentParser()
    p.add_argument("--dataset"); p.add_argument("--min-faithfulness", type=float)
    a = p.parse_args()

    results = Client().evaluate(target, data=a.dataset, evaluators=[faithfulness])
    score = mean_score(results, key="faithfulness")
    print(f"faithfulness={score:.3f} (floor={a.min_faithfulness})")
    if score < a.min_faithfulness:
        sys.exit(1)              # non-zero exit fails the CI job -> deploy blocked
```

```python
# Anti-pattern: calling the provider by a FLOATING alias and shipping a new prompt
# straight to 100% traffic with only unit tests. The alias means the provider can
# silently update the model under you (a "deploy" you never made), and the un-gated
# prompt change can tank answer quality for every user at once, invisibly.
from anthropic import Anthropic
client = Anthropic()

SYSTEM_PROMPT = "You are a support agent. Be extra concise now."   # edited, un-evaluated

def answer(q: str) -> str:
    return client.messages.create(
        model="claude-latest",            # floating alias -> silent, untriggered changes
        system=SYSTEM_PROMPT,             # new prompt, 0% -> 100% with no eval, no canary
        max_tokens=1024,
        messages=[{"role": "user", "content": q}],
    ).content[0].text

# Correct approach: PIN an explicit model snapshot, pull the prompt by a versioned tag
# from the registry (so promotion is eval-gated + a reversible tag move), and let the
# rollout controller canary it. Behavior is now reproducible and every change is gated.
from langsmith import Client
ls = Client()

MODEL_PIN = "claude-opus-4-8"            # explicit dated/pinned snapshot, not an alias

def answer(q: str) -> str:
    prompt = ls.pull_prompt("support-agent:prod")   # env-tagged, versioned prompt commit
    payload = prompt.invoke({"question": q})
    return client.messages.create(
        model=MODEL_PIN, max_tokens=1024, **to_anthropic(payload)
    ).content[0].text
# What breaks without this: with "claude-latest", a provider model update changes prod
# behavior overnight with zero code change and no eval — undiagnosable from your git log;
# and a 0%->100% prompt push means a quality regression hits every customer before you
# see a single metric. Pinning + registry tag + eval gate + canary makes each change
# reproducible, gated, gradual, and revertible by moving the tag back.
```

```yaml
# Scenario: the service must be safe to kill/replace during a rollout (canary, autoscale)
# without losing conversations, and traffic must only hit READY pods. Statelessness +
# health checks make any version interchangeable; state lives in the Postgres checkpointer.
# (Kubernetes probe spec; app exposes /healthz and /version.)
containers:
  - name: agent-svc
    image: registry.example.com/agent-svc:1.4.2   # immutable, pinned image tag
    env:
      - name: DATABASE_URL                          # secrets injected, never baked in
        valueFrom: { secretKeyRef: { name: agent-secrets, key: database-url } }
      - name: ANTHROPIC_API_KEY
        valueFrom: { secretKeyRef: { name: agent-secrets, key: anthropic-key } }
    readinessProbe:                                 # only serve traffic if deps are up
      httpGet: { path: /healthz, port: 8000 }       # /healthz checks Postgres/checkpointer
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:                                  # restart a hung process
      httpGet: { path: /livez, port: 8000 }
      periodSeconds: 20
```

---

## Common Pitfalls & Misconceptions

- **"Green unit tests mean it's safe to ship"** — beginners transfer the deterministic-software intuition where a passing suite covers behavior. LLM behavior lives in prompts/models/configs a compiler never checks, so identical inputs can degrade in quality while every test stays green; the real gate is a regression eval on a golden set, not the test suite.
- **Calling models by a floating alias (`latest`/undated)** — it feels convenient and "always up to date." But the provider silently updates what the alias resolves to, so prod behavior changes with no code change, no PR, and no eval — an untraceable deploy; pin an explicit dated snapshot and re-pin only through an eval-gated migration.
- **Big-bang deploying a new prompt to 100%** — because a prompt "isn't code," people treat it as a trivial edit and push it everywhere at once. A prompt change *is* a behavior change; ship it through shadow/canary with quality+cost+latency rollback triggers so a regression harms a few requests, not all of them.
- **Watching only latency/errors on a canary** — teams reuse ops dashboards that never measured quality or cost. An AI canary can be fast and 200-OK while giving worse answers or burning 3× the tokens; your rollback triggers must include a quality metric and a cost-per-request metric, not just p95 and error rate.
- **Holding conversation state in process memory** — a session dict feels simplest. But then killing a pod during a rollout or scale-down drops live conversations and makes versions non-interchangeable; keep the service stateless with state in the Postgres checkpointer/store keyed by `thread_id` so any version can be swapped safely.
- **Decoupling prompts from deploys, then skipping the eval gate** — once a registry lets you flip prompts without redeploying, it's tempting to "just change it in prod." Fast iteration is the benefit, but an un-gated tag flip is simply a faster way to ship a regression; the eval gate must run against the prompt version before the tag moves.

---

## Key Definitions

| Term | Definition |
|---|---|
| Eval-in-CI gate | A required CI stage that runs a regression eval on a golden dataset and blocks deploy if the score falls below a threshold or regresses vs the baseline. |
| Golden set / dataset | A curated set of representative inputs with reference outputs/criteria used to score a candidate version's quality. |
| Regression testing (AI) | Comparing a new version against a baseline on a dataset to detect quality drops before shipping. |
| Model snapshot / pin | An explicit, dated model identifier (vs a floating alias) that guarantees the same model version is used, for reproducibility. |
| Floating alias | A model name (e.g. `latest`, undated) whose target the provider can silently update, changing behavior with no code change. |
| Shadow deployment | Mirroring live traffic to a new version whose responses are logged/compared but never returned to users (0% user impact). |
| Canary deployment | Routing a small % of real traffic to a new version and watching metrics before widening. |
| Blue-green deployment | Two full environments (only one live); flip 100% of traffic between them for instant rollback. |
| A/B deployment | Splitting traffic between two versions to compare them on a quality/business metric. |
| Rollback trigger | A metric threshold (quality, cost, latency, error rate) that automatically aborts a rollout and reverts traffic. |
| Progressive delivery | Releasing gradually to a growing traffic subset with metric-driven automated promotion or rollback. |
| Prompt registry | A versioned store of prompts (by commit hash / tag / environment) that services pull at runtime, decoupling prompt changes from code deploys. |
| Statelessness | The service holds no per-conversation state in process; state lives in an external checkpointer/store so instances are interchangeable. |
| Readiness / liveness probe | Health checks: readiness gates whether a pod receives traffic (deps up); liveness restarts a hung process. |
| Model deprecation | A provider's announced retirement of a model (with a shutdown date and recommended replacement) requiring migration. |

---

## Summary / Quick Recall

- AI deploys differ because behavior lives in **non-deterministic artifacts** (prompts/models/tools/indexes) — passing unit tests is necessary but not sufficient; the real gate is a **regression eval on a golden set**.
- **Pin everything**: prompt commit, dated model snapshot, tool-schema version, index+embedding version — a release is a pinned *tuple*; floating aliases invite silent provider-driven behavior changes.
- Ship via **progressive delivery** — shadow (0% impact) → canary (small %) → full — with **rollback triggers on quality, cost, AND latency**, not just error rate.
- Keep the service **stateless** (state in the Postgres checkpointer/store, keyed by `thread_id`) with **liveness/readiness health checks** so any version is safe to kill, swap, or roll back.
- **Decouple** prompt/model changes from code via a **registry/feature flags** for fast, reversible iteration — but keep the eval gate: an un-gated flip is just a faster regression.
- **Model deprecations** are scheduled migrations when you're pinned (OpenAI ≥6mo GA notice, Anthropic ≥60-day notice): re-pin the replacement → eval → canary → promote before the shutdown date.

---

## Self-Check Questions

1. Why is "all unit tests pass" an insufficient gate for shipping an LLM/agentic service, and what gate replaces it?

   <details><summary>Answer</summary>

   Because an LLM service's behavior is produced by non-deterministic artifacts (prompt wording, model snapshot, tool schema, retrieval config) that a compiler/test suite doesn't check — you can have 100% green unit tests (routing works, JSON parses, 200 returned) while answer *quality* silently regresses, since the same input can yield different, worse output. The replacing gate is a **regression eval on a golden set**: run the candidate over curated inputs, score with evaluators, and block deploy below a threshold or on regression vs baseline. The tempting wrong answer — "add more unit tests" — fails because unit tests assert deterministic properties, not output quality; no number of them measures faithfulness or answer correctness on real inputs.

   </details>

2. You maintain a FastAPI agent pinned to `claude-opus-4-8`. A teammate proposes switching the model call to `claude-latest` "so we always get the newest model automatically." What do you tell them and why?

   <details><summary>Answer</summary>

   Reject it: a floating alias lets the provider **silently update** the model your prod service resolves to, so behavior changes with no code change, no PR, and no eval — an untraceable, ungated deploy you never triggered. Keep an explicit **dated/pinned snapshot** so the release is reproducible, and adopt newer models only through an eval-gated migration (re-pin in stage → golden-set regression eval → canary → promote). The tempting "but we get improvements for free" view is wrong because "newest" is not "better for *your* task or budget," and an unpinned change can regress quality/cost with zero warning.

   </details>

3. **Which TWO** of the following are correct about progressive delivery and rollback for AI services?
   - A. A shadow deployment mirrors live traffic to the new version but never returns its responses to users, so it has zero user impact.
   - B. Because latency and HTTP error rate are the standard signals, they are sufficient rollback triggers for an AI canary.
   - C. Rollback triggers for an AI service should include a quality metric and a cost-per-request metric in addition to latency/errors.
   - D. Blue-green deployment sends a small percentage of traffic to the new version and gradually increases it.
   - E. Canary is safe to skip entirely as long as unit tests pass.

   <details><summary>Answer</summary>

   **A and C.** A is correct: shadow mirrors traffic to v2 whose outputs are logged/compared but discarded, giving real-input signal at 0% user impact. C is correct: an AI version can be fast and return 200 while giving worse answers or burning far more tokens, so quality and cost must be first-class rollback triggers. B is the most tempting distractor and is wrong precisely because latency/errors miss quality and cost regressions — the failure modes unique to AI. D describes a *canary*, not blue-green (blue-green flips 100% between two full environments). E is wrong: unit tests don't measure quality, so skipping the canary removes your last small-blast-radius safety net.

   </details>

4. Your team can push prompt changes at will via a registry without redeploying code. Over a month, answer quality slowly drifts down and no one can pinpoint when. What most likely went wrong, and how do you fix the process?

   <details><summary>Answer</summary>

   Decoupling prompts from deploys removed the redeploy friction but the team also (implicitly) removed the **eval gate** — people edited the prod prompt tag directly, so ungated changes accumulated with no golden-set check and no baseline comparison, producing untracked drift. The fix: require that any prompt version pass the **regression eval on the golden set** before its tag can be promoted to prod (gate the tag move, not just the code deploy), and keep prompts **versioned by commit hash** so each change is attributable and revertible. Blaming the model or adding compute misses it — the process gap is an ungated, unversioned config path, and the remedy is to reattach the eval gate to the registry promotion, not to change models.

   </details>

5. A stakeholder wants the highest-quality answers and proposes swapping in a newer, "smarter" reasoning model, deployed straight to 100% of traffic to move fast. Evaluate this against an eval-gated shadow→canary rollout under a p95 ≤ 3 s SLO and a per-request cost budget.

   <details><summary>Answer</summary>

   The big-bang 100% swap is the wrong trade: a smarter reasoning model can *raise* answer quality while *breaking* the latency SLO (more thinking tokens → higher p95) and the cost budget (more tokens per request), and pushing it to everyone at once means those regressions hit every user before any metric is read. The eval-gated shadow→canary flow is better: the golden-set eval front-loads a quality check, **shadow** measures the new model's real-input latency/cost/quality at 0% user impact, and **canary at ~5%** with rollback triggers on quality *and* p95 *and* cost/request catches the SLO/budget breach on a small slice, auto-reverting. The tempting "smarter model is strictly better, ship it" view fails because quality is only one of three constraints — a rollout that wins on quality but violates latency or unit economics is a failed rollout, and only gradual, metric-gated delivery surfaces that safely.

   </details>

---

## Further Reading

- [OpenAI — Deprecations](https://platform.openai.com/docs/deprecations) — *verified 2026-07-29* — Provider policy on model retirement notice periods (≥6 months for GA models), dated snapshots vs aliases, and recommended replacements — the source for treating deprecations as scheduled, eval-gated migrations.
- [Anthropic — Model deprecations](https://docs.anthropic.com/en/docs/about-claude/model-deprecations) — *verified 2026-07-29* — Model lifecycle states (active/legacy/deprecated/retired), ≥60-day retirement notice, and how to audit which API keys still call a deprecated model.
- [Anthropic — Models overview](https://docs.anthropic.com/en/docs/about-claude/models/overview) — *verified 2026-07-29* — Confirms every Claude model ID is a pinned snapshot and that aliases are convenience pointers — the basis for model pinning.
- [LangSmith — Evaluation](https://docs.langchain.com/langsmith/evaluation) — *verified 2026-07-29* — Offline vs online evaluation, datasets, evaluators, experiments, and regression testing — the mechanics of the eval-in-CI gate.
- [LangSmith — Manage prompts programmatically](https://docs.langchain.com/langsmith/manage-prompts-programmatically) — *verified 2026-07-29* — Prompt versioning by commit hash/tag, pulling a pinned prompt version, and the stale-while-revalidate cache for hot-swapping prompts without a redeploy.
- [Argo Rollouts — Concepts (Progressive Delivery)](https://argo-rollouts.readthedocs.io/en/stable/concepts/) — *verified 2026-07-29* — Definitions of progressive delivery, canary, and blue-green, and metric-driven automated promotion/rollback and blast-radius trade-offs.
