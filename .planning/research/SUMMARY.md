# Project Research Summary

**Project:** Maud
**Domain:** Claude Code plugin — AI-assisted software development orchestrator
**Researched:** 2026-05-08
**Confidence:** HIGH

## Executive Summary

Maud is a Claude Code plugin built entirely from markdown files, JSON manifests, and optional bash scripts — there is no runtime to choose and no framework to evaluate. The "stack" decision that matters most is the plugin format itself: a `.claude-plugin/plugin.json` manifest, a single stateful `commands/maud.md` orchestrator, and a set of worker agents in `agents/`. The architectural constraint that shapes everything else is fundamental and platform-imposed: **subagents cannot spawn other subagents**, so the entire control loop — state reads, phase transitions, agent spawning, user gates — must live in the `/maud` slash command's main conversation context. Everything downstream of that constraint is validated pattern: agents are leaf workers, state lives in `.maud/` on disk, git is the backup layer.

The recommended approach is to build Maud in strict dependency order — scaffold the plugin, then the state machine, then agents one at a time — and to ship v1 with a single concrete project-type vertical (CLI/plugin) fully working before generalizing. The core differentiator is mandatory user verification at every phase boundary, which is both the product thesis and the primary source of implementation risk: too few gates and Maud becomes GSD (decisions happen silently); too many gates and users experience approval fatigue and stop reading before approving. The calibration requires tiering prompts visually — heavy open-ended elicitation for consequential decisions, light confirmation for mechanical ones, silent notify for internal agent work.

The dominant risks are not technical — the plugin format is well-documented and GSD is a working reference. The risks are product: (1) project-type-aware flexibility becoming vaporware if abstractions are built before concrete verticals, (2) the bootstrap trap of trying to use Maud-on-Maud before it is stable enough, and (3) the six architectural anti-patterns borrowed from GSD's failure modes that Maud is explicitly designed to avoid.

## Key Findings

### Recommended Stack

Maud's "stack" is the Claude Code plugin format. Core format choices: `.claude-plugin/plugin.json` manifest (JSON, minimal, auto-discovery handles the rest), `commands/maud.md` (single user-facing slash command, markdown with YAML frontmatter), `agents/<name>.md` (worker agents, same format), and `.maud/` in the user's project directory for all state (read/written via standard `Read`/`Write`/`Bash`/`Edit` tools). No MCP server, no hooks in v1, no skills. The plugin loads locally via `claude --plugin-dir ./maud` with hot-reload via `/reload-plugins`.

**Core technologies:**
- `commands/maud.md`: stateful orchestrator — controls flow, spawns agents, mediates user gates
- `agents/<name>.md`: leaf workers — spawned via `Agent` tool; never spawn other agents
- `.maud/state.json`: machine-canonical state — the dispatcher input read at every `/maud` invocation
- `.maud/` directory: all project state — `PROJECT.md`, `profile.md`, `research/`, `architecture.md`, `design/`, `stories/`, `log.md`
- `templates/` and `references/`: plugin-internal prompt fragments loaded via `@${CLAUDE_PLUGIN_ROOT}/...`

Critical rules: `plugin.json` must live in `.claude-plugin/`; `commands/`, `agents/`, `hooks/` must be at plugin root (most common mistake per official docs). Tool `Task` was renamed to `Agent` in v2.1.63. Kebab-case naming required — underscores rejected by schema.

### Expected Features

**Must have (table stakes):**
- Single stateful `/maud` command that reads `.maud/` and resumes with "you are here" summary
- Initialization: idea → clarifying questions → MVP-defined PROJECT.md
- Research as distinct frontloaded phase with two user gates (approve dimensions; approve synthesis)
- Architecture artifact with user gate; design as own phase with project-type-appropriate format and user gate
- Story extraction from approved design + architecture into flat priority-ordered list
- Trello-style states: Backlog / In Progress / Done; iteration loop with tests, commits, verification
- Automated agentic verification (code review, test coverage, QA, security) + final manual user gate
- Resumability, append-only audit trail, subagent spawning, git integration, audibles

**Should have (competitive differentiators):**
- Mandatory user approval at every phase boundary — no competitor does this consistently
- Project-type-aware structure — no competitor adapts research dims, design format, iteration shape, verification criteria per project type
- Design as distinct artifact separate from architecture
- Flat priority-ordered story list (Trello model) — unique in AI orchestration tool space
- Tiered verification prompts (heavy / light / notify)

**Defer:**
- Security scan as separate pass (v1.x), more project types (v1.x), brownfield support (v2), web GUI, marketplace publication, multi-user collaboration (all v2+)

### Architecture Approach

Orchestrator-in-command, workers-in-agents pattern with three deliberate divergences from GSD: project-type-aware `profile.md` driving dynamic structure; flat story list instead of roadmap-of-phases; mandatory `log.md` contract for every spawned agent. The supervisor loop lives in `commands/maud.md` — platform-imposed, not a design choice. State is machine-canonical JSON for the dispatcher, human-readable markdown for everything else. Git is the backup layer.

**Major components:**
1. `commands/maud.md` (orchestrator) — reads `state.json`, dispatches on phase, spawns agents, mediates all gates, writes state on advance
2. `agents/*.md` (leaf workers) — researcher, designer, architect, story-extractor, prioritizer, iterator, four parallel verifiers; each gets pre-assembled inlined context, writes output files, appends log entry, returns structured marker
3. `.maud/` state directory — `state.json` (dispatcher), `PROJECT.md` (frozen spec), `profile.md` (project-type shape), `research/`, `architecture.md`, `design/`, `stories/INDEX.md`, `verifications/`, `log.md`
4. `templates/` + `references/` — plugin-internal, not user state
5. Git — mandatory; every gate-approved transition commits

Key patterns (all validated in GSD source): pre-assembled context injection (not `@` references in agent prompts), parallel spawn for independent work, structured return markers, verification-gated phase transitions.

### Critical Pitfalls

1. **Multiple-choice rubber-stamp approvals** — GSD's confirmed failure mode. Prevention: open-ended elicitation on consequential decisions; minimum-content threshold; reject single-token approvals with targeted follow-up.
2. **Verification fatigue from over-gating** — inverse failure mode. Gate at phase boundaries only. Target: fewer than 10 heavy prompts per project lifecycle. Tier prompts visually.
3. **Flexibility becoming vaporware** — abstractions before concrete verticals. Prevention: v1 ships one vertical (CLI/plugin) fully end-to-end. Generalize only after third concrete instance.
4. **Per-phase research creep** — re-running research in later phases defeats frontloaded thinking. Prevention: dimensions set once; later phases consume `.maud/research/` artifacts, never regenerate.
5. **Context pollution across agents** — hallucinated decisions from over-broad context bundles. Prevention: named, minimal, file-path-based context bundle per spawn; agents must cite files for any decision.
6. **State file corruption** — partial writes brick the project. Prevention: atomic writes (temp + fsync + rename), git backup, lock file, validation on resume.
7. **Dogfooding bootstrap trap** — using unstable Maud to fix Maud. Prevention: name the concrete second project before starting; Maud-on-Maud disallowed during v1.

## Implications for Roadmap

### Phase 1: Plugin Scaffold and State Machine
**Rationale:** Every other phase depends on the plugin loading and state dispatching correctly. Infrastructure with no agents; smallest surface area while validating the dev loop.
**Delivers:** Loadable plugin, `state.json` dispatch, init flow, atomic-write helpers, lock file, `log.md` infrastructure.
**Avoids:** Pitfall 8 (state corruption), Pitfall 11 (command proliferation)
**Research flag:** Standard pattern — GSD and `plugin-dev` plugin are direct references. No deeper research needed.

### Phase 2: Approval Mechanic and Phase Transition Protocol
**Rationale:** Every subsequent phase reuses this. Must be correct before any content phases are built. Highest-risk product decision.
**Delivers:** `AskUserQuestion` gate pattern with three options, visual tiering (heavy / light), minimum-content threshold, single-token-approval rejection, prompt count instrumentation.
**Avoids:** Pitfall 1 (rubber-stamping), Pitfall 5 (verification fatigue)
**Research flag:** Novel calibration — no reference implementation. Plan for iteration after first complete project run.

### Phase 3: Research Phase (First Agent Pipeline)
**Rationale:** First phase with real agent spawning. Validates the parallel-spawn and structured-return-marker patterns that all later agents reuse. Earliest meaningful content dependency.
**Delivers:** `maud-researcher` agent, parallel orchestration of N researchers, `maud-research-synthesizer` agent, two user gates, context-bundle protocol.
**Avoids:** Pitfall 2 (research creep — dimensions set once here), Pitfall 7 (context pollution — bundle protocol established here), Pitfall 9 (hallucinated decisions — citation protocol established here)
**Research flag:** Well-documented. GSD's `gsd-project-researcher.md` is a direct reference. No deeper research needed.

### Phase 4: Architecture and Design Phases
**Rationale:** Both depend on research; both feed story extraction. Bundled because they share the same gate pattern and the order between them is profile-driven.
**Delivers:** `maud-architect` agent, `maud-designer` agent, meta-step ("what does design mean for this project?"), profile-driven branching, user-iteration loop on design, CLI and web-app profile templates.
**Avoids:** Pitfall 4 (no design phase — required and unskippable), Pitfall 3 (rigid templates — profile drives format)
**Research flag:** The meta-step is the most novel piece of Maud with no reference implementation. Requires careful prototyping with at least 2 concrete project types before moving on.

### Phase 5: Story Extraction, Prioritization, and Iteration Loop
**Rationale:** Depends on approved design + architecture. These form a tight dependency chain best established together.
**Delivers:** `maud-story-extractor`, `maud-prioritizer`, `maud-iterator` agents, git integration, JIT mini-design loop as opt-in, audibles via `$ARGUMENTS` parsing.
**Avoids:** Pitfall 3 (rigid templates — flat list), Pitfall 2 (no per-story research re-runs)
**Research flag:** Git branching strategy needs a concrete default in profile.md. Iterator + git is the most complex agent interaction — plan for `maxTurns` cap and checkpoint markers.

### Phase 6: Parallel Verification Suite
**Rationale:** Depends on iteration loop delivering committed code. Four independent verifiers run in parallel — most complex parallel spawn in Maud.
**Delivers:** Four verifier agents (security, code-review, test-coverage, qa), parallel orchestration, synthesis of all four reports, story-completion gate, Done transition with deploy step per profile.
**Avoids:** Pitfall 5 (verification fatigue — one gate at story completion), Pitfall 16 (unclear done definitions — success criteria enforced)
**Research flag:** Per-project-type verification criteria need concrete definition before this phase is testable. Security verifier may be deferrable to v1.x.

### Phase 7: Project-Type Profiles Catalog and Bootstrap Test
**Rationale:** Profiles can only be correctly designed after the full loop is working. Finalizes v1 vertical and names the bootstrap test.
**Delivers:** Finalized `cli-plugin.md` and `web-app.md` profiles, named concrete second project, v1 declaration criterion, README, LICENSE.
**Avoids:** Pitfall 6 (flexibility-as-vaporware), Pitfall 10 (bootstrap trap), Pitfall 18 (hypothetical-user features)
**Research flag:** Mechanical once the loop is proven. Document clearly what is and is not implemented for each profile.

### Phase Ordering Rationale
- Infrastructure before agents (Phases 1–2 before 3–7): state schema bugs and approval mechanic errors compound across all later phases
- Research before design/architecture (Phase 3 before 4): hard dependency — both consume research artifacts
- Design + architecture before stories (Phase 4 before 5): hard sync point — story extraction requires both approved
- Iteration before verification (Phase 5 before 6): verifiers read committed code; iterator produces it
- Full loop before profiles (Phase 6 before 7): profiles encode what each phase means; only writable correctly after all phases are working
- One concrete vertical before generalizing: YAGNI — generalize only after third concrete instance

### Research Flags

Phases needing deeper research or calibration:
- **Phase 2 (Approval Mechanic):** Novel calibration problem. Plan for post-first-project iteration.
- **Phase 4 (Design Phase):** Meta-step is novel. Requires careful prototyping with real project types.
- **Phase 5 (Iteration Loop):** Git strategy needs concrete default. Iterator + git is Maud's most complex agent.
- **Phase 6 (Verification Suite):** Per-project-type criteria need concrete definition before testable.

Phases with standard patterns:
- **Phase 1 (Plugin Scaffold):** GSD + `plugin-dev` are direct references. Minimal risk.
- **Phase 3 (Research Pipeline):** GSD researcher is a working reference. Confirmed patterns.
- **Phase 7 (Profiles Catalog):** Mechanical discipline, not research.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Plugin format verified against 2026 official docs and locally-cached canonical examples. GSD is a working reference on disk. Skills-vs-commands convergence is MEDIUM but `commands/` for v1 is well-supported by reference plugins. |
| Features | HIGH (GSD) / MEDIUM (competitive) | GSD examined directly. Spec Kit, Kiro, Cursor, Cline, Aider confirmed with multiple sources. Devin, BMAD-METHOD MEDIUM (single-source synthesis). Categorical findings robust. |
| Architecture | HIGH | Orchestrator-in-command constraint confirmed in official docs and GSD source. Specific schema choices are MEDIUM until validated by use. |
| Pitfalls | HIGH (GSD failure modes) / MEDIUM-HIGH (industry) | Four GSD failure modes are product requirements. Approval fatigue (93% auto-approval) confirmed by multiple sources. State corruption confirmed by GitHub issue #29051. |

**Overall confidence:** HIGH

### Gaps to Address

- `/maud:maud` vs `/maud` invocation UX (GitHub issue #15882) — validate during Phase 1 whether typeahead makes this tolerable
- Concurrent `log.md` writes from parallel agents — validate atomicity empirically during Phase 3
- Audible intent parsing from `$ARGUMENTS` — test with real user phrasing during Phase 5, adjust keyword set
- Approval threshold calibration — instrument a real project during Phase 2 to count prompts and calibrate
- Profile catalog completeness — document CLI (full) / web-app (partial) / game (recognized, not implemented) boundary explicitly
