# Knowledge Skill Visibility: 提高知识技能触发率

## 问题陈述

`capturing-knowledge` 和 `refreshing-knowledge` 技能已在 2026-06-15 加入项目，但实际使用中 Agent 经常忘记调用它们。具体表现为：

1. **finish 流程中忽略** — `finishing-a-development-branch` 步骤 1.5 被静默跳过
2. **debug 结束后忽略** — 问题解决了就直接收工
3. **其他阶段忘记查看已有知识** — 做计划/设计前不查 `docs/solutions/`

## 根因分析

五个结构性原因：

1. **缺乏自动触发** — 知识技能完全依赖 Agent 自觉调用，没有 session 生命周期钩子
2. **没有 SessionStop 钩子** — 只有 SessionStart，会话结束时无最后提醒
3. **步骤 1.5 静默跳过** — "Skip this step silently if…" 给了 Agent 过大裁量权
4. **跨技能引用太弱** — 列为 "Related skills" 无约束力
5. **引导技能未提及** — `using-superpowers` 对知识技能零可见性

## 设计目标

- 不增加强制流程负担（避免为记录而编造虚假知识）
- 在真正容易遗忘的关键节点设置刚性提醒
- 让 Agent 从会话开始就知道知识技能存在
- 改动最小化，不动架构

## 方案

### 1. `finishing-a-development-branch` 步骤 1.5：静默跳过 → 显式确认

将步骤 1.5 的跳过条件从 "silently skip" 改为必须显式做出 yes/no 决定。

改动前（第40-48行）：
```markdown
### Step 1.5: Document Learnings

**Before integrating, check whether any non-trivial problem was solved on this branch.**

If yes — …use capturing-knowledge to document it now…

Skip this step silently if:
- The branch only contains mechanical changes
- No new root causes were discovered
```

改动后：
```markdown
### Step 1.5: Document Learnings

**Before integrating, you MUST make an explicit decision about knowledge capture.**

Review the work on this branch and answer:

> Did this branch solve any non-trivial problem? For example:
> - A bug was root-caused and fixed
> - A tricky integration was worked out
> - A non-obvious pattern or approach was established
> - A design decision was made that future contributors should understand

**If YES:** Use `superpowers:capturing-knowledge` to document it now, while the context is still fresh. Once the branch is merged and the session moves on, the institutional knowledge fades.

**If NO:** State explicitly why — for example "mechanical changes only (dependency bump)" or "one-line typo fix" — then continue.

**You may NOT proceed to Step 2 without making this decision explicitly.** Silently skipping is not allowed.
```

同时更新 Red Flags 和 Common Mistakes：

Red Flags 新增：
```
- Proceeding past Step 1.5 without explicitly stating whether knowledge was captured
- Silently deciding "not worth documenting" without justification
```

Common Mistakes 新增：
```
**Silently skipping knowledge capture**
- **Problem:** Assume changes are "too simple" without checking
- **Fix:** Step 1.5 now requires explicit yes/no decision with brief justification
```

### 2. `systematic-debugging` 结尾：弱引用 → 独立提醒小节

在 Phase 4 末尾新增 Phase 4.5，在 Red Flags 之前：

```markdown
### Phase 4.5: Capture the Solution

**After the fix is confirmed, make an explicit decision about knowledge capture.**

Ask yourself:
- Was the root cause non-obvious?
- Did the investigation surface patterns others would benefit from knowing?
- Is there a `docs/solutions/` directory where this belongs?

**If the answer to any of these is yes**, use `superpowers:capturing-knowledge` to document the root cause and solution while context is fresh.

**If the answer to all is no**, note briefly why (e.g., "trivial typo, self-evident fix") and continue.

**This step is NOT optional to skip silently.** You must at minimum think about it and make a conscious decision.
```

同时更新 「When Process Reveals "No Root Cause"」 第3条：

改动前：
```markdown
3. Document what you investigated
```

改动后：
```markdown
3. Document what you investigated — if the investigation itself was instructive, use
   `superpowers:capturing-knowledge` to record the diagnostic approach, even when
   no definitive root cause was found
```

保持 Phase 1-4 不变。Related skills 列表保留不动。

### 3. `using-superpowers` 引导：零可见性 → 显式提及

两处改动：

**Skill Priority（第101-109行）** — 新增第三级：

```markdown
1. **Process skills first** (brainstorming, systematic-debugging) - these determine HOW to approach the task
2. **Implementation skills second** (frontend-design, mcp-builder) - these guide execution
3. **Knowledge skills at key milestones** (capturing-knowledge, refreshing-knowledge) - when a bug is fixed, a non-trivial problem is solved, or a branch is being finished, capture the learning. Before starting a new plan or design, check `docs/solutions/` for relevant recorded knowledge.
```

**Red Flags 表** — 新增一行：

```markdown
| "I'll document this later" | Context fades. Capture while it's fresh or it won't happen. |
```

## 不改的部分

- 不增加 SessionStop 钩子 — 这个改动范围太大，且知识技能还在早期验证阶段
- 不在 `writing-plans` 中加强制 — 保持现有 "if docs/solutions/ exists, check it" 的条件检查
- 不动 `refreshing-knowledge` 本身 — 它的触发时机在 `capturing-knowledge` 步骤 2.5 已有提及
- 不改流程图 — `using-superpowers` 的流程图是给消息响应用的，不适合塞知识技能

## 波及范围

| 文件 | 改动类型 |
|------|----------|
| `skills/finishing-a-development-branch/SKILL.md` | 步骤 1.5 重写 + Red Flags + Common Mistakes |
| `skills/systematic-debugging/SKILL.md` | 新增 Phase 4.5 + 更新 No Root Cause 第3条 |
| `skills/using-superpowers/SKILL.md` | Skill Priority 新增第三级 + Red Flags 新增一行 |

不涉及 hooks、参考文件、其他技能。
