---
name: design-codebase-docs
description: Use when the user says "generate docs", "build out documentation", "document this codebase", "design our doc standard", or "NeoHive should help with docs". Multi-phase orchestrator that designs a documentation standard tailored to the codebase through Socratic dialogue (audience, depth, reference vs tutorial, per-boundary vs per-subservice), saves it to NeoHive as a reusable convention, generates 2-3 sample pages to validate the approach, then hands off to a fresh session for full generation. Output is the agreed standard plus validated samples, not a complete doc set.
---

# Design a Documentation Standard with NeoHive

You orchestrate a documentation design + sample-generation workflow. This skill's output is NOT a complete set of docs; it is an **agreed-upon standard plus 2 to 3 validated sample pages**, after which the user starts a fresh session to actually generate the bulk of the docs. This split exists because generating a full doc set in one session causes context rot and produces inconsistent output.

**Mental model:** you drive the user through phases, dispatch helpers for heavy lifting where the runtime supports it, and save the outcome to NeoHive so a future session can pick up where you left off.

## Phase 0 — Load existing context

Call `memory_context` with this exact task description:

> "documentation standards and golden standard for this codebase"

Report what came back. If a prior gold standard is already stored, skip to Phase 4 and ask the user if they want to use it as-is, tweak it, or start over.

## Phase 1 — Discover the current state of docs

Survey existing documentation in the repo and report:

1. README files (location, length, primary audience — infer from content)
2. `/docs` directory layout if present (tree, top-level sections)
3. Inline doc density — pick 3 representative source files and estimate docstring/comment coverage
4. Doc generators in use — look for mkdocs.yml, docusaurus.config.\*, sphinx conf.py, typedoc.json, rustdoc settings
5. Cross-references — grep for broken links like `](./` paths and report count
6. Identify the codebase's primary logical boundaries (subservices, packages, apps) — give a flat list

If this repo has an indexed NeoHive hive of type `repo`, prefer `memory_recall` for items 1, 3, and 6 — semantic recall on "documentation overview", "module structure top-level boundaries", "README primary audience" will surface relevant snippets faster than walking the tree.

Summarize the discovery back to the user in 5–8 lines, then stop for acknowledgement.

## Phase 2 — Propose a logical split

Based on the discovery, propose 2–3 ways to split the docs. Ask:

> How should the docs be organized at the top level?
>
> Options:
>
> - (Recommended) By subservice/package — matches the repo's apparent boundaries
> - By audience — separate tracks for users, developers, ops
> - By topic — guides, reference, tutorials, ADRs as parallel trees
> - Flat — one bucket per logical area, cross-linked

After they pick, ask one follow-up if needed: "Within each <X>, do you want guides first, reference first, or both at equal weight?"

## Phase 3 — Elicit the "gold standard" (Socratic)

This is the heart of the skill. You ask questions **one at a time**, each building on the last, until you have a clear, write-downable standard. Don't skip questions even if you think you know the answer.

Ask these in order (one question per turn):

### 3.1 Primary audience

> Who is the primary reader of these docs 6 months from now?
> Options: New hires / developers joining the codebase (Recommended), External users / customers, Product managers / non-technical stakeholders, Ops / SRE

### 3.2 Secondary audience

> Who's the second most important reader?
> Options: dynamically exclude the Phase 3.1 answer; include `Nobody — optimize only for primary`.

### 3.3 Optimize for speed or depth

> If a reader has 10 minutes, what should they come away with?
> Options:
>
> - (Recommended) The ability to run the code and extend it in an obvious way
> - A mental model of the architecture
> - A list of every public API they can call
> - Answers to the top 3 support questions

### 3.4 Reference vs tutorial weight

> What's the balance between tutorial-style prose and reference-style tables/signatures?
> Options:
>
> - (Recommended) 70% tutorial, 30% reference — lead with concepts
> - 50/50
> - 30% tutorial, 70% reference — assume reader can read code
> - Reference only — prose lives in commit messages

### 3.5 Code example density

> How heavily should pages lean on code examples?
> Options:
>
> - (Recommended) Every concept gets a runnable example
> - Examples only for non-obvious patterns
> - Examples as footnotes, linked separately
> - No inline examples — link to /examples

### 3.6 Where they feel gaps now

This is open-ended — let the user type freely.

> Where do current docs feel weakest?
> Suggested starters: Onboarding / getting started, API reference completeness, Architecture / 'why' explanations, Error messages and troubleshooting

### 3.7 Style constraints

> Any style rules to enforce? (Pick all that apply)
>
> - Line length 80/100
> - Second-person ("you") not third-person
> - No emojis
> - American English
> - Present tense for behavior, past for decisions

## Phase 4 — Draft the gold standard document

Assemble all 7 answers into a single document called `docs/DOC_STANDARD.md` (or similar — ask the user). Structure:

```markdown
# Documentation Standard

_Generated by NeoHive design-codebase-docs on <date>. Edit freely; rerun to regenerate samples._

## Audience

- Primary: <answer>
- Secondary: <answer>

## Goals

<from 3.3: what a 10-minute read delivers>

## Structure

- Top-level split: <from Phase 2>
- Within each unit: <from follow-up>
- Tutorial/reference balance: <from 3.4>

## Page template

Every page includes, in this order:

1. One-line purpose
2. <...derived from the above answers...>

## Code examples

<from 3.5>

## Style

<from 3.7>

## Known gap focus

<from 3.6 — the first 3 pages generated should prioritize this>
```

Write this file. Show it to the user. Ask:

> Does this standard capture what you want?
> Options: (Recommended) Yes, save to NeoHive and generate samples / Tweak a section — I'll tell you which / Start over — the audience/goals are wrong

If "tweak": ask what section, rewrite, re-preview.

## Phase 5 — Save the gold standard to NeoHive

Call `list_hives`. Ask which hive (default to the one matching the repo). Then `memory_store` with:

- `hive`: chosen hive
- `type`: `convention`
- `content`: the full `DOC_STANDARD.md` contents (this is the canonical reference for future sessions)
- `tags`: `["documentation", "standard", "gold-standard", <repo-name>, <primary-audience-keyword>]`
- `importance`: 9
- `metadata`: `{"source": "design-codebase-docs", "file": "docs/DOC_STANDARD.md"}`

Confirm the store succeeded. Report the memory ID.

## Phase 6 — Generate 2–3 sample pages

Pick target pages that stress-test the standard:

- One tutorial page (exercises "speed/depth" + "example density" decisions)
- One reference page (exercises "ref vs tutorial" + "style constraints")
- One architecture-level page (exercises "secondary audience" if different from primary)

Ask once: "Generate the 3 sample pages back-to-back in a single pass (fast, less oversight) or one-at-a-time with my review between each (slower, course-correctable)?"

After generation, present each page to the user and ask whether to keep, tweak, regenerate, or fix the standard. If the standard needs fixing, loop back to Phase 4. Every tweak is a learning — store it in NeoHive as an `insight` memory referencing the standard's memory ID.

## Phase 7 — Hand off to a fresh session

This is critical. Do NOT attempt to generate the rest of the docs in this session. Tell the user, verbatim:

> ✓ Gold standard saved to NeoHive (id: <memory-id>)
> ✓ `docs/DOC_STANDARD.md` written
> ✓ <N> sample pages validated
>
> **Now start a fresh Codex session.** Generating the full doc set in this conversation would cause context rot — the model loses precision as the session grows. In the new session, run:
>
>     load-context  generating docs per DOC_STANDARD.md
>
> That will pull your gold standard out of NeoHive and Codex can generate page-by-page with a clean context per batch. If you want to keep going anyway in this session, say "keep going" and I will — just know quality may drift.

If the user says "keep going", acknowledge the risk and proceed, but work in batches of 5 pages with a context-check between each batch.

## Important rules

- **Never skip Phase 3.** The gold standard is the whole value of this skill. Don't let the user rush past it.
- **Never write a doc page before Phase 5.** Samples come after the standard is saved.
- **One question per turn.** Even if two questions feel related, keep them separate — users process decisions better one at a time.
- **Every phase ends with a visible checkpoint.** The user should always know which phase they're in and what's next.
- **Store learnings as you go.** Each user correction in Phase 6 is a memory worth keeping — use `insight` type, importance 6-7, linked to the standard.
- **Respect the context-rot recommendation.** If the user insists on generating everything in-session, warn clearly and batch.
