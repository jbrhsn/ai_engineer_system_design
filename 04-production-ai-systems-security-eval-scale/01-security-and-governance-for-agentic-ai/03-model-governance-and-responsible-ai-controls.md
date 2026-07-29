# Model Governance and Responsible-AI Controls

**Section:** 04 Production AI Systems → Security & Governance for Agentic AI | **Est. time:** 3 hrs | **Interview relevance:** Medium-High — governance is what separates candidates who have *shipped* regulated AI from those who have only prototyped; if you can name the frameworks (NIST AI RMF, EU AI Act tiers, ISO/IEC 42001) and the concrete artifacts (model cards, use registry, approval gates, audit logs) that wrap your technical controls, you signal production maturity.

---

## TL;DR

Model governance is the **organizational wrapper** around the technical security controls from chapters 01–02: the policies, artifacts, sign-offs, and records that make an AI system accountable, auditable, and defensible to a regulator or an auditor. It is anchored by three reference frameworks — the **NIST AI Risk Management Framework** (four functions: Govern, Map, Measure, Manage), the **EU AI Act** (risk-tiered obligations: unacceptable/prohibited → high-risk → limited/transparency → minimal), and **ISO/IEC 42001** (a certifiable AI Management System). The work is concrete, not abstract: classify each use case by risk, register it in an inventory, require a model/system card, pin and version every model and prompt, gate deployment behind an approval workflow, log every inference for traceability, and attach human oversight where risk demands it. **The one thing to remember: governance is not bureaucracy that slows shipping and it is not just security — it is the accountability layer (who approved what, on which model version, with what evidence) that lets you ship *and defend* an AI system in a regulated environment.**

---

## ELI5 — Explain It Like I'm 5

Think about how a commercial airplane factory operates. The machines that bend metal and the robots that rivet wings are the actual *technology* — but nobody is allowed to build a plane just because they have the machines. There is a whole layer around the machines: inspectors who sign off before a part ships, a logbook recording exactly which batch of metal went into which plane, checklists that must be completed before a design is approved, and a rule that a human engineer must review certain safety-critical steps. If a plane later has a problem, investigators can pull the logbook and trace the exact part, the exact approval, and the exact person who signed it. Governance for AI is that inspection-and-logbook layer, not the machines: it is the record-keeping, the sign-offs, and the human checks that make the whole operation *accountable*. The common misconception is that this layer is pointless paperwork that just slows the factory down, or that it is "the same thing as locking the factory doors" (security) — but locks stop intruders, whereas the logbook and sign-offs prove *your own* operation was run responsibly and let you answer "who approved this, on what evidence?" when someone asks.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain what model governance is and why it is the organizational wrapper around (not a replacement for) the threat model and defenses from chapters 01–02.
- [ ] Describe the NIST AI RMF's four functions (Govern, Map, Measure, Manage) and place a project's activities into them.
- [ ] Classify an AI use case against EU AI Act risk tiers and select the governance controls the tier implies.
- [ ] Design the core governance artifacts — model/system card, AI use registry entry, model/prompt version pin, approval gate, and structured audit-log event — for a production agent.
- [ ] Decide when human oversight, red-teaming, and a bias/fairness or PII assessment are *required* versus optional for a given use case.

---

## Visual Overview

### NIST AI RMF — The Four Functions

```
                    ┌───────────────────────────────────┐
                    │            GOVERN                  │
                    │  (culture, policy, accountability, │
                    │   roles — cuts across all others)  │
                    └───────────────────────────────────┘
                       │            │            │
                       ▼            ▼            ▼
              ┌──────────┐   ┌───────────┐   ┌──────────┐
              │   MAP    │──►│  MEASURE  │──►│  MANAGE  │
              │ context, │   │ analyze,  │   │ prioritize│
              │ risks,   │   │ assess,   │   │ respond, │
              │ use-case │   │ track     │   │ monitor, │
              │ framing  │   │ metrics   │   │ mitigate │
              └──────────┘   └───────────┘   └──────────┘
                       ▲                          │
                       └──────── feedback ────────┘
   GOVERN is continuous and enables the other three functions.
```

### Governance Lifecycle — Use-Case Intake to Audit

```
 use-case
 intake ──► RISK ──► required ──► APPROVAL ──► DEPLOY ──► MONITOR ──► AUDIT
 (registry  CLASS-   controls    GATE          (pinned    (metrics,   (traceable
  entry)    IFICATION selected   (sign-off      version)   drift,      evidence
            (tier)    (card,     recorded)                 incidents)   on demand)
                      oversight,      │                        │            │
                      eval, PII)      └──── reject / send back ─┘            │
                                                                            │
            └───────────────── re-review on cadence / on change ◄───────────┘
```

### EU AI Act Risk Tiers (More Risk = More Obligation)

```
   PROHIBITED        ┌─────────────────────────────────────┐
   (unacceptable) ── │ social scoring, manipulative AI —    │  banned
                     │ not allowed to be placed on market   │
                     └─────────────────────────────────────┘
   HIGH-RISK      ── │ risk mgmt system, data governance,   │  heavy
                     │ technical docs, record-keeping,      │  obligations
                     │ human oversight, accuracy/robustness │
   LIMITED        ── │ transparency: users must be told     │  light
   (transparency)    │ they are interacting with AI         │  obligations
   MINIMAL        ── │ no specific obligations              │  (voluntary)
```

### Where This Chapter Sits vs Chapters 01–02

```
   ch01 THREAT MODEL          ch02 DEFENSES              ch03 GOVERNANCE (this)
   ─────────────────          ────────────              ──────────────────────
   what can go wrong ──────►  technical controls  ──►   organizational wrapper
   (prompt injection,         (input filtering,          (who approved it, on
    tool abuse, data           sandboxing, least-         which version, logged
    exfiltration)              privilege tools)           where, reviewed by whom)
        │                          │                            │
        └── attackers ─────────────┘                            └── auditors /
                                                                    regulators
```

---

## Key Concepts

### Model Governance (the organizational wrapper)

**What it is.** Model governance is the set of policies, roles, artifacts, and repeatable processes an organization uses to decide *whether* an AI system may be built and deployed, *under what controls*, and *how it is held accountable* over its lifecycle — distinct from the technical defenses that stop attacks.

**How it works mechanistically.** Governance operates as a control loop wrapped around the engineering work: an intake step classifies a proposed use case by risk; that classification *selects* a required set of controls (documentation, human oversight, evaluation, PII handling); an approval authority signs off before deployment; production emits audit records; and a monitoring/re-review cadence feeds findings back into the next decision. The mechanism that makes it "governance" rather than "good engineering" is the **separation of who decides from who builds** plus a durable **record of the decision and its evidence**, so accountability survives staff turnover and can be reconstructed for an auditor.

**Where it appears in real systems.** Concretely it shows up as a model registry with an approval status field (e.g. MLflow Model Registry stages/aliases, or a governed model catalog), a use-case intake form/ticket, an owned RACI for AI systems, and a policy gate in the deployment pipeline that refuses to promote an unapproved model version. It wraps — it does not replace — the input filtering, tool sandboxing, and least-privilege controls from chapter 02.

### NIST AI Risk Management Framework (Govern / Map / Measure / Manage)

**What it is.** The NIST AI RMF (AI 100-1, released Jan 2023, voluntary) is a framework for managing risks to individuals, organizations, and society across the AI lifecycle, organized into four **functions**: **Govern, Map, Measure, Manage**.

**How it works mechanistically.** **Govern** is a cross-cutting, continuous function — it establishes the culture, policies, accountability structures, and roles that *enable* the other three, and it is not a one-time step. **Map** establishes context: what the system is for, who is affected, and what risks the deployment context creates. **Measure** analyzes, assesses, and tracks those risks with quantitative and qualitative methods (this is where your evaluation work — see this section's eval chapter — plugs in). **Manage** prioritizes and acts on the measured risks: allocate resources, respond, monitor, and mitigate. Findings loop back to Map/Measure as the system and its context change. NIST publishes a companion **Playbook** (with Govern/Map/Measure/Manage sub-categories) and a **Generative AI Profile** (AI 600-1, 2024) that tailors the functions to GenAI-specific risks.

**Where it appears in real systems.** Teams use the four functions as the top-level taxonomy for a governance program: a "Govern" policy doc + RACI, a "Map" use-case registry entry and risk classification, "Measure" as the eval/red-team results attached to a model card, and "Manage" as the monitoring dashboards and incident-response runbook. Auditors and enterprise procurement increasingly ask "how do you map to the NIST AI RMF?" as a maturity signal.

### EU AI Act Risk Tiers (a brief, accurate mention)

**What it is.** The EU AI Act is horizontal EU regulation that classifies AI systems by **risk** and attaches obligations accordingly: **unacceptable-risk** systems are *prohibited* (e.g. social scoring, certain manipulative techniques); **high-risk** systems (e.g. AI in recruitment, credit, essential services, biometric ID under Annex III) carry heavy provider obligations; **limited-risk** systems carry lighter **transparency** obligations (users must be told they are interacting with AI — relevant to most chatbots); **minimal-risk** systems are unregulated.

**How it works mechanistically.** The tier is determined by the *use case and context*, not the model's raw capability. High-risk providers must, among other things, establish a **risk-management system** across the lifecycle, perform **data governance** on training/validation/test data, produce **technical documentation**, design for **record-keeping** (logging), enable **human oversight**, and meet **accuracy, robustness, and cybersecurity** targets. General-purpose AI (GPAI) model providers have their own documentation obligations, with additional model-evaluation/adversarial-testing/incident-reporting duties for models presenting *systemic risk*. Obligations phase in on staggered deadlines after entry into force.

**Where it appears in real systems.** In practice a team maps each product to a tier during intake; a customer-facing assistant that is purely informational often lands in *limited risk* (add an "I am an AI" disclosure), while the same technology used to screen job applicants lands in *high-risk* and pulls in the full documentation + human-oversight + logging stack. Getting the tier right *is* the first governance decision.

### ISO/IEC 42001 — AI Management System (AIMS)

**What it is.** ISO/IEC 42001:2023 is the world's first certifiable **AI management system** standard: it specifies requirements for establishing, implementing, maintaining, and continually improving an AIMS using the familiar **Plan-Do-Check-Act** management-system pattern (the same shape as ISO 27001 for infosec).

**How it works mechanistically.** Rather than dictating technical details of any single model, 42001 requires the *organization* to put policies, objectives, roles, risk-assessment, and continual-improvement processes in place for responsible AI — then those can be **audited and certified** by an external body. It pairs naturally with ISO 27001 (information security) and ISO/IEC 42005 (AI impact assessment). Certification is the artifact enterprises use to *demonstrate* responsible-AI governance to customers and regulators without exposing internals.

**Where it appears in real systems.** For an applied AI engineer this surfaces as: an enterprise buyer's security questionnaire asking "are you ISO 42001 certified / working toward it?", and internally as the management-system scaffolding (AI policy, risk register, internal audits) that your model cards and registry feed into.

### Core Governance Artifacts (cards, provenance, registry, versioning, audit)

**What they are.** The concrete, reviewable outputs governance produces: **model/system cards** (structured documentation of a model's or system's purpose, data, evaluation, limitations, and intended use), **data provenance & lineage** (where training/RAG data came from and how it flowed), an **AI use registry/inventory** (a catalog of every AI use case with owner, risk tier, and status), **model/prompt versioning + approval workflow** (pinned, immutable versions promoted only via recorded sign-off), and **audit logging & traceability** (a durable, queryable record of inputs, model version, decisions, and actors per inference).

**How they work mechanistically.** A **model card** attaches evaluation and limitation evidence to a specific model version so a reviewer can judge fitness; a **system card** does the same at the assembled-system level (model + tools + guardrails). **Provenance/lineage** lets you answer "what data trained/grounded this output?" — essential for a data-deletion request or a contamination incident. The **registry** makes the portfolio *knowable* (you cannot govern what you cannot see — "shadow AI" is the governance failure). **Versioning + approval** makes deployments *reproducible and attributable*: every promotion pins a model hash and prompt version and records who approved it. **Audit logging** makes runtime behavior *reconstructable*: which prompt, which model version, which tools fired, what the human-oversight decision was.

**Where they appear in real systems.** Model cards ship as Markdown/YAML alongside the model or in the registry (Hugging Face model cards, Google Model Cards); lineage lives in tools like MLflow, OpenLineage, or a data catalog; the registry is a governed catalog or even a maintained spreadsheet at small scale; approval gates are CI/CD promotion checks; audit logs are structured JSON events shipped to an append-only store with defined retention.

### Human Oversight, Red-Teaming, and Incident Response for AI

**What they are.** **Human oversight** = a defined human role empowered to review, override, or halt AI decisions (ranging from human-in-the-loop approval to human-on-the-loop monitoring). **Red-teaming** = adversarial testing that deliberately tries to make the system misbehave (jailbreaks, prompt injection, harmful-content elicitation) before and after deployment. **AI incident response** = the runbook for when the system does misbehave in production (harmful output, data leak, prompt-injection compromise).

**How they work mechanistically.** Oversight is *designed in*, not bolted on: for a high-risk decision you route the model's proposed action to a human queue with enough context to overrule it, and you log the human's decision. Red-teaming feeds the **Measure** function — findings become required fixes or documented residual risk on the model card, and it is an explicit EU AI Act obligation for GPAI systemic-risk models. AI incident response extends your existing security IR with AI-specific triggers (a jailbreak in the wild, a detected data-exfiltration via tool, a fairness regression) and connects to the EU AI Act's serious-incident reporting duties.

**Where they appear in real systems.** Oversight appears as a LangGraph interrupt / human-approval node before a high-impact tool call (see the human-in-the-loop control from chapter 02); red-teaming appears as a scheduled adversarial-eval suite and pre-launch exercise; incident response appears as an on-call runbook plus the audit-log queries you rely on to scope the blast radius.

### Responsible-AI Dimensions (bias/fairness, transparency, PII & data minimization, content safety)

**What they are.** The substantive quality dimensions governance is meant to protect: **bias/fairness** (does the system produce systematically worse outcomes for protected groups?), **transparency** (are users and reviewers told what the system is and how it decides?), **PII handling & data minimization** (collect/retain only the personal data you need, and protect it), and a **content-safety policy** (what the system must refuse to produce).

**How they work mechanistically.** Bias/fairness is assessed with disaggregated evaluation — measure quality *per subgroup*, not just in aggregate — and mitigated at data, prompt, or post-processing level. Data minimization is enforced by **PII redaction/tokenization at ingestion** and short retention windows, so a breach or subpoena exposes less. Transparency is enforced by AI-disclosure UX (the EU AI Act limited-risk requirement) and by publishing model-card limitations. Content safety is enforced by a policy that maps to guardrail rules (input/output classifiers from chapter 02) *and* a documented refusal taxonomy so behavior is consistent and defensible.

**Where they appear in real systems.** Fairness surfaces as a disaggregated metrics table in the model card; PII handling surfaces as a redaction middleware in the ingestion pipeline plus a documented retention policy; transparency surfaces as an on-screen "AI-generated" label; content safety surfaces as the guardrail config plus the written policy the guardrail implements.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Risk-tier threshold for mandatory approval | Which use cases must pass a formal approval gate before deploy | Require sign-off for anything at *limited-risk with PII* and above; auto-approve only internal, non-PII, low-impact tools. Map to EU AI Act tier: any high-risk use case is *always* gated. |
| Approval authority (single vs review board) | Who can sign off a deployment | Single owner for low-risk internal tools; a cross-functional review board (eng + legal/privacy + risk) for high-risk or externally-facing PII systems. Never let the builder be the sole approver of their own high-risk system. |
| Audit-log retention period | How long inference/decision records are kept | Set to the longest of: regulatory requirement (often 6–12 mo+ for regulated sectors), incident-investigation need, and contractual SLA. For high-risk EU AI Act systems, retain to satisfy record-keeping obligations; balance against data-minimization for PII in the logs. |
| Human-oversight mode (in-the-loop vs on-the-loop vs none) | Whether/how a human gates or monitors decisions | Human-*in*-the-loop (blocking approval) for irreversible or high-harm actions (payments, medical, hiring); human-*on*-the-loop (monitoring + override) for high-volume reversible actions; none only for demonstrably low-risk outputs. |
| Model-card required fields | The minimum documentation before a version can be promoted | Require intended use, training/grounding data provenance, eval results (incl. disaggregated fairness), known limitations, and safety evaluation for any external or high-risk model; a lighter subset for internal low-risk. |
| Re-review / re-certification cadence | How often an approved system is re-assessed | Re-review on any material change (new model version, new tool, expanded data) AND on a fixed cadence (e.g. every 6–12 months) for high-risk systems; drift or an incident triggers immediate re-review. |

### Worked Example: Requirement → Decision

**Given:** Your team wants to ship a **customer-facing support agent for a health-insurance company**. It answers member questions, looks up claims and coverage (which contain PII/PHI), and can draft appeal letters. Legal is nervous, the company operates in the EU and the US, and leadership wants it live in a quarter. You must define the governance controls before launch.

- **Step 1 — Identify the goal:** Produce a defensible governance package — risk classification plus the specific required controls — that lets the agent ship *and* survive an audit, without treating governance as a launch blocker.
- **Step 2 — Define inputs:** The use-case description; the data it touches (PHI/PII claims data); the actions it can take (read claims, draft — not submit — appeals); jurisdictions (EU + US); the existing chapter-02 defenses (guardrails, tool sandboxing, least-privilege claim lookups scoped to the authenticated member).
- **Step 3 — Define outputs:** A registry entry (owner, tier, status); a completed **system card**; an approval sign-off record from the review board; a versioned/pinned model + prompt; structured audit logging; a monitoring + incident-response plan; a defined human-oversight point.
- **Step 4 — Apply constraints:** PHI/PII → data minimization + redaction in logs and short retention; EU exposure → this is *not* prohibited and likely *not* Annex-III high-risk (it is informational support, and it does *not* decide claim eligibility) but it *is* at least **limited-risk** requiring an AI-disclosure to the user; drafting-not-submitting keeps the highest-harm action (denying/approving a claim) out of scope; the quarter deadline means scope the controls to the tier, don't gold-plate.
- **Step 5 — Select the approach:** Classify as **limited-risk with sensitive data**, register it, and require: (a) an on-screen **AI-disclosure**; (b) a **system card** documenting data provenance, eval results including a **disaggregated fairness check** (does answer quality degrade for any member demographic?), and limitations; (c) **PII/PHI redaction** before anything is written to audit logs, with a defined retention window; (d) **human-in-the-loop** only on the appeal-letter draft (a human agent must review before it is sent — the one action with real member impact), while pure Q&A runs human-on-the-loop; (e) a **pinned model + prompt version behind a review-board approval gate**; (f) **structured audit logging** of member-scoped lookups. *Rationale vs alternatives:* full Annex-III high-risk treatment would be over-engineering (the system doesn't make the eligibility decision), while treating it as minimal-risk would be non-compliant (it handles PHI and talks to consumers, so disclosure + logging + oversight-on-the-appeal are the correct, proportionate controls).

---

## Implementation

```yaml
# Scenario: Before a support-agent model version can be promoted to production in a
# regulated (PHI) setting, governance requires a completed, versioned system card so a
# reviewer/auditor can judge fitness and later reconstruct WHAT was approved. This card
# is the artifact the approval gate checks for. Fields mirror NIST AI RMF Map/Measure
# outputs and EU AI Act technical-documentation expectations.
system_card:
  system_name: member-support-agent
  version: "2.4.0"                      # pinned; matches the promoted release tag
  owner: ai-platform-team
  risk_tier: limited_risk_with_phi      # from intake classification
  intended_use: "Answer member coverage/claims questions; draft (not submit) appeals."
  out_of_scope: ["claim eligibility decisions", "auto-submitting appeals"]
  components:
    model: {name: "gpt-4o", version_pin: "gpt-4o-2024-08-06"}   # exact model pin
    prompt_version: "sys-prompt@v11"
    tools: ["get_member_claims (member-scoped, read-only)"]
    guardrails: ["input-injection-filter", "output-pii-leak-check"]  # from ch02
  data_provenance:
    grounding_source: "internal coverage KB, snapshot 2026-06-01"
    training_data: "vendor base model; no customer PHI used in fine-tuning"
  evaluation:
    overall_answer_accuracy: 0.91
    disaggregated_fairness:               # per-subgroup, not just aggregate
      by_plan_type: {gold: 0.92, silver: 0.90, medicaid: 0.89}   # gap monitored
    red_team_summary: "jailbreak + PHI-exfiltration suite: 0 criticals, 2 lows fixed"
  limitations: ["may be stale for coverage changed after KB snapshot"]
  human_oversight: "in-the-loop review required before any appeal draft is sent"
  approved_by: null                       # filled by the approval gate, below
```

```python
# Scenario: A structured audit event so every production inference is TRACEABLE — you
# can later answer "which model version handled this member, what tools fired, and did
# a human review the high-impact action?" PII is redacted before logging (data
# minimization) so the log itself isn't a new breach surface.
import json, hashlib, datetime

def audit_event(*, request_id, member_id, model_pin, prompt_version,
                tools_called, human_review, decision_summary):
    return json.dumps({
        "ts": datetime.datetime.now(datetime.timezone.utc).isoformat(),
        "request_id": request_id,
        # Pseudonymize the subject: keep a stable hash for tracing, not raw PHI/PII.
        "member_ref": hashlib.sha256(member_id.encode()).hexdigest()[:16],
        "model_version": model_pin,          # exact pin -> reproducible + attributable
        "prompt_version": prompt_version,
        "tools_called": tools_called,        # e.g. ["get_member_claims"]
        "human_in_loop": human_review,       # {"required": true, "reviewer": "...", "action": "approved"}
        "decision": decision_summary,        # redacted summary, no raw PHI in the log
    })
# Shipped to an append-only store with a defined retention window (regulatory + IR need).
```

```python
# Anti-pattern: deploy "whatever's latest" with no version pin, no approval record,
# and log the raw request/response. This is unauditable (can't say which model made a
# decision), non-reproducible (a silent provider model update changes behavior), and
# non-compliant (raw PHI/PII sits in logs, violating data minimization).
def serve(req):
    resp = call_model(model="gpt-4o", messages=req.messages)   # floating alias, no pin
    log.info(f"req={req.raw} resp={resp}")                      # raw PHI in the log!
    return resp                                                # no approval gate at all

# Correct approach: promotion is gated on an APPROVED, pinned system card; serving uses
# the pinned version; logging is the redacted structured event above.
APPROVED = {}  # registry: {system_name: {"version": ..., "model_pin": ..., "approved_by": ...}}

def promote(system_card: dict, approver: str):
    if system_card["approved_by"]:            # builder can't self-approve high-risk
        raise ValueError("already has approver; use review board")
    system_card["approved_by"] = approver     # recorded sign-off = accountability
    APPROVED[system_card["system_name"]] = {
        "version": system_card["version"],
        "model_pin": system_card["components"]["model"]["version_pin"],
        "prompt_version": system_card["components"]["prompt_version"],
        "approved_by": approver,
    }

def serve(req, system_name="member-support-agent"):
    cfg = APPROVED.get(system_name)
    if not cfg:                                # policy gate: no approval -> no serve
        raise RuntimeError("blocked: no approved version in registry")
    resp = call_model(model=cfg["model_pin"], messages=req.messages)  # pinned version
    log_line = audit_event(                    # redacted, traceable
        request_id=req.id, member_id=req.member_id, model_pin=cfg["model_pin"],
        prompt_version=cfg["prompt_version"], tools_called=req.tools_called,
        human_review=req.human_review, decision_summary=req.summary)
    audit_store.append(log_line)
    return resp
# What breaks without this: after an incident you cannot prove which model version acted,
# a provider update silently changes behavior (non-reproducible), and raw PHI in logs is
# itself a reportable breach. The fix makes every decision attributable, reproducible,
# and minimized — the three things an auditor asks for.
```

---

## Common Pitfalls & Misconceptions

- **Treating governance as "just security with more paperwork"** — beginners lump it in with chapter 02 because both are "the boring safety stuff." Security stops attackers (external threat); governance proves *your own* operation was run responsibly and answers "who approved this, on what evidence?" — you need both, and one cannot substitute for the other.
- **Governance = bureaucracy that slows shipping** — teams skip it to move fast, assuming it only adds friction. Proportionate governance is *risk-tiered*: low-risk internal tools auto-approve in minutes, and the heavy controls apply only where harm is real — the friction you feel is usually gold-plating a low-risk system, not governance itself.
- **Deploying against a floating model alias with no version pin** — "always latest" feels like a feature, so people point at `gpt-4o` not `gpt-4o-2024-08-06`. A silent provider update then changes behavior with no record; pin the exact version and record it so deployments are reproducible and attributable.
- **Logging raw prompts/responses for "debuggability"** — full-fidelity logs feel maximally useful. But raw PII/PHI in logs is a *new breach surface* and violates data minimization; redact/pseudonymize before logging and keep a traceable reference, not the raw sensitive data.
- **Measuring only aggregate quality** — a single accuracy number looks like enough evidence. Aggregate metrics hide subgroup harm (bias); governance requires **disaggregated** evaluation so a fairness regression for one group can't hide behind a good average.
- **No AI inventory ("shadow AI")** — teams govern the one system they know about and miss the five others quietly using an LLM. You cannot govern what you cannot see; a maintained use registry/inventory is the precondition for every other control.

---

## Key Definitions

| Term | Definition |
|---|---|
| Model governance | The policies, roles, artifacts, and processes that decide whether/how an AI system is deployed and hold it accountable over its lifecycle — the organizational wrapper around technical controls. |
| NIST AI RMF | NIST's voluntary AI Risk Management Framework (AI 100-1), organized into four functions: Govern, Map, Measure, Manage. |
| Govern / Map / Measure / Manage | The four AI RMF functions: cross-cutting policy/accountability (Govern), context & risk framing (Map), risk analysis & metrics (Measure), and prioritize/respond/monitor (Manage). |
| EU AI Act risk tiers | The Act's classification: prohibited (unacceptable), high-risk, limited-risk (transparency), and minimal-risk — obligations scale with the tier. |
| ISO/IEC 42001 | The 2023 international standard specifying a certifiable AI Management System (AIMS) using Plan-Do-Check-Act. |
| Model card / system card | Structured documentation of a model's (or assembled system's) purpose, data, evaluation, limitations, and intended use, attached to a specific version. |
| Data provenance / lineage | The record of where data came from and how it flowed through training/grounding, enabling deletion and contamination tracing. |
| AI use registry / inventory | A catalog of every AI use case with owner, risk tier, and status — the antidote to "shadow AI." |
| Approval gate | A pipeline/process control that refuses to promote a model version to production without a recorded sign-off. |
| Audit logging & traceability | Durable, queryable records of inputs, model version, tools, actors, and decisions per inference, enabling reconstruction after the fact. |
| Human oversight | A defined human role empowered to review, override, or halt AI decisions (in-the-loop, on-the-loop). |
| Red-teaming | Adversarial testing that deliberately attempts to make the system misbehave, feeding the Measure function. |
| Data minimization | Collecting and retaining only the personal data required, for the shortest necessary time — limiting breach/subpoena exposure. |

---

## Summary / Quick Recall

- Governance is the **organizational wrapper** (who approved what, on which version, logged where, reviewed by whom) around the technical defenses — not a substitute for them and not just paperwork.
- Anchor on three frameworks: **NIST AI RMF** (Govern/Map/Measure/Manage), the **EU AI Act** (prohibited → high-risk → limited/transparency → minimal), and **ISO/IEC 42001** (certifiable AI Management System).
- The lifecycle is **intake → risk classification → required controls → approval gate → deploy → monitor → audit**, with re-review on change or cadence.
- Core artifacts: **model/system card, data provenance/lineage, use registry/inventory, pinned model+prompt versions with recorded approval, structured redacted audit logs.**
- Controls are **risk-tiered**: heavy human oversight, red-teaming, and fairness/PII assessments apply where harm is real; low-risk internal tools stay lightweight.
- Responsible-AI dimensions to defend: **bias/fairness (disaggregated eval), transparency (AI disclosure), PII handling + data minimization, content-safety policy.**
- The three questions an auditor asks — *attributable, reproducible, minimized* — map directly to version pinning, approval records, and PII-redacted logging.

---

## Self-Check Questions

1. Name the four functions of the NIST AI Risk Management Framework and state which one is cross-cutting and continuous.

   <details><summary>Answer</summary>

   The four functions are **Govern, Map, Measure, and Manage**. **Govern is the cross-cutting, continuous function** — it establishes the culture, policies, accountability, and roles that *enable* Map (context/risk framing), Measure (risk analysis and metrics), and Manage (prioritize/respond/monitor). The tempting wrong answer is to list Govern as just the "first step" alongside the others; that is wrong because Govern is not a one-time phase — the AI RMF explicitly treats it as running throughout and underpinning the other three, which is why it's drawn wrapping around them rather than as a sequential box.

   </details>

2. You are launching an internal, non-PII developer tool that summarizes your own public API docs for staff. A colleague insists it needs a full review-board approval, disaggregated fairness testing, and human-in-the-loop oversight before launch. How do you respond?

   <details><summary>Answer</summary>

   Governance is **risk-tiered**, so this low-risk, internal, non-PII, non-consequential tool does *not* warrant the heavy controls reserved for high-risk systems. Register it in the AI inventory, pin the model/prompt version, and use lightweight single-owner approval — but full review-board sign-off, disaggregated fairness assessment, and blocking human-in-the-loop are disproportionate here (there is no protected-group decision and no external/PII harm). The tempting wrong answer is "always apply the maximum controls to be safe" — that is exactly the gold-plating that makes teams see governance as pure friction; proportionality is a governance principle, not a loophole.

   </details>

3. **Which TWO** of the following are the *governance* reasons to pin an exact model version (e.g. `gpt-4o-2024-08-06`) rather than a floating alias (`gpt-4o`)?
   - A. It makes deployments reproducible — the same version can be re-run to reproduce a past decision.
   - B. It guarantees the model will never produce a biased output.
   - C. It makes decisions attributable — an audit can tie a specific output to the exact version that produced it.
   - D. It reduces token cost per request.
   - E. It encrypts the audit log at rest.

   <details><summary>Answer</summary>

   **A and C.** Pinning makes deployments **reproducible** (A) — a silent provider update to a floating alias would change behavior with no record — and **attributable** (C), so an auditor or incident investigator can tie an output to the exact version that produced it. B is wrong: pinning a version does nothing to *guarantee* freedom from bias (you still need disaggregated fairness evaluation, and a pinned model can be biased). D and E are unrelated — pinning is about reproducibility/attribution, not cost or encryption. Reproducible + attributable are two of the three things (with *minimized*) an auditor asks for.

   </details>

4. A customer-facing chatbot for a bank currently logs the full raw prompt and response (including account numbers) "for debugging," runs against the floating `gpt-4o` alias, and has no recorded approval. A regulator asks you to prove which model version denied a specific customer's request last month and to demonstrate data minimization. Which failures block you, and what is the corrected design?

   <details><summary>Answer</summary>

   Three governance failures block you: (1) the **floating alias with no version pin** means you cannot prove which model version acted (non-attributable, non-reproducible); (2) the **raw PII in logs** violates data minimization and is itself a breach surface; (3) **no recorded approval** means no accountability trail. The corrected design pins the exact model + prompt version behind a **recorded approval gate**, and logs a **structured, PII-redacted/pseudonymized audit event** (stable hash reference, model version, tools, human-review decision, redacted summary) to an append-only store with a defined retention window. The tempting wrong answer is "just encrypt the logs" — encryption protects data at rest but does not fix attribution, reproducibility, or data minimization (you're still *retaining* the raw PII you didn't need).

   </details>

5. Your company is deploying two systems: (A) an informational customer-support chatbot that answers product questions, and (B) an AI system that screens and ranks job applicants. Both use the same underlying model. Compare their EU AI Act risk tiers and the governance implications, and justify why the tier differs despite the shared model.

   <details><summary>Answer</summary>

   The tier is set by the **use case and context, not the model's raw capability**, so the shared model is irrelevant to classification. (A) the informational support chatbot is **limited-risk** — its main obligation is a **transparency/AI-disclosure** (tell users they're talking to AI) — while (B) applicant screening is an **Annex III high-risk** use case, pulling in the full stack: risk-management system, data governance, technical documentation, record-keeping/logging, **human oversight**, and accuracy/robustness/cybersecurity targets, plus a disaggregated fairness assessment given the discrimination risk. The tempting wrong answer is "same model, so same tier/controls" — that misunderstands the Act entirely: identical technology can land in wildly different tiers because obligations attach to the *risk of the application to people*, not to the model.

   </details>

---

## Further Reading

- [NIST AI Risk Management Framework (AI RMF 1.0)](https://www.nist.gov/itl/ai-risk-management-framework) — *verified 2026-07-29* — Official NIST landing page for the AI RMF and its four functions (Govern, Map, Measure, Manage), with links to the AI RMF 1.0 PDF and the Generative AI Profile.
- [NIST AI RMF Playbook](https://airc.nist.gov/airmf-resources/playbook/) — *verified 2026-07-29* — Suggested actions aligned to each Govern/Map/Measure/Manage sub-category, from NIST's Trustworthy & Responsible AI Resource Center.
- [High-level Summary of the EU AI Act](https://artificialintelligenceact.eu/high-level-summary/) — *verified 2026-07-29* — Authoritative summary of the Act's risk tiers (prohibited, high-risk, limited/transparency, minimal) and provider obligations, maintained by the Future of Life Institute.
- [ISO/IEC 42001:2023 — AI management systems](https://www.iso.org/standard/42001) — *verified 2026-07-29* — Official ISO page for the world's first certifiable AI management system standard (AIMS), using the Plan-Do-Check-Act pattern.
- [OWASP LLM Applications Cybersecurity and Governance Checklist](https://genai.owasp.org/resource/llm-applications-cybersecurity-and-governance-checklist-english/) — *verified 2026-07-29* — OWASP GenAI Security Project checklist for leaders covering AI governance, risk, and control practices for LLM applications.
- [OWASP GenAI — LLM and Generative AI Security Solutions Landscape](https://genai.owasp.org/resource/llm-and-generative-ai-security-solutions-landscape/) — *verified 2026-07-29* — Reference guide mapping the categories of controls (including governance/inventory) used to secure LLM and GenAI applications in production.
