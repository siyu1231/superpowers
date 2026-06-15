# Schema and Templates

## Track Classification

| `problem_type` | Track | Category directory |
|----------------|-------|--------------------|
| `build_error` | Bug | `docs/solutions/build-errors/` |
| `test_failure` | Bug | `docs/solutions/test-failures/` |
| `runtime_error` | Bug | `docs/solutions/runtime-errors/` |
| `performance_issue` | Bug | `docs/solutions/performance-issues/` |
| `database_issue` | Bug | `docs/solutions/database-issues/` |
| `security_issue` | Bug | `docs/solutions/security-issues/` |
| `ui_bug` | Bug | `docs/solutions/ui-bugs/` |
| `integration_issue` | Bug | `docs/solutions/integration-issues/` |
| `logic_error` | Bug | `docs/solutions/logic-errors/` |
| `dependency_issue` | Bug | `docs/solutions/dependency-issues/` |
| `architecture_pattern` | Knowledge | `docs/solutions/architecture-patterns/` |
| `design_pattern` | Knowledge | `docs/solutions/design-patterns/` |
| `tooling_decision` | Knowledge | `docs/solutions/tooling-decisions/` |
| `convention` | Knowledge | `docs/solutions/conventions/` |
| `workflow_issue` | Knowledge | `docs/solutions/workflow-issues/` |
| `developer_experience` | Knowledge | `docs/solutions/developer-experience/` |
| `documentation_gap` | Knowledge | `docs/solutions/documentation-gaps/` |
| `best_practice` | Knowledge | `docs/solutions/best-practices/` *(fallback — prefer a narrower type)* |

## Frontmatter Schema

### Required Fields (Both Tracks)

| Field | Type | Notes |
|-------|------|-------|
| `title` | string | Brief, descriptive title |
| `date` | date | `YYYY-MM-DD` — creation date |
| `category` | string | Directory name (e.g., `performance-issues`) |
| `module` | string | Primary module/component/feature area affected |
| `problem_type` | enum | See Track Classification table above |
| `severity` | enum | `critical` / `high` / `medium` / `low` / `info` |

### Bug Track Fields

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `symptoms` | Required | array | Observable error messages, behaviors |
| `root_cause` | Required | enum | See Root Cause Enums |
| `resolution_type` | Required | enum | See Resolution Type Enums |

### Knowledge Track Fields

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `applies_when` | Required | array | Conditions where this guidance applies |

### Optional Fields (Both Tracks)

| Field | Type | Notes |
|-------|------|-------|
| `tags` | array | Keywords for search (tool names, concepts) |
| `related_components` | array | Other modules/components touched |
| `last_updated` | date | `YYYY-MM-DD` — added when updating an existing doc |

## Enum Values

### `root_cause`
`configuration_error` | `state_management_issue` | `race_condition` | `incorrect_assumption` | `external_dependency` | `logic_error` | `missing_validation` | `environment_issue` | `version_incompatibility` | `other`

### `resolution_type`
`code_change` | `configuration_change` | `dependency_update` | `environment_fix` | `refactoring` | `workaround` | `documentation_only`

## YAML Safety Rules

Array items must be wrapped in double quotes when the value:
- Starts with: `` ` ``, `[`, `*`, `&`, `!`, `|`, `>`, `%`, `@`, `?`
- Contains the substring `": "`

Example:
```yaml
symptoms:
  - "Build fails with `Cannot find module`"  # quoted: starts with `
  - Plain error message without special chars  # no quotes needed
```

---

## Bug Track Template

```markdown
---
title: [Title]
date: YYYY-MM-DD
category: [category-slug]
module: [module/feature area]
problem_type: [enum]
symptoms:
  - [Observable symptom]
root_cause: [enum]
resolution_type: [enum]
severity: [enum]
tags:
  - [keyword]
---

# [Title]

## Problem
[1-2 sentences describing the issue and its impact]

## Symptoms
- [Exact error message or observable behavior]
- [Secondary symptom if any]

## What Didn't Work
- [Attempted fix and why it failed]
- [Another approach tried and why it didn't address root cause]

## Solution
[Step-by-step fix. Include before/after code when applicable.]

```[language]
// Before
...

// After
...
```

## Why This Works
[Root cause explanation and why this solution addresses it — not just what the fix does but why the problem existed]

## Prevention
- [Concrete practice, test, or guardrail to avoid recurrence]
- [Another prevention measure]

## Related
- [Link to related issue, PR, or other solution doc]
```

---

## Knowledge Track Template

```markdown
---
title: [Title]
date: YYYY-MM-DD
category: [category-slug]
module: [module/feature area]
problem_type: [enum]
applies_when:
  - [Condition where this applies]
severity: [enum]
tags:
  - [keyword]
---

# [Title]

## Context
[What situation, friction, or gap prompted this guidance — the "before" state]

## Guidance
[The practice, pattern, or recommendation. Include code examples when they clarify the guidance.]

```[language]
// Example demonstrating the guidance
...
```

## Why This Matters
[Rationale and impact. What goes wrong when this isn't followed? What improves when it is?]

## When to Apply
- [Condition or situation where this applies]
- [Another condition]

## Examples
[Concrete before/after or usage examples showing the guidance in practice]

## Related
- [Link to related doc, issue, or PR]
```

---

## Vocabulary Capture Rules

### What Qualifies as a Domain Term

Include a term in `CONCEPTS.md` when **all** of these hold:
- It is an **entity, named process, or status concept** with project-specific meaning
- A new engineer would need it to follow conversations and code in this area
- It is not self-explanatory from its name alone

**Do not include:**
- Generic programming concepts (`interface`, `module`, `function`)
- File paths, class names, or function signatures (these drift)
- Configuration values, thresholds, or counts (these drift)
- Terms only meaningful in a single file or function

### Qualifying Bar

**Conservative at creation.** When bootstrapping `CONCEPTS.md`, prefer clear core domain nouns over borderline cases. A borderline term defers to a later run. Quantity follows quality — seed the important nouns, not every candidate.

**Normal bar for updates.** When `CONCEPTS.md` already exists, apply the standard qualifying criteria without extra conservatism.

### Entry Format

One-sentence definition. Add behavioral rules only when non-obvious.

Example:
```markdown
## BriefSystem
The pipeline that converts raw email inputs into structured client briefs. Runs as a background job triggered by inbound email receipt.
```

### Coherence Neighborhood

When adding or editing an entry, inspect its coherence neighborhood:
- Cluster siblings (entries in the same domain group)
- Terms it cross-references or that reference it

Within that neighborhood: fix glossary violations (implementation specifics — file paths, class names, current-config values) and refresh entries where the current learning's evidence shows drift. **Scope: neighborhood only, never a full-file audit.** If a neighbor would require investigation this learning didn't do, flag it for `/refreshing-knowledge` rather than editing on a guess.

### CONCEPTS.md Preamble (use when creating the file)

```markdown
# Concepts

> Shared domain vocabulary for this project — entities, named processes, and status concepts with project-specific meaning. Seeded with core domain vocabulary, then accretes as capturing-knowledge and refreshing-knowledge process learnings; direct edits are fine. Glossary only, not a spec or catch-all.
```
