# Maud

## What This Is

Maud is a Claude Code plugin that walks a developer from idea to shipped software using real-life project management techniques — frontloaded discussion, research, design, and architecture; stories as the unit of work; iterative build with explicit user verification at every phase transition. v1 is a CLI plugin storing all state in `.maud/`, intended for solo developers and founders building greenfield projects.

## Core Value

Maud takes a developer from idea to shipped software with deep PM thinking baked in upfront — and **the user explicitly verifies every phase transition before moving forward**, so no consequential decision happens behind their back. That single principle is the antidote to GSD's failure mode of "wild decisions you never heard about" and is the thing that, if it fails, makes Maud worthless.

## Requirements

### Validated

(None yet — ship to validate)

### Active

<!-- All hypotheses until shipped and validated. -->

**Entry point + state**

- [ ] User runs `/maud` from inside a Claude Code session in their project directory and Maud takes over
- [ ] `/maud` is a single, stateful command — re-running it reads `.maud/` and resumes from the current step
- [ ] On resume, Maud "brings the user up to speed" (current phase, story counts: Done / In Progress / Backlog) before continuing
- [ ] All state lives in `.maud/` in the user's project (no global state)

**Initialization**

- [ ] User writes an initial prompt describing what they want to build
- [ ] Maud asks clarifying questions until it has enough to define an MVP
- [ ] Initialization ends with explicit user verification before advancing

**Research**

- [ ] Maud proposes research dimensions appropriate for this project type (competitor research, tech stack, UI patterns, feasibility, etc.) — user approves/edits before any research runs
- [ ] Research executes (using agents) and writes findings to `.maud/research/`
- [ ] Maud synthesizes findings into recommendations and presents them to the user
- [ ] User must approve the synthesized recommendations before advancing

**Design**

- [ ] Design artifacts live in `.maud/design/`; the *format* of design artifacts is determined by project type (e.g. screens/pages for a web app; sprite sheets/maps for a game; structured spec for a CLI)
- [ ] Maud first works out *what design means for this project* (this may itself be a research/brainstorm step), then generates design artifacts
- [ ] User and Maud iterate on the design until the user approves it
- [ ] If the user already has a design, Maud uses it instead of generating from scratch

**Architecture**

- [ ] Architecture is its own artifact (e.g. `.maud/architecture.md`) covering tech stack and system shape, including local dev and production environments
- [ ] Architecture is informed by research and design; the *order* between architecture and design is project-type-dependent (Maud decides)
- [ ] Architecture explicitly avoids premature optimization — keep it as simple as possible
- [ ] User must approve architecture before advancing

**Stories**

- [ ] Stories are extracted from approved design + architecture and live in `.maud/stories/` (one file per story, plus an index)
- [ ] Each story is human-readable, not exhaustive (assumes common sense), and contains: title, description, references to relevant design docs, success criteria, optional checklist of sub-tasks
- [ ] Each story is individually buildable and shippable (the meaning of "shippable" is project-dependent — could be merge-to-main early on, deploy-to-prod later)
- [ ] Stories are flat (no nesting in v1) — a story may have a checklist of sub-tasks, but those aren't full stories
- [ ] User must approve the story breakdown before advancing

**Prioritization**

- [ ] Stories use Trello-style states: `Backlog` / `In Progress` / `Done`
- [ ] Maud orders stories by priority and surfaces blockers/dependencies to enable parallelization
- [ ] User must approve the priority order before iteration begins
- [ ] User can change priority, add stories, or remove stories at any time

**Iteration (the build loop)**

- [ ] When a story enters `In Progress`, Maud runs the iteration loop: optional JIT deeper-design step (the per-story mini-loop) → write tests where they make sense → implement → run tests → commit
- [ ] The exact iteration shape (TDD-first vs tests-alongside, manual UI verification cadence, playtest loop, etc.) is project-type-dependent and determined during research
- [ ] Git is used effectively: branch per story (or commit per checklist item where appropriate), with sensible commit messages
- [ ] User can call audibles at any point during user-facing interaction (add a story, reorder, change scope)

**Verification**

- [ ] All verification steps are automated agentic passes: code review, test coverage check, QA, security scanning
- [ ] Specific verification criteria for each project are determined during the research phase
- [ ] After all automated passes succeed, the user does the final manual verification
- [ ] User explicitly approves the story before it moves to `Done`
- [ ] On approval, the story is "deployed" in whatever sense makes sense for the project (merge to main, deploy to staging, deploy to prod, etc.)

**Agent log + context management**

- [ ] `.maud/log.md` (or equivalent) is an audit trail where every agent writes a short note: what it did, what it found, what it decided
- [ ] Long-running operations (research, verification, iteration) spawn isolated agents so the user's main context window stays clean
- [ ] Relevant context from the main conversation is persisted to disk so other agents (and resumed sessions) can pick up where things left off

**Phase transitions**

- [ ] Every phase outlined above ends with explicit user verification before advancing — Maud never silently moves to the next step

### Out of Scope

- **Brownfield projects** — greenfield only for v1; brownfield support is a future expansion
- **Milestones / epics** — v1 keeps a flat priority-ordered story list; one of the stories can be "launch to production" or whatever fits
- **Nested stories** — checklists are sufficient for v1; revisit nesting only if reality demands it
- **Web app / GUI / Trello-like visual interface** — CLI-only for v1; richer UI is a long-term productization step
- **Marketplace publication** — local-only install for v1; publish to Claude Code marketplace later
- **Multi-user / collaboration features** — single-user solo workflow for v1
- **Integrations with external PM tools** (Trello/Jira/Linear/etc.) — far-future productization
- **A formal "v2" release** — Maud will be improved iteratively while being used to build other things; no big-bang version cut

## Context

- **Built with GSD as the bootstrap.** Maud's v1 is being implemented using GSD itself, then once functional Maud will be used to build subsequent projects (and to iteratively improve itself).
- **Inspired by GSD's strengths.** Local context management, agent spawning with scoped context, conversation-to-disk persistence, resumability — these all carry over.
- **Designed in reaction to GSD's weaknesses.** GSD is too rigid (arbitrary phase counts, fixed criteria templates), interrupts the user too thinly (multiple-choice prompts that feel like rubber-stamping), makes consequential decisions without explicit approval, has no design phase, and forces a per-phase research+plan cycle when frontloaded thinking would be more honest.
- **Real-life PM lineage.** The flow draws on 20 years of freelance + startup PM experience: heavy frontloaded thinking, story-based execution, priority-driven iteration, audibles welcome, software is never truly "done."
- **The Trello mental model.** The eventual UX vision (post-v1) is something Trello-like — three columns, manual movement, lightweight cards. v1 just stores this on disk.

## Constraints

- **Distribution**: Claude Code plugin format (slash commands + agents + `.claude-plugin` manifest) — this is what the platform supports
- **Local-only**: All state in `.maud/` in the user's project; no global state, no remote services in v1
- **CLI-only UX**: No GUI in v1 — Claude Code session is the entire interface
- **Greenfield only**: v1 assumes an empty project directory; brownfield is explicitly excluded
- **Single-user**: No collaboration / multi-user state
- **Philosophy: flexibility over rigidity**: Maud should determine structure dynamically (number of research dimensions, design format, iteration loop shape, verification criteria) based on project type rather than imposing templates
- **Self-host bootstrap**: v1 is built with GSD; the success criterion is that Maud-built-with-GSD is good enough to build the *next* project on its own

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Single `/maud` command, stateful | Reduces cognitive load; resumable from any phase via `.maud/` state | — Pending |
| Stories as unit (flat list, no phases) | Each story independently shippable; matches Trello mental model; allows audibles | — Pending |
| Frontload all discussion / research / design / architecture | Per-phase mini-loops in GSD produce "wild decisions"; upfront holistic thinking is the bulk of real PM value | — Pending |
| Per-story JIT mini-loop is allowed | Reality diverges from design; need an escape hatch when a story warrants deeper work, without making it mandatory ceremony | — Pending |
| **Explicit user verification at every phase boundary** | The core differentiator from GSD — no consequential decision happens silently | — Pending |
| Project-type-aware structure (research dims, design format, iteration loop, verification criteria) | Web vs iOS vs CLI vs game have radically different needs; framework should adapt rather than impose | — Pending |
| Trello-style states: Backlog / In Progress / Done | Matches the user's actual workflow; simple; can evolve later | — Pending |
| Architecture as its own artifact; order vs design is project-dependent | Sometimes architecture feeds design choices, sometimes design drives architecture — Maud decides per project | — Pending |
| All verification automated agentically; user manual verification is the final gate | Automation handles toil; user retains final authority | — Pending |
| `.maud/log.md` as agent audit trail | User can review what agents checked, found, and decided across stories | — Pending |
| Built with GSD as v1 bootstrap | Eat your own dogfood; GSD is sufficient to ship v1 | — Pending |
| Local-only install for v1 | Marketplace adds release/distribution complexity — defer until product is good | — Pending |

---
*Last updated: 2026-05-08 after initialization*
