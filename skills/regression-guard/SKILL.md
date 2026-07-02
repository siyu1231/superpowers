---
name: regression-guard
description: Called by subagent-driven-development — dispatches verification runner subagents per case, debug subagents on failure, and accumulates cases into the regression library
---

# Regression Guard

## Overview

**You are a coordinator, not an executor.** You never run CLI commands yourself. You never read implementation code yourself. You never fix anything yourself. Your job: write briefs, dispatch fresh subagents, read their reports, route verdicts.

**Core principle:** Fresh subagent per verification run + fresh subagent per fix = no judgment contamination.

## When to Use

This skill is **never invoked standalone.** It is called by other skills at specific gates:

| Caller | Context |
|--------|---------|
| `subagent-driven-development` | After implementation + reviews pass, human gate confirms |

**Project type gating:** Code projects only. Documentation, design specs, and other non-code artifacts skip regression verification. The calling skill determines project type.

## The Coordinator Loop

```
For EACH case:
  1. Write verification brief → .superpowers/regression-guard/verify-brief-<case.id>.md
  2. Dispatch verification runner (verification-runner-prompt.md)
     Runner: TOGGLE=0 vs expect.off, TOGGLE=1 vs expect.on → writes report, returns PASS | FAIL
  3. Route verdict:
     PASS → next case (or uncertainty verification if flagged)
     FAIL → write debug brief → dispatch debugger (debug-dispatch-prompt.md)
            → re-dispatch FRESH verification runner → repeat until PASS or 3 rounds
  4. uncertainty_verification == true?
     → dispatch clean-context verifier (clean-context-verifier-prompt.md)
     → FAIL → debug fixer → re-verify → loop (max 3 rounds)

ALL cases passed → write to regression/cases.json
```

## Process

**Triggered by:** `subagent-driven-development` after implementation + reviews pass + human gate confirmation.

**Input:** `regression-cases.json` from brainstorming for the current feature.

1. Validate case file against `case-schema.json`
2. For each case: dispatch verification runner → read report → PASS or debug-fix loop
3. Dispatch uncertainty verifier if case requires it
4. On all pass: write cases into `regression/cases.json`

**Library write (MANDATORY):**
1. Read existing library (create `{ "version": "1", "cases": [] }` if absent)
2. For each verified case: new `id` → append; existing `id` with changed expectations → update; same `id` and expectations → skip
3. Write and re-read to verify
4. **Write failure = BLOCKED.** Cases verified but NOT persisted — fix the write error, do not proceed without a written library.

## Dispatch Conventions

Every "dispatch" in this skill follows the same mechanics. The template files are Agent-call specifications — read them, fill them, call the Agent tool.

**For each dispatch:**

1. **Read the template file** — it contains the Agent parameters in `Subagent (type):` format
2. **Replace every `<PLACEHOLDER>`** with the actual value — never leave a placeholder literal
3. **Map to Agent tool call:**
   - `Subagent (<type>):` → `subagent_type`
   - `description:` → `description`
   - `model:` → `model` (pick from Model Selection table below)
   - `prompt: |` → `prompt`
4. **Brief files are the subagent's input** — always Write the brief file BEFORE dispatching. The subagent reads it via its prompt's `<BRIEF_FILE>` reference
5. **Report files are the subagent's output** — always Read the report file AFTER the subagent returns. The report IS your evidence; never "double check" by re-running commands yourself

Never paste case content, failure details, or implementation code directly into a dispatch prompt. Write them to the brief file and pass only the file path.

## Dispatching Sub-agents

### Verification Runner

For each case, dispatch a fresh subagent. Do NOT run CLI commands yourself.

1. **Write the brief:** `.superpowers/regression-guard/verify-brief-<case.id>.md`
   Content: `command`, `toggle`, `expect.on`, `expect.off` — the subagent reads this as its single source of truth
2. **Read the template:** [verification-runner-prompt.md](verification-runner-prompt.md)
   Replace `<case.id>`, `<case.name>`, `<BRIEF_FILE>`, `<REPORT_FILE>` with actual values
   Set `model:` from Model Selection table (cheapest tier — this is mechanical comparison)
3. **Dispatch via Agent tool** using the filled template parameters
4. **Read the report** at `<REPORT_FILE>` — this IS your evidence. Do NOT re-run CLI to "double check"
5. **Route:** PASS → next case / FAIL → dispatch debug fixer (below)

### Debug Fixer

Triggered when verification runner returns FAIL.

1. **Write the debug brief:** `.superpowers/regression-guard/debug-brief-<case.id>-round<N>.md`
   Content: the runner's full verification report (verbatim mismatches, actual vs expected), the complete case definition (command, toggle, execution_flow, both expect blocks), and the fix contract (re-run BOTH toggle states after fixing, confirm both match)
2. **Read the template:** [debug-dispatch-prompt.md](debug-dispatch-prompt.md)
   Replace `<BRIEF_FILE>`, `<REPORT_FILE>`, `<case.id>`, `<case.name>`, `<round_n>` with actual values
   Set `model:` from Model Selection table
3. **Dispatch via Agent tool** using the filled template parameters
4. **Verify the fix report** at `<REPORT_FILE>` has all three evidence items:
   - Root cause identified
   - Changes made (files, line ranges, rationale)
   - Fix verification: commands run (TOGGLE=0 AND TOGGLE=1), output observed, exit codes
   If any item missing → send the subagent back with "report missing <item>, re-run and complete the report"
5. **Re-dispatch a FRESH verification runner** for the same case. Never reuse the previous runner — its context knows about the failure. A fresh runner has naive eyes
6. **Still FAIL?** → back to step 1 with round N+1. Loop until PASS or 3 rounds total

### Clean-Context Verifier

Triggered when deterministic verification passes and `uncertainty_verification: true`.

1. **Write the brief:** `.superpowers/regression-guard/verify-brief-<case.id>.md`
   Content ONLY: `case.name`, `case.command`, `case.expect.on`, `case.execution_flow`
   MUST NOT include: implementation code, file paths, design docs, toggle mechanism details — the subagent must judge output as a naive user would
2. **Read the template:** [clean-context-verifier-prompt.md](clean-context-verifier-prompt.md)
   Replace `<BRIEF_FILE>`, `<case.id>`, `<case.name>` with actual values
   Set `model:` from Model Selection table (standard tier — this is a judgment task)
3. **Dispatch via Agent tool** using the filled template parameters
4. **Route:** PASS → continue / FAIL → dispatch debug fixer → re-dispatch FRESH clean-context verifier → loop until pass or 3 rounds

## Routing Debug Status

| Status | Action |
|--------|--------|
| **DONE** | Fix applied and verified. Confirm report has all three evidence items. Re-dispatch verification runner. |
| **DONE_WITH_CONCERNS** | Fix works but subagent flagged doubts. If concerns suggest case expectations may be wrong (not code) → escalate to human. Otherwise, re-dispatch runner and note concerns in progress ledger. |
| **BLOCKED** | Cannot determine root cause or fix. Assess: (1) insufficient context → provide more, re-dispatch same model; (2) needs more reasoning → re-dispatch stronger model; (3) case expectations may be wrong → escalate with "code produces X, case expects Y — which governs?"; (4) uncertain → escalate with full report. |

**3-round hard limit:** After 3 failed rounds on a single case, STOP. Report: which case, what was attempted, current state. If 3 rounds fail on 2+ cases: the problem may be in case expectations, not code.

## Model Selection

Use the least powerful model for each role. Always specify explicitly — omitted = inherits session's most expensive model.

| Subagent Role | Model Tier | Rationale |
|---------------|------------|-----------|
| **Verification runner** | Cheapest available | Mechanical: run commands, check strings. No judgment. |
| **Debug fixer (round 1)** | Standard | Multi-step: read code, trace logic, find root cause, apply fix. |
| **Debug fixer (round 2+)** | Standard or higher | Escalation: first fix didn't work. Upgrade if round 1 used cheap model. |
| **Clean-context verifier** | Standard | Judgment task: evaluate output quality, tone, completeness. |

## File Handoffs

Every artifact crossing the subagent boundary goes through files. Never paste bulk content into dispatch prompts.

| Artifact | Format | Path Pattern | Lifecycle |
|----------|--------|-------------|-----------|
| Verification brief | Markdown | `.superpowers/regression-guard/verify-brief-<case.id>.md` | Written before dispatch; deleted after case passes |
| Verification report | Markdown | `.superpowers/regression-guard/verify-report-<case.id>.md` | Written by runner; retained until case passes |
| Debug brief | Markdown | `.superpowers/regression-guard/debug-brief-<case.id>-round<N>.md` | Written before dispatch; deleted after case passes |
| Debug report | Markdown | `.superpowers/regression-guard/debug-report-<case.id>-round<N>.md` | Written by debugger; retained until case passes |
| Progress ledger | Plain text | `.superpowers/regression-guard/progress.md` | Persistent across session; survives compaction |

## Progress Ledger

Track at `.superpowers/regression-guard/progress.md`. Check at start. Trust the ledger over memory after compaction.

```
<case.id>: VERIFIED (rounds: <N>)
<case.id>: FIXED round <N> (commits <base7>..<head7>)
<case.id>: EXHAUSTED after 3 rounds — human decision required
Library: written (<N> added, <M> updated) → regression/cases.json
```

## Case Library Operations

**Path:** `regression/cases.json` (configurable)

**Schema validation:** MUST validate against `skills/regression-guard/case-schema.json` before processing.

**Write logic (Current Feature Verification only):**

```
For each verified case:
  case.id not in library → APPEND
  case.id exists, expect changed → UPDATE
  case.id exists, expect unchanged → SKIP
```

Append-only with in-place updates. Never delete. Never reorder.

**Case JSON format:**

```json
{
  "version": "1",
  "cases": [
    {
      "id": "email-valid-001",
      "name": "validate valid email format",
      "command": "./app validate --email test@example.com",
      "toggle": "FEATURE_EMAIL_VALIDATION",
      "trigger_condition": "input is a valid email format; toggle controls whether validation executes",
      "execution_flow": [
        "1. Parse --email argument",
        "2. Check FEATURE_EMAIL_VALIDATION toggle",
        "3. If ON: validate and return result",
        "4. If OFF: skip and return not available"
      ],
      "expect": {
        "on":  { "exit_code": 0, "stdout_contains": ["valid"], "stderr_empty": true },
        "off": { "exit_code": 1, "stdout_contains": ["not available"], "stderr_empty": true }
      },
      "uncertainty_verification": false
    }
  ]
}
```

## Comparison Rules

Implemented by the verification runner subagent. You (coordinator) read the runner's report — do not re-compare yourself.

| Rule | How |
|------|-----|
| `exit_code` | Exact integer match |
| `stdout_contains` | Every string must appear as a case-sensitive substring in stdout |
| `stderr_empty` | When `true`, stderr must be zero bytes |

## Red Flags — STOP

- Running a CLI command yourself instead of dispatching a verification runner
- Reading implementation code after a runner reports FAIL (route to debug subagent)
- Fixing code yourself instead of dispatching a debug subagent
- Re-running a command to "double check" the runner's report
- Skipping a case because it "always passes"
- Running fewer than both ON and OFF states
- Attempting round 4 of debug-fix on any case
- Invoking this skill standalone
- Fabricating regression cases
- Claiming "all pass" without every case verified by a fresh runner subagent
- Accepting a runner report missing actual output, or a debug report without fix verification evidence

## Integration

### Called By

| Skill | At This Point |
|-------|---------------|
| `subagent-driven-development` | After implementation + reviews pass, human gate confirms |

### Dispatches

| Subagent | Template | When |
|----------|----------|------|
| Verification runner | `verification-runner-prompt.md` | Every case — runs CLI, compares output |
| Debug fixer | `debug-dispatch-prompt.md` | Runner returns FAIL — finds root cause, fixes code |
| Clean-context verifier | `clean-context-verifier-prompt.md` | `uncertainty_verification: true` — judges output quality with zero implementation knowledge |

### Case File Contract

| File | Produced By | Consumed By |
|------|-------------|-------------|
| Feature `regression-cases.json` | `brainstorming` | `regression-guard` |
| `regression/cases.json` (library) | `regression-guard` (writes on pass) | `regression-guard` |

## Prompt Templates

- [verification-runner-prompt.md](verification-runner-prompt.md) — Dispatch verification runner
- [debug-dispatch-prompt.md](debug-dispatch-prompt.md) — Dispatch debug subagent on verification failure
- [clean-context-verifier-prompt.md](clean-context-verifier-prompt.md) — Dispatch clean-context verifier for uncertainty verification
