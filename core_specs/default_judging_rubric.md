# Default Judging Rubric

This is the fallback rubric activated by Startup at Phase 1 the moment a project is created. It is **always live** during Phase 1 — even if no judging criteria have been announced yet — so that downstream phases (ideation scoring, pitch calibration, demo emphasis) always have a rubric to ground against.

When official judging criteria are released, the user resolves the conflict via one of three actions:

1. **Replace** — official criteria wholesale supersede the default. The active rubric becomes a verbatim mapping of the official criteria.
2. **Supplement** — official criteria are partial; defaults fill the gaps. Both sources are merged, with weights normalized to sum to 1.0.
3. **Keep default** — official criteria are vague or absent and the default is judged a closer fit. The user makes this call explicitly; the system logs it.

The five default criteria below are derived from the universal patterns observable across hackathons regardless of organizer type. Weights sum to **1.0**. Sub-questions are the questions a judge in that category typically asks themselves while watching a pitch.

This file is loaded into `Section_ActiveJudgingRubric.data.criteria` at project creation time.

---

## Criterion 1 — Innovation & Novelty

**Weight: 0.20**

**Description:**
Measures the degree to which the solution represents a genuinely novel approach — either to the problem itself, the user it serves, the technology applied, or the business model. Innovation is judged relative to the predicted average solution for this brief, not in absolute terms. A familiar technology applied to a problem nobody else has framed correctly can score higher here than a flashy technology applied to a well-known problem.

**Judge Questions:**
- Have I seen this exact solution today already? Yesterday? At the last hackathon?
- Is the novelty in the problem framing, the solution mechanism, or the application context?
- Does the team articulate *why* their approach is different, or do they assume I'll see it?
- Is this novel in a way that matters, or novel in a way that's just unusual?
- Would a domain expert say this is a new angle, or that this is a beginner's misunderstanding of an old angle?

---

## Criterion 2 — Problem-Solution Fit

**Weight: 0.25**

**Description:**
Measures how directly and effectively the solution addresses the actual problem identified in the brief, and how well it serves the real users affected by that problem. A solution can be novel and technically excellent and still fail this criterion if it solves an adjacent problem or serves a different user than the brief specifies. This is the criterion most commonly underweighted by teams who fall in love with their idea before validating its fit.

**Judge Questions:**
- Does the solution actually solve the problem in the brief, or a related problem the team found more interesting?
- Is there evidence the team understands the real users — pain points, contexts, constraints?
- When the team describes who this is for, can I picture that person clearly?
- Does the solution address the root cause or a downstream symptom?
- If this product existed, would the affected users actually use it?

---

## Criterion 3 — Technical Soundness

**Weight: 0.20**

**Description:**
Measures the quality, depth, and integrity of the technical execution. Includes appropriate technology choices, working core functionality, sensible architecture, and absence of obvious cut corners that compromise the demonstration. Technical soundness is not about complexity — a simple working solution scores higher than a complex broken one. Judges with technical backgrounds weight this higher; non-technical judges still notice when something obviously breaks.

**Judge Questions:**
- Did the demo work, or did the team have to apologize and explain?
- Are the technology choices justified by the problem, or did the team chase buzzwords?
- Does the architecture suggest the team thought about scale, edge cases, and failure modes — or only the happy path?
- If I asked how a specific feature works under the hood, would the team have a real answer?
- Is the AI/ML/data layer (if present) genuinely doing work, or is it a wrapper around something trivial?

---

## Criterion 4 — Feasibility & Viability

**Weight: 0.15**

**Description:**
Measures whether the solution could plausibly exist beyond the hackathon — as a real product, deployment, or initiative. Includes business viability (is there a path to sustainability?), operational viability (could this be built and maintained?), and adoption viability (could the target users actually access and use it?). This criterion separates demos from prototypes-of-real-things.

**Judge Questions:**
- If this team got funding tomorrow, could they ship this within a year?
- Who pays for this — the user, an institution, an advertiser, a grant — and is that path realistic?
- Are the assumptions about user adoption based on data or wishful thinking?
- What does the regulatory or compliance picture look like, and has the team thought about it?
- Does the team understand what it would actually cost to operate this?

---

## Criterion 5 — Presentation & Communication

**Weight: 0.20**

**Description:**
Measures how effectively the team conveys their work in the pitch and demo. Includes narrative clarity, demo quality, pitch pacing, ability to handle questions, and the polish of supporting materials. This is the criterion most often dismissed as "soft" by technical teams and most often decisive in close calls. A clear pitch of a 70%-quality solution frequently outscores a muddled pitch of a 90%-quality solution.

**Judge Questions:**
- After 30 seconds, did I understand what this team built and why?
- Did the demo show me the thing actually working, or did the team narrate around a static screen?
- Could the team answer my questions without flinching, or did they freeze?
- Did the pitch end at a memorable moment, or did it trail off into apologies?
- If I had to summarize this team's work in one sentence to my fellow judges right now, could I?

---

## Weighting Summary

| Criterion | Weight |
|---|---|
| Innovation & Novelty | 0.20 |
| Problem-Solution Fit | 0.25 |
| Technical Soundness | 0.20 |
| Feasibility & Viability | 0.15 |
| Presentation & Communication | 0.20 |
| **Total** | **1.00** |

Problem-Solution Fit carries the highest weight because it is the most common failure mode across hackathons regardless of organizer type. Feasibility is weighted lowest because hackathons explicitly tolerate low-feasibility demonstrations more than other criteria.

---

## Editing the Default Rubric

The default rubric is editable via Settings (JSON-configurable). Validation rules:

- All criterion weights must sum to exactly 1.0 (within 0.001 tolerance for floating-point).
- Each criterion must have a name, description, weight, and at least one judge question.
- Criterion names must be unique within the rubric.

A user-edited default rubric is stored as a Vault-level setting and applied to all new projects from that point forward. Existing projects retain whatever rubric was active when they were created.

---

*This file is the source of truth for the default rubric loaded at project creation. Any changes to the canonical defaults — not user customizations — should be made here and committed.*
