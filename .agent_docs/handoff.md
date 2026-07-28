# Project Handoff

## Project Summary

**AI System Design for AI Engineers** is a Markdown-only learning repository for AI/ML system design interview preparation at the Applied AI Engineer level — designing and defending production-grade agentic AI, RAG, and multi-agent architectures with a security-first, evaluation-first lens. It targets an engineer with 4 years of hands-on production experience (LangGraph multi-agent systems, RAG pipelines, FastAPI/PostgreSQL) formalizing that experience for interviews, not learning AI from scratch. Time budget: ~20–30 hours across 14 chapters.

**Key artifacts:**
- `AGENTS.md` — authoritative authoring rules (Content Depth Rules 1–9, file naming, template mapping). Read before populating any content.
- `templates/` — fully populated reference templates (`chapter-notes-template.md`, `interview-prep-template.md`, `section-index-template.md`, `capstone-template.md`, `authoring-guidelines.md`, `README.md`). Never edit these originals — copy their structure into target files.
- 5 numbered section folders (`01-`–`05-`), each with 2–3 chapter subfolders. Every chapter folder has topic notes (`01-*.md`–`03-*.md`, or `01`–`04` for the one dense chapter) plus an `interview-prep.md` numbered immediately after the last topic note.
- `capstone/project-brief.md` — integrative multi-agent platform design project.
- Section 01 (`01-classical-systems-refresher/`) is now **fully authored** (9 files). All other section content files remain blank stubs (`<!-- stub: populate using templates/ -->`).

**Architecture:** Section → Chapter (two levels only, no module layer). Sections ramp: 01 classical systems refresher (light) → 02 LLM serving/RAG → 03 agentic/multi-agent (LangGraph) → 04 production concerns (OWASP security, eval, scaling) → 05 interview practice (delivery framework, worked case studies mapped to the user's real IBM projects, mock question banks). Filenames are **locked** — real topic-specific kebab-case slugs were chosen deliberately since chapters may forward-link to not-yet-written stubs.

**Critical constants:** lowercase-hyphen naming; no thought-leadership files; no lab files; interview-prep is the only auxiliary file type. Do not rename or renumber any file/folder — this breaks forward links.

---

## Session Log

### Session: 2026-07-29 (current)
**Files touched (all under `01-classical-systems-refresher/`, uncommitted working-tree changes):**
- `README.md` (section index)
- `01-scalability-and-distributed-systems-primer/` — `01-scalability-load-balancing-and-caching.md`, `02-consistency-availability-and-the-cap-theorem.md`, `03-message-queues-and-asynchronous-processing.md`, `04-interview-prep.md`
- `02-apis-databases-and-data-platforms-for-ai/` — `01-rest-and-fastapi-service-design-for-ai-systems.md`, `02-relational-vs-nosql-vs-vector-storage-tradeoffs.md`, `03-etl-pipelines-and-data-platforms-at-scale.md`, `04-interview-prep.md`
**Summary:** Populated the entire `01-classical-systems-refresher` section using an orchestrator + subagent workflow (agent acted as orchestrator; all authoring delegated to `general` subagents, which invoked the `author-chapter` skill). Authored 6 topic notes (each full `chapter-notes-template.md` with ELI5, Visual Overview, per-concept what/how/where, Key Parameters table, 5-step Worked Example, anti-pattern + fix, 3-part pitfalls, 5 Self-Check Qs incl. multi-select, verified official-doc Further Reading), 2 interview-prep files grounded in the authored notes (7 conceptual + 2 scenario + 1 design Q each), and the section README (5 hrs estimated, 2-chapter table). One subagent (scalability) initially returned a research brief for approval; it was resumed via `task_id` to complete the write and passed its quality gate.
**Outcome:** All 9 files in section 01 populated and verified (zero stub markers remain; every Further Reading link verified live with webfetch). Changes are **uncommitted** in the working tree — not yet committed to git.

### Session: 2026-07-28 (previous)
**Files touched:** Full repo scaffold — `AGENTS.md`, `README.md`, `templates/` (6 files), `00-roadmap/learning-roadmap.md`, `progress-tracker.md`, `capstone/project-brief.md`, all 5 section `README.md` stubs, and 14 chapter folders across `01-classical-systems-refresher/` through `05-interview-practice-case-studies-and-drills/` (43 topic-note stubs + 14 interview-prep stubs).
**Summary:** Ran the `create-learning-repo` workflow end-to-end: researched current sources (OWASP GenAI LLM Top 10:2025, Anthropic context-engineering post, LangChain/LangGraph docs, Ragas docs, Hello Interview ML system design framework, system-design-primer) to ground the curriculum; confirmed scope with the user (20–30h budget, mostly AI-native with light classical refresher, interview-prep only, lowercase-hyphen naming, no 3rd hierarchy level); wrote `AGENTS.md` and all templates; scaffolded 73 total files across 5 sections / 14 chapters with real topic-specific filenames.
**Outcome:** Full skeleton committed to git (`5d54117 chore: scaffold full repo structure with stubs, templates, and AGENTS.md`). Working tree was clean. Repo ready for content authoring — no chapters populated yet.

---

## Open Items / Next Steps

- [ ] Section 01 authored content is **uncommitted** — the 9 modified files under `01-classical-systems-refresher/` need to be committed to git (user has not yet requested a commit).

Suggested (not blocking) next authoring target for a future session: begin section `02-llm-serving-and-rag-architecture/`, starting with `01-llm-inference-and-serving-economics/01-llm-inference-fundamentals-latency-and-throughput.md`, using the same orchestrator + `author-chapter` subagent workflow.

---

## Quick Reference

- **No build system, no tests** — this is a pure Markdown content repo.
- **Authoring source of truth:** `AGENTS.md` at repo root — read before populating any stub.
- **Template files (reference-only, never edit):** `templates/chapter-notes-template.md`, `templates/interview-prep-template.md`, `templates/section-index-template.md`, `templates/capstone-template.md`, `templates/authoring-guidelines.md`.
- **Stub marker:** unpopulated files contain exactly `<!-- stub: populate using templates/ -->` — intentional, not missing content.
- **Skills for authoring:** `author-chapter` (researches live sources, follows `chapter-notes-template.md` + Content Depth Rules 1–9, runs a quality gate); `generate-practice-exam` (only after chapters are authored — grounds questions in existing content).
- **Orchestrator pattern that worked this session:** delegate one file per `general` subagent; pass the exact target path, the topic framing, and the mandatory template/section requirements; run subagents in parallel. If a subagent stops to ask for approval, resume it with its `task_id`.
- **Chapter file numbering:** topic notes `01`–`03` (or `01`–`04` for the dense chapter `05-.../02-worked-case-studies-.../`); interview-prep is always the number immediately after the last topic note (`04` or `05`).
- **Filenames are locked** — never rename/renumber; chapters may forward-link to stubs not yet written.
- **Section structure:** Section → Chapter only (no module layer).
- **5 sections:** `01-classical-systems-refresher/` (**fully authored**), `02-llm-serving-and-rag-architecture/`, `03-agentic-and-multi-agent-systems/`, `04-production-ai-systems-security-eval-scale/`, `05-interview-practice-case-studies-and-drills/`.
- **Dense chapter:** `05-.../02-worked-case-studies-from-production-agentic-systems/` has 4 topic notes (user's real IBM projects: document extraction/dispatch, self-service analytics/RBAC, conversational analytics/NL-to-SQL, employee query routing) + `05-interview-prep.md`.
- **No labs, no thought-leadership files** — only topic notes + interview-prep per chapter.
- **Markdown style:** H1 title, H2 major sections, H3 sub-sections; all code blocks carry a language tag; `---` separates major sections; Self-Check answers use `<details><summary>Answer</summary>`.
- **External links rule:** official documentation only, verified with `webfetch`, format `[Title](url) — *verified YYYY-MM-DD*`.
- **Git:** repo git-initialized; scaffold committed as `5d54117`. Section 01 authored content is currently **uncommitted** in the working tree. Untracked: `.agent_docs/`, `06-interview-questions-dump/`.
- **Verify no stubs remain in a folder:** `grep -rl "stub: populate" <folder>/` should return nothing.
