---
name: regression-guard
description: Called by subagent-driven-development and finishing-a-development-branch — dispatches verification runner subagents per case, debug subagents on failure, and accumulates cases into the regression library
---

# Regression Guard

## Overview

**You are a coordinator, not an executor.** You never run CLI commands yourself. You never read implementation code yourself. You never fix anything yourself. Your job is to dispatch fresh subagents for every step and route their reports.

This is identical to how SDD works: read the plan → dispatch implementer → read report → dispatch reviewer → read report → next task. Here: read cases → dispatch verification runner → read report → if FAIL, dispatch debug fixer → re-dispatch verification runner → read report → next case.

**Why this matters:** When you run a CLI command yourself and see wrong output, you immediately know why it's wrong and how to fix it. You cannot then objectively judge a fix — you'll rationalize the output you just made the code produce. By dispatching every step, you stay clean.

**Core principle:** Fresh subagent per verification run + fresh subagent per fix = no judgment contamination.

**Violating the letter of this process is violating the spirit of regression prevention.**

## The Iron Law

```
NO MERGE WITHOUT FRESH REGRESSION EXECUTION AGAINST THE CASE LIBRARY
```

If every case hasn't been verified by a fresh subagent in this session, you cannot claim the branch is clean.

## When to Use

This skill is **never invoked standalone.** It is called by other skills at specific gates:

| Caller | Context | Mode |
|--------|---------|------|
| `subagent-driven-development` | After implementation + reviews pass, human gate confirms | Current Feature Verification |
| `finishing-a-development-branch` | Before merge/PR, as final quality gate | Full Regression |

**Project type gating:** This skill runs ONLY for code projects — projects whose primary artifact is an executable program or service. Documentation projects, design specs, and other non-code artifacts skip regression verification entirely. The calling skill determines project type before invoking this skill.

**Do NOT invoke this skill directly.** It has no standalone use case. If you think you need it outside these two gates, the calling skill's gate logic is broken — report that to your human partner.

## The Coordinator Loop

```
READ cases from source
  │
  │  Source = regression-cases.json (current feature)
  │       OR regression/cases.json (full library)
  │
  ▼
For EACH case ────────────────────────────────────────────────┐
  │                                                           │
  │  Step 1: Write verification brief                         │
  │  (case command + toggle + both expect blocks)              │
  │                                                           │
  ▼                                                           │
  Step 2: Dispatch verification runner subagent               │
  │   (verification-runner-prompt.md)                         │
  │   Subagent: runs CLI with TOGGLE=0, compares expect.off   │
  │            runs CLI with TOGGLE=1, compares expect.on     │
  │            writes structured report                       │
  │            returns: PASS | FAIL                           │
  │                                                           │
  ▼                                                           │
  Runner verdict?                                             │
  │                                                           │
  ├─ PASS ──────────────────────────────────────────┐        │
  │                                                  │        │
  └─ FAIL ──→ Record mismatches from report          │        │
               │                                     │        │
               ▼                                     │        │
             Dispatch debug fixer subagent           │        │
             (debug-dispatch-prompt.md)              │        │
             Brief includes: runner's report,        │        │
             failed case definition, fix contract    │        │
               │                                     │        │
               ▼                                     │        │
             Debug returns:                          │        │
             DONE | DONE_WITH_CONCERNS | BLOCKED     │        │
               │                                     │        │
               ▼                                     │        │
             BLOCKED? ──→ escalate to human partner  │        │
               │                                     │        │
               ▼                                     │        │
             Re-dispatch verification runner         │        │
             (fresh subagent, same case)             │        │
               │                                     │        │
               ▼                                     │        │
             Runner: PASS now?                       │        │
               ├─ Yes → continue                     │        │
               └─ No  → back to debug (max 3 rounds) │        │
               │                                     │        │
               ▼                                     │        │
  Deterministic PASS ────────────────────────────────┘        │
  │                                                           │
  ▼                                                           │
uncertainty_verification == true?                             │
  │                                                           │
  YES ──→ Dispatch clean-context verifier                     │
  │       (clean-context-verifier-prompt.md)                  │
  │       ├─ PASS → continue                                  │
  │       └─ FAIL → debug fixer → re-verify → loop            │
  │                                                           │
  NO                                                          │
  │                                                           │
  ▼                                                           │
Case PASSED ──────────────────────────────────────────────────┘
  │
  ▼
ALL cases passed?
  │
  ▼
Current Feature Verification mode?
  ├─ YES → Write/update cases in regression/cases.json
  │         ├─ Success → DONE (cases persisted)
  │         └─ Failure → BLOCKED (verification passed but cases lost)
  └─ NO  → (Full Regression mode: no write)
```

**Each case gets at most 3 debug-fix rounds.** After 3 rounds on a single case, STOP and report to your human partner.

**Note on the diagram:** `TOGGLE` is a placeholder for `<case.toggle>`. When `case.toggle` is `FEATURE_X`, the verification runner executes `FEATURE_X=0 <command>` and `FEATURE_X=1 <command>`.

## Two Modes

### Mode 1: Current Feature Verification

**Triggered by:** `subagent-driven-development` after implementation + reviews pass + human gate confirmation.

**Input:** The `regression-cases.json` file produced during brainstorming for the current feature.

**Behavior:**
1. Validate the case file against `case-schema.json`
2. For each case: dispatch verification runner → read report → PASS or debug-fix loop
3. Dispatch uncertainty verifier if case requires it
4. On all pass: write cases into `regression/cases.json`
5. Report to human partner

**Library write on success — MANDATORY:**

After all cases pass, write to `regression/cases.json`:

1. Read the existing library (create `{ "version": "1", "cases": [] }` if absent)
2. For each verified case: new `id` → append; existing `id` with changed expectations → update; same `id` and expectations → skip
3. Write and re-read to verify the write
4. **If write fails:** BLOCKED. Cases were verified but NOT persisted. Fix the write error — do not proceed without a written library.

### Mode 2: Full Regression

**Triggered by:** `finishing-a-development-branch` before merge/PR.

**Input:** The accumulated case library at `regression/cases.json`.

**Behavior:**
1. Validate the library against `case-schema.json`
2. For each case: dispatch verification runner → read report → PASS or debug-fix loop
3. **Do NOT write to the case library** — this mode only verifies
4. Report results

**If `regression/cases.json` does not exist:** skip and report. Do not fabricate cases.

## File Handoffs

Every artifact crossing the subagent boundary goes through files. You never paste bulk content into dispatch prompts — it stays in your context forever.

| Artifact | Format | Path Pattern | Lifecycle |
|----------|--------|-------------|-----------|
| Verification brief | Markdown | `.superpowers/regression-guard/verify-brief-<case.id>.md` | Written before dispatching runner; deleted after case passes |
| Verification report | Markdown | `.superpowers/regression-guard/verify-report-<case.id>.md` | Written by runner; retained until case passes |
| Debug brief | Markdown | `.superpowers/regression-guard/debug-brief-<case.id>-round<N>.md` | Written before dispatching debug; deleted after case passes |
| Debug report | Markdown | `.superpowers/regression-guard/debug-report-<case.id>-round<N>.md` | Written by debugger; retained until case passes |
| Progress ledger | Plain text | `.superpowers/regression-guard/progress.md` | Persistent across session; survives compaction |

## Dispatching the Verification Runner

For each case, dispatch a fresh subagent. **You never run the CLI commands yourself.**

**Step 1: Write the verification brief.** Create `.superpowers/regression-guard/verify-brief-<case.id>.md`. Content: the case's `command`, `toggle`, `expect.on`, `expect.off`. This is the subagent's single source of truth.

**Step 2: Dispatch the subagent** using the template at [verification-runner-prompt.md](verification-runner-prompt.md). Fill placeholders: `<case.id>`, `<case.name>`, `<BRIEF_FILE>`, `<REPORT_FILE>`. Use the cheapest model tier — this is mechanical execution and string comparison, not judgment.

**Step 3: Read the report.** The runner returns PASS or FAIL with a structured report at `<REPORT_FILE>`. Read the report file — do NOT re-run the CLI command yourself to "double check." The runner's report IS your evidence.

**Step 4: Route the verdict.**

- **PASS** — Record in progress ledger, continue to next case (or uncertainty verification if needed)
- **FAIL** — The report contains exact mismatches (expected vs actual for each field). Route these to a debug subagent.

## Dispatching the Debug Fixer

Triggered when a verification runner returns FAIL.

<HARD-GATE>
You MUST dispatch a fresh debug subagent. You are a coordinator — you never read the implementation code, you never run the CLI commands yourself. The runner's report tells you WHAT failed. The debug subagent figures out WHY and fixes it.
</HARD-GATE>

**Step 1: Write the debug brief.** Create `.superpowers/regression-guard/debug-brief-<case.id>-round<N>.md`. Content: the runner's full verification report (mismatches, actual output, expected values), the full case definition, and the fix contract.

**Step 2: Dispatch** using [debug-dispatch-prompt.md](debug-dispatch-prompt.md). Fill all placeholders. Choose model per Model Selection below.

**Step 3: Verify the fix.** After the subagent returns, confirm the report contains fix verification evidence (commands run, output observed, exit codes for both toggle states). If missing, send it back.

**Step 4: Re-dispatch a FRESH verification runner** for the same case. Do NOT use the previous runner — its context knows about the failure. A fresh runner approaches the case with naive eyes, just like the first time.

**Step 5: If runner still returns FAIL, that's round 2.** Loop until PASS or 3 rounds total.

## Handling Debug Subagent Status

Debug subagents report one of three statuses:

**DONE:** Fix applied and verified. Confirm the report has all three evidence items. Re-dispatch verification runner to confirm.

**DONE_WITH_CONCERNS:** Fix works but subagent flagged doubts. Read concerns. If they suggest the case expectations may be wrong (not the code), escalate to human partner. Otherwise, re-dispatch verification runner and note concerns in progress ledger.

**BLOCKED:** Cannot determine root cause or fix. Assess:
1. Context insufficient → provide more context, re-dispatch with same model
2. Task needs more reasoning → re-dispatch with stronger model
3. Case expectations may be wrong → escalate to human: "code produces X, case expects Y — which governs?"
4. Uncertain → escalate with full report

## Uncertainty Verification Dispatch

When deterministic verification passes and `uncertainty_verification: true`, dispatch a clean-context verifier.

**Step 1: Write the verification brief.** Create `.superpowers/regression-guard/verify-brief-<case.id>.md`. Content (and NOTHING more): `case.name`, `case.command`, `case.expect.on`, `case.execution_flow`. **MUST NOT contain:** implementation code, file paths, design docs, toggle mechanism details.

**Step 2: Dispatch** using [clean-context-verifier-prompt.md](clean-context-verifier-prompt.md). Subagent receives ONLY the brief file.

**Step 3: Handle verdict.**

- **PASS** — Record in progress ledger, continue
- **FAIL** — Dispatch debug fixer (the failure report includes verbatim actual output → route to debug). After fix, re-dispatch a FRESH clean-context verifier. Loop until pass or 3 rounds.

## The 3-Round Hard Limit

```
Round 1: Runner FAIL → debug fixer → runner PASS? → PASS
Round 2: Still FAIL → debug fixer → runner PASS? → PASS  
Round 3: Still FAIL → debug fixer → runner PASS? → PASS
Round 4: DOES NOT EXIST. STOP. Report to human partner.
```

After 3 failed rounds on a single case, STOP. Report: which case, what was attempted each round, current state, ask for decision. **If 3 rounds fail on 2+ cases:** the problem may be in case expectations, not code.

## Model Selection

Use the least powerful model for each role:

| Subagent Role | Model Tier | Rationale |
|---------------|------------|-----------|
| **Verification runner** | Cheapest available | Mechanical: run commands, check strings. No judgment, no code reading. |
| **Debug fixer (round 1)** | Standard (e.g., sonnet) | Multi-step: read code, trace logic, find root cause, apply fix. |
| **Debug fixer (round 2+)** | Standard or higher | Escalation: first fix didn't work. Upgrade if round 1 used cheap model. |
| **Clean-context verifier** | Standard (e.g., sonnet) | Judgment task: evaluate output quality, tone, completeness. |

**Always specify the model explicitly.** Omitted = inherits session's most expensive model.

## Progress Ledger

Track at `.superpowers/regression-guard/progress.md`. Check at start. Trust the ledger over memory after compaction.

```
# After case passes:
<case.id>: VERIFIED (rounds: <N>)

# After each fix:
<case.id>: FIXED round <N> (commits <base7>..<head7>)

# After exhaustion:
<case.id>: EXHAUSTED after 3 rounds — human decision required

# After library write:
Library: written (<N> added, <M> updated) → regression/cases.json
```

## Case Library Operations

### Path: `regression/cases.json` (configurable)

### Schema Validation

MUST validate against `skills/regression-guard/case-schema.json` before processing any case file.

### Write Logic (Current Feature Verification only)

```
For each verified case:
  case.id not in library → APPEND
  case.id exists, expect changed → UPDATE
  case.id exists, expect unchanged → SKIP
```

Append-only with in-place updates. Never delete. Never reorder. **Write failure = BLOCKED.**

### Case JSON Format

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

These are implemented by the verification runner subagent. You (the coordinator) read the runner's report — you do not re-compare yourself.

| Rule | How |
|------|-----|
| `exit_code` | Exact integer match |
| `stdout_contains` | Every string must appear as a case-sensitive substring in stdout |
| `stderr_empty` | When `true`, stderr must be zero bytes |

## Project Type Gating

Regression verification applies **only to code projects** (primary artifact is an executable or service). Skip for documentation, design specs, libraries without CLI, or plugins without runnable commands. The calling skill determines project type.

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "This case is too simple to fail" | Simple cases catch the most regressions. Dispatch the runner. |
| "I already tested this manually" | Manual memory ≠ fresh execution. Dispatch the runner. |
| "The change couldn't affect this case" | You don't know until the runner runs it. |
| "3 rounds is enough to skip one case" | No case is exempt. |
| "Just skip this one and continue" | Skipping one = skipping all. |
| "I'll add the case to the library later" | Later never happens. Write on pass. |
| "The toggle is always on in this environment" | Then OFF will fail honestly. That's evidence. |
| "This project doesn't need regression" | If it's a code project with CLI, it needs regression. |
| "I can run this command myself — faster" | You are a coordinator. Running CLI yourself defeats the purpose. |

## Red Flags — STOP

- **Running a CLI command yourself** instead of dispatching a verification runner (you are a coordinator, not an executor)
- **Reading implementation code** after a runner reports FAIL (route to debug subagent instead)
- **Fixing code yourself** instead of dispatching a debug subagent
- **Re-running a command to "double check"** the runner's report (the report IS your evidence)
- **Skipping a case** because it "always passes"
- **Running fewer than both** ON and OFF states
- **Accepting a runner report** that's missing actual output
- **Accepting a debug report** without fix verification evidence
- **Attempting round 4** of debug-fix on any case
- **Writing to the case library** during Full Regression mode
- **Invoking this skill standalone**
- **Fabricating regression cases**
- **Skipping the progress ledger**
- **Claiming "all pass"** without every case verified by a fresh runner subagent

## Integration

### Called By

| Skill | At This Point |
|-------|---------------|
| `subagent-driven-development` | After implementation + reviews pass, human gate confirms |
| `finishing-a-development-branch` | Before merge/PR |

### Dispatches

| Subagent | Template | When |
|----------|----------|------|
| Verification runner | `verification-runner-prompt.md` | Every case — runs CLI, compares output |
| Debug fixer | `debug-dispatch-prompt.md` | Runner returns FAIL — finds root cause, fixes code |
| Clean-context verifier | `clean-context-verifier-prompt.md` | `uncertainty_verification: true` — judges output quality with zero implementation knowledge |

### Case File Contract

| File | Produced By | Consumed By |
|------|-------------|-------------|
| Feature `regression-cases.json` | `brainstorming` | `regression-guard` (Current Feature Verification) |
| `regression/cases.json` (library) | `regression-guard` (writes on pass) | `regression-guard` (Full Regression) |

## Prompt Templates

- [verification-runner-prompt.md](verification-runner-prompt.md) — Dispatch verification runner to execute CLI commands
- [debug-dispatch-prompt.md](debug-dispatch-prompt.md) — Dispatch debug subagent on verification failure
- [clean-context-verifier-prompt.md](clean-context-verifier-prompt.md) — Dispatch clean-context verifier for uncertainty verification

## The Bottom Line

**You are a coordinator. You never run CLI commands. You never read implementation code. You never fix anything.**

Your job: write briefs, dispatch fresh subagents, read their reports, route verdicts. The subagents do the work. You keep the ledger and make the decisions.

Every case, every time. No shortcuts. Toggles ON and OFF.
