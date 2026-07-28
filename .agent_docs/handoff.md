# Project Handoff

## Project Summary

**AI System Design for AI Engineers** is a Markdown-only learning repository for AI/ML system design interview preparation at the Applied AI Engineer level — designing and defending production-grade agentic AI, RAG, and multi-agent architectures with a security-first, evaluation-first lens. It targets an engineer with 4 years of hands-on production experience (LangGraph multi-agent systems, RAG pipelines, FastAPI/PostgreSQL) formalizing that experience for interviews, not learning AI from scratch. Time budget: ~20–30 hours across 14 chapters.

**Key artifacts:**
- `AGENTS.md` — authoritative authoring rules (Content Depth Rules 1–9, file naming, template mapping). Read before populating any content.
- `templates/` — fully populated reference templates (`chapter-notes-template.md`, `interview-prep-template.md`, `section-index-template.md`, `capstone-template.md`, `authoring-guidelines.md`, `README.md`). Never edit these originals — copy their structure into target files.
- 5 numbered section folders (`01-`–`05-`), each with 2–3 chapter subfolders. Every chapter folder has topic notes plus an `interview-prep.md` numbered immediately after the last topic note.
- `capstone/project-brief.md` — integrative multi-agent platform design project.
- **Sections 01, 02, and 05 are fully authored.** Sections 03 (`03-agentic-and-multi-agent-systems/`) and 04 (`04-production-ai-systems-security-eval-scale/`) content files remain blank stubs (`<!-- stub: populate using templates/ -->`).
- **Section 05, Chapter 4** (`04-cross-cutting-concerns-and-tradeoff-patterns/`) holds `00-golden-rulebook-cheatsheet.md` + nine pattern notes (`01`–`09`) + `10-interview-prep.md`, PLUS a `golden_rulebook_examples/` subfolder of worked system-design questions grounded in the cheatsheet.

**Architecture:** Section → Chapter (two levels only, no module layer). Sections ramp: 01 classical systems refresher (light) → 02 LLM serving/RAG → 03 agentic/multi-agent (LangGraph) → 04 production concerns (OWASP security, eval, scaling) → 05 interview practice (delivery framework, worked case studies, mock question banks, cross-cutting trade-off patterns + golden-rulebook worked examples). Filenames are **locked** — real topic-specific kebab-case slugs were chosen deliberately since chapters may forward-link to not-yet-written stubs.

**Critical constants:** lowercase-hyphen naming; no thought-leadership files; no lab files; interview-prep is the only auxiliary file type. Do not rename or renumber any file/folder — this breaks forward links.

---

## Session Log

### Session: 2026-07-29 (current)
**Files touched (in `golden_rulebook_examples/`):**
- `05-interview-practice-case-studies-and-drills/04-cross-cutting-concerns-and-tradeoff-patterns/golden_rulebook_examples/` — five worked-question files:
  - `01-sample-question-latency-and-scalability.md`, `02-sample-question-security-and-reliability.md`, `03-sample-question-evaluation-and-cost.md`, `04-sample-question-privacy-observability-versioning.md` (created, then **revised** to the rich format — committed in `2376625`).
  - `05-sample-question-enterprise-document-qa-platform.md` (**new, untracked** — not yet committed).
- `.agent_docs/handoff.md`.
**Summary:** Built and iterated a `golden_rulebook_examples/` folder of worked system-design questions, all grounded strictly in `00-golden-rulebook-cheatsheet.md`, via the orchestrator + `general` subagent workflow (agent orchestrated only; every file written by a subagent in-run). Three passes: (1) two initial files (latency/scalability, security/reliability); (2) two more (evaluation/cost, privacy/observability/versioning) — user constraint: no video-based scenarios; (3) user judged the short single-axis Q&A too thin and supplied a reference-quality **multi-part "System Design:" prompt** (Enterprise Document Q&A Platform — Context / numbered Requirements+SLOs / 6-facet "Your task" list). Per that decision this run: added the Enterprise Doc Q&A prompt verbatim as `05` with a full worked answer, and **revised `01`–`04` to the same rich format** (each now a `## The Question` with Context + Requirements + numbered task list, then `## Worked Answer` with a clarify/SLO opener + `### Answer` with one `####` per task + ASCII diagrams, then `### How the cheatsheet was used` traceability + `## Cheatsheet elements referenced` index). Every recommendation traces to a named cheatsheet element; unit prices in cost models are explicitly labelled assumptions; no invented vendor/product names. All five verified: no stub markers, no video scenarios, four structural anchors present in each.
**Outcome:** Five worked-example files complete and verified. `01`–`04` committed in `2376625`; `05` (Enterprise Doc Q&A) is **written but untracked/uncommitted** — the only pending git action.

### Session: 2026-07-29 (previous)
**Files touched (all committed in `22b7069`):**
- New chapter `05-interview-practice-case-studies-and-drills/04-cross-cutting-concerns-and-tradeoff-patterns/` (11 files): `00-golden-rulebook-cheatsheet.md`; nine pattern notes `01`–`09` (latency, scalability/throughput, evaluation/QA, cost/token-efficiency, reliability/failure, security/safety, data-privacy/governance, observability/monitoring, versioning/change-mgmt); `10-interview-prep.md`.
- `05-interview-practice-case-studies-and-drills/README.md` (chapters table → 4 rows, est. time 8→11 hrs, overview/outcomes/how-it-fits/study-tips updated to include the new chapter + cheatsheet).
- `.agent_docs/handoff.md`.
**Summary:** Expanded section 05 with a new **cross-cutting concerns** chapter via the orchestrator + subagent workflow (agent orchestrated only; all authoring done by `general` subagents told to write in-run and webfetch-verify every Further Reading link). Per user decisions this run: (1) new chapter placed as `04`, one note per concern axis, using a **custom pattern structure** (TL;DR → ELI5 → Learning Objectives → Visual Overview → The Core Problem → Resolution Options table + per-option prose → Key Parameters → Worked Example → Implementation w/ anti-pattern → Pitfalls → Definitions → Quick Recall → Self-Check → Further Reading); (2) note 01 (latency) authored by the orchestrator as the validated reference, remaining 7 delegated in parallel; (3) added **versioning** and **observability** notes on request — versioning added as `09` and the pre-existing interview-prep renamed `09→10` (verified no external references first) to keep interview-prep last per AGENTS.md; (4) added the `00-golden-rulebook-cheatsheet.md` synthesizing all nine notes into a 60-second framework + 9-concern master table + golden rules + decision cues, linking to every sibling note. All 11 files verified stub-free.
**Outcome:** Chapter 4 fully authored, verified, and **committed** (`22b7069`); working tree clean. This chapter intentionally deviates from the AGENTS.md "topic notes 01–03 / interview-prep 04" convention (10 numbered files, interview-prep at `10`), like Ch2/Ch3 already do.

---

## Open Items / Next Steps

- [ ] `05-interview-practice-case-studies-and-drills/04-cross-cutting-concerns-and-tradeoff-patterns/golden_rulebook_examples/05-sample-question-enterprise-document-qa-platform.md` — written and verified but **untracked**; `git add` + commit it (the other four `golden_rulebook_examples/` files are already committed in `2376625`).

Carry-over (from prior sessions, not blocking):
- [ ] `05-.../03-rapid-fire-mock-question-bank-and-self-assessment/02-mock-question-bank-multi-agent-and-orchestration.md` and `.../03-mock-question-bank-security-eval-and-scaling.md` — grounded in official docs because their source sections (03 and 04) were still stubs; align for consistency once sections 03 and 04 are authored.

Suggested (not blocking) next authoring target: section `03-agentic-and-multi-agent-systems/`, starting with `01-agent-design-patterns/01-react-and-tool-calling-agent-fundamentals.md`, using the orchestrator + `author-chapter` (or `general`) subagent workflow. This section is LangGraph-heavy — subagents must verify LangGraph API details against current official docs (per AGENTS.md). Sections 03 and 04 are the only remaining unauthored curriculum content.

---

## Quick Reference

- **No build system, no tests** — this is a pure Markdown content repo.
- **Authoring source of truth:** `AGENTS.md` at repo root — read before populating any stub.
- **Template files (reference-only, never edit):** `templates/chapter-notes-template.md`, `templates/interview-prep-template.md`, `templates/section-index-template.md`, `templates/capstone-template.md`, `templates/authoring-guidelines.md`.
- **Stub marker:** unpopulated files contain exactly `<!-- stub: populate using templates/ -->` — intentional, not missing content.
- **Skills for authoring:** `author-chapter` (researches live sources, follows `chapter-notes-template.md` + Content Depth Rules 1–9, runs a quality gate); `generate-practice-exam` (grounds questions in authored content only).
- **Orchestrator pattern that works:** delegate one file per `general` subagent; pass the exact target path, topic framing, and mandatory section requirements; run subagents in parallel; explicitly instruct each subagent to WRITE in-run. For cheatsheet-grounded work, **paste the relevant cheatsheet content verbatim into the subagent prompt** and forbid facts outside it. If one subagent gets cancelled/stops, re-dispatch it (fresh) or resume with its `task_id`.
- **`golden_rulebook_examples/` folder (new this session):** worked system-design interview questions grounded strictly in `00-golden-rulebook-cheatsheet.md`. **House format** (per user's reference example): H1 → one-line "grounded in `../00-golden-rulebook-cheatsheet.md`" intro → `## The Question` (**Context** paragraph + `**Requirements:**` bullets with SLOs/numbers + `**Your task:**` numbered 5–6 facets) → `## Worked Answer` (clarify/SLO opener + `### Answer` with one `####` per task + ≥1 ```text ASCII diagram) → `### How the cheatsheet was used` (maps decisions to named cheatsheet elements) → `## Cheatsheet elements referenced` (bullet index). Rules for these files: cheatsheet is the ONLY source (no Further Reading / web links, no invented product/vendor names, cost-model unit prices labelled as assumptions), and **no video-based scenarios** (user constraint).
- **Custom file structures used in section 05:** Ch1 delivery-framework notes = standard chapter-notes template; Ch2 worked case studies = case-study walkthrough; Ch3 mock banks = Q-bank (~24 Qs + scorecard); Ch4 pattern notes = problem→resolution-options-table→scenario; Ch4 `00-golden-rulebook-cheatsheet.md` = dense skim-first reference; `golden_rulebook_examples/` = system-design worked questions (see above).
- **Chapter file numbering:** standard chapters use topic notes `01`–`03` + interview-prep at the next number; **exceptions:** `05-.../02-worked-case-studies-.../` uses `01`–`04` + `05-interview-prep`, and `05-.../04-cross-cutting-concerns-and-tradeoff-patterns/` uses `00-cheatsheet` + `01`–`09` + `10-interview-prep` (its `golden_rulebook_examples/` subfolder numbers `01`–`05`, not part of the locked curriculum numbering).
- **Filenames are locked** — never rename/renumber; chapters may forward-link to stubs not yet written.
- **Section structure:** Section → Chapter only (no module layer).
- **5 sections:** `01-classical-systems-refresher/` (**authored**), `02-llm-serving-and-rag-architecture/` (**authored**), `03-agentic-and-multi-agent-systems/` (stubs), `04-production-ai-systems-security-eval-scale/` (stubs), `05-interview-practice-case-studies-and-drills/` (**authored**, 4 chapters + golden_rulebook_examples).
- **No labs, no thought-leadership files** — only topic notes + interview-prep (+ the one cheatsheet + the worked-examples folder).
- **Markdown style:** H1 title, H2 major sections, H3 sub-sections; all code blocks carry a language tag (```text for ASCII diagrams); `---` separates major sections; Self-Check / drill answers use `<details><summary>Answer</summary>`.
- **External links rule (chapter notes):** official/authoritative documentation only, verified with `webfetch`, format `[Title](url) — *verified YYYY-MM-DD*`. No Medium/YouTube. (The `golden_rulebook_examples/` files deliberately carry NO external links — cheatsheet-only.)
- **Git:** commits so far — `f15e3ea` initial, `5d54117` scaffold, `04475b3` sections 01+02 content, `22b7069` section 05 content (four chapters incl. cross-cutting), `2376625` golden-rulebook worked examples `01`–`04` + handoff. **Pending:** `golden_rulebook_examples/05-sample-question-enterprise-document-qa-platform.md` is untracked. Note: `06-interview-questions-dump/question1.md` is a tracked empty placeholder outside the numbered curriculum.
- **Verify no stubs remain in a folder:** `grep -rl "stub: populate" <folder>/` should return nothing.
- **Verify a golden_rulebook_examples file's integrity:** confirm the four anchors (`## The Question`, `## Worked Answer`, `### How the cheatsheet was used`, `## Cheatsheet elements referenced`), no `<!-- stub`, and no `video`/`transcod` matches.
