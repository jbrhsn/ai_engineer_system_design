# Authoring Guidelines & Quality Rubric

## Voice & Tone

- Write for **learning**, not documentation. Explain *why* things work the way they do.
- Active voice. Concrete examples over abstract definitions.
- Target voice: "knowledgeable colleague at a whiteboard" — not a textbook, not a marketing page.
- Assume the reader has 4 years of hands-on agentic AI/RAG experience but is formalizing that knowledge for system design interviews.

## Depth Calibration

- **Key Concepts (incl. ELI5, Worked Example):** ~75% of authoring effort. This is where teaching happens.
- **Implementation snippets:** ~15%. Must be realistic and runnable. At least one anti-pattern.
- **Self-Check Questions:** ~10% but non-negotiable. 5 questions, spanning recall → application → analysis.
- Do not invert these proportions. A chapter with 10 code examples and 2 sentences of explanation teaches nothing.

## ELI5 Requirements

Every chapter opens with an ELI5 section. Rules:
- Plain English, zero jargon.
- A concrete everyday analogy that maps structurally onto the concept — not just a vague comparison.
- 3–6 sentences, prose only.
- Must address the most common misconception about the topic.

## Worked Examples

Every chapter must have at least one complete Worked Example following the Requirement → Decision format:
- Given a realistic scenario (not a toy example) — prefer scenarios grounded in enterprise/production AI systems (e.g. document extraction pipelines, executive dashboards, RBAC-gated data access) similar to real deployment contexts
- Step through goal → inputs → outputs → constraints → approach + rationale
- If no selection decision exists, use a failure diagnosis walkthrough instead

## Visual Overview

Include a `## Visual Overview` section in a topic note when the subject has a pipeline, architecture, decision path, or before/after contrast that is easier to grasp visually than in prose. It is **recommended, not required** — omit for purely conceptual topics with nothing to diagram.

**Placement:** After `## Learning Objectives`, before `## Key Concepts`.

**Format rules:**
- Each diagram under its own `### [Diagram Title]` sub-header
- Diagrams in plain fenced code blocks — no language tag
- Use box-drawing characters: `──►` for flow arrows; `│ ├ └ ─ ┌ ┐ ┘` for tree/box structure; `▲ ▼ ◄ ►` for labels
- Aim for 2–4 diagrams per note; a single well-drawn diagram is better than four cluttered ones

**Diagram types that work well for this topic:**
- **Pipeline flow** — e.g. Query ──► Retrieve ──► Rerank ──► Generate ──► Guardrail check
- **Decision tree** — e.g. "single agent vs multi-agent" or "RAG vs fine-tuning vs long-context"
- **Side-by-side comparison** — e.g. supervisor pattern vs plan-and-execute pattern
- **Before / after** — e.g. naive prompt-stuffing vs context-engineered agent

## Self-Check Questions

- 5 questions per chapter. Distribution: Q1 recall, Q2–Q3 application, Q4–Q5 analysis/trade-off.
- At least 1 must be multi-select ("Which TWO...").
- Every answer must explain why the correct answer is right AND why the main distractor(s) are wrong.
- Use `<details><summary>Answer</summary>` for all answers.
- One-word answers are non-compliant.

## Source Hygiene

- Cite source URL and retrieval date for every specific doc, API reference, or changelog entry.
- Flag fast-evolving features with: `> ⚠️ Fast-evolving: verify against current official docs before relying on this.` (LangGraph APIs, OWASP LLM Top 10 revisions, and Ragas metrics all move quickly.)
- Official documentation only — no third-party blogs, Medium, or YouTube.

## Blueprint Drift Warning

Agentic AI frameworks (LangGraph/LangChain), security guidance (OWASP GenAI), and evaluation tooling (Ragas) all evolve rapidly. If you are authoring more than 6 months after this repo was created, verify current official documentation before writing. Do not assume the original research summary is still current.

## Quality Checklist

Run before marking a chapter complete:

- [ ] TL;DR ends with a bolded "one thing to remember"
- [ ] ELI5 uses a concrete structural analogy, no jargon, addresses a misconception
- [ ] Every Key Concepts sub-section answers: What? How does it work? Where does it appear in real systems?
- [ ] Key Parameters table exists (or explicit "no configurable parameters" note)
- [ ] Worked Example follows the Requirement → Decision 5-step format
- [ ] At least 2 implementation snippets from different angles
- [ ] At least 1 anti-pattern snippet with corrected version
- [ ] All snippets start with a `# Scenario:` or `# Anti-pattern:` comment
- [ ] Visual Overview present (if topic has a visualisable process) — 2–4 diagrams, each under a `###` sub-header in a plain fenced block
- [ ] Pitfalls have all 3 parts: label + why beginners make it + correct mental model
- [ ] 5 Self-Check questions spanning 3 cognitive levels
- [ ] At least 1 multi-select question
- [ ] All answers explain why correct AND why distractors fail
- [ ] Further Reading uses only official docs, all links verified with webfetch
- [ ] No filler — every sentence earns its space
