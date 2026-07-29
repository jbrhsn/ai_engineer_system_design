# 03 — Agentic and Multi-Agent Systems

**Estimated time:** 27 hrs | **Prerequisites:** Section 02 (LLM Serving and RAG Architecture) — this section builds directly on retrieval and the agentic-retrieval bridge from that section's final chapter

## Overview

This section is where single-shot LLM serving turns into *systems that act*: it takes the retrieval substrate from section 02 and wraps it in loops, tools, state, and coordination. You move through the full arc — single-agent design patterns (ReAct, plan-and-execute, reflection, framework selection), then multi-agent orchestration in LangGraph (state graphs, supervisor and hierarchical topologies, human-in-the-loop and durable execution), and finally context engineering and agent memory (the finite, degrading token budget every long-running agent must manage). Throughout, the lens is security-first and evaluation-first: bounding loops so cost and termination are guaranteed, justifying the token-cost multiplier before adding an agent, and treating context as a resource to curate rather than a window to enlarge — the exact trade-offs interviewers use to tell shippers from readers.

## Learning Outcomes

By completing this section you will be able to:
- Explain the core agent loop — a model calling tools until it decides to stop — and design ReAct, plan-and-execute, and reflection topologies while bounding every loop for guaranteed termination and cost.
- Select the least machinery that meets a requirement across the control-vs-abstraction spectrum (raw tool-calling loop → `create_agent` → LangGraph), driven by concrete needs for persistence, HITL, and multi-agent coordination rather than framework hype.
- Model a multi-agent system as a LangGraph `StateGraph` — nodes, edges, and reducer-typed shared state — and choose between single-agent, supervisor, network/swarm, and hierarchical topologies while quantifying the token-cost multiplier a split incurs.
- Make an agent durable and pausable with a checkpointer and `interrupt()`, choosing the right backend and implementing human approval gates, crash recovery, and time travel.
- Manage the finite, degrading context window through write/select/compress/isolate strategies — trimming, compaction, note-taking, tool-result offload — and design short-term (thread-scoped, checkpointer-backed) versus long-term (cross-thread, store-backed) memory.

## Chapters

| # | Chapter | Est. time | Files |
|---|---|---|---|
| 1 | Agent Design Patterns — the ReAct/tool-calling loop and its bounding, plan-and-execute and reflection/self-critique patterns, and criteria-based framework selection across the control-vs-abstraction spectrum. | 8.5 hrs | notes + interview-prep |
| 2 | Multi-Agent Orchestration with LangGraph — StateGraph primitives (nodes, edges, reducer-typed state, super-steps), supervisor/network/hierarchical topologies and the token-cost multiplier, and durable execution with checkpointers, `interrupt()`, and time travel. | 9.5 hrs | notes + interview-prep |
| 3 | Context Engineering & Agent Memory — context engineering vs prompt engineering and context rot, window management (trim/compact/note-take/offload), and short-term vs long-term memory architectures with deliberate write and selective retrieval. | 9 hrs | notes + interview-prep |

## How This Section Fits

This section consumes what section 02 built: RAG stops being a single retrieval hop and becomes a loop an agent controls, and the vector store reappears as long-term semantic memory an agent deliberately writes to and selectively retrieves from. It assumes that retrieval-and-serving substrate rather than re-teaching it. Forward, it feeds **section 04 — Production AI Systems**, where the security, evaluation, and scaling concerns raised here as a lens (bounded loops, cost multipliers, HITL approval gates, context-rot degradation) become first-class design and defense topics. It also directly stocks **section 05 — Interview Practice**: the single-agent-vs-multi-agent split, the framework-selection procedure, and the context-budget trade-off are among the highest-yield case-study questions in agentic-systems interviews.

## Study Tips

- **LangGraph is the through-line framework for the whole section** — the StateGraph, checkpointer, `Command`, and `interrupt()` primitives from Chapter 2 recur in the memory chapter and the framework-selection chapter. These frameworks evolve quickly, so verify any specific API detail (node signatures, `Command(goto=..., update=...)`, checkpointer backends, store namespaces) against the current official LangGraph docs before relying on it in an answer.
- **The two highest-yield interview trade-offs live here: single-agent vs multi-agent, and the context budget.** Be able to defend *not* splitting into multiple agents (a raw loop or single agent is the right answer more often than candidates think) and to quantify the ~15× token multiplier a multi-agent system signs up for. On context, internalize that a bigger window is not the fix — "context rot" means you curate the window (write/select/compress/isolate), not enlarge it.
- **Practice bounding every loop out loud.** ReAct iterations, replan/reflection loops, and cyclic graphs all run forever if unbounded — naming the iteration cap or `recursion_limit` as a termination-and-cost guarantee is a fast credibility signal.
- **Keep short-term and long-term memory strictly separate in your mental model:** thread-scoped state behind a checkpointer vs cross-thread data in a store you deliberately write and selectively retrieve. Collapsing them into "we keep the whole history" is the exact mistake the memory chapter exists to correct.
- Do the interview-prep file in each chapter last, as a self-test; if you can answer the framework-selection, topology-choice, and context-management questions without re-reading the notes, you are ready to advance to section 04.
