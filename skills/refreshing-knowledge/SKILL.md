---
name: refreshing-knowledge
description: Use when docs/solutions/ may have drifted from current code, the user asks to "refresh learnings", "audit docs/solutions/", or "clean up stale docs", or capturing-knowledge flags an older doc as superseded
---

# Refreshing Knowledge

## Overview

Keep `docs/solutions/` lean and trustworthy as code evolves. Review existing learnings against the current codebase, then apply one of five maintenance actions per doc.

**Core principle:** A stale doc is worse than no doc — it actively misleads. Run this skill whenever a learning may have drifted.

**Announce at start:** "I'm using the refreshing-knowledge skill to audit and refresh the knowledge base."

## When to Use

**Use when:**
- A refactor, migration, rename, or dependency upgrade just completed
- `/capturing-knowledge` flagged an older doc as potentially superseded
- The user asks to "refresh learnings", "audit docs/solutions/", or "clean up stale docs"
- Docs/solutions/ hasn't been reviewed in a while and the codebase has moved on

**Don't use for:**
- General code review or debugging work
- Creating new solution docs (use `/capturing-knowledge`)

## Maintenance Actions

For each candidate doc, classify into exactly one of five outcomes:

| Action | Meaning | When |
|--------|---------|------|
| **Keep** | Still accurate and useful | No drift detected |
| **Update** | Core solution is correct, but references drifted | Paths renamed, links broken, metadata stale — solution approach unchanged |
| **Consolidate** | Two+ docs overlap heavily, both correct | One subsumes the other; merge unique content, delete subsumed |
| **Replace** | Old guidance is now misleading, better approach exists | Solution conflicts with current code or architecture changed |
| **Delete** | No longer applicable or distinct | Implementation gone, problem domain gone, no successor |

## Investigation Scope

### Scope Detection

If the user provided a scope hint, try these matching strategies in order until one finds results:
1. **Directory match** — `docs/solutions/[hint]/`
2. **Frontmatter match** — grep `module`, `component`, `tags` fields for the hint
3. **Filename match** — partial filename match
4. **Content search** — keyword search within file contents

If no scope hint was provided, process everything under `docs/solutions/` (excluding `README.md` and any `_archived/` directory).

If a scope hint was provided but produced no matches, report the miss and ask for clarification — do not silently fall back to processing everything.

### Route by Scope

| Scope | Interaction style |
|-------|-------------------|
| 1–2 docs | Investigate directly, present recommendation |
| 3–8 docs | Investigate in parallel, present grouped recommendations |
| 9+ docs | Triage first (read frontmatter only, cluster by module/component), recommend starting area, then proceed in batches |

## Phase 0: Assess and Route

1. Discover candidate docs, estimate scope
2. For broad scope (9+ docs): read frontmatter only for all candidates, cluster by module/component, identify high-impact areas (dense clusters where referenced files may no longer exist), recommend a starting cluster
3. For focused/batch scope: proceed directly to Phase 1

## Phase 1: Investigate

For each doc in scope, read it and cross-reference against the current codebase.

**Dimensions to check:**
- **References** — do file paths, class names, modules still exist?
- **Recommended solution** — does the fix still match how the code actually works?
- **Code examples** — do snippets reflect the current implementation?
- **Related docs** — are cross-referenced docs still present and consistent?
- **Overlap** — while investigating, note when another in-scope doc covers the same problem domain, referenced files, or recommended solution

**Investigation depth:** Match to the doc's specificity — a doc referencing exact file paths needs more verification than one describing a general principle.

### Update vs Replace: The Critical Distinction

- **Update territory** — references moved but the solution approach is the same. Paths, names, links drifted, but the core recommendation still holds.
- **Replace territory** — the solution itself changed. The recommended fix conflicts with current code, the architecture shifted, or the preferred pattern is different.

**The boundary:** if you find yourself rewriting the solution section or changing what the doc recommends, that is Replace, not Update.

## Phase 1.5: Document-Set Analysis

After investigating individual docs, evaluate the set as a whole.

### Overlap Detection

For docs sharing the same module, component, tags, or problem domain, compare across five dimensions:
- Problem statement, solution shape, referenced files, prevention rules, root cause

High overlap across 3+ dimensions → strong **Consolidate** signal.

Ask: "Would a future maintainer need to read both to get the current truth, or is one mostly repeating the other?"

### Supersession Signals

Look for "older narrow precursor, newer broader doc" patterns:
- Newer doc covers the same files and workflow but with better scope or context
- Older doc describes a specific incident that a newer doc generalizes into a pattern

When a newer doc clearly subsumes an older one: the older is a Consolidate candidate.

### Cross-Doc Conflicts

If Doc A says "always use approach X" while Doc B says "avoid approach X" — contradictions are more urgent than individual staleness. Flag for immediate resolution.

## Phase 2: Classify

Assign each doc one recommended action using the investigation evidence. See `references/per-action-flows.md` for the detailed execution flow per action.

### Delete: The Three Conditions

**Auto-delete only when all three hold:**
1. The implementation is gone (code, workflow, feature no longer exists)
2. The problem domain is gone (the app no longer deals with what the doc addresses)
3. Inbound links are absent or unambiguously decorative

If any condition fails:
- Inbound links that are **substantive** (citing doc relies on this one for content not stated inline) → classify as Replace
- Inbound links that are **decorative** ("see also" pointers) → Delete is still fine; clean up citations in the same action
- Problem domain still active under a new implementation → classify as Replace
- Genuine ambiguity → mark stale (add `status: stale`, `stale_reason`, `stale_date` to frontmatter)

**Before classifying Delete:** search repo markdown files (not source code) for the doc's filename slug to find inbound links.

**No archive directory.** When a doc is removed, git history is the archive. Never create `_archived/`.

### Replace: Evidence Requirement

Replace requires enough codebase evidence to write a trustworthy successor. Assess:
- **Sufficient:** You understand both what the old doc recommended and what the current approach is — investigation found the current code patterns.
- **Insufficient:** The drift is too fundamental to document confidently from a file scan alone. → Mark stale instead; recommend `/capturing-knowledge` for when the user next encounters that area.

## Phase 3: Confirm (Interactive Mode)

For straightforward Keep/Update cases: apply directly, no confirmation needed.

Ask the user (via `AskUserQuestion`) only when:
- Right action is genuinely ambiguous (Update vs Replace vs Consolidate vs Delete)
- About to Delete and auto-delete criteria aren't all clearly met
- About to Consolidate and canonical doc choice isn't clear-cut
- About to Replace (show what old doc recommended vs what current code does)

Question style: one question at a time, multiple choice, lead with recommendation, explain rationale in one sentence.

## Phase 4: Execute

For each doc, execute the flow matching its classification. Read `references/per-action-flows.md` for the step-by-step execution of each action.

## Phase 4.5: Vocabulary Reconciliation

After per-doc actions complete, reconcile `CONCEPTS.md` with what the investigation found.

Read `references/vocabulary-rules.md` before proceeding.

1. **Aggregate** domain terms flagged across Phase 1's investigation. Union multiple shades of precision for the same term into one entry.
2. **If `CONCEPTS.md` exists:** add missing qualifying terms, refine entries where the corpus surfaced new precision. Backfill core domain nouns of the area in scope that are central but missing.
3. **If `CONCEPTS.md` does not exist** and at least one term qualifies: bootstrap it. Seed the core domain nouns of the area in scope, not just the surfaced terms. Write the preamble from the schema reference first.
4. **Scrub violations** in any entry you touch: implementation specifics (file paths, class names, function signatures), current-config values, duplicates, entries that lean on undefined siblings.

Record outcome in the report: "Vocabulary: scanned, no qualifying terms" or "Vocabulary: N terms added, N refined, N scrubbed".

Apply silently — no user prompt.

## Discoverability Check

After the report, check whether `AGENTS.md` or `CLAUDE.md` surfaces `docs/solutions/` to future agents. Same logic as in `/capturing-knowledge`. If `CONCEPTS.md` exists, run a parallel check for it too.

If edits are needed: get consent in interactive mode; surface as a recommendation in the report if unattended.

## Phase 5: Commit

After all actions execute and the report is generated, skip this phase if no files were modified.

1. Detect current branch (`git branch --show-current`)
2. Stage only files this skill modified
3. Present commit options based on context:
   - **On main/master:** offer to create a branch + commit + PR (recommended), or commit directly
   - **On feature branch:** offer to commit to current branch (recommended) or create a separate branch
4. Write a descriptive commit message summarizing actions (e.g., "docs: update 3 stale learnings, consolidate 2 overlapping docs, delete 1 obsolete doc")

## Output

```
Refresh Summary
===============
Scanned: N docs

Kept:        X
Updated:     Y
Consolidated: C
Replaced:    Z
Deleted:     W
Marked stale: S

Vocabulary: [scanned, no qualifying terms | created with N entries | updated — N added, N refined, N scrubbed]

--- Per-file results ---
[For each file: path, action, evidence, what was done]
```

For **Keep** outcomes: list them under "Reviewed, no changes" so the result is visible without creating git churn.

## Common Mistakes

| Wrong | Correct |
|-------|---------|
| Updating a doc when the solution itself changed | Check: is the solution approach the same? If not, that's Replace |
| Deleting without checking for substantive inbound links | Always check before classifying Delete |
| Deleting when only the implementation moved (not the problem domain) | Problem domain still active → Replace, not Delete |
| Creating an `_archived/` directory | Delete; git history is the archive |
| Asking questions about whether code changes were intentional | Stay in your lane — doc accuracy only, not code review |

## Red Flags — Stop and Reconsider

- About to delete a doc but haven't checked whether the problem domain is still active
- About to delete a doc but haven't searched for inbound links
- Rewriting the solution section of a doc and calling it "Update" → that's Replace
- More than three docs getting Delete classification without any Consolidate or Replace → investigate whether the problem domains are truly gone
- Scrubbing vocabulary entries without having read the vocabulary rules reference
