# Vocabulary Capture Rules

## What Qualifies as a Domain Term

Include a term in `CONCEPTS.md` when **all** of these hold:
- It is an **entity, named process, or status concept** with project-specific meaning
- A new engineer would need it to follow conversations and code in this area
- It is not self-explanatory from its name alone

**Do not include:**
- Generic programming concepts (`interface`, `module`, `function`)
- File paths, class names, or function signatures (these drift)
- Configuration values, thresholds, or counts (these drift)
- Terms only meaningful in a single file or function

## Entry Format

One-sentence definition. Add behavioral rules only when non-obvious.

Example:
```markdown
## BriefSystem
The pipeline that converts raw email inputs into structured client briefs. Runs as a background job triggered by inbound email receipt.
```

## Bootstrap Structure

When creating `CONCEPTS.md` from scratch, start with this preamble:

```markdown
# Concepts

> Shared domain vocabulary for this project — entities, named processes, and status concepts with project-specific meaning. Seeded with core domain vocabulary, then accretes as capturing-knowledge and refreshing-knowledge process learnings; direct edits are fine. Glossary only, not a spec or catch-all.
```

Then add entries. Let term count drive shape: 1–4 terms → flat headings, more → cluster by domain relationship.

## Scrub Violations

During refresh, scan existing entries for content that violates the above criteria:
- Implementation specifics (file paths, class names, function signatures, code references)
- Current-config values (thresholds, counts, enum values that will drift)
- Status / owner / date metadata
- Duplicates of terms covered under a different name
- Entries that lean on an undefined project-specific sibling (add the sibling or rephrase)

Rewrite or consolidate violating entries. A full sweep is appropriate during refresh — bounded to the area in scope.

## Scope Discipline

Seed and reconcile entries only for the area in scope during the current refresh run. Do not expand to other categories or run a repo-wide audit. The report should note that additional entries are likely from refresh runs on other scopes.
