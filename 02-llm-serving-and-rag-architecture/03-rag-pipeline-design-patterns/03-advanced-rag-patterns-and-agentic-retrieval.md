# Advanced RAG Patterns and Agentic Retrieval

**Section:** 02 LLM Serving & RAG Architecture → RAG Pipeline Design Patterns | **Est. time:** 3 hrs | **Interview relevance:** High — the "how would you make this RAG system not answer wrongly / handle multi-step questions" follow-up separates senior candidates, and this is the bridge to the agentic-systems section.

---

## TL;DR

Naive RAG runs one fixed loop: embed the query, fetch top-*k*, stuff into the prompt, generate. Advanced RAG breaks that rigidity at three points — it *reshapes the query* before retrieval (rewriting, HyDE, multi-query, step-back), *routes* it to the right index, and *checks the retrieved evidence* before trusting it (corrective/self-RAG grading loops). Agentic retrieval goes furthest: the LLM itself decides *whether*, *what*, and *how many times* to retrieve, treating the retriever as a callable tool inside a controlled loop. Every one of these buys accuracy with extra LLM calls, latency, and failure surface, so the engineering skill is knowing which knob to add and where to cap it. **The one thing to remember: each advanced pattern is an extra LLM round-trip you are paying for — add it only when a measured retrieval failure justifies the latency and cost, and always bound any loop.**

---

## ELI5 — Explain It Like I'm 5

Imagine a librarian helping you answer a hard question. A lazy librarian grabs the first three books off the shelf matching your exact words and reads them to you, even if they are useless — that is naive RAG. A good research librarian instead *rephrases your fuzzy question into a few sharper ones*, walks to the *right section* of the library instead of the whole building, and — crucially — *skims the books first and, if they are off-topic, goes back and searches again* before answering. An expert librarian will even decide that your question ("what time do you close?") needs *no books at all*, and just answers from memory. The common mistake is thinking the expensive research librarian is always the better choice: if your question is simple and the first shelf already has the answer, sending the librarian on three extra trips just makes you wait longer and costs more for the same answer.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain and choose among query-transformation techniques (rewriting, HyDE, multi-query, step-back) based on the failure they fix.
- [ ] Design a query router and a self-querying retriever that extract metadata filters from natural language.
- [ ] Implement a corrective/self-RAG grade→(re-search)→generate loop and articulate why it reduces hallucination.
- [ ] Design an agentic retrieval system where the LLM calls retrieval as a tool, and bound its iteration to control cost/latency.
- [ ] Compare advanced RAG patterns against naive RAG on the accuracy vs latency/cost/complexity axes and justify when *not* to add machinery.

---

## Visual Overview

### Query Transformation Fan-Out (multi-query / HyDE)

```
                        ┌──► variant 1 ──► retrieve ──┐
user query ──► LLM ─────┼──► variant 2 ──► retrieve ──┼──► dedup / fuse ──► top-k ──► generate
 (rewrite/expand)       └──► variant 3 ──► retrieve ──┘

HyDE special case:
user query ──► LLM writes a FAKE ideal answer ──► embed that answer ──► retrieve real docs
              (hypothetical doc bridges the question↔answer vocabulary gap)
```

### Corrective / Self-RAG Loop (retrieve → grade → decide)

```
query ──► retrieve ──► GRADE each doc (relevant? yes/no)
                          │
              ┌───────────┴────────────┐
        all/most relevant          not relevant
              │                         │
              ▼                         ▼
          generate                rewrite query ──► retrieve again
              │                         │  (bounded by max_iterations)
              ▼                         └────────────┐
           answer                                    ▼
                                          give up / fallback answer
```

### Agentic Retrieval-as-a-Tool Loop

```
              ┌─────────────────────────────────────────┐
              ▼                                           │
user ──► [ AGENT: model.bind_tools([retriever]) ]        │ (loop, capped)
              │                                           │
     tool_calls present? ──► YES ──► run retriever ──► ToolMessage ──┘
              │
              └► NO ──► answer directly (no retrieval)
```

### When Do I Add Advanced Machinery? (decision tree)

```
Is naive top-k retrieval already meeting your recall + faithfulness targets?
├── Yes ──► STOP. Ship naive RAG. Adding loops only adds latency/cost.
└── No  ──► What is the observed failure?
            ├── query wording misses good docs   ──► query transformation (rewrite/HyDE/multi-query)
            ├── right docs live in wrong index    ──► query routing / self-querying filters
            ├── retrieves irrelevant docs, still answers ──► corrective/self-RAG grading loop
            └── needs several dependent lookups   ──► agentic / multi-hop retrieval (capped)
```

---

## Key Concepts

### Query Transformation (rewriting, HyDE, multi-query, step-back)

**What it is.** A family of techniques that rewrite or expand the user's raw query *before* retrieval so the embedding better matches the stored chunks.

**How it works mechanistically.** The gap advanced RAG attacks is that a user's phrasing and the document's phrasing often embed to different regions of vector space. **Query rewriting** uses an LLM to clean a conversational/ambiguous query into a standalone search query (resolving "it", "that one", etc.). **Multi-query expansion** asks the LLM for N paraphrases, retrieves for each, then de-duplicates/fuses the union — this raises recall by covering multiple phrasings. **HyDE (Hypothetical Document Embeddings)** flips the problem: it asks the LLM to *write a fake ideal answer*, embeds that hypothetical answer, and retrieves against it, because an answer embeds closer to real answer-chunks than a question does. **Step-back prompting** asks a more general "background" question first (e.g. "what are the general rules for X?") to retrieve foundational context before the specific query.

**Where it appears in real systems.** In LangChain, `MultiQueryRetriever` wraps a base retriever and an LLM to auto-generate variants; the LangGraph agentic-RAG tutorial implements rewriting explicitly as a `rewrite_question` node that reformulates the query and routes back to the retrieval step. Each variant is an extra embedding call (cheap) but multi-query multiplies your vector-DB QPS by N.

### Query Routing across indexes / data sources

**What it is.** A decision step that picks *which* index, tool, or data source (or several) should handle a query, instead of always hitting one vector store.

**How it works mechanistically.** A router is an LLM given the query plus a set of "choices," each described by a natural-language description of what it is good for; the LLM (via text completion or, more reliably, a function-calling/structured-output schema) selects one or more. A *single-selector* picks one route; a *multi-selector* fans out to several and merges results. This lets you keep, say, a policy-docs index, a product-catalog index, and a SQL tool separate and route "how much does plan X cost?" to the catalog and "what's our refund policy?" to the docs.

**Where it appears in real systems.** LlamaIndex ships `RouterQueryEngine` / `RouterRetriever` with `PydanticSingleSelector`, `PydanticMultiSelector`, `LLMSingleSelector`, and `LLMMultiSelector`, each fed a list of `QueryEngineTool`/`RetrieverTool` objects whose `description` field is what the selector reasons over. The quality of routing is almost entirely determined by how well you write those descriptions.

### Self-Querying / Metadata-Filter Extraction

**What it is.** Turning a natural-language query into *both* a semantic search string *and* a structured metadata filter (e.g. `year >= 2023 AND author = "Weng"`), so retrieval combines similarity with exact attribute filtering.

**How it works mechanistically.** An LLM is given the query plus a schema describing the available metadata fields and their types, and it emits a structured object: the semantic sub-query and a set of filter predicates. The vector store then applies the filter as a pre- or post-filter alongside ANN search. This solves queries that mix "about topic Y" (semantic) with hard constraints ("published after 2023," "from the finance department") that pure similarity ignores.

**Where it appears in real systems.** This is LlamaIndex's "auto-retrieval" (`VectorIndexAutoRetriever` with a `VectorStoreInfo` schema) and LangChain's `SelfQueryRetriever`; the extracted filter becomes the `filter=`/metadata-filter argument passed to Pinecone, Qdrant, Weaviate, or `pgvector`. A high-cardinality field (e.g. thousands of author names) needs the valid values injected into the prompt or the LLM will invent filter values that match nothing.

### Corrective RAG (CRAG) and Self-RAG (grade → regenerate loops)

**What it is.** Patterns that insert a *grading* step between retrieval and generation: the system judges whether retrieved documents are relevant/sufficient and, if not, corrects course (re-search, rewrite, or fall back to web search) before generating.

**How it works mechanistically.** After retrieval, a grader LLM scores each document's relevance to the question (often a binary yes/no via structured output). **Corrective RAG (CRAG)** uses that grade to trigger a corrective action — commonly discarding irrelevant chunks and doing a supplemental web search — before generating. **Self-RAG** additionally has the model reflect on its own generation (is the answer supported by the context? is it useful?) and regenerate if not. The net effect: the system is allowed to say "these docs are bad, try again" instead of confidently answering from irrelevant context, which is a primary hallucination source.

**Where it appears in real systems.** The LangGraph agentic-RAG tutorial implements this exact loop: a `grade_documents` conditional edge uses a model with a `GradeDocuments` Pydantic schema (`binary_score: 'yes'|'no'`) to route to either `generate_answer` (relevant) or `rewrite_question` (not relevant), which then loops back. The grading model is a separate LLM call per retrieval, so grading roughly doubles per-turn LLM cost.

### Agentic Retrieval (retrieval as a tool)

**What it is.** Instead of a fixed pipeline, the LLM is an agent that *decides whether to retrieve at all*, chooses what to search for, reads results, and may retrieve again — retrieval is one tool among possibly several.

**How it works mechanistically.** The retriever is wrapped as a tool and bound to the model (`model.bind_tools([retriever_tool])`). On each turn the model either emits a normal answer (no retrieval needed — e.g. "hello!") or emits a tool call with a search query it composed itself. A conditional edge inspects whether the last message has `tool_calls`: if yes, a `ToolNode` runs the retriever and appends a `ToolMessage`; the loop returns to the model, which can retrieve again or answer. What makes it "agentic" rather than a fixed retrieve-then-generate pipeline is precisely that retrieval runs *only when the model requests it*.

**Where it appears in real systems.** The LangGraph Graph API expresses this with `StateGraph(MessagesState)`, `add_node`, `add_conditional_edges`, `ToolNode`, and `compile()`; the model node is `generate_query_or_respond`. This is the direct on-ramp to the multi-agent systems covered in Section 03 — an agent that retrieves is the simplest tool-using agent.

### Multi-hop Retrieval and GraphRAG (brief)

**What it is.** Multi-hop retrieval answers questions that require chaining several dependent lookups ("Who was the CEO of the company that acquired X in the year Y released Z?"). GraphRAG retrieves over a knowledge graph of entities and relationships rather than flat text chunks.

**How it works mechanistically.** In multi-hop, the answer to the first retrieval becomes the query for the second; an agentic loop naturally supports this because the model reads hop-1's `ToolMessage` and composes hop-2's query. **GraphRAG** first *builds* a property graph by having an LLM extract `(entity, relation, entity)` triples from chunks, then retrieves by traversing relationships and/or by vector-matching graph nodes — this excels at "connect-the-dots" and global-summary questions that flat top-*k* fragments.

**Where it appears in real systems.** LlamaIndex's `PropertyGraphIndex` builds graphs with `kg_extractors` (`SimpleLLMPathExtractor`, `SchemaLLMPathExtractor` with an entity/relation schema) and retrieves via `index.as_retriever(include_text=True, similarity_top_k=...)`. Graph construction is expensive (an LLM extraction pass over every chunk), which is the main trade-off.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Number of query variants (multi-query) | How many paraphrases are generated and retrieved | Start at 3; raise toward 5 only if recall is still short and your vector DB can absorb N× QPS — each variant is an extra retrieval. |
| `max_iterations` (retrieve/grade loop) | Hard cap on re-search cycles before forced answer/fallback | Set to 2–3; an uncapped loop is the #1 runaway-cost bug in agentic RAG. Below 2 you lose the corrective benefit. |
| Grading threshold / mode | How strictly retrieved docs are judged relevant | Use strict binary yes/no when wrong answers are costly (legal/medical); loosen to a score threshold when recall matters more than precision. |
| Router selector type | Single vs multi, LLM vs Pydantic/function-calling | Prefer Pydantic (function-calling) selectors for reliable parsing; use multi-selector only when a query legitimately spans sources, since it multiplies retrieval cost. |
| Router choice descriptions | The text the router reasons over to pick a route | Write one crisp sentence per route stating *when to use it*; vague descriptions are the top cause of misrouting — not the model. |
| `similarity_top_k` (per hop) | Docs pulled each retrieval in a multi-hop/graph loop | Keep small (2–4) per hop; multi-hop stacks context fast and blows the context window + cost if each hop over-fetches. |

### Worked Example: Requirement → Decision

**Given:** You are building a Q&A assistant over a bank's internal policy corpus. Compliance requires that a *wrong* answer is far more damaging than a *slow* or *"I don't know"* answer. Naive top-*k* RAG currently answers ~15% of questions from irrelevant chunks (it retrieves something loosely similar and confidently answers anyway). Latency budget is generous (up to ~6 s), cost is secondary to correctness.

- **Step 1 — Identify the goal:** Cut confidently-wrong answers by refusing to generate from irrelevant context, without needing perfect first-pass retrieval.
- **Step 2 — Define inputs:** User question (often conversational), a single policy-docs vector index with `department` and `effective_year` metadata, an LLM for generation and grading.
- **Step 3 — Define outputs:** Either a grounded answer citing relevant chunks, or an explicit "not covered in policy" fallback — never an answer built on graded-irrelevant chunks.
- **Step 4 — Apply constraints:** Hallucination tolerance ≈ zero; latency ≤ 6 s allows ~2 extra LLM round-trips; cost secondary; auditability required (must log why an answer was/ wasn't produced).
- **Step 5 — Select the approach:** Use **corrective RAG** — add a `grade_documents` step after retrieval that routes relevant docs to generation and irrelevant ones to a single query rewrite + re-retrieve, capped at `max_iterations = 2`, falling back to "not covered" if still ungraded-relevant. *Rationale vs alternatives:* full agentic multi-tool retrieval is unnecessary (single index, no multi-hop) and adds latency/failure surface; pure multi-query raises recall but doesn't stop answering from bad context — only the grading loop directly targets the observed "answers from irrelevant chunks" failure, which is exactly what the compliance requirement forbids.

---

## Implementation

```python
# Scenario: A support bot over one policy index answers "hi" and "reset my password"
# equally by always retrieving. We want retrieval to fire ONLY when the model judges
# it needs documents, so trivial turns skip the vector DB entirely (latency + cost win).
# API verified against LangGraph Graph API / agentic-RAG tutorial (docs.langchain.com).
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode
from langchain.chat_models import init_chat_model
from langchain.tools import tool

model = init_chat_model("openai:gpt-4o-mini", temperature=0)

@tool
def retrieve_policies(query: str) -> str:
    """Search internal policy documents and return relevant passages."""
    docs = _get_retriever().invoke(query)         # your vector store retriever
    return "\n\n".join(d.page_content for d in docs)

def generate_query_or_respond(state: MessagesState):
    # Model decides: answer directly, OR emit a tool call to retrieve.
    response = model.bind_tools([retrieve_policies]).invoke(state["messages"])
    return {"messages": [response]}

def route_on_tool_calls(state: MessagesState):
    last = state["messages"][-1]
    return "retrieve" if getattr(last, "tool_calls", None) else END

workflow = StateGraph(MessagesState)
workflow.add_node(generate_query_or_respond)
workflow.add_node("retrieve", ToolNode([retrieve_policies]))
workflow.add_edge(START, "generate_query_or_respond")
workflow.add_conditional_edges(
    "generate_query_or_respond", route_on_tool_calls,
    {"retrieve": "retrieve", END: END},
)
workflow.add_edge("retrieve", "generate_query_or_respond")
graph = workflow.compile()
```

```python
# Anti-pattern: a self-RAG "keep re-searching until the docs look good" loop with NO cap.
# When retrieval genuinely can't find the answer (topic not in corpus), grade_documents
# returns "no" forever, the graph rewrites+re-retrieves+re-grades on every cycle, and a
# single user turn can burn dozens of LLM calls and hang past any latency budget.
def grade_documents(state):                       # BROKEN: unbounded
    if _relevant(state):
        return "generate_answer"
    return "rewrite_question"                      # -> retrieve -> grade -> rewrite -> ...

# Correct approach: track an iteration count in state and force a fallback at the cap.
from typing import Literal

class RagState(MessagesState):
    retries: int                                   # extra field on the state schema

MAX_RETRIES = 2

def grade_documents(state: RagState) -> Literal["generate_answer", "rewrite_question", "give_up"]:
    if _relevant(state):
        return "generate_answer"
    if state.get("retries", 0) >= MAX_RETRIES:     # bound the loop
        return "give_up"                           # -> node that returns "not covered in policy"
    return "rewrite_question"

def rewrite_question(state: RagState):
    # increment the counter every time we loop back, so the cap is actually enforced
    return {"messages": [_rewritten(state)], "retries": state.get("retries", 0) + 1}
# What breaks without this: cost and latency are unbounded and non-deterministic; the fix
# makes worst-case cost = MAX_RETRIES + 1 retrievals and guarantees termination.
```

---

## Common Pitfalls & Misconceptions

- **Adding agentic complexity where naive RAG suffices** — teams reach for CRAG/agents because they are the "advanced" answer, assuming more machinery is strictly better. It isn't: every added step is an LLM round-trip that raises latency, cost, and the number of things that can fail; add a pattern only when a *measured* retrieval failure (low recall, answering-from-irrelevant, multi-hop miss) justifies it.
- **Unbounded retrieve/grade/agent loops** — beginners implement the retry loop without a cap because the happy path always terminates in testing. In production, a query the corpus simply can't answer makes the loop rewrite-and-retry forever; always thread a `max_iterations`/`retries` counter through state and force a fallback branch.
- **Blaming the router model for misrouting** — when a router picks the wrong index, people swap to a bigger LLM. The real cause is almost always vague tool/index *descriptions*: the selector reasons over your one-line description, so a precise "use this for refund and cancellation policy questions" fixes far more misroutes than a smarter model.
- **Treating HyDE as free accuracy** — HyDE is assumed to always help because it "understands intent," but it adds a full generation call *and* can hallucinate a misleading hypothetical that pulls retrieval off-topic for niche/factual queries; it helps most when the question↔answer vocabulary gap is large, not universally.
- **Self-querying with unconstrained filter fields** — extracting metadata filters is treated as safe, but on high-cardinality fields the LLM invents values (`author = "J. Smith"` when the corpus stores `"Smith, John"`), returning zero results; inject the valid values or normalize before filtering.

---

## Key Definitions

| Term | Definition |
|---|---|
| Query transformation | Rewriting/expanding the user query before retrieval to close the question↔document embedding gap (rewrite, multi-query, HyDE, step-back). |
| HyDE | Hypothetical Document Embeddings — generate a fake ideal answer, embed *it*, and retrieve against that embedding instead of the question. |
| Step-back prompting | Retrieving on a more general "background" question first to gather foundational context before the specific query. |
| Query routing | An LLM decision that selects which index/tool/data source (single or multi) should serve a query, based on their descriptions. |
| Self-querying / auto-retrieval | Extracting both a semantic sub-query and a structured metadata filter from natural language, then filtering the vector search. |
| Corrective RAG (CRAG) | Grading retrieved docs and taking a corrective action (re-search / web fallback) before generating, rather than answering from bad context. |
| Self-RAG | RAG where the model reflects on retrieval relevance *and* on its own generation's groundedness, regenerating when unsupported. |
| Agentic retrieval | Retrieval exposed as a tool the LLM chooses to call — the model decides whether, what, and how many times to retrieve. |
| Multi-hop retrieval | Chaining dependent retrievals where each hop's result seeds the next hop's query. |
| GraphRAG | Retrieval over an LLM-constructed knowledge graph of entity/relation triples instead of flat text chunks. |

---

## Summary / Quick Recall

- Naive RAG = one fixed loop; advanced RAG intervenes at the query, the routing, and the evidence-checking stages.
- Query transformation (rewrite/multi-query/HyDE/step-back) fixes *"the right docs exist but the query doesn't match them."*
- Routing + self-querying fix *"the right docs are in another index / behind a metadata filter."*
- CRAG/Self-RAG add a grade step so the system re-searches or refuses instead of answering from irrelevant context — the main hallucination cure.
- Agentic retrieval lets the model decide *whether/what/when* to retrieve via `bind_tools` + a conditional-edge loop; it's the on-ramp to multi-agent systems.
- Every pattern trades accuracy for extra LLM calls, latency, and failure surface — add on evidence, and **always cap loops** with `max_iterations`.

---

## Self-Check Questions

1. What is the core mechanism of HyDE, and how does it differ from ordinary query rewriting?

   <details><summary>Answer</summary>

   HyDE has the LLM generate a *hypothetical answer* to the query, embeds that answer, and retrieves against it — the premise being that an answer embeds closer to real answer-chunks than a question does. Ordinary query rewriting just cleans/reformulates the *question* itself (e.g. resolving pronouns) and still embeds a question. The tempting wrong answer is "they're the same, both reword the query" — the distinction is *what gets embedded*: HyDE embeds a synthetic answer, rewriting embeds a better question.

   </details>

2. You have separate indexes for HR policies, engineering runbooks, and a SQL database of employee records. A user asks "how many days of leave does the engineering handbook say I get, and how many have I used?" Which advanced pattern(s) best fit, and why?

   <details><summary>Answer</summary>

   A **multi-selector query router** (the question spans HR policy *and* the SQL records) plus, ideally, an **agentic/multi-hop** step since the two sub-answers are independent lookups that must be combined. A single-selector router is wrong here because it would pick only one source and miss half the question; pure query transformation is wrong because the failure isn't phrasing — it's that the answer lives across two different data sources.

   </details>

3. **Which TWO** of the following are correct statements about corrective/self-RAG grading loops?
   - A. The grader adds at least one extra LLM call per retrieval, roughly increasing per-turn cost.
   - B. Grading eliminates the need to bound retry iterations.
   - C. Grading directly reduces answering-from-irrelevant-context, a common hallucination source.
   - D. Grading improves recall by generating more query variants.
   - E. Grading replaces the need for a reranker in all cases.

   <details><summary>Answer</summary>

   **A and C.** A is correct because the grader is a separate LLM invocation per retrieval, so it roughly doubles per-turn LLM cost. C is correct because refusing to generate from graded-irrelevant docs is exactly what stops confident-but-wrong answers. B is the most tempting distractor and is wrong — grading *causes* a loop that must still be bounded with `max_iterations`, or it can run forever when the corpus can't answer. D describes multi-query expansion, not grading. E is false: grading judges relevance to decide re-search, but a reranker still orders retained docs.

   </details>

4. Your agentic RAG system's cost per conversation is wildly variable and occasionally 20× the median. Traces show some turns issue many retrieval tool calls. What is the most likely root cause and the fix?

   <details><summary>Answer</summary>

   The retrieve/grade (or agent tool) loop is **unbounded**: for queries the corpus can't satisfy, the grader keeps returning "not relevant," the graph rewrites and re-retrieves indefinitely, and a single turn accumulates many LLM + retrieval calls. The fix is to thread an iteration counter through the graph state and route to a fallback ("not covered") branch once `max_iterations` (typically 2–3) is hit, making worst-case cost deterministic. Swapping models or increasing `top_k` would not address the runaway loop.

   </details>

5. A stakeholder proposes replacing your working naive top-*k* RAG (meeting recall and faithfulness targets, p95 latency 900 ms) with a full agentic CRAG pipeline "to be state of the art." Evaluate the trade-off and recommend.

   <details><summary>Answer</summary>

   Recommend **against** it. The advanced pipeline's benefit is corrective grading and adaptive retrieval, but those pay off only when there's a *measured* failure — and the current system already meets recall and faithfulness targets. Adding grading + agent loops means 2–3× the LLM calls, higher and more variable latency (blowing the 900 ms p95), higher cost, and more failure surface, all for no accuracy gain on the current workload. The correct move is to keep naive RAG and revisit only if evaluation surfaces a specific failure (e.g. multi-hop questions or answering-from-irrelevant); "state of the art" is not a requirement, meeting the targets is.

   </details>

---

## Further Reading

- [Build a custom RAG agent with LangGraph (agentic RAG + document grading loop)](https://docs.langchain.com/oss/python/langgraph/agentic-rag) — *verified 2026-07-29* — Official tutorial for retrieval-as-a-tool, `grade_documents` conditional edge, and rewrite/re-retrieve loop.
- [LangGraph Graph API (state, nodes, edges, conditional edges, ToolNode)](https://docs.langchain.com/oss/python/langgraph/graph-api) — *verified 2026-07-29* — Reference for `StateGraph`, `MessagesState`, `add_conditional_edges`, and `compile()` used to build bounded retrieval loops.
- [LlamaIndex Routers (RouterQueryEngine / selectors)](https://developers.llamaindex.ai/python/framework/module_guides/querying/router/) — *verified 2026-07-29* — Official guide to single/multi and LLM/Pydantic selectors for routing across data sources.
- [LlamaIndex Retriever module guide](https://developers.llamaindex.ai/python/framework/module_guides/querying/retriever/) — *verified 2026-07-29* — Retriever abstractions and modes underpinning advanced retrieval composition.
- [LlamaIndex Property Graph Index (GraphRAG construction & querying)](https://developers.llamaindex.ai/python/framework/module_guides/indexing/lpg_index_guide/) — *verified 2026-07-29* — Official guide to building/querying knowledge graphs with `kg_extractors` for GraphRAG.
