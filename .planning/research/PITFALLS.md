# Pitfalls Research

**Domain:** Claude Code plugin orchestrating AI-assisted software development project management (greenfield, solo-dev, dogfooded)
**Researched:** 2026-05-08
**Confidence:** MEDIUM-HIGH (HIGH for user-confirmed GSD failure modes; MEDIUM for ecosystem patterns from web sources; HIGH for solo-dev / dogfooding tradeoffs which are well-established)

This document is the single most important research artifact for Maud, because Maud is *defined* by the failure modes it refuses to repeat. Most other tools in this space have shipped some version of these mistakes; the user's PROJECT.md explicitly names four of them as anti-patterns to avoid. The job here is to confirm those, expand the surface to cover the rest of the orchestrator/PM/plugin/solo-dev landscape, and bind every pitfall to a phase that prevents it.

A few framing notes:

- "User-called-out" pitfalls (the four GSD failure modes) are flagged HIGH confidence — these are product requirements, not hypotheses.
- "Industry-confirmed" pitfalls are HIGH-MEDIUM where a 2026 source corroborates a known 2024-2025 pattern.
- Two pitfalls (flexibility-becomes-vaporware and verification-becomes-exhausting) are calibration questions the user explicitly raised; both are treated as Critical because they are the most likely ways Maud's stated principles eat themselves.

---

## Critical Pitfalls

### Pitfall 1: Multiple-Choice Rubber-Stamp Prompts (User-Called-Out GSD Failure Mode)

**What goes wrong:**
Maud asks the user to "approve" by presenting a pre-formed multiple-choice question with options A/B/C, where A is what Maud already wanted to do. The user reflexively picks A (or "looks good") and Maud advances. The user has performed approval theater, not approval. Consequential decisions are made without real user input because the prompt format itself precludes meaningful intervention.

**Why it happens:**
1. Multiple-choice prompts are easier to engineer than open-ended elicitation — they parse cleanly, they're cheap in tokens, and they always produce a "decision."
2. The agent's incentive is to keep moving; presenting a closed set lets it advance after a single token of user input.
3. Approval-fatigue dynamics: even a thoughtful user, after the 5th-6th prompt of a session, starts pattern-matching and picking the obvious option without re-engaging. Industry data on Claude Code shows users approve ~93% of permission prompts — the same dynamic applies to design approvals (Sources: Developers Digest, Molten.bot).

**Consequences:**
The product silently makes the same wild decisions GSD makes, with a rubber stamp on top. Worse than no approval — it produces *false* confidence that the user is in control.

**How to avoid:**
- **Open-ended elicitation by default** for consequential decisions (architecture, scope, story breakdown). Maud presents *its current thinking* and asks "what would you change, add, remove, or push back on?" — not "approve A/B/C?"
- **Closed-form only for genuinely binary mechanical questions** ("commit now? y/n", "rerun tests? y/n"). Closed-form prompts are an exception, not the default.
- **Reject single-token approvals on consequential decisions.** If the user types "lgtm" on the architecture phase, Maud says: "before I proceed — what about [specific concern from architecture doc] do you want me to handle differently?" Force at least one round of substantive engagement.
- **Minimum-content threshold** for approval messages on phase transitions (research, design, architecture, stories, prioritization). If the user's response is shorter than N characters or matches a list of approval-stems ("looks good", "lgtm", "ok", "yes"), Maud asks one targeted follow-up before accepting.

**Warning signs:**
- User's session transcripts show >70% one-line approvals.
- User ends a session having approved a phase they can't summarize back accurately.
- Maud offers options A/B/C and the user picks the first option in a majority of phases.

**Phase to address:** **Phase: Phase Transition / Approval Mechanic** (a foundational phase that defines how *all* phase transitions work, before any individual phase is built). Every later phase reuses this primitive.

---

### Pitfall 2: Per-Phase Research+Plan Cycles That Should Be Frontloaded (User-Called-Out GSD Failure Mode)

**What goes wrong:**
Each phase ships its own mini research-then-plan loop, so the project never gets a coherent upfront pass. The architecture phase researches its own concerns without seeing the design's concerns. The stories phase plans without understanding the architecture's tradeoffs. Decisions made early get re-litigated late. Research in phase N invalidates assumptions in phase N-2. The project spirals.

**Why it happens:**
1. Phase-local mini-loops are easier to template and ship — each phase looks like a self-contained agent runbook.
2. It feels safer ("we'll figure out research when we need it") but actually defers the hardest thinking past the point of cheap reversibility.
3. The user-stated PROJECT.md insight: "frontloaded thinking is the bulk of real PM value" — i.e. real PMs do not research-as-you-go; they do a heavy upfront pass *before* committing to a story breakdown.

**Consequences:**
- Stories built on stale research; architecture rewrites; designs that conflict with constraints discovered during build.
- Each phase feels productive ("we did research! we made a plan!") but the project as a whole drifts.
- The dogfooding consequence is severe: Maud building Maud cannot afford this drift, because every drift cascades into the next project Maud builds.

**How to avoid:**
- **Research and design happen in dedicated frontloaded phases**, before stories are extracted. PROJECT.md already establishes Research → Design → Architecture → Stories as the canonical order.
- **Research dimensions are determined at one moment** (Research phase entry), then *executed* in parallel by agents. There is no "research round 2" baked into the next phase.
- **Per-story JIT mini-loop is allowed but must be explicit and rare.** PROJECT.md already calls this out: "an escape hatch when a story warrants deeper work, without making it mandatory ceremony." This phrasing must be enforced — the JIT mini-loop should *not* run on every story by default.
- **No phase template starts with "research." After the Research phase concludes, later phases consume the artifacts in `.maud/research/`, they do not regenerate them.**
- **The Stories phase rejects "we'll figure that out during implementation"** for architectural questions. If a story would require an architectural decision, it's a signal the Architecture phase finished prematurely and needs revisiting.

**Warning signs:**
- A `research/` subdirectory or research-style notes appearing inside individual story files.
- Architecture decisions being introduced for the first time in a story-implementation iteration.
- The phrase "let's research that real quick" appearing during the build loop.
- Stories frequently expanding scope upon entering In Progress.

**Phase to address:** **Phase: Roadmap Structure & Phase Order** (defines the canonical phase sequence and explicitly prohibits research/design re-runs in later phases). Then enforced by the **Phase: Stories** template, which forbids new architectural research inside a story.

---

### Pitfall 3: Arbitrary Phase Counts and Rigid Templates (User-Called-Out GSD Failure Mode)

**What goes wrong:**
Maud imposes "every project has exactly 5 phases" or "every architecture artifact has these 9 sections" regardless of project type. A CLI tool gets a UI-design phase. A game project gets a database-schema section. A 3-screen web app gets the same phase count as a multi-service backend. The framework's structure leaks into the work and produces ceremony that doesn't fit reality.

**Why it happens:**
1. Templates are easier to ship and test than dynamic structure. "Always 5 phases" passes a unit test; "the right number of phases for this project" doesn't.
2. Documentation-driven design: the docs need a canonical example, the canonical example becomes the template, the template becomes the law.
3. GSD's specific failure (per the user): arbitrary phase counts and fixed criteria templates that don't account for project type.

**Consequences:**
- Solo developers do the ceremony, hate the ceremony, abandon Maud. (Confirmed by 2026 lightweight-PM tool research: solo devs explicitly want "no fancy metrics, no sprint tracking, no required ceremony.")
- The project type fights the template — work either gets crammed into the wrong shape or skipped entirely with hand-waving.

**How to avoid:**
- **Project-type detection is its own first-class step, not an afterthought.** Maud asks "what are you building?" early, classifies (web app / CLI / game / mobile / library / etc.), and the phase shape is determined by classification.
- **No fixed phase count.** Phase shape is a tree of *required* phases (Research, Design, Architecture, Stories, Iteration) and *project-type-dependent* phases (UI design for web apps; level design for games; spec format for CLIs; deployment topology for backends).
- **Template artifacts are starting points, not contracts.** A design doc for a CLI is structured differently from one for a game. Maud's design-phase agent generates a *project-appropriate* artifact format, then asks the user to verify it before populating.
- **Resist over-parameterization too.** If "project-type-aware" becomes "user must answer 47 classification questions," it's just a different ceremony. Classify with one open-ended question and fill in via clarifying conversation.
- **PROJECT.md is a v1 example of escaping a template** — it's prose with conventional sections, not a rigid form.

**Warning signs:**
- Two different project types end up with identical artifact structures.
- The Design phase produces a "design.md" with sections for things the project doesn't have (database schema for a static site).
- Users skip phases because "this doesn't apply to my project."
- Phase names in code are numeric (`phase_3`) rather than semantic (`research`, `design`).

**Phase to address:** **Phase: Project-Type Classification & Adaptive Structure** (an early phase that runs before Research; it determines which phases exist and what their artifacts look like).

---

### Pitfall 4: No Design Phase (User-Called-Out GSD Failure Mode)

**What goes wrong:**
The framework jumps from "what to build" to "how to build it" without explicitly designing. There's no artifact that says "the home page has these sections, this is how the user navigates between them" or "this CLI command takes these flags and outputs in this format." Stories get written from architectural intuition rather than designed user-facing behavior.

**Why it happens:**
1. AI coding assistants are trained on code, not on design artifacts. The natural flow is requirements → code; "design" feels like extra ceremony.
2. Design varies wildly by project type, so generic frameworks skip it as "too domain-specific."
3. Designers and developers historically work in different tools; PM tools that originated in dev culture often have no design phase at all.

**Consequences:**
- Stories describe behavior at the wrong level — too implementation-y or too vague.
- The user discovers UX problems during build, not during design. Build-time discovery is 10x more expensive than design-time discovery.
- For solo developers especially: design is the moment the *whole product* gets a sanity check. Skipping it means there's no holistic review before stories.

**How to avoid:**
- **Design is a required, named phase** (already in PROJECT.md). Cannot be skipped.
- **Design is project-type-aware** (already in PROJECT.md): screens for web apps, sprite sheets / level layouts for games, structured spec for CLIs, API surface for libraries, etc.
- **The Design phase begins with a meta-step**: "What does design mean for this project?" — Maud and the user agree on the artifact format *before* generating it. This prevents the mismatch where Maud produces wireframes for a CLI tool.
- **If the user already has design (sketches, Figma, prior spec), Maud uses it.** Don't regenerate from scratch.
- **Design artifacts are referenced explicitly from stories.** Each story cites the design doc(s) it implements. This binds stories to a coherent product, not to local intuition.

**Warning signs:**
- Stories that describe behavior the design artifact doesn't mention.
- The design artifact is empty or trivial ("a CLI tool that takes commands").
- The Design phase finishes in <2 user turns. (Real design is iterative; if it converged immediately, either the project is trivial or design was skipped.)
- Design artifacts and architecture artifacts contain duplicate information — sign that one was generated from the other rather than independently.

**Phase to address:** **Phase: Design Phase Definition** (establishes the design phase's mechanics, including the meta-step of agreeing on artifact format). Reinforced by **Phase: Stories** which requires every story to cite a design doc.

---

### Pitfall 5: Verification Becomes Exhausting (User-Raised Calibration Question)

**What goes wrong:**
"User verifies every phase" turns into "user verifies every micro-step." Maud asks for approval on every research dimension, every design tweak, every story re-order, every commit. The user, faced with the 30th approval prompt, starts auto-approving. The differentiator from GSD evaporates: the prompts are explicit, but they're rubber-stamped because there are too many of them. This is approval fatigue (industry-confirmed; Claude Code data shows ~93% auto-approval at scale).

**Why it happens:**
1. The principle "user verifies every phase transition" is correct — but "phase" is ambiguous. If Maud reads it as "every meaningful unit of work," every story becomes a phase, every story sub-task becomes a sub-phase, and verification compounds.
2. Engineering discipline cuts the wrong way: "always ask before doing" feels safer than "ask sometimes," so the agent over-asks.
3. There is no friction differentiation — every prompt looks the same, so the user can't tell the structural-architectural decisions from the cosmetic ones.

**Consequences:**
- Users abandon Maud as exhausting.
- For users who don't abandon, real consequential decisions get the same auto-pilot treatment as trivial ones.
- The differentiator collapses. Maud becomes GSD with extra steps.

**How to avoid:**
- **Verify only at *phase boundaries*, not within phases.** PROJECT.md already specifies this: "every phase outlined above ends with explicit user verification." There is no per-story-within-iteration verification *until the story is complete*. Within a story's iteration loop (test, implement, run tests, commit), Maud doesn't ask for approval — it just runs and reports.
- **Tier verification by reversibility and scope** (industry pattern: Risk-Based Friction):
  - **Heavy verification (open-ended elicitation):** Research synthesis, Design, Architecture, Story breakdown, Prioritization. These are 5-7 prompts in the entire project lifecycle.
  - **Light verification (single confirmation):** Story Done (the user does manual final verification per PROJECT.md), git commit messages on demand.
  - **No verification (just notify):** Test runs, intermediate agent steps, log entries, file writes within `.maud/`.
- **Visually differentiate consequential vs trivial prompts.** A heavy verification prompt looks structurally different — section dividers, multi-section presentation, explicit "this is a major decision" framing. A light prompt is one line.
- **Audibles are zero-friction.** The user can intervene at any time without Maud asking — Maud watches for user input continuously, doesn't gatekeep.
- **Calibrate by counting prompts in a typical session.** Target: <10 heavy prompts per project lifecycle. <30 light prompts. If a real session shows >50 prompts, the calibration is wrong.

**Warning signs:**
- Approval prompts >10 in a single phase.
- User auto-completing prompts ("y", "ok", "lgtm") at increasing rates over a session.
- Same approval prompt asked twice in different forms (Maud's accidentally re-asking).
- Story-level prompts and project-level prompts look identical in transcript.

**Phase to address:** **Phase: Phase Transition / Approval Mechanic** (defines the *taxonomy* of approvals: heavy vs light vs notify) and **Phase: Iteration Loop** (defines that within-iteration steps do NOT prompt).

---

### Pitfall 6: Flexibility Becomes Vaporware (User-Raised Calibration Question)

**What goes wrong:**
"Project-type-aware flexibility" is the right idea, but it can collapse into "every aspect of Maud is configurable, and nothing is concretely built." The Research phase is "however it should be for this project." The Design phase is "whatever design means here." The Iteration loop is "whatever fits." End result: Maud has no concrete behavior — it's an empty shell that delegates everything back to the user. Users open it expecting a tool and find a meta-framework. They abandon.

**Why it happens:**
1. Flexibility is intellectually attractive — "we'll handle every project type!" — but operationally vaporous. Building 1 concrete vertical is harder than promising 5 abstract ones.
2. The YAGNI failure mode: generalize before you have 2-3 concrete instances. Without concrete instances to generalize *from*, the abstraction is guessed at and brittle.
3. Maud is built greenfield with no users — there are no real project types yet beyond "Maud itself" — so the temptation to design for hypothetical project types is enormous.

**Consequences:**
- v1 ships with the architecture for flexibility but no actual behaviors.
- The bootstrap test (Maud builds the next project) fails because Maud doesn't know what to do for "the next project" without a concrete vertical.
- The product reads as vaporware to anyone trying it.

**How to avoid:**
- **v1 ships with one concrete project-type vertical end-to-end** before generalizing. Best candidate: **CLI tool / Claude Code plugin** — because that *is* what Maud itself is (dogfood-friendly), and the second project Maud builds is likely also in this space.
- **Flexibility is achieved by adding a second concrete vertical, not by abstracting the first.** When Maud is used to build a web app, that's the moment to extract the abstraction over CLI + web. Don't pre-abstract.
- **Project-type adaptation in v1 is allowed to be coarse.** Three classes is enough: (a) CLI/plugin/library; (b) web app (frontend or full-stack); (c) game/visual/simulation. v1 implements (a) fully, (b) partially, (c) recognized but not implemented.
- **No "framework for frameworks."** Maud should not have a configuration DSL for project types. Project types live as concrete agent prompts and concrete artifact templates, not as data-driven configurations.
- **Apply the "rule of three"** before generalizing any phase template: don't extract a shared abstraction until it's been written 3 times concretely.

**Warning signs:**
- Code paths that say "if project_type == 'X' do A, elif 'Y' do B, else raise NotImplementedError" — and only X is implemented.
- Documentation describing what Maud "can" do in the abstract, not what it does for a specific project.
- The bootstrap test (Maud builds the next project) gets blocked on an unimplemented project type.
- Users opening Maud and finding configuration questions instead of action.

**Phase to address:** **Phase: Pick the V1 Vertical** (an explicit decision phase: which project type does Maud actually build end-to-end in v1? Everything else is recognized but not implemented). Reinforced by **Phase: Iteration Loop Definition** (defines the loop concretely for the v1 vertical, not abstractly for all verticals).

---

### Pitfall 7: Context Pollution Across Agents

**What goes wrong:**
Long-running agents (research, verification, iteration) leak context to each other. The research agent for "competitor analysis" inherits the full main conversation including the user's stack preferences, biasing its findings. The story-implementation agent inherits the design phase's debate, picking up rejected ideas as if they were decisions. State leaks both ways: agents pull in irrelevant context from the main session, and unrelated agent outputs end up in other agents' working sets.

**Why it happens:**
1. Spawning an agent via subagent / Task tool can either pass the full context or a scoped one — the cheap default is "pass everything that might be relevant."
2. Without explicit context boundaries, every agent gets a slightly different copy of "everything," and information that should be private (e.g., user's offhand opinions) becomes load-bearing for later agents.
3. Industry-confirmed: 2026 multi-agent orchestration writeups specifically call out "sensitive data in one agent's context can leak to another, requiring explicit data classification, context filtering between agents, and audit logging of all data flows" (Source: Cogent).

**Consequences:**
- Hallucinated decisions ("the user said no to React" when they did not).
- Drift from user intent across phases.
- Tokens burned on irrelevant context, which forces summarization, which loses fidelity.

**How to avoid:**
- **Each spawned agent gets a *minimal, named, written* context bundle** (a file in `.maud/contexts/` or equivalent). The agent does NOT see the main conversation transcript; it only sees the inputs explicitly named in its bundle.
- **Inputs are paths to artifacts**, not pasted prose. The research agent gets `path: .maud/PROJECT.md`, not 4KB of pasted summary. This forces a single source of truth.
- **Outputs are also files**, not return values to the main conversation. The main conversation reads the output file. (PROJECT.md already mandates this for `.maud/log.md`.)
- **No "kitchen sink" context arguments.** Forbid context-bundle entries like "everything in `.maud/` so far."
- **The audit log (`.maud/log.md`) records context-bundle contents per spawn**, so the user can trace later which agent saw what.

**Warning signs:**
- An agent's output references a fact the user mentioned only in offhand chat.
- Two agents disagree about a decision because they read different versions of the context.
- Token usage on agent spawns trends upward over a project lifetime (sign of accumulating context bloat).
- The main conversation feels "fuller" the longer the project runs — sign that agent results are bloating context instead of being filed and forgotten.

**Phase to address:** **Phase: Agent Context Boundaries** (defines the context-bundle protocol; what files an agent can read; what its output format is; how `.maud/log.md` records it).

---

### Pitfall 8: State File Corruption (`.maud/` Bricked Mid-Project)

**What goes wrong:**
`.maud/` state files get partially written, concurrently edited, or corrupted. The user resumes `/maud` and gets a JSON parse error or, worse, silent state loss. The project's audit trail and roadmap are gone. This has happened with `claude.json` itself (industry-confirmed: GitHub issue #29051 documents `claude.json` corruption from concurrent writes due to non-atomic write paths).

**Why it happens:**
1. Multiple `claude` processes can write to the same state file concurrently. Without file locking and atomic temp-then-rename, partial writes survive.
2. Editors auto-save; a user editing a story file while Maud is also writing can produce a merge conflict in markdown.
3. JSON in particular is brittle to partial writes; a half-written JSON file parses to nothing.

**Consequences:**
- Project state is lost; user must restart from a previous git commit (if one exists) or from scratch.
- Worse: silent data loss where Maud thinks it has fewer stories than it does.
- Trust collapse: even one corruption event makes users distrustful of the tool.

**How to avoid:**
- **Atomic writes for all `.maud/` state mutations:** write to `.maud/tmp/<file>.tmp`, fsync, then rename over the destination. (Industry-standard pattern; cited in 2026 crash-safe JSON guides.)
- **Prefer markdown with YAML frontmatter over JSON for human-edited files.** Markdown's failure mode is a malformed section, which is recoverable; JSON's failure mode is total parse failure. PROJECT.md, stories, design docs are markdown. JSON is reserved for *machine-only* state (e.g., `.maud/state.json` if needed).
- **Git is the backup layer.** `.maud/` is committed; every major write triggers an autocommit (or at least a stage). Recovery is `git checkout`.
- **Detect and reject concurrent `/maud` sessions.** A `.maud/.lock` file (with PID and started-at) prevents two `/maud` invocations from racing on the same project. If the lock exists and the process is alive, the second session refuses to start; if the process is dead, the lock is reclaimed.
- **Validate state on session resume.** Before resuming, parse all `.maud/` files; if any fail validation, surface the failure to the user with recovery options (restore from git, edit manually, reinitialize).

**Warning signs:**
- JSON parse errors on `/maud` resume.
- Story counts that don't match the index file.
- Modified-time skew between `index.md` and individual story files.
- Two `claude` processes running in the same project directory.

**Phase to address:** **Phase: State File Format and Persistence** (decides markdown-first; defines atomic-write helpers; defines the lock file). Reinforced by **Phase: Resume Behavior**.

---

### Pitfall 9: Hallucinated Decisions and Cascading Errors

**What goes wrong:**
An agent invents a fact (a user preference, an architectural constraint, a previous decision) and acts on it. Subsequent agents act on the agent's output as if it were ground truth. By story 5, the project has 3 decisions the user never made, and they're load-bearing. Industry-confirmed: 2026 production reports describe "an agent acts on its own wrong output ... the error compounds, and by the time a human notices, the damage has spread across multiple systems" (Source: NimbleBrain).

**Why it happens:**
1. LLMs confidently fill gaps. If asked "what database did we pick?" with no prior context, a strong model will pick one and assert it.
2. Context pollution (Pitfall 7) interacts: a fact mentioned in one agent's run gets passed forward as decided.
3. No clean source-of-truth mechanism: when the agent isn't sure, it doesn't know where to look.

**Consequences:**
- The user discovers in story 7 that "we decided on Postgres" and they don't remember deciding.
- Refactor cascades when the hallucinated decision is unwound.
- Trust collapse: once the user catches one, they suspect every decision.

**How to avoid:**
- **Every consequential decision is recorded in a single canonical file** (PROJECT.md `Key Decisions` table, plus `.maud/architecture.md`, plus `.maud/design/`). Agents are told: if a decision is not in these files, it is not decided.
- **Agents must cite the file and line for any decision they reference.** "Per architecture.md L23, we use SQLite." If they cannot cite, they must surface the gap to the user and ask, not guess.
- **Phase-transition gates check decision-citation coverage.** Before advancing past Architecture, every architectural decision in the artifact must trace to the user's actual approval (recorded in `.maud/log.md`).
- **The audit log (`.maud/log.md`) is the chain-of-custody.** Every agent writes "I decided X because of Y from Z." User can review.
- **Reduce "fill-the-gap" pressure:** prompts explicitly tell agents that "I don't know, ask the user" is the correct answer when facts are missing.

**Warning signs:**
- A story references a decision the user can't trace in PROJECT.md / architecture.md.
- The audit log has decisions made by agents (not by the user).
- Agents disagree across spawns on a basic fact.

**Phase to address:** **Phase: Source-of-Truth Discipline** (defines decision-citation protocol; requires `.maud/log.md` chain-of-custody). Reinforced in **Phase: Architecture** and **Phase: Stories**.

---

### Pitfall 10: Dogfooding Bootstrap Trap (Maud Builds Maud, Then Can't Be Used Until It's Reliable)

**What goes wrong:**
Maud is built using GSD. The success criterion is "Maud can build the next project." But the gap between "Maud is good enough to be used" and "Maud is reliable enough to bootstrap itself" is large. If Maud is buggy when used to build the next project, you can't use Maud to fix Maud easily — you fall back to GSD, which is the thing you're trying to escape. Worse, GSD-built Maud may have GSD's failure modes baked in subtly (e.g., a phase template that has 5 phases by accident).

**Why it happens:**
1. Self-hosting is a known-hard pattern (compiler bootstrapping is the classic case): there's a reliability cliff before the tool can be its own development environment.
2. The bootstrap pressure encourages premature feature work in v1: "we need this for the *next* project, so add it now."
3. The temptation to debug Maud-with-Maud kicks in too early, before Maud is stable enough to iterate on itself.

**Consequences:**
- Either v1 over-scopes (adds features for hypothetical next projects) or under-scopes (can't actually build anything).
- The "use Maud to build Maud" handoff is risky; if it fails, the user reverts to GSD and Maud is shelved.

**How to avoid:**
- **Define a concrete bootstrap test** before v1 starts: "Maud-built-with-GSD can build [specific small next project, e.g., a Claude Code plugin called Foo] end-to-end without falling back to GSD." This is the v1 success criterion. PROJECT.md already has this; make it concrete by naming the second project up front.
- **First project Maud builds (after itself) is small and similar in shape** — another Claude Code plugin or CLI tool — so Maud doesn't need to handle a new project type yet.
- **Maud-on-Maud iteration is disallowed for v1.** Improvements to Maud during v1 are made via GSD. Only after v1 ships and the bootstrap test passes does Maud-on-Maud iteration begin.
- **Keep v1 scope minimal.** Resist adding features that "would be nice for the second project." If v1 doesn't strictly need it for itself + bootstrap test, defer.
- **Guard rails for self-modification (post-v1):** when Maud edits its own plugin code, it does so on a branch, runs its own test suite, and requires user approval before merging — same discipline as any other story.

**Warning signs:**
- v1 scope creeping with "we'll need it for the next project."
- Discussions of Maud-builds-Maud iteration starting before v1 ships.
- The bootstrap test ("Maud builds project Foo") is not concretely named.
- Plans to use Maud to fix Maud during v1.

**Phase to address:** **Phase: Define V1 Scope and Bootstrap Test** (early phase; names the bootstrap test concretely; locks scope to "minimum to pass the bootstrap test"). Enforced throughout via scope discipline.

---

## Moderate Pitfalls

### Pitfall 11: Improperly Scoped Slash Commands (Plugin Anti-Pattern)

**What goes wrong:**
Maud is split into multiple slash commands (`/maud:research`, `/maud:design`, `/maud:story`) instead of a single stateful `/maud`. The user has to remember which command to run when. The state model leaks into the UX. Industry pattern (2026): "If you have a long list of complex, custom slash commands, you've created an anti-pattern" (Source: Builder.io / Shrivu Shankar).

**Why it happens:**
Splitting commands is the obvious decomposition for engineers; each command maps to a phase. But it's a bad UX: the user has to know the phase model.

**Prevention:**
PROJECT.md already mandates a single stateful `/maud`. Reaffirm this: there is *one* user-facing command. Internal organization can be many sub-agents and many internal modes, but the user types `/maud` and Maud routes based on `.maud/` state.

**Phase to address:** **Phase: Plugin Surface Definition** (lock single-command surface).

---

### Pitfall 12: Hook Misconfigurations and Permissions Issues

**What goes wrong:**
Maud uses hooks (PreToolUse, etc.) to enforce behavior — say, "no commits without test pass." Hook configurations are sensitive to scope precedence: managed > command-line > local > project > user. A hook in `~/.claude/settings.json` doesn't fire in a project; a hook in `.claude/settings.local.json` overrides the committed one silently. (Industry-confirmed: 30+ open Claude Code issues on permission/hook scope precedence.)

**Why it happens:**
Settings precedence is non-obvious; community workarounds are emerging but not standardized.

**Prevention:**
- **Maud's plugin manifest specifies hooks at project scope** (`.claude/settings.json` committed to the repo, not `.local.json`).
- **Document the scope hierarchy** in user-facing docs.
- **Self-test hooks on `/maud` startup:** verify expected hooks are loaded; if not, surface an error with diagnostic info.
- **Prefer hooks over the permission system** for any logic that must always fire (industry pattern: hooks are more reliable than permission rules).

**Phase to address:** **Phase: Plugin Hook & Permissions Setup**.

---

### Pitfall 13: Monolithic Agent Prompts That Break on Small Changes

**What goes wrong:**
Each agent has a 2000-line prompt that does many things. Editing one section breaks unrelated behavior. Composition is impossible — agents can't be reused. Industry pattern: "Skills cut token usage by 25-40% compared to monolithic prompts" (Source: newline).

**Why it happens:**
1. Easier to write one big prompt than to design a composition.
2. Prompts grow over time as edge cases are added; nobody refactors.

**Prevention:**
- **Modular agent prompts:** core role + composable instruction blocks (similar to GSD's pattern of `<role>`, `<philosophy>`, `<execution_flow>` sections).
- **Skills for cross-cutting capabilities** (e.g., "writing markdown frontmatter," "atomic file writes") rather than duplicating in every agent.
- **Test prompts with concrete fixtures** before shipping changes.
- **Cap agent prompt size** — soft limit at e.g. 800 lines, hard requirement to refactor at 1500.

**Phase to address:** **Phase: Agent Prompt Architecture** (defines modular composition). Enforced by **Phase: Build Each Agent** (each agent must conform).

---

### Pitfall 14: Plugin Version Skew and Manifest Drift

**What goes wrong:**
Maud is installed locally; the user updates it; an existing project's `.maud/` was created with v0.3 conventions and doesn't match v0.4. `/maud` resume fails or, worse, silently misinterprets state.

**Why it happens:**
Local install = no managed migration path. User has multiple projects at multiple versions.

**Prevention:**
- **Version stamp in `.maud/config.json` (or equivalent)** captures the Maud version that initialized the project.
- **On `/maud` start, compare installed version vs project version.** If different, surface the diff and offer a migration path or a "stay on old version" path.
- **Migrations are explicit, named, and idempotent.** No silent "upgrade-on-resume."
- **v1 is forgiving:** since v1 is local-only and probably used by one developer, full migration tooling is overkill. A version warning + manual migration notes are sufficient.

**Phase to address:** **Phase: Versioning and Migration Strategy** (likely small in v1).

---

### Pitfall 15: Dead Backlog Stories (PM Anti-Pattern)

**What goes wrong:**
Stories accumulate in `Backlog` for months. They become stale (the project moved on), or the user doesn't remember why they're there. The backlog becomes a graveyard, not a queue.

**Why it happens:**
PM tools optimize for adding stories, not pruning them. Solo developers especially never have time to groom backlogs.

**Prevention:**
- **Periodic backlog review** suggested but not forced. When the user runs `/maud` after N days idle, Maud surfaces "you have X stories in Backlog older than Y days. Want to review?"
- **Stories have a `created_at` and optional `expires_at`.** Stale stories are visually flagged.
- **Allow story deletion as a first-class operation.** Removing a story is normal, not a failure.
- **Don't auto-delete.** Solo devs hate losing work.

**Phase to address:** **Phase: Story Lifecycle & Backlog Management**.

---

### Pitfall 16: Unclear "Done" Definitions

**What goes wrong:**
A story is marked Done but the user later realizes it wasn't really done — tests didn't cover the edge case, or "deployed" meant "merged to main" not "live in production." Story-level done-ness drifts.

**Why it happens:**
"Done" varies by project type (PROJECT.md acknowledges this) and by story. Without explicit success criteria per story, "done" becomes whatever the agent decided.

**Prevention:**
- **Each story has explicit `success_criteria`** (already in PROJECT.md). The story can't move to Done unless all criteria are checked.
- **Project-level "shippable" definition** is set during Architecture phase (merge-to-main vs deploy-to-staging vs deploy-to-prod) and stories inherit it unless they override.
- **The user's manual final verification** (PROJECT.md) is the last gate. No automated approval.
- **Audit log records the verification:** which criteria passed, who/what verified each.

**Phase to address:** **Phase: Story Schema** (defines `success_criteria` field) and **Phase: Verification Pipeline** (defines automated → manual gate).

---

### Pitfall 17: Lost Audit Trail (Agent Log Becomes Useless)

**What goes wrong:**
`.maud/log.md` either bloats (every agent writes a paragraph; log becomes unreadable) or starves (agents log nothing useful and the log is just timestamps). Industry pattern: shift from "State Logging" to "Intent Logging" — the log should capture *what was decided and why*, not just *what happened* (Source: LoginRadius).

**Why it happens:**
1. No format guidance → freestyle entries → bloat.
2. Format too rigid → entries are perfunctory and skip rationale → starvation.

**Prevention:**
- **Structured log entries:** timestamp, agent name, phase, action, decision (if any), citation. Free-text rationale is bounded (~3 sentences).
- **Log writes are append-only.** Never edit prior entries.
- **Log is human-scrollable.** Sectioned by phase for navigability.
- **Spawn-context bundle is referenced (not pasted) in the log entry.** Keeps log readable.

**Phase to address:** **Phase: Audit Log Schema**.

---

### Pitfall 18: Solo Dev Builds Features for Hypothetical Other Users

**What goes wrong:**
Maud accumulates features the developer doesn't actually use, justified by "someone else might want this." The product gets heavier; the bootstrap test gets harder to pass; v1 ships late.

**Why it happens:**
Tool-builder bias: building tools is fun, building only the parts you use is constraining.

**Prevention:**
- **YAGNI rule, strict in v1:** if the feature isn't used by *the user themselves* in the next 30 days, defer.
- **PROJECT.md already excludes most "nice-to-have" features** (multi-user, brownfield, integrations, marketplace). Honor those exclusions.
- **Track feature usage during dogfooding.** If the user hasn't used a feature in a month, consider removing it.

**Phase to address:** Cross-cutting; enforced by **Phase: V1 Scope** discipline.

---

## Minor Pitfalls

### Pitfall 19: Architectural Drift via Stylistically-Clean-But-Inconsistent Code

**What goes wrong:** AI generates clean code that doesn't match the rest of the codebase's conventions. Over time, codebase becomes a mosaic.

**Prevention:** Agents read `.maud/architecture.md` and the existing codebase before generating. Lint and style rules in repo are enforced. Per-story code review agent flags drift.

**Phase to address:** **Phase: Iteration Loop** (verification step includes consistency check).

---

### Pitfall 20: Negative-Only Constraints in Agent Prompts

**What goes wrong:** Agent prompts say "never use X" without saying what to use instead. Agent gets stuck when X seems necessary. (Industry pattern.)

**Prevention:** For every "don't do X" rule, provide "do Y instead." Audit prompts for negative-only constraints before shipping.

**Phase to address:** **Phase: Agent Prompt Architecture**.

---

### Pitfall 21: CLAUDE.md / Plugin Bloat via @-References

**What goes wrong:** Maud's CLAUDE.md or top-level prompts @-reference many files; each reference embeds the whole file in context every run. Token bloat. (Industry pattern.)

**Prevention:** Use *paths* (not @-references) and tell the agent when to read each file. Lazy-load.

**Phase to address:** **Phase: Plugin Surface / CLAUDE.md Design**.

---

### Pitfall 22: User Can't Tell Where They Are

**What goes wrong:** User resumes `/maud`, sees a flurry of agent activity, can't tell which phase they're in or what decisions are pending.

**Prevention:** Resume always begins with a "you are here" summary: current phase, story counts, last user-approved checkpoint, next required action. PROJECT.md already specifies this.

**Phase to address:** **Phase: Resume Behavior**.

---

### Pitfall 23: Premature Optimization

**What goes wrong:** v1 pursues optimizations (parallel agent spawns, caching, fancy prompt compression) before v1 even works.

**Prevention:** v1 ships in "obvious correct slow" mode. Optimizations only after the bootstrap test passes.

**Phase to address:** Cross-cutting; **Phase: V1 Scope** discipline.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| JSON for state files (vs. markdown) | Easy to parse programmatically | Brittle to corruption; not human-friendly | Only for machine-only state (e.g., locks, version stamps); never for user-facing state |
| Big monolithic agent prompts | Fast to write, single source for one role | Edits break things; hard to compose | Until a third reuse case appears, then refactor |
| Sync writes (no atomic temp+rename) | Less code | Concurrent corruption risk | Never for `.maud/` state; acceptable for transient `.maud/tmp/` files |
| Hard-coded project type = "CLI" | Ships fastest in v1 | Locks Maud to one vertical | Acceptable for v1 vertical scope; must be planned-for in v2 |
| No version stamp in `.maud/` | Saves a file | Future migrations are guesswork | Never |
| Agent passes full context to subagent | Less code | Context pollution, token cost | Never; always pass scoped bundles |
| Single approval prompt format for everything | Visual consistency | Approval fatigue | Never; tier prompts visually |
| No git autocommit | Less Git noise | State recovery requires manual git skills | Acceptable for v1 if user is asked to commit themselves; offer autocommit as opt-in |

---

## Integration Gotchas

Maud has minimal external integrations in v1 (PROJECT.md: local-only, no external PM tools, no remote services). The integrations that exist are:

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Claude Code plugin manifest | Hooks/settings at user scope instead of project scope | Plugin specifies hooks/settings in project-level files committed to the repo |
| Git | Committing `.maud/` partially (some state outside Git) | All `.maud/` is committed; document this in README so users don't gitignore it |
| Subagents (Task tool) | Passing main conversation as subagent context | Pass scoped context-bundle via files in `.maud/contexts/` |
| Hooks (PreToolUse etc.) | Assuming hooks always fire across scopes | Self-test hooks on startup; warn if expected hooks not loaded |
| File system | Non-atomic writes to `.maud/` files | Atomic temp+fsync+rename helper used for all state mutations |

---

## Performance Traps

These are minor for v1 (single-user, local) but worth flagging:

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Re-spawning agents for every story with full project context | Token cost grows linearly with project size | Pass minimal scoped bundles per spawn | Around 20+ stories or after several phases of accumulated artifacts |
| Reading the whole audit log into context | Slow `/maud` resume | Index the log; on resume, read only the most recent N entries plus a phase-summary line | Around 100+ log entries |
| Reading entire research artifacts into every iteration agent | Token bloat per story | Stories cite specific research/design files; iteration agent reads only cited files | When research artifacts exceed ~5k tokens combined |
| Synchronous test runs blocking user | Long stretches with no feedback | Stream test output to user; spawn long verification as background agent (PROJECT.md already mandates this) | Whenever tests take >30s |

---

## Security Mistakes

Maud is local-only, single-user, no remote services. Security surface is therefore narrow but not zero:

| Mistake | Risk | Prevention |
|---------|------|------------|
| Agent can write outside `.maud/` and project root | Untrusted prompt could exfiltrate or modify user's machine | Hooks restrict tool calls to project root; PreToolUse hook denies writes outside the project tree |
| Logging secrets (API keys, tokens) in `.maud/log.md` | Secrets committed to git | Log entries are template-bounded (no raw command output); secrets-detection hook on commit |
| Agent reads sensitive files (`.env`, `~/.ssh/`) into context | Context leaks into LLM provider | Default-deny path patterns: `.env`, `~/.aws/`, `~/.ssh/`, etc. are not readable by agents |
| `.maud/` accidentally `.gitignore`d | Loss of audit trail and recoverability | README + Maud init explicitly documents committing `.maud/`; `/maud` warns if `.maud/` matches a gitignore pattern |
| Plugin runs arbitrary user input as commands | Prompt injection from a malicious story description | Stories pass through Maud agents as data, never as commands; commands originate from Maud's prompt, not from `.maud/` content |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Maud asks closed-form approvals on consequential decisions | Approval theater; rubber-stamping | Open-ended elicitation on consequential decisions (Pitfall 1) |
| User can't intervene mid-phase without restarting | Audibles become impossible | Maud listens for user input at every agent transition; "audibles welcome" is a first-class feature (PROJECT.md) |
| Long agent runs with no progress signal | User doesn't know if it's working or hung | Stream agent log entries to user as they happen |
| Re-asking the same question across phases | User feels unheard | Decisions are recorded in PROJECT.md / architecture.md; agents check before asking |
| Verbose phase intros | User skims and misses important info | Resume summary is short; <10 lines |
| No way to undo a phase advance | One mis-approval ruins the run | Git history is the undo; document `/maud rollback` (or equivalent) as a real command |
| Project-type misclassification stuck the project in wrong vertical | User has to restart | Classification is reversible; user can revisit and reclassify |
| User can't see what an agent is about to do | Trust collapse on surprise | Agents announce intent before acting on consequential changes |

---

## "Looks Done But Isn't" Checklist

Things that appear complete but are missing critical pieces. Use during build verification.

- [ ] **Phase Approval Mechanic:** Often missing the *open-ended* mode — closed-form is shipped but heavy elicitation is not. Verify: open a project, hit a phase boundary, confirm Maud presents thinking and asks "what would you change?" not "approve A/B/C?"
- [ ] **Project-Type Adaptation:** Often missing the meta-step ("what does design mean here?"). Verify: initialize a CLI project and a web app project; confirm design artifacts differ in shape, not just content.
- [ ] **Atomic State Writes:** Often missing on the unhappy path. Verify: kill `/maud` mid-write; resume; confirm no corruption.
- [ ] **Audit Log:** Often missing decision *rationale*, only has actions. Verify: read a project's log and confirm you can answer "why was X decided?" from the log alone.
- [ ] **Bootstrap Test:** Often missing a concrete second project. Verify: name the second project up front; build it before declaring v1 done.
- [ ] **Resume Summary:** Often missing the "next required action" line. Verify: resume an in-progress project; confirm summary tells you exactly what to do next.
- [ ] **Story Success Criteria:** Often missing for trivial-looking stories. Verify: every story file has a non-empty `success_criteria` block.
- [ ] **Single `/maud` Command:** Often broken by adding "helper" subcommands. Verify: only `/maud` is exposed; no `/maud:foo` companions.
- [ ] **Greenfield-Only Guardrail:** Often missing the rejection path. Verify: run `/maud` in a non-empty project; confirm Maud declines (or asks for explicit override).
- [ ] **Approval Tiering:** Often missing visual differentiation. Verify: in a session, count approval prompts; if all look the same, tiering is missing.
- [ ] **JIT Mini-Loop is Rare:** Often becomes ceremony. Verify: in a typical project, JIT mini-loops fire on <30% of stories.
- [ ] **Context Bundle Per Spawn:** Often passes too much. Verify: inspect `.maud/log.md`; each spawn entry should list specific input files, not "all of `.maud/`."

---

## Recovery Strategies

When pitfalls occur despite prevention:

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| State file corruption (P8) | LOW | `git checkout -- .maud/` to restore; if no commit, recreate from `.maud/log.md` history |
| Hallucinated decision discovered late (P9) | MEDIUM | Stop current phase; correct PROJECT.md / architecture.md; rewind affected stories to Backlog with note explaining what changed |
| Approval fatigue setting in (P5) | LOW | Reduce prompt frequency; introduce visual tiering retroactively; user explicitly recalibrates verification grain |
| Flexibility-as-vaporware (P6) | HIGH | Hard-pivot v1 scope to one concrete vertical; cut abstract code paths; ship narrow before broad |
| Bootstrap test failing (P10) | HIGH | Fall back to GSD for the next iteration of Maud; do NOT try to fix Maud with Maud; treat the failure as v1 not-yet-shipped |
| Dead backlog (P15) | LOW | Schedule a one-time review session; bulk-archive stale stories with a note |
| Context pollution (P7) | MEDIUM | Audit recent agent runs; identify which decisions came from agents vs user; re-derive from PROJECT.md ground truth |
| Plugin version skew (P14) | LOW-MEDIUM | Pin Maud version per project (`.maud/config.json`); migrate manually when ready |
| Per-phase research creep (P2) | MEDIUM | Stop the offending phase; consolidate research-style notes back into `.maud/research/`; refactor phase prompts to forbid re-research |

---

## Pitfall-to-Phase Mapping

How roadmap phases should address each pitfall. Phase names below are *suggested* — actual roadmap phasing is the next planning step.

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| 1. Multiple-choice rubber-stamping | Phase Transition / Approval Mechanic | Manual test: every phase boundary uses open-ended elicitation on consequential decisions |
| 2. Per-phase research+plan cycles | Roadmap Structure & Phase Order | No phase template includes a research step after the Research phase |
| 3. Arbitrary phase counts / rigid templates | Project-Type Classification & Adaptive Structure | Two project types produce structurally different artifacts |
| 4. No design phase | Design Phase Definition | Every project has a non-trivial design artifact; stories cite design |
| 5. Verification fatigue | Phase Transition / Approval Mechanic + Iteration Loop | Heavy prompts <10 per project lifecycle; visual tiering present |
| 6. Flexibility-as-vaporware | Pick the V1 Vertical | One project type is end-to-end functional; bootstrap test passes |
| 7. Context pollution across agents | Agent Context Boundaries | `.maud/log.md` shows scoped context-bundle per spawn |
| 8. State file corruption | State File Format and Persistence + Resume Behavior | Atomic-write helper used; lock file present; resume validates |
| 9. Hallucinated decisions | Source-of-Truth Discipline + Architecture + Stories | Every decision in artifacts is cited; agents fail-loud on missing facts |
| 10. Dogfooding bootstrap trap | Define V1 Scope and Bootstrap Test | Concrete second project named; v1 ships before Maud-on-Maud iteration |
| 11. Improperly scoped slash commands | Plugin Surface Definition | Only `/maud` exposed |
| 12. Hook misconfigurations | Plugin Hook & Permissions Setup | Self-test on startup confirms expected hooks load |
| 13. Monolithic agent prompts | Agent Prompt Architecture | Prompt size budget enforced; modular composition |
| 14. Plugin version skew | Versioning and Migration Strategy | Version stamp present; warning on mismatch |
| 15. Dead backlog stories | Story Lifecycle & Backlog Management | Stale stories surfaced; deletion is first-class |
| 16. Unclear "Done" definitions | Story Schema + Verification Pipeline | Every story has explicit `success_criteria` |
| 17. Lost audit trail | Audit Log Schema | Log entries follow structured schema with rationale |
| 18. Features for hypothetical users | V1 Scope (cross-cutting) | Every v1 feature is used by the developer themselves |
| 19. Architectural drift in code | Iteration Loop (verification step) | Code-review agent checks consistency |
| 20. Negative-only constraints | Agent Prompt Architecture | Every prompt audited for "do X instead of Y" formulations |
| 21. CLAUDE.md @-bloat | Plugin Surface / CLAUDE.md Design | Lazy-load via paths, not @-references |
| 22. User can't tell where they are | Resume Behavior | Resume produces "you are here" summary |
| 23. Premature optimization | V1 Scope (cross-cutting) | v1 ships before any optimization work |

---

## Sources

**User-confirmed (PROJECT.md):** Four explicit GSD failure modes (rubber-stamp prompts, per-phase research cycles, rigid templates, no design phase) and two calibration questions (vaporware risk, verification fatigue) — HIGH confidence as product requirements.

**AI agent orchestration / multi-agent failure modes (MEDIUM-HIGH):**
- [When AI Agents Collide: Multi-Agent Orchestration Failure Playbook for 2026 — Cogent](https://cogentinfo.com/resources/when-ai-agents-collide-multi-agent-orchestration-failure-playbook-for-2026)
- [AI Agent Failure Modes — NimbleBrain](https://nimblebrain.ai/why-ai-fails/agent-governance/agent-failure-modes/)
- [7 AI Agent Failure Modes and How to Prevent Them — Galileo](https://galileo.ai/blog/agent-failure-modes-guide)
- [AI Agent Failure Pattern Recognition — MindStudio](https://www.mindstudio.ai/blog/ai-agent-failure-pattern-recognition)
- [Black box AI drift — Stack Overflow Blog](https://stackoverflow.blog/2026/04/23/black-box-ai-drift-ai-tools-are-making-design-decisions-nobody-asked-for/)
- [Why AI Agents Break: A Field Analysis of Production Failures — Arize](https://arize.com/blog/common-ai-agent-failures/)
- [The Hidden Risk of AI-Generated Code — Dev Journal](https://earezki.com/ai-news/2026-05-04-the-ai-code-bug-nobody-catches-until-its-too-late/)

**Approval fatigue / verification calibration (MEDIUM-HIGH):**
- [Approval Fatigue Is an Agent Security Bug — Developers Digest](https://www.developersdigest.tech/blog/approval-fatigue-agent-security-bug)
- [The Agent Approval Fatigue Problem — Molten.Bot](https://molten.bot/blog/agent-approval-fatigue/)
- [Claude Code auto mode: a safer way to skip permissions — Anthropic](https://www.anthropic.com/engineering/claude-code-auto-mode)
- [Approval Fatigue — Encyclopedia of Agentic Coding Patterns](https://aipatternbook.com/approval-fatigue)
- [The 'AI Agent Fatigue' Problem — Ralphable](https://ralphable.com/blog/ai-agent-fatigue-claude-code-atomic-skills-solution)
- [AI UX Patterns: Verification — ShapeofAI](https://www.shapeof.ai/patterns/verification)

**Claude Code plugin / slash command / subagent (HIGH for official, MEDIUM for community):**
- [Slash Commands — Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/slash-commands)
- [Create custom subagents — Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [Configure permissions — Claude Code Docs](https://code.claude.com/docs/en/permissions)
- [How I Use Every Claude Code Feature — Shrivu Shankar](https://blog.sshh.io/p/how-i-use-every-claude-code-feature)
- [How I use Claude Code (+ my best tips) — Builder.io](https://www.builder.io/blog/claude-code)
- [Claude Skills and Subagents Reduce Prompt Bloat — newline](https://www.newline.co/@Dipen/claude-skills-and-subagents-reduce-prompt-bloat--f2920804)
- [Claude Code Hooks: Complete Guide — claudefa.st](https://claudefa.st/blog/tools/hooks/hooks-guide)
- [How to Fix Claude Code's Broken Permissions — DEV Community](https://dev.to/boucle2026/how-to-fix-claude-codes-broken-permissions-with-hooks-23gl)
- [Prompting best practices — Claude Docs](https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/claude-4-best-practices)

**State files / atomic writes (HIGH):**
- [Crash-safe JSON at scale — DEV Community](https://dev.to/constanta/crash-safe-json-at-scale-atomic-writes-recovery-without-a-db-3aic)
- [claude.json corrupted by concurrent writes — GitHub anthropics/claude-code#29051](https://github.com/anthropics/claude-code/issues/29051)
- [A brief analysis of JSON file-backed storage — Mozilla](https://mozilla.github.io/firefox-browser-architecture/text/0012-jsonfile.html)

**Solo dev / dogfooding / bootstrap (HIGH):**
- [Self-hosting (compilers) — Wikipedia](https://en.wikipedia.org/wiki/Self-hosting_(compilers))
- [selfdogfood — IndieWeb](https://indieweb.org/selfdogfood)
- [Lightweight Project Management for Devs 2026 — GitScrum](https://gitscrum.com/en/solutions/pains/lightweight-project-management-for-programmers)
- [Solo Developer Project Management Systems 2025 — Apatero Blog](https://apatero.com/blog/solo-developer-project-management-systems-2025)

**YAGNI / abstraction / over-engineering (HIGH):**
- [bliki: Yagni — Martin Fowler](https://martinfowler.com/bliki/Yagni.html)
- [YAGNI Principle — GeeksforGeeks](https://www.geeksforgeeks.org/software-engineering/what-is-yagni-principle-you-arent-gonna-need-it/)
- [Do you really need that abstraction? — CodeOpinion](https://codeopinion.com/do-you-really-need-that-abstraction-or-generic-code-yagni/)

**Audit logging for AI agents (MEDIUM-HIGH):**
- [Auditing and Logging AI Agent Activity — LoginRadius](https://www.loginradius.com/blog/engineering/auditing-and-logging-ai-agent-activity)
- [MCP Audit Logging — Tetrate](https://tetrate.io/learn/ai/mcp/mcp-audit-logging)
- [Audit Logging for AI: What Should You Track — Medium](https://medium.com/@pranavprakash4777/audit-logging-for-ai-what-should-you-track-and-where-3de96bbf171b)

---

*Pitfalls research for: Claude Code plugin orchestrating AI-assisted software development project management*
*Researched: 2026-05-08*
