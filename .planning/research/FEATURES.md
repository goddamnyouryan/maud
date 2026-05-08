# Feature Research

**Domain:** AI-assisted software development orchestration tool (Claude Code plugin)
**Researched:** 2026-05-08
**Confidence:** MEDIUM-HIGH (GSD examined directly; competitor landscape from 2026 sources)

---

## Scope of This Research

Maud sits at the intersection of three product categories:

1. **AI-assisted dev orchestration / agent frameworks** — GSD, Spec Kit, Aider, Cursor Plan Mode, Cline, Roo Code, Devin, OpenHands, Kiro, BMAD-METHOD, Claude Code's built-ins
2. **Traditional project management tools** — Trello, Pivotal Tracker, Linear, Jira, Asana
3. **Idea-to-shipped builders** — v0, Bolt.new, Replit Agent, Lovable

Maud's hypothesis is that the gap between (1) and (2) is real: AI coding tools either skip PM thinking entirely (vibe coding) or impose rigid templates (Spec Kit, Kiro). The best PM tools have no AI awareness. Maud wants to do real PM with an AI-aware build loop, governed by a single explicit principle: **the user verifies every phase boundary**.

What's already done in this space (from research):

- Plan-before-act split: **done many times** (Aider /architect, Cursor Plan, Cline, Roo Code, Spec Kit, Kiro)
- Spec → plan → tasks pipeline: **done** (Spec Kit, Kiro)
- Repo-aware planning: **done** (Cline, Cursor)
- Persistent file-based state: **done** (Spec Kit's `.specify/`, GSD's `.planning/`, Cursor's plan markdown)
- Approval at phase boundaries: **partially done** (Spec Kit has validation gates; Kiro generates everything in one pass; Devin treats checkpoints as advisory not blocking)
- Trello-style columns for AI-driven work: **not done** — closest is Kiro's task list with concurrent execution, but it's not Backlog/InProgress/Done with manual ordering
- Project-type-aware structure (different design format / iteration shape per project type): **not done** in any tool examined — all impose a single template

What's genuinely new in Maud is the **combination**, not any single feature. Be honest about this in the differentiator section.

---

## Feature Landscape

### Table Stakes (Users Expect These)

If any of these are missing, Maud will feel half-built compared to GSD, Spec Kit, or Cline. Users coming from those tools will silently downgrade their expectations.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Single stateful entry command | GSD has `/gsd:resume`, Spec Kit/Kiro auto-detect state, Cursor Plan persists. A plugin that doesn't pick up where you left off feels broken. | LOW | Already required by PROJECT.md ("re-running `/maud` reads `.maud/`"). Implementation = read `.maud/` and dispatch on phase. |
| Local file-based state in project directory | Spec Kit (`.specify/`), GSD (`.planning/`), Cursor plans (markdown), all do this. Files = inspectable, diffable, gitignorable. Required for resumability. | LOW | Required by PROJECT.md (`.maud/`). Just discipline about *what* lives in the dir. |
| Initial idea → clarifying questions → MVP definition | Every tool surveyed does this in some form. Spec Kit `/speckit.specify` + `/speckit.clarify`, Cursor's "asks clarifying questions to refine your plan", BMAD Analyst agent, Kiro requirements phase. Users expect to be interviewed, not just transcribed. | MEDIUM | Required by PROJECT.md. The questioning quality is what separates good from rote — see GSD's `references/questioning.md` (collaborative, not interrogative). |
| Research as a distinct phase (not just web searches sprinkled in) | Spec Kit, Kiro, BMAD, GSD all separate research from implementation. Tools that don't do this (Bolt, Replit Agent) are explicitly *not* doing PM — they're vibe coding. | MEDIUM | Maud's twist: user approves *what to research* before research runs, then approves the *synthesis* before advancing. Two gates inside one phase. |
| Architecture / tech-stack decision as an explicit artifact | Spec Kit `/speckit.plan` produces this. Kiro design phase. BMAD architecture phase. GSD `STACK.md` and `ARCHITECTURE.md` in research output. Without this, agents pick libraries randomly later. | MEDIUM | PROJECT.md specifies `.maud/architecture.md`. Format should be lean — "as simple as possible, no premature optimization." |
| Story / task / unit-of-work breakdown | Spec Kit `/speckit.tasks`, Kiro task list, BMAD epics+stories, GSD plans-within-phases, Pivotal/Linear/Trello stories. Universal. The unit of work *must* be smaller than "the whole project." | MEDIUM | PROJECT.md specifies `.maud/stories/` (one file per story, plus index). Flat list (no nesting), individually shippable, references design. |
| Iterative implementation loop with tests | Aider auto-commits, Devin iterates until tests pass, Cursor agents "iterate until tests pass," Cline executes step-by-step with approval. Test-aware execution is universal. | MEDIUM | PROJECT.md: "write tests where they make sense → implement → run tests → commit." Project-type determines TDD-first vs alongside. |
| Git integration (branches, atomic commits, sensible messages) | Aider's signature feature. GSD has commit-per-task and branching strategies. Cline auto-commits. Users assume their AI tool will not leave their git history a mess. | MEDIUM | GSD already has this pattern; Maud should adopt it. Branch-per-story or commit-per-checklist-item. |
| Resumability across sessions | GSD's `STATE.md` + `.continue-here` files. Spec Kit's filesystem state. Cursor Plan persists across sessions. Devin's async model is built on this. Without resumability, every context-window break wipes progress. | MEDIUM | PROJECT.md specifies `.maud/log.md` and "brings the user up to speed." Implementation = read state, summarize position, offer next action. |
| Audit trail of what was decided / why | GSD's per-plan SUMMARY.md, Spec Kit's persistent specs, BMAD's documentation-as-artifact. Users want to know "why does the codebase look like this?" months later. | LOW-MEDIUM | PROJECT.md specifies `.maud/log.md`. Just append-only is fine for v1; structured queries are over-engineering. |
| Subagent spawning for long operations (clean main context) | Claude Code's subagent pattern, GSD's executor/verifier/researcher agents, OpenHands' programmable agents. The reason: research/verification eats context, leaving none for the conversation. | MEDIUM | PROJECT.md specifies this. Direct port of GSD's subagent pattern likely works. |
| Verification before "done" (tests pass, code reviewed, etc.) | Devin runs tests, Cursor agents iterate to green, GSD has gsd-verifier, Cline shows diffs before applying, Aider auto-commits. The expectation is: **AI does not declare victory unilaterally**. | MEDIUM | PROJECT.md specifies automated agentic verification (review, coverage, QA, security) + final manual user gate. |
| Trello-like state model (Backlog / In Progress / Done) | Trello, Linear, Pivotal, Jira all do this in some form. Familiar mental model for any developer. | LOW | PROJECT.md specifies these three exact states. v1 is just file directories or a status field per story. |
| Add / reorder / remove stories at any time ("audibles") | Linear/Trello/Pivotal all let you reprioritize freely. Pivotal's "icebox" is exactly this. Hardcoded plans (Spec Kit, Kiro) feel rigid by comparison. | LOW-MEDIUM | PROJECT.md says "audibles welcome." Implementation = let user edit priority list / story files between iterations. |
| Story success criteria / acceptance criteria | Pivotal accepted/rejected states, Linear/Jira acceptance criteria, GSD success criteria, Kiro EARS-formatted acceptance criteria. Universally expected per story. | LOW | PROJECT.md specifies "success criteria" as part of each story file. |
| Markdown-everywhere (human-readable, diffable, copyable) | GSD, Spec Kit, Kiro, Cursor plans — all markdown. Anything else (JSON, custom format) feels alien in a developer-tool context. | LOW | Implicit in PROJECT.md; just discipline about format. |

### Differentiators (Competitive Advantage)

These are features Maud could *win* on. Each one is held to: "is this actually new, or did someone already do it?"

| Feature | Value Proposition | Complexity | Honest assessment of novelty |
|---------|-------------------|------------|------------------------------|
| **Mandatory user approval at every phase boundary** | Antidote to GSD's "wild decisions you never heard about" and Devin's "checkpoint, not gate" model. The user is in control of every transition; nothing material happens silently. This is the *core* differentiator. | MEDIUM | **Genuinely differentiated.** Spec Kit has approval gates *between* phases but is vague about granularity. Devin's checkpoints are explicitly non-blocking. Cline approves *each step* (too granular — verification fatigue). Maud's pitch is **gate at every phase boundary, frictionless within phases**. The novelty is the discipline of *where* the gates are, not that gates exist. |
| **Project-type-aware structure** (research dimensions, design format, iteration loop, verification criteria all dynamic) | Web app vs iOS vs game vs CLI have radically different needs. Spec Kit/Kiro/BMAD use the same template for all. Maud detects project type and adapts: design = screens for web, sprite-sheets for game, structured spec for CLI; iteration = browser-verify for web, simulator-verify for iOS, playtest for game. | HIGH | **Genuinely differentiated, but high implementation risk.** I could not find any tool that does this in research. Closest is BMAD's brown/greenfield split (binary). Maud needs a robust "sense the project type" step plus per-type templates/heuristics. The cost of getting this wrong is templates that don't fit, which destroys the value. Recommend: prototype 3-4 project types in v1 (web app, CLI, iOS, game), explicit "other" path that asks the user. |
| **Frontloaded thinking** (research / design / architecture done holistically once, not per-phase loops) | GSD's failure mode: per-phase research+plan creates fragmented thinking and "wild decisions." Real PM does the heavy thinking up front. | MEDIUM | **Differentiated against GSD specifically; less novel against Spec Kit/Kiro/BMAD which also frontload.** The win is over GSD; the question is whether new users care. The deeper value is *combined with* the gating discipline: user sees the whole plan before any code is written. |
| **Design as its own artifact, distinct from architecture** | Spec Kit collapses these into `/speckit.plan`. Kiro has design phase but it's tech design. BMAD has UX expert agent but it's persona-based. Maud separates: **design = what the thing is** (screens, sprite sheets, command spec), **architecture = how it's built** (stack, deployment, system shape). | MEDIUM | **Moderately differentiated.** Kiro and BMAD have design concepts but conflate UX with tech design. The cleanest analog is real-world PM where design (Figma) and architecture (system diagrams) are distinct disciplines. The value: forces the AI to think about the *user-facing thing* before the *implementation*. |
| **Stories as flat priority-ordered list (Trello model) for AI-driven work** | All AI tools surveyed use either ordered task lists (Spec Kit, Kiro) or branching trees (BMAD epics/stories). None use Trello's mental model. Trello's strength: trivial to reorder, easy to add/remove, no scoping ceremony. | LOW-MEDIUM | **Differentiated.** This is unique in AI orchestration. The risk: developers used to Spec Kit's strict ordering may dislike the implied "plan can change underneath you" model. Mitigation: every reorder is a phase transition and gets the user-approval gate. |
| **JIT per-story mini-design loop (escape hatch from frontloaded plan)** | Frontloading is a hypothesis, not a guarantee. Sometimes a story turns out to need deeper investigation. Maud allows a story to enter a mini-design loop without making it ceremony for every story. | LOW-MEDIUM | **Mildly differentiated.** GSD has "discovery levels" (Level 1/2/3) which is similar in spirit. Aider doesn't enforce one. The novelty is making it *opt-in per story* rather than mandatory or absent. The honest value: it's just good design — the structure of "frontload + escape hatch" is what every senior engineer does anyway. |
| **Automated agentic verification with manual user gate as final authority** | Devin runs tests; Cursor iterates to green; GSD has verifier. None of them combine: parallel automated passes (review + coverage + QA + security) + then user signoff. Spec Kit's `/speckit.analyze` is consistency check, not security/QA. | MEDIUM | **Moderately differentiated.** The pieces exist individually; combining them is the win. Per-project-type criteria (e.g., "what does QA mean for a CLI?") amplifies this. |
| **The combination itself: a coherent end-to-end PM-flavored AI dev loop with real gates** | Each individual feature exists somewhere. Maud's claim is: the *integrated whole*, with the gating discipline, is the product. | (system-level) | **This is the actual differentiator.** Be careful not to oversell individual features. Maud's value is the orchestration of these into a working flow, with the user in the driver's seat at every phase boundary. The closest comparable orchestration is Spec Kit + Kiro merged together; neither does what Maud is proposing as a single tool. |

### Anti-Features (Commonly Requested, Often Problematic)

Features that would feel natural to add but should be deliberately deferred or rejected. Confirmed against PROJECT.md's Out of Scope section.

| Anti-Feature | Why Requested | Why Problematic | Alternative |
|--------------|---------------|-----------------|-------------|
| Web GUI / Trello-like visual interface | "But Trello has a UI!" Users expect drag-and-drop boards because Maud's mental model invokes Trello. | Doubles surface area. Trello-on-disk markdown serves the same purpose — humans can read directories, edit files, see diffs in git. GUI is a productization step, not a v1 problem. | **Confirmed in PROJECT.md Out of Scope.** v1 = `.maud/stories/` directory + a status header in each story file. Future productization can wrap a UI around this. |
| Multi-user / collaboration features | "Eventually I'll work with someone else…" People say this about every solo tool. | Multi-user changes everything: conflict resolution, state synchronization, presence, comments. Different product. Solving it pre-PMF burns 10× the effort for 0× the validation. | **Confirmed in PROJECT.md Out of Scope.** Stay solo. If demand surfaces post-launch, treat as new product. |
| Integrations with external PM tools (Trello / Jira / Linear sync) | "Couldn't I just sync my Maud stories to Linear?" Sounds like a small feature. | Each integration is a maintenance burden + contract negotiation with a third-party API surface. Stories' meaning differs across tools. The user's mental model is the source of truth, not an external system. | **Confirmed in PROJECT.md Out of Scope.** Add later only if a specific user explicitly demands it (and pay them for the case study). |
| Brownfield support in v1 | "I want to use this on my existing project!" Most projects are existing. | Brownfield means reading + understanding existing code, inferring conventions, respecting patterns. Whole different research domain (codebase mapping). GSD has a `map-codebase` workflow; BMAD treats brownfield as separate flow. Forces v1 scope to balloon. | **Confirmed in PROJECT.md Out of Scope.** Greenfield-only is the right v1 boundary. Brownfield is a clean v2 scope. |
| Marketplace publication / global install | "Make it discoverable on the Claude Code marketplace!" | Distribution is a separate problem (versioning, doc site, release cadence). Local-only means iterating without breaking other users. | **Confirmed in PROJECT.md Out of Scope.** Ship locally, dogfood, productize later. |
| Milestones / epics / nested stories | "What if a story is too big?" — natural agile thinking. | v1 with flat stories validates the simpler model. If a story is too big, split it into two flat stories — this is exactly what Trello users do. Adding nesting/epics adds 3× the data model complexity. | **Confirmed in PROJECT.md Out of Scope.** Flat list. A story with a sub-task checklist is sufficient. Revisit only if real users hit the wall. |
| Multiple-choice rubber-stamp prompts ("approve A / B / C?") | Cline-style step-by-step approval, GSD's AskUserQuestion micro-prompts. Feels safe but creates verification fatigue. | Cline users famously get exhausted approving every line edit. GSD's narrow prompts feel like rubber-stamping ("yes, obviously"). The user stops reading carefully. | **Maud's gates are at *phase boundaries* only, with the *full proposed artifact* visible.** "Here's the synthesized research / design / story breakdown — approve, edit, or rerun." Per PROJECT.md philosophy. |
| Time estimates / velocity tracking / story points | Pivotal Tracker has all of this. Linear too. Looks "professional." | Solo developer + AI = velocity is meaningless. Estimates were already a lie when humans gave them; with AI they're nonsensical. PROJECT.md explicitly says "no time estimates." | Don't add. Stories are units of *intent*, not time. (GSD's STATE.md has performance metrics — Maud should *not* copy this.) |
| Comprehensive pre-flight checklists ("constitution," "principles") | Spec Kit's `/speckit.constitution`. Looks rigorous. | Adds ceremony before the user gets to express what they want to build. Frontloads more, not better. | Constraints / decisions naturally accumulate in PROJECT.md (validated requirements, key decisions). No separate constitution artifact needed. |
| Multiple agent personas (PM / Architect / Dev / QA chats) | BMAD-METHOD's signature feature. Looks comprehensive. | Persona theatre. The user doesn't actually want to "talk to the architect agent" — they want the design done well. Skill-based subagents (research, executor, verifier) are functional; persona-flavored ones are dressing. | Maud has functional agents (research, executor, verifier) inherited from GSD. Don't add personas. |
| Built-in project templates ("greenfield SaaS," "iOS app starter") | Lovable / v0 / Bolt offer these. Feels welcoming. | Templates fight against project-type-aware *flexibility*. The whole point is Maud asks questions and figures out the project type, not picks from a dropdown. | Per PROJECT.md philosophy: flexibility over rigidity. No starter templates. The clarifying questions phase + research determine structure. |
| "Always-on" autonomous mode (let it run for hours, come back to a PR) | Devin's signature feature. Trendy. | Directly contradicts PROJECT.md's core value: explicit verification at every phase boundary. Devin and Maud are opposite philosophies. | Don't add. Maud is defined *against* this model. Documenting this contrast is part of the pitch. |
| Real-time multi-agent execution / agent teams | Claude Code's "Agent Teams" feature, OpenHands' multi-session. | More complex; user can't track what's happening; defeats the gating model. | Single sequential flow with subagents spawned for specific tasks. Parallelism is for verification (review + coverage + security can run in parallel) not for *invention*. |

---

## Feature Dependencies

```
[Single stateful /maud command]
    └── requires ──> [.maud/ filesystem state schema]
                         └── requires ──> [PROJECT.md / state files / log.md formats]

[Initialization / clarifying questions]
    └── produces ──> [PROJECT.md (idea + MVP definition)]
                         └── feeds ──> [Research phase]

[Research phase (proposal → approval → execute → synthesis → approval)]
    └── produces ──> [.maud/research/*.md]
                         └── feeds ──> [Architecture] AND [Design]
                         └── informs ──> [Iteration loop shape, verification criteria]

[Architecture phase] ⟷ [Design phase]
    (project-type-dependent which comes first; both feed Stories)
    └── produces ──> [architecture.md, .maud/design/*]
                         └── feeds ──> [Story extraction]

[Story extraction]
    └── produces ──> [.maud/stories/*.md + index]
                         └── feeds ──> [Prioritization]

[Prioritization (Backlog ordering)]
    └── enables ──> [Iteration loop]

[Iteration loop (per-story)]
    └── may invoke ──> [JIT mini-design loop]
    └── produces ──> [code commits, tests]
                         └── triggers ──> [Verification (automated + manual)]

[Verification]
    └── on success → [Story → Done, deploy/merge]
    └── on failure → [Loop back to iteration]

[User-approval gate]
    └── inserted at ──> [End of every phase above]

[Subagent spawning + log.md] ──> orthogonal, supports all phases

[Audibles (add/reorder/remove stories)]
    └── allowed at ──> [Any user-facing moment]
    └── feeds back into ──> [Prioritization]
```

### Dependency Notes

- **`.maud/` schema is foundational.** Every feature reads/writes it. Get this right early — schema migrations are painful in greenfield-but-already-shipped state.
- **Subagent + log.md are orthogonal.** They support every phase but don't depend on phase content. Build them as infrastructure, then use them.
- **Research → both Architecture AND Design.** Both downstream artifacts depend on research. The order between Arch and Design is project-type-dependent (per PROJECT.md), so the dependency graph branches there.
- **Story extraction depends on BOTH approved design AND approved architecture.** Cannot start until both are locked. This is a hard sync point.
- **Iteration loop is the per-story workhorse.** Everything before it is ceremony enabling it; everything after it is verification ratifying it.
- **JIT mini-design loop conflicts with frontloaded thinking.** Resolve: JIT is the explicit escape hatch when frontloaded design is insufficient for a specific story. Document this tension; don't pretend it doesn't exist.
- **User-approval gate is a cross-cutting feature.** Treat it as a system property, not a feature in any single phase. The implementation is shared across all phases (same approval pattern, same artifact-presentation discipline).
- **Audibles depend on the priority list being mutable.** Implementation: stories are files; priority is a list (or per-story field); both are user-editable at any time. This is mostly a *design discipline*, not heavy code.

---

## MVP Definition

### Launch With (v1)

The minimum viable bootstrap. If any of these is missing, Maud cannot do its claimed job for the user's first project.

- [ ] **Single `/maud` command, stateful, reads `.maud/` and resumes** — entire UX hinges on this
- [ ] **Initialization: idea → clarifying questions → MVP-defined PROJECT.md** — the front door
- [ ] **Research phase: propose dimensions → user approves → run research (subagents) → synthesize → user approves** — Maud's frontloaded-thinking promise hinges on this
- [ ] **Architecture artifact (`.maud/architecture.md`) — user approves before advancing**
- [ ] **Design artifact (`.maud/design/`) in project-type-appropriate format — user approves before advancing**
- [ ] **Story extraction (`.maud/stories/*.md` + index) from approved design + architecture — user approves before advancing**
- [ ] **Prioritization: ordered Backlog — user approves before iteration begins**
- [ ] **Iteration loop: per-story build (tests where appropriate, implement, commit) with project-type-aware shape**
- [ ] **Audibles: user can add/reorder/remove stories during user-facing moments**
- [ ] **Verification: automated agentic passes (code review, test coverage, basic QA) + final user manual gate**
- [ ] **Trello-style states (`Backlog` / `In Progress` / `Done`) with user-approval to transition `In Progress` → `Done`**
- [ ] **Subagent spawning for long operations (research, verification, iteration)**
- [ ] **`.maud/log.md` audit trail (append-only)**
- [ ] **Phase transitions: explicit user verification at every boundary** (this is the core value, not a feature)
- [ ] **Git integration: branch per story or sensible commits per checklist item, sensible commit messages**
- [ ] **Resumability: re-running `/maud` brings user up to speed (current phase, story counts)**
- [ ] **Project-type detection / awareness — must support at least 3 project types in v1 (recommend: web app, CLI, iOS), with explicit "other / explain" fallback**
- [ ] **Deployment step (project-dependent): merge-to-main, deploy-to-staging, or deploy-to-prod, determined per project**

These are not optional — every requirement above is also in PROJECT.md's Active Requirements section. This MVP definition restates them as a launch checklist.

### Add After Validation (v1.x)

Add once core flow has shipped at least one project end-to-end and feedback validates the gating model.

- [ ] **Security scan as separate verification pass** — listed in PROJECT.md but may be deferrable past v1.0 if QA / coverage / review covers enough
- [ ] **More project types** — beyond the 3-4 v1 supports; e.g., game, library, browser extension, ML model
- [ ] **Smarter resumability** — currently "tell me where I am." Could become "you have a half-finished story; want to continue or audible?"
- [ ] **Performance metrics** — only if user feedback demands it; otherwise omit (anti-feature watch)
- [ ] **Per-story commit message templates / branch naming conventions** — currently sensible defaults; may want user customization
- [ ] **Better synthesis presentation** — research synthesis currently a single document; might benefit from comparison tables, decision matrices, etc.
- [ ] **Story dependencies / blockers** (explicit) — Pivotal-style. Currently parallelization "surfaces blockers" via prioritization step; might warrant first-class field if real projects demand it
- [ ] **Reflection / retro after Done** — "what went well in this story, anything to update for next one?" lightweight, optional

### Future Consideration (v2+)

Defer until v1 is dogfooded and product-market fit is real.

- [ ] **Brownfield support** — explicit v2 scope per PROJECT.md
- [ ] **Web GUI / Trello-style visual interface** — productization step
- [ ] **Marketplace publication** — distribution, not product
- [ ] **Multi-user / collaboration** — different product
- [ ] **External PM tool integrations (Linear / Jira / Trello sync)** — far future
- [ ] **Milestones / epics / nested stories** — only if real projects break under flat list
- [ ] **Custom verification criteria templates** — let advanced users override per-project-type defaults
- [ ] **Plugin / extension hooks for community-defined project types** — Spec Kit's "extensions" model is good prior art

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Single stateful `/maud` command | HIGH | LOW | **P1** |
| `.maud/` state schema | HIGH | MEDIUM | **P1** |
| Initialization + clarifying Qs | HIGH | MEDIUM | **P1** |
| Research phase (with both gates) | HIGH | MEDIUM | **P1** |
| Architecture artifact + gate | HIGH | LOW-MEDIUM | **P1** |
| Design artifact + gate (project-type-aware) | HIGH | HIGH | **P1** |
| Story extraction + gate | HIGH | MEDIUM | **P1** |
| Prioritization + gate | HIGH | LOW | **P1** |
| Iteration loop (project-type-aware shape) | HIGH | HIGH | **P1** |
| Verification (automated + manual gate) | HIGH | MEDIUM | **P1** |
| Trello-style states (Backlog/InProgress/Done) | HIGH | LOW | **P1** |
| Audibles | HIGH | LOW-MEDIUM | **P1** |
| Subagent spawning | HIGH | MEDIUM | **P1** |
| `.maud/log.md` audit trail | MEDIUM | LOW | **P1** |
| Resumability | HIGH | MEDIUM | **P1** |
| Git integration (branches, commits) | HIGH | MEDIUM | **P1** |
| Project-type-aware deployment step | MEDIUM | MEDIUM | **P1** |
| Per-story JIT mini-design loop | MEDIUM | MEDIUM | **P1** (escape hatch — needed for honesty) |
| Security scan verification pass | MEDIUM | MEDIUM | **P2** |
| Better synthesis presentation (tables, etc.) | LOW-MEDIUM | LOW-MEDIUM | **P2** |
| More project types (game, library, etc.) | MEDIUM | MEDIUM | **P2** |
| Story dependencies / blockers (first-class) | MEDIUM | LOW-MEDIUM | **P2-P3** |
| Smarter resumability (deeper context) | LOW-MEDIUM | MEDIUM | **P2-P3** |
| Brownfield support | HIGH (eventually) | HIGH | **P3** (v2) |
| Web GUI | MEDIUM (eventually) | HIGH | **P3** (v2) |
| Performance metrics | LOW | LOW | **Skip** (anti-feature watch) |
| Multi-user collaboration | MEDIUM (eventually) | VERY HIGH | **Skip** (out of scope) |

**Priority key:**
- **P1** — Required for v1 launch. Excluded from this list = Maud is incomplete.
- **P2** — Should add post-launch once core flow is validated.
- **P3** — Future versions; deferred deliberately.
- **Skip** — In Anti-Features above; do not add.

---

## Competitor Feature Analysis

For each major competitor, what they do, and Maud's intended approach.

| Feature | Spec Kit | Kiro | GSD (predecessor) | Cline / Plan Mode | Devin | Maud's approach |
|---------|----------|------|-------------------|-------------------|-------|-----------------|
| **Phase model** | Specify → Plan → Tasks → Implement (+ Constitution) | Requirements → Design → Implementation (one pass, no gates) | Discovery / Discuss / Plan / Execute / Verify (per-phase loops) | Plan Mode → Act Mode (binary) | Interactive Planning then autonomous run | **Init → Research → Arch+Design (order project-dependent) → Stories → Prioritize → Iterate (per-story) → Verify (per-story) → Done. Gates at every boundary.** |
| **Approval granularity** | Validation gates between phases (vague granularity) | None — one-pass generation | AskUserQuestion micro-prompts (often rubber-stamp) | Per-step approval (verification fatigue) | Plan is "checkpoint, not gate" — proceeds unless user intervenes | **Phase boundaries only. Full artifact presented; user approves, edits, or reruns. No micro-prompts inside phases.** |
| **Project-type awareness** | None — same template all projects | None — same template | None | None | None | **Detected during init / research; influences design format, iteration shape, verification criteria, deployment.** Genuine differentiator. |
| **Design as own artifact** | No (collapsed into Plan) | Partial (tech design only) | No | No | No | **Yes. `.maud/design/` separate from `.maud/architecture.md`. Format is project-type-dependent (screens / sprite-sheets / CLI spec / etc.).** |
| **Unit of work** | Tasks (ordered list) | Tasks (DAG, parallel) | Plans within Phases (waves) | Steps in plan | Whole-task delegation | **Stories (flat, individually shippable, Trello-state).** |
| **State location** | `.specify/` markdown | IDE-managed specs | `.planning/` markdown | Cline's repo index + chat | Cloud sandbox | `.maud/` markdown — local, project-scoped, gitignorable |
| **Audit trail** | Specs persist as artifacts | Specs persist | SUMMARY.md per plan | None explicit | PR description | `.maud/log.md` (append-only) |
| **Verification** | `/speckit.analyze` (consistency only) | Tests if user runs them | gsd-verifier agent | Tests run via Act mode | Iterates until tests pass | **Automated parallel passes (review + coverage + QA + security) + manual user gate.** Per-project-type criteria. |
| **Resumability** | Filesystem state | IDE state | STATE.md + .continue-here | Repo-aware on restart | Built for async | `/maud` re-entry reads `.maud/` and brings user up to speed |
| **Audibles (mid-flow scope changes)** | Not native | Limited | Limited | Plan can be edited | Limited | **First-class. User can add / reorder / remove stories at any user-facing moment.** |
| **Greenfield-only?** | No (both) | No (both) | Both (loosely) | Both | Both | **Yes (v1). Brownfield is v2.** |
| **Multi-user?** | No | No (IDE) | No | No | Yes (cloud) | **No (single-user, by design).** |
| **GUI?** | No (CLI / IDE) | Yes (IDE) | No | Yes (VSCode) | Yes (web) | **No (Claude Code session is the UI).** |
| **Distribution model** | NPM + CLI + agent integrations | IDE download | Local install (slash commands + agents) | VSCode extension | SaaS | **Local Claude Code plugin (v1). Marketplace later.** |

### Key competitive insights

1. **Spec Kit is Maud's closest direct competitor.** Phase model is similar (specify → plan → tasks → implement vs. init → research → arch/design → stories → iterate). Differences: Spec Kit collapses design into plan, has no Trello-state, no project-type awareness, no audibles. Maud's pitch: "Spec Kit, but with real PM thinking and Trello-style flexibility."

2. **Kiro is the closest "spec-driven IDE."** Generates everything in one pass *without* approval gates — opposite philosophy from Maud. Use this in marketing: "Kiro generates the whole spec; Maud builds it with you, gate by gate."

3. **GSD's failure modes are Maud's design opportunities.** Per-phase loops → frontloaded. Multiple-choice prompts → full-artifact gates. Rigid phase counts → flexible stories. No design phase → design as first-class. Document this lineage; don't hide it.

4. **Devin / OpenHands / Cursor agents are the "autonomous" pole** Maud is defined *against*. They run for hours; Maud insists on user gates. Be explicit: Maud is for developers who want PM rigor *with* AI, not autonomy from AI.

5. **v0 / Bolt / Replit Agent / Lovable serve a different audience.** They're idea-to-shipped *fast*; they explicitly skip PM. They do not compete with Maud. (If a user wanted Bolt, they wouldn't be here.) Mention this only to clarify positioning, not as a feature comparison.

6. **No tool surveyed has Trello-style flat-story-list + Backlog/InProgress/Done columns.** This is real white space. The risk is users expect Linear/Jira sophistication and find Maud sparse; mitigation is messaging — "Trello mental model intentionally; if you want Linear, use Linear."

7. **Pivotal Tracker has the most relevant traditional-PM lineage for solo dev + AI.** Story types (feature/bug/chore/release), states with rejection, icebox model, blockers. Maud should *not* import Pivotal's complexity (story points, velocity), but its workflow shape (icebox = backlog, accepted = done) is a useful reference.

---

## Sources

**Primary source — read directly:**
- `/Users/ryan/.claude/get-shit-done/` — full GSD reference implementation (workflows, templates, references). Confidence: HIGH.

**Maud project docs:**
- `/Users/ryan/Documents/programming/maud/.planning/PROJECT.md` — current project definition. Used to confirm Out of Scope alignment.

**AI orchestration tools (web research):**
- [GitHub Spec Kit (github.com/github/spec-kit)](https://github.com/github/spec-kit) — closest competitor. HIGH confidence on its structure.
- [Spec Kit documentation](https://github.github.com/spec-kit/) — phase model, command set.
- [Kiro IDE specs documentation](https://kiro.dev/docs/specs/) — requirements/design/tasks model. MEDIUM-HIGH confidence.
- [Cursor Plan Mode](https://cursor.com/blog/plan-mode) and [Plan Mode docs](https://cursor.com/docs/agent/plan-mode) — plan-then-act IDE pattern. HIGH confidence.
- [Cline Plan & Act docs](https://docs.cline.bot/features/plan-and-act) — plan/act split, repo indexing. MEDIUM-HIGH confidence.
- [Aider chat modes](https://aider.chat/docs/usage/modes.html) — architect/editor mode split. HIGH confidence.
- [Aider 2026 guide (DeployHQ)](https://www.deployhq.com/guides/aider) — current usage patterns.
- [Claude Code extend with skills (code.claude.com/docs/en/skills)](https://code.claude.com/docs/en/skills) — skills/commands/subagents/plugins distinction.
- [Devin coding agents 101](https://devin.ai/agents101) — checkpoint-not-gate philosophy. MEDIUM confidence.
- [OpenHands](https://www.openhands.dev/) — open-source autonomous platform. MEDIUM confidence.
- [BMAD-METHOD on GitHub](https://github.com/bmad-code-org/BMAD-METHOD) — agentic agile framework. MEDIUM confidence.
- [Coding agents 2026 comparison (Codersera)](https://codersera.com/blog/ai-coding-agents-complete-guide-2026/) — landscape overview. MEDIUM confidence.

**Traditional PM tools (web research):**
- [Pivotal Tracker terminology](https://www.pivotaltracker.com/help/articles/terminology/) and [story states](https://www.pivotaltracker.com/help/articles/story_states/) — story types, workflow. HIGH confidence.
- [Bugs and Chores estimation (Pivotal blog)](https://www.pivotaltracker.com/blog/bugs-chores-estimate-estimate) — confirmed: don't estimate bugs/chores.
- [Linear vs Jira vs Trello 2026 (prommer.net)](https://prommer.net/en/tech/guides/linear-vs-jira-vs-trello/) — feature comparison. MEDIUM confidence.
- [Trello vs Jira (Toptal)](https://www.toptal.com/agile/trello-vs-jira-comparison) — confirmed: Trello has no epics/stories/sprints, just cards.

**Idea-to-shipped tools (web research):**
- [v0 alternatives 2026 (Builder.io)](https://www.builder.io/blog/v0-alternatives) — landscape.
- [AI app builders compared 2026 (NovaKit)](https://www.novakit.ai/blog/ai-builders-comparison-bolt-lovable-v0-novakit) — Bolt/Lovable/v0/Replit feature shape. MEDIUM confidence.
- [2026 AI Coding Platform Wars (Aftab on Medium)](https://medium.com/@aftab001x/the-2026-ai-coding-platform-wars-replit-vs-windsurf-vs-bolt-new-f908b9f76325) — flow comparison.

**Confidence note:** GSD findings are HIGH (read directly). Spec Kit, Kiro, Cursor Plan, Cline, Aider, Pivotal Tracker findings are HIGH-MEDIUM (multiple corroborating sources). Devin, OpenHands, BMAD-METHOD, idea-to-shipped tools are MEDIUM (single-source synthesis from web search; not deeply verified). The competitive landscape moves fast (especially in 2026); treat specifics as snapshot-of-research-day, not durable fact. The *categorical* findings (no tool combines Trello-state + project-type-awareness + every-phase-gates + design-as-distinct-artifact) are robust across all sources.

---

*Feature research for: AI-assisted development orchestration tool (Claude Code plugin)*
*Researched: 2026-05-08*
