# Mock Question Bank — Multi-Agent Systems & Orchestration

**Section:** 05 · Interview Practice → 03 · Rapid-Fire Mock Question Bank & Self-Assessment | **Use:** rapid-fire self-drill

---

## How to Use This Bank

Cover each answer, say your response out loud in under 60 seconds, then reveal and score yourself. Rounds escalate from recall → applied scenario → analysis/trade-off → multiple-choice, mirroring how an interviewer probes depth after a crisp opening answer. Grounding is current official documentation (LangGraph/LangChain OSS docs and Anthropic engineering posts) because the section-03 chapters that will eventually cover this material are still stubs — verify framework specifics against the linked docs before relying on any API detail.

---

## Rapid-Fire Round (Recall)

**1. What is the ReAct pattern?**

<details><summary>Answer</summary>

**Reason + Act**: the LLM interleaves reasoning traces with tool calls in a loop — it thinks, calls a tool, observes the result, thinks again, and repeats until it decides to answer. In practice this is "an LLM autonomously using tools in a loop" (Anthropic's working definition of an agent). The loop needs an explicit stopping condition (task complete or max iterations) or it can run indefinitely.
</details>

**2. Tool-calling vs. function-calling — same thing or different?**

<details><summary>Answer</summary>

Effectively the same mechanism under two names. The model is given tool/function schemas (name, description, JSON-schema parameters); when it wants to act it emits a structured tool-use block naming the tool and arguments rather than free text. "Function calling" was the earlier OpenAI term; "tool calling" is the more general/current term. The runtime executes the call and feeds the result back into context.
</details>

**3. What is a supervisor (orchestrator-worker) topology?**

<details><summary>Answer</summary>

A central agent decomposes a task, delegates sub-tasks to specialized worker agents, and synthesizes their results. Anthropic calls it the **orchestrator-workers** workflow; their multi-agent Research system uses a lead agent that spawns parallel subagents with isolated context windows. Key trait: all coordination and routing flows through the supervisor.
</details>

**4. What is a checkpointer in LangGraph, and what does it give you?**

<details><summary>Answer</summary>

A checkpointer persists a thread's graph state as snapshots (checkpoints) keyed by a `thread_id`. It provides short-term, thread-scoped memory: conversation continuity, human-in-the-loop pauses/resumes, time travel, and fault tolerance. `InMemorySaver` is for dev; `PostgresSaver`/`SqliteSaver` persist across process restarts.
</details>

**5. Checkpointer vs. Store — what's the distinction?**

<details><summary>Answer</summary>

A **checkpointer** persists graph-state snapshots scoped to a single thread (short-term memory). A **Store** persists application-defined key-value data across threads (long-term, cross-thread memory — user preferences, facts, shared knowledge). Most production apps use both: checkpointer for the current conversation, Store for durable knowledge.
</details>

**6. What is "durable execution"?**

<details><summary>Answer</summary>

The ability to run a stateful, long-running agent so that if it crashes or is interrupted it resumes from the last checkpoint rather than restarting from scratch. In LangGraph this is delivered by the persistence layer — checkpoints written as the graph advances let execution recover mid-run. Anthropic frames the same idea as "agents are stateful and errors compound," so they combine checkpoints with retry logic.
</details>

**7. What is the plan-and-execute pattern, and how does it differ from ReAct?**

<details><summary>Answer</summary>

Plan-and-execute first produces an explicit multi-step plan, then executes the steps (often replanning after each). ReAct decides the next action one step at a time. Plan-and-execute front-loads reasoning (better for long-horizon tasks, fewer model calls on the planning path); ReAct is more reactive and adaptive step-to-step. Both still need a loop bound.
</details>

**8. What is the reflection (evaluator-optimizer) pattern?**

<details><summary>Answer</summary>

One LLM call generates a response; a second call critiques it against criteria; the generator revises — looping until the evaluation passes or a limit is hit. Anthropic calls this **evaluator-optimizer**. It fits tasks with clear evaluation criteria where iterative refinement measurably helps (e.g., literary translation, multi-round search).
</details>

---

## Applied Round (Scenario)

**9. A stakeholder asks for a "multi-agent system." When should you push back and use a single agent with tools instead?**

<details><summary>Answer</summary>

Default to a single agent with well-designed tools; add agents only when a concrete pressure demands it. The LangChain docs name three legitimate drivers: **context management** (one agent has too many tools / too much domain context to reason well), **distributed development** (separate teams own separate capabilities), and **parallelization** (independent subtasks run concurrently). **Trade-off named:** multi-agent buys parallelism and context isolation but Anthropic measured ~15× the token cost of a chat and heavy coordination complexity — only justified when task value is high and work is genuinely parallelizable.
</details>

**10. Your research agent keeps spawning far too many subagents for trivial queries. How do you fix it?**

<details><summary>Answer</summary>

Embed explicit effort-scaling heuristics in the orchestrator prompt (Anthropic's lesson): simple fact-finding = 1 agent, 3–10 tool calls; direct comparison = 2–4 subagents; only complex research uses 10+. Give the lead agent clear task boundaries per subagent so they don't duplicate work. **Trade-off named:** you're trading a little prompt-engineering effort and some rigidity for a large reduction in wasted tokens and latency — the alternative (letting the model self-judge effort) was their top early failure mode.
</details>

**11. Design a customer-refund agent that can issue refunds. Where do you insert human-in-the-loop?**

<details><summary>Answer</summary>

Put a human approval interrupt immediately **before** the irreversible action (the refund tool call), not at the end. In LangGraph, an `interrupt` inside the node pauses the graph and persists state via the checkpointer; a human approves/edits, then you resume from that exact checkpoint. **Trade-off named:** HITL adds latency and human cost, so gate only high-risk / irreversible / low-confidence actions — read-only lookups run autonomously.
</details>

**12. Your agent occasionally loops forever calling tools. How do you bound it?**

<details><summary>Answer</summary>

Add explicit stopping conditions: a max-iteration / recursion limit, a token or wall-clock budget, and a terminal "final answer" branch the model can take. Anthropic recommends stopping conditions (e.g., max iterations) precisely to "maintain control." **Trade-off named:** a tight bound can truncate legitimately long tasks, so pair the hard limit with graceful degradation (return best-effort answer + note) rather than a silent crash.
</details>

**13. A long-running agent conversation exceeds the model's context window mid-task. What do you do?**

<details><summary>Answer</summary>

Apply context-engineering techniques: **compaction** (summarize the conversation and reinitialize a fresh window with the summary + most-recent files), **structured note-taking** (write progress to an external NOTES/memory file re-loaded later), or **sub-agent architectures** (isolate deep work in subagents that return ~1–2k-token distilled summaries). **Trade-off named:** compaction risks dropping subtle-but-critical detail, so tune the compaction prompt for recall first, then precision.
</details>

**14. Two teams want to own a "billing" capability and a "search" capability independently inside one product. Which multi-agent pattern fits?**

<details><summary>Answer</summary>

The **subagents** pattern — a main agent coordinates specialized subagents invoked as tools, giving each team a clear boundary to develop and maintain independently while also supporting parallel and multi-hop calls. The LangChain pattern table rates subagents highest for distributed development and parallelization. **Trade-off named:** subagents are stateless per invocation (strong context isolation) but repeat the full flow each call — one extra model call vs. handoffs/router because results route back through the main agent.
</details>

---

## Analysis / Trade-off Round

**15. Supervisor (centralized) vs. handoff/decentralized topology — justify a choice.**

<details><summary>Answer</summary>

**Supervisor:** all routing passes through one orchestrator — easy to reason about, centralized control, natural for parallel fan-out and result synthesis; costs an extra hop and can bottleneck (Anthropic's lead agent runs subagents synchronously, blocking on the slowest). **Handoffs:** control transfers agent-to-agent via tool calls that mutate routing state — fewer calls on repeat/multi-hop conversational turns and agents can talk to the user directly, but it executes sequentially so it can't parallelize breadth-first work and is harder to trace. **Justify:** choose supervisor for breadth-first parallel research; choose handoffs for stateful conversational flows with sequential multi-hop routing.
</details>

**16. Single-agent-with-tools vs. multi-agent for a code-migration task — which and why?**

<details><summary>Answer</summary>

Lean single-agent (or a light orchestrator). Anthropic explicitly notes coding tasks have fewer truly parallelizable subtasks than research, agents aren't yet great at real-time delegation, and coding needs shared context — so multi-agent's context isolation hurts more than it helps. Multi-agent shines on breadth-first, parallelizable, context-exceeding work (their research eval beat single-agent by 90.2%). **Justify:** match topology to parallelizability and shared-context needs, not to perceived sophistication.
</details>

**17. Compare context-window management strategies: compaction vs. note-taking vs. sub-agents.**

<details><summary>Answer</summary>

Per Anthropic's context-engineering guidance: **compaction** maintains conversational flow for extensive back-and-forth (summarize + continue) but can lose subtle detail. **Structured note-taking** excels for iterative work with clear milestones — persistent memory at minimal overhead, but relies on the agent writing good notes. **Sub-agent architectures** handle complex parallel exploration by isolating heavy context in workers that return distilled summaries, at high token cost and coordination complexity. **Justify:** they're complementary — pick by task shape (conversational vs. milestone-driven vs. parallel-exploration), and combine them for long-horizon work.
</details>

**18. Why does "context rot" make bigger context windows an incomplete solution?**

<details><summary>Answer</summary>

Attention is a finite budget: transformers compute n² pairwise token relationships, and recall degrades as tokens grow (context rot), so more tokens mean diminishing returns and reduced precision on retrieval/long-range reasoning — a gradient, not a cliff. Simply waiting for larger windows doesn't remove pollution/relevance problems. **Justify:** the goal is the *smallest set of high-signal tokens* that maximizes the desired outcome, which is why curation (compaction, just-in-time retrieval, sub-agents) still matters regardless of window size.
</details>

**19. When is a deterministic *workflow* the right call over an autonomous *agent*?**

<details><summary>Answer</summary>

Anthropic's distinction: **workflows** orchestrate LLMs/tools through predefined code paths (prompt chaining, routing, parallelization) — predictable, consistent, cheaper, best when the task decomposes into fixed subtasks. **Agents** let the LLM dynamically direct its own process — needed for open-ended problems where you can't predict the number/order of steps. **Justify:** agents trade latency, cost, and compounding-error risk for flexibility; use the simplest thing that works and add autonomy only when a fixed path can't express the task.
</details>

---

## Multiple-Choice Rapid Check

**20. A checkpointer in LangGraph primarily provides:**
a) Cross-thread long-term memory of user facts
b) Short-term, thread-scoped state persistence enabling resume and HITL
c) A vector index for semantic retrieval
d) Automatic prompt compression

<details><summary>Answer</summary>

**b.** A checkpointer persists thread-scoped graph-state snapshots, enabling conversation continuity, HITL pauses, time travel, and fault-tolerant resume. **(a) is the tempting distractor** — but cross-thread long-term memory is the job of a **Store**, not a checkpointer; conflating the two is the classic mistake. (c) and (d) are unrelated mechanisms.
</details>

**21. Anthropic's evaluator-optimizer workflow is characterized by:**
a) A router classifying inputs to specialized downstream tasks
b) A central LLM delegating subtasks to workers
c) One LLM generating while another critiques in a refinement loop
d) Running the same prompt many times and voting

<details><summary>Answer</summary>

**c.** Generator + evaluator loop with iterative feedback until criteria are met. **(b) is the tempting distractor** — that's the *orchestrator-workers* workflow, which decomposes/delegates rather than critiques-and-refines. (a) is routing; (d) is parallelization/voting.
</details>

**22. According to Anthropic's data, a multi-agent system uses roughly how many times the tokens of a plain chat interaction?**
a) ~1.5×  b) ~4×  c) ~15×  d) ~100×

<details><summary>Answer</summary>

**c (~15×).** Anthropic reported single agents use ~4× chat tokens and multi-agent systems ~15×. **(b) ~4× is the tempting distractor** — that's the single-agent figure, not multi-agent. This is why multi-agent is reserved for high-value, parallelizable tasks.
</details>

**23. Which TWO are legitimate reasons the LangChain docs give for adopting a multi-agent architecture?**
a) It always reduces total token cost
b) Context management — avoiding overloading one model's context/tool set
c) Distributed development — independent teams owning capabilities
d) It guarantees deterministic, reproducible runs

<details><summary>Answer</summary>

**b and c** (with parallelization, these are the three stated drivers). **(a) is the tempting distractor** — multi-agent typically *increases* token cost (see Q22), it doesn't reduce it. **(d) is wrong** because multi-agent systems are non-deterministic with emergent behavior; agents can take different valid paths on identical inputs, which is exactly why end-state (not step-by-step) evaluation is recommended.
</details>

**24. The safest, lightest-touch form of context compaction Anthropic describes is:**
a) Deleting the system prompt once the agent is warmed up
b) Clearing stale tool call results deep in message history
c) Dropping the user's original request after step 1
d) Truncating the newest messages to keep the oldest

<details><summary>Answer</summary>

**b.** Once a tool result is buried deep in history, the agent rarely needs the raw output again — clearing it is low-risk, high-value (now a Claude platform feature). **(d) is the tempting distractor** — truncating the *newest* messages is backwards; recency and the active plan are exactly what you must preserve. (a) and (c) discard high-signal anchoring context.
</details>

---

## Self-Assessment Scorecard

Grade each row honestly: **✓** = explain cold in an interview · **△** = shaky · **✗** = can't yet.

| Topic area | Can I explain it cold? (✓/△/✗) | Where to review |
|---|---|---|
| ReAct & tool/function calling loop | | `03-agentic-and-multi-agent-systems/01-agent-design-patterns/01-react-and-tool-calling-agent-fundamentals.md` *(stub — not yet authored)*; Anthropic *Building effective agents* |
| Plan-and-execute & reflection patterns | | `03-agentic-and-multi-agent-systems/01-agent-design-patterns/02-plan-and-execute-and-reflection-patterns.md` *(stub — not yet authored)*; Anthropic *Building effective agents* (evaluator-optimizer) |
| Workflow vs. agent decision | | Anthropic *Building effective agents*; LangGraph *Workflows and agents* docs |
| LangGraph state graphs & orchestration primitives | | `03-agentic-and-multi-agent-systems/02-multi-agent-orchestration-with-langgraph/01-langgraph-state-graphs-and-orchestration-primitives.md` *(stub — not yet authored)*; LangGraph *Workflows and agents* docs |
| Supervisor / hierarchical topologies | | `03-agentic-and-multi-agent-systems/02-multi-agent-orchestration-with-langgraph/02-supervisor-and-hierarchical-multi-agent-topologies.md` *(stub — not yet authored)*; LangChain *Multi-agent* docs; Anthropic *Multi-agent research system* |
| Human-in-the-loop & durable execution | | `03-agentic-and-multi-agent-systems/02-multi-agent-orchestration-with-langgraph/03-human-in-the-loop-and-durable-execution.md` *(stub — not yet authored)*; LangGraph *Persistence* & *Interrupts* docs |
| Context engineering vs. prompt engineering | | `03-agentic-and-multi-agent-systems/03-context-engineering-and-agent-memory/01-context-engineering-vs-prompt-engineering.md` *(stub — not yet authored)*; Anthropic *Effective context engineering* |
| Context-window management (compaction / note-taking) | | `03-agentic-and-multi-agent-systems/03-context-engineering-and-agent-memory/02-context-window-management-compaction-and-note-taking.md` *(stub — not yet authored)*; Anthropic *Effective context engineering* |
| Agent memory: short-term vs. long-term | | `03-agentic-and-multi-agent-systems/03-context-engineering-and-agent-memory/03-agent-memory-architectures-short-term-vs-long-term.md` *(stub — not yet authored)*; LangGraph *Persistence* (checkpointer vs. store) |

> Note: the section-03 grounding chapters above are currently **empty stubs**; until they are authored, review directly against the official docs in Further Reading.

---

## Further Reading

*Official documentation only — verified live before publication.*

- [Building effective agents (Anthropic Engineering)](https://www.anthropic.com/engineering/building-effective-agents) — *verified 2026-07-29*
- [How we built our multi-agent research system (Anthropic Engineering)](https://www.anthropic.com/engineering/multi-agent-research-system) — *verified 2026-07-29*
- [Effective context engineering for AI agents (Anthropic Engineering)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — *verified 2026-07-29*
- [LangGraph — Workflows and agents](https://docs.langchain.com/oss/python/langgraph/workflows-agents) — *verified 2026-07-29*
- [LangGraph — Persistence (checkpointers & stores)](https://docs.langchain.com/oss/python/langgraph/persistence) — *verified 2026-07-29*
- [LangGraph — Interrupts (human-in-the-loop)](https://docs.langchain.com/oss/python/langgraph/interrupts) — *verified 2026-07-29*
- [LangChain — Agents (tool-calling)](https://docs.langchain.com/oss/python/langchain/agents) — *verified 2026-07-29*
- [LangChain — Multi-agent patterns](https://docs.langchain.com/oss/python/langchain/multi-agent) — *verified 2026-07-29*
