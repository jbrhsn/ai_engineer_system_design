# Versioning & Change-Management Patterns for AI System Design

**Section:** 05 Interview Practice → Cross-Cutting Concerns & Trade-off Patterns | **Interview relevance:** High — "how do you ship a change safely and reproduce a past behaviour?" is a near-guaranteed follow-up once you propose any production LLM/RAG/agent system, and it forces you to reason about model/prompt/index pinning, eval-gated rollout, and the lineage needed to answer "which model+prompt+index produced *this* output?"

---

## TL;DR

Production AI systems must evolve — new models ship, prompts get tuned, embedding models improve — but every change risks silent behaviour drift that a plain unit test cannot catch. You manage this by treating model version, prompt, tool/output schema, and RAG index as *pinned, versioned artifacts*, promoting a new version only after it passes a regression evaluation, and rolling it out gradually (canary → full) with a fast rollback path. The through-line is reproducibility: you must be able to reconstruct exactly which model + prompt + index version produced any logged output, or you cannot debug, audit, or safely revert. **The one thing to remember: never float to "latest" in production — pin every version, eval-gate every promotion, and log the version triple on every output so behaviour is reproducible and reversible.**

---

## ELI5 — Explain It Like I'm 5

Imagine a bakery that sells a chocolate cake, and the recipe card is kept in a locked box with a version number on it. If the baker quietly swaps in a new flour supplier without changing the card, customers might suddenly get a different-tasting cake and nobody knows why — that is what happens when you let a system quietly use the "latest" model. The safe bakery instead writes a new recipe card (v2), bakes a few test cakes, and has tasters compare them to the old cake *before* selling v2 to everyone; if the tasters complain, they tear up v2 and go back to v1 instantly. They also stamp every cake box with the recipe number, the oven used, and the flour batch, so if one customer complains they can look up exactly how that cake was made. The big mistake beginners make is thinking "newer is always better, just upgrade" — but without a taste-test and a version stamp you can't tell if you made things worse, and you can't undo it.

---

## Learning Objectives

By the end of this note you will be able to:
- [ ] Pin a model, prompt, and RAG index as versioned artifacts and explain why floating to a `-latest` alias is unsafe in production.
- [ ] Design an evaluation-gated rollout (regression eval → canary → full) with a defined rollback trigger for a model or prompt change.
- [ ] Diagnose and migrate an embedding-model change that requires re-embedding, using a dual-index cutover to avoid mixing incompatible vectors.
- [ ] Justify a versioning/change-management choice against its trade-off (velocity vs reproducibility, rollout cost vs blast radius) in an interview.

---

## Visual Overview

### Eval-gated promotion pipeline (dev → canary → prod, with rollback)

```
   New model or prompt (v2, pinned)
        │
        ▼
   ┌─────────────┐   fail   ┌────────────────────────┐
   │ Regression  │ ───────► │ Block promotion; keep  │
   │ eval on     │          │ v1 serving 100%        │
   │ golden set  │          └────────────────────────┘
   └─────┬───────┘
         │ pass (score ≥ threshold, no regressions)
         ▼
   ┌─────────────┐  metrics/eval regress in canary
   │ Canary 5%   │ ──────────────────────────────────►  ROLLBACK
   │ v2 / 95% v1 │        (repoint alias to v1)          instant
   └─────┬───────┘
         │ canary healthy (quality + latency + error rate OK)
         ▼
   ┌─────────────┐
   │ Ramp 25→50  │ ──► 100% v2  (v1 retained for fast revert)
   │ →100% v2    │
   └─────────────┘
```

### Embedding-model change forces re-index (dual-index migration)

```
 BEFORE (broken shortcut)                AFTER (dual-index cutover)
 ─────────────────────────               ─────────────────────────────
 query ──embed(new)──► [index]           query ──embed(v1)──► [index v1] ◄─ live
                        built with              (serve reads here)
                        embed(OLD)         build in parallel:
   ✗ query vector and stored               corpus ──embed(v2)──► [index v2]
     vectors live in DIFFERENT             then eval v2 retrieval vs v1
     spaces → garbage neighbours           ▼ pass
                                          flip reader ──embed(v2)──► [index v2]
                                          retire [index v1] after retention window
```

---

## The Core Problem

Every production AI system sits between two forces pulling in opposite directions. On one side is **velocity of change**: providers deprecate models on fixed timelines, better/cheaper models and embedding models appear, and prompts need continuous tuning as failure modes surface — so the system *must* keep changing. On the other side is **reproducibility and safety**: an LLM's behaviour is a function of the exact model weights, the exact prompt, and (for RAG) the exact index and embedding model, and any of those changing can silently shift outputs in ways no schema check or unit test detects. A model provider's routing or safety-classifier update can even alter behaviour on a *fixed* model ID without any code change on your side.

The interview question is almost never "should you upgrade" — it is "how do you upgrade *without* breaking reproducibility, and how do you prove and undo the change." Two things must be separated because they are governed by different mechanisms:

- **Pinning & lineage** — freezing each component to a specific version and recording, per output, the exact `(model, prompt, index)` triple that produced it. This is what makes behaviour reproducible and auditable.
- **Safe rollout & rollback** — gating a version change on a regression evaluation, shipping it to a small slice first, and keeping the previous version instantly revertible. This is what bounds the blast radius of a change that passed offline checks but regresses in production.

A system that pins everything but ships changes blind will still cause outages; a system that canaries carefully but logs no version triple can never explain *why* an output was wrong. You need both.

---

## Resolution Options

| Option | How it works | Buys you | Costs you | Reach for it when… |
|---|---|---|---|---|
| **Model pinning** | Reference a specific dated/immutable model ID, never a `-latest` alias | Reproducible behaviour; upgrades happen only when you choose | Must actively track deprecations and migrate before shutdown | Always in prod — the default |
| **Prompt registry / versioning** | Store prompts as versioned artifacts (commit hash, tags) in a registry, pull by version | Diff-able, revertible prompts; decouples prompt edits from code deploys | Extra infra; must pin the version, not the mutable tag | Prompts change often or are tuned by non-engineers |
| **Eval-gated promotion** | Run a regression eval on a golden set; promote only if score ≥ threshold with no regressions | Catches silent quality drift before users see it | Requires a maintained labelled eval set; adds a step to release | Any model/prompt/embedding change |
| **Canary / A/B rollout** | Route a small traffic % to the new version, compare live metrics, then ramp | Bounds blast radius; real-traffic validation | Serving two versions at once; needs metric attribution by version | Higher-risk changes on live traffic |
| **Shadow deployment** | Run the new version on real inputs in parallel but discard its output (users see the old one) | Zero user risk while collecting real-input behaviour | Double inference cost; no live user-facing validation | Validating a risky change before any user exposure |
| **RAG index / embedding versioning** | Version the index alongside the embedding model; re-embed on embedder change via dual-index cutover | Query and stored vectors stay in the same space | Full re-embed cost; storage for two indexes during migration | The embedding model or chunking changes |
| **Schema / contract versioning** | Version tool signatures and structured-output schemas; support N and N+1 during transition | Non-breaking evolution of tools/outputs | Transitional dual-support code | Changing a tool signature or output schema |
| **Dataset & fine-tune lineage** | Version training data + config; link each fine-tuned model to the data/run that produced it | Reproducible, auditable fine-tunes | Data/versioning infra (e.g. registry) | You fine-tune or maintain eval datasets |

**Model pinning** — a pinned model ID maps to a single fixed weight snapshot for the lifetime of that ID, whereas a convenience alias (e.g. a `-latest` or a dateless pointer that resolves to "the most recent snapshot") can silently change under you. Mechanically you set the `model` field to the immutable ID and treat any change to it as a code change that goes through review and eval. Note the subtlety from provider docs: even a pinned ID can shift behaviour if the provider updates serving infrastructure (routers, safety classifiers, sampling) — pinning removes *weight* drift, not all drift, which is exactly why you still monitor by version. It appears as `model="<dated-or-immutable-id>"` in every completion call.

**Prompt registry / versioning** — prompts are inputs that change behaviour as much as code, so they are stored as artifacts with immutable version identifiers (a commit hash or a pinned tag) rather than string literals scattered in code. Under the hood a registry stores each prompt as a versioned object; you pull a *specific* commit hash at runtime, and diff two versions to review a change. It appears as a pull-by-hash call (e.g. `pull_prompt("my-prompt:<commit-hash>")`) or a checked-in prompt file with a version field. The trap is pulling by a mutable tag like `:latest`, which reintroduces the drift you were trying to avoid.

**Eval-gated promotion** — before any new model/prompt/embedding version serves traffic, it is run against a curated golden set and scored; promotion is blocked unless the aggregate score clears a threshold *and* no previously-passing case regresses. This is a CI-style gate in front of the deploy. It appears as an evaluation job in the release pipeline whose pass/fail decides promotion — see `03-evaluation-and-quality-assurance-patterns.md` for how to build the eval set and metrics that make this gate meaningful.

**Canary / A/B rollout** — a routing layer sends a small, growing fraction of live traffic to the new version while the rest stays on the old one; you compare quality and operational metrics *attributed by version* and ramp only if the canary is healthy. It appears as a traffic-split config keyed on version, with dashboards sliced by version. It bounds blast radius because a bad change only hits the canary slice before you catch it.

**Shadow deployment** — the new version runs on the same real inputs as production but its output is logged and discarded, never shown to users; you compare shadow output against the live output offline. Mechanically the request fans out to both versions and only the incumbent's response is returned. It appears as a parallel "shadow" call in the serving path. It gives real-input validation at zero user risk, at the cost of double inference.

**RAG index / embedding versioning** — retrieval only works if the query is embedded by the *same* model that embedded the stored corpus, because different embedding models produce vectors in incompatible spaces. So the index is versioned in lockstep with its embedding model; changing the embedder means re-embedding the entire corpus into a new index. It appears as an index-version tag paired with an embedder-version tag, and a dual-index migration during cutover (build v2 alongside v1, eval, then flip the reader).

**Schema / contract versioning** — tool signatures and structured-output schemas are contracts between the model and your code; changing them can break parsing or tool dispatch. You version the schema and support the old and new versions simultaneously during a transition window rather than flipping atomically. It appears as a `version` field on the schema/tool and branching that accepts N and N+1.

**Dataset & fine-tune lineage** — a fine-tuned model is only reproducible if you can point to the exact training data and config that produced it; a model registry links each model version to the run, data, and params behind it. It appears as a registered model version tagged with its dataset version and a `validation_status`/`champion` alias for the promoted one.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Model ID pinning (`model=`) | Whether you serve a fixed snapshot or a moving alias | Always pin an immutable/dated ID in prod; treat a change to it as a reviewed, eval-gated code change — never point at `-latest` |
| Prompt version reference | Which prompt snapshot is pulled at runtime | Pull by commit hash, never by a mutable tag like `:latest`; store the hash used with the output |
| Eval pass threshold to promote | The score a new version must clear to be promoted | Set to "≥ current prod baseline on the golden set AND zero regressions on previously-passing cases," not an absolute number picked from thin air |
| Canary traffic % | Fraction of live traffic on the new version | Start small (~1–5%) and ramp only after quality + latency + error-rate hold; size it so a bad version's blast radius is tolerable |
| Rollback trigger | The condition that auto-reverts to the previous version | Define a concrete metric bound *before* rollout (e.g. eval/quality metric drops below baseline or error rate exceeds X) so rollback is mechanical, not a judgment call |
| Index-version retention | How long the previous index/embedder version is kept | Keep the old index until the new one is validated in prod *and* past your rollback window; only then retire it |

### Worked Example: Requirement → Decision

**Given:** A live customer-support RAG system currently embeds both the corpus and queries with embedding model **E1** and serves from index **I1**. A newer embedding model **E2** promises better retrieval. The team wants to upgrade without breaking live retrieval or losing the ability to reproduce past answers.

- **Step 1 — Identify the goal:** Upgrade the embedding model to improve retrieval quality while keeping query and stored vectors in the *same* space at all times and preserving reproducibility.
- **Step 2 — Define inputs:** The existing corpus, index I1 built with E1, live query traffic, a golden retrieval eval set (queries with known-relevant chunks), and E2.
- **Step 3 — Define outputs:** A new index I2 fully re-embedded with E2, a validated quality comparison of I2 vs I1, and a cutover that flips the query embedder to E2 exactly when the reader flips to I2.
- **Step 4 — Apply constraints:** Query and stored vectors must be produced by the same embedding model (mixing E1 queries against E2 vectors yields garbage neighbours); no retrieval downtime; must be able to roll back; must log which index/embedder version served each answer.
- **Step 5 — Select the approach:** Use a **dual-index migration**. Keep I1 (E1) serving live while re-embedding the whole corpus into I2 (E2) in parallel; run the retrieval eval on I2 vs I1; if I2 clears the threshold, atomically flip the query embedder to E2 and the reader to I2 together, retaining I1 through the rollback window. Rationale vs alternatives: swapping the embedder in place on I1 is the anti-pattern — it mixes incompatible vector spaces and silently breaks retrieval; re-embedding without an eval gate risks shipping a regression; a canary on the *reader* alone can't help because the query-embedding side must move in lockstep, so the atomic embedder+index flip is the correct unit of change.

---

## Implementation

```python
# Scenario: our prod chatbot calls the model provider daily. We must NOT let a
# provider update silently change behaviour (a regression we can't reproduce),
# and we must be able to reconstruct which model+prompt produced any logged output.

# Anti-pattern: floating to a moving alias — behaviour drifts on any provider
# update, and logs can't tell you which weights actually answered.
def answer_bad(client, user_msg: str) -> str:
    resp = client.responses.create(
        model="gpt-5.6",              # alias → resolves to "latest" snapshot; can change under you
        input=user_msg,
    )
    return resp.output_text            # nothing logged about which version ran

# Correct approach: pin an immutable/dated model ID, pull the prompt by commit
# hash, and log the (model, prompt) version triple with the output for lineage.
PINNED_MODEL = "gpt-5.6-sol"          # a specific catalog model ID, treated as code
PROMPT_REF   = "support-answer:6f3a1c" # pinned prompt commit hash, not ":latest"

def answer_good(client, prompt_registry, logger, user_msg: str) -> str:
    prompt = prompt_registry.pull_prompt(PROMPT_REF)     # reproducible prompt version
    resp = client.responses.create(
        model=PINNED_MODEL,
        input=prompt.format(question=user_msg),
    )
    logger.log(                          # lineage → see 08-observability-and-monitoring-patterns.md
        output=resp.output_text,
        model_version=PINNED_MODEL,
        prompt_version=PROMPT_REF,
        # index_version=... when RAG is involved
    )
    return resp.output_text
```

```python
# Scenario: upgrading the embedding model behind a live RAG system. We must keep
# stored and query vectors in the same space and be able to roll back — so we
# build a new index in parallel, eval it, and flip the query embedder in lockstep.

# Anti-pattern: swap the embedding model but query the OLD index. Query vectors
# (E2) and stored vectors (E1) are in incompatible spaces → nearest-neighbour
# results are meaningless, and the failure is SILENT (no error, just bad recall).
def retrieve_bad(index_v1_built_with_E1, query, embed_E2):
    q = embed_E2(query)                       # E2 query vector...
    return index_v1_built_with_E1.search(q)   # ...searched against E1 vectors → garbage

# Correct approach: dual-index cutover. Re-embed the whole corpus with E2 into a
# NEW index, eval retrieval vs the old one, then flip embedder + reader together.
def migrate_embedding_model(corpus, embed_E2, eval_retrieval, threshold):
    index_v2 = build_index(corpus, embedder=embed_E2)     # full re-embed into v2
    score_v2 = eval_retrieval(index_v2, embed_E2)         # eval-gate the upgrade
    if score_v2 < threshold:
        return None                                        # block: keep serving v1
    # atomic flip: query embedder AND reader move to E2/v2 in lockstep
    return {"reader_index": index_v2, "query_embedder": embed_E2}
    # retain v1 index through the rollback window before retiring it
```

---

## Common Pitfalls & Misconceptions

- **Floating to a `-latest` (or dateless alias) model in production** — beginners assume "latest" means "best and stable," but an alias can resolve to a new snapshot that shifts behaviour with no code change on your side; the correct mental model is that the model ID is part of your source of truth, so pin an immutable version and upgrade only through a reviewed, eval-gated change.
- **Swapping the embedding model without re-indexing** — the instinct is "the embedder is just a config value, change it and go," but query vectors and stored vectors must come from the *same* model or they live in incompatible spaces and retrieval silently degrades; the correct model is that embedder and index are one versioned unit — changing the embedder means re-embedding the whole corpus (dual-index cutover) and flipping both sides together.
- **Treating a passing offline eval as proof it's safe to ship to 100%** — a green regression eval feels like a guarantee, but the golden set never covers all live traffic, so an unseen distribution can still regress in production; the correct model is eval-gate *then* canary — the eval blocks obvious regressions and the canary bounds the blast radius of the ones it missed (links to `03-evaluation-and-quality-assurance-patterns.md`).
- **Logging outputs without the version triple** — it seems enough to log the request and response, but when an output is later found to be wrong you cannot reproduce it without knowing the exact model + prompt + index that produced it; the correct model is that every logged output must carry its `(model, prompt, index)` versions so any behaviour is reproducible, auditable, and diffable (links to `08-observability-and-monitoring-patterns.md`).

---

## Key Definitions

| Term | Definition |
|---|---|
| Model pinning | Referencing a specific immutable/dated model ID (not a moving alias) so the served weights are fixed and behaviour is reproducible until you deliberately upgrade. |
| Prompt registry | A store that holds prompts as versioned artifacts (commit hashes, tags) so they can be pulled by version, diffed, and reverted independently of code deploys. |
| Eval-gated rollout | Promoting a new model/prompt/index version only after it clears a regression evaluation on a golden set with no regressions on previously-passing cases. |
| Canary deployment | Routing a small, growing fraction of live traffic to a new version while comparing metrics by version, ramping only if healthy. |
| Shadow deployment | Running a new version on real inputs in parallel while discarding its output (users get the old version), to observe behaviour at zero user risk. |
| Re-embedding / re-indexing | Rebuilding a vector index by embedding the corpus again — required whenever the embedding model (or chunking) changes so query and stored vectors share one space. |
| Dual-index migration | Building the new index alongside the live one, evaluating it, then atomically flipping the query embedder and reader to the new index while retaining the old for rollback. |
| Lineage | The recorded chain linking an output (or a fine-tuned model) back to the exact versions/data/config that produced it, enabling reproducibility and audit. |

---

## Summary / Quick Recall

- Pin every version (model, prompt, index, schema); never serve a `-latest`/moving alias in prod.
- A pinned model ID stops weight drift but not provider infra drift — so still monitor by version.
- Gate every version change on a regression eval, then canary; the eval catches obvious regressions, the canary bounds the rest.
- Embedder and index are one versioned unit — changing the embedder forces a full re-embed via dual-index cutover, flipping both sides together.
- Keep the previous version (model/prompt/index) instantly revertible; define the rollback trigger as a concrete metric bound before rollout.
- Log the `(model, prompt, index)` triple on every output — no lineage means no reproducibility, no audit, no debugging.
- Version tool/output schemas and fine-tune datasets too; support N and N+1 during a schema transition.

---

## Self-Check Questions

1. Why is it unsafe to set `model` to a provider's moving `-latest` (or dateless) alias in a production system?

   <details><summary>Answer</summary>

   Because an alias can resolve to a new model snapshot at any time, silently changing behaviour with no code change on your side — which breaks reproducibility (you can't tell which weights answered a past request) and can introduce regressions you never tested. Pinning an immutable/dated model ID fixes the weights until you deliberately upgrade. The tempting wrong view is "latest is always the newest and best, so just use it": newer is not automatically better for *your* task, and even if it were, an uncontrolled swap you can't reproduce or roll back is an operational hazard, not a feature.

   </details>

2. Your team wants to tune the system prompt frequently, including edits by non-engineers, without a full code deploy each time. What versioning approach do you propose and what's the one rule you must enforce?

   <details><summary>Answer</summary>

   Use a prompt registry: store the prompt as a versioned artifact (commit hashes / tags) and pull it at runtime, decoupling prompt edits from code deploys and giving you diffs and reverts. The one rule: pull by an *immutable* version identifier (commit hash), never by a mutable tag like `:latest` — otherwise a prompt edit silently changes production behaviour, reintroducing exactly the drift you were avoiding, and you lose the ability to say which prompt version produced a given output. The tempting wrong answer — "just let it pull the latest prompt" — is what breaks reproducibility.

   </details>

3. A newer embedding model (E2) tests better offline. Walk through how you'd deploy it behind a live RAG index currently built with E1, without breaking retrieval.

   <details><summary>Answer</summary>

   Do a dual-index migration: keep the E1 index serving live while you re-embed the entire corpus with E2 into a new index; run a retrieval eval comparing E2/new-index against E1/old-index; if it clears the threshold, atomically flip the *query* embedder to E2 and the reader to the new index together, retaining the E1 index through the rollback window. The critical constraint is that query and stored vectors must come from the same embedding model. The tempting wrong answer — "just point queries at E2 and keep the old index" — silently mixes incompatible vector spaces so nearest-neighbour results become meaningless with no error raised.

   </details>

4. **Which TWO** of the following are required to make a production LLM output *reproducible and auditable* after the fact?
   - A. The pinned model version that produced it
   - B. The end user's IP address
   - C. The exact prompt version (and index version, if RAG) used
   - D. The total token count of the response
   - E. The wall-clock latency of the request

   <details><summary>Answer</summary>

   **A and C.** Reproducing a past output means re-running the *same* inputs through the *same* components, so you need the pinned model version (A) and the exact prompt version — plus the index/embedder version for RAG (C). Together they are the version triple that defines behaviour. B, D, and E are useful operational/observability metadata but none of them determine what the model *produced*: IP address is identity/security context, token count and latency are performance signals. The most tempting wrong pick is D (token count), because it *describes* the output — but knowing how many tokens came back does not let you regenerate or explain the content; only the model+prompt+index versions do.

   </details>

5. A model upgrade passes your offline regression eval with a higher score than the current prod version. A colleague argues you can therefore promote it straight to 100% of traffic and skip a canary. Is skipping the canary justified? Why or why not?

   <details><summary>Answer</summary>

   No. A passing offline eval is necessary but not sufficient: the golden set is a finite sample and cannot cover the full distribution of live traffic, so an unseen input distribution, a latency/cost regression, or an interaction with real retrieval can still degrade production despite the higher offline score. The eval gate and the canary address different risks — the eval blocks *known/measurable* regressions, the canary *bounds the blast radius* of the ones the eval missed and validates on real traffic with a defined rollback trigger. The colleague's reasoning is the tempting distractor: it treats the eval score as a full production guarantee, when its real role is a promotion filter, not a substitute for a controlled rollout (links to `05-reliability-and-failure-handling-patterns.md` for the rollback path).

   </details>

---

## Further Reading

- [OpenAI — Model deprecations](https://platform.openai.com/docs/deprecations) — *verified 2026-07-29* — deprecation vs legacy vs sunset, minimum notice periods per model class, and dated shutdown/replacement tables — the concrete reason you must pin and plan migrations rather than assume a model stays available.
- [OpenAI — Models catalog](https://platform.openai.com/docs/models) — *verified 2026-07-29* — the current catalog with explicit model IDs and aliases, illustrating the difference between a pinned model ID and a convenience alias you must not float on in prod.
- [Anthropic — Models overview](https://docs.anthropic.com/en/docs/about-claude/models/overview) — *verified 2026-07-29* — notes that every Claude model ID is a pinned snapshot (dateless IDs are still fixed, not evergreen pointers) and lists legacy/deprecated models with retirement dates.
- [Anthropic — Model IDs and versioning](https://docs.anthropic.com/en/docs/about-claude/models/model-ids-and-versions) — *verified 2026-07-29* — how model IDs are structured, why a dateless ID is a fixed snapshot rather than a "latest" pointer, and that serving infrastructure can shift behaviour even when weights are fixed.
- [LangSmith — Manage prompts programmatically](https://docs.langchain.com/langsmith/manage-prompts-programmatically) — *verified 2026-07-29* — pushing/pulling prompts as versioned artifacts and pulling a specific commit hash (`prompt:12344e88`) — the prompt-registry / prompt-pinning pattern in practice.
- [MLflow — Model Registry](https://mlflow.org/docs/latest/ml/model-registry/) — *verified 2026-07-29* — versioning, lineage (which run/data produced a model), aliases (e.g. `@champion`) and validation-status tags for governed promotion and rollback of model/fine-tune versions.
- [pgvector — README](https://github.com/pgvector/pgvector) — *verified 2026-07-29* — vector index types (HNSW/IVFFlat), `REINDEX`, and storing multiple embedding versions in one table via per-`model_id` partial indexes — the mechanics behind re-indexing and dual-index migration.
