# Startup — Execution Instructions

**Version 1.0 — 2026-05-01**
*A sprint-by-sprint, prompt-by-prompt build plan for the Startup application.*

This document translates the build order in `STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md` §11.2 into executable mini-sprints. Each mini-sprint is sized to be small enough that Cursor or Antigravity can finish it in one focused pass without losing the thread, but big enough that you aren't paying agent overhead for trivia.

Roughly **8 sprints, ~53 mini-sprints, ~6 weeks of part-time work.**

---

## How to Use This Document

1. **Work top-to-bottom.** Sprints and mini-sprints are ordered by dependency. Skipping ahead breaks downstream work.
2. **Use this doc, not the spec, as your driver.** The spec is the canonical source of truth for *what* to build. This doc tells you *in what order* and *how to drive Cursor through it*.
3. **Each mini-sprint has one of three forms:**
   - **🤖 Agent task** — copy the prompt into Cursor or Antigravity and let it execute.
   - **🧑 Manual task** — you do it yourself. Setup, account work, design decisions, dogfooding.
   - **🤝 Hybrid** — manual setup followed by an agent-driven scaffold prompt.
4. **Don't skip the verification step.** After each mini-sprint, do the listed check before moving on. Cursor's failure mode is silent — code compiles but doesn't actually work.
5. **Commit to git after every mini-sprint.** No exceptions. Use the mini-sprint number in the commit message (e.g., `feat: sprint 2.4 — generate function with context selection`).

---

## Workflow Conventions

### Before You Start: Cursor / Antigravity Setup

Add these files to your tool's project context so every prompt grounds in them automatically. In Cursor, add them to **Docs**; or pin them as **Rules** if your tool supports it. In Antigravity, add them to the workspace's reference set.

- `STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md` *(canonical spec — primary)*
- `pcd_schema.json` *(canonical PCD schema)*
- `PROMPTS.md` *(canonical prompt content)*
- `default_judging_rubric.md` *(rubric content)*
- `STARTUP_WORKFLOW_AND_APP_OVERVIEW.md` *(product overview, for tone/intent)*
- `DESIGN.md` *(your design language doc — you'll author this in Sprint 0)*

Also create a `project_context.mdc` in your repo root containing the constraints below. Cursor will read this on every request.

```markdown
# Project Constraints — Startup

- Stack: React 18 + Vite + TypeScript (strict) + Tailwind + Zustand + Dexie.js + Ajv.
- No SSR. No backend database. No telemetry.
- All persistence is IndexedDB via Dexie. All sensitive data is AES-GCM encrypted via Web Crypto.
- The Cloudflare Worker proxy is a single stateless endpoint: POST /api/extract.
- Prompt assembly is deterministic templating in TypeScript. Gemini Flash is used ONLY for intake extraction.
- Free-tier-everything. No paid services in the MVP path.
- The PCD schema (pcd_schema.json) is canonical. JSON is the in-memory format. Markdown is render-only.
- When in doubt, defer to the implementation spec. When the spec is silent, ask before inventing.
```

### When to Use Planning Mode

I've flagged each mini-sprint with a **🧠 Planning Mode** indicator:

- **YES** — the task touches multiple files, requires understanding existing code, or involves architectural decisions. Use Cursor's plan-then-act mode (or Antigravity's equivalent) so the agent thinks before editing.
- **NO** — the task is a focused scaffold, single-file addition, or direct port from a spec section. Planning mode adds latency without quality gain.

### After Every Mini-Sprint

1. Run `pnpm typecheck` (or `npm run typecheck`) — must pass.
2. Run `pnpm dev` and click through the changed surface area visually.
3. Commit with a meaningful message.
4. If anything is broken, fix it *now*. Do not stack mini-sprints on broken foundations.

### Dogfood Checkpoints

You'll run Startup against itself at three points:

- **End of Sprint 2** — Generate the Phase 2 Problem Analysis prompt for an imaginary hackathon. Paste a fake LLM response back. Confirm the PCD updates.
- **End of Sprint 4** — Run a full Phase 1→4 pass on a past hackathon problem statement.
- **End of Sprint 6** — Run a full Phase 1→7 pass. This is your real readiness check.

---

## Sprint 0 — Pre-Build Setup

**Goal:** All the manual work that has to happen before Cursor can do anything useful.

---

### 0.1 — Create Accounts and Get Keys 🧑

**Manual task.** Allow ~30 minutes.

You need:
- **Cloudflare account** (free) — for Pages and Workers.
- **Google AI Studio account** (free) — for a Gemini API key with Flash access. Generate a key and store it somewhere you won't lose it. You'll paste it into Worker secrets in Sprint 1.
- **GitHub account** (you have this) — create an empty private repo named `startup`.
- **Domain** *(optional)* — defer until after MVP ships.

**Verification:** You can log into all three. You have a Gemini API key copied. You have an empty `startup` repo cloned to your machine.

---

### 0.2 — Author DESIGN.md 🧑

**Manual task. This is yours, not Cursor's.** Allow 1–3 hours.

The frontend will be visually consistent only if you decide what "Startup looks like" *before* the agent starts building screens. If you skip this, every screen will be slightly off-brand and Sprint 6's polish pass will be 3× longer.

**Produce a single `DESIGN.md` file** at the repo root containing:

1. **Mood / personality** — 3–5 adjectives. Startup is a command center, not a chatbot. Examples: *prescriptive, calm, dense, technical, slightly intimidating.* Write yours.
2. **Color palette** — 6–10 named CSS custom properties as Tailwind theme extensions. Define `--color-bg`, `--color-surface`, `--color-text`, `--color-text-muted`, `--color-accent`, `--color-warning`, `--color-danger`, `--color-success`, `--color-border`. Pick actual hex values. Reference inspirations (Linear, Vercel, Anthropic console) but commit to one direction.
3. **Typography scale** — pick one sans-serif (e.g. Inter, Geist, IBM Plex Sans) and one mono. Define heading sizes and body sizes.
4. **Spacing rhythm** — 4 / 8 / 12 / 16 / 24 / 32 / 48 / 64 px. Tailwind's default is fine; commit to it.
5. **Component primitives' visual language** — describe in one sentence each how the following look: cards, buttons (primary / secondary / danger), inputs, modals, badges, the phase-navigator sidebar, the diff preview.
6. **The Phase 4 gate visual treatment** — this is the most important visual element in the app. How does the lock icon look? How does the workspace change tone after the gate is crossed? Specify it.
7. **Three reference screenshots** — paste image links or local paths to three real apps whose visual language you want to echo.

**Verification:** You could hand `DESIGN.md` to a stranger and they could tell you what Startup looks like.

---

### 0.3 — Initialize the Repo and Tooling 🤝

**Manual setup, then one agent prompt to scaffold.**

First, manually:
1. `cd startup && git init` (if not done).
2. Add a `.gitignore` for Node.
3. Push the empty repo to GitHub.

Then run this prompt.

**🧠 Planning Mode: NO**

```
You are scaffolding the Startup repo. This is a frontend React app + a Cloudflare Worker proxy in one repo.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §8.1 (Stack) and §8.2 (Directory structure) before doing anything.

GOAL
Create the directory structure shown in §8.2 with the minimum scaffold to run `pnpm dev` from `frontend/` and `pnpm dev` from `proxy/`. No business logic yet.

DELIVERABLES
1. Top-level `package.json` with pnpm workspaces pointing to `frontend/` and `proxy/`.
2. `frontend/` — Vite + React 18 + TypeScript (strict mode), Tailwind CSS configured, react-router-dom v6 installed, Zustand installed, Dexie.js installed, Ajv installed, marked installed, lucide-react installed, react-hook-form + zod installed. Vite config for SPA. A single placeholder route at `/` rendering the text "Startup — placeholder".
3. `proxy/` — Cloudflare Workers project (TypeScript), `wrangler.toml` with placeholder `ALLOWED_ORIGINS`, single placeholder fetch handler that returns 404 for everything. No business logic.
4. Root `README.md` listing pnpm commands: `pnpm install`, `pnpm --filter frontend dev`, `pnpm --filter proxy dev`.
5. Tailwind preset that imports the color tokens defined in `DESIGN.md` (read it, ingest the palette, set them as CSS variables in `frontend/src/styles/tokens.css`, and reference them in `tailwind.config.ts` as `theme.extend.colors`).

CONSTRAINTS
- TypeScript strict mode ON.
- No SSR. No Next.js.
- Do not write business logic. Do not create files outside the §8.2 tree.

VERIFY
- `pnpm install` succeeds at root.
- `pnpm --filter frontend dev` opens a browser at the placeholder text.
- `pnpm --filter proxy dev` starts wrangler dev.
- `pnpm --filter frontend typecheck` passes.

REPORT
List every file created and confirm each verification step passed.
```

**Verification:** Both dev servers run. Tailwind classes work in the placeholder page. Commit.

---

### 0.4 — Deploy a Hello-World to Cloudflare Pages and Workers 🧑

**Manual task.** Allow ~30 minutes.

Get the deploy pipeline working *now*, before there's anything to deploy. Catch infra problems early when there's nothing else to debug.

1. Connect the GitHub `startup` repo to Cloudflare Pages. Build command: `pnpm --filter frontend build`. Output directory: `frontend/dist`.
2. Confirm the placeholder page is live at `https://startup.pages.dev` (or whatever subdomain you got).
3. Deploy the Worker via `pnpm --filter proxy deploy` (or `wrangler deploy`). Confirm 404 at the Worker URL.
4. Set `GEMINI_API_KEY` as a Worker secret: `wrangler secret put GEMINI_API_KEY`.
5. Set `ALLOWED_ORIGINS` in `wrangler.toml` to your Pages URL.

**Verification:** Pages URL loads the placeholder. Worker URL returns 404. Worker has the secret set (`wrangler secret list` shows `GEMINI_API_KEY`).

---

### 0.5 — Test Infrastructure 🤖

Set this up before Sprint 1 so every subsequent mini-sprint's test instructions land in a real, consistent home. Without this, Cursor will install Vitest piecemeal in different folders and your tests will be inconsistent garbage.

**🧠 Planning Mode: NO** *(focused configuration task)*

```
You are setting up the test infrastructure for the Startup repo. Both packages need testing; the conventions need to be consistent.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §8.2 (Directory structure) for context on where things live.

GOAL
Vitest configured for `frontend/` and `proxy/`. React Testing Library set up in `frontend/`. Root-level `pnpm test` runs both. One smoke test per package proving the setup works. Test conventions documented.

DELIVERABLES
1. `frontend/`:
   - Install dev deps: `vitest`, `@vitest/ui`, `@testing-library/react`, `@testing-library/jest-dom`, `@testing-library/user-event`, `jsdom`.
   - `frontend/vitest.config.ts` — environment `jsdom`, globals true, setupFiles pointing to a setup file.
   - `frontend/src/test/setup.ts` — imports `@testing-library/jest-dom`, configures any global mocks (none needed yet).
   - `frontend/src/test/fixtures/` — directory placeholder with a `README.md` explaining: this is where shared test fixtures (sample PCDs, sample Flash responses, etc.) go. Add a `samplePCDs.ts` placeholder exporting `EMPTY_PCD_FIXTURE` (the result of createEmptyPCD with minimal valid inputs — write this once createEmptyPCD lands in 1.2; for now, a stub `{} as any` with a TODO comment).
   - `frontend/package.json` scripts: `"test": "vitest run"`, `"test:watch": "vitest"`, `"test:ui": "vitest --ui"`.
   - One smoke test: `frontend/src/test/smoke.test.tsx` — renders a trivial component, asserts text is in the document. Proves React Testing Library + jsdom + Vitest are wired up correctly.

2. `proxy/`:
   - Install dev deps: `vitest`, `@cloudflare/vitest-pool-workers` (or use plain Vitest if you prefer simpler setup — your call, but document the choice).
   - `proxy/vitest.config.ts` — Workers pool config OR plain node config.
   - `proxy/package.json` scripts: `"test": "vitest run"`, `"test:watch": "vitest"`.
   - One smoke test: `proxy/src/smoke.test.ts` — asserts the worker handler returns 405 for GET. Proves the Worker test setup works.

3. Root `package.json` script: `"test": "pnpm -r test"` so a single `pnpm test` at the repo root runs both packages.

4. `project_context.mdc` — append a "Testing conventions" section:
   - Tests live next to the code they test, in `*.test.ts(x)` files. NOT in a separate `__tests__` directory.
   - Shared fixtures live in `frontend/src/test/fixtures/`.
   - Pure logic gets unit tests. UI gets minimal integration tests via React Testing Library — one happy path, one failure path. Don't pursue coverage; pursue confidence.
   - Test descriptions follow the pattern `describe('<thing>', () => it('<observable behavior>', ...))`.
   - Mock the Cloudflare Worker proxy in frontend tests via Vitest's `vi.mock('../proxy/client')`. Don't make real network calls.
   - When a bug is fixed, write a regression test covering it. Bugfix without test = the bug returns.

5. CI hook (lightweight): a GitHub Action `.github/workflows/test.yml` that runs `pnpm install` and `pnpm test` on push and PR. No coverage gating, no deploy gating. Just a green/red signal.

CONSTRAINTS
- No testing libraries beyond what's listed above. No Jest, no Mocha, no Playwright. Vitest only.
- The smoke tests are deliberately trivial. They prove infrastructure, not behavior.
- The fixtures placeholder is a placeholder — populate it as real fixtures emerge across Sprints 1–7.

VERIFY
- `pnpm test` at root runs both packages and both smoke tests pass.
- `pnpm --filter frontend test:watch` opens watch mode.
- The GitHub Action runs green on a fresh push.

REPORT
Confirm each verify step. Paste the final scripts entries from both package.jsons.
```

**Verification:** Both smoke tests pass. CI runs green on push. Commit.

---

## Sprint 1 — Foundation

**Goal:** Encryption, persistence, and the schema-driven shell. By end of sprint, you can create a project (Phase 1 inputs only) and it persists across reloads through an encrypted store.

**Maps to spec §11.2 Milestone 1.**

---

### 1.1 — Persistence + Encryption + Lock Screen 🤖

This combines tasks 2 from §11.2 (Dexie schema, Web Crypto, lock screen, first-run onboarding) into one mini-sprint because they are tightly coupled — you cannot build any of them without the others.

**🧠 Planning Mode: YES** *(touches multiple foundational files; agent must understand the encryption flow before writing any code)*

```
You are building the persistence and encryption layer for Startup. This is the load-bearing foundation; everything else depends on it.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §8.3 (Data persistence layer) and §8.4 (Encryption flow) and §9.1 (Application Lock Screen) and §9.2 (First-Run Onboarding) before writing any code.

GOAL
Implement the Dexie store, the AES-GCM encryption layer with PBKDF2 key derivation, the lock screen, and the first-run onboarding flow. After this mini-sprint, the app should require a password on every open and persist all data encrypted at rest.

DELIVERABLES
1. `frontend/src/lib/persistence/db.ts` — Dexie class exactly matching §8.3. Tables: `projects`, `vault_judges_organizers`, `vault_winning_solutions`, `vault_boilerplate`, `settings`. Indexes per §8.3. The encrypted record shape per `EncryptedProject` etc.
2. `frontend/src/lib/persistence/crypto.ts` — exports `deriveKey(password, salt)`, `encrypt(plaintext, key)`, `decrypt(ciphertext, iv, key)`. PBKDF2 SHA-256 210,000 iterations. AES-GCM. All using Web Crypto API. No third-party crypto libraries.
3. `frontend/src/lib/persistence/migrate.ts` — placeholder for future schema_version migrations. For now, a no-op that returns the input as-is. Document its purpose with a top-of-file comment.
4. `frontend/src/stores/auth.ts` — Zustand store holding the in-memory derived key. Exposes `unlock(password)`, `lock()`, `isUnlocked()`. Auto-lock after 30 min inactivity (window event listeners — `mousemove`, `keydown`, `visibilitychange`). The key never persists; only sits in memory.
5. `frontend/src/screens/lock/LockScreen.tsx` — implements §9.1 exactly. Password input, unlock button, first-time setup link only when no verification token exists in `settings`, anti-brute-force lockout (30 sec after 5 fails). Uses the auth store.
6. `frontend/src/screens/onboarding/FirstRunOnboarding.tsx` — implements §9.2 exactly. Four steps. Strength meter on password (use `zxcvbn` — install it). Typed-phrase confirmation in step 2. On finish, derive key, write the verification token (an encrypted constant), seed default settings, route to `/dashboard`.
7. `frontend/src/app/Routes.tsx` — initial routing. `/` → LockScreen if a verification token exists, else FirstRunOnboarding. `/dashboard` → placeholder "Dashboard" component for now (will be filled in 1.4). All non-public routes redirect to `/` if `auth.isUnlocked()` is false.

CONSTRAINTS
- TypeScript strict. No `any`.
- Web Crypto only. No `crypto-js` or other JS crypto libraries.
- The derived key MUST NOT be persisted anywhere — not localStorage, not Dexie, not Zustand persist middleware. In-memory only.
- Salt is per-installation, stored unencrypted in `settings`. IV is per-encrypt-call, stored alongside the ciphertext.
- All copy follows §10.1 prescriptive voice. The lock screen's setup link and irrecoverability warning use the exact phrasing in §9.1.

VERIFY
- `pnpm typecheck` passes.
- App boots to onboarding on first run.
- After onboarding, app boots to lock screen on subsequent reloads.
- Wrong password rejects. After 5 fails, 30-sec lockout activates.
- Reloading mid-session takes you back to the lock screen (key in memory only).
- A test record written to `projects` table is visibly encrypted in DevTools → Application → IndexedDB (you should see ciphertext bytes, not plaintext).

REPORT
For each verify step, paste the result. If any step fails, do not move to the next mini-sprint.
```

**Verification:** Open the app, set up a password, lock it, reload, unlock again. Open DevTools → Application → IndexedDB → StartupDB → `settings` and confirm the verification token field is binary, not human-readable text.

---

### 1.2 — PCD Schema Import, Validator, and createEmptyPCD 🤖

**🧠 Planning Mode: NO** *(spec gives exact deliverables; mostly typing work)*

```
You are wiring up the PCD schema as the canonical type source for the Startup application.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §2 (The PCD), §2.7 (createEmptyPCD), and §8.2 (Directory structure). Read pcd_schema.json — it is the canonical schema.

GOAL
Make the PCD schema available as a typed runtime artifact. Implement Ajv validation. Implement createEmptyPCD.

DELIVERABLES
1. Copy `pcd_schema.json` into `frontend/src/lib/pcd/pcd_schema.json` (or import it directly from project root via Vite's JSON import — pick whichever is cleaner; document your choice).
2. `frontend/src/lib/pcd/types.ts` — generated by `json-schema-to-typescript`. Add an npm script `gen:types` to `frontend/package.json` that produces this file from `pcd_schema.json`. Run it once. Commit the generated file. Document in a top-of-file comment that this file is auto-generated and not to be edited by hand.
3. `frontend/src/lib/pcd/schema.ts` — re-exports the generated types under cleaner names: `PCD`, `SectionEnvelope<T>`, `PhaseGateLogEntry`, etc. This is the import path the rest of the app uses.
4. `frontend/src/lib/pcd/validate.ts` — Ajv wrapper. Exports `validatePCD(pcd: unknown): { valid: boolean; errors: ErrorObject[] }`. Loads schema once at module init. Uses Ajv with `strict: false` to handle the JSON Schema draft 2020-12 features.
5. `frontend/src/lib/pcd/createEmptyPCD.ts` — exact spec from §2.7. Takes hackathon profile inputs and the resolved active rubric, returns a fully-formed PCD with every section's envelope correctly initialized (`is_populated: false`, etc., except for `hackathon_profile` and `active_judging_rubric` which are populated immediately).

CONSTRAINTS
- The Ajv validator must accept the schema as-is. If it fails to compile, FIX the schema-loading code, not the schema.
- `createEmptyPCD` must produce output that passes `validatePCD()`. Add a unit test that proves this.
- TypeScript strict.

VERIFY
- `pnpm typecheck` passes.
- A unit test (use Vitest — install it now if not present) validates that `createEmptyPCD({...minimal valid inputs...})` returns a PCD that passes Ajv validation.
- Manual: import `validatePCD` in a scratch file, pass it `{}`, confirm it returns `valid: false` with sensible errors.

REPORT
Paste the test output. Paste the npm scripts you added.
```

**Verification:** Run `pnpm test`. The validation test passes. Commit.

---

### 1.3 — Default Rubric Parser 🤖

**🧠 Planning Mode: NO** *(small, well-specified)*

```
You are implementing the Default Judging Rubric loader.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §7 (Default Judging Rubric). Read default_judging_rubric.md — it is the source of truth for rubric content.

GOAL
Parse default_judging_rubric.md into the JSON shape defined for `active_judging_rubric.data`. Cache the result. Expose it via `getDefaultRubric()`.

DELIVERABLES
1. Copy `default_judging_rubric.md` into `frontend/src/lib/rubric/default.md` and configure Vite to import its raw text (use `?raw` import).
2. `frontend/src/lib/rubric/parseDefault.ts` — exports `parseDefaultRubric(markdown: string): ActiveJudgingRubricData`. Walks H2 headers (`## Criterion N — Name`). Extracts: `name`, `description` (paragraph following the header), `weight` (parsed from a `**Weight:** 0.20` line), `judge_questions` (bulleted list under a `**Judge questions:**` subheader). Strict — if any criterion is malformed, throw with a descriptive error.
3. `frontend/src/lib/rubric/getDefaultRubric.ts` — exports `getDefaultRubric()` which calls the parser at module-init time, caches the result, and returns it. Also exports `resetCachedRubric()` for the settings flow that overrides the default (this hooks in later — for now it's a stub that clears the cache).
4. Unit test: parsing the bundled `default.md` produces 5 criteria with weights summing to 1.0 (±0.001).

CONSTRAINTS
- Pure functions. No side effects beyond the cache.
- Use a basic markdown parser (`marked` already installed) or just regex — your choice, but justify in a comment if you use regex for anything beyond H2 detection.

VERIFY
- `pnpm test` passes the new test.
- `getDefaultRubric()` returns valid data shape (validate it against the `active_judging_rubric` fragment of pcd_schema.json — you have Ajv now).

REPORT
Paste the test output. Paste the parsed rubric structure (just the names and weights).
```

**Verification:** Test passes. Inspect the parsed output — it matches the markdown content.

---

### 1.4 — Project Dashboard Skeleton 🤖

**🧠 Planning Mode: YES** *(coordinated UI build with state, persistence, navigation)*

```
You are building the Project Dashboard — the home screen.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §9.3 (Project Dashboard) and §10 (UX Principles, especially 10.1 prescriptive voice). Read DESIGN.md for visual treatment.

GOAL
Implement the Project Dashboard with active and archived project lists, the New Project CTA, lock-now and Settings/Vault links in the header, and an empty state. Project cards display from real Dexie data (encrypted projects, decrypted on read).

DELIVERABLES
1. `frontend/src/stores/projects.ts` — Zustand store. Loads all projects from Dexie on unlock. Exposes `projects`, `archivedProjects`, `loadProjects()`, `archiveProject(id)`, `renameProject(id, newName)`, `deleteProject(id)`, `exportProjectMarkdown(id)`. All Dexie reads decrypt the ciphertext using the auth store's session key. All writes re-encrypt before persisting.
2. `frontend/src/lib/persistence/projectRepo.ts` — the actual encrypt/decrypt round-trip wrapper around Dexie's projects table. The Zustand store should call this repo, not Dexie directly.
3. `frontend/src/screens/dashboard/ProjectDashboard.tsx` — implements §9.3 exactly. Header with wordmark, lock button (calls `auth.lock()`), Settings link (placeholder route for now), Vault link (placeholder route for now). New Project primary CTA → routes to `/projects/new` (placeholder for 1.5). Active projects list. Archived collapsed list. Empty state per §10.1 ("Create your first project to begin.").
4. `frontend/src/components/ProjectCard.tsx` — name, hackathon name, current phase badge, time-to-deadline if set, stale-section count if any (placeholder zero for now). Right-click / kebab menu with Archive, Rename, Delete (typed-phrase confirmation), Export PCD (stub for now).
5. Phase badge component used in 4 — small pill labeled "Phase N" with color from DESIGN.md tokens.
6. Routing: `/dashboard` is the dashboard. Default route after unlock is `/dashboard`.

CONSTRAINTS
- All copy follows §10.1 prescriptive voice. No "Welcome to Startup!" — the empty state directs the user.
- Tailwind utility classes only; reference design tokens via Tailwind's theme.extend.colors.
- No mock data — the dashboard reads real Dexie state. If empty, show the empty state.
- Delete uses typed-phrase confirmation per §10.4 (the user types the project name to confirm).

VERIFY
- Empty state renders on a fresh install.
- Manually inserting a test project record (use a temporary "Add test project" button you can delete in the next mini-sprint, or use DevTools to call the repo directly) shows a card.
- Rename works and persists across reload.
- Delete with typed-phrase confirmation works.
- Lock button returns to lock screen.

REPORT
Screenshots of: empty state, dashboard with 1 project, the rename modal, the delete typed-phrase confirmation modal.
```

**Verification:** Click through every interaction. Persistence survives reload. Commit.

---

### 1.5 — New Project Setup Form (Phase 1 inputs only) 🤖

Per §11.2 M1's "Note on Rubric Mapping Intake": the **judging criteria textarea is disabled** in this mini-sprint. We re-enable it in Sprint 2 once the proxy and intake pipeline land.

**🧠 Planning Mode: YES** *(form is large; integrates with rubric loading and createEmptyPCD)*

```
You are building the New Project Setup form.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §9.4 (New Project Setup) and §11.2 M1 "Note on Rubric Mapping Intake at M1".

GOAL
Implement the form per §9.4. On submit, load the default rubric, call createEmptyPCD with the form values, encrypt and write to Dexie, navigate to a placeholder Project Workspace at Phase 1.

DELIVERABLES
1. `frontend/src/screens/projectSetup/NewProjectSetup.tsx` — full form per §9.4. Use react-hook-form + zod for validation. Required: hackathon name, organizer name. All other fields optional.
2. The "Judging criteria" textarea is rendered DISABLED with this exact tooltip: "Coming in next build — paste official criteria from Settings once available." Add a comment in the code referencing M1's deferral note.
3. The Vault auto-query hook is also a stub for now: comment marker `// TODO Sprint 5.3 — Vault auto-query` placed where §9.4 says it goes.
4. On submit:
   - Validate the form.
   - Call `getDefaultRubric()`.
   - Call `createEmptyPCD({ ...formValues, default_rubric: defaultRubric, rubric_source: 'default' })`.
   - Validate the resulting PCD against Ajv. On validation failure, show a generic error toast and DO NOT write — log the Ajv errors to console for debugging.
   - Encrypt and persist via the project repo.
   - Navigate to `/projects/:id` (placeholder route showing project name and "Phase 1 workspace coming in Sprint 2").
5. `/projects/:id` — minimal placeholder route. Just shows the project name, current phase, and a "back to dashboard" link.

CONSTRAINTS
- The form is long. Group it visually per §9.4: Hackathon details, Organizer + Sponsors, Problem & criteria, Timeline, Format & team. Each group is its own card-like container.
- Datetime pickers: use a basic native `<input type="datetime-local">` for MVP. No date picker library.
- Sponsors are a dynamic list (add / remove rows).
- All required fields per §9.4. Submit is disabled while form invalid.

VERIFY
- Form validates correctly.
- Submitting writes a project. After submit, the dashboard shows the new card.
- The created PCD passes Ajv validation (verify by logging it).
- The judging criteria field is visibly disabled with the tooltip.

REPORT
Screenshots: form (empty state), form (filled), dashboard showing the new project card. Paste the validated PCD JSON of one created project (redact anything sensitive).
```

**Verification:** Create 2–3 fake projects. Confirm each appears on the dashboard. Reload — they persist. Commit.

---

### 1.6 — Debug Panel + Logger + Error Boundary 🤖

You're about to spend Sprints 2–7 watching prompts generate, intakes parse, and PCDs commit. When something fails silently — and Cursor's failure mode is *always* silent — you need somewhere to look that isn't DevTools. This mini-sprint adds an in-app debug surface so future dogfooding doesn't require browser-tab archaeology.

**🧠 Planning Mode: NO** *(small, focused, self-contained additions)*

```
You are adding a lightweight debug surface to Startup. This is for the developer (you), not the end user.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §8.6 (deliberately not in MVP — note "no telemetry/analytics" and "no error reporting service"). This mini-sprint stays inside those constraints: nothing is sent off-device, nothing persists across reloads.

GOAL
A ring-buffer logger, a top-level React error boundary, a keyboard-toggle debug drawer, and structured console.error in the Worker proxy on Flash failures. All in-memory. All local. Off-by-default in production builds.

DELIVERABLES

1. `frontend/src/lib/log/logger.ts` — exports `log` with methods `debug(scope, msg, payload?)`, `info(scope, msg, payload?)`, `warn(scope, msg, payload?)`, `error(scope, msg, payload?)`. Each call:
   - Pushes to a module-level ring buffer (cap 200 entries; drop oldest on overflow).
   - Mirrors to `console[level]` with a tagged prefix `[scope]`.
   - Records `{ level, scope, msg, payload, timestamp }`.
   Also exports `getRingBuffer()` (returns a copy) and `clearRingBuffer()`. Pure module state — no React, no persistence.

2. `frontend/src/lib/log/scopes.ts` — string union of allowed scope names so callers don't typo: `'auth' | 'persistence' | 'pcd' | 'prompts' | 'intake' | 'proxy' | 'stateMachine' | 'vault' | 'ui'`. Logger methods take this union for their first arg.

3. `frontend/src/components/debug/ErrorBoundary.tsx` — React error boundary at the app root (wrap the router in it). On caught error:
   - Calls `log.error('ui', err.message, { stack, componentStack })`.
   - Renders a fallback UI: a card with the error message, a "Reload app" button, a "Show last 20 log entries" toggle that displays the tail of the ring buffer. Visual treatment per DESIGN.md's danger token.

4. `frontend/src/components/debug/DebugDrawer.tsx` — slide-out panel pinned bottom of viewport. Toggle with `Ctrl+Shift+D` / `⌘+Shift+D`. Renders the ring buffer in reverse-chronological order with level color coding, scope filter chips, level filter chips, a clear button, and a "Copy all as JSON" button (for pasting into a bug report). Auto-refreshes every 1s while open.
   - The drawer is rendered always but only mounts contents when `isOpen` is true. Toggle state is local component state, not persisted.
   - Hidden in production builds via `import.meta.env.PROD` check at the top of the component — return `null` in production. (You CAN flip a build flag to enable it in production for debugging; document this in `project_context.mdc`.)

5. `frontend/src/lib/log/wireExisting.ts` — replace existing `console.log`/`console.error` calls in already-built modules with the structured logger. Specifically:
   - `auth` store's unlock/lock/auto-lock events → `log.info('auth', ...)`.
   - `persistence/projectRepo.ts` reads/writes → `log.debug('persistence', ...)` with project id only, NEVER ciphertext or plaintext content.
   - `pcd/validate.ts` validation failures → `log.warn('pcd', 'PCD validation failed', { errors })`.
   - `rubric/parseDefault.ts` parse errors → `log.error('pcd', ...)`.
   This wiring touches the existing files; do it surgically, one file at a time, and run `pnpm test` after each edit.

6. `proxy/src/worker.ts` — add structured `console.error` on Flash 502 failures and on rate-limit hits (info level). Format as JSON: `console.error(JSON.stringify({ event: 'flash_error', detail, requestId }))`. Cloudflare's Workers logs surface this in `wrangler tail`. Do NOT log `body.raw_input`, `body.prompt`, or any user content. Log only metadata: intake_id, response status, timing, error message. Add a brief comment in the Worker explaining the redaction rule.

7. Anti-leak invariant: write a unit test asserting that `log.info('persistence', 'project loaded', { id, ciphertext })` does NOT include the ciphertext field in the ring-buffer entry. Implement this by having the logger strip any keys named `ciphertext`, `plaintext`, `password`, `derivedKey`, `apiKey`, `raw_input`, `prompt` from logged payloads (recursively, via a small redactor utility). Document the redacted key list in a top-of-file comment.

CONSTRAINTS
- In-memory only. Nothing in IndexedDB, nothing in localStorage, nothing sent over the network.
- The redactor runs on every log call. It is a hard floor, not a convention.
- Production builds hide the drawer entirely. Error boundary still catches and renders fallback in production, but its log preview is also gated behind a build-flag check.
- Ring buffer cap is fixed at 200. If you find yourself wanting more, you have a bigger problem to debug differently.

VERIFY
- Cause an error in a component (temporarily throw in render). Confirm the error boundary catches it, the log entry appears in the ring buffer, and the fallback UI renders.
- Open the debug drawer (`⌘+Shift+D`). See log entries. Filter by scope. Clear. Copy as JSON — paste somewhere, confirm structure.
- The redaction test passes: a payload containing `ciphertext` is logged with `ciphertext` stripped.
- Build for production (`pnpm --filter frontend build`). Run the production build locally (`pnpm --filter frontend preview`). Confirm `⌘+Shift+D` does nothing — the drawer is gone.
- `wrangler tail` against the deployed Worker shows the structured error JSON when you trigger a Flash failure (you can simulate by passing an invalid schema to `/api/extract`).

REPORT
- Paste the redactor's redacted-keys list.
- Screenshot of the debug drawer with sample log entries.
- Screenshot of the error boundary fallback.
- Confirm production build hides the drawer.
- Paste a `wrangler tail` line showing structured Worker error logging.
```

**Verification:** Trigger an error → caught by boundary, logged, visible in drawer. Production build hides the drawer. Worker errors are structured. Commit.

**End of Sprint 1.** You can now create encrypted projects with all Phase 1 inputs except official rubric mapping. Solid foundation, with a debug surface for everything that comes next.

---

## Sprint 2 — The Prompt-Generate-Paste-Back Loop

**Goal:** End-to-end vertical slice for **one** prompt-intake pair (Phase 2 Problem Analysis). Once this works, Sprint 3 is just replication across all phases.

**Maps to spec §11.2 Milestone 2.**

---

### 2.1 — Cloudflare Worker /api/extract Endpoint 🤖

**🧠 Planning Mode: YES** *(the proxy is the only server-side code; getting it right matters)*

```
You are implementing the Cloudflare Worker proxy that fronts Gemini Flash.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §8.5 (The Cloudflare Worker proxy).

GOAL
A single POST /api/extract endpoint that proxies extraction requests to Gemini Flash with response_schema. Origin-checked. Rate-limited. Stateless.

DELIVERABLES
1. `proxy/src/worker.ts` — exact implementation per §8.5. Origin check against `ALLOWED_ORIGINS`. Rate limit via Workers KV (~300/day soft cap per IP).
2. `proxy/src/rateLimit.ts` — KV-based per-IP counter, daily reset. Helper exposed as `isRateLimited(request, env)`.
3. `proxy/wrangler.toml` — KV namespace binding for rate limiting. Two environments: `staging` and `production` with separate KV namespaces. Both reference the same `GEMINI_API_KEY` secret name.
4. The Worker's request body shape MUST match what the frontend sends in 2.7: `{ intake_id: string, raw_input: string, schema: object, prompt: string }`. Validate this shape; return 400 on malformed body.
5. The Worker calls Gemini Flash at the correct endpoint with `generationConfig: { response_mime_type: 'application/json', response_schema: body.schema, temperature: 0.2 }`. Use the model name `gemini-2.5-flash` (or the latest free-tier Flash model — verify against the current Google AI docs and use whatever is current).
6. Response shape: `{ data: <stringified JSON of Flash's text content>, raw: <full Flash response object>, error?: string }`.

CONSTRAINTS
- Stateless. No logging of raw_input or prompt content to KV or any external service.
- The rate limit counter is the ONLY thing written to KV.
- API key is read only from Worker secrets (`env.GEMINI_API_KEY`). Never log it.
- Strong CORS handling: respond to OPTIONS preflight; only allow Origin in `ALLOWED_ORIGINS`.

VERIFY
- `pnpm --filter proxy dev` runs locally.
- Curl a malformed body → 400.
- Curl from a disallowed origin → 403.
- Curl 301 times in a day from one IP → 429 on the 301st.
- Curl with a valid body and a trivially small schema (e.g., `{type:"object",properties:{x:{type:"string"}}}`) and a prompt asking the model to extract "hello" — confirm Flash returns valid JSON and the proxy returns it cleanly.
- Deploy to Cloudflare and re-run the live curl tests.

REPORT
Paste curl commands and outputs for each verify step. Confirm the deployed Worker URL.
```

**Verification:** Real curl against deployed Worker returns extracted JSON. Commit.

---

### 2.2 — PromptDefinition Registry + RuntimeInputDef Types 🤖

**🧠 Planning Mode: YES** *(types are the contract for everything that follows)*

```
You are implementing the PromptDefinition registry and its supporting types.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §4.1 (PromptDefinition shape), §4.2 (RuntimeInputDef), §4.3 (ContextSelector). Read PROMPTS.md to see the structure of actual prompt content.

GOAL
Define the type contracts and stand up the registry as an empty array. Add ONE entry: phase_2_problem_analysis. This is the slice we'll wire end-to-end before replicating.

DELIVERABLES
1. `frontend/src/lib/prompts/types.ts` — exports `PromptDefinition`, `ContextSelector`, `RuntimeInputDef`, `GrillMeMode`, `PromptId` (string union of all known prompt IDs — for now just `'phase_2_problem_analysis'`).
2. `frontend/src/lib/prompts/registry.ts` — exports `PROMPT_REGISTRY: Record<PromptId, PromptDefinition>`. For now, only the one entry.
3. `frontend/src/lib/prompts/templates/phase_2_problem_analysis.ts` — exports the role_assignment, task_specification_template, output_template_hint, and any constraint strings as constants. Source the content directly from PROMPTS.md (find the Phase 2 Problem Analysis prompt content there). Do not paraphrase — port verbatim.
4. The PromptDefinition for `phase_2_problem_analysis` declares context_selectors per §4.3 selection rules table (required: `hackathon_profile.data.problem_statement_raw`, `hackathon_profile.data.theme`; optional: `judge_intelligence`, `active_judging_rubric`). No runtime_inputs (Phase 2 Problem Analysis doesn't take any). Declares its produces_intake_id as `phase_2_problem_analysis_intake` (the intake itself comes in 2.7).

CONSTRAINTS
- Type-safe. Every PromptDefinition field must align with the types in §4.1. If you find ambiguity in §4.1, choose the strictest interpretation and document the choice in a comment.
- Verbatim port of PROMPTS.md content. The registry is the single source of truth for prompt content at runtime; the templates module mirrors what's in PROMPTS.md.
- DO NOT implement the generate function yet — just types and the registry.

VERIFY
- `pnpm typecheck` passes.
- Importing `PROMPT_REGISTRY` and accessing `PROMPT_REGISTRY.phase_2_problem_analysis` works in a scratch file.
- The role_assignment and task_specification_template strings, when concatenated, match what's in PROMPTS.md for Phase 2 Problem Analysis.

REPORT
List all files created. Paste the PromptDefinition for phase_2_problem_analysis.
```

**Verification:** Typecheck passes. Manually compare the template string content to PROMPTS.md.

---

### 2.3 — schemaToOutputTemplate Function 🤖

**🧠 Planning Mode: NO** *(focused single-function implementation)*

```
You are implementing the schemaToOutputTemplate function — the bit that turns a JSON Schema fragment into a human-readable output template the external LLM can follow.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §4.4 (Output template generation from schema). Read pcd_schema.json to see the shape of `problem_analysis` data — that's the schema fragment we'll exercise this with.

GOAL
A pure function that takes a JSON Schema object and returns a Markdown-formatted output template. The template tells the external LLM exactly what structure to produce.

DELIVERABLES
1. `frontend/src/lib/prompts/schemaToOutputTemplate.ts` — exports `schemaToOutputTemplate(schemaFragment: object, options?): string`. Walks the schema. Renders each field as a numbered or bulleted markdown line with field name, type hint, and description (from the JSON Schema's `description` keyword if present). Arrays get an example shape. Nested objects render with indentation.
2. Unit tests covering: a flat object schema, a nested object schema, an array of strings, an array of objects, a schema with descriptions, a schema without descriptions.
3. Specific test: rendering the `problem_analysis.data` fragment from pcd_schema.json produces a sensible template that a human could follow. Snapshot-test the output (use Vitest snapshots).

CONSTRAINTS
- Pure function. No side effects.
- Output is markdown, not JSON.
- The template must be readable by humans first, parseable by LLMs second.
- Handle missing `description` gracefully — fall back to the field name as the description.

VERIFY
- All tests pass.
- Manual: render the problem_analysis fragment, eyeball the output, confirm a human could follow it to produce the right shape.

REPORT
Paste the rendered template for problem_analysis. Paste test output.
```

**Verification:** The rendered template reads naturally.

---

### 2.4 — Generate Function with Deterministic Context Selection 🤖

**🧠 Planning Mode: YES** *(this is the assembly logic; many moving parts)*

```
You are implementing the prompt generate function — the deterministic assembly that turns a PromptDefinition + the current PCD into a copyable prompt string.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §4 in full. Especially §4.3 (Context selection), §4.4 (Output template generation), §4.5 (Generation flow), §4.6 (Flash NOT used for generation), §4.7 (Grill-Me mode rewrite).

GOAL
A single `generate(promptId, pcd, runtimeInputs, grillMode): GenerateResult` function that assembles the full prompt or returns a clear error if preconditions are unmet.

DELIVERABLES
1. `frontend/src/lib/prompts/generate.ts` — exports the generate function. Returns `{ ok: true, prompt: string, includedContext: ContextProvenance[] } | { ok: false, error: GenerateError }`.
2. The generate function:
   - Looks up the PromptDefinition by id.
   - Validates each `ContextSelector`: required-and-empty → return error with `precondition_unmet` and the missing section name.
   - Resolves selectors against the PCD. Inclusion rules: `'always'`, `'if_populated'`, `'if_user_research_complete'`, `'if_user_provided'`. Implement all four.
   - Assembles the prompt in §4.5 order: preamble, role, context, task spec (with runtime input substitution), output template (from schemaToOutputTemplate, called against the intake's target schema fragment).
   - If `grillMode !== 'off'` and the PromptDefinition declares grill_me_modes, replace the task spec with the Grill-Me variant per §4.7. Substitution of runtime inputs happens BEFORE the Grill-Me rewrite — the Grill-Me variant operates on filled values. Use the Interactive vs Auto template per the mode.
3. `frontend/src/lib/prompts/generate.types.ts` — `ContextProvenance`, `GenerateError`, `GenerateResult`. ContextProvenance = which sections were included, with their last_updated and populated_by_phase, for the "Show injected context" panel.
4. Unit tests: 
   - Generate the phase_2_problem_analysis prompt against an empty PCD → expect `precondition_unmet` error naming `hackathon_profile.data.problem_statement_raw`.
   - Generate it against a PCD with a problem statement filled in → expect a string ending in the output template.
   - Generate it with grillMode='interactive' (note: phase 2 problem analysis is NOT Grill-Me eligible per current spec — pick a Grill-Me-eligible prompt for this test, OR add a temporary grill_me_modes to the test fixture).
5. Phase Gate Log: log a `prompt_generated` event with payload `{ prompt_id, timestamp, included_vault_ids, grill_me_mode, runtime_inputs_used }`. For now, just write to a Zustand store; the persistent log lands in 4.3.

CONSTRAINTS
- Pure logic. No DOM, no persistence side effects (except the in-memory log entry).
- Deterministic: same inputs → same output (modulo timestamp).
- TypeScript strict. Exhaustive switch on inclusion_rule.

VERIFY
- All tests pass.
- Manual: in a scratch component, fill in a problem statement and theme on a project's hackathon_profile, call generate(), paste the result into Claude/Gemini and confirm a sensible problem analysis comes back.

REPORT
Paste a generated prompt for phase_2_problem_analysis with sample inputs.
```

**Verification:** A real LLM produces a sensible analysis when given the generated prompt. This is the first taste of dogfooding.

---

### 2.5 — Project Workspace Shell + Phase Navigator Sidebar 🤖

**🧠 Planning Mode: YES** *(the dominant screen in the app; layout decisions echo through all later sprints)*

```
You are building the Project Workspace shell and Phase Navigator sidebar. This is the dominant screen of the app per §9.5.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §9.5 (Project Workspace) in full. Read §10.3 (Phase 4 gate is sacred). Read DESIGN.md for visual treatment.

GOAL
Build the workspace layout — header, sidebar with phase nodes, and a body area with placeholder cards for Phase Briefing, Completion Checklist, Prompt Generator, Research Intake, Phase Output Preview, and the Advance CTA. The body fills with real content in 2.6 (Prompt Generator) and 2.8 (Intake) and Sprint 3 (the rest).

DELIVERABLES
1. `frontend/src/screens/workspace/ProjectWorkspace.tsx` — implements the layout in §9.5's ASCII diagram. Routes: `/projects/:id`. Reads project from store; if not found, redirect to dashboard. Decrypts and loads the PCD into a workspace-scoped Zustand store.
2. `frontend/src/stores/workspace.ts` — Zustand store. `currentProject`, `pcd`, `currentPhase`, actions: `loadProject(id)`, `setCurrentPhase(n)`, `commitPCD(updatedPCD)` (validates with Ajv, encrypts, persists, refreshes store).
3. `frontend/src/components/PhaseNavigator.tsx` — sidebar per §9.5. Phase nodes 1-7. Vault link separately. Current phase highlighted. Completed phases marked ✓. **Phase 4 gate has a lock icon that visibly closes when crossed** — implement this even though crossing logic comes in Sprint 4. For now, show the lock as open. Stale-section count badge if any (placeholder zero — wires up in Sprint 4). Backward navigation is allowed for phases before Phase 4; clicking a future phase that isn't unlocked shows a tooltip "Complete Phase N to unlock". Forward navigation by clicking a phase node ONLY works for already-unlocked phases.
4. `frontend/src/components/workspace/WorkspaceBody.tsx` — placeholder layout with labeled empty card slots for: Phase Briefing, Completion Checklist, Prompt Generator, Research Intake, Phase Output Preview. Each card slot is its own component file (empty for now): `PhaseBriefingCard`, `CompletionChecklistCard`, `PromptGeneratorCard`, `ResearchIntakeCard`, `PhaseOutputPreviewCard`. Per-phase content rendering (Phase 1 vs Phase 2 etc.) is dispatched via a switch on `currentPhase`. For now, all phases show the same placeholder cards.
5. Header bar with project name, current phase badge, time-to-deadline indicator.

CONSTRAINTS
- Tailwind. Reference DESIGN.md tokens.
- Phase 4 gate visual treatment per §10.3.
- Layout must be responsive enough that the sidebar collapses gracefully on narrow viewports.
- Card components must accept children — they're shells, the content fills in over coming sprints.

VERIFY
- Click a project on the dashboard → workspace loads with the correct project name and phase.
- Phase Navigator highlights the current phase.
- Clicking a backward phase (when before Phase 4) sets currentPhase to that phase.
- Clicking a forward phase shows the locked-tooltip behavior.
- Time-to-deadline shows correctly for projects with deadlines, hides for those without.
- The workspace persists scroll/UI state across navigation within itself.

REPORT
Screenshots of the workspace at Phase 1, Phase 4 (with closed lock — temporary visual test, you can hardcode currentPhase for the screenshot then revert), and a Phase 2 view. Paste the file tree under `frontend/src/screens/workspace/`.
```

**Verification:** Layout matches §9.5's ASCII diagram. Visual identity matches DESIGN.md. Commit.

---

### 2.6 — Prompt Generator Card UI + Clipboard 🤖

**🧠 Planning Mode: NO** *(focused single-card implementation; pattern for the rest)*

```
You are wiring up the Prompt Generator card — where a user clicks "Generate Prompt" and gets a copyable block.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §4.5 (Generation flow) and §9.5 (Prompt Generator section of workspace) and §10.6 (Speed under pressure — copy must be one click).

GOAL
A working Prompt Generator card for `phase_2_problem_analysis`. User clicks Generate → sees the generated prompt in a code block → clicks Copy → flash confirmation. "Show injected context" toggle expands the provenance panel. Recommended LLM is shown next to Copy.

DELIVERABLES
1. `frontend/src/components/workspace/PromptGeneratorCard.tsx` — fills in from 2.5's placeholder. Renders one entry per prompt registered for the current phase (for now, only `phase_2_problem_analysis` exists; the card filters by phase via the registry).
2. Each prompt entry renders:
   - Title (from the PromptDefinition).
   - Recommended-LLM label (from settings; default "Claude").
   - "Generate Prompt" button. On click, calls `generate()` from 2.4 with the current PCD and any runtime inputs (none for this prompt).
   - On success: render a fenced code block with the prompt, a Copy button (uses `navigator.clipboard.writeText`), and a "Show injected context" toggle that expands a panel showing the included sections, their last_updated, and their populated_by_phase.
   - On `precondition_unmet` error: show a clear alert directing the user back to the phase that should have populated the missing data. Disabled CTA with the specific section name in the tooltip.
3. Copy confirmation: brief flash ("Copied!") with a 1-second auto-dismiss.
4. Grill-Me toggle UI is **NOT** in this mini-sprint — it lands in Sprint 6.8. Skip it.

CONSTRAINTS
- One click to copy. No confirmation modal in between.
- The "Show injected context" panel is collapsed by default (transparency by toggle, not by overload).
- DESIGN.md visual language.
- Accessibility: Copy button is a real button, not a div with onClick.

VERIFY
- Empty PCD → Generate is disabled with the precondition tooltip.
- Fill in problem_statement_raw on the PCD via the PCD Viewer (which doesn't exist yet — for this test, use DevTools to call commitPCD with a manually-edited PCD) → Generate enables.
- Generate → see the prompt rendered. Copy → check clipboard.
- Toggle "Show injected context" → see the included sections.
- Paste the generated prompt into Claude — get a sensible response back.

REPORT
Screenshots: card disabled state with tooltip; card with generated prompt and copy success; expanded "Show injected context" panel.
```

**Verification:** End-to-end: edit PCD via DevTools → generate → copy → paste into Claude → get response. Commit.

---

### 2.7 — IntakeDefinition Registry + extract.ts 🤖

**🧠 Planning Mode: YES** *(types + registry + the proxy client in one)*

```
You are implementing the IntakeDefinition registry and the extract.ts client that calls the Cloudflare Worker proxy.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §6 (Intake Parsing) in full. §6.1, §6.2, §6.3, §6.6 are the most relevant.

GOAL
Define the IntakeDefinition type. Stand up an empty registry with one entry: `phase_2_problem_analysis_intake`. Implement extract.ts that POSTs to /api/extract and returns a typed FlashResult.

DELIVERABLES
1. `frontend/src/lib/intake/types.ts` — exports `IntakeDefinition`, `FlashResult`, `IntakeOutcome`, `PostExtractValidator`. Types per §6.2.
2. `frontend/src/lib/intake/registry.ts` — exports `INTAKE_REGISTRY: Record<IntakeId, IntakeDefinition>`. One entry: `phase_2_problem_analysis_intake`. Its `target_schema_fragment` is the `problem_analysis.data` slice from pcd_schema.json (load it dynamically from the schema, don't hand-copy). `target_pcd_path` is `sections.problem_analysis.data`. `confidence_threshold` 0.7. Required fields: per the schema's `required` array on problem_analysis.data. No post_extract_validators yet.
3. `frontend/src/lib/intake/extract.ts` — exports `extract(intakeId, rawInput): Promise<FlashResult>`. Builds the request body per §6.6: `{ intake_id, raw_input, schema, prompt }`. Schema is the IntakeDefinition's target_schema_fragment plus a `_confidence` parallel object per §6.3. Prompt is the boilerplate extraction prompt from §6.3 with `{pasted_user_input}` substituted. POSTs to the Worker URL (read from settings; default to deployed Worker). Parses proxy response into FlashResult.
4. `frontend/src/lib/proxy/client.ts` — thin fetch wrapper. Handles 429 (rate limited) with a clear user-facing error. Handles 502 (Flash error) similarly. Network errors return a typed error.
5. `frontend/src/lib/intake/buildSchema.ts` — utility that takes a JSON Schema fragment and produces the augmented schema with `_confidence` mirror object for Flash to fill. Pure function; unit-tested.

CONSTRAINTS
- TypeScript strict.
- Never call Flash directly from the frontend. Always through the proxy.
- The proxy URL is configurable via Settings (defaults to deployed Worker — you'll add the Settings UI later, for now hardcode the deployed URL with a TODO comment).
- DO NOT wire intake into the UI yet — that's 2.8.

VERIFY
- `pnpm typecheck` passes.
- Unit test for buildSchema: passing the problem_analysis.data fragment produces an augmented schema with parallel `_confidence` keys.
- Manual integration test: from a scratch component, call `extract('phase_2_problem_analysis_intake', '<paste a real problem analysis output here>')` and confirm Flash returns parseable JSON. The console logs the FlashResult.

REPORT
Paste the FlashResult from the manual test. Confirm the data and confidence shapes match expectations.
```

**Verification:** Manual call returns structured data. The proxy isn't logging your input. Commit.

---

### 2.8 — Three-Tier Outcome Handler + Diff Preview + Commit 🤖

**🧠 Planning Mode: YES** *(this is the closing of the loop; multi-step UX with multiple failure modes)*

```
You are closing the prompt-generate-paste-back loop. The Research Intake card accepts a paste, calls extract.ts, processes the three-tier outcome, shows a diff preview, and commits to the PCD.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §6.4 (Three-tier failure handling), §6.5 (Diff preview — never silent writes), §10.2 (Transparency by default), §10.4 (Friction where friction belongs).

GOAL
A working Research Intake card for `phase_2_problem_analysis_intake`. Paste output → click Process → see the three-tier outcome handled correctly → diff preview → user clicks Apply Changes → PCD updates and persists.

DELIVERABLES
1. `frontend/src/lib/intake/handle.ts` — exports `processIntake(result: FlashResult, def: IntakeDefinition): IntakeOutcome`. Exact logic from §6.4.
2. `frontend/src/lib/intake/diff.ts` — exports `computeDiff(currentPCD, intakeDef, extractedData): Diff`. Returns sections being touched, fields being added (green), fields being replaced (yellow with old value), fields being cleared (red). Pure function.
3. `frontend/src/components/workspace/ResearchIntakeCard.tsx` — fills in from 2.5's placeholder. Renders one entry per intake registered for the current phase (for now, only the Phase 2 Problem Analysis intake). Each entry has:
   - A labeled textarea ("Paste the Problem Analysis output from your external LLM").
   - A "Process" button. On click, calls `extract()` then `processIntake()`.
   - On Tier 1 (clean): show diff preview with green/yellow/red coloring. "Apply Changes" CTA. On click, commit to PCD via the workspace store.
   - On Tier 2 (partial): same diff preview but with missing/low-confidence/flagged fields highlighted and inline-editable. User fills gaps. Apply Changes commits.
   - On Tier 3 (failed): manual entry form generated from the schema fragment (use a generic JSON-Schema-to-form component — write a minimal one here; the full one ships in Sprint 3.6, just stub the basic fields you need). Side panel shows the raw paste for reference. "Try Again" button retries with a stricter retry prompt (just append a stricter preamble — keep it simple).
4. `frontend/src/components/DiffPreview.tsx` — reusable component rendering a Diff per §6.5.
5. `frontend/src/lib/pcd/applyDiff.ts` — pure function: takes a PCD and a Diff, returns the new PCD with the changes applied. Used at commit time. Logs to the section's `user_overrides` array if any field was inline-edited (the intake handler annotates the diff with edited fields).

CONSTRAINTS
- The diff preview cannot be skipped per §6.5.
- Apply Changes is a single click for clean extracts (low friction per §10.4).
- The applied PCD must pass Ajv validation BEFORE persistence. If validation fails, surface the errors and DO NOT persist.
- Inline edits in Tier 2 update local state only until Apply Changes commits.

VERIFY
- Generate a Phase 2 Problem Analysis prompt (from 2.6).
- Paste it into Claude, get a real response.
- Paste Claude's response into the intake card.
- Click Process. Observe a clean Tier 1 outcome with a populated diff preview.
- Click Apply Changes. Inspect the PCD via DevTools → confirm `problem_analysis.data` is populated, `is_populated` is true, `last_updated` is recent.
- Reload the app. The data persists.
- Test Tier 2: deliberately paste a stub response missing required fields. Confirm partial-extract UI lets you fill them.
- Test Tier 3: paste random garbage. Confirm the manual entry form appears.

REPORT
Screenshots of: clean diff preview, partial diff with inline edits, failed extract with manual entry form. Paste the resulting PCD's problem_analysis section after a successful commit.
```

**Verification:** Full loop works end-to-end with a real LLM. Reload-survives. Commit.

---

### 2.9 — Dogfood Checkpoint #1 🧑

**Manual task.** Allow ~30 minutes.

You now have one prompt-intake pair working end-to-end. Before replicating across all phases, run it once through your own brain.

1. Create a project for a real or imaginary hackathon. Fill in the problem statement.
2. Generate the Phase 2 Problem Analysis prompt.
3. Run it in your preferred LLM (Claude, Gemini Pro — your call).
4. Paste back. Apply changes.
5. Open the resulting PCD JSON via DevTools. Read it. Is the problem analysis sensible? Did Flash extract it correctly into the schema?
6. **Write down what felt slow, awkward, or unclear.** Keep these notes. You'll reference them in Sprint 6 polish.

**Verification:** You have a 5-line note titled "Dogfood Checkpoint 1 — friction list." If everything felt smooth, that's the note.

**End of Sprint 2.** The core mechanic works. Everything from here is replication and rounding out.

---

## Sprint 3 — All Phases

**Goal:** Replicate the prompt-intake mechanism across every phase. By end of sprint, every prompt and every intake from PROMPTS.md is registered and runnable.

**Maps to spec §11.2 Milestone 3.** I'm splitting M3's first task ("Fill out PromptDefinition + IntakeDefinition registries for all phases") into three mini-sprints because it's too big for one Cursor session.

---

### 3.1 — PromptDefinitions for Phases 1 and 2 🤖

**🧠 Planning Mode: YES** *(many definitions, must align with PROMPTS.md verbatim and §4.3 selection rules)*

```
You are populating the PromptDefinition registry for Phase 1 and Phase 2.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §4.1, §4.3 (selection rules table), §4.7 (Grill-Me eligibility per phase). Read PROMPTS.md and locate every Phase 1 and Phase 2 prompt.

GOAL
Add PromptDefinition entries for every Phase 1 and Phase 2 prompt. Verbatim port of content from PROMPTS.md into template files. Correct context_selectors per §4.3.

DELIVERABLES
For each prompt below, create a `frontend/src/lib/prompts/templates/<id>.ts` exporting role_assignment, task_specification_template, output_template_hint, and constraints. Add a registry entry in `frontend/src/lib/prompts/registry.ts`. Add the id to the `PromptId` union.

Phase 1:
- `phase_1_judge_research`
- `phase_1_rubric_mapping` *(this is the intake-paired prompt for the Rubric Mapping flow; see §7.2)*

Phase 2:
- `phase_2_problem_analysis` *(already done in 2.2 — verify it conforms to the new pattern, refactor if needed)*
- `phase_2_competitive_premortem`
- `phase_2_user_research_prompt_package`

For each:
1. Port the prompt content from PROMPTS.md verbatim.
2. Define context_selectors per §4.3's selection rules table. Where the table doesn't list a prompt explicitly, infer the right selectors from the prompt's content needs.
3. Set grill_me_modes correctly per §4.7 (which prompts are Grill-Me-eligible in v1.5).
4. Set produces_intake_id (the matching intake will be added in Sprint 3.4).

CONSTRAINTS
- VERBATIM port from PROMPTS.md. If you find any tension between PROMPTS.md and the spec, side with PROMPTS.md and add a comment flagging the discrepancy.
- Each template file is self-contained — no cross-imports between templates.
- Type-check after each entry. Don't accumulate errors.

VERIFY
- `pnpm typecheck` passes.
- The Prompt Generator card for Phase 1 (when current_phase=1) lists the Phase 1 prompts.
- The Prompt Generator card for Phase 2 lists the Phase 2 prompts including the existing `phase_2_problem_analysis`.
- Each can be generated against a sufficiently-populated PCD and produces sensible output when copied to a real LLM.

REPORT
List every PromptDefinition added with its produces_intake_id and grill_me_modes.
```

**Verification:** Phase 1 and Phase 2 cards each list multiple prompts. Each generates cleanly. Commit.

---

### 3.2 — PromptDefinitions for Phase 3 🤖

**🧠 Planning Mode: YES** *(Phase 3 has the most prompts and the most complex ordering rules)*

```
You are populating the PromptDefinition registry for Phase 3 — the most complex phase.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §4.5 (Phase 3 ordering enforcement). Read PROMPTS.md for all Phase 3 prompt content.

GOAL
Add PromptDefinitions for Step 3a–3e and the downstream prompts. Implement the Step 3e gate that disables it until 3b/3c/3d have produced data.

DELIVERABLES
1. PromptDefinitions for:
   - `phase_3_idea_seed_challenge` — Grill-Me REQUIRED (interactive | auto). Has a runtime_input `seed_idea_text` (textarea, required, max_length 2000).
   - `phase_3_scamper`
   - `phase_3_jtbd`
   - `phase_3_brainwriting`
   - `phase_3_divergent_candidates` — has runtime_inputs `candidate_count_min` (integer, default 3, min 3), `candidate_count_max` (integer, default 5, max 7), `archetypes_enforced` (boolean, default true). Grill-Me OPTIONAL.
   - `phase_3_differentiation_stress_test`
   - `phase_3_downselection_verification`
2. Step 3e gate: in the generate function (you may need to extend it), check that at least one of `ideation_record.data.scamper_outputs`, `ideation_record.data.jtbd_analysis`, or `ideation_record.data.brainwriting_round` is populated before allowing `phase_3_divergent_candidates` to generate. Return `precondition_unmet` if not. The Prompt Generator card displays this as a disabled CTA with the missing precondition tooltip per §4.5.
3. Phase 3 ordering in the Prompt Generator card display: 3a (only when seed exists), 3b, 3c, 3d, 3e, downstream. Implement an ordering hint on each PromptDefinition (`display_order: number`) and sort the cards by it.
4. Verbatim port from PROMPTS.md.

CONSTRAINTS
- The Step 3e gate is enforced both at generation (returning precondition_unmet) AND at the UI level (disabled CTA with tooltip).
- Step 3a's Grill-Me is REQUIRED — the toggle UI in Sprint 6.8 should not show "Off" for this prompt. Note this in a code comment for whoever builds the toggle.

VERIFY
- All Phase 3 cards visible at current_phase=3 in the workspace.
- Step 3e is disabled until any of 3b/3c/3d data exists.
- Generating Step 3e after 3b is populated works.
- Generating Step 3a with a seed text in the runtime input produces a valid Grill-Me Interactive prompt.

REPORT
Confirm each gate behavior with screenshots.
```

**Verification:** Phase 3 cards stack in the right order. The Step 3e gate behaves correctly.

---

### 3.3 — PromptDefinitions for Phases 4, 5, 6, 7 + Addendum 🤖

**🧠 Planning Mode: YES** *(many definitions; Phase 4 has its own ordering acknowledgment)*

```
You are completing the PromptDefinition registry — Phases 4, 5, 6, 7, and the Addendum prompt.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §4.5 (Phase 4 ordering — Skip Design Grill acknowledgment). Read PROMPTS.md for all remaining prompt content.

GOAL
Round out the registry with every remaining prompt. Implement the Phase 4 Skip Design Grill acknowledgment.

DELIVERABLES
1. PromptDefinitions for:
   - Phase 4: `phase_4_design_grill` (Grill-Me REQUIRED), `phase_4_product_brief`, `phase_4_tid`, `phase_4_demo_script`.
   - Phase 5: typically has no LLM prompts (it's the build phase) — verify against PROMPTS.md and add any that exist (e.g., a build-checklist generation prompt).
   - Phase 6: `phase_6_pitch_storyboard`, `phase_6_pitch_review`, `phase_6_qa_prep`, `phase_6_judge_qa_drill` (Grill-Me OPTIONAL).
   - Phase 7: `phase_7_debrief`.
   - `addendum` — uses runtime_inputs for `build_state_snapshot`, `proposed_change_description`, `user_justification` per §4.1's Addendum example. Not Grill-Me eligible.
2. Phase 4 ordering enforcement per §4.5: when the user generates any of `phase_4_product_brief`, `phase_4_tid`, or `phase_4_demo_script` BEFORE running `phase_4_design_grill` for this project, surface the one-time "Skip Design Grill" inline acknowledgment checkbox. After the user checks it, log a `phase_4_design_grill_skipped` Phase Gate Log event (one per project, not per prompt) and never show the checkbox again for this project.
3. Verbatim port from PROMPTS.md.

CONSTRAINTS
- The Skip Design Grill acknowledgment is a checkbox INLINE above the Generate Prompt button, not a modal.
- It does not block generation — checking it just unlocks generation.
- Once acknowledged, it's persisted in the PCD (or the Phase Gate Log) so it doesn't reappear.

VERIFY
- All cards appear at correct phases.
- The Skip Design Grill acknowledgment shows correctly the first time, and never again after acknowledgment.
- The Addendum prompt's runtime_inputs render correctly as form fields.
- Generating any of these against a sufficiently-populated PCD produces a sensible LLM output.

REPORT
Confirm each phase's prompt count matches PROMPTS.md.
```

**Verification:** Every phase has its prompts registered. Commit.

---

### 3.4 — IntakeDefinitions for All Phases 🤖

**🧠 Planning Mode: YES** *(many intakes, each with a target schema fragment from pcd_schema.json)*

```
You are populating the IntakeDefinition registry for all phases.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §6.2 (IntakeDefinition shape) and §6.4 (three-tier handling). Read pcd_schema.json to identify the target schema fragments.

GOAL
For every PromptDefinition that has a `produces_intake_id`, ensure a matching IntakeDefinition exists in the registry. Each correctly maps to a target_pcd_path and target_schema_fragment.

DELIVERABLES
1. IntakeDefinitions for every produces_intake_id used in the prompt registry. List them by phase:
   - Phase 1: `judge_research_intake`, `rubric_mapping_intake`.
   - Phase 2: `problem_analysis_intake` (already exists), `competitive_premortem_intake`, `user_research_intake`.
   - Phase 3: `idea_seed_challenge_intake`, `scamper_intake`, `jtbd_intake`, `brainwriting_intake`, `divergent_candidates_intake` (with post-extract enrichment per §6.2 — assign ULID, set is_remix=false, remix_source_ids=[]), `differentiation_stress_test_intake`, `downselection_verification_intake`.
   - Phase 4: `design_grill_intake`, `product_brief_intake`, `tid_intake`, `demo_script_intake`.
   - Phase 6: `pitch_storyboard_intake`, `pitch_review_intake`, `qa_prep_intake`, `judge_qa_drill_intake` (with replace-or-supplement choice per §6.5).
   - Phase 7: `debrief_intake`.
   - `addendum_intake` — appends to `pcd.addenda[]` (special handling per §6.2; merges runtime form inputs with extracted content).
2. For each, target_schema_fragment is a slice of pcd_schema.json — use a helper that loads the schema once and exposes named slices (`getSchemaFragment(path)`).
3. Confidence threshold: default 0.7 for most; 0.6 for highly subjective extractions (judge personas, competitive premortem); document the choice per intake in a comment.
4. Required fields: read from the schema fragment's `required` array.
5. Post-extract enrichment for `divergent_candidates_intake` per §6.2.
6. Special addendum intake handling per §6.2.

CONSTRAINTS
- The schema fragment is loaded dynamically; do not duplicate schema content in TypeScript.
- TypeScript strict.

VERIFY
- For every prompt's produces_intake_id, an IntakeDefinition exists.
- For 2-3 intakes (pick a Phase 3 and a Phase 4 one), exercise the full loop: generate → run in real LLM → paste back → confirm clean Tier 1 outcome → apply changes → confirm PCD updated correctly.

REPORT
List every IntakeDefinition with its target_pcd_path and confidence_threshold. Confirm dogfood test results for the chosen 2-3 intakes.
```

**Verification:** Every prompt has a matching intake. Spot-checked end-to-end. Commit.

---

### 3.5 — Phase Briefings + Completion Checklists 🤖

**🧠 Planning Mode: NO** *(content-heavy; mostly transcribing the spec into TSX)*

```
You are filling in the Phase Briefing and Completion Checklist cards for every phase.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §9.5 and STARTUP_WORKFLOW_AND_APP_OVERVIEW.md (each phase's "Phase Output" and "Phase Completion Requirement" sections — these are the source of checklist content).

GOAL
For each phase 1-7, populate the PhaseBriefingCard with directive copy and the CompletionChecklistCard with the actual completion requirements derived from the workflow doc.

DELIVERABLES
1. `frontend/src/lib/phaseContent/briefings.ts` — exports a `Record<PhaseNumber, PhaseBriefing>` where PhaseBriefing has `title`, `purpose` (1-2 sentences), `directive_actions` (string[]), `output_summary`. Source the content from the workflow overview's per-phase sections. Use prescriptive voice per §10.1.
2. `frontend/src/lib/phaseContent/checklists.ts` — exports a `Record<PhaseNumber, ChecklistItem[]>`. Each ChecklistItem has `label`, `is_required: boolean`, and a `is_complete: (pcd: PCD) => boolean` predicate. Predicates are pure functions of the PCD state. Examples:
   - Phase 1: "Hackathon profile complete" → `pcd.sections.hackathon_profile.is_populated`.
   - Phase 2: "Competitive Pre-mortem completed" (REQUIRED, hard block) → `pcd.sections.competitive_landscape.is_populated`.
   - Phase 3: "Step 3b SCAMPER completed" → `pcd.sections.ideation_record.data.scamper_outputs?.length > 0`.
3. Wire up `PhaseBriefingCard` and `CompletionChecklistCard` to render from these. The checklist shows ✓ for complete items, an empty box for incomplete required items (orange/red accent), an empty box for incomplete optional items (muted accent).
4. The "Advance to next phase" CTA at the bottom of the workspace is enabled only when all required checklist items for the current phase are complete. Hover state on a disabled CTA shows a tooltip listing missing required items per §10.1.

CONSTRAINTS
- Predicates are pure functions of the PCD. No async, no DOM access.
- Prescriptive voice. Every directive_action is an imperative.
- The Phase 2 Competitive Pre-mortem checklist item has a visible "REQUIRED — hard block" treatment per the spec's Phase 2 hard-block rule.

VERIFY
- For each phase, the briefing and checklist visibly reflect actual PCD state.
- Toggling a section's data via DevTools updates the checklist in real time.
- Advance CTA is disabled when required items are missing; enabled when all are complete.
- Tooltip on disabled Advance CTA names the missing items.

REPORT
Screenshots of: Phase 1 workspace, Phase 2 workspace with Competitive Pre-mortem incomplete (CTA disabled with tooltip), Phase 3 workspace with several checklist items in progress.
```

**Verification:** Phase 2 hard block is visible and enforced. Commit.

---

### 3.6 — State Machine: Transitions, Completion Gates, Advance CTA Logic 🤖

**🧠 Planning Mode: YES** *(load-bearing logic; mistakes here cascade)*

```
You are implementing the state machine logic that controls phase transitions.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §5 (State Machine & Reconciliation) — sections on ALLOWED_TRANSITIONS and PhaseCompletionRequirement specifically. The Phase 4 gate is sacred per §10.3.

GOAL
Implement ALLOWED_TRANSITIONS, the per-phase completion gate logic, and the Advance CTA. Ban forward motion past Phase 4 by anything other than the Addendum Protocol (which lands in 4.6). For now, just enforce that Phase 4 is one-way: once `phase_4_gate_crossed` is true, no backward transitions through it.

DELIVERABLES
1. `frontend/src/lib/stateMachine/transitions.ts` — exports `ALLOWED_TRANSITIONS`, `canTransition(fromPhase, toPhase, pcd): { ok: boolean, reason?: string }`. Rules:
   - Forward: `n → n+1` only if Phase n's required completion gates pass.
   - Backward: `n → m` where m < n, ALLOWED for all phases EXCEPT cannot pass through Phase 4 once `phase_4_gate_crossed`.
   - Phase 4 completion sets `phase_4_gate_crossed: true` permanently.
2. `frontend/src/lib/stateMachine/completion.ts` — exports `getCompletionStatus(phase, pcd): { complete: boolean, missing: string[] }`. Uses the checklist predicates from 3.5.
3. `frontend/src/components/workspace/AdvanceCTA.tsx` — bottom-of-workspace CTA. Disabled with tooltip listing missing items if completion fails. Enabled and primary when all gates pass. For Phase 4, the CTA reads "Cross the Phase 4 Gate" with different visual treatment per §10.3 — typed-phrase confirmation modal on click. After phase 4 advance, set `phase_4_gate_crossed: true` in the PCD.
4. Backward navigation via Phase Navigator (already exists from 2.5) calls `canTransition` and rejects with a tooltip if disallowed.
5. Override flow: for non-Phase-4 phases, if completion gates fail and the user wants to advance anyway, surface an "Override" link in the disabled CTA's tooltip. Clicking it opens a modal requiring a written justification (min 80 chars). On submit, log a `phase_completion_overridden` event in the Phase Gate Log (the in-memory log from 2.4 — persistence comes in 4.3).

CONSTRAINTS
- Phase 4 advance has typed-phrase confirmation: user types "lock scope" or similar exact phrase per §10.4.
- Phase 4 advance is irreversible. After it's set, no backward transitions through it. Period.
- Override of any non-Phase-4 phase is allowed but logged.

VERIFY
- Phase 1 → 2 only when Phase 1 gates pass.
- Backward Phase 3 → 2 works.
- Phase 4 → 5 has typed-phrase confirmation.
- After Phase 4 cross, attempting to navigate back to Phase 3 fails with a clear message.
- Override path on Phase 2 (despite Competitive Pre-mortem being incomplete) requires justification and logs the event.

REPORT
Screenshots: phase 4 typed-phrase confirmation modal, override modal with justification, Phase Navigator after Phase 4 crossed (locked).
```

**Verification:** Phase 4 gate is unbreakable. Backward navigation works for non-gated phases. Commit.

---

### 3.7 — PCD Viewer (Read-Only Render) 🤖

**🧠 Planning Mode: YES** *(rendering the entire PCD; needs care)*

```
You are building the PCD Viewer — the read-only full-page view of the current PCD.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §9.5 (PCD Viewer references), §10.5 (PCD as spine — surface it everywhere). Read §2 (PCD overview).

GOAL
A full-page route at `/projects/:id/pcd` that renders the entire PCD with collapsible sections, provenance metadata per section, and section-level Edit buttons (the editor lands in 3.8).

DELIVERABLES
1. `frontend/src/screens/pcd/PCDViewer.tsx` — full page. Header with project name, phase badge, Export button (writes markdown via `renderPCDToMarkdown`, see point 3 below; saves as `<project_name>.md`). Side TOC of all sections. Main column renders each section as a collapsible card.
2. Each section card shows:
   - Section name.
   - Phase badge (which phase populated it).
   - "Last updated" timestamp.
   - Stale-flag if `is_stale === true` with the `stale_reason` (the staleness propagation lands in 4.1; for now, the field will always be false, but render it correctly when set).
   - The data, rendered per its shape. For markdown-string fields (e.g., `team_idea_seed.original_text`), render as markdown via `marked`. For structured fields, render as a definition list or a simple table. Use a pattern of "section type → renderer function" so adding a new renderer is trivial.
   - An "Edit" button (stub — opens the modal that lands in 3.8).
3. `frontend/src/lib/pcd/render.ts` — implements `renderPCDToMarkdown(pcd: PCD): string`. Walks the PCD in section order and produces a clean markdown export. The export ends with the embedded JSON block per §2.1. Pure function, deterministic.
4. The PCD Viewer is reachable from the workspace via a "Show PCD" affordance in the header per §10.5.

CONSTRAINTS
- The entire viewer is read-only. Edit happens in the modal from 3.8.
- Long sections collapse by default. The user expands what they want to see.
- Vault references (which arrive in Sprint 5) get clickable treatment — for now, render any `vault_ref:<id>` strings as plain text with a TODO marker.

VERIFY
- The viewer shows every populated section.
- Empty sections render as "Not yet populated" with prescriptive copy directing to the relevant phase.
- Export download produces a well-formed markdown file ending with the JSON block.
- The exported file's JSON block parses back to the original PCD when copy-pasted.

REPORT
Screenshots: PCD viewer with several sections populated, export download verified.
```

**Verification:** Export the PCD, open the markdown file, confirm it's readable and the JSON block at the end parses. Commit.

---

### 3.8 — Section Editor (Generic JSON-Schema-to-Form) 🤖

**🧠 Planning Mode: YES** *(reusable component used by Section Editor AND Tier 2 partial-extract review)*

```
You are building the generic JSON-Schema-to-form renderer and the Section Editor modal that uses it.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §9.15 (Section Editor) and §6.4 (Tier 2 partial extract — same component is used there).

GOAL
A reusable component `SchemaForm` that takes a JSON Schema fragment + initial value + onSubmit callback and renders an editable form. The Section Editor modal wraps it for the PCD Viewer's Edit buttons. Refactor the Tier 2 inline-edit form from 2.8 to use this component.

DELIVERABLES
1. `frontend/src/components/SchemaForm.tsx` — the renderer. Supports:
   - `string` (textarea or input depending on `format`/`maxLength`)
   - `integer` / `number` (number input with min/max validation)
   - `boolean` (checkbox)
   - `enum` (dropdown)
   - `array` of strings (chip input or comma-separated textarea)
   - `array` of objects (repeated nested form rows with add/remove)
   - `object` (nested SchemaForm — recursion)
   - `oneOf` / `anyOf` (radio selector for variant + nested form for the chosen variant — needed for the `Adjustment` shape per §6.2)
   - Reads `description` from each property and shows it as inline help text.
   - Validates against the schema on submit (Ajv).
2. `frontend/src/screens/pcd/SectionEditor.tsx` — the modal. Renders SchemaForm. On save, validates against the section's schema fragment, applies a diff preview (reuse DiffPreview from 2.8), commits via the workspace store.
3. Refactor 2.8's Tier 2 inline-edit to use SchemaForm with the field paths flagged for editing.

CONSTRAINTS
- The form is not exhaustive — it supports the field types actually used in pcd_schema.json. If it encounters an unsupported field type, render a JSON-textarea fallback with validation on save.
- Logs to `user_overrides` for every field touched per the SectionEnvelope contract.
- Do not break 2.8's tier-2 flow — verify it still works after refactor.

VERIFY
- Open the PCD Viewer. Click Edit on the `hackathon_profile` section. Modify a field. Save. Confirm the change persists and `user_overrides` records it.
- Re-test 2.8's tier-2 partial extract flow — still works.
- Try editing an array-of-objects section (e.g., `judge_intelligence.data.judge_personas`). Add, remove, reorder rows.

REPORT
Screenshots: Section Editor for hackathon_profile, Section Editor for judge_personas with array editing, post-save with user_overrides visible in DevTools.
```

**Verification:** Edit any section through the UI; persistence works; user_overrides logged. Commit.

**End of Sprint 3.** Every prompt and intake is registered. Every phase has its briefing, checklist, transitions, and editing affordances. The PCD is fully editable.

---

## Sprint 4 — Reconciliation and the Phase 4 Gate

**Goal:** Backward navigation correctly propagates staleness. The Phase 4 gate, scope-lock confirmation, exports, and Addendum Protocol all work.

**Maps to spec §11.2 Milestone 4.**

---

### 4.1 — Dependency Graph + propagateStaleness 🤖

**🧠 Planning Mode: YES** *(graph algorithm + side-effect propagation; subtle bugs here destroy trust)*

```
You are implementing the staleness propagation logic.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §5.2 (PCD_DEPENDENCY_GRAPH) and §5.3 (propagateStaleness algorithm).

GOAL
A pure function `propagateStaleness(pcd, changedSection): PCD` that walks the dependency graph and marks every downstream section is_stale=true with a clear stale_reason. Wired into the workspace store: every commit triggers propagation.

DELIVERABLES
1. `frontend/src/lib/stateMachine/dependencies.ts` — exports `PCD_DEPENDENCY_GRAPH: Record<SectionId, SectionId[]>` per §5.2. Each entry maps a section to its downstream dependents.
2. `frontend/src/lib/stateMachine/reconcile.ts` — exports `propagateStaleness(pcd, changedSectionId): PCD`. Walks the graph BFS from the changed section. For each visited downstream section that is currently `is_populated`, sets `is_stale=true` with `stale_reason: "Upstream section <name> was updated on <timestamp>"`. Handles cycles gracefully (the graph should be acyclic, but defend against it).
3. Wire propagation into `workspace.commitPCD()`: detect which sections changed (compare old vs new PCD), call propagateStaleness for each changed section, persist the resulting PCD.
4. Backward navigation via Phase Navigator triggers a reconciliation check: it identifies sections populated AFTER the target phase and surfaces a confirmation modal asking the user whether to mark them stale or keep them current. User choice is recorded.
5. Unit tests for propagateStaleness covering: single change with one downstream, cascading change, no-op when downstream is already stale, no-op when downstream isn't populated yet.

CONSTRAINTS
- Pure function. Returns a new PCD; never mutates the input.
- Stale propagation is automatic on commit; the user explicitly confirms it on backward navigation.
- The stale_reason is human-readable per §10.2.

VERIFY
- Edit `hackathon_profile` after Phase 2 has populated `problem_analysis` → `problem_analysis.is_stale` becomes true with a clear reason.
- Backward-navigate from Phase 3 to Phase 1 → reconciliation modal appears listing affected sections.
- All unit tests pass.

REPORT
Paste the staleness reason text for the hackathon_profile → problem_analysis case. Paste test output.
```

**Verification:** Edit upstream, see downstream go stale. Commit.

---

### 4.2 — Stale Resolution UI 🤖

**🧠 Planning Mode: YES** *(three resolution paths, each with its own implications)*

```
You are building the Stale Resolution UI.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §5.4 (Stale resolution flows) and §10.5 (Stale sections visually distinct everywhere).

GOAL
For each stale section, surface three resolution options: Regenerate (re-run the prompt that produced it), Confirm-Valid (mark not-stale without changes), or Manual Edit (open Section Editor).

DELIVERABLES
1. `frontend/src/components/StaleSectionBanner.tsx` — appears at the top of any stale section in the PCD Viewer and at the top of the Workspace Body when current_phase has stale sections. Shows the stale_reason and the three CTAs.
2. The "Regenerate" CTA opens the Prompt Generator card for the section's source prompt, pre-scrolled into view, with a small note: "Regenerating <section> — its current data will be replaced when you commit the new intake."
3. "Confirm Valid" — single click. Sets is_stale=false, adds a user_override entry noting the confirmation. Logs `stale_resolved_confirm_valid` in the Phase Gate Log.
4. "Manual Edit" — opens the Section Editor (3.8). On save, sets is_stale=false.
5. Phase Navigator stale-count badge wires up: counts sections with is_stale=true and shows a count badge on the sidebar (per §9.5).
6. The PCD Viewer renders stale sections with a distinct visual treatment per §10.5 (e.g., yellow left border, "Stale" pill badge).

CONSTRAINTS
- Stale resolution is one of three explicit user actions; do not auto-resolve.
- Visual treatment must be unmistakable — staleness is a primary signal.

VERIFY
- Cause a stale propagation. Open the PCD Viewer. Stale section is visibly distinct.
- Try each resolution path. Each works, each logs the right Phase Gate Log event.
- Phase Navigator stale-count badge updates correctly.

REPORT
Screenshots of: stale banner with three CTAs, stale section visual treatment in PCD Viewer, Phase Navigator with stale-count badge.
```

**Verification:** All three resolution paths work. Commit.

---

### 4.3 — Phase Gate Log Persistence + Viewer 🤖

**🧠 Planning Mode: NO** *(persistence + a read-only screen)*

```
You are persisting the Phase Gate Log and building the viewer.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §5.5 (Phase Gate Log) and §9.14 (Phase Gate Log Viewer).

GOAL
Move the Phase Gate Log from the in-memory Zustand store (where it landed in 2.4 and 3.6) into the PCD's `phase_gate_log` array, persisted on every event. Build the read-only viewer.

DELIVERABLES
1. Refactor: every place currently logging an event to the in-memory store now appends an entry to `pcd.phase_gate_log[]` and commits via the workspace store. Verify all event types still work.
2. `frontend/src/screens/phaseGateLog/PhaseGateLogViewer.tsx` — implements §9.14. Filter bar (event type, phase, date range), chronological list, expanded details with payload + user_justification. Export-to-CSV button.
3. Reachable from the workspace header.

CONSTRAINTS
- Read-only. No editing of log entries.
- Append-only. Existing entries are never mutated.
- CSV export is plain text, no library — just join with commas/newlines.

VERIFY
- Existing log events from 2.4, 3.6, 4.1, 4.2 all flow into `pcd.phase_gate_log[]`.
- Reload — the log persists.
- Filter by event type works.
- CSV export downloads.

REPORT
Paste a sample of 5 log entries from a real test session.
```

**Verification:** Commit.

---

### 4.4 — Phase 4 Gate Logic + Scope-Lock Confirmation 🤖

**🧠 Planning Mode: YES** *(the most important UX moment in the app)*

```
You are finalizing the Phase 4 gate. Most of the logic exists from 3.6 — this mini-sprint is the polish, the visual treatment, and the one-way enforcement.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §5.1 (Phase 4 gate) and §10.3 (Phase 4 gate is sacred) and §10.4 (high-friction moments).

GOAL
The Phase 4 advance is unmistakable, irreversible, and recorded. Visual treatment per §10.3 is fully implemented.

DELIVERABLES
1. Phase 4 advance modal: full-screen dimmed overlay. Headline: "Cross the Phase 4 Gate". Body explains: scope is about to lock, post-gate changes require the Addendum Protocol, all three Phase 4 documents are about to be marked Definitive. Typed-phrase confirmation: user types `lock scope` exactly. Submit button activates only when the typed phrase matches.
2. On submit:
   - Set `pcd.phase_4_gate_crossed: true` and `pcd.phase_4_gate_crossed_at: <ISO timestamp>`.
   - Log `phase_4_gate_crossed` event with full payload.
   - Mark Phase 4 nav node visually locked per §10.3.
   - Surface a one-time toast: "Scope is locked. Phase 6 (Pitch Prep) is now available in parallel with Phase 5 (Build)." with a "Got it" dismiss.
3. After cross: any `canTransition` call attempting backward transition through Phase 4 returns `{ ok: false, reason: 'Phase 4 gate has been crossed. Use the Addendum Protocol for scope changes.' }`. Visible in tooltips.
4. Phase Navigator: Phase 4 lock icon visibly closes (animate the transition if you want; not required). Lock icon is unmistakable.
5. After cross, the Workspace Body for any phase before 4 is read-only — no Generate Prompt buttons, no Edit affordances. Banner explains scope is locked.

CONSTRAINTS
- Typed-phrase exact match: lowercase "lock scope". Whitespace trimmed.
- The advance is irreversible. Resist any urge to add an "undo" path.
- Pre-Phase-4 phases become read-only after cross. New events still log (e.g., a stale-resolution from Phase 6 won't actually edit Phase 1).

VERIFY
- Walk through a project to Phase 4. Cross the gate. Confirm the visual treatment.
- After cross, attempt to navigate to Phase 1 → tooltip shows the gate message.
- After cross, Phase 1 workspace shows read-only banner.
- Phase Gate Log shows the cross event.

REPORT
Screenshots of: typed-phrase confirmation modal, Phase Navigator after cross with locked icon, Phase 1 workspace in read-only mode, Phase Gate Log entry.
```

**Verification:** The gate is unmistakable and irreversible. Commit.

---

### 4.5 — Document Export Center 🤖

**🧠 Planning Mode: NO** *(focused screen with download buttons)*

```
You are building the Document Export Center.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §9.9 (Document Export Center).

GOAL
A dedicated route at `/projects/:id/exports` listing the three Phase 4 outputs (Product Brief, TID, Demo Script) and the full PCD as downloadable markdown files.

DELIVERABLES
1. `frontend/src/screens/exports/DocumentExportCenter.tsx` — implements §9.9. Card per artifact with last-updated timestamp, Download button, Copy-to-clipboard button, Preview link (opens the relevant section of the PCD Viewer).
2. Helper functions:
   - `exportProductBrief(pcd): string` — renders only the Product Brief section as standalone markdown.
   - `exportTID(pcd): string` — TID rendering. Includes any addenda appended at the bottom of the file under a clear "## Addenda" heading.
   - `exportDemoScript(pcd): string` — demo script + backup plan.
   - `exportFullPCD(pcd): string` — already exists from 3.7's renderPCDToMarkdown.
3. Each download triggers a browser file save with sensible filename: `<project>-product-brief.md`, etc.
4. "Cursor handoff" note per §9.9.

CONSTRAINTS
- All exports are markdown. No PDF generation.
- Files are saved client-side via Blob + URL.createObjectURL.
- The TID export includes addenda explicitly per the spec.

VERIFY
- Generate the three Phase 4 outputs end-to-end. Cross the gate. Open exports.
- Download each file. Open in a markdown viewer. Confirm the content is well-formed.
- Add a sample addendum (you'll fully test this after 4.6); confirm it appears in the TID export.

REPORT
Confirm each artifact downloads. Paste a snippet of one downloaded markdown file.
```

**Verification:** Markdown files open cleanly in any viewer. Commit.

---

### 4.6 — Addendum Protocol Interface 🤖

**🧠 Planning Mode: YES** *(post-gate-only, friction-heavy, custom flow)*

```
You are building the Addendum Protocol interface — the only way to change scope after the Phase 4 gate.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §9.13 (Addendum Protocol Interface), §6.2 (Addendum intake specific handling), §4.1 (Addendum PromptDefinition example), Workflow doc on the Addendum Protocol.

GOAL
A route reachable only after Phase 4 gate is crossed, with a friction-heavy form, addendum prompt generation, intake handling, and append-to-PCD wiring.

DELIVERABLES
1. `frontend/src/screens/addendum/AddendumProtocolInterface.tsx` — implements §9.13. Reachable only when `phase_4_gate_crossed: true`. Otherwise routes back to dashboard with a tooltip.
2. Form: build-state textarea (paste of Cursor markdown), proposed change description (min 200 chars), justification (min 80 chars). All required.
3. "Generate Addendum Prompt" CTA: calls `generate('addendum', pcd, runtimeInputs)` with the form values as runtime_inputs. Surfaces the generated prompt in a copyable block with a Copy button.
4. Below the prompt: paste-back textarea + "Process" button. Calls extract for `addendum_intake`. Three-tier outcome handling (existing component reused).
5. On clean intake: shows preview of the addendum content. "Append to Project" CTA. On click, appends to `pcd.addenda[]` (the addendum_intake's special handling per §6.2 mints a ULID, sets created_at, attaches user_justification and build_state_snapshot, merges the LLM-extracted body).
6. Below the form: list of existing addenda for this project, each with timestamp, justification preview, and a click-to-expand body. Read-only — addenda are never edited.
7. Phase Gate Log: appending an addendum logs an `addendum_appended` event with the addendum_id and justification.

CONSTRAINTS
- Per §10.4, this is high-friction: large warning banner up top, all fields required, justification min 80 chars enforced.
- The addendum is appended to the TID, never inlined. The TID export from 4.5 surfaces them under a separate Addenda heading.
- Once appended, an addendum is permanent. No edit, no delete.

VERIFY
- Cross the Phase 4 gate. Navigate to the Addendum Protocol.
- Submit the form. Generate the prompt. Run it in Claude with a fake build state.
- Paste the response. Process. Apply.
- Open the TID export — addendum appears at the bottom.
- Reload — addendum persists.

REPORT
Screenshots: warning banner, full form, generated prompt, addendum preview, TID export with addendum appended.
```

**Verification:** Full addendum flow works end-to-end. Commit.

**End of Sprint 4.** The state machine is robust. Phase 4 is sacred. Documents export. Addenda work. Run **Dogfood Checkpoint #2** now.

---

### 4.7 — Dogfood Checkpoint #2 🧑

**Manual task.** Allow ~2 hours.

Pick a real past hackathon problem statement (use one from your DOST or other event). Walk through Phase 1 → Phase 4 fully:

1. Create the project. Fill in everything you know about that hackathon.
2. Generate, paste, intake every Phase 1 prompt.
3. Same for Phase 2.
4. Same for Phase 3 (skip 3a since you're testing the standard path).
5. Same for Phase 4. Cross the gate.
6. Export the three Phase 4 documents. Open them in Cursor as if you were about to build.
7. Test the addendum flow with a fake change request.
8. Backward-navigate from Phase 3 → Phase 2 mid-test, edit a section, watch staleness propagate.

**Write a friction-list note titled "Dogfood Checkpoint 2."** This is your real readiness signal — if Phase 4 outputs aren't usable in Cursor, fix that before anything else.

**Verification:** You have actual exported documents that you would feed Cursor for a real hackathon. They look like they'd work.

---

## Sprint 5 — Vault

**Goal:** The persistent cross-project infrastructure works. Auto-queries surface relevant entries during project creation and other key moments.

**Maps to spec §11.2 Milestone 5.**

---

### 5.1 — Vault Top-Level UI 🤖

**🧠 Planning Mode: YES** *(three tabs, each with list / search / tags / sort)*

```
You are building the Vault's top-level UI.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §3 (The Vault) and §9.10 (Vault — Top Level).

GOAL
Three-tabbed Vault screen (Judges & Organizers, Winning Solutions, Boilerplate Library) with search, tag filtering, sort, and Add-new CTAs.

DELIVERABLES
1. `frontend/src/screens/vault/VaultTopLevel.tsx` — implements §9.10. Tabs as URL routes: `/vault/judges`, `/vault/solutions`, `/vault/boilerplate`. Future tab "Client Vault" disabled with stretch-goal tag.
2. `frontend/src/components/VaultEntryList.tsx` — generic list view shared across tabs. Takes a vault store, a column config, and renders search box, tag chip filters, sort dropdown, list of entries. Click entry → entry detail (5.2).
3. `frontend/src/stores/vault.ts` — three Zustand stores (one per Vault type) backed by the corresponding Dexie tables from 1.1. Each exposes: `entries`, `loadAll()`, `search(query, tags)`, `add(entry)`, `update(id, entry)`, `delete(id)`. Encrypt content fields (notes, descriptions); leave indexed fields plaintext per 1.1's data model.
4. Tag input is a chip control: type, comma-separate, enter to add. Tag filtering is multi-select chips.

CONSTRAINTS
- Search is keyword-based, runs against the indexed plaintext fields. Tag filter is exact-match.
- Adding an entry is via a "+ Add new" button at the top of the list, opening the entry detail screen in create mode.

VERIFY
- All three tabs load.
- Add an entry to each. Search finds it. Tag filtering filters correctly.
- Reload — entries persist.

REPORT
Screenshots of each tab with sample entries.
```

**Verification:** Commit.

---

### 5.2 — Vault Entry Detail / Edit Screens 🤖

**🧠 Planning Mode: YES** *(one screen handles all three types via varying field configs)*

```
You are building the Vault entry detail and edit screen.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §9.11 (Vault Entry Detail / Edit).

GOAL
A single screen at `/vault/:type/:id` (and `/vault/:type/new` for create) that renders the appropriate form fields per entry type, supports edit and save, and lists projects that reference this entry.

DELIVERABLES
1. `frontend/src/screens/vault/VaultEntryDetail.tsx` — implements §9.11. Field configs per type:
   - All: id (read-only when editing), name, tags, created/updated timestamps, free-form notes textarea.
   - Judge/Organizer: type (dropdown: Judge / Organizer / Sponsor), organization, professional background (textarea), events appeared at (array of strings), observed tendencies (textarea).
   - Winning Solution: event, organizer, domain, year, summary, why-it-won, source URL.
   - Boilerplate: component name, category (dropdown: frontend-ui / backend / ai-integration / deployment), description, used-in events (array), what worked (textarea), what didn't (textarea), repo pointer (URL).
2. Reuse SchemaForm from 3.8 by defining a JSON Schema fragment per type and passing it in.
3. References panel: list of projects that reference this Vault entry. References are stored on the project side via `vault_ref:<id>` strings in PCD sections that pull Vault content (Phase 1 surfaces, Phase 4 boilerplate refs). For now, scan all projects' PCDs for matching `vault_ref:` strings — slow but correct for MVP.
4. Save / Delete CTAs. Delete requires typed-phrase confirmation per §10.4. Delete cascades by clearing references in PCDs and logging `vault_entry_deleted_with_references` events to affected projects' Phase Gate Logs.

CONSTRAINTS
- Encrypted fields (notes, professional_background, etc.) are decrypted only when the entry is opened.
- Creating a new entry uses the same screen with empty defaults.

VERIFY
- Add a Judge entry. Edit it. Save. Reload. Persists.
- Reference an entry from a project (this requires Sprint 5.3 wiring; for now, you can manually inject a `vault_ref:` into a PCD via DevTools to test). Confirm references panel shows the project.
- Delete an entry — references in projects are cleared and logged.

REPORT
Screenshots: Judge detail screen, Winning Solution detail, Boilerplate detail.
```

**Verification:** Commit.

---

### 5.3 — Vault Auto-Query at Phase 1, 2, 4 🤖

**🧠 Planning Mode: YES** *(three integration points, each different)*

```
You are wiring up the Vault auto-query — the moments where Startup surfaces relevant Vault entries during the workflow.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §3.4 (Auto-query points), §9.4 (New Project Setup), §4.3 (Vault references in prompt context).

GOAL
At Phase 1 project create, surface matching Judges/Organizers and Winning Solutions. At Phase 2 (Competitive Pre-mortem), surface relevant Winning Solutions. At Phase 4 (TID generation), surface relevant Boilerplate Components. User opt-in adds them as `vault_ref:<id>` to the relevant PCD section, which the prompt generator includes in context.

DELIVERABLES
1. `frontend/src/lib/vault/autoQuery.ts` — exports `queryAtProjectCreate(profile)`, `queryAtPhase2(pcd)`, `queryAtPhase4(pcd)`. Each returns `VaultMatch[]` with `entry`, `matched_on` (which fields matched), and `match_score`.
2. Replace the stub in 1.5's New Project Setup: after the form passes validation, before commit, run `queryAtProjectCreate` on the form values. If matches found, surface an inline panel: "We found these in your Vault that might be relevant. Include any to seed Phase 1?" Each match is selectable. Selected matches are added as `vault_ref:<id>` in the new PCD's `judge_intelligence.data.vault_seeded_refs[]` (add this field to the schema if not present — it's a list of vault_ref strings).
3. At Phase 2 workspace open, run `queryAtPhase2` once and surface a callout above the Competitive Pre-mortem prompt card: "X relevant past winning solutions in your Vault. Add to context?" Selecting them appends to `competitive_landscape.data.vault_seeded_refs[]`.
4. At Phase 4 workspace open, similar callout for Boilerplate matches → `technical_direction.data.vault_seeded_refs[]`.
5. Update the prompt generator (4.3): when a section has `vault_seeded_refs`, fetch the referenced Vault entries and inject their content into the context. Include a `VaultQueryHint` per §6.2 in the generated prompt's preamble explaining what was pulled.

CONSTRAINTS
- Matching logic for MVP is keyword-based: organizer name match, sponsor name match, domain keyword overlap, year proximity. Document the match_score formula in code.
- User opt-in is explicit. Auto-queries surface; they don't auto-add.
- The prompt context injection respects the existing context selection logic from 2.4.

VERIFY
- Add a Judge entry to the Vault matching an organizer name. Create a new project with that organizer. Confirm the auto-query surfaces the entry.
- Opt in. Confirm the PCD's `judge_intelligence.data.vault_seeded_refs` contains the ref.
- Generate a Phase 1 Judge Research prompt — the included context now references the Vault entry.

REPORT
Screenshots of the auto-query callouts at each phase.
```

**Verification:** Vault content actually flows into prompts. Commit.

---

### 5.4 — Vault References Rendered in PCD Viewer 🤖

**🧠 Planning Mode: NO** *(small UX fix)*

```
You are wiring up the Vault reference rendering in the PCD Viewer.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §10.5 (Every Vault reference is clickable to its entry).

GOAL
Anywhere a `vault_ref:<id>` appears in the PCD, render it as a clickable chip showing the entry's name and type, linking to the Vault entry detail.

DELIVERABLES
1. `frontend/src/components/VaultRefChip.tsx` — takes a vault_ref string, looks up the entry across the three Vault stores, renders the entry name + type pill. Click → routes to `/vault/<type>/<id>`.
2. Update the PCD Viewer's section renderers (3.7) to detect `vault_ref:` strings in any string field or array-of-strings and render them as VaultRefChips.
3. The TODO marker from 3.7 is removed.

CONSTRAINTS
- If a vault_ref points to a deleted entry, render a muted "Deleted entry (id=...)" chip with no click action.

VERIFY
- Open a PCD with vault_seeded_refs. Confirm chips render with names.
- Click a chip → routes to the entry detail.
- Delete an entry then revisit the PCD → chip shows the deleted state.

REPORT
Screenshot of PCD Viewer with vault chips visible.
```

**Verification:** Commit.

**End of Sprint 5.** Vault is a real cross-project asset.

---

## Sprint 6 — Polish

**Goal:** Settings, keyboard shortcuts, prescriptive copy, the remaining Phase 5/6/7 affordances, Grill-Me toggle, External Context Notes, Handoff Package, Phase 3 candidate-review UI.

**Maps to spec §11.2 Milestone 6.**

---

### 6.1 — Settings Screen with JSON Editors 🤖

**🧠 Planning Mode: NO** *(focused screen)*

```
You are building the Settings screen.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §9.12 (Settings) and §7.3 (Default rubric customization).

GOAL
A settings page with expandable panels for: Default Judging Rubric (JSON editable), LLM Preferences (JSON), Proxy URL, Deadline Notification Intervals, Export Format Preferences, Grill-Me default mode. Plus Change Password and Clear All Data flows.

DELIVERABLES
1. `frontend/src/screens/settings/Settings.tsx` — implements §9.12.
2. Each setting persists to the `settings` Dexie table per the encryption model from 1.1.
3. Default Rubric panel: JSON textarea, validate-on-save (weights sum to 1.0±0.001 etc. per §7.3), reset-to-bundled-default link.
4. Change Password: requires current password + new password. Re-encrypts the entire IndexedDB store with the new key. This is the most error-prone flow in the app — write it carefully. On failure mid-flow, do NOT lose data; rollback by keeping the old encrypted store until the new one is fully written.
5. Clear All Data: typed-phrase confirmation ("delete everything"). Wipes all Dexie tables. Routes back to the first-run onboarding.

CONSTRAINTS
- JSON editors are textareas with on-save validation. No Monaco or other heavy editors in MVP.
- Change Password's atomicity is critical. Test it carefully.

VERIFY
- Edit each setting. Persist across reload.
- Change Password: succeed once, then fail with wrong current password.
- Clear All Data: with typed phrase, wipes everything; without, no-op.

REPORT
Screenshots of each settings panel.
```

**Verification:** Commit.

---

### 6.2 — Keyboard Shortcuts 🤖

**🧠 Planning Mode: NO** *(small, well-scoped)*

```
You are adding keyboard shortcuts.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §10.6 (Speed under pressure).

GOAL
Implement: ⌘P / Ctrl-P (project switcher), ⌘E (export PCD), ⌘L (lock app), ⌘? (cheatsheet modal).

DELIVERABLES
1. `frontend/src/lib/shortcuts/registry.ts` — global shortcut registry. Uses `window.addEventListener('keydown')` with proper event filtering (don't fire shortcuts when an input is focused, except ⌘L).
2. `frontend/src/components/ProjectSwitcher.tsx` — modal triggered by ⌘P. Searchable list of all projects. Enter to navigate.
3. `frontend/src/components/ShortcutCheatsheet.tsx` — modal listing all shortcuts.
4. ⌘E exports the current project's PCD via the same path as the PCD Viewer's Export button.
5. ⌘L calls `auth.lock()`.

CONSTRAINTS
- Shortcuts must not collide with native browser shortcuts (⌘W close tab, etc.).
- All shortcuts use Cmd on macOS, Ctrl on other platforms — detect with `navigator.platform`.

VERIFY
- Each shortcut works in the workspace, dashboard, and PCD viewer.
- Shortcuts don't fire while typing in inputs (except ⌘L).

REPORT
Confirm each shortcut.
```

**Verification:** Commit.

---

### 6.3 — Empty States + Prescriptive Copy Pass 🤖

**🧠 Planning Mode: YES** *(touches every screen; a coordinated voice pass)*

```
You are doing a full prescriptive-copy pass.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §10.1 (Prescriptive voice everywhere) thoroughly.

GOAL
Every empty state, error message, disabled CTA tooltip, and microcopy moment in the app is rewritten in prescriptive voice. The user is never asked what they want; they are told what to do next.

DELIVERABLES
1. Walk every screen built so far. Replace passive copy with directive copy. Examples:
   - "No projects yet" → "Create your first project to begin."
   - "Field required" → "Paste the SCAMPER output into the Research Intake below."
   - "Please complete required fields" → "Fill in the hackathon name and organizer name to create the project."
2. Every disabled CTA has a tooltip naming the missing requirement.
3. Phase Briefings begin with imperative verbs.
4. Compile a list of all changes and document them in `frontend/CHANGELOG.md` under a "Prescriptive Voice Pass" entry.

CONSTRAINTS
- Voice is directive but not commanding-in-a-bad-way. Read like a sharp coworker, not a drill instructor.
- Don't soften copy with "please" or "kindly" — those are passive.
- Errors describe the specific action: not "Invalid input", but "The submission deadline must be after the start date."

VERIFY
- Walk through every screen. List any remaining passive-voice phrases. Fix them.
- The empty state on the dashboard now reads as a clear call to action.

REPORT
Paste 10 before/after copy changes from the CHANGELOG.
```

**Verification:** Walk every screen with a friend who hasn't seen it before — they should always know what to do next.

---

### 6.4 — Backward-Navigation Reconciliation Polish 🤖

**🧠 Planning Mode: NO** *(refining an existing flow)*

```
You are polishing the backward-navigation reconciliation modal that landed in 4.1.

GOAL
The reconciliation modal triggered by backward navigation is clear, fast, and trustworthy.

DELIVERABLES
1. Refine the modal: list affected sections by name with previews ("Phase 2 - Problem Analysis: 4 root causes, 3 HMW reframes"), show stale_reason previews, single "Confirm" CTA that applies the staleness propagation, single "Cancel" that returns to the current phase.
2. Add a "Don't propagate this time" option that records a `stale_propagation_skipped` event in the Phase Gate Log with a justification field.

CONSTRAINTS
- The modal is the only friction in backward navigation. Make it fast — single screen, no multi-step flow.

VERIFY
- Multiple back-nav scenarios produce sensible reconciliation modals.

REPORT
Screenshot of the modal with multiple affected sections.
```

**Verification:** Commit.

---

### 6.5 — Phase 5 Build Checklist 🤖

**🧠 Planning Mode: NO**

```
You are implementing the Phase 5 Build Checklist.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md and STARTUP_WORKFLOW_AND_APP_OVERVIEW.md Phase 5 sections. Phase 5 has limited LLM activity — the workspace is primarily a checklist.

GOAL
Phase 5 workspace renders a build checklist derived from the TID's MoSCoW list, screen breakdown, and demo-prep items. User checks items off as completed.

DELIVERABLES
1. `frontend/src/lib/phaseContent/buildChecklist.ts` — `deriveBuildChecklist(pcd): BuildChecklistItem[]`. Walks `technical_direction.data.tid` (the TID intake's output) and produces items for every feature, screen, demo-prep item, and backup-plan item.
2. `frontend/src/screens/workspace/Phase5Workspace.tsx` — special-case workspace for Phase 5. Renders the checklist with checkboxes, group headers (Features / Screens / Demo Prep / Backup Plan), progress bars per group.
3. Time-to-deadline warnings at 50%, 75%, 90% per §9.3.
4. Checklist completion state persists in `pcd.phase_5_build_checklist` (add this field if not present in the schema; mirror the SectionEnvelope pattern).

CONSTRAINTS
- Phase 5 has no Prompt Generator or Research Intake cards. The workspace is dominated by the checklist.
- All other workspace affordances (PCD Viewer, Vault link, Phase Gate Log) remain available.

VERIFY
- Cross Phase 4 with Phase 4 outputs populated. Open Phase 5. Checklist renders.
- Check items, reload, persist.
- Time warnings fire at the right intervals (test by setting a near-future deadline).

REPORT
Screenshots: Phase 5 workspace with partially-completed checklist.
```

**Verification:** Commit.

---

### 6.6 — Phase 6 Outputs (Pitch Storyboard, Pitch Review, Q&A Prep) 🤖

**🧠 Planning Mode: NO** *(structured output rendering for already-built prompts)*

```
You are wiring up Phase 6's structured output rendering.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md (Phase 6 sections in spec) and STARTUP_WORKFLOW_AND_APP_OVERVIEW.md (Phase 6).

GOAL
The Phase 6 prompts are already registered (3.3). The intakes are too (3.4). What's missing is friendly per-section rendering of pitch storyboards, pitch reviews, and Q&A preps in the PCD Viewer and the workspace's Phase Output Preview.

DELIVERABLES
1. `frontend/src/components/pcdSections/PitchStoryboardRenderer.tsx` — renders the storyboard's scene list as a timeline. Each scene shows time window, narrative talking points, recommended visuals, hooks.
2. `frontend/src/components/pcdSections/PitchReviewRenderer.tsx` — per-judge-persona objection cards.
3. `frontend/src/components/pcdSections/QAPrepRenderer.tsx` — sortable list of likely questions with recommended answers and one-sentence fallbacks.
4. Hook these into the PCD Viewer's section renderer dispatch (3.7).
5. Phase 6 workspace renders these in the Phase Output Preview card.

CONSTRAINTS
- Renderers are read-only. Editing happens via Section Editor.

VERIFY
- Run Phase 6 prompts end-to-end. Confirm renderers display sensibly.

REPORT
Screenshots of each renderer with sample data.
```

**Verification:** Commit.

---

### 6.7 — Phase 7 Debrief + Vault Update Proposals 🤖

**🧠 Planning Mode: YES** *(involves cross-system: PCD intake produces structured proposals that target the Vault)*

```
You are wiring up Phase 7's flow.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md (Phase 7) and STARTUP_WORKFLOW_AND_APP_OVERVIEW.md (Phase 7) and §3 (Vault).

GOAL
After Phase 7's debrief intake, surface structured Vault update proposals: "Add this judge", "Add this past winning solution", "Update this boilerplate component". User opts in per proposal, which writes to the appropriate Vault store.

DELIVERABLES
1. The `phase_7_debrief` intake (already exists from 3.4) is extended to produce structured `vault_update_proposals[]` per the spec's Phase 7 description. Update the IntakeDefinition's target_schema_fragment to include this array.
2. `frontend/src/screens/workspace/Phase7Workspace.tsx` — special-case workspace. Standard layout PLUS a "Vault Update Proposals" card that lists each proposal with its type (judge / solution / boilerplate), a preview of the entry to be created or updated, and Accept / Decline buttons. Accept writes to the Vault. Decline marks the proposal as declined.
3. After Phase 7 is complete and all proposals resolved, mark the project archived and surface a "Project archived" banner.

CONSTRAINTS
- Vault writes from Phase 7 follow the encrypted-content rule from 1.1.
- Declined proposals are recorded in the project's Phase Gate Log for audit.

VERIFY
- Cross Phase 4, simulate getting through to Phase 7. Run the debrief prompt with a fake hackathon outcome.
- Confirm proposals appear. Accept some, decline others. Check Vault entries are created.
- Project shows archived state.

REPORT
Screenshots of Phase 7 workspace with proposals.
```

**Verification:** Commit.

---

### 6.8 — Grill-Me Toggle UI on Eligible Prompt Cards 🤖

**🧠 Planning Mode: YES** *(touches the Prompt Generator card; per-prompt toggle behavior)*

```
You are adding the Grill-Me toggle UI to eligible Prompt Generator cards.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §4.7 (Grill-Me Mode) and §11.2 M6 task 36.

GOAL
Each PromptGeneratorCard with `grill_me_modes` declared shows a three-position toggle (Off / Interactive / Auto). For prompts where Grill-Me is REQUIRED (Step 3a, Phase 4 Design Grill), the toggle shows only Interactive / Auto (no Off). User's choice flows into the generate function.

DELIVERABLES
1. Add a `<GrillModeToggle>` component above the Generate Prompt button on eligible cards.
2. The component reads default mode from settings (lands in 6.1 — `grill_me_default_mode`).
3. Toggle state is component-local (resets per card render). User changes per-prompt.
4. The Generate Prompt CTA calls generate(...) with the selected mode.
5. When mode is not Off, surface the "Export Handoff Package" button per §11.2 M6 task 38 (lands in 6.10).
6. Visual treatment: toggle is a segmented control. Required-mode prompts (only Interactive / Auto) have the segment for Off muted/hidden with a small note "Grill-Me required for this prompt."

CONSTRAINTS
- Toggle state is ephemeral — not persisted. Default is the user's settings preference.
- Prompts without `grill_me_modes` declared show no toggle.

VERIFY
- Step 3a Idea Seed Challenge card shows only Interactive / Auto.
- Phase 3 Divergent Candidates card shows Off / Interactive / Auto with default Off (per user settings).
- Generated prompts differ correctly between Off and Interactive (the task spec is Grill-Me when on).

REPORT
Screenshots: required-mode card, optional-mode card, Off vs Interactive generated prompt comparison.
```

**Verification:** Commit.

---

### 6.9 — External Context Notes Editor 🤖

**🧠 Planning Mode: NO**

```
You are adding the External Context Notes editor.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §11.2 M6 task 37 and the workflow doc on External Context Notes.

GOAL
A markdown textarea bound to `pcd.external_context_notes` (whose section envelope already exists). Surfaced in the PCD Viewer and as a side-panel on Grill-Me-eligible Prompt Generator cards. Inclusion in prompts follows the `'if_user_provided'` rule from 2.4.

DELIVERABLES
1. `frontend/src/components/ExternalContextNotesEditor.tsx` — markdown textarea with live render preview toggle. Save persists to the workspace store.
2. PCD Viewer renders this section near the bottom (it's free-form, low-priority for read).
3. Side-panel on Grill-Me cards: a slide-out drawer with the editor, plus a "Last updated" timestamp.

CONSTRAINTS
- Pure markdown — no rich-text editor.
- Inclusion rule `'if_user_provided'` already implemented in 2.4. Verify it works correctly: include only when the data string is non-empty.

VERIFY
- Edit notes via PCD Viewer. Persist. Reload.
- Open a Grill-Me-eligible prompt card. Drawer toggles in/out.
- Generate a prompt with notes empty → notes not in context. Generate with notes filled → notes appear in context.

REPORT
Screenshot of the side-panel drawer.
```

**Verification:** Commit.

---

### 6.10 — Handoff Package Export Button 🤖

**🧠 Planning Mode: NO**

```
You are adding the Handoff Package export.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §11.2 M6 task 38 and §4.5 (handoff package).

GOAL
On Grill-Me-eligible prompt cards (when toggle is Interactive or Auto), an "Export Handoff Package" button produces a single markdown file containing: full PCD render + external_context_notes + the generated prompt. User uploads this to their external LLM's web UI before starting a long Grill-Me session.

DELIVERABLES
1. `frontend/src/lib/prompts/exportHandoffPackage.ts` — `exportHandoffPackage(pcd, generatedPrompt): { filename: string, content: string }`. Concatenates renderPCDToMarkdown(pcd) + a separator + the external_context_notes (if populated) + a separator + the generated prompt. Filename: `<project>-handoff-<timestamp>.md`.
2. Button on Grill-Me prompt cards, visible only when toggle is non-Off. Click → download.

CONSTRAINTS
- The package is a snapshot. It does not live-update.
- Mention this in a small note next to the button: "Snapshot — re-export to capture later changes."

VERIFY
- Generate a Grill-Me prompt. Export the package. Open the file.
- Confirm the structure: PCD, notes, prompt — separated cleanly.

REPORT
Paste the structure of an exported package.
```

**Verification:** Commit.

---

### 6.11 — Phase 3 Candidate-Review UI 🤖

**🧠 Planning Mode: YES** *(custom UI: browse, compare, remix flow)*

```
You are building the Phase 3 candidate-review and remix UI.

Read STARTUP_IMPLEMENTATION_SPEC_2026-05-01_v1_2.md §11.2 M6 task 39 and §6.2 (post-extract enrichment for divergent_candidates_intake).

GOAL
After the Divergent Candidates intake commits, the user can browse the candidate portfolio (5 candidates, scored on Innovation / Impact / Feasibility), compare scores by archetype (Safe Utility, Weird Gem, Moonshot, Social Impact), and remix two or more candidates into a hybrid that's stored as a new CandidateIdea with `is_remix=true` and `remix_source_ids` populated.

DELIVERABLES
1. `frontend/src/screens/phase3/CandidateReview.tsx` — special-case section rendered in Phase 3 workspace after the Divergent Candidates intake commits. Lists candidates as cards with scores and archetype badges. Score-comparison view: a small radar or bar chart per candidate; or a side-by-side score table.
2. Remix flow: select 2+ candidates, click "Remix". A modal opens with a name field, a description textarea, and the source candidate ids. On save, write a new `CandidateIdea` to `ideation_record.data.candidate_portfolio.candidates[]` with `is_remix=true` and `remix_source_ids: [...]`.
3. After review, the user clicks "Choose this direction" on one candidate (original or remix). This writes to `ideation_record.data.chosen_direction` and unlocks the Downselection Verification prompt (already registered).

CONSTRAINTS
- Remix is a structured action. The user must give the remix a name and a description; it's not a silent merge.
- Choosing a direction is a deliberate click, not a drag.

VERIFY
- Run Phase 3 end-to-end through Step 3e. Browse the resulting candidates.
- Remix two candidates. Confirm the new candidate appears with is_remix=true.
- Choose a direction. Confirm chosen_direction updates and Downselection Verification prompt becomes generatable.

REPORT
Screenshots: candidate browser, score comparison, remix modal, post-choice state.
```

**Verification:** Commit.

**End of Sprint 6.** Feature-complete.

---

## Sprint 7 — Final Dogfooding and Hardening

**Goal:** Catch what dogfooding reveals. Ship.

---

### 7.1 — Dogfood Checkpoint #3: Full End-to-End Run 🧑

**Manual task.** Allow ~4–6 hours.

Run a real or simulated hackathon end-to-end through the app. Phase 1 → Phase 7 (or as much as possible without an actual hackathon).

Specific things to verify:

- New Project Setup is fast and pleasant.
- Every prompt generates without error.
- Every paste-back parses cleanly (or fails gracefully into Tier 2/Tier 3).
- The Phase 4 gate feels appropriately serious.
- Cursor can actually consume the exported TID and Product Brief productively.
- Phase 5 build checklist is useful.
- Phase 6 outputs are pitch-ready.
- Phase 7 debrief proposals make sense.
- Vault auto-queries find the right entries when seeded.
- Backward navigation + staleness propagation works.
- Addendum flow works.
- Lock + unlock + Change Password works without data loss.

**Take notes obsessively.** Group them into:
- **Bugs** — definitely broken.
- **Friction** — works, but slows you down.
- **Polish** — works, but feels off.

**Verification:** A friction list exists. You used Startup for real-feeling work and didn't bypass it.

---

### 7.2 — Bugfix Pass on Dogfood Findings 🤖

**🧠 Planning Mode: YES** *(touching many areas based on findings)*

```
You are fixing the bugs and friction items discovered in Dogfood Checkpoint #3.

The user will paste a bug list below. For each item, you will:
1. Reproduce it (or identify the exact code path).
2. Fix it.
3. Add or update a regression test where reasonable.
4. Verify the fix doesn't break adjacent flows.

[USER: paste your friction list here, one item per line, with brief reproduction notes.]

CONSTRAINTS
- Do not introduce new features. This is a bugfix pass.
- One item per commit. Each commit message names the bug.
- If a fix is more than 1 hour of work, escalate it to a follow-up task and skip it for now — note it in REMAINING_FRICTION.md at the repo root.

VERIFY
- Each fix has its own test or manual repro confirmation.
- The full app still passes typecheck and lint.

REPORT
For each item: commit SHA, fix summary, verification step.
```

**Verification:** Each item resolved or deferred with a note.

---

### 7.3 — Production Deploy and Final Smoke Test 🧑

**Manual task.**

1. Ensure `main` is clean. Tag a `v1.0.0` release.
2. Confirm Cloudflare Pages built the latest. Visit the production URL.
3. Set up password from scratch on a fresh browser profile (incognito).
4. Walk through the Phase 1 → Phase 2 prompt loop using a real LLM.
5. Confirm the Worker proxy is healthy: check Cloudflare Workers analytics.
6. Verify rate limiting fires correctly (smoke test: send 301 quick requests, confirm 429).

**Verification:** A clean install on production URL works for a real Phase 1–2 walkthrough.

---

## What Comes After v1.0

The spec calls out stretch goals in §8.6 and §11.3. Don't touch them until you've used Startup at a real hackathon and won (or come close enough that Startup wasn't the bottleneck).

After your first competition with Startup in production:

1. Update this doc with what was wrong.
2. Update PROMPTS.md with prompts that worked or didn't.
3. Update DESIGN.md with visual decisions you'd revisit.
4. Run Phase 7 against your own hackathon outcome and let the Vault grow.

The system compounds. Every hackathon is a debrief that improves the next one.

---

## Appendix A — Mini-Sprint Summary Table

| # | Title | Type | Planning | Sprint |
|---|-------|------|----------|--------|
| 0.1 | Create accounts and get keys | 🧑 | — | 0 |
| 0.2 | Author DESIGN.md | 🧑 | — | 0 |
| 0.3 | Initialize repo and tooling | 🤝 | NO | 0 |
| 0.4 | Hello-world deploy | 🧑 | — | 0 |
| 0.5 | Test infrastructure | 🤖 | NO | 0 |
| 1.1 | Persistence + encryption + lock | 🤖 | YES | 1 |
| 1.2 | PCD schema + validator | 🤖 | NO | 1 |
| 1.3 | Default rubric parser | 🤖 | NO | 1 |
| 1.4 | Project Dashboard skeleton | 🤖 | YES | 1 |
| 1.5 | New Project Setup form | 🤖 | YES | 1 |
| 1.6 | Debug panel + logger + error boundary | 🤖 | NO | 1 |
| 2.1 | Cloudflare Worker proxy | 🤖 | YES | 2 |
| 2.2 | PromptDefinition registry types | 🤖 | YES | 2 |
| 2.3 | schemaToOutputTemplate | 🤖 | NO | 2 |
| 2.4 | Generate function | 🤖 | YES | 2 |
| 2.5 | Project Workspace shell | 🤖 | YES | 2 |
| 2.6 | Prompt Generator card | 🤖 | NO | 2 |
| 2.7 | IntakeDefinition + extract.ts | 🤖 | YES | 2 |
| 2.8 | Three-tier handler + diff + commit | 🤖 | YES | 2 |
| 2.9 | Dogfood Checkpoint #1 | 🧑 | — | 2 |
| 3.1 | Prompts: Phase 1 + 2 | 🤖 | YES | 3 |
| 3.2 | Prompts: Phase 3 | 🤖 | YES | 3 |
| 3.3 | Prompts: Phases 4–7 + addendum | 🤖 | YES | 3 |
| 3.4 | Intakes: all phases | 🤖 | YES | 3 |
| 3.5 | Briefings + checklists | 🤖 | NO | 3 |
| 3.6 | State machine + Advance CTA | 🤖 | YES | 3 |
| 3.7 | PCD Viewer | 🤖 | YES | 3 |
| 3.8 | Section Editor | 🤖 | YES | 3 |
| 4.1 | Dependency graph + propagateStaleness | 🤖 | YES | 4 |
| 4.2 | Stale resolution UI | 🤖 | YES | 4 |
| 4.3 | Phase Gate Log persistence + viewer | 🤖 | NO | 4 |
| 4.4 | Phase 4 gate finalize | 🤖 | YES | 4 |
| 4.5 | Document Export Center | 🤖 | NO | 4 |
| 4.6 | Addendum Protocol Interface | 🤖 | YES | 4 |
| 4.7 | Dogfood Checkpoint #2 | 🧑 | — | 4 |
| 5.1 | Vault top-level UI | 🤖 | YES | 5 |
| 5.2 | Vault entry detail / edit | 🤖 | YES | 5 |
| 5.3 | Vault auto-query | 🤖 | YES | 5 |
| 5.4 | Vault refs in PCD Viewer | 🤖 | NO | 5 |
| 6.1 | Settings screen | 🤖 | NO | 6 |
| 6.2 | Keyboard shortcuts | 🤖 | NO | 6 |
| 6.3 | Prescriptive copy pass | 🤖 | YES | 6 |
| 6.4 | Backward-nav reconciliation polish | 🤖 | NO | 6 |
| 6.5 | Phase 5 Build Checklist | 🤖 | NO | 6 |
| 6.6 | Phase 6 output renderers | 🤖 | NO | 6 |
| 6.7 | Phase 7 Debrief + Vault proposals | 🤖 | YES | 6 |
| 6.8 | Grill-Me toggle UI | 🤖 | YES | 6 |
| 6.9 | External Context Notes editor | 🤖 | NO | 6 |
| 6.10 | Handoff Package export | 🤖 | NO | 6 |
| 6.11 | Phase 3 candidate-review UI | 🤖 | YES | 6 |
| 7.1 | Dogfood Checkpoint #3 | 🧑 | — | 7 |
| 7.2 | Bugfix pass | 🤖 | YES | 7 |
| 7.3 | Production deploy | 🧑 | — | 7 |

**Totals:** 53 mini-sprints. 40 agent-driven, 8 manual, 1 hybrid, 4 dogfood/QA.

---

*End of Execution Instructions. Drive top-to-bottom. Commit after every mini-sprint. Dogfood at every checkpoint. Build velocity compounds — your second hackathon with Startup will be twice as fast as the first.*
