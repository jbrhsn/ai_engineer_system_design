# 01 — Classical Systems Refresher

**Estimated time:** 5 hrs | **Prerequisites:** None

## Overview

This is the opening, deliberately lighter section of the curriculum: a fast refresher on the classical distributed-systems and data fundamentals that every AI system design still rests on, reframed toward LLM/RAG serving. It exists to reload the vocabulary and trade-off reasoning — scalability, CAP/consistency, async messaging, API and storage design — before the AI-native sections layer inference, retrieval, and agents on top. For an experienced Applied AI Engineer this is review, but review that establishes the shared language the rest of the interview is conducted in.

## Learning Outcomes

By completing this section you will be able to:
- Reason about horizontal vs vertical scaling, load balancing, and caching, and map each to the specific pressures of LLM/RAG serving (token cost, latency, GPU utilisation).
- State the CAP theorem precisely, extend it with PACELC, and defend a consistency/availability choice for a given AI serving or state-management scenario.
- Choose and justify delivery semantics (at-most-once, at-least-once, exactly-once) for async and queue-backed AI workloads such as batch embedding or long-running agent jobs.
- Design a REST/FastAPI service surface for an AI system and select among relational, NoSQL, and vector storage based on the workload's real access patterns.
- Sketch an ETL / data-platform pipeline that feeds ingestion and retrieval at scale, and articulate its failure and freshness trade-offs.

## Chapters

| # | Chapter | Est. time | Files |
|---|---|---|---|
| 1 | Scalability and Distributed Systems Primer | 2.5 hrs | notes + interview-prep |
| 2 | APIs, Databases, and Data Platforms for AI | 2.5 hrs | notes + interview-prep |

## How This Section Fits

As the first section, it has no prerequisites and connects back to nothing prior — it is the foundation the rest of the curriculum assumes rather than re-teaches. It unlocks section **02 — LLM Serving & RAG**, where the same scaling, consistency, and storage trade-offs reappear specialised to inference endpoints, embedding stores, and retrieval pipelines. Treat the concepts here as the baseline an interviewer will expect you to reach for reflexively before you ever discuss a model.

## Study Tips

- This is a refresher: if you already run production distributed systems, move quickly through the parts you know cold and spend your saved time on the AI-specific reframings rather than the classical definitions.
- The two interview-dense spots to slow down on are **CAP / PACELC** (Chapter 1, note 2) and **delivery semantics** (Chapter 1, note 3) — interviewers probe these hard because they separate people who memorised acronyms from people who can defend a trade-off under follow-up questions. Be able to justify a choice, not just recite the theorem.
- For Chapter 2, focus on the *decision rules* for storage selection rather than feature lists; the interview value is in matching a store to an access pattern, especially the relational-vs-vector boundary that recurs throughout the RAG sections.
- Do the interview-prep file in each chapter last, as a self-test — if you can answer those without re-reading the notes, you are ready to advance to section 02.
