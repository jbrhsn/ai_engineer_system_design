# Context Engineering vs Prompt Engineering

**Section:** 03 Agentic & Multi-Agent Systems → Context Engineering & Agent Memory | **Est. time:** 3 hrs | **Interview relevance:** High — this is the hot senior-level framing; if you can only "write a better prompt" you can't explain why long-running agents degrade, and every memory/compaction question downstream assumes you already think in *context*, not prompts.

---

## TL;DR

**Prompt engineering** is crafting the wording of a single instruction (mostly the system prompt) for a one-shot task. **Context engineering** is the broader discipline of curating and maintaining the *entire* set of tokens the model sees at inference time — system prompt, tool definitions, retrieved documents, message history, memory, and tool results — across many turns of an agent loop. It matters for agents specifically because a loop *accumulates* tokens every turn, and context is a **finite, degrading resource**: as the window fills, models suffer "context rot" (recall drops) because attention is a limited budget spread over n² token relationships. The job is to find the *smallest set of high-signal tokens* that produce the desired behaviour, using four levers — **write, select, compress, isolate** — plus a system prompt at the right "altitude" and a curated, just-in-time tool/context set. **The one thing to remember: prompt engineering optimizes one message; context engineering manages the whole finite, ever-growing token state of a multi-turn agent so it doesn't drown in its own history.**

---

## ELI5 — Explain It Like I'm 5

Prompt engineering is like writing one really clear question on a sticky note before you hand it to a helper. Context engineering is like being the person who sets up and *maintains the whole desk* the helper works at during a long job: which reference books are open, which notes are pinned up, which drawers of tools are within reach, and — crucially — clearing away the finished paperwork so the desk doesn't get so cluttered that the helper can't find the one page that matters. The helper has only so much attention; a buried desk makes them miss things even though everything is technically "there." The common misconception is that a struggling long-running assistant just needs a *better-worded question* — but usually the real fix is tidying the desk: removing stale clutter, only pulling out the reference the helper needs right now, and keeping the toolbox small enough that they never grab the wrong tool.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Distinguish prompt engineering from context engineering and explain why the shift is driven by multi-turn agents rather than one-shot prompts.
- [ ] Explain "context is a finite, degrading resource" — context rot, lost-in-the-middle, and the attention-budget mechanism behind it.
- [ ] Compare the four core context-management strategies (write, select, compress, isolate) and pick the right one for a given failure mode.
- [ ] Calibrate a system prompt to the right "altitude" and curate a minimal tool set + just-in-time context instead of front-loading everything.
- [ ] Diagnose an agent whose accuracy degrades over many turns as a *context* problem and choose a remediation.

---

## Visual Overview

### Single-Shot Prompt vs Accumulating Multi-Turn Context

```
PROMPT ENGINEERING (one-shot)          CONTEXT ENGINEERING (agent loop)
──────────────────────────             ────────────────────────────────
                                        turn 1  [sys][tools][user]
[ system prompt ]                            │
[ user message  ] ──► model ──► answer  turn 2  [sys][tools][user][AI][tool result]
      (done)                                 │
                                        turn 3  [sys][tools][user][AI][tool result]
one message, tuned once                        [AI][tool result][retrieved docs]...
                                             │        ▲ context keeps GROWING
                                        turn N  [ ...everything above + more... ]
                                                 curated EACH turn, not once
```

### The Attention Budget Fills Up Over Turns

```
context window (finite token budget)
turn 1  ▓░░░░░░░░░  high-signal, model sharp
turn 5  ▓▓▓▓▓░░░░░  useful + some stale tool results
turn 12 ▓▓▓▓▓▓▓▓▓░  bloated: old outputs, dead ends, dup docs
turn 20 ▓▓▓▓▓▓▓▓▓▓  FULL ──► "context rot": recall + reasoning degrade
                     └── the fix is curation, not a bigger window
```

### The Four Context-Management Strategies

```
              CONTEXT MANAGEMENT
        ┌───────────┬───────────┬───────────┬───────────┐
     WRITE        SELECT     COMPRESS     ISOLATE
   (save it     (retrieve   (shrink it   (split it
    outside)     what's      in place)    across
       │         needed)        │         agents)
       ▼            ▼            ▼            ▼
   scratchpad   just-in-     summarize /  sub-agents
   / NOTES.md   time RAG,    compaction,  with clean
   / memory     tool result  tool-result  context
   tool         top-k        clearing     windows
```

### Diagnosing a Degrading Long-Running Agent

```
Agent accuracy drops after many turns?
├── Is the window near/over its limit? ──► COMPRESS (compaction / summarize)
├── Is stale tool output crowding it?  ──► COMPRESS (clear old tool results)
├── Does it re-derive facts it "knew"? ──► WRITE (persist notes to memory)
├── Is it pulling in too many docs?    ──► SELECT (tighter top-k, JIT retrieval)
└── Are unrelated subtasks colliding?  ──► ISOLATE (sub-agents, clean contexts)
```

---

## Key Concepts

### Prompt Engineering vs Context Engineering

**What it is.** Prompt engineering is the practice of writing and organizing an LLM's *instructions* — chiefly the system prompt and few-shot examples — for optimal one-shot outcomes. Context engineering is the superset discipline: curating and maintaining the *optimal set of tokens (information)* present during inference, including everything that lands in the window outside the prompt — tools, external/retrieved data, message history, and memory.

**How it works mechanistically.** In the one-shot era, most use cases (classification, generation) were dominated by a single prompt, so "find the right words" was the whole game. Agents changed the shape of the problem: an agent runs in a loop, and every turn *generates more tokens that could be relevant next turn* (tool calls, observations, reasoning). That evolving universe of possible information must be cyclically refined — decided anew each turn — rather than authored once. So context engineering is *iterative curation*: at each step you choose what to pass to the model, whereas prompt engineering is the discrete, up-front task of wording a message. Anthropic frames context engineering as "the natural progression of prompt engineering."

**Where it appears in real systems.** Prompt engineering lives in a static `system_prompt=` string. Context engineering lives in the *control flow around the model*: what you append to the message list each turn, a LangChain `SummarizationMiddleware` that rewrites history, a `@before_model` hook that trims messages, a retrieval step that injects top-k chunks, or a memory tool that reads/writes outside the window. The system prompt is one component of context — an important one, but no longer the whole job.

### Context Is a Finite, Degrading Resource (Context Rot)

**What it is.** The principle that an LLM's usable context is not "free capacity up to the max token count" but a scarce **attention budget** with diminishing marginal returns — every token added depletes it, and recall degrades as the window grows. This degradation is often called **context rot**; the related "lost-in-the-middle" effect is the tendency to recall information at the start/end of a long context better than the middle.

**How it works mechanistically.** Transformers let every token attend to every other token, producing n² pairwise relationships for n tokens; as context length grows, the model's ability to capture all those relationships is stretched thin, creating tension between context *size* and attention *focus*. Models are also trained on distributions where short sequences dominate, so they have fewer specialized parameters for very-long-range dependencies. The result is a *performance gradient, not a hard cliff*: a model stays capable at long context but shows reduced precision on retrieval and long-range reasoning. Needle-in-a-haystack benchmarks (e.g. Chroma's context-rot study) demonstrate recall dropping as token count rises. The practical upshot: *good context engineering finds the smallest possible set of high-signal tokens that maximize the likelihood of the desired outcome.*

**Where it appears in real systems.** You see it as an agent that answered crisply at turn 2 but hallucinates or "forgets" an earlier instruction at turn 25, or a RAG pipeline whose answer quality falls when top-k is raised from 5 to 50. LangChain's short-term-memory docs state it directly: even when a model *supports* the full context length, most LLMs "still perform poorly over long contexts… get 'distracted' by stale or off-topic content," which is why trimming/summarizing exists.

### The Four Strategies: Write, Select, Compress, Isolate

**What they are.** The four high-level levers for keeping context tight. **Write** = save information *outside* the window (a scratchpad, notes file, or memory store) so it persists without occupying tokens. **Select** = pull in *only* the tokens needed right now (retrieval, just-in-time loading of a doc by reference). **Compress** = shrink what's already in the window in place (summarize old turns, clear stale tool results). **Isolate** = split work across separate context windows (sub-agents each with a clean window).

**How they work mechanistically.** *Write*: an agent persists a plan/notes to memory (e.g. `NOTES.md` or a memory tool) and re-reads them later, so multi-hour tasks survive context resets without carrying every token forward. *Select*: instead of pre-loading all data, the agent keeps lightweight identifiers (file paths, queries, links) and loads data on demand — progressive disclosure. *Compress*: compaction summarizes a near-full window and reinitializes a new one with the summary, preserving decisions/open bugs while discarding redundant tool output; clearing old tool results is the "lightest-touch" form. *Isolate*: sub-agents each explore with their own window (tens of thousands of tokens) but return only a ~1–2k-token distilled summary, keeping the lead agent's context clean. (Compaction and memory get dedicated later chapters; here they're the map.)

**Where they appear in real systems.** *Write/Select* → a memory tool + retrieval node; Claude Code's `CLAUDE.md` (written up front) plus `glob`/`grep` (just-in-time selection). *Compress* → LangChain `SummarizationMiddleware(trigger=("tokens", 4000), keep=("messages", 20))`, or tool-result clearing on the Claude Developer Platform. *Isolate* → the orchestrator-worker multi-agent research pattern where subagents return condensed findings.

### System-Prompt Altitude

**What it is.** "Altitude" is the right level of specificity for a system prompt — the Goldilocks zone between hardcoded brittle logic and vague hand-waving.

**How it works mechanistically.** Too *low* (hardcoded if/else rules for every edge case) makes the prompt fragile and high-maintenance and fights the model's own reasoning; too *high* (vague, "be helpful") gives no concrete signal and falsely assumes shared context. The optimal altitude is specific enough to guide behaviour yet flexible enough to leave the model strong heuristics — you strive for the *minimal set of information that fully outlines expected behaviour* (minimal ≠ short; you still supply enough to pin down the behaviour). Practically: start with a minimal prompt on the best model, then add clear instructions/examples targeted at observed failure modes rather than pre-stuffing every rule.

**Where it appears in real systems.** Organize the prompt into delineated sections (`<background_information>`, `<instructions>`, `## Tool guidance`, `## Output description`) with Markdown headers or XML tags. In multi-agent research, Anthropic found prompts should instill *heuristics* ("start broad then narrow," "scale effort to query complexity") rather than rigid rules — a direct application of correct altitude.

### Tool Curation

**What it is.** Deliberately keeping the tool set minimal and unambiguous, because tool definitions are part of context and every tool competes for the model's decision at each turn.

**How it works mechanistically.** Tools define the contract between the agent and its action/information space; each definition consumes input tokens and adds a choice. The dominant failure mode is *bloated tool sets* with overlapping functionality that create ambiguous "which tool?" decisions — "if a human engineer can't definitively say which tool to use, an AI agent can't either." Curating a minimal viable set both saves tokens and makes long-interaction pruning more reliable. This is why tools must be self-contained, robust to error, and token-efficient in what they *return* (a verbose tool result is itself context bloat).

**Where it appears in real systems.** At the API level this is the `tools=[...]` array; the rule of thumb is to keep the set small and non-overlapping (this chapter's sibling on tool-calling notes the ~<20 tools heuristic). In multi-agent research, giving agents explicit tool-selection heuristics and rewriting bad tool descriptions cut task-completion time ~40%.

### Just-in-Time Context (Progressive Disclosure)

**What it is.** Loading context *at runtime by reference* when it's needed, instead of pre-processing and front-loading everything into the window before inference.

**How it works mechanistically.** The agent maintains lightweight identifiers — file paths, stored queries, web links — and uses tools to dynamically pull the underlying data only when required. This mirrors human cognition (we don't memorize a corpus; we use file systems and bookmarks to fetch on demand), and it enables *progressive disclosure*: each interaction yields metadata (file names, sizes, timestamps) that guides the next retrieval, so the agent assembles understanding layer by layer while keeping only what's necessary in working memory. The trade-off: runtime exploration is slower than reading pre-computed data, and it demands good tools/heuristics or the agent wastes context chasing dead ends. A **hybrid** strategy (drop some data up front for speed, explore the rest just-in-time) suits less-dynamic domains.

**Where it appears in real systems.** Claude Code is the canonical example: `CLAUDE.md` files are loaded up front, while `glob`/`grep`/Bash `head`/`tail` let it query large data and files on demand without ever loading full objects — bypassing stale indexes. In LangGraph you implement JIT selection as a retrieval tool the model calls, or a `@before_model` step that injects only the currently relevant slice of state.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Context budget target (tokens) | The working ceiling you curate *toward* (well below the model's hard max) | Target the *smallest* high-signal set; trigger compression well before the hard limit (e.g. LangChain `SummarizationMiddleware(trigger=("tokens", 4000))`) — never plan to fill the window, since rot sets in before the max. |
| System-prompt altitude | Specificity of the system prompt | Start minimal on the strongest model; add instructions/examples *only* for observed failure modes. If it reads like brittle if/else, raise altitude; if the model is guessing intent, lower it. |
| Number of tools exposed | How many tools the model chooses among per turn | Keep the set small and non-overlapping (~<20); if two tools have overlapping purpose or you can't say which to use when, merge/prune or split into isolated agents. |
| Retrieval top-k into context | How many retrieved chunks are injected per turn | Use the smallest k that covers the answer; raising k trades recall for rot and cost. Prefer just-in-time retrieval by reference over dumping a large k every turn. |
| Compaction trigger + keep window | When to summarize and how much recent detail to retain | Trigger near (not at) the budget; tune the summary prompt for *recall first* (capture all relevant trace info), then improve *precision*; keep the most-recent N messages/files verbatim. |
| Tool-result retention | Whether raw tool outputs stay in history | Clear or summarize old tool results once acted on — the "lightest-touch" compaction; a raw result deep in history rarely needs to be re-seen. |

### Worked Example: Requirement → Decision

**Given:** You run a coding-assistant agent that helps engineers migrate a large legacy codebase. It works well for the first ~15 minutes, but on long sessions users report it "loses the plot": it forgets an architectural decision it made earlier, re-reads files it already summarized, and its edit quality drops noticeably. Traces show the message history has ballooned to ~150k tokens, dominated by full raw outputs of `read_file` and `grep` calls from an hour ago, and the model is now missing details that are technically still in the window.

- **Step 1 — Identify the goal:** Restore reliable, coherent behaviour across a long-horizon session without shrinking the task — i.e. keep the agent sharp as the token count exceeds what fits comfortably in the attention budget.
- **Step 2 — Define inputs:** The growing message list (system prompt + tool defs + dozens of tool results + AI turns), the current file/edit state, and the architectural decisions made earlier in the session.
- **Step 3 — Define outputs:** A curated context each turn that preserves the high-signal state (decisions made, open bugs, the migration plan, the few files in active use) while shedding low-signal bulk (stale raw tool dumps), so the model keeps performing.
- **Step 4 — Apply constraints:** Session length exceeds a single comfortable window (long-horizon), latency budget forbids re-reading everything each turn, and losing a subtle-but-critical decision is worse than losing a stale file dump (recall-first compaction). This is diagnosed as a *context* problem, not a prompt-wording problem.
- **Step 5 — Select the approach:** Combine **compress + write**: enable **compaction** (summarize the near-full history into a high-fidelity summary — preserving decisions/open bugs, discarding redundant tool output — and continue with the summary plus the few most-recently-accessed files) and add **structured note-taking** (persist the migration plan/decisions to a `NOTES.md`/memory the agent re-reads). *Rationale vs alternatives:* just enlarging the context window doesn't help — rot and distraction persist regardless of max size; better system-prompt wording (prompt engineering) can't fix a window drowning in stale tokens; isolate/sub-agents would help if independent subtasks collided, but here the problem is one long coherent thread whose *history* is the bloat, so compress+write is the direct fix.

---

## Implementation

```python
# Scenario: A long-running support/research agent's history grows every turn and it
# starts to "forget" and degrade. We manage context as a finite resource by (a) keeping
# only the currently relevant slice, (b) summarizing when we approach a budget, and (c)
# persisting durable facts OUTSIDE the window (write) rather than carrying every token.
# APIs verified against LangChain short-term-memory docs (docs.langchain.com).
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware
from langgraph.checkpoint.memory import InMemorySaver

# COMPRESS: summarize earlier turns before the window fills; keep recent detail verbatim.
# Trigger well BELOW the model's hard max so rot never sets in.
agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[search_docs, fetch_ticket],          # SELECT: a small, non-overlapping tool set
    middleware=[
        SummarizationMiddleware(
            model="anthropic:claude-haiku-4-5",  # cheap model does the summarizing
            trigger=("tokens", 8000),            # compact as we approach the budget target
            keep=("messages", 20),               # retain the most recent high-signal turns
        )
    ],
    checkpointer=InMemorySaver(),                # WRITE: state persists outside the prompt
)
# Result: worst-case context stays near the budget target instead of growing unbounded,
# so recall/reasoning stay stable across a long multi-turn session.
```

```python
# Anti-pattern: "context maximalism" — dump the ENTIRE tool catalog, the FULL raw history,
# and EVERY retrieved doc into the window every single turn, assuming more context = smarter.
# This is the #1 context-engineering mistake: it burns the attention budget, triggers context
# rot (recall drops, model gets distracted by stale tokens), and inflates cost/latency.
def build_context_WRONG(user_msg, all_tools, full_history, vector_store):
    tools = all_tools                                  # e.g. 60 overlapping tools
    docs = vector_store.similarity_search(user_msg, k=50)   # 50 chunks, mostly noise
    # every prior raw tool result kept verbatim forever:
    messages = full_history + [{"role": "user", "content": user_msg}]
    return messages, tools, docs   # window balloons; model "loses the plot" by turn ~20

# Correct approach: curate the SMALLEST set of high-signal tokens each turn.
def build_context_RIGHT(user_msg, tool_registry, history, vector_store, notes_store):
    tools = tool_registry.select_relevant(user_msg, max_tools=8)  # SELECT: minimal, non-overlapping
    docs = vector_store.similarity_search(user_msg, k=5)          # SELECT: smallest k that answers
    history = compact_if_over_budget(history, budget_tokens=8000) # COMPRESS: summarize old turns
    history = clear_stale_tool_results(history)                  # COMPRESS: drop acted-on raw dumps
    plan = notes_store.read("plan.md")                           # WRITE: durable facts live outside
    messages = [{"role": "system", "content": f"Plan so far:\n{plan}"}] + history \
             + [{"role": "user", "content": user_msg}]
    return messages, tools, docs
# What breaks without this: the maximalist version degrades precisely BECAUSE it added tokens —
# attention is finite (n^2 relationships), so signal gets diluted by stale/irrelevant context,
# and a bigger context window does NOT fix it. Curation, not capacity, is the lever.
```

---

## Common Pitfalls & Misconceptions

- **"Just write a better prompt"** — beginners treat every agent failure as a wording problem because prompt engineering is what they learned first. For multi-turn agents the usual culprit is the *accumulated context state* (bloat, stale tool results, missing memory), so the fix is curating tokens (compress/write/select), not rephrasing the system prompt.
- **"Bigger context window solves it"** — the max token count is advertised like usable capacity, so more looks strictly better. Recall degrades (context rot) well before the hard limit because attention is a finite budget over n² relationships; you must curate *toward the smallest high-signal set*, not fill the window.
- **"More tools / more retrieved docs = smarter agent"** — it feels safer to expose everything and inject a large top-k. Every tool definition and every chunk is input tokens that dilute attention and create ambiguous decisions; a small, non-overlapping tool set and the smallest sufficient k outperform maximalism.
- **"Context is authored once, up front"** — coming from one-shot prompting, people configure the context at start and never touch it. Context engineering is *iterative* — the curation decision recurs every turn as the loop generates new tokens, so you need runtime machinery (middleware, retrieval, memory), not a static string.
- **"Altitude means shorter is always better"** — after hearing "minimal context," teams strip prompts to vague one-liners and the agent guesses intent. Minimal means the *smallest set that fully specifies the behaviour* — sometimes long; the failure at the other extreme is brittle hardcoded if/else logic, so you calibrate, not just shrink.

---

## Key Definitions

| Term | Definition |
|---|---|
| Prompt engineering | Writing and organizing an LLM's instructions (chiefly the system prompt) for optimal one-shot outcomes. |
| Context engineering | Curating and maintaining the optimal set of tokens (system prompt, tools, retrieved data, history, memory, tool results) present during inference, iteratively across an agent loop. |
| Context (the token set) | The full set of tokens included when sampling from the model at a given step — everything the model can "see," not just the prompt. |
| Attention budget | The finite capacity a model has to attend across its context; every added token depletes it, giving diminishing returns. |
| Context rot | The observed decline in a model's recall/precision as the number of tokens in the context window grows. |
| Lost-in-the-middle | The tendency to recall information at the start/end of a long context more reliably than information in the middle. |
| Write / Select / Compress / Isolate | The four context-management strategies: save outside the window / retrieve only what's needed / shrink in place / split across separate windows. |
| System-prompt altitude | The right specificity level for a system prompt — between brittle hardcoded logic and vague guidance. |
| Just-in-time context | Loading data at runtime by reference (paths, queries, links) when needed, rather than front-loading everything (progressive disclosure). |
| Compaction | Summarizing a near-full context window and reinitializing with the summary, preserving key decisions while discarding redundant tokens. |

---

## Summary / Quick Recall

- **Prompt engineering = one message; context engineering = the whole token state** of a multi-turn agent, curated iteratively each turn.
- Agents force the shift because the **loop accumulates tokens** — every tool call/observation is new context to refine, not a fixed prompt.
- **Context is finite and degrading:** attention is a budget (n² relationships), so recall drops as the window fills (**context rot**, lost-in-the-middle) — a gradient, not a cliff.
- The goal is the **smallest set of high-signal tokens** that yields the desired behaviour — a bigger window is not the fix.
- Four levers: **Write** (save outside), **Select** (retrieve just-in-time), **Compress** (summarize/clear), **Isolate** (sub-agents with clean windows).
- Tune the **system prompt to the right altitude** (not brittle, not vague), keep a **minimal non-overlapping tool set**, and prefer **just-in-time context** over front-loading.
- Diagnose a degrading long-running agent as a **context problem** and map the symptom to a strategy (full window → compress; re-derives facts → write; too many docs → select; colliding subtasks → isolate).

---

## Self-Check Questions

1. In one sentence each, define prompt engineering and context engineering, and state the single factor that makes context engineering necessary for agents but not for one-shot classification.

   <details><summary>Answer</summary>

   Prompt engineering is crafting the wording of a single instruction (chiefly the system prompt) for a one-shot task; context engineering is curating and maintaining the *entire* token state (system prompt, tools, retrieved data, history, memory, tool results) at inference, iteratively across turns. The distinguishing factor is the **agent loop accumulating tokens over many turns** — a one-shot classification has a fixed input authored once, whereas an agent generates new context (tool calls, observations) every turn that must be re-curated. The tempting wrong answer is "agents just need longer prompts" — the issue isn't prompt length, it's that context is *dynamic and growing*, which wording alone can't manage.

   </details>

2. An engineer raises their RAG agent's retrieval `top-k` from 5 to 50 to "give the model more to work with," and answer quality *drops*. Explain what happened in terms of this chapter's core principle.

   <details><summary>Answer</summary>

   Adding 45 more chunks spent the model's finite **attention budget** on mostly low-signal tokens, triggering **context rot** — recall and reasoning precision degrade as the window fills, and the answer-bearing chunks get diluted/lost among noise (including lost-in-the-middle effects). The principle is that good context engineering seeks the *smallest set of high-signal tokens*, so the fix is a smaller sufficient k (and/or just-in-time retrieval), not more. The tempting wrong answer is "the embeddings must be bad" — retrieval quality may be fine; the regression is caused by *quantity* overwhelming attention, which is a context-management failure, not an embedding failure.

   </details>

3. **Which TWO** of the following are correct applications of the write/select/compress/isolate strategies?
   - A. A coding agent persists its migration plan to a `NOTES.md` file it re-reads later, instead of keeping every planning token in the window (Write).
   - B. Increasing the model's maximum context window from 200k to 1M tokens counts as Compress because more fits.
   - C. Summarizing a near-full conversation into a high-fidelity summary and continuing from it, keeping the few most-recent files verbatim (Compress).
   - D. Exposing all 60 available tools every turn so the model always has options (Select).
   - E. Deleting the system prompt to save tokens once the agent "knows" the task (Isolate).

   <details><summary>Answer</summary>

   **A and C.** A is Write — persisting durable facts *outside* the window so they survive without occupying tokens. C is Compress (compaction) — shrinking the in-window history via a summary while retaining recent high-signal detail. B is wrong: enlarging the max window is *capacity*, not a management strategy, and rot still occurs before the limit — it is not "Compress." D is the opposite of Select (it's context maximalism / poor tool curation, which dilutes attention and creates ambiguous tool choices). E is not Isolate at all and is harmful — the system prompt is high-signal context defining behaviour, and Isolate specifically means splitting work across *separate* context windows (sub-agents), not deleting instructions.

   </details>

4. Your agent works well early in a session but after ~25 turns it forgets an instruction it followed earlier, even though that instruction is technically still in the message history. A teammate proposes swapping to a model with a larger context window. Evaluate this fix and propose a better one.

   <details><summary>Answer</summary>

   A larger window is unlikely to fix it: the failure is **context rot / lost-in-the-middle** — as the window grows, attention (finite, n² relationships) is stretched thin and mid-context instructions lose salience; a bigger max just delays, not removes, the degradation, and models are trained more on shorter sequences. Better fixes manage the *content*: **Compress** (compaction/summarize old turns, clear stale tool results) to reduce dilution, and/or **Write** (persist the key instruction to memory and re-inject it) so it stays high-salience. The larger-window proposal is the tempting distractor because "more capacity" sounds like more memory — but the problem is attention allocation over the tokens already present, which capacity alone doesn't solve.

   </details>

5. You must design context handling for two agents: (a) a customer-support bot with long back-and-forth conversations, and (b) a research agent that must explore many independent sub-questions in parallel and exceeds a single window. Which strategies fit each, and why not just use the same approach for both?

   <details><summary>Answer</summary>

   (a) The support bot has one long *coherent* conversational thread, so **Compress (compaction/summarization)** fits — it maintains conversational flow while keeping the window bounded; note-taking helps for milestones. (b) The research agent has *independent parallel* subtasks whose combined context exceeds one window, so **Isolate (sub-agent architecture)** fits — each subagent explores in its own clean window and returns a distilled ~1–2k-token summary, keeping the lead agent's context clean and enabling parallel token budgets. Using one approach for both is wrong because the strategies target different failure shapes: compaction on the research agent would serialize independent work and bottleneck it, while sub-agents for a single chatty thread add coordination overhead and latency with no parallelism to exploit — you match the strategy to whether the bloat is *one growing thread* (compress) or *colliding independent workloads* (isolate).

   </details>

---

## Further Reading

- [Effective context engineering for AI agents (Anthropic)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — *verified 2026-07-29* — Primary source for the prompt→context shift, context-as-finite-resource / attention budget, system-prompt altitude, tool curation, just-in-time context, and compaction/note-taking/sub-agents.
- [Building effective agents (Anthropic)](https://www.anthropic.com/engineering/building-effective-agents) — *verified 2026-07-29* — The "LLMs using tools in a loop" definition of agents and the augmented-LLM (retrieval + tools + memory) building block that context engineering curates.
- [How we built our multi-agent research system (Anthropic)](https://www.anthropic.com/engineering/multi-agent-research-system) — *verified 2026-07-29* — Real-world isolation via sub-agents with separate context windows, heuristic-based prompting (altitude), and long-horizon context/memory management.
- [Short-term memory (LangChain)](https://docs.langchain.com/oss/python/langchain/short-term-memory) — *verified 2026-07-29* — Practical context management: trimming, deleting, and `SummarizationMiddleware` for keeping a conversation within the model's usable context.
- [Prompt engineering overview (Anthropic docs)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) — *verified 2026-07-29* — The prompt-engineering baseline that context engineering builds on, for the contrast this chapter draws.
