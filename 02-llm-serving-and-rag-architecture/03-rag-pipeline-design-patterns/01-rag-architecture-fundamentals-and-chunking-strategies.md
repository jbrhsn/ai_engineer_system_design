# RAG Architecture Fundamentals and Chunking Strategies

**Section:** 02 LLM Serving & RAG Architecture → RAG Pipeline Design Patterns | **Est. time:** 2.5 hrs | **Interview relevance:** High — "walk me through a RAG pipeline" and "how would you chunk these documents" are near-universal openers, and chunking is the single decision that most silently caps retrieval quality.

---

## TL;DR

Retrieval-Augmented Generation (RAG) grounds an LLM in an external corpus so it can answer from *your* current, private, citable data instead of its frozen parametric memory. The pipeline splits cleanly into an **indexing-time** path (load → chunk → embed → store) that runs offline, and a **query-time** path (embed query → retrieve top-k → augment the prompt → generate) that runs per request. Chunking is the pivotal indexing decision: chunks are the atomic units you embed and retrieve, so a chunk that is too big dilutes its own embedding, one that is too small loses the context needed to answer, and one that cuts a table or sentence in half poisons retrieval for every future query. **The one thing to remember: you never retrieve documents — you retrieve chunks, so the chunk boundary *is* the retrieval unit, and getting it wrong at index time caps every downstream metric no reranker can fully recover.**

---

## ELI5 — Explain It Like I'm 5

Imagine an open-book exam where you're allowed to bring a fat textbook, but you only have 30 seconds to find the right passage before you must answer. If someone has already gone through the book and stuck labelled sticky-tabs on coherent, self-contained sections — "how to change a tyre," "what tyre pressure to use" — you flip straight to the right tab, read one clean section, and answer correctly. If instead they tore the book into 500 identical scraps by counting letters, one scrap ends mid-sentence and the next begins mid-table, so even the right scrap doesn't make sense on its own and you fumble. RAG is the exam, the sticky-tabbing is chunking, and the flip-to-the-tab is retrieval. The common mistake is thinking RAG just means "paste the whole book into the answer box" — you can't, the box is too small and the model gets lost, so the real work is cutting the book into the right-sized, self-contained pieces *before* the exam ever starts.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain why RAG exists over pure parametric knowledge, citing grounding, freshness, private-data access, and source attribution.
- [ ] Draw the canonical RAG pipeline and correctly separate indexing-time stages from query-time stages.
- [ ] Compare fixed-size, recursive/structure-aware, and semantic chunking, and select one from a document type and a retrieval-quality goal.
- [ ] Set `chunk_size`, `chunk_overlap`, splitter type, and `top-k` from stated constraints, and justify each value.
- [ ] Design a parent-document (small-to-big) retrieval scheme and explain how it decouples the embedding unit from the context unit.

---

## Visual Overview

### RAG Pipeline — Index-Time vs Query-Time

```
INDEX-TIME  (offline, runs on ingest / on a schedule)
  Raw sources ──► Load ──► Chunk ──► Embed ──► Store
  (PDF, HTML,      to      to        to       in vector
   DB, API)     Documents  chunks    vectors  index + metadata
                                                   │
══════════════════════════════════════════════════│═══════════
                                                    ▼
QUERY-TIME  (online, runs per user request)      [Vector Index]
  User query ──► Embed query ──► Retrieve top-k ──►─┘
                                     │
                                     ▼
              Augment prompt (query + retrieved chunks + instructions)
                                     │
                                     ▼
                    LLM generate ──► Grounded, cited answer
```

### Chunking Strategy Comparison

```
FIXED-SIZE            RECURSIVE / STRUCTURE-AWARE      SEMANTIC
count N chars,        try big separators first,        embed sentences,
cut hard              fall back to smaller             cut where meaning
                                                        shifts
┌──────────────┐      ┌──────────────┐                ┌──────────────┐
│ ...pressure  │      │ ## Tyres     │                │ Topic A       │
│ should be 32 │◄cut  │ Pressure 32  │  respects      │ (3 sentences) │
├──────────────┤ mid- │ psi. Rotate  │  ¶ / heading   ├──────────────┤
│ psi. Rotate  │ line │ every 10k km │  boundaries    │ Topic B       │
└──────────────┘      └──────────────┘                └──────────────┘
 cheap, dumb           cheap, sane defaults            costs embeddings,
 breaks tables         RECOMMENDED default             best coherence
```

### Small-to-Big (Parent-Document) Retrieval

```
INDEX small child chunks (precise to embed/match)
   child_1 ─┐
   child_2 ─┼─► each child stores parent_id ──► docstore holds big parents
   child_3 ─┘

QUERY  ──► match child_2 (tight semantic hit)
                 │ look up parent_id
                 ▼
        return PARENT chunk (full section) to the LLM
   → matched precisely, answered with full context
```

---

## Key Concepts

### Why RAG Exists (grounding vs parametric knowledge)

**What it is.** RAG is an architecture that retrieves relevant text from an external store at query time and injects it into the prompt, so generation is *grounded* in retrieved evidence rather than only the model's internal ("parametric") weights.

**How it works mechanistically.** A base LLM answers purely from patterns compressed into its weights during training — that knowledge is frozen at the training cutoff, cannot include your private documents, and offers no verifiable source. RAG intercepts the request, runs a similarity search over a corpus you control, and prepends the top matches to the prompt. The model then does what it is good at — reading and synthesising the supplied text — instead of recalling from memory. This buys four things pure fine-tuning does not: **freshness** (update the index, not the weights), **private-data access**, **grounding** (answers trace to supplied text), and **citations** (you know which chunk produced the claim).

**Where it appears in real systems.** As documented in the LlamaIndex "Introduction to RAG," a query engine converts the query to an embedding, the vector store returns numerically similar nodes, and those nodes plus the query plus a prompt template go to the LLM. In LangChain's RAG tutorial the same split is stated explicitly: an offline *indexing* phase and an online *retrieve → generate* phase.

### The Canonical Pipeline Stages

**What it is.** The end-to-end sequence: ingest → chunk → embed → index (indexing-time), then retrieve → augment prompt → generate (query-time).

**How it works mechanistically.** *Load* pulls raw bytes from PDFs, HTML, databases, or APIs into `Document` objects. *Chunk* (a.k.a. split / node-parse) breaks large documents into smaller units because large chunks are harder to search and blow the context budget. *Embed* converts each chunk to a fixed-length vector capturing meaning. *Store* writes the chunk text, its vector, and metadata into a vector index for approximate-nearest-neighbour search. At query time the query is embedded with the *same* model, the index returns the top-k nearest chunks, those chunks are formatted into the prompt, and the LLM generates.

**Where it appears in real systems.** LangChain's tutorial names four indexing steps — Load, Split, Embed, Store — using `RecursiveCharacterTextSplitter`, an `Embeddings` model (e.g. `OpenAIEmbeddings(model="text-embedding-3-large")`), and a `VectorStore` (`InMemoryVectorStore`, `PGVector`, Pinecone, etc.), then a `Retriever` and a chat model at query time. The critical mental model: **indexing-time cost is paid once and amortised; query-time cost and latency are paid on every request** — so expensive work (LLM-based metadata extraction, semantic chunking) belongs at index time.

### Chunking Strategies

**What it is.** The rule that decides where one chunk ends and the next begins. Three families dominate: fixed-size, recursive/structure-aware, and semantic.

**How it works mechanistically.** *Fixed-size* counts N characters (or tokens) and cuts, ignoring meaning — cheap but it slices sentences and tables mid-way. *Recursive/structure-aware* splitting tries an ordered list of separators (e.g. `["\n\n", "\n", " ", ""]`), splitting on the largest natural boundary that keeps chunks under the size limit and only falling back to finer separators when a piece is still too big — this respects paragraph and heading structure. Structure-aware variants parse the format explicitly (Markdown headings, HTML tags, code syntax) so a chunk aligns to a section or function. *Semantic* chunking embeds sentences and inserts a break where the embedding distance between adjacent sentences spikes, so each chunk is topically coherent — at the cost of running an embedding model during ingestion.

**Where it appears in real systems.** LangChain documents `RecursiveCharacterTextSplitter` as "the recommended `TextSplitter` for generic text." LlamaIndex ships `SentenceSplitter` (default, respects sentence boundaries), `MarkdownNodeParser` / `HTMLNodeParser` / `JSONNodeParser` (structure-aware), `CodeSplitter` (syntax-aware via tree-sitter), and `SemanticSplitterNodeParser` (embedding-distance breakpoints).

### Chunk Size and Overlap Trade-offs

**What it is.** `chunk_size` sets how much text each chunk holds; `chunk_overlap` sets how many characters/tokens the end of one chunk repeats at the start of the next.

**How it works mechanistically.** A single embedding vector must summarise the whole chunk. A large chunk averages many topics into one blurry vector, so it matches many queries weakly (low precision) and wastes context tokens; a tiny chunk has a sharp vector but may omit the surrounding sentence a reader needs to interpret it (lost context, low recall of the full answer). Overlap is insurance against a fact landing exactly on a boundary: repeating a slice at the seam means a sentence split across two chunks still appears intact in at least one. Overlap costs storage and duplicate retrievals, so it is kept modest — a common starting ratio is ~10–20% of `chunk_size`.

**Where it appears in real systems.** LangChain's tutorial uses `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)`; LlamaIndex's `SentenceSplitter(chunk_size=1024, chunk_overlap=20)` shows a token-oriented default. The exact numbers are corpus-dependent and should be tuned against a retrieval eval set, not copied blindly.

### Parent-Document / Small-to-Big Retrieval

**What it is.** A pattern that embeds *small* child chunks for precise matching but returns a *larger* parent unit to the LLM for full context.

**How it works mechanistically.** Small chunks give crisp, well-targeted embeddings, so retrieval matches the right spot. But a small chunk alone often lacks the context to answer. Small-to-big decouples the two roles: you index the small children (each carrying a pointer to its parent), match on them at query time, then swap in the parent chunk (or a fixed sentence window) before sending to the model. You get the precision of small embeddings and the completeness of large context simultaneously.

**Where it appears in real systems.** LlamaIndex implements this via `SentenceWindowNodeParser` (embed one sentence, store a surrounding window in metadata, then swap it in with a `MetadataReplacementNodePostProcessor`) and via the `HierarchicalNodeParser` + `AutoMergingRetriever` (merge retrieved children back into their parent). The same idea underlies LangChain's parent-document retrieval pattern.

### Metadata Attachment and Contextual Chunk Headers

**What it is.** Structured fields (source URL, section title, page, date, author) and short descriptive prefixes stored alongside each chunk.

**How it works mechanistically.** Metadata enables *pre-filtering* — restrict the ANN search to `product == "X"` or `date > 2024` before scoring similarity — which cuts the candidate set and blocks cross-tenant leakage. It also powers citations, because each returned chunk knows its origin. A *contextual chunk header* prepends the document title or section path into the chunk text before embedding, so a chunk that merely says "It supports up to 128k tokens" still embeds and reads as being about "Model Y — Context Window." In LlamaIndex, `Node` objects inherit the parent `Document`'s metadata automatically, and extractors like `TitleExtractor`, `SummaryExtractor`, and `QuestionsAnsweredExtractor` can generate header/metadata fields at ingest.

**Where it appears in real systems.** LangChain's search tool prefixes each chunk with a `# Source:` header so the model can attribute claims; LlamaIndex's metadata-extractor modules run inside an `IngestionPipeline` alongside the node parser.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| `chunk_size` | Text per chunk (chars or tokens) | Start ~1000 chars / ~256–512 tokens for prose; go smaller (256–512 chars) for dense fact lookup/FAQ, larger for narrative/reasoning. Never exceed the embedding model's max input. |
| `chunk_overlap` | Repeated span at chunk seams | Set to ~10–20% of `chunk_size` (e.g. 200 for 1000). Raise if facts span boundaries; lower toward 0 for clean structure-aware splits to cut duplication. |
| Splitter type | How boundaries are chosen | Default to recursive/structure-aware (`RecursiveCharacterTextSplitter`, `MarkdownNodeParser`). Use semantic only when topic drift within sections hurts recall and you can afford ingest-time embeddings. |
| `separators` (recursive) | Ordered boundary hierarchy | Order coarse→fine (`["\n\n", "\n", ". ", " ", ""]`). For code/markdown use format-specific separators so functions/sections stay intact. |
| `top-k` | Chunks retrieved per query | Start k=3–5. Raise k when recall is the bottleneck (facts scattered across chunks); keep low to protect context budget and precision. Pair a high k with a reranker. |

---

### Worked Example: Requirement → Decision

**Given:** You are indexing a library of technical product PDFs — each mixes long prose, numbered procedures, and specification *tables*. Support engineers ask precise questions ("What is the max operating temperature of the X200?") and the answer often lives in a table cell. Goal is high retrieval precision with verifiable citations; hallucinating a spec is unacceptable.

- **Step 1 — Identify the goal.** Maximise retrieval precision so the *exact* passage (including the right table row) is surfaced, and every answer cites its source page — a wrong spec is a safety/liability problem.
- **Step 2 — Define inputs.** Heterogeneous PDFs: prose paragraphs, procedures, and tables where meaning depends on the table's header row and its title.
- **Step 3 — Define outputs.** Self-contained chunks whose embedding reflects one coherent topic, each carrying `source`, `page`, and `section_title` metadata, sized within the embedding model's input limit.
- **Step 4 — Apply constraints.** Tables must not be split mid-row (a half-table embeds meaninglessly); latency budget rules out semantic chunking of the *entire* corpus every request, but ingest is offline so index-time cost is acceptable; answers must be attributable.
- **Step 5 — Select the approach.** Use a **structure-aware parser** that extracts tables as atomic units (via a PDF/layout parser) and a recursive splitter for prose, attach `page`/`section_title` metadata plus a **contextual header** (e.g. "X200 › Specifications"), and layer **small-to-big retrieval** so a tight child match returns the full table/section. Rationale vs alternatives: naive fixed-size splitting would slice tables and destroy precision; whole-corpus semantic chunking adds ingest cost without solving the table-integrity problem, which structure-aware parsing addresses directly.

---

## Implementation

```python
# Scenario: index heterogeneous technical docs so each chunk stays a coherent
# unit and a fact split across a paragraph boundary is still fully present in
# at least one chunk. Recommended generic default per LangChain docs.
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,          # ~256-512 tokens of prose
    chunk_overlap=200,        # ~20% insurance against boundary-split facts
    separators=["\n\n", "\n", ". ", " ", ""],  # coarse -> fine
)
chunks = splitter.split_documents(docs)  # docs: list[Document]
# Metadata (e.g. {"source": url}) on each Document is inherited by its chunks,
# enabling citations and metadata pre-filtering at query time.
```

```python
# Anti-pattern: cut every N characters with no regard for structure. On a doc
# containing a spec table, this slices rows in half; each half embeds to a
# vague vector, so the right table is never the top-k match. Retrieval precision
# collapses and no reranker downstream can rebuild the destroyed table.
from langchain_text_splitters import CharacterTextSplitter

bad = CharacterTextSplitter(separator="", chunk_size=500, chunk_overlap=0)
chunks = bad.split_documents(docs)   # tables and sentences cut mid-way

# Correct approach: parse structure first (keep tables/sections atomic), then
# decouple the match unit from the context unit with small-to-big retrieval.
from llama_index.core.node_parser import SentenceWindowNodeParser

node_parser = SentenceWindowNodeParser.from_defaults(
    window_size=3,                               # embed 1 sentence...
    window_metadata_key="window",                # ...but keep 3 on each side
    original_text_metadata_key="original_sentence",
)
nodes = node_parser.get_nodes_from_documents(docs)
# At query time, a MetadataReplacementNodePostProcessor swaps the matched
# sentence for its full window before the chunk reaches the LLM:
#   precise match (small) + complete context (big).
```

---

## Common Pitfalls & Misconceptions

- **"RAG just means stuffing documents into the prompt."** — Beginners equate RAG with pasting text, because that is the visible last step. In reality the value is in *retrieval*: chunking, embedding, and top-k selection decide *which* few passages appear; the prompt can only hold what retrieval found, so most quality is won or lost before generation.
- **Chunks too large dilute relevance.** — It feels safer to include more text per chunk "so the answer is definitely in there," but one vector must summarise the whole chunk, so a multi-topic chunk matches many queries weakly and buries the answer among noise. Right model: one chunk ≈ one coherent idea, sized so its embedding is sharp.
- **Chunks too small lose context.** — Shrinking chunks for precise embeddings feels like it must help recall, but a 1-sentence chunk like "It requires 12V" is unusable without knowing what "it" is. Right model: keep the *match* unit small if you like, but return a larger *context* unit (small-to-big) so the LLM sees enough to answer.
- **Copying `chunk_size=1000, overlap=200` as universal truth.** — Tutorials show these numbers, so they get treated as law. They are a reasonable *starting point* for generic prose only; the right values depend on document type and query pattern and must be tuned against a retrieval eval set.
- **Forgetting to attach metadata at ingest.** — Metadata feels like an afterthought versus "getting retrieval working," but without `source`/`section` you cannot cite, cannot pre-filter, and cannot enforce tenant isolation. Right model: metadata is part of the chunk contract, not an add-on.

---

## Key Definitions

| Term | Definition |
|---|---|
| RAG | Architecture that retrieves external text at query time and injects it into the prompt so generation is grounded in a controllable corpus rather than only model weights. |
| Parametric knowledge | Information compressed into an LLM's weights during training; frozen at the training cutoff and not source-attributable. |
| Chunk / Node | The atomic unit of text that is embedded, indexed, and retrieved; a piece of a source Document, carrying inherited metadata. |
| Indexing-time | The offline pipeline (load → chunk → embed → store) run once per corpus update, not per query. |
| Query-time | The online pipeline (embed query → retrieve → augment → generate) run on every request. |
| Recursive splitting | Splitting that tries an ordered list of separators, using the largest natural boundary that keeps chunks under the size limit. |
| Semantic chunking | Splitting at points where the embedding distance between adjacent sentences spikes, producing topically coherent chunks. |
| chunk_overlap | The span of text repeated between consecutive chunks to prevent a fact from being lost on a boundary. |
| Small-to-big (parent-document) | Retrieval pattern that matches on small child chunks but returns a larger parent unit for context. |
| Contextual chunk header | A short descriptive prefix (title/section) prepended to a chunk before embedding so it is self-locating. |

---

## Summary / Quick Recall

- RAG beats pure parametric knowledge on freshness, private data, grounding, and citations — update the index, not the weights.
- The pipeline splits into index-time (load→chunk→embed→store, paid once) and query-time (embed→retrieve→augment→generate, paid per request).
- You retrieve *chunks*, not documents — the chunk boundary is the retrieval unit, so chunking caps everything downstream.
- Default to recursive/structure-aware splitting; reserve semantic chunking for when topic drift hurts recall and ingest cost is acceptable.
- Big chunks → blurry embeddings and diluted relevance; tiny chunks → lost context. Small-to-big retrieval gets both.
- `chunk_size` ~1000 chars / `overlap` ~10–20% is a *starting point*, not a law — tune against a retrieval eval set.
- Attach `source`/`section`/date metadata at ingest for citations, pre-filtering, and tenant isolation.

---

## Self-Check Questions

1. In the canonical RAG pipeline, which stages run at indexing-time versus query-time?

   <details><summary>Answer</summary>

   Indexing-time: load → chunk → embed → store (run offline, once per corpus update). Query-time: embed the query → retrieve top-k → augment the prompt → generate (run per request). The distinction matters because expensive work (semantic chunking, LLM metadata extraction) belongs at index-time where cost is amortised; the tempting error is to treat chunking as a per-query step — it is not, chunks are produced once and reused across all queries.

   </details>

2. You are building a RAG system over an internal API reference where answers are single precise facts (default values, return types). Retrieval keeps returning long, vaguely-relevant chunks and the model misses the exact value. What single change do you try first, and why?

   <details><summary>Answer</summary>

   Reduce `chunk_size` (and use structure-aware splitting so each API entry is its own chunk). Long chunks average many facts into one blurry embedding, so the exact-fact query matches weakly; smaller, entry-aligned chunks produce sharp embeddings that rank the right passage first. Raising `top-k` is the tempting wrong first move — it adds more noisy long chunks and burns context budget without fixing the root cause, which is chunk granularity.

   </details>

3. **Which TWO** of the following are genuine advantages of RAG over fine-tuning a model on the same corpus?
   - A. Knowledge can be updated by re-indexing without retraining the model.
   - B. It eliminates the possibility of hallucination entirely.
   - C. Answers can cite the specific source passage that produced them.
   - D. It removes the need to choose an embedding model.
   - E. It guarantees lower query latency than a non-RAG call.

   <details><summary>Answer</summary>

   **A and C.** RAG updates knowledge by changing the index, not the weights (A), and because answers are grounded in retrieved chunks, each claim can be traced to a source (C). B is wrong — RAG *reduces* but never *eliminates* hallucination; the model can still misread or ignore retrieved text. D is wrong — RAG *requires* an embedding model for both chunks and queries. E is the most tempting distractor but is false: RAG adds a retrieval step, so a single RAG call is generally *higher* latency than a bare LLM call, not lower.

   </details>

4. A teammate proposes semantic chunking for a 50-million-document web corpus refreshed hourly, arguing it gives the most coherent chunks. Evaluate the trade-off and state whether you'd approve it.

   <details><summary>Answer</summary>

   Likely reject for this corpus. Semantic chunking embeds sentences during ingestion to find topic-shift breakpoints, so its cost scales with corpus size *and* refresh frequency — 50M docs re-chunked hourly means enormous, recurring embedding cost and ingest latency. Coherence is real but marginal over recursive/structure-aware splitting for most web text. Approve semantic chunking only where topic drift within sections measurably hurts recall (e.g. dense mixed-topic prose) and ingest volume is modest. The trap is optimising chunk coherence in isolation while ignoring that indexing cost, unlike a one-off, is paid on every hourly refresh.

   </details>

5. Compare naive fixed-size chunking against small-to-big (parent-document) retrieval for a corpus of contracts where clauses are short but interpreting one clause requires its surrounding section. Which wins, and what is the mechanism?

   <details><summary>Answer</summary>

   Small-to-big wins. Fixed-size chunking forces one impossible choice: small chunks embed the clause precisely but strip the surrounding context needed to interpret it, while large chunks include context but produce blurry embeddings that match poorly. Small-to-big *decouples* the two — embed the short clause (a small child, precise match) but return the full parent section to the LLM (complete context). Mechanism: children carry a pointer to their parent; retrieval matches on children, then swaps in the parent (or sentence window) before generation. The tempting wrong answer is "just use large fixed chunks for context," which sacrifices the retrieval precision that surfaced the right clause in the first place.

   </details>

---

## Further Reading

- [Build a RAG agent with LangChain (RAG tutorial)](https://python.langchain.com/docs/tutorials/rag/) — *verified 2026-07-29* — Canonical indexing (Load/Split/Embed/Store) then Retrieve/Generate split, with `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)` and embeddings/vector-store choices.
- [Introduction to RAG — LlamaIndex](https://developers.llamaindex.ai/python/framework/understanding/rag/) — *verified 2026-07-29* — The five RAG stages (Loading, Indexing, Storing, Querying, Evaluation) and the Node/Document/Retriever concepts.
- [Node Parser Modules — LlamaIndex](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/) — *verified 2026-07-29* — `SentenceSplitter`, `MarkdownNodeParser`, `CodeSplitter`, `SemanticSplitterNodeParser`, and `SentenceWindowNodeParser` for small-to-big retrieval.
- [Node Parser Usage Pattern — LlamaIndex](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/) — *verified 2026-07-29* — How node parsers chunk Documents into Nodes and inherit metadata, standalone and inside an ingestion pipeline.
- [Metadata Extraction Usage Pattern — LlamaIndex](https://developers.llamaindex.ai/python/framework/module_guides/loading/documents_and_nodes/usage_metadata_extractor/) — *verified 2026-07-29* — `TitleExtractor`, `SummaryExtractor`, and `QuestionsAnsweredExtractor` for enriching chunk metadata / contextual headers at ingest.
