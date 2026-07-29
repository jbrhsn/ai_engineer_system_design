# Context Window Management — Compaction and Note-Taking

**Section:** 03 Agentic & Multi-Agent Systems → Context Engineering & Agent Memory | **Est. time:** 3 hrs | **Interview relevance:** High — long-running agents *will* overflow the context window, and how you keep them coherent past that point (trim vs. compact vs. note-take vs. offload) is a core design-and-trade-off question in any agentic-systems interview.

---

## TL;DR

As an agent loops — calling tools, reading observations, reasoning again — its message history grows monotonically and eventually approaches the model's context window. This chapter covers the four tactics for managing that growth: **trimming/pruning** (drop old messages by count or token budget), **compaction/summarization** (summarize older turns into a condensed form and continue), **note-taking/scratchpad** (write durable notes to external state the agent re-reads), and **tool-result management** (truncate or offload large payloads and pass references instead). They map onto the write/select/compress/isolate framing from chapter 01: compaction is *compress*, note-taking is *write + select*, offload is *isolate*. Each has a cost — compaction loses fidelity (you can summarize away the exact detail you later need), note-taking adds retrieval complexity — so the real skill is choosing the lightest tactic that preserves what the task actually needs. **The one thing to remember: a bigger context window does not solve this — models suffer "context rot" (accuracy degrades as tokens grow), so you must actively curate the window, not just enlarge it.**

---

## ELI5 — Explain It Like I'm 5

Imagine a detective working a long case with only a small whiteboard. As clues pile up, the board fills — and there's no room to write the next lead. A bad detective just erases the top of the board (the original assignment: "find the missing necklace") to make room, then forgets what they were even looking for. A good detective does two smarter things: when the board gets full, they *compress* the old clues into one tidy paragraph ("suspects A and B ruled out; focus on the gardener") and wipe the rest, keeping the summary and the case goal pinned in the corner. And for anything they might need in exact detail later — a phone number, an address — they jot it in a *notebook* they can flip back to, instead of cramming it on the board. The common wrong idea is "just get a bigger whiteboard." But even a huge board gets messy and hard to scan, and the detective starts missing clues that are right in front of them — so the trick is never a bigger board, it's *deciding what stays on the board and what gets summarized or filed away*.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain why enlarging the context window does not solve long-horizon coherence (context rot / attention budget) and why active curation is required.
- [ ] Implement message trimming by token budget and by last-N using `trim_messages` and `RemoveMessage` / `REMOVE_ALL_MESSAGES` in a LangChain/LangGraph agent.
- [ ] Design a compaction loop with `SummarizationMiddleware` (trigger + keep) and articulate the fidelity-loss trade-off it introduces.
- [ ] Implement a note-taking scratchpad in agent state and offload oversized tool results to external storage, passing back references instead of raw payloads.
- [ ] Compare trimming vs. compaction vs. note-taking vs. offload and select the right combination under a token-budget and fidelity constraint.

---

## Visual Overview

### The Compaction Loop

```
                 turn N, N+1, N+2 ...  (history grows)
                        │
                        ▼
        ┌──────────────────────────────────┐
        │ token count ≥ trigger threshold?  │
        └──────────────────────────────────┘
             │ NO                    │ YES
             ▼                       ▼
        continue as-is        summarize OLD turns ──► [SUMMARY msg]
             │                       │                     │
             │                keep last-K turns verbatim ──┤
             │                       ▼                     ▼
             └──────────────► new window = SYSTEM + SUMMARY + recent turns
                                     │
                                     └──► continue the run
```

### Trim Last-N vs. Summarize (fidelity vs. recall)

```
ORIGINAL:  [system][goal][t1][t2][t3][t4][t5][t6]

TRIM last-N (keep 3, drop rest):
           [system]............................[t4][t5][t6]
           └ fast, zero LLM cost, but t1's facts are GONE ─┘

SUMMARIZE (compact old, keep recent):
           [system][SUMMARY of t1–t3][t4][t5][t6]
           └ costs an LLM call; keeps GIST of t1–t3, loses exact detail ┘
```

### Note-Taking to External State + Re-Read

```
   agent turn ──► write_note("finding: API rate limit is 100/s")
                        │
                        ▼
              ┌───────────────────┐   survives OUTSIDE
              │ state["notes"] /   │   the context window
              │ NOTES.md / store   │
              └───────────────────┘
                        ▲
   later turn ──► read_notes() ───┘   pull back only what's needed
```

### Tool-Result Offload with Reference IDs

```
tool returns 200 KB JSON
        │
        ▼
┌─────────────────────────────┐
│ store raw payload externally │──► blob store / state / file
│ return {id, summary, schema} │
└─────────────────────────────┘
        │
        ▼
context gets  {"ref": "res_abc", "rows": 4210, "cols": [...]}   ◄── ~50 tokens
        │
        ▼
agent later ──► fetch_result("res_abc", filter=...)  # loads only the slice it needs
```

---

## Key Concepts

### Message Trimming / Pruning

**What it is.** Trimming is the mechanical removal of messages from the history before the model is called — either keeping only the last-N messages or keeping as many recent messages as fit under a token budget — so the request stays inside the context window.

**How it works mechanistically.** You count tokens (or messages) in the current history and drop from one end until you're under the limit. `trim_messages(messages, max_tokens=..., token_counter=..., strategy="last")` keeps the *most recent* tokens; `strategy="first"` keeps the oldest. Because a valid history must satisfy provider rules (start with a `HumanMessage` or a `SystemMessage`+`HumanMessage`, and every tool-call `AIMessage` must be followed by its `ToolMessage`), you set `include_system=True` and `start_on="human"` so trimming doesn't produce an invalid sequence. To make the trim *persist* in a stateful LangGraph agent (not just for one call), you emit `RemoveMessage` objects — `RemoveMessage(id=m.id)` to drop specific messages, or `RemoveMessage(id=REMOVE_ALL_MESSAGES)` followed by the messages you want to keep — which the `add_messages` reducer applies to state.

**Where it appears in real systems.** In a LangChain agent you attach a `@before_model` middleware that rewrites `state["messages"]` (returning `RemoveMessage(id=REMOVE_ALL_MESSAGES)` plus the kept messages) so the pruned history is used on every model call. `token_counter="approximate"` (backed by `count_tokens_approximately`) is recommended on the hot path where exact counting isn't worth the latency; pass the chat model itself as `token_counter` when you need provider-exact counts.

### Summarization / Compaction

**What it is.** Compaction is summarizing the older portion of a conversation into a condensed message and continuing from that summary plus the most recent turns — trading exact history for a shorter, still-coherent context. Anthropic calls it "the first lever in context engineering" for long-horizon coherence.

**How it works mechanistically.** When the history crosses a trigger (a token count or message count), a *separate* summarization model call is asked to distill the old messages — preserving decisions, unresolved problems, and key facts while discarding redundant tool chatter — into a single summary message. The old messages are then removed and replaced by `[SYSTEM][SUMMARY][recent K turns]`, and the agent loop continues. The art is *what to keep vs. discard*: Anthropic recommends tuning the compaction prompt for **recall first** (capture everything relevant) then improving **precision** (cut the superfluous), because over-aggressive compaction silently drops "subtle but critical context whose importance only becomes apparent later."

**Where it appears in real systems.** LangChain ships `SummarizationMiddleware(model=..., trigger=("tokens", 4000), keep=("messages", 20))` — `trigger` sets when compaction fires, `keep` sets how much recent history stays verbatim, and it inserts the summary automatically. This is the productionized form of the Claude Code pattern (summarize the message history, then continue with the summary plus the few most-recently-accessed files). It is *compress* in the write/select/compress/isolate framing from chapter 01.

### Note-Taking / Scratchpad (Agentic Memory)

**What it is.** Note-taking is having the agent write durable notes — a TODO list, a findings file, a running plan — to state or storage *outside* the context window, then re-read them later, so information survives compaction and context resets.

**How it works mechanistically.** You add a state field (e.g. `notes: list[str]`) or a file (`NOTES.md`) and give the agent a tool that appends to it. In LangGraph a tool writes to short-term state by returning `Command(update={"notes": [...], "messages": [ToolMessage(...)]})`; the `notes` field lives in the agent's `AgentState` and persists via the checkpointer across turns even as messages get trimmed or compacted. Later, a tool or a `@before_model` hook reads those notes back into context. Crucially, notes are *selectively re-read* — you pull back only the relevant slice, not the whole file — which is why this scales past the context window. The cost is added retrieval complexity: the agent must decide *what* to write and *when* to read, and stale or bloated notes create their own noise.

**Where it appears in real systems.** LangChain distinguishes short-term memory (thread-scoped `AgentState`, updated via `Command`/`ToolMessage`) from long-term memory (cross-thread `Store` with `store.put(namespace, key, value)` / `store.search(...)`). A findings file for a single research run is short-term state; a user-preference profile reused across sessions is a long-term `Store` entry. Anthropic's Claude "playing Pokémon" and the file-based memory tool are canonical examples of an agent maintaining a NOTES-style scratchpad across thousands of steps.

### Tool-Result Management (Truncation & Offload)

**What it is.** Tool-result management is preventing large tool outputs (a 200 KB JSON payload, a full document, a huge query result) from being dumped verbatim into context — instead truncating them, or storing them externally and passing back a compact reference (an ID + summary + schema).

**How it works mechanistically.** The tool wrapper intercepts the raw result. If it's small, it passes through; if it's large, it (a) truncates to the first/last N characters with a "[truncated]" marker, or (b) writes the full payload to state or a blob store and returns a lightweight handle — `{"ref": "res_abc", "rows": 4210, "summary": "...", "columns": [...]}` — that costs tens of tokens instead of tens of thousands. A later tool call dereferences the handle to fetch only the slice needed (a filtered query, a specific page). This is the "just-in-time" retrieval pattern: keep lightweight identifiers in context, load full data on demand. Anthropic notes that clearing old tool results is "one of the safest, lightest-touch forms of compaction" — once a tool result is deep in history, the agent rarely needs the raw bytes again.

**Where it appears in real systems.** A LangChain tool returns a `Command` that writes the raw payload to a state field and puts only the reference in the `ToolMessage` content the model sees. The reference pattern mirrors the `Store`/file-path approach in the "just-in-time" section of Anthropic's context-engineering guidance, and the Claude Developer Platform ships tool-result-clearing as a managed feature.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| `max_tokens` (`trim_messages`) | Token (or message, if `token_counter=len`) budget the trimmed history must fit under | Set to a fraction (~50–70%) of the model's window, leaving headroom for the *next* model output and tool definitions — never set it to the full window. |
| `strategy` (`first` / `last`) | Which end of the history to keep | Use `"last"` for conversational agents (recent turns matter most); use `"first"` only when the opening context is the payload you must preserve. |
| `include_system` / `start_on` | Whether the system/goal message survives and that the result is a valid sequence | Always `include_system=True` + `start_on="human"` for chat models, or trimming can drop the goal or produce a provider-invalid history. |
| `trigger` (`SummarizationMiddleware`, e.g. `("tokens", 4000)`) | When compaction fires | Trigger *before* you hit the hard window limit (leave room for the summary call + next turn); tune to your model's window, not a fixed number. |
| `keep` (`SummarizationMiddleware`, e.g. `("messages", 20)`) | How many recent turns stay verbatim after compaction | Keep enough recent turns that in-flight tool-call/result pairs stay intact; too few and the agent loses immediate working context right after a compaction. |
| Tool-output truncation length | Max chars/tokens of a tool result allowed into context before truncate/offload | Set to the largest result the model actually needs to read inline (often a few KB); above it, offload and return a reference. |
| Notes read strategy (whole-file vs. selective) | How much of the scratchpad is pulled back into context | Re-read only the relevant slice (query/section), not the whole notes file — re-injecting everything re-creates the overflow you were avoiding. |

### Worked Example: Requirement → Decision

**Given:** You are building a deep-research agent. For a single query it runs 40+ tool calls — web searches, page fetches, and a SQL analytics tool that can return 50–200 KB result sets — over ~20 minutes. Partway through a run it starts hitting the model's context limit and either errors or begins "forgetting" the original research question and earlier findings. The final report must cite specific numbers it discovered early in the run, and cost per run is already high.

- **Step 1 — Identify the goal:** Keep the agent coherent and grounded across 40+ tool calls that exceed the context window, without losing (a) the original question and (b) specific facts needed for the final report.
- **Step 2 — Define inputs:** A growing message history (reasoning + tool calls + observations), large SQL/fetch payloads, and the fixed research question in the first turns.
- **Step 3 — Define outputs:** A context window that stays under budget every turn while preserving the goal and the concrete findings needed for citations.
- **Step 4 — Apply constraints:** Context window is finite and degrades before the hard limit (context rot); some early facts are needed *exactly* (numbers to cite) so cannot be summarized to a gist; cost is high so gratuitous LLM summarization calls hurt; the run is long-horizon (compaction alone will eventually blur early detail).
- **Step 5 — Select the approach:** Combine three tactics, not one. **(1) Tool-result offload** for the SQL/fetch payloads — store raw results externally, put only a reference + summary in context (removes the biggest source of bloat cheaply). **(2) Note-taking** — the agent writes exact findings/citations to a `notes` state field as it discovers them, so specific numbers survive verbatim outside the window. **(3) Compaction** with `SummarizationMiddleware` as the backstop for the conversational reasoning once it crosses a token trigger. *Rationale vs. alternatives:* pure trimming is wrong — it would silently drop the early findings the report must cite; pure compaction is wrong — summarizing away exact numbers defeats the citation requirement and still can't hold 200 KB payloads; offload + notes preserve the must-keep specifics losslessly, and compaction handles only the compressible reasoning chatter. This mirrors Anthropic's own guidance: compaction for conversational flow, note-taking for iterative work with clear milestones.

---

## Implementation

```python
# Scenario: A long-running chat agent must stay under its context window without
# ever dropping the system/goal message. We trim by TOKEN BUDGET before each model
# call, keep the system message and the most recent turns, and PERSIST the trim into
# LangGraph state (RemoveMessage) so history doesn't just re-grow next turn.
# APIs verified against docs.langchain.com short-term-memory + trim_messages reference.
from typing import Any
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import before_model
from langchain.messages import trim_messages
from langchain_core.messages import RemoveMessage
from langgraph.graph.message import REMOVE_ALL_MESSAGES
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.runtime import Runtime


@before_model
def trim_to_budget(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    kept = trim_messages(
        state["messages"],
        max_tokens=3000,               # ~fraction of the window; leaves output headroom
        token_counter="approximate",   # fast counting on the hot path
        strategy="last",               # keep the most recent turns
        include_system=True,           # never drop the system/goal message
        start_on="human",              # keep the resulting history provider-valid
    )
    if len(kept) == len(state["messages"]):
        return None                    # nothing to trim this turn
    # Persist the trim: clear state, then re-add only the kept messages.
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES), *kept]}


agent = create_agent(
    "openai:gpt-5.5",
    tools=[...],
    middleware=[trim_to_budget],
    checkpointer=InMemorySaver(),
)
```

```python
# Scenario: A deep-research agent overflows context on long runs and must keep the
# reasoning coherent (compaction) while preserving exact findings for citations
# (note-taking) — because summarization would blur the precise numbers it must cite.
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import SummarizationMiddleware
from langchain.tools import tool, ToolRuntime
from langchain.messages import ToolMessage
from langgraph.types import Command
from langgraph.checkpoint.memory import InMemorySaver


class ResearchState(AgentState):
    notes: list[str]                   # durable scratchpad, survives compaction/trim


@tool
def record_finding(finding: str, runtime: ToolRuntime) -> Command:
    """Save an exact finding (e.g. a number to cite) to durable notes."""
    return Command(update={
        "notes": [finding],            # appended to state, OUTSIDE the message window
        "messages": [ToolMessage("saved", tool_call_id=runtime.tool_call_id)],
    })


@tool
def read_findings(runtime: ToolRuntime) -> str:
    """Re-read saved findings when composing the final report."""
    return "\n".join(runtime.state.get("notes", [])) or "(no findings yet)"


agent = create_agent(
    "anthropic:claude-sonnet-4-6",
    tools=[record_finding, read_findings, ...],
    state_schema=ResearchState,
    middleware=[SummarizationMiddleware(          # compress reasoning chatter
        model="anthropic:claude-haiku-4-5",       # cheaper model for the summary call
        trigger=("tokens", 8000),                 # fire before the hard limit
        keep=("messages", 20),                    # keep recent turns verbatim
    )],
    checkpointer=InMemorySaver(),
)
```

```python
# Anti-pattern: "just keep the last N messages" with no pinning and no offload.
# This drops the original task/goal (it's the OLDEST message) so the agent forgets
# what it was doing, AND it lets a 200 KB tool result sit in context, blowing the
# budget in a single turn.
def before_model_BROKEN(state):
    return {"messages": state["messages"][-6:]}   # goal message falls off the front
# ...and a tool that dumps everything:
@tool
def run_query_BROKEN(sql: str) -> str:
    return json.dumps(db.execute(sql))            # could be 200 KB straight into context

# Correct approach: PIN the system/goal message, summarize/keep the middle, and
# OFFLOAD large tool results to state, returning only a compact reference.
@before_model
def before_model_FIXED(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    msgs = state["messages"]
    if len(msgs) <= 7:
        return None
    kept = [msgs[0], *msgs[-6:]]                   # msgs[0] = pinned system/goal
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES), *kept]}

@tool
def run_query_FIXED(sql: str, runtime: ToolRuntime) -> Command:
    rows = db.execute(sql)
    payload = json.dumps(rows)
    if len(payload) < 4000:                        # small enough to read inline
        return Command(update={"messages": [ToolMessage(payload, tool_call_id=runtime.tool_call_id)]})
    ref = blob_store.put(payload)                  # offload the big payload
    handle = {"ref": ref, "rows": len(rows), "columns": list(rows[0].keys())}
    return Command(update={"messages": [ToolMessage(json.dumps(handle), tool_call_id=runtime.tool_call_id)]})
# What breaks without this: the naive last-N silently amnesiacs the agent (goal gone)
# and a single fat tool result overflows the window; pinning + offload keep the goal
# and cap per-turn token cost at the reference size (~tens of tokens) instead of ~50 K.
```

---

## Common Pitfalls & Misconceptions

- **"Just use a bigger context window."** — Beginners assume overflow is purely a capacity problem, so a 1M-token model must fix it. But models exhibit *context rot* — recall and long-range reasoning degrade as tokens grow, well before the hard limit — so a fuller window is a *worse-performing* window; you must curate, not just enlarge.
- **Naive last-N trimming that drops the goal** — the last-N heuristic feels safe because "recent = relevant," and the original task is the *oldest* message, so it's the first thing dropped. Pin the system/goal message (and any anchor facts) explicitly and only trim/summarize the middle — recency is not the same as importance.
- **Compacting away must-keep detail** — summarization looks lossless because the gist survives, so teams compact everything aggressively. Compaction is lossy by design; tune it for recall first, and route facts you'll need *exactly* (IDs, numbers, citations) to note-taking or offload instead of trusting a summary to preserve them.
- **Dumping raw tool results into context** — a tool "just returns its output," so a 200 KB payload lands verbatim in the window and blows the budget in one turn. Treat tool outputs as untrusted-size: truncate small overflows, offload large ones to state/blob storage, and pass a reference the agent can dereference on demand.
- **Notes that grow without bound and get fully re-read** — note-taking feels free, so agents append endlessly and re-inject the whole file, recreating the overflow they were avoiding. A scratchpad is only useful if you write selectively and re-read *slices*; unbounded notes are just a slower context leak.

---

## Key Definitions

| Term | Definition |
|---|---|
| Context window management | The active practice of keeping an agent's model input under budget as its history grows, via trimming, compaction, note-taking, and tool-result offload. |
| Context rot | The empirically observed degradation in a model's recall and long-range reasoning as the number of tokens in its context increases, even below the hard limit. |
| Trimming / pruning | Removing messages from history by count (last-N) or token budget before a model call to fit the context window. |
| `trim_messages` | LangChain utility that trims a message list to a `max_tokens` (or message) budget with a `first`/`last` strategy, preserving a valid sequence. |
| `RemoveMessage` / `REMOVE_ALL_MESSAGES` | Objects the `add_messages` reducer interprets as instructions to delete specific messages (by id) or clear the entire history in LangGraph state. |
| Compaction / summarization | Summarizing older turns into a condensed message and continuing from `[system][summary][recent]`; the primary lever for long-horizon coherence. |
| `SummarizationMiddleware` | LangChain middleware that fires on a `trigger` (tokens/messages), summarizes old history, and keeps `keep` recent turns verbatim. |
| Note-taking / scratchpad | The agent writing durable notes to external state/files (short-term `AgentState` or long-term `Store`) that survive outside the context window and are re-read selectively. |
| Tool-result offload | Storing a large tool payload externally and passing a compact reference (id + summary + schema) into context instead of the raw bytes. |
| Just-in-time retrieval | Keeping lightweight identifiers (refs, file paths, queries) in context and loading full data on demand, rather than pre-loading everything. |

---

## Summary / Quick Recall

- Agent history grows every turn and *will* overflow; a bigger window doesn't fix it because of **context rot** — you must actively curate the window.
- **Trimming** (last-N / token budget via `trim_messages` + `RemoveMessage`) is cheapest but lossy and dumb — it drops the *oldest* messages, which often includes the goal, so **pin** the system/goal message.
- **Compaction** (`SummarizationMiddleware`, `trigger` + `keep`) preserves the *gist* of old turns at the cost of an LLM call and *exact* fidelity — tune for recall first.
- **Note-taking** writes durable facts to state/`Store` that survive compaction and resets; route must-keep exact details (numbers, IDs, citations) here, and re-read *slices*, not the whole file.
- **Tool-result offload** stops giant payloads from bloating context — store externally, pass a reference; clearing old tool results is a safe, light compaction.
- Real long-horizon agents **combine** tactics (offload + notes + compaction), matching each to what the task must preserve — trade fidelity loss (compaction) against retrieval complexity (note-taking).
- Maps to chapter 01's framing: compaction = *compress*, note-taking = *write + select*, offload = *isolate*.

---

## Self-Check Questions

1. What is "context rot," and why does it mean that switching to a model with a larger context window does not, by itself, fix a long-running agent's coherence problems?

   <details><summary>Answer</summary>

   Context rot is the empirically observed degradation in a model's ability to accurately recall information and reason over long ranges *as the number of tokens in its context grows* — accuracy declines before the hard window limit is even reached (Anthropic attributes it to the finite "attention budget" of the transformer's all-pairs attention). So a bigger window doesn't fix coherence because a fuller context is a *lower-precision* context; the agent gets distracted by stale/off-topic tokens and misses information that is literally present. The tempting wrong answer — "a 1M-token window means you never overflow, so the problem is solved" — confuses *fitting* the tokens with the model *using* them well; you still must curate the window.

   </details>

2. You have a chat agent using naive last-6-message trimming. Users report that after a long conversation the agent "forgets what task it was given at the start." What is the root cause and the minimal fix?

   <details><summary>Answer</summary>

   The root cause is that the original task/goal is in the *oldest* message, and last-N trimming drops from the front — so the goal is the first thing evicted while recent small talk survives. The minimal fix is to **pin** the system/goal message and only trim the middle: keep `messages[0]` (or set `include_system=True` + `start_on="human"` in `trim_messages`) and drop from the interior, e.g. return `[RemoveMessage(id=REMOVE_ALL_MESSAGES), messages[0], *messages[-5:]]`. The tempting wrong answer — "increase N to keep more messages" — only delays the failure (the goal still falls off eventually) and wastes tokens; the issue is *which* message you keep, not how many.

   </details>

3. **Which TWO** of the following are appropriate when a research agent's SQL tool can return 200 KB result sets and it must later cite exact figures from early in the run?
   - A. Offload the raw 200 KB payload to external storage and return a reference (id + row count + schema) into context.
   - B. Rely on `SummarizationMiddleware` to compress the 200 KB result into the running summary so the numbers are preserved.
   - C. Have the agent write the specific figures it needs to cite into a durable `notes` state field via a `Command` update.
   - D. Increase `max_tokens` in `trim_messages` so the full payload always fits.
   - E. Truncate the payload to the first 500 characters and discard the rest permanently.

   <details><summary>Answer</summary>

   **A and C.** A is correct because offloading caps the per-turn token cost at the reference size (~tens of tokens) while keeping the full data retrievable on demand — the just-in-time pattern. C is correct because exact figures needed for citation must be preserved *losslessly*, and a durable notes field survives trimming/compaction outside the message window. B is the most tempting distractor and is wrong: compaction is lossy by design and will blur or drop exact numbers — never trust a summary to preserve figures you must cite verbatim. D just guarantees you overflow the window with one payload; E throws away the very numbers you need.

   </details>

4. Compare trimming and compaction for a 40-tool-call agent. Under what condition does compaction justify its extra cost over trimming, and what does it still fail to guarantee?

   <details><summary>Answer</summary>

   Trimming is free (no LLM call) but *dumb* — it deletes old turns wholesale, losing any information in them. Compaction costs an extra summarization model call but preserves the *gist* of the old turns (decisions, open problems, key facts), so it justifies its cost when the *early reasoning still matters to later steps* — i.e. long-horizon tasks where dropping old turns entirely would break coherence. What compaction still fails to guarantee is *exact* fidelity: it can summarize away a precise detail (a number, an ID) whose importance only becomes clear later. The right move is compaction for the compressible reasoning **plus** note-taking/offload for the must-keep exact facts — not compaction alone.

   </details>

5. A teammate proposes solving all long-run overflow with a single tactic: aggressive `SummarizationMiddleware` compacting everything down to a tight summary as early as possible. Evaluate this choice and its trade-off.

   <details><summary>Answer</summary>

   It's the wrong single choice. Aggressive early compaction maximizes token savings but maximizes *fidelity loss* — Anthropic warns that over-aggressive compaction drops "subtle but critical context whose importance only becomes apparent later," and it still cannot hold large raw payloads (a summary can't reconstruct a 200 KB result). The correct mental model is that no single tactic wins: tune compaction for **recall first, then precision**; route large tool outputs to **offload**; route exact must-keep facts to **note-taking**; and use compaction only for the genuinely compressible conversational reasoning. The tempting wrong answer — "one clean summary is simplest and cheapest" — optimizes token count while quietly destroying the information the task depends on.

   </details>

---

## Further Reading

- [LangChain — Short-term memory (trim / delete / summarize messages)](https://docs.langchain.com/oss/python/langchain/short-term-memory) — *verified 2026-07-29* — Canonical patterns for `trim_messages`, `RemoveMessage`/`REMOVE_ALL_MESSAGES`, `SummarizationMiddleware`, and `@before_model` history management in an agent.
- [LangChain — `trim_messages` API reference](https://reference.langchain.com/python/langchain-core/messages/trim_messages) — *verified 2026-07-29* — Full signature and semantics of `max_tokens`, `strategy` (`first`/`last`), `token_counter` (including `"approximate"`), `include_system`, and `start_on`.
- [LangChain — Middleware overview (`SummarizationMiddleware`)](https://docs.langchain.com/oss/python/langchain/middleware) — *verified 2026-07-29* — How compaction and other hooks attach to the agent loop via `create_agent(middleware=[...])`.
- [LangChain — Long-term memory (`Store`, namespaces, `put`/`search`)](https://docs.langchain.com/oss/python/langchain/long-term-memory) — *verified 2026-07-29* — Persisting notes/facts across threads with a store, the durable backing for cross-session note-taking.
- [Anthropic — Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — *verified 2026-07-29* — Authoritative treatment of context rot, compaction, structured note-taking, tool-result clearing, and just-in-time retrieval.
