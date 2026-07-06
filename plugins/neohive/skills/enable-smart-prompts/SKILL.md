---
name: enable-smart-prompts
description: Use when the user says "make NeoHive smarter", "rewrite my prompts before searching memory", "enable smart recall", or during `getting-started` Phase 4. Generates a tailored prompt-rewriting helper that uses a small model (Haiku by default) to rewrite the user's prompt into a good `memory_recall` query, decides when lookup is worthwhile, calls NeoHive, and surfaces only the most relevant results. Codex's hook surface is more limited than Claude's, so the helper installs as a shell-callable script that the user can wire into their Codex prompt flow via whatever pre-prompt mechanism their build supports.
---

# Enable Smart-Recall Prompts (Codex)

You help the user install a customized prompt-rewriting helper that intercepts their prompt, uses a small model to formulate a good NeoHive query, calls `memory_recall`, and surfaces the most relevant results.

This is a **dynamic setup** — every user has a different hive layout, shell, API key location, and tolerance for latency. You walk them through each choice with a strong recommended default, then write the script.

> ⚠️ Codex platform note: Codex does not currently document a stable `UserPromptSubmit`-equivalent hook the way Claude Code does. The generated script is installed as a standalone tool. Wiring it into the prompt path depends on your Codex distribution — instructions are in Phase 4.

## Phase 0 — Check prerequisites

Before asking anything, verify:

```bash
command -v claude >/dev/null && echo "claude-cli: OK" || echo "claude-cli: MISSING (needed for headless model invocation)"
command -v curl   >/dev/null && echo "curl: OK"       || echo "curl: MISSING"
command -v python3>/dev/null && echo "python3: OK"    || echo "python3: MISSING"
[ -n "${ANTHROPIC_API_KEY:-}" ] && echo "ANTHROPIC_API_KEY: set" || echo "ANTHROPIC_API_KEY: not set (required for headless claude)"
```

If `claude-cli` or `python3` is missing, stop and tell the user to install them. If the API key is missing, tell them to export it first and point at https://console.anthropic.com/.

(The query rewriter uses the Claude CLI because it has the best small-model latency/quality at the moment. If you'd prefer to use an OpenAI model instead, edit the script after generation — the rewrite-and-filter calls are the only two model invocations.)

## Phase 1 — Gather configuration

Ask these in sequence, one at a time:

### 1. Which hive to target

Call `list_hives`. Ask which hive the helper should search on every prompt. Default: all hives (cross-hive RRF), which calls `memory_recall` without a `hive` param.

### 2. Which model drives the query rewriter

Options:
- `claude-haiku-4-5` (Recommended) — fast + cheap
- `claude-sonnet-4-6` — more accurate, slower, ~10x cost
- `claude-opus-4-7` — overkill, only for very noisy hives

### 3. Trigger policy

Options:
- (Recommended) Every prompt longer than 10 chars — skips short clarifications
- Only when prompt contains a keyword the user picks
- Every prompt — no filtering
- Manual only — user triggers via an env flag

If "keyword": ask for the keyword(s).

### 4. Install location

Options:
- (Recommended) `~/.codex/hooks/neohive-smart-recall.sh` — personal, all projects
- `./.codex/hooks/neohive-smart-recall.sh` — this project only
- Just show me the script — I'll place it myself

### 5. Disable-flag name

Options:
- (Recommended) `NEOHIVE_SMART_DISABLED`
- `NEOHIVE_HOOK_DISABLED`
- Custom — user types

## Phase 2 — Preview the generated script

Build the script from the template at `${CODEX_PLUGIN_ROOT}/skills/enable-smart-prompts/template.sh`, substituting the chosen values. Show the final script to the user in a fenced code block. Summarize:

```
Generated helper with:
  • Hive:          <hive-or-all>
  • Model:         <model>
  • Trigger:       <policy>
  • Install path:  <path>
  • Disable flag:  <env-var>
```

Ask: "Install this helper now?"
Options:
- (Recommended) Yes, write it
- No — I want to tweak the script first

If "tweak": ask what to change, regenerate, re-preview.
If "no": stop with "Nothing written."

## Phase 3 — Write the script

Create parent directories if needed. Write the script to the chosen location with mode `0755`. Show:

```
Wrote <path> (N bytes, mode 755).
```

## Phase 4 — Wire it into Codex (per-platform guidance)

This is the most platform-dependent step. Tell the user:

> The helper is installed as a standalone script. To make it run on every prompt, you need a Codex feature that supports a pre-prompt hook. Currently:
>
> - **If your Codex distribution supports a `UserPromptSubmit`-style hook in `~/.codex/config.json`**, add an entry pointing at `<install-path>`.
> - **If not**, call the script manually before sending a prompt:
>
>       export NEOHIVE_SMART_RUN=1
>       echo "your prompt here" | <install-path>
>
>   and paste the output into your Codex session as context.
> - **Watch the [NeoHive Codex plugin docs](https://github.com/NeoHiveAi/NeoHiveCodex)** for updates as Codex's hook surface matures — this skill will be updated to register the hook automatically once a stable API exists.

## Phase 5 — Verification

Tell the user:

> Test it manually: pipe a sample prompt to the script and confirm output:
>
>     echo "how do we handle auth in this codebase" | <install-path>
>
> Expected: a "NeoHive smart context:" block with 1–3 relevant memories, or silent exit (no context found).
>
> Disable temporarily: `export <DISABLE_FLAG>=1` in your shell.
> Disable permanently: remove the entry from your Codex config, or delete the script.

## Important rules

- **Never overwrite an existing helper at the target path without confirmation.** If the file exists, show its contents and ask whether to replace.
- **Never put the API key in the generated script.** The script reads `$ANTHROPIC_API_KEY` at runtime.
- **Never hardcode the hive UUID in the script.** It discovers the MCP URL the same way the standard plugin does (via Codex MCP config files in `~/.codex/`).
- **Always set a `--max-time` on every `curl` and `claude -p` call.** A slow helper blocks every prompt.
- **Gracefully exit 0 on any failure.** A broken helper must never block the user's prompt from reaching Codex.
