# Project Handoff

## Project Summary

**AI System Design for AI Engineers** is a Markdown-only learning repository for AI/ML system design interview preparation at the Applied AI Engineer level — designing and defending production-grade agentic AI, RAG, and multi-agent architectures with a security-first, evaluation-first lens. It targets an engineer with 4 years of hands-on production experience (LangGraph multi-agent systems, RAG pipelines, FastAPI/PostgreSQL) formalizing that experience for interviews, not learning AI from scratch. Time budget: ~20–30 hours across 14 chapters.

**Key artifacts:**
- `AGENTS.md` — authoritative authoring rules (Content Depth Rules 1–9, file naming, template mapping). Read before populating any content.
- `templates/` — fully populated reference templates (`chapter-notes-template.md`, `interview-prep-template.md`, `section-index-template.md`, `capstone-template.md`, `authoring-guidelines.md`, `README.md`). Never edit these originals — copy their structure into target files.
- 5 numbered section folders (`01-`–`05-`), each with 2–3 chapter subfolders. Every chapter folder has topic notes (`01-*.md`–`03-*.md`, or `01`–`04` for the one dense chapter) plus an `interview-prep.md` numbered immediately after the last topic note.
- `capstone/project-brief.md` — integrative multi-agent platform design project.
- **Sections 01, 02, and 05 are now fully authored.** Sections 03 (`03-agentic-and-multi-agent-systems/`) and 04 (`04-production-ai-systems-security-eval-scale/`) content files remain blank stubs (`<!-- stub: populate using templates/ -->`).

**Architecture:** Section → Chapter (two levels only, no module layer). Sections ramp: 01 classical systems refresher (light) → 02 LLM serving/RAG → 03 agentic/multi-agent (LangGraph) → 04 production concerns (OWASP security, eval, scaling) → 05 interview practice (delivery framework, worked case studies mapped to the user's real IBM projects, mock question banks). Filenames are **locked** — real topic-specific kebab-case slugs were chosen deliberately since chapters may forward-link to not-yet-written stubs.

**Critical constants:** lowercase-hyphen naming; no thought-leadership files; no lab files; interview-prep is the only auxiliary file type. Do not rename or renumber any file/folder — this breaks forward links.

---

## Session Log

### Session: 2026-07-29 (current)
**Files touched:**
- `05-interview-practice-case-studies-and-drills/` (14 files): section `README.md`; `01-delivery-framework-for-ai-system-design-interviews/` — 3 topic notes (`01`–`03`) + `04-interview-prep.md`; `02-worked-case-studies-from-production-agentic-systems/` — 4 case studies (`01`–`04`) + `05-interview-prep.md`; `03-rapid-fire-mock-question-bank-and-self-assessment/` — 3 mock banks (`01`–`03`) + `04-interview-prep.md`.
- Also authored earlier the same date and already committed (`04475b3`): all of `01-classical-systems-refresher/` (9 files) and `02-llm-serving-and-rag-architecture/` (13 files).
- `.agent_docs/handoff.md`.
**Summary:** Populated all of section 05 (Interview Practice) via the orchestrator + subagent workflow — agent orchestrated only; 14 `general` subagents did all authoring, each told to write in-run. Per two user decisions this run: (1) Ch2 case studies are realistic reconstructions from the filename slugs with NO fabricated proprietary/IBM-internal specifics; (2) custom structures per file type — Ch1 delivery notes use the standard chapter-notes template, Ch2 uses a case-study walkthrough structure (Prompt → Requirements → Architecture → Decisions/Tradeoffs → Deep Dive → Failure Modes → Implementation Sketch → 45-min delivery plan → Q&A drill → Definitions → Further Reading), Ch3 uses a Q-bank format (~24 Qs each: recall/applied/analysis/MCQ rounds + self-assessment scorecard). All Further Reading verified live via webfetch (Hello Interview, Anthropic, OWASP LLM Top 10:2025, LangGraph/LangChain, PostgreSQL RLS/roles, Ragas, pgvector/FAISS/Sentence-Transformers). Earlier the same date, sections 01 and 02 were authored via the identical workflow and committed as `04475b3`.
**Outcome:** Sections 01, 02, and 05 authored and verified (zero stub markers remain). Sections 01+02 are committed (`04475b3`); **section 05's 14 files are uncommitted** in the working tree.

### Session: 2026-07-28 (previous)
**Files touched:** Full repo scaffold — `AGENTS.md`, `README.md`, `templates/` (6 files), `00-roadmap/learning-roadmap.md`, `progress-tracker.md`, `capstone/project-brief.md`, all 5 section `README.md` stubs, and 14 chapter folders across `01-classical-systems-refresher/` through `05-interview-practice-case-studies-and-drills/` (43 topic-note stubs + 14 interview-prep stubs).
**Summary:** Ran the `create-learning-repo` workflow end-to-end: researched current sources (OWASP GenAI LLM Top 10:2025, Anthropic context-engineering post, LangChain/LangGraph docs, Ragas docs, Hello Interview ML system design framework, system-design-primer) to ground the curriculum; confirmed scope with the user (20–30h budget, mostly AI-native with light classical refresher, interview-prep only, lowercase-hyphen naming, no 3rd hierarchy level); wrote `AGENTS.md` and all templates; scaffolded 73 total files across 5 sections / 14 chapters with real topic-specific filenames.
**Outcome:** Full skeleton committed to git (`5d54117 chore: scaffold full repo structure with stubs, templates, and AGENTS.md`). Working tree was clean. Repo ready for content authoring — no chapters populated yet.

---

## Open Items / Next Steps

- [ ] Section 05 authored content is **uncommitted** — the 14 modified files under `05-interview-practice-case-studies-and-drills/` (plus `.agent_docs/handoff.md`) need to be committed to git (user has not yet requested a commit).
- [ ] `05-.../03-rapid-fire-mock-question-bank-and-self-assessment/02-mock-question-bank-multi-agent-and-orchestration.md` and `.../03-mock-question-bank-security-eval-and-scaling.md` — these were grounded in official docs because their source sections (03 and 04) were still stubs; revisit for consistency once sections 03 and 04 are authored.

Suggested (not blocking) next authoring target: section `03-agentic-and-multi-agent-systems/`, starting with `01-agent-design-patterns/01-react-and-tool-calling-agent-fundamentals.md`, using the same orchestrator + `author-chapter` subagent workflow. This section is LangGraph-heavy — subagents must verify LangGraph API details against current official docs (per AGENTS.md). Sections 03 and 04 are the only remaining unauthored content.

---

## Quick Reference

- **No build system, no tests** — this is a pure Markdown content repo.
- **Authoring source of truth:** `AGENTS.md` at repo root — read before populating any stub.
- **Template files (reference-only, never edit):** `templates/chapter-notes-template.md`, `templates/interview-prep-template.md`, `templates/section-index-template.md`, `templates/capstone-template.md`, `templates/authoring-guidelines.md`.
- **Stub marker:** unpopulated files contain exactly `<!-- stub: populate using templates/ -->` — intentional, not missing content.
- **Skills for authoring:** `author-chapter` (researches live sources, follows `chapter-notes-template.md` + Content Depth Rules 1–9, runs a quality gate); `generate-practice-exam` (grounds questions in authored content only).
- **Orchestrator pattern that works:** delegate one file per `general` subagent; pass the exact target path, topic framing, and mandatory template/section requirements; run subagents in parallel; explicitly instruct each subagent to WRITE in-run (do not stop for approval). If one stops to ask, resume it with its `task_id`.
- **Custom file structures used in section 05:** delivery-framework notes = standard chapter-notes template; worked case studies = case-study walkthrough (Prompt→Requirements→Architecture→Decisions→Deep Dive→Failure Modes→Implementation Sketch→45-min plan→Q&A→Definitions→Further Reading); mock banks = Q-bank (~24 Qs, recall/applied/analysis/MCQ rounds + scorecard). All still obey AGENTS.md style + depth spirit.
- **Chapter file numbering:** topic notes `01`–`03` (or `01`–`04` for the dense chapter `05-.../02-worked-case-studies-.../`); interview-prep is always the number immediately after the last topic note (`04`, or `05` for the dense chapter).
- **Filenames are locked** — never rename/renumber; chapters may forward-link to stubs not yet written.
- **Section structure:** Section → Chapter only (no module layer).
- **5 sections:** `01-classical-systems-refresher/` (**authored**), `02-llm-serving-and-rag-architecture/` (**authored**), `03-agentic-and-multi-agent-systems/` (stubs), `04-production-ai-systems-security-eval-scale/` (stubs), `05-interview-practice-case-studies-and-drills/` (**authored**).
- **Dense chapter:** `05-.../02-worked-case-studies-from-production-agentic-systems/` has 4 case-study notes (topics: document extraction/dispatch, self-service analytics/RBAC, conversational analytics/NL-to-SQL, employee query routing) + `05-interview-prep.md`.
- **No labs, no thought-leadership files** — only topic notes + interview-prep per chapter.
- **Markdown style:** H1 title, H2 major sections, H3 sub-sections; all code blocks carry a language tag; `---` separates major sections; Self-Check / drill answers use `<details><summary>Answer</summary>`.
- **External links rule:** official/authoritative documentation only, verified with `webfetch`, format `[Title](url) — *verified YYYY-MM-DD*`. No Medium/YouTube.
- **Git:** commits so far — `f15e3ea` initial, `5d54117` scaffold, `04475b3` sections 01+02 content + handoff. Section 05 content is currently **uncommitted**. Note: `06-interview-questions-dump/question1.md` is a tracked empty placeholder outside the numbered curriculum.
- **Verify no stubs remain in a folder:** `grep -rl "stub: populate" <folder>/` should return nothing.
