# Startup — Session Notes (2026-04-29)

> Compressed handoff of this session for use in `external_context_notes` or as a primer for the next LLM session. Drop into the field, paraphrase down further if needed.

## What this session accomplished

Updated two project artifacts to reflect the v1.5 vision (Grill-Me Mode, Phase 3 substep restructure, External Context Notes, Handoff Package):

1. **`STARTUP_IMPLEMENTATION_SPEC.md`** — surgically edited from 1446 → 1794 lines. Foundational decisions (JSON canonical PCD, IndexedDB, encryption, Phase 4 gate semantics, Cloudflare Worker proxy) untouched.
2. **`pcd_schema.json`** — additive-only edits, 1268 → 1392 lines. Validated against Draft 2020-12 metaschema. Positive + negative tests confirm new constraints bite.

## Spec changes (locked in)

- **§2.3** — `external_context_notes` added to section list.
- **§2.6 (NEW)** — Documents the additive schema fragments: `team_idea_seed`, `candidate_portfolio` w/ `CandidateIdea` shape, `archetypes_enforced`, `selected_candidate_id`, `external_context_notes` envelope.
- **§4.2** — `PromptDefinition` extended with optional `grill_me_modes: ('interactive' | 'auto')[]`. Four new prompt entries added: `phase_3_idea_seed_challenge`, `phase_3_divergent_candidates`, `phase_4_design_grill`, `phase_6_judge_qa_drill`. SCAMPER/JTBD/Brainwriting/Stress Test/Downselection retained unchanged.
- **§4.3** — `ContextSelector.inclusion_rule` enum gained `'if_user_provided'` for the External Context Notes injection. Per-phase context-selection table updated for all new prompts and for SCAMPER/JTBD/Brainwriting now optionally reading `team_idea_seed`.
- **§4.5** — Prompt-generation flow now branches on Grill-Me, surfaces a Handoff Package export button when Grill-Me is active, and enforces Phase 3 ordering (`phase_3_divergent_candidates` is precondition-gated on at least one of SCAMPER/JTBD/Brainwriting being populated).
- **§4.7 (NEW)** — Full Grill-Me Mode definition. Two sub-modes (Interactive, Auto), wholesale task-spec rewrite, where Grill-Me appears (table), Phase Gate Log entries, explicit list of what Grill-Me does NOT change (output schema, Phase 4 gate, allowed transitions).
- **§5.2** — Phase 3 completion gate now requires ≥1 pipeline output AND `candidate_portfolio.candidates.length >= 3` AND non-empty `selected_idea`. Step 3a explicitly optional.
- **§5.3** — `external_context_notes: []` added to dependency graph as a leaf with no upstream deps (never auto-stale).
- **§6.2** — Four new `IntakeDefinition` entries: `idea_seed_challenge_intake`, `divergent_candidates_intake`, `design_grill_intake`, `judge_qa_drill_intake`. Notes on diff-preview semantics for each.
- **§11.2** — Build-order Milestone 6 extended with Grill-Me toggle UI, External Context Notes editor, Handoff Package export button, and the Phase 3 candidate-review/remix UI.

## Schema changes (locked in)

- `sections.required` and `sections.properties` — both extended with `external_context_notes`.
- `Section_IdeationRecord.data` gained `team_idea_seed`, `candidate_portfolio` (with `minItems: 3, maxItems: 5`), `archetypes_enforced`. Each `CandidateIdea` requires `id`, `name`, `description`, three `*_score` fields (1–5), `distinguishing_factors`. Optional fields: `archetype`, `is_remix`, `remix_source_ids`.
- `Section_ChosenDirection.data` gained optional `selected_candidate_id`.
- New `$defs/Section_ExternalContextNotes` — standard envelope, `data` typed as `string` (Markdown body).

## Open issue to resolve before further build

**Enum drift on `challenge_outcome`.** Three forms exist across the project documents:

- **Schema (this session):** `["adopted", "refined", "rejected"]` — written verbatim from the user's prompt for the schema task.
- **Implementation spec (this session):** `["adopt", "refine_and_adopt", "reject"]` — written from the workflow doc's prose ("Adopt as seed / Refine and adopt / Reject").
- **Workflow doc (v1.5):** verbal phrases "Adopt as seed", "Refine and adopt", "Reject".

The Flash extractor will fail intake if these don't match. **Action item:** pick one canonical form, propagate it across (a) the schema enum, (b) the `idea_seed_challenge_intake` extraction prompt template in §6.2 of the spec, (c) the `phase_3_idea_seed_challenge` prompt's task spec. Recommendation: use `["adopted", "refined", "rejected"]` from the schema since it's the canonical store and the constraint actually validates.

## Reaffirmed design invariants

These were treated as untouchable during this session and remain so:

- JSON canonical PCD; Markdown rendered, never read back as canonical.
- Local-first IndexedDB (Dexie); Web Crypto AES-GCM with PBKDF2.
- Cloudflare Worker as stateless proxy; no backend database.
- Gemini Flash used only for intake extraction in MVP — not for prompt assembly.
- Phase 4 gate is one-way; Design Grill (Step 4a) does not cross it; only the three documents + typed-phrase scope-lock confirmation does.
- Competitive Pre-mortem is the only hard block in the system.
- Grill-Me modifies only component [4] task specification; preamble, role, context, and output template are unchanged. The Resolved Summary conforms to the same intake schema as the standard one-shot.
- Pre-existing Phase 3 prompts (SCAMPER, JTBD, Brainwriting, Stress Test, Downselection) retained verbatim; Divergent Candidates is a synthesis step that consumes them.

## Next likely tasks

In order, if continuing the implementation pre-build:

- [ ] 1. Resolve the `challenge_outcome` enum mismatch.
- [x] 2. Author the full `role_assignment` and `task_specification_template` strings for the four new prompts (currently abbreviated in the spec). The Idea Seed Challenge and Design Grill task specs should embed the Grill-Me Interactive/Auto wrappers from §4.7.
- [ ] 3. Define `chosenDirectionAdjustmentsSchema` — the auxiliary JSON Schema fragment the Design Grill intake writes through. Spec says `{ field_path, proposed_value, rationale }` records, surfaced one-per-row in the diff preview.
- [ ] 4. Decide whether `archetypes_enforced` defaults to `true` at PCD creation (current spec implies yes; schema doesn't enforce a default — add `"default": true` if desired).
