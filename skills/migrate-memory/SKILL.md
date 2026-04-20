---
name: migrate-memory
description: Scan local project memory files (CLAUDE.md, AGENTS.md, GEMINI.md, .codex/rules, .cursor/rules, .claude/rules) and migrate the PROJECT-SPECIFIC entries into NeoHive so the whole team can share them. User-preference entries are filtered out. Always read-only until the user explicitly confirms.
---

# Migrate Local Memory Into NeoHive

You are migrating a user's existing project knowledge (`CLAUDE.md`, `AGENTS.md`, rules directories from other AI tools) into NeoHive so it becomes shared team memory instead of per-machine memory.

**Two non-negotiable rules for this skill:**

1. **Read-only until confirmation.** Scan, classify, present a preview. Do NOT call `memory_store` until the user explicitly says go.
2. **Project-specific only.** User-preference content ("I prefer tabs", "my editor is helix") must never land in a shared hive. If classification is ambiguous, default to excluding.

## Arguments

If the user passed `review=each`, pause between each candidate for per-item approval. Otherwise batch confirm at the end.

## Phase 1 — Discover local memory sources

Scan these locations relative to the current working directory:

```bash
# Project-scoped (CANDIDATES for migration)
for f in ./CLAUDE.md ./AGENTS.md ./GEMINI.md; do
  [ -f "$f" ] && echo "$f ($(wc -c < "$f") bytes)"
done
find ./.codex/rules ./.cursor/rules ./.claude/rules -type f \( -name '*.md' -o -name '*.mdc' \) 2>/dev/null
find ./docs -maxdepth 2 -type f \( -name 'CONVENTIONS*.md' -o -name 'CONTRIBUTING.md' \) 2>/dev/null | head -20

# User-scoped (scanned for context only; NEVER migrated)
for f in "$HOME/.codex/AGENTS.md" "$HOME/.claude/CLAUDE.md" "$HOME/.cursor/rules"/*; do
  [ -e "$f" ] && echo "user: $f (reference only, will NOT migrate)"
done
```

Report findings as a compact block:

```
Project-scoped sources found:
  - ./AGENTS.md (4.2 KB)
  - ./.codex/rules/testing.md (1.1 KB)
User-scoped (will be skipped):
  - ~/.codex/AGENTS.md (8.7 KB)
```

If nothing project-scoped is found, stop: "No project-scoped memory files found. Nothing to migrate. If you've stored knowledge elsewhere, point me at the file(s)."

## Phase 2 — Parse into candidate memories

Break each project-scoped file into **atomic candidate memories**. A candidate is a single self-contained directive, convention, decision, or insight that would make sense read in isolation 6 months from now.

Heuristics:

- One markdown bullet → one candidate
- One paragraph containing a rule ("Always use X", "Never do Y") → one candidate
- A section like "## Testing" containing multiple rules → one candidate per rule, not the whole section
- Code examples: keep the surrounding prose + example together as one candidate

Skip content that is:

- Pure metadata (version headers, tables of contents)
- Installation or setup instructions (these belong in README, not memory)
- Transient state (TODO lists, "currently broken" sections)

## Phase 3 — Classify each candidate

For each candidate, assign:

**Scope:** `project` | `user` | `ambiguous`

- `project`: references this specific codebase, team practices, domain-specific rules. Examples: "We use sqlite-vec for embeddings", "Always run tests through `.venv/bin/python3`", "The `starlang` rule format requires X".
- `user`: personal preferences, editor settings, generic "I prefer X". Examples: "I prefer tabs", "Use fish shell", "My name is X".
- `ambiguous`: could be either. Example: "Never use `--no-verify`" (could be personal discipline OR a team rule).

**Type:** one of `directive`, `convention`, `decision`, `insight`, `error_pattern`, `syntax_rule`, `semantic_rule`, `example_pattern`, `idiom`.

**Importance:** 1–10. Rules/musts → 8–9. Conventions → 6–7. Insights/gotchas → 5–7.

**Tags:** 3–6 domain-specific terms someone would search for.

## Phase 4 — Pick a target hive

Call `list_hives`. Report the list with descriptions. Ask the user which hive should receive these memories. Recommend the hive whose description best matches the project (e.g. a hive with "securisource" in its description for a securisource repo).

If only one hive exists, skip the question: "Using the only available hive: `<name>`."

## Phase 5 — Preview and confirm

Build a preview table of **only the `project`-scoped candidates**:

```
Ready to migrate N project-scoped memories to hive `<hive-name>`.
(Skipping M user-scoped + K ambiguous candidates — see below.)

 #    type         imp   content (first 80 chars)
 ───  ───────────  ────  ──────────────────────────────────────────
 1    directive     9    Always run python scripts through .venv/...
 2    convention    7    Prefer dedicated tools (grep, read) over...

Skipped as user-preference:
  - "I prefer fish shell" (from ~/.codex/AGENTS.md)
Skipped as ambiguous (migrate manually if you want these):
  - "Never use --no-verify" (could be personal or team rule)
```

Then ask the user to choose one:

- **Yes, migrate all** (recommended)
- **Yes, but let me exclude some first** — user names numbers to drop, re-preview
- **Re-classify the ambiguous ones** — walk through ambiguous items
- **No, abort migration**

If abort: print "Migration aborted. No memories stored." and stop.

If `review=each` was passed in arguments, override the above: walk through candidates one by one with yes/no prompts.

## Phase 6 — Deduplicate and store

For each approved candidate:

1. Call `memory_recall` with a query derived from the candidate content, `limit=3`.
2. Read results:
   - **Strong match (score > 0.85 and semantically identical):** skip, report "already known".
   - **Weak/partial match:** store anyway — new semantic angle.
   - **No match:** store.
3. Call `memory_store` with:
   - `hive`: the chosen hive
   - `content`: the candidate content (full, not truncated)
   - `type`, `tags`, `importance` from Phase 3
   - `metadata`: `{"source": "migrate-memory", "origin_file": "<path>", "origin_line": <line>}`

Do writes sequentially. If any write fails, report the error and ask whether to continue with remaining candidates or abort.

## Phase 7 — Summary

Print:

```
Migration complete.
  Candidates found:     N
  Migrated:             X (IDs: ...)
  Already in NeoHive:   Y (skipped via dedup)
  User-scoped:          Z (skipped)
  Ambiguous:            W (skipped — migrate manually if needed)

Next: update your AGENTS.md / CLAUDE.md to reference NeoHive instead
of duplicating these rules. Run revise-vector-memory at the end of
each session to keep the hive fresh.
```

## Important rules

- **NEVER write to a hive before Phase 5 confirmation.**
- **NEVER migrate user-scoped files.** Contents of `~/.codex/`, `~/.claude/`, `~/.cursor/` are per-user by definition.
- **NEVER guess at classification.** When ambiguous, mark ambiguous and surface it.
- **ALWAYS preserve the full content.** Do not summarize candidates before storing.
- **If `list_hives` fails, stop immediately.** Tell the user the MCP server is not reachable and point them at the `getting-started` skill.
