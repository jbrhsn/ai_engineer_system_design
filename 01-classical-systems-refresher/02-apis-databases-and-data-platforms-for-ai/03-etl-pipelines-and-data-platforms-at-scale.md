# ETL Pipelines and Data Platforms at Scale

**Section:** Classical Systems Refresher → APIs, Databases, and Data Platforms for AI | **Est. time:** 2.5 hrs | **Interview relevance:** High — every RAG/agentic design question eventually becomes "how does the data get in, stay fresh, and not cost a fortune?"

---

## TL;DR

ETL/ELT pipelines move data from source systems into a store where it can be queried, analysed, or embedded, and at scale they are run as orchestrated DAGs on a scheduler (e.g. Apache Airflow) with retries, backfills, and idempotency guarantees. For an Applied AI Engineer, the same machinery powers the RAG ingestion path — extract documents → chunk → embed → upsert into a vector store — and the hard problems shift to *incremental re-indexing* (only re-embed what changed) and *freshness-vs-cost* trade-offs. Batch is cheap and simple; streaming is fresh and expensive; a lakehouse tries to give you both storage models at once. **The one thing to remember: a data pipeline is only production-grade when re-running it produces the same result (idempotency) — for AI systems that means hash-based incremental upserts, never a blind full re-embed.**

---

## ELI5 — Explain It Like I'm 5

Imagine a librarian who each night takes the day's new and edited books, cuts long books into chapter-sized cards so they fit the card catalogue, writes a summary code on each card, and files it in the right drawer. If a book didn't change, she leaves its cards alone — she doesn't re-copy the whole book, because that wastes ink and time. She works from a checklist (the schedule) with steps that must happen in order: you can't file a card before you've cut it. If she gets interrupted and starts over, she must end up with exactly one correct card per chapter — not two copies — so she checks the summary code before filing. The common mistake beginners make is thinking the librarian should re-copy the *entire* library every night to "be safe"; the correct picture is that she only touches the books that actually changed and uses the code to prove she isn't duplicating work.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Compare ETL vs ELT and batch vs streaming, and select the right one under a stated freshness/cost constraint.
- [ ] Explain how a scheduler executes a DAG (dependencies, retries, backfills, data intervals) using Airflow as the reference implementation.
- [ ] Design an idempotent pipeline task so that re-runs and backfills do not duplicate or corrupt data.
- [ ] Distinguish a data lake, a data warehouse, and a lakehouse, and justify one for a given workload.
- [ ] Design an incremental RAG document-ingestion pipeline that re-embeds only changed content via hash-based upserts.

---

## Visual Overview

### RAG Ingestion Pipeline (Extract → Chunk → Embed → Index)

```
                                  ┌──────────────────────────┐
 Source docs ──► Extract/Load ──► │ hash each doc, compare to │
 (S3, wiki,      (parse PDF,      │ record manager            │
  DB, API)        HTML, md)       └──────────┬───────────────┘
                                             │ changed / new only
                                             ▼
                 Chunk (split) ──► Embed (embedding model) ──► Upsert into vector store
                 chunk_size=1000    text-embedding-3-large      (delete stale chunks
                 overlap=200                                     for changed sources)
```

### Batch vs Streaming Contrast

```
BATCH                                   STREAMING
┌───────────────────────┐               ┌───────────────────────┐
│ run on a schedule      │               │ react per-event        │
│ (e.g. @daily 02:00)    │               │ (as data arrives)      │
│ process a bounded set  │               │ process unbounded flow │
│ latency: minutes–hours │               │ latency: sub-second–s  │
│ cost: low (amortised)  │               │ cost: high (always on) │
│ freshness: stale       │               │ freshness: near-live   │
└───────────────────────┘               └───────────────────────┘
        chosen when history/cost win        chosen when freshness wins
```

### Orchestrated DAG (Airflow tasks + dependencies)

```
extract ──► validate ──► chunk ──► embed ──► upsert ──► reindex_check
                │                                            ▲
                └────────── on failure: retry (2x) ──────────┘
   scheduler creates one DAG Run per data interval; backfill fills past intervals
```

### Storage Decision Tree

```
Do you need cheap storage of raw / unstructured / multimodal data?
├── Yes ──► Do you ALSO need ACID + SQL + BI-grade query speed on it?
│           ├── Yes ──► Lakehouse (Delta Lake / Iceberg over object store)
│           └── No  ──► Data Lake (raw files on S3/GCS/ADLS)
└── No  ──► Structured, curated, heavy SQL analytics? ──► Data Warehouse (Snowflake/BigQuery)
```

---

## Key Concepts

### ETL vs ELT

**What it is.** ETL (Extract, Transform, Load) transforms data *before* loading it into the destination; ELT (Extract, Load, Transform) loads raw data first and transforms it *inside* the destination store. **How it works.** In ETL, a compute layer (a Python task, a Spark job) reshapes and cleans records in flight, so the warehouse only ever sees clean rows; in ELT, raw data lands in cheap storage and transformations run as SQL/`dbt` models against the warehouse's own engine, exploiting its elastic compute. The shift from ETL to ELT happened because cloud warehouses (Snowflake, BigQuery) made in-warehouse compute cheap and separable from storage. **Where it appears.** In an Airflow DAG, ETL is a `transform` task that runs before the `load` task; ELT is a `load_raw` task followed by SQL-based `transform` tasks that `CREATE TABLE AS SELECT` inside the warehouse. For AI, the "T" often includes chunking and embedding — which are expensive enough that you almost always want them *incremental*, not re-run wholesale.

### Batch vs Streaming

**What it is.** Batch processing operates on a bounded, finite dataset on a schedule; stream processing operates on an unbounded flow of events as they arrive. **How it works.** A batch job reads a partition (e.g. "yesterday's rows"), processes all of it, writes output, and exits; a streaming system (Kafka + Flink/Spark Structured Streaming) keeps a long-running process that consumes each event, updates state, and emits results continuously with windowing to bound aggregations. The core trade-off is freshness vs cost and complexity: streaming gives sub-second freshness but pays for always-on compute and exactly-once semantics; batch amortises cost across a big run but leaves data stale between runs. **Where it appears.** Airflow schedules batch DAGs (`schedule="@daily"`); a Delta Lake table can serve as both a batch table and a streaming source/sink, letting streaming ingest and batch historical backfill coexist. In RAG, most knowledge bases are re-indexed in nightly batches; only latency-critical sources (breaking support tickets, live pricing) justify a streaming embed path.

### Orchestration: DAGs and Schedulers

**What it is.** A DAG (directed acyclic graph) is a model that encapsulates a workflow — its tasks, their order/dependencies, and its schedule — with no cycles so execution always terminates. **How it works.** A scheduler (Airflow's) parses DAG definitions, and for each `schedule` interval creates a *DAG Run* tied to a *data interval* (a window of data the tasks operate on); tasks become *Task Instances* that run on workers only when their upstream dependencies satisfy the `trigger_rule` (default `all_success`). Airflow's *logical date* marks the intended start of the data interval, which is what makes a run reproducible: the task reads/writes the partition for *that* interval, not "now". **Where it appears.** You declare dependencies with `first_task >> [second, third]`; you set retries and shared config via `default_args={"retries": 2}`; the scheduler enforces `min_file_process_interval` when parsing DAG files, which is why heavy work must live inside task `execute()` bodies, not at the top level of the DAG file.

### Idempotency and Backfills

**What it is.** Idempotency means re-running a task produces the same end state — no duplicates, no corruption; a backfill creates runs for *past* dates so a new or fixed pipeline can populate historical intervals. **How it works.** Airflow's best-practice guidance is to treat tasks like database transactions: never `INSERT` (a retry duplicates rows) — use `UPSERT`; never read "the latest available data" — read a *specific partition* keyed by `data_interval_start`; never call `datetime.now()` inside critical logic, because it yields different output on each run. Backfill runs many DAG Runs across a date range (`airflow backfill create --dag-id ... --from-date ... --to-date ...`) with `--reprocess-behavior` (`none`/`failed`/`completed`) controlling whether existing runs are recreated, and `--max-active-runs` bounding concurrency. **Where it appears.** In AI ingestion this maps directly: an `upsert` into a vector store keyed by a deterministic chunk ID is idempotent; a hash-based record manager makes a full re-run a no-op for unchanged docs; a backfill re-embeds a date range after you change chunking parameters.

### Data Lake vs Warehouse vs Lakehouse

**What it is.** A data lake stores raw files (any format) cheaply on object storage; a data warehouse stores structured, curated tables optimised for SQL analytics; a lakehouse adds warehouse-grade guarantees (ACID transactions, schema enforcement, SQL) *on top of* lake storage. **How it works.** A lake is just files on S3/GCS/ADLS with no transactional layer, so concurrent writers can leave readers seeing inconsistent data; a warehouse ingests into a proprietary columnar engine with strong consistency but higher storage cost and less flexibility for unstructured/multimodal data; a lakehouse (Delta Lake, Apache Iceberg) writes a transaction log alongside the files, giving serializable isolation, `MERGE`-based upserts, time travel (versioned snapshots), and unified batch+streaming access over the same table. **Where it appears.** Delta Lake explicitly "unifies streaming and batch" and supports `MERGE` for change-data-capture and SCD operations; for AI, time travel gives reproducible training/eval snapshots, and cheap lake storage holds the raw documents while the vector store holds the derived embeddings.

### RAG Ingestion, Chunking, and Embedding Pipelines

**What it is.** The RAG ingestion pipeline is an ETL pipeline whose "transform" is document loading, chunking, and embedding, and whose "load" is an upsert into a vector store. **How it works.** Documents are loaded into a `Document` (page_content + metadata), split by a `RecursiveCharacterTextSplitter` that recursively breaks on separators (newlines, spaces) until each chunk hits a target size — a page is too coarse for retrieval, so splitting keeps relevant passages from being diluted; each chunk is passed to an embedding model (`embeddings.embed_query`) producing a fixed-length vector, and vectors are written with `vector_store.add_documents(...)`. Chunk size and overlap are the main quality levers: too large dilutes relevance, too small loses context; overlap preserves continuity across boundaries. **Where it appears.** `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200, add_start_index=True)` is the recommended generic splitter; `add_start_index=True` records each chunk's character offset for provenance. The embedding step is the cost and latency hotspot of the whole pipeline, which is why re-indexing must be incremental.

### Incremental Re-indexing (Hash-based Upsert)

**What it is.** Incremental re-indexing re-embeds and re-writes only the chunks whose source content changed since the last run, instead of rebuilding the whole index. **How it works.** LangChain's `index()` API uses a `RecordManager` (a timestamped set) plus a content hash (`key_encoder`, default `sha1`, with `sha256`/`sha512`/`blake2b` available) to track what is already in the store; on each run it hashes each document, skips unchanged ones, upserts new/changed ones, and — with `cleanup="incremental"` — deletes stale chunks tied to a `source_id_key` that were *not* seen this run. `cleanup="full"` deletes everything the loader didn't return (requires the loader to return the *entire* dataset); `scoped_full` deletes stale docs only for source IDs seen this run, useful when you can't load everything at once. **Where it appears.** `index(docs, record_manager, vector_store, cleanup="incremental", source_id_key="source")` — set `force_update=True` only when you deliberately re-embed everything (e.g. after switching embedding models); pick as large a `batch_size` as possible so chunks sharing a source ID land in one batch and avoid redundant work.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| `schedule` (Airflow) | How often a DAG Run is created | Use `@daily`/cron for batch freshness in hours; use `@continuous`/streaming only when freshness < minutes is a hard requirement. |
| `default_args={"retries": N}` | Automatic task retry count | Set 2–3 for tasks hitting flaky external APIs (embedding endpoints, source APIs); set 0 for tasks that must not partially re-run without idempotency. |
| `--reprocess-behavior` (backfill) | Whether existing runs in the range are recreated | Use `none` to only fill gaps; `failed` to retry only failures; `completed` to force full reprocessing after a logic change. |
| `--max-active-runs` (backfill) | Concurrent DAG Runs during backfill | Set to your embedding API's rate-limit headroom; too high triggers 429s, too low wastes the backfill window. |
| `chunk_size` | Target characters per chunk | Larger (≥1000) for narrative docs where context matters; smaller (~300–500) for dense FAQ/code where precise retrieval beats context. |
| `chunk_overlap` | Characters shared between adjacent chunks | ~10–20% of `chunk_size` so an answer spanning a boundary isn't cut in half; set 0 only when chunks are already semantically complete. |
| `cleanup` (`index()`) | How stale chunks are removed | `incremental` for continuous changing corpora; `full` only when the loader returns the entire dataset; `None` for append-only. |
| `key_encoder` (`index()`) | Hash used to detect changed content | Use `sha256`+ for adversarial/multi-tenant inputs; `sha1` is faster but not collision-resistant. |
| `batch_size` (`index()`) | Docs per indexing batch | Set as large as memory allows so chunks of one source share a batch and avoid redundant re-work. |

### Worked Example: Requirement → Decision

**Given:** A support-knowledge-base RAG system indexes ~50,000 help-centre articles. Editors change 200–500 articles per day. Answers must reflect edits within 24 hours. Embedding calls cost money and are rate-limited. Re-embedding all 50k articles nightly is wasteful and slow.

- **Step 1 — Identify the goal:** Keep the vector index consistent with the source within a 24h freshness window at minimum embedding cost, and make re-runs safe.
- **Step 2 — Define inputs:** The article corpus (with a stable `source` id and last-modified metadata per article) loaded each night; the current vector store; a record manager tracking prior content hashes.
- **Step 3 — Define outputs:** A vector store where exactly the changed articles' chunks are re-embedded and upserted, stale chunks for edited articles are deleted, and unchanged articles are untouched.
- **Step 4 — Apply constraints:** 24h freshness (so nightly batch, not streaming, is sufficient); bounded embedding cost (so no full re-embed); idempotency (a retried or backfilled run must not duplicate chunks); rate limits (so bounded concurrency).
- **Step 5 — Select the approach:** A nightly Airflow DAG (`schedule="@daily"`, `retries=2`) whose transform uses `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)` and LangChain `index(..., cleanup="incremental", source_id_key="source")` with a hash `key_encoder`. Rationale vs alternatives: streaming would meet freshness but wastes always-on cost for a 24h SLA; a nightly *full* re-embed meets correctness but blows the cost/rate-limit budget; hash-based incremental upsert meets freshness, cost, and idempotency simultaneously.

---

## Implementation

```python
# Scenario: nightly RAG re-indexing where re-embedding all 50k docs is too slow and
# too expensive; we must re-embed ONLY changed articles and never duplicate chunks
# even if the task retries or is backfilled. Idempotency comes from the record manager.
from langchain.indexes import SQLRecordManager, index
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_postgres import PGVector

splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200, add_start_index=True)
vector_store = PGVector(embeddings=OpenAIEmbeddings(model="text-embedding-3-large"),
                        collection_name="help_center", connection="postgresql+psycopg://...")

record_manager = SQLRecordManager("pg/help_center", db_url="postgresql+psycopg://...")
record_manager.create_schema()

def reindex(docs):
    chunks = splitter.split_documents(docs)            # transform: chunk
    result = index(                                    # load: hash-diff + upsert + prune
        chunks, record_manager, vector_store,
        cleanup="incremental",     # delete stale chunks for changed sources only
        source_id_key="source",    # group chunks by their originating article
    )
    return result   # {'num_added', 'num_updated', 'num_skipped', 'num_deleted'}
```

```python
# Anti-pattern: full re-embed on every run. This is NOT idempotent w.r.t. cost, and if
# it uses INSERT it also duplicates rows on retry. It re-embeds unchanged docs (50k calls
# nightly), blows the rate limit, and can leave duplicates if the run is retried midway.
def reindex_wrong(docs):
    vector_store.delete_collection()                      # nukes everything...
    chunks = splitter.split_documents(docs)
    vector_store.add_documents(chunks)                    # ...then re-embeds ALL 50k docs
    # Retry after a mid-run failure => partial delete + partial add => inconsistent index.

# Correct approach: let a record manager diff by content hash so unchanged docs are skipped,
# and use upsert semantics so a retry converges to one correct chunk per source.
def reindex_right(docs):
    chunks = splitter.split_documents(docs)
    return index(chunks, record_manager, vector_store,
                 cleanup="incremental", source_id_key="source")
    # num_skipped stays high (unchanged docs), num_added/num_updated tracks real edits,
    # and re-running immediately yields all-skipped: the definition of an idempotent load.
```

```python
# Scenario: an Airflow batch DAG that orchestrates the ingestion steps in order with
# retries, reading a specific data interval (never `now()`) so backfills are reproducible.
import datetime
from airflow.sdk import DAG
from airflow.providers.standard.operators.python import PythonOperator

with DAG(
    dag_id="rag_help_center_ingest",
    start_date=datetime.datetime(2026, 1, 1),
    schedule="@daily",
    catchup=False,
    default_args={"retries": 2},           # flaky embedding API => auto-retry
) as dag:
    extract = PythonOperator(task_id="extract", python_callable=lambda **c: load_changed(
        partition=c["data_interval_start"]))   # read THIS interval's partition, not latest
    embed_upsert = PythonOperator(task_id="embed_upsert", python_callable=run_incremental_index)
    extract >> embed_upsert                  # dependency: cannot embed before extract
```

---

## Common Pitfalls & Misconceptions

- **Thinking "re-run safe" means "re-run everything"** — Beginners equate correctness with rebuilding the whole index each run, because a full rebuild "can't be wrong." The correct mental model is idempotency: a re-run should converge to the same state cheaply, which a hash-diff + upsert achieves without re-processing unchanged data.
- **Using `INSERT` (or a delete-then-add) in a task** — Beginners write straightforward inserts because that's how they'd load data once. Airflow's own guidance is to treat tasks like transactions and use `UPSERT`; a retried `INSERT` duplicates rows and a mid-run delete-then-add leaves an inconsistent store.
- **Reading "the latest data" inside a task** — Beginners call `now()` or query the newest rows, assuming the run happens at "the current time." Because backfills and retries run for *past* logical dates, you must read the partition keyed by `data_interval_start`, or the same DAG produces different output on every run.
- **Confusing streaming with "always better because it's fresher"** — Beginners reach for streaming to look modern. Streaming pays always-on compute and exactly-once complexity; if the SLA is hours (most RAG corpora), a scheduled batch DAG is cheaper and simpler and still meets freshness.
- **Treating a data lake as a warehouse** — Beginners dump files in S3 and expect ACID/SQL guarantees. A raw lake has no transaction layer; if you need consistent concurrent reads/writes and `MERGE` upserts, use a lakehouse format (Delta/Iceberg) that adds a transaction log over the files.

---

## Key Definitions

| Term | Definition |
|---|---|
| ETL / ELT | Extract-Transform-Load (transform before loading) vs Extract-Load-Transform (load raw, transform inside the destination). |
| DAG | Directed acyclic graph modelling a workflow's tasks, dependencies, and schedule; acyclicity guarantees termination. |
| DAG Run / data interval | One scheduled execution of a DAG bound to a specific window of data the tasks operate on. |
| Logical date | The intended start of a DAG Run's data interval — the key to reproducible, backfill-safe runs (not wall-clock time). |
| Idempotency | Property that re-running an operation yields the same end state, with no duplication or corruption. |
| Backfill | Creating DAG Runs for past intervals to populate history after a new or changed pipeline. |
| Data lake / warehouse / lakehouse | Raw files on cheap object storage / curated structured tables for SQL analytics / warehouse-grade ACID+SQL layered over lake storage. |
| Chunking | Splitting a document into smaller retrieval-sized units (`chunk_size`, `chunk_overlap`) before embedding. |
| Incremental re-indexing | Re-embedding and upserting only the chunks whose source content changed, tracked by a record manager and content hash. |
| Record manager | A timestamped store of document hashes that lets the indexer skip unchanged docs and prune stale ones. |

---

## Summary / Quick Recall

- ETL transforms before load; ELT loads raw then transforms in-warehouse — cloud compute made ELT the default.
- Batch = cheap + stale on a schedule; streaming = fresh + expensive + always-on; choose by the freshness SLA.
- A scheduler runs a DAG as DAG Runs over data intervals; tasks fire on `trigger_rule` when upstreams succeed.
- Idempotency = re-runnable without duplication: use UPSERT, read a specific partition, never `now()`.
- Backfill fills past intervals; `--reprocess-behavior` and `--max-active-runs` control what/how fast.
- Lakehouse (Delta/Iceberg) adds ACID, MERGE upserts, and time travel over cheap lake storage; unifies batch + streaming.
- RAG ingestion is ETL: extract → chunk → embed → upsert; re-index incrementally via hash-based upsert, never a blind full re-embed.

---

## Self-Check Questions

1. What does it mean for an ETL task to be *idempotent*, and name one Airflow-recommended technique to achieve it?

   <details><summary>Answer</summary>

   Idempotency means re-running the task produces the same end state with no duplicates or corruption. An Airflow-recommended technique is to use `UPSERT` instead of `INSERT` (so a retry doesn't duplicate rows); reading a specific partition keyed by `data_interval_start` (rather than "latest") is another. The tempting-but-wrong answer "just add more retries" fails: retries without idempotency multiply the damage, because each retry re-runs a non-idempotent side effect.

   </details>

2. You maintain a RAG index over 50k docs where ~300 change daily and the freshness SLA is 24 hours. How should you re-index, and why not stream?

   <details><summary>Answer</summary>

   Run a nightly batch DAG with hash-based incremental indexing (`index(..., cleanup="incremental", source_id_key="source")`) so only the ~300 changed docs are re-embedded and stale chunks pruned. Streaming is the wrong choice here because a 24h SLA doesn't require sub-second freshness, and streaming would pay for always-on compute plus exactly-once complexity for no benefit. A nightly *full* re-embed is also wrong: it meets correctness but wastes ~50k embedding calls and can hit rate limits.

   </details>

3. **Which TWO** of the following make an Airflow DAG safe to backfill over past dates?
   - A. Calling `datetime.now()` inside the transform to timestamp rows
   - B. Reading and writing a partition keyed by `data_interval_start`
   - C. Using `UPSERT` semantics when writing results
   - D. Setting `schedule="@continuous"`
   - E. Doing expensive work at the top level of the DAG file

   <details><summary>Answer</summary>

   **B and C.** B ensures each backfilled run reads/writes the correct historical partition, so output depends on the logical date, not wall-clock time. C ensures a re-created run (backfill of an already-run date) converges to one correct row rather than duplicating. A is wrong because `now()` yields different output every run, breaking reproducibility — the most tempting distractor since timestamps feel harmless. D concerns freshness, not backfill safety; E hurts scheduler performance and is unrelated to correctness.

   </details>

4. A team stores raw PDFs and JSON in S3 and complains that concurrent jobs sometimes read half-written, inconsistent data. What storage change fixes this, and what do you gain beyond consistency?

   <details><summary>Answer</summary>

   Move from a raw data lake to a lakehouse format (Delta Lake or Apache Iceberg) that writes a transaction log over the files, giving ACID/serializable isolation so readers never see inconsistent data. Beyond consistency you gain `MERGE`-based upserts (for CDC/incremental loads), schema enforcement, and time travel (versioned snapshots for reproducible training/eval). Simply "move to a warehouse" is a weaker answer: it adds cost and loses the cheap, flexible storage of unstructured/multimodal data that the lake provided.

   </details>

5. Your nightly incremental re-index shows `num_skipped` dropping to near zero and `num_added` spiking, even though editors changed only a few docs. What likely happened, and how do you confirm vs fix?

   <details><summary>Answer</summary>

   Something changed the content hash for nearly every document — most likely you altered chunking parameters (e.g. `chunk_size`/`chunk_overlap`) or the embedding model, or switched the `key_encoder`, so the record manager sees every chunk as "new." Confirm by diffing the current splitter/model config against the previous run; if the change was intentional (e.g. new embedding model), this is expected and you'd set `force_update=True` for one full pass. If unintentional, revert the config so hashes match again. The tempting wrong diagnosis — "the source data all changed" — is unlikely when editors touched only a few docs; the hash inputs changed, not the source.

   </details>

---

## Further Reading

- [Apache Airflow — Dags (Core Concepts)](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html) — *verified 2026-07-28* — DAG declaration, dependencies, DAG Runs, data intervals, logical date, and trigger rules.
- [Apache Airflow — Backfill](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/backfill.html) — *verified 2026-07-28* — reprocess behavior, concurrency control, and reproducing past intervals.
- [Apache Airflow — Best Practices](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html) — *verified 2026-07-28* — idempotency guidance (UPSERT, specific partitions, avoiding `now()`), top-level code, and testing.
- [LangChain — Build a semantic search engine](https://docs.langchain.com/oss/python/langchain/knowledge-base) — *verified 2026-07-28* — document loading, `RecursiveCharacterTextSplitter` (chunk_size/overlap), embeddings, and vector-store indexing.
- [LangChain Core — `index()` API reference](https://reference.langchain.com/python/langchain-core/indexing/api/index) — *verified 2026-07-28* — incremental/full/scoped_full cleanup, record manager, `source_id_key`, `key_encoder`, and `batch_size`.
- [Delta Lake — Introduction](https://docs.delta.io/latest/delta-intro.html) — *verified 2026-07-28* — lakehouse architecture: ACID transactions, MERGE upserts, streaming+batch unification, and time travel.
