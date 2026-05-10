# Requirements: Maud

**Defined:** 2026-05-10
**Core Value:** Take a project from idea to launch using local, human-readable markdown that the user can edit at any time.

## v1 Requirements

Each requirement maps 1:1 to a backlog story (or sub-task) in `.maud/stories/`.

### Setup

- [ ] **SETUP-01**: Markdown links across all `.maud/` files are consistent and resolve correctly
- [ ] **SETUP-02**: Bare-bones Claude Code plugin scaffold exists and `/maud:*` commands are discoverable in Claude Code

### Discussion

- [ ] **DISC-01**: GSD provides unvarnished critique of Maud's structure, missing tools/metadata, and overall feasibility

### /maud:plan Command

- [ ] **PLAN-01**: `/maud:plan` deeply questions the user about their project and writes `.maud/README.md` (the project file)
- [ ] **PLAN-02**: `/maud:plan` performs meta-cognition to determine the project-specific `.maud/planning/` folder structure and creates the empty subfolders
- [ ] **PLAN-03**: `/maud:plan` spawns research agents — one per planning subfolder — that populate each subfolder with relevant research
- [ ] **PLAN-04**: `/maud:plan` synthesizes `.maud/README.md` plus all planning research into "the plan" at `.maud/planning/README.md`, ruthlessly avoiding scope creep
- [ ] **PLAN-05**: `/maud:plan` optionally generates design (offering: skip / Claude-generated / external AI tools / human designer / wireframes) when the project requires it
- [ ] **PLAN-06**: `/maud:plan` generates `.maud/stories/` (with `README.md`, `backlog/`, `current/`, `complete/`) and populates the backlog with stories derived from the plan, in initial priority order

### /maud:build Command

- [ ] **BUILD-01**: `/maud:build` pops the top story off `.maud/stories/backlog/` and moves it to `current/`
- [ ] **BUILD-02**: `/maud:build` asks the user any clarifying questions about the current story before building
- [ ] **BUILD-03**: `/maud:build` builds the story end-to-end in Claude Code
- [ ] **BUILD-04**: `/maud:build` runs validation agents that verify the work meets the story's intent
- [ ] **BUILD-05**: `/maud:build` presents the completed work to the user for approval
- [ ] **BUILD-06**: `/maud:build` on approval merges to main and launches to production where relevant; story moves from `current/` to `complete/`

## v2 Requirements

Deferred to future milestones.

### Self-hosting

- **SELF-01**: Use Maud (v1) to rebuild Maud — eat our own dog food
- **SELF-02**: Use the rebuilt Maud to build an unrelated software project (TBD)

### Story metadata

- **META-01**: Stories support tags
- **META-02**: Stories support blockers / dependencies
- **META-03**: Stories support links to design artifacts

## Out of Scope

| Feature | Reason |
|---------|--------|
| Cloud-hosted state | Maud is local-first by design — `.maud/` lives in the repo |
| Non-markdown artifact formats | Readability and direct user editing is the core value |
| Real-time multi-user collaboration | Single-user, local CLI workflow |
| Non-Claude-Code execution surfaces (web UI, mobile) | Claude Code plugin is the only target |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| SETUP-01 | Phase 1 | Pending |
| DISC-01 | Phase 2 | Pending |
| SETUP-02 | Phase 3 | Pending |
| PLAN-01 | Phase 4 | Pending |
| PLAN-02 | Phase 5 | Pending |
| PLAN-03 | Phase 6 | Pending |
| PLAN-04 | Phase 7 | Pending |
| PLAN-05 | Phase 8 | Pending |
| PLAN-06 | Phase 9 | Pending |
| BUILD-01 | Phase 10 | Pending |
| BUILD-02 | Phase 11 | Pending |
| BUILD-03 | Phase 12 | Pending |
| BUILD-04 | Phase 13 | Pending |
| BUILD-05 | Phase 14 | Pending |
| BUILD-06 | Phase 15 | Pending |

**Coverage:**
- v1 requirements: 15 total
- Mapped to phases: 15
- Unmapped: 0 ✓

---
*Requirements defined: 2026-05-10*
*Last updated: 2026-05-10 after initial definition*
