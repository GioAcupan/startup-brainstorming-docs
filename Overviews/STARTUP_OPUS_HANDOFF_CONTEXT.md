# STARTUP — Compressed Chat Context for Opus Handoff (v1.1, HITL augmented)

This document summarizes the full design conversation that produced the Startup workflow and app overview. It now includes the later human-in-the-loop (HITL) enhancements requested by the user. Use it as a primer to understand the decisions made, the reasoning behind them, and the constraints to respect when producing the implementation specification.

The primary output document is: `STARTUP_WORKFLOW_AND_APP_OVERVIEW.md`

---

## Who the User Is

- CS student in the Philippines, early-stage coder (C, Python data sci libraries, basic HTML/CSS). Self-described as rusty but improving.
- Has joined 2 hackathons so far, placed 4th in both. Primary goals: prize money and clout, with a secondary goal of eventually being good enough to review and contribute to code directly.
- Core team: himself (backend work, UX critique, AI tooling) and Julian (data engineering, pitching/presenting). Possibly adding Fergus as dedicated UI/UX. Other teammates recruited per hackathon.
- Cousin runs a business and has offered him a lead tech/AI role building client software — this is the real-world stretch goal that the workflow must eventually adapt to.
- Current AI toolset: Cursor (primary coding), Gemini (free trial, orchestration), Claude (paid monthly when needed), Perplexity Education Pro. Exploring CLI tools (Gemini CLI, Codex, free-claude-code via NVIDIA NIM).

---

## Why Startup Exists: The Pain Points It Solves

These were derived from his two hackathon experiences and drove every design decision:

1. **Domain intelligence arrived too late.** In the DOST hackathon, critical domain frameworks (What-So-What-Now-What, Data Maturity Lifecycle, Walk-Crawl-Run-Fly) were revealed by mentors near the end of build — after architecture was set. The fix: a dedicated intelligence-gathering phase before ideation.

2. **No deliberate differentiation.** When problem statements are specific, teams converge. They knew differentiation had to happen at the problem-understanding level, not the feature level, but had no systematic method. The fix: the Competitive Pre-mortem as a hard requirement in Phase 2.

3. **Mid-build pivots killed momentum.** User interviews and mentor feedback arrived during development, forcing a late adjustment that caused cramming and nearly missed the deadline. The fix: Phase 4 as a one-way scope lock gate.

4. **Judges didn't understand what was built.** In the DOST hackathon, judges thought they submitted a Figma prototype, not a working app. The fix: Demo Script + Backup Plan as a first-class Phase 4 output, and Pitch Review as a Phase 6 deliverable.

5. **Ideation-to-coding handoff was broken.** First hackathon: winging it, spaghetti code from ad-hoc AI generation. The fix: a PRD (Technical Implementation Document) as the structured handoff artifact to Cursor.

6. **Criteria arrived late or changed.** Hackathons are inconsistent about when they release judging criteria, and criteria can even change between qualifying and final rounds. The fix: a Default Judging Rubric that activates immediately and gets replaced/supplemented when official criteria arrive.

---

## Key Design Decisions and Their Rationale

### The Workflow is a State Machine, Not a Waterfall

Phases can be navigated backward when new information arrives (e.g., late user research, late criteria release). However, Phase 4 completion is a strict one-way gate. After Phase 4, the only path to scope changes is the Addendum Protocol, which is deliberately friction-heavy and logged.

**Rationale:** Flexibility is needed because hackathon information (criteria, judges, user access) arrives unpredictably. But unbounded flexibility causes the mid-build pivots that killed them in DOST. The gate creates a forcing function.

**Implementation note:** Backward transitions must trigger a reconciliation check — the app identifies which PCD sections were populated after the phase being returned to and flags them as potentially stale.

### The Two-Layer AI Architecture

Startup's internal engine is **Gemini Flash (free tier)**. It handles context management, prompt engineering, and output parsing/ingestion. It does NOT do heavy reasoning or verbose generation.

Heavy generation is handled by **external LLMs** (Claude, Gemini Pro, ChatGPT, Deepseek — user's choice per moment). The app generates an optimized, context-grounded prompt → user executes it in their preferred LLM → user pastes output back → app parses and extracts structured data into the PCD.

**Rationale:** Gemini Flash is unreliable for verbose, nuanced generation. The external LLM handoff keeps quality high while keeping the app free to run. It also keeps the human in the loop at every major reasoning step, which functions as a quality gate.

**Key implementation requirement:** The intake step must not raw-append outputs. Gemini Flash must extract structured information from pasted outputs and write it into the correct PCD schema fields.

### The Project Context Document (PCD) is the Spine

Everything the app knows lives in the PCD. Every prompt is grounded in it. Every output enriches it. It is a structured markdown file with a fixed schema (see overview doc for full section list). It is exportable at any time as a standalone file for use in Cursor.

**Rationale:** Provides persistent context across phases, across backward transitions, and across the external LLM handoffs. Also serves as the master context file handed to Cursor at build time.

### Phase 4 Produces Three Documents, Not One

The original PRD concept was split into three distinct artifacts with distinct audiences:

- **Product Brief** — human-readable, Shape Up format, for judges/mentors/teammates. Concise and persuasive.
- **Technical Implementation Document (TID)** — exhaustive, for Cursor/CLI. Includes full feature list with acceptance criteria, tech stack with justification, system architecture, data models, API specs, and critically a **screen-by-screen breakdown** (every screen: name, purpose, UI elements, interactions, connections to other screens, AI-powered elements).
- **Demo Script & Backup Plan** — what to show, in what order, pre-loaded data requirements, what not to show, and a full failure contingency (screenshots, screen recording).

**Rationale for screen-by-screen breakdown:** AI coding tools hallucinate UI when not given specifics. Describing every screen in detail constrains the AI's imagination productively and prevents UX drift.

### Judge Modeling is a First-Class Citizen

Judge personas are built in Phase 1, not as an afterthought. They are constructed from: organizer and sponsor type (strong signal for judge background), LinkedIn research (app generates search strategy, user executes in Gemini/Perplexity), historical judge database (persistent across hackathons), and past winner patterns.

**Rationale:** Judges are often not announced until close to submission. Building personas from proxy signals (organizer, sponsor, domain) allows judge-optimized design decisions to be made early, even without named judges.

**The persistent Judge & Organizer Database** is maintained across all hackathons (Phase 0) and auto-queried when a new project is created. It compounds in value over time.

### The Competitive Pre-mortem is the Only Hard Block

Of all the workflow steps, the Competitive Pre-mortem in Phase 2 is the only one that is a strict prerequisite for advancement — the app will not let you proceed without it.

**Rationale:** This is the step most teams skip, and skipping it is exactly why teams end up building the same thing as everyone else. Making it a hard block encodes the most important lesson from the DOST hackathon experience.

### Research Handoff Model Applies Universally

The same model used for market research (app generates questions, user runs them in Gemini/Perplexity, pastes back) applies to all external intelligence gathering: judge research, domain framework discovery, LinkedIn judge searches, past winner analysis. The app is the research director; external tools are the research executors.

### The Boilerplate Starter Kit is Part of the System

A pre-built library of reusable components (UI, backend architecture templates, deployment configs) maintained in Phase 0 and referenced in Phase 4 when generating the TID prompt. This is how build velocity compounds across hackathons — you don't start from zero every time.

### Free-First Cost Architecture

Design around the free constraint from the start. Gemini Flash free tier is the primary engine. The NVIDIA NIM / free-claude-code path is experimental and optional. Paid LLMs (Claude, Gemini Pro) are user-invoked manually at high-stakes moments, not integrated into the app's API calls. Paid features are a stretch goal for when there is an income stream.

---

## Human-in-the-Loop Ideation: The Grill-Me Mode and Divergent Candidate Selection

*These features were added after the initial design, in response to the user's desire for deeper human-AI collaboration during ideation. They emphasize "shared understanding," divergent thinking, and conversational challenge over purely automated prompt chains.*

### Optional "Grill-Me" Mode for Critical Phases

**What it is:**  
A togglable mode applied to specific prompts, particularly in Phases 2–4 and Phase 6. When active, the external LLM is instructed to conduct an **adversarial Socratic interrogation** instead of a single‑shot analysis. The LLM asks the user one question at a time, probing every branch of the design tree, until all assumptions are tested. The user can choose between **Interactive** (live Q&A) or **Auto‑Grill** (the LLM simulates both sides and self‑critiques). In either case, the final output is a **Resolved Summary** that captures the decisions reached.

**Why it matters:**  
The original workflow relied on the user simply running a prompt and pasting the result. This could lead to "blind vibe coding" — accepting AI output without true critical engagement. The Grill‑Me mode forces the human to articulate the *why* behind every choice, which:

- Builds a shared mental model between the user and the AI,
- Surfaces hidden risks and weak logic,
- Ensures that features and assumptions are truly understood, not just AI‑generated,
- Produces a stronger, more defensible direction because it has survived adversarial scrutiny.

The Auto‑Grill option preserves value even under time pressure: the LLM will still challenge its own output internally, delivering a more rigorous result than a plain one‑shot prompt.

### External Context Notes — Memory Across Sessions

**What it is:**  
A new, optional section in the Project Context Document (`external_context_notes`). The user can manually paste compressed summaries of past external LLM conversations (especially Grill‑Me dialogues) before starting a new session. The app can inject this field into the context of any new prompt.

**Why it matters:**  
Conversations with external LLMs are not persisted in the PCD. Over multiple handoffs, the powerful LLM can lose the thread. `external_context_notes` gives the user a lightweight way to carry forward the key insights and agreements without bloating the PCD or requiring the app to track chat history. It ensures the external LLM stays anchored even across many interactions.

### Phase 3 Overhaul — Idea Seed Challenge, Retained Pipeline, and Divergent Candidate Generation

The user explicitly requested that the original ideation pipeline (SCAMPER, JTBD, Brainwriting) remain in place. It is now **augmented** with two new capabilities:

**3a. Idea Seed Challenge (optional, before the pipeline)**  
If the user already has a team idea, they can enter it here. The system generates a Grill‑Me prompt that stress‑tests the idea against the hackathon profile, problem analysis, and competitive pre‑mortem. The LLM returns a verdict: adopt as seed, refine and adopt, or reject. The user then decides whether to seed the remaining ideation with this idea or discard it and proceed normally.

**3b. Divergent Candidate Generation (after the pipeline)**  
After SCAMPER, JTBD, and Brainwriting have produced a wide solution space, the system generates a **single** prompt for a multi‑turn external LLM session to synthesize everything into **3–5 radically different candidate app ideas**. Each candidate is scored (plain text) for Innovation, Impact, and Feasibility. The prompt enforces, by default, four diversity archetypes:

- **Safe Utility** – practical, high‑feasibility MVP,
- **Weird Gem** – unconventional, pushes boundaries,
- **Moonshot** – technically ambitious, using emerging tech,
- **Social Impact** – focused on marginalized or forgotten users.

The generation avoids any idea that overlaps with the competitive pre‑mortem's predicted "average team" solutions. The user can toggle archetypes off, but the default pushes for deliberate variety. After the candidates are pasted back, the user can refine them via further dialogue, "remix" features across candidates, and choose a final direction.

**Why this matters:**  
The original flow risked converging on a single, safe idea too early. By generating a **candidate portfolio**, the system:

- Embraces divergent thinking before convergence,
- Gives the team concrete options to discuss, remix, and combine,
- Protects against "premature gravity"—fixation on the first decent idea,
- Aligns directly with how creative teams work: exploring widely, then choosing deliberately.

Retaining the older pipeline means the new generation builds on rich ideation depth, not just a raw problem statement.

### Optional "Design Grill" Before Phase 4 Scope Lock

**What it is:**  
Before the user crosses the one‑way Phase 4 gate, they can invoke an optional Grill‑Me session that focuses on the *what* (features, user flows, design decisions) rather than low‑level technical implementation. The LLM, armed with the PCD and judge personas, questions the proposed scope.

**Why it matters:**  
It adds a final layer of human‑AI alignment before locking scope. The team can validate that every feature in the MVP truly serves a demonstrable purpose, anticipate judge skepticism, and remove unnecessary complexity—all without breaking the sanctity of the gate.

### Optional "Judge Q&A Drill" in Phase 6

**What it is:**  
In addition to the static Q&A Prep document, the user can run a live mock‑judge Grill‑Me. The external LLM embodies the judge personas from Phase 1 and fires questions one‑by‑one. In Auto mode, it simulates both sides and produces a refined Q&A document.

**Why it matters:**  
Pitch practice is often the weakest part of hackathon preparation. The drill turns the abstract judge profiles into a realistic interrogation, forcing the team to rehearse answers under pressure. It makes the final pitch far more resilient because the team has already heard—and answered—the hardest questions.

### Prescription and the Phase 4 Gate Remain Sacred

All these conversational features are **optional subphases** or **toggles**. The underlying prescriptive state machine, the hard requirement for the competitive pre‑mortem, and the irreversible Phase 4 scope lock are unchanged. Human‑in‑the‑loop collaboration is encouraged, but the system still enforces a disciplined path from research to locked scope.

---

## What the App Is NOT

- Not a generic AI assistant or chatbot.
- Not a code generator — it hands off to Cursor/CLI, it does not write code itself.
- Not an autonomous research agent — it directs research, the user executes it.
- Not a team collaboration tool in MVP — solo only, collaboration is a stretch goal.
- Not a live chat platform; all conversational interactions happen in external LLMs via handoff, never inside the app.

---

## Stretch Goals (Explicitly Out of Scope for MVP)

- Team collaboration / shared project state
- Autonomous deep research (replacing Gemini/Perplexity handoff)
- Claude Code / CLI direct integration from within the app
- Richer LLM routing (auto-select model per task)
- Analytics layer (win rate tracking, phase correlation analysis)
- Client engagement mode (adapted workflow for cousin's business context)

---



## Tone and Constraints for the Implementation Spec

- The user is a student, not a seasoned engineer. The spec should be buildable by someone learning system design, ideally with Cursor as the primary coding tool.
- Free and cheap infrastructure is a hard constraint, not a preference.
- The app will be used under time pressure during hackathons. UX must be fast, clear, and low-friction.
- Prescriptive over flexible — the app should tell the user what to do next, not ask them what they want to do.
- The human‑in‑the‑loop features are optional and must not slow down a user who wants to run the original, streamlined path.

---

*Handoff document version: 1.1 — Updated with HITL Grill-Me mode, divergent candidate generation, Design Grill, Judge Q&A Drill, and External Context Notes.*
