# State: Maud

## Project Reference

See: `.planning/PROJECT.md` (updated 2026-05-10)

**Core value:** Take a project from idea to launch using local, human-readable markdown that the user can edit at any time.
**Current focus:** Phase 2 — Discuss Maud with GSD

## Milestone

**v1** — Build the Maud Claude Code plugin (`/maud:plan` + `/maud:build`) using GSD as the bootstrapping framework.

## Current Phase

**Phase 2: Discuss Maud with GSD** — Pending plan

## Phase Progress

| # | Phase | Status |
|---|-------|--------|
| 1 | Fix `.maud/` markdown links | ✓ Complete |
| 2 | Discuss Maud with GSD | ○ Pending |
| 3 | Setup Claude Code Plugin | ○ Pending |
| 4 | `/maud:plan` — Generate Project File | ○ Pending |
| 5 | `/maud:plan` — Generate Planning Structure | ○ Pending |
| 6 | `/maud:plan` — Run Research Agents | ○ Pending |
| 7 | `/maud:plan` — Generate "The Plan" | ○ Pending |
| 8 | `/maud:plan` — Generate Design (Optional) | ○ Pending |
| 9 | `/maud:plan` — Generate Stories | ○ Pending |
| 10 | `/maud:build` — Pop Top Story | ○ Pending |
| 11 | `/maud:build` — Ask Clarifying Questions | ○ Pending |
| 12 | `/maud:build` — Build the Story | ○ Pending |
| 13 | `/maud:build` — Validation Agents | ○ Pending |
| 14 | `/maud:build` — Present for Approval | ○ Pending |
| 15 | `/maud:build` — Merge & Launch | ○ Pending |

**Progress:** █░░░░░░░░░ 7% (1/15 phases complete)

## Workflow Config

See: `.planning/config.json`

- Mode: yolo
- Depth: standard
- Parallelization: sequential
- Research agent: off
- Plan check: on
- Verifier: on
- Model profile: balanced

## Accumulated Decisions

| Decision | Phase | Rationale |
|----------|-------|-----------|
| README.md convention for .maud/ directories | 01-01 | GitHub auto-renders README.md for folder browsing; enables (folder/) links without a static-site pipeline |
| Standard relative paths for .maud/ links | 01-01 | Leading-slash paths don't resolve in any standard renderer; relative paths work everywhere |
| Full cascade of rename to .planning/ docs | 01-01 | Keeps ROADMAP/REQUIREMENTS/PROJECT coherent with actual file tree |

## Session Continuity

**Last session:** 2026-05-11
**Stopped at:** Phase 1 verified complete; ready to start Phase 2 (Discuss Maud with GSD)
**Resume file:** None

## Notes

- Project is greenfield from a code perspective but the `.maud/` reference structure was hand-crafted by the user — treat it as the spec for what the plugin must produce.
- The user's instruction was explicit: skip questioning, skip research, one phase per backlog story (and per sub-task file). The roadmap above honors that mapping.

---
*Last updated: 2026-05-11 after Phase 1 verification passed*
