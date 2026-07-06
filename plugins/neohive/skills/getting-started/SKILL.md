---
name: getting-started
description: First-run setup for NeoHive. Walks a new user through verifying the MCP server, setting up auth, generating a project-specific topology block in AGENTS.md, and migrating existing project memory. Invoke this once per machine after installing the neohive plugin.
---

# Getting Started with NeoHive

You are onboarding a user who has just installed the NeoHive plugin for Codex. Your job is to get them from zero to a fully working setup — MCP reachable, memory migrated, helpers configured — without ever leaving them staring at a blank screen.

**Golden rule for this skill: never act silently.** Narrate every step. Every decision has a recommended default. Every write is gated by an explicit confirmation.

## Phase 0 — Tell the user what's about to happen

Open with this exact script (do not paraphrase):

> I'll walk you through setting up NeoHive on this machine. This takes 3–5 minutes and covers:
>
> 1. Confirming your NeoHive server is reachable
> 2. (Optional) Setting up your auth token
> 3. Generating a project-specific topology block in your `AGENTS.md`
> 4. Migrating existing project knowledge into NeoHive
> 5. (Optional) Enabling the smart-recall helper
>
> You can stop at any point by saying "stop" or answering "skip" to a step.

Wait for acknowledgement (any affirmative reply, or just continue if they say nothing).

## Phase 1 — Register and verify the MCP server

The plugin ships **without** a pre-configured MCP server, so the first task is registering your NeoHive gateway with Codex.

### 1a. Check whether a NeoHive MCP is already registered

Run this to detect any `*neohive*`-keyed MCP server in the user's Codex config:

```bash
python3 - <<'PY' 2>/dev/null || echo "no neohive MCP found"
import json, os, glob
found = []
candidates = [os.path.expanduser("~/.codex/config.json")] + glob.glob(os.path.expanduser("~/.codex/mcp*.json"))
for path in candidates:
    try:
        with open(path) as f: data = json.load(f)
    except Exception: continue
    servers = data.get("mcpServers", data) if isinstance(data, dict) else {}
    if not isinstance(servers, dict): continue
    for k, v in servers.items():
        if "neohive" in k.lower() and isinstance(v, dict):
            found.append(f"{path}: {k} -> {v.get('url', '<no url>')}")
for line in found or ["no neohive MCP found"]:
    print(line)
PY
```

- **If one is found:** call `list_hives` and interpret per the table below.
- **If none is found:** guide the user to register one (see 1b), then rerun `list_hives`.

### 1b. Registering a server (only if none found)

Tell the user:

> The NeoHive plugin doesn't bundle a default MCP server — you register yours explicitly through Codex's MCP configuration.
>
> Name the server with a key containing `neohive` (e.g. `neohive`) and point it at your gateway URL (e.g. `https://your-neohive-host/hiveminds/<hive-id>/mcp`). See the [Codex MCP docs](https://developers.openai.com/codex/plugins/build) for the exact configuration path on your platform.
>
> After registering, restart Codex and rerun the `getting-started` skill.

Pause here until the user confirms they've registered it, or say "skip" to jump to Phase 6.

### 1c. Verify with `list_hives`

Once a server is registered, call `list_hives` and interpret:

| Outcome                  | What to tell the user                                                                                                                                         |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Returns hives            | "Connected. I can see N hives: X, Y, Z." Proceed to Phase 2.                                                                                                  |
| Empty list               | "Server is reachable but reports no hives. Confirm with your admin — without at least one hive, NeoHive has nowhere to store memories." Pause for user input. |
| Tool unavailable / error | "I can't reach the NeoHive MCP server." Run the diagnostics below.                                                                                            |

### Diagnostics if unreachable

Run these checks and report results in a compact block:

```bash
# 1. Is NEOHIVE_TOKEN set?
[ -n "${NEOHIVE_TOKEN:-}" ] && echo "token set" || echo "token not set"
# 2. Can we reach the registered server?
python3 - <<'PY' 2>/dev/null
import json, os, glob, urllib.request, ssl
url = None
for path in [os.path.expanduser("~/.codex/config.json")] + glob.glob(os.path.expanduser("~/.codex/mcp*.json")):
    try:
        with open(path) as f: data = json.load(f)
    except Exception: continue
    servers = data.get("mcpServers", data) if isinstance(data, dict) else {}
    if not isinstance(servers, dict): continue
    for k, v in servers.items():
        if "neohive" in k.lower() and isinstance(v, dict) and v.get("url"):
            url = v["url"]; break
    if url: break
if not url:
    print("no neohive URL registered — rerun 1b"); raise SystemExit
try:
    ctx = ssl.create_default_context(); ctx.check_hostname = False; ctx.verify_mode = ssl.CERT_NONE
    with urllib.request.urlopen(urllib.request.Request(url, method="GET"), timeout=5, context=ctx) as r:
        print(f"HTTP {r.status} from {url}")
except Exception as e:
    print(f"unreachable: {type(e).__name__}: {e}")
PY
```

Then offer the user: "Fix token now", "I'll fix it later and restart Codex", "Skip MCP setup for now". If they skip, jump to Phase 6 with a warning that memory features won't work.

## Phase 2 — Auth token (only if needed)

If `list_hives` succeeded, skip this phase. Otherwise ask: "Does your NeoHive server require a bearer token?"

- **Yes — I have one** (recommended): show:
    > Export it before launching Codex:
    >
    > ```bash
    > export NEOHIVE_TOKEN="your-token-here"
    > ```
    >
    > Add that line to your shell rc (`~/.bashrc`, `~/.zshrc`, or `~/.config/fish/config.fish`) so it persists. Then restart Codex and rerun this skill.
- **Yes — I need to get one from my admin**: point them at your team's NeoHive admin, then stop.
- **No — it's open**: continue.
- **I'm not sure**: offer to try the call without a token first. If it fails, come back here.

## Phase 3 — Generate project AGENTS.md topology

Now that the MCP is reachable, generate a project-specific topology block in `./AGENTS.md`. This is what makes the model reliable about _which_ hive to query and _where_ new writes should land — without it, NeoHive tool calls run blind because the model has no project-level context for the hive layout.

Ask the user:

> Generate a project topology block in ./AGENTS.md? (Recommended — improves tool-calling accuracy for everyone on this repo.)

- **Yes** (recommended) — invoke the `generate-agents-md` skill
- **Yes, but let me review the table before writing** — invoke `generate-agents-md` (it has its own review gates)
- **Skip — I'll run generate-agents-md later**

If yes, invoke the `generate-agents-md` skill. The sub-skill handles its own confirmation gates (synthesis review + diff review), so this phase just waits for it to return. When it returns, report: "Topology block written to ./AGENTS.md (N hives mapped)."

If skip, tell the user they can run `generate-agents-md` anytime to add the block, and continue.

## Phase 4 — Migrate existing project memory

Ask the user:

> Want me to scan this project for existing knowledge (AGENTS.md, CLAUDE.md, `.codex/rules`, `.cursor/rules`) and migrate the project-specific parts into NeoHive?

- **Yes** (recommended) — invoke the `migrate-memory` skill
- **Yes, but let me review each memory first** — invoke `migrate-memory` with argument `review=each`
- **Skip — nothing worth migrating**
- **Skip — I'll do this manually later**

Wait for the migrate skill to complete. Report: "Migration done — N memories stored." Then continue.

## Phase 5 — (Optional) Enable smart-recall helper

Ask the user:

> Want a smarter prompt-rewriting helper? It uses a small model to rewrite each prompt into a good `memory_recall` query before searching, surfacing only the most relevant memories.

- **Yes** (recommended) — invoke the `enable-smart-prompts` skill
- **Skip — I'll enable later**

If the user opts in, wait for the skill to complete.

## Phase 6 — Final summary

Print a checklist. Use ✓ / ○ prefixes:

```
✓ MCP server reachable (N hives: ...)
✓ Auth token configured
✓ Project topology block in ./AGENTS.md (N hives mapped)
✓ N project memories migrated
○ Smart-recall helper (skipped)
```

Then this exact closing block:

> **You're set. Two things to remember:**
>
> 1. At the start of a new session, invoke `load-context` with a short description of what you're working on. That pre-loads relevant memory.
> 2. At the end of a session, invoke `capture-session-learnings` so new insights get captured.
>
> When docs feel stale, try `design-codebase-docs`. When you add/remove hives, re-run `generate-agents-md`. Rerun `getting-started` anytime to revisit these steps.

## Important rules

- **Never call `memory_store` directly from this skill.** Delegate to `migrate-memory` or `capture-session-learnings`.
- **Never edit the user's shell rc files yourself.** Show the command, let them paste.
- **If the user says "stop" or "skip" at any phase, stop immediately** and print the Phase 6 summary with what's done so far.
- **If any sub-skill fails, surface the error plainly** and offer to skip that phase rather than retrying silently.
