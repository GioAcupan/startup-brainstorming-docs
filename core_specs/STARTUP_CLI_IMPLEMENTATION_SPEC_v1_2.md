# Startup — CLI Implementation Specification

*Version 1.2 — 2026-05-17*

> Companion documents (lived inside `.startup/` at runtime, read at build time): `pcd_schema.json`, `PROMPTS.md`, `default_judging_rubric.md`.

> **v1.2 changelog.** Adds **mode-aware procedure routing** to support both standard-time and same-day hackathons. A new `mode` field on the hackathon profile (`"standard" | "fast"`) is set at project init and determines which procedure files the helper dispatches to. The `procedures/` directory is restructured into `standard/`, `fast/`, and `shared/` subtrees; the helper gains a `procedure-path` resolver subcommand, a `mode` subcommand, and a `--mode` flag on `init`; the Phase 3 → 4 advancement precondition becomes mode-aware. One prompt (`phase_3_divergent_candidates`) gains a `{fast_mode}` runtime placeholder allowing it to synthesize without upstream ideation prereqs; all other prompts are mode-agnostic and reused unchanged. The Phase 4 gate ceremony, intake pipeline, schema, Vault, and skill catalog are mode-agnostic — skills are mode-aware via the underlying procedures, not via separate names. A new milestone **M3.5 — Fast Mode** is added between M3 and M4 in the build order. A new **Appendix C — Mode Architecture Reference** consolidates the per-phase deltas and design rationale.

> **v1.1 changelog (preserved for context).** Refactored the directory model from "one `.startup/` per project repo" to "one workspace hosts many projects." Per-project state moved to `.startup/projects/<id>/`; workspace-shared assets (procedures, prompts, schema, helper, Vault) stayed at `.startup/` root. Added `list`, `switch`, and `archive` helper subcommands, and `/startup:list` and `/startup:switch` utility skills. The shared `~/.startup-vault/` directory was dropped — the Vault now lives at workspace level and is shared across all projects in that workspace.

This document is the technical implementation specification for the CLI version of the Startup application. It is intended to be consumed by an AI coding agent (Cursor, Claude Code, or equivalent) as the master build context.

It is not a product overview. It assumes the workflow design — phases, prompts, PCD schema, the Phase 4 gate, the Vault, the Default Judging Rubric — already exists and is captured in the companion documents. This spec translates that design into a concrete CLI runtime: directory layout, helper-script surface, procedure files, adapter integration with Claude Code and Cursor.

---

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Foundational Decisions](#2-foundational-decisions)
3. [Directory Layout](#3-directory-layout)
4. [The Agnostic Core](#4-the-agnostic-core)
5. [The Node Helper — `startup-tools`](#5-the-node-helper--startup-tools)
6. [Prompt Assembly](#6-prompt-assembly)
7. [State Machine & Reconciliation](#7-state-machine--reconciliation)
8. [Intake Pipeline](#8-intake-pipeline)
9. [Claude Code Adapter](#9-claude-code-adapter)
10. [Cursor Adapter](#10-cursor-adapter)
11. [Skill Catalog](#11-skill-catalog)
12. [Build Order](#12-build-order)

---

## 1. Overview & Architecture

Startup runs as a single workspace directory that hosts many hackathon projects side by side. The workspace contains all shared assets — procedures, prompts, schema, helper, Vault — and a `projects/` subdirectory where each hackathon gets its own state folder. The user opens the workspace once in their AI coding agent (Claude Code primarily, Cursor as a credit-friendly fallback) and stays there across hackathons; switching projects is a single command, not a folder copy.

There is no server, no database, no web UI. State lives in plain files. Logic lives in markdown procedures. Mutations go through a small Node.js helper.

The system has three layers:

**The Agnostic Core (`.startup/`).** Everything that *is* Startup — PCD state, schema, procedures, prompts, intake definitions, the Vault, the Node helper. Agent-neutral by construction. Any agent that can read files, follow markdown instructions, and run shell commands can drive this layer.

**The Claude Code Adapter (`.claude/`).** Skills that wrap procedures as slash commands, two subagents for context-protected work (intake parsing and the Grill-Me adversarial mode), and two hooks (SessionStart banner, PreCompact handoff). The adapter is thin — typically each skill file is under 50 lines and delegates to a procedure.

**The Cursor Adapter (`.cursor/`).** A single rules file that points Cursor's agent at the core's entry document and lists invokable commands. Cursor lacks subagent isolation, lifecycle hooks, and auto-discovery, so the experience is honestly degraded relative to Claude Code. Documented openly here rather than papered over.

The user-facing interaction loop, regardless of adapter:

```
User invokes command
  → Adapter loads pointer
  → Helper resolves the active project from .startup/active.txt
  → Procedure (in core) is read and executed
  → Procedure calls startup-tools for state queries (active project's PCD)
  → Prompt is assembled from PROMPTS.md + selected PCD context
  → Output is either executed inline by the agent (default)
    or copied for paste-out to an external LLM (--copy-prompt flag)
  → User runs /startup:intake when ready to commit results
  → Intake subagent parses the inbox file, produces a diff
  → User approves; startup-tools applies the diff atomically to the active project
```

### 1.1 What this architecture buys

- **Build speed.** No frontend, no auth, no encryption layer, no deployment. A solo student working with Cursor can stand up the MVP in days, not months.
- **Editability.** Refining the workflow is editing a markdown file. The procedure files double as documentation and as runtime instructions.
- **Portability.** The core works in any CLI agent that reads files and runs shell commands. New agents get supported with a thin adapter.
- **Auditable state.** The PCD is plain JSON on disk; the phase gate log is an append-only JSONL file; the Vault is a directory of JSON files. Everything inspectable with `cat` and `jq`.
- **No vendor lock-in.** No proprietary runtime, no API keys baked in, no required SaaS dependencies. Optional integrations (paid LLMs, paid agents) are opt-in.

### 1.2 What this architecture costs

- **Directive UI is downgraded.** A web app's cards-and-progress-bars are replaced by a status banner and explicit user invocation. The status banner is informative but less in-your-face.
- **Phase-gate enforcement is advisory, not hard.** A web app could refuse to let the user advance; the CLI version warns loudly and requires a typed acknowledgment, but the user can always edit the PCD file directly. This is acceptable because the user *is* the user and sets up their own constraints.
- **Cursor parity is incomplete.** Without subagents, context-pollution is unavoidable when running intake parsing or the Grill-Me dialogue. Without hooks, auto-resume after compaction is not available. The user accepts these limits when choosing the Cursor adapter.

---

## 2. Foundational Decisions

These are locked. Every other decision in this document is downstream of these.

| Decision | Choice | Why |
|---|---|---|
| Canonical PCD format | JSON on disk, validated against `pcd_schema.json` | Typed access for procedures, helper, and subagents. Markdown views are rendered from JSON, never read back as canonical. |
| Workspace model | One workspace hosts many projects | Each project gets a subdirectory at `.startup/projects/<id>/`. Shared assets (procedures, prompts, schema, helper, Vault) live at workspace root. The user opens the workspace once and switches projects via command. |
| Active project pointer | `.startup/active.txt` (one-line file containing a project ULID) | Single source of truth for "which project am I working on right now." Read by every helper invocation. Updated by `startup-tools switch`. |
| Persistence | Plain files | Free, no auth, no DB, fully inspectable. Encryption is not addressed at the runtime layer — if the user wants protection, they use disk-level encryption or a private git repo. |
| Primary agent | Claude Code | Has skills, subagents, and hooks — the three primitives Startup leans on. The agent itself does the heavy reasoning that the web-app design delegated to external LLMs. |
| Secondary agent | Cursor (agent mode) | The user has community credits. Supported via a thin rules-file adapter with documented feature gaps. |
| Cross-LLM optionality | Preserved via `--copy-prompt` flag | Every skill defaults to inline execution; passing `--copy-prompt` instead writes the assembled prompt to a file for paste-out to Gemini Pro, Claude.ai, Deepseek, Perplexity, etc. |
| State mutation policy | All PCD writes go through `startup-tools` | The model assembles prompts and parses outputs; the helper handles file I/O, schema validation, atomic writes, and staleness propagation. Model and state-mutation responsibilities are separated. |
| Intake mechanism | File-based inbox + subagent parsing | Pasted external-LLM output goes into the active project's `inbox/`; `/startup:intake` spawns a subagent that parses, validates, and produces a diff preview before commit. |
| Helper language | Node.js | Fits the user's stack, runs everywhere Claude Code and Cursor run, has Ajv for schema validation. Single-file CLI, no dependencies beyond Ajv. |
| Prompt content source | `.startup/PROMPTS.md` (workspace-shared) | The authoritative copy of every prompt's role assignment and task spec lives here. Procedures load by section heading. Edits apply to every project in the workspace. |
| Schema source | `.startup/pcd_schema.json` (workspace-shared) | The authoritative PCD schema. Helper validates against this. Subagents extract conforming to fragments of this. |
| Rubric source | `.startup/default_judging_rubric.md` (workspace-shared) | The default rubric. Loaded into `active_judging_rubric.data.criteria` at project init. User can edit; edits apply to *new* projects, not retroactively. |
| Vault location | `.startup/vault/` (workspace-shared) | All projects in a workspace share the same Vault. Cross-project learning is automatic. A second workspace (e.g., for client work) would have its own Vault. |
| Procedure granularity | One procedure file per prompt, per mode | Each prompt in `PROMPTS.md` maps to one procedure file per active mode. Standard-mode procedures live in `procedures/standard/`, fast-mode procedures in `procedures/fast/`, mode-agnostic procedures in `procedures/shared/`. Each procedure still maps to one skill in `.claude/skills/`; the skill resolves the correct procedure path at runtime via `startup-tools procedure-path`. |
| Hackathon mode | Declared at project init via `--mode <standard\|fast>`; stored on `hackathon_profile.data.mode` | Standard mode is the default and runs the full diligence flow. Fast mode targets same-day hackathons (build time under one day) and routes to a compressed procedure tree. Mode is per-project, not per-workspace. The Phase 4 gate, intake pipeline, schema, Vault, and skill catalog are mode-agnostic. Only procedure orchestration and one prompt's task spec differ between modes. |

Two things deliberately not chosen:

- **No plugin packaging in v1.** Plugins are for distribution. The user is solo. When sharing matters, packaging is a one-day job.
- **No MCP server in v1.** The PCD is just files. Adding an MCP layer is a stretch goal for when the data shape stabilizes and external-tool integration becomes desirable.

---

## 3. Directory Layout

Everything Startup needs lives in a single workspace directory. The workspace contains workspace-shared assets at its root and per-project state under `projects/`. Adapter directories (`.claude/`, `.cursor/`) sit alongside `.startup/` at the workspace root and are shared across all projects in the workspace.

```
<workspace-root>/                       # The Startup workspace — one install hosts many projects
│
├── .startup/                           # Agnostic core
│   │
│   │ === Workspace-shared assets ===
│   ├── STARTUP.md                      # Entry document for any agent reading cold
│   ├── pcd_schema.json                 # PCD schema (workspace-shared)
│   ├── PROMPTS.md                      # Prompt catalog (workspace-shared)
│   ├── default_judging_rubric.md       # Default rubric (workspace-shared)
│   ├── active.txt                      # One-line file: ULID of the active project
│   ├── procedures/                     # Per-prompt procedure files (workspace-shared)
│   │   ├── standard/                   # Default mode — full diligence flow
│   │   │   ├── phase-1/
│   │   │   │   └── judge-research.md
│   │   │   ├── phase-2/
│   │   │   │   ├── problem-analysis.md
│   │   │   │   ├── competitive-premortem.md
│   │   │   │   ├── research-hypotheses.md
│   │   │   │   └── user-research-package.md
│   │   │   ├── phase-3/
│   │   │   │   ├── idea-seed-challenge.md
│   │   │   │   ├── scamper.md
│   │   │   │   ├── jtbd.md
│   │   │   │   ├── brainwriting.md
│   │   │   │   ├── divergent-candidates.md
│   │   │   │   ├── differentiation-stress-test.md
│   │   │   │   └── downselection.md
│   │   │   ├── phase-4/
│   │   │   │   ├── design-grill.md
│   │   │   │   ├── product-brief.md
│   │   │   │   ├── tid.md
│   │   │   │   └── demo-script.md
│   │   │   └── phase-6/
│   │   │       ├── pitch-brief.md
│   │   │       ├── pitch-review.md
│   │   │       ├── qa-prep.md
│   │   │       └── judge-qa-drill.md
│   │   ├── fast/                       # Same-day mode — compressed orchestration
│   │   │   ├── phase-1/
│   │   │   │   └── judge-research.md            # one-shot Claude (not Deep Research); Vault-heavy
│   │   │   ├── phase-2/
│   │   │   │   ├── problem-analysis.md          # compressed; relaxed minimums
│   │   │   │   └── competitive-premortem.md     # mandatory; same prompt as standard
│   │   │   ├── phase-3/
│   │   │   │   ├── idea-seed-challenge.md       # optional; identical to standard
│   │   │   │   └── divergent-candidates.md      # invokes prompt with {fast_mode: true}
│   │   │   ├── phase-4/
│   │   │   │   ├── product-brief.md             # parallel-execution choreography
│   │   │   │   ├── tid.md                       # parallel-execution choreography
│   │   │   │   └── demo-script.md               # parallel-execution choreography
│   │   │   └── phase-6/
│   │   │       └── pitch-brief.md               # truncated; review and Q&A deferred or skipped
│   │   └── shared/                     # Mode-agnostic procedures (referenced by both modes)
│   │       ├── phase-7/
│   │       │   └── debrief.md
│   │       └── addendum/
│   │           └── addendum-protocol.md
│   ├── intake/                         # Per-intake JSON definitions (workspace-shared catalog)
│   │   ├── intake_phase_1_judge_research.json
│   │   ├── intake_phase_2_problem_analysis.json
│   │   └── ... (one per entry in PROMPTS.md intake registry)
│   ├── vault/                          # Vault, shared across all projects in this workspace
│   │   ├── judges/
│   │   ├── organizers/
│   │   ├── winning_solutions/
│   │   └── boilerplate/
│   ├── bin/
│   │   └── startup-tools.js            # The Node helper (workspace-shared)
│   │
│   │ === Per-project state ===
│   └── projects/
│       ├── 01HQ...-dost-hackathon/     # One folder per project, named <ULID>-<slug>
│       │   ├── pcd.json                # The Project Context Document for this project
│       │   ├── phase_gate_log.jsonl    # Append-only event log for this project
│       │   ├── handoff.json            # Session-resumption state for this project
│       │   └── inbox/                  # File-based intake landing zone for this project
│       │       └── archive/            # Processed inbox files moved here
│       ├── 01HR...-shopee-hackathon/
│       │   └── ... (same shape)
│       └── _archived/                  # Archived projects moved here (read-only by convention)
│           └── 01HK...-old-hackathon/
│
├── .claude/                            # Claude Code adapter (optional, workspace-shared)
│   ├── settings.json                   # Hooks configuration
│   ├── skills/
│   │   ├── startup-status/
│   │   │   └── SKILL.md
│   │   ├── startup-new-project/
│   │   │   └── SKILL.md
│   │   ├── startup-phase-2-premortem/
│   │   │   └── SKILL.md
│   │   └── ... (one per skill in the catalog)
│   └── agents/
│       ├── intake-parser.md
│       └── grill-me.md
│
└── .cursor/                            # Cursor adapter (optional, workspace-shared)
    └── rules/
        └── startup.mdc
```

The workspace can itself be a git repo. Tracking the workspace in git is the recommended pattern — it versions procedure refinements, prompt iterations, and rubric tweaks across hackathons. Whether to also track per-project state (`.startup/projects/*`) in git is a per-user choice; common patterns are (a) commit everything for full audit history, or (b) gitignore `projects/` if hackathon state contains sensitive material.

### 3.1 Path conventions

Throughout the rest of this spec, paths are written in one of two forms:

- **Workspace-relative paths** start at the workspace root and include `.startup/...` — these refer to workspace-shared assets that exist independent of any project. Example: `.startup/PROMPTS.md`.
- **Active-project paths** are written as `<active-project>/...` and refer to files inside the currently active project's directory. The helper resolves `<active-project>` to `.startup/projects/<active-ulid>/` at runtime, where `<active-ulid>` is read from `.startup/active.txt`. Example: `<active-project>/pcd.json` resolves to `.startup/projects/01HQ.../pcd.json`.

Procedure files use the `<active-project>/` form throughout. The helper handles resolution. The agent never has to know the active project's ULID — it just runs helper commands that operate on the active project by default.

### 3.2 The Vault

The Vault lives at `.startup/vault/` and is shared across all projects in the workspace. There is no separate "shared vault" at the OS level — the workspace *is* the unit of sharing. Cross-project learning happens automatically because every project in the workspace reads from the same Vault directory.

If the user later wants a separate Vault (e.g., for a client-work workspace with different domain knowledge), they create a second workspace. Each workspace has its own Vault. No cross-workspace sync is provided in v1; manual copy is the escape hatch.

`startup-tools vault-match` queries the Vault for entries relevant to the active project's hackathon profile. `startup-tools vault-add <type> <file>` adds new entries. There is no "import to project" step anymore — Vault entries are always live for whichever project is active.

---

## 4. The Agnostic Core

This section specifies the contents of `.startup/`. Each subsection describes one file or subdirectory: what it contains, who reads it, who writes it.

### 4.1 STARTUP.md — the entry document

`STARTUP.md` is the single document an agent reads to understand Startup. Any agent — Claude Code, Cursor, Gemini CLI, anything — that can read this file should be able to operate the system after reading it.

Its required sections:

1. **What this is.** One paragraph describing Startup as a workflow engine, the role of the PCD, the role of phases, the role of the agent.
2. **How to find your bearings.** Instructions to run `node .startup/bin/startup-tools.js status` first, every session, before doing anything else. The status output shows the active project, current phase, and pending items.
3. **The workspace model.** Explanation that one workspace hosts many projects, that `.startup/active.txt` points at the currently active project, that the user switches projects with `/startup:switch <project>` or `startup-tools switch`, and that the agent never has to know the active project's ULID — the helper resolves it automatically.
4. **The phase model.** Brief overview of Phases 0–7 with one-line descriptions and pointers to the procedure directories. Explanation that procedures are organized by mode (`procedures/standard/`, `procedures/fast/`, `procedures/shared/`) and that the active project's mode (set at init via `--mode`) determines which subtree is used. The agent never reads procedure files by hardcoded path — it always asks the helper via `procedure-path <prompt_id>`.
5. **How procedures work.** Procedures are markdown files with numbered steps. Each step is either a shell command to run or an action for the agent to take. The agent follows them in order without paraphrasing.
6. **How prompts work.** Prompt content (role assignments, task specifications, output templates) lives in `PROMPTS.md`. Procedures reference prompts by their `phase_X_name` heading. The standard five-component assembly order is: format-discipline preamble, role assignment, context injection, task specification, output template.
7. **How state mutation works.** All PCD writes go through `startup-tools`. The agent never edits `pcd.json` directly. Specifically: reads use `startup-tools render-context`, writes use `startup-tools apply-diff`, validation uses `startup-tools validate`. Every helper command operates on the active project unless `--project <ulid>` is passed.
8. **How intake works.** External-LLM outputs go into the active project's `inbox/` directory (resolved by the helper, the agent doesn't compute the path). The `intake` procedure picks them up, parses to the schema, produces a diff for user review, and (on approval) commits via the helper.
9. **The Phase 4 gate.** A statement that Phase 4 completion is a one-way scope-lock gate, that backward transitions from Phase 5+ require the Addendum Protocol, and that the gate is enforced advisorily in the CLI (with typed acknowledgment) rather than as a hard block. The gate is per-project — crossing it in one project does not affect others.
10. **Commands reference.** A table of every `startup-tools` subcommand with one-line description.
11. **Adapter-specific notes.** A short section pointing to `.claude/` if using Claude Code, `.cursor/` if using Cursor.

The document should be ~300–500 lines. It is the contract between the core and the agent. It should be edited carefully and kept versioned.

### 4.2 pcd.json — the Project Context Document

`pcd.json` is the canonical state of one project. Its schema is `pcd_schema.json`. It lives at `<active-project>/pcd.json` (e.g., `.startup/projects/01HQ.../pcd.json`). It is initialized by `startup-tools init` (which creates the project subdirectory) and mutated only via `startup-tools apply-diff`.

The schema (per `pcd_schema.json`) defines a structure:

```
{
  "schema_version": "1.0.0",
  "project_id": "<ULID>",
  "project_name": "...",
  "created_at": "<ISO-8601>",
  "archived_at": null,
  "current_phase": 0,
  "phase_4_gate_crossed": false,
  "sections": {
    "hackathon_profile": { "data": {...}, "is_populated": true, "is_stale": false, "last_updated": "...", "populated_by_phase": 0 },
    "judge_intelligence": { ... },
    "active_judging_rubric": { ... },
    "problem_analysis": { ... },
    "competitive_landscape": { ... },
    "user_research": { ... },
    "ideation_record": { ... },
    "candidate_portfolio": { ... },
    "chosen_direction": { ... },
    "scope_definition": { ... },
    "technical_direction": { ... },
    "debrief_record": { ... },
    "external_context_notes": { ... }
  },
  "phase_gate_log": [ ... ]
}
```

Every section wrapper has the same four metadata fields: `is_populated`, `is_stale`, `last_updated`, `populated_by_phase`. These are used by the state machine for reconciliation (Section 7).

The PCD is plain JSON on disk, pretty-printed, UTF-8. The user can `cat` it, `jq` it, or open it in an editor. Direct edits work but bypass validation and staleness propagation — accepted, but flagged in the procedure documentation as a last resort.

### 4.3 Procedures (`.startup/procedures/`)

A procedure is a markdown file specifying how to execute one prompt from `PROMPTS.md`. There is one procedure per prompt, organized by phase. The procedure encodes the *how*; the prompt content is the *what*.

Every procedure has this structure:

```markdown
# <Phase N — Step Name>

Maps to `PROMPTS.md` prompt: `<phase_X_name>`
Produces intake: `<intake_X_name>` (or "none" for direct PCD writes)

## Preconditions

[Plain-language statement of what must be true before this procedure runs.
The agent verifies this in Step 1.]

## Steps

### Step 1: Verify preconditions

Run: `node .startup/bin/startup-tools.js precondition <prompt_id>`

If exit code is nonzero, stop. Tell the user what's missing using the
helper's stderr output. Suggest the procedure(s) that would unblock them.

### Step 2: Load required context

Run: `node .startup/bin/startup-tools.js render-context <section_ids...>`

The required sections are listed in the prompt's "Required context" field
in PROMPTS.md. Read the output — it is markdown-rendered PCD sections.

### Step 3: Load optional context

If the prompt has optional context (Vault matches, etc.), run:
`node .startup/bin/startup-tools.js render-context <section_ids...> --optional`

Vault matches by hackathon profile keywords are returned by:
`node .startup/bin/startup-tools.js vault-match`

### Step 4: Assemble the prompt

Use the standard five-component structure:

1. The format-discipline preamble (constant — see PROMPTS.md "The
   Format-Discipline Preamble" section, copied verbatim).
2. The role assignment (from PROMPTS.md, under `prompt: phase_X_name`,
   `Role assignment` subsection, copied verbatim).
3. The context injection (the rendered context blocks from Steps 2–3,
   each prefixed with `## CONTEXT — <Section_Name>`).
4. The task specification (from PROMPTS.md, `Task specification`
   subsection, with `{token}` placeholders resolved from the PCD).
5. The output template (from PROMPTS.md, `Output template` subsection,
   or derived from `intake/<intake_id>.json` if the illustrative
   template would drift from the schema).

### Step 5: Execute or copy

Default behavior: execute the assembled prompt yourself. Stream the
output. Save it to `<active-project>/inbox/<prompt_id>-<timestamp>.md`
so the intake procedure can pick it up. Use the helper to resolve the
inbox path: `node .startup/bin/startup-tools.js inbox-path` returns
the active project's inbox directory.

If the user invoked with `--copy-prompt`: write the assembled prompt
(not the response) to `<active-project>/inbox/_pending-prompt-<prompt_id>.md`
and tell the user to run it in their external LLM of choice, then save
the response into the same inbox directory.

### Step 6: Hint at next step

Tell the user: "Run `/startup:intake <intake_id>` when you're ready to
commit this to the PCD. Or run this procedure again with adjusted
context if the output didn't land."

## Notes

[Any prompt-specific gotchas, common LLM failure modes, hard blocks,
or design rationale that would be useful to the executing agent.
E.g., for the Competitive Pre-mortem: "This is the only hard-blocked
prompt. Phase 2 cannot advance without it. Be specific in the role-
assignment — vague archetypes defeat the purpose."]
```

The procedure file is what makes the system agent-agnostic. Every step is either a shell command (which works the same in any agent) or a natural-language instruction (which a competent agent can follow).

#### 4.3.1 Mode-aware procedure routing

Procedures are organized by mode under `procedures/`:

- `procedures/standard/` — the full-diligence flow used by default and for any hackathon with a multi-day build window.
- `procedures/fast/` — the compressed flow used for same-day hackathons (build window under one day). Drops prompts that compound their value over days (research hypotheses, user research, the SCAMPER/JTBD/Brainwriting ideation chain, the differentiation stress test, the Design Grill, Phase 6 pitch review and Q&A prep). Preserves the load-bearing pieces (competitive pre-mortem, divergent candidates, all three Phase 4 documents, the Phase 4 gate ceremony). Choreographs Phase 4 documents in parallel.
- `procedures/shared/` — procedures that are genuinely identical across modes: Phase 7 debrief, the Addendum Protocol.

The active project's mode lives at `sections.hackathon_profile.data.mode` (`"standard"` or `"fast"`). Skills do not know the mode directly — they resolve the right procedure file by calling:

```bash
node .startup/bin/startup-tools.js procedure-path <prompt_id>
```

The helper looks up the active project's mode, picks the matching subtree, and returns the full procedure file path. If no mode-specific procedure exists (e.g., `phase_3_scamper` in fast mode), the helper exits with a clear error rather than falling back silently — fast mode legitimately skips those procedures, and an attempt to invoke one is a signal that something is wrong upstream (usually a stale skill auto-invocation or a user expecting standard behavior in a fast project).

**What's mode-agnostic and what isn't.** The schema is mode-agnostic — fast mode just populates fewer sections; populated sections have the same shape. The intake pipeline is mode-agnostic — the same intake definitions run when invoked, just fewer are invoked in fast mode. The Vault is mode-agnostic — both modes read the same workspace Vault. The Phase 4 gate is mode-agnostic — identical typed-acknowledgment ceremony, identical one-way semantics. The skill catalog is mode-agnostic — `/startup:phase-4-tid` is one skill name; it dispatches to `procedures/standard/phase-4/tid.md` or `procedures/fast/phase-4/tid.md` based on the active project's mode. Only the orchestration in procedure files, the Phase 3 → 4 advancement precondition, and one prompt's task spec actually differ between modes.

**Procedure file authoring discipline.** When the same content would appear in both `procedures/standard/<phase>/<name>.md` and `procedures/fast/<phase>/<name>.md`, it should move to `procedures/shared/<name>.md` instead. Drift between near-identical sibling procedures is the main risk this architecture introduces; the mitigation is aggressive use of `shared/`. If a phase's procedure files are more than 75% identical between the two modes, treat that as a smell and refactor.

### 4.4 Intake definitions (`.startup/intake/`)

Each intake is one JSON file. Naming: `<intake_id>.json` matching the intake IDs in PROMPTS.md's "Flash Extraction Notes" section.

The format:

```json
{
  "id": "intake_phase_2_competitive_premortem",
  "display_name": "Competitive Pre-mortem",
  "target_pcd_path": "sections.competitive_landscape.data",
  "target_schema_fragment": { /* JSON Schema fragment matching the target section */ },
  "extraction_prompt": "You are extracting structured data from a markdown document. The source is the response of an LLM that was asked to produce a Competitive Pre-mortem. Extract the predicted_average_solutions array, the spaces_to_avoid array, and the differentiation_seams array into JSON conforming to the schema fragment provided. Also set premortem_completed to true. Output ONLY the JSON object. No commentary.",
  "required_fields": ["predicted_average_solutions", "spaces_to_avoid", "premortem_completed"],
  "optional_fields": ["differentiation_seams"],
  "confidence_threshold": 0.7,
  "commit_strategy": "direct",
  "post_extract_validators": [
    {
      "rule": "min_array_length",
      "path": "predicted_average_solutions",
      "min": 3,
      "max": 5,
      "severity": "flag",
      "message": "Pre-mortem should have 3-5 archetypes. Got {actual}."
    }
  ],
  "extraction_notes_from_prompts_md": "[Copied from PROMPTS.md Flash Extraction Notes section for this intake — common variations, things to watch for, cross-section behavior.]"
}
```

Intake definitions are read by the intake subagent. The subagent loads the intake definition for the target intake, reads the inbox file, runs extraction against the schema fragment, applies post-extract validators, and produces a diff.

The `commit_strategy` enum has three values:

- `direct` — the extracted object becomes the value at `target_pcd_path`.
- `apply_adjustments` — the extracted object is a list of field-level adjustments to apply (used by Design Grill, which produces adjustments to `chosen_direction` rather than overwriting it).
- `append` — the extracted object is appended to an existing array at `target_pcd_path` (used by addenda, debrief vault proposals).

### 4.5 The Inbox (`<active-project>/inbox/`)

The inbox is the staging area for external-LLM outputs awaiting intake. It is **per-project** — each project has its own inbox at `.startup/projects/<ulid>/inbox/`. Files from one project's inbox are never visible to another project.

File naming convention: `<prompt_id>-<ISO-8601-timestamp>.md`. Example: `phase_2_competitive_premortem-2026-05-17T14:32:08.md`.

Reserved filenames:
- `_pending-prompt-<prompt_id>.md` — written by procedures when `--copy-prompt` is used. Contains the assembled prompt itself, ready for paste-out.
- `_diff-<intake_id>-<timestamp>.json` — written by the intake subagent, contains the proposed diff awaiting user approval.

After successful intake, files are moved to `<active-project>/inbox/archive/`. After failed intake, files stay in the inbox with a `.failed` suffix and a sibling `.error.txt` explaining what went wrong.

### 4.6 The Vault (`.startup/vault/`)

The Vault lives at the **workspace** level and is shared across every project in the workspace. There is no per-project Vault snapshot, no import step, no shared OS-level Vault outside the workspace. Cross-project learning is automatic by virtue of all projects reading from the same directory.

Structure:

```
.startup/vault/
├── judges/
│   └── <ulid>.json                    # One file per judge entry
├── organizers/
│   └── <ulid>.json
├── winning_solutions/
│   └── <ulid>.json
└── boilerplate/
    └── <slug>/                        # Folder per boilerplate kit
        ├── README.md
        └── <kit files>
```

Vault entries are JSON with a small structured schema (defined alongside as `vault_schema.json`). The user can manually edit Vault entries; changes take effect immediately for all projects in the workspace. `startup-tools vault-match` returns Vault entries relevant to the active project's hackathon profile (filtered by domain, organizer type, sponsor names, theme keywords). Phase 7 Debrief outputs may produce structured Vault update proposals; these go through the standard intake-diff-approve flow before being written.

If the user later wants Vault isolation between contexts (e.g., a separate workspace for client work where the knowledge shouldn't mingle), they create a second workspace. Each workspace has its own Vault.

### 4.7 Phase Gate Log (`<active-project>/phase_gate_log.jsonl`)

An append-only JSONL file, **per-project**. Each project's log is independent. One JSON object per line. Each entry records a significant state-changing event:

```json
{"timestamp": "2026-05-17T15:00:00Z", "event_type": "phase_advanced", "from_phase": 1, "to_phase": 2, "user_acknowledged_gaps": []}
{"timestamp": "2026-05-17T15:30:00Z", "event_type": "phase_4_gate_crossed", "phase": 4, "scope_lock_acknowledgment": "I understand scope is locked from this point forward."}
{"timestamp": "2026-05-17T16:00:00Z", "event_type": "addendum_initiated", "addendum_id": "add_01HQ...", "justification": "..."}
{"timestamp": "2026-05-17T16:15:00Z", "event_type": "design_grill_skipped", "phase": 4, "first_document": "product_brief"}
```

Event types: `project_created`, `project_switched_to`, `phase_advanced`, `phase_reverted`, `phase_4_gate_crossed`, `addendum_initiated`, `addendum_completed`, `design_grill_skipped`, `intake_committed`, `manual_pcd_edit`, `vault_updated`, `project_archived`, `mode_changed`.

The log is the source of truth for the Phase 7 Debrief for that project. It is exported alongside the PCD.

### 4.8 Handoff (`<active-project>/handoff.json`)

Per-project. Written by the `PreCompact` hook (Claude Code only) or manually via `startup-tools handoff`. Read by the `SessionStart` hook or `startup-tools resume`. When the user switches projects, the previous project's handoff is preserved untouched — switching back later restores its resumption hint.

Schema:

```json
{
  "written_at": "2026-05-17T16:45:00Z",
  "current_phase": 3,
  "current_step_in_procedure": "phase-3/scamper.md:Step 4",
  "last_skill_invoked": "startup:phase-3-scamper",
  "pending_inbox_files": ["phase_3_scamper-2026-05-17T16:40:00.md"],
  "pending_diff_files": [],
  "recent_decisions_summary": "User decided to prioritize the 'invert' SCAMPER bucket for divergent candidate generation. Three SCAMPER buckets are populated; four remain.",
  "resumption_hint": "Continue by running /startup:intake intake_phase_3_scamper, then /startup:phase-3-jtbd."
}
```

The session-start banner reads this file and surfaces the resumption hint if it exists and is fresh (< 7 days).

---

## 5. The Node Helper — `startup-tools`

A single-file Node.js script at `.startup/bin/startup-tools.js`. ~300–500 lines. Dependencies: Ajv (schema validation) and `ulid` (ID generation). Pure CommonJS or ESM, no build step.

### 5.1 CLI surface

The helper is invoked as `node .startup/bin/startup-tools.js <subcommand> [args]`. The procedure files use this form throughout for portability.

**Active-project resolution.** Every subcommand that operates on project state (PCD reads, diffs, inbox, log, handoff, phase ops, addenda) implicitly targets the active project — the one whose ULID is in `.startup/active.txt`. To target a different project for a single command, pass `--project <ulid>` (or `--project <slug>`). To change the active project persistently, use `switch`. Subcommands that operate at workspace level (`list`, `switch`, `archive`, `vault-*`, `validate-workspace`) ignore the active-project pointer.

**Two special cases.** `init` creates a new project and, by default, sets it as the active project (overridable with `--no-activate`). `workspace-init` (described below) scaffolds the workspace itself the very first time and is typically only run once.

Subcommands:

| Subcommand | Purpose | Output |
|---|---|---|
| `workspace-init` | Scaffold a brand-new workspace. Creates `.startup/` with shared assets (schema, PROMPTS, rubric, procedures, intake defs, helper), but no projects. Run once per workspace. | Confirmation. |
| `init [--name <name>] [--mode <standard\|fast>] [--no-activate]` | Scaffold a new project inside the workspace. Creates `.startup/projects/<ulid>-<slug>/` with initial `pcd.json`, empty inbox, empty log. Sets `sections.hackathon_profile.data.mode` from `--mode` (default `standard`). Updates `active.txt` unless `--no-activate`. | New project ULID, mode, and confirmation. |
| `list [--include-archived]` | List all projects in the workspace with metadata (name, current phase, last activity, active marker). | Table to stdout. |
| `switch <ulid_or_slug>` | Set the active project. Validates the target exists and is not archived. Appends `project_switched_to` to the target's log. | Confirmation showing previous and new active project. |
| `archive [--project <ulid>]` | Move the active (or specified) project into `.startup/projects/_archived/`. Sets `archived_at` in the PCD. If archiving the active project, clears `active.txt`. | Confirmation. |
| `unarchive <ulid_or_slug>` | Restore an archived project to the active list. Clears `archived_at`. | Confirmation. |
| `status [--project <ulid>]` | Print full status of the active (or specified) project. | Multi-line summary: project name, current phase, populated sections, stale sections, pending inbox files, last activity. |
| `banner` | Print a compact banner of the active project (used by SessionStart hook). If no active project, prints workspace-level summary and a hint to run `/startup:new-project` or `/startup:switch`. | One-screen summary. |
| `mode` | Print the active project's mode (`standard` or `fast`) to stdout. Used by skills that need to make mode-dependent decisions. | One word. |
| `procedure-path <prompt_id>` | Resolve the procedure file path for a given prompt in the active project's mode. Checks `procedures/<mode>/...` first, then `procedures/shared/...`. Exits nonzero if no procedure exists for this prompt in this mode (e.g., `phase_3_scamper` in fast mode — legitimately skipped, not an error in the workflow but a signal the caller shouldn't be invoking it). | Path to stdout. |
| `set-mode <standard\|fast>` | Change the active project's mode. Writes to `sections.hackathon_profile.data.mode`. Logs a `mode_changed` event to `phase_gate_log.jsonl`. Surfaces any sections that may now be considered gaps (e.g., switching standard → fast leaves SCAMPER/JTBD populated but mode-skipped; switching fast → standard surfaces "newly missing" sections that the standard flow expects). Does NOT delete data; only changes routing. | Confirmation and summary of newly-gap-or-orphan sections. |
| `precondition <prompt_id>` | Verify a prompt's preconditions are met for the active project. | Exits 0 if OK, nonzero with stderr explanation if not. |
| `render-context <section_ids...> [--optional]` | Render selected PCD sections of the active project as markdown. | Markdown to stdout. |
| `render-pcd` | Render the full PCD of the active project as markdown (for export). | Full markdown to stdout. |
| `validate` | Validate the active project's `pcd.json` against `pcd_schema.json`. | Exits 0 if valid, nonzero with errors. |
| `validate-workspace` | Sanity-check the workspace: shared assets present, every project's PCD valid, `active.txt` points at a real project. | Exits 0 if OK, nonzero with per-issue report. |
| `apply-diff <diff-file>` | Validate and apply a JSON diff to the active project's `pcd.json`. Updates `is_populated`, `last_updated`, `populated_by_phase`. Triggers staleness propagation. | Confirmation, list of fields changed, list of newly stale sections. |
| `advance-phase` | Attempt to advance `current_phase` of the active project. Verifies all preconditions for the next phase. Writes phase_gate_log entry. | Confirmation or list of blockers. |
| `back-phase <target_phase>` | Revert `current_phase` of the active project. Marks all sections populated after that phase as stale. Writes log entry. | Confirmation and list of newly stale sections. |
| `cross-phase-4-gate` | Cross the one-way gate for the active project. Requires a typed acknowledgment passed as an argument. Sets `phase_4_gate_crossed = true`. | Confirmation. |
| `inbox-path` | Print the active project's inbox directory absolute path. Used by procedures so the agent doesn't have to compute it. | Path to stdout. |
| `inbox-list` | List pending inbox files in the active project with metadata. | Table to stdout. |
| `inbox-archive <file>` | Move a processed inbox file to the active project's archive. | Confirmation. |
| `vault-list <type>` | List vault entries by type (workspace-shared). | Table to stdout. |
| `vault-get <type> <id>` | Print a vault entry as JSON. | JSON to stdout. |
| `vault-add <type> <file>` | Add a new vault entry from a JSON file (workspace-shared, affects all projects). | New entry ID. |
| `vault-match` | Find Vault entries matching the active project's hackathon profile. | Markdown-rendered list, suitable for context injection. |
| `gate-log <event_type> [payload-json]` | Append an event to the active project's `phase_gate_log.jsonl`. | Confirmation. |
| `handoff` | Write `handoff.json` for the active project from current state. | Confirmation. |
| `resume` | Read the active project's `handoff.json` and produce a resumption hint. | Hint to stdout. |
| `addendum-init <justification>` | Begin the Addendum Protocol for the active project (must have crossed Phase 4 gate). | New addendum ID and procedure pointer. |
| `addendum-finalize <addendum_id> <diff-file>` | Commit an addendum to the active project after user approval. | Confirmation. |

### 5.2 Behavior contracts

**Atomicity.** `apply-diff` writes to a temp file (`pcd.json.tmp`), validates the result, then renames over `pcd.json`. Validation failure leaves `pcd.json` unchanged. Same pattern for vault writes.

**Schema enforcement.** Every write is validated against `pcd_schema.json` before commit. Validation errors are surfaced verbatim with JSON paths.

**Staleness propagation.** When a section is updated, any downstream section that was populated *after* it (by phase number) is flagged as `is_stale: true` if and only if there is a documented dependency. Dependencies are encoded in a small static map inside the helper:

```js
const SECTION_DEPENDENCIES = {
  // section_id: [list of section_ids that depend on it]
  hackathon_profile: ['judge_intelligence', 'active_judging_rubric', 'problem_analysis'],
  judge_intelligence: ['competitive_landscape', 'chosen_direction'],
  problem_analysis: ['competitive_landscape', 'ideation_record'],
  competitive_landscape: ['ideation_record', 'candidate_portfolio'],
  user_research: ['ideation_record', 'candidate_portfolio', 'chosen_direction'],
  ideation_record: ['candidate_portfolio'],
  candidate_portfolio: ['chosen_direction'],
  chosen_direction: ['scope_definition', 'technical_direction'],
  scope_definition: ['technical_direction'],
};
```

Backward transitions trigger the same logic — reverting to Phase N marks all sections with `populated_by_phase > N` as stale.

**Exit codes.** 0 = success. 1 = validation error. 2 = precondition unmet. 3 = file/IO error. 4 = user-confirmation required (e.g., crossing Phase 4 gate without acknowledgment).

**No network.** The helper makes no network calls. Period.

**No model.** The helper does no LLM inference. It is pure file and validation logic. Reasoning is the agent's job.

### 5.3 Implementation notes

- Single file. No external dependencies beyond Ajv and ulid (both in npm, both stable).
- Use `process.argv` parsing manually — no Commander.js needed at this scale.
- Use the synchronous Node FS API throughout. Async adds complexity without benefit at this scale.
- Pretty-print all JSON writes with 2-space indentation.
- Append-only JSONL writes use `appendFileSync`.
- The helper is invoked many times per session. Keep cold-start fast — no top-level module imports that aren't needed for every command.

---

## 6. Prompt Assembly

Prompts are assembled by procedures, not by the helper. The procedure file specifies the steps; the agent (Claude or Cursor) does the concatenation; the prompt content comes from `PROMPTS.md`.

### 6.1 The five-component structure

Every assembled prompt has this shape, in this order:

```
[1] FORMAT-DISCIPLINE PREAMBLE   ← constant, copied verbatim from PROMPTS.md
[2] ROLE ASSIGNMENT              ← from PROMPTS.md, per-prompt
[3] CONTEXT INJECTION            ← rendered PCD sections + optional Vault matches
[4] TASK SPECIFICATION           ← from PROMPTS.md, per-prompt, with placeholders resolved
[5] OUTPUT TEMPLATE              ← from PROMPTS.md illustrative + schema-derived
```

This structure is invariant. Procedures do not deviate.

### 6.2 Token resolution

PROMPTS.md uses `{snake_case_token}` placeholders in task specifications. Examples: `{hackathon_name}`, `{theme}`, `{organizer_name}`.

Tokens resolve to dotted paths in the PCD. The agent resolves these by reading the relevant PCD section. If a token has no resolvable value, the agent stops and surfaces the gap to the user — it does not silently insert empty strings or hallucinate values.

The token-to-path mapping is documented in `STARTUP.md` Section "How prompts work" and is straightforward:
- `{hackathon_name}` → `sections.hackathon_profile.data.hackathon_name`
- `{theme}` → `sections.hackathon_profile.data.theme`
- `{sponsor_list}` → joined comma-separated of `sections.hackathon_profile.data.sponsors[]`

For tokens not following this naming pattern (rare), the prompt's `Required context` field in PROMPTS.md lists the source paths explicitly.

### 6.3 Context injection

The agent calls `startup-tools render-context <section_ids...>` to get markdown-rendered PCD sections. Each section is prefixed with a header:

```markdown
## CONTEXT — Hackathon Profile

[rendered hackathon_profile section as markdown]

## CONTEXT — Problem Analysis

[rendered problem_analysis section as markdown]

## CONTEXT — Vault: Judge & Organizer References

[rendered vault matches as markdown]
```

The order matches the prompt's `Required context` field. Optional context follows required context. Vault matches come last.

### 6.4 Output template derivation

Each prompt has an illustrative output template in PROMPTS.md. The actual template the agent uses should be derived from the corresponding intake schema fragment in `.startup/intake/<intake_id>.json` whenever the illustrative version is shorter or older than the schema.

In practice, the procedure tells the agent: "Use the illustrative template from PROMPTS.md. If the schema fragment includes fields the illustrative version omits, append them. If the illustrative version includes fields the schema doesn't, drop them."

This keeps PROMPTS.md as the human-readable reference and `intake/*.json` as the authoritative schema source.

### 6.5 The `--copy-prompt` flag

When the user invokes a skill with `--copy-prompt`, the procedure changes Step 5:

- Default: assemble prompt, execute, save response to `<active-project>/inbox/`.
- `--copy-prompt`: assemble prompt, write to `<active-project>/inbox/_pending-prompt-<prompt_id>.md`, tell user to run externally.

The agent resolves `<active-project>` via `startup-tools inbox-path`.

The format of `_pending-prompt-<prompt_id>.md`:

```markdown
# Prompt for <prompt_id>

Run this in your preferred external LLM (Claude.ai, Gemini Pro, Deepseek, etc.).
Save the response to the inbox directory printed by:
  node .startup/bin/startup-tools.js inbox-path
Use filename: <prompt_id>-<YYYY-MM-DDTHH:mm:ss>.md
Then run: /startup:intake <intake_id>

---

[The full assembled prompt, ready to copy-paste]
```

This preserves cross-LLM optionality at zero friction in the default path.

### 6.6 Runtime placeholders and the `{fast_mode}` flag

PROMPTS.md prompts may include runtime placeholders that procedures resolve at assembly time. Most placeholders are content tokens (`{hackathon_name}`, `{theme}`, etc.) resolved from the PCD. A small number are *behavior placeholders* that the procedure file resolves from the active project's mode or from skill flags.

The relevant behavior placeholder for fast mode is `{fast_mode}` on the `phase_3_divergent_candidates` prompt. When the prompt is invoked from `procedures/standard/phase-3/divergent-candidates.md`, the placeholder resolves to `false` and the task spec includes the standard instruction to synthesize from upstream SCAMPER/JTBD/Brainwriting outputs (with those outputs injected as context). When invoked from `procedures/fast/phase-3/divergent-candidates.md`, the placeholder resolves to `true`; the task spec adjusts to instruct the LLM to synthesize candidates directly from problem analysis, competitive landscape, and (if present) the team idea seed — without requiring any ideation-pipeline context. The archetype-coverage discipline and the disqualification rule against `spaces_to_avoid` overlap remain identical in both flavors.

This is the only prompt that requires a fast-mode-aware variant. All other prompts either run identically in both modes (`phase_2_competitive_premortem`, `phase_4_product_brief`, `phase_4_tid`, `phase_4_demo_script`, `phase_6_pitch_brief`, `phase_7_debrief`) or are entirely skipped in fast mode (`phase_2_research_hypotheses`, `phase_2_user_research_package`, `phase_3_scamper`, `phase_3_jtbd`, `phase_3_brainwriting`, `phase_3_differentiation_stress_test`, `phase_4_design_grill`, `phase_6_pitch_review`, `phase_6_qa_prep`, `phase_6_judge_qa_drill`). The procedure files in `procedures/fast/` simply do not invoke the skipped prompts.

**Why not multiple variant prompts in PROMPTS.md.** Centralizing the prompt registry is more important than avoiding a single conditional in one task spec. Adding `phase_3_divergent_candidates_fast` as a separate prompt definition would mean two near-identical prompt entries that drift over time. The runtime placeholder keeps both flavors anchored to one prompt definition, one role assignment, one negative-constraints list. The drift surface stays minimal.

---

## 7. State Machine & Reconciliation

The phase model is a directed graph with forward and backward edges, and one absorbing one-way gate at Phase 4.

### 7.1 Phases

| Phase | Name | Key Deliverables |
|---|---|---|
| 0 | Setup & Vault Refresh | Vault import, hackathon profile capture |
| 1 | Intelligence Gathering | Judge personas, active rubric, past winner patterns |
| 2 | Problem Deconstruction | Problem analysis, competitive pre-mortem, research hypotheses, user research |
| 3 | Differentiated Ideation | SCAMPER, JTBD, brainwriting, divergent candidates, downselection |
| 4 | Solution Design & Scope Lock | Product Brief, TID, Demo Script — **gate phase** |
| 5 | Build | Code (handed off to Cursor / Claude Code) |
| 6 | Pitch & Presentation | Pitch brief, pitch review, Q&A prep, judge drill |
| 7 | Post-Hackathon Debrief | Debrief, Vault updates |

### 7.2 Forward transitions

`advance-phase` checks the *advancement preconditions* for the next phase. The Phase 3 → 4 precondition is mode-aware; all others are mode-agnostic:

- Phase 0 → 1: hackathon_profile populated.
- Phase 1 → 2: judge_intelligence populated, active_judging_rubric set.
- Phase 2 → 3: problem_analysis populated, **competitive_landscape.data.premortem_completed === true** (the only hard block, **enforced in both modes**).
- Phase 3 → 4 (standard mode): ideation_record has at least one of {scamper_outputs, jtbd_analysis, brainwriting_round} populated, candidate_portfolio populated, chosen_direction set.
- Phase 3 → 4 (fast mode): candidate_portfolio populated, chosen_direction set. The ideation prerequisite (scamper/jtbd/brainwriting) is dropped because fast mode synthesizes candidates directly from problem analysis and competitive landscape via the `{fast_mode}`-flagged divergent-candidates prompt.
- Phase 4 → 5: Product Brief, TID, and Demo Script all in scope_definition / technical_direction. **Crossing this phase requires the user to type the scope-lock acknowledgment. Identical in both modes.**
- Phase 5 → 6: User-driven; no automated check.
- Phase 6 → 7: pitch_brief populated. (Fast mode produces pitch_brief from the truncated Phase 6 procedure; the precondition is otherwise identical.)
- Phase 7 → archived: debrief_record populated.

If any precondition fails, `advance-phase` exits nonzero with a list of gaps. The user can:
- Address the gaps and retry.
- Override with `--acknowledge-gaps "<reason>"` (logged in phase_gate_log).

The **competitive_premortem** hard block is the only one where override is *not* offered — the helper refuses regardless. This is encoded in the procedure file's Notes section and in the helper's logic.

### 7.3 Backward transitions

`back-phase <target>` is allowed up to and including Phase 4. After Phase 4 is crossed, backward transitions require the Addendum Protocol.

Mechanics:
1. Set `current_phase = target`.
2. For every section S where `S.populated_by_phase > target`, set `S.is_stale = true`.
3. Append a `phase_reverted` event to phase_gate_log.
4. Surface to the user a list of newly stale sections with a suggestion: "These sections were derived from earlier phases. Review and either re-run the producing procedure, or mark as still-valid with `startup-tools mark-valid <section>`."

`mark-valid` is a manual override that clears `is_stale` without re-running anything. Useful when the user judges that a backward transition didn't actually invalidate downstream work.

### 7.4 The Phase 4 gate

Crossing Phase 4 is permanent. Mechanics:

1. User invokes `/startup:advance-phase` from Phase 4.
2. Helper checks that Product Brief, TID, and Demo Script are populated.
3. Helper prints the scope-lock language and prompts the user to type:
   > *"I understand scope is locked. Changes from this point require the Addendum Protocol."*
4. On exact match, helper sets `phase_4_gate_crossed = true` and `current_phase = 5`.
5. Helper appends a `phase_4_gate_crossed` event.

After the gate, `back-phase` to any phase ≤ 4 is refused by the helper. The only paths to scope adjustment are:
- The Addendum Protocol (which is logged, friction-heavy, and produces a structured addendum record).
- Manual PCD editing (last resort, flagged in the log as `manual_pcd_edit`).

### 7.5 The Addendum Protocol

After the Phase 4 gate is crossed, scope changes go through:

1. User invokes `/startup:addendum`.
2. Procedure prompts the user for: change summary, justification, current build state snapshot.
3. Helper assigns an addendum ID and creates a draft.
4. Procedure runs the Addendum Protocol prompt from PROMPTS.md, producing impact analysis (impacted parts, revised do-not-build list, risks introduced).
5. User reviews and approves.
6. `startup-tools addendum-finalize` applies the diff to scope_definition and appends an `addendum_completed` event.

Original do-not-build entries must remain in the revised list (verified by the intake's post-extract validator). New entries can be added with a `NEW-tagged` marker.

### 7.6 Manual PCD edits

Users can edit `pcd.json` directly. Doing so:
- Bypasses schema validation (but `startup-tools validate` will catch this on next invocation).
- Bypasses staleness propagation.
- Is logged on the next helper invocation as a `manual_pcd_edit` event (the helper detects the change by comparing `pcd.json` mtime to the last log timestamp).

This is an explicit escape hatch, not a recommended path. The procedure documentation discourages it.

### 7.7 Mode declaration and switching

Mode is declared once at project init via `startup-tools init --mode <standard|fast>`. The value lives at `sections.hackathon_profile.data.mode` and propagates to every subsequent helper invocation (precondition checks, procedure resolution, advance-phase logic, status output).

**Switching mid-event.** The user can switch modes after init via `startup-tools set-mode <target>`. The mechanics:

- **standard → fast.** Allowed at any phase up to and including Phase 4. Sections populated in standard mode (SCAMPER, JTBD, Brainwriting, hypotheses, user research, stress test results) are preserved verbatim and remain in the PCD — they are simply no longer required by the advancement preconditions. The next phase's procedure file changes (now sourced from `procedures/fast/`). No data is lost.

- **fast → standard.** Allowed at any phase up to and including Phase 4. Sections that standard mode expects but fast mode skipped (e.g., scamper_outputs, research_hypotheses) become *gaps* — `is_populated: false` and the advance-phase precondition will refuse to pass without them. The helper surfaces the new gaps explicitly in the `set-mode` output so the user knows what they're now expected to fill.

- **After the Phase 4 gate is crossed**, mode is locked. `set-mode` refuses with `exit code 2` and a message: "Phase 4 gate has been crossed; mode is fixed for the remainder of the project. Use the Addendum Protocol for scope changes."

**Why mode-switching is allowed.** Time estimates change mid-event. A "same-day" hackathon may unexpectedly extend; a "week-long" hackathon may compress to a sprint. The user is the user; the system trusts them to know when their available time has changed. The Phase 4 gate lock is the only inviolable mode-related constraint, and it exists because scope is committed at the gate — not because mode itself is sacred.

Every mode transition writes a `mode_changed` event to the phase gate log with the from-mode, to-mode, current phase, and list of newly-gap-or-orphan sections.

---

## 8. Intake Pipeline

The intake pipeline turns pasted external-LLM output into validated PCD section updates. It is the most reliability-sensitive part of the system.

### 8.1 Three-layer reliability model

**Layer 1: Structured prompts.** Every prompt in PROMPTS.md mandates an explicit output template. When the user pastes back, the input is already roughly shaped like the schema. Extraction is mapping-by-header, not free-form parsing.

**Layer 2: Subagent isolation.** Intake parsing runs in a Claude Code subagent (or, in Cursor, a freshly opened agent session — see Section 10). The subagent has only the inbox file and the intake definition in its context, no noise from the main conversation. It produces structured output that conforms to the schema fragment.

**Layer 3: Diff preview + user approval.** No diff is committed automatically. The subagent writes a proposed diff to `<active-project>/inbox/_diff-<intake_id>-<timestamp>.json`. The main session shows the diff. The user approves, edits, or rejects.

### 8.2 The intake flow, end to end

```
1. User runs /startup:intake <intake_id>
2. Procedure verifies an inbox file matching <intake_id> exists. If multiple, picks newest.
3. Procedure spawns the intake-parser subagent with:
   - Inbox file path
   - Intake definition path (.startup/intake/<intake_id>.json)
   - Target schema fragment
4. Subagent:
   a. Reads the inbox file.
   b. Reads the intake definition.
   c. Performs extraction (model generates JSON conforming to schema fragment).
   d. Runs post-extract validators.
   e. Computes confidence score from validator results.
   f. Writes proposed diff to the active project's inbox as `_diff-<intake_id>-<timestamp>.json`
   g. Returns a structured summary: confidence tier, fields extracted,
      validator flags, fields requiring user review.
5. Main session reads the diff file and renders it as a human-readable
   diff preview (markdown table or side-by-side).
6. User responds with one of:
   - "approve" → procedure runs `startup-tools apply-diff <diff-file>`
   - "edit" → user edits the diff file directly, then says "approve"
   - "reject" → procedure archives the inbox file with .failed suffix
   - "rerun" → procedure deletes the diff and re-spawns the subagent
7. On approve: helper applies, moves inbox file to archive, appends
   intake_committed event to log, surfaces any propagated staleness.
```

### 8.3 Confidence tiers

The subagent grades every extraction into one of three tiers:

- **Green (confidence ≥ 0.9):** All required fields extracted cleanly, no validator flags, no ambiguity. Diff preview is brief; user can approve without close reading.
- **Yellow (0.7 ≤ confidence < 0.9):** Required fields present but one or more validators flagged warnings, or one or more fields needed inference. Diff preview highlights the flagged areas with explanations. User reviews carefully.
- **Red (confidence < 0.7):** Required fields missing, validators failed, or the source document didn't match the expected shape. Subagent does not produce a commit-ready diff; instead it returns a structured "this didn't extract cleanly, here's what went wrong" report. User decides whether to re-run the producing procedure (with adjusted context) or edit manually.

Confidence threshold per intake is configurable in the intake definition file (`confidence_threshold` field).

### 8.4 Cross-section commits

Some intakes write to more than one section. Example: the TID intake populates `scope_definition.data.moscow` and `technical_direction.data.tech_stack` as well as its primary target. The intake definition lists all paths; the helper validates each before commit; the diff preview shows all affected sections.

### 8.5 Manual intake

If the user produces an output they want to commit without going through the parser, they can:
1. Save the output as JSON conforming to the schema fragment.
2. Run `startup-tools apply-diff <file.json>` directly.

This is documented as the escape hatch when the subagent is repeatedly mis-extracting and the user wants to short-circuit.

---

## 9. Claude Code Adapter

The Claude Code adapter consists of three components: the skill catalog, two subagents, and two hooks. All live under `.claude/` at the project root.

### 9.1 Skills

A skill file at `.claude/skills/<skill-name>/SKILL.md` has this shape:

```markdown
---
name: startup:phase-2-premortem
description: Run the Competitive Pre-mortem analysis for Phase 2. The only hard-blocked prompt in the workflow — Phase 2 cannot complete without it. Use when the team has completed problem analysis and is ready to map the crowded space before ideating. Auto-invoke when the user mentions "premortem", "crowded space", or asks about predicted competitor solutions for the current brief.
---

You are running the Phase 2 Competitive Pre-mortem procedure.

1. Run `node .startup/bin/startup-tools.js procedure-path phase_2_competitive_premortem` to resolve the correct procedure file for the active project's mode.
2. Read and follow the steps in the resolved procedure file exactly. Do not paraphrase the procedure's instructions — execute them.
3. The procedure references `PROMPTS.md` for prompt content. Load prompt content verbatim, do not rewrite.
4. The procedure references `startup-tools` for state ops. Run those commands as specified, do not skip them.
5. If preconditions fail, stop and report to the user.
6. If `procedure-path` exits nonzero (this prompt is skipped in the active project's mode), tell the user that this prompt is not part of their current mode's flow, and suggest `/startup:set-mode standard` if they want the full diligence flow.
7. After execution, suggest the next step (typically `/startup:intake intake_phase_2_competitive_premortem`).

Flags this skill supports:
- `--copy-prompt`: Write the assembled prompt to the active project's inbox instead of executing inline. The user will run it in an external LLM. Use `node .startup/bin/startup-tools.js inbox-path` to resolve the path.
- `--mode <interactive|auto>`: For prompts that support Grill-Me mode, choose between interactive Q&A or auto-grill. Defaults vary per prompt — see procedure.
```

Skill files are intentionally thin. The reasoning logic is in the procedure. The skill exists to:
- Register a slash command (`/startup:phase-2-premortem`).
- Provide an auto-invoke description so Claude can suggest the skill when the user's wording matches.
- Pass through flags.

### 9.2 Subagents

Two subagents live at `.claude/agents/`:

**`.claude/agents/intake-parser.md`**

```markdown
---
name: intake-parser
description: Parses external-LLM outputs from the active project's inbox into structured PCD updates. Use whenever a /startup:intake skill needs to extract structured data from a markdown response. Returns a diff for user review; does not commit directly.
tools:
  - Read
  - Write
  - Bash
---

You are the Startup intake parser. You convert pasted external-LLM output into typed PCD section updates.

Your inputs (passed in by the spawning procedure) are:
1. An inbox file path — a markdown file inside the active project's inbox directory. The spawning procedure resolves this path via `node .startup/bin/startup-tools.js inbox-path` and passes the full path explicitly. You do not compute the path yourself.
2. An intake definition path (`.startup/intake/<intake_id>.json` — workspace-shared, same path regardless of active project).
3. The target PCD path (from the intake definition).
4. The output diff directory — the same active project inbox directory as input 1. The spawning procedure passes this explicitly.

Your process:
1. Read the inbox file.
2. Read the intake definition, especially:
   - target_schema_fragment
   - extraction_prompt
   - required_fields
   - optional_fields
   - post_extract_validators
   - extraction_notes_from_prompts_md (common variations, things to watch for)
3. Extract JSON conforming to target_schema_fragment. Output JSON only. No commentary in the JSON itself.
4. Run post-extract validators against the extracted JSON. Compute a confidence score:
   - 1.0 if all required fields present, no validator flags.
   - Subtract 0.1 per validator flag (severity: flag).
   - Subtract 0.3 per validator failure (severity: fail).
   - Subtract 0.2 per missing optional field that the source clearly tried to address.
   - Floor at 0.0.
5. Write a proposed diff to `<output-diff-directory>/_diff-<intake_id>-<timestamp>.json` (the directory was passed in by the spawning procedure). The diff structure:
   {
     "intake_id": "...",
     "target_pcd_path": "...",
     "commit_strategy": "direct|apply_adjustments|append",
     "extracted_data": { ... },
     "validator_results": [ ... ],
     "confidence_score": 0.85,
     "confidence_tier": "yellow",
     "review_notes": "Plain-language summary of what was extracted, what was inferred, and what needs user attention."
   }
6. Return a brief summary to the main session: confidence tier, fields extracted, flags requiring user attention. Do not return the full extracted data — that's in the diff file.

Constraints:
- Do not invent values. If a required field is absent in the source, leave it absent and flag.
- Do not paraphrase the source. If the source says "users want X", extract that, do not rewrite to "X is needed by users".
- Do not write to pcd.json. Only write to the diff file. The main session and helper handle commit.
- If the source document looks materially off-template (e.g., the LLM produced an essay instead of the requested structure), set confidence_tier to red and return a diagnostic rather than a forced extraction.
```

**`.claude/agents/grill-me.md`**

```markdown
---
name: grill-me
description: Runs the adversarial Socratic interrogation for Grill-Me-enabled prompts (Design Grill, Differentiation Stress Test, Judge Q&A Drill). Probes the user's reasoning one question at a time until all branches are tested, then returns a Resolved Summary. Use when a procedure marks the prompt as requiring Grill-Me mode.
tools:
  - Read
  - Write
---

You are conducting an adversarial Socratic interrogation. Your job is to stress-test the team's reasoning, not to validate it.

Your inputs are:
1. The PCD context relevant to this interrogation (rendered markdown).
2. The prompt content (role assignment, task spec) from PROMPTS.md.
3. The mode: "interactive" (live Q&A with the user) or "auto" (you simulate both sides).

Your process — interactive mode:
1. Identify every branch in the design tree that should be tested. List them internally.
2. Ask the user one question at a time. Each question probes a specific branch. Press until the branch is fully resolved — the user has either defended the position or conceded an adjustment.
3. Do not give up early. Do not accept hedge language. "We'll figure that out later" is not a resolution.
4. Move to the next branch when the current one is resolved.
5. When all branches are tested, emit the Resolved Summary in the format specified by the prompt (see PROMPTS.md output template for the specific prompt being grilled).

Your process — auto mode:
1. Identify every branch.
2. For each branch, simulate the dialogue: pose the question, generate the most credible team response, then continue pressing until the branch is resolved.
3. Be honest about the simulated team's likely weaknesses. Do not let the simulated team off easy.
4. Emit the Resolved Summary in the format specified by the prompt.

Output: write the Resolved Summary to the active project's inbox as `<prompt_id>-<timestamp>.md` (the spawning procedure passes the inbox path explicitly via `node .startup/bin/startup-tools.js inbox-path`). The intake procedure then picks it up. Return a brief summary to the main session: which branches were tested, headline findings, how many adjustments were proposed.

Constraints:
- Adversarial, not destructive. The job is to surface real weaknesses, not to manufacture criticism.
- Specific, not abstract. "Consider whether X" is not a question. "If the API call returns 429, what does the demo show?" is.
- Do not propose adjustments outside the prompt's scope. Design Grill adjustments target chosen_direction, not tech stack or architecture.
- Do not pad the adjustment list. An empty list is a valid outcome when the team's reasoning survives interrogation.
```

### 9.3 Hooks

Two hooks configured in `.claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node .startup/bin/startup-tools.js banner"
          }
        ]
      }
    ],
    "PreCompact": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node .startup/bin/startup-tools.js handoff"
          }
        ]
      }
    ]
  }
}
```

**SessionStart** runs `startup-tools banner` at the start of every Claude Code session. The banner output is short — current project, current phase, pending items, resumption hint if a recent handoff exists. The user sees their bearings without typing anything.

**PreCompact** runs `startup-tools handoff` immediately before context compaction. The helper reads the active project's PCD and inbox state and writes `<active-project>/handoff.json`. On the next session, the banner picks it up.

If the user is not in a Startup project (no `.startup/` directory), the helper exits silently and the banner shows nothing.

### 9.4 settings.json full example

```json
{
  "hooks": {
    "SessionStart": [
      { "hooks": [{ "type": "command", "command": "node .startup/bin/startup-tools.js banner" }] }
    ],
    "PreCompact": [
      { "hooks": [{ "type": "command", "command": "node .startup/bin/startup-tools.js handoff" }] }
    ]
  }
}
```

No model overrides, no tool restrictions, no MCP servers in MVP. Lean.

---

## 10. Cursor Adapter

The Cursor adapter is a single rules file. It is intentionally minimal because Cursor's agent lacks the primitives Claude Code provides.

### 10.1 The rules file

`.cursor/rules/startup.mdc`:

```markdown
---
description: Startup workflow engine for hackathon brainstorming and planning. Always load when working in a project that has a .startup/ directory.
globs:
  - ".startup/**"
  - ".cursor/**"
alwaysApply: true
---

# Startup CLI Workflow

This project uses the Startup workflow engine. State, procedures, prompts, and tooling all live in `.startup/`.

## On every session start

Before responding to anything else, run:

```bash
node .startup/bin/startup-tools.js status
```

Read the output to understand the project's current phase, populated sections, and pending items. Use this to ground your suggestions.

## Available commands

Startup commands are not slash commands in Cursor — they are procedure files you execute. To run any command, **first resolve the active project's mode**, then locate the procedure under the matching subtree.

Resolve the active mode:

```bash
node .startup/bin/startup-tools.js mode
```

Resolve a specific procedure file path for the active project's mode:

```bash
node .startup/bin/startup-tools.js procedure-path <prompt_id>
```

The helper looks under `.startup/procedures/<mode>/` first, then `.startup/procedures/shared/`. If it exits nonzero, the prompt is legitimately skipped in the active mode — do not invoke it.

Common procedures (paths shown for standard mode; in fast mode the equivalents live under `procedures/fast/` and the set is smaller):

- New project: `.startup/procedures/shared/new-project.md` (or use `node .startup/bin/startup-tools.js init --mode <standard|fast>`)
- Status check: just run `node .startup/bin/startup-tools.js status`
- Phase 2 Competitive Pre-mortem: `.startup/procedures/standard/phase-2/competitive-premortem.md` (same file referenced in fast mode under `procedures/fast/phase-2/competitive-premortem.md`)
- Intake (process pasted external-LLM output): `.startup/procedures/shared/intake.md`
- Phase advance: `.startup/procedures/shared/advance-phase.md`

The full procedure catalog is in `.startup/STARTUP.md`.

## How to operate

1. The user requests an action ("run the premortem", "intake the latest output", etc.).
2. You identify the matching procedure file.
3. You read and follow the procedure exactly. Do not paraphrase or improvise.
4. Use `node .startup/bin/startup-tools.js <subcommand>` for all state operations.
5. Prompt content lives in `.startup/PROMPTS.md`. Load by section heading. Do not rewrite.

## Differences from Claude Code

Cursor lacks the subagent and hook primitives Claude Code has. This means:

- **Intake parsing runs in your main context.** When the user asks you to intake an inbox file, you do the parsing yourself rather than spawning an isolated subagent. The intake procedure (`.startup/procedures/shared/intake.md`) accounts for this — follow it.
- **No SessionStart banner.** You need to manually run `startup-tools status` at the start of each session.
- **No auto-resume.** If a session is compacted, the user must manually request a status check on the next session.
- **Grill-Me mode runs in main context.** Same as intake — no subagent isolation. Be more disciplined about not letting the Grill-Me dialogue pollute follow-up tasks.

## Constraints

- Never edit any project's `pcd.json` directly. Use `startup-tools apply-diff`. The active project's PCD lives at `.startup/projects/<active-ulid>/pcd.json`; the helper resolves the path so you don't have to.
- Never paraphrase prompts from `PROMPTS.md`. Copy verbatim.
- The Phase 4 gate is one-way (per-project). After crossing it for a project, scope changes for that project require the Addendum Protocol. The gate ceremony is identical in both modes.
- The Competitive Pre-mortem is the only hard-blocked prompt. Phase 2 cannot advance without it. The helper enforces this — do not override. Enforced in both standard and fast mode.
- The active project's mode (`standard` or `fast`) determines which procedure files are valid to execute. Always resolve procedure paths via `startup-tools procedure-path <prompt_id>`. If the helper says a prompt is skipped in the active mode, do not try to execute it.
- Workspace-shared assets (`.startup/PROMPTS.md`, `.startup/procedures/`, `.startup/vault/`, etc.) are visible to every project. Edits to them affect future and current projects alike.
```

### 10.2 What's degraded in Cursor

| Capability | Claude Code | Cursor |
|---|---|---|
| Slash command invocation | Yes (`/startup:foo`) | No — user references procedures by path or name |
| Subagent isolation | Yes — intake and Grill-Me run in clean context | No — main context absorbs intake parsing |
| SessionStart banner | Automatic | Manual `startup-tools status` |
| Auto-resume after compaction | Yes via PreCompact + SessionStart hooks | None — user re-orients manually |
| Skill auto-invocation | Yes — skill descriptions trigger relevance matching | None — user explicitly names the procedure |
| Phase-gate hard-block enforcement | Helper-enforced (works in both) | Helper-enforced (works in both) |

The bottom row is the point: every primitive that matters for correctness — schema validation, gate enforcement, atomic writes, staleness propagation — lives in `startup-tools` and works identically in both adapters. The features Cursor loses are ergonomic, not foundational.

---

## 11. Skill Catalog

The skill catalog is the user-facing surface of the Claude Code adapter. There is one skill per prompt in `PROMPTS.md`, plus a handful of utility skills for state operations.

### 11.1 Prompt-driven skills (one per prompt in PROMPTS.md)

Each of these maps 1:1 to a prompt and to a procedure file. **Skill names are mode-agnostic.** Each skill calls `startup-tools procedure-path <prompt_id>` to resolve the correct procedure file for the active project's mode. In fast mode, several prompt-driven skills are valid skills to *have registered* but will exit with a "this prompt is skipped in fast mode" message if invoked (because the corresponding procedure file does not exist in `procedures/fast/`). This is deliberate — the skill catalog stays stable across modes; the dispatch layer enforces the mode-specific subset.

| Skill | Procedure | Phase |
|---|---|---|
| `/startup:phase-1-judge-research` | `phase-1/judge-research.md` | 1 |
| `/startup:phase-2-problem-analysis` | `phase-2/problem-analysis.md` | 2 |
| `/startup:phase-2-premortem` | `phase-2/competitive-premortem.md` | 2 |
| `/startup:phase-2-research-hypotheses` | `phase-2/research-hypotheses.md` | 2 |
| `/startup:phase-2-user-research-package` | `phase-2/user-research-package.md` | 2 |
| `/startup:phase-3-seed-challenge` | `phase-3/idea-seed-challenge.md` | 3 |
| `/startup:phase-3-scamper` | `phase-3/scamper.md` | 3 |
| `/startup:phase-3-jtbd` | `phase-3/jtbd.md` | 3 |
| `/startup:phase-3-brainwriting` | `phase-3/brainwriting.md` | 3 |
| `/startup:phase-3-divergent-candidates` | `phase-3/divergent-candidates.md` | 3 |
| `/startup:phase-3-stress-test` | `phase-3/differentiation-stress-test.md` | 3 |
| `/startup:phase-3-downselection` | `phase-3/downselection.md` | 3 |
| `/startup:phase-4-design-grill` | `phase-4/design-grill.md` | 4 |
| `/startup:phase-4-product-brief` | `phase-4/product-brief.md` | 4 |
| `/startup:phase-4-tid` | `phase-4/tid.md` | 4 |
| `/startup:phase-4-demo-script` | `phase-4/demo-script.md` | 4 |
| `/startup:phase-6-pitch-brief` | `phase-6/pitch-brief.md` | 6 |
| `/startup:phase-6-pitch-review` | `phase-6/pitch-review.md` | 6 |
| `/startup:phase-6-qa-prep` | `phase-6/qa-prep.md` | 6 |
| `/startup:phase-6-judge-drill` | `phase-6/judge-qa-drill.md` | 6 |
| `/startup:phase-7-debrief` | `phase-7/debrief.md` | 7 |
| `/startup:addendum` | `addendum/addendum-protocol.md` | post-4 |

### 11.2 Utility skills

| Skill | Purpose |
|---|---|
| `/startup:new-project` | Scaffold a new project in the workspace. Runs `startup-tools init` (accepts `--fast` to declare fast mode at creation; defaults to standard). Sets the new project as active, then walks the user through Phase 0 setup. |
| `/startup:set-mode` | Change the active project's mode (`standard` ↔ `fast`). Refuses if the Phase 4 gate has been crossed. Surfaces newly-gap-or-orphan sections so the user sees what changed in advancement requirements. |
| `/startup:list` | List all projects in the workspace with current phase, last activity, and active marker. |
| `/startup:switch` | Switch the active project. Runs `startup-tools switch <ulid_or_slug>`. Status banner refreshes on next session-start (or `/startup:status` now). |
| `/startup:archive` | Archive the active project (or a specified one). Moves it to `_archived/` and clears `active.txt` if it was active. |
| `/startup:status` | Print the full status of the active project. Useful mid-session when bearings get lost. |
| `/startup:intake` | Process the latest (or specified) inbox file in the active project. Spawns intake-parser subagent. Surfaces diff for approval. |
| `/startup:advance-phase` | Attempt to advance current_phase. Prints precondition status. Handles Phase 4 gate ceremony. |
| `/startup:back-phase` | Revert to an earlier phase. Surfaces newly stale sections. |
| `/startup:cross-gate` | Cross the Phase 4 gate. Runs the typed-acknowledgment ceremony. |
| `/startup:vault-add` | Add a new entry to the workspace Vault (judges, organizers, etc.). Affects all projects in the workspace. |
| `/startup:export-pcd` | Render the full PCD as markdown to a specified file. For Cursor handoff at Phase 4→5 — also writes the resulting file to a target build repo if `--target` is passed. |
| `/startup:rubric-update` | Open the active project's rubric for editing (used when official criteria are released). Walks through Replace / Supplement / Keep-default. |

Total: 22 prompt-driven skills + 13 utility skills = **35 skills** at MVP.

### 11.3 Skill auto-invocation

Each skill's `description` frontmatter field is the auto-invocation signal. Examples:

- `/startup:phase-2-premortem` — "Use when the user mentions premortem, crowded space, predicted competitor solutions, or asks 'what would other teams build for this brief'."
- `/startup:intake` — "Use when the user says they've pasted output back, finished running a prompt externally, or asks to commit external-LLM results to the project."

Auto-invocation reduces the cognitive load of remembering exact command names. The user can talk naturally and Claude routes.

---

## 12. Build Order

The build order is phased to deliver a usable system at each milestone. A solo developer working in Cursor should be able to ship M1 in a focused weekend.

### M0 — Foundation (1–2 days)

Goal: a workspace can be scaffolded, a project initialized, status checked, and the PCD validated. No prompts work yet. Mode-aware infrastructure is scaffolded but not yet exercised.

Deliverables:
- `.startup/STARTUP.md` (entry document, including workspace + active-project explanation, including a brief mention that procedures are mode-routed and mode is set at project init)
- `.startup/bin/startup-tools.js` with `workspace-init`, `init` (with `--mode` flag accepted and stored, default `standard`), `list`, `switch`, `status`, `banner`, `mode`, `procedure-path` (returns empty for now since no procedure files exist yet, but the resolver logic is implemented and tested with a stub), `validate`, `validate-workspace`, `render-context`, `render-pcd`, `apply-diff`, `gate-log`, `inbox-path`
- `.startup/pcd_schema.json` (copied from project knowledge; ensure `hackathon_profile.data.mode` is recognized — if not present in the current schema, add as an optional enum `"standard" | "fast"` defaulting to `"standard"`)
- `.startup/PROMPTS.md` (copied; mark in a TODO that `phase_3_divergent_candidates` will need a `{fast_mode}` placeholder added in M3.5)
- `.startup/default_judging_rubric.md` (copied)
- Empty `.startup/procedures/standard/`, `.startup/procedures/fast/`, `.startup/procedures/shared/` directories scaffolded
- Workspace + first-project directory scaffolding (including `active.txt`, `projects/`, and the empty `vault/`)

Test: run `node startup-tools.js workspace-init`, then `init --name "Test Hackathon"`, verify `active.txt` was created and points at the new project, inspect `projects/<ulid>/pcd.json`, verify `mode` is set to `"standard"`. Run `init --name "Fast Test" --mode fast --no-activate`, run `list`, verify both projects appear with the active marker and the second shows mode `fast`. Run `switch <ulid>` to flip active, run `status`, verify the second project's state is now shown including its mode. Run `mode`, verify it prints `fast`.

### M1 — End-to-end on one prompt (2–3 days)

Goal: one prompt can be run end-to-end in standard mode. Pick `phase_2_competitive_premortem` since it's the most important and best-documented. Fast mode is not exercised in this milestone.

Deliverables:
- `.startup/procedures/standard/phase-2/competitive-premortem.md` (full procedure)
- `.startup/intake/intake_phase_2_competitive_premortem.json`
- `.claude/skills/startup-phase-2-premortem/SKILL.md` (uses `startup-tools procedure-path` to resolve its procedure)
- `.claude/agents/intake-parser.md`
- `.claude/settings.json` with both hooks
- Helper subcommands needed: `precondition`, `advance-phase`, `back-phase`, `inbox-list`, `inbox-archive`, `handoff`, `resume`
- One utility skill: `/startup:intake`

Test: from a populated Phase 1 PCD (standard mode), run `/startup:phase-2-premortem`, get a prompt assembled and executed by Claude, output lands in inbox, run `/startup:intake`, subagent extracts, user approves, diff applied, premortem_completed becomes true, Phase 2 → 3 advancement is unblocked. Verify `procedure-path phase_2_competitive_premortem` returns `procedures/standard/phase-2/competitive-premortem.md`.

### M2 — Complete Phase 2 + Phase 3 (standard mode) (4–6 days)

Goal: full coverage of the two ideation-heavy phases in **standard mode**. Tests the procedure pattern at scale. Fast mode procedures are added in M3.5.

Deliverables:
- All Phase 2 and Phase 3 procedures in `procedures/standard/`, intake definitions, and skills.
- The Grill-Me subagent (used by Differentiation Stress Test).
- Utility skills: `/startup:status`, `/startup:advance-phase`, `/startup:back-phase`, `/startup:new-project` (without `--fast` flag yet), `/startup:list`, `/startup:switch`, `/startup:vault-add`.

Test: run a full Phase 1 → Phase 3 workflow on a fake hackathon brief in standard mode. Verify staleness propagation when reverting Phase 2 → 1.

### M3 — Phase 4 gate + Phase 5 handoff (standard mode) (3–5 days)

Goal: produce the TID and cross the gate in **standard mode**. This is the most important deliverable since the TID is what feeds Cursor at build time.

Deliverables:
- All Phase 4 procedures (Design Grill, Product Brief, TID, Demo Script) in `procedures/standard/phase-4/`.
- `/startup:cross-gate` with the typed-acknowledgment ceremony (mode-agnostic — same ceremony for both modes).
- `/startup:export-pcd` for handing off to the build phase.
- The Addendum Protocol procedure in `procedures/shared/addendum/` and `/startup:addendum`.
- The Cursor adapter (`.cursor/rules/startup.mdc`) with mode-awareness noted in its operating instructions.

Test: run a project from new-project through Phase 4 gate crossing in standard mode. Export the PCD. Use the exported TID as input to a separate Cursor session, verify the agent understands it.

### M3.5 — Fast Mode (2–3 days)

Goal: deliver the same-day variant. Adds the fast procedure tree for Phases 1–4, the `{fast_mode}` placeholder, and the mode-switching helper commands. Phase 6 fast variant is deferred to M4.

Deliverables:
- `procedures/fast/phase-1/judge-research.md` — one-shot Claude (not Deep Research), Vault-heavy, lower-confidence personas accepted.
- `procedures/fast/phase-2/problem-analysis.md` — compressed (relaxed minimums on hmw_reframes; same prompt definition, different optional-context posture).
- `procedures/fast/phase-2/competitive-premortem.md` — same prompt as standard; procedure file may differ only in surrounding choreography (e.g., suggests running in parallel with problem-analysis in two LLM tabs).
- `procedures/fast/phase-3/idea-seed-challenge.md` — identical to standard; consider whether to move this to `procedures/shared/phase-3/idea-seed-challenge.md` if it's truly identical (apply the 75%-identical refactor rule from §4.3.1).
- `procedures/fast/phase-3/divergent-candidates.md` — invokes `phase_3_divergent_candidates` with `{fast_mode: true}` resolved at assembly time. Context injection drops scamper_outputs / jtbd_analysis / brainwriting_round; adds explicit instruction in task spec that synthesis comes from problem_analysis + competitive_landscape (+ team_idea_seed if present).
- `procedures/fast/phase-4/product-brief.md`, `procedures/fast/phase-4/tid.md`, `procedures/fast/phase-4/demo-script.md` — same prompts as standard; procedure files choreograph parallel execution and explicitly tell the user "open three LLM tabs, run all three assembled prompts simultaneously, intake them as they come back in any order." No Design Grill step.
- `phase_3_divergent_candidates` prompt updated in `PROMPTS.md` with the `{fast_mode}` runtime placeholder and corresponding task-spec branch.
- Helper subcommand additions: `mode`, `procedure-path`, `set-mode`. `init` learns `--mode` flag.
- Phase 3 → 4 advance-phase precondition becomes mode-aware (drops scamper/jtbd/brainwriting requirement in fast mode).
- `/startup:new-project` learns `--fast` flag.
- New utility skill: `/startup:set-mode`.
- `phase_4_gate_crossed` event handler in `set-mode` refuses with exit code 2.

Test: run `init --mode fast --name "Same-Day Test"`. Walk a full Phase 1 → Phase 4 gate crossing in fast mode. Verify wall-clock target (30–45 minutes excluding LLM execution time). Verify the gate ceremony is identical to standard mode. Verify that attempting `/startup:phase-3-scamper` in a fast-mode project returns a clean "this prompt is skipped in fast mode" error rather than crashing. Verify that `set-mode standard` mid-Phase-2 surfaces newly-gap sections (scamper, jtbd, brainwriting, research_hypotheses) and that `set-mode fast` mid-Phase-2 of a standard project preserves their data but no longer requires them for advancement.

### M4 — Phase 6 + Phase 7 + Polish (3–5 days)

Goal: close the loop. Pitch prep and post-event debrief, both modes.

Deliverables:
- All Phase 6 procedures in `procedures/standard/phase-6/` and skills.
- `procedures/fast/phase-6/pitch-brief.md` — truncated Phase 6 variant; review and Q&A prep skipped.
- Phase 7 debrief procedure at `procedures/shared/phase-7/debrief.md` (mode-agnostic) and skill, with the Phase 7 intake's Vault update proposals routed through the standard diff-approve flow into the workspace-shared `vault/`.
- `/startup:archive` for retiring completed projects after debrief.
- `/startup:rubric-update` for handling late criteria announcements.
- Hardening: input validation on all helper subcommands, better error messages, basic logging.

### M5 — Real hackathon test

Use the system in an actual hackathon. Track every friction point. Use the Phase 7 debrief to identify which procedures need refinement. The system is the system, but the procedures and prompts will keep improving across events.

### Deferred (stretch)

- Plugin packaging (`.claude-plugin/plugin.json`) for one-command install across machines.
- MCP server exposing PCD reads/writes for non-CLI integrations.
- Team collaboration mode (shared PCD via git, conflict resolution).
- Autonomous deep research (replacing the paste-back pattern for Phase 1 research).
- Analytics layer (win-rate tracking across events).
- Client engagement mode (workflow adaptation for paid client work).

---

## Appendix A — Glossary

- **Workspace** — A single Startup install hosting many hackathon projects. Lives at any directory the user chooses; contains `.startup/`, optionally `.claude/` and `.cursor/`. One workspace = one shared Vault, one set of procedures and prompts, many projects.
- **Active project** — The project all helper commands operate on by default. Identified by the ULID in `.startup/active.txt`. Changed via `startup-tools switch` or `/startup:switch`.
- **PCD** — Project Context Document. The canonical state of one project. `<active-project>/pcd.json`.
- **Procedure** — A markdown file describing how to execute one workflow step. Lives in `.startup/procedures/` (workspace-shared).
- **Skill** — Claude Code's primitive for slash-command-invokable instructions. Wraps a procedure.
- **Subagent** — A separate Claude instance with its own context. Used for intake parsing and Grill-Me.
- **Hook** — A Claude Code lifecycle event handler. Used for SessionStart banner and PreCompact handoff.
- **Inbox** — `<active-project>/inbox/`. Where external-LLM outputs land before intake. Per-project.
- **Vault** — Library of judges, organizers, winning solutions, boilerplate. Lives at `.startup/vault/`, shared across all projects in the workspace.
- **Phase Gate Log** — `<active-project>/phase_gate_log.jsonl`. Per-project append-only record of significant events.
- **Phase 4 gate** — The one-way scope-lock transition between Phase 4 and Phase 5. After crossing, scope changes require the Addendum Protocol. Per-project.
- **Addendum Protocol** — The structured, friction-heavy process for accepting post-gate scope changes.
- **Intake** — The process of parsing pasted external-LLM output into structured PCD updates.
- **Grill-Me** — Adversarial Socratic interrogation mode for stress-testing reasoning. Available for select prompts.
- **`startup-tools`** — The Node.js helper at `.startup/bin/startup-tools.js`. Handles all state mutations. Operates on the active project unless `--project <ulid>` is passed.
- **Mode** — A per-project setting (`"standard"` or `"fast"`) declared at project init and stored at `sections.hackathon_profile.data.mode`. Determines which procedure subtree the helper dispatches to. Switchable mid-event up to (and including) Phase 4; locked after the Phase 4 gate is crossed.
- **Standard mode** — The default. Full diligence flow: Phase 1 Deep Research, Phase 2 hypotheses and user research, Phase 3 SCAMPER + JTBD + Brainwriting + divergent candidates + stress test, Phase 4 with optional Design Grill, Phase 6 with pitch review and Q&A prep. Targets hackathons with multi-day build windows.
- **Fast mode** — The compressed variant. Phase 1 one-shot research, Phase 2 problem analysis + competitive pre-mortem only, Phase 3 divergent candidates only (synthesized directly from problem + competitive landscape via the `{fast_mode}` placeholder), Phase 4 three documents in parallel, Phase 6 pitch brief only. Preserves the competitive pre-mortem hard block and the Phase 4 gate ceremony. Targets same-day hackathons (build window under one day). Wall-clock target for Phases 1–4: 30–45 minutes excluding LLM execution time.

---

## Appendix B — Inheritance from companion documents

This spec inherits, by reference, the full contents of:

- **`PROMPTS.md`** — every prompt's role assignment, task specification, output template, negative constraints, and design rationale. Procedures load prompt content from this file by section heading. Total: 22 prompts.
- **`pcd_schema.json`** — the canonical PCD schema (1402 lines, JSON Schema draft 2020-12). Helper validates against this. Subagents extract conforming to fragments of this.
- **`default_judging_rubric.md`** — the five-criterion default rubric (Innovation 0.20, Problem-Solution Fit 0.25, Technical Soundness 0.20, Feasibility 0.15, Presentation 0.20). Loaded into `active_judging_rubric.data.criteria` at project init.

When any of the above changes, the corresponding procedure / intake / skill should be reviewed for drift. The system is designed so that schema-level changes ripple through validation automatically, but prompt-level changes require manual procedure review (this is the cost of having the prompt content human-authored rather than schema-derived).

The v1.2 addition of the `{fast_mode}` runtime placeholder on `phase_3_divergent_candidates` is one such manual review point: edits to that prompt's task specification or negative constraints must be reviewed against both placeholder branches.

---

## Appendix C — Mode Architecture Reference

This appendix consolidates the fast-mode design rationale and per-phase deltas in one place. It is reference material for anyone reading the spec cold or extending mode-aware behavior in future versions.

### C.1 Why fast mode exists

Two hackathon shapes dominate the user's calendar: multi-day events with weeks of build time after the brief drops, and same-day events with under a day of total build time. The standard flow (Phases 1–4 pre-build) takes 3.5–7 hours of focused work end-to-end, which is rational when the build window is weeks but catastrophic when the build window is six to ten hours total. Skipping the workflow entirely in same-day events forfeits the competitive edge the workflow exists to produce — most painfully, the competitive pre-mortem (the only hard block in the workflow, and the single most reliable predictor of placement in past hackathons).

Fast mode is the compromise. It preserves the load-bearing pieces and cuts the diligence layers that compound their value over days. Wall-clock target for Phases 1–4: 30–45 minutes excluding LLM execution time.

### C.2 What's preserved and what's cut

**Preserved in fast mode (load-bearing):**

- The competitive pre-mortem (Phase 2). Hard block in both modes. Same-day teams converge faster, not slower — the pre-mortem is *more* important under time pressure, not less.
- The divergent candidates synthesis (Phase 3). Produces the candidate portfolio that the team chooses among. In fast mode, this is the only Phase 3 prompt; it synthesizes directly from problem analysis + competitive landscape rather than from SCAMPER/JTBD/Brainwriting.
- All three Phase 4 documents (Product Brief, TID, Demo Script). The TID prevents AI-coded spaghetti; the Demo Script prevents the "judges thought it was a Figma prototype" failure mode. Both non-negotiable.
- The Phase 4 gate ceremony. Identical typed-acknowledgment, identical one-way semantics.
- The Phase 7 debrief. Mode-agnostic; same procedure file.

**Cut or deferred in fast mode (diligence that compounds over days):**

- Phase 1 Deep Research → one-shot Claude with Vault matches doing more of the work.
- Phase 2 research hypotheses → skipped entirely. Hypotheses earn their keep when there's time to validate them; in a same-day window there isn't.
- Phase 2 user research package → skipped. Real-user research is impossible in the time window.
- Phase 3 SCAMPER, JTBD, Brainwriting → skipped. The divergent net is the cost; the candidate portfolio is what matters and can be synthesized from problem analysis + competitive landscape.
- Phase 3 differentiation stress test → skipped. Nice-to-have, not load-bearing.
- Phase 4 Design Grill → skipped. Adversarial scope review is high-value but high-friction; fast mode trades it for direct execution.
- Phase 6 pitch review, Q&A prep, judge drill → deferred or skipped. Pitch brief alone is the minimum viable Phase 6 output.

### C.3 Per-phase deltas

| Phase | Standard | Fast |
|---|---|---|
| 0 | Vault refresh, profile capture | Identical |
| 1 | Deep Research for judge personas | One-shot Claude, Vault-heavy, lower-confidence personas accepted |
| 2 | problem_analysis + competitive_premortem + research_hypotheses + user_research_package (sequential) | problem_analysis + competitive_premortem only; procedure suggests running in parallel in two LLM tabs |
| 3 | 3a optional → SCAMPER → JTBD → Brainwriting → Divergent Candidates → optional stress test | 3a optional → Divergent Candidates only; prompt receives `{fast_mode: true}` so it synthesizes from problem analysis + competitive landscape |
| 4 | optional Design Grill → Product Brief → TID → Demo Script (sequential) → gate ceremony | Three documents generated and intaken in parallel → gate ceremony (no Design Grill) |
| 5 | Build | Identical (much shorter time horizon) |
| 6 | Pitch Brief → Pitch Review → Q&A Prep → Judge Drill (parallel with build) | Pitch Brief only, deferred until build is functionally complete |
| 7 | Debrief | Identical (mode-agnostic procedure in `procedures/shared/`) |

### C.4 What the helper does differently in fast mode

The mode-aware surface in `startup-tools` is intentionally narrow:

- `init --mode <standard|fast>` accepts the flag and stores it on the new project.
- `procedure-path <prompt_id>` checks `procedures/<mode>/...` first, then `procedures/shared/...`. Exits nonzero if no procedure exists for the prompt in the active mode.
- `precondition phase_3_divergent_candidates` — passes in fast mode even if no scamper/jtbd/brainwriting outputs exist; in standard mode requires at least one.
- `advance-phase` Phase 3 → 4 — drops the ideation prerequisite in fast mode.
- `set-mode` — switches mode mid-event up to (and including) Phase 4; refuses after the Phase 4 gate has been crossed.

Everything else — `apply-diff`, schema validation, staleness propagation, the gate ceremony, the intake pipeline — is identical across modes.

### C.5 What prompts change

Of the 22 prompts in `PROMPTS.md`, **one** has a mode-aware task specification:

- `phase_3_divergent_candidates` — uses `{fast_mode}` placeholder. In standard mode (`{fast_mode}: false`), instructs the LLM to synthesize from SCAMPER + JTBD + Brainwriting outputs in context. In fast mode (`{fast_mode}: true`), instructs the LLM to synthesize directly from problem_analysis + competitive_landscape + (if present) team_idea_seed. Archetype discipline and the `spaces_to_avoid` disqualification rule are identical in both branches.

All other prompts run identically when invoked, or are skipped entirely (the fast-mode procedure file simply does not exist for skipped prompts).

### C.6 Drift risk and mitigation

The main risk this architecture introduces is drift between standard and fast procedure files that share most of their content. The mitigation is the `procedures/shared/` directory: any procedure that is identical (or near-identical) across modes belongs there, not duplicated into both subtrees.

The authoring discipline: if a phase's procedure files are more than 75% identical between the two modes, refactor the common content into a single file at `procedures/shared/` and have the mode-specific files reference it (or move the whole file there if there's nothing mode-specific left).

The audit discipline: when editing a procedure in `procedures/standard/`, check whether the sibling in `procedures/fast/` needs the same edit. Add this check to the Phase 7 debrief checklist after any hackathon where workflow refinement is proposed.

---

*End of Startup CLI Implementation Specification v1.2.*
