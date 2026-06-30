# CLI 回归验证系统 — 设计文档

**状态:** 设计完成
**目标:** 构建以 CLI 命令为中心的回归验证系统，将手动测试验证自动化，让 AI agent 能够通过 CLI 命令自主验证功能、修复问题、积累回归案例。

## 问题

AI 开发中主要时间花在手动测试验证上。当前 Superpowers 的技能链（brainstorming → writing-plans → SDD → finishing）缺少系统化的验证阶段：
- 没有机制积累回归案例防止退步
- 没有 CLI 级别的自动化验证循环
- 确定性验证和不确定性验证没有分开处理
- 功能开关（feature toggle）概念存在于模板中但未被强制执行

## 设计概览

### 新建技能

| 技能 | 职责 |
|---|---|
| `regression-guard` | CLI 命令级回归验证引擎：确定性验证（ON/OFF 对照）、案例库读写、失败时自动派子 agent debug 修复并循环验证、通过后写入 / 更新案例库 |
| `clean-context-verification` | 不确定性验证：干净上下文子 agent，只知正确预期、不知实现细节；失败时同样派 debug 子 agent 修复并重新验证 |

### 增强现有技能

| 技能 | 改动 |
|---|---|
| `brainstorming` | CLI Entry Point 从建议升级为硬性要求；Agent Verifiability 产出 `regression-cases.json`（含 ON/OFF 对照 + uncertainty 标记） |
| `subagent-driven-development` | 实现 + review 通过后插入人工闸门，用户确认后调用 regression-guard 验证当前功能 |
| `finishing-a-development-branch` | 合并 / PR 前插入全量回归闸门，运行案例库中所有历史案例 |

### 完整开发流程

```
brainstorming ──→ design.md + regression-cases.json (代码项目)
     │                    （CLI 入口是硬性要求）
     ▼
writing-plans ──→ implementation plan
     ▼
SDD (subagent-driven-development)
  ├─ implement → spec review → quality review → fix loop → 通过
  │
  └─→ 代码项目？
        ├─ 否 → 跳过验证
        └─ 是 →
              │
            【人工闸门】"是否进入验证阶段？"
              │  用户确认
              ▼
            regression-guard: 当前功能验证
              ├─ 确定性验证: TOGGLE=0/1 CLI 执行 → 对比期望
              ├─ 失败 → 子 agent (systematic-debugging) 修复 → 重验（单案例最多3轮）
              ├─ 不确定性验证（如标记需要）:
              │   子 agent (clean-context-verification) 干净上下文验证
              │   └─ 失败 → 子 agent (debug) 修复 → 重验
              └─ 全部通过 → 写入案例库
                   ├─ 新增: append 到 regression/cases.json
                   └─ 同 id 但期望变化: update
     ▼
finishing-a-development-branch
  │
  └─→ 代码项目？
        ├─ 否 → 跳过全量回归
        └─ 是 →
              │
            【全量回归闸门】
            regression-guard: 运行案例库
              └─ 失败 → debug 修复 → 重新验证
```

---

## 回归案例 JSON 数据结构

### 单条案例结构

```json
{
  "version": "1",
  "feature": "email-validation",
  "toggle": "FEATURE_EMAIL_VALIDATION",
  "cases": [
    {
      "id": "email-valid-001",
      "name": "验证合法邮箱格式",
      "command": "./app validate --email test@example.com",
      "trigger_condition": "输入为合法邮箱格式 test@example.com，FEATURE_EMAIL_VALIDATION 环境变量控制是否执行校验",
      "execution_flow": [
        "1. 解析 --email 参数",
        "2. 检查 FEATURE_EMAIL_VALIDATION 环境变量",
        "3. 若开启：执行邮箱格式校验，返回 valid",
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

### 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | string | 唯一标识。写入案例库时，相同 `id` 更新，新 `id` 追加 |
| `name` | string | 人类可读的案例名称，描述测试的行为 |
| `command` | string | 验证的 CLI 命令 |
| `trigger_condition` | string | 功能触发的确切条件，边界条件说明 |
| `execution_flow` | string[] | 执行步骤，按顺序排列，说明每步的中间状态 |
| `expect.on` | object | 开关开启时的期望：`exit_code`、`stdout_contains`、`stderr_empty` |
| `expect.off` | object | 开关关闭时的期望：`exit_code`、`stdout_contains`、`stderr_empty` |
| `uncertainty_verification` | boolean | 是否需要干净上下文子 agent 验证（LLM 输出、自然语言、主观质量） |

### 案例库累积后形态

```json
{
  "version": "1",
  "cases": [
    { "id": "email-valid-001", "name": "验证合法邮箱格式", ... },
    { "id": "email-invalid-002", "name": "拒绝非法邮箱", ... },
    { "id": "login-timeout-001", "name": "登录超时处理", ... }
  ]
}
```

### 设计原则

- **`stdout_contains` 而非精确匹配** — 输出中可能夹杂时间戳、颜色代码等动态内容，用包含判断比精确匹配更健壮。如需精确匹配，可通过命令参数控制（如 `--plain` 关闭颜色）。
- **`id` 作为唯一键** — 当新功能改变了旧命令的期望行为，同一 `id` 的案例会被更新；全新功能则用新 `id` 追加。
- **ON/OFF 对照内置** — `expect.on` 和 `expect.off` 直接定义两种开关状态下的期望差异。Agent 执行验证时同一命令跑两遍（TOGGLE=0, TOGGLE=1），验证差异化行为。

---

## regression-guard 技能

### 技能定位

被两个节点调用：
1. **SDD 实现完成后的当前功能验证**（由 subagent-driven-development 调度）
2. **finishing-a-development-branch 中的全量回归**（由 finishing 技能调度）

两种场景下共享同一个验证引擎，区别仅在于案例来源：当前功能验证读取本次 brainstorming 产出的案例，全量回归读取历史案例库。

### 核心循环

```
读入案例文件 (本次的 JSON 或案例库)
        │
        ▼
  对每条案例 ──────────────────────────────────────┐
        │                                           │
        ▼                                           │
  ┌ 确定性验证 ─────────────────────────┐            │
  │                                     │            │
  │  1. TOGGLE=0 执行 command            │            │
  │     对比 expect.off                  │            │
  │     ├─ 匹配 → 继续                    │           │
  │     └─ 不匹配 → 记录失败 ──────────┐  │            │
  │                                     │  │            │
  │  2. TOGGLE=1 执行 command            │  │            │
  │     对比 expect.on                   │  │            │
  │     ├─ 匹配 → 继续                    │  │            │
  │     └─ 不匹配 → 记录失败 ──────────┘  │            │
  │                                     │            │
  └─────────────────────────────────────┘            │
        │                                           │
        ▼                                           │
  有失败？ ── 是 ──→ 派子 agent                      │
        │          (systematic-debugging)             │
        │          修复后重新确定性验证，重跑该案例       │
        │          循环直到通过 ──────────────────────┤
       否                                           │
        │                                           │
        ▼                                           │
  uncertainty_verification == true?                  │
        │                                           │
       是                                           │
        │                                           │
        ▼                                           │
  ┌ 不确定性验证 ─────────────────────────┐          │
  │  派子 agent                            │          │
  │  (clean-context-verification)           │          │
  │  干净上下文 + 正确预期                   │          │
  │  ├─ 通过 → 继续                         │          │
  │  └─ 失败 → 子 agent(debug) 修复         │          │
  │           → 重跑不确定性验证             │          │
  │           → 循环直到通过                 │          │
  └─────────────────────────────────────┘            │
        │                                           │
        ▼                                           │
  全部通过 → 写入案例库（仅当前功能验证模式）  ────────┘
```

### 关键行为规则

| 规则 | 内容 |
|---|---|
| **最大重试** | 每条案例最多 3 轮 debug 修复，超过则暂停并向人类报告，等待决策 |
| **失败报告** | 每轮失败输出：哪个案例、ON 还是 OFF 状态、实际输出 vs 期望输出 |
| **debug 子 agent 上下文** | 传入失败案例的完整信息 + 当前代码实现，加载 `systematic-debugging` 技能 |
| **回写案例库** | 仅在「当前功能验证」模式下执行；全量回归模式不写案例库。新增案例 → append；同 `id` 但期望变化 → update |
| **案例库路径** | 默认 `regression/cases.json`，可由项目配置 |
| **全量回归模式** | 读取案例库中所有案例，逐条执行验证。失败同样进入 debug 修复循环。不写入案例库 |

---

## clean-context-verification 技能

### 技能定位

处理确定性验证无法覆盖的场景 — LLM 调用、自然语言生成、主观质量判断等。核心机制：子 agent 获得干净上下文 — 只能看到正确预期的描述，不接触任何实现代码、设计文档、开发讨论。这模拟真实用户判断，避免被实现细节带偏。

### 核心流程

```
主 agent (regression-guard)
        │
        │  传入: 案例的 name + expect.on + execution_flow
        │  （不含任何实现代码、文件路径、开发讨论）
        │
        ▼
  子 agent (clean-context-verification)
        │
        │  1. 执行 command (TOGGLE=1)
        │  2. 根据 expect.on 判断输出是否正确
        │  3. 用自然判断验证质量（格式、语气、完整性等）
        │
        ▼
  输出判决 ──→ 通过
        │
       失败
        │
        ▼
  主 agent 收到失败报告
        │
        │  传入: 失败描述 + 当前代码实现
        │  （此时才暴露实现细节给 debug 子 agent）
        │
        ▼
  子 agent (systematic-debugging) 修复
        │
        ▼
  重新派 clean-context-verification 验证
        │
        ▼
  循环直到通过（最多 3 轮）
```

### 干净上下文的边界

| 传入子 agent 的内容 | 不传入的内容 |
|---|---|
| 案例 `name`（预期行为描述） | 实现代码 |
| 案例 `command`（CLI 命令） | 设计文档 / spec |
| 案例 `expect.on`（期望结果） | 文件路径 |
| 案例 `execution_flow`（执行流程） | 开发中的讨论 |
| | 开关 / toggle 机制细节 |

### 触发条件

回归案例中 `uncertainty_verification: true`。该标记在 brainstorming 阶段由设计者确定：该功能是否涉及 LLM 输出、自然语言、或主观质量。

---

## 现有技能增强

### brainstorming

**CLI 强制要求（针对被开发的目标项目）：**

Agent Verifiability 的 CLI Entry Point 从建议升级为硬性要求 — 但仅适用于**目标代码项目**（Superpowers 正在帮助开发的那个项目）。如果目标是文档、设计规范、或其他不天然适合 CLI 的产物，则不做强制。

```markdown
#### CLI Entry Point
- 对于代码项目，每个功能必须有 CLI 入口。命令: `<command> <args>`
- 没有 CLI 入口 = 不可由 agent 自动验证。
- 设计评审时，代码项目缺少 CLI 入口视为阻塞项。
```

**适用范围判断：** brainstorming 阶段根据被开发项目的性质判断是否需要 CLI。启发式规则：该项目的主产物是可执行的程序/服务 → 必须有 CLI；主产物是文档/设计 → 不需要。

**新增回归案例产出步骤：**

```markdown
#### Regression Cases → 写入 regression-cases.json
- 每条案例包含 ON/OFF 对照
- 标记是否需要 uncertainty_verification
- 仅代码项目需要产出回归案例
```

**设计评审闸门：** spec self-review 新增检查项：「如果是代码项目，每个功能是否定义了 CLI 入口？」

**对于目标项目开发者的约束：** 代码项目必须暴露 CLI 接口供 agent 验证。这本质上是「为了 agent 可验证而做的适配层」。CLI 规范建议：Unix 风格子命令、`--flag` 参数化、exit code 遵循约定（0 成功，非 0 失败）、stdout 输出结果。

### subagent-driven-development

在实现完成、spec review + quality review 通过后，插入人工闸门和验证阶段：

```
SDD: implement → spec review → quality review → fix loop → 通过
                                                          │
                                                     【新增】判断项目类型
                                                     ├─ 非代码项目 → 跳过验证阶段
                                                     └─ 代码项目 →
                                                          │
                                                     【新增】人工闸门
                                                     "是否进入验证阶段？"
                                                          │ 用户确认
                                                          ▼
                                                     regression-guard
                                                     当前功能验证
                                                     （含 debug 修复循环）
                                                     （通过后写入案例库）
```

人工闸门设计原则：不自动进入验证。Agent 必须向用户出示当前状态并等待确认。这符合 Superpowers 的「human partner in control」原则。

**非代码项目直接跳过验证阶段** — 回归案例和 CLI 验证对文档、设计规范等项目类型不适用。

### finishing-a-development-branch

在合并 / PR 前，插入全量回归闸门：

```
finishing: 展示分支状态 → 选择集成方式
                              │
                         【新增】判断项目类型
                         ├─ 非代码项目 → 跳过全量回归
                         └─ 代码项目 →
                              │
                         【新增】全量回归闸门
                         运行 regression-guard
                         全量案例库验证
                         ├─ 通过 → 继续合并/PR 流程
                         └─ 失败 → debug 修复循环 → 重新验证
```

**非代码项目直接跳过全量回归** — 没有 CLI 入口的项目不产生回归案例，不需要此步骤。

---

## Agent Verifiability（本设计自身的可验证性）

### CLI Entry Point
- 本设计产出的是技能文件 + 回归案例 JSON 格式，不涉及运行时 CLI。验证方式：
  - 回归案例格式：用 JSON Schema 校验
  - 技能行为：通过 evals 场景验证 end-to-end

### Feature Toggle
- 不适用（本设计是技能 / 格式定义，不是运行时功能）

### Uncertainty Verification Needed?
- **No** — 本设计是结构化的技能定义和 JSON 格式，不涉及 LLM 调用或自然语言生成

---

## 验证

- 回归案例 JSON 格式由 JSON Schema 验证
- 技能行为通过 evals 场景验证：新建场景覆盖「当前功能验证通过后写入案例库」「确定性验证失败→debug 修复→通过」「不确定性验证子 agent 判决」「全量回归发现退步→修复」
- 三个增强点（brainstorming、SDD、finishing）的改动不影响现有 evals 场景通过
- 新增案例库读写操作的边界条件测试：空案例库、冲突 id 更新、无效 JSON 格式
