# NeoHiveCodex

An OpenAI Codex plugin for the [NeoHive](https://github.com/NeoHiveAi) cognitive memory system. Install this plugin to wire Codex into any NeoHive MCP server — persistent semantic memory across sessions, automatic context recall, and post-session learning extraction.

## Quick Start

### 1. Install the plugin

Follow your Codex plugin install flow for GitHub sources pointing at `NeoHiveAi/NeoHiveCodex`. See the [official Codex plugin docs](https://developers.openai.com/codex/plugins/build) for the current install command on your platform.

### 2. Run the guided setup

Invoke the `getting-started` skill. It walks through registering a NeoHive MCP server, setting up auth, generating a project-specific topology block in your `AGENTS.md`, migrating any existing project memory (`CLAUDE.md`, `AGENTS.md`, `.cursor/rules`, `.codex/rules`) into NeoHive, and optionally enabling the smart-recall helper. The plugin does **not** ship a pre-configured MCP server — you register your own gateway through Codex's MCP configuration. ~3–5 minutes.

### 3. (Optional) Environment overrides

If your NeoHive server requires auth, export a bearer token before launching Codex:

```bash
export NEOHIVE_TOKEN="your-token-here"
```

Suppress the in-response hint emitted by `memory_recall` / `memory_context`:

```bash
export NEOHIVE_MCP_HINTS=0
```

## Available Components

| Type  | Name                        | Purpose                                                                                                                                                                                                                                                                               |
| ----- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rules | `rules/neohive.md`          | Persistent tool-usage instructions: when to call `memory_context` / `memory_recall` / `memory_store`, recall-first guidance for codebase exploration, prose-encoded guidance for delegated exploration (the Codex analogue of Claude's `explore-neohive` subagent + PreToolUse hook). |
| Skill | `getting-started`           | First-run setup orchestrator: verifies MCP, sets up auth, generates topology, migrates memory, surfaces next steps.                                                                                                                                                                   |
| Skill | `load-context`              | Pre-load relevant NeoHive memories for the current task via `memory_context`. Run at the start of every session.                                                                                                                                                                      |
| Skill | `generate-agents-md`        | Survey connected hives and write a project-specific topology block into `./AGENTS.md` (hive table, write-routing, session-start non-negotiables). Re-runnable when hives change.                                                                                                      |
| Skill | `migrate-memory`            | Scan local memory files (`CLAUDE.md`, `AGENTS.md`, `.cursor/rules`, `.codex/rules`) and migrate project-scoped entries into NeoHive.                                                                                                                                                  |
| Skill | `design-codebase-docs`      | Design a documentation gold standard through guided dialogue, save to NeoHive, validate with 2–3 sample pages, then hand off to a fresh session.                                                                                                                                      |
| Skill | `enable-smart-prompts`      | Generate a tailored prompt-rewriting helper that rewrites prompts with a small model before querying NeoHive. Codex hook-wiring is platform-dependent — see the skill for current guidance.                                                                                           |
| Skill | `capture-session-learnings` | End-of-session extraction of learnings, corrections, and insights into NeoHive.                                                                                                                                                                                                       |

The slugs `start`, `revise-vector-memory`, and `generate-docs` remain as deprecated aliases that redirect to the new names; they will be removed in a future minor release.

## Repository Layout

```
NeoHiveCodex/
├── .codex-plugin/
│   └── plugin.json                       # Plugin manifest
├── rules/
│   └── neohive.md                        # Persistent tool-usage instructions
├── skills/
│   ├── getting-started/SKILL.md
│   ├── load-context/SKILL.md
│   ├── capture-session-learnings/SKILL.md
│   ├── generate-agents-md/SKILL.md
│   ├── migrate-memory/SKILL.md
│   ├── design-codebase-docs/SKILL.md
│   ├── enable-smart-prompts/
│   │   ├── SKILL.md
│   │   └── template.sh
│   ├── start/SKILL.md                    # deprecated alias
│   ├── revise-vector-memory/SKILL.md     # deprecated alias
│   └── generate-docs/SKILL.md            # deprecated alias
└── README.md
```

## Codex vs Claude — Feature Notes

Codex's plugin surface is leaner than Claude Code's. Two Claude-only mechanisms are encoded as prose in `rules/neohive.md` rather than as runtime hooks:

- **`explore-neohive` subagent (Claude)** → Codex has no per-agent tool allowlist, so the rules file instructs you to apply the same "recall first, traverse second" pattern manually and to include that directive when delegating exploration to another agent or task runner.
- **`PreToolUse` hook on Glob/Grep (Claude)** → Codex has no equivalent hook surface, so the rules file prompts you to mentally check "would `memory_recall` answer this?" before running broad filesystem searches in indexed projects.

The `enable-smart-prompts` skill installs a standalone shell helper. Wiring it into the prompt path depends on which Codex distribution you use and whether it exposes a `UserPromptSubmit`-equivalent — see the skill for current guidance.

## Versioning

Plugins are cached by version at `~/.codex/plugins/cache/$MARKETPLACE_NAME/$PLUGIN_NAME/$VERSION/`. Bump `version` in `.codex-plugin/plugin.json` on every change — without a bump, installed users will not see updates. For local development, restart Codex after edits so the local install picks up changes.
