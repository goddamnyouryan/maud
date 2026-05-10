---
phase: 01-fix-maud-markdown-links
plan: 01
subsystem: docs
tags: [markdown, links, git-mv, readme]

# Dependency graph
requires: []
provides:
  - "9 .maud/**/index.md files renamed to README.md with git history preserved"
  - "All in-.maud/ markdown links rewritten to standard relative paths"
  - "Broken typo link (/complete/scrap-gsd.md] repaired to (complete/scrap-gsd.md)"
  - "Prose mentions of index.md -> README.md updated across .maud/ and .planning/"
  - "All 19 .maud/ links verified to resolve on disk via link-walk script"
affects:
  - "All future phases referencing .maud/README.md as the project file"
  - "Phase 4 (generate project file), Phase 7 (generate the plan), Phase 9 (generate stories)"
  - "Phases 10-15 (build-command stories reference .maud/stories/backlog/build-command/README.md)"

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "README.md convention for .maud/ directory indexes (GitHub auto-renders for directory links)"
    - "Standard relative paths for all .maud/ markdown links (no leading slash)"

key-files:
  created:
    - ".maud/README.md (was .maud/index.md)"
    - ".maud/planning/README.md (was .maud/planning/index.md)"
    - ".maud/planning/architecture/README.md"
    - ".maud/planning/competitors/README.md"
    - ".maud/planning/design/README.md"
    - ".maud/planning/technology/README.md"
    - ".maud/stories/README.md (was .maud/stories/index.md)"
    - ".maud/stories/backlog/build-command/README.md"
    - ".maud/stories/backlog/plan-command/README.md"
  modified:
    - ".maud/stories/backlog/plan-command/generate-project-file.md"
    - ".maud/stories/backlog/plan-command/generate-planning-structure.md"
    - ".maud/stories/backlog/plan-command/generate-the-plan.md"
    - ".maud/stories/backlog/plan-command/generate-stories.md"
    - ".maud/stories/backlog/plan-command/research-agents.md"
    - ".planning/ROADMAP.md"
    - ".planning/REQUIREMENTS.md"
    - ".planning/PROJECT.md"

key-decisions:
  - "README.md is the canonical index filename for .maud/ directories — GitHub auto-renders it for folder browsing, enabling (folder/) style links to work without a static-site pipeline"
  - "Standard relative paths (e.g., backlog/discuss-maud-with-gsd.md) replace all fake root-relative paths (/backlog/discuss-maud-with-gsd.md)"
  - "Full cascade: .planning/ROADMAP.md, REQUIREMENTS.md, PROJECT.md updated to stay coherent with the renamed files"

patterns-established:
  - "Directory link pattern: [Section](subfolder/) — resolves to subfolder/README.md on GitHub"
  - "File link pattern: [Story](backlog/story-name.md) — standard relative path from file's location"

# Metrics
duration: 110min
completed: 2026-05-10
---

# Phase 1 Plan 1: Fix .maud/ Markdown Links Summary

**9 index.md files renamed to README.md via git mv, all 19 .maud/ links rewritten to standard relative paths, broken typo repaired, and cascade updates applied to ROADMAP/REQUIREMENTS/PROJECT**

## Performance

- **Duration:** ~110 min
- **Started:** 2026-05-10T03:02:50Z
- **Completed:** 2026-05-10T04:52:38Z
- **Tasks:** 3
- **Files modified:** 18 (9 renamed, 9 content-edited)

## Accomplishments

- Renamed all 9 `.maud/**/index.md` files to `README.md` via `git mv` — history preserved, rename detection confirmed (R100)
- Converted all fake root-relative links (`/competitors/`, `/backlog/discuss-maud-with-gsd.md`, etc.) to standard relative paths; link-walk script confirms all 19 links resolve
- Fixed typo `[Carve up remainder of old GSD for scraps](/complete/scrap-gsd.md]` → `(complete/scrap-gsd.md)` — balanced parens, no leading slash
- Updated all prose mentions of `index.md` inside `.maud/` story files and cascaded all path references in `.planning/ROADMAP.md` (20 occurrences), `REQUIREMENTS.md` (3), `PROJECT.md` (3)

## Task Commits

Each task was committed atomically:

1. **Task 1: Rename .maud index.md files to README.md** - `cd733ce` (chore)
2. **Task 2: Rewrite .maud links + repair typo** - `396cc10` (fix)
3. **Task 3: Cascade rename refs into .planning docs** - `83a85eb` (docs)

**Plan metadata:** (pending — this commit)

## Files Created/Modified

**Renamed (git mv — 9 files):**
- `.maud/README.md` — top-level project index
- `.maud/planning/README.md` — planning dimension index + prose updated
- `.maud/planning/architecture/README.md` — architecture subfolder index
- `.maud/planning/competitors/README.md` — competitors subfolder index
- `.maud/planning/design/README.md` — design subfolder index
- `.maud/planning/technology/README.md` — technology subfolder index
- `.maud/stories/README.md` — stories index (project "UI")
- `.maud/stories/backlog/build-command/README.md` — build-command sub-task index
- `.maud/stories/backlog/plan-command/README.md` — plan-command sub-task index (link text updated)

**Content-edited (9 files):**
- `.maud/stories/backlog/plan-command/generate-project-file.md` — prose: index.md -> README.md (2 mentions)
- `.maud/stories/backlog/plan-command/generate-planning-structure.md` — prose: index.md -> README.md (1 mention)
- `.maud/stories/backlog/plan-command/generate-the-plan.md` — prose: index.md -> README.md (2 mentions)
- `.maud/stories/backlog/plan-command/generate-stories.md` — prose: index.md -> README.md (3 mentions)
- `.maud/stories/backlog/plan-command/research-agents.md` — prose: index.md -> README.md (1 mention)
- `.planning/ROADMAP.md` — path refs: index.md -> README.md (20 occurrences)
- `.planning/REQUIREMENTS.md` — path refs: index.md -> README.md (3 occurrences)
- `.planning/PROJECT.md` — path refs: index.md -> README.md (3 occurrences)

## Decisions Made

- README.md convention for .maud/ directories — GitHub auto-renders this filename for folder browsing, making `(folder/)` style directory links work without a static-site pipeline. No other standard markdown renderer auto-renders `index.md`.
- ROADMAP.md line 25 glob pattern (`.maud/**/index.md`) intentionally left unchanged — it documents the name of what was renamed, not a path reference. The `.maud/[^[:space:]`]*index\.md` grep correctly identifies this as a false positive.
- ROADMAP.md line 22 historical typo `(/complete/scrap-gsd.md]` kept verbatim — it's quoted documentation of the broken link that was repaired, not an instruction to recreate it.

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None — no external service configuration required.

## Next Phase Readiness

- `.maud/` link graph is fully coherent — all 19 links resolve on disk
- `.planning/ROADMAP.md`, `REQUIREMENTS.md`, `PROJECT.md` are consistent with the renamed files
- Phase 2 (Discuss Maud with GSD) can proceed immediately — it reads `.maud/stories/README.md` as the story source and that file is clean

---
*Phase: 01-fix-maud-markdown-links*
*Completed: 2026-05-10*
