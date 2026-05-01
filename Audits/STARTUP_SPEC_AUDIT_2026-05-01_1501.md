# Startup Implementation Spec — Full Audit

> Audit of `STARTUP_IMPLEMENTATION_SPEC_2026-04-30_0015_md.md` (2259 lines), cross-referenced against `pcd_schema.json`, `PROMPTS.md`, `STARTUP_WORKFLOW_AND_APP_OVERVIEW.md`, and `default_judging_rubric.md`.
> Date: 2026-05-01.
> Goal: surface every drift, gap, and ambiguity that would force Cursor to make undocumented decisions during build.

---

## Summary

- 4 🔴 RED — block build or contradict authoritative sources
- 15 🟡 YELLOW — implementable, but force Cursor to invent decisions that should be in the spec
- 6 🟢 GREEN — cosmetic, fix-when-touching

The spec is solid in its core skeleton (PCD model, factory, prompt-generation flow, intake parsing, state machine, encryption, screens). The drift is concentrated in three seams: (a) the Phase Gate Log vocabulary, (b) the Addendum Protocol, and (c) supporting types/constants that were referenced but never defined. None of the RED items require workflow re-design — they're plumbing.

**Recommended fix order:** RED #1 → RED #2 + #3 (one decision covers both) → RED #4 → YELLOW cluster around state-machine constants (#5, #6) → YELLOW screen-spec gaps (#7, #8, #9) → remaining YELLOWs → GREENs. Targeted edit, not a re-write.

---

## 🔴 RED — Build-breaking or schema-contradicting

### RED-1 — Stale `data.body_markdown` reference at §4.3

**Location:** Line 744.

**What's wrong.** §4.3 defines the `'if_user_provided'` inclusion rule:

> The section is included only when `is_populated === true` AND `data.body_markdown` is non-empty.

But the schema and §2.6 (fixed last session) make `external_context_notes.data` a bare string, not an object with a `body_markdown` field. `data.body_markdown` does not exist.

**Why it matters.** Code implementing this inclusion rule will reference an undefined field and either (a) always treat the section as empty, or (b) crash at runtime depending on TS strictness.

**Fix.** Replace `data.body_markdown` with `data` (or `data.length > 0` if you want to express the non-empty check explicitly).

> One-line edit. This is a leftover from the §2.6 cleanup last session — the prior session noted the §2.7 initial-state table works as a drift-detection seam against §2.6, but didn't catch this one because §4.3 wasn't touched then. The §2.6 → §4.3 axis needs a one-time sweep.

---

### RED-2 — Phase Gate Log event_type enum drift

**Location:** Lines 781, 981, 1407.

**What's wrong.** The schema (`PhaseGateLogEntry.event_type`, schema lines 175-187) allows exactly these values:

```
phase_completed, phase_skipped_with_override, competitive_premortem_override,
backward_navigation, staleness_resolved_regenerate, staleness_resolved_confirm_valid,
staleness_resolved_manual_edit, phase_4_gate_crossed, addendum_created, vault_query_logged
```

The spec writes four event types that are not in this enum:

| Event type | Spec line | Context |
|---|---|---|
| `prompt_generated` | 781 | Logged on every prompt generation, with `runtime_inputs_used` payload |
| `grill_me_session_initiated` | 981 | Logged when a Grill-Me prompt is generated |
| `grill_me_session_resolved` | 981 | Logged when the Resolved Summary is intaked |
| `grill_me_adjustment_dismissed` | 1407 | Logged when a Design Grill adjustment row is dismissed |

**Why it matters.** Any attempt to write these will fail Ajv validation, which `createEmptyPCD` runs (§2.7.4 line 364) and which any subsequent `validatePCD` call will run on every save. The factory's defensive validation will keep this from corrupting persisted state, but it will hard-fail the user's session at the point of logging.

The prior session was explicit about this discipline (§2.7.2 line 278: "the log's `event_type` enum does not include such an event, and... when a real reason emerges to need a `project_created` event, the enum can be extended then"). That discipline was not applied to these four.

**Fix — pick one:**

- **Option A (extend the schema, recommended):** Add the four event types to the enum. Rationale: the audit trail value of these events is real — the Phase 7 Debrief needs prompt-generation history, and Grill-Me session lifecycle is genuine state. The "don't add vocabulary speculatively" discipline applies to genuinely speculative cases; here the spec already specifies write-sites, so the enum is downstream of an existing decision.
- **Option B (drop the writes):** Strike the log-write language at lines 781, 981, 1407. Phase Gate Log doesn't track these events.

If A: pair with RED-3 below (you'll need a payload field).

---

### RED-3 — Phase Gate Log has no slot for structured payloads

**Location:** Lines 781 (`runtime_inputs_used: {...}`), 981 (`{ prompt_id, mode, timestamp }`, `{ prompt_id, intake_id, timestamp }`), 1407 ("the full record").

**What's wrong.** The spec describes structured payloads accompanying log entries. The schema's `PhaseGateLogEntry` has only:

```json
{ "timestamp", "phase", "event_type", "details": "string", "user_justification": "string|null" }
```

`details` is a free-form string. There is no provision for structured payload data. The `addendum_created` event (which is in the enum) also has no clear payload home — the spec just says "Append `addendum_created` event" without specifying what details accompany it.

**Why it matters.** The intent of the spec is clearly to log structured data (the Debrief reads these to correlate decisions). Forcing the implementer to JSON-stringify into `details` works but is awkward, and the spec doesn't say to do that — Cursor will probably try to add fields and discover schema validation rejects them.

**Fix.** Add a typed payload field to `PhaseGateLogEntry`:

```json
{
  "type": ["object", "null"],
  "description": "Event-specific structured data. Shape varies by event_type. Null when not applicable.",
  "additionalProperties": true
}
```

Or, keep `details` as the only payload site and explicitly state in the spec that structured payloads JSON-stringify into `details`. Less clean, but no schema change.

Strongly recommend the schema change. Phase Gate Log is the single most important audit artifact; making it a dumping ground of stringified JSON for structured events undercuts its value.

> **Note on RED-2 + RED-3 together:** These are really one decision. If you extend the enum, you also need somewhere to put the payloads. The cleanest move is one schema diff: add the four event types AND add a structured `payload` field. Single schema change, both REDs resolved.

---

### RED-4 — Addendum Protocol references undefined intake and prompt

**Location:** §5.5 (line 1182), §9.11 (lines 2055-2070).

**What's wrong.** §5.5 line 1182 says:

> System generates Addendum Prompt for external LLM
> User runs externally, pastes back
> System extracts via the addendum intake schema (§6)

But §6 (Intake Parsing) has no addendum intake. The v1.5 intake registry additions list four intakes (`idea_seed_challenge_intake`, `divergent_candidates_intake`, `design_grill_intake`, `judge_qa_drill_intake`); none is the addendum. §4.2 (PromptDefinition registry) has no addendum prompt either.

§9.11 (Addendum Protocol Interface) describes a paste-back flow with "extracted addendum preview" — an LLM round-trip. So an intake/prompt pair is necessary.

The schema has `TIDAddendum` with required fields `addendum_id`, `created_at`, `user_justification`, `content_markdown`, `build_state_snapshot`. The spec doesn't specify how the form-input maps to this shape.

**Why it matters.** The Addendum Protocol is in MVP scope (per §11.1 line 2180: "Addendum Protocol works post-gate"). Cursor will not be able to build it from the spec as written.

**Fix.** Add to the registries:

```ts
// PromptDefinition for addendum
{
  id: 'addendum_protocol_addendum_prompt',
  phase: 5,  // post-gate, before phase 7
  display_name: 'Scope Change Addendum',
  recommended_external_llm: 'claude',
  role_assignment: '...',
  context_selectors: [/* chosen_direction, technical_implementation_document, scope_definition */],
  vault_query_hints: null,
  task_specification_template: '...',
  runtime_inputs: [
    { name: 'build_state_snapshot', kind: 'textarea', required: true, ... },
    { name: 'proposed_change_description', kind: 'textarea', required: true, ... },
    { name: 'user_justification', kind: 'textarea', required: true, ... },
  ],
  target_intake_schema_path: 'addenda',
  produces_intake_id: 'addendum_intake',
  // grill_me_modes omitted — addendum is not Grill-Me-eligible
}

// IntakeDefinition for addendum
{
  id: 'addendum_intake',
  display_name: 'Scope Change Addendum',
  target_pcd_path: 'addenda',
  target_schema_fragment: tidAddendumSchema,  // from $defs/TIDAddendum
  extraction_prompt_template: '...',
  required_fields: ['content_markdown'],   // others come from runtime inputs
  optional_fields: [],
  confidence_threshold: 0.7,
  commit_strategy: 'apply_adjustments',  // or 'direct' with append semantics
}
```

This will need a new commit_strategy variant if `'apply_adjustments'` doesn't fit (it currently writes against `chosen_direction.data` field-paths, not appends to a top-level array). The cleanest path is probably a third variant: `'append'`, with `target_pcd_path` resolving to an array and the extracted object appended. Adds one branch to the commit handler — clean, declarative, future-proof for any "log-shaped" intakes.

Also: addendum form supplies `addendum_id` (mint a ULID app-side, like `divergent_candidates_intake` post-extract enrichment), `created_at` (now()), `user_justification` and `build_state_snapshot` (from runtime inputs). Only `content_markdown` is genuinely Flash-extracted from the LLM response.

---

## 🟡 YELLOW — Implementable but ambiguous, force Cursor to make undocumented decisions

### YELLOW-5 — Staleness propagation depends on undefined constants

**Location:** §5.3 lines 1107, 1113, 1118, plus the `transitiveClosure` function call at 1107.

**What's wrong.** `propagateStaleness` references four constants/functions that are never defined:

- `INVERTED_DEPENDENCY_GRAPH` — derived from `PCD_DEPENDENCY_GRAPH`, but the inversion logic is unspecified
- `PHASE_4_OUTPUT_SECTIONS` — never enumerated
- `PHASE_6_OUTPUT_SECTIONS` — never enumerated
- `transitiveClosure(node, graph)` — function shape never specified

All inferable, but each has multiple plausible implementations.

**Fix.** Add a small subsection §5.3.1:

```ts
const PHASE_4_OUTPUT_SECTIONS = [
  'scope_definition', 'technical_direction', 'product_brief',
  'technical_implementation_document', 'demo_script_and_backup_plan',
];

const PHASE_6_OUTPUT_SECTIONS = [
  'pitch_brief', 'judge_objection_map', 'qa_prep_document',
];

// INVERTED_DEPENDENCY_GRAPH: { upstreamSection: [downstream sections that depend on it] }
// Derived once at module load from PCD_DEPENDENCY_GRAPH.
const INVERTED_DEPENDENCY_GRAPH: Record<string, string[]> = invertGraph(PCD_DEPENDENCY_GRAPH);

function transitiveClosure(start: string, graph: Record<string, string[]>): Set<string> { ... }
```

Two functions and two constants. Five minutes to add; saves Cursor inventing names that won't match the schema.

---

### YELLOW-6 — `vault_query_logged` event in schema with no spec write-site

**Location:** Schema enum line 186; no corresponding spec write.

**What's wrong.** The schema reserves `vault_query_logged` as an allowed event_type. §3.3-3.4 describe Vault auto-queries but never log them. The spec doesn't say when this event fires.

**Fix.** Either describe the write-site (e.g., "every Vault auto-query that surfaces ≥1 result writes a `vault_query_logged` event with details: `{ store, query, result_count }`") or strike from the enum.

Recommend keeping it and describing the write-site — the audit trail of "what Vault entries were considered for this project" is genuinely useful for the Phase 7 Debrief.

---

### YELLOW-7 — Phase 5 Build Checklist screen not specified in §9

**Location:** §9.5 line 1957 (referenced as "Phase-specific variation"); §11.2 line 2235 (build order item 33).

**What's wrong.** §9.5 says Phase 5 "body is replaced by the Build Checklist (derived from TID features and screens)" but no §9 sub-section specifies the screen. Build order has it as a milestone-6 polish item.

**Fix.** Add §9.x "Phase 5 Build Checklist" with the same level of detail as the other screens: derivation rule (which TID fields produce which checklist items), checkbox-state persistence (presumably a non-PCD store?), connection to Addendum Protocol entry point. Even half a page is enough.

---

### YELLOW-8 — External Context Notes editor placement underspecified

**Location:** §11.2 build order item 37; absent from §9.5 and §9.6.

**What's wrong.** Build order says "External Context Notes editor (Markdown textarea bound to `external_context_notes`; surfaced in PCD Viewer and as a side-panel on Grill-Me-eligible Prompt Generator cards)". But §9.5 (Project Workspace) and §9.6 (PCD Viewer) don't enumerate it as a UI element. Cursor would have to choose placement.

**Fix.** Add one bullet to §9.6 ("External Context Notes panel: Markdown editor bound to `external_context_notes.data`; surfaced as a section header on the PCD Viewer with edit-in-place affordance") and one bullet to §9.5's Prompt Generator card description ("on Grill-Me-eligible cards, expand to surface the External Context Notes side-panel for the duration of the dialogue").

---

### YELLOW-9 — Phase 3 candidate-review/remix UI not in §9

**Location:** §11.2 build order item 39 references it; no §9 entry.

**What's wrong.** Build order item 39: "Phase 3 candidate-review UI: browse `candidate_portfolio.candidates[]`, compare scores by archetype, remix flow that writes back as a new `CandidateIdea` with `is_remix=true`...". This is a non-trivial UI surface (filterable list, comparison view, remix authoring) and has no §9 spec.

**Fix.** Add §9.x "Phase 3 Candidate Review" — similar in size to §9.13 Section Editor. Should specify: how the list renders archetype groupings, what fields are visible per card, how the remix flow collects `remix_source_ids` (multi-select), how the remix is named/scored (LLM-assisted? manual? hybrid?), and what writes back to PCD.

---

### YELLOW-10 — `_confidence` vs `confidence` and the implicit transformation chain

**Location:** §6.3 (lines 1431-1467), §6.4 (line 1502), §8.5 (line 1797).

**What's wrong.** Three slightly inconsistent shapes:

- Flash returns: `{ "data": {...}, "_confidence": {...} }` (per §6.3 example)
- Worker returns: `{ data: <raw text>, raw: <full Flash response> }` (per §8.5 line 1796)
- `processIntake` consumes: `result.data`, `result.confidence` (not `_confidence`), `result.raw`, `result.error` (per §6.4)

The transformation steps from Flash → proxy → FlashResult are not made explicit. Where does parsing happen? Where does `_confidence` get renamed to `confidence`? Where does `error` come from on the FlashResult side (the worker only returns it on `flashResponse.ok === false`)?

**Fix.** Add §6.6 substep showing the FlashResult shape and where transformations happen:

```ts
type FlashResult = {
  data: unknown;          // the parsed inner `data` from Flash's JSON output
  confidence: Record<string, number>;  // parsed inner `_confidence`, key renamed
  raw: unknown;           // full Flash response for debugging
  error: string | null;
};

// In frontend/src/lib/intake/extract.ts, after fetching from /api/extract:
const proxyResp = await fetch('/api/extract', {...}).then(r => r.json());
const parsed = JSON.parse(proxyResp.data);  // Flash returned JSON-stringified text
return {
  data: parsed.data,
  confidence: parsed._confidence,
  raw: proxyResp.raw,
  error: proxyResp.error ?? null,
};
```

10-line addition; closes the loop.

---

### YELLOW-11 — TS types referenced without definition

**Location:** Multiple.

**What's wrong.** The following types are referenced but never have a TS shape declared in the spec:

- `UserOverride` (§2.2 line 85) — inferable from schema's `user_overrides.items`
- `GlobalSettings` (§3.1 line 391) — never defined anywhere
- `VaultQueryHint` (§4.2 line 590) — never defined; this drives Vault auto-suggestions on prompt generation
- `FlashResult` (§6.4 line 1475) — see YELLOW-10
- `IntakeOutcome` (§6.4 line 1475) — discriminated union over `tier: 'clean' | 'partial' | 'failed'`, inferable from function returns

**Fix.** A short "Common Types" subsection somewhere (could fit in §8.2 alongside the directory structure, or as a §6.0 preamble). Specify each. `VaultQueryHint` is the most important — it shapes the auto-query behavior described in §3.4 but is currently a stub.

---

### YELLOW-12 — TypeScript codegen from JSON Schema not specified

**Location:** §8.2 line 1650 ("schema.ts # imports pcd_schema.json, exposes typed shape").

**What's wrong.** The spec says `schema.ts` imports the JSON Schema and exposes typed shape, but doesn't specify how. Options:

- `json-schema-to-typescript` (codegen, build step)
- `quicktype` (codegen)
- Hand-authored TS types kept in sync with the schema (manual)
- Runtime-only typing via Ajv (no static types beyond `unknown`)

These produce meaningfully different developer ergonomics. Cursor will pick one and you'll inherit that choice.

**Fix.** Recommend `json-schema-to-typescript` with a build script (`npm run gen:types`). It's the most common approach for this exact pattern, has good defaults, and the generated file goes alongside the source schema. Add one line to §8.2 and a one-line npm script in the package.json sketch (which the spec doesn't include, separately).

---

### YELLOW-13 — Worker rate limiting comment is technically inaccurate

**Location:** §8.5 line 1770.

**What's wrong.** Comment says "(per IP, in-memory KV; replace with Durable Object if abuse appears)". Cloudflare Workers don't have in-memory KV between requests — Workers KV is durable (eventually consistent), and per-request memory doesn't persist. The phrase "in-memory KV" is contradictory.

**Fix.** Pick one: Workers KV (cheap, eventually-consistent, fine for soft rate limits) or Durable Objects (strictly-consistent, more expensive, fine for hard limits). For an MVP personal-use app, Workers KV is the right answer. Replace the comment to: `// Naive rate limit (per IP, via Workers KV; ~300/day soft cap. Switch to Durable Objects only if abuse appears).`

---

### YELLOW-14 — Worker `ALLOWED_ORIGINS` not specified

**Location:** §8.5 line 1768.

**What's wrong.** `ALLOWED_ORIGINS` is referenced as a constant. Where does it live? Hardcoded? `wrangler.toml` env var? The spec doesn't say, but origin validation is the only gate keeping arbitrary callers off the proxy.

**Fix.** Specify: stored as an env var in `wrangler.toml` (set per environment — local dev, preview, production), accessed as `env.ALLOWED_ORIGINS` and split on commas. Single line of clarification.

---

### YELLOW-15 — Phase 2 completion gate path qualification

**Location:** §5.2 line 1050 ("At least one `differentiation_angles` entry exists").

**What's wrong.** The schema has both:
- `problem_analysis.data.differentiation_angles` (array of strings)
- `competitive_landscape.data.differentiation_seams` (array of strings — different concept)

The completion gate references `differentiation_angles` unqualified. Cursor would need to know which section it's in.

**Fix.** Replace "At least one `differentiation_angles` entry exists" with `problem_analysis.data.differentiation_angles.length >= 1`. Same precision as the `competitive_landscape.data.premortem_completed === true` flag in the same row.

---

### YELLOW-16 — §6.5 merge-vs-replace UX is too vague

**Location:** §6.5 line 1555.

**What's wrong.** "the user can opt to merge instead of replace where it makes sense — for arrays, this means concatenation; for objects, field-level union with old-wins-on-conflict by default." When does the affordance appear? Per-field toggle? Per-section? Default behavior? Implementation guidance is needed.

**Fix.** Specify: the diff preview shows a single merge/replace toggle at the section level, defaulting to replace when the section is being re-populated by an intake. For `judge_qa_drill_intake` the toggle is explicit (per §6.2 line 1413); for other intakes, replace is the only option in v1.5. Or — drop the merge feature entirely from MVP and add it later when a real use-case demands it.

The current text reads like "this is a feature" but is too vague to build. Either elevate to a real spec or move to "Stretch goals" in §8.6.

---

### YELLOW-17 — §6.4 "Tier 0" is in code comment only

**Location:** §6.4 line 1476.

**What's wrong.** The code has a comment "// Tier 0: Flash error or schema validation failure" but the prose describes only Tiers 1, 2, 3, and the runtime returns `tier: 'failed'` (not `'tier_0'`) for that case. So Tier 0 is comment-only and collapses to the same runtime tier as Tier 3.

**Fix.** Drop the "Tier 0" comment; it's a mental model that doesn't match runtime. Or add a Tier 0 prose section if the distinction is intended to surface in UI. Either way, the comment-vs-code-vs-prose mismatch is confusing.

---

### YELLOW-18 — Demo Script `backup_plan` completion criterion is vague

**Location:** §5.2 line 1052.

**What's wrong.** "demo script has `backup_plan`" — but `backup_plan` is always a present object in the schema (it's a property without `required` on the parent), and its sub-fields are all optional. What does "has" mean?

**Fix.** Specify: at least one of `backup_plan.screenshots_required`, `backup_plan.screen_recording_segments`, or `backup_plan.live_failure_handling_script` is non-empty.

---

### YELLOW-19 — PROMPTS.md not referenced anywhere in the spec

**Location:** Spec contains no reference to PROMPTS.md.

**What's wrong.** PROMPTS.md (2497 lines) is the source of truth for the actual prompt content (full role assignments, task specifications, intake-extraction guidance per intake). The spec references abbreviated content in §4.2 and an unnamed `templates/` folder in §8.2 line 1659. Cursor needs to know PROMPTS.md is the input.

**Fix.** Update the §1 header note to include PROMPTS.md as a companion doc, and add a note in §4.2 at the end of the v1.5 prompt registry block: "Full prompt content (role assignments, task specifications, negative constraints, output templates) lives in `PROMPTS.md`. The TypeScript template files in `frontend/src/lib/prompts/templates/` are direct ports of the PROMPTS.md content; PROMPTS.md is canonical for prompt vs spec drift on emission contracts (per the §1 invariants)." Add a parallel note in §6.2 about intake extraction guidance.

---

## 🟢 GREEN — Cosmetic, observation, low priority

### GREEN-20 — §2.1 example markdown has nested code-fence collision

**Location:** §2.1 lines 63-69.

**Detail.** The example showing "markdown export ends with an HTML-commented fenced code block containing the raw JSON" uses ```` ```markdown ```` as the outer fence and ```` ```json ```` inside it; the inner fence prematurely closes the outer. Renders weirdly when this spec is itself rendered as markdown. Implementer can intuit intent.

**Fix.** Use four-backtick outer fence: ````` ````markdown `````.

---

### GREEN-21 — `project_context.mdc` referenced but doesn't exist

**Location:** §1 header line 3; PROMPTS.md header line 3.

**Detail.** `.mdc` is Cursor's project-rules file format. Listed as a companion doc but not yet authored. Likely a Phase 0 stretch — the boilerplate the user maintains across projects.

**Fix.** Either author it (separate task — could be a thin pointer file: `# Startup project rules. See STARTUP_IMPLEMENTATION_SPEC.md, PROMPTS.md, and pcd_schema.json. When in doubt, schema is canonical.`) or strike from the header. Author it eventually — Cursor honors `.mdc` files automatically when present.

---

### GREEN-22 — Version naming overload

**Location:** Three coexisting numbers — spec "Version 1.0", PCD `schema_version` "1.0.0", "v1.5" features.

**Detail.** Three independent version axes. Mostly fine for humans who understand context; could trip up Cursor reading literally.

**Fix.** Optional: add a one-paragraph "Versioning" note in §1: "This spec is at Version 1.0. The PCD JSON schema has its own `schema_version: '1.0.0'`. References to 'v1.5' throughout describe the current iteration of the prompt and intake registries (HITL features) which is the only version this spec covers."

---

### GREEN-23 — Phase 3 completion gate over-specifies what schema enforces

**Location:** §5.2 line 1051 ("`candidate_portfolio.candidates` contains ≥ 3 entries").

**Detail.** Schema already enforces `minItems: 3, maxItems: 5` on `candidates`. Any populated `candidate_portfolio` automatically satisfies this. The completion gate as written is technically correct but redundant.

**Fix.** Tighten to `ideation_record.data.candidate_portfolio !== null` or leave as-is for human readability. Low priority.

---

### GREEN-24 — Phase Gate Log Viewer cuttability vs Debrief consumption

**Location:** §11.3 cut-list item 5 vs §5.6 / §11.2 item 21.

**Detail.** §11.3 says the Phase Gate Log Viewer is cuttable ("data is still recorded, just not browsable"). §9.12 mentions "Export to CSV button (for the Phase 7 Debrief)". If the Viewer is cut, where does the CSV export live?

**Fix.** One sentence in §11.3: "When cut, raw log data persists in IndexedDB; export-to-CSV is moved to the Settings → Danger Zone → Export Data section."

---

### GREEN-25 — Default Rubric: JSON-edit vs markdown-source semantics

**Location:** §7.1 (markdown source-of-truth) vs §9.10 ("Default Judging Rubric: JSON textarea").

**Detail.** The bundled default lives as markdown (`default_judging_rubric.md`), parsed at startup. The user customizes via JSON in Settings. The override semantics — where the override is stored, when it takes precedence, what "Reset to bundled default" does — are not fully spelled out.

**Fix.** One sentence in §7.3: "User-customized rubric JSON is stored in the `settings` Dexie table under key `default_rubric_override`. When present, this overrides the bundled markdown parse for newly-created projects. 'Reset to bundled default' clears this key. Existing projects retain whatever rubric was active at their creation time."

---

## Cross-cutting observation (not flagged)

The spec is well-organized into **registry-shaped declarative artifacts** (`PromptDefinition`, `IntakeDefinition`, `ALLOWED_TRANSITIONS`, `PCD_DEPENDENCY_GRAPH`, `PhaseCompletionRequirement`). This is the right shape for an AI coding tool — Cursor adds entries rather than authoring procedural code. The discipline of "add new vocabulary only when a real case appears" (declarative > imperative when bounded, per §1 invariants) is being respected for `commit_strategy` and `post_extract_validators`.

The drift catalogued above is concentrated in the same seams every time: (a) declared values in `pcd_schema.json` enums must match write-sites in the spec, (b) declared types must have shapes specified somewhere, (c) declared screens must have §9 entries. All three seams are the same kind of consistency check, and they could be enforced by a future automated linter walking the spec for symbols and grepping the schema and §9 for definitions. Not a near-term task — flagging the pattern.

---

## Recommendation on what to do next

1. **Fix RED-1** (one-line edit; matches the §2.6 cleanup discipline).
2. **Make the schema decision implied by RED-2 + RED-3** (one schema diff: extend the event_type enum, add a structured `payload` field to `PhaseGateLogEntry`). This is the only one of the four REDs that genuinely opens a design question rather than just patching a hole.
3. **Fix RED-4** by authoring the addendum prompt+intake registry entries. This may also need a third `commit_strategy: 'append'` variant.
4. **Sweep YELLOWs** in clusters: state-machine constants (5, 6), screen specs (7, 8, 9), supporting types (10, 11, 12), worker hardening (13, 14), micro-cleanups (15-19).
5. **GREENs are fix-when-touching** — don't make a dedicated pass.

Estimated work: 4-8 hours of careful editing, no new design decisions beyond RED-3 and RED-4. After this pass, the spec is build-ready and Cursor can work from it without ambiguity.

The spec was solid coming in. This audit is finding plumbing leaks, not architectural problems.
