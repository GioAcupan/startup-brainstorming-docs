# Startup — Session Notes (2026-04-30, 9:00 AM)

> Compressed handoff of this session. Drop into `external_context_notes` or use as a primer for the next LLM session.

## What this session accomplished

1. **Closed Task 3 from prior session's "Next likely tasks" — `createEmptyPCD()` factory specified.** New subsection §2.7 in `STARTUP_IMPLEMENTATION_SPEC.md` (~187 lines added). Specifies: function signature with testability hooks (`now`, `newId`); required inputs (`project_name`, `hackathon_profile`, `default_rubric`) plus optional `vault_references`; always-on Ajv validation against the produced PCD; per-section initial-state table for all 18 sections; root-level field initial-state table; `populated_by_phase` semantic for unpopulated sections with `user_research` and `external_context_notes` carve-outs documented; complete code skeleton with local `populated()`/`empty()` helpers; three Milestone-1 tests (Ajv-validates, `archetypes_enforced === true`, deterministic when seeded). Anchored cross-references: §2.3 (canonical phase-population mapping), §2.6 (`archetypes_enforced` three-place-alignment rationale), §9.4 (calling site in New Project Setup).

2. **Fixed §2.6 spec-vs-schema drift on `external_context_notes` shape.** Spec TS sketch had `data: { body_markdown: string }`; schema (`Section_ExternalContextNotes`) declares `data: { type: "string" }`. Schema canonical, spec sketch was wrong. Replaced the sketch with `SectionEnvelope<string>` and added a one-line cross-reference to the schema. The factory's initial value for this section is consequently `""` (bare empty string), not `{ body_markdown: '' }`. This drift was caught incidentally while authoring §2.7's initial-state table; fixed in the same session because deferring it would have left §2.7 inconsistent with §2.6.

## Resolved from prior session's "Next likely tasks"

- ✅ **Task 3** — `createEmptyPCD()` factory specified (the gap was: "the factory needs an explicit list of every scalar default it sets, and the spec doesn't yet enumerate it").

## Still open (carried forward)

- **Task 5** — Default Judging Rubric full content definition. Still untouched. Now top of the queue.
- **Milestone 1 build** per §11.2. Pre-build spec is now substantively closed for v1.5 scope; `createEmptyPCD()` was the last factory-shaped gap.
- **Deferred schema consideration: `populated_by_phase` nullability.** The factory currently uses a forward-looking semantic (`populated_by_phase = the phase that will populate this section`) for unpopulated sections. The schema requires the field as non-null integer 1-7. The alternative — making the field nullable in the schema — would be the only knock-on schema change a future session might consider. Defer until a real reason emerges (e.g., debugging where the field's forward-looking value misleads someone reading the data outside the §2.7.3 context). The convention is documented in §2.7.3 and is internally consistent with how `last_updated` is treated on unpopulated sections.

## Pushback recorded this session

None substantive. (No design issue reached the threshold of pushback against either the prior session's record or the user's framing.)

## New items flagged this session

1. **Three §2.7 design decisions made with rationale; reversal cost noted in case any need revisiting.**
   - **Always-on Ajv validation inside `createEmptyPCD`** (throws on schema failure). Cost is microseconds; benefit is loud regression detection the day someone edits `pcd_schema.json` without updating the factory. To gate behind `NODE_ENV !== 'production'` later: one-line guard.
   - **No synthetic `project_created` event in `phase_gate_log`.** The enum (schema lines 175-188) does not include one, and root-level `created_at` is the system of record for project creation time. Consistent with the prior session's "don't add vocab speculatively" discipline.
   - **Factory requires a resolved rubric as input.** The caller (New Project Setup) handles the Rubric Mapping Intake's replace/supplement/keep-default branches before invoking the factory. The factory does not know about the resolution flow — it just receives the resolved value with whatever `rubric_source` was chosen. Keeps the factory simple.

2. **Process note: behavioral correction received early in the session.** I opened by listing three already-closed dispositions (the `challenge_outcome` enum asymmetry, the two UI-label drifts in the workflow/handoff docs, the `judge_qa_drill_intake` "replace vs supplement" UI choice) under the framing "seams I'll be alert to." User correctly flagged this as treating closed decisions as live — naming a hypothetical `commit_strategy: 'direct_with_merge_choice'` was the architectural-completeness instinct the prior session explicitly warned against. Owned the mistake and dropped all three from working memory for the rest of the session. Carrying forward as a process note: when picking up cumulative project context, "tracking" already-disposed items quietly re-opens them; the discipline is to drop them, not to namecheck them.

3. **Drift-detection seam, observed.** The §2.6 `external_context_notes` shape drift was caught only because §2.7 needed to enumerate every section's `data` initial value, which forced reading the schema directly. The §2.7 initial-state table now serves as a cross-check against §2.6 — if a future schema change to a section's `data` shape is not mirrored into both §2.6 and §2.7, a session touching either should catch the inconsistency. Don't formalize this as a process; just note the seam exists.

## Reaffirmed design invariants

Untouched, plus two additions and one strengthening:

- JSON canonical PCD; Markdown rendered, never read back.
- Local-first IndexedDB (Dexie); Web Crypto AES-GCM with PBKDF2.
- Cloudflare Worker as stateless proxy; no backend database.
- Gemini Flash used only for intake extraction in MVP.
- Phase 4 gate is one-way; Design Grill (Step 4a) does not cross it.
- Competitive Pre-mortem is the only hard block.
- Grill-Me modifies only components [3] task specification and [5] output schema at generation time; the Resolved Summary conforms to the same intake schema as the standard one-shot.
- Pre-existing Phase 3 prompts (SCAMPER, JTBD, Brainwriting, Stress Test, Downselection) retained verbatim.
- Schema is canonical for enum drift; **strengthened: schema is also canonical for structural shape drift** (this session's `external_context_notes` fix establishes the precedent — when spec sketch and schema disagree on a `data` shape, schema wins, spec gets fixed).
- Prompt canonical for prompt-vs-spec drift on emission contracts.
- Declarative > imperative when use-case set is bounded (`commit_strategy`, `post_extract_validators` continue to follow this).
- **New: `createEmptyPCD` is the only construction site for fresh PCD instances.** All other state evolution is mutation of an existing PCD via intakes, the Section Editor, or reconciliation. If a second construction path is ever proposed, treat it as an architecture review trigger.
- **New: factory must always validate its output via Ajv.** §2.7 establishes this for `createEmptyPCD`; the same principle should apply to any future PCD-producing factory.

## File state at end of session

One output file in `/mnt/user-data/outputs/`, cumulative through this session's work:

- `STARTUP_IMPLEMENTATION_SPEC.md` — 2259 lines (was 2072 at session start). Carries the §2.7 addition (~187 lines) and the §2.6 `external_context_notes` shape fix (~3 line delta).

Other project files (`pcd_schema.json`, `PROMPTS.md`, `STARTUP_WORKFLOW_AND_APP_OVERVIEW.md`, `STARTUP_OPUS_HANDOFF_CONTEXT.md`, `default_judging_rubric.md`) **untouched this session** — re-upload only `STARTUP_IMPLEMENTATION_SPEC.md` into project knowledge before the next session, otherwise that session will read stale.

## Next likely tasks

In order, if continuing:

1. **Task 5 — Default Judging Rubric full content definition.** Begin by reading `default_judging_rubric.md` (untouched since project start) against `Section_ActiveJudgingRubric` schema and the Phase 1 workflow. Likely needs decisions on: weight distribution across the five universal criteria (Innovation & Novelty, Problem-Solution Fit, Technical Soundness, Feasibility & Viability, Presentation & Communication); judge-question authoring conventions (per criterion: how many, what voice, how concrete); and how the default content interacts with the official-criteria override semantics (replace / supplement / keep-default per §7.2) when the Rubric Mapping Intake runs.

2. **Begin Milestone 1 build** per §11.2. The pre-build spec is closed for v1.5 scope; `createEmptyPCD()` was the last gap. The §11.2 build order step 3 ("PCD schema imported; Ajv validator; createEmptyPCD function") is now directly executable from the §2.7 specification.

3. **Optional: revisit `populated_by_phase` nullability** if Milestone 4's reconciliation work surfaces a legitimate need. Until then, defer.
