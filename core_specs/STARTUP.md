# STARTUP.md — Workspace Entry Document

*This is the contract between Startup's agnostic core and any AI coding agent (Claude Code, Cursor, or future equivalent) operating the system.*

*If you are an agent reading this for the first time in a session: read it in order, top to bottom, once. Then jump to Section 2 and run the orientation command before doing anything else.*

---

## 1. What this is

**Startup** is a CLI-based workflow engine for hackathon ideation, planning, and post-event learning. It is not a web app, not a SaaS, not a server. It is a directory of markdown procedures, JSON state, and a single Node.js helper, all driven by you — the AI coding agent.

The system has one job: take a hackathon brief and walk the user through eight phases of work (intelligence gathering, problem deconstruction, differentiated ideation, scope-locked solution design, build, pitch prep, post-event debrief) using a fixed library of prompts and a typed state model.

Three concepts you must hold from the start:

- **The PCD** (Project Context Document) is the canonical state of one hackathon project. One JSON file. Schema-validated. Lives at `<active-project>/pcd.json`. Every prompt reads from it, and every intake writes to it through the helper.
- **Phases** are a directed graph with forward edges, backward edges (Phases 0–4), and one absorbing one-way gate at the end of Phase 4. They constrain what can be done when.
- **Your role.** You assemble prompts, execute them (or hand them out to the user for external execution), parse outputs into structured updates, and surface diffs for user approval. You do not invent procedure. You do not edit state directly. The Node helper (`startup-tools`) handles all file mutation.

You are the reasoning layer. The helper is the state layer. The procedures and prompts are the choreography. Keep those roles clean and the system works.

---

## 2. How to find your bearings

**Every session, before any other action, run:**

```bash
node .startup/bin/startup-tools.js status
```

The output tells you: which project is active, what phase it's in, which PCD sections are populated, which are stale, which inbox files are pending intake, and what the most recent activity was. This is the only call you make without consulting a procedure first.

If the output says no active project is set, the workspace is fresh or the user just archived the last project. Suggest one of:

- `node .startup/bin/startup-tools.js list` — see what projects exist in the workspace.
- `/startup:new-project` (Claude Code) or follow `.startup/procedures/phase-0/new-project.md` (Cursor) — scaffold a new project.
- `/startup:switch <ulid_or_slug>` (Claude Code) or `node .startup/bin/startup-tools.js switch <ulid_or_slug>` (any agent) — activate an existing project.

If the helper exits with a nonzero code on any call, **stop**. Read the stderr message. Surface it to the user. Do not improvise around helper failures — they exist to prevent data corruption.

If a recent handoff file exists (`<active-project>/handoff.json` written less than 7 days ago), the status banner will surface its `resumption_hint`. Honor it: that's where the previous session expected this one to pick up.

---

## 3. The workspace model

**This is the single most important concept to internalize.** Get this wrong and every subsequent operation lands on the wrong project.

### One workspace, many projects

A Startup workspace is a directory (any location the user chose) containing `.startup/` and optionally `.claude/` and `.cursor/`. **One workspace hosts many hackathon projects side by side.** The user opens the workspace once in their agent and stays there across hackathons.

The layout:

```
<workspace-root>/
├── .startup/
│   ├── STARTUP.md                  ← you are reading this
│   ├── active.txt                  ← one-line file: ULID of the active project
│   ├── pcd_schema.json             ← workspace-shared
│   ├── PROMPTS.md                  ← workspace-shared
│   ├── default_judging_rubric.md   ← workspace-shared
│   ├── procedures/                 ← workspace-shared (one file per prompt)
│   ├── intake/                     ← workspace-shared (one JSON per intake)
│   ├── vault/                      ← workspace-shared (judges, organizers, etc.)
│   ├── bin/startup-tools.js        ← workspace-shared helper
│   └── projects/
│       ├── 01HQ...-dost-hackathon/
│       │   ├── pcd.json
│       │   ├── phase_gate_log.jsonl
│       │   ├── handoff.json
│       │   └── inbox/
│       ├── 01HR...-shopee-hackathon/
│       │   └── ...
│       └── _archived/
│           └── 01HK...-old-hackathon/
├── .claude/   (optional adapter)
└── .cursor/   (optional adapter)
```

**Workspace-shared assets** (procedures, prompts, schema, helper, Vault) live at `.startup/` root. They are read by every project. Edits to them apply to every project in the workspace, immediately.

**Per-project state** (PCD, log, inbox, handoff) lives under `.startup/projects/<ulid>-<slug>/`. Each project's state is isolated from the others.

### The active project pointer

The file `.startup/active.txt` contains exactly one line: the ULID of the currently active project. **You never read this file directly.** The helper resolves it on every invocation.

This means: when a procedure tells you to operate on `<active-project>/pcd.json` or `<active-project>/inbox/`, you do not compute the path. You call the helper. The helper substitutes the right project automatically.

- `startup-tools inbox-path` → prints the absolute path to the active project's inbox.
- `startup-tools status` → prints the active project's full state.
- `startup-tools apply-diff <file>` → applies a diff to the active project's PCD.

If the user needs to target a different project for one command, they (or you, on their instruction) pass `--project <ulid_or_slug>`. To change the active project persistently, run `startup-tools switch <ulid_or_slug>`.

**You never need to know the active project's ULID.** The helper handles it. If you find yourself trying to compute a project path manually, stop — there is a helper command for it.

### Path conventions in this document and in procedures

Two forms are used:

- **Workspace-relative paths** start with `.startup/...`. They refer to workspace-shared assets that exist independent of any project. Example: `.startup/PROMPTS.md`, `.startup/procedures/phase-2/competitive-premortem.md`.
- **Active-project paths** are written as `<active-project>/...`. They refer to files inside the currently active project's directory. The helper resolves `<active-project>` at runtime. Example: `<active-project>/pcd.json`, `<active-project>/inbox/`.

Procedures use these conventions throughout. Do not substitute literal paths.

---

## 4. The phase model

Startup uses an eight-phase model (Phases 0–7). Each phase has named deliverables and procedure files at `.startup/procedures/phase-<N>/`.

| Phase | Name                         | One-line summary                                                                                       |
| ----- | ---------------------------- | ------------------------------------------------------------------------------------------------------ |
| 0     | Setup & Vault Refresh        | Capture the hackathon profile; activate the default rubric; surface relevant Vault entries.            |
| 1     | Intelligence Gathering       | Research judges, organizers, and past-winner patterns to anchor downstream decisions.                  |
| 2     | Problem Deconstruction       | Decompose the brief, run the Competitive Pre-mortem, frame research hypotheses, package user research. |
| 3     | Differentiated Ideation      | SCAMPER, JTBD, brainwriting, divergent candidates, stress test, downselection.                         |
| 4     | Solution Design & Scope Lock | Design Grill, Product Brief, TID, Demo Script. **This is the gate phase.**                             |
| 5     | Build                        | Code. Handed off to Cursor or Claude Code in build mode. Startup mostly steps aside here.              |
| 6     | Pitch & Presentation         | Pitch brief, pitch review, Q&A prep, judge drill.                                                      |
| 7     | Post-Hackathon Debrief       | Debrief; Vault updates proposed and applied.                                                           |

The full procedure index by phase lives at `.startup/procedures/<phase>/`. Each phase directory contains one markdown file per prompt that runs in that phase. See Section 10 for the corresponding skill names.

Forward and backward transitions, plus the Phase 4 gate, are described in Section 9.

---

## 5. How procedures work

A **procedure** is a markdown file at `.startup/procedures/<phase>/<name>.md`. There is one procedure per prompt in `PROMPTS.md`. The procedure tells you, step by step, exactly how to execute that prompt.

### Procedure structure

Every procedure has the same shape:

```markdown
# <Phase N — Step Name>

Maps to PROMPTS.md prompt: <phase_X_name>
Produces intake: <intake_X_name>

## Preconditions

[what must be true before running]

## Steps

**Step 1: Verify preconditions**
[shell command]
**Step 2: Load required context**
[shell command]
**Step 3: Load optional context**
[shell command + Vault]
**Step 4: Assemble the prompt**
[the five-component structure — see Section 6]
**Step 5: Execute or copy**
[inline execution by default; --copy-prompt writes for paste-out]
**Step 6: Hint at next step**
[tell user what to run next, typically /startup:intake]

## Notes

[prompt-specific gotchas, common LLM failure modes, hard blocks]
```

### How to execute a procedure

1. **Read it.** Open the procedure file the skill (or user request) points at.
2. **Follow the steps in order.** Each step is either a shell command (run it; check the exit code) or an action (do exactly what it says).
3. **Do not paraphrase the procedure's instructions.** If Step 2 says "Run `node .startup/bin/startup-tools.js render-context hackathon_profile judge_intelligence`", run that. Do not improvise a different set of sections because you think they might be more useful.
4. **Do not skip helper calls.** If a procedure tells you to verify preconditions before assembling a prompt, run the precondition check. The helper will block you if the project isn't ready — that's a feature.
5. **Stop on nonzero exit.** Any helper command exiting with a nonzero code is a hard stop. Read stderr. Surface the issue to the user. Suggest a remediation if one is obvious.
6. **Respect the procedure's Notes section.** Prompt-specific failure modes and hard blocks live there. They are not optional.

The procedure file is what makes the system agent-agnostic. Every step is either a portable shell command or a natural-language instruction a competent agent can follow. New agents are supported by writing a thin adapter; the procedures do not change.

---

## 6. How prompts work

Prompt **content** (role assignments, task specifications, output templates) lives in one place: `.startup/PROMPTS.md`. Prompts are referenced from procedures by section heading — typically `phase_X_name` (e.g., `phase_2_competitive_premortem`).

### The five-component assembly structure

Every prompt you assemble has this shape, in this order, every time:

1. **Format-Discipline Preamble** — a constant block defined at the top of `PROMPTS.md` under the section "The Format-Discipline Preamble." Copy verbatim.
2. **Role Assignment** — from PROMPTS.md under `prompt: <phase_X_name>` → `Role assignment` subsection. Copy verbatim.
3. **Context Injection** — rendered PCD sections (from `startup-tools render-context`) plus optional Vault matches (from `startup-tools vault-match`). Each rendered block is prefixed with `## CONTEXT — <Section_Name>`. Order: required sections first (in the order listed in the prompt's `Required context` field), then optional sections, then Vault matches.
4. **Task Specification** — from PROMPTS.md `Task specification` subsection. Resolve `{snake_case_token}` placeholders against the PCD (see token resolution below).
5. **Output Template** — from PROMPTS.md `Output template` subsection. If the corresponding intake schema fragment (`.startup/intake/<intake_id>.json` → `target_schema_fragment`) defines fields the illustrative template omits, append them. If the illustrative template includes fields the schema does not, drop them.

This order is invariant. Procedures do not deviate. Do not invent your own structure.

### Token resolution

Task specifications use `{snake_case_token}` placeholders. Resolve them by reading the relevant PCD section. The naming convention is straightforward:

- `{hackathon_name}` → `sections.hackathon_profile.data.hackathon_name`
- `{theme}` → `sections.hackathon_profile.data.theme`
- `{sponsor_list}` → comma-joined values of `sections.hackathon_profile.data.sponsors[]`

If a token does not resolve (the source field is absent or empty), **stop and surface the gap to the user.** Do not insert empty strings. Do not hallucinate values. The user will either populate the missing field or instruct you on how to proceed.

For tokens with non-obvious resolution paths, the prompt's `Required context` field in `PROMPTS.md` lists the source paths explicitly.

### The `--copy-prompt` flag

By default, you execute the assembled prompt yourself (inline) and save the response to `<active-project>/inbox/<prompt_id>-<timestamp>.md`. If the user invokes a skill with `--copy-prompt`, the procedure changes Step 5: instead of executing, you **write the assembled prompt** to `<active-project>/inbox/_pending-prompt-<prompt_id>.md` and tell the user to run it in an external LLM (Claude.ai, Gemini Pro, Deepseek, Perplexity) and save the response back to the inbox.

This preserves cross-LLM optionality without adding friction to the default path.

---

## 7. How state mutation works

**This rule is not negotiable. Internalize it before you touch any file.**

> **You never edit `pcd.json` directly. Ever. All PCD writes go through `startup-tools apply-diff`.**

The Node helper at `.startup/bin/startup-tools.js` is the only component that mutates project state. It owns atomicity, schema validation, staleness propagation, and event logging. If you write to `pcd.json` with a text editor or a shell redirect, you bypass all of those guarantees and break the system's invariants.

### What goes through the helper

| Operation                                 | Subcommand                             | Notes                                                                                                                                  |
| ----------------------------------------- | -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Read PCD sections (for context injection) | `render-context <section_ids...>`      | Returns markdown-rendered sections to stdout.                                                                                          |
| Read the full PCD (for export)            | `render-pcd`                           | Returns full markdown to stdout.                                                                                                       |
| Write PCD sections                        | `apply-diff <diff-file>`               | Validates against schema. Atomic temp-file rename. Updates `is_populated`, `last_updated`, `populated_by_phase`. Propagates staleness. |
| Validate the PCD                          | `validate`                             | Exits 0 if valid. Errors print with JSON paths.                                                                                        |
| Advance / revert phase                    | `advance-phase`, `back-phase <target>` | Checks preconditions; writes to phase_gate_log; propagates staleness on revert.                                                        |
| Cross Phase 4 gate                        | `cross-phase-4-gate`                   | Requires typed acknowledgment (see Section 9).                                                                                         |
| Append to phase gate log                  | `gate-log <event_type> [payload]`      | Append-only. Never modify existing log entries.                                                                                        |

### What you produce, what the helper consumes

Your role in a state mutation:

1. The intake subagent (or you, in Cursor's main context) **parses** the inbox file into a structured diff at `<active-project>/inbox/_diff-<intake_id>-<timestamp>.json`.
2. You **show the diff to the user** as a human-readable preview.
3. On user approval, **the helper applies it**: `node .startup/bin/startup-tools.js apply-diff <diff-file>`.

You never construct a PCD in memory and write it out. You produce diffs; the helper applies them.

### Atomicity, validation, staleness — all in the helper

- **Atomicity.** `apply-diff` writes to a `.tmp` file, validates, then renames over `pcd.json`. Failure leaves the original untouched.
- **Schema enforcement.** Every write is validated against `.startup/pcd_schema.json` before commit. Errors surface with JSON paths.
- **Staleness propagation.** When a section is updated, downstream sections (per a static dependency map inside the helper) get flagged `is_stale: true`. Reverting phases triggers the same logic. You do not need to track this — the helper handles it and reports newly stale sections in stdout.

### Manual edits are an escape hatch, not a path

The user can `vim pcd.json` if they choose to. Doing so bypasses validation and staleness propagation. The helper detects the change on its next invocation (by comparing mtime to the last log timestamp) and appends a `manual_pcd_edit` event to the log. This exists as a last resort. **Do not suggest it.** If a workflow needs a manual edit, that's a sign the procedure or intake definition is incomplete — surface it as such.

---

## 8. How intake works

**Intake** is the process of turning an inbox file (an external-LLM response, or your own response saved as a markdown file) into a validated PCD update.

The flow:

1. A procedure's Step 5 saved a response to `<active-project>/inbox/<prompt_id>-<timestamp>.md`. (Find the inbox path with `node .startup/bin/startup-tools.js inbox-path` — do not compute it.)
2. The user runs `/startup:intake <intake_id>` (Claude Code) or follows `.startup/procedures/intake.md` (Cursor).
3. The intake procedure spawns the **intake-parser subagent** (Claude Code) — or, in Cursor, runs the same parsing logic in the main context. The subagent receives: the inbox file path, the intake definition (`.startup/intake/<intake_id>.json`), and the inbox directory for diff output.
4. The subagent reads the inbox file, extracts JSON conforming to the intake's `target_schema_fragment`, runs `post_extract_validators`, computes a confidence score, and writes a proposed diff to `<active-project>/inbox/_diff-<intake_id>-<timestamp>.json`. It returns a brief summary (confidence tier, fields extracted, flags) — not the full extracted data.
5. The main session reads the diff file and **shows the user a human-readable preview** (markdown table or side-by-side).
6. The user responds with one of: `approve`, `edit`, `reject`, `rerun`.
   - `approve` → run `node .startup/bin/startup-tools.js apply-diff <diff-file>`.
   - `edit` → user edits the diff file directly, then says `approve`.
   - `reject` → archive the inbox file with `.failed` suffix.
   - `rerun` → delete the diff and re-spawn the subagent (e.g., with adjusted context).
7. On approve: helper applies the diff, moves the inbox file to `<active-project>/inbox/archive/`, appends `intake_committed` to the phase gate log, and surfaces any newly stale sections.

### Confidence tiers

Every extraction is graded:

- **Green (≥ 0.9)** — all required fields clean, no flags. Diff preview is brief; user can approve quickly.
- **Yellow (0.7 – 0.9)** — required fields present but one or more validators flagged warnings, or fields needed inference. Highlight flagged areas. User reviews carefully.
- **Red (< 0.7)** — required fields missing or source didn't match the expected shape. Subagent returns a diagnostic, not a commit-ready diff. Recommend re-running the producing procedure with adjusted context, or manual extraction.

The per-intake threshold lives in `.startup/intake/<intake_id>.json` → `confidence_threshold`.

### Never commit without user approval

The intake subagent **does not write to `pcd.json`.** It only writes to the diff file. The user approves; the helper applies. Skipping this step breaks the system's audit guarantee.

---

## 9. The Phase 4 gate

Phase 4 ends with a **one-way scope-lock gate.** It is the single most important state transition in the workflow. Read this section twice.

### What the gate is

The gate sits between Phase 4 (Solution Design & Scope Lock) and Phase 5 (Build). Phase 4 produces three deliverables: the Product Brief, the TID (Technical Implementation Document), and the Demo Script. Once those are populated, the user can attempt to cross.

The gate is:

- **One-way.** After crossing, `back-phase` to any phase ≤ 4 is refused by the helper. The only legitimate path to scope adjustment is the **Addendum Protocol** (see below).
- **Per-project.** Crossing the gate in one project has no effect on any other project in the workspace. Each project has its own `phase_4_gate_crossed` flag.
- **Advisorily enforced.** The CLI cannot physically prevent a determined user from editing `pcd.json` to flip the flag. Enforcement is via a typed acknowledgment ceremony plus log entries — friction, not lockout. This is acceptable because the user *is* the user and built this system to constrain themselves.

### The crossing ceremony

When the user runs `/startup:cross-gate` (or `node .startup/bin/startup-tools.js cross-phase-4-gate`):

1. The helper verifies Product Brief, TID, and Demo Script are populated.

2. The helper prints the scope-lock language and prompts the user to type, exactly:
   
   > *"I understand scope is locked. Changes from this point require the Addendum Protocol."*

3. On exact match, the helper sets `phase_4_gate_crossed = true`, `current_phase = 5`, and appends a `phase_4_gate_crossed` event to the phase gate log.

4. On mismatch or refusal, the helper exits without changing state.

**Do not paraphrase or shorten the acknowledgment text.** Exact-match enforcement is the friction. If the user types something slightly different, ask them to retype it verbatim.

### After the gate: the Addendum Protocol

Post-gate scope changes go through `.startup/procedures/addendum/addendum-protocol.md`, invoked by `/startup:addendum`. The protocol is deliberately friction-heavy:

1. The user provides a change summary, justification, and current build-state snapshot.
2. The helper assigns an addendum ID and creates a draft.
3. The Addendum Protocol prompt produces impact analysis (impacted parts, revised do-not-build list, risks introduced).
4. The user reviews and approves.
5. `startup-tools addendum-finalize <addendum_id> <diff-file>` applies the diff to `scope_definition` and appends an `addendum_completed` event.

**Original `do_not_build` entries must remain in the revised list** (verified by the intake's post-extract validator). New entries can be added with a `NEW-` tag.

The Addendum Protocol is the *only* permitted path for post-gate scope change. Manual PCD edits are still possible but are logged as `manual_pcd_edit` events and represent a known integrity violation. Discourage them.

---

## 10. Commands reference

Every `startup-tools` subcommand, with a one-line description. Invoke as `node .startup/bin/startup-tools.js <subcommand> [args]`.

Subcommands operating on project state implicitly target the **active project**. To target a different project for one call, pass `--project <ulid_or_slug>`. To change the active project persistently, use `switch`.

| Subcommand                                     | Purpose                                                                                                                                                                               |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `workspace-init`                               | Scaffold a brand-new workspace at the current directory. Creates `.startup/` with shared assets and an empty `projects/`. Run once per workspace.                                     |
| `init [--name <name>] [--no-activate]`         | Scaffold a new project inside the workspace. Creates `.startup/projects/<ulid>-<slug>/` with initial `pcd.json`, empty inbox, empty log. Updates `active.txt` unless `--no-activate`. |
| `list [--include-archived]`                    | List all projects in the workspace with name, current phase, last activity, and active marker.                                                                                        |
| `switch <ulid_or_slug>`                        | Set the active project. Validates the target exists and is not archived. Appends `project_switched_to` to the target's log.                                                           |
| `archive [--project <ulid>]`                   | Move the active (or specified) project to `.startup/projects/_archived/`. Clears `active.txt` if the archived project was active.                                                     |
| `unarchive <ulid_or_slug>`                     | Restore an archived project.                                                                                                                                                          |
| `status [--project <ulid>]`                    | Print full status of the active (or specified) project: name, phase, populated sections, stale sections, pending inbox files, last activity.                                          |
| `banner`                                       | Print a compact banner of the active project (used by SessionStart hook).                                                                                                             |
| `precondition <prompt_id>`                     | Verify a prompt's preconditions are met for the active project. Exits 0 if OK, nonzero with explanation if not.                                                                       |
| `render-context <section_ids...> [--optional]` | Render selected PCD sections of the active project as markdown for prompt context injection.                                                                                          |
| `render-pcd`                                   | Render the full PCD of the active project as markdown (for export).                                                                                                                   |
| `validate`                                     | Validate the active project's `pcd.json` against `pcd_schema.json`.                                                                                                                   |
| `validate-workspace`                           | Sanity-check the workspace: shared assets present, every project's PCD valid, `active.txt` resolves.                                                                                  |
| `apply-diff <diff-file>`                       | Validate and apply a JSON diff to the active project's `pcd.json`. Atomic. Triggers staleness propagation.                                                                            |
| `advance-phase`                                | Attempt to advance `current_phase`. Verifies preconditions. Writes phase gate log entry.                                                                                              |
| `back-phase <target_phase>`                    | Revert `current_phase`. Marks all sections populated after the target as stale. Writes log entry. Refused if the Phase 4 gate has been crossed.                                       |
| `cross-phase-4-gate`                           | Cross the one-way gate. Requires the typed acknowledgment as an argument. Sets `phase_4_gate_crossed = true`.                                                                         |
| `inbox-path`                                   | Print the active project's inbox directory absolute path. Procedures use this so the agent never computes paths manually.                                                             |
| `inbox-list`                                   | List pending inbox files in the active project with metadata.                                                                                                                         |
| `inbox-archive <file>`                         | Move a processed inbox file to the active project's archive.                                                                                                                          |
| `vault-list <type>`                            | List vault entries by type. Workspace-shared.                                                                                                                                         |
| `vault-get <type> <id>`                        | Print a vault entry as JSON.                                                                                                                                                          |
| `vault-add <type> <file>`                      | Add a new vault entry from a JSON file. Affects all projects in the workspace.                                                                                                        |
| `vault-match`                                  | Find Vault entries matching the active project's hackathon profile. Returns markdown for context injection.                                                                           |
| `gate-log <event_type> [payload-json]`         | Append an event to the active project's `phase_gate_log.jsonl`.                                                                                                                       |
| `handoff`                                      | Write `handoff.json` for the active project from current state. Called by the PreCompact hook (Claude Code) or manually.                                                              |
| `resume`                                       | Read the active project's `handoff.json` and produce a resumption hint.                                                                                                               |
| `addendum-init <justification>`                | Begin the Addendum Protocol for the active project. Requires Phase 4 gate crossed.                                                                                                    |
| `addendum-finalize <addendum_id> <diff-file>`  | Commit an addendum after user approval.                                                                                                                                               |

**Exit codes.** 0 = success. 1 = validation error. 2 = precondition unmet. 3 = file/IO error. 4 = user confirmation required.

**Behavior contracts.** The helper makes no network calls. The helper runs no LLM inference. All writes are schema-validated, atomic, and logged. If you see network or model behavior coming from anything named "startup-tools," it is not this helper.

> **Note on `mark-valid`.** The state-machine section of the implementation spec (Section 7.3) mentions a `startup-tools mark-valid <section>` subcommand for clearing `is_stale` after a backward transition that didn't actually invalidate downstream work. It is not listed in the canonical subcommand table (Section 5.1). Treat it as a planned subcommand to be confirmed when the helper is implemented — do not assume it exists until you see it in the helper's `--help` output.

---

## 11. Adapter-specific notes

Startup runs under any agent that can read files and execute shell commands. Two adapters are provided in v1; both are workspace-shared (one set of adapter files for all projects in the workspace).

### Claude Code (`.claude/`)

Claude Code is the **polished runtime**. It is the primary adapter and the one the spec optimizes for.

- `.claude/skills/<skill-name>/SKILL.md` — one skill per prompt, plus 12 utility skills. Skills register slash commands (`/startup:phase-2-premortem`, `/startup:intake`, `/startup:status`, etc.) and delegate to procedures.
- `.claude/agents/intake-parser.md` — the subagent that parses inbox files into structured diffs in isolation.
- `.claude/agents/grill-me.md` — the adversarial subagent used by Design Grill, Differentiation Stress Test, and Judge Q&A Drill.
- `.claude/settings.json` — configures two hooks: `SessionStart` (runs `startup-tools banner`) and `PreCompact` (runs `startup-tools handoff`).

If `.claude/` exists in this workspace, prefer slash-command invocation. The full skill catalog is in Section 11 of the implementation spec.

### Cursor (`.cursor/`)

Cursor is **functional but ergonomically degraded**. It exists because the user has community credits and may sometimes prefer it; it is honestly documented rather than papered over.

- `.cursor/rules/startup.mdc` — a single always-applied rules file pointing Cursor's agent at this STARTUP.md and listing common procedures.

What Cursor lacks (and the impact):

- **No slash commands.** Users reference procedures by path or by request ("run the premortem"). You match the request to the right procedure file in `.startup/procedures/`.
- **No subagent isolation.** Intake parsing and Grill-Me dialogue run in the main context. Be disciplined about not letting the Grill-Me adversarial frame leak into follow-up tasks.
- **No SessionStart hook.** Manually run `node .startup/bin/startup-tools.js status` at the start of every session.
- **No PreCompact hook.** If a session is compacted, there is no automatic handoff write. The user re-orients on the next session by running `status` manually.

What Cursor preserves: every primitive that matters for correctness — schema validation, atomic writes, gate enforcement, staleness propagation — lives in `startup-tools` and works identically in both adapters. The features Cursor loses are ergonomic, not foundational.

### Other agents

Any agent that can (a) read markdown files, (b) execute shell commands, (c) keep instructions stable across a session can drive this workspace. Read this STARTUP.md, follow procedures, use the helper. If you are such an agent, start at Section 2.

---

*End of STARTUP.md. If anything in this document conflicts with `STARTUP_CLI_IMPLEMENTATION_SPEC_v1_1.md` (or its latest versioned successor), the spec wins. Surface the discrepancy to the user so the document can be reconciled.*
