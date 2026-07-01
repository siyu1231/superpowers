---
name: regression-guard
description: Called by subagent-driven-development and finishing-a-development-branch — runs CLI-level regression cases with ON/OFF toggle comparison, dispatches debug subagents on failure, and accumulates cases into the regression library
---

# Regression Guard

## Overview

Manual re-testing of CLI commands after every change is the largest source of wasted agent time and undetected regressions. A feature that "passed yesterday" means nothing — only fresh CLI execution against expected output counts.

**Core principle:** Every feature that touches a CLI must ship with automated regression cases. ON/OFF toggle comparison proves the feature is gated, not just "always on by accident."

**Violating the letter of this process is violating the spirit of regression prevention.**

## The Iron Law

```
NO MERGE WITHOUT FRESH REGRESSION EXECUTION AGAINST THE CASE LIBRARY
```

If you haven't run every regression case in this session, you cannot claim the branch is clean.

## The Second Iron Law

```
NO INLINE FIXES ON VERIFICATION FAILURE — ALWAYS DISPATCH A FRESH SUBAGENT
```

When a CLI command does not produce the expected output, you are the **judge**, not the **fixer**. Your job is to detect the mismatch, report it precisely, and dispatch a fresh subagent to investigate and fix. Fixing it yourself contaminates your judgment — you will rationalize the output you just made the code produce.

**No exceptions:**
- "This is a one-line fix" → dispatch
- "I can see exactly what's wrong" → dispatch
- "It's faster to fix it myself" → dispatch
- "Just this once" → dispatch

A fresh subagent reads the code with naive eyes. You cannot — you already know what the output "should" be.

## When to Use

This skill is **never invoked standalone.** It is called by other skills at specific gates:

| Caller | Context | Mode |
|--------|---------|------|
| `subagent-driven-development` | After implementation + reviews pass, human gate confirms | Current Feature Verification |
| `finishing-a-development-branch` | Before merge/PR, as final quality gate | Full Regression |

**Project type gating:** This skill runs ONLY for code projects — projects whose primary artifact is an executable program or service. Documentation projects, design specs, and other non-code artifacts skip regression verification entirely. The calling skill determines project type before invoking this skill.

**Do NOT invoke this skill directly.** It has no standalone use case. If you think you need it outside these two gates, the calling skill's gate logic is broken — report that to your human partner.

## The Verification Loop

```
READ cases from source
  │
  │  Source = regression-cases.json (current feature)
  │       OR regression/cases.json (full library)
  │
  ▼
For EACH case ──────────────────────────────────────────────┐
  │                                                         │
  ▼                                                         │
┌─ DETERMINISTIC VERIFICATION ──────────────────────────┐   │
│                                                       │   │
│  1. Execute: TOGGLE=0 <command>                       │   │
│     Compare against expect.off                        │   │
│     ├─ Match → continue                               │   │
│     └─ Mismatch → record FAILURE ──────┐              │   │
│                                        │              │   │
│  2. Execute: TOGGLE=1 <command>        │              │   │
│     Compare against expect.on          │              │   │
│     ├─ Match → continue                │              │   │
│     └─ Mismatch → record FAILURE ──────┤              │   │
│                                        │              │   │
└────────────────────────────────────────┼──────────────┘   │
  │                                      │                  │
  ▼                                      ▼                  │
Any FAILURES? ── YES ──► Dispatch debug subagent            │
  │                      (systematic-debugging)              │
  │                      Fix → re-verify this case           │
  │                      Loop until pass or 3 rounds         │
  │                      ────────────────────────────────────┤
  │                                                         │
  NO                                                        │
  │                                                         │
  ▼                                                         │
uncertainty_verification == true?                           │
  │                                                         │
  YES ──► Dispatch clean-context-verification subagent      │
  │       ├─ Pass → continue                                │
  │       └─ Fail → debug subagent → re-verify              │
  │              Loop until pass or 3 rounds                │
  │                                                         │
  NO                                                        │
  │                                                         │
  ▼                                                         │
Case PASSED ────────────────────────────────────────────────┘
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

**Each case gets at most 3 debug-fix-retry rounds.** After 3 failed rounds on a single case, STOP and report to your human partner. Do not attempt round 4.

**Note on the diagram:** `TOGGLE=0` and `TOGGLE=1` in the verification loop are placeholders. Each case defines its own toggle environment variable via the `toggle` field. When `case.toggle` is `FEATURE_X`, execute `FEATURE_X=0 <command>` and `FEATURE_X=1 <command>`.

## Two Modes

### Mode 1: Current Feature Verification

**Triggered by:** `subagent-driven-development` after implementation + reviews pass + human gate confirmation.

**Input:** The `regression-cases.json` file produced during brainstorming for the current feature. This file lives alongside the design doc and contains only the cases for this feature.

**Behavior:**
1. Read the feature's `regression-cases.json`
2. Run the verification loop on every case
3. On failure: debug-fix loop (max 3 rounds per case)
4. On all pass: write/update cases into the shared library at `regression/cases.json`
5. Report results to your human partner

**Library write on success — MANDATORY, not optional:**

After all cases pass, write to `regression/cases.json`:

1. Read the existing library (create `{ "version": "1", "cases": [] }` if absent)
2. For each verified case, apply the merge rule:
   - New `id` not in library → append to `cases` array
   - Existing `id` with changed expectations → update that entry
   - Existing `id` with identical expectations → no-op (skip)
3. Write the updated library back to `regression/cases.json`
4. Verify the write: re-read the file and confirm the case count is correct

**If the write fails for any reason** (file not writable, JSON invalid, case count mismatch):
```
BLOCKED: Cannot write to regression/cases.json

<specific error>

Verification passed but cases were NOT persisted. The next full regression
will not include these cases. Fix the write error and re-run the write,
or manually append the verified cases.
```

Do NOT proceed past this point with unwritten cases. Do NOT claim
"verification complete" if the library was not updated. A feature without
persisted regression cases is a feature that will regress silently.

### Mode 2: Full Regression

**Triggered by:** `finishing-a-development-branch` before merge/PR.

**Input:** The accumulated case library at `regression/cases.json`.

**Behavior:**
1. Read `regression/cases.json`
2. Run the verification loop on EVERY case in the library
3. On failure: debug-fix loop (max 3 rounds per case)
4. **Do NOT write to the case library** — this mode only verifies, it never accumulates
5. Report results to your human partner

**If `regression/cases.json` does not exist:** there is no accumulated regression library. Skip full regression and report that fact. Do not fabricate cases.

## Failure Handling

### Failure Report Format

When a case fails verification, report with this structure:

```
CASE: <case.id> — <case.name>
STATE: ON (TOGGLE=1) | OFF (TOGGLE=0)
COMMAND: <case.command>
EXPECTED: <expect.on or expect.off content>
ACTUAL:
  exit_code: <N>
  stdout: <actual stdout>
  stderr: <actual stderr>
MISMATCH: <specific field that differs and how>
```

### Debug Dispatch

<HARD-GATE>
When a case fails verification, you MUST dispatch a fresh subagent to fix it. Fixing the failure yourself is a violation of the Second Iron Law. You are the judge — the subagent is the fixer. These roles do not merge.
</HARD-GATE>

**This is not optional.** On every verification failure, you WILL:

1. Write the debug brief file
2. Dispatch a fresh subagent using the template
3. Wait for its report
4. Verify the report contains fix evidence
5. Re-run the case yourself to confirm

If you catch yourself thinking "this is simple enough to fix inline," re-read the Second Iron Law. The subagent exists for a reason: you cannot judge output you just made the code produce.

On failure, DO NOT paste failure details directly into the subagent prompt. Use file handoff — the same pattern that keeps SDD dispatches tight:

**Step 1: Write the debug brief file.** Content: the failure report, the full case definition, and the fix contract. Path: `.superpowers/regression-guard/debug-brief-<case.id>-round<N>.md`. This file is the subagent's single source of truth — exact values appear only here.

**Step 2: Compose the dispatch.** The subagent prompt contains: (1) one line on which case failed; (2) the brief path, introduced as "read this first — it is your debugging target"; (3) the report file path and report contract; (4) the global constraints.

**Step 3: Dispatch the subagent** loaded with `superpowers:systematic-debugging`, using the dispatch template at [debug-dispatch-prompt.md](debug-dispatch-prompt.md). Fill all placeholders: `<case.id>`, `<case.name>`, `<round_n>`, `<BRIEF_FILE>`, `<REPORT_FILE>`. Choose the model per Model Selection below.

**Step 4: Verify the fix.** After the subagent returns, confirm the report contains: (a) the verification commands run, (b) the output observed, (c) matching exit codes for both toggle states. If any of these are missing, send the subagent back to complete the report. Only after all three are present, re-run the case verification yourself.

**Step 5: Clean up the brief.** The brief file is scratch — its content is already in the report. Do not leave stale briefs for the next round.

### Handling Debug Subagent Status

Debug subagents report one of three statuses. Handle each appropriately:

**DONE:** The subagent found the root cause, applied a fix, and verified it. Confirm the report contains fix verification evidence (commands run, output observed, exit codes for both toggle states). If all three are present, re-run the case verification. If the report is incomplete, send the subagent back to finish it — a fix without verification evidence is not a fix.

**DONE_WITH_CONCERNS:** The fix works but the subagent flagged doubts (fragility, side effects, incomplete understanding). Read the concerns before re-verifying. If the concerns suggest the case itself may be wrong (not the code), escalate to your human partner — "the fix passes but there are concerns worth reviewing." Otherwise, re-verify and note the concerns in the progress ledger.

**BLOCKED:** The subagent cannot determine the root cause or cannot fix it without touching code it doesn't understand. Do NOT re-dispatch the same subagent. Assess the blocker:
1. Context problem → provide more context (e.g., additional file paths, design doc references) and re-dispatch with the same model
2. Task requires more reasoning → re-dispatch with a more capable model
3. Case expectations may be wrong → escalate to human partner: "the code produces X, the case expects Y — which governs?"
4. If uncertain which → escalate to human partner with the subagent's full report

**Never** accept a silent retry without changing something. If the subagent said it's stuck, re-dispatching with identical context produces identical results.

### Uncertainty Verification Dispatch

When deterministic verification passes and `uncertainty_verification: true`, the calling agent must dispatch a clean-context verifier. This is a separate subagent with its own dispatch contract — DO NOT merge it into the debug dispatch.

**Step 1: Write the verification brief.** Create `.superpowers/regression-guard/verify-brief-<case.id>.md`. Content (and NOTHING more):

- `case.name` — what behavior is expected, in plain language
- `case.command` — the exact CLI command to run
- `case.expect.on` — expected exit code and stdout content
- `case.execution_flow` — the steps the feature should follow

**CRITICAL: What MUST NOT be in the brief:**
- Implementation source code or file paths
- Design documents, specs, or development discussion
- The toggle mechanism details (the command string already includes the toggle)
- Any information about HOW the feature is built

If any of these leak into the brief, the verifier's judgment is contaminated. Write the brief, then re-read it — if a single file path or code snippet appears, delete it.

**Step 2: Dispatch the subagent** using the template at [clean-context-verifier-prompt.md](clean-context-verifier-prompt.md). Fill the placeholders: `<case.id>`, `<case.name>`, `<BRIEF_FILE>`. The subagent receives ONLY the brief file — no other context about the project.

**Step 3: Handle the verdict.**

**PASS:** The verifier confirmed output quality. The case passes uncertainty verification. Mark in the progress ledger and continue.

**FAIL:** The verifier found concrete issues in the output. The failure report includes verbatim actual output. NOW — and only now — expose implementation details to a debug subagent:

1. Write a debug brief containing: the verifier's failure report (issues + verbatim output), the implementation code paths, and the fix contract
2. Dispatch a debug subagent using [debug-dispatch-prompt.md](debug-dispatch-prompt.md)
3. After fix, re-run uncertainty verification with a FRESH clean-context subagent (the previous verifier's context is contaminated)
4. Loop until pass or 3 rounds total (verifier → debug → re-verify = 1 round)

**Step 4: Clean up.** Delete the verification brief after the case passes. The report from the clean-context verifier is preserved in the debug report chain.

### The 3-Round Hard Limit

```
Round 1: Failure → debug subagent → fix → re-verify
Round 2: Still failure → debug subagent → fix → re-verify
Round 3: Still failure → debug subagent → fix → re-verify
Round 4: DOES NOT EXIST. STOP. Report to human partner.
```

After 3 failed rounds on a single case, STOP the entire verification. Do not skip the case and continue with others. Do not "try one more thing." Report:
- Which case failed
- What was attempted in each round
- Current state (what's still wrong)
- Ask your human partner for a decision

**If 3 rounds fail on 2+ different cases:** question whether the verification approach itself is sound. The problem may be in how cases are defined, not in the code.

## Model Selection

Use the least powerful model that can handle each subagent role:

| Subagent Role | Model Tier | Rationale |
|---------------|------------|-----------|
| **Debug (deterministic failure)** | Standard (e.g., sonnet) | Multi-step investigation: read code, trace logic, identify root cause, apply fix. Mid-tier floor — cheap models take 2-3× turns and miss root causes. |
| **Debug (round 2+, same case)** | Standard or higher | Escalation: the first fix didn't work. Use same tier or upgrade if round 1 was on a cheap model. |
| **Clean-context verifier** | Standard (e.g., sonnet) | Judgment task: evaluate output quality, tone, completeness. Not mechanical — requires independent assessment. |

If the case is purely mechanical (one-line fix, misconfigured flag, wrong exit code), a cheap model is acceptable for round 1.

**Always specify the model explicitly when dispatching.** An omitted model inherits your session's model — silently the most expensive — defeating this section.

## Progress Ledger

Conversation memory does not survive compaction. Track verification state in a ledger at `.superpowers/regression-guard/progress.md`.

**At start of verification:**
```bash
cat "$(git rev-parse --show-toplevel)/.superpowers/regression-guard/progress.md" 2>/dev/null || echo "NO_LEDGER"
```
Cases listed there as verified are DONE — do not re-verify them. Resume at the first case not marked verified.

**After a case passes (both deterministic and uncertainty):** append one line:
```
<case.id>: VERIFIED (rounds: <N>, toggle ON+OFF match, <uncertainty status>)
```

**After debug subagent fix lands:** append the fix commit range:
```
<case.id>: FIXED round <N> (commits <base7>..<head7>)
```

**After 3-round exhaustion on a case:** append:
```
<case.id>: EXHAUSTED after 3 rounds — human decision required
```

**After library write (current-feature mode):** append:
```
Library: written (<N> cases added, <M> updated) → regression/cases.json
```

The ledger is your recovery map after compaction. Trust it over your own recollection. `git clean -fdx` will destroy it (it's git-ignored scratch); if that happens, recover by re-running all cases.

## File Handoffs

Every artifact crossing the subagent boundary goes through files, not pasted text. Pasting bulk content into your dispatch pollutes your context permanently — every later turn re-reads it.

| Artifact | Format | Path Pattern | Lifecycle |
|----------|--------|-------------|-----------|
| Debug brief | Markdown | `.superpowers/regression-guard/debug-brief-<case.id>-round<N>.md` | Written before dispatch; deleted after verification completes for that case |
| Debug report | Markdown | `.superpowers/regression-guard/debug-report-<case.id>-round<N>.md` | Written by subagent; retained until case passes (reference for re-verification) |
| Progress ledger | Plain text | `.superpowers/regression-guard/progress.md` | Persistent across the verification session; survives compaction |

## Case Library Operations

### Library Path

Default: `regression/cases.json` at the project root.

If the project uses a different path, it is specified during brainstorming and carried through the case definition.

### Schema Validation

The case library and feature-level case files MUST validate against `skills/regression-guard/case-schema.json`. Before processing any case file, validate it against the schema. If validation fails, report the specific validation errors and stop — do not execute commands from an invalid case file.

### Write Logic (Current Feature Verification only)

```
For each case in the just-verified feature set:
  If case.id exists in library:
    If case.expect changed → UPDATE the entry
    If case.expect unchanged → SKIP (no-op)
  Else:
    APPEND case to library's cases array
```

Never delete cases from the library. Never reorder cases. Append-only with in-place updates for changed expectations.

### Case JSON Format

Each case file (current feature or accumulated library) uses this structure:

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
        "3. If ON: execute email format validation, return valid",
        "4. If OFF: skip validation, return not available"
      ],
      "expect": {
        "on": {
          "exit_code": 0,
          "stdout_contains": ["valid"],
          "stderr_empty": true
        },
        "off": {
          "exit_code": 1,
          "stdout_contains": ["not available"],
          "stderr_empty": true
        }
      },
      "uncertainty_verification": false
    }
  ]
}
```

**Feature-level file** (from brainstorming) may also include a top-level `feature` and `toggle` field. The library strips these — only `version` and `cases` are stored.

**Field reference:**

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Unique key. Same `id` = same case across updates. |
| `name` | Yes | Human-readable description of what is being verified. |
| `command` | Yes | CLI command to execute. Shell-parseable string. |
| `toggle` | Yes | Environment variable name that gates the feature (e.g., `FEATURE_EMAIL_VALIDATION`). |
| `trigger_condition` | Yes | Exact condition that triggers the behavior; boundary specification. |
| `execution_flow` | Yes | Ordered execution steps showing intermediate states. |
| `expect.on` | Yes | Expected result when toggle is ON (TOGGLE=1). |
| `expect.off` | Yes | Expected result when toggle is OFF (TOGGLE=0). |
| `uncertainty_verification` | Yes | Whether clean-context verification subagent is needed. |

## Comparison Rules

Three comparison operators are available. All must match for a verification to pass.

### `exit_code`

Exact integer match. `0` means success, non-zero means the exit code the command should produce.

```
Expected: { "exit_code": 0 }
Actual: exit code 0 → MATCH
Actual: exit code 1 → MISMATCH
```

### `stdout_contains`

Array of strings. **Every** string must appear as a substring in stdout. Order does not matter. This is substring matching, not exact matching — stdout may contain timestamps, color codes, or other dynamic content.

```
Expected: { "stdout_contains": ["valid", "accepted"] }
Actual stdout: "Email is valid and accepted" → MATCH (both substrings found)
Actual stdout: "Email is valid" → MISMATCH ("accepted" not found)
Actual stdout: "VALID" → MISMATCH ("valid" not found — case-sensitive substring)
```

**Case-sensitive.** Use exact strings. If the command supports `--plain` or `--no-color` to strip ANSI codes, prefer that in the command definition.

When no stdout content is expected, use an empty array: `"stdout_contains": []`.

### `stderr_empty`

Boolean. When `true`, stderr must be completely empty (zero bytes). When `false`, stderr content is ignored.

```
Expected: { "stderr_empty": true }
Actual: stderr is "" → MATCH
Actual: stderr is "warning: deprecation" → MISMATCH
```

## Project Type Gating

Regression verification applies **only to code projects.** A code project is one whose primary artifact is an executable program, service, library with a CLI, or similar runnable software.

**Skip regression verification entirely when:**
- The project produces documentation, design specs, or prose
- The project has no CLI entry points
- The project is a plugin that does not expose runnable commands

The calling skill (`subagent-driven-development` or `finishing-a-development-branch`) is responsible for determining project type before invoking this skill. If this skill is invoked for a non-code project, report the error to your human partner — a gate upstream failed.

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "This case is too simple to fail" | Simple cases catch the most regressions. Run it. |
| "I already tested this manually" | Manual memory ≠ fresh execution. Run the command. |
| "The change couldn't affect this case" | You don't know what you broke until you run it. |
| "3 rounds is enough to skip one case" | The limit is per case. No case is exempt. |
| "Just skip this one and continue" | Skipping one = skipping all. The loop doesn't continue past failure. |
| "I'll add the case to the library later" | Later never happens. Write on pass or report why not. |
| "The toggle is always on in this environment" | Then OFF verification will fail honestly. That's evidence, not a reason to skip. |
| "This project doesn't need regression" | If it's a code project with CLI, it needs regression. No exceptions. |
| "I can fix this myself — it's faster" | You are the judge, not the fixer. Dispatch a subagent. Fixing inline contaminates your judgment. |
| "This is just a one-line change" | One line or one hundred — dispatch. A fresh subagent reads the code with naive eyes. |
| "I already know what's wrong" | Knowing and verifying are different. Let the subagent confirm and fix. |

## Red Flags — STOP

If you catch yourself thinking any of these, STOP. You are about to violate the regression guard:

- **Skipping a case** because it "always passes"
- **Running fewer than both** ON and OFF for a case
- **Accepting a partial match** on stdout_contains
- **Rounding exit codes** (0 = success, anything else = "non-zero" but ignoring the specific value)
- **"The output looks right"** without checking every stdout_contains substring
- **Skipping stderr check** when stderr_empty is true
- **Attempting round 4** of debug-fix on any case
- **Writing to the case library** during Full Regression mode
- **Invoking this skill standalone** instead of through the proper gate
- **Fabricating regression cases** when `regression/cases.json` doesn't exist for Full Regression
- **Running verification on a non-code project** (upstream gate failed — report it)
- **Claiming "all pass"** without running every case fresh in this session
- **Pasting failure details into the dispatch prompt** instead of writing a debug brief file (file handoff is not optional — your context is finite)
- **Accepting a debug subagent report** without verifying it contains: fix verification commands run, output observed, exit codes for both toggle states
- **Skipping the progress ledger** (after compaction you will not remember what passed)
- **Fixing the failure yourself** instead of dispatching a debug subagent (you are the judge, not the fixer — Second Iron Law)
- **Thinking "this is too simple to need a subagent"** (every fix gets a fresh subagent, no exceptions)
- **Writing the fix and then verifying it yourself** (you cannot judge output you just made the code produce — this is why the subagent exists)

**All of these mean: STOP. Re-read the Iron Law. Run every case, check every expectation, or report honestly that you cannot.**

## Integration

### Called By

| Skill | At This Point |
|-------|---------------|
| `subagent-driven-development` | After implementation + spec review + quality review pass, after human gate confirms "enter verification phase" |
| `finishing-a-development-branch` | Before merge/PR, as the full regression gate |

### Dispatches

| Skill | When |
|-------|------|
| `systematic-debugging` | Any case fails deterministic verification — dispatched as subagent to find root cause and fix |
| `clean-context-verification` | Case has `uncertainty_verification: true` — dispatched as subagent for independent judgment |

### Case File Contract

| File | Produced By | Consumed By |
|------|-------------|-------------|
| Feature `regression-cases.json` | `brainstorming` | `regression-guard` (Current Feature Verification mode) |
| `regression/cases.json` (library) | `regression-guard` (writes on pass) | `regression-guard` (Full Regression mode) |

## Prompt Templates

- [debug-dispatch-prompt.md](debug-dispatch-prompt.md) — Dispatch debug subagent on verification failure
- [clean-context-verifier-prompt.md](clean-context-verifier-prompt.md) — Dispatch clean-context verifier for uncertainty verification

## The Bottom Line

**Every CLI feature must prove it still works before merging.**

Toggles ON and OFF. Every case. Every time. No shortcuts for "simple" cases, no skipping for "unrelated" changes, no round 4.

Run the commands. Check every expectation. Then — and only then — report the result.
