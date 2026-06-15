# Per-Action Execution Flows

Reference for Phase 4 of the refreshing-knowledge skill. One flow per action — only one flow runs per candidate doc.

---

## Keep Flow

**When:** The doc is still accurate and the solution still reflects current code.

1. Do not edit the file
2. In the report, note that it was reviewed and remains trustworthy
3. List under "Reviewed, no changes" section — visible without git churn
4. Only add `last_refreshed:` to frontmatter if you are already making a meaningful update for another reason (e.g., fixing a related doc in the same cluster)

---

## Update Flow

**When:** The core solution is still valid but references have drifted — paths renamed, links broken, module names changed, code snippets stale.

**The boundary:** if you find yourself rewriting the solution or changing what the doc recommends, stop — that is Replace, not Update.

1. Identify the specific drifted references (list them before editing)
2. Apply in-place edits:
   - Renamed file paths and class names
   - Broken links → update or remove
   - Stale module names or component references
   - Code snippets that no longer reflect current implementation (cosmetically, not substantively)
   - Stale `tags`, `related_components`, or `module` frontmatter fields
3. Add `last_updated: YYYY-MM-DD` to frontmatter
4. Do not change the title or core sections unless the framing materially shifted
5. Report: path, what was updated, why

---

## Consolidate Flow

**When:** Two or more docs overlap heavily across 3+ dimensions and both are materially correct — one subsumes the other.

**Distinguish from Delete:** Use Consolidate when the subsumed doc has unique content worth preserving (edge cases, alternative approaches, extra prevention rules). If it adds nothing the canonical doc doesn't already cover, use Delete instead.

1. **Identify the canonical doc** — usually the most recent, broadest, most accurate doc in the cluster; the one a maintainer should find first
2. **Identify content unique to the subsumed doc** — sections, examples, prevention rules, or edge cases not already in the canonical doc
3. **Merge unique content** into the canonical doc:
   - Add it as a section, addendum, or additional bullet — preserve the canonical doc's existing structure
   - Add `last_updated: YYYY-MM-DD` to the canonical doc's frontmatter
4. **Update cross-references:** search repo markdown for citations of the subsumed doc and update them to point to the canonical doc
5. **Delete the subsumed doc** — no archival; git history preserves it
6. Report: canonical path, what was merged, subsumed path deleted, cross-references updated

---

## Replace Flow

**When:** The old doc's guidance is now misleading — the recommended solution conflicts with current code, the architecture shifted, or the preferred pattern changed. Evidence to write a trustworthy successor must be in hand.

**If evidence is insufficient:** Mark stale instead (see stale-marking below) and recommend `/capturing-knowledge` for the user's next encounter with that area.

1. **Assess evidence sufficiency:**
   - You understand both what the old doc recommended AND what the current approach is
   - Investigation found current code patterns, new file locations, changed architecture
   - If not: mark stale, skip to stale-marking

2. **Write the successor** (dispatch as a subagent for context isolation):
   - Use the same doc structure as in `/capturing-knowledge` references
   - Base content on investigation evidence already gathered — do not invent claims
   - Write to the same category directory (or a more accurate one if the category changed)
   - New filename if the problem framing changed materially; same filename if the problem is the same

3. **Validate frontmatter** of the new doc (check required fields, enum values, YAML-safety quoting)

4. **Delete the old doc** — apply the inbound-link cleanup first:
   - Decorative citations (see-also pointers): remove the citation in the same commit
   - Substantive citations: update to point to the new successor doc

5. Report: old path deleted, new path created, what changed, cross-references updated

---

## Delete Flow

**When:** All three auto-delete conditions hold:
1. The implementation is gone
2. The problem domain is gone
3. Inbound links are absent or unambiguously decorative

**Before deleting:**

1. **Final inbound-link check** (even if you checked in Phase 2):
   - Grep repo markdown files (not source code) for the filename slug
   - Read context lines around each match (e.g., Grep's `-B 2 -A 2`)
   - Classify each citation: decorative (see-also, bare attribution) vs substantive (citing doc relies on this one for content not stated inline)
   - Substantive citation found → reclassify as Replace; do not proceed with Delete
   - Decorative citations only → proceed; clean them up in the same commit

2. **Clean up decorative citations:**
   - Remove bare entry lines and see-also pointers in other docs
   - Keep the surrounding doc intact — only remove the specific citation

3. **Delete the file** — `git rm docs/solutions/[category]/[filename].md`

4. Report: path deleted, why (implementation gone, problem domain gone), citations cleaned up

---

## Stale-Marking

**When:** Classification is genuinely ambiguous, or Replace evidence is insufficient.

This is not a permanent state — it marks work for a future session when the user has fresh context.

1. Add to the doc's frontmatter:
   ```yaml
   status: stale
   stale_reason: [one sentence: what was found and why action couldn't be determined]
   stale_date: YYYY-MM-DD
   ```
2. Do not change any other content
3. Report: path, what evidence was found, why action couldn't be determined, recommendation for next step (usually: run `/capturing-knowledge` after next encounter with this area)
