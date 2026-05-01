# Startup — Session Notes (2026-04-29, 1:30 PM)

> Compressed handoff of this session. Drop into `external_context_notes` or use as a primer for the next LLM session.

## What this session accomplished

1. **Authored full content for the four v1.5 Grill-Me prompts** in `PROMPTS.md` (1991 → 2497 lines, +506). Each entry follows the existing format: metadata, role assignment, task specification, illustrative output template, negative constraints, rationale. Placed in workflow order:
   - `phase_3_idea_seed_challenge` → before SCAMPER (Step 3a)
   - `phase_3_divergent_candidates` → between Brainwriting and Differentiation Stress Test (Step 3e)
   - `phase_4_design_grill` → before Product Brief (Step 4a)
   - `phase_6_judge_qa_drill` → after QA Prep (Phase 6 optional drill)

2. **Updated `divergent_candidates_intake` in spec §6.2** (1820 → 1832 lines, +12) to match the new prompt. Removed the directive telling Flash to generate ULIDs and set `is_remix`/`remix_source_ids` defaults. Added subsection **"Post-extract enrichment for `divergent_candidates_intake`"** documenting the three-step flow: Flash extracts (without app-managed fields) → app enriches (`id` via `ulid` npm, `is_remix: false`, `remix_source_ids: []`) → schema validation runs on enriched object. Established as the canonical pattern for any future intake needing app-minted identifiers.

## Resolved from prior session's to-do list

- ✅ **Task 2** (Author full role_assignment and task_specification_template for the four new prompts) — DONE. Full text lives in `PROMPTS.md` per the spec's explicit guidance that prompt content lives in the prompts module, not the spec body.

## Still open (carried forward)

- **Task 1** — `challenge_outcome` enum drift. **The drift is 4-way, not 3-way as prior notes recorded.** Schema (`pcd_schema.json` line 766) actually uses `["adopt_as_seed", "refined", "rejected"]` — NOT `["adopted", "refined", "rejected"]`. The four disagreeing forms across the project:
  - Schema: `["adopt_as_seed", "refined", "rejected"]`
  - Spec §6.2 `idea_seed_challenge_intake` line 951: `["adopt_as_seed", "refine_and_adopt", "reject"]`
  - Spec §4.7 line 685: `adopt_as_seed | refine_and_adopt | reject`
  - PROMPTS.md `phase_3_idea_seed_challenge` (this session): `adopt_as_seed | refined | rejected` — written against schema since schema is canonical
  - Recommendation unchanged: adopt the schema's form and propagate to spec §6.2 and §4.7.
- **Task 3** — define `chosenDirectionAdjustmentsSchema`. The new `phase_4_design_grill` prompt uses a `field_path` syntax with array-suffix convention (`key_differentiators[+]`, `[-]`, or unmodified). The schema must match this syntax when authored.
- **Task 4** — `archetypes_enforced` default decision.
- **Task 5** — begin Milestone 1 build per §11.2.

## New items flagged this session

1. **`ContextSelector` needs a "runtime input" mode.** The new `phase_3_idea_seed_challenge` prompt uses `{seed_idea_text}` — a token filled at generation time from a textarea, not from a PCD path. The current `ContextSelector` design in §4.3 only handles PCD-resolved tokens. Add an inclusion rule (`'runtime_input'` or similar) plus a generator path that pulls from the prompt-form's user input.
2. **`phase_4_design_grill.target_intake_schema_path` ambiguity.** Spec stub points to `sections.chosen_direction.data`, but the prompt emits adjustment records conforming to `chosenDirectionAdjustmentsSchema` (auxiliary) which then writes through to `chosen_direction` *after* user confirmation. This is a two-stage write. The `target_intake_schema_path` field semantics should be clarified during task 3 — either with a second value to express the auxiliary-then-target pattern, or by inlining the auxiliary schema differently.
3. **`phase_6_judge_qa_drill` enforces non-null `associated_persona`.** Schema (`Section_QAPrep`) allows `associated_persona: null`; the new prompt forbids null. Prompt-level rule not enforceable by schema. Intake validator should flag (not fail) null-persona entries from this specific intake.

## Reaffirmed design invariants

Untouched and remain so:
- JSON canonical PCD; Markdown rendered, never read back.
- Local-first IndexedDB (Dexie); Web Crypto AES-GCM with PBKDF2.
- Cloudflare Worker as stateless proxy; no backend database.
- Gemini Flash used only for intake extraction in MVP.
- Phase 4 gate is one-way; Design Grill (Step 4a) does not cross it.
- Competitive Pre-mortem is the only hard block.
- Grill-Me modifies only component [4] task specification at generation time; the Resolved Summary conforms to the same intake schema as the standard one-shot.
- Pre-existing Phase 3 prompts (SCAMPER, JTBD, Brainwriting, Stress Test, Downselection) retained verbatim.

## Next likely tasks

In order, if continuing pre-build:

1. Resolve the 4-way `challenge_outcome` enum drift (task 1). Recommended canonical: `["adopt_as_seed", "refined", "rejected"]`. Propagate to spec §6.2 line 951 and §4.7 line 685.
2. Define `chosenDirectionAdjustmentsSchema` (task 3) — align with the `field_path` syntax used by `phase_4_design_grill`; resolve the `target_intake_schema_path` ambiguity at the same time.
3. Decide `archetypes_enforced` default (task 4) — recommend `true`, add `"default": true` to schema.
4. Extend `ContextSelector` for runtime-input tokens (new this session).
5. Begin Milestone 1 build per §11.2.
