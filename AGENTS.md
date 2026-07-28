# AGENTS.md

This file provides guidance to agents working in this repository.

---

## What This Repo Is

A Markdown-only learning repository for **AI System Design for AI Engineers** (agentic AI, RAG, and multi-agent production systems, with a security-first and evaluation-first lens). No build system, no tests. All content is `.md` files. The agent's job is always one of: populate a stub, write an index, or update structured Markdown.

**Goal:** Prepare for AI/ML system design interviews at the "Applied AI Engineer" level — designing and defending production-grade agentic AI, RAG, and multi-agent architectures, including security, evaluation, and scaling trade-offs. Builds on 4 years of hands-on experience with LangGraph multi-agent systems, RAG pipelines, and FastAPI/PostgreSQL production deployments.

**Source of truth:** No single certification exists for this topic. Authoritative sources are: the OWASP GenAI Security Project (LLM Top 10:2025), Anthropic's Applied AI engineering blog (context engineering, multi-agent research systems), LangChain/LangGraph official documentation, Ragas evaluation documentation, and Hello Interview's ML System Design delivery framework (adapted for agentic/RAG systems). Always prefer the current official docs for fast-moving framework details (LangGraph API, evaluation tooling).

---

## Critical Conventions

### Stub files are intentionally empty
All `.md` files (except those in `templates/`) were created as single-line stubs. An empty file is NOT missing content — populate only what the user explicitly requests.

### Never edit template originals
Templates in `templates/` are reference-only. Copy content from a template into the target file; never modify the template itself.

### Standard Chapter Template
The authoritative template is `templates/chapter-notes-template.md`. Every notes file must follow this structure in this order:

1. **TL;DR** — 2–4 sentences ending with a bolded "one thing to remember"
2. **ELI5** — Mandatory plain-language analogy section, no jargon
3. **Learning Objectives** — Specific, testable, action-verb outcomes
4. **Visual Overview** — Recommended when the topic has a visualisable process; 2–4 ASCII diagrams in plain fenced blocks under `###` sub-headers; placed after Learning Objectives, before Key Concepts
5. **Key Concepts** — Each sub-section: definition + mechanism + real-world manifestation (specific API, config, architecture pattern)
6. **Implementation** — ≥2 snippets (different angles) including one anti-pattern
7. **Common Pitfalls** — Each: bolded label + why beginners make it + correct mental model
8. **Key Definitions** — Precise, scoped definitions only
9. **Summary / Quick Recall** — 3–7 scannable takeaways
10. **Self-Check Questions** — 5 questions spanning recall → application → analysis; ≥1 multi-select
11. **Further Reading** — Official docs only, all links verified

### Template → destination mapping

| Template | Destination |
|---|---|
| `chapter-notes-template.md` | `[chapter]/01-*.md` – `03-*.md` (or `04-*.md` for the one dense chapter) |
| `interview-prep-template.md` | `[chapter]/NN-interview-prep.md` (NN = next number after topic notes) |
| `section-index-template.md` | `[section]/README.md` |
| `capstone-template.md` | `capstone/project-brief.md` |

### This repo has no module layer
Structure is Section → Chapter (two levels), not Section → Module → Chapter. There is no `module-index-template.md` in this repo.

### Lab numbering
Not applicable — this repo has no lab files.

### Index placement
- Section-level index (`README.md`) exists at scaffold time — do not recreate.

### External links
Official documentation only. No third-party blogs, Medium, or YouTube.
Format: `[Title](url) — *verified YYYY-MM-DD*`
Verify every URL with `webfetch` before writing.

---

## Content Depth Rules

These rules are topic-agnostic and govern authoring quality regardless of subject matter.

### Rule 1 — ELI5 is mandatory and must use a structural analogy
Every notes file must open with an ELI5 section after TL;DR:
- Plain English, zero jargon
- A concrete everyday analogy that maps structurally onto the technical concept
- Specific enough that a complete beginner could build the correct mental model from it
- 3–6 sentences, prose only, no bullet lists
- Non-compliant: "Think of X as a way to represent Y." (too vague — no structure)
- Compliant: names a familiar object, maps its mechanism to the technical process, explicitly corrects the most common misconception

### Rule 2 — Every concept sub-section must explain the mechanism
Each Key Concepts sub-header must answer three questions:
1. What is it? (1–2 sentence definition)
2. How does it work under the hood? (2–4 sentences on the mechanism — the process or system behaviour that produces the result)
3. Where does it appear in real systems? (specific API call, config field, architecture component, or observable output — e.g. a LangGraph node, a FastAPI middleware, a vector DB index parameter)
Answering only question 1 is non-compliant.

### Rule 3 — Key Parameters sub-section is required for configurable topics
Any chapter covering a component with tunable settings must include a **Key Parameters / Configuration Knobs** table: `Parameter | What it controls | Decision rule`. The Decision rule must be a concrete actionable rule, not a restatement of the parameter's purpose. If no configurable parameters exist, write "No configurable parameters for this topic." and continue.

### Rule 4 — Worked Example is required in every chapter
Every notes file must include a **Worked Example: Requirement → Decision** sub-section following this structure:
- Given: a realistic scenario in plain English
- Step 1 — Identify the goal
- Step 2 — Define inputs
- Step 3 — Define outputs
- Step 4 — Apply constraints (constraints relevant to this domain and topic)
- Step 5 — Select the approach with a one-sentence rationale vs alternatives
If no selection decision exists, substitute a realistic failure diagnosis walkthrough.

### Rule 5 — Snippets must be scenario-first, not topic-first
Every code or config snippet must begin with a comment naming the real-world problem being solved.
Non-compliant: a comment that only names the feature or command being demonstrated.
Compliant: a comment that states the concrete operational goal the snippet achieves and the constraint that makes it the right choice.
At least one snippet per file must be an anti-pattern (`# Anti-pattern:`) immediately followed by the corrected version with an explanation of what breaks.

### Rule 6 — Pitfalls must have three parts
Each pitfall bullet: (1) **bolded label**, (2) one sentence on why beginners make this mistake, (3) one sentence on the correct mental model. Bare bullets are non-compliant.

### Rule 7 — Answer rationales must cover all options
Every Self-Check answer must explain why the correct answer is right AND why the main distractor(s) are wrong. One-word rationales are non-compliant. For multi-select, explain why both correct answers qualify AND why the most tempting wrong answer fails.

### Rule 8 — Self-Check questions must span cognitive levels
Required distribution: Q1 recall, Q2–Q3 application, Q4–Q5 analysis/trade-off. Five recall questions is non-compliant even if one is multi-select.

### Rule 9 — Visual Overview is recommended for visualisable topics
When a topic involves a pipeline, decision path, architecture, or before/after contrast, include a `## Visual Overview` section placed **after `## Learning Objectives` and before `## Key Concepts`**. Format: each diagram under its own `### [Diagram Title]` sub-header inside a plain fenced code block (no language tag). Use `──►` for flow arrows and `│ ├ └ ─ ┌ ┐` for tree/box structure. Aim for 2–4 diagrams. Omit this section only for purely conceptual topics where no process or structure exists to diagram.

---

## File Naming Rules

- Notes: `01-kebab-case.md`, `02-kebab-case.md`, `03-kebab-case.md` (zero-padded); the one dense chapter (`05-interview-practice.../02-worked-case-studies-.../`) uses `01`–`04`
- Interview prep: `04-interview-prep.md` for standard chapters, `05-interview-prep.md` for the dense chapter — always the number immediately after the last topic note
- Section folders: `01-section-name/` … `05-section-name/`
- Chapter folders: `01-chapter-name/` … `03-chapter-name/` (numbered within their parent section)

---

## Markdown Style
- H1 for file title, H2 for major sections, H3 for sub-sections
- All code blocks carry a language tag
- Horizontal rules (`---`) separate every major section
- Self-Check answers use `<details><summary>Answer</summary>` collapsible blocks
- HTML comments (`<!-- -->`) carry authoring guidance in templates — preserve them

## What Not to Do
- Do not populate stubs without explicit user instruction
- Do not invent framework API details (e.g. LangGraph node signatures) from memory alone — verify against current official docs, since these frameworks evolve quickly
- Do not add content not traceable to an authoritative source
- Do not renumber or rename files or folders after scaffold — filenames are locked from Phase 5. Authored chapters may forward-link to not-yet-written stubs; those links resolve when the target is populated. Renaming to "fix" a link breaks every other reference to that file.
- Do not link to third-party blogs, Medium, or YouTube
- Do not skip or merge sections without user approval
