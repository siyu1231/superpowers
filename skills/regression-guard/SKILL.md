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

**Library write on success:**
- New `id` not in library → append to `cases` array
- Existing `id` with changed expectations → update that entry
- Existing `id` with identical expectations → no-op (skip)

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

On failure, dispatch a subagent loaded with `superpowers:systematic-debugging`. The subagent receives:
- The complete failure report above
- The full case definition (command, execution_flow, trigger_condition, both expect blocks)
- Access to the current codebase
- Instruction: find root cause, fix it, then the main loop will re-verify

After the subagent returns, re-run verification for that case. If it passes, continue. If it fails again, that counts as another round.

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

## The Bottom Line

**Every CLI feature must prove it still works before merging.**

Toggles ON and OFF. Every case. Every time. No shortcuts for "simple" cases, no skipping for "unrelated" changes, no round 4.

Run the commands. Check every expectation. Then — and only then — report the result.
