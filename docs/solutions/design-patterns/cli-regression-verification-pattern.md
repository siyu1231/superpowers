---
title: CLI Regression Verification Pattern
date: 2026-06-30
category: design-patterns
module: skills/regression-guard
problem_type: design_pattern
applies_when:
  - Building an AI-agent-driven development workflow that needs automated verification
  - Designing regression testing that doesn't rely on unit test frameworks
  - Adding feature toggles for agent-verifiable behavior comparison
  - Integrating verification into an existing skill pipeline
severity: info
tags:
  - regression-guard
  - clean-context-verification
  - feature-toggle
  - cli
  - verification
  - skill-design
---

# CLI Regression Verification Pattern

## Context

AI-driven development spends most of its time on manual testing verification. Existing Superpowers skills (brainstorming → writing-plans → SDD → finishing) lacked a systematic verification phase. The project needed a way to:

1. Verify features at the CLI level (not just code-level unit tests)
2. Compare behavior with features ON vs OFF to confirm toggle-controlled behavior
3. Accumulate regression cases so new features don't break old ones
4. Handle both deterministic verification (CLI output matching) and uncertainty verification (LLM output quality)

## Guidance

### Core Architecture: Two New Skills + Three Enhancements

The verification system consists of two new skills and three enhancements to existing skills:

**New skills:**
- `regression-guard` — CLI verification engine called at two gated checkpoints
- `clean-context-verification` — Uncertainty verification dispatched by regression-guard

**Enhanced skills:**
- `brainstorming` — Produces regression case JSON alongside design spec
- `subagent-driven-development` — Verification gate after all tasks complete
- `finishing-a-development-branch` — Full regression gate before merge/PR

### Regression Case Format (JSON)

Cases are structured, machine-parseable, and include ON/OFF comparison:

```json
{
  "version": "1",
  "cases": [
    {
      "id": "email-valid-001",
      "name": "验证合法邮箱格式",
      "command": "./app validate --email test@example.com",
      "toggle": "FEATURE_EMAIL_VALIDATION",
      "trigger_condition": "输入为合法邮箱格式，开关控制是否执行校验",
      "execution_flow": ["1. 解析参数", "2. 检查开关", "3. 执行/跳过校验"],
      "expect": {
        "on":  { "exit_code": 0, "stdout_contains": ["valid"], "stderr_empty": true },
        "off": { "exit_code": 1, "stdout_contains": ["not available"], "stderr_empty": true }
      },
      "uncertainty_verification": false
    }
  ]
}
```

Key design decisions:
- **`stdout_contains` over exact match** — output may include timestamps, colors, etc.
- **`id` as unique key** — same id updates existing entry, new id appends
- **`uncertainty_verification` flag** — marks cases needing clean-context LLM judgment

### Feature Toggle: Environment Variables

Use environment variables as the toggle mechanism: `FEATURE_X=1` enables, `FEATURE_X=0` disables. This is zero-dependency, language-agnostic, and naturally isolated per process. Verification runs the same command twice — once with the toggle ON, once OFF — and compares both outputs against expectations.

### Project Type Gating

CLI requirement and regression verification apply only to **code projects** (primary deliverable is an executable program or service). Non-code projects (documentation, design specs, library-only packages) skip verification entirely. The heuristic: "does the project's primary deliverable run as a process?"

### Two-Mode Verification

The regression-guard skill operates in two modes:
1. **Current Feature Verification** — triggered after SDD implementation. Reads the feature's regression-cases.json. On pass: writes cases into the library. On failure: dispatches debug subagent, re-verifies (max 3 rounds).
2. **Full Regression** — triggered during finishing. Reads the accumulated library. Verifies all cases. Does NOT write to library.

### Failure Recovery Loop

On verification failure: dispatch a debug subagent (systematic-debugging) → fix the code → re-run verification → loop (max 3 rounds per case). After 3 failed rounds: pause and report to human partner.

### Clean Context Verification

For features involving LLM output, natural language, or subjective quality: the verifier subagent receives only the expected behavior description and CLI command — NO implementation code, file paths, or design docs. This simulates a naive end user judgment, avoiding implementation bias.

## Why This Matters

Without this pattern:
- **Manual verification is ad-hoc** — no record of what was tested, can't re-run
- **Regressions accumulate silently** — each new feature risks breaking prior work
- **Toggle behavior goes untested** — only tested in the ON state
- **Agent verification is unreliable** — implementation-aware agents rationalize broken output

With this pattern:
- **Every feature ships with automated CLI verification cases**
- **Case library accumulates over time** — full regression catches regressions
- **ON/OFF comparison proves toggle isolation**
- **Clean-context verification catches what implementation-aware review misses**

## When to Apply

- When designing a new Superpowers skill that produces verifiable output
- When evaluating whether a feature is ready for merge/PR
- When the design spec includes an Agent Verifiability section with regression cases
- When building an AI-agent-friendly development workflow that needs automated CLI verification gate

## Examples

### Brainstorming produces regression cases

In the design spec's Agent Verifiability section, each feature defines its CLI entry point and regression cases. The brainstorming skill produces a `regression-cases.json` alongside `design.md`.

### SDD invokes verification gate

After all implementation tasks complete and final review passes, SDD checks: is this a code project? If yes, present the human gate; if confirmed, run regression-guard in current-feature mode.

### Finishing invokes full regression

Before merge or PR, finishing checks: does `regression/cases.json` exist? If yes, run regression-guard in full-regression mode — all accumulated cases must pass.

## Related

- Design spec: `docs/superpowers/specs/2026-06-30-cli-regression-verification-design.md`
- Implementation plan: `docs/superpowers/plans/2026-06-30-cli-regression-verification.md`
