# Project Handoff

## Project Summary

**AI System Design for AI Engineers** is a Markdown-only learning repository for AI/ML system design interview preparation at the Applied AI Engineer level — designing and defending production-grade agentic AI, RAG, and multi-agent architectures with a security-first, evaluation-first lens. It targets an engineer with 4 years of hands-on production experience (LangGraph multi-agent systems, RAG pipelines, FastAPI/PostgreSQL) formalizing that experience for interviews, not learning AI from scratch. Time budget: ~20–30 hours across 14 chapters.

**Key artifacts:**
- `AGENTS.md` — authoritative authoring rules (Content Depth Rules 1–9, file naming, template mapping). Read before populating any content.
- `templates/` — fully populated reference templates (`chapter-notes-template.md`, `interview-prep-template.md`, `section-index-template.md`, `capstone-template.md`, `authoring-guidelines.md`, `README.md`). Never edit these originals — copy their structure into target files.
- 5 numbered section folders (`01-`–`05-`), each with 2–3 chapter subfolders. Every chapter folder has topic notes plus an `interview-prep.md` numbered immediately after the last topic note.
- `capstone/project-brief.md` — integrative multi-agent platform design project.
- **Sections 01, 02, and 05 are fully authored.** Sections 03 (`03-agentic-and-multi-agent-systems/`) and 04 (`04-production-ai-systems-security-eval-scale/`) content files remain blank stubs (`<!-- stub: populate using templates/ -->`).
- **Section 05 was expanded** with a new Chapter 4, `04-cross-cutting-concerns-and-tradeoff-patterns/`: a `00-golden-rulebook-cheatsheet.md` quick-reference, nine pattern notes (`01`–`09`: latency, scalability, evaluation, cost, reliability, security, data privacy, observability, versioning) + `10-interview-prep.md`.

**Architecture:** Section → Chapter (two levels only, no module layer). Sections ramp: 01 classical systems refresher (light) → 02 LLM serving/RAG → 03 agentic/multi-agent (LangGraph) → 04 production concerns (OWASP security, eval, scaling) → 05 interview practice (delivery framework, worked case studies, mock question banks, cross-cutting trade-off patterns). Filenames are **locked** — real topic-specific kebab-case slugs were chosen deliberately since chapters may forward-link to not-yet-written stubs.

**Critical constants:** lowercase-hyphen naming; no thought-leadership files; no lab files; interview-prep is the only auxiliary file type. Do not rename or renumber any file/folder — this breaks forward links.

---

## Session Log

### Session: 2026-07-29 (current)
**Files touched (all committed in `22b7069`):**
- New chapter `05-interview-practice-case-studies-and-drills/04-cross-cutting-concerns-and-tradeoff-patterns/` (11 files): `00-golden-rulebook-cheatsheet.md`; nine pattern notes `01`–`09` (latency, scalability/throughput, evaluation/QA, cost/token-efficiency, reliability/failure, security/safety, data-privacy/governance, observability/monitoring, versioning/change-mgmt); `10-interview-prep.md`.
- `05-interview-practice-case-studies-and-drills/README.md` (chapters table → 4 rows, est. time 8→11 hrs, overview/outcomes/how-it-fits/study-tips updated to include the new chapter + cheatsheet).
- `.agent_docs/handoff.md`.
**Summary:** Expanded section 05 with a new **cross-cutting concerns** chapter via the orchestrator + subagent workflow (agent orchestrated only; all authoring done by `general` subagents told to write in-run and webfetch-verify every Further Reading link). Per user decisions this run: (1) new chapter placed as `04`, one note per concern axis, using a **custom pattern structure** (TL;DR → ELI5 → Learning Objectives → Visual Overview → The Core Problem → Resolution Options table + per-option prose → Key Parameters → Worked Example → Implementation w/ anti-pattern → Pitfalls → Definitions → Quick Recall → Self-Check → Further Reading); (2) note 01 (latency) authored by the orchestrator as the validated reference, remaining 7 delegated in parallel; (3) added **versioning** and **observability** notes on request — versioning added as `09` and the pre-existing interview-prep renamed `09→10` (verified no external references first) to keep interview-prep last per AGENTS.md; (4) added the `00-golden-rulebook-cheatsheet.md` synthesizing all nine notes into a 60-second framework + 9-concern master table + golden rules + decision cues, linking to every sibling note. All 11 files verified stub-free.
**Outcome:** Chapter 4 fully authored, verified, and **committed** (`22b7069`); working tree clean. This chapter intentionally deviates from the AGENTS.md "topic notes 01–03 / interview-prep 04" convention (10 numbered files, interview-prep at `10`), like Ch2/Ch3 already do.

### Session: 2026-07-29 (previous)
**Files touched:**
- `05-interview-practice-case-studies-and-drills/` (14 files): section `README.md`; `01-delivery-framework-for-ai-system-design-interviews/` — 3 topic notes (`01`–`03`) + `04-interview-prep.md`; `02-worked-case-studies-from-production-agentic-systems/` — 4 case studies (`01`–`04`) + `05-interview-prep.md`; `03-rapid-fire-mock-question-bank-and-self-assessment/` — 3 mock banks (`01`–`03`) + `04-interview-prep.md`.
- Also authored earlier the same date and committed (`04475b3`): all of `01-classical-systems-refresher/` (9 files) and `02-llm-serving-and-rag-architecture/` (13 files).
- `.agent_docs/handoff.md`.
**Summary:** Populated all of section 05 (Interview Practice) via the orchestrator + subagent workflow — agent orchestrated only; 14 `general` subagents did all authoring, each told to write in-run. Per two user decisions that run: (1) Ch2 case studies are realistic reconstructions from the filename slugs with NO fabricated proprietary/IBM-internal specifics; (2) custom structures per file type — Ch1 delivery notes use the standard chapter-notes template, Ch2 uses a case-study walkthrough structure, Ch3 uses a Q-bank format (~24 Qs each: recall/applied/analysis/MCQ rounds + self-assessment scorecard). All Further Reading verified live via webfetch. Sections 01 and 02 were authored earlier the same date and committed as `04475b3`.
**Outcome:** Sections 01, 02, and 05 authored and verified (zero stub markers). Section 05's 14 files were committed later in `22b7069` (this session's commit).

---

## Open Items / Next Steps

No open items from this session — Chapter 4 is complete, verified stub-free, and committed (`22b7069`); working tree is clean.

Carry-over (from prior sessions, not blocking):
- [ ] `05-.../03-rapid-fire-mock-question-bank-and-self-assessment/02-mock-question-bank-multi-agent-and-orchestration.md` and `.../03-mock-question-bank-security-eval-and-scaling.md` — grounded in official docs because their source sections (03 and 04) were still stubs; align for consistency once sections 03 and 04 are authored.

Suggested (not blocking) next authoring target: section `03-agentic-and-multi-agent-systems/`, starting with `01-agent-design-patterns/01-react-and-tool-calling-agent-fundamentals.md`, using the orchestrator + `author-chapter` (or `general`) subagent workflow. This section is LangGraph-heavy — subagents must verify LangGraph API details against current official docs (per AGENTS.md). Sections 03 and 04 are the only remaining unauthored content.

---

## Quick Reference

- **No build system, no tests** — this is a pure Markdown content repo.
- **Authoring source of truth:** `AGENTS.md` at repo root — read before populating any stub.
- **Template files (reference-only, never edit):** `templates/chapter-notes-template.md`, `templates/interview-prep-template.md`, `templates/section-index-template.md`, `templates/capstone-template.md`, `templates/authoring-guidelines.md`.
- **Stub marker:** unpopulated files contain exactly `<!-- stub: populate using templates/ -->` — intentional, not missing content.
- **Skills for authoring:** `author-chapter` (researches live sources, follows `chapter-notes-template.md` + Content Depth Rules 1–9, runs a quality gate); `generate-practice-exam` (grounds questions in authored content only).
- **Orchestrator pattern that works:** delegate one file per `general` subagent; pass the exact target path, topic framing, and mandatory template/section requirements; run subagents in parallel; explicitly instruct each subagent to WRITE in-run and to webfetch-verify every Further Reading URL before writing it. If one stops to ask, resume it with its `task_id`. For a synthesis/index file, have subagents READ the sibling notes first so claims stay consistent.
- **Custom file structures used in section 05:** Ch1 delivery-framework notes = standard chapter-notes template; Ch2 worked case studies = case-study walkthrough (Prompt→Requirements→Architecture→Decisions→Deep Dive→Failure Modes→Implementation Sketch→45-min plan→Q&A→Definitions→Further Reading); Ch3 mock banks = Q-bank (~24 Qs, recall/applied/analysis/MCQ rounds + scorecard); **Ch4 pattern notes = problem→resolution-options-table→scenario custom structure** (TL;DR, ELI5, Learning Objectives, Visual Overview, The Core Problem, Resolution Options, Key Parameters, Worked Example, Implementation w/ anti-pattern, Pitfalls, Definitions, Quick Recall, Self-Check, Further Reading); **Ch4 `00-golden-rulebook-cheatsheet.md` = dense skim-first reference** (60-second framework, clarifying-questions checklist, 9-concern master table linking to notes 01–09, golden rules, decision cues, trade-off one-liners, red flags, 2-minute scan). All still obey AGENTS.md style + depth spirit.
- **Chapter file numbering:** standard chapters use topic notes `01`–`03` + interview-prep at the next number; **exceptions:** `05-.../02-worked-case-studies-.../` uses `01`–`04` + `05-interview-prep`, and `05-.../04-cross-cutting-concerns-and-tradeoff-patterns/` uses `00-cheatsheet` + `01`–`09` + `10-interview-prep`.
- **Filenames are locked** — never rename/renumber; chapters may forward-link to stubs not yet written. (The `09→10-interview-prep` rename this session was a one-time safe fix done before the file had any external references or a commit.)
- **Section structure:** Section → Chapter only (no module layer).
- **5 sections:** `01-classical-systems-refresher/` (**authored**), `02-llm-serving-and-rag-architecture/` (**authored**), `03-agentic-and-multi-agent-systems/` (stubs), `04-production-ai-systems-security-eval-scale/` (stubs), `05-interview-practice-case-studies-and-drills/` (**authored**, now 4 chapters).
- **Dense chapters in section 05:** `02-worked-case-studies-.../` (4 case studies + interview-prep); `04-cross-cutting-concerns-and-tradeoff-patterns/` (cheatsheet + 9 pattern notes + interview-prep).
- **No labs, no thought-leadership files** — only topic notes + interview-prep (+ the one cheatsheet) per chapter.
- **Markdown style:** H1 title, H2 major sections, H3 sub-sections; all code blocks carry a language tag; `---` separates major sections; Self-Check / drill answers use `<details><summary>Answer</summary>`.
- **External links rule:** official/authoritative documentation only, verified with `webfetch`, format `[Title](url) — *verified YYYY-MM-DD*`. No Medium/YouTube.
- **Git:** commits so far — `f15e3ea` initial, `5d54117` scaffold, `04475b3` sections 01+02 content, `22b7069` section 05 content (all four chapters incl. cross-cutting) + handoff. Working tree currently clean. Note: `06-interview-questions-dump/question1.md` is a tracked empty placeholder outside the numbered curriculum.
- **Verify no stubs remain in a folder:** `grep -rl "stub: populate" <folder>/` should return nothing.
