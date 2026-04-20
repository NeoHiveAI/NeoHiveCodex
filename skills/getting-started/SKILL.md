---
name: getting-started
description: First-run setup for NeoHive. Walks a new user through verifying the MCP server, setting up auth, migrating existing project memory, and enabling optional helpers. Invoke this once per machine after installing the neohive plugin.
---

# Getting Started with NeoHive

You are onboarding a user who has just installed the NeoHive plugin for Codex. Your job is to get them from zero to a fully working setup — MCP reachable, memory migrated, helpers configured — without ever leaving them staring at a blank screen.

**Golden rule for this skill: never act silently.** Narrate every step. Every decision has a recommended default. Every write is gated by an explicit confirmation.

## Phase 0 — Tell the user what's about to happen

Open with this exact script (do not paraphrase):

> I'll walk you through setting up NeoHive on this machine. This takes 3–5 minutes and covers:
>   1. Confirming your NeoHive server is reachable
>   2. (Optional) Setting up your auth token
>   3. Migrating existing project knowledge into NeoHive
>
> You can stop at any point by saying "stop" or answering "skip" to a step.

Wait for acknowledgement (any affirmative reply, or just continue if they say nothing).

## Phase 1 — Verify MCP reachability

Call `list_hives` immediately. Interpret the outcome:

| Outcome | What to tell the user |
|---|---|
| Returns hives | "Connected. I can see N hives: X, Y, Z." Proceed to Phase 2. |
| Empty list | "Server is reachable but reports no hives. Confirm with your admin — without at least one hive, NeoHive has nowhere to store memories." Pause for user input. |
| Tool unavailable / error | "I can't reach the NeoHive MCP server." Run the diagnostics below. |

### Diagnostics if unreachable

Run these checks and report results in a compact block:

```bash
# 1. Is .mcp.json present in the plugin install?
ls ~/.codex/plugins/cache/*/neohive/*/.mcp.json 2>&1 || echo "missing"
# 2. Is NEOHIVE_TOKEN set?
[ -n "${NEOHIVE_TOKEN:-}" ] && echo "token set" || echo "token not set"
# 3. Can we reach the server?
grep -oE 'https?://[^"]+' ~/.codex/plugins/cache/*/neohive/*/.mcp.json 2>/dev/null | head -1 | xargs -I{} curl -sS -o /dev/null -w "HTTP %{http_code}\n" --max-time 5 "{}" 2>&1 || true
```

Then offer the user: "Fix token now", "I'll fix it later and restart Codex", "Skip MCP setup for now". If they skip, jump to Phase 4 with a warning that memory features won't work.

## Phase 2 — Auth token (only if needed)

If `list_hives` succeeded, skip this phase. Otherwise ask: "Does your NeoHive server require a bearer token?"

- **Yes — I have one** (recommended): show:
  > Export it before launching Codex:
  > ```bash
  > export NEOHIVE_TOKEN="your-token-here"
  > ```
  > Add that line to your shell rc (`~/.bashrc`, `~/.zshrc`, or `~/.config/fish/config.fish`) so it persists. Then restart Codex and rerun this skill.
- **Yes — I need to get one from my admin**: point them at your team's NeoHive admin, then stop.
- **No — it's open**: continue.
- **I'm not sure**: offer to try the call without a token first. If it fails, come back here.

## Phase 3 — Migrate existing project memory

Ask the user:

> Want me to scan this project for existing knowledge (AGENTS.md, CLAUDE.md, `.codex/rules`, `.cursor/rules`) and migrate the project-specific parts into NeoHive?

- **Yes** (recommended) — invoke the `migrate-memory` skill
- **Yes, but let me review each memory first** — invoke `migrate-memory` with argument `review=each`
- **Skip — nothing worth migrating**
- **Skip — I'll do this manually later**

Wait for the migrate skill to complete. Report: "Migration done — N memories stored." Then continue.

## Phase 4 — Final summary

Print a checklist. Use ✓ / ○ prefixes:

```
✓ MCP server reachable (N hives: ...)
✓ Auth token configured
✓ N project memories migrated
```

Then this exact closing block:

> **You're set. Two things to remember:**
>   1. At the start of a new session, invoke the `start` skill with a short description of what you're working on. That pre-loads relevant memory.
>   2. At the end of a session, invoke `revise-vector-memory` so new insights get captured.
>
> When docs feel stale, try `generate-docs`. Rerun `getting-started` anytime to revisit these steps.

## Important rules

- **Never call `memory_store` directly from this skill.** Delegate to `migrate-memory` or `revise-vector-memory`.
- **Never edit the user's shell rc files yourself.** Show the command, let them paste.
- **If the user says "stop" or "skip" at any phase, stop immediately** and print the Phase 4 summary with what's done so far.
- **If any sub-skill fails, surface the error plainly** and offer to skip that phase rather than retrying silently.
