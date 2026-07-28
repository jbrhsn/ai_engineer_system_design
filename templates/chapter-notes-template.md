# [Chapter Title]

**Section:** [Section] | **Est. time:** [X hrs] | **Interview relevance:** [High/Medium — one line on why]

---

## TL;DR

<!-- 2–4 sentences. What this topic is, why it matters, and the single most important thing to remember.
     End with: **The one thing to remember: ...** -->

---

## ELI5 — Explain It Like I'm 5

<!-- Mandatory. 3–6 sentences of plain English using a concrete everyday analogy.
     The analogy must map structurally onto the technical concept — not just vaguely compare.
     No jargon in this section. Prose only, no bullets.
     Non-compliant: "Think of X as a way to represent Y." (too vague)
     Compliant: name a familiar object, map its mechanism to the technical process,
     and explicitly correct the most common misconception. -->

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] [Action-verb outcome — specific and testable, e.g. "Design X to achieve Y under constraint Z"]
- [ ] [Outcome 2]
- [ ] [Outcome 3–5 total — use verbs: implement, diagnose, compare, explain, design]

---

## Visual Overview

<!-- Recommended. Include when the topic has a visualisable process, architecture, or decision flow.
     Aim for 2–4 diagrams. Place each under its own ### sub-header inside a plain fenced code block
     (no language tag). Use box-drawing characters for structure:
       Flows (left-to-right):  ──►   Vertical:  │  ├  └
       Corners:  ┌ ┐ ┘ └       Labels: ▲ ▼ ◄ ►
     Good diagram types:
       - Pipeline / data flow  (e.g. Query ──► Retrieve ──► Rerank ──► Generate)
       - Decision tree         (branch on a key condition, e.g. latency budget)
       - Architecture comparison (side-by-side option A vs option B)
       - Before / after        (anti-pattern state vs correct state)
     If the topic is purely conceptual with nothing to visualise, omit this section. -->

### [Diagram Title — e.g. "RAG Pipeline Flow"]

```
[left-to-right or top-to-bottom diagram here]
Input ──► Step 1 ──► Step 2 ──► Output
```

### [Diagram Title — e.g. "Architecture Decision Tree"]

```
Is condition X true?
├── Yes ──► Approach A
└── No  ──► Is condition Y true?
            ├── Yes ──► Approach B
            └── No  ──► Approach C
```

---

## Key Concepts

<!-- For EACH concept sub-section, answer all three questions:
     1. What is it? (1–2 sentence definition)
     2. How does it work mechanistically? (2–4 sentences on the process/behaviour that produces the result)
     3. Where does it appear in real systems? (specific API call, config field, architecture component —
        e.g. a LangGraph node, a FastAPI middleware, a vector DB index parameter, an eval metric)
     A sub-section that only answers question 1 is non-compliant. -->

### [Concept A]

### [Concept B]

### [Concept C]

### Key Parameters / Configuration Knobs

<!-- Required for any chapter covering a configurable component.
     "Decision rule" must be a concrete actionable rule, not a restatement of the parameter.
     If no configurable parameters exist for this topic, write: "No configurable parameters for this topic." -->

| Parameter | What it controls | Decision rule |
|---|---|---|
| [param] | [what it does] | [when to set it to X vs Y] |

### Worked Example: Requirement → Decision

<!-- Mandatory. Walk through one complete, realistic decomposition.
     Given: [plain-English scenario — not "Example: configure X"]
     Step 1 — Identify the goal: [what outcome is needed]
     Step 2 — Define inputs: [what data/config/context enters]
     Step 3 — Define outputs: [what the downstream step expects]
     Step 4 — Apply constraints: [constraints relevant to this domain and topic —
       e.g. latency budget, hallucination tolerance, data residency, cost per query]
     Step 5 — Select the approach: [specific pattern/tool + one-sentence rationale vs alternatives]
     If no selection decision exists, substitute a realistic failure diagnosis walkthrough. -->

---

## Implementation

<!-- At least 2 code or config snippets from different angles.
     Every snippet starts with a comment naming the business/operational problem it solves.
     At least one snippet must be an anti-pattern, labeled # Anti-pattern: or # Wrong:,
     immediately followed by the corrected version with an explanation of what breaks. -->

```[language]
# Scenario: [the real-world problem this solves]

```

```[language]
# Anti-pattern: [describe the wrong approach and why it fails]

# Correct approach:

```

---

## Common Pitfalls & Misconceptions

<!-- Each bullet: (1) bolded label, (2) why beginners make this mistake, (3) correct mental model.
     Bare bullets with no explanation are non-compliant. -->

- **[Pitfall label]** — [Why the wrong intuition forms]. [Correct mental model].
- **[Pitfall label]** — [Why the wrong intuition forms]. [Correct mental model].
- **[Pitfall label]** — [Why the wrong intuition forms]. [Correct mental model].

---

## Key Definitions

| Term | Definition |
|---|---|
| [Term] | [Precise, scoped definition — not a dictionary entry] |

---

## Summary / Quick Recall

- [Key takeaway 1 — one line, scannable]
- [Key takeaway 2]
- [3–7 takeaways total — designed for a 60-second pre-interview scan]

---

## Self-Check Questions

<!-- 5 questions. Cognitive level distribution:
     Q1: recall (definition or fact)
     Q2–Q3: application (apply concept to a new scenario)
     Q4–Q5: analysis or trade-off (compare options, select best under constraints)
     At least 1 must be multi-select ("Which TWO...").
     Every answer: explain why correct AND why main distractor(s) are wrong.
     One-word answers are non-compliant. -->

1. [Recall question]

   <details><summary>Answer</summary>

   [Why this is the correct answer. Why the most tempting wrong answer fails.]

   </details>

2. [Application question]

   <details><summary>Answer</summary>

   [Answer with rationale.]

   </details>

3. **Which TWO** [multi-select question]
   - A.
   - B.
   - C.
   - D.
   - E.

   <details><summary>Answer</summary>

   [Why both correct answers qualify. Why the most tempting wrong answer fails.]

   </details>

4. [Analysis question]

   <details><summary>Answer</summary>

   [Answer with rationale.]

   </details>

5. [Trade-off question]

   <details><summary>Answer</summary>

   [Answer with rationale.]

   </details>

---

## Further Reading

<!-- Official documentation only. No third-party blogs, Medium, or YouTube.
     Format: [Title](url) — *verified YYYY-MM-DD* — [one-line description]
     Verify every URL with webfetch before writing. -->

- [Title](url) — *verified YYYY-MM-DD* — [description]
