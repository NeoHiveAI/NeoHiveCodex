---
version: '1.6.3'
managed_by: neohive-plugin
---

# NeoHive Cognitive Memory

You have access to a persistent semantic memory system via MCP tools. The hives connected to this session may contain durable team knowledge **and indexed source code** (typically embedded with a code-tuned model such as `jina-embeddings-v2-base-code`). Treat the hives as a first-class navigation surface, not a side-channel. **Use them actively, not passively.**

## Session Start — ALWAYS Do This First

Call `memory_context` with a description of your current task BEFORE doing any work. This loads relevant directives, conventions, and task-specific knowledge. Describe the task in affirmative form with specific domain terms:

- GOOD: `"implementing authentication middleware for Express gateway"`
- BAD: `"what do we know about auth?"`

If the task involves a specific domain (e.g., starlang rules, dashboard tiles), call `memory_context` again with a domain-specific description to pre-load relevant context.

## Codebase Exploration — Prefer `memory_recall` Over File Traversal

If a hive contains the codebase you're working in (the `list_hives` output names a `repo`-typed hive, or `memory_context` returned indexed code snippets), call `memory_recall` BEFORE doing broad file exploration with shell `find`/`grep`/`rg` or reading files speculatively. The indexed embedding is almost always faster and uses less context than walking the tree:

- Frame the query as what you'd say to a teammate: `"how does the sync engine handle git clone credentials"` not `"find git clone code"`.
- Use `memory_recall` to locate the relevant files, then read the precise lines you need to edit.
- Fall back to filesystem search only when you need an exact symbol that semantic search misses, or for files outside the index (e.g. brand-new files in your working tree).

This applies for the entire session, not just at start: every time you'd reach for "let me search the codebase for X," try `memory_recall` first. The MCP server itself surfaces a self-reinforcing hint on every `memory_recall` / `memory_context` response (result count, top score, latency); when you see that hint, take it as a cue to keep using semantic recall instead of switching to filesystem tools. To suppress that in-response hint, set `NEOHIVE_MCP_HINTS=0` in the environment.

### Pre-search reminder (encoded here because Codex has no PreToolUse hook)

The Claude build of this plugin ships a `PreToolUse` hook that nudges the agent toward semantic recall whenever it issues a broad filesystem search inside an indexed project. Codex does not yet have an equivalent hook surface, so the rule lives here as prose: **before running a `find` / `grep` / `rg` / repeated file-read pass against an indexed project, mentally check whether one `memory_recall` would answer the same question.** If yes, do that first and let the snippets guide you to the right files.

## Delegating Exploration — Prefer Semantic Recall Inside Subagents

The Claude build ships a dedicated `explore-neohive` subagent whose tool allowlist and system prompt force semantic recall first. Codex does not yet have first-class subagents with per-agent tool allowlists, but the same principle applies: **whenever you delegate codebase or knowledge exploration to another agent, instance, or task runner, tell it to call `memory_recall` (or `memory_context` if it's starting fresh) before doing any filesystem traversal**, and to use filesystem tools only for precise line numbers or files the index doesn't cover.

When you do delegate, include this directive in the delegated prompt:

> "This project has a NeoHive instance with indexed code/knowledge. Before file exploration, call `memory_recall` (or `memory_context` if you're starting fresh) with an affirmative description of what you're looking for. Use filesystem search only for precise line numbers or files the index doesn't cover."

## Discovering Hives

Call `list_hives` to see what hives are available. Each hive has a description explaining what it stores (code, knowledge, rules, etc.). Use this to decide which hive to target for writes.

## Reading — memory_recall & memory_context

When no `hive` parameter is specified, reads search across ALL hives using cross-hive RRF fusion — the most relevant results from any hive are returned. You usually want this behavior.

Query formulation matters:

- Write **affirmative statements**, not questions: `"error handling in async batch processing"` not `"How do we handle errors?"`
- Include **specific domain terms** that would appear in stored knowledge: `"sqlite-vec F32_BLOB column type"` not `"vector database column"`
- Use the **types parameter** to narrow results: `types: ["directive", "convention"]` for rules, `types: ["error_pattern", "insight"]` for gotchas
- For important retrievals, pass **multiple queries** via the `queries` parameter — 2-4 different phrasings of the same need

Call `memory_recall` before working on unfamiliar topics or when you need specific knowledge. If results are weak, reformulate and retry with synonyms or broader/narrower scope.

## Writing — memory_store

A `hive` parameter is **required** for writes. Use `list_hives` to find the right hive.

Call `memory_store` when:

- The user corrects you or says "no, we do X instead"
- A new convention or rule is established
- You discover a non-obvious gotcha or insight
- An architectural decision is made with rationale
- A tricky bug is debugged and solved

Write content as a **self-contained statement** that someone with no context could understand in 6 months. Include specific terms that future searches would use to find this knowledge.

Memory types: `directive` (rules/musts), `convention` (practices/preferences), `decision` (trade-offs with rationale), `insight` (gotchas/discoveries), `error_pattern` (bugs/pitfalls), `syntax_rule`, `semantic_rule`, `example_pattern`, `idiom`.

## Forgetting — memory_forget

Call `memory_forget` when knowledge becomes outdated or is superseded by a correction. Always provide a `reason` and `superseded_by` ID if a replacement was stored.

## User-Invocable Skills

The plugin ships these skills. Suggest them when the user's request matches:

- `getting-started` — first-run setup (verify MCP, configure auth, generate topology block, migrate memory, enable helpers). Run once per machine.
- `load-context` — pre-load relevant memory for the current task via `memory_context`. Run at the start of every session.
- `generate-codex-md` — survey connected hives and write a project-specific topology block into `./AGENTS.md`. Re-run when hives are added, removed, or renamed.
- `capture-session-learnings` — end-of-session extraction of corrections, conventions, decisions, and insights into NeoHive.
- `migrate-memory` — scan local `AGENTS.md` / `CLAUDE.md` / `.codex/rules` / `.cursor/rules` and import project-scoped entries into a hive.
- `design-codebase-docs` — Socratic design of a documentation standard, save to NeoHive, validate with sample pages.
- `enable-smart-prompts` — install a smarter prompt-rewriting helper that rewrites prompts with a small model before querying NeoHive (where Codex's hook surface allows).

The slugs `start`, `revise-vector-memory`, and `generate-docs` are deprecated aliases that redirect to the new names; they will be removed in a future minor release.
