---
name: clean-context-verification
description: Called by regression-guard when uncertainty_verification=true — dispatches a subagent with clean context (only correct expectations, no implementation details) to verify LLM output, natural-language generation, or subjective quality
---

# Clean Context Verification

## Overview

Implementation-aware review is blind review. An agent that has read the source code, design docs, and implementation discussion sees what it expects to see — not what the output actually contains. The only honest quality judgment comes from a verifier who knows nothing about how the feature was built.

**Core principle: The verifier must be as naive as the end user. Any exposure to implementation details biases the judgment.**

**Violating the letter of this boundary is violating the spirit of independent verification.**

## The Iron Law

```
THE VERIFIER MUST HAVE ZERO KNOWLEDGE OF HOW THE FEATURE IS IMPLEMENTED
```

If the verifier has read source code, file paths, design documents, or dev discussions, it is not a verifier — it is a confirmer. Confirmers find what they expect. Verifiers find what is actually there.

## When to Use

This skill is **never invoked standalone.** It is dispatched by `regression-guard` when a case has `uncertainty_verification: true`:

| Caller | Trigger | Context |
|--------|---------|---------|
| `regression-guard` | Case `uncertainty_verification: true` | After deterministic verification passes (exit_code, stdout_contains, stderr_empty all match) |

**`uncertainty_verification: true` means:** the feature involves LLM calls, natural-language generation, or subjective quality judgments. Deterministic checks (exit codes, substring presence) cannot capture whether the output is actually good — only a naive human-like judgment can.

**Do NOT invoke this skill directly.** It has no standalone use case. If you think you need it outside the regression-guard loop, the calling skill's gate logic is broken — report that to your human partner.

## The Process

```
REGRESSION-GUARD (calling agent)
  │
  │  Deterministic checks passed
  │  uncertainty_verification: true
  │
  ▼
BUILD CLEAN CONTEXT ────────────────────────────────────────┐
  │                                                        │
  │  regression-guard strips ALL implementation details    │
  │  and provides ONLY:                                    │
  │    • case.name                                         │
  │    • case.command                                      │
  │    • case.expect.on                                    │
  │    • case.execution_flow                               │
  │                                                        │
  │  ABSOLUTELY EXCLUDED:                                  │
  │    • toggle name, env var                              │
  │    • implementation source code                        │
  │    • file paths                                        │
  │    • design documents                                  │
  │    • dev discussions                                   │
  │    • why the feature was built                         │
  │    • how the toggle gates the behavior                 │
  │                                                        │
  ▼                                                        │
CLEAN-CONTEXT VERIFICATION SUBAGENT                        │
  │                                                        │
  │  Execute: case.command                                 │
  │  Judge: Does output match expect.on behaviors?         │
  │  Judge: Does execution_flow match observed behavior?   │
  │  Judge: Would a naive user find this output correct?   │
  │  (Has NO implementation knowledge)                     │
  │                                                        │
  ▼                                                        │
VERDICT                                                     │
  │                                                        │
  ├─ PASS                                                  │
  │    │                                                   │
  │    └─ Report to regression-guard → Case passes         │
  │                                                        │
  └─ FAIL ─────────────────────────────────────┐           │
       │                                       │           │
       ▼                                       │           │
  REGRESSION-GUARD:                            │           │
       │                                       │           │
       │  NOW expose implementation            │           │
       │  to debug subagent                    │           │
       │  (systematic-debugging)              │           │
       │                                       │           │
       ▼                                       │           │
  DEBUG SUBAGENT:                              │           │
       │                                       │           │
       │  Has FULL implementation access       │           │
       │  Finds root cause                     │           │
       │  Applies fix                          │           │
       │                                       │           │
       ▼                                       │           │
  RE-VERIFY WITH CLEAN CONTEXT ────────────────────────────┘
       │
       │  New subagent, fresh clean context
       │  Max 3 rounds total
       │
       ▼
  AFTER ROUND 3: STOP
       │
       ▼
  Report to human partner:
    • What the case expects
    • What each round found
    • What was attempted
    • Current state
```

**Each case gets at most 3 verification rounds.** After 3 failed rounds on a single case, STOP and report to your human partner. Do not attempt round 4. Do not skip and continue with other cases.

## Clean Context Boundary

The boundary between the calling agent (regression-guard, which has full implementation knowledge) and the verification subagent (which must have none) is absolute. Crossing it in either direction invalidates the verification.

### What IS Provided to the Verifier

| Field | Source | Purpose |
|-------|--------|---------|
| `case.name` | Regression case | What the feature is called (e.g., "validate valid email format") |
| `case.command` | Regression case | Exact command to execute |
| `case.expect.on` | Regression case | Expected output when the feature is active |
| `case.execution_flow` | Regression case | Ordered steps describing what the command should do |

### What MUST NOT Be Provided to the Verifier

| Information | Why It Must Be Excluded |
|-------------|------------------------|
| Implementation source code | Creates expectation bias — verifier sees what the code intends, not what the output shows |
| File paths (source or test) | Reveals implementation structure, hints at how things work |
| Design documents | Contain reasoning that biases judgment toward "it follows the design" |
| Development discussions | Expose tradeoffs and decisions that frame the output as "reasonable given constraints" |
| Toggle name or environment variable | Reveals implementation gating — the verifier judges output, not toggle mechanics |
| Toggle ON/OFF state | The verifier should not know which toggle state produced the output; it judges only whether the output matches the described behavior |
| Why the feature was built | Irrelevant to whether the output is correct |
| Who built it or when | Creates social bias — output from respected contributors gets softer judgment |
| Known limitations or edge cases | Primes the verifier to excuse those specific issues |

**The rule is simple: if it is not `name`, `command`, `expect.on`, or `execution_flow`, it does not cross the boundary.**

## Why Clean Context Matters

An implementation-aware agent reviewing output suffers from expectation bias. It knows what the code is supposed to do, so it interprets ambiguous output as correct. It knows which edge cases were discussed, so it looks for those and nothing else. It knows the constraints the developer faced, so it rationalizes shortcomings as "best effort given the circumstances."

A naive user has none of this context. They type a command and see output. If the output is confusing, incomplete, misleading, or wrong, they experience that directly. They don't think "well, the implementation approach makes this hard" — they think "this doesn't work."

The clean-context verifier replicates the user's experience. It catches:

- **Hallucinated confidence:** Output that sounds authoritative but is factually wrong
- **Vague hand-waving:** Output that uses plausible generalities instead of specific answers
- **Missing steps:** The execution_flow says step 3 should happen, but the output jumps from step 2 to step 4
- **Wrong tone or format:** Output that technically contains the right words but presents them in a broken or misleading way
- **Silent failures:** Output that reports success but didn't actually do what the execution_flow describes
- **Prompt-leak artifacts:** Output that contains internal instructions, system prompts, or implementation details that should never reach the user

These are exactly the failures that implementation-aware review misses — because the reviewer already knows what the output was supposed to say.

## Verdict Format

Every verdict must include specific evidence. Vague judgments are not verdicts — they are impressions.

### PASS

```
PASS

EVIDENCE:
  • Command executed: <case.command>
  • Exit code: <N> (matches expect.on)
  • Output contains all expected behaviors from execution_flow:
    - Step 1 (<description>): <observed evidence in output>
    - Step 2 (<description>): <observed evidence in output>
    - Step 3 (<description>): <observed evidence in output>
  • Output is coherent, complete, and would satisfy a naive user
  • No misleading claims, hallucinations, or prompt-leak artifacts detected

VERDICT: Output matches all specified behaviors. Feature works as described.
```

### FAIL

```
FAIL

ISSUES:
  • <Issue 1>: <specific description of what is wrong>
  • <Issue 2>: <specific description of what is wrong>

EXPECTED BEHAVIOR (from execution_flow):
  - Step <N>: <what should have happened>

ACTUAL OUTPUT:
  """
  <verbatim command output>
  """

WHY A USER WOULD BE CONFUSED:
  <explanation from the perspective of someone with no implementation knowledge>

VERDICT: Output does not match specified behaviors. Feature needs repair.
```

**A FAIL verdict MUST include the actual output verbatim.** Without it, the debug subagent cannot diagnose the problem. A FAIL verdict without actual output evidence is not a verdict — it is a feeling.

**Do not soften FAIL verdicts.** "Mostly works but..." is a FAIL. "Close enough" is a FAIL. "A user might figure it out" is a FAIL. If the output does not match the execution_flow precisely, it fails.

## Retry Limit

```
Round 1: Clean-context subagent → FAIL → debug subagent → fix → re-verify
Round 2: Clean-context subagent → FAIL → debug subagent → fix → re-verify
Round 3: Clean-context subagent → FAIL → debug subagent → fix → re-verify
Round 4: DOES NOT EXIST. STOP. Report to human partner.
```

**Each round uses a fresh subagent with clean context.** The verifier from round 2 must not know what the round 1 verifier saw. Each round starts from zero implementation knowledge. This prevents the verifier from developing its own expectation bias across rounds.

After 3 failed rounds, STOP the entire verification for that case. Do not skip it and continue. Do not "try one more approach." Report:

- Which case failed
- What each round found (verdicts from all 3 rounds)
- What was fixed in each round
- Current state (what is still wrong)
- Ask your human partner for a decision

**If 3 rounds fail on 2+ different cases that both use `uncertainty_verification`:** question whether the case definitions themselves are correct. The issue may be in how `execution_flow` or `expect.on` was specified during brainstorming, not in the implementation.

## Rationalization Prevention

| Excuse | Reality |
|--------|---------|
| "The verifier just needs to know the toggle name" | Any implementation detail is a leak. The toggle name reveals gating structure. |
| "It's faster if I don't spawn a subagent" | Speed is not the goal. Independent judgment is. |
| "I can simulate clean context in my head" | You cannot unknow what you know. Contamination is irreversible. |
| "The output is clearly wrong without clean context" | Then it should have failed deterministic checks. If it didn't, clean-context verification is needed to find why. |
| "One round is enough — the fix is obvious" | The fix that was obvious with implementation knowledge may not fix what a user sees. |
| "The verifier saw the output last round, it's already biased" | That's why each round uses a FRESH subagent. No carryover. |
| "This case is simple enough to skip uncertainty verification" | If `uncertainty_verification: true` is set, it's there for a reason. Run it. |
| "The deterministic checks passed, how bad can it be?" | Deterministic checks catch substring presence, not output quality. Very bad output can still contain all required substrings. |

## Red Flags — STOP

If you catch yourself thinking any of these, STOP. You are about to violate the clean context boundary:

- **Passing implementation details** to the verifier "so it understands what to look for"
- **Skipping the subagent** and judging output yourself
- **Reusing the same subagent** across rounds instead of spawning fresh
- **Softening a FAIL verdict** because "it's close"
- **Providing toggle mechanics** (env var name, ON/OFF state) to the verifier
- **Explaining why the output is the way it is** before the verifier judges it
- **Accepting a verdict without verbatim output** in a FAIL report
- **Attempting round 4** after 3 failures
- **Invoking this skill standalone** instead of through regression-guard
- **Skipping uncertainty verification** when `uncertainty_verification: true` is set
- **Letting the verifier see** the expect.off block, source paths, or any implementation artifact
- **Claiming "clean context" while passing** the execution_flow with implementation commentary embedded

**All of these mean: STOP. Re-read the Iron Law. A contaminated verifier is not a verifier. Spawn fresh or report honestly.**

## Integration

### Called By

| Skill | At This Point |
|-------|---------------|
| `regression-guard` | After deterministic verification passes on a case with `uncertainty_verification: true` |

### Dispatches

| Skill | When |
|-------|------|
| (none directly) | This skill produces a verdict; regression-guard dispatches `systematic-debugging` on FAIL |

### Context Contract

| Item | Provided By | Must Cross Boundary? |
|------|-------------|---------------------|
| `case.name` | regression-guard | YES — the verifier needs to know what feature to judge |
| `case.command` | regression-guard | YES — the verifier must execute this exact command |
| `case.expect.on` | regression-guard | YES — the verifier judges output against these expectations |
| `case.execution_flow` | regression-guard | YES — the verifier checks that each step actually happens |
| `case.toggle` | regression-guard | NO — implementation detail |
| `case.expect.off` | regression-guard | NO — the verifier judges only the active output |
| `case.trigger_condition` | regression-guard | NO — boundary specification, not output expectation |
| Source code | codebase | NO — absolute contamination |
| Design documents | project | NO — biases toward design intent |
| Any file path | codebase | NO — reveals structure |

### The Boundary is the Calling Agent's Responsibility

`regression-guard` owns the clean-context boundary. It must strip all implementation details before dispatching the verification subagent. If the verifier receives tainted context, the fault is in regression-guard, not in this skill — but the verification is invalid either way.

When dispatching: spawn a brand-new subagent whose system prompt and context contain ONLY the four allowed fields. Do not forward the conversation. Do not include prior messages. Do not attach files. The subagent's first and only knowledge of the feature comes from `name`, `command`, `expect.on`, and `execution_flow`.

## The Bottom Line

**An implementation-aware reviewer cannot judge what a user will see.**

Strip everything. Spawn fresh. Judge naively. Report honestly.

After 3 rounds, stop and ask your human partner — because at that point, either the implementation is wrong, or the case expectations are wrong, and the agent cannot fairly decide which.
