# Requirements: Maud

**Defined:** 2026-05-08
**Core Value:** Maud takes a developer from idea to shipped software with deep PM thinking baked in upfront — and the user explicitly verifies every phase transition before moving forward, so no consequential decision happens behind their back.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Plugin Infrastructure (PI)

- [ ] **PI-01**: User installs Maud locally via `claude --plugin-dir <path-to-maud>` and Claude Code recognizes it without errors
- [ ] **PI-02**: Plugin manifest at `.claude-plugin/plugin.json` declares name, version, description, author per Claude Code 2026 schema
- [ ] **PI-03**: Plugin exposes a single user-facing slash command (`/maud`) that loads without configuration
- [ ] **PI-04**: Plugin exposes worker agents (researcher, designer, architect, story-extractor, prioritizer, iterator, verifiers) under `agents/`, dispatched only by `/maud`
- [ ] **PI-05**: Plugin includes `templates/` and `references/` directories for reusable prompt fragments, loaded into commands/agents via `@${CLAUDE_PLUGIN_ROOT}/...`
- [ ] **PI-06**: All plugin-internal paths use `${CLAUDE_PLUGIN_ROOT}`; no hardcoded user paths

### State & Resumability (STATE)

- [ ] **STATE-01**: On first run in a directory, `/maud` creates `.maud/` and seeds initial state files
- [ ] **STATE-02**: `.maud/state.json` records current phase plus any sub-state needed by the dispatcher
- [ ] **STATE-03**: Re-running `/maud` reads `.maud/state.json` and resumes from the recorded phase without prompting the user to re-enter prior context
- [ ] **STATE-04**: On resume, Maud presents a "you are here" summary (current phase, story counts: Done / In Progress / Backlog) before continuing
- [ ] **STATE-05**: `.maud/log.md` is an append-only audit trail; every spawned agent writes a short entry (what it did, what it found, what it decided) before returning
- [ ] **STATE-06**: All committable state is committed via git when a phase transition is approved
- [ ] **STATE-07**: User can run `/maud` after a `/clear` (or in a new session) and pick up exactly where they left off

> v1 assumes a single sequential `/maud` session per project — no concurrency, no lock file. Atomic-write / locking concerns are deferred until they bite.

### Initialization (INIT)

- [ ] **INIT-01**: User runs `/maud` in an empty (or fresh) directory and is prompted to describe what they want to build
- [ ] **INIT-02**: Maud asks open-ended clarifying questions until it has enough to define an MVP (not multiple-choice rubber-stamping)
- [ ] **INIT-03**: Initialization produces `.maud/PROJECT.md` containing: what the product is, core value, MVP scope, constraints, key decisions made during questioning
- [ ] **INIT-04**: User explicitly approves PROJECT.md before initialization advances to research

### Research (RES)

- [ ] **RES-01**: Maud proposes a list of research dimensions appropriate for this project's type (e.g., for a web app: competitor research, tech stack, UI patterns, feasibility) and asks the user for direction on what to research
- [ ] **RES-02**: Research executes via parallel sub-agents that write findings to `.maud/research/<dimension>.md`
- [ ] **RES-03**: A synthesizer agent produces `.maud/research/SUMMARY.md` consolidating findings into actionable recommendations
- [ ] **RES-04**: User explicitly approves the synthesized research at the end of the phase, before advancing — and may request additional research, revisions, or rerun

### Architecture (ARCH)

- [ ] **ARCH-01**: Maud produces `.maud/architecture.md` covering tech stack, system shape, local dev setup, and production environment
- [ ] **ARCH-02**: Architecture is informed by approved research; it explicitly avoids premature optimization and aims for the simplest viable design
- [ ] **ARCH-03**: User iterates with Maud on architecture until satisfied
- [ ] **ARCH-04**: User explicitly approves architecture before advancing
- [ ] **ARCH-05**: The order between architecture and design is project-type-dependent — Maud determines and announces the order, doesn't impose a fixed sequence

### Design (DSGN)

- [ ] **DSGN-01**: Design artifacts live in `.maud/design/` in a format determined by the design meta-step (see PTA-05) — e.g., screens for a web app; sprite sheets / maps for a game; structured spec for a CLI; wireframes for a mobile app
- [ ] **DSGN-02**: If the user already has a design (provides files, references, etc.), Maud uses it instead of generating from scratch
- [ ] **DSGN-03**: User and Maud iterate on design (additions, deletions, MVP-trimming) until the user approves it
- [ ] **DSGN-04**: User explicitly approves design before advancing to story extraction

### Stories (STOR)

- [ ] **STOR-01**: Stories are extracted from approved design + architecture and saved to `.maud/stories/` (one file per story plus `INDEX.md`)
- [ ] **STOR-02**: Each story file contains: title, human-readable description (assumes common sense, not exhaustive), references to relevant design/architecture docs, success criteria, optional checklist of sub-tasks
- [ ] **STOR-03**: Each story is individually buildable and shippable (where "shippable" is project-dependent — could be merge-to-main, deploy-to-staging, or deploy-to-prod)
- [ ] **STOR-04**: Stories are flat (no nesting in v1); a story may have a checklist of sub-tasks but those aren't full stories
- [ ] **STOR-05**: User explicitly approves the story breakdown before prioritization

### Prioritization (PRIO)

- [ ] **PRIO-01**: Stories use Trello-style states: `Backlog`, `In Progress`, `Done` (recorded as a status field in each story file or in the index)
- [ ] **PRIO-02**: Maud proposes a priority order for the Backlog, surfacing inter-story dependencies and blockers
- [ ] **PRIO-03**: User explicitly approves the priority order before iteration begins
- [ ] **PRIO-04**: User can re-prioritize at any time (between stories, mid-story, or via audible)

### Iteration (ITER)

- [ ] **ITER-01**: When a story enters `In Progress`, Maud runs an iteration loop: optional JIT deeper-design step, write tests where they make sense, implement, run tests, commit
- [ ] **ITER-02**: The iteration loop shape (TDD-first vs tests-alongside, manual UI verification cadence, playtest loop, etc.) is project-type-dependent and was determined during research
- [ ] **ITER-03**: User can invoke a per-story mini-design loop (JIT) when a story warrants deeper design before implementation; this is opt-in, not mandatory
- [ ] **ITER-04**: Each story is built on its own git branch (or commit-per-checklist-item where appropriate per project type), with sensible commit messages
- [ ] **ITER-05**: Tests are run before any commit and must pass (or the user must explicitly override)
- [ ] **ITER-06**: All iteration work is captured in `.maud/log.md` for audit

### Verification (VER)

- [ ] **VER-01**: After iteration completes for a story, Maud spawns automated verification agents in parallel: code review, test coverage check, QA, security scan
- [ ] **VER-02**: Verification criteria for each project are determined during the research phase (not hardcoded)
- [ ] **VER-03**: Verification results are summarized and presented to the user
- [ ] **VER-04**: After all automated passes succeed (or are explicitly overridden), the user does final manual verification
- [ ] **VER-05**: User explicitly approves the story before it moves to `Done`
- [ ] **VER-06**: On approval, the story is "deployed" in whatever sense makes sense for the project (merge to main, deploy to staging, deploy to prod, etc.) — the deployment action is project-type-dependent

### Phase Gates (GATE)

- [ ] **GATE-01**: User approval gates exist at the end of every phase and the end of every story — not at finer granularity
- [ ] **GATE-02**: Every phase gate produces a persistent artifact (file or directory) capturing all decisions and outputs of that phase; a phase cannot complete without producing its artifact (the anti-vaporware invariant)
- [ ] **GATE-03**: Approval gates use open-ended elicitation, not multiple-choice rubber-stamping — the user must be able to add notes, request revisions, or send Maud back
- [ ] **GATE-04**: Approval gates present the full produced artifact to the user (path, contents, summary)
- [ ] **GATE-05**: User can reject, revise, or rerun at any gate; Maud loops until the user approves
- [ ] **GATE-06**: Within a phase, Maud may ask the user for *direction* (e.g. "what should I research?") without that constituting a heavy approval gate; only end-of-phase / end-of-story carry the formal verification protocol

### Audibles (AUD)

- [ ] **AUD-01**: At any user-facing moment, the user can add a story, reorder priorities, change scope, or remove a story
- [ ] **AUD-02**: Audibles are most natural during manual verification but are accepted at any conversation turn
- [ ] **AUD-03**: An audible that materially changes scope triggers a return to the appropriate earlier phase (e.g. adding a feature may require updating design)
- [ ] **AUD-04**: Audibles are recorded in `.maud/log.md`

### Project-Type Awareness (PTA)

- [ ] **PTA-01**: Maud detects (or asks about) project type during initialization
- [ ] **PTA-02**: Project type affects: research dimensions proposed, order of architecture vs design, design artifact format, iteration loop shape, verification criteria, deployment action
- [ ] **PTA-03**: Project type is recorded in `.maud/profile.md` so all subsequent agents and phases consume the same shape
- [ ] **PTA-04**: v1 supports the project type of the bootstrap test target end-to-end (currently a simple JS canvas game — web + game development; subject to change). Other project types are recognized with an explicit "explain what you want" fallback rather than a hardcoded template
- [ ] **PTA-05**: For *every* phase, Maud runs a meta-step ("what does this phase mean for this project?") as soon as it has enough information to answer — not just for design. Meta-decisions are written to `.maud/profile.md`. Examples: "what does research mean for a JS canvas game?", "what does verification mean for a CLI plugin?", "what does deployment mean here?"
- [ ] **PTA-06**: Meta-decisions are revisable — if the project changes shape mid-flight, Maud re-runs the relevant phase's meta-step and updates `profile.md` (with user approval)

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Multi-Type Support (MTS)

- **MTS-01**: Full support for web app project type (screens, browser-verify iteration, deploy-to-staging)
- **MTS-02**: Full support for iOS app project type (wireframes, simulator-verify, TestFlight deploy)
- **MTS-03**: Full support for game project type (sprite sheets, playtest loop, build-and-distribute)
- **MTS-04**: Full support for library project type (API design, integration tests, package publish)

### Brownfield (BR)

- **BR-01**: Maud detects existing code and offers a brownfield flow
- **BR-02**: Codebase mapping pass ingests existing architecture, conventions, patterns
- **BR-03**: Inferred Validated requirements seeded from existing code
- **BR-04**: Architecture/design artifacts respect existing patterns rather than re-inventing

### Distribution (DIST)

- **DIST-01**: Maud is published to the Claude Code marketplace with a `marketplace.json` entry
- **DIST-02**: Versioned releases with a CHANGELOG
- **DIST-03**: README documenting install, walkthrough, FAQ
- **DIST-04**: A "first-run tutorial" that demonstrates the full flow

### Productization (PROD)

- **PROD-01**: Web GUI / Trello-style visual interface for stories
- **PROD-02**: Multi-user collaboration features
- **PROD-03**: Integrations with external PM tools (Trello / Jira / Linear sync)
- **PROD-04**: Milestones / epics / nested stories

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Brownfield projects | Greenfield-only for v1; brownfield is a clean v2 scope (different research domain, codebase mapping needed) |
| Milestones / epics | Flat priority-ordered story list is sufficient for v1; a story can be "launch to production" |
| Nested stories | Checklists per story are sufficient; nesting is data-model complexity for unclear value |
| Web app / GUI / Trello visual interface | CLI-only via Claude Code is the entire UX in v1; GUI is a productization step |
| Marketplace publication | Local-only install for v1; reduces release/distribution complexity while iterating |
| Multi-user / collaboration | Solo workflow only; multi-user would change the entire product |
| External PM tool integrations | Far-future productization; not relevant to solo dogfooding |
| Multiple-choice rubber-stamp prompts | Antithetical to core value; rejected by design |
| Hard rules / fixed numbers ("≤10 prompts per project," "5 phases per roadmap," etc.) | The number of prompts, dimensions, phases, stories, and so on is project- and user-dependent — not a quota |
| Time estimates / velocity / story points | Meaningless for solo + AI; PROJECT.md explicitly excludes |
| Concurrent `/maud` sessions on the same project | v1 is single-session sequential — locking / concurrency deferred until a real need surfaces |
| Team / multi-user collaboration | v1 is for solo individual developers and founders only |
| Pre-flight constitution / principles artifact | Adds ceremony before user expresses intent; constraints accumulate naturally in PROJECT.md |
| Persona-flavored agents (PM/Architect/Dev personas) | BMAD-style persona theatre; Maud uses functional agents only |
| Built-in starter templates ("greenfield SaaS," etc.) | Fights against project-type-aware flexibility; templates pre-decide what should emerge from questioning |
| Always-on autonomous mode | Directly contradicts "user verifies every phase boundary" core value |
| Real-time multi-agent execution / agent teams | Parallelism is for verification, not invention; defeats gating model |
| A formal "v2" big-bang release | Maud iterates while being used to build other things; no big version cut |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| PI-01 through PI-06 | TBD | Pending |
| STATE-01 through STATE-07 | TBD | Pending |
| INIT-01 through INIT-04 | TBD | Pending |
| RES-01 through RES-04 | TBD | Pending |
| ARCH-01 through ARCH-05 | TBD | Pending |
| DSGN-01 through DSGN-04 | TBD | Pending |
| STOR-01 through STOR-05 | TBD | Pending |
| PRIO-01 through PRIO-04 | TBD | Pending |
| ITER-01 through ITER-06 | TBD | Pending |
| VER-01 through VER-06 | TBD | Pending |
| GATE-01 through GATE-06 | TBD | Pending |
| AUD-01 through AUD-04 | TBD | Pending |
| PTA-01 through PTA-06 | TBD | Pending |

**Coverage:**
- v1 requirements: 61 total across 13 categories
- Mapped to phases: 0 (filled during roadmap creation)
- Unmapped: 61 ⚠️

---
*Requirements defined: 2026-05-08*
*Last updated: 2026-05-08 after initial definition*
