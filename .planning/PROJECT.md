# Maud

## What This Is

Maud is a Claude Code plugin for project management, inspired by GSD but adapted to Ryan's personal style. It generates a `.maud/` directory of human-readable markdown files that describe and drive a project from idea to launch. Highly extensible and adaptable per-project, with meta-introspection so the framework can reason about its own structure.

## Core Value

Take a project from idea to launch using local, human-readable markdown that the user can edit at any time — no opaque tooling, no cloud dependencies.

## Requirements

### Validated

- ✓ The hand-crafted `.maud/` reference structure exists and demonstrates the desired shape — used as the spec for what `/maud:plan` should produce.

### Active

- [ ] Fix all markdown links in `.maud/` so they're consistent
- [ ] GSD-style critique of the Maud approach (unvarnished thoughts on structure, gaps, missing tools)
- [ ] Bare-bones Claude Code plugin scaffolding so `/maud:*` commands run
- [ ] `/maud:plan` — generate `.maud/index.md` (the project file) via deep questioning
- [ ] `/maud:plan` — determine project-specific `.maud/planning/` structure (meta-cognition step)
- [ ] `/maud:plan` — run research agents to populate `.maud/planning/` subfolders
- [ ] `/maud:plan` — synthesize research into "the plan" at `.maud/planning/index.md`
- [ ] `/maud:plan` — (optional) generate design (skip / Claude-generated / external tooling / human designer)
- [ ] `/maud:plan` — generate `.maud/stories/` and populate the backlog from the plan
- [ ] `/maud:build` — pop the top story off the backlog
- [ ] `/maud:build` — ask the user clarifying questions about the story
- [ ] `/maud:build` — build the story in Claude Code
- [ ] `/maud:build` — validation agents verify the work
- [ ] `/maud:build` — present completed work to user for approval
- [ ] `/maud:build` — on approval, merge to main and launch to production if relevant

### Out of Scope

- Cloud-hosted state — Maud is local-first, all artifacts live in `.maud/`
- Non-markdown formats for project artifacts — readability and direct user editing is the point
- Rebuilding Maud with itself in this milestone — that's the next milestone after the plugin works
- Using Maud on a non-Maud project in this milestone — same, deferred to a future milestone

## Context

- Bootstrapping with GSD as a "flywheel": GSD builds the first Maud plugin, then Maud rebuilds itself, then Maud builds an unrelated project (TBD).
- The `.maud/` directory in this repo was hand-crafted by Ryan as the canonical example/spec. Treat it as ground truth for what `/maud:plan` should produce.
- The backlog at `.maud/stories/index.md` is the source of work. Each backlog story (and each sub-task file inside `plan-command/` and `build-command/`) maps to a GSD phase.
- Old GSD scraps were already removed — this is a fresh start.
- Claude Code plugin docs: https://code.claude.com/docs/en/plugins

## Constraints

- **Platform**: Claude Code plugin — must work within the Claude Code plugin model (slash commands, agents, hooks)
- **Storage**: Local filesystem only — `.maud/` directory, no external services
- **Format**: Human-readable markdown — every artifact must be directly editable by the user
- **Editability**: User edits files directly to change the project; "approved" / chat is the trigger to advance, not the medium for changes

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Bootstrap with GSD before self-hosting on Maud | Maud doesn't exist yet; need a working planning/execution system to build it | — Pending |
| One phase per backlog story (and per sub-task file) | User wants execution to map 1:1 to the hand-crafted backlog | — Pending |
| Build-command sub-tasks split into phases despite no separate files | User explicitly chose to split each checklist item into its own phase | — Pending |
| Skip per-phase research workflow agent | `.maud/` already has all needed context; research would be padding | — Pending |
| Sequential plan execution within phases | User preference | — Pending |

---
*Last updated: 2026-05-10 after initialization*
