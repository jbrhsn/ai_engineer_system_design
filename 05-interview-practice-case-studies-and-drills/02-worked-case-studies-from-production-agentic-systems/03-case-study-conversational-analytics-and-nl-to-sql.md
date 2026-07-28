# Case Study — Conversational Analytics and NL-to-SQL

**Section:** 05 Interview Practice → Worked Case Studies from Production Agentic Systems | **Interview relevance:** High — "design a chatbot that answers business questions over our warehouse" is a canonical Applied-AI onsite prompt, and the multi-turn + accuracy angle is where most candidates fall apart.

---

## TL;DR

A conversational analytics assistant turns plain-English questions ("what were sales last quarter?" → "and by region?") into correct SQL against a data warehouse, executes it, and returns a clean answer plus a chart. The hard parts are **not** SQL syntax — they are (1) **carrying conversation context** so follow-ups like "and by region?" resolve against the previous turn, (2) **grounding generation in the real schema** so the model uses columns and joins that actually exist, and (3) **verifying the query executed and returned something sane** before you show it to a user. The winning architecture rewrites each turn into a standalone question, retrieves the relevant slice of schema plus similar few-shot examples, generates SQL, and runs an **execute → catch-error → repair** loop before formatting results. **The one thing to remember: the SQL generator's accuracy is capped by two upstream steps it does not control — how well you resolved the follow-up into a standalone question, and how well you grounded the prompt in the actual schema — so spend your design budget there, not on prompt-tuning the generator.**

---

## The Prompt

> "Design a conversational analytics assistant. Business users ask natural-language questions over a data warehouse — 'What was revenue by product category last quarter?' — and can ask follow-ups in context — 'and by region?', 'only for enterprise accounts', 'now show it as month-over-month'. The system must generate accurate SQL, execute it against the warehouse, and return clear results with an appropriate visualization. Target the assistant at non-technical analysts. Walk me through the architecture, how you keep multi-turn context, how you maximize SQL accuracy, and how you measure it."

Clarifying questions a strong candidate asks in the first two minutes:

- **Read-only?** Assume yes — analytics, no writes/DDL. (This narrows the security surface; the sibling case study covers RBAC/validation in depth.)
- **Schema scale?** Assume a warehouse with ~200 tables but a curated **semantic layer / catalog** exposing ~30 analytics-ready tables and metrics.
- **Latency budget?** Assume p95 < 8 s end-to-end for typical aggregations; heavy scans can stream/queue.
- **Who are the users?** Non-technical analysts — so ambiguity handling and clear "here's what I ran" transparency matter more than raw power.
- **Accuracy bar?** We must define and measure it — **execution accuracy** on a curated eval set, not vibes.

---

## Step 1 — Requirements & Scoping

### Functional requirements

- Accept a natural-language question and return: (a) a rendered answer, (b) the SQL that produced it, (c) an appropriate visualization (table / bar / line).
- **Multi-turn context**: resolve follow-ups ("and by region?", "just last month", "why?") against the prior turn(s).
- **Schema grounding**: only reference tables/columns that exist; pick correct joins and grain.
- **Ambiguity handling**: when a question is under-specified ("top customers" — by what, over what window?), either apply a documented default or ask one crisp clarifying question.
- **Self-correction**: if the generated SQL errors or returns obviously empty/degenerate results, repair and retry (bounded).
- **Transparency**: always show the executed SQL and a one-line plain-English restatement of what was computed.

### Non-functional requirements

| Dimension | Target | Why it drives design |
|---|---|---|
| **Accuracy** | ≥ 90% execution accuracy on curated eval set | The single number that makes or breaks trust; forces schema grounding + repair. |
| **Latency** | p95 < 8 s for aggregations | Budgets: context rewrite (~0.5 s) + retrieval (~0.2 s) + generation (~2–3 s) + execute (~1–4 s) + format (~0.5 s). |
| **Multi-turn** | Follow-ups resolve within same "thread" | Requires per-conversation state keyed by `thread_id`. |
| **Cost** | Bounded LLM calls/turn | Repair loop capped (e.g. 2 retries); schema retrieved, not dumped whole. |
| **Safety** | Read-only, row-limited, timeout-guarded | Even in an accuracy-focused build, never let generated SQL run unbounded. |

### Assumptions stated up front

- Warehouse is columnar (BigQuery / Snowflake / Redshift-class); SQL dialect is fixed and known to the generator.
- A **semantic catalog** exists (or we build a thin one): table descriptions, column descriptions, types, primary/foreign keys, and a handful of canonical metric definitions.
- We have (or can label) ~100–300 (question, SQL) pairs for few-shot retrieval and for the eval set.

---

## Step 2 — High-Level Architecture

The pipeline is a directed flow with one **loop** (self-repair). Conversation state is threaded through every turn.

```
                          ┌──────────────────────────────────────────────┐
                          │  CONVERSATION STATE (per thread_id)            │
                          │  • prior questions + resolved SQL              │
                          │  • entities/filters in scope (dims, dates)     │
                          │  • last result schema (columns returned)       │
                          └───────────────┬──────────────────────────────┘
                                          │ (read/write each turn)
 User turn                                ▼
 "and by region?" ──►  ┌────────────────────────────┐
                       │ 1. CONTEXT RESOLUTION        │  rewrite follow-up into a
                       │    (query rewrite / coref)   │  STANDALONE question:
                       └──────────────┬───────────────┘  "revenue by product
                                      │                    category AND region,
                                      ▼                    last quarter"
                       ┌────────────────────────────┐
                       │ 2. SCHEMA GROUNDING          │  retrieve relevant tables +
                       │    • RAG over table/column   │  columns + K few-shot
                       │      descriptions (catalog)  │  (question, SQL) examples
                       │    • few-shot example store  │
                       └──────────────┬───────────────┘
                                      ▼
                       ┌────────────────────────────┐
                       │ 3. SQL GENERATION            │  grounded prompt ──► candidate SQL
                       └──────────────┬───────────────┘
                                      ▼
                       ┌────────────────────────────┐        error / empty / degenerate
                       │ 4. EXECUTE (read-only,       │───────────────┐
                       │    LIMIT + timeout)          │               │
                       └──────────────┬───────────────┘               ▼
                                      │ success              ┌────────────────────┐
                                      │                      │ 5. SELF-REPAIR      │
                                      │◄─────────────────────│  feed error/empty   │
                                      │   (bounded retries)  │  back to generator  │
                                      ▼                      └────────────────────┘
                       ┌────────────────────────────┐
                       │ 6. RESULT FORMATTER          │  choose viz (table/bar/line),
                       │    + SQL echo + restatement  │  restate what was computed,
                       └──────────────┬───────────────┘  write result schema back to state
                                      ▼
                              Answer + chart + SQL
```

### Components named

- **Conversation memory / state** — a per-`thread_id` store (LangGraph checkpointer) holding prior turns, resolved SQL, in-scope filters/dimensions, and the columns returned last turn.
- **Context resolution (query rewrite)** — an LLM step that folds the follow-up into a self-contained question using the state.
- **Semantic catalog** — table/column descriptions, types, keys, and canonical metric definitions; the ground truth the generator is grounded against.
- **Few-shot / RAG over schema** — retrieve the *relevant* tables and the *most similar* labeled (question, SQL) examples instead of dumping the whole schema.
- **SQL generator** — LLM producing dialect-correct SQL from the grounded prompt.
- **Self-correction / repair loop** — execute, catch DB errors or empty results, feed the error text back, regenerate (bounded).
- **Result formatter** — picks visualization from result shape and echoes SQL + a plain-English restatement.

---

## Step 3 — Key Design Decisions & Trade-offs

### Decision 1 — Multi-turn context: rewrite to a standalone question vs. append raw history

- **Options.** (a) Pass the whole chat transcript to the SQL generator and hope it resolves "and by region?"; (b) run a dedicated **query-rewrite** step that emits a self-contained question first, then generate SQL from that.
- **Pick.** (b) Explicit query rewrite.
- **Why.** Coreference ("it", "that", "and by region?") and carried filters ("only enterprise accounts" from three turns ago) are lost when the generator has to juggle resolution *and* SQL synthesis in one shot. Isolating rewrite gives you a debuggable, evaluable artifact ("did we resolve the follow-up correctly?") separate from "did we write correct SQL?".
- **What you give up.** An extra LLM call (~0.5 s, small model is fine) and one more failure point — mitigated because a bad rewrite is easy to detect and log.

### Decision 2 — Schema grounding: full schema dump vs. retrieved schema + few-shot

- **Options.** (a) Paste all 30 table DDLs into the prompt; (b) retrieve only tables/columns relevant to the resolved question (RAG over the catalog) plus the top-K similar labeled examples.
- **Pick.** (b) Retrieve schema slice + few-shot examples.
- **Why.** Dumping everything burns tokens, raises latency, and *lowers* accuracy — the model gets distracted by irrelevant tables and picks wrong joins. Retrieval focuses attention; few-shot examples teach dialect quirks and your house join conventions far better than prose instructions.
- **What you give up.** Retrieval can miss a needed table (schema-linking recall). Mitigate by embedding column descriptions (not just names), over-retrieving (top-8 tables), and letting the repair loop recover from "unknown column."

### Decision 3 — Correctness: single-shot generation vs. execute-and-repair loop

- **Options.** (a) Generate SQL once, run it, surface whatever happens; (b) generate, execute in a read-only sandbox with `LIMIT` + timeout, and on error/empty result feed the failure back for a bounded repair.
- **Pick.** (b) Execute-and-repair.
- **Why.** The database is a free, perfect oracle for a whole class of errors — bad column names, type mismatches, invalid GROUP BY. Executing and reflecting on the error message recovers most syntactic/schema-linking failures automatically and is the single highest-ROI accuracy lever after schema grounding.
- **What you give up.** Extra latency/cost per repair. Cap retries (e.g. 2) and short-circuit on the first success. Note: repair fixes *errors*, not *silently-wrong* results — see Decision 5.

### Decision 4 — Ambiguity: guess silently vs. default-with-disclosure vs. always ask

- **Options.** (a) Silently pick an interpretation; (b) apply a documented default and *tell the user* what you assumed; (c) always ask a clarifying question.
- **Pick.** (b) Default-with-disclosure, escalating to (c) only when confidence is low or the choice materially changes the answer.
- **Why.** Non-technical users hate being interrogated on every turn, but silent guessing destroys trust when it's wrong. Applying a sensible default ("assuming revenue = net_revenue, last 90 days") and stating it lets users self-correct in one follow-up. Reserve a hard clarifying question for genuinely fork-in-the-road ambiguity (e.g. two plausible "customer" tables).
- **What you give up.** A default can still be wrong; disclosure is the safety valve, so the restatement is non-optional.

### Decision 5 — Trust: return the raw result vs. verify-before-present

- **Options.** (a) Format whatever the query returned; (b) run cheap sanity checks first (row count within sane bounds, no all-NULL key column, aggregate not obviously degenerate) and flag/repair suspicious output.
- **Pick.** (b) Verify-before-present.
- **Why.** A query can execute perfectly and be **plausibly wrong** — e.g. an inner join that silently drops rows, or a date filter off by a quarter. These never throw an error, so the repair loop won't catch them. Lightweight result checks plus the echoed SQL + restatement give the user the artifacts to notice.
- **What you give up.** You cannot fully verify semantic correctness without a reference; you buy *detectability*, not a guarantee. That's why the eval set (Step 4) is the real backstop.

---

## Step 4 — Deep Dive: Maximizing NL-to-SQL Accuracy Across Multi-Turn Context

This is the part interviewers push on. Accuracy is a chain: **rewrite quality × schema-linking recall × generation × repair**. Weak links dominate.

### 4.1 Schema linking is the accuracy bottleneck

Most NL-to-SQL failures are not "the model can't write SQL" — they're "the model referenced a column that doesn't exist" or "joined on the wrong key." **Schema linking** is the sub-problem of mapping question phrases → the right tables/columns/joins.

Mechanism to maximize recall:
- **Embed descriptions, not just identifiers.** Index each column as `table.column — <human description> — <type>`. A user says "revenue"; the column is `net_sales_amt`. Only the description bridges that gap.
- **Over-retrieve then let the model prune.** Retrieve top-8 candidate tables; a too-tight top-2 that misses a join table is unrecoverable, while extra tables cost only tokens.
- **Inject join keys explicitly.** Provide PK/FK relationships in the grounded prompt so the model doesn't invent `ON a.id = b.id`.

### 4.2 Follow-up resolution feeds the whole chain

"and by region?" is meaningless without the prior turn. The rewrite step reads conversation state and emits a standalone question. Two mechanisms make it robust:
- **Carry structured scope, not just text.** Persist the *resolved filters/dimensions* of the last turn (`{metric: net_revenue, time: last_quarter, group_by: [product_category]}`) so "and by region?" becomes a structured delta (`group_by += region`) that's trivially correct, with the free-text rewrite as fallback.
- **Reset detection.** Detect topic changes ("actually, show me headcount") so you don't drag stale filters into an unrelated question.

### 4.3 Self-repair turns the DB into a verifier

The execute step is a truth oracle for a large error class. On failure, feed the generator: the original question, the failing SQL, and the **verbatim DB error** ("column `regoin` does not exist"). Models are very good at fixing a named error. Bound it to ~2 retries; log every repair as a signal for which schema descriptions or few-shot examples to improve.

### 4.4 Measuring it: execution accuracy, not string match

Never grade NL-to-SQL by comparing SQL *strings* — two correct queries can differ syntactically. Grade by **execution**:

- **Execution accuracy** — run generated SQL and the gold SQL against a fixed snapshot; compare the returned result sets (order-insensitive where appropriate). This is the industry-standard metric and what Ragas's execution-based **DataCompy score** operationalizes (compare result DataFrames row/column-wise).
- **SQL semantic equivalence** — for cases where execution is expensive/infeasible, an LLM judge (Ragas `SQLSemanticEquivalence`) checks whether two queries are equivalent given the schema, absorbing harmless syntactic differences (`active = 1` vs `active = true`).
- **Multi-turn eval** — evaluate *conversations*, not just isolated questions: seed a thread, replay a scripted follow-up sequence, and score each turn's execution accuracy so you catch context-loss regressions specifically.

Track accuracy sliced by turn depth (turn 1 vs. turn 3+) — a drop with depth is the fingerprint of a context-resolution bug, not a generation bug.

---

## Failure Modes & Mitigations

| Failure | Why it happens | Mitigation |
|---|---|---|
| **Wrong join** (row explosion or silent drop) | Model guesses the join key; inner join silently drops unmatched rows | Inject PK/FK relationships into the grounded prompt; result-verify row counts; prefer few-shot examples that show the house join pattern. |
| **Hallucinated column** | Column not retrieved, so the model invents a plausible name | Execute-and-repair (DB says "unknown column"); improve column-description embeddings; over-retrieve schema. |
| **Lost follow-up context** ("and by region?" → full-table query) | Follow-up sent to generator without resolution; stale filters dropped | Dedicated query-rewrite step reading conversation state; carry *structured* scope deltas, not just text. |
| **Silently wrong-but-plausible result** | Query runs fine but wrong grain / off-by-a-quarter date filter | Verify-before-present checks; echo SQL + plain-English restatement; the curated execution-accuracy eval set is the real backstop. |
| **Ambiguity resolved wrong** | "top customers" defaulted to the wrong metric/window | Default-with-disclosure ("assuming net_revenue, last 90 days"); escalate to a clarifying question when the choice is a genuine fork. |
| **Runaway query** | Model writes an unbounded scan over a huge fact table | Enforce read-only role, mandatory `LIMIT`, and a statement timeout at the execution boundary. |

---

## Implementation Sketch

### Anti-pattern: single-shot NL→SQL with no grounding, no execution, dropped context

```python
# Anti-pattern: answer a conversational analytics turn by asking the model for SQL
# in one shot, with the raw chat history glued in, no schema, and no execution check.
# Breaks on: follow-ups ("and by region?"), hallucinated columns, and wrong-but-
# plausible output — none of which this code can detect.
def answer(question: str, history: list[str], llm, db) -> str:
    prompt = "\n".join(history) + f"\nUser: {question}\nWrite the SQL:"
    sql = llm.invoke(prompt).content          # no schema grounding -> invented columns
    rows = db.run(sql)                         # no LIMIT/timeout, no error handling
    return str(rows)                           # no viz, no SQL echo, no restatement
```

Why it breaks: the generator must resolve "and by region?" *and* recall the schema *and* write correct SQL in a single call. It routinely references columns that don't exist (`region` when the column is `sales_region`), and because nothing executes-and-checks, an error either 500s the user or a wrong result is shown as fact.

### Corrected: rewrite → ground → generate → execute-and-repair → format

```python
# Scenario: a conversational analytics turn where the follow-up ("and by region?")
# must resolve against prior context, generate schema-grounded SQL, and be verified
# by execution before we show it — because non-technical users trust whatever we return.
from langgraph.checkpoint.memory import InMemorySaver  # per-thread conversation state

def resolve_followup(question, state, llm):
    # Fold the follow-up into a STANDALONE question using structured scope from last turn.
    prior = state.get("scope", {})   # e.g. {"metric": "net_revenue", "time": "last_quarter",
                                     #       "group_by": ["product_category"]}
    return llm.invoke(
        f"Prior scope: {prior}\nFollow-up: {question}\n"
        "Rewrite as a single self-contained analytics question."
    ).content

def ground(standalone_q, catalog, examples):
    # Retrieve ONLY relevant tables/columns (descriptions embedded) + K similar (Q, SQL)
    # pairs — focused context beats dumping all 30 DDLs, which lowers accuracy.
    tables = catalog.retrieve_tables(standalone_q, k=8)      # includes PK/FK + descriptions
    shots  = examples.retrieve(standalone_q, k=3)            # dialect + house join patterns
    return tables, shots

def generate_and_repair(standalone_q, tables, shots, llm, db, max_retries=2):
    sql = llm.invoke(build_prompt(standalone_q, tables, shots)).content
    for _ in range(max_retries + 1):
        result = db.run_readonly(sql, limit=10_000, timeout_s=15)  # sandboxed oracle
        if result.ok and not result.suspicious():   # verify: not empty/degenerate
            return sql, result
        # Feed the verbatim DB error (or "empty result") back — the DB is the verifier.
        sql = llm.invoke(
            f"Question: {standalone_q}\nFailing SQL: {sql}\n"
            f"Error: {result.error or 'returned empty/degenerate result'}\nFix it:"
        ).content
    return sql, result  # surface best-effort + a caveat if still failing

def answer(question, thread_id, deps):
    state = deps.checkpointer.get(thread_id)
    standalone_q = resolve_followup(question, state, deps.llm_small)
    tables, shots = ground(standalone_q, deps.catalog, deps.examples)
    sql, result = generate_and_repair(standalone_q, tables, shots, deps.llm, deps.db)
    deps.checkpointer.put(thread_id, update_scope(state, standalone_q, result))
    # Formatter: pick viz from result shape, ECHO the SQL, and restate what we computed.
    return format_answer(result, sql, restatement=standalone_q)
```

The corrected version separates the concerns the anti-pattern conflated: resolution (evaluable on its own), grounding (focused retrieval), generation, execution-verification (the repair loop), and transparent presentation (SQL echo + restatement).

---

## How to Present This in 45 Minutes

- **0–5 min — Scope.** Ask the clarifying questions (read-only? schema scale? latency? users? accuracy bar?). State assumptions out loud. Write the accuracy target on the board — it anchors everything.
- **5–12 min — Requirements.** Split functional vs. non-functional; call out multi-turn, accuracy, latency budget, and the read-only safety floor.
- **12–22 min — Architecture.** Draw the flow: state → rewrite → ground → generate → execute-repair → format. Name each component and say one sentence on why it exists.
- **22–35 min — Deep dive.** Go to NL-to-SQL accuracy: schema linking as the bottleneck, structured-scope follow-up resolution, execute-and-repair, and **execution accuracy** as the metric. This is where you earn the offer.
- **35–42 min — Failure modes.** Walk 3–4 from the table, especially "silently wrong-but-plausible" and "lost follow-up context" — these show senior judgment.
- **42–45 min — Trade-offs recap.** Name what you gave up (extra LLM call for rewrite, retrieval-miss risk, repair latency) and how you'd measure whether the design is working.

---

## Interview Q&A Drill

<details>
<summary>Answer — "How would you measure accuracy, and why not just diff the SQL string?"</summary>

- Grade by **execution accuracy**: run the generated SQL and the gold SQL against a fixed warehouse snapshot and compare result sets (order-insensitive where appropriate). Two correct queries can differ in syntax, aliasing, or join style, so string diffing produces false negatives.
- Operationalize with Ragas's execution-based **DataCompy score** (compares result DataFrames row/column-wise → precision/recall/F1) as the primary metric.
- Add **SQL semantic equivalence** (LLM judge, e.g. Ragas `SQLSemanticEquivalence`) for cases where execution is too expensive; it absorbs harmless differences like `active = 1` vs `active = true`.
- Evaluate **conversations**, not isolated questions, and slice accuracy by turn depth to isolate context-loss bugs.
</details>

<details>
<summary>Answer — "Which two design choices most improve NL-to-SQL accuracy, and why those two?"</summary>

- **Schema grounding via focused retrieval + few-shot** and **execute-and-repair** are the two highest-ROI levers.
- Schema grounding fixes the dominant failure class (hallucinated columns / wrong joins) by putting the *right* tables, descriptions, and PK/FK relationships — plus house-style examples — in front of the model, instead of a distracting full-schema dump.
- Execute-and-repair turns the database into a free, perfect verifier for the entire class of executable errors; feeding the verbatim error back recovers most schema-linking mistakes automatically.
- The tempting wrong answer is "use a bigger model / prompt-tune the generator." That helps marginally, but the generator's ceiling is set upstream by grounding and by follow-up resolution — tuning it can't recover a column that was never retrieved or a follow-up that was never resolved.
</details>

<details>
<summary>Answer — "A user asks 'revenue by category last quarter', then 'and by region?' — trace what happens."</summary>

- Turn 1 resolves to `{metric: net_revenue, time: last_quarter, group_by: [product_category]}`, generates and executes SQL, and writes that structured scope + the returned columns into conversation state keyed by `thread_id`.
- Turn 2's **context-resolution** step reads the scope and treats "and by region?" as a structured delta: `group_by += region`. It emits the standalone question "net revenue by product category **and region**, last quarter."
- That standalone question flows through grounding (retrieve tables incl. the region dimension), generation, and the execute-repair loop exactly like a first turn.
- Carrying *structured* scope (not just chat text) is what makes the delta trivially correct; the free-text rewrite is the fallback if the structured path can't resolve it.
</details>

<details>
<summary>Answer — "The SQL executed with no error but the number looks wrong. How does your system handle that?"</summary>

- This is the **silently wrong-but-plausible** failure — the repair loop won't catch it because nothing threw an error.
- **Verify-before-present** runs cheap sanity checks (row count in sane bounds, no all-NULL key columns, aggregate not degenerate) and flags suspicious output.
- Always **echo the executed SQL** and a **plain-English restatement** ("net revenue, grouped by category, Q3 2025") so the user can catch a wrong grain or off-by-a-quarter filter.
- The durable backstop is the **curated execution-accuracy eval set**: you cannot fully verify semantic correctness at runtime without a reference, so you invest in offline evals that would have caught the wrong-join/wrong-grain pattern before it shipped.
</details>

<details>
<summary>Answer — "How do you keep multi-turn context without blowing up latency and cost?"</summary>

- Use a **per-`thread_id` checkpointer** (LangGraph persistence: short-term, thread-scoped memory) to store prior turns, resolved scope, and last result schema — not the raw growing transcript in every generator call.
- Run the **rewrite step on a small/cheap model**; it only needs to fold a follow-up into a standalone question, not write SQL.
- Retrieve a **schema slice**, never the full schema, so token cost stays flat as the warehouse grows.
- **Cap the repair loop** (e.g. 2 retries) and short-circuit on first success; prune old checkpoints so long conversations don't grow state unboundedly.
</details>

---

## Key Definitions

| Term | Definition |
|---|---|
| **NL-to-SQL (Text-to-SQL)** | Translating a natural-language question into an executable SQL query against a known schema. |
| **Schema linking** | Mapping phrases in the question to the correct tables, columns, and join keys — the dominant accuracy bottleneck. |
| **Context resolution / query rewrite** | Folding a follow-up turn ("and by region?") into a self-contained standalone question using conversation state. |
| **Conversation state / thread** | Per-conversation memory (keyed by `thread_id`) holding prior turns, resolved scope, and last result schema. |
| **Schema grounding** | Injecting relevant table/column descriptions, types, and PK/FK relationships (plus few-shot examples) into the generation prompt. |
| **Execute-and-repair loop** | Running generated SQL, catching DB errors/empty results, and feeding the failure back for bounded regeneration. |
| **Execution accuracy** | Correctness metric: does the generated query return the same result set as the gold query on a fixed snapshot. |
| **SQL semantic equivalence** | Whether two queries are logically equivalent given the schema, ignoring harmless syntactic differences. |
| **Semantic catalog** | Curated metadata layer: table/column descriptions, types, keys, and canonical metric definitions. |
| **Verify-before-present** | Cheap sanity checks on a returned result set before showing it, to catch plausible-but-wrong output. |

---

## Further Reading

- [Agents — LangChain (create_agent, thread_id conversation continuity)](https://docs.langchain.com/oss/python/langchain/agents) — *verified 2026-07-29*
- [Persistence — LangGraph (checkpointers for thread-scoped conversation memory)](https://docs.langchain.com/oss/python/langgraph/persistence) — *verified 2026-07-29*
- [SQL metrics — Ragas (execution-based DataCompy score, SQL semantic equivalence)](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/sql/) — *verified 2026-07-29*
