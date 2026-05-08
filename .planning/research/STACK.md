# Stack Research — Maud

**Domain:** Claude Code plugin (markdown + JSON; orchestrator + sub-agents; project-local state)
**Researched:** 2026-05-08
**Confidence:** HIGH for plugin format, slash command, agent, hook, and skill schemas (verified against https://code.claude.com/docs/en/plugins-reference, /plugins, /skills, /sub-agents, /hooks, /plugin-marketplaces). MEDIUM for skill-vs-command tradeoff because the convergence ("custom commands have been merged into skills") is recent enough that the official `plugin-dev` plugin still treats them as separate.

## TL;DR

Maud is built from **markdown files + JSON manifests + (optionally) bash scripts**. There is no compiled code, no language runtime to choose, no "framework" decision. The "stack" is entirely Claude Code's plugin format. Choices that matter:

1. **Plugin format** (`.claude-plugin/plugin.json` manifest + auto-discovered `commands/`, `agents/`, `skills/`, `hooks/`, `.mcp.json`).
2. **Authoring layer for the user-facing entrypoint**: a slash command at `commands/maud.md`. (NOT a skill, despite the official 2026 convergence — see "Recommended Stack" rationale.)
3. **Sub-agent layer**: per-task `.md` files in `agents/`, dispatched by the main `/maud` command via the `Agent` tool (formerly `Task`).
4. **Project state layer**: a `.maud/` directory in the user's working dir, written via the standard `Read`/`Write`/`Edit`/`Bash` tools. **No special API.** State is just files Claude reads and writes.
5. **Local development**: `claude --plugin-dir ./maud` (no marketplace publish needed). `/reload-plugins` picks up edits without restart.
6. **Future marketplace publish**: ship a sibling repo (or same repo with `.claude-plugin/marketplace.json` at root + `plugins/maud/`) — incremental, no rewrite required.

## Recommended Stack

### Core "Technologies" (file formats)

| Component | Format | Where | Why |
|-----------|--------|-------|-----|
| Plugin manifest | JSON | `.claude-plugin/plugin.json` | Required by Claude Code to recognize the directory as a plugin. Only `name` is technically required; everything else is metadata. |
| Slash command (entrypoint) | Markdown w/ YAML frontmatter | `commands/maud.md` | Single user-facing trigger `/maud:maud` or `/maud` (depending on namespacing). Slash commands are still the right primitive for stateful, user-driven workflows even though "custom commands have been merged into skills" in 2026. See rationale below. |
| Sub-agents | Markdown w/ YAML frontmatter | `agents/<name>.md` | Each agent runs in its own context window with its own tools/model. The orchestrator dispatches them via the `Agent` tool, getting only the summary back. This is exactly GSD's model and exactly what Maud needs for research/verifier/executor agents. |
| Hooks (optional, v1) | JSON | `hooks/hooks.json` | Useful for `SessionStart` (preload `.maud/STATE.md` if present) and `Stop` (audit-trail append). Skip in v1 if `/maud` itself can do the same work — keeps surface area small. |
| MCP servers | JSON | `.mcp.json` | **Not needed for v1.** Maud's tools are all built-in (Read/Write/Bash/Edit/Grep/Glob/Agent/AskUserQuestion). |
| Templates / references | Markdown | `templates/`, `references/` (any plugin-rooted name) | Loaded into commands/agents via `@${CLAUDE_PLUGIN_ROOT}/templates/foo.md`. Mirror GSD's `templates/` and `references/` directories. |
| Project state (per-user) | Markdown + JSON | `.maud/` in user's CWD | Authored by Claude using standard tools. `.maud/log.md` for audit trail. Mirrors GSD's `.planning/`. |

### Plugin Manifest — `.claude-plugin/plugin.json`

**v1 minimal manifest** (HIGH confidence — verified against https://code.claude.com/docs/en/plugins-reference):

```json
{
  "$schema": "https://json.schemastore.org/claude-code-plugin-manifest.json",
  "name": "maud",
  "version": "0.1.0",
  "description": "Walks you from idea to shipped software using PM techniques: research and architecture upfront, then iterate on prioritized stories.",
  "author": {
    "name": "Ryan MacInnes",
    "email": "ryan.macinnes@gmail.com"
  },
  "keywords": ["project-management", "planning", "workflow", "agents"],
  "license": "MIT"
}
```

That is the entire required manifest. Auto-discovery handles the rest (`commands/`, `agents/`, `skills/`, `hooks/hooks.json`, `.mcp.json` are scanned automatically). Specify custom paths only if you want to override defaults.

**Critical rules (HIGH confidence — official docs explicitly call this out as the most common mistake):**
- `plugin.json` MUST live in `.claude-plugin/` subdirectory.
- `commands/`, `agents/`, `skills/`, `hooks/` MUST be at plugin root, NOT inside `.claude-plugin/`.
- Use `${CLAUDE_PLUGIN_ROOT}` for any path inside the plugin (commands, hooks, agents — never hardcode).
- Use `${CLAUDE_PLUGIN_DATA}` for plugin state that should survive plugin updates (NOT the user's project state — that's `.maud/` in CWD).

### Slash Command — `commands/maud.md`

**HIGH confidence — schema verified against https://code.claude.com/docs/en/slash-commands and frontmatter-reference.md in plugin-dev:**

```markdown
---
description: Resume or start your project. Stateful — reads .maud/ to know where you are.
argument-hint: [optional-subcommand]
allowed-tools: Read, Write, Edit, Bash, Grep, Glob, Agent, AskUserQuestion
model: sonnet
---

<context>
@${CLAUDE_PLUGIN_ROOT}/templates/state.md
@${CLAUDE_PLUGIN_ROOT}/references/workflow.md
</context>

<process>
1. Check if `.maud/` exists. If not → onboarding flow.
2. Read `.maud/STATE.md` to determine resume point.
3. Dispatch to sub-agents based on phase (research / design / iterate / verify).
...
</process>
```

**Frontmatter fields used (all official, HIGH confidence):**
- `description` — shown in `/help`. ≤60 chars.
- `argument-hint` — autocomplete hint (e.g. `[optional-subcommand]`).
- `allowed-tools` — comma-separated or YAML list. Use the `Bash(git:*)` filter pattern when you only need git, etc. The tool **`Task` was renamed to `Agent` in v2.1.63** — old `Task(...)` still works as alias, but use `Agent` in new code.
- `model` — `sonnet` (default-grade), `opus` (deep reasoning), `haiku` (cheap), or `inherit`. For a stateful orchestrator that will spawn opus agents for hard things and haiku agents for cheap things, `sonnet` is the right default.
- Optional: `disable-model-invocation: true` if you want `/maud` to be user-only (not callable by `SlashCommand` programmatically). Probably **not** what Maud wants — being model-invocable lets parent sessions/hooks resume Maud automatically.

**Dynamic content syntax:**
- `$ARGUMENTS` — full arg string. `$1`, `$2`, … — positional. `$ARGUMENTS[N]` — same as `$N`.
- `@path/to/file` — file is read into context before the prompt runs (use `@${CLAUDE_PLUGIN_ROOT}/...` for plugin files; bare `@.maud/STATE.md` for user-project files).
- ``!`bash command` `` — output is captured into context before the prompt runs (preprocessing, not Claude execution). Multi-line: ```` ```! …code… ``` ````.

### Slash command vs. Skill — IMPORTANT 2026 nuance

The 2026 official docs say "Custom commands have been merged into skills." `commands/foo.md` and `skills/foo/SKILL.md` both create `/foo`. Both are auto-discovered. Both work. But:

- **Skills** are designed to be **autonomously invoked by Claude when relevant** (description-driven). They're great for "knowledge that Claude should pull in when topic X comes up."
- **Commands** are designed to be **user-triggered explicit workflows**. The `disable-model-invocation: true` field on a skill effectively makes it command-like.
- A skill's body **stays in context for the rest of the session** once invoked. A command's body is processed once.

**Recommendation for Maud (MEDIUM confidence — leaning on the docs' "task content" guidance):** Use `commands/maud.md` for the entrypoint. `/maud` is an explicit user trigger with side effects (writes to `.maud/`, dispatches agents, mutates project state); you don't want Claude auto-invoking it because the conversation drifted near "planning". You can still set `disable-model-invocation: false` so other commands can chain to `/maud`. If you later add reference material ("how Maud thinks about story sizing"), put **that** in a `skills/` folder so it's auto-loaded when relevant.

Concrete v1 layout:
- `commands/maud.md` — the orchestrator (user-invokable; `disable-model-invocation` left at default false).
- `commands/maud-status.md`, `commands/maud-pause.md`, `commands/maud-debug.md` — secondary user-facing commands (mirror GSD's `progress`, `pause-work`, `debug`).
- `skills/` — empty in v1. Add later for reference/onboarding content.

### Sub-Agents — `agents/<name>.md`

**HIGH confidence — schema verified against https://code.claude.com/docs/en/sub-agents and feature-dev plugin's `agents/code-architect.md`:**

```markdown
---
name: maud-researcher
description: Researches the project domain ecosystem before roadmap creation. Spawned by /maud during the research phase.
tools: Read, Write, Bash, Grep, Glob, WebSearch, WebFetch
model: inherit
color: cyan
---

You are a Maud research agent. ...
```

**Frontmatter fields (only `name` and `description` are required; everything else optional):**
- `name` — kebab-case, 3-50 chars. Becomes `maud:maud-researcher` when called from the plugin.
- `description` — drives auto-dispatch. The plugin-dev skill recommends "Use this agent when [conditions]. Typical triggers include [...]" prose, but feature-dev's actual agents use a single concise sentence. Either works. **For an orchestrator like Maud where the *parent* command explicitly dispatches agents (not Claude's auto-routing), the description is mostly documentation** — keep it short and clear.
- `tools` — array or comma-separated. **Plugin agents do NOT support `hooks`, `mcpServers`, or `permissionMode` (security restriction).** If you need those, the agent must live in `~/.claude/agents/` instead of inside the plugin. Maud agents will not need them, so this is fine.
- `disallowedTools` — denylist, mutually composable with `tools` allowlist.
- `model` — `inherit` (use parent's model, recommended for most agents), `sonnet`, `opus`, `haiku`, or full ID like `claude-opus-4-7`.
- `color` — visual UI tag. Allowed values per official 2026 docs: `red`, `blue`, `green`, `yellow`, `purple`, `orange`, `pink`, `cyan`. (Note: plugin-dev's agent-development skill says only six colors. Trust the 2026 official docs.)
- `effort` — `low`/`medium`/`high`/`xhigh`/`max`. New in 2026.
- `maxTurns` — caps agentic loop length. Useful for verifiers.
- `memory` — `user`/`project`/`local` for persistent cross-session memory directory. **Not for Maud v1** (state belongs in `.maud/`, not in agent memory).
- `isolation: worktree` — runs the agent in a temporary git worktree. Excellent fit for verifier agents that should not touch the working tree mid-iteration.

**How agent dispatch works inside the plugin (HIGH confidence):**
The orchestrator command says, in prose: "Use the `maud-researcher` agent to investigate the domain." Claude sees the available agents (loaded via `description`) and uses the `Agent` tool to spawn one. The agent runs in its own context, returns a summary. Claude's main thread continues from there. This is exactly GSD's pattern — see `/Users/ryan/.claude/agents/gsd-project-researcher.md` for a working reference (note GSD uses `tools: Read, Write, Bash, Grep, Glob, WebSearch, WebFetch, mcp__context7__*` — wildcards in the tool list are valid).

### Hooks — `hooks/hooks.json` (optional in v1)

**HIGH confidence — 28+ events listed in official 2026 hooks docs.** For Maud v1, only two are likely useful:

```json
{
  "description": "Maud audit trail and resume",
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume",
        "hooks": [{
          "type": "command",
          "command": "bash ${CLAUDE_PLUGIN_ROOT}/hooks/scripts/notify-resume.sh",
          "timeout": 5
        }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Agent",
        "hooks": [{
          "type": "command",
          "command": "bash ${CLAUDE_PLUGIN_ROOT}/hooks/scripts/log-agent.sh",
          "timeout": 5
        }]
      }
    ]
  }
}
```

**Hook types (HIGH confidence):** `command` (bash script), `http` (POST to URL), `mcp_tool` (call MCP tool), `prompt` (LLM evaluates), `agent` (full subagent verifier). For `.maud/log.md` audit-trail Maud only needs `command`.

**Cut from v1**: hooks add complexity, restart-required-to-reload (HIGH confidence — hooks load at session start; `/reload-plugins` does pick up changes per the most recent docs but the older skill says restart). If `/maud` itself can append to `.maud/log.md` whenever it dispatches an agent, you avoid the hook entirely. Add hooks in v2 if you need the audit trail to capture work outside `/maud` invocations.

### Settings — `settings.json` (plugin-level, optional)

**HIGH confidence — new in 2026.** A plugin-level `settings.json` at plugin root applies defaults when the plugin is enabled. Currently only `agent` and `subagentStatusLine` are honored. **Skip for v1.** Maud doesn't want to override the user's main agent.

### File layout — Recommended for Maud v1

```
maud/                                  # Repo root
├── .claude-plugin/
│   └── plugin.json                    # Manifest (name, version, description, author)
├── commands/
│   ├── maud.md                        # /maud — main orchestrator, single stateful entrypoint
│   ├── maud-status.md                 # /maud-status — show .maud/STATE.md
│   ├── maud-pause.md                  # /maud-pause — checkpoint and exit
│   └── maud-debug.md                  # /maud-debug — diagnostic dump
├── agents/
│   ├── maud-project-researcher.md     # Research domain ecosystem
│   ├── maud-roadmapper.md             # Build phase structure
│   ├── maud-planner.md                # Detailed plan for one phase
│   ├── maud-executor.md               # Run a plan with atomic commits
│   ├── maud-security-verifier.md      # Security review
│   ├── maud-code-reviewer.md          # Code review
│   ├── maud-test-coverage.md          # Test coverage check
│   └── maud-qa.md                     # End-user QA
├── templates/                          # Loaded via @${CLAUDE_PLUGIN_ROOT}/templates/...
│   ├── PROJECT.md
│   ├── ROADMAP.md
│   ├── STATE.md
│   ├── PHASE.md
│   └── log-entry.md
├── references/                         # Loaded via @${CLAUDE_PLUGIN_ROOT}/references/...
│   ├── workflow.md                     # The PM methodology Maud encodes
│   ├── questioning.md                  # How to ask clarifying questions well
│   └── verification.md                 # Verification patterns
├── README.md                           # User-facing install + usage doc
├── LICENSE
└── .gitignore
```

**Conventions:**
- `templates/` and `references/` are not auto-discovered by Claude Code — they're just plugin-internal directories that commands and agents load via `@${CLAUDE_PLUGIN_ROOT}/...` prefixes. This is exactly how GSD does it (`/Users/ryan/.claude/get-shit-done/templates/` and `/references/`), and the same pattern is used by the official `plugin-dev` plugin (in its `skills/*/references/` and `skills/*/examples/` directories).
- All agent names prefixed `maud-` to keep namespacing visually clean even though the plugin namespace already prefixes them as `maud:maud-foo`.
- `.maud/` is **NOT** in this tree — that's the user's project-state directory, created by `/maud` in the user's CWD on first run. Same way GSD creates `.planning/` in the user's project, not in `~/.claude/get-shit-done/`.

### How the plugin reads/writes the user's working directory

**HIGH confidence — there's no special "plugin filesystem API."** Plugin commands and agents call the standard tools the user has granted access to:

- `Read` to read `.maud/STATE.md`, `.maud/PROJECT.md`, etc.
- `Write` and `Edit` to update them.
- `Bash` to run `git`, `mkdir -p .maud`, etc.
- `Grep`/`Glob` to scan the user's codebase.

The plugin runs in the user's session with the user's tool permissions and the user's working directory. `${CLAUDE_PLUGIN_ROOT}` is for *plugin-internal* files (templates, references, scripts). **Bare relative paths in commands resolve to the user's CWD** — this is precisely how `/maud` reads `.maud/STATE.md` without doing anything special.

### Distribution — local install for v1

**HIGH confidence — official quickstart says exactly this:**

```bash
# Development:
claude --plugin-dir /path/to/maud

# Hot reload after edits:
/reload-plugins        # Reloads commands, skills, agents, hooks, MCP, LSP
```

That's it. No package manager, no install step, no marketplace required. You can develop the entire plugin and dogfood it on real projects without ever publishing.

**For the future v2 marketplace publish (HIGH confidence):**

Add `.claude-plugin/marketplace.json` at the **repo root** (separate from the plugin's `.claude-plugin/plugin.json`):

```json
{
  "$schema": "https://json.schemastore.org/claude-code-marketplace.json",
  "name": "maud-marketplace",
  "owner": { "name": "Ryan MacInnes", "email": "ryan.macinnes@gmail.com" },
  "plugins": [
    {
      "name": "maud",
      "source": "./",
      "description": "Walks you from idea to shipped software using PM techniques.",
      "category": "productivity"
    }
  ]
}
```

Then push to GitHub. Users install via `/plugin marketplace add ryan/maud` then `/plugin install maud@maud-marketplace`. **Reserved marketplace names** (cannot be used): `claude-code-plugins`, `claude-plugins-official`, `anthropic-marketplace`, etc. — `maud` and `maud-marketplace` are fine.

You can also submit to the official Anthropic marketplace via https://claude.ai/settings/plugins/submit when ready.

## Alternatives Considered

| Recommended | Alternative | When to use alternative |
|-------------|-------------|--------------------------|
| Slash command (`commands/maud.md`) for entrypoint | Skill (`skills/maud/SKILL.md`) | If you want Claude to *auto-invoke* Maud when the user says "let's plan a feature." But Maud has heavy side effects (creates `.maud/`, dispatches agents) — explicit invocation is safer. The skill body also persists in context for the rest of the session, which is bloat for a one-shot orchestrator. |
| Sub-agents in `agents/` (model-routed via `Agent` tool) | Inlining all logic in one giant `commands/maud.md` | Inlining keeps everything in the main thread's context window — fine for ~5 phases, broken at 20+. Sub-agents preserve context exactly because that's what they're for. |
| `commands/` for v1, `skills/` for v2 reference content | All-skills (modern 2026 idiom) | The convergence is real but new. `plugin-dev` itself still ships separate `commands/` and `skills/` directories. Following the working reference plugin is safer than chasing the bleeding edge. |
| Hooks deferred to v2 | Hooks for audit trail in v1 | Hooks need a Claude restart to reload (or `/reload-plugins`); they add complexity; `/maud` can append to `.maud/log.md` itself. Add when you need to capture *non-Maud* work. |
| No MCP server | Custom MCP server for state management | MCP servers are processes that need to start. Maud's "state" is plain markdown files; introducing a server is overkill until/unless you need cross-machine sync, which is out of scope for v1. |
| `${CLAUDE_PLUGIN_ROOT}` everywhere | Hardcoded paths during dev | Hardcoded paths break the moment you publish or another machine installs. There's literally no upside. |
| `.maud/` in user CWD | `${CLAUDE_PLUGIN_DATA}` for project state | `${CLAUDE_PLUGIN_DATA}` is per-user, plugin-global. It's for things like cached `node_modules`. **Project state must live in the project**, not in the plugin's data dir. |
| Bash + jq for any scripts | Python/Node | Bash + jq is what every plugin example uses (plugin-dev's `scripts/` are all `.sh`). No runtime to install. Drop down to Node only if you genuinely need it (you don't, for v1). |

## What NOT to Use

| Avoid | Why | Use instead |
|-------|-----|-------------|
| Putting `commands/`, `agents/`, `hooks/` inside `.claude-plugin/` | Official docs explicitly call this out as the most common mistake. Components must be at plugin root. | Plugin root: `maud/commands/`, `maud/agents/`, `maud/hooks/hooks.json`. Only `plugin.json` lives in `.claude-plugin/`. |
| Hardcoded paths (`/Users/ryan/...`, `~/.claude/...`) anywhere in plugin files | Will break on every other user's machine. | `${CLAUDE_PLUGIN_ROOT}/...` for plugin files; bare relative paths or `.maud/...` for user-project files. |
| Underscores in plugin/agent/command names | Schema validation rejects `my_plugin`. Naming pattern is `^[a-z][a-z0-9]*(-[a-z0-9]+)*$`. | Kebab-case: `maud-researcher`, not `maud_researcher`. |
| Writing Python/TypeScript/Go for plugin logic | Plugins are markdown + JSON + bash. Compiled code is for MCP servers (which Maud doesn't need). | Keep everything in markdown prompts + bash scripts via `Bash` tool. |
| `Task(...)` in 2026 code | Renamed to `Agent` in v2.1.63 (alias still works). | `Agent(...)` syntax. e.g. `tools: Agent(maud-researcher, maud-planner), Read, Bash`. |
| Reserved marketplace names | Will be blocked at install. | Avoid `claude-plugins-official`, `anthropic-marketplace`, etc. `maud-marketplace` is safe. |
| `version: "1.0"` (two-segment) | Schema requires semver MAJOR.MINOR.PATCH. | `version: "0.1.0"`. |
| Putting agent state in agent `memory` field | That's user/project memory, not project state. Wrong scope, wrong abstraction. | `.maud/STATE.md` and friends, written via Read/Write tools. |
| Skipping `${CLAUDE_PLUGIN_DATA}` for cached deps if/when you add them | Plugin updates rotate `${CLAUDE_PLUGIN_ROOT}`; `${CLAUDE_PLUGIN_DATA}` survives. | If you ever bundle `node_modules` or models, install into `${CLAUDE_PLUGIN_DATA}`. (Not needed for Maud v1.) |
| Inline `tools: "*"` in commands | Lazy and grants more than you need. Restrict to actual needs. | Explicit list. e.g. `Read, Write, Edit, Bash, Grep, Glob, Agent, AskUserQuestion`. |

## Stack Patterns by Variant

**If v1 (local-only, single user):**
- Use `commands/maud.md` + `agents/*.md` + `templates/` + `references/`.
- Skip hooks, skip MCP, skip skills, skip `.lsp.json`, skip monitors.
- Test with `claude --plugin-dir /path/to/maud`.

**If v2 (publish to marketplace):**
- Add repo-root `.claude-plugin/marketplace.json` (note: SEPARATE from `maud/.claude-plugin/plugin.json`).
- Decide on versioning: pin `version` in `plugin.json` for "stable" releases, OR omit it and let commit SHA be the version (every push = new version, useful while iterating).
- Keep marketplace and plugin in same repo: `marketplace.json` lists the plugin with `"source": "./"`.

**If you later need persistent plugin-side state (caches, ML models, etc.):**
- Use `${CLAUDE_PLUGIN_DATA}` (resolves to `~/.claude/plugins/data/{plugin-id}/`). Survives plugin updates.

**If you later need to capture work outside `/maud` (e.g. log every commit during a phase):**
- Add `hooks/hooks.json` with `PostToolUse` matchers on `Bash` (filter for `git commit`) and/or `Write|Edit`.

## Version Compatibility

| Component | Version note |
|-----------|--------------|
| Claude Code | Tested against 2026 docs (current at time of research). The `Task → Agent` rename happened in v2.1.63. The plugin-monitor feature requires v2.1.105+. Plugin-dev features are in current stable. |
| Plugin schema | `$schema: https://json.schemastore.org/claude-code-plugin-manifest.json` — Claude Code ignores this at load but it gives you editor autocomplete. Worth including. |
| Marketplace schema | `$schema: https://json.schemastore.org/claude-code-marketplace.json` — same deal. |

## Concrete Reference Files (already on disk)

These are working implementations to copy idioms from:

- `/Users/ryan/.claude/get-shit-done/` — GSD's full source. Note this is *not* a plugin (it's installed as `~/.claude/commands/gsd/*` + `~/.claude/agents/gsd-*`), but the command and agent files use the **same** schema as plugin commands/agents. Steal liberally.
- `/Users/ryan/.claude/commands/gsd/new-project.md` — concrete example of a stateful orchestrator command using `@absolute-path` includes (you'll convert these to `@${CLAUDE_PLUGIN_ROOT}/...` in Maud).
- `/Users/ryan/.claude/agents/gsd-project-researcher.md` — concrete example of a research sub-agent (the very file Maud is using to research itself).
- `/Users/ryan/.claude/plugins/marketplaces/claude-plugins-official/plugins/plugin-dev/` — the **official Anthropic plugin for building plugins**. Cached locally. Contains:
  - `.claude-plugin/plugin.json` — minimal real-world manifest.
  - `commands/create-plugin.md` — a real plugin command.
  - `agents/agent-creator.md`, `agents/skill-reviewer.md`, `agents/plugin-validator.md` — three real plugin agents.
  - `skills/plugin-structure/`, `skills/command-development/`, `skills/agent-development/`, `skills/hook-development/`, `skills/skill-development/`, `skills/mcp-integration/`, `skills/plugin-settings/` — seven canonical skills with `references/`, `examples/`, and `scripts/` subdirectories. **This is the gold-standard layout for a multi-component plugin.**
- `/Users/ryan/.claude/plugins/marketplaces/claude-plugins-official/plugins/feature-dev/` — a more orchestrator-shaped reference (single command + multiple agents pattern, very close to Maud's shape).
- `/Users/ryan/.claude/plugins/marketplaces/claude-plugins-official/plugins/example-plugin/` — minimal "everything" reference (manifest + 1 command + 2 skills + 1 MCP).
- `/Users/ryan/.claude/plugins/marketplaces/claude-plugins-official/.claude-plugin/marketplace.json` — the canonical 90KB marketplace.json. Skim the entries (`feature-dev`, `code-review`, `plugin-dev`) to see the marketplace-entry shape.

## Sources

- **HIGH confidence (official 2026 docs)**:
  - https://code.claude.com/docs/en/plugins — plugin authoring quickstart and structure
  - https://code.claude.com/docs/en/plugins-reference — full plugin.json schema, all directory locations, environment variables, version management
  - https://code.claude.com/docs/en/plugin-marketplaces — marketplace.json schema and source types
  - https://code.claude.com/docs/en/slash-commands (via skills page) — frontmatter fields, `$ARGUMENTS`, `@file`, `` !`bash` ``
  - https://code.claude.com/docs/en/sub-agents — agent frontmatter, `Task→Agent` rename, all 14+ frontmatter fields
  - https://code.claude.com/docs/en/skills — skill frontmatter (incl. `disable-model-invocation`, `user-invocable`, `paths`, `context: fork`, `${CLAUDE_SKILL_DIR}`), command-vs-skill convergence
  - https://code.claude.com/docs/en/hooks — full 28-event hook list, hook types, schemas
- **HIGH confidence (locally-cached authoritative content from official `plugin-dev` plugin)**:
  - `/Users/ryan/.claude/plugins/marketplaces/claude-plugins-official/plugins/plugin-dev/skills/plugin-structure/SKILL.md` and `references/manifest-reference.md`
  - `…/plugin-dev/skills/command-development/SKILL.md` and `references/frontmatter-reference.md` and `references/plugin-features-reference.md`
  - `…/plugin-dev/skills/agent-development/SKILL.md`
  - `…/plugin-dev/skills/hook-development/SKILL.md`
  - `…/plugin-dev/skills/skill-development/SKILL.md`
- **HIGH confidence (working reference plugins inspected)**:
  - `…/plugins/plugin-dev/.claude-plugin/plugin.json`
  - `…/plugins/feature-dev/agents/code-architect.md` and `commands/feature-dev.md`
  - `…/plugins/example-plugin/.mcp.json`
  - `…/.claude-plugin/marketplace.json`
- **HIGH confidence (working orchestrator-pattern reference, non-plugin)**:
  - `/Users/ryan/.claude/get-shit-done/` (GSD on disk, version 1.11.1)
  - `/Users/ryan/.claude/commands/gsd/*.md` and `/Users/ryan/.claude/agents/gsd-*.md`

---

## Confidence Assessment

| Area | Confidence | Reason |
|------|-----------|--------|
| Plugin manifest format (`plugin.json`) | HIGH | Verified against current 2026 official docs + locally-cached canonical examples in `claude-plugins-official` marketplace. |
| Slash command frontmatter | HIGH | Schema verified against official docs; example patterns verified against `feature-dev/commands/feature-dev.md` and GSD commands. Note: `Task→Agent` tool rename in v2.1.63 caught from current docs. |
| Sub-agent frontmatter | HIGH | Verified against current docs; minor discrepancy between plugin-dev's "Use this agent when…" prose template and feature-dev's terse single-line description, but both are valid — only `name` and `description` are required by schema. |
| Hook events and `hooks.json` schema | HIGH | 28 events enumerated in current docs; schema verified; the `prompt` and `agent` hook types are newer additions and well-documented. |
| Skills vs commands convergence | MEDIUM | The 2026 convergence ("custom commands have been merged into skills") is recent enough that the official `plugin-dev` plugin still teaches them as distinct. Maud should use `commands/` for v1 (matches reference plugins, matches GSD pattern, gives correct semantics). |
| Marketplace publish path | HIGH | Schema and walkthrough explicit in current docs; confirmed against the cached `claude-plugins-official` marketplace.json. Local-dev path (`--plugin-dir`) is the official documented dev workflow. |
| `.maud/` state mechanics | HIGH | Plugins read/write user CWD via standard tools — confirmed in plugin-features-reference and verified by GSD's `.planning/` working precedent. |
| Plugin-internal vs user-project paths | HIGH | `${CLAUDE_PLUGIN_ROOT}` semantics explicit in docs; `${CLAUDE_PLUGIN_DATA}` for plugin-side persistence; bare paths resolve to user CWD. |

## Open Questions

- **Skill-vs-command for sub-orchestrators**: If Maud later wants smaller, focused, user-trigger-able pieces (e.g. `/maud-quick-research [topic]`), should those be additional commands or skills with `disable-model-invocation: true`? Either works. Defer to phase-level research when it comes up.
- **`/reload-plugins` reliability for hooks**: Older skill content says "hooks require restart"; newer docs say `/reload-plugins` reloads hooks. Worth empirically confirming when Maud first ships hooks (probably v2).
- **Plugin agents lack `permissionMode`**: Plugin-shipped agents cannot set `permissionMode`, `hooks`, or `mcpServers` (security). If Maud wants a verifier agent that runs in `plan` (read-only) mode, it must rely on tool restrictions (`disallowedTools: Write, Edit`) rather than permission modes. Workable but worth flagging during agent design.
- **GSD's `gsd-codebase-mapper` writes files directly to reduce orchestrator context**: this is a great pattern — Maud's research agents should do the same (write to `.maud/research/*.md` rather than returning all findings inline). Confirmed possible (agents have `Write` tool); just a design decision.

## Roadmap Implications

This stack research suggests phase ordering:

1. **Phase: Plugin scaffold** — manifest + minimal `commands/maud.md` + `--plugin-dir` boot. Smallest possible plugin that loads and prints "hello." Validates dev loop.
2. **Phase: State layer** — `.maud/` directory creation, `STATE.md` schema, the resume/onboarding fork. No agents yet; just the orchestrator reading and writing files.
3. **Phase: First research agent** — port one sub-agent (likely `maud-project-researcher`, mirroring GSD's). Validates the dispatch-and-summarize pattern.
4. **Phase: Roadmap + plan agents** — add `maud-roadmapper`, `maud-planner`. Now `/maud` can take a project from idea to plan.
5. **Phase: Execution + verification agents** — `maud-executor` + the four verifiers (`security`, `code-review`, `test-coverage`, `qa`).
6. **Phase: Templates + references library** — extract patterns into `templates/` and `references/`, mirror GSD's structure.
7. **Phase: Marketplace publish prep** — `marketplace.json`, README, LICENSE, version pin. Defer past v1 if not needed.
