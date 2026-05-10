# Roadmap: Maud

**Created:** 2026-05-10
**Phases:** 15
**Coverage:** 15/15 v1 requirements mapped ✓

Each phase corresponds 1:1 to a backlog story (or sub-task file) in `.maud/stories/`. The order matches the order of the backlog. Phase numbering is the build order.

---

## Phase 1: Fix `.maud/` Markdown Links

**Goal:** All markdown links in `.maud/` resolve correctly and follow a consistent style/convention.

**Requirements:** SETUP-01

**Story:** `.maud/stories/README.md` — top backlog item ("Fix all the links in all of the markdowns. Make them consistent.")

**Success criteria:**
1. Every relative link in every `.md` file under `.maud/` resolves to an existing file or directory
2. Link style is consistent across files (one chosen convention, applied everywhere)
3. Broken links found in the existing `.maud/` (e.g., `(/complete/scrap-gsd.md]` in `stories/README.md`) are repaired

**Plans:** 1 plan
- [ ] 01-01-PLAN.md — Rename `.maud/**/index.md` to `README.md`, rewrite links to standard relative paths, fix typo, cascade prose updates into `.planning/{ROADMAP,REQUIREMENTS,PROJECT}.md`

---

## Phase 2: Discuss Maud with GSD

**Goal:** Produce GSD's unvarnished written critique of the Maud approach so the user can refine the design before building.

**Requirements:** DISC-01

**Story:** `.maud/stories/backlog/discuss-maud-with-gsd.md`

**Success criteria:**
1. A written critique exists addressing each of the user's questions: structure soundness, will-it-work feasibility, logic gaps, missing GSD ports, additional tools, story metadata (tags / design / blockers)
2. The critique is direct and specific — flags concrete weaknesses, not generic praise
3. The user has reviewed the critique and any resulting structural changes are captured (either applied to `.maud/` or recorded as future v2 requirements)

---

## Phase 3: Setup Claude Code Plugin

**Goal:** Bare-bones Claude Code plugin exists and `/maud:plan` and `/maud:build` are discoverable from within Claude Code, even if they're stubs.

**Requirements:** SETUP-02

**Story:** `.maud/stories/backlog/setup-claude-code-plugin.md`

**Success criteria:**
1. Plugin scaffolding is in place per https://code.claude.com/docs/en/plugins
2. Running Claude Code in a project with the plugin installed surfaces `/maud:plan` and `/maud:build` as available slash commands
3. Both commands run without error (stub responses are acceptable — implementation comes in later phases)

---

## Phase 4: `/maud:plan` — Generate Project File

**Goal:** `/maud:plan` deeply questions the user and produces `.maud/README.md`, the source of truth for what the project is.

**Requirements:** PLAN-01

**Story:** `.maud/stories/backlog/plan-command/generate-project-file.md`

**Success criteria:**
1. Command opens with "What are you working on?" and follows up with progressively deeper questions until the project's core is clear
2. Command writes `.maud/README.md` capturing the project's purpose, audience, and how it works
3. Output `.maud/README.md` is human-readable, matches the example in this repo's `.maud/README.md`, and the user agrees it accurately describes their project

---

## Phase 5: `/maud:plan` — Generate Planning Structure

**Goal:** Meta-cognition step — `/maud:plan` decides which planning dimensions matter for *this specific* project and creates the empty `.maud/planning/<dimension>/` folders.

**Requirements:** PLAN-02

**Story:** `.maud/stories/backlog/plan-command/generate-planning-structure.md`

**Success criteria:**
1. Command reads `.maud/README.md` and reasons about which planning subfolders fit the project (e.g., `competitors/`, `architecture/`, `technology/`, `design/` — or others)
2. Empty subfolders are created under `.maud/planning/`, one per chosen dimension
3. The choice of dimensions is project-appropriate (not boilerplate) and the user can see/edit the chosen structure before proceeding

---

## Phase 6: `/maud:plan` — Run Research Agents

**Goal:** For each planning subfolder, a dedicated research agent investigates and writes its findings into that subfolder.

**Requirements:** PLAN-03

**Story:** `.maud/stories/backlog/plan-command/research-agents.md`

**Success criteria:**
1. One research agent runs per subfolder created in Phase 5
2. Each agent writes useful, project-specific research artifacts in whatever markdown format best fits its dimension
3. Subfolders are non-empty after the step completes; the user can review the artifacts and edit them directly

---

## Phase 7: `/maud:plan` — Generate "The Plan"

**Goal:** Synthesize `.maud/README.md` + all `.maud/planning/<dimension>/` research into `.maud/planning/README.md` — "the plan" — ruthlessly scoped, no creep.

**Requirements:** PLAN-04

**Story:** `.maud/stories/backlog/plan-command/generate-the-plan.md`

**Success criteria:**
1. `.maud/planning/README.md` exists and articulates the approach the project will take
2. The plan is minimal — no scope beyond what the research and project file justify; assumptions are surfaced as questions, not silently committed
3. The user can review and edit the plan, and "approves" it before the next phase advances

---

## Phase 8: `/maud:plan` — Generate Design (Optional)

**Goal:** Determine whether the project needs a design step, and if so, produce or coordinate one. Cleanly skip when not applicable (Maud itself, for example).

**Requirements:** PLAN-05

**Story:** `.maud/stories/backlog/plan-command/generate-design.md`

**Success criteria:**
1. Command decides — with the user — whether design is needed (skipped for projects like Maud, default-language for iOS, custom for SaaS)
2. When design is needed, the command offers concrete paths: Claude generates it / external AI tool / human designer / wireframe shortcut / defer
3. When design is skipped or completed, the user can advance to story generation

---

## Phase 9: `/maud:plan` — Generate Stories

**Goal:** Create `.maud/stories/` (with `README.md`, `backlog/`, `current/`, `complete/`) and populate the backlog from the plan, in an initial priority order the user can edit.

**Requirements:** PLAN-06

**Story:** `.maud/stories/backlog/plan-command/generate-stories.md`

**Success criteria:**
1. `.maud/stories/` exists with `README.md`, `backlog/`, `current/`, `complete/` subdirectories
2. Backlog contains stories derived from `.maud/README.md`, `.maud/planning/README.md`, and any design — in a defensible initial order
3. `.maud/stories/README.md` lists all stories and serves as the project's "UI"; the user can re-prioritize by editing files directly

---

## Phase 10: `/maud:build` — Pop Top Story

**Goal:** `/maud:build` pulls the top story off `.maud/stories/backlog/` and moves it to `current/`.

**Requirements:** BUILD-01

**Story:** `.maud/stories/backlog/build-command/README.md` (sub-task: "Takes the top story off of the backlog")

**Success criteria:**
1. Command identifies the top story in `.maud/stories/backlog/` (by file order in `README.md`)
2. The story file is moved into `.maud/stories/current/`
3. `.maud/stories/README.md` is updated to reflect the move; the change is committed to git

---

## Phase 11: `/maud:build` — Ask Clarifying Questions

**Goal:** Before building, `/maud:build` asks the user any open questions about the current story so the build doesn't proceed on shaky assumptions.

**Requirements:** BUILD-02

**Story:** `.maud/stories/backlog/build-command/README.md` (sub-task: "Asks the user questions about it if they have any")

**Success criteria:**
1. Command reads the current story and surfaces any ambiguities or missing information as questions to the user
2. When the story is clear enough, the command says so and offers to skip questioning
3. User answers (or skips) and the answers are captured for use during the build step

---

## Phase 12: `/maud:build` — Build the Story

**Goal:** Implement the current story end-to-end in Claude Code.

**Requirements:** BUILD-03

**Story:** `.maud/stories/backlog/build-command/README.md` (sub-task: "Builds the story in Claude Code")

**Success criteria:**
1. Code/content for the story is implemented to a runnable, testable state
2. Changes are committed atomically (one or more commits scoped to this story)
3. The implementation visibly addresses what the story describes — no off-scope work mixed in

---

## Phase 13: `/maud:build` — Validation Agents

**Goal:** After building, validation agents independently verify the work matches the story's intent.

**Requirements:** BUILD-04

**Story:** `.maud/stories/backlog/build-command/README.md` (sub-task: "Have validation agents verify the work")

**Success criteria:**
1. One or more validation agents run against the built work and produce a verdict
2. Verdicts are written somewhere the user can see them (e.g., a comment on the story or a validation report file)
3. Failed validations block advancement to user approval; passing ones advance the flow

---

## Phase 14: `/maud:build` — Present for Approval

**Goal:** Show the user the completed work and any validation reports, and ask for approval.

**Requirements:** BUILD-05

**Story:** `.maud/stories/backlog/build-command/README.md` (sub-task: "Presents it to the user for approval")

**Success criteria:**
1. Command shows a clear summary of what changed, links to the diff, and surfaces validation results
2. User can approve, reject (with notes), or request changes
3. The approval state is captured on the story so the next sub-step knows what to do

---

## Phase 15: `/maud:build` — Merge & Launch

**Goal:** On approval, merge the work to main and launch to production where relevant; move the story from `current/` to `complete/`.

**Requirements:** BUILD-06

**Story:** `.maud/stories/backlog/build-command/README.md` (sub-task: "Once approved, merges into main, launches to production, if relevant.")

**Success criteria:**
1. Approved work is merged to `main` (working tree clean, no conflicts)
2. If the project has a production target, the launch step runs (deploy command, publish, etc.); if not, this step cleanly no-ops
3. The story file moves from `.maud/stories/current/` to `.maud/stories/complete/`, `.maud/stories/README.md` is updated, and the changes are committed

---

## Phase Dependency Notes

- Phases 1–3 are setup; 1 and 2 are independent of 3, but 3 unlocks all `/maud:*` work.
- Phases 4–9 are sequential within `/maud:plan` — each step's output feeds the next.
- Phases 10–15 are sequential within `/maud:build` — they're literally the steps of one command's flow.
- Phase 8 (Generate Design) is optional per the story; the command must support skipping cleanly.

---

*Last updated: 2026-05-10 after initial roadmap creation*
