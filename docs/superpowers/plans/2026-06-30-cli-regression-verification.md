# CLI 回归验证系统 — 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建 CLI 命令级回归验证系统（regression-guard + clean-context-verification 两个新技能），并增强 brainstorming / SDD / finishing 三个技能，形成完整的「设计 → 开发 → 验证 → 积累 → 全量回归」闭环。

**Architecture:** 两个新技能、三个现有技能增强。regression-guard 是核心验证引擎，被 SDD（当前功能验证）和 finishing（全量回归）两个节点调用。clean-context-verification 是独立的不确定性验证技能，由 regression-guard 按需派发。brainstorming 增强产出回归案例 JSON，SDD 增强插入验证闸门，finishing 增强插入全量回归闸门。

**Tech Stack:** Markdown 技能文件（SKILL.md），JSON 回归案例格式。零外部依赖。

## Global Constraints

- 零第三方依赖（Superpowers 插件设计原则）
- 技能文件遵循现有格式：frontmatter (name/description) + Markdown body
- 回归案例库路径：默认 `regression/cases.json`，可由项目配置
- 验证仅适用于代码项目；非代码项目（文档、设计规格等）跳过整个验证流程
- CLI 入口强制要求仅适用于代码项目
- 每条案例最多 3 轮 debug 修复重试
- 人工闸门：SDD 后不自动进入验证，必须用户确认

---

### Task 1: 新建 regression-guard 技能

**Files:**
- Create: `skills/regression-guard/SKILL.md`

**Interfaces:**
- Consumes: 回归案例 JSON 文件（本次产生的 `regression-cases.json` 或案例库 `regression/cases.json`），环境变量开关
- Produces: 验证结果报告（通过 / 失败详情），更新后的案例库文件

- [ ] **Step 1: 创建技能文件，包含 frontmatter 和核心流程**

写入 `skills/regression-guard/SKILL.md`：

```markdown
---
name: regression-guard
description: Use when verification is needed for a code project — runs CLI-level regression cases with ON/OFF toggle comparison, dispatches debug subagents on failure, and accumulates cases into the regression library
---

# Regression Guard

## Overview

Verify features through CLI command execution with environment-variable toggle comparison. On failure, dispatch a debug subagent to fix the issue, then re-verify. Accumulate verified cases into a persistent regression library.

**Core principle:** Every feature must prove itself at the CLI level, with the toggle ON and OFF, before it is considered done.

**Violating the letter of the rules is violating the spirit of the rules.**

## When to Use

**Called by:**
- `subagent-driven-development` — after all tasks complete, to verify the current feature
- `finishing-a-development-branch` — before merge/PR, to run full regression

**Not used standalone** — it is an engine invoked by other skills at gated checkpoints.

## The Verification Loop

```
Read case file (current-feature JSON or full case library)
        │
        ▼
  For each case ────────────────────────────────────────┐
        │                                                 │
        ▼                                                 │
  ┌ Deterministic Verification ──────────────┐            │
  │                                           │            │
  │  1. TOGGLE=0 execute command              │            │
  │     Compare against expect.off             │            │
  │     ├─ Match → continue                    │           │
  │     └─ Mismatch → record failure ────────┐ │            │
  │                                           │ │            │
  │  2. TOGGLE=1 execute command              │ │            │
  │     Compare against expect.on              │ │            │
  │     ├─ Match → continue                    │ │            │
  │     └─ Mismatch → record failure ────────┘ │            │
  │                                           │            │
  └───────────────────────────────────────────┘            │
        │                                                 │
        ▼                                                 │
  Any failures? ── Yes ──→ Dispatch debug subagent        │
        │          (systematic-debugging)                   │
        │          Fix → re-run deterministic for this case  │
        │          Loop until pass (max 3 rounds) ─────────┤
        No                                                 │
        │                                                 │
        ▼                                                 │
  uncertainty_verification == true?                        │
        │                                                 │
       Yes                                                │
        │                                                 │
        ▼                                                 │
  ┌ Uncertainty Verification ──────────────────┐          │
  │  Dispatch subagent                          │          │
  │  (clean-context-verification)               │          │
  │  Clean context + correct expectations        │          │
  │  ├─ Pass → continue                         │          │
  │  └─ Fail → debug subagent fix               │          │
  │           → re-run uncertainty verification  │          │
  │           → loop until pass (max 3 rounds)   │          │
  └─────────────────────────────────────────────┘          │
        │                                                 │
        ▼                                                 │
  All passed → Write to case library ─────────────────────┘
  (current-feature mode only; full-regression mode skips this)
```

## Two Modes

### Mode 1: Current Feature Verification

Triggered by `subagent-driven-development` after implementation completes and user confirms the verification gate.

**Input:** The `regression-cases.json` produced by brainstorming alongside the design spec. Contains cases for the feature just built.

**Behavior:**
1. Read the feature's regression case file
2. Run the verification loop (deterministic + uncertainty as marked)
3. On failure: dispatch subagent with `systematic-debugging` skill to fix the code, then re-run the failed case
4. On all-pass: write cases into the case library:
   - New `id` → append to `regression/cases.json`
   - Existing `id` with changed expectations → update the entry
5. Report: cases added, cases updated, total library size

### Mode 2: Full Regression

Triggered by `finishing-a-development-branch` before merge/PR.

**Input:** The full case library at `regression/cases.json`.

**Behavior:**
1. Read all cases from the library
2. Run the verification loop on every case
3. On failure: dispatch debug subagent to fix, re-run
4. Do NOT write to the case library (no new cases being added)
5. Report: passed / failed / fixed count

## Failure Handling

### Deterministic Failure Report

When a CLI command's output does not match expectations, report:

```
Case: <case.name> (<case.id>)
Toggle state: ON / OFF
Command: <case.command>
Expected (stdout_contains): <case.expect.on.stdout_contains>
Actual stdout: <actual>
Exit code: expected <expected>, got <actual>
```

### Debug Dispatch

When dispatching the debug subagent, provide:
- The full failure report above
- The current implementation code (file paths and content)
- Instruction: "Load systematic-debugging skill. Fix the code so this CLI command produces the expected output."

The debug subagent must NOT modify the regression case — only the implementation code.

### Retry Limit

Maximum 3 debug rounds per case. If a case still fails after 3 rounds:
- Stop and report to the human partner
- Show: the case, all 3 failure reports, what changed in each fix attempt
- Wait for human decision (adjust case expectations, adjust implementation approach, or skip)

## Case Library Operations

### Path

Default: `regression/cases.json` in the project root. Configurable via project settings.

### Write Logic (current-feature mode only)

After all cases for the current feature pass:

1. Read existing library (create empty if not exists)
2. For each case from the current feature:
   - If `id` already exists in library → update the entry (feature changed existing behavior)
   - If `id` is new → append to library
3. Write the updated library back

### Library Format

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
      "execution_flow": [
        "1. 解析 --email 参数",
        "2. 检查 FEATURE_EMAIL_VALIDATION",
        "3. 若开启：执行校验，返回 valid",
        "4. 若关闭：跳过校验，返回 not available"
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

## Comparison Rules

### stdout_contains

Check that every string in `stdout_contains` appears in the command's stdout. Order does not matter. Substring match.

### exit_code

Exact integer match.

### stderr_empty

When `true`, stderr must be empty (zero-length). When `false`, stderr is not checked (may contain diagnostics, logs, etc.).

## Project Type Gating

This skill runs ONLY for code projects. Non-code projects (documentation, design specs, etc.) skip verification entirely. The calling skill (SDD or finishing) checks the project type before invoking regression-guard.

## Red Flags

**Never:**
- Modify regression cases to match actual output (cases define expectations — fix the code, not the case)
- Skip the OFF-toggle verification (both ON and OFF must be checked)
- Proceed past a failing case without attempting debug fix
- Exceed 3 debug rounds without human approval
- Write to the case library in full-regression mode
- Run verification for non-code projects

## Integration

**Called by:**
- `subagent-driven-development` — current feature verification gate
- `finishing-a-development-branch` — full regression gate

**Dispatches:**
- `systematic-debugging` — on verification failure
- `clean-context-verification` — on uncertainty verification

**Related skills:**
- `brainstorming` — produces the regression case JSON
- `test-driven-development` — implementers use TDD; regression cases are the acceptance layer above unit tests
- `verification-before-completion` — evidence before claims principle
```

- [ ] **Step 2: 验证技能文件结构符合现有模式**

检查：
- frontmatter 包含 `name` 和 `description`
- 技能内容包含 Overview、核心流程、行为规则、Red Flags
- 遵循现有技能的写作风格和术语（"human partner"、"core principle"等）

- [ ] **Step 3: 提交**

```bash
git add skills/regression-guard/SKILL.md
git commit -m "feat: add regression-guard skill — CLI-level verification with toggle comparison"
```

---

### Task 2: 新建 clean-context-verification 技能

**Files:**
- Create: `skills/clean-context-verification/SKILL.md`

**Interfaces:**
- Consumes: 案例的 `name`、`command`、`expect.on`、`execution_flow`（干净上下文，不含实现代码）
- Produces: PASS / FAIL 判决，失败时附带具体问题描述

- [ ] **Step 1: 创建技能文件**

写入 `skills/clean-context-verification/SKILL.md`：

```markdown
---
name: clean-context-verification
description: Use when a feature involves LLM output, natural-language generation, or subjective quality — dispatches a subagent with clean context (only correct expectations, no implementation details) to verify output quality
---

# Clean Context Verification

## Overview

Verify features whose correctness cannot be judged deterministically — LLM output, natural language generation, subjective quality. A subagent receives ONLY the expected behavior description and the CLI command, with zero exposure to implementation code, design docs, or development discussion. It judges output as a real user would.

**Core principle:** The verifier must be as naive as the end user. Any exposure to implementation details biases the judgment.

## When to Use

Triggered by `regression-guard` when a case has `uncertainty_verification: true`. This flag is set during the brainstorming phase.

Not invoked standalone — it is a verification tool called by regression-guard.

## The Process

```
Calling agent (regression-guard)
        │
        │  Provides: case.name + case.command + case.expect.on + case.execution_flow
        │  (NO implementation code, file paths, design docs, or dev discussion)
        │
        ▼
  Subagent (clean-context-verification)
        │
        │  1. Execute command (TOGGLE=1)
        │  2. Compare output against expect.on
        │  3. Judge quality: format, tone, completeness, correctness
        │  4. Return: PASS with evidence, or FAIL with specific issues
        │
        ▼
  Verdict ──→ PASS (with evidence)
        │
       FAIL (with specific issues)
        │
        ▼
  Calling agent receives failure report
        │
        │  NOW exposes implementation details to debug subagent
        │  Provides: failure description + implementation code
        │
        ▼
  Debug subagent (systematic-debugging) fixes the code
        │
        ▼
  Re-run clean-context-verification
        │
        ▼
  Loop until pass (max 3 rounds)
```

## Clean Context Boundary

### What the Verifier Receives

| Field | Source | Purpose |
|-------|--------|---------|
| `case.name` | Regression case | What behavior is expected, in plain language |
| `case.command` | Regression case | The exact CLI command to run |
| `case.expect.on` | Regression case | Expected exit code and stdout content |
| `case.execution_flow` | Regression case | The steps the feature should follow |

### What the Verifier MUST NOT Receive

- Implementation source code
- File paths in the project
- Design documents or specs
- Development discussion or commit history
- Toggle mechanism details (the verifier just runs the command with TOGGLE=1)
- Any information about HOW the feature is built

### Why Clean Context Matters

An agent that sees the implementation will rationalize its output. It will say "this matches what the code intends" rather than "this matches what the user needs." Clean context forces the verifier to judge output purely as a consumer, catching issues that implementation-aware review misses:
- Confusing error messages the developer thought were clear
- Missing output the developer forgot to include
- Broken formatting the developer's eyes skip over
- Responses that are technically correct but unhelpful

## Verdict Format

### PASS

```
Verdict: PASS
Evidence:
- Command executed: <command>
- Exit code: <code> (expected <expected>)
- stdout contains all expected strings: <list>
- Quality assessment: <brief natural-language judgment>
```

### FAIL

```
Verdict: FAIL
Issues found:
1. <specific issue>: <what was expected vs what was observed>
2. <specific issue>: ...
Command output (actual):
<stdout>
```

## Retry Limit

Maximum 3 verification rounds (verifier → debug fix → re-verify). After 3 rounds without passing, report to the human partner with:
- The case description
- All 3 failure reports and corresponding fix attempts
- A recommendation: adjust expectations, redesign the feature, or manual intervention required

## Red Flags

**Never:**
- Expose implementation code to the verifier subagent
- Expose file paths, design docs, or dev discussion to the verifier
- Accept a PASS verdict without specific evidence
- Skip re-verification after a debug fix
- Exceed 3 rounds without human approval

## Integration

**Called by:** `regression-guard` (during both current-feature verification and full regression)

**Dispatches:** `systematic-debugging` — on verification failure (dispatched by regression-guard, not by this skill directly)

**Related to:** `regression-guard` — the parent skill that dispatches this verification
```

- [ ] **Step 2: 验证技能文件完整性**

- [ ] **Step 3: 提交**

```bash
git add skills/clean-context-verification/SKILL.md
git commit -m "feat: add clean-context-verification skill — clean-context uncertainty verification"
```

---

### Task 3: 增强 brainstorming 技能 — CLI 要求和回归案例产出

**Files:**
- Modify: `skills/brainstorming/SKILL.md`

**Interfaces:**
- Consumes: 无（现有技能增强）
- Produces: 更新后的 brainstorming 流程，在 Agent Verifiability 部分强制 CLI 入口并产出回归案例 JSON

- [ ] **Step 1: 在 Key Principles 之后、Visual Companion 之前插入 Agent Verifiability 章节**

定位到 `## Key Principles` 部分末尾（约第 141 行后），`## Visual Companion` 之前，插入：

```markdown
## Agent Verifiability (required for every feature in code projects)

Every feature module in a design spec MUST include an **Agent Verifiability** section. The AI fills it autonomously based on the design discussion; the human reviews and corrects during spec review.

### CLI Entry Point (mandatory for code projects)

- Command to invoke this feature: `<command> <args>`
- **For code projects, every feature must have a CLI entry point.** No CLI entry = not agent-verifiable = design incomplete.
- Missing CLI entry for a code project is a blocking issue at design review.
- Non-code projects (documentation, design specs, libraries without executables) are exempt from this requirement.

### Project Type Determination

During brainstorming, determine whether the target is a code project:
- **Code project** → must define CLI entries → must produce regression cases
- **Non-code project** → CLI and regression cases are not required

Heuristic: does the project's primary deliverable include an executable program or service? If yes, it's a code project.

### Deterministic Verification: Three Dimensions

| Dimension | Question | Example |
|-----------|----------|---------|
| **Trigger condition** | What exact conditions activate this feature? What is the boundary between "triggers" and "does not trigger"? | `FEATURE_X=1` AND input is valid email → trigger; empty email → no trigger; invalid email → no trigger |
| **Execution flow** | What steps execute, in what order? What is the intermediate state after each step? | ① validate input → ② lookup record → ③ apply transform → ④ write result → ⑤ return |
| **Success result** | What is the observable output and side effects when the feature completes successfully? | exit=0, stdout contains token, DB `last_login` updated, no passwords in output |

### Feature Toggle
- Toggle name and mechanism: `FEATURE_LOGIN_V2=1` (env var)
- Behavior when OFF: `exit 1, "login: not available"`

### Regression Cases (code projects only)

Each feature MUST produce a `regression-cases.json` file alongside the design spec. Each case includes:

```json
{
  "id": "<unique-case-id>",
  "name": "<human-readable description>",
  "command": "<cli-command>",
  "toggle": "<ENV_VAR_NAME>",
  "trigger_condition": "<exact conditions that activate this feature>",
  "execution_flow": ["step 1", "step 2", "..."],
  "expect": {
    "on": {
      "exit_code": 0,
      "stdout_contains": ["expected output when ON"],
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
```

### Uncertainty Verification Needed?
- **No** — this feature is purely deterministic; CLI exit code and output are sufficient to judge correctness.
- **Yes** — this feature involves LLM calls, natural-language generation, or subjective quality judgments.
  → Set `uncertainty_verification: true`. After implementation, `regression-guard` will dispatch `clean-context-verification`.

### Spec Self-Review Addition

The spec self-review checklist gains one additional check:
- **CLI entry check (code projects):** Does every feature define a CLI entry point? Does every feature produce regression cases?
```

- [ ] **Step 2: 更新 Checklist 中的第 7 步（Spec self-review）**

在 Checklist 部分（约第 30 行）的步骤 7 中，增强自审检查项。找到：

```markdown
7. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
```

替换为：

```markdown
7. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope. For code projects: check every feature has CLI entry + regression cases. (see below)
```

- [ ] **Step 3: 在 "Designing for agent verifiability" 之后添加相关技能引用**

在 Key Principles 之后、Agent Verifiability 章节末尾，添加：

```markdown
## Related Skills

- **superpowers:test-driven-development** — The three dimensions defined in Agent Verifiability become the test blueprint in TDD's RED phase.
- **superpowers:regression-guard** — Runs the regression cases produced here. Invoked after SDD implementation and during finishing.
- **superpowers:clean-context-verification** — Invoked by regression-guard when Uncertainty Verification is marked "Yes".
```

- [ ] **Step 4: 提交**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat: add Agent Verifiability section to brainstorming — CLI requirement and regression case output"
```

---

### Task 4: 增强 subagent-driven-development 技能 — 插入验证闸门

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md`

**Interfaces:**
- Consumes: 现有 SDD 流程
- Produces: 在所有任务完成、final review 之后增加人工闸门和验证阶段

- [ ] **Step 1: 在 Example Workflow 中 final review 之后插入验证阶段**

定位到 Example Workflow 中 `[After all tasks]` 之后、`[Dispatch final code-reviewer]` 之后、`Done!` 之前（约第 328-332 行）。

当前内容：

```
[After all tasks]
[Dispatch final code-reviewer]
Final reviewer: All requirements met, ready to merge

Done!
```

替换为：

```
[After all tasks]
[Dispatch final code-reviewer]
Final reviewer: All requirements met, ready to merge

[Check project type]
If this is NOT a code project → skip verification, go to Done!

If this IS a code project:
  Present to human partner:
    "All tasks complete and reviewed. The design includes regression cases.
     Ready to enter the verification phase?"

  [Wait for human confirmation]

  [Human confirms]
  Load superpowers:regression-guard skill
  Run current-feature verification mode on the regression-cases.json
  from the design spec.

  regression-guard:
    - Runs each CLI case with TOGGLE=0 and TOGGLE=1
    - On failure: dispatches debug subagent → re-verifies → loops
    - If uncertainty_verification=true: dispatches clean-context-verification
    - On all-pass: writes cases to regression/cases.json

  [Verification complete — cases accumulated]

Done!
```

- [ ] **Step 2: 在 Integration 部分添加 regression-guard 引用**

定位到 `## Integration` 部分（约第 407 行），添加一行：

```markdown
- **superpowers:regression-guard** — CLI verification after all tasks complete (code projects only)
```

- [ ] **Step 3: 提交**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "feat: add verification gate to SDD — regression-guard after all tasks"
```

---

### Task 5: 增强 finishing-a-development-branch 技能 — 插入全量回归闸门

**Files:**
- Modify: `skills/finishing-a-development-branch/SKILL.md`

**Interfaces:**
- Consumes: 现有 finishing 流程
- Produces: 在 Step 1（验证测试通过）之后、Step 2（检测环境）之前插入全量回归闸门

- [ ] **Step 1: 在 Step 1 和 Step 2 之间插入 Step 1.6 全量回归闸门**

定位到约第 56 行，Step 1.5 之后、Step 2 之前。插入：

```markdown
### Step 1.6: Full Regression Guard (code projects only)

**Check project type:** If this is not a code project, or if `regression/cases.json` does not exist, skip this step and continue to Step 2.

**If this is a code project with an existing case library:**

```bash
# Check if regression case library exists
if [ -f regression/cases.json ]; then
  echo "Case library found. Running full regression..."
else
  echo "No regression case library found. Skipping regression guard."
fi
```

If `regression/cases.json` exists:
1. Load `superpowers:regression-guard` skill
2. Run full-regression mode: execute every case in the library
3. On failure: dispatch debug subagent → fix → re-verify → loop (max 3 rounds per case)
4. Report results: passed, failed-and-fixed, still-failing

**If any case still fails after 3 rounds:**
```
Regression guard found <N> case(s) still failing after 3 fix attempts:

<case-id>: <case-name>
  Command: <command>
  Expected: <expect.on>
  Actual: <actual output>

Cannot proceed with merge/PR while regression cases fail.
Fix these manually or adjust the case expectations to match the current behavior.
```

Stop. Do not proceed to Step 2 until all regression cases pass.

**If all cases pass (or were fixed):** Continue to Step 2.
```

- [ ] **Step 2: 提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat: add full regression gate to finishing — run case library before merge"
```

---

### Task 6: 端到端验证 — 确认技能间接口一致

**Files:**
- Read: `skills/regression-guard/SKILL.md`
- Read: `skills/clean-context-verification/SKILL.md`
- Read: `skills/brainstorming/SKILL.md`
- Read: `skills/subagent-driven-development/SKILL.md`
- Read: `skills/finishing-a-development-branch/SKILL.md`

**Interfaces:**
- Consumes: 所有前序任务的产出
- Produces: 一致性验证报告

- [ ] **Step 1: 检查技能间引用完整性**

验证以下交叉引用是否存在且一致：

| 调用方 | 被调用方 | 引用方式 |
|--------|---------|---------|
| brainstorming | regression-guard | Related Skills 中列出 |
| brainstorming | clean-context-verification | Related Skills 中列出 |
| SDD | regression-guard | Integration 中列出，Example Workflow 中展示调用 |
| finishing | regression-guard | Step 1.6 中通过 `superpowers:regression-guard` 引用 |
| regression-guard | clean-context-verification | 不确定性验证流程中派发 |
| regression-guard | systematic-debugging | 失败修复中派发 |

- [ ] **Step 2: 检查项目类型判断的一致性**

验证三个技能中对「代码项目」的判断逻辑一致：
- brainstorming: "Heuristic: does the project's primary deliverable include an executable program or service?"
- SDD: "If this is NOT a code project → skip verification"
- finishing: "If this is not a code project, or if regression/cases.json does not exist, skip this step"

- [ ] **Step 3: 检查 JSON 格式一致性**

验证 brainstorming 中定义的回归案例 JSON 结构与 regression-guard 中描述的库格式完全一致。

- [ ] **Step 4: 如发现不一致，修复并提交**

```bash
git add -A
git commit -m "fix: ensure cross-skill interface consistency for regression verification"
```

---

### Task 7: 创建回归案例 JSON Schema（可选增强）

**Files:**
- Create: `skills/regression-guard/case-schema.json`

**Interfaces:**
- Consumes: 回归案例 JSON 结构定义（来自 brainstorming 和 regression-guard）
- Produces: JSON Schema 文件，用于验证案例格式

- [ ] **Step 1: 创建 JSON Schema**

写入 `skills/regression-guard/case-schema.json`：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Regression Case Library",
  "type": "object",
  "required": ["version", "cases"],
  "properties": {
    "version": { "type": "string", "const": "1" },
    "cases": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "name", "command", "toggle", "trigger_condition", "execution_flow", "expect", "uncertainty_verification"],
        "properties": {
          "id": { "type": "string", "minLength": 1 },
          "name": { "type": "string", "minLength": 1 },
          "command": { "type": "string", "minLength": 1 },
          "toggle": { "type": "string", "minLength": 1 },
          "trigger_condition": { "type": "string", "minLength": 1 },
          "execution_flow": {
            "type": "array",
            "items": { "type": "string" },
            "minItems": 1
          },
          "expect": {
            "type": "object",
            "required": ["on", "off"],
            "properties": {
              "on": {
                "type": "object",
                "required": ["exit_code", "stdout_contains"],
                "properties": {
                  "exit_code": { "type": "integer" },
                  "stdout_contains": {
                    "type": "array",
                    "items": { "type": "string" }
                  },
                  "stderr_empty": { "type": "boolean" }
                }
              },
              "off": {
                "type": "object",
                "required": ["exit_code", "stdout_contains"],
                "properties": {
                  "exit_code": { "type": "integer" },
                  "stdout_contains": {
                    "type": "array",
                    "items": { "type": "string" }
                  },
                  "stderr_empty": { "type": "boolean" }
                }
              }
            }
          },
          "uncertainty_verification": { "type": "boolean" }
        }
      }
    }
  }
}
```

- [ ] **Step 2: 在 regression-guard SKILL.md 中引用 Schema**

在 `skills/regression-guard/SKILL.md` 的 Case Library Operations 部分添加一行：

```markdown
### Schema Validation

The case library and feature-level case files MUST validate against `skills/regression-guard/case-schema.json`. Before processing any case file, validate it. If validation fails, report the error and stop.
```

- [ ] **Step 3: 提交**

```bash
git add skills/regression-guard/case-schema.json skills/regression-guard/SKILL.md
git commit -m "feat: add JSON Schema for regression case validation"
```
