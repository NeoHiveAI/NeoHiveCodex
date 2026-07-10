# NeoHiveCodex

An OpenAI Codex plugin for the [NeoHive](https://github.com/NeoHiveAi) cognitive memory system. Install this plugin to wire Codex into any NeoHive MCP server — persistent semantic memory across sessions, automatic context recall, and post-session learning extraction.

## Quick Start

### 1. Install the plugin

Follow your Codex plugin install flow for GitHub sources pointing at `NeoHiveAi/NeoHiveCodex`. See the [official Codex plugin docs](https://developers.openai.com/codex/plugins/build) for the current install command on your platform.

### 2. Run the guided setup

Invoke the `getting-started` skill. It walks through verifying the MCP server, setting up auth, generating a project-specific topology block in your `AGENTS.md`, migrating any existing project memory (`CLAUDE.md`, `AGENTS.md`, `.cursor/rules`, `.codex/rules`) into NeoHive, and pointing you at the other skills. ~3–5 minutes.

### 3. (Optional) Environment overrides

If your NeoHive server requires auth, export a bearer token before launching Codex:

```bash
export NEOHIVE_TOKEN="your-token-here"
```

## Available Skills

| Name                   | Purpose                                                                                                                                                                          |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `getting-started`      | First-run setup orchestrator: verifies MCP, sets up auth, generates topology block, migrates memory, surfaces next steps.                                                        |
| `start`                | Pre-load relevant NeoHive memories for the current task via `memory_context`.                                                                                                    |
| `generate-agents-md`   | Survey connected hives and write a project-specific topology block into `./AGENTS.md` (hive table, write-routing, session-start non-negotiables). Re-runnable when hives change. |
| `migrate-memory`       | Scan local memory files (`CLAUDE.md`, `AGENTS.md`, `.cursor/rules`, `.codex/rules`) and migrate project-scoped entries into NeoHive.                                             |
| `generate-docs`        | Design a documentation gold standard through a guided dialogue, save to NeoHive, validate with 2–3 sample pages, then hand off to a fresh session.                               |
| `revise-vector-memory` | End-of-session extraction of learnings, corrections, and insights into NeoHive.                                                                                                  |

A smart-recall hook equivalent (from the Claude build) is omitted here — the current Codex plugin spec does not document user-prompt hooks. If Codex adds that surface, this plugin will gain a `generate-post-submit-hook` skill.

## Repository Layout

```
NeoHiveCodex/
├── .codex-plugin/
│   └── plugin.json             # Plugin manifest
├── .mcp.json                   # HTTP MCP server registration
├── skills/
│   ├── start/SKILL.md
│   ├── getting-started/SKILL.md
│   ├── generate-agents-md/SKILL.md
│   ├── migrate-memory/SKILL.md
│   ├── generate-docs/SKILL.md
│   └── revise-vector-memory/SKILL.md
└── README.md
```

## Versioning

Plugins are cached by version at `~/.codex/plugins/cache/$MARKETPLACE_NAME/$PLUGIN_NAME/$VERSION/`. Bump `version` in `.codex-plugin/plugin.json` on every change — without a bump, installed users will not see updates. For local development, restart Codex after edits so the local install picks up changes.
