# Phase 1: Fix `.maud/` Markdown Links - Context

**Gathered:** 2026-05-10
**Status:** Ready for planning

<domain>
## Phase Boundary

Repair broken markdown links in `.maud/` and apply one consistent link convention across all link instances. Includes the cascading file/text-reference work that follows from the chosen convention. Does NOT include adding link-checking tooling, rewriting unrelated prose, or expanding the convention to other parts of the repo beyond what's necessary to keep references coherent.

</domain>

<decisions>
## Implementation Decisions

### Index file convention — switch from `index.md` to `README.md`
- Every `index.md` file inside `.maud/` is renamed to `README.md`
- This is the chosen workaround for the "directory link" problem: with `README.md`, links like `[Section](folder/)` render correctly on GitHub web (GitHub auto-renders `README.md` for folder browsing, mimicking the `index.html` behavior the user was hoping for)
- Rationale captured: there is no universal `index.md` auto-render — only static-site generators (Jekyll/Hugo/MkDocs/Pages) treat `index.md` like `index.html`. GitHub web, Obsidian, VS Code preview, and raw filesystem do not. `README.md` is the only filename GitHub auto-renders for raw-repo browsing today.

### Scope of the rename — full cascade
- Rename actual files: every `.maud/**/index.md` → `.maud/**/README.md`
- Update prose references inside `.maud/`: any sentence mentioning `index.md` (e.g., "writes `.maud/index.md`", "the plan lives in `planning/index.md`", "stories listed in `index.md`") becomes `README.md`
- Update references **outside** `.maud/` too: `.planning/ROADMAP.md`, `.planning/REQUIREMENTS.md`, and `.planning/PROJECT.md` all reference `.maud/index.md`, `.maud/planning/index.md`, `.maud/stories/index.md`, `.maud/stories/backlog/build-command/index.md`, etc. All of these get updated. Reason: leaving them stale would immediately break the GSD planning docs' coherence with the actual file tree.
- This is technically slightly outside the strict phase boundary ("links in `.maud/`"), but the cost is small and the coherence benefit is large.

### Broken link repair
- Fix the typo `[Carve up remainder of old GSD for scraps](/complete/scrap-gsd.md]` in `.maud/stories/index.md` — `]` should be `)`
- After the rename, this link target also becomes `complete/scrap-gsd.md` (the scrap file itself is not an index, so it stays a regular `.md`)
- All 19 existing link targets must resolve after the work is done; verify by walking each link

### Claude's Discretion
- **Path style** for non-directory links: Claude picks. Default = standard relative paths (e.g., `../complete/scrap-gsd.md` from `.maud/stories/README.md`). Renders cleanly in GitHub, Obsidian, VS Code, and raw filesystem without configuration. Avoid the existing "fake root-relative" form (`/competitors/`, `/complete/scrap-gsd.md`) — leading-slash paths don't resolve in any standard renderer.
- **Verification method** for "all links resolve" — Claude picks. Likely a one-shot link walk (script or manual checklist) since this is a one-time cleanup, not an ongoing concern. No CI / lint rule needed for this phase.
- **Primary renderer assumption** — GitHub web (since the README.md choice is optimized for it). Claude can assume GitHub web is the source-of-truth renderer.

</decisions>

<specifics>
## Specific Ideas

- The user explicitly wanted the `(folder/)` style to "just work" the way `index.html` does in web. The `README.md` rename is the only way to get that behavior on GitHub web without a static-site pipeline.
- The user is aware this rename has downstream consequences for Phases 4, 7, 9 (where the plugin is supposed to "write `.maud/index.md`", etc.). After this phase, those phase descriptions should read `README.md` instead — the rename in `.planning/ROADMAP.md` and `.planning/REQUIREMENTS.md` makes that automatic.
- Existing link patterns to be repaired (sample): `(/competitors/)`, `(/backlog/discuss-maud-with-gsd.md)`, `(/complete/scrap-gsd.md]`, `(generate-design.md)`. Mix of "fake root-relative" and plain relative.

</specifics>

<deferred>
## Deferred Ideas

- **Automated link checking / CI gate** — would prevent future link rot, but out of scope for a one-time cleanup phase. Add to backlog if link rot becomes a recurring problem.
- **Wikilink-style links (`[[Foo]]`)** — Obsidian-flavored, considered briefly during gray-area enumeration. Not chosen; standard markdown links keep portability.
- **Renaming `index.md` outside `.maud/`** (e.g., in `.planning/`) — out of scope. Only `.maud/` adopts the `README.md` convention.

</deferred>

---

*Phase: 01-fix-maud-markdown-links*
*Context gathered: 2026-05-10*
