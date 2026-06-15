---
name: capturing-knowledge
description: Use after solving a non-trivial problem — documents the solution to docs/solutions/ while context is fresh so the team never solves the same problem twice. Auto-triggers on "that worked", "it's fixed", "working now", "problem solved"
---

# Capturing Knowledge

## Overview

Document a recently solved problem before context fades. Each captured solution compounds — the next occurrence takes minutes instead of hours.

**Core principle:** Write once, find forever. The compounding value only materializes when solutions are documented while the root cause is still fresh.

**Announce at start:** "I'm using the capturing-knowledge skill to document this solution."

## When to Use

**Use when:**
- A bug is fixed and the root cause is understood
- A workflow pattern or best practice emerged from solving a problem
- A debugging session surfaced non-obvious knowledge worth preserving
- The user says "that worked", "it's fixed", "problem solved", "working now"

**Don't use when:**
- The problem is not yet solved or the fix is unverified
- It's a trivial fix (simple typo, obviously wrong variable name — no institutional value)
- The same problem is already fully documented in an existing doc

## Two Tracks

Classify before writing — the track determines the doc structure.

| Track | For | Required fields |
|-------|-----|-----------------|
| **Bug** | Defects, failures, errors, regressions | symptoms, root_cause, resolution_type |
| **Knowledge** | Patterns, best practices, workflow guidance, architectural decisions | applies_when |

**Classify by `problem_type`** — see `references/schema-and-templates.md` for the full enum and category mapping.

## Workflow

### Phase 1: Research

Run these in parallel (dispatch as subagents) or sequentially in a single pass if context is tight.

**WAIT for all Phase 1 tasks to complete before Phase 2.**

#### Context Analyzer
- Extract the problem and solution from conversation history
- Read `references/schema-and-templates.md` to determine track, `problem_type`, and target category directory
- Suggest filename: `[sanitized-problem-slug].md` — no date suffix; `date:` frontmatter is the canonical date
- Return: YAML frontmatter skeleton (including `category:` field), category directory path, suggested filename, track

#### Solution Extractor
- Read `references/schema-and-templates.md` for the appropriate track template
- Build the doc body using the track template:
  - **Bug track:** Problem → Symptoms → What Didn't Work → Solution (with code) → Why This Works → Prevention
  - **Knowledge track:** Context → Guidance (with examples) → Why This Matters → When to Apply → Examples
- Return: complete doc body

#### Related Docs Finder
- Search `docs/solutions/` for overlapping docs
- Grep frontmatter fields before reading files: `title:.*<keyword>`, `tags:.*<keyword>`, `module:.*<module>`, `problem_type:.*<type>`. Read only strong candidate files in full.
- Score overlap across five dimensions: problem statement, root cause, solution approach, referenced files, prevention rules
  - **High (4–5 dimensions match):** update the existing doc — do not create a duplicate
  - **Moderate (2–3 dimensions):** create new doc; flag overlap in Phase 2.5
  - **Low (0–1 dimensions):** create new doc
- Return: overlap score, matching dimensions, path of overlapping doc if any

### Phase 2: Assemble and Write

**The primary deliverable is ONE file.** Phase 1 subagents return text data only — they do not write files. The orchestrator (main context) writes the single output file. Beyond the solution doc, two maintenance side effects are expected (not extra deliverables):
- `CONCEPTS.md` — created or updated in Phase 2.4
- `AGENTS.md` / `CLAUDE.md` — a small edit during the Discoverability Check if a gap is found

1. **Check overlap assessment:**
   - **High overlap → update** the existing doc with fresher context (new code examples, updated references, additional prevention tips). Preserve the existing file path and structure. Add `last_updated: YYYY-MM-DD` to frontmatter. Do not change the title unless the problem framing materially shifted.
   - **Low/moderate → create** a new file.

2. **Assemble doc:** Use the template from `references/schema-and-templates.md`. Apply YAML-safety quoting for array items that start with `` ` ``, `[`, `*`, `&`, `!`, `|`, `>`, `%`, `@`, `?`, or contain `": "` — wrap in double quotes.

3. **Create directory if needed:** `mkdir -p docs/solutions/[category]/`

4. **Write file:** `docs/solutions/[category]/[filename].md`

### Phase 2.4: Vocabulary Capture

Read `references/schema-and-templates.md` section "Vocabulary Capture Rules" before proceeding. The criteria are non-obvious — pre-judging from memory misses qualifying terms.

Scan the new doc **and** the surrounding conversation for domain terms (entities, named processes, status concepts with project-specific meaning).

- **`CONCEPTS.md` exists:** Add missing qualifying terms. Refine existing entries where new precision surfaced. Inspect the *coherence neighborhood* of any entry you touch (its cluster siblings and cross-referenced terms) — fix drift in directly related entries but don't audit the whole file.
- **`CONCEPTS.md` does not exist and at least one term qualifies:** Create it. Alongside the surfaced term, seed the core domain nouns of the area the fix touched (not just the single surfaced term). Write the preamble first, then add entries.
- **No terms qualify:** Record that outcome explicitly — "Vocabulary capture: scanned, no qualifying terms." Never silently skip.

Apply silently — no user prompt.

### Phase 2.5: Selective Refresh Check

After writing, consider whether an older doc may now be stale:

**Suggest `/refreshing-knowledge` when:**
- A related doc recommends an approach the new fix contradicts
- The fix supersedes an older documented solution
- The work involved a refactor, migration, rename, or dependency upgrade that likely invalidated older references
- Moderate overlap was detected and consolidation might help

**Don't suggest it when:**
- No related docs were found
- Related docs are still consistent with the new learning
- Context is already tight (mention it as a next step instead)

When suggesting, name the narrowest useful scope: a specific file, a module/component name, or a category directory.

### Discoverability Check

After writing, check whether `AGENTS.md` or `CLAUDE.md` (whichever is the substantive instruction file — one may `@`-include the other) would lead an agent to discover `docs/solutions/`.

An agent should learn three things: (1) a searchable knowledge store exists, (2) how it's structured (category dirs, YAML frontmatter fields: `module`, `tags`, `problem_type`), (3) when to consult it.

If not:
- Find where a mention fits naturally. A single line added to an existing directory listing or conventions section beats a new headed section.
- In interactive mode: show the proposed edit and get consent via `AskUserQuestion` before applying.
- In a script/automation context: apply silently.

If `CONCEPTS.md` exists at repo root: run the same discoverability check for it.

## Document Structure

See `references/schema-and-templates.md` for:
- Complete frontmatter schema and all enum values
- Bug track template
- Knowledge track template
- Category → directory mapping
- Vocabulary capture rules

## Output

```
✓ Knowledge captured

File: docs/solutions/[category]/[filename].md  (created | updated)
Track: bug | knowledge
Overlap: none | low | moderate — see [path] | high — existing doc updated
Vocabulary: [N terms added/created | scanned, no qualifying terms]
Discoverability: [passed | edit applied to AGENTS.md | consent pending]
Refresh recommendation: [none | /refreshing-knowledge [scope]]
```

## Common Mistakes

| Wrong | Correct |
|-------|---------|
| Creating a new doc when an existing doc covers the same root cause | Check overlap first; update existing if high overlap |
| Phase 1 subagents write intermediate files to disk | Subagents return text data only; orchestrator writes one final file |
| Starting assembly before all Phase 1 tasks return | Wait for all Phase 1 results before Phase 2 |
| Silently skipping vocabulary scan | Always scan; always record outcome |
| Documenting an unverified fix | Only run this skill after the fix is confirmed working |

## Red Flags — Stop and Reconsider

- About to create a new doc about a bug that looks just like an existing doc → check overlap
- Tempted to skip the overlap check because "this feels different" → run it anyway
- Writing a doc before the fix is verified working → verify first
- Vocabulary capture output is blank with no explanation → always record outcome
- About to write multiple solution files in one run → the primary deliverable is ONE file
