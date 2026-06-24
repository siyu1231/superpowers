# Superpowers

> **这是 [obra/superpowers](https://github.com/obra/superpowers) 的分支，新增了两个知识积累技能：**
> - **`capturing-knowledge`（捕获知识）** — 解决非平凡问题后，趁上下文还新鲜时将解决方案文档化到 `docs/solutions/`
> - **`refreshing-knowledge`（刷新知识）** — 定期对照代码库审查 `docs/solutions/`，执行 Keep / Update / Consolidate / Replace / Delete 操作以保持文档准确
>
> 这些技能已嵌入现有工作流：`brainstorming`、`writing-plans` 和 `systematic-debugging` 会在开始工作前自动查阅 `docs/solutions/`；`finishing-a-development-branch` 会在合并前提醒记录经验。

Superpowers 是一套完整的编程代理软件开发方法论，基于一组可组合的技能和一组确保代理使用它们的初始指令构建。

[English version](README.md)

## 招聘！

我们正在招聘一位全职负责 Superpowers 社区和代码工作的人员。
详情请见：https://primeradiant.com/jobs/superpowers-community-engineer/
如果你认识合适的人选，请务必推荐。

## 快速开始

给你的编程代理赋予超能力：[Claude Code](#claude-code)、[Antigravity](#antigravity)、[Codex App](#codex-app)、[Codex CLI](#codex-cli)、[Cursor](#cursor)、[Factory Droid](#factory-droid)、[Gemini CLI](#gemini-cli)、[GitHub Copilot CLI](#github-copilot-cli)、[Kimi Code](#kimi-code)、[OpenCode](#opencode)、[Pi](#pi)。

## 工作原理

从你启动编程代理的那一刻起，它就会开始工作。当它发现你在构建某个东西时，它*不会*直接跳到写代码。相反，它会退后一步，先问清楚你真正想做什么。

当它从对话中梳理出规格说明后，会分成足够短的段落展示给你，让你真正能阅读和消化。

在你认可设计方案后，你的代理会制定一个实施计划——清晰到连一个热情但品味糟糕、缺乏判断力、毫无项目背景且讨厌测试的初级工程师都能执行。它强调真正的红/绿 TDD、YAGNI（你不会需要它）和 DRY（不要重复自己）。

然后，当你说"开始"时，它会启动一个*子代理驱动开发*流程，让代理们完成每个工程任务，检查并审查它们的工作，然后继续推进。你的代理经常可以自主工作几个小时而不偏离你制定的计划。

这背后还有很多细节，但这就是系统的核心。而且因为这些技能是自动触发的，你不需要做任何特殊操作。你的编程代理自然而然地拥有了超能力。

## 商业服务

如果你在企业中使用 Superpowers，并需要商业支持、额外工具或消费管理，欢迎联系我们：sales@primeradiant.com。

## 安装

安装方式因平台而异。如果你使用多个平台，需要为每个平台分别安装 Superpowers。

### Claude Code

- 将此仓库注册为 marketplace 来源：

  ```bash
  /plugin marketplace add siyu1231/superpowers

  /plugin install superpowers@superpowers
  ```

### Antigravity

从此仓库安装 Superpowers 插件：

```bash
agy plugin install https://github.com/siyu1231/superpowers
```

Antigravity 会在会话开始时运行插件的启动钩子，因此 Superpowers 从第一条消息起就已激活。使用相同命令重新安装即可更新。

### Codex App

Superpowers 可通过[官方 Codex 插件市场](https://github.com/openai/plugins)获取。

- 在 Codex 应用中，点击侧边栏中的 Plugins。
- 你应该会在 Coding 分类下看到 `Superpowers`。
- 点击 Superpowers 旁边的 `+` 并按照提示操作。

### Codex CLI

Superpowers 可通过[官方 Codex 插件市场](https://github.com/openai/plugins)获取。

- 打开插件搜索界面：

  ```bash
  /plugins
  ```

- 搜索 Superpowers：

  ```bash
  superpowers
  ```

- 选择 `Install Plugin`。

### Cursor

- 在 Cursor Agent 聊天中运行：

  ```text
  /plugin marketplace add siyu1231/superpowers
  /plugin install superpowers@superpowers
  ```

### Factory Droid

- 注册 marketplace：

  ```bash
  droid plugin marketplace add https://github.com/siyu1231/superpowers
  ```

- 安装插件：

  ```bash
  /plugin install superpowers@superpowers
  ```

或者克隆仓库到本地，通过 `--plugin-dir` 启动：

  ```bash
  git clone https://github.com/siyu1231/superpowers.git ~/superpowers
  claude --plugin-dir ~/superpowers
  ```

### Gemini CLI

- 安装扩展：

  ```bash
  gemini extensions install https://github.com/siyu1231/superpowers
  ```

- 后续更新：

  ```bash
  gemini extensions update superpowers
  ```

### OpenCode

OpenCode 使用自己的插件安装方式；即使你已在其他平台中使用，也需要单独安装 Superpowers。

编辑 `opencode.json` 并添加：

  ```json
  {
    "plugin": ["superpowers@git+https://github.com/siyu1231/superpowers.git"]
  }
  ```

### GitHub Copilot CLI

- 注册 marketplace：

  ```bash
  copilot plugin marketplace add siyu1231/superpowers
  ```

- 安装插件：

  ```bash
  copilot plugin install superpowers@superpowers
  ```

### Kimi Code

Superpowers 已在 Kimi Code 的插件市场中提供。

- 打开 Kimi Code 的插件管理器：

  ```text
  /plugins
  ```

- 前往 `Marketplace` > `Superpowers` 并安装。

- 或直接从仓库安装：

  ```text
  /plugins install https://github.com/obra/superpowers
  ```

- 详细文档：[docs/README.kimi.md](docs/README.kimi.md)

### OpenCode

OpenCode 使用自己的插件安装方式；即使你已在其他平台中使用，也需要单独安装 Superpowers。

- 告诉 OpenCode：

  ```
  Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
  ```

- 详细文档：[docs/README.opencode.md](docs/README.opencode.md)

### Pi

从此仓库安装 Superpowers 作为 Pi 包：

```bash
pi install git:github.com/obra/superpowers
```

本地开发时，将此仓库作为临时包加载运行 Pi：

```bash
pi -e /path/to/superpowers
```

Pi 包会加载 Superpowers 技能和一个小型扩展——该扩展会在会话启动和压缩后重新注入 `using-superpowers` 引导程序。Pi 原生支持技能，因此不需要兼容性 `Skill` 工具。子代理和任务列表工具仍是可选的 Pi 配套包。

## 基本工作流

1. **brainstorming（头脑风暴）** — 在写代码前激活。通过提问澄清需求、细化想法，探索替代方案，分部分展示设计并获取用户确认。保存设计文档。

2. **using-git-worktrees（使用 Git 工作树）** — 设计批准后激活。在新分支上创建隔离的工作空间，运行项目设置，验证干净的测试基准。

3. **writing-plans（编写计划）** — 在获得批准的设计下激活。将工作拆分为小任务（每个 2–5 分钟）。每个任务都有精确的文件路径、完整代码和验证步骤。

4. **subagent-driven-development（子代理驱动开发）** 或 **executing-plans（执行计划）** — 有计划后激活。为每个任务分派全新的子代理，经过两阶段审查（规格合规、然后代码质量），或分批执行并设有人工检查点。

5. **test-driven-development（测试驱动开发）** — 在实现过程中激活。强制执行红-绿-重构：先写一个失败的测试，看着它失败，写最小量的代码，看着它通过，然后提交。在测试之前写的代码会被删除。

6. **requesting-code-review（请求代码审查）** — 在任务之间激活。对照计划进行审查，按严重程度报告问题。严重问题会阻止进度。

7. **finishing-a-development-branch（完成开发分支）** — 所有任务完成时激活。验证测试通过，提供选项（合并/PR/保留/丢弃），清理工作树。

**代理在执行任何任务前都会检查相关技能。** 这是强制性的工作流，不是建议。

## 技能库

### 测试
- **test-driven-development（测试驱动开发）** — 红-绿-重构循环（包含测试反模式参考）

### 调试
- **systematic-debugging（系统化调试）** — 四阶段根因分析流程（包含根因追踪、纵深防御、基于条件的等待等技术）
- **verification-before-completion（完成前验证）** — 确保问题真正被修复

### 知识管理
- **capturing-knowledge（捕获知识）** — 将已解决的问题文档化到 `docs/solutions/`，采用双轨道系统（bug / knowledge），包含重叠检测和 CONCEPTS.md 词汇捕获
- **refreshing-knowledge（刷新知识）** — 随时间推移维护 `docs/solutions/` 的准确性，执行 Keep / Update / Consolidate / Replace / Delete 操作

### 协作
- **brainstorming（头脑风暴）** — 苏格拉底式的设计细化
- **writing-plans（编写计划）** — 详细的实施计划
- **executing-plans（执行计划）** — 带检查点的分批执行
- **dispatching-parallel-agents（分派并行代理）** — 并行的子代理工作流
- **requesting-code-review（请求代码审查）** — 提交前审查清单
- **receiving-code-review（接收代码审查）** — 响应审查反馈
- **using-git-worktrees（使用 Git 工作树）** — 并行开发分支
- **finishing-a-development-branch（完成开发分支）** — 合并/PR 决策工作流
- **subagent-driven-development（子代理驱动开发）** — 带两阶段审查的快速迭代（规格合规，然后代码质量）

### 元技能
- **writing-skills（编写技能）** — 遵循最佳实践创建新技能（包含测试方法论）
- **using-superpowers（使用 Superpowers）** — 技能系统简介

## 理念

- **测试驱动开发** — 永远先写测试
- **系统化优于临时处理** — 流程优于猜测
- **降低复杂度** — 简洁为首要目标
- **用证据说话** — 在宣布成功前先验证

阅读[原始发布公告](https://blog.fsck.com/2025/10/09/superpowers/)。

## 参与贡献

以下是 Superpowers 的一般贡献流程。请注意，我们通常不接受新技能的贡献，任何对技能的修改必须在所有支持的编程代理上正常工作。

1. Fork 仓库
2. 切换到 `dev` 分支
3. 为你的工作创建分支
4. 遵循 `writing-skills` 技能来创建和测试新的或修改后的技能
5. 提交 PR，务必填写完整的 PR 模板。

技能行为测试使用 [superpowers-evals](https://github.com/prime-radiant-inc/superpowers-evals/) 提供的 drill 评估框架，克隆到 `evals/` 目录——详见 `evals/README.md`。插件基础设施测试位于 `tests/` 目录，通过相应的 `run-*.sh` 或 `npm test` 运行。

完整指南请参见 `skills/writing-skills/SKILL.md`。

## 更新

Superpowers 的更新因编程代理而异，但通常是自动的。

## 许可证

MIT License — 详见 LICENSE 文件

## 可视化伴侣遥测

由于技能和插件不会向创作者提供任何反馈，我们无法知道有多少人在使用 Superpowers。默认情况下，brainstorming 的可选可视化伴侣功能中的 Prime Radiant 标志会从我们的网站加载。它包含正在使用的 Superpowers 版本号，但**不包含**关于你的项目、提示词或编程代理的任何细节。我们不会看到你的点击记录或你正在构建的内容。这仅帮助我们大致了解有多少人在使用 Superpowers 以及他们使用的版本。这是 100% 可选的。要禁用此功能，请将环境变量 `SUPERPOWERS_DISABLE_TELEMETRY` 设为任意真值。Superpowers 也会遵循 Claude Code 的 `DISABLE_TELEMETRY` 和 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 退出选项。

## 社区

Superpowers 由 [Jesse Vincent](https://blog.fsck.com) 和 [Prime Radiant](https://primeradiant.com) 的其他成员共同打造。

- **Discord**：[加入我们](https://discord.gg/35wsABTejz) 获取社区支持、提问并分享你使用 Superpowers 构建的项目
- **Issues**：https://github.com/obra/superpowers/issues
- **发布公告**：[订阅](https://primeradiant.com/superpowers/) 获取新版本通知
