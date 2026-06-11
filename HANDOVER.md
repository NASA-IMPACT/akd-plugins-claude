# IESO Worldview — Claude Code plugin handover

A pick-up-cold reference for continuing work on the IESO Worldview
Claude Code plugin. The plugin lets a Claude Code user explore NASA
Worldview in natural language: discover satellite layers, generate
reproducible permalinks, and iterate on the live map. It bundles
nine MCP servers (CMR, IESO validation, GeoUI, Worldview permalink,
UMM-Vis lookup, Earthdata-search, PDF parser, SDE, EONET) plus
Playwright (browser automation), wired to a single skill that
defines the multi-step assistant behavior.

The agent **runs on the user's own Claude Code subscription** — no
separate LLM API key. The original Python agent that this was
derived from (built on AKD's `PydanticAIBaseAgent`) stays behind in
the experiment worktree; the plugin is fully declarative.

## Status (last verified 2026-06-11)

- Repo published: <https://github.com/sanzog03/claude-skills-and-plugins>
- Working install command (one-liner, no clone needed):
  ```
  /plugin marketplace add sanzog03/claude-skills-and-plugins
  /plugin install akd-plugins@akd-plugins
  ```
- Confirmed working: skill loads as `/akd-plugins:worldview-assistant`;
  CMR + Playwright MCPs connect; tool calls execute end-to-end
  (CMR → UMM-Vis → GeoUI render → browser_navigate); Chromium opens
  a Worldview window and the map renders.

## Architecture (final)

```
claude-skills-and-plugins/        ← single-plugin repo (currently)
├── .claude-plugin/
│   ├── plugin.json               manifest + userConfig token prompts
│   └── marketplace.json          GitHub source for distribution
├── .mcp.json                     9 HTTP MCPs + 1 stdio (Playwright)
├── README.md                     user-facing install + usage
├── HANDOVER.md                   this file
└── skills/
    └── worldview-assistant/      the skill
        ├── SKILL.md              persona + multi-step PROCESS
        ├── references/           tool specs, contexts, guardrails
        └── assets/               1500+ catalog/story files (heavy)
```

**Key decisions and why:**

- **Plugin lives at repo root, not in a subdir.** The nested layout
  (`ieso-worldview-plugin/`) required the `path` field on
  marketplace.json's GitHub source, which the current stable Claude
  Code release doesn't support — installs failed with "source type
  your Claude Code version does not support." Flat single-plugin
  layout means `source` is the simple
  `{type: github, repo: …}` form that every Claude Code version
  accepts. When you add a second plugin, give it its own repo (or
  revisit when `path` lands).
- **Tokens never live in plugin files.** `plugin.json`'s `userConfig`
  block declares four prompts; Claude Code interactively asks for
  them at install time and stores them in the OS keychain. `.mcp.json`
  references them as `${user_config.<key>}` in auth headers. Safe to
  publish the repo.
- **No CDP / `--remote-debugging-port` setup.** Earlier work
  (in the marimo notebook) needed the CDP-attach trick to dodge
  anyio's same-task enter/exit rule. In a plugin the stdio Playwright
  MCP runs once per Claude Code session — its headed browser persists
  across turns naturally, so plain `npx @playwright/mcp@latest`
  suffices.
- **Skill (not subagent) for the persona.** Skills auto-invoke from
  their `description` field when a query matches; the user can also
  call explicitly as `/akd-plugins:worldview-assistant`.
  Subagents would have been overkill and don't add structured-output
  enforcement anyway (plugins lose the AKD `check_output` validation
  contract).

## Tokens (`userConfig`)

Four FastMCP project tokens (`fmcp_…`) prompted at install. All four
are marked `required: false` — leave blank to skip that tool family.

| userConfig key | Servers it auths |
|---|---|
| `ieso_mcp_key` | GeoUI render/get-state, worldview-permalink, UMM-Vis lookup, Earthdata-search, PDF parser (shared) |
| `vector_db_token` | IESO validation (`search_worldview_layers`, `validate_temporal_coverage`) |
| `sde_mcp_key` | SDE fallback search |
| `eonet_token` | EONET natural-event search |

The actual `fmcp_…` token values come from Sanjog out-of-band, not
from the repo.

## MCP servers (`.mcp.json`)

| name | type | auth | tools |
|---|---|---|---|
| `cmr` | http | none | search_collections, get_granules, get_collection_metadata |
| `ieso-validation` | http | `vector_db_token` | search_worldview_layers, validate_temporal_coverage |
| `geoui` | http | `ieso_mcp_key` | geo_ui_render_intent_tool, geo_ui_get_state_tool |
| `worldview-permalink` | http | `ieso_mcp_key` | worldview_permalink_tool (superseded by GeoUI but kept) |
| `umm-vis-lookup` | http | `ieso_mcp_key` | umm_vis_lookup_tool |
| `earthdata-search` | http | `ieso_mcp_key` | earthdata_search_landing_page_tool |
| `pdf-parser` | http | `ieso_mcp_key` | pdf_parser_tool |
| `sde-search` | http | `sde_mcp_key` | sde_search_tool |
| `eonet` | http | `eonet_token` | eonet_search_tool |
| `playwright` | stdio | n/a | browser_navigate, browser_evaluate, browser_snapshot, … |

The GeoUI tools are the **primary visualization path**; the
permalink tool is left wired but superseded.

## Skill (`skills/worldview-assistant/SKILL.md`)

The skill was ported from a polished v2 export (originally exported
from a care-workspace). Two changes from the export:

1. **Frontmatter collapsed.** The export had two stacked frontmatter
   blocks (a care-workspace wrapper + the real skill frontmatter);
   plugins need exactly one. The resulting `name: worldview-assistant`
   is what determines the invocation handle.
2. **Tool-invocation contradiction removed.** The export's "## Tools"
   section said "tools are markdown specs — write a `uv run python`
   or `curl` client to invoke them." Replaced with "tools are MCP
   tools — call them directly by name."

**PROCESS section was rewritten to use the GeoUI tools.** The v2
export used `worldview_permalink_tool` and had no browser tools.
The new flow:

- Step 5 — iterative refinement (when adjusting an already-visualized
  state): read the live URL via `browser_evaluate` ONCE per turn →
  parse with `geo_ui_get_state_tool` → modify only the fields the
  user asked to change. Auto-injected base/reference layers in the
  parsed intent carry through untouched.
- Step 6 — visualization: build a GeoIntent → call
  `geo_ui_render_intent_tool` → immediately `browser_navigate` to
  the returned URL. Turn-completion gate: NO more tool calls after
  navigate succeeds.

**Loop-prevention guards** are codified in the skill's Hard
Constraints (one `browser_evaluate` per turn; never re-observe after
navigate). These were ported from `ieso_w_geoui/agent.py` after a
real bug — the original agent looped `browser_evaluate` ↔
`geo_ui_get_state_tool` ~50 times on a "zoom here" request before
the user manually stopped it. The fix: forbid post-write
re-observation in the same turn.

**Extension declaration rule** (also in the skill): to use any GeoUI
extension (compare, chart, raster-styling), you MUST both populate
its namespaced fields AND declare its URI in `geoui_extensions`. The
GeoUI render tool rejects intents that have `compare:*` fields without
the extension URI declared — this was a real benchmark failure.

## Journey / gotchas (so future-you doesn't re-walk them)

1. **anyio + marimo + MCPServerStdio.** Pre-entering an
   `MCPServerStdio` in one marimo cell and reusing across turns
   fails with `RuntimeError: Attempted to exit cancel scope in a
   different task than it was entered in`. anyio binds the MCP's
   internal task group to the entering task; marimo's per-cell
   tasks die on return. Fix: launch Chromium externally (CDP attach).
   **Doesn't apply in the plugin** — Claude Code's stdio MCP
   lifecycle is one-per-session, so plain `npx @playwright/mcp@latest`
   without CDP works.
2. **AKD's hard-capped `request_limit`.** `PydanticAIBaseAgentConfig.
   _wire_usage_limits` model_validator caps `request_limit` at
   `max_tool_iterations` (≤ 50). Threading cumulative usage from
   `run_context` then made it effectively a session-wide cap. Fix in
   the marimo chat handler: pass `usage=None` per turn, accumulate
   cum_usage manually. **Doesn't apply in the plugin** — the LLM is
   the user's Claude, not pydantic_ai.
3. **`worldview_permalink_tool` retired, GeoUI took its place.**
   The original agent used the permalink tool directly. The GeoUI
   Protocol added an abstraction layer (GeoIntent ⟷ URL) so the same
   intent can render to other map apps later. `worldview-permalink`
   stays wired in the plugin but the skill doesn't drive it.
4. **Local install vs. GitHub install rendered differently.**
   `/plugin marketplace add <local-path>` treats that path as the
   marketplace root. `/plugin marketplace add owner/repo` clones the
   repo and looks for `.claude-plugin/marketplace.json` at the *repo
   root* — so any working local layout doesn't automatically work via
   GitHub. The `path` field on GitHub sources lets a plugin live in
   a subdir, but isn't supported on current stable Claude Code.
5. **Tool-call runaway loops.** Strong system-prompt guards
   (once-per-turn observation, turn-completion gate) are the only
   thing preventing the agent from polling `browser_evaluate` after
   every action. If the loop returns, the next escalation is making
   the guard a Hard Constraint refusal class (currently it's a
   Hard Constraint description, which the model has more latitude
   to interpret).
6. **FastMCP cold-start latency.** Free FastMCP instances hibernate
   when idle; first call after a quiet period can take 10–25s. Not a
   bug, just expected. If `/mcp` shows an "error" status on first
   look, retry — it usually recovers.

## Recipient install (the share blurb)

> **IESO Worldview — Claude Code plugin**
>
> Explore NASA Worldview in natural language. Runs on your own Claude
> Code subscription — no LLM API key.
>
> **Prerequisites:**
> - Claude Code (`claude --version` should print)
> - Node.js (`node --version` should print)
>
> **Install (inside Claude Code):**
> ```
> /plugin marketplace add sanzog03/claude-skills-and-plugins
> /plugin install akd-plugins@akd-plugins
> ```
>
> On install you'll be prompted for four FastMCP tokens (`fmcp_…`).
> Get them from Sanjog out-of-band; paste each into its prompt. Leave
> any blank to skip the corresponding tool (the plugin still works
> with reduced capability).
>
> First time you ask a question that opens the map, Playwright will
> download Chromium (~150 MB, one minute) — that's normal.
>
> Try: *"Show me MODIS Aqua aerosol optical depth over the Gulf of
> Mexico for 2025-09-15."*

## Continuing work elsewhere

**Setting up on a fresh machine / workspace:**

```bash
git clone git@github.com:sanzog03/claude-skills-and-plugins.git
cd claude-skills-and-plugins
```

**Iteration workflow:**

1. Edit any file (`SKILL.md`, `.mcp.json`, references, etc.).
2. In Claude Code: `/reload-plugins` picks up the change without
   restart. Exception: edits to `userConfig` in `plugin.json`
   require a `/plugin uninstall` + `/plugin install` cycle to
   re-prompt for tokens.
3. Commit + push as usual.

**To test changes against the GitHub-distributed version (recommended
before declaring something fixed):**

```
/plugin uninstall akd-plugins
/plugin marketplace remove akd-plugins
/plugin marketplace add sanzog03/claude-skills-and-plugins
/plugin install akd-plugins@akd-plugins
```

(Remove before re-add — Claude Code caches the marketplace clone, and
stale caches cause confusing errors.)

**Diagnostics if things break:**

- `/plugin` → Errors tab — manifest load issues
- `/mcp` — per-server connection state
- `tail -f ~/.claude/logs/claude-code.log` — stderr from MCP
  subprocesses (Playwright especially)

## Open TODOs / known limitations

- **`assets/` is heavy (~1,500 files).** A local catalog/story
  mirror inherited from the v2 export. Probably not all load-bearing
  for the plugin once `search_worldview_layers` is wired. Trim if
  repo size becomes a problem.
- **`worldview_permalink_tool` is wired but unused.** The skill
  doesn't drive it (GeoUI tools took over). Can be removed from
  `.mcp.json` once you're confident nothing references it.
- **Single-plugin repo.** When you add a second plugin, the
  "claude-skills-and-plugins" name becomes misleading (it now hosts
  exactly one plugin). Either rename, give the second plugin its own
  repo, or wait for Claude Code's `path`-on-GitHub-source support and
  restore the workspace pattern.
- **`marketplace.json` source matches the GitHub `owner/repo` of
  THIS repo.** If you fork or move the repo, `source.repo` in
  `marketplace.json` and the share-blurb URLs need to be updated.
- **No real test for the FastMCP cold-start UX.** First call after
  hibernation may visibly delay; the user should know that's
  expected. Worth a note in README if not already there.
- **Audience strategy unresolved.** Plugin distribution requires
  Claude Code + Node.js — fine for "knows how to install Claude Code
  but not MCPs," not for "non-technical end user on phone/web."
  Earlier discussions explored hosted-web-app and claude.ai-connector
  paths; deferred.

## Related context (pointers, not requirements)

- **Original Python agent** (the source of the system prompt and the
  GeoUI Protocol design):
  `/Users/thapa24-mbp/Devs/akd/akd-ext/.worktrees/experiment-ieso_agent_with_geoui_protocol/ieso_w_geoui/agent.py`.
  Plugin's skill is derived but not synced — they diverge.
- **v2 skill export** (the starting point for SKILL.md and the
  references tree): `/Users/thapa24-mbp/Downloads/ieso-worldviewagent-v2/`
  (left pristine).
- **Marimo chat notebook** (the pre-plugin runtime):
  `ieso_w_geoui/notebooks/chat.py` in the same worktree. Useful as a
  reference for how the agent was driven when it had AKD scaffolding.
- **GeoUI Protocol implementation** (the Python adapter functions
  that the GeoUI MCP server wraps):
  `ieso_w_geoui/adapter.py` and `ieso_w_geoui/extensions/` in the
  worktree.
- **Poster project** (the broader academic context — NASA-ESA
  workshop, "Agentic UI for GeoAI"): `~/Devs/workshops/esa-nasa/geo-ui-poster/`.
