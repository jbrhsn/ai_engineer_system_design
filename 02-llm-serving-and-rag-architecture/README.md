# 02 — LLM Serving and RAG Architecture

**Estimated time:** 8 hrs | **Prerequisites:** Section 01 (or equivalent distributed-systems, API, and storage fundamentals)

## Overview

This is the first AI-native section and the retrieval-and-serving foundation of the whole curriculum: it takes the classical scaling, consistency, and storage trade-offs from section 01 and specialises them to language models — how inference actually costs latency and money, how embeddings turn text into searchable vectors, and how RAG pipelines wire retrieval into generation without hallucinating. Almost every AI system design interview lives here, because a serving-cost or retrieval-quality trade-off is the fastest way for an interviewer to see whether you have shipped production systems or only read about them. Master this section and you have the substrate that section 03's agents are built on top of.

## Learning Outcomes

By completing this section you will be able to:
- Reason quantitatively about LLM inference — prefill vs decode, latency vs throughput, batching, and KV-cache pressure — and estimate the token cost and tail latency of a given serving design.
- Compare model serving architectures and local-vs-hosted deployment options, and defend a build/buy/self-host decision against cost, latency, privacy, and utilisation constraints.
- Choose embedding models and vector representations, then select and tune a vector index (recall vs latency vs memory) for a stated retrieval workload.
- Design hybrid (dense + lexical) retrieval with reranking, and justify when each stage earns its added latency and cost.
- Architect an end-to-end RAG pipeline — chunking, retrieval, grounding, and faithfulness/hallucination controls — and extend it toward advanced and agentic retrieval that bridges into multi-agent systems.

## Chapters

| # | Chapter | Est. time | Files |
|---|---|---|---|
| 1 | LLM Inference and Serving Economics | 2.5 hrs | notes + interview-prep |
| 2 | Embeddings and Vector Search | 2.5 hrs | notes + interview-prep |
| 3 | RAG Pipeline Design Patterns | 3 hrs | notes + interview-prep |

## How This Section Fits

This section is where section 01's fundamentals stop being generic and become AI-specific: horizontal scaling becomes GPU batching, the relational-vs-vector storage boundary becomes concrete index tuning, and delivery semantics resurface in batch embedding and retrieval pipelines. It assumes that baseline and does not re-teach it. Forward, it unlocks section **03 — Agentic and Multi-Agent Systems**: the final chapter's advanced and *agentic RAG* patterns are the deliberate bridge, where retrieval stops being a single hop and starts being a loop an agent controls. Treat RAG here as the last "single-shot" architecture before state, tools, and multi-step planning enter the picture.

## Study Tips

- The cost-and-latency framing introduced in Chapter 1 recurs through the entire section — carry it forward as a lens, not a one-time topic. Every later design choice (which index, how many rerank candidates, how big a chunk) is ultimately a latency-and-dollars trade-off, and interviewers reward candidates who name that trade-off explicitly.
- The interview-dense spots to slow down on: **ANN recall-vs-latency tuning** (Chapter 2 — be able to reason about the index knobs, not just name HNSW/IVF), **hybrid search + reranking** (Chapter 2 — know *when* each stage is worth its cost), **chunking strategy** (Chapter 3 — the highest-leverage, most-overlooked retrieval-quality lever), and **hallucination / faithfulness design** (Chapter 3 — "how do you make it not make things up" is a near-guaranteed follow-up).
- Practise sketching the full RAG data flow end to end from memory — ingestion → chunk → embed → index → retrieve → rerank → ground → generate → evaluate. Being able to draw it and then defend one node's design choice under questioning is the core deliverable of this section.
- Do the interview-prep file in each chapter last, as a self-test; if you can answer those without re-reading the notes — especially the retrieval-quality and serving-cost questions — you are ready to advance to section 03.
