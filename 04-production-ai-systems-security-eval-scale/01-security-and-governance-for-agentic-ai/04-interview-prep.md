# Security & Governance for Agentic AI — Interview Prep

**Section:** Production AI Systems — Security, Evaluation & Scale | **Role target:** Applied AI Engineer / AI Systems Engineer (agentic AI, RAG, multi-agent production systems)

---

## Core Conceptual Questions

These test whether you understand the fundamentals.

| Question | Key points to cover | Common weak-answer trap |
|---|---|---|
| What is the OWASP LLM Top 10 (2025) and which entries bite hardest for *agentic* systems specifically? | It's the industry-standard risk taxonomy for LLM applications: LLM01 Prompt Injection, LLM02 Sensitive Information Disclosure, LLM03 Supply Chain, LLM04 Data & Model Poisoning, LLM05 Improper Output Handling, LLM06 Excessive Agency, LLM07 System Prompt Leakage, LLM08 Vector & Embedding Weaknesses, LLM09 Misinformation, LLM10 Unbounded Consumption. For agents the sharpest are **LLM01 (prompt injection)** and **LLM06 (excessive agency)** because an agent turns model output into *actions* — injection + excessive agency together is how a compromised prompt becomes a real-world side effect. LLM05 (output handling), LLM08 (RAG/vector weaknesses) and LLM10 (unbounded consumption) round out the agentic attack surface. | Reciting the list as a generic "security checklist" with no ranking, or treating all ten as equally relevant to an agent; missing that agency is what upgrades injection from a text problem to a breach. |
| Distinguish direct from indirect prompt injection, and explain why it can't be "solved" at the model layer. | **Direct**: the user types adversarial instructions straight into the prompt ("ignore previous instructions…"). **Indirect**: malicious instructions are embedded in content the agent *retrieves or reads* — a web page, a PDF, a RAG document, a tool result — and the model can't reliably tell trusted instructions from untrusted data because both arrive as the same token stream. There is no clean in-band separator between "instructions" and "data," so no prompt or model tweak fully closes it; it's mitigated with **defense-in-depth**, not solved. | "We fixed it by telling the model in the system prompt to ignore injected instructions," or "input filtering solves it" — both assume a model-layer fix exists; indirect injection from retrieved content bypasses input filters entirely. |
| What is Excessive Agency (LLM06) and how do you constrain it? | An agent granted more capability, permissions, or autonomy than the task requires — too many tools, over-broad scopes, or the ability to take irreversible actions unsupervised. The blast radius of a compromised or hallucinating agent equals the union of everything its tools can do. Constrain via **least-privilege tool design**, **tool allow-listing**, scoped/short-lived credentials per action, and **human-in-the-loop gates** on irreversible or high-impact operations. | Framing it as "the model made a mistake" rather than "we granted more authority than needed"; assuming a better prompt reduces agency — agency is an *authorization* property, not a prompting one. |
| Why is tool-layer RBAC / least-privilege the key *architectural* control for agents rather than prompt hygiene? | Prompt-level instructions are advisory and bypassable (injection defeats them); authorization enforced at the **tool execution layer** is not — the agent can only ever do what its granted permissions allow, regardless of what any injected text tells it. You enforce authz where the action actually executes: allow-list tools per agent/role, check the caller's permissions server-side before the tool runs, scope credentials to the minimum, and validate/constrain tool arguments. This bounds blast radius by construction. | "We put the security rules in the system prompt" — the system prompt is not a security boundary (see LLM07 System Prompt Leakage); treating authz as something the model self-enforces rather than something the runtime enforces. |
| Name the major AI governance frameworks and what each contributes. | **NIST AI RMF** — voluntary risk-management framework structured around four functions: **Govern, Map, Measure, Manage** (Govern = culture/policy/accountability; Map = context & risk framing; Measure = assess/track risks; Manage = prioritize & respond). **EU AI Act** — regulation with **risk tiers** (unacceptable / high / limited / minimal) driving obligations. **ISO/IEC 42001** — a certifiable AI *management system* standard (the "ISO 9001 for AI"). Operationalized through model/system cards, data provenance, an AI use registry, versioning + approval workflows, audit logging, red-teaming, and human oversight. | Name-dropping "responsible AI" with no framework specifics; getting the RMF functions wrong (it's Govern/Map/Measure/Manage, not "identify/protect/respond"); conflating the EU AI Act (law, risk-tiered) with NIST RMF (voluntary, function-based) or ISO 42001 (management-system certification). |
| What is Unbounded Consumption (LLM10) and why does it matter more for agents? | Uncontrolled resource use — token spend, tool-call volume, API cost, wall-clock — from unbounded loops, recursive tool calls, or adversarial "denial-of-wallet" prompts. Agents amplify it because a single request can fan out into many model + tool calls. Mitigate with per-session iteration caps, timeouts, rate limits, and cost/quota budgets. | Treating it purely as an availability/DoS concern and ignoring the *cost* (denial-of-wallet) angle; assuming token limits alone bound an agent that can call tools in a loop. |

---

## Applied / Scenario Questions

### Q1 — Your RAG agent ingests customer-uploaded PDFs and support tickets, then can call internal tools (lookup, update, email). Security flags that a malicious document could hijack it. How do you defend it?

**Strong answer framework:**

- **Name the threat precisely:** this is **indirect prompt injection (LLM01)** via retrieved/ingested content combined with **excessive agency (LLM06)** — the uploaded doc carries instructions, and the agent's tools give those instructions a real-world effect. Filtering the *user's* message does nothing here because the payload rides in the retrieved document.
- **Defense-in-depth, not a single control:** apply **spotlighting** (delimit and mark retrieved content as untrusted data so the model treats it as data, not instructions), content/output filtering, and **constrained outputs** (the model emits structured tool requests, not free text your code blindly executes).
- **Make the tool layer the real boundary:** enforce **tool allow-listing** and **RBAC/least-privilege** so even a fully hijacked prompt can only invoke the narrow, read-scoped tools this agent is authorized for; validate tool arguments server-side (e.g. the `email` tool can only send to verified internal addresses, never an attacker-supplied one).
- **Gate the irreversible actions:** route state-changing tools (`update`, outbound `email`) through a **human-in-the-loop approval** step.
- **Trade-off framing:** spotlighting + filtering add some latency and produce **false positives** (legitimate content flagged) that hurt UX; HITL on every action kills throughput. The right cut is: cheap, always-on controls (allow-listing, arg validation, spotlighting) on the read path, and reserve the expensive HITL friction for irreversible/high-blast-radius actions only. Log everything for audit.

### Q2 — Leadership wants to ship an autonomous agent that can browse the web and act on findings (book, purchase, post). Legal and security are nervous. Where do you push back, and what do you propose?

**Strong answer framework:**

- **Start from blast radius:** an agent reading **untrusted web content** and taking **irreversible actions** is the textbook worst case — indirect injection (LLM01) meets excessive agency (LLM06) meets unbounded consumption (LLM10, denial-of-wallet on purchases). Push back on "fully autonomous" for money-moving/public-posting actions.
- **Separate trust boundaries:** treat everything the browser returns as untrusted data. Consider a **dual-LLM pattern** — a quarantined model processes the untrusted page content and only emits structured, non-executable data; a privileged planner never sees raw untrusted tokens and drives the tools. This structurally prevents injected page instructions from reaching the action-taking path.
- **Least-privilege + caps:** allow-list a minimal tool set, scope credentials per action, and set hard spend/rate/iteration budgets to cap denial-of-wallet.
- **Human-in-the-loop on the irreversible tier:** purchase/booking/posting require explicit approval; reversible reads run autonomously.
- **Governance wrap:** an **AI use registry** entry, a **system card** documenting scope and limits, **audit logging** of every action and its approver, and **red-teaming** the injection paths before launch. Under the **EU AI Act** this could land in a higher risk tier depending on use, which raises the oversight/documentation bar.
- **Trade-off framing:** dual-LLM and HITL add latency, cost (two model passes), and UX friction; full autonomy maximizes speed but makes the blast radius unbounded. The defensible position is autonomy on reversible reads, gated approval on irreversible actions — trading throughput for safety and auditability exactly where the action is irreversible.

---

## System Design / Architecture Questions

### Q — Design a secure agent that browses/reads untrusted external content (web pages, third-party docs) and can take actions on internal systems, with governance suitable for a regulated org.

**Approach:**

1. **Clarify requirements.** Which actions are read-only vs. state-changing vs. irreversible (money/PII/public)? Data sensitivity (PII in the internal systems)? Latency/UX budget vs. tolerance for approval friction? Cost/spend ceiling per session? Regulatory posture (EU AI Act risk tier, ISO 42001 / NIST RMF program)? Is human oversight mandatory, and for which actions?

2. **Threat-model first (trust boundaries + blast radius).** Draw the boundary: external content is **untrusted input**; internal tools are the **high-value assets**. Enumerate the relevant OWASP entries — **LLM01** (indirect injection from browsed content), **LLM06** (excessive agency on internal tools), **LLM05** (improper output handling if tool args are unvalidated), **LLM08** (poisoned/adversarial content polluting the vector store), **LLM10** (unbounded consumption / denial-of-wallet). Define blast radius = union of what the granted tools can do, and design to shrink it.

3. **Defense-in-depth architecture.**
   - **Input/content layer:** **spotlight** and delimit all retrieved/browsed content as untrusted data; run content filtering; never concatenate untrusted content into the instruction channel unmarked.
   - **Model isolation:** a **dual-LLM split** — a quarantined LLM parses untrusted content into structured data only; a privileged planner LLM (which never sees raw untrusted tokens) selects actions. **Constrained/structured outputs** so the runtime dispatches only well-formed tool calls.
   - **Authorization layer (the real boundary):** **tool allow-listing** per agent role + **RBAC / least-privilege** enforced server-side at tool execution, scoped short-lived credentials per action, and server-side **argument validation** (e.g. recipients/accounts restricted to verified allow-lists).
   - **Human-in-the-loop gates:** irreversible/high-impact tools pause for approval.
   - **Consumption controls:** iteration caps, timeouts, rate limits, and per-session spend budgets (LLM10).

4. **Governance & audit layer.**
   - **Audit logging** of every tool call, its arguments, the model rationale/trace, and the human approver — immutable and queryable.
   - **AI use registry** entry; **model card / system card** documenting scope, limits, and evaluated risks; **versioning + approval workflow** for prompt/model/tool changes.
   - **Data provenance** on ingested content; **bias/PII controls** on inputs and outputs.
   - **Red-teaming** the injection and privilege-escalation paths pre-launch; map controls to **NIST AI RMF (Govern/Map/Measure/Manage)** and the applicable **EU AI Act** tier / **ISO 42001** management system.

5. **Justify choices and name trade-offs.**
   - **Authz at the tool layer over prompt rules:** the only enforceable boundary; a hijacked prompt still can't exceed granted permissions. Trade-off: more server-side plumbing vs. a boundary that actually holds.
   - **Dual-LLM:** structurally prevents injected instructions from reaching the action path. Trade-off: extra model pass → latency + cost; justified because the content is untrusted and actions are consequential.
   - **HITL only on the irreversible tier:** trades throughput/latency for safety + auditability where it matters, without taxing every read.
   - **Consumption caps:** trade a rare accuracy loss on pathological runs for bounded cost and no denial-of-wallet.
   - **Governance overhead:** documentation/registry/audit cost real effort but is the price of operating in a regulated tier and is what makes incidents investigable.

---

## Vocabulary That Signals Expertise

Use these terms naturally — don't force them:

- **Indirect (vs. direct) prompt injection** — when explaining that the dangerous payload arrives in *retrieved/browsed* content, not the user's message, and therefore bypasses input filtering.
- **Excessive agency (LLM06)** — when framing risk as an *authorization/capability* property (too many tools, too broad scope) rather than a model mistake.
- **Blast radius** — when sizing the impact of a compromised or hallucinating agent as the union of what its tools can do.
- **Trust boundary** — when separating untrusted external content from privileged internal tools/assets in a threat model.
- **Defense-in-depth** — when arguing prompt injection is mitigated by layered controls, not a single fix.
- **Spotlighting** — when marking/delimiting retrieved content as untrusted data so the model treats it as data, not instructions.
- **Dual-LLM pattern** — when isolating a quarantined content-parsing model from a privileged action-taking model.
- **Tool allow-listing / RBAC / least-privilege / constrained outputs** — when describing the enforceable authorization boundary at the tool layer.
- **Human-in-the-loop (HITL) gate** — when requiring approval on irreversible/high-impact actions.
- **Denial-of-wallet / unbounded consumption (LLM10)** — when discussing cost-based abuse and the need for caps/budgets.
- **NIST AI RMF (Govern / Map / Measure / Manage)** — when structuring a governance program by its four functions.
- **EU AI Act risk tiers / ISO 42001** — when placing obligations by risk level or referencing a certifiable AI management system.
- **Model card / system card, AI use registry, data provenance, audit trail, red-teaming** — when describing concrete governance artifacts and controls.

---

## Vocabulary That Signals Weakness

Avoid these — they signal outdated or shallow understanding:

- **"We tell the model in the prompt not to do that."** — Red flag: the system prompt is advisory and leakable (LLM07); it is not a security boundary and injection overrides it.
- **"Prompt injection is solved by input filtering."** — Red flag: input filters miss **indirect** injection from retrieved/browsed content entirely; injection is mitigated by defense-in-depth, not solved.
- **"Security is a content filter on inputs and outputs."** — Red flag: reduces a multi-layer problem (authz, agency, consumption, governance) to one control; ignores the tool/authorization layer where breaches actually happen.
- **"We'll add governance/audit later."** — Red flag: audit trails, use registry, and approval workflows are load-bearing controls, not paperwork; retrofitting them after an incident is too late and often legally required up front.
- **"The agent just needs access to everything to be useful."** — Red flag: textbook excessive agency; blast radius = everything, and one injection compromises all of it.
- **"The model wouldn't do that / it's trained to be safe."** — Red flag: relies on model behavior for what must be an *enforced* authorization control at the runtime.
- **"We log requests" (as the whole answer on governance).** — Red flag: conflates basic logging with an auditable trail of actions, arguments, rationale, and approvers required for real oversight.

---

## STAR Answer Frame

**Situation:** I owned a production LangGraph multi-agent RAG system on FastAPI/PostgreSQL where a research agent ingested customer-supplied documents and third-party web content, then called internal tools (lookup, record update, outbound notification). A security review flagged that a malicious ingested document could steer the agent — classic indirect prompt injection meeting excessive agency, with no enforced boundary between untrusted content and the action-taking tools.

**Task:** Close the injection-to-action path and stand up defensible governance without gutting the agent's usefulness or interactive latency — and do it in a way security and (given a regulated customer base) compliance would sign off on.

**Action:** I threat-modeled the flow against the OWASP LLM Top 10, fixing on LLM01, LLM05, LLM06, and LLM10, and mapped trust boundaries so all ingested/browsed content was treated as untrusted data. I (1) added **spotlighting** to delimit retrieved content and moved the agent to **constrained/structured tool outputs**; (2) split parsing from action with a **dual-LLM** boundary so the privileged planner never saw raw untrusted tokens; (3) enforced **RBAC/least-privilege and tool allow-listing server-side** at tool execution with per-action scoped credentials and **argument validation** (notifications restricted to verified internal recipients); (4) routed irreversible actions through a **human-in-the-loop approval** gate with a Postgres-backed audit trail capturing arguments, rationale, and approver; (5) set **iteration/spend caps** against denial-of-wallet; and (6) registered the system in the **AI use registry** with a **system card**, versioning/approval workflow, and a **red-team** pass on the injection paths — mapping the whole control set to **NIST AI RMF (Govern/Map/Measure/Manage)**.

**Result:** The red-team's indirect-injection payloads that previously reached a tool call were fully contained — post-fix, no injected instruction could invoke an unauthorized or irreversible action (blast radius reduced to the allow-listed read tools). HITL was scoped to only the irreversible tier, so median interactive latency was unaffected while every state-changing action gained an approver and an immutable audit record — which is what unblocked the compliance sign-off for the regulated deployment.

---

## Red Flags Interviewers Watch For

Specific to security & governance for agentic AI:

- **No tool-layer authorization** — relying on prompt instructions instead of server-side RBAC/least-privilege at tool execution; the single clearest sign the candidate doesn't know where the real security boundary is.
- **Trusting the system prompt as a control** — treating the system prompt as a security boundary despite prompt injection (LLM01) and system prompt leakage (LLM07).
- **No handling of indirect injection from retrieved content** — defending only the user's input and ignoring that RAG/browsed/ingested content is the real injection vector; no spotlighting, dual-LLM, or untrusted-data separation.
- **Unbounded agency / spend** — granting broad tool access and no caps, with no notion of blast radius or denial-of-wallet (LLM06 + LLM10).
- **No human-in-the-loop on irreversible actions** — letting the agent autonomously take money-moving, destructive, or public actions with no approval gate.
- **No threat model** — jumping to controls without enumerating trust boundaries, assets, and the relevant OWASP entries; controls with no threats behind them.
- **No audit trail / governance artifacts** — no immutable log of actions/arguments/approvers, no AI use registry, model/system card, versioning/approval, or red-teaming; "we'll add governance later."
- **Framework name-dropping without substance** — invoking "responsible AI" or naming NIST/EU AI Act/ISO 42001 but unable to say what each contributes or getting the RMF functions (Govern/Map/Measure/Manage) wrong.
