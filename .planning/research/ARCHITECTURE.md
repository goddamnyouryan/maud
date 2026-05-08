# Architecture Research

**Domain:** Claude Code plugin (stateful project-management orchestrator)
**Researched:** 2026-05-08
**Confidence:** HIGH (Claude Code plugin platform, GSD as concrete reference) / MEDIUM (specific `.maud/` schema choices — judgment calls until validated by use)

---

## Executive Summary

Maud is a Claude Code plugin shaped almost identically to GSD: a marketplace-distributable directory containing a `.claude-plugin/plugin.json` manifest, a `commands/` directory with one stateful slash command, and an `agents/` directory with the worker agents the command spawns via the `Agent` tool (formerly `Task`).

The single most consequential constraint, **confirmed in the official Claude Code docs and observable in GSD's `run-milestone.md` (line 22-24)**: subagents cannot spawn other subagents. This means **the orchestrator loop must live in the slash command's main conversation context**, not in an agent. Every state read, every phase-transition decision, every `AskUserQuestion` to the user, and every `Agent(...)` spawn must originate from the `/maud` command body itself. Agents are leaf workers that get pre-assembled context, do one job, and return a structured result. This is the dominant architectural reality and shapes everything else.

Maud reuses GSD's overall topology (orchestrator + leaf agents + on-disk state) but should diverge from GSD's structure in three ways: (1) add a project-type-aware **project profile** that drives dynamic structure rather than hardcoded phase shapes, (2) replace GSD's roadmap-of-phases model with a **flat priority-ordered story list** (Trello-style), and (3) elevate the agent log from a side artifact to a **mandatory write-on-exit contract** for every spawned agent.

The state machine is best modeled as a **directed graph with explicit verification gates between every transition**, not a strict linear pipeline — Maud's core differentiator from GSD is that no phase can advance without explicit user approval, and the user must be able to call audibles (re-prioritize, add/remove stories, revisit design) without breaking state. The state file (`.maud/state.json`) is the single source of truth for "where the user is" and is read at the top of every `/maud` invocation.

---

## Standard Architecture

### System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Claude Code Session (main thread)                  │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  /maud  ←  the ONLY slash command. Stateful. Single entry.     │  │
│  │  ────────────────────────────────────────────────────────────  │  │
│  │  1. Read .maud/state.json                                       │  │
│  │  2. Branch on current phase                                     │  │
│  │  3. Load relevant artifacts                                     │  │
│  │  4. Spawn agents (parallel where possible)                      │  │
│  │  5. Synthesize results, present to user                         │  │
│  │  6. AskUserQuestion at every phase boundary                     │  │
│  │  7. On approval: write state.json, log to log.md, advance       │  │
│  └────────────────────────────────────────────────────────────────┘  │
│           │                       │                       │           │
│           │  Agent() spawn        │  Agent() spawn        │  Agent()  │
│           ▼                       ▼                       ▼           │
│  ┌────────────────┐      ┌────────────────┐     ┌────────────────┐   │
│  │  researcher    │      │  designer      │     │  iterator      │   │
│  │  (one of N)    │      │  (one of N)    │     │  (per story)   │   │
│  │  ────────────  │      │  ────────────  │     │  ────────────  │   │
│  │  fresh ctx     │      │  fresh ctx     │     │  fresh ctx     │   │
│  │  reads state   │      │  reads state   │     │  reads state   │   │
│  │  reads inputs  │      │  reads inputs  │     │  reads inputs  │   │
│  │  writes files  │      │  writes files  │     │  writes files  │   │
│  │  appends log   │      │  appends log   │     │  appends log   │   │
│  │  returns text  │      │  returns text  │     │  returns text  │   │
│  └────────────────┘      └────────────────┘     └────────────────┘   │
│                                                                        │
│  Verifiers (security, code-review, test-coverage, qa) follow the      │
│  same shape — spawned in parallel after iteration, each reads the     │
│  same diff/working-tree, each writes a verification report file.      │
└──────────────────────────────────────────────────────────────────────┘
                              ▲
                              │ all I/O
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         .maud/  (on-disk state)                       │
│                                                                        │
│  state.json          ←  current phase, current story, sub-state       │
│  PROJECT.md          ←  frozen initial spec + user goals              │
│  profile.md          ←  project-type-aware shape (research dims,      │
│                         design format, iteration loop, verification)  │
│  research/           ←  research outputs + RESEARCH-SUMMARY.md        │
│  architecture.md     ←  tech stack + system shape                     │
│  design/             ←  artifacts; format dictated by profile.md      │
│  stories/            ←  one .md per story + INDEX.md priority list    │
│  log.md              ←  append-only audit trail (every agent writes)  │
│  config.json         ←  workflow prefs (commit cadence, model profile)│
└──────────────────────────────────────────────────────────────────────┘
                              ▲
                              │ git tracks everything
                              ▼
                          [user's repo]
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| **Plugin manifest** (`.claude-plugin/plugin.json`) | Identity, version, marketplace metadata. Required for distribution. | Static JSON, ~10 lines. |
| **`/maud` slash command** (`commands/maud.md`) | The supervisor. Stateful entry point. Reads state, decides next step, spawns agents, mediates user verification, writes state on advance. NEVER does sub-research itself; always delegates. | Markdown file with YAML frontmatter (`allowed-tools: Read, Write, Bash, Agent, AskUserQuestion`). Body is the long playbook with one section per phase. |
| **Worker agents** (`agents/<name>.md`) | Single-purpose contexts. Each gets pre-assembled prompt with state/inputs inlined, does its job, writes outputs to `.maud/`, appends a log entry, returns structured marker (e.g. `## RESEARCH COMPLETE`). | Markdown file with YAML frontmatter (`name`, `description`, `tools`, `model`). Body is the role definition. |
| **Reference docs** (`references/*.md`) | Reusable prompt fragments injected into command/agent bodies via `@path/to/file.md` references. Things like questioning techniques, verification patterns, audit-log format. | Markdown, version-controlled with the plugin. |
| **Templates** (`templates/*.md`) | File templates the orchestrator/agents use when writing outputs. E.g. story template, log entry template, design artifact templates per project type. | Markdown with placeholders. |
| **Project state** (`.maud/`) | Single source of truth for project. Read at every `/maud` invocation. Written only by the orchestrator (state.json, log.md commits) and agents (specific output files in their swimlane). | Mix of JSON (machine state) and Markdown (human content). |
| **Git** | Implicit but mandatory component. Every advancement is a commit. Every story branch is a real git branch. Audit-trail-as-history. | Driven by orchestrator bash commands. |

### Why a single command instead of `/maud:init`, `/maud:next`, `/maud:done`, etc.

GSD has 28 commands and the user's mental model is "which one do I run now?" — that's the cognitive load Maud is explicitly removing. With a single stateful command, the user runs `/maud`, the command reads `.maud/state.json`, and it decides what phase you're in. **The state is the dispatcher**, not the command name.

This matches the [open Claude Code feature request](https://github.com/anthropics/claude-code/issues/15882) for un-namespaced single-plugin commands but works around the current limitation: the command will be `/maud:maud` until that lands, OR Maud can ship as a single-command plugin with the workaround of users typing `/maud:maud` (which the typeahead will match on `/maud`). **This is a real friction point worth flagging as a pitfall — verify in PITFALLS.md.**

---

## Recommended Project Structure

The plugin source repo (the thing that gets installed):

```
maud/                              # plugin root (becomes ~/.claude/plugins/<name>/<version>/maud or local --plugin-dir)
├── .claude-plugin/
│   └── plugin.json                # {name, description, version, author}
├── commands/
│   └── maud.md                    # the ONE slash command — orchestrator loop
├── agents/
│   ├── researcher.md              # one researcher; spawned N times in parallel with different dimensions
│   ├── research-synthesizer.md    # collapses parallel research into RESEARCH-SUMMARY.md
│   ├── designer.md                # design artifact generator; format depends on profile
│   ├── architect.md               # architecture.md author
│   ├── story-extractor.md         # turns design+architecture → stories/
│   ├── prioritizer.md             # orders stories, surfaces dependencies
│   ├── iterator.md                # per-story build loop (test-write-commit)
│   ├── verifier-security.md       # parallel verifier #1
│   ├── verifier-code-review.md    # parallel verifier #2
│   ├── verifier-test-coverage.md  # parallel verifier #3
│   └── verifier-qa.md             # parallel verifier #4
├── references/                    # injected into command/agent bodies via @path
│   ├── log-format.md              # exact log entry template + when to append
│   ├── verification-protocol.md   # how verifiers run + how to interpret results
│   ├── state-schema.md            # state.json schema authority
│   ├── questioning.md             # discovery/clarification patterns (borrow from GSD)
│   └── project-types.md           # what "project type" means + the catalog
├── templates/
│   ├── PROJECT.md                 # initial-spec template
│   ├── state.json                 # initial state.json scaffolding
│   ├── story.md                   # one-story-file template
│   ├── stories-INDEX.md           # priority-ordered index template
│   ├── architecture.md            # architecture.md template
│   ├── log-entry.md               # exact format every agent appends
│   └── profiles/                  # project-type profiles
│       ├── web-app.md
│       ├── cli.md
│       ├── game.md
│       └── ...
└── README.md                      # human-facing install + usage doc
```

### State directory layout (the user's `.maud/` in their project)

```
.maud/
├── state.json              # MACHINE STATE — current phase, current story, sub-state, last-agent
├── PROJECT.md              # frozen-after-init initial spec + clarification answers
├── profile.md              # project-type-aware shape; chosen during init
├── config.json             # user prefs (commit cadence, model profile, automation level)
├── research/
│   ├── <dim-1>.md          # one file per research dimension (dimensions chosen per project)
│   ├── <dim-2>.md
│   └── SUMMARY.md          # synthesizer output; THE canonical research summary
├── architecture.md         # tech stack + system shape (single file)
├── design/                 # format dictated by profile.md
│   ├── (web app)  pages/   wireframes/   components/
│   ├── (game)     sprites/ levels/       mechanics.md
│   ├── (cli)      commands.md   subcommand-specs/
│   └── ... (project-type-dependent)
├── stories/
│   ├── INDEX.md            # priority-ordered list with statuses (Backlog/In Progress/Done)
│   ├── 0001-<slug>.md      # one file per story; numeric prefix = creation order, NOT priority
│   ├── 0002-<slug>.md
│   └── ...
├── verifications/          # per-story verification outputs
│   └── 0003-<slug>/
│       ├── security.md
│       ├── code-review.md
│       ├── test-coverage.md
│       └── qa.md
└── log.md                  # APPEND-ONLY audit trail — every agent writes one entry on exit
```

### Structure Rationale

- **`state.json` (JSON, not markdown):** machine-readable, easy to parse with `jq`, schema-versioned, fast to read at orchestrator boot. Markdown is for humans; this is for the dispatcher.
- **`PROJECT.md` (markdown, frozen):** the human-readable record of "what we agreed to build." Frozen after initialization (per the user's explicit verification principle); changes require a re-init or a documented amendment.
- **`profile.md` (markdown, dynamic):** the project-type-aware shape. This is what makes Maud not-rigid. Picked during init from `templates/profiles/*` or generated for a novel project type. Defines: research dimensions, design artifact format, iteration loop shape, verification criteria.
- **`research/` flat (not nested):** research dimensions are flat. Each file is one dimension. SUMMARY.md is the synthesis. Mirrors GSD's `.planning/research/` exactly because that pattern works.
- **`architecture.md` single file:** PROJECT.md says "as simple as possible." A single architecture file forces consolidation; if it grows beyond a screen, the team can extract sections into subdocs in v2. **GSD's split (STACK.md + ARCHITECTURE.md + PITFALLS.md) is a research artifact pattern; for the user's actual architecture decisions, one file is right.**
- **`design/` heterogeneous:** can't be standardized — that's the whole point of project-type-aware. The profile dictates structure. Maud must NOT impose a layout here.
- **`stories/` flat with INDEX.md:** one file per story enables: easy diff per story, easy git-branch-per-story commits, parallel agents writing different stories without merge conflicts. INDEX.md is the priority list and status board (Trello model in markdown).
- **`verifications/<story>/<verifier>.md`:** parallel verifier outputs go in a story-scoped folder so artifacts don't clobber each other. Removes a coordination problem.
- **`log.md` single append-only file:** simplicity beats sophistication. Every agent's last act is "append my entry." A single file means: easy to grep, easy to scroll, easy to feed back into a future agent's context, easy to diff.

### What we're NOT doing (and why)

- **No `phases/` directory like GSD has** — Maud doesn't have phases-as-units; it has stories-as-units. The phase concept maps to states in the state machine (init/research/design/etc.) but those states don't have output directories of their own beyond what's already named.
- **No nested stories.** PROJECT.md says explicitly: flat list with checklists. Nesting can be added when reality demands it.
- **No per-phase `CONTEXT.md` files like GSD.** Maud's frontloaded model means context is captured once in PROJECT.md + research + design + architecture and persists for all stories. Per-story JIT context, when needed, is captured inside the story file itself.
- **No `STATE.md` like GSD.** GSD's `STATE.md` is a hand-curated digest. Maud's `state.json` is machine-canonical. Status presentation to the user (the "bring you up to speed" output) is computed from state.json + stories/INDEX.md + log.md tail at runtime, not stored.

---

## Architectural Patterns

### Pattern 1: Orchestrator-in-Command, Workers-in-Agents

**What:** All control flow, state reads/writes, user interaction (`AskUserQuestion`), and agent spawning happens in `commands/maud.md`. Agents only do single-shot work and return.

**When to use:** Always. This is mandatory in Claude Code because subagents cannot spawn other subagents.

**Trade-offs:**
- **Pro:** Crystal-clear data flow. The command owns state. Agents are stateless workers.
- **Pro:** Matches GSD's `run-milestone.md` architectural constraint ("the supervisor loop runs here — in this slash command's main conversation context. It CANNOT be delegated to a spawned agent because subagents cannot spawn their own subagents")
- **Con:** The `commands/maud.md` file is large (GSD's `run-milestone.md` is 58KB). Mitigate with `@references/*` and `@templates/*` injection so the command body stays a control-flow skeleton, not a wall of prompts.

**Example (sketch):**
```markdown
# commands/maud.md
---
name: maud
description: Idea-to-shipped project management orchestrator
allowed-tools: [Read, Write, Edit, Bash, Agent, AskUserQuestion]
---

@references/log-format.md
@references/state-schema.md

## Step 0: Read state
```bash
STATE=$(cat .maud/state.json 2>/dev/null || echo "{}")
PHASE=$(echo "$STATE" | jq -r '.phase // "uninitialized"')
```

## Step 1: Dispatch on phase

If $PHASE == "uninitialized": jump to Init section
If $PHASE == "research_pending": jump to Research section
If $PHASE == "research_synthesizing": ...
[...]

## Init section
[orchestrator code that asks clarifying questions, writes PROJECT.md,
 spawns no agents, then transitions phase via AskUserQuestion gate]

## Research section
[orchestrator reads profile.md to determine research dimensions,
 then spawns N researcher agents in parallel via Agent(),
 then spawns research-synthesizer, then user-verification gate]
```

### Pattern 2: Pre-Assembled Context Injection

**What:** Before every `Agent(prompt=...)` call, the orchestrator reads relevant files into bash variables and inlines them into the agent's prompt. The agent does NOT discover its context; it receives it.

**When to use:** Every agent spawn. This is GSD's `run-milestone.md` pattern (lines 196-201: "CRITICAL: Before every Task call, read file contents into bash variables. The `@` file reference syntax does NOT work in Task prompts — content must be inlined.").

**Trade-offs:**
- **Pro:** Agent context is fully reproducible from the command transcript.
- **Pro:** Avoids agent burning tokens on Read tool calls for files the orchestrator already knows it needs.
- **Pro:** Forces the orchestrator to be explicit about what each agent should see.
- **Con:** Verbose command body. Mitigate with helper bash functions defined inline.

**Example:**
```bash
PROJECT_CONTENT=$(cat .maud/PROJECT.md)
PROFILE_CONTENT=$(cat .maud/profile.md)
LOG_TAIL=$(tail -50 .maud/log.md)

Task(
  prompt="First, read /Users/.../agents/researcher.md for your role.

<project>
$PROJECT_CONTENT
</project>

<profile>
$PROFILE_CONTENT
</profile>

<recent_activity>
$LOG_TAIL
</recent_activity>

<dimension>
$DIMENSION
</dimension>

<output>
Write findings to .maud/research/$DIMENSION.md
Append entry to .maud/log.md per @references/log-format.md
Return: ## RESEARCH COMPLETE / ## RESEARCH BLOCKED
</output>",
  subagent_type="general-purpose",
  model="$RESEARCHER_MODEL",
  description="Research $DIMENSION"
)
```

### Pattern 3: Parallel Spawn for Independent Work, Sequential for Dependent

**What:** Spawn multiple `Agent()` calls in a single message when their work is independent (e.g. four research dimensions, four verifiers). Spawn sequentially when later agents need earlier outputs (e.g. researchers → research-synthesizer).

**When to use:**
- **Parallel:** Researchers (N independent dimensions), verifiers (security/code-review/test-coverage/qa all read the same diff)
- **Sequential:** Research → synthesis; design → story extraction; story implementation → verifications → user gate

**Trade-offs:**
- **Pro (parallel):** Latency wins. Four 90-second agents in parallel finish in 90s, not 360s.
- **Pro (parallel):** Each agent has fresh context, no cross-contamination.
- **Con (parallel):** All results return to the orchestrator's context simultaneously, can be expensive in tokens. Mitigate with structured-result-only returns and agents writing details to disk.

**Example:**
```
[Single message containing four Task calls, all in parallel:]
Task(prompt=..., description="Security verifier")
Task(prompt=..., description="Code-review verifier")
Task(prompt=..., description="Test-coverage verifier")
Task(prompt=..., description="QA verifier")
```

### Pattern 4: Structured Return Markers

**What:** Every agent returns text containing a `## STATUS_MARKER` line. The orchestrator greps the return for the marker to decide next action.

**When to use:** Every agent. Mandatory.

**Trade-offs:**
- **Pro:** Deterministic dispatch. No prompt-engineering "did the agent succeed?" inference.
- **Pro:** Easy to test (write fake agent returns, check orchestrator behavior).
- **Con:** Requires discipline in agent prompts. Mitigate with a `references/return-markers.md` file enumerating valid markers per agent type.

**Markers Maud uses:**
- `## RESEARCH COMPLETE` / `## RESEARCH BLOCKED`
- `## DESIGN PROPOSED` / `## DESIGN ITERATION NEEDED`
- `## STORIES EXTRACTED` (returns count + path to INDEX.md)
- `## STORY COMPLETE` / `## STORY BLOCKED` / `## STORY CHECKPOINT` (mid-iteration user input needed)
- `## VERIFICATION PASS` / `## VERIFICATION FAIL` / `## VERIFICATION FLAGGED` (passed but with notes)

### Pattern 5: Append-Only Audit Log as Agent Contract

**What:** Every agent's prompt ends with a mandatory instruction: "Before returning, append a log entry to `.maud/log.md` using @references/log-format.md." The orchestrator checks for the entry post-spawn (via `tail` + grep on the agent's identifier) and warns if missing.

**When to use:** Every agent.

**Trade-offs:**
- **Pro:** User can grep one file to understand "what did Maud do today?"
- **Pro:** Future agents can `tail .maud/log.md` to recover recent context.
- **Pro:** It's the simplest possible audit mechanism — no DB, no JSON parser, just `>>`.
- **Con:** Concurrent parallel agents may interleave writes. **Mitigate** by having each agent atomically append a fully-formed multi-line block (one shell append, not multiple writes), or use a per-agent log file that gets concatenated by the orchestrator.

**Recommended log entry format (in `references/log-format.md`):**
```
## [ISO-timestamp] [agent-name] [phase: <phase>] [story: <id-or-none>]
**Did:** [one sentence of what was done]
**Found:** [optional — surprising or important findings]
**Decided:** [optional — judgment calls made]
**Wrote:** [list of files written, paths relative to repo root]
**Status:** complete | blocked | checkpoint
[blank line]
```

### Pattern 6: Verification-Gated Phase Transitions

**What:** Every transition between phases includes an `AskUserQuestion` gate where the user must explicitly approve. The orchestrator NEVER advances state.json silently. State.json updates happen ONLY after user confirmation.

**When to use:** Every phase boundary. This is Maud's core differentiator.

**Trade-offs:**
- **Pro:** This IS the product. Without it Maud is just GSD with renamed phases.
- **Con:** More user interruption than GSD. Mitigate by making each gate substantive — show real synthesized content, not "are you sure?"

**Example:**
```
Orchestrator presents synthesized research summary, then:

AskUserQuestion(
  question="Research is complete. Here's the synthesis: [...]\n\nDoes this match the project as you understand it?",
  options=[
    "1. Approve — advance to design",
    "2. Adjust — let me clarify or re-run a dimension",
    "3. Stop here — I want to think before continuing"
  ]
)

Only on option 1 does the orchestrator:
- update state.json: phase = "design_pending"
- append log entry: "User approved research → advancing to design"
- git commit
- continue to design section
```

---

## Data Flow

### Top-Level Lifecycle Flow

```
User runs /maud
    ↓
Orchestrator reads .maud/state.json
    ↓
Orchestrator dispatches on phase value
    ↓
[For each phase, the same shape:]
    ↓
1. Orchestrator reads inputs from .maud/
    ↓
2. Orchestrator spawns agent(s) with inlined context
    ↓
3. Agent(s) write outputs to .maud/<phase-folder>/
    ↓
4. Agent(s) append entry to .maud/log.md
    ↓
5. Agent(s) return structured marker
    ↓
6. Orchestrator reads agent outputs from disk
    ↓
7. Orchestrator synthesizes + presents to user
    ↓
8. AskUserQuestion: approve / adjust / stop
    ↓
9. On approve: orchestrator writes new state.json, git commits, advances phase
   On adjust: orchestrator re-runs current phase with user input
   On stop: orchestrator exits cleanly; next /maud invocation resumes here
```

### State Transitions (Read/Write Map)

| Phase | Reads | Writes | Notes |
|-------|-------|--------|-------|
| init | (none — fresh) | `PROJECT.md`, `profile.md`, `state.json`, `config.json`, `log.md` | Orchestrator does this directly (no agents) since it's pure conversation |
| research | `PROJECT.md`, `profile.md` | `research/<dim>.md` (per-agent), `research/SUMMARY.md` (synthesizer), `log.md` | N parallel researchers + 1 synthesizer |
| architecture | `PROJECT.md`, `profile.md`, `research/SUMMARY.md` | `architecture.md`, `log.md` | One architect agent |
| design | `PROJECT.md`, `profile.md`, `research/SUMMARY.md`, `architecture.md` | `design/*` (format per profile), `log.md` | One designer agent (multiple iterations until user approves); profile.md decides if design comes before or after architecture |
| stories | `PROJECT.md`, `architecture.md`, `design/*` | `stories/*.md`, `stories/INDEX.md`, `log.md` | One story-extractor agent |
| prioritization | `stories/INDEX.md`, `architecture.md` | `stories/INDEX.md` (rewritten with priorities + dependencies), `log.md` | One prioritizer agent |
| iteration_active | `stories/<current>.md`, `architecture.md`, relevant `design/*` | code (in src/, not .maud/), `log.md` | One iterator agent per story |
| iteration_verifying | code (current diff), `stories/<current>.md` | `verifications/<story>/<verifier>.md`, `log.md` | 4 parallel verifier agents |
| story_complete | verifier outputs | `stories/INDEX.md` (move story to Done), `log.md`; git merge to main per profile | Orchestrator action; story-level user gate |
| done | (terminal) | (none) | All stories Done; orchestrator presents summary and offers "add more stories?" loop |

### Concurrency model

Within a single `/maud` invocation:
- Orchestrator runs sequentially (it IS the main thread).
- Agents spawned in the same message run in parallel; agents in subsequent messages run sequentially with the orchestrator in between.
- **Two `/maud` invocations should never overlap** — the user's session is single-threaded by Claude Code design. State.json doesn't need locking because there's only one writer (the orchestrator).

---

## State Machine

### Phases (states)

```
                       ┌──────────────┐
                       │ uninitialized │  (no .maud/ yet)
                       └──────┬───────┘
                              │ /maud invoked
                              ▼
                       ┌──────────────┐
                       │     init     │  (gather idea, write PROJECT.md, choose profile)
                       └──────┬───────┘
                              │ user approves PROJECT.md  ← GATE
                              ▼
                       ┌──────────────┐
                       │  research_   │  (orchestrator proposes dimensions; user edits)
                       │  scoping     │
                       └──────┬───────┘
                              │ user approves dimensions  ← GATE
                              ▼
                       ┌──────────────┐
                       │  research_   │  (parallel agents writing research/<dim>.md)
                       │  running     │
                       └──────┬───────┘
                              │ all agents complete (auto)
                              ▼
                       ┌──────────────┐
                       │  research_   │  (synthesizer writes SUMMARY.md)
                       │  synthesizing│
                       └──────┬───────┘
                              │ synthesis presented
                              │ user approves SUMMARY     ← GATE
                              ▼
                ┌─────────────┴─────────────┐
                │  Branch on profile.md:    │
                │  arch-first OR design-first│
                └─────┬─────────────────┬───┘
                      │                 │
                      ▼                 ▼
              ┌──────────────┐  ┌──────────────┐
              │ architecture │  │   design     │  (one of these runs first
              └──────┬───────┘  └──────┬───────┘   based on profile)
                     │ user approves   │ user iterates
                     │     ← GATE      │ until approved ← GATE
                     ▼                 ▼
              ┌──────────────┐  ┌──────────────┐
              │   design     │  │ architecture │  (then the other)
              └──────┬───────┘  └──────┬───────┘
                     │ user approves   │ user approves
                     │     ← GATE      │     ← GATE
                     └────┬────────────┘
                          ▼
                   ┌──────────────┐
                   │   stories    │  (story-extractor writes stories/)
                   └──────┬───────┘
                          │ user approves story breakdown ← GATE
                          ▼
                   ┌──────────────┐
                   │prioritization│  (prioritizer rewrites INDEX.md)
                   └──────┬───────┘
                          │ user approves priority ← GATE
                          ▼
                   ┌──────────────┐
                   │  iteration_  │ ←─────────┐
                   │   selecting  │           │
                   └──────┬───────┘           │
                          │ user picks story  │
                          │ (or Maud picks    │
                          │  top-priority)    │
                          │      ← GATE       │
                          ▼                   │
                   ┌──────────────┐           │
                   │  iteration_  │           │
                   │    active    │           │
                   └──────┬───────┘           │
                          │ iterator finishes │
                          │ (auto)            │
                          ▼                   │
                   ┌──────────────┐           │
                   │  iteration_  │           │
                   │   verifying  │           │
                   └──────┬───────┘           │
                          │ all 4 verifiers   │
                          │ pass / user       │
                          │ approves    ← GATE│
                          ▼                   │
                   ┌──────────────┐           │
                   │    story_    │           │
                   │   complete   │           │
                   └──────┬───────┘           │
                          │                   │
                          ├── stories left? ──┘
                          │
                          │ all stories Done
                          ▼
                   ┌──────────────┐
                   │     done     │  (offer: add more stories? close out?)
                   └──────────────┘
```

### Verification gates (the `← GATE` markers above)

Every gate uses `AskUserQuestion` with at least these three options:
1. **Approve** — advance phase
2. **Adjust** — re-run current phase with user feedback
3. **Stop** — exit cleanly; resume here on next `/maud`

Some gates have a fourth option specific to the phase (e.g. design has "iterate again on a specific artifact"). The orchestrator MUST handle every option explicitly — no falling through.

### Audibles (out-of-band transitions)

PROJECT.md says: "User can change priority, add stories, or remove stories at any time" and "User can call audibles at any point during user-facing interaction." This means the state machine has out-of-band transitions FROM most phases:

- From any phase: `/maud add-story` (TBD whether this is a separate command, a `--flag` on `/maud`, or just a free-form user input the orchestrator parses)
- From iteration_selecting: re-prioritize, add, remove
- From any architecture/design phase: revisit a previous artifact (returns to that phase with current state preserved)

**Recommendation for v1:** keep `/maud` as the single command. When the user types `/maud` and adds free-form text (e.g. `/maud reprioritize stories`), the orchestrator detects the audible intent and branches to the relevant phase rather than dispatching on state.json. This requires the orchestrator to parse `$ARGUMENTS` early. Detect a few keywords (reprioritize, add story, remove story, revisit design, change architecture); fall back to "what do you want?" if unclear.

### state.json schema (proposed)

```json
{
  "schema_version": 1,
  "created_at": "2026-05-08T13:00:00Z",
  "updated_at": "2026-05-08T15:30:00Z",
  "phase": "iteration_active",
  "phase_history": [
    { "phase": "init", "completed_at": "2026-05-08T13:10:00Z" },
    { "phase": "research_scoping", "completed_at": "2026-05-08T13:15:00Z" },
    { "phase": "research_running", "completed_at": "2026-05-08T13:25:00Z" },
    ...
  ],
  "current_story_id": "0003-add-login-form",
  "iteration_substate": "implementing",
  "iteration_substates_completed": ["jit_design", "test_writing"],
  "verifications_in_flight": [],
  "last_agent": "iterator",
  "last_agent_completed_at": "2026-05-08T15:25:00Z",
  "last_user_gate": {
    "phase": "iteration_active",
    "answer": "approve",
    "at": "2026-05-08T14:00:00Z"
  },
  "profile": "web-app",
  "config_ref": ".maud/config.json"
}
```

This is the **dispatcher input**. The orchestrator's first action is `jq -r '.phase'` to know where to go.

---

## Anti-Patterns

### Anti-Pattern 1: Putting orchestration logic in agents

**What people do:** Try to make `iterator.md` agent spawn its own verifier agents because "the iteration includes verification."

**Why it's wrong:** Subagents cannot spawn subagents (confirmed in [official docs](https://code.claude.com/docs/en/sub-agents): "Subagents cannot spawn other subagents"). The agent will fail or fall back to doing the work itself, defeating the parallelism.

**Do this instead:** The orchestrator (`commands/maud.md`) owns the iteration→verification handoff. The iterator agent finishes implementing + commits + returns; the orchestrator then spawns the four verifiers in parallel.

### Anti-Pattern 2: Hand-edited state.json

**What people do:** Document state.json schema in a way that invites users to edit it directly.

**Why it's wrong:** The orchestrator's invariants depend on state.json being internally consistent. A user who edits `phase` without also updating `current_story_id` breaks the dispatcher.

**Do this instead:** state.json is OWNED BY THE ORCHESTRATOR. Document this. Provide an audible-style command for state edits (e.g. `/maud reset` to reset to last gate) rather than expecting users to edit by hand. PROJECT.md is the user-editable artifact.

### Anti-Pattern 3: Skipping the verification gate when it "feels obvious"

**What people do:** Hard-code an "auto-advance if all verifiers pass" path to reduce friction.

**Why it's wrong:** This is GSD's failure mode. The user MUST explicitly approve every transition; that's the entire product thesis. The friction IS the value.

**Do this instead:** Make the gate fast (one keypress in `AskUserQuestion`) but never optional. Display a rich summary at each gate so the keypress is informed.

### Anti-Pattern 4: Per-phase command proliferation

**What people do:** Add `/maud:research`, `/maud:design`, `/maud:next-story` as the project grows.

**Why it's wrong:** This is GSD's mistake — 28 commands and growing cognitive load. The whole point of the single-command model is "one command, state decides."

**Do this instead:** When a new operation is needed, add it as: (a) a sub-state the orchestrator handles, or (b) an audible (free-form `/maud <intent>`). Don't add new top-level commands.

### Anti-Pattern 5: Synchronous verifier serialization

**What people do:** Run security verifier, then code-review, then test-coverage, then QA.

**Why it's wrong:** They're independent. Serializing turns 4×60s = 240s into 60s with no quality benefit.

**Do this instead:** Spawn all four `Agent()` calls in a single message. They run in parallel. Each writes to its own file in `verifications/<story>/`. Orchestrator reads all four after they all complete.

### Anti-Pattern 6: Storing prompts inside state.json or PROJECT.md

**What people do:** "Memo-ize" agent prompts in state.json so the orchestrator doesn't have to re-build them.

**Why it's wrong:** Prompts mutate as Maud evolves; state.json should be ephemeral data. Conflating them means upgrading Maud breaks old projects.

**Do this instead:** Prompts live in `agents/*.md` and `templates/*.md` in the plugin directory. The orchestrator builds prompts fresh from current versions. State.json holds only data.

---

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Git | Bash commands from orchestrator | Mandatory. Every gate-approved transition is a commit. Stories may have a branch each. |
| Context7 / WebFetch / WebSearch | Researcher agents call directly via their `tools` field | Standard Claude Code agent tool grants. |
| External MCP servers | `mcpServers` in agent frontmatter when relevant (e.g. for browser-based design or QA verifier) | Per-agent scoping per [docs](https://code.claude.com/docs/en/sub-agents#scope-mcp-servers-to-a-subagent). |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| Orchestrator ↔ agents | Inlined-prompt context in, structured-marker text + on-disk files out | Confirmed pattern in GSD |
| Orchestrator ↔ user | `AskUserQuestion` for gates, freeform text I/O between | Gates are mandatory |
| Agents ↔ disk | Each agent has a designated swimlane (research/, design/, etc.); agents do NOT read other agents' files in the same phase (orchestrator synthesizes) | Prevents race conditions in parallel spawns |
| Agents ↔ log.md | Append-only contract; one entry per agent per invocation | Single shared file; entries atomic per agent |

---

## Suggested Build Order

The natural sequence respecting dependencies:

1. **Plugin scaffolding** — `.claude-plugin/plugin.json`, `commands/maud.md` skeleton, `agents/` empty, `references/` empty, `templates/` empty. Verify it loads with `claude --plugin-dir ./maud`.
   - *Dependency:* none
   - *Verifies:* the plugin format works at all

2. **State schema authority** — `references/state-schema.md`, `templates/state.json`, plus a tiny `commands/maud.md` that just reads/writes state.json and dispatches on phase. Hardcode 2 phases (`uninitialized` → `init`) and prove the dispatch works.
   - *Dependency:* (1)
   - *Verifies:* the dispatcher pattern works; resumability works
   - *Why first:* every other phase depends on the dispatcher being solid. If state.json is wrong-shaped after init, every later phase inherits the bug.

3. **Init phase (no agents yet)** — orchestrator-only: clarifying questions via `AskUserQuestion`, write PROJECT.md, choose profile, write profile.md, set state.json phase to `research_scoping`.
   - *Dependency:* (2)
   - *Verifies:* user-interaction loop, gate pattern

4. **Log infrastructure** — `references/log-format.md`, `templates/log-entry.md`, plus orchestrator code that appends "user approved init" entries. **Don't add agents until logging is solid** because agents depend on logging.
   - *Dependency:* (3)
   - *Verifies:* append-only audit trail

5. **First agent: researcher** — single agent that reads PROJECT.md + profile.md + a dimension, writes `research/<dim>.md`, appends log, returns marker. Test with one dimension first.
   - *Dependency:* (4)
   - *Verifies:* `Agent()` spawn pattern, structured returns, log contract

6. **Parallel research orchestration** — orchestrator spawns N researchers in one message. Verify all complete and write distinct files.
   - *Dependency:* (5)
   - *Verifies:* parallel spawn pattern (load-bearing for verifiers later)

7. **Research synthesizer agent** — sequential after researchers; reads all `research/*.md`, writes `SUMMARY.md`.
   - *Dependency:* (6)
   - *Verifies:* sequential agent chain

8. **Research → architecture → design phases** — each adds an agent and a gate. Profile.md decides architecture-or-design-first; orchestrator branches.
   - *Dependency:* (7)
   - *Verifies:* profile-driven branching, multi-artifact phases

9. **Story extraction + INDEX.md format** — story-extractor agent + the `stories/INDEX.md` priority/status format. Hand-test with a known design+architecture.
   - *Dependency:* (8)
   - *Verifies:* the unit-of-work model works

10. **Prioritization phase** — prioritizer agent rewrites INDEX.md with priority order + dependency graph.
    - *Dependency:* (9)

11. **Iteration loop (single story, no verifiers)** — iterator agent picks one story, runs the iteration loop (JIT design optional, write tests, implement, commit). Manually skip verification at this stage.
    - *Dependency:* (10)
    - *Verifies:* code-modifying agents, git integration

12. **Parallel verifier suite** — 4 verifier agents (security, code-review, test-coverage, qa). Spawn in parallel. Orchestrator reads all 4 reports, presents synthesis.
    - *Dependency:* (11)
    - *Verifies:* the verification gate, the most complex parallel spawn in Maud

13. **Done phase + audibles** — terminal phase + the audible-handling pattern (`/maud reprioritize`, etc.).
    - *Dependency:* (12)

14. **Project-type profiles catalog** — multiple `templates/profiles/*.md` covering web-app, cli, game (at minimum). Test with each.
    - *Dependency:* (13)
    - *Why last:* you only know what a profile needs to encode after the rest works

This is dependency-honest. Steps 1–4 are infrastructure with no agents. Steps 5–7 prove the agent pattern works. Steps 8–12 add real Maud features. Step 14 generalizes.

**The minimum-viable Maud (could ship and use)**: steps 1–11 with hardcoded "web-app" profile. Verifiers (12) and profiles catalog (14) can be v1.x.

---

## Scaling Considerations

| Concern | At 1 project (now) | At 10 projects | At 100 projects (across users) |
|---------|--------------------|----|----|
| state.json size | ~5KB, fine | Still ~5KB per project | Same — local-only |
| log.md size | Append-only, ~100 entries per medium project, fine in one file | Still per-project | Per-project; consider `log/YYYY-MM.md` rotation if individual projects get huge |
| Number of stories | 10–50 typical, INDEX.md is fine | Same — Maud is single-project | If project sizes grow, add `stories/index/` with subindexes, but only if real |
| Plugin update path | `git pull` on plugin dir, `/reload-plugins` | Same | Marketplace versioning kicks in (post-v1 per PROJECT.md) |
| Agent prompt evolution | Orchestrator builds fresh from current `agents/*.md` each invocation | Old projects pick up new prompts automatically — verify state.json schema_version compatibility | Need migration path if state.json schema changes — `schema_version` field exists for this reason |

**The only real scaling concern:** state.json schema evolution. Bake `schema_version` in from day one (proposed schema does). Provide an in-orchestrator migration when bumping it.

---

## Comparison with GSD Architecture

This is the explicit GSD-vs-Maud comparison the quality gate requires.

| Concern | GSD | Maud | Why Maud differs |
|---------|-----|------|------------------|
| **Number of commands** | 28 in `~/.claude/commands/gsd/` | 1 (`/maud:maud`) | Single stateful command per PROJECT.md; reduce cognitive load |
| **State location** | `.planning/STATE.md` (markdown digest, hand-curated by orchestrator) | `.maud/state.json` (machine-canonical) | Maud's dispatcher needs reliable parsing; markdown digest is for humans, JSON is for code |
| **Entry artifact** | `PROJECT.md` (still markdown, frozen after init) | `PROJECT.md` (same — no change here) | The user-facing initial-spec model is good; keep it |
| **Workflow shape** | Phases (sequential, fixed-count, planned upfront in ROADMAP.md) | Stories (flat priority list, audibles welcome, no phases as units) | PROJECT.md explicit: "stories as unit (flat list, no phases)" |
| **Research model** | 4 fixed dimensions (STACK, FEATURES, ARCHITECTURE, PITFALLS) | N project-type-aware dimensions (chosen during init from profile) | PROJECT.md: "Maud proposes research dimensions appropriate for this project type" |
| **Architecture artifact** | `.planning/research/ARCHITECTURE.md` (a research output) AND ROADMAP-derived structure | `.maud/architecture.md` (single canonical artifact, separate from research) | PROJECT.md: "Architecture is its own artifact" |
| **Design phase** | Absent | First-class phase with profile-dictated artifact format | PROJECT.md: design is a core differentiator from GSD |
| **Verification model** | Optional verifier agent (config flag), runs after phase | Mandatory 4-verifier parallel suite per story + user manual gate | PROJECT.md: "All verification steps are automated agentic passes" + "user does the final manual verification" |
| **User gates** | Limited: only escalations + roadmap approval. Most work auto-advances. | EVERY phase boundary has an explicit gate. | This IS Maud's value. PROJECT.md: "Explicit user verification at every phase boundary." |
| **Audibles** | Limited support; phases are mostly fixed once roadmap is approved | First-class: re-prioritize, add/remove stories, revisit design, all from `/maud` | PROJECT.md: "Audibles welcome" |
| **Audit log** | None (decisions tracked in PROJECT.md table; no per-agent audit trail) | `log.md` mandatory append-only — every agent writes one entry | PROJECT.md: ".maud/log.md as agent audit trail" |
| **Phase directories** | `.planning/phases/01-foo/` with PLAN.md + SUMMARY.md + VERIFICATION.md per phase | None — stories are flat files, no per-phase scratch space | Phases aren't units in Maud |
| **Templates location** | `~/.claude/get-shit-done/templates/` | `<plugin-root>/templates/` | Same pattern, different name; well-validated |
| **References location** | `~/.claude/get-shit-done/references/` | `<plugin-root>/references/` | Same pattern; well-validated |
| **Profile/project-type awareness** | None (one shape fits all) | Central — `profile.md` per project drives research dims, design format, iteration shape, verifier criteria | PROJECT.md: project-type-aware structure |
| **Orchestrator location** | Distributed across `run-milestone.md`, `new-project.md`, `discuss-phase.md`, etc. | Concentrated in `commands/maud.md` (the only command) | Single command means single supervisor file |
| **Agent invocation location** | Always from a slash command (the supervisor pattern) | Same — confirmed by Claude Code's "subagents can't spawn subagents" rule | This is platform-imposed, not a design choice |

### What we keep from GSD (validated by use)

- The Pattern 2 (Pre-Assembled Context Injection) — read into bash variables, inline into agent prompts, never use `@` in `Task` prompts. **GSD's `run-milestone.md` line 196-201 confirms this is required**, not stylistic.
- Structured-return-marker pattern (`## RESEARCH COMPLETE`, etc.).
- Templates directory with file templates.
- References directory with reusable prompt fragments.
- Per-agent model resolution from a model_profile config (`quality | balanced | budget`).
- Git-as-audit-history (every advance is a commit).

### What we drop from GSD (failure modes)

- Multi-command surface area.
- Roadmap-of-phases planning.
- Per-phase mini-loops (research-plan-execute repeated per phase).
- Auto-advance through phases (Maud's gates are mandatory).
- Hand-curated STATE.md (replaced by machine-canonical state.json).

### What we add (genuinely new)

- Project profiles (project-type awareness baked into the shape, not tacked on).
- Append-only `log.md` audit trail with mandatory agent contract.
- Mandatory parallel verifier suite per story.
- First-class design phase.
- Story-as-unit + INDEX.md priority list (Trello model).

---

## Sources

### HIGH confidence (Claude Code official docs, code I read)

- [Claude Code: Create plugins](https://code.claude.com/docs/en/plugins) — plugin manifest format, directory structure (`.claude-plugin/plugin.json` only in `.claude-plugin/`; `commands/`, `agents/`, etc. at plugin root), namespacing rules.
- [Claude Code: Create custom subagents](https://code.claude.com/docs/en/sub-agents) — agent file format, frontmatter fields, **the constraint that subagents cannot spawn other subagents** (confirmed: "Subagents cannot spawn other subagents. If your workflow requires nested delegation, use Skills or chain subagents from the main conversation"), tool restriction patterns.
- GSD source code I read directly:
  - `/Users/ryan/.claude/commands/gsd/run-milestone.md` — confirms the supervisor-loop-in-command pattern; explicit constraint at line 22-24: "the supervisor loop runs here — in this slash command's main conversation context. It CANNOT be delegated to a spawned agent because subagents cannot spawn their own subagents (Task tool limitation)."
  - `/Users/ryan/.claude/commands/gsd/new-project.md` — concrete example of init flow with parallel `Task()` calls in single message.
  - `/Users/ryan/.claude/agents/gsd-project-researcher.md` — agent definition format and structured-return-marker pattern.
  - `/Users/ryan/.claude/get-shit-done/templates/state.md` — STATE.md template (informs why we go JSON instead).
  - `/Users/ryan/.claude/get-shit-done/templates/research-project/*.md` — research artifact patterns.
- `/Users/ryan/Documents/programming/maud/.planning/PROJECT.md` — Maud's frozen initial spec, source of truth for product decisions.

### MEDIUM confidence (cross-verified web sources)

- [GitHub issue #15882: Plugin commands always namespaced](https://github.com/anthropics/claude-code/issues/15882) — confirms `/maud:maud` (not `/maud`) is the current invocation; this is a known limitation with an open feature request. **Flag as a pitfall in PITFALLS.md.**
- [Anthropic: Customize Claude Code with plugins](https://claude.com/blog/claude-code-plugins) — official announcement of plugin system, marketplace flow.
- [GitHub: ComposioHQ/awesome-claude-plugins](https://github.com/ComposioHQ/awesome-claude-plugins) — surveys real-world plugin examples (architecture is consistent with what's in the docs).

### LOW confidence (single source / community blog, flagged for validation)

- [Designbeep: Claude Code Agent Teams](https://designbeep.com/2026/05/02/claude-code-agent-teams-how-to-orchestrate-ai-subagents-for-real-development-work/) — community write-up of orchestrator patterns; consistent with my reading but not authoritative.
- [MindStudio: Claude Code Parallel Agents](https://www.mindstudio.ai/blog/claude-code-agent-teams-parallel-agents) — community discussion of parallelism patterns; matches GSD's actual code.

---

## Open questions / gaps to validate

1. **`/maud:maud` vs `/maud` invocation** — verify whether the user finds `/maud:maud` acceptable or whether this is a v1 blocker. May need workaround (alias? secondary command?). PITFALLS.md should flag this.
2. **Audible parsing** — the proposal of "user types `/maud reprioritize`" assumes the orchestrator can robustly detect intent from `$ARGUMENTS`. Test with concrete user phrasings during build step 13.
3. **Concurrent log.md appends from parallel agents** — the recommendation is "one shell append per agent." Verify this is atomic in practice on macOS/Linux (it should be for small writes < PIPE_BUF) but if Windows users matter for v1, test.
4. **Profile catalog completeness** — only know what's in PROJECT.md (web app, iOS, CLI, game). Step 14 should not try to be exhaustive; build the profile interface so new profiles can be added without code changes.
5. **Story-branch-per-story vs commit-per-checklist-item** — PROJECT.md leaves this open ("branch per story (or commit per checklist item where appropriate)"). Iteration profile should encode the choice. Build step 11 should pick a default and let profile override.

---

*Architecture research for: Claude Code plugin (stateful project-management orchestrator)*
*Researched: 2026-05-08*
