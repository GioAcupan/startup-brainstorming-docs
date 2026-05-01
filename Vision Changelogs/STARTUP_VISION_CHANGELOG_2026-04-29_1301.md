# STARTUP — Vision Changelog
### v1.0 → v1.5

---

## v1.1 — Grill-Me Mode & Cross-Session Context

**What changed:** Introduced the concept of adversarial Socratic interrogation as an optional mode the user can toggle on any eligible prompt. Instead of the external LLM generating output in one shot, it becomes a relentless questioner — probing assumptions, reasons, implications, and blind spots one question at a time before producing a final Resolved Summary. Two sub-modes: Interactive (user answers live) and Auto-Grill (LLM simulates both sides).

**Why it matters:** One-shot generation produces answers. Grill-Me produces *tested* answers. The mode exists because the biggest failure point in hackathon ideation is not weak output — it is unexamined assumptions getting baked into the project too early. Grill-Me forces those assumptions to the surface before they become expensive.

**Also added:** The `External Context Notes` PCD field — a free-form area where the user can paste compressed summaries of past LLM sessions to give future sessions a running memory. And the **Handoff Package** export — a single markdown file (full PCD + External Context Notes) that can be uploaded to an external LLM's interface to maintain context across a long Grill-Me dialogue without requiring Startup to manage live chat history.

---

## v1.2 — Phase 3 Redesigned: Human-Led Divergent Ideation

**What changed:** Phase 3 was restructured around two new subphases. First, an optional **Idea Seed Challenge** (Step 3a): if the user has a preconceived idea, they enter it here and it is immediately subjected to a Grill-Me interrogation before it is allowed to influence ideation. The verdict — Adopt, Refine and adopt, or Reject — is the user's call. Second, the sequential SCAMPER/JTBD/Brainwriting pipeline was replaced with a single **Divergent Candidate Generation** prompt (Step 3b at the time) that produces 3–5 radically different candidates structured around four enforced diversity archetypes: Safe Utility, Weird Gem, Moonshot, and Social Impact. After candidates are generated, the user can browse them, compare scores, and remix elements across candidates before selecting a final direction.

**Why it matters:** The old sequential pipeline produced a wide range of raw ideas but left the user to do all the synthesis mentally. The new structure makes the diversity explicit and enforced — the LLM cannot converge on similar candidates because the archetypes structurally prevent it. The Idea Seed Challenge addresses a real failure mode: users who enter ideation with a fixed idea they never truly interrogate, and end up building the thing they already wanted to build regardless of what the process surfaced.

**Also added:** `Candidate Portfolio` as a new top-level PCD field, distinct from the Ideation Record. Remix notes and component sources are tracked.

---

## v1.3 — Grill-Me Expanded to Phase 4 and Phase 6

**What changed:** Grill-Me Mode was extended to two more phases.

In **Phase 4**, an optional **Design Grill** (Step 4a) was added before the three core document prompts are generated. It interrogates the scope at the product-decision level — features, user flows, edge cases, what belongs in the MVP and what doesn't — without touching low-level technical implementation. Adjustments surfaced by the grill are reviewed and confirmed before the documents are written. The Phase 4 gate is unchanged: it is crossed only after all three documents are generated, reviewed, and scope lock is explicitly confirmed.

In **Phase 6**, an optional **Judge Q&A Drill** was added after the static Q&A Prep prompt. Instead of producing a list to study, it embodies the actual judge personas from Phase 1 and fires questions at the user one at a time, just as a real panel would. In Auto-Grill mode, it simulates both sides and produces a refined Q&A document. The drill can run in parallel with Phase 5 build and does not block submission.

**Why it matters:** The Design Grill addresses the gap between choosing a direction (Phase 3) and documenting it (Phase 4) — a moment where untested scope assumptions get quietly locked in. The Judge Q&A Drill addresses the difference between knowing your answers and being able to deliver them under adversarial questioning. A static Q&A list is preparation. The drill is rehearsal.

---

## v1.4 — Cohesion Pass

**What changed:** No new features. Six consistency fixes across the document to bring all sections into alignment with the changes introduced in v1.1–v1.3.

- Grill-Me Mode introduction rewritten to enumerate all four formal appearances in the workflow (Phase 3 Seed Challenge, Phase 3 candidate refinement, Phase 4 Design Grill, Phase 6 Q&A Drill) instead of vaguely pointing to Phases 2 and 3.
- Workflow Summary updated for Phase 4 and Phase 6 nodes to surface their optional subphases.
- Phase Workspace Research Intake Module example updated — removed stale reference to "SCAMPER Ideation Output" that no longer matched Phase 3's structure at the time.
- Judge Q&A Drill repositioned before Phase 6's Phase Output line (it had been placed after, breaking the structural pattern every other phase follows).
- Phase 4 Completion Requirement updated to explicitly acknowledge Step 4a and its sequencing rule.
- Settings & Configuration updated to include a Grill-Me default mode preference.

---

## v1.5 — SCAMPER, JTBD, and Brainwriting Restored as Subphases

**What changed:** The three structured ideation methods removed in v1.2 were restored — but with a different role than they had in v1.0. They are now Steps 3b (SCAMPER), 3c (JTBD), and 3d (Brainwriting), each as its own subphase with its own prompt, intake, and PCD write. The Divergent Candidate Generation prompt is renumbered to Step 3e and reframed as a **synthesis step**: it no longer generates candidates from scratch, but draws on the full Ideation Record produced in Steps 3b–3d as grounding context. Steps 3b–3d are required before Step 3e is unlocked.

**Why it matters:** In v1.2, removing the sequential methods made the Divergent Candidate Generation prompt do too much — it was being asked to explore the problem space and produce structured candidates in a single pass, without any prior ideation to draw from. Restoring SCAMPER, JTBD, and Brainwriting as upstream subphases solves this: by the time Step 3e runs, the problem space has already been mapped from multiple angles. The archetypes prompt becomes a synthesis of real exploration rather than a cold generation task. The quality ceiling of Step 3e is directly proportional to the richness of what Steps 3b–3d produced.

**The key design distinction from v1.0:** In the original, the sequential pipeline *was* the ideation — SCAMPER, JTBD, and Brainwriting produced the shortlist directly. In v1.5, they are *inputs* to the ideation. The Divergent Candidate Generation step is the actual convergence mechanism, and it is smarter for having the earlier outputs to work from.

---

*This document tracks vision-level changes only. For implementation details, refer to the full `STARTUP_WORKFLOW_AND_APP_OVERVIEW.md`.*
