# STARTUP

### A Workflow & AI Command Center for Winning Competitive Product-Building Events

---

## Overview

Startup is a solo, web-based AI command center that orchestrates the end-to-end process of going from a hackathon brief to a submission-ready, judge-optimized product. It is prescriptive by design — it tells you what to do next, enforces process discipline, and maintains the full context of your project across every phase.

Startup is not a generic AI assistant. It is a purpose-built workflow engine grounded in the belief that most hackathon teams lose not because their idea is bad, but because they run a broken process: late domain intelligence, no deliberate differentiation strategy, mid-build scope pivots, and poor translation of good work into compelling presentations. Startup fixes the process so the quality of your idea has a fair chance of actually winning.

While designed primarily for hackathons, its foundations are solid enough to be adapted to real-world client software engagements as a stretch goal.

---

## Core Design Principles

**Context is the spine.** Every phase produces structured outputs that are written into a central, living Project Context Document. Every subsequent phase reads from this document. Nothing is lost between phases. The context grows richer as you move forward.

**Startup directs. Humans and powerful LLMs execute.** Startup's internal AI engine (Gemini Flash, free tier) handles context management, prompt engineering, and output ingestion. It does not do heavy reasoning or verbose generation. Instead, it generates optimized prompts — grounded in the project context — that you execute in an external LLM of your choice (Claude, Gemini Pro, ChatGPT, Deepseek). You paste the result back into Startup, which parses it, extracts the structured information, and writes it into the project context. Lightweight in, heavyweight out, structured back in.

**Research is directed, not autonomous.** For anything requiring deep external research — domain frameworks, judge profiles, past winning solutions, market data — Startup generates precise, structured research questions and prompts. You execute these in Gemini Deep Research or Perplexity and paste back the findings. Startup ingests them into the project context.

**The Phase 4 gate is sacred.** The workflow is a state machine. You can move backward through phases when new information arrives. But Phase 4 completion is a one-way gate. Once crossed, scope is locked. The only exception is the Addendum Protocol, which is deliberately friction-heavy.

**Prescriptive but not imprisoning.** Startup warns you before you skip something critical, tracks completion of required steps per phase, and discourages shortcuts. It does not block you outright — you can override with explicit acknowledgment. But the defaults push you toward the right behavior.

---

## The Project Context Document

The Project Context Document (PCD) is the single most important artifact in the Startup system. It is a structured markdown file that lives inside the app and is automatically updated as you complete each phase.

Every AI prompt Startup generates is grounded in the current state of the PCD. Every output you paste back in is parsed, structured, and written into the appropriate section of the PCD. It is never raw-appended — Startup always extracts the relevant information and maps it to a consistent schema.

The PCD contains the following top-level sections, populated progressively across phases:

- **Hackathon Profile** — organizer, sponsors, theme, timeline, format, known criteria
- **Judge Intelligence** — judge personas, scoring tendencies, past winner patterns
- **Active Judging Rubric** — real criteria if provided, default rubric if not
- **Problem Analysis** — problem statement parse, root causes, stakeholder map, HMW reframes
- **Competitive Landscape** — competitive pre-mortem, likely competitor directions
- **User Research** — personas, interview insights, validated pain points
- **Ideation Record** — SCAMPER ideation output, JTBD analysis, Brainwriting results, seed challenge outcome, and downselection rationale; written progressively across Steps 3b–3e
- **Candidate Portfolio** — structured list of 3–5 divergent candidates from Phase 3, each with name, description, and Innovation / Impact / Feasibility scores
- **Chosen Direction** — selected solution, key differentiators, design rationale
- **Scope Definition** — MoSCoW feature list, MVP definition, explicit out-of-scope items
- **Technical Direction** — tech stack decisions, architecture summary, data requirements
- **Phase Gate Log** — record of all phase completions, overrides, and addenda
- **Debrief Record** — post-event retrospective, learnings, boilerplate updates
- **External Context Notes** — a user-maintained, free-form Markdown field where you can manually paste compressed summaries of past external LLM interactions. Particularly useful before starting a new Grill-Me session: by summarizing prior decisions made in long dialogues and pasting them here, you give the next LLM session a running memory of where the project stands without needing to replay the full conversation. When this field is populated, the Prompt Generator can optionally inject it into the context window of any generated prompt.

The PCD is also exportable at any time as a standalone markdown file, making it portable to Cursor or any external tool as a master context document.

---

## The Workflow

The workflow consists of seven phases plus one emergency protocol. Phases 1–6 are active during a hackathon event. Phase 0 is ongoing between events. Phase 7 happens immediately after. The Addendum Protocol can only be triggered after Phase 4.

---

### Phase 0 — Pre-Hackathon Readiness

**When:** Ongoing, between events. Not tied to any specific hackathon.

**Purpose:** Build and maintain the standing infrastructure that makes every hackathon run faster and smarter. This is the compounding advantage — every hackathon you complete makes the next one easier.

**Human Tasks:**

- Maintain the **Boilerplate Starter Kit**: a library of pre-built, reusable components and templates including common frontend UI components (navigation, dashboards, forms, data visualization, chatbot UI), backend architecture templates for common hackathon patterns (AI-connected APIs, auth-free quick-deploy setups), and deployment configurations for fast hosting. Updated after every hackathon based on what was built.
- Maintain the **Judge & Organizer Database**: a persistent record of hackathon judges, organizers, and sponsors encountered across events. Populated via LinkedIn research and post-event observation. Contains: names, organizations, professional backgrounds, judging tendencies observed, and which events they appeared at.
- Maintain the **Winning Solutions Archive**: a growing record of what has won past hackathons — by organizer, by theme, by domain. Used to inform competitive pre-mortems in Phase 2.
- Review upcoming hackathon calendars and identify target events.

**App Role:**

- Provides a structured interface for updating the boilerplate library (tagging components, noting what worked and what didn't).
- Provides a searchable database interface for the judge/organizer records.
- Surfaces relevant past entries automatically when a new hackathon is registered in Phase 1.

---

### Phase 1 — Intelligence Gathering

**When:** Triggered by hackathon registration or problem statement release.

**Purpose:** Build a complete intelligence picture of the hackathon before a single idea is considered. Understand who is organizing it, who will judge it, what they care about, and what winning looks like in this specific context.

**Trigger:** You create a new project in Startup and input the known hackathon details: organizer name, sponsors, theme or track, problem statement (if available), judging criteria (if available), timeline, format (in-person, hybrid, virtual), and team size requirements.

**App Tasks:**

- Parses all inputs and populates the Hackathon Profile section of the PCD.
- Cross-references the organizer and sponsor names against the Judge & Organizer Database and surfaces any matching historical entries.
- Activates the **Default Judging Rubric** immediately, regardless of whether official criteria have been provided. The default rubric is built around five universally weighted criteria: Innovation & Novelty, Problem-Solution Fit, Technical Soundness, Feasibility & Viability, and Presentation & Communication. Each criterion has a description and a set of questions judges in that category typically ask.
- If official criteria are provided, the app parses them, maps them onto the rubric schema, and replaces or supplements the defaults accordingly. It flags any criteria that don't map cleanly and asks for your interpretation.
- Generates a structured **Judge Research Prompt Package**: a set of precise research questions and search strategies you will execute in Gemini Deep Research or Perplexity. Covers: likely judge profiles based on organizer and sponsor type, LinkedIn search strategies for surfacing past judges, past winning solutions from this or similar events, and domain landscape questions specific to the hackathon theme.

**Human Tasks:**

- Execute the research prompts in Gemini Deep Research or Perplexity.
- Paste findings back into the app's research intake module.
- Optionally add specific judge names or profiles if they are announced early.

**App Tasks (post-intake):**

- Parses all pasted research.
- Constructs **Judge Personas**: composite profiles of the likely judging panel, including professional background, industry lens, what they likely value in solutions, what they are likely skeptical of, and what kinds of solutions have impressed similar judges historically.
- Populates the Judge Intelligence section of the PCD.
- Flags if the judging criteria (real or default) may shift based on the organizer profile, and notes what to watch for.

**Phase Output:** A fully populated Hackathon Intelligence Brief — organizer and sponsor profile, judge personas, active judging rubric, past winner patterns, and a list of flagged unknowns to monitor. All stored in the PCD.

**Phase Completion Requirement:** Hackathon profile complete, judge personas generated (even if based on assumptions), active rubric confirmed. You acknowledge any flagged unknowns before proceeding.

---

### Phase 2 — Problem Deconstruction

**When:** Triggered by Phase 1 completion.

**Purpose:** Deeply understand the problem before generating any solutions. More importantly, find the non-obvious angle — the interpretation of the problem that the average team will miss and that creates the space for a genuinely differentiated solution.

**App Tasks:**

- Reads the problem statement and the Hackathon Intelligence Brief from the PCD.
- Generates a **Problem Analysis Prompt**: a structured prompt for an external LLM that will perform deep problem parsing. The prompt instructs the LLM to apply 5 Whys root cause analysis, stakeholder mapping (who is affected, who is responsible, who has authority, who influences the outcome), and "How Might We" reframing — pushing toward non-obvious angles rather than the first-order interpretation of the brief.
- Generates a **Competitive Pre-mortem Prompt**: a structured prompt asking the external LLM to model what the average team will build for this brief. Given the problem statement, the domain, and the hackathon context, what are the three to five solutions that most teams will converge on? What features will they build? What framework will they apply? This is the map of the crowded space — the explicit purpose is to show you where not to go.
- Generates a **User Research Prompt Package**: specific interview questions, user observation guides, and survey questions tailored to the problem statement and the target user personas the app has inferred from the brief. These are intended to be executed with real users as early as possible, ideally before Phase 3, but tracked as a parallel activity if time doesn't allow.

**Human Tasks:**

- Execute the Problem Analysis Prompt and Competitive Pre-mortem Prompt in your external LLM.
- Paste results back into the app.
- Begin user research in parallel if access to target users is available. Paste findings into the user research intake module as they arrive.

**App Tasks (post-intake):**

- Parses problem analysis output and populates the Problem Analysis section of the PCD.
- Parses competitive pre-mortem output and populates the Competitive Landscape section.
- Parses any user research input and populates the User Research section.
- Identifies and flags the top two to three non-obvious angles that emerged — the spaces where differentiation is possible.
- Checks: has the Competitive Pre-mortem been completed? If not, the app blocks progression with a warning. This is the one step that cannot be skipped, because skipping it is exactly how teams end up building the same thing as everyone else.

**Phase Output:** A populated Problem Analysis (root causes, stakeholder map, HMW reframes), a Competitive Pre-mortem (what the average team will build and why), initial user personas, and a set of flagged differentiation opportunities. All in the PCD.

**Phase Completion Requirement:** Problem analysis complete, competitive pre-mortem complete (hard requirement), at least one differentiation angle identified. User research is tracked but not a hard blocker — the app notes whether it has been done and flags the gap if not.

---

### Phase 3 — Differentiated Ideation

**When:** Triggered by Phase 2 completion.

**Purpose:** Generate a wide, deliberately diverse solution space and converge on a single chosen direction — one that scores well against the active judging rubric and avoids the crowded space identified in the competitive pre-mortem. Phase 3 is human-led: the app generates the prompts, but the quality of what comes out depends on the thinking you bring into it.

---

#### Step 3a — Idea Seed Challenge *(Optional)*

If you already have a preconceived idea before ideation begins, enter it here before proceeding. Skipping this step is fine — if you have no starting idea, move directly to Step 3b.

**App Tasks:**

- Accepts a free-text description of your seed idea.
- Reads the full Phase 1–2 PCD context (problem analysis, competitive pre-mortem, judge personas, active rubric).
- Generates an **Idea Seed Challenge Prompt** in Grill-Me mode (always on for this step — this is adversarial by design). The external LLM is instructed to adopt the role of a relentless skeptic — not to critique the idea in one pass, but to interrogate it through targeted, one-at-a-time questions using the six Socratic question types: clarification, probing assumptions, probing reasons, exploring viewpoints, examining implications, and meta-questioning. It does not accept vague answers. It surfaces hidden risks, untested assumptions, and blind spots by probing every branch of the idea's logic until all lines of questioning are resolved.
- The prompt can be run in Interactive mode (you answer questions live in the external LLM's chat) or Auto-Grill mode (the LLM simulates the full dialogue internally).

**Human Tasks:**

- Execute the Idea Seed Challenge prompt in your external LLM.
- If running Interactive mode, engage with the questions directly until the LLM signals completion.
- Paste back only the final structured verdict (not the full dialogue transcript).

**App Tasks (post-intake):**

- Accepts the structured verdict: **"Adopt as seed,"** **"Refine and adopt"** (with the refined idea text), or **"Reject."**
- Surfaces the verdict for your review. You make the final call — the app records your decision.
- If adopted or refined, the seed idea is carried forward as a starting point in Step 3e. If rejected, Step 3e proceeds without it.
- Records the Idea Seed Challenge outcome in the PCD's `candidate_portfolio` field (as a preamble to the candidates generated in Step 3e).

---

#### Step 3b — SCAMPER Ideation

**App Tasks:**

- Reads the Phase 1–2 PCD plus the Idea Seed Challenge outcome (if Step 3a was completed).
- Generates a **SCAMPER Ideation Prompt** for execution in an external LLM. The prompt instructs the LLM to run a multi-persona dialogue — speaking as the target user, a domain expert, and a technical expert — and to explore all seven SCAMPER dimensions (Substitute, Combine, Adapt, Modify, Put to other uses, Eliminate, Reverse) against the problem space. Each dimension is applied in turn, with the three personas responding to each other's perspectives.
- The prompt embeds the problem analysis, differentiation angles, and competitive pre-mortem so the exploration is grounded and avoids retreading predicted average-team directions.

**Human Tasks:**

- Execute the SCAMPER Ideation prompt in your external LLM.
- Paste the output back into the app.

**App Tasks (post-intake):**

- Parses the SCAMPER output and writes it to the `ideation_record` field of the PCD.
- Surfaces a brief summary of the most promising angles that emerged for your review before proceeding.

---

#### Step 3c — Jobs-to-Be-Done Analysis

**App Tasks:**

- Reads the current PCD including the SCAMPER output now written to the Ideation Record.
- Generates a **JTBD Analysis Prompt** for execution in an external LLM. The prompt instructs the LLM to surface the functional, emotional, and social jobs the target user is trying to accomplish — and, critically, to identify unmet job dimensions that current solutions in the competitive landscape ignore. These gaps are the ground where differentiation is most achievable.

**Human Tasks:**

- Execute the JTBD Analysis prompt in your external LLM.
- Paste the output back into the app.

**App Tasks (post-intake):**

- Parses the JTBD output and appends it to the `ideation_record` in the PCD.
- Flags any unmet job dimensions that were not already captured in the Phase 2 differentiation angles — these are new signal and are highlighted separately.

---

#### Step 3d — Brainwriting Round

**App Tasks:**

- Reads the current PCD including all Ideation Record entries produced so far.
- Generates a **Brainwriting Prompt** for execution in an external LLM. The prompt instructs the LLM to produce two lists: five conventional ideas and five unconventional ideas. Both lists are written with explicit knowledge of the competitive pre-mortem — any idea that appears on the average-team solution map is disqualified before being proposed. The unconventional list is additionally instructed to push against the grain of any idea already present in the SCAMPER and JTBD outputs, not to repeat promising angles but to find new ones.

**Human Tasks:**

- Execute the Brainwriting prompt in your external LLM.
- Paste the output back into the app.

**App Tasks (post-intake):**

- Parses the Brainwriting output and appends it to the `ideation_record` in the PCD.
- The full Ideation Record — SCAMPER output, JTBD analysis, and Brainwriting results — is now complete and ready to be injected as context into Step 3e.

---

#### Step 3e — Divergent Candidate Generation

This is the synthesis step. Everything produced in Steps 3b–3d feeds directly into this prompt as context, giving the external LLM a rich, grounded ideation base to draw from when generating the final candidate portfolio.

**App Tasks:**

- Reads the full PCD — Phase 1–2 outputs plus all Ideation Record entries from Steps 3b–3d — and the Idea Seed Challenge outcome (if Step 3a was completed).
- Generates a single **Divergent Candidate Generation Prompt** for execution in an external LLM. The LLM is asked to synthesize the ideation work done so far and produce 3–5 radically different candidate ideas. It is not starting from scratch — it is building on the SCAMPER dimensions explored, the job gaps identified, and the brainwriting territory mapped.
- The prompt embeds:
  - The full Ideation Record (SCAMPER, JTBD, Brainwriting outputs) as grounding context.
  - The adopted seed idea, if one exists, as a named starting point.
  - **Enforced Diversity Archetypes** (on by default, toggleable off in the Phase Workspace). The four archetypes the LLM must represent across its candidates are:
    - *Safe Utility* — practical, high-feasibility MVP that solves the core problem cleanly.
    - *Weird Gem* — unconventional approach that pushes against expected solution patterns.
    - *Moonshot* — technically ambitious, leverages emerging or underused technology.
    - *Social Impact* — centers marginalized or overlooked users and their specific context.
  - Explicit instruction to avoid any candidate idea that overlaps with the average-team solutions identified in the competitive pre-mortem. Ideas that appear on that list are disqualified before they are proposed.
  - If Grill-Me refinement is toggled on, an additional instruction directing the LLM to briefly interrogate each candidate's core assumption before finalizing it — using the same adversarial Socratic approach — and to revise any candidate that does not survive that interrogation.

**Human Tasks:**

- Execute the Divergent Candidate Generation prompt in your external LLM.
- If the session includes a Grill-Me refinement dialogue and you are running in Interactive mode, participate in the questioning for any candidates the LLM challenges.
- If user research results arrived during this phase, paste them into the user research intake module before executing this prompt — the app will incorporate them into the context injection.
- Paste back the final structured candidate list once the LLM signals it is complete.

**App Tasks (post-intake):**

- Accepts the structured candidate list. Each candidate must include: a name, a one-line description, and plain-text scores (1–5) for Innovation, Impact, and Feasibility.
- Populates the `candidate_portfolio` field of the PCD with all candidates, their scores, and any Grill-Me refinement notes.
- Presents a candidate review UI in the Phase Workspace: browse candidates, compare scores, and optionally **remix** — combine elements from two or more candidates into a new hybrid direction. Any remix is recorded with its component sources.
- Once you select a final direction (original candidate or remix), the app records it in `chosen_direction` with your stated rationale.
- Generates a **Downselection Verification**: checks the chosen direction against the competitive pre-mortem. If the chosen direction closely matches what the average team is predicted to build, the app surfaces a strong warning and asks you to confirm or reconsider before proceeding.

---

**Phase Output:** The full Ideation Record (SCAMPER, JTBD, Brainwriting), the candidate portfolio with scores, the seed challenge outcome (if run), the selected direction (original or remixed), and the downselection verification result. All written to the PCD.

**Phase Completion Requirement:** Steps 3b, 3c, and 3d must all be completed and pasted back before Step 3e is unlocked — the Ideation Record outputs are required context for the Divergent Candidate Generation prompt. Step 3e must be completed, final direction selected and confirmed, downselection verification passed or explicitly overridden with written rationale. Step 3a is optional and does not block completion.

---

### Phase 4 — Solution Design & Scope Lock

**When:** Triggered by Phase 3 completion. This phase ends at the one-way gate.

**Purpose:** Translate the chosen direction into three precise, actionable documents that your build tools can execute against. Make every important decision before writing a single line of code. Lock scope permanently.

---

#### Step 4a — Design Grill *(Optional)*

Before generating the three core documents and confirming scope lock, you may enable a Design Grill. This is a Grill-Me Mode interrogation of the "what" of the product — features, user flows, edge cases, and design decisions — conducted before any document is written, so that what gets locked is already stress-tested.

**App Tasks:**

- Generates a **Design Grill Prompt** in Grill-Me mode (Interactive or Auto-Grill). The external LLM is given the current PCD — chosen direction, problem analysis, competitive landscape, and judge personas — and is instructed to adopt the adversarial Socratic role, probing the scope using the six question types. It explicitly stays away from low-level technical implementation; the interrogation concerns product decisions, not architecture.
- Example question lines the LLM is instructed to pursue: "Why is this feature included in the MVP? What user need does it address that cannot be deferred?" "If a judge sees this flow, what might they find confusing?" "What happens if the API you depend on fails during the demo?" "Is there anything in this scope that exists because it seemed impressive rather than because it solves the problem?"
- The prompt can be run in Interactive mode (you answer the LLM's questions live) or Auto-Grill mode (the LLM simulates the full dialogue internally).

**Human Tasks:**

- Execute the Design Grill prompt in your external LLM.
- If running Interactive mode, engage with each question until the LLM signals completion.
- Paste back only the final Resolved Summary — a structured set of confirmed adjustments to the intended scope and product brief.

**App Tasks (post-intake):**

- Parses the Resolved Summary and surfaces the proposed scope adjustments for your review.
- You confirm or dismiss each adjustment. Confirmed adjustments are written into the PCD as amendments to the Chosen Direction before the three core documents are generated.
- Records the Design Grill outcome in the Phase Gate Log.

This step does not cross the Phase 4 gate. The gate is crossed only after the three documents are generated, reviewed, and scope lock is explicitly confirmed. Running the Design Grill makes the lock more defensible — it does not delay or bypass it.

If skipped, the three document prompts proceed directly from the current PCD state.

---

**App Tasks:**

- Reads the full PCD (incorporating any Design Grill adjustments if Step 4a was completed).
- Generates three document production prompts for execution in an external LLM:

**Prompt 1 — The Product Brief Prompt**
Instructs the external LLM to produce a concise, human-readable pitch document in Shape Up format covering: the problem statement (in the team's own framing, not the organizer's verbatim language), the appetite (what the team commits to building within the available time), the solution sketch (what it does, how it works, what makes it different), key user flows, explicit rabbit holes to avoid, and out-of-scope items. Written for humans — judges, mentors, teammates. Concise and persuasive.

**Prompt 2 — The Technical Implementation Document Prompt**
Instructs the external LLM to produce an exhaustive technical specification for use by Cursor and CLI coding tools. Covers: complete feature list with acceptance criteria per feature, MoSCoW prioritization, recommended tech stack with specific frameworks and libraries justified against the hackathon constraints (speed of development, free/cheap deployment, AI integration patterns), system architecture overview, data models and schema, API design or third-party integrations needed, a screen-by-screen breakdown (every screen in the application: its name, purpose, all content and UI elements present, all user interactions possible, all connections to other screens, and any AI-powered elements), edge cases to handle, explicit do-not-build list, and known technical risks.

**Prompt 3 — The Demo Script & Backup Plan Prompt**
Instructs the external LLM to produce a structured demo guide covering: the core user flow to demonstrate (the shortest path through the app that proves the solution works), what pre-loaded data to prepare before the demo, the exact sequence of screens to show, what to say at each step, what not to show (features that might break or distract), a time budget for the demo segment, and a full backup plan (what screenshots to prepare, what screen recording to produce, how to handle a live failure gracefully).

**Human Tasks:**

- Execute all three prompts in your external LLM.
- Paste all three outputs back into the app.
- Review all three documents carefully. The app surfaces a structured review checklist — has the MVP been clearly defined? Are all screens described? Is the out-of-scope list explicit? Is the backup plan concrete?
- Confirm scope lock. This is a deliberate, acknowledged action in the app — not a passive button click.

**App Tasks (post-intake):**

- Stores all three documents in the PCD under Scope Definition and Technical Direction.
- Marks the Phase 4 gate as crossed.
- Exports the Technical Implementation Document and Product Brief as downloadable files ready to be fed into Cursor as context.
- Begins surfacing the Presentation Prep track (Phase 6 can now begin in parallel with Phase 5).

**Phase Output:** The Product Brief, the Technical Implementation Document, the Demo Script & Backup Plan. The Phase 4 gate is crossed. Scope is locked.

**Phase Completion Requirement:** All three documents generated and reviewed, checklist completed, scope lock explicitly confirmed. Step 4a is optional and does not block completion — but if the Design Grill was run, its Resolved Summary must be pasted back and adjustments confirmed before the three document prompts are generated.

---

### The Addendum Protocol

**When:** Post-Phase 4 gate, only when a scope change is absolutely necessary.

**Trigger:** You initiate the Addendum Protocol from the app. The app immediately surfaces a warning: scope is locked, changes carry risk to build quality and timeline, this action is logged. You must provide a written justification before proceeding.

**Human Tasks:**

- Export the current project state from Cursor as a markdown file (a snapshot of what has been built, what is in progress, what is not started).
- Write a description of the proposed change: what is being added, modified, or removed, and why it is necessary.
- Paste both into the app.

**App Tasks:**

- Generates an Addendum Prompt for an external LLM: given the current build state, the existing Technical Implementation Document, and the proposed change, produce a concise addendum that specifies only what changes, what it impacts in the existing architecture, and what the revised do-not-build list looks like. The addendum does not rewrite the original TID — it annotates it.
- After you paste back the result, parses it and adds it to the PCD with a timestamp, your written justification, and a flag marking it as a post-gate change.
- Updates the Phase Gate Log.

**Output:** A TID Addendum document. The original TID is preserved. The addendum is appended. The change is logged permanently.

---

### Phase 5 — Build

**When:** Triggered by Phase 4 gate crossing. Runs until submission deadline.

**Purpose:** Execute the build against the Technical Implementation Document. Startup's active role shrinks here — Cursor and CLI tools take over. But Startup remains open as a reference and a progress tracker.

**App Role:**

- Maintains a **Build Checklist** derived from the Technical Implementation Document: each feature from the MoSCoW list, each screen from the screen breakdown, the demo preparation items, and the backup plan preparation items. You check items off as they are completed.
- Tracks time against the submission deadline and surfaces warnings at key intervals (50% of time elapsed, 75% elapsed, 90% elapsed).
- Provides instant access to any PCD section for reference during build — you should never need to leave the app to find a decision that was made.
- Hosts the Addendum Protocol if needed.
- Runs the Presentation Prep track (Phase 6) in parallel.

**Human Tasks:**

- Build the product using Cursor / CLI tools with the TID as primary context.
- Check off build checklist items as completed.
- Prepare pre-loaded demo data.
- Prepare backup demo materials (screenshots, screen recording).

**Phase Output:** A working product, demo data, and backup materials.

---

### Phase 6 — Pitch & Presentation

**When:** Can begin immediately after Phase 4 gate crossing, runs in parallel with Phase 5.

**Purpose:** Translate the Solution Package into a winning pitch. Optimize every element of the presentation for the specific judges and rubric identified in Phase 1.

**App Tasks:**

- Reads the full PCD — judge personas, active rubric, Product Brief, chosen direction, and differentiation rationale.
- Generates a **Pitch Storyboard Prompt**: instructs an external LLM to produce a scene-by-scene breakdown of the full pitch, structured using the AIDA framework (Attention, Interest, Desire, Action), timed to the hackathon's pitch duration, and calibrated to the judge personas. Each scene includes: time window, narrative talking points, recommended visual or slide content, data or metric emphasis, and emotional/logical hook.
- Generates a **Pitch Review Prompt**: instructs an external LLM to simulate the judge panel (one persona per judge type identified in Phase 1) reviewing the pitch storyboard. For each judge persona, surfaces the top three likely objections, explains why each objection could cost points, and recommends a specific change to the pitch to preempt it.
- Generates a **Q&A Prep Prompt**: instructs an external LLM to generate the twenty most likely judge questions, sorted by judge persona, with recommended answers and one-sentence fallback answers for questions you can't fully answer.

**Human Tasks:**

- Execute all three prompts in your external LLM.
- Paste results back into the app.
- Build the actual pitch deck using the storyboard as the blueprint.
- Rehearse using the Q&A prep document.

**App Tasks (post-intake):**

- Stores all three presentation documents in the PCD.
- Surfaces a Presentation Readiness Checklist: pitch deck complete, demo rehearsed, demo data loaded, backup materials ready, Q&A prep reviewed.

---

#### Optional — Judge Q&A Drill

In addition to the static Q&A Prep document, you may run a live mock-judge drill. Where the Q&A Prep prompt generates a prepared list to study, the Judge Q&A Drill puts you in the room — or simulates it.

**App Tasks:**

- Generates a **Judge Q&A Drill Prompt** in Grill-Me mode (Interactive or Auto-Grill). The external LLM is instructed to embody the exact judge personas constructed in Phase 1 — their professional backgrounds, industry lenses, known values, and established skepticisms — and to fire questions one at a time, just as a real panel would. It does not produce a list; it conducts an interrogation. Each question follows from the previous answer (or simulated answer), pressing on any weak response rather than moving on neutrally.
- The prompt draws on the full Phase 6 PCD context: pitch storyboard, pitch review objections, product brief, and chosen direction, so the LLM's questions are calibrated to what the pitch actually claims.

**Human Tasks (Interactive mode):**

- Execute the Judge Q&A Drill prompt in your external LLM.
- Answer each question as you would in the actual presentation.
- After the session ends, paste back the Resolved Summary — a sharpened Q&A document capturing the questions asked, the answers that held, and any refined responses for answers that did not.

**Auto-Grill mode:**

- The LLM simulates both the judge panel and the team's responses, challenges any answer it deems weak, and produces a refined Q&A document without requiring your live participation. Best used when time is short or as a pre-check before running Interactive mode.

**App Tasks (post-intake):**

- Accepts the Resolved Summary and writes it to the PCD as either a replacement or supplement to the static Q&A Prep document (your choice at intake).
- Updates the Presentation Readiness Checklist to mark the drill as completed.

This drill is optional and can be run at any point after Phase 6 prompts are complete, in parallel with Phase 5 build. It does not block submission.

---

**Phase Output:** Pitch storyboard, pitch review with objection preemptions, Q&A prep document, and — if the drill was run — a sharpened Q&A Resolved Summary. Presentation readiness checklist.

---

### Phase 7 — Post-Hackathon Debrief

**When:** Triggered immediately after the event ends, win or lose.

**Purpose:** Extract every useful signal from the hackathon and feed it back into the standing infrastructure. This is how the system compounds — every hackathon you complete makes the next one materially better.

**App Tasks:**

- Generates a **Debrief Prompt**: a structured retrospective prompt covering what went well in each phase, what went wrong, what was discovered too late, what the judges said (if feedback was given), and what would be done differently.
- After paste-back, generates specific update instructions for Phase 0 infrastructure: what components should be added or improved in the boilerplate, what entries to add to the Judge & Organizer Database, what to add to the Winning Solutions Archive.

**Human Tasks:**

- Execute the debrief prompt in your external LLM.
- Paste back into the app.
- Implement the boilerplate updates.
- Add new judge and organizer entries to the database.
- Update the Winning Solutions Archive.

**Phase Output:** Debrief document, structured update instructions for Phase 0 infrastructure. PCD archived with the event record.

---

## The App: Feature Overview

The following describes the key functional areas of the Startup application.

---

### Project Dashboard

The home screen of the app. Shows all active and archived projects. Each project displays: hackathon name, current phase, phase completion status, time remaining to deadline (if set), and a quick-access link to the PCD.

Creating a new project initiates Phase 1 intake.

---

### Phase Navigator

A persistent sidebar or header present throughout every phase. Displays all phases as nodes in a state machine. The current phase is highlighted. Completed phases are marked. The Phase 4 gate is visually distinct — it has a lock icon that changes state when crossed.

Backward navigation is available for all phases before Phase 4. Clicking back to an earlier phase triggers a reconciliation check: the app identifies which PCD sections were populated after that phase and flags them as potentially stale, asking you to confirm whether they need to be revisited. You can dismiss the flag with acknowledgment.

Phase 4 and beyond cannot be navigated backward. The Addendum Protocol is the only path to changes.

---

### Phase Workspace

Each phase has a dedicated workspace containing:

- **Phase Briefing**: a plain-language explanation of what this phase is for, what it produces, and why it matters to winning.
- **Completion Checklist**: the required steps for this phase. Startup tracks completion and warns before you attempt to advance without completing required items.
- **Prompt Generator**: the core function. Based on the current PCD state and the active phase, Startup generates the appropriate prompts. Prompts are displayed in a formatted, copyable block with a one-click copy button. Each prompt is labeled with its purpose and the recommended LLM to use for it.
- **Research Intake Module**: a structured paste-back interface. Each intake field is labeled for what it expects (e.g., "SCAMPER Ideation Output", "JTBD Analysis", "Competitive Pre-mortem Analysis", "Design Grill Resolved Summary"). After pasting, you confirm and Startup processes the input, extracts structured data, updates the PCD, and shows you a summary of what was added.
- **Phase Output Preview**: a read-only view of all PCD sections populated by this phase, so you can see what the phase has produced before advancing.

When a prompt is generated in Grill-Me mode, the Prompt Generator also surfaces an optional **Export Full Handoff Package** button. This exports a single markdown file containing the current full PCD plus the External Context Notes field. You upload this file directly to the external LLM's web interface alongside the generated prompt, giving the LLM complete project context for the duration of a long Grill-Me dialogue without requiring Startup to manage live chat history. The handoff package is a snapshot — it does not update during the session. When the session ends and you paste back the Resolved Summary, the PCD is updated in the normal way.

---

### Project Context Document Viewer

A full-page view of the PCD in its current state. Readable, structured, and exportable as a markdown file at any time. Sections are collapsible. Each section shows when it was last updated and which phase produced it.

This is the document you hand to Cursor as master context. It is also the document that grows into the client brief in the real-world adaptation.

---

### Document Export Center

A dedicated export interface for the Phase 4 outputs. Provides individually downloadable files for: the Product Brief, the Technical Implementation Document, and the Demo Script & Backup Plan. Also provides a full PCD export. Files are formatted as clean markdown, ready for direct use in Cursor.

---

### Boilerplate Library

A browsable, tagged library of reusable components and templates maintained from Phase 0. Components are tagged by type (frontend UI, backend architecture, AI integration pattern, deployment config) and by past usage (which hackathon it was used in, how well it worked). New components can be added at any time. Phase 4 will reference relevant components from this library when generating the Technical Implementation Document prompt.

---

### Judge & Organizer Database

A searchable, persistent database of hackathon judges, organizers, and sponsors. Each entry contains: name, organization, professional background, events appeared at, observed judging tendencies, and any notes. Entries are created manually or prompted by the app during Phase 7 debrief. The database is automatically queried during Phase 1 when a new hackathon is registered.

---

### Addendum Protocol Interface

Accessible only after Phase 4 gate is crossed. Entry requires written justification. Interface accepts: the current Cursor project state file (markdown), the proposed change description, and produces the TID Addendum. All addenda are logged with timestamps and justifications in the Phase Gate Log.

---

### Settings & Configuration

- LLM preferences: which external LLM to recommend for which prompt types (configurable based on your current subscriptions and preferences).
- Gemini Flash API key configuration for the app's internal engine.
- Default rubric editor: customize the fallback judging rubric used when organizers haven't released criteria.
- Deadline notifications: configure warning intervals for Phase 5 time tracking.
- Export format preferences.
- Grill-Me default mode: set your preferred default for the Off / Interactive / Auto toggle (applies across all eligible prompts; can always be overridden per prompt in the Phase Workspace).

---

## Workflow Summary

```
[ PHASE 0: Pre-Hackathon Readiness ] ←→ ongoing maintenance
           ↓
[ PHASE 1: Intelligence Gathering ]
  Input: Hackathon details
  Output: Intelligence Brief, Judge Personas, Active Rubric
           ↓
[ PHASE 2: Problem Deconstruction ]
  Input: Brief + Phase 1 PCD
  Output: Problem Analysis, Competitive Pre-mortem, User Personas
  HARD REQUIREMENT: Competitive Pre-mortem must be complete
           ↓
[ PHASE 3: Differentiated Ideation ]
  Input: Phase 1+2 PCD
  Step 3a (optional): Idea Seed Challenge → Grill-Me verdict on preconceived idea
  Step 3b: SCAMPER Ideation → multi-persona exploration across seven dimensions
  Step 3c: Jobs-to-Be-Done Analysis → unmet job gaps and differentiation ground
  Step 3d: Brainwriting Round → 5 conventional + 5 unconventional ideas
  Step 3e: Divergent Candidate Generation → 3–5 archetyped candidates using 3b–3d as context, optional Grill-Me refinement
  Output: Ideation Record, Candidate Portfolio, Remix option, Chosen Direction
  USER RESEARCH intake can arrive here or Phase 2
           ↓
[ PHASE 4: Solution Design & Scope Lock ] ← ONE-WAY GATE
  Input: Full PCD to date
  Step 4a (optional): Design Grill → Grill-Me stress-test of scope before documents are written
  Output: Product Brief + Technical Implementation Doc + Demo Script
  GATE CROSSED: scope is locked
           ↓
[ ADDENDUM PROTOCOL ] ← emergency only, logged, discouraged
           ↓
[ PHASE 5: Build ]               [ PHASE 6: Pitch Prep ] ← parallel
  Cursor / CLI executes TID        Input: PCD + Judge Personas
  App tracks checklist             Output: Storyboard, Review, Q&A Prep
                                   Optional: Judge Q&A Drill (Grill-Me)
           ↓                               ↓
                    [ SUBMISSION ]
                           ↓
              [ PHASE 7: Debrief ]
                Output: Debrief Doc, Phase 0 Updates
```

---

## The Prompting Architecture

Every prompt Startup generates follows a consistent structure, regardless of phase:

1. **Role assignment** — the LLM is told what expert it is embodying for this task.
2. **Context injection** — the relevant sections of the PCD are embedded. The app selects only what is needed for the current task to keep prompts focused.
3. **Task specification** — precise instructions on what to produce, in what format, at what depth.
4. **Constraints** — explicit guidance on what to avoid (over-generalization, feature overlap with the competitive pre-mortem, etc.).
5. **Output schema** — a defined structure for the output so the app can reliably parse and ingest it.

This architecture means that as the PCD grows richer, the prompts automatically become more specific and more grounded. The system improves itself as you use it.

When a prompt is toggled into Grill-Me mode (see Optional Grill-Me Mode below), components (3) and (5) are modified: the task specification replaces the standard "produce X analysis" instruction with a set of Socratic interviewing instructions that define the adversarial role, the six question types, and the pacing rules; and the output schema adds a placeholder for the Resolved Summary while optionally including a marker block containing the dialogue instructions, so the external LLM can reference them throughout a long interactive session.

---

## Optional Grill-Me Mode

Standard prompts in Startup follow a one-shot generation model: the external LLM receives the full context and task, produces a structured output, and you paste it back. For most phases under time pressure, this is the right default. But there are moments where the real bottleneck is not output quality but assumption quality — where committing to an output without first stress-testing the thinking behind it is the mistake. Grill-Me exists for those moments. It appears at four specific points in the workflow: the Idea Seed Challenge in Phase 3 (always on), the Divergent Candidate Generation refinement in Phase 3 (optional), the Design Grill in Phase 4 (optional, before scope lock), and the Judge Q&A Drill in Phase 6 (optional, before submission). The toggle is also available on Phase 2 prompts for users who want to interrogate their problem framing before moving to ideation.

Grill-Me mode transforms the external LLM from a generator into an adversarial Socratic interrogator. Instead of producing an analysis, it stress-tests your thinking — probing every branch of the project's design tree, resolving dependencies one by one, and refusing to accept vague or untested answers. The goal is not to produce output faster; it is to forge a genuine shared understanding between you and the AI by forcing you to articulate the "what" and the "why" at every level. The output that finally gets pasted into Startup is harder, cleaner, and better-grounded than anything a one-shot prompt could produce.

### How Grill-Me Challenges Assumptions

When a prompt is in Grill-Me mode, the external LLM adopts the role of a skeptical critic — a Gatekeeper — rather than a supportive assistant. It employs six types of Socratic questions across the session:

1. **Clarification** — "What do you mean by 'user-friendly' in this context?"
2. **Probing Assumptions** — "You assume users will pay for a premium tier. What evidence supports that?"
3. **Probing Reasons** — "Why is real-time sync essential for the MVP version?"
4. **Exploring Viewpoints** — "How would a non-technical judge evaluate this demo?"
5. **Examining Implications** — "If the user goes offline during a sync, what happens to the data?"
6. **Meta-questioning** — "Why do you think we've focused so much on the dashboard and not the input flow?"

The LLM asks one question at a time (in Interactive mode), preventing cognitive overload and allowing each branch to be resolved before the next opens. It does not accept vague answers. It deliberately surfaces hidden risks, weak logic, and blind spots. The session continues until all lines of questioning are resolved, at which point the LLM produces a final **Resolved Summary** — a structured output that conforms to the standard output template for that phase. This Resolved Summary, and only this, is what you paste back into the Research Intake Module. The full Q&A transcript is not stored in the PCD, keeping it lean and decision-focused.

### Two Sub-Modes

**Interactive Grill** — The external LLM asks you one question at a time and you respond directly in its chat interface. The back-and-forth continues until the LLM signals it has exhausted its lines of questioning, then it produces the Resolved Summary. Best used when you want to actively participate in the interrogation and refine your thinking through dialogue. Slower but more thorough.

**Auto-Grill (Auto-Simulate)** — The LLM simulates both sides of the conversation internally. It generates questions, produces plausible answers, challenges them, pushes back on its own answers, and iterates until a Resolved Summary is produced — all without requiring your input. This preserves the adversarial self-critique without the time cost of a live dialogue. Best used when you are confident in your current thinking and want the stress-test to run on its own, surfacing only the gaps it finds.

### How to Toggle It

Each eligible prompt in the Phase Workspace has a three-position toggle: **Off / Interactive / Auto**. Off is the default. When toggled to Interactive or Auto, Startup modifies the generated prompt before displaying it for copy — replacing the standard task specification with Socratic interviewing instructions, specifying which question types to use, setting the pacing rules (one question at a time for Interactive, free simulation for Auto), and updating the output schema to expect a Resolved Summary. The toggle does not change the context injection or constraints — only the task specification and output schema are altered.

You can always switch a toggle back to Off and use the standard one-shot generation. Grill-Me is an option, not an obligation.

---

## Adaptability: Beyond Hackathons

The Startup workflow maps cleanly to real-world client engagements with the following substitutions:

| Hackathon Context         | Client Engagement Context    |
| ------------------------- | ---------------------------- |
| Hackathon brief           | Client problem brief         |
| Judging rubric            | Client success criteria      |
| Judge personas            | Client stakeholder personas  |
| Competitive pre-mortem    | Market / competitor analysis |
| Phase 4 gate / scope lock | Statement of Work sign-off   |
| Pitch & presentation      | Client proposal / demo       |
| Post-hackathon debrief    | Project retrospective        |

The Project Context Document becomes the living project brief. The Technical Implementation Document becomes the technical specification. The Product Brief becomes the client-facing proposal. The core discipline — intelligence before ideation, differentiation before design, scope lock before build — is as valuable in client work as it is in hackathons.

Startup is built for hackathons. It is structured for everything else.

---

## Stretch Goals

The following are explicitly out of scope for the MVP but designed into the system's foundations so they are addable without architectural rewrites:

- **Team collaboration mode** — shared project state, role-based views, real-time or async updates to the PCD.
- **Autonomous deep research** — replacing the Gemini/Perplexity handoff with in-app web research.
- **Claude Code / CLI integration** — feeding the TID directly into a CLI coding agent from within the app.
- **Richer LLM routing** — automatic selection of the optimal LLM per prompt type based on task complexity and cost.
- **Analytics layer** — tracking win rate, which phases correlate with better outcomes, which prompt types produce the best outputs.
- **Client engagement mode** — dedicated configuration adapting the workflow for professional client work.

---

*Document version: 1.5 — Phase 3 restructured: SCAMPER (Step 3b), JTBD (Step 3c), and Brainwriting (Step 3d) restored as sequential subphases; Divergent Candidate Generation renumbered to Step 3e and updated to consume Steps 3b–3d outputs as grounding context. Workflow Summary, PCD Ideation Record, and Research Intake Module example updated accordingly.*
