# Startup — Implementation Specification

> Version 1.0 · Companion docs: `pcd_schema.json`, `default_judging_rubric.md`, `project_context.mdc`

This document is the technical implementation specification for the Startup application. It is intended to be consumed by Cursor (or another AI coding tool) as the master build context.

It is **not** a product overview — for that, see `STARTUP_WORKFLOW_AND_APP_OVERVIEW.md`. It assumes the workflow design and design rationale already exist, and translates them into concrete technical decisions: data shapes, prompt-generation logic, state-machine semantics, intake parsing, application architecture.

---

## Table of Contents

1. [Foundational Decisions](#1-foundational-decisions)
2. [The PCD — Project Context Document](#2-the-pcd--project-context-document)
3. [The Vault — Cross-Project Infrastructure](#3-the-vault--cross-project-infrastructure)
4. [Prompt Generation Logic](#4-prompt-generation-logic)
5. [State Machine & Reconciliation](#5-state-machine--reconciliation)
6. [Intake Parsing](#6-intake-parsing)
7. [Default Judging Rubric](#7-default-judging-rubric)
8. [Application Architecture](#8-application-architecture)
9. [Screen Specifications](#9-screen-specifications)
10. [UX Principles](#10-ux-principles)
11. [Build Order & MVP Cut Line](#11-build-order--mvp-cut-line)

---

## 1. Foundational Decisions

These are locked. Every other decision in this document is downstream of these.

| Decision             | Choice                                                                                   | Why                                                                                                                                          |
| -------------------- | ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Canonical PCD format | JSON, validated against schema                                                           | Enables typed access for reconciliation, prompt selection, staleness tracking. Markdown is rendered from JSON, never read back as canonical. |
| Persistence          | Local-first IndexedDB (Dexie.js)                                                         | Free, no auth required, works offline, fast. Cloud sync is a stretch goal.                                                                   |
| Encryption           | Web Crypto API (AES-GCM with PBKDF2 key derivation)                                      | Single local password protects the IndexedDB store. Browser-native, no dependencies. No password recovery — forget it, start over.           |
| Internal AI engine   | Gemini Flash (`gemini-2.5-flash` or successor on free tier) via Cloudflare Workers proxy | Free tier sufficient for context selection, prompt assembly, and structured extraction. Proxy keeps API key server-side.                     |
| External LLMs        | User-driven, manual paste back                                                           | Claude / Gemini Pro / ChatGPT / Deepseek / Perplexity. Startup never calls these directly.                                                   |
| Frontend             | React + Vite + TypeScript                                                                | Cursor-friendly, fast iteration, no SSR complexity.                                                                                          |
| State                | Zustand                                                                                  | Minimal boilerplate, plays well with IndexedDB persistence.                                                                                  |
| Hosting              | Cloudflare Pages (frontend) + Cloudflare Workers (proxy)                                 | Both free tier. Single ecosystem.                                                                                                            |
| Schema validation    | Ajv (JSON Schema draft 2020-12)                                                          | Industry standard, fast, well-typed.                                                                                                         |
| Markdown rendering   | `marked` for render, `unified` ecosystem if richer needs emerge                          | Deterministic JSON-to-Markdown is custom code, not library-driven, because the format is project-specific.                                   |

Two things deliberately not chosen:

- **No Next.js, no SSR.** This is a single-user offline-capable app. SSR adds nothing and complicates IndexedDB access patterns.
- **No backend database.** The Cloudflare Worker is a stateless proxy. All state lives in the browser. This is a hard constraint and removing it triggers an architecture review.

---

## 2. The PCD — Project Context Document

The full canonical schema lives in `pcd_schema.json`. This section explains the design and how the schema is used at runtime.

### 2.1 Why JSON canonical, Markdown rendered

The PCD is the spine of the system. Every prompt reads from it, every intake writes to it, every reconciliation walks its dependency graph. These operations require typed field access — knowing that `problem_analysis.data.five_whys_chain` is an array of objects with specific shape, not a blob of text. JSON Schema gives us that for free, plus runtime validation, plus self-documentation.

Markdown is the export and human-read format only. A deterministic `renderPCDToMarkdown(pcd: PCD): string` function produces the human-readable view shown in the PCD Viewer screen and exported for Cursor handoff. This function is pure, idempotent, and never inverted — markdown is never read back as canonical.

When exporting to Cursor, the markdown export ends with an HTML-commented fenced code block containing the raw JSON:

```markdown
<!-- Startup PCD machine-readable state below. Do not edit by hand. -->
```json
{ ...full PCD object... }
```

```
This preserves the option to re-import a Cursor-edited file later, even though MVP doesn't support that.

### 2.2 The SectionEnvelope pattern

Every section in the PCD is wrapped in a common envelope (defined as `$defs/SectionEnvelope` in the schema) that carries metadata:

```ts
type SectionEnvelope<T> = {
  data: T;                          // section-specific content
  is_populated: boolean;            // false until phase fills it
  is_stale: boolean;                // true when upstream changed
  stale_reason: string | null;      // human-readable explanation
  last_updated: string;             // ISO 8601
  populated_by_phase: number;       // 1-7
  extraction_confidence: number | null;  // 0-1, from Flash
  user_overrides: UserOverride[];   // audit trail of manual edits
};
```

The envelope pattern means every section can participate in the same reconciliation logic, the same staleness detection, the same audit trail — without per-section special-casing.

### 2.3 Section list and which phase populates each

| Section                             | Populated by Phase | Notes                                                                                                                                                                                                                       |
| ----------------------------------- | ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `hackathon_profile`                 | 1                  | Direct user input, not LLM-extracted.                                                                                                                                                                                       |
| `judge_intelligence`                | 1                  | LLM-extracted from Judge Research Prompt outputs.                                                                                                                                                                           |
| `active_judging_rubric`             | 1                  | Pre-populated from `default_judging_rubric.md` at project create. Updated when official criteria arrive.                                                                                                                    |
| `problem_analysis`                  | 2                  | LLM-extracted from Problem Analysis Prompt output.                                                                                                                                                                          |
| `competitive_landscape`             | 2                  | LLM-extracted from Competitive Pre-mortem output. **Hard requirement for Phase 2 completion.**                                                                                                                              |
| `user_research`                     | 2 or 3             | Tracked but not blocking. May arrive in either phase.                                                                                                                                                                       |
| `ideation_record`                   | 3                  | Multi-step extraction across the ideation pipeline.                                                                                                                                                                         |
| `chosen_direction`                  | 3                  | Direct user input + premortem overlap check.                                                                                                                                                                                |
| `scope_definition`                  | 4                  | Extracted from TID Prompt output.                                                                                                                                                                                           |
| `technical_direction`               | 4                  | Extracted from TID Prompt output.                                                                                                                                                                                           |
| `product_brief`                     | 4                  | Extracted from Product Brief Prompt output.                                                                                                                                                                                 |
| `technical_implementation_document` | 4                  | Extracted from TID Prompt output (the dominant Phase 4 artifact).                                                                                                                                                           |
| `demo_script_and_backup_plan`       | 4                  | Extracted from Demo Script Prompt output.                                                                                                                                                                                   |
| `pitch_brief`                       | 6                  | Extracted from Pitch Brief Prompt output. Lightweight — bullets, not script.                                                                                                                                                |
| `judge_objection_map`               | 6                  | Extracted from Pitch Review Prompt output.                                                                                                                                                                                  |
| `qa_prep_document`                  | 6                  | Extracted from Q&A Prep Prompt output.                                                                                                                                                                                      |
| `debrief_record`                    | 7                  | Extracted from Debrief Prompt output.                                                                                                                                                                                       |
| `external_context_notes`            | —                  | User-maintained Markdown free-text. Never auto-populated, never auto-stale. `is_populated` flips to `true` on first user save. Optionally injected into Grill-Me prompts via the `'if_user_provided'` selector rule (§4.3). |

### 2.4 Phase Gate Log

The `phase_gate_log` is an append-only event stream. It records every meaningful state transition: phase completions, overrides, backward navigations, staleness resolutions, the Phase 4 gate crossing, addendum creations, and Vault queries.

The log is never edited or deleted. It serves three purposes: post-hoc debugging, the Phase 4 audit trail, and inputs to the Phase 7 debrief.

### 2.5 Vault references

Vault entries are referenced by ID, never copied into the PCD. This means a Vault entry can be edited later (e.g., adding observations after the hackathon) without retroactively mutating completed projects. The PCD records *which* Vault entries informed it; the Vault entries themselves remain Vault-side.

### 2.6 Schema fragment additions for v1.5

These additions are made directly to `pcd_schema.json` and surfaced here for traceability. They are additive — no existing field is renamed, retyped, or removed.

#### `ideation_record.data` additions

```ts
// existing fields (scamper_outputs, jtbd_analysis, brainwriting_round, ...) unchanged.
team_idea_seed: {
  original_text: string;            // user's preconceived idea, verbatim
  challenge_outcome: 'adopt_as_seed' | 'refined' | 'rejected';
  refined_text: string | null;      // populated only when outcome is 'refined'
  verdict_rationale: string;        // Resolved Summary rationale from the Grill-Me dialogue
} | null;

candidate_portfolio: {
  candidates: CandidateIdea[];      // 3-5 entries, written by the Divergent Candidates intake
} | null;

archetypes_enforced: boolean;       // mirrors the Step 3e toggle state at generation time;
                                    // captured for audit so debriefs can correlate
                                    // diversity-on vs diversity-off runs.
                                    // Default: true. Set explicitly at fresh-PCD init
                                    // by the createEmptyPCD() factory; not relied on
                                    // via Ajv schema defaults (Ajv runs without
                                    // `useDefaults: true` so genuinely-missing fields
                                    // elsewhere don't get silently filled).

type CandidateIdea = {
  id: string;                       // ULID; referenced from chosen_direction.data.selected_candidate_id
  name: string;
  description: string;              // one-line
  innovation_score: number;         // 1-5 integer
  impact_score: number;             // 1-5 integer
  feasibility_score: number;        // 1-5 integer
  archetype: 'safe_utility' | 'weird_gem' | 'moonshot' | 'social_impact' | 'unassigned';
  distinguishing_factors: string[]; // why this candidate is structurally different from the others
  is_remix: boolean;                // true when produced by the candidate-review remix UI
  remix_source_ids: string[];       // candidate ids combined to produce a remix; empty otherwise
};
```

`candidate_portfolio.candidates[]` is the canonical home for the divergent candidates produced at Step 3e. The pre-v1.5 schema referred to this content as living directly under `ideation_record`; v1.5 formalizes it as a named subfield.

**`archetypes_enforced` default — decision recorded.** Default value is `true`. Rationale: the four diversity archetypes (Safe Utility, Weird Gem, Moonshot, Social Impact) exist specifically to counteract the "premature gravity" failure mode the workflow doc names — without enforcement, the LLM converges on one or two practical candidates and the portfolio loses its divergence value. Defaulting to `true` makes the disciplined behavior the path of least resistance; users who want to opt out can flip the toggle in the Phase Workspace before generation, and the toggle state is captured in this field for audit. The choice is preserved in three places that must agree: the schema (`pcd_schema.json` adds `"default": true` on this field), the runtime input on `phase_3_divergent_candidates` (§4.2 sets `default: true` on the boolean `RuntimeInputDef`), and the `createEmptyPCD()` factory (sets the field to `true` when initializing a fresh `ideation_record` envelope, since Ajv runs without `useDefaults: true`).

#### `chosen_direction.data` addition

```ts
// existing fields (selected_idea, user_rationale, key_differentiators,
// premortem_overlap_check) unchanged.
selected_candidate_id: string | null;  // CandidateIdea.id from candidate_portfolio.candidates[],
                                       // OR null when the chosen direction was authored manually
                                       // rather than selected from the portfolio.
```

When non-null, the PCD Viewer resolves this id at render time to show provenance ("Selected: Candidate X — Weird Gem archetype").

#### `external_context_notes` (top-level section)

```ts
external_context_notes: SectionEnvelope<{
  body_markdown: string;            // free-form Markdown; user-authored only
}>;
```

Wrapped in the standard `SectionEnvelope`. `is_populated` becomes `true` on first save. `is_stale` is never set by the staleness algorithm (§5.3) — the dependency graph entry is empty.

---

## 3. The Vault — Cross-Project Infrastructure

The Vault is the supra-PCD layer. It holds infrastructure that persists across hackathons and is queried into individual projects on demand.

### 3.1 Vault structure

```ts
type Vault = {
  hackathon_vault: HackathonVault;
  client_vault: ClientVault | null;  // null in MVP; scaffolded for stretch
  global_settings: GlobalSettings;
};

type HackathonVault = {
  judges_and_organizers: JudgeOrganizerEntry[];
  winning_solutions: WinningSolutionEntry[];
  boilerplate_components: BoilerplateEntry[];
};
```

Two top-level partitions: **Hackathon Vault** (active in MVP) and **Client Vault** (empty placeholder for the real-world stretch goal). The partitioning ensures the future real-world client mode never contaminates the hackathon-side data and vice versa.

### 3.2 Vault entry shapes

```ts
type JudgeOrganizerEntry = {
  id: string;                      // ULID
  type: 'judge' | 'organizer' | 'sponsor' | 'mixed';
  name: string;
  organization: string | null;
  professional_background: string;
  events_appeared_at: string[];    // event names
  observed_judging_tendencies: string[];
  notes: string;
  tags: string[];                  // free-form, used for filtering
  created_at: string;
  updated_at: string;
};

type WinningSolutionEntry = {
  id: string;
  event_name: string;
  organizer: string | null;
  domain: string | null;
  year: number | null;
  solution_summary: string;
  why_it_won: string;
  source: string | null;           // URL or reference
  tags: string[];
  created_at: string;
  updated_at: string;
};

type BoilerplateEntry = {
  id: string;
  component_name: string;
  category: 'frontend_ui' | 'backend_architecture' | 'ai_integration' | 'deployment_config' | 'other';
  description: string;             // what it is, what it does — NO CODE
  used_in_events: string[];
  what_worked: string;
  what_didnt: string;
  repo_pointer: string | null;     // path or URL to the actual repo, for the user's reference only
  tags: string[];
  created_at: string;
  updated_at: string;
};
```

**Critical: BoilerplateEntry stores descriptions, not code.** The actual code lives in the user's own repos. Startup only knows entries exist, what they're good for, and how to reference them when generating the TID prompt.

### 3.3 Querying the Vault

Vault queries are plaintext keyword + tag filtering over indexed fields, executed against IndexedDB indexes via Dexie.

```ts
type VaultQuery = {
  store: 'judges_and_organizers' | 'winning_solutions' | 'boilerplate_components';
  text_query?: string;             // matched against name, description, summary
  tag_filters?: string[];          // AND-combined
  field_filters?: Record<string, string>;  // exact-match on specific fields, e.g. { domain: 'health' }
};
```

No SQL. Keyword search is sufficient for a personal database that will grow to hundreds of entries over years. If it ever needs more, the schema supports adding indexes without breaking changes.

### 3.4 When the Vault is queried

| Phase | Auto-query                                                | Trigger                                  |
| ----- | --------------------------------------------------------- | ---------------------------------------- |
| 1     | `judges_and_organizers` by organizer name + sponsor names | Project creation                         |
| 1     | `winning_solutions` by domain or theme                    | Project creation                         |
| 2     | `winning_solutions` by domain                             | Competitive Pre-mortem Prompt generation |
| 4     | `boilerplate_components` by inferred tech stack needs     | TID Prompt generation                    |

Auto-query results are surfaced to the user in the Phase Workspace as suggestions. The user explicitly opts each result in or out of the prompt context. This keeps the user in control and prevents irrelevant Vault noise from polluting prompts.

### 3.5 Vault entries in PCD

When a user opts a Vault result into a phase's prompt, the entry's ID is added to `pcd.vault_references.<store>_ids`. The entry contents are NOT copied into the PCD — only the ID is recorded. At render time, the PCD viewer can resolve the IDs to show the user what informed each phase.

This means Vault entries can be edited later without retroactively changing past PCDs. If a JudgeOrganizerEntry is updated with new notes after the hackathon, the old PCD still references it but renders against the latest version. This is intentional — the historical record is the Phase Gate Log, not the Vault entries.

---

## 4. Prompt Generation Logic

This section specifies how Gemini Flash assembles the prompts that the user executes in external LLMs.

### 4.1 The prompt template structure

Every Startup-generated prompt has five components, in this order:

```
[1] FORMAT-DISCIPLINE PREAMBLE
[2] ROLE ASSIGNMENT
[3] CONTEXT INJECTION
[4] TASK SPECIFICATION
[5] OUTPUT TEMPLATE (derived from target schema)
```

This shape is invariant. Every prompt-generation function produces output in this shape.

#### [1] Format-discipline preamble

A constant block prepended to every prompt:

```
You will be given an output template at the end of this prompt. Adhere to it
exactly. Do not produce executive summaries, conclusions, introductions, or
commentary outside the specified sections. Do not add citations unless the
template requests them. Output only what the template defines, in markdown.
```

This is the single biggest lever for reliable extraction downstream. It gets the LLM to push back against its own helpfulness reflex.

#### [2] Role assignment

Phase-specific. Examples:

- Phase 2 Problem Analysis: `"You are a senior product strategist with deep experience in problem decomposition. Your job is to surface non-obvious angles, not to summarize."`
- Phase 2 Competitive Pre-mortem: `"You are a hackathon judge who has watched hundreds of teams pitch the same brief. Your job is to predict what the average team will build, so this team can avoid that crowded space."`
- Phase 3 SCAMPER: `"You are simulating a three-way dialogue between a target user, a domain expert, and a technical expert. Each persona contributes to each SCAMPER dimension."`

#### [3] Context injection

The selected slice of the PCD relevant to this prompt. Selection rules in §4.3.

Format: a series of labeled markdown sections, each rendered from the relevant PCD section by the same `renderPCDToMarkdown` function used for export. Sections are introduced with a header so the LLM understands the structure:

```markdown
## CONTEXT — Hackathon Profile
[rendered hackathon_profile section]

## CONTEXT — Active Judging Rubric
[rendered active_judging_rubric section]

## CONTEXT — Vault: Judge & Organizer References
[rendered Vault entries the user opted in]
```

#### [4] Task specification

Phase-specific instructions describing what to produce, with explicit constraints. These include negative constraints derived from the PCD (e.g., "Do not generate ideas that overlap with the predicted average solutions listed in the context above.").

#### [5] Output template

A markdown skeleton showing exactly the section headers and structure the LLM should produce. This is **derived programmatically from the target intake schema** so prompt and schema can never drift.

Example for SCAMPER intake (target schema: `Section_IdeationRecord.data.scamper_outputs`):

```markdown
## OUTPUT TEMPLATE (follow exactly)

## Substitute
- (one bullet per substitute idea)

## Combine
- (one bullet per combine idea)

## Adapt
- (one bullet per adapt idea)

## Modify
- (one bullet per modify idea)

## Put to Other Use
- (one bullet per repurpose idea)

## Eliminate
- (one bullet per eliminate idea)

## Reverse
- (one bullet per reverse idea)
```

The template generator walks the target JSON Schema fragment and produces the appropriate markdown skeleton automatically. This is a small reusable module — `schemaToOutputTemplate(jsonSchemaFragment): string`.

### 4.2 The `PromptDefinition` registry

Every prompt the system can generate is registered as a `PromptDefinition`:

```ts
type PromptDefinition = {
  id: string;                      // e.g. 'phase_2_problem_analysis'
  phase: number;
  display_name: string;            // shown in UI
  recommended_external_llm: 'claude' | 'gemini_pro' | 'chatgpt' | 'deepseek' | 'perplexity_deep_research' | 'gemini_deep_research';
  role_assignment: string;
  context_selectors: ContextSelector[];   // §4.3 — resolves PCD slices into component [3] context
  vault_query_hints: VaultQueryHint[] | null;  // auto-suggest Vault entries
  task_specification_template: string;    // string with {placeholder} slots filled from PCD and runtime_inputs
  runtime_inputs?: RuntimeInputDef[];     // §4.3 — values collected from the prompt-form at generation
                                          // time (textareas, settings, toggles). Filled into
                                          // task_specification_template and/or role_assignment by name.
                                          // Distinct from context_selectors: these are NOT PCD-derived.
  target_intake_schema_path: string;      // dotted path to the JSON Schema fragment for output
  produces_intake_id: string;             // matches an IntakeDefinition (§6.2)
  grill_me_modes?: ('interactive' | 'auto')[];  // §4.7 — when present, the prompt can be toggled
                                                 // into Grill-Me mode at generation time. Empty
                                                 // or omitted = standard one-shot generation only.
                                                 // Both modes listed = user picks per-generation.
};
```

The registry is a static TypeScript constant. It defines all ~15-20 prompts the system can generate. Adding a new prompt = adding an entry to this registry plus a corresponding intake definition.

#### v1.5 prompt registry additions

The following four prompt definitions are added in v1.5. The pre-existing Phase 3 prompts (SCAMPER, JTBD, Brainwriting, Differentiation Stress Test, Downselection) are **retained unchanged** — they continue to run as Steps 3b–3d (and the existing downstream steps) before Divergent Candidates.

```ts
// Step 3a — gates the seed before it influences ideation. Always Grill-Me.
{
  id: 'phase_3_idea_seed_challenge',
  phase: 3,
  display_name: 'Idea Seed Challenge',
  recommended_external_llm: 'claude',
  role_assignment: '...adversarial Socratic skeptic interrogating a preconceived idea...',
  context_selectors: [/* see §4.3 */],
  vault_query_hints: null,
  task_specification_template: '...stress-test the seed against the brief, problem analysis, and crowded space; emit a verdict: adopt_as_seed | refined | rejected...',
  runtime_inputs: [
    {
      name: 'seed_idea_text',
      kind: 'textarea',
      label: 'Your seed idea',
      placeholder: 'Describe the preconceived idea you want stress-tested before ideation begins.',
      required: true,
      max_length: 2000,
      injects_into: 'task_specification',  // {seed_idea_text} appears in component [4]
    },
  ],
  target_intake_schema_path: 'sections.ideation_record.data.team_idea_seed',
  produces_intake_id: 'idea_seed_challenge_intake',
  grill_me_modes: ['interactive', 'auto'],   // required — no off mode
}

// Step 3e — synthesizes the ideation pipeline outputs into 3-5 archetyped candidates.
{
  id: 'phase_3_divergent_candidates',
  phase: 3,
  display_name: 'Divergent Candidate Generation',
  recommended_external_llm: 'claude',        // also valid: 'gemini_pro'
  role_assignment: '...synthesist drawing on prior ideation outputs to produce 3-5 radically different candidates...',
  context_selectors: [/* see §4.3 */],
  vault_query_hints: null,
  task_specification_template:
    '...synthesize the SCAMPER, JTBD, and Brainwriting outputs in CONTEXT; produce {candidate_count_min}-{candidate_count_max} radically different candidate ideas. ' +
    'When archetypes_enforced=true, distribute candidates across the four archetypes: Safe Utility, Weird Gem, Moonshot, Social Impact. ' +
    'When archetypes_enforced=false, the LLM is free to produce candidates without archetype distribution. ' +
    'Each candidate: name, one-line description, integer scores 1-5 for innovation, impact, feasibility, and a list of distinguishing factors...',
  runtime_inputs: [
    {
      name: 'candidate_count_min',
      kind: 'integer',
      label: 'Minimum candidates',
      default: 3,
      min: 3,
      max: 5,
      injects_into: 'task_specification',
    },
    {
      name: 'candidate_count_max',
      kind: 'integer',
      label: 'Maximum candidates',
      default: 5,
      min: 3,
      max: 5,
      injects_into: 'task_specification',
    },
    {
      name: 'archetypes_enforced',
      kind: 'boolean',
      label: 'Enforce diversity archetypes',
      default: true,                         // §2.6 / pcd_schema.json — see Task 4 alignment
      injects_into: 'task_specification',
      // Toggle state mirrors into ideation_record.data.archetypes_enforced at intake time
      // for audit (see §2.6). UI surfaces this as the Step 3e "Enforce Diversity Archetypes"
      // toggle described in the workflow doc.
    },
  ],
  target_intake_schema_path: 'sections.ideation_record.data.candidate_portfolio',
  produces_intake_id: 'divergent_candidates_intake',
  grill_me_modes: ['interactive', 'auto'],   // optional refinement
}

// Step 4a — stress-tests scope before the three Phase 4 documents are generated.
{
  id: 'phase_4_design_grill',
  phase: 4,
  display_name: 'Design Grill',
  recommended_external_llm: 'claude',
  role_assignment: '...adversarial Socratic skeptic stress-testing the chosen direction at the product-decision level — features, flows, edge cases — without dipping into low-level technical implementation...',
  context_selectors: [/* see §4.3 */],
  vault_query_hints: null,
  task_specification_template: '...interrogate the scope and emit a Resolved Summary listing confirmed adjustments to the chosen direction and intended scope...',
  // For this prompt, target_intake_schema_path names the COMMIT TARGET (the
  // base path against which each adjustment's `field_path` resolves). The
  // shape Flash extracts INTO is the auxiliary `chosenDirectionAdjustmentsSchema`
  // referenced from the IntakeDefinition (§6.2). Together with
  // `commit_strategy: 'apply_adjustments'` on the intake side, this is the
  // canonical pattern for any future intake that produces a list of patches
  // against an existing section rather than a wholesale section payload.
  target_intake_schema_path: 'sections.chosen_direction.data',
  produces_intake_id: 'design_grill_intake',
  grill_me_modes: ['interactive', 'auto'],   // required — Design Grill IS the Grill-Me; no off mode
}

// Phase 6 (parallel with Phase 5) — adversarial mock judge panel.
{
  id: 'phase_6_judge_qa_drill',
  phase: 6,
  display_name: 'Judge Q&A Drill',
  recommended_external_llm: 'claude',
  role_assignment: '...embody the Phase 1 judge personas and fire questions one at a time as the actual panel would...',
  context_selectors: [/* see §4.3 */],
  vault_query_hints: null,
  task_specification_template: '...drill the team on Q&A; emit a Resolved Summary as a sharpened Q&A document conforming to the qa_prep_document schema...',
  target_intake_schema_path: 'sections.qa_prep_document.data',
  produces_intake_id: 'judge_qa_drill_intake',
  grill_me_modes: ['interactive', 'auto'],   // required — drill IS the Grill-Me; no off mode
}
```

The role assignment and task specification strings are abbreviated above for readability. The full content per prompt is authored in the prompts module (`frontend/src/lib/prompts/templates/`) in the same style as existing prompts.

**On `runtime_inputs`.** Two of the v1.5 prompts carry placeholders that are filled at generation time from a prompt-form widget rather than from PCD content: `phase_3_idea_seed_challenge` consumes a textarea (`{seed_idea_text}`); `phase_3_divergent_candidates` consumes two integer inputs (`{candidate_count_min}`, `{candidate_count_max}`) and a boolean (`{archetypes_enforced}`). These are formalized as `RuntimeInputDef` entries on the `PromptDefinition` rather than as `ContextSelector`s, because they are not PCD slices — they are user-typed-or-toggled values collected at the moment of generation. The `ContextSelector` abstraction stays narrow (resolve a PCD section into component [3]); the `runtime_inputs` array describes the prompt-form fields the Phase Workspace renders just above the "Generate Prompt" button.

### 4.3 Context selection — what gets injected

The most important design decision in this section: prompts are **context-selective, not full-PCD**. Stuffing the whole PCD into every prompt wastes tokens, dilutes attention, and hurts output quality. The system picks only what the prompt needs.

A `ContextSelector` declares which PCD sections (and optionally which subfields) a prompt requires:

```ts
type ContextSelector = {
  section: keyof PCD['sections'];
  fields?: string[];               // dotted paths within the section; default = all
  required: boolean;               // if true and section is empty, prompt generation fails with a clear error
  inclusion_rule?: 'always' | 'if_populated' | 'if_user_research_complete' | 'if_user_provided';
};
```

The `'if_user_provided'` rule is the inclusion gate for `external_context_notes`. The section is included only when `is_populated === true` AND `data.body_markdown` is non-empty. It is the only rule whose source is a section the user authors directly rather than the system populates via intake.

#### Runtime inputs — values from the prompt-form widget

`ContextSelector` is intentionally narrow: it resolves PCD slices into component [3] context injection. It does NOT cover values typed or toggled by the user at the moment of generation — those are a different concept and live on `PromptDefinition.runtime_inputs`. A `RuntimeInputDef` declares one prompt-form field:

```ts
type RuntimeInputDef =
  | {
      name: string;                          // matches a {placeholder} token in template strings
      kind: 'textarea';
      label: string;
      placeholder?: string;
      required: boolean;
      max_length?: number;
      injects_into: 'task_specification' | 'role_assignment';  // which template the {name} appears in
    }
  | {
      name: string;
      kind: 'integer';
      label: string;
      default: number;
      min?: number;
      max?: number;
      injects_into: 'task_specification' | 'role_assignment';
    }
  | {
      name: string;
      kind: 'boolean';
      label: string;
      default: boolean;
      injects_into: 'task_specification' | 'role_assignment';
    };
```

The `Phase Workspace` Prompt Generator card renders one widget per `RuntimeInputDef` immediately above the "Generate Prompt" button, in the order declared. On generate, values are collected, the relevant template (`task_specification_template` or `role_assignment`) has its `{name}` tokens substituted, and the Grill-Me task-spec rewrite (§4.7) — if active — runs after substitution so the Grill-Me variant sees the final filled values.

**Why a separate concept rather than extending `ContextSelector`.** `ContextSelector` resolves named PCD sections into a deterministic, audit-logged injection (the "Show Injected Context" toggle in §4.4 displays exactly what was pulled in). Extending it to also cover prompt-form widgets would conflate two different lifecycles: PCD slices come from saved state and are reproducible across re-generations of the same prompt; runtime inputs come from the user's current keystrokes and are not reproducible. Keeping them as separate fields on `PromptDefinition` means the resolver function for context (read PCD) and the collector function for runtime inputs (read DOM form state) are independent and individually testable. It also keeps the "Show Injected Context" panel honest — it shows PCD provenance only, while the Phase Gate Log records the runtime-input values separately for audit (`runtime_inputs_used: { name: value, ... }` on the `prompt_generated` log entry).

**Validation.** Runtime inputs are validated client-side at generate time. `textarea` with `required: true` and empty value blocks generation with a focused error on the field. `integer` outside `[min, max]` blocks generation. Failed validation surfaces inline next to the offending widget; the "Generate Prompt" button is disabled while any field is invalid. No PCD writes occur during runtime-input collection — these values are ephemeral to the generation step except where the prompt's intake explicitly mirrors a value into the PCD on the return trip (`archetypes_enforced` is the one such case in v1.5; the seed text is captured into `team_idea_seed.original_text` only via the Resolved Summary intake, not via the runtime-input echo).

#### Selection rules per phase

| Prompt                              | Required Context                                                                                                                                                                                                                                                    | Optional Context                                                                                                                   |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Phase 1 Judge Research Prompt       | `hackathon_profile`                                                                                                                                                                                                                                                 | Vault matches on organizer/sponsor                                                                                                 |
| Phase 2 Problem Analysis            | `hackathon_profile.data.problem_statement_raw`, `hackathon_profile.data.theme`                                                                                                                                                                                      | `judge_intelligence` (for rubric calibration), `active_judging_rubric`                                                             |
| Phase 2 Competitive Pre-mortem      | `hackathon_profile`, `problem_analysis`                                                                                                                                                                                                                             | Vault matches on `winning_solutions` by domain                                                                                     |
| Phase 3 SCAMPER                     | `problem_analysis`, `competitive_landscape`, `user_research` (if populated)                                                                                                                                                                                         | `active_judging_rubric`, `ideation_record.data.team_idea_seed` (if populated)                                                      |
| Phase 3 JTBD                        | `problem_analysis`, `user_research` (if populated)                                                                                                                                                                                                                  | `ideation_record.data.team_idea_seed` (if populated)                                                                               |
| Phase 3 Brainwriting                | `problem_analysis`, `competitive_landscape.data.spaces_to_avoid`                                                                                                                                                                                                    | `ideation_record.data.scamper_outputs`, `ideation_record.data.jtbd_analysis`, `ideation_record.data.team_idea_seed` (if populated) |
| Phase 3 Idea Seed Challenge         | `hackathon_profile`, `problem_analysis`, `competitive_landscape`, the user's seed text (passed at generation time, not from PCD)                                                                                                                                    | `judge_intelligence`, `active_judging_rubric`, `external_context_notes` (`if_user_provided`)                                       |
| Phase 3 Divergent Candidates        | `problem_analysis`, `competitive_landscape`, `judge_intelligence`, `active_judging_rubric`, `ideation_record.data.scamper_outputs`, `ideation_record.data.jtbd_analysis`, `ideation_record.data.brainwriting_round` (at least one of these three must be populated) | `ideation_record.data.team_idea_seed` (if adopted), `user_research` (if populated), `external_context_notes` (`if_user_provided`)  |
| Phase 3 Differentiation Stress Test | `competitive_landscape`, `ideation_record`                                                                                                                                                                                                                          | —                                                                                                                                  |
| Phase 3 Downselection               | `active_judging_rubric`, `ideation_record`, `competitive_landscape`                                                                                                                                                                                                 | `judge_intelligence`                                                                                                               |
| Phase 4 Product Brief               | `chosen_direction`, `problem_analysis.data.team_framing`, `hackathon_profile`                                                                                                                                                                                       | `judge_intelligence`                                                                                                               |
| Phase 4 Design Grill                | `chosen_direction`, `scope_definition` (if populated), `problem_analysis`, `competitive_landscape`, `judge_intelligence`                                                                                                                                            | `active_judging_rubric`, `external_context_notes` (`if_user_provided`)                                                             |
| Phase 4 TID                         | `chosen_direction`, `scope_definition` (if populated), `technical_direction` (if populated), `hackathon_profile.data.timeline`                                                                                                                                      | Vault matches on boilerplate components                                                                                            |
| Phase 4 Demo Script                 | `chosen_direction`, `scope_definition`, `technical_implementation_document`                                                                                                                                                                                         | `judge_intelligence`                                                                                                               |
| Phase 6 Pitch Brief                 | `judge_intelligence`, `active_judging_rubric`, `chosen_direction`, `product_brief`, `competitive_landscape.data.spaces_to_avoid`                                                                                                                                    | `hackathon_profile.data.timeline.final_pitch_at`                                                                                   |
| Phase 6 Pitch Review                | `judge_intelligence`, `pitch_brief`, `chosen_direction`                                                                                                                                                                                                             | —                                                                                                                                  |
| Phase 6 Q&A Prep                    | `judge_intelligence`, `pitch_brief`, `chosen_direction`, `technical_implementation_document.data.known_technical_risks`                                                                                                                                             | `competitive_landscape`                                                                                                            |
| Phase 6 Judge Q&A Drill             | `judge_intelligence`, `pitch_brief`, `qa_prep_document`, `chosen_direction`, `product_brief`                                                                                                                                                                        | `judge_objection_map`, `competitive_landscape`, `external_context_notes` (`if_user_provided`)                                      |
| Phase 7 Debrief                     | full PCD                                                                                                                                                                                                                                                            | —                                                                                                                                  |

The Phase 7 Debrief is the only prompt that legitimately needs the full PCD, because it's a retrospective on everything.

### 4.4 The "Show Context" toggle

Every Phase Workspace shows a toggle: **Show Injected Context**. Off by default. When on, a panel beside the generated prompt displays exactly which PCD sections were injected, in what order, with character counts. This is for debugging and for the user's own confidence — they can verify Startup is doing the right thing.

This is a stated UX principle: transparency by default, even when transparency is hidden behind a toggle.

### 4.5 Prompt generation flow

```
User clicks "Generate Prompt" in Phase Workspace
    │
    ▼
Look up PromptDefinition by current phase + button clicked
    │
    ▼
If runtime_inputs is populated, collect values from the prompt-form widgets
  (textarea, integer, boolean) and validate client-side. Block on invalid.
    │
    ▼
If grill_me_modes is populated AND user has selected Interactive or Auto
  via the per-prompt toggle, mark this generation as Grill-Me; otherwise
  proceed as standard one-shot.
    │
    ▼
Resolve ContextSelectors against current PCD state
    │
    ├─ Required section missing/unpopulated → fail with actionable error
    │
    ▼
Render selected sections to markdown
    │
    ▼
Resolve Vault query hints; show user the matches
    │
    ▼
User selects which Vault entries to include
    │
    ▼
Call schemaToOutputTemplate(target_intake_schema_path)
    │
    ▼
Substitute runtime_inputs values into task_specification_template and
  role_assignment by {name}. Then assemble final prompt:
  preamble + role + context + (task spec OR Grill-Me task spec) + template
    │  (Grill-Me mode rewrites only [4] task specification — see §4.7. Components
    │   [1] preamble, [2] role, [3] context, [5] output template are unchanged.
    │   Runtime input substitution happens BEFORE the Grill-Me rewrite so the
    │   Grill-Me variant operates on filled values, not raw {placeholder} tokens.)
    │
    ▼
Display in copyable code block
Show recommended external LLM next to copy button
    │
    ▼
If Grill-Me mode is active, also surface an "Export Handoff Package" button:
  builds a single .md file containing the full PCD render + external_context_notes
  (if populated) + the generated prompt text; user uploads this directly to the
  external LLM's web UI to give it durable context for a long Grill-Me session
  without Startup managing live chat history.
    │
    ▼
Log generation in Phase Gate Log (prompt_id, timestamp, included_vault_ids,
  grill_me_mode, runtime_inputs_used)
```

#### Phase 3 ordering enforcement

The Prompt Generator surfaces Phase 3 prompts in a specific order, mirroring the Step 3a–3e workflow:

1. Step 3a — Idea Seed Challenge (optional; visible only when the user has supplied a seed).
2. Step 3b — SCAMPER.
3. Step 3c — JTBD.
4. Step 3d — Brainwriting.
5. Step 3e — Divergent Candidate Generation.

`phase_3_divergent_candidates` is `disabled` in the Prompt Generator card UI until at least one of `ideation_record.data.scamper_outputs`, `ideation_record.data.jtbd_analysis`, or `ideation_record.data.brainwriting_round` is populated. This is enforced at the generation function level (returning a `precondition_unmet` error) and at the UI level (the card shows a disabled CTA with a tooltip naming the missing precondition). The remaining downstream prompts (Differentiation Stress Test, Downselection) sit below Step 3e in the card stack and are unaffected by this gate.

### 4.6 Gemini Flash's role

Notice: **Gemini Flash is NOT used in prompt generation in MVP.** Prompt assembly is deterministic template-and-context-selection logic written in TypeScript. Flash is used only for intake parsing (§6).

This is a deliberate simplification. Adding Flash to prompt assembly later — for example, to dynamically refine the task specification based on PCD content — is a v2 enhancement. For MVP, deterministic templating is more reliable and easier to debug.

### 4.7 Grill-Me Mode

Grill-Me is a per-prompt modifier, not a separate prompt type. When a `PromptDefinition` declares `grill_me_modes` and the user toggles it on for a specific generation, the prompt assembler rewrites only component **[4] task specification**. Components [1] preamble, [2] role, [3] context, and [5] output template are produced exactly as in standard generation. The output schema is unchanged because the Resolved Summary the LLM ultimately emits must match the same intake schema the standard one-shot would have produced.

#### Two sub-modes

```ts
type GrillMeMode = 'interactive' | 'auto';
```

**Interactive.** The external LLM acts as a Socratic skeptic. It asks the user **one question at a time**, drawn from six question types (clarification, probing assumptions, probing reasons, exploring viewpoints, examining implications, meta-questioning). It waits for each answer before issuing the next question. It refuses vague answers and probes them. The dialogue continues across all branches of the decision tree until every line of questioning is resolved. At that point — and only at that point — the LLM emits a single Resolved Summary that conforms to the prompt's output template. The user pastes only the Resolved Summary back into Startup. The full Q&A transcript is not stored.

**Auto.** The external LLM simulates both sides of the dialogue internally — generating questions, producing plausible answers, challenging those answers, iterating — and emits the Resolved Summary directly without any user turn. Faster, less rigorous, useful when the user is confident in current thinking and wants the stress-test to run unattended.

#### Task specification rewrite

The standard task spec ("produce X analysis according to the template below") is replaced — wholesale, not appended — with a Grill-Me task spec. Both variants embed the original task's deliverable description so the LLM still knows what schema the Resolved Summary must match.

**Interactive task spec (template):**

```
You will not produce the deliverable described below in one shot. Instead,
you will conduct an adversarial Socratic interrogation with the user to
stress-test the thinking behind the deliverable BEFORE producing it.

INTERROGATION RULES:
- Adopt the role of a relentless skeptic. You are NOT a supportive assistant.
- Ask exactly ONE question per turn. Wait for the user's answer before
  asking the next.
- Draw from these six question types as appropriate to each branch:
  1. Clarification ("What do you mean by X in this context?")
  2. Probing assumptions ("You assume Y — what evidence supports that?")
  3. Probing reasons ("Why is Z essential rather than nice-to-have?")
  4. Exploring viewpoints ("How would persona P evaluate this?")
  5. Examining implications ("If A happens, what then?")
  6. Meta-questioning ("Why are we focusing here and not elsewhere?")
- Refuse vague answers. Probe them with another question.
- Track which branches of the decision tree remain unresolved. Continue
  until every branch is resolved.
- When and only when every branch is resolved, emit a final RESOLVED SUMMARY
  conforming exactly to the OUTPUT TEMPLATE at the end of this prompt.
  Do not emit the summary earlier. Do not emit the summary alongside any
  more questions.

DELIVERABLE THE INTERROGATION TARGETS:
{original_task_specification}
```

**Auto task spec (template):**

```
You will simulate an adversarial Socratic interrogation internally — both
the skeptic's questions and the most plausible user answers — to stress-test
the thinking behind the deliverable described below. You will not output
any of the simulated dialogue. You will output ONLY a final RESOLVED SUMMARY
conforming exactly to the OUTPUT TEMPLATE at the end of this prompt.

SIMULATION RULES:
- Generate questions across the six Socratic question types (clarification,
  probing assumptions, probing reasons, exploring viewpoints, examining
  implications, meta-questioning).
- Produce plausible user answers grounded in the CONTEXT above.
- Challenge weak answers. Iterate until every branch resolves.
- Only then synthesize the RESOLVED SUMMARY.

DELIVERABLE THE SIMULATION TARGETS:
{original_task_specification}
```

`{original_task_specification}` is the same string the standard generation flow would have used as component [4]. The Grill-Me wrapper preserves it verbatim so the LLM knows the structure and depth the Resolved Summary must hit.

#### Where Grill-Me appears

| Prompt                          | Grill-Me availability                                                                 |
| ------------------------------- | ------------------------------------------------------------------------------------- |
| `phase_3_idea_seed_challenge`   | **Required** — both modes available, off is not an option                             |
| `phase_3_divergent_candidates`  | Optional — both modes available                                                       |
| `phase_4_design_grill`          | **Required** by design — the prompt has no non-Grill-Me variant; both modes available |
| `phase_6_judge_qa_drill`        | **Required** by design — both modes available                                         |
| `phase_2_problem_analysis`      | Optional — both modes available (per workflow doc, Phase 2 prompts may be toggled)    |
| `phase_2_competitive_premortem` | Optional — both modes available                                                       |
| All other prompts               | Not Grill-Me-eligible (`grill_me_modes` omitted)                                      |

The "required" markers above are enforced by the Prompt Generator card UI: for those prompts, the toggle does not have an "Off" position — the user must choose Interactive or Auto.

#### Phase Gate Log entry

Every Grill-Me generation appends a log entry of type `grill_me_session_initiated` carrying `{ prompt_id, mode, timestamp }`. On successful intake of the Resolved Summary, a `grill_me_session_resolved` entry is appended carrying `{ prompt_id, intake_id, timestamp }`. The Seed Challenge verdict (`adopt_as_seed | refined | rejected`) and Design Grill confirmed adjustments are also reflected in their respective intake writes per §6.

#### What Grill-Me does NOT change

- **Output schema.** The Resolved Summary still conforms to the same intake schema the standard one-shot prompt would produce. Intake processing is identical (§6).
- **Phase 4 gate semantics.** The Design Grill (`phase_4_design_grill`) does not cross the gate. The gate is crossed only after the three Phase 4 documents are generated, reviewed, and scope lock is explicitly confirmed (§5.2). Adjustments produced by the Design Grill are written into the PCD before the three document prompts are generated, so the documents reflect the stress-tested scope, but the gate logic is untouched.
- **Allowed transitions.** The state machine (§5.1) is unchanged. Grill-Me prompts run inside their phase, not as separate phases.

---

## 5. State Machine & Reconciliation

### 5.1 The phase state machine

Phases are nodes; allowed transitions are edges.

```
       ┌─────┐
       │  0  │  (Vault-level, ongoing — not part of project state)
       └─────┘

  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
  │  1   │───▶│  2   │───▶│  3   │───▶│  4   │═══▶│  5   │───▶│  7   │
  └──────┘    └──────┘    └──────┘    └──────┘    └──────┘    └──────┘
     ▲           ▲           ▲           │           │
     │           │           │           │           │
     └───────────┴───────────┘           │           │
       (backward navigation              │           ▼
        allowed pre-gate)                │        ┌──────┐
                                         └───────▶│  6   │ (parallel with 5)
                                                  └──────┘

  ═══▶ = one-way gate (Phase 4 → 5). Crossing it is permanent.
        Post-gate scope changes go through the Addendum Protocol only.
```

#### Allowed transitions

```ts
const ALLOWED_TRANSITIONS: Record<number, number[]> = {
  1: [2],                  // forward only from 1 (no phase 0 navigation)
  2: [1, 3],               // backward to 1 or forward to 3
  3: [1, 2, 4],            // backward to 1 or 2, forward to 4
  4: [5],                  // ONE-WAY GATE: forward only, ever
  5: [6, 7],               // forward to 6 (parallel) or 7 (post-event)
  6: [5, 7],               // can return focus to 5; forward to 7
  7: [],                   // terminal
};
```

Backward navigation pre-gate triggers reconciliation (§5.3). Forward navigation requires phase completion checks (§5.2).

### 5.2 Phase completion gates

Each phase has a `PhaseCompletionRequirement`:

```ts
type PhaseCompletionRequirement = {
  phase: number;
  required_sections_populated: string[];   // section keys that must have is_populated=true
  required_flags: string[];                // e.g. 'competitive_landscape.data.premortem_completed'
  user_acknowledgments: string[];          // explicit confirmation gates
  override_allowed: boolean;               // if true, override requires written justification
};
```

| Phase | Required Populated                                                                                                             | Required Flags                                                                                                                                                                                                                                                                       | Acknowledgments                                                                                                                                                                                                     | Override                                   |
| ----- | ------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| 1     | `hackathon_profile`, `judge_intelligence`, `active_judging_rubric`                                                             | —                                                                                                                                                                                                                                                                                    | Acknowledge all `flagged_unknowns`                                                                                                                                                                                  | Yes                                        |
| 2     | `problem_analysis`, `competitive_landscape`                                                                                    | `competitive_landscape.data.premortem_completed === true`                                                                                                                                                                                                                            | At least one `differentiation_angles` entry exists                                                                                                                                                                  | **No** (the only hard block in the system) |
| 3     | `ideation_record`, `chosen_direction`                                                                                          | At least one of `ideation_record.data.scamper_outputs`, `ideation_record.data.jtbd_analysis`, `ideation_record.data.brainwriting_round` is populated; `ideation_record.data.candidate_portfolio.candidates` contains ≥ 3 entries; `chosen_direction.data.selected_idea` is non-empty | If `premortem_overlap_check.overlap_detected === true`, then `user_acknowledged_overlap === true` with written rationale in log. Step 3a (Idea Seed Challenge) is **optional** and does not block phase completion. | Yes (premortem overlap only)               |
| 4     | `product_brief`, `technical_implementation_document`, `demo_script_and_backup_plan`, `scope_definition`, `technical_direction` | TID has `screens` array non-empty; demo script has `backup_plan`                                                                                                                                                                                                                     | Phase 4 review checklist completed; **scope lock explicitly confirmed** (typed phrase, not a button)                                                                                                                | No                                         |
| 5     | —                                                                                                                              | —                                                                                                                                                                                                                                                                                    | —                                                                                                                                                                                                                   | — (no gate; ends at submission)            |
| 6     | `pitch_brief`, `judge_objection_map`, `qa_prep_document`                                                                       | —                                                                                                                                                                                                                                                                                    | Presentation Readiness Checklist complete                                                                                                                                                                           | Yes                                        |
| 7     | `debrief_record`                                                                                                               | —                                                                                                                                                                                                                                                                                    | —                                                                                                                                                                                                                   | —                                          |

Phase 2 is the only hard block. Phase 4 is a one-way gate but its completion can be deferred indefinitely — what cannot happen is crossing it without all conditions met.

### 5.3 Backward navigation and the dependency graph

When the user navigates from phase N back to phase M (where M < N), the system must determine which downstream sections become potentially stale. This is done via a **section-level dependency graph**, not phase-level.

#### The dependency graph (constant)

```ts
const PCD_DEPENDENCY_GRAPH: Record<string, string[]> = {
  // section → sections it depends on (i.e., upstream sections)

  hackathon_profile: [],
  judge_intelligence: ['hackathon_profile'],
  active_judging_rubric: ['hackathon_profile'],

  problem_analysis: ['hackathon_profile'],
  competitive_landscape: ['hackathon_profile', 'problem_analysis'],
  user_research: ['hackathon_profile', 'problem_analysis'],

  ideation_record: ['problem_analysis', 'competitive_landscape', 'user_research', 'active_judging_rubric'],
  chosen_direction: ['ideation_record', 'competitive_landscape'],

  scope_definition: ['chosen_direction', 'hackathon_profile'],
  technical_direction: ['chosen_direction', 'scope_definition'],

  // Phase 4 outputs — depend on upstream but become STALE-EXEMPT after Phase 4 gate
  product_brief: ['chosen_direction', 'problem_analysis', 'hackathon_profile'],
  technical_implementation_document: ['chosen_direction', 'scope_definition', 'technical_direction'],
  demo_script_and_backup_plan: ['chosen_direction', 'scope_definition', 'technical_implementation_document'],

  // Phase 6 outputs — exempt from staleness graph by design (living documents)
  pitch_brief: [],
  judge_objection_map: [],
  qa_prep_document: [],

  debrief_record: [],

  // User-authored free-text — never stale-tracked. Always at the leaves of the graph.
  external_context_notes: [],
};
```

#### Staleness propagation algorithm

When a section S is updated (via intake, manual edit, or backward navigation):

```
function propagateStaleness(updatedSection: string, pcd: PCD): void {
  // 1. Determine which sections depend on the updated section, transitively
  const downstream = transitiveClosure(updatedSection, INVERTED_DEPENDENCY_GRAPH);

  // 2. For each downstream section:
  for (const section of downstream) {
    // Phase 4 gate exemption: if gate is crossed AND section was populated by phase 4,
    // do not mark stale. Scope changes go through Addendum Protocol only.
    if (pcd.phase_4_gate_crossed && PHASE_4_OUTPUT_SECTIONS.includes(section)) {
      continue;
    }

    // Phase 6 exemption: pitch outputs are never stale-tracked (by design).
    if (PHASE_6_OUTPUT_SECTIONS.includes(section)) {
      continue;
    }

    // Skip if section isn't even populated yet — nothing to be stale.
    if (!pcd.sections[section].is_populated) continue;

    // Mark stale with a specific reason.
    pcd.sections[section].is_stale = true;
    pcd.sections[section].stale_reason =
      `${updatedSection} was updated on ${now()}; this section was derived from the previous version.`;
  }
}
```

The two exemptions are critical:

1. **Phase 4 gate exemption** preserves the sanctity of the gate. Once crossed, scope is locked, period. Upstream changes after the gate cannot mark Phase 4 outputs stale — they would otherwise create a backdoor scope-change path that bypasses the Addendum Protocol.
2. **Phase 6 exemption** reflects the design decision that pitch outputs are living documents, manually revised by the user as the build progresses. Stale-tracking would create useless friction.

### 5.4 Resolving stale sections

A stale section blocks no individual operation, but **phase advancement is gated on no upstream stale sections existing**. The user cannot move from Phase 3 to Phase 4 if any section depended on by Phase 3 outputs (i.e., any of `problem_analysis`, `competitive_landscape`, `user_research`, `active_judging_rubric`, `ideation_record`) is stale.

The user resolves staleness via three actions, surfaced in the section's view:

#### Resolution 1: Regenerate

The system re-runs the original phase prompt with current upstream context, the user runs it externally, pastes back, and the section is replaced. The system tracks this as `staleness_resolved_regenerate` in the Phase Gate Log.

#### Resolution 2: Confirm-as-still-valid

The user reviews the upstream change and decides this section doesn't need updating. One click; staleness flag clears; Phase Gate Log records `staleness_resolved_confirm_valid` with timestamp.

#### Resolution 3: Manual edit

The user opens the section in an editor view and makes targeted changes. On save, `is_stale` clears and `user_overrides` gets a new entry. Phase Gate Log records `staleness_resolved_manual_edit`.

The Phase Navigator and the PCD Viewer both surface stale-section counts visibly. Stale sections are highlighted with a clear visual treatment (color + icon) so the user always knows where reconciliation is needed.

### 5.5 The Addendum Protocol

Triggered only when `phase_4_gate_crossed === true`. Gives the user a controlled, friction-heavy path to make a scope change.

```
User clicks "Initiate Addendum Protocol"
    │
    ▼
Modal: "Scope is locked. Changes carry risk. This action is permanently logged.
       Continue?"
    │
    ▼
Form requires:
  - Pasted Cursor build-state markdown export
  - Written justification (min 80 chars, validated client-side)
  - Description of proposed change
    │
    ▼
System generates Addendum Prompt for external LLM
    │
    ▼
User runs externally, pastes back
    │
    ▼
System extracts via the addendum intake schema (§6)
    │
    ▼
Append to pcd.addenda[]
Append "addendum_created" event to phase_gate_log
    │
    ▼
Original TID is preserved unchanged. Addendum is shown alongside it
in the TID viewer with a visual separator.
```

The original TID is **never edited** post-gate. Addenda are append-only annotations.

### 5.6 The Phase Gate Log as the source of truth

When in doubt about what happened, what was overridden, what was acknowledged, what was reconciled — the Phase Gate Log is the answer. It is append-only, timestamped, and exported with the PCD. This is also the primary input to the Phase 7 Debrief.

---

## 6. Intake Parsing

This is where Gemini Flash earns its keep. Intake parsing converts pasted external-LLM output (raw markdown/text) into typed PCD section data.

### 6.1 The three-layer reliability model

Reliability comes from three layers, applied in order. Each layer makes the next one's job smaller.

**Layer 1: Structured prompt outputs.** Every Startup-generated prompt mandates an exact output template (§4.1, component 5). When the user pastes back, the input is already roughly shaped like the schema. Extraction is mostly mapping-by-header, not free-form parsing.

**Layer 2: Native structured output mode.** The Flash API is called with `response_mime_type: "application/json"` and a `response_schema` matching the target intake schema. The model literally cannot return non-conforming JSON — the decoder enforces it. This eliminates "model hallucinated a field name" bugs entirely.

**Layer 3: Three-tier failure handling.** Even with the above, extraction can fall short. The system grades every result and surfaces it to the user appropriately.

### 6.2 The `IntakeDefinition` registry

Every intake the system can perform is registered:

```ts
type IntakeDefinition = {
  id: string;                                      // matches PromptDefinition.produces_intake_id
  display_name: string;
  target_pcd_path: string;                         // e.g. 'sections.ideation_record.data.scamper_outputs'
  target_schema_fragment: JSONSchema;              // the schema the extracted JSON must conform to
  extraction_prompt_template: string;              // what we tell Flash to do
  required_fields: string[];                       // dotted paths within the extracted object that must be non-empty
  optional_fields: string[];
  confidence_threshold: number;                    // below this, surface "review required" UI; default 0.7
  commit_strategy: 'direct' | 'apply_adjustments'; // how the extracted object reaches target_pcd_path. See below.
  post_extract_validators?: PostExtractValidator[];// per-intake rules that schema cannot express. See below.
};

type PostExtractValidator =
  | {
      rule: 'no_null_at';
      path: string;          // dotted path with [] to denote arrays, e.g. 'anticipated_questions[].associated_persona'
      severity: 'flag' | 'fail';
      message: string;       // human-readable; surfaced in partial-review UI when rule trips
    };
// Vocabulary intentionally minimal. Add new rule kinds here as concrete cases emerge,
// not speculatively. Each rule is declarative so the registry stays a pure data structure.
```

Like the PromptDefinition registry, this is a static TypeScript constant. Adding a new intake = adding an entry.

**`commit_strategy` semantics.** Most intakes are `'direct'`: the object Flash extracts is the data the diff preview proposes against `target_pcd_path` (one section, one diff). The exception is `design_grill_intake`, which is `'apply_adjustments'`: Flash extracts a list of `{field_path, proposed_value, rationale}` adjustments validated against an auxiliary schema (`chosenDirectionAdjustmentsSchema`, defined below); the diff preview shows one row **per adjustment**, each resolving against `target_pcd_path` joined with the adjustment's `field_path`. The user confirms or dismisses each row independently — partial commits are explicitly supported. This is what the workflow doc means by "you confirm or dismiss each adjustment." Future intakes that produce patches rather than wholesale section payloads MAY adopt `'apply_adjustments'` and provide their own auxiliary schema; the commit handler's switch is on this field, not on the intake `id`.

**`post_extract_validators` semantics.** JSON Schema sometimes can't express a rule that holds for one intake but not another targeting the same path. The canonical case in v1.5 is `associated_persona` on Q&A entries: the schema (`Section_QAPrep`) allows `null` because the standard `intake_phase_6_qa_prep` permits it, but `judge_qa_drill_intake` forbids it (the drill prompt's negative constraint: "Do NOT produce questions without an `associated_persona`"). Schema can't enforce that; the schema is shared across both intakes. `post_extract_validators` is the seam. Each validator runs after schema validation passes, before the diff preview opens. A `'flag'` severity routes the intake into Tier 2 (partial review) regardless of whether schema validation and `required_fields` were clean — the flagged paths render as editable rows so the user can fill them before commit. A `'fail'` severity routes to Tier 3 (manual entry) and is reserved for invariant violations that should never auto-commit. The vocabulary is deliberately narrow: only `'no_null_at'` exists in v1.5; new rule kinds are added when a real case appears, not on speculation.

#### v1.5 intake registry additions

The pre-existing intakes for SCAMPER, JTBD, Brainwriting, Differentiation Stress Test, and Downselection are **retained unchanged** — their target paths, schemas, required fields, and confidence thresholds are not modified. The four additions below correspond to the four new prompts in §4.2:

```ts
{
  id: 'idea_seed_challenge_intake',
  display_name: 'Idea Seed Challenge — Resolved Summary',
  target_pcd_path: 'sections.ideation_record.data.team_idea_seed',
  // schema fragment in §2.6: { original_text, challenge_outcome, refined_text, verdict_rationale }
  target_schema_fragment: ideationRecordSchema.properties.data.properties.team_idea_seed,
  extraction_prompt_template:
    'Extract the Resolved Summary of the Idea Seed Challenge into the target schema. ' +
    'challenge_outcome must be exactly one of: "adopt_as_seed", "refined", "rejected". ' +
    'refined_text is required only when challenge_outcome is "refined"; otherwise null.',
  required_fields: ['original_text', 'challenge_outcome', 'verdict_rationale'],
  optional_fields: ['refined_text'],
  confidence_threshold: 0.7,
  commit_strategy: 'direct',
}

{
  id: 'divergent_candidates_intake',
  display_name: 'Divergent Candidates — Portfolio',
  target_pcd_path: 'sections.ideation_record.data.candidate_portfolio',
  target_schema_fragment: ideationRecordSchema.properties.data.properties.candidate_portfolio,
  extraction_prompt_template:
    'Extract the candidate portfolio. Each candidate must include name, one-line description, ' +
    'integer scores 1-5 for innovation_score, impact_score, feasibility_score, archetype ' +
    '(one of safe_utility | weird_gem | moonshot | social_impact | unassigned), and a non-empty ' +
    'distinguishing_factors array. Do NOT emit id, is_remix, or remix_source_ids — these fields ' +
    'are populated by the app post-extract before schema validation runs (see the post-extract ' +
    'enrichment note below the registry).',
  required_fields: ['candidates'],   // array length validated separately (3-5)
  optional_fields: [],
  confidence_threshold: 0.7,
  commit_strategy: 'direct',
}

{
  id: 'design_grill_intake',
  display_name: 'Design Grill — Confirmed Adjustments',
  target_pcd_path: 'sections.chosen_direction.data',
  // Auxiliary schema. Flash output is a list of adjustment records; each
  // adjustment's field_path resolves against target_pcd_path at commit time.
  // See `chosenDirectionAdjustmentsSchema` below this block for the full
  // schema definition.
  target_schema_fragment: chosenDirectionAdjustmentsSchema,
  extraction_prompt_template:
    'Extract the Resolved Summary as a list of confirmed adjustments to the chosen direction. ' +
    'Each adjustment names the field_path within chosen_direction.data it targets ' +
    '(one of: selected_idea, user_rationale, key_differentiators, key_differentiators[+], ' +
    'or key_differentiators[-]) and the proposed_value (string for scalar fields and for ' +
    '[+]/[-] suffixes; array of strings only when field_path is the bare key_differentiators). ' +
    'Also extract the interrogation_summary paragraph. Adjustments are surfaced to the user ' +
    'for per-item confirm/dismiss in the diff preview before commit; an empty adjustments ' +
    'array is a valid outcome and means the chosen direction survived intact.',
  required_fields: ['adjustments', 'interrogation_summary'],
  optional_fields: [],
  confidence_threshold: 0.7,
  commit_strategy: 'apply_adjustments',
}

{
  id: 'judge_qa_drill_intake',
  display_name: 'Judge Q&A Drill — Sharpened Q&A',
  target_pcd_path: 'sections.qa_prep_document.data',
  target_schema_fragment: qaPrepDocumentSchema.properties.data,
  extraction_prompt_template:
    'Extract the sharpened Q&A document from the Resolved Summary. Each entry must include ' +
    'question, recommended_answer, and fallback_answer; associated_persona is REQUIRED for this ' +
    'intake (the drill prompt forbids null personas — every drill question is asked by a specific ' +
    'judge persona). On commit, the user chooses (in the diff preview UI) whether the result ' +
    'REPLACES or SUPPLEMENTS the existing qa_prep_document — see §6.5 for replace-vs-merge semantics.',
  required_fields: ['anticipated_questions'],
  optional_fields: [],
  confidence_threshold: 0.7,
  commit_strategy: 'direct',
  post_extract_validators: [
    {
      rule: 'no_null_at',
      path: 'anticipated_questions[].associated_persona',
      severity: 'flag',
      message:
        'The Judge Q&A Drill requires every question to name the judge persona that asked it. ' +
        'Entries with a null associated_persona are flagged for manual fill before commit. ' +
        '(The shared qa_prep_document schema permits null because the standard one-shot QA Prep ' +
        'intake allows aggregate questions; the drill does not.)',
    },
  ],
}
```

#### Auxiliary schema: `chosenDirectionAdjustmentsSchema`

This schema lives alongside the intake registry (e.g., in `frontend/src/lib/intake/schemas.ts`) and is exported as the `target_schema_fragment` for `design_grill_intake`. It is **not** part of `pcd_schema.json` because it never appears in a committed PCD instance — it shapes the auxiliary patch object Flash extracts, which the commit handler then translates into per-field writes against `chosen_direction.data` per the `apply_adjustments` strategy.

```json
{
  "$id": "chosenDirectionAdjustmentsSchema",
  "type": "object",
  "required": ["adjustments", "interrogation_summary"],
  "additionalProperties": false,
  "properties": {
    "adjustments": {
      "type": "array",
      "description": "Proposed adjustments to chosen_direction.data. May be empty when the interrogation surfaced none — an empty list is a valid Resolved Summary outcome.",
      "items": {
        "type": "object",
        "required": ["field_path", "proposed_value", "rationale"],
        "additionalProperties": false,
        "properties": {
          "field_path": {
            "type": "string",
            "enum": [
              "selected_idea",
              "user_rationale",
              "key_differentiators",
              "key_differentiators[+]",
              "key_differentiators[-]"
            ],
            "description": "Target field within chosen_direction.data. Suffix '[+]' means append proposed_value to the array; '[-]' means remove the proposed_value entry by exact string match. No suffix on the array field 'key_differentiators' means wholesale replacement. Scalar fields (selected_idea, user_rationale) are always replaced. premortem_overlap_check and selected_candidate_id are deliberately not in this enum — they carry phase-earlier provenance and are out of bounds for Design Grill adjustments (enforced at prompt level by negative constraint, at schema level by enum closure)."
          },
          "proposed_value": {
            "oneOf": [
              { "type": "string" },
              { "type": "array", "items": { "type": "string" } }
            ],
            "description": "New value. Schema accepts string-or-array-of-strings; the type narrowing depends on field_path and is enforced at commit time, not at schema validation, because Flash extraction is fuzzy and the per-field-path narrowing is cleaner expressed in code than in JSON Schema if/then. Commit-time rule: scalar field_paths and the [+]/[-] suffixed forms require string; the bare 'key_differentiators' field_path requires array."
          },
          "rationale": {
            "type": "string",
            "minLength": 1,
            "description": "1-3 sentences explaining the specific finding from the interrogation that produced this adjustment. Empty rationale is rejected because adjustments without rationale dissolve at the diff preview step."
          }
        }
      }
    },
    "interrogation_summary": {
      "type": "string",
      "minLength": 1,
      "description": "One paragraph naming which branches were tested and the headline finding from each. Required even when adjustments is empty — captures provenance for the Phase Gate Log so a debrief can answer 'what did the Design Grill actually surface?' even on a no-adjustments outcome."
    }
  }
}
```

**Commit-time field_path resolution.** The intake handler walks `adjustments[]` and, for each entry, computes the diff-preview row:

| `field_path`                | Action                | Resolved write                                                              |
| --------------------------- | --------------------- | --------------------------------------------------------------------------- |
| `selected_idea`             | replace scalar        | `chosen_direction.data.selected_idea = proposed_value`                      |
| `user_rationale`            | replace scalar        | `chosen_direction.data.user_rationale = proposed_value`                     |
| `key_differentiators`       | replace whole array   | `chosen_direction.data.key_differentiators = proposed_value`                |
| `key_differentiators[+]`    | append item           | `chosen_direction.data.key_differentiators.push(proposed_value)`            |
| `key_differentiators[-]`    | remove by exact match | `chosen_direction.data.key_differentiators = filter(x => x !== proposed_value)` — if no match found, the diff row is flagged "no-op" and the user is asked whether to keep, edit, or dismiss |

The user confirms or dismisses each row independently in the diff preview. Dismissed adjustments are written into the Phase Gate Log (`grill_me_adjustment_dismissed` entries with the full record) but not applied. Confirmed adjustments commit before the three Phase 4 document prompts run (§5.2), so those documents reflect the stress-tested scope.

**Why these five field_paths and no others.** The Design Grill operates on the *product decision* — what the thing is, why it was chosen, what makes it different. Those map exactly to `selected_idea`, `user_rationale`, and `key_differentiators`. `premortem_overlap_check` is provenance from Phase 2's competitive analysis and shouldn't be retroactively edited via Grill-Me. `selected_candidate_id` is provenance from Step 3e's candidate portfolio. Letting the Design Grill rewrite either would silently break the project's audit trail. If the interrogation surfaces a finding that the chosen direction overlaps with the predicted average team's space (which `premortem_overlap_check` records), the right action is to revisit Phase 3, not paper over it with a post-hoc field rewrite — the state machine (§5.1) supports backward navigation specifically for this case.

`scope_definition.data` and `product_brief.data` are not addressable via this schema because they are empty at Step 4a — the three Phase 4 documents have not been generated yet. If a future workflow change moved Design Grill to run *after* the documents (which would change the gate semantics and is out of scope for v1.5), the field_path enum would expand and `target_pcd_path` would change accordingly.

For `judge_qa_drill_intake`, the diff preview offers an explicit "Replace existing" / "Supplement existing" toggle when `qa_prep_document` is already populated — matching the user choice the workflow doc describes for the post-drill intake.

#### Post-extract enrichment for `divergent_candidates_intake`

The `candidate_portfolio.candidates[]` schema requires `id` (a stable identifier referenced by `chosen_direction.data.selected_candidate_id`), and the TypeScript shape in §2.6 additionally specifies `is_remix` and `remix_source_ids` for runtime consistency. Flash is instructed to omit all three. Between Flash's structured output and schema validation against `target_schema_fragment`, the intake handler runs an enrichment pass on each extracted candidate:

- **`id`** — assigned a fresh ULID using the `ulid` npm package. ULIDs are minted app-side because (a) LLMs do not produce valid Crockford-base32-encoded ULIDs reliably; the strings they emit may pass a length check but fail the alphabet check or the timestamp ordering invariant, (b) generation must be deterministic and testable, and (c) the time-ordered property is only meaningful when ids are minted at intake time, not at LLM generation time. Required for schema validation to pass — without it the `required: ["id", ...]` check fails.
- **`is_remix`** — set to `false`. Remixes are produced exclusively via the in-app candidate-review UI (Phase 3 workspace, see §11.2 milestone 6); by construction, anything the LLM returned is a non-remix. Not required by the schema, but enriched for TS-shape consistency.
- **`remix_source_ids`** — set to `[]` for the same reason.

Schema validation runs against the **enriched** object. Any failure mode of enrichment (e.g., the ULID library throwing) is treated as a hard intake error and surfaced to the user as a failed extraction; no partial writes occur.

This is the only v1.5 intake that uses post-extract enrichment. The other three additions either have no app-managed fields (`idea_seed_challenge_intake`, `judge_qa_drill_intake`) or write through an auxiliary schema where identity is not app-minted (`design_grill_intake` — adjustments are reviewed and applied by `field_path`, not stored under stable ids). When future intakes introduce app-managed identifiers, they SHOULD follow the same pattern: instruct Flash to omit the fields, enrich post-extract, validate enriched.

### 6.3 The Flash extraction prompt

Flash is invoked with a prompt that has the same shape every time, parameterized by the IntakeDefinition:

```
You are a structured data extractor. Your only job is to extract information
from the input text below into JSON conforming to the schema attached to this
request via response_schema.

EXTRACTION RULES:
- Map content to fields as faithfully as possible.
- If a required field cannot be confidently extracted, set it to its empty
  type (empty string, empty array) — do NOT invent content.
- For each top-level field you extract, also report a confidence value
  between 0 and 1 in the parallel `_confidence` object (also defined in
  the schema).
- Do not add commentary. Output JSON only.

INPUT TEXT TO EXTRACT FROM:
---
{pasted_user_input}
---
```

The `_confidence` object is part of the response_schema for every intake. It mirrors the structure of the data fields but holds floats. The system reads it to apply the confidence threshold.

Example response shape for SCAMPER:

```json
{
  "data": {
    "substitute": ["...", "..."],
    "combine": ["...", "..."],
    "...": "..."
  },
  "_confidence": {
    "substitute": 0.95,
    "combine": 0.88,
    "...": "..."
  }
}
```

### 6.4 The three-tier failure handling

After Flash returns:

```
function processIntake(result: FlashResult, def: IntakeDefinition): IntakeOutcome {
  // Tier 0: Flash error or schema validation failure
  if (result.error || !validateAgainstSchema(result.data, def.target_schema_fragment)) {
    return { tier: 'failed', raw: result.raw, error: result.error };
  }

  // Run per-intake post-extract validators (declarative rules schema can't express).
  // A 'fail' severity routes to Tier 3; a 'flag' severity feeds the flagged paths
  // into Tier 2 alongside the standard required/confidence checks.
  const validatorResults = (def.post_extract_validators ?? [])
    .map(v => ({ v, result: runValidator(result.data, v) }))
    .filter(({ result }) => !result.ok);

  if (validatorResults.some(({ v }) => v.severity === 'fail')) {
    return {
      tier: 'failed',
      raw: result.raw,
      error: validatorResults.find(({ v }) => v.severity === 'fail')!.v.message,
    };
  }

  const validatorFlaggedPaths = validatorResults
    .filter(({ v }) => v.severity === 'flag')
    .flatMap(({ result }) => result.flagged_paths);

  // Compute aggregate confidence and missing-required-field check
  const missingRequired = def.required_fields.filter(f => isEmpty(result.data, f));
  const lowConfidenceFields = Object.entries(result.confidence)
    .filter(([_, c]) => c < def.confidence_threshold)
    .map(([f, _]) => f);

  if (
    missingRequired.length === 0 &&
    lowConfidenceFields.length === 0 &&
    validatorFlaggedPaths.length === 0
  ) {
    return { tier: 'clean', data: result.data };
  }

  return {
    tier: 'partial',
    data: result.data,
    missingRequired,
    lowConfidenceFields,
    validatorFlaggedPaths,
  };
}
```

#### Tier 1: Clean extract

All required fields present, all confidences above threshold. The system shows a **diff preview** of what will be added to the PCD: which sections are being touched, what's being added vs. replaced. The user clicks "Confirm" to commit. Single click, low friction, full transparency.

#### Tier 2: Partial extract

Some required fields missing OR some fields below confidence threshold OR a `'flag'`-severity `post_extract_validator` tripped. The system shows the same diff preview, but with the missing/low-confidence/flagged fields highlighted and inline-editable. The user fills gaps manually before confirming. This is the most common case for messy paste-back; designing for it is essential. Validator-flagged rows display the validator's `message` as the inline hint so the user knows *why* the field is being asked for despite passing schema validation.

#### Tier 3: Failed extract

Schema validation failed entirely OR Flash errored. The system does NOT write to the PCD. Instead, it shows a manual entry form pre-populated with the raw paste in a side panel for reference, and lets the user enter the data field-by-field. The system also offers a **"Try Again"** button that resubmits to Flash with a stricter retry prompt:

```
The previous extraction attempt failed. Pay especially close attention to
the schema. Required fields: {list}. Common mistakes to avoid: empty arrays
when content is present, wrong nesting depth, fields outside the schema.
```

If retry also fails, manual entry is the only path. This is acceptable — manual entry is rare, and the worst outcome is corrupted PCD state, which manual entry avoids.

### 6.5 The diff preview — never silent writes

Every successful intake produces a proposed diff against current PCD state. The diff shows:

- **Sections being touched** (highlighted)
- **Fields being added** (green)
- **Fields being replaced** (yellow, with old value visible)
- **Fields being cleared** (red, rare; explicit user action only)

The user clicks "Apply Changes" to commit. The diff preview cannot be skipped. This is one click of friction in exchange for the system being debuggable and trustworthy.

When a section already has content (e.g., the user is re-running an intake on a section that was previously populated), the diff shows replacement explicitly. The user can opt to merge instead of replace where it makes sense — for arrays, this means concatenation; for objects, field-level union with old-wins-on-conflict by default.

### 6.6 Intake calls the Flash proxy

Flash is never called from the browser directly. The Cloudflare Worker proxy receives the intake request, attaches the API key, calls Flash, and returns the result. The proxy is stateless — it does not log requests beyond what's needed for rate limiting and error tracking.

```
Browser → POST /api/extract
  body: { intake_id, raw_input, schema, prompt }
       │
       ▼
Cloudflare Worker
  - Validate request shape
  - Add API key from Worker secrets
  - Call Gemini Flash with response_mime_type and response_schema
  - Return { data, confidence, raw, error? }
       │
       ▼
Browser → process via three-tier handler
       │
       ▼
Diff preview → user confirms → commit to IndexedDB
```

The proxy has no other endpoints in MVP. Future endpoints (e.g., for telemetry or multi-user sync) are out of scope.

### 6.7 Intake idempotency

Re-running an intake on the same input must produce the same result (modulo Flash temperature). The system does not memoize across runs — each paste-back is a fresh extraction. This is the simplest correct behavior and has no observable downside given the manual nature of the workflow.

---

## 7. Default Judging Rubric

The full rubric content lives in `default_judging_rubric.md`. This section specifies how it is used by the application.

### 7.1 Loading

At project creation, the application reads `default_judging_rubric.md`, parses its weighted criteria into the JSON shape defined in `Section_ActiveJudgingRubric`, and writes it to `pcd.sections.active_judging_rubric.data` with `rubric_source: 'default'`.

A simple parser extracts criteria from the markdown file by walking H2 headers (`## Criterion N — Name`). The parser is deterministic and lives in `src/lib/rubric/parseDefault.ts`. It is run at app startup once and cached, then re-run on settings change.

### 7.2 Replacing with official criteria

When the user pastes official judging criteria during Phase 1, the system runs an intake — the Rubric Mapping Intake. Flash is asked to map each official criterion onto the default rubric schema (name, description, weight, judge_questions), and to flag any official criteria that don't map cleanly.

The user reviews the mapping and selects:

- **Replace**: official criteria become the active rubric. `rubric_source: 'official'`.
- **Supplement**: unmapped official criteria are added to the default rubric; weights are renormalized to sum to 1.0. `rubric_source: 'official_with_default_supplement'`.
- **Keep default**: discard the mapping. `rubric_source: 'default'`. Logged with rationale.

### 7.3 User customization of the default

The user can edit the default rubric via Settings (JSON-configurable, no bespoke UI in MVP). Changes apply to all newly-created projects from that point forward. Existing projects retain their snapshotted rubric.

Validation: weights must sum to 1.0 (±0.001), each criterion needs name + description + weight + at least one judge_question, names must be unique. On invalid save, the form rejects with a specific error.

---

## 8. Application Architecture

### 8.1 Stack

```
Frontend:           React 18 + Vite + TypeScript (strict mode)
Routing:            react-router-dom v6
State:              Zustand (with persist middleware for non-PCD UI state)
Persistence:        IndexedDB via Dexie.js
Encryption:         Web Crypto API (AES-GCM, PBKDF2 key derivation)
Schema validation:  Ajv (JSON Schema draft 2020-12)
Markdown render:    marked (display) + custom JSON-to-Markdown function (export)
Styling:            Tailwind CSS
Icons:              lucide-react
Forms:              react-hook-form + zod for non-PCD forms

Backend (proxy):    Cloudflare Workers (TypeScript)
                    Single endpoint: POST /api/extract
                    Secrets: GEMINI_API_KEY

Hosting:            Cloudflare Pages (frontend) + Cloudflare Workers (proxy)
                    Both free tier. Single ecosystem.
```

### 8.2 Directory structure

```
startup/
├── frontend/
│   ├── src/
│   │   ├── app/                     # React app shell, routes
│   │   ├── screens/                 # Top-level screens (one folder per screen in §9)
│   │   ├── components/              # Shared UI components
│   │   ├── lib/
│   │   │   ├── pcd/                 # PCD types, schema validation, render
│   │   │   │   ├── schema.ts        # imports pcd_schema.json, exposes typed shape
│   │   │   │   ├── createEmptyPCD.ts
│   │   │   │   ├── render.ts        # JSON → Markdown
│   │   │   │   └── validate.ts      # Ajv wrapper
│   │   │   ├── vault/               # Vault types, queries, persistence
│   │   │   ├── prompts/
│   │   │   │   ├── registry.ts      # PromptDefinition[]
│   │   │   │   ├── generate.ts      # context selection, template assembly
│   │   │   │   ├── schemaToTemplate.ts
│   │   │   │   └── templates/       # task specification templates per phase
│   │   │   ├── intake/
│   │   │   │   ├── registry.ts      # IntakeDefinition[]
│   │   │   │   ├── extract.ts       # calls proxy
│   │   │   │   ├── handle.ts        # three-tier outcome
│   │   │   │   └── diff.ts          # diff preview computation
│   │   │   ├── rubric/
│   │   │   │   ├── parseDefault.ts  # markdown → JSON
│   │   │   │   └── default.md       # bundled copy of default_judging_rubric.md
│   │   │   ├── stateMachine/
│   │   │   │   ├── transitions.ts   # ALLOWED_TRANSITIONS
│   │   │   │   ├── completion.ts    # PhaseCompletionRequirement[]
│   │   │   │   ├── dependencies.ts  # PCD_DEPENDENCY_GRAPH
│   │   │   │   └── reconcile.ts     # propagateStaleness
│   │   │   ├── persistence/
│   │   │   │   ├── db.ts            # Dexie schema
│   │   │   │   ├── crypto.ts        # encryption layer
│   │   │   │   └── migrate.ts       # schema_version migrations
│   │   │   └── proxy/
│   │   │       └── client.ts        # fetch wrapper for Worker
│   │   └── stores/                  # Zustand stores
│   ├── public/
│   ├── package.json
│   ├── vite.config.ts
│   └── tailwind.config.ts
└── proxy/
    ├── src/
    │   └── worker.ts                # Cloudflare Worker, single /api/extract endpoint
    ├── wrangler.toml
    └── package.json
```

### 8.3 Data persistence layer

```ts
// frontend/src/lib/persistence/db.ts
import Dexie from 'dexie';

class StartupDB extends Dexie {
  projects!: Dexie.Table<EncryptedProject, string>;
  vault_judges_organizers!: Dexie.Table<EncryptedVaultEntry, string>;
  vault_winning_solutions!: Dexie.Table<EncryptedVaultEntry, string>;
  vault_boilerplate!: Dexie.Table<EncryptedVaultEntry, string>;
  settings!: Dexie.Table<EncryptedSettings, string>;

  constructor() {
    super('StartupDB');
    this.version(1).stores({
      projects: 'id, project_name, current_phase, created_at, archived_at',
      vault_judges_organizers: 'id, name, organization, *tags',
      vault_winning_solutions: 'id, event_name, organizer, domain, year, *tags',
      vault_boilerplate: 'id, component_name, category, *tags',
      settings: 'key',
    });
  }
}

type EncryptedProject = {
  id: string;
  project_name: string;          // searchable indexes are stored unencrypted
  current_phase: number;
  created_at: string;
  archived_at: string | null;
  ciphertext: ArrayBuffer;       // AES-GCM ciphertext of the full PCD JSON
  iv: ArrayBuffer;               // initialization vector
};
```

**What is encrypted vs not:**

- The PCD content blob is encrypted.
- IndexedDB indexes (project name, phase, dates, tags) are stored in plaintext to enable querying without decrypting every record.

This is a deliberate trade-off. An adversary with raw IndexedDB access could see "you have a project called X in Phase 3" but not its contents. Acceptable for a personal-tool threat model.

### 8.4 Encryption flow

```
First app run:
  - User sets a local password (min 12 chars, strength meter shown)
  - Password is run through PBKDF2 (210,000 iterations, SHA-256, salt stored
    unencrypted in `settings`)
  - Derived key is held in memory only — never persisted
  - Verification token (encrypted constant) is written to `settings` so future
    runs can validate the password is correct without decrypting any project

Every app open:
  - Password prompt (cannot dismiss)
  - Derive key, decrypt verification token, validate
  - Hold derived key in memory for the session

Every PCD read:  decrypt project ciphertext with session key
Every PCD write: re-encrypt project ciphertext with session key, write back

Lock action:     wipe key from memory, return to password prompt
                 (also triggered automatically on tab inactivity > 30 min)
```

### 8.5 The Cloudflare Worker proxy

```ts
// proxy/src/worker.ts
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    if (request.method !== 'POST') return new Response('Method not allowed', { status: 405 });
    if (new URL(request.url).pathname !== '/api/extract') return new Response('Not found', { status: 404 });

    // Origin check — only the deployed frontend is allowed
    const origin = request.headers.get('origin');
    if (!ALLOWED_ORIGINS.includes(origin ?? '')) return new Response('Forbidden', { status: 403 });

    // Naive rate limit (per IP, in-memory KV; replace with Durable Object if abuse appears)
    if (await isRateLimited(request, env)) return new Response('Too many requests', { status: 429 });

    const body = await request.json() as ExtractRequest;
    const flashResponse = await fetch(GEMINI_FLASH_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-goog-api-key': env.GEMINI_API_KEY,
      },
      body: JSON.stringify({
        contents: [{ parts: [{ text: body.prompt }] }],
        generationConfig: {
          response_mime_type: 'application/json',
          response_schema: body.schema,
          temperature: 0.2,  // low for extraction
        },
      }),
    });

    if (!flashResponse.ok) {
      const errText = await flashResponse.text();
      return Response.json({ error: 'flash_error', detail: errText }, { status: 502 });
    }

    const flashData = await flashResponse.json();
    return Response.json({
      data: flashData.candidates[0].content.parts[0].text,
      raw: flashData,
    });
  },
};
```

Single endpoint, stateless, no logging beyond rate-limit counters. The API key never leaves the Worker.

### 8.6 What's deliberately not in MVP architecture

These are stretch goals; the architecture supports adding them without rewrites.

- **No telemetry / analytics.** No tracking pixel, no event log to a server. The Phase Gate Log is local-only.
- **No cloud sync.** Stretch goal. The encrypted IndexedDB store can be exported as a single file (encrypted blob + plaintext index manifest) for manual backup or device transfer.
- **No multi-user.** Single user, single device. The encryption layer makes it inadvisable to have multiple users on the same browser profile.
- **No CI/CD specified.** Cloudflare Pages auto-deploys on push to main; that's the entire pipeline. Tests are run locally pre-commit.
- **No error reporting service** (Sentry etc.). MVP relies on console + manual debugging. Adding Sentry later is trivial.

---

## 9. Screen Specifications

This section enumerates every screen in the MVP application: name, purpose, UI elements, interactions, connected screens. The screen breakdown is the most important context for AI-assisted UI implementation; following it prevents UX drift.

### 9.1 Application Lock Screen

**Purpose:** First screen on every app open. Prompts for the local password to unlock the encrypted store.

**UI elements:**

- App logo/wordmark
- Password input field (masked)
- "Unlock" button
- "First time? Set up Startup" link (only shown when no encrypted store exists yet)
- Subtle note: "Your data lives only on this device. There is no password recovery."

**Interactions:**

- Submit password → derive key, validate against verification token → on success route to Project Dashboard
- On failure → show error, retry; lock for 30 seconds after 5 consecutive failures (anti-brute-force)
- "Set up" link → onboarding flow that creates the password + verification token

**Connected screens:** → Project Dashboard (on success), → First-Run Onboarding (on link click)

### 9.2 First-Run Onboarding

**Purpose:** Set the local password, optionally seed the Vault, optionally configure Gemini Flash proxy URL if self-hosted.

**UI elements:**

- Step 1: Set password (with strength meter, confirm field, irrecoverable warning)
- Step 2: Acknowledge data locality and irrecoverability (typed phrase confirmation: "I understand this data lives only on this device.")
- Step 3 (optional): Configure proxy URL (defaults to deployed Cloudflare Worker)
- Step 4: Take me to my dashboard

**Connected screens:** → Project Dashboard

### 9.3 Project Dashboard

**Purpose:** The home screen. Shows all projects (active + archived), provides entry to Vault and Settings, and is the launch point for new projects.

**UI elements:**

- Header: app wordmark, lock-now button, settings link, Vault link
- "New Project" primary CTA
- Active Projects section: list of cards. Each card shows project name, hackathon name, current phase (with phase badge), time remaining (if deadline set), stale-section count if any, and a click-to-open hit area
- Archived Projects section: collapsed by default, expandable list. Smaller cards, read-only badge
- Empty state when no projects: directive message ("Create your first project to begin.")

**Interactions:**

- "New Project" → New Project Setup screen
- Click project card → Project Workspace at the project's current phase
- Right-click / kebab menu on card → Archive, Rename, Delete (with confirmation), Export PCD

**Connected screens:** → New Project Setup, → Project Workspace, → Vault, → Settings, → Application Lock Screen (on lock)

### 9.4 New Project Setup

**Purpose:** The Phase 1 intake form. Collects all known hackathon details and creates the PCD with `current_phase: 1`.

**UI elements:**

- Hackathon name (required)
- Organizer: name (required), type (dropdown), notes (optional)
- Sponsors: dynamic list of {name, industry, tier}
- Theme / track (optional)
- Problem statement (large textarea, optional but strongly recommended)
- Judging criteria (large textarea, optional — triggers Rubric Mapping Intake if filled)
- Timeline: start, submission deadline, final pitch (datetime pickers)
- Format: dropdown (in-person / hybrid / virtual / unknown)
- Team constraints: min/max size, current team members
- "Create Project" CTA

**Interactions:**

- On create → run Vault auto-queries (judges/organizers by name, winning solutions by domain), surface results inline as opt-in suggestions before proceeding
- After Vault opt-in → write PCD with default rubric loaded; navigate to Project Workspace at Phase 1
- If judging criteria provided → on submit, immediately trigger the Rubric Mapping Intake flow before completing Phase 1 setup

**Connected screens:** → Project Workspace (Phase 1)

### 9.5 Project Workspace (the dominant screen)

**Purpose:** The unified phase workspace. Layout adapts per phase but the structure is constant.

**Layout:**

```
┌────────────────────────────────────────────────────────────────────┐
│ Header bar: project name • current phase badge • time-to-deadline  │
├──────────────┬─────────────────────────────────────────────────────┤
│              │                                                     │
│   Phase      │   Phase Workspace Body                              │
│   Navigator  │                                                     │
│   (sidebar)  │   ┌─────────────────────────────────────────────┐   │
│              │   │ Phase Briefing (collapsible)                │   │
│   • Phase 0  │   └─────────────────────────────────────────────┘   │
│   • Phase 1✓ │   ┌─────────────────────────────────────────────┐   │
│   • Phase 2  │   │ Completion Checklist                        │   │
│   • Phase 3  │   └─────────────────────────────────────────────┘   │
│   ▶ Phase 4🔒│   ┌─────────────────────────────────────────────┐   │
│   • Phase 5  │   │ Prompt Generator (stack of generatable     │   │
│   • Phase 6  │   │   prompts for this phase)                   │   │
│   • Phase 7  │   └─────────────────────────────────────────────┘   │
│              │   ┌─────────────────────────────────────────────┐   │
│   [stale: 2] │   │ Research Intake (per intake, paste-back     │   │
│              │   │   form + last-extracted preview)            │   │
│              │   └─────────────────────────────────────────────┘   │
│              │   ┌─────────────────────────────────────────────┐   │
│              │   │ Phase Output Preview (read-only render of   │   │
│              │   │   sections this phase produced)             │   │
│              │   └─────────────────────────────────────────────┘   │
│              │                                                     │
│              │   [Advance to next phase] (gated)                   │
└──────────────┴─────────────────────────────────────────────────────┘
```

**UI elements (sidebar):**

- Phase nodes 1-7 (Phase 0 is a separate "Vault" link, not a node)
- Current phase highlighted
- Completed phases marked ✓
- Phase 4 has a lock icon; locks visually when crossed
- Stale-section count badge at the bottom, click to jump to PCD Viewer with stale filter
- Backward-navigation click: confirmation modal warning about reconciliation, then jumps to that phase

**UI elements (body):**

- Phase Briefing: directive plain-language explanation of the phase, what it produces, why it matters. 2-4 paragraphs. Collapsible, default open on first phase visit
- Completion Checklist: required items per phase with checkbox state; greyed if blocked by other items
- Prompt Generator: each prompt for the phase as a card with title, recommended LLM, "Generate" button. Clicking expands the card to show: the assembled prompt in a copyable code block, a "Show injected context" toggle, a "Copy" button, recommended LLM badge, link out to that LLM's web UI
- Research Intake: each intake for the phase as a card with title, paste-back textarea, "Process" button. After processing, shows the three-tier outcome UI (clean / partial / failed)
- Phase Output Preview: read-only PCD render for sections populated by this phase, with "edit" affordance per section
- "Advance to next phase" CTA at the bottom: enabled only when completion requirements pass; shows a tooltip explaining what's missing when disabled

**Phase-specific variations:**

- **Phase 4:** the body grows a "Scope Lock Confirmation" panel with a typed-phrase confirmation (the user types "lock scope" exactly to confirm). The Advance CTA is replaced by "Cross the Phase 4 Gate" with a permanent-action warning
- **Phase 5:** body is replaced by the Build Checklist (derived from TID features and screens) + a reference panel with quick links to TID, Demo Script, Product Brief. Time-to-deadline gets prominent placement. "Initiate Addendum Protocol" link visible
- **Phase 6:** runs side-by-side with Phase 5; the sidebar shows Phase 5 and Phase 6 both as "active". Body is the standard Phase Workspace structure for Phase 6 prompts and intakes
- **Phase 7:** body has the Debrief intake plus a "Vault Update Proposals" panel after the debrief is processed, surfacing the proposed Vault additions for one-click application

**AI-powered elements:**

- Prompt Generator: assembles prompts from PCD via context selection (deterministic, not Flash)
- Research Intake: calls Flash via proxy for structured extraction
- Stale detection: deterministic graph-walk; surfaced visually

**Connected screens:** → PCD Viewer, → Vault, → Document Export Center (Phase 4+), → Addendum Protocol Interface (post-gate), → Project Dashboard

### 9.6 PCD Viewer

**Purpose:** Full read-only-by-default render of the entire PCD. Section-by-section. Filterable by phase, populated state, and stale state.

**UI elements:**

- Filter bar: phase (multi-select), populated/empty toggle, stale-only toggle
- TOC with anchor links to each section
- Each section rendered as: section header, populated/stale badges, last-updated timestamp, populated-by-phase badge, content (markdown-rendered), inline "Edit" button, "Resolve staleness" controls if stale
- Vault references panel: list of Vault entry cards that informed any section (resolved by ID at render time)
- "Export PCD" button: downloads the full markdown file (with embedded JSON code block at end)

**Interactions:**

- Click "Edit" on any section → opens the section in a structured form editor (per-field, validated against schema)
- Click "Resolve staleness" → modal with three options (Regenerate, Confirm-as-valid, Manual edit)
- Click any Vault reference → navigates to that Vault entry

**Connected screens:** → Project Workspace (back), → Vault entry detail, → Document Export Center

### 9.7 Document Export Center

**Purpose:** Download the Phase 4 outputs as standalone files for use in Cursor, plus the full PCD export.

**UI elements:**

- Cards for each downloadable artifact:
  - Product Brief (.md)
  - Technical Implementation Document (.md)
  - Demo Script & Backup Plan (.md)
  - Full PCD (.md, includes embedded JSON)
- Each card shows: last updated, download button, "Copy to clipboard" button, preview link
- Note about Cursor handoff: "Drop these files into your Cursor project as context. The TID is the primary build reference."

**Connected screens:** → PCD Viewer, → Project Workspace

### 9.8 Vault — Top Level

**Purpose:** Browse and search the persistent cross-project infrastructure.

**UI elements:**

- Tabs: Judges & Organizers | Winning Solutions | Boilerplate Library
- Each tab is a list view with: search box, tag filter chips, sort controls, "Add new" CTA
- Future tab placeholder: "Client Vault" (disabled, with "Stretch goal" tag)

**Interactions:**

- Search → keyword + tag filter executed against IndexedDB indexes
- Click entry → entry detail screen
- Add new → entry creation form

**Connected screens:** → Vault entry detail, → Vault entry creation, → Project Dashboard

### 9.9 Vault Entry Detail / Edit

**Purpose:** View and edit a single Vault entry. One screen handles all three entry types via different field configurations.

**UI elements (varies by type):**

- All types: id (read-only), name, tags, created/updated timestamps, free-form notes
- Judge/Organizer: type, organization, professional background, events appeared at, observed tendencies
- Winning Solution: event, organizer, domain, year, summary, why it won, source URL
- Boilerplate: component name, category, description, used-in events, what worked, what didn't, repo pointer
- "References" panel: which projects reference this entry (read-only list)
- Save / Delete CTAs

**Connected screens:** → Vault top level, → Project Workspace (when accessed via reference)

### 9.10 Settings

**Purpose:** All app-level configuration. JSON-editable for MVP.

**UI elements:**

- Sections (each as an expandable panel):
  - Default Judging Rubric: JSON textarea with validate-on-save, reset-to-bundled-default link
  - LLM Preferences: JSON object mapping prompt-type-IDs to recommended-LLM strings; affects which LLM is suggested in Prompt Generator
  - Proxy URL: defaults to deployed Worker; editable for self-hosters
  - Deadline notification intervals: array of percentages (default `[50, 75, 90]`)
  - Export format preferences: include-machine-readable-block toggle, etc.
- Below settings: "Change password" flow (re-encrypts the entire IndexedDB store with a new key)
- Below that: "Danger zone" — Clear all data (typed-phrase confirmation, irreversible)

**Connected screens:** → Project Dashboard

### 9.11 Addendum Protocol Interface

**Purpose:** Controlled, friction-heavy entry point for post-gate scope changes.

**UI elements:**

- Top: large warning banner ("Scope is locked. Changes are permanently logged.")
- Form:
  - Cursor build-state markdown paste (large textarea)
  - Proposed change description (textarea, min 200 chars)
  - Written justification (textarea, min 80 chars)
- "Generate Addendum Prompt" CTA → expands to show the prompt and a paste-back area
- After paste-back and processing: extracted addendum preview, "Append to Project" CTA
- Below the form: list of existing addenda for this project, each with timestamp + justification

**Connected screens:** → Project Workspace, → PCD Viewer (TID section, where addenda are shown)

### 9.12 Phase Gate Log Viewer

**Purpose:** Read-only audit trail of every state transition.

**UI elements:**

- Filter bar: event type, phase, date range
- Chronological list of log entries with: timestamp, phase badge, event type badge, details, user_justification (where present)
- Export to CSV button (for the Phase 7 Debrief)

**Connected screens:** → Project Workspace, → PCD Viewer

### 9.13 Section Editor (modal/panel)

**Purpose:** Structured per-field editing of any PCD section. Used from the PCD Viewer's "Edit" buttons.

**UI elements:**

- Header: section name, phase badge, populated/stale state
- Form fields generated from the section's JSON Schema fragment (uses a generic JSON-Schema-to-form renderer; same pattern as the partial-extract review form in §6)
- "Save" CTA (validates against schema before write; logs to user_overrides on each field touched)
- "Cancel" CTA

**Connected screens:** → PCD Viewer, → Project Workspace

---

## 10. UX Principles

These are stated as build-time guidance for every screen, microcopy decision, and interaction flow.

### 10.1 Prescriptive voice everywhere

The user has chosen to use Startup precisely because they want a system that tells them what to do. Every piece of copy in the app should be directive, not optional.

- Phase Briefings begin with imperative verbs: "Run the competitive pre-mortem." not "You may wish to run the competitive pre-mortem."
- Empty states tell the user the next action: "Create your first project to begin." not "Welcome to Startup!"
- Error messages tell the user what to do: "Paste the SCAMPER output into the Research Intake below." not "Field is required."
- Disabled CTAs explain what's missing in a tooltip: "Complete the Competitive Pre-mortem to advance." not "Disabled."

### 10.2 Transparency by default, even hidden behind a toggle

The user must always be able to see why the system did what it did.

- The Prompt Generator has a "Show injected context" toggle on every prompt
- Every intake produces a diff preview the user must confirm before commit
- The Phase Gate Log is accessible from every workspace
- Stale flags name the upstream change that caused them

### 10.3 The Phase 4 gate is sacred

UI copy and visual treatment must reinforce the gate.

- Phase 4 sidebar node has a lock icon that visibly closes when crossed
- The Phase 4 advance CTA has different language ("Cross the Phase 4 Gate") and requires a typed-phrase confirmation
- Post-gate scope changes are reachable only via the Addendum Protocol entry, never via "edit TID"
- Addenda are visually distinct from the original TID — they are appended, not blended

### 10.4 Friction where friction belongs

Friction is added deliberately at high-stakes actions, removed everywhere else.

High-friction (typed phrase, confirmation modal, justification required):

- Crossing the Phase 4 gate
- Initiating the Addendum Protocol
- Overriding a phase completion gate
- Clearing all data

Low-friction (one click):

- Generating a prompt
- Copying a prompt
- Confirming an intake diff
- Marking a stale section as still valid
- Advancing a phase when all gates pass

### 10.5 The PCD is the spine — surface it everywhere

The user should never feel disconnected from the PCD state.

- Every workspace has a "Show PCD" affordance
- Every Vault reference is clickable to its entry
- Every section in any rendered context shows its provenance (last updated, populated by phase)
- Stale sections are visually distinct in every view they appear in

### 10.6 Speed under pressure

Hackathons run hot. Common actions must be fast.

- Keyboard shortcuts: ⌘P / Ctrl-P opens project switcher; ⌘E exports PCD; ⌘L locks the app
- Prompt copy is one click and shows a confirmation flash
- The next required action is always visible in the workspace without scrolling

---

## 11. Build Order & MVP Cut Line

### 11.1 MVP definition

MVP is the smallest version of Startup that can run a real hackathon end-to-end. Specifically:

- A user can create a project from a hackathon brief
- The system loads the Default Judging Rubric automatically
- All Phase 1-7 prompts can be generated, the user runs them externally, paste-back works, the PCD fills up
- The Phase 4 gate works correctly and the three Phase 4 outputs export as markdown for Cursor
- The Vault stores judges/organizers/winning-solutions/boilerplate descriptions and supports keyword/tag search
- Backward navigation triggers staleness propagation correctly
- Addendum Protocol works post-gate
- Local password encryption protects everything

If those things work end-to-end, the MVP ships.

### 11.2 Recommended build order

This order is optimized for being able to dogfood Startup partway through its own build — running Startup's own build with Startup's own workflow.

**Milestone 1 — Foundation (week 1)**

1. Vite + React + TS + Tailwind scaffold; Cloudflare Pages deploy pipeline
2. Dexie schema; Web Crypto encryption layer; lock screen; first-run onboarding
3. PCD schema imported; Ajv validator; createEmptyPCD function
4. Project Dashboard skeleton; New Project Setup form (Phase 1 inputs only, no Vault auto-query yet)
5. Default Rubric parser; loads at project create

**Milestone 2 — The Prompt-Generate-Paste-Back loop (week 2)**
6. Cloudflare Worker proxy with single /api/extract endpoint
7. PromptDefinition registry stub for one prompt (Phase 2 Problem Analysis)
8. schemaToOutputTemplate function
9. Generate function with deterministic context selection
10. Project Workspace shell with Phase Navigator sidebar
11. Prompt Generator card UI; copy-to-clipboard
12. IntakeDefinition registry stub for one intake (Phase 2 Problem Analysis)
13. Three-tier outcome handler; diff preview; commit-to-PCD

At this point, Startup can do its core trick — generate a prompt, accept paste-back, write to PCD — for one prompt-intake pair. Everything from here is rounding out the scope.

**Milestone 3 — All phases (week 3)**
14. Fill out PromptDefinition + IntakeDefinition registries for all phases
15. Phase Briefings + Completion Checklists for all phases
16. State machine: ALLOWED_TRANSITIONS, completion gates, advance CTA logic
17. PCD Viewer with section-level render and edit
18. Section Editor (generic JSON-Schema-to-form)

**Milestone 4 — Reconciliation and Phase 4 gate (week 4)**
19. Dependency graph + propagateStaleness
20. Stale resolution UI (regenerate / confirm-valid / manual edit)
21. Phase Gate Log + viewer
22. Phase 4 gate logic + scope-lock confirmation UI
23. Document Export Center (Phase 4 outputs + full PCD export)
24. Addendum Protocol interface

**Milestone 5 — Vault (week 5)**
25. Vault top-level UI
26. Vault entry detail/edit screens for all three types
27. Vault auto-query on Phase 1 + Phase 2 + Phase 4
28. Vault references rendered in PCD Viewer

**Milestone 6 — Polish (week 6)**
29. Settings screen with JSON editors (including Grill-Me default mode preference)
30. Keyboard shortcuts
31. Empty states and prescriptive copy pass
32. Backward-navigation reconciliation flow polish
33. Phase 5 Build Checklist
34. Phase 6 Pitch Brief / Objection Map / Q&A Prep
35. Phase 7 Debrief + Vault update proposals
36. Grill-Me toggle UI on eligible Prompt Generator cards (Off / Interactive / Auto where optional; Interactive / Auto only where required per §4.7)
37. External Context Notes editor (Markdown textarea bound to `external_context_notes`; surfaced in PCD Viewer and as a side-panel on Grill-Me-eligible Prompt Generator cards)
38. Handoff Package export button on Grill-Me prompt cards (§4.5)
39. Phase 3 candidate-review UI: browse `candidate_portfolio.candidates[]`, compare scores by archetype, remix flow that writes back as a new `CandidateIdea` with `is_remix=true` and `remix_source_ids` populated

This is roughly six weeks of part-time work for a student-level builder using Cursor. Real timelines will vary; build in milestone-sized chunks and dogfood from Milestone 2 forward.

### 11.3 What to cut if running short

If hackathon season hits and Startup itself isn't done, cut in this order:

1. Phase 7 Debrief (can be done manually in any LLM)
2. Phase 6 outputs (use any LLM directly)
3. Vault — boilerplate library (least-used of the three Vault stores)
4. Settings UI for non-rubric items
5. Phase Gate Log Viewer (data is still recorded, just not browsable)

Do **not** cut: PCD schema integrity, the Phase 4 gate, the Competitive Pre-mortem hard block, the encryption layer, the diff preview before commit. These are the load-bearing pieces.

---

*End of specification. Questions on implementation should be resolved against this document, the PCD schema, and the Default Judging Rubric. When in doubt, the schema is canonical.*
