# 05 — Interview Practice: Case Studies and Drills

**Estimated time:** 11 hrs | **Prerequisites:** Sections 01–04 (ideally all four, since this section synthesises and drills everything) — though Chapter 1, the delivery framework, is usable standalone at any point.

## Overview

This is the final, capstone-style section of the curriculum: where the knowledge from sections 01–04 stops being studied and starts being *rehearsed under interview conditions*. You learn a repeatable delivery framework for running an AI system design interview, work four detailed case studies drawn from real production agentic systems, self-drill with rapid-fire mock question banks, and finally consolidate the recurring cross-cutting concerns (latency, scalability, evaluation, cost, reliability, security, privacy, observability, versioning) into reusable trade-off patterns you can reach for on any prompt. Nothing new is taught here that the earlier sections did not establish — the value is in synthesis, timing, and articulation: turning what you know into what you can deliver on a whiteboard while an interviewer probes.

## Learning Outcomes

By completing this section you will be able to:
- Run an AI system design interview end to end with a repeatable delivery framework — clarifying an ambiguous prompt, scoping ruthlessly, driving the whiteboard, and communicating trade-offs out loud rather than in your head.
- Reconstruct four production-grade agentic architectures from a prompt — document extraction with multi-agent dispatch, self-service analytics with RBAC and SQL validation, conversational analytics with NL-to-SQL, and employee query routing across data sources — and defend each design decision against cost, latency, security, and evaluation constraints.
- Map an open-ended interview prompt onto the specific patterns from sections 02–04 (retrieval, orchestration, security, evaluation, scaling) and select the right one under time pressure.
- Answer rapid-fire questions across RAG/retrieval, multi-agent/orchestration, and security/eval/scaling at recall speed, exposing the topics where your knowledge is shallow and needs another pass.
- Recognise which cross-cutting concern (latency, scalability, evaluation, cost, reliability, security, data privacy, observability, or versioning) an open-ended prompt is really probing, and answer with the right resolution pattern and its trade-offs rather than optimising a single axis.
- Self-assess readiness: identify which prior sections to revisit before you sit a real interview.

## Chapters

| # | Chapter | Est. time | Files |
|---|---|---|---|
| 1 | Delivery Framework for AI System Design Interviews | 2 hrs | 3 notes + interview-prep |
| 2 | Worked Case Studies from Production Agentic Systems | 4 hrs | 4 case-study notes + interview-prep (the dense chapter) |
| 3 | Rapid-Fire Mock Question Bank and Self-Assessment | 2 hrs | 3 mock banks + interview-prep |
| 4 | Cross-Cutting Concerns and Trade-off Patterns | 3 hrs | 9 pattern notes + interview-prep |

## How This Section Fits

This is the last section, and unlike every prior one it unlocks nothing further — it is where the entire curriculum gets cashed in. Section 01's distributed-systems vocabulary, section 02's serving-and-retrieval trade-offs, section 03's agentic and multi-agent patterns, and section 04's production security, evaluation, and scaling discipline all reappear here, not as topics to learn but as tools to reach for under questioning. The four worked case studies in Chapter 2 map onto real production agentic systems, so they are the closest this repo comes to a live interview — each one forces you to compose patterns from several earlier sections into a single coherent design. Chapter 4 then abstracts the recurring follow-up axes every interviewer probes — latency, scalability, evaluation, cost, reliability, security, data privacy, observability, versioning — into nine reusable problem → resolution-options → scenario notes, so that whatever the prompt, you have a ready trade-off pattern rather than a blank stare. Treat this section as the rehearsal room after four sections of practice, the rapid-fire banks in Chapter 3 as the flashcards that tell you what to relearn before showtime, and Chapter 4 as the trade-off cheat-sheet you rehearse until each concern's options are reflexive.

## Study Tips

- **Do Chapter 1 first, and do it before you feel "ready."** The delivery framework is what keeps you from freezing on an ambiguous prompt; it is the one chapter worth studying even if you have not finished sections 03–04. Internalise the clarify → scope → drive → communicate loop until it is reflexive.
- **Work the Chapter 2 case studies as timed drills, not reading.** For each one, cover the solution, set a timer, and attempt the design from the prompt alone — clarifying questions first, then a whiteboard sketch, then trade-offs spoken aloud. Only then read the worked solution and diff it against yours. This is the densest chapter (four full case studies); budget the most time here and expect to revisit it.
- **Use the Chapter 3 mock banks for spaced repetition,** not a single pass. Cycle through them across several days so recall gets faster, and use wrong answers as a map back to the specific earlier chapter to reread.
- **Heads-up on prerequisites:** the multi-agent/orchestration and security/eval/scaling mock banks (Chapter 3, notes 2 and 3) lean on sections **03** and **04**. If those sections are still stubs in your copy, those two banks will reference material you have not yet studied — do the RAG/retrieval bank first (grounded in the completed section 02) and return to the other two after sections 03–04 are authored.
- **Treat Chapter 4 as a trade-off cheat-sheet, not linear reading.** Each of the nine notes follows the same problem → resolution-options → scenario shape, so drill them by axis: given a mock prompt, name which concern is being probed and recite its resolution options with their costs out loud. These notes cross-link to sections 02–04 for the underlying mechanics; use a weak answer as a pointer back to the source chapter. **Start (and end) with `04-.../00-golden-rulebook-cheatsheet.md`** — the one-page framework + 9-concern sweep table that indexes the whole chapter and is the file to skim in the last minutes before an interview.
- Do the interview-prep file in each chapter last, as a self-test; if you can deliver a case study cold, clear the mock banks at speed, and answer any cross-cutting follow-up with its trade-offs, you are ready to sit the real thing.
