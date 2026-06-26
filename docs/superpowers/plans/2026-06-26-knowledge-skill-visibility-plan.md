# Knowledge Skill Visibility Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修改三个技能文件，将知识技能的触发从「静默跳过/弱引用」改为「显式确认/独立提醒」

**Architecture:** 纯技能文本修改，无代码逻辑变更。每个文件独立改动，无依赖关系。改动模式统一：silently skip → explicit decision。

**Tech Stack:** Markdown 技能文件，无需编程语言

---

## File Structure

| 文件 | 职责 | 改动类型 |
|------|------|----------|
| `skills/finishing-a-development-branch/SKILL.md` | 完成开发分支流程 | 重写步骤 1.5 + 新增 Red Flag + 新增 Common Mistake |
| `skills/systematic-debugging/SKILL.md` | 系统化调试流程 | 新增 Phase 4.5 + 更新 No Root Cause 第3条 |
| `skills/using-superpowers/SKILL.md` | 会话启动引导 | Skill Priority 新增第三级 + Red Flags 新增一行 |

---

### Task 1: 改造 finishing-a-development-branch 步骤 1.5

**Files:**
- Modify: `skills/finishing-a-development-branch/SKILL.md`

- [ ] **Step 1: 替换步骤 1.5 内容（第40-48行）**

将现有的静默跳过版本替换为显式确认版本：

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

- [ ] **Step 2: 在 Red Flags 列表新增条目（第235行附近，"Never:" 列表之后、"Always:" 列表之前）**

在 "Never:" 列表末尾添加：
```markdown
- Proceed past Step 1.5 without explicitly stating whether knowledge was captured
- Silently decide "not worth documenting" without justification
```

- [ ] **Step 3: 在 Common Mistakes 表新增条目（第203行附近，"No confirmation for discard" 之后）**

```markdown
**Silently skipping knowledge capture**
- **Problem:** Assume changes are "too simple" without checking
- **Fix:** Step 1.5 now requires explicit yes/no decision with brief justification
```

- [ ] **Step 4: 提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "fix: make finishing step 1.5 knowledge capture an explicit decision

Replace silent skip with required yes/no decision, add red flags
and common mistakes entries."
```

---

### Task 2: 在 systematic-debugging Phase 4 之后新增 Phase 4.5

**Files:**
- Modify: `skills/systematic-debugging/SKILL.md`

- [ ] **Step 1: 在 Phase 4 和 Red Flags 之间插入 Phase 4.5（第219行 "## Red Flags" 之前）**

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

- [ ] **Step 2: 更新 "No Root Cause" 第3条（第277行附近）**

将：
```markdown
3. Document what you investigated
```

替换为：
```markdown
3. Document what you investigated — if the investigation itself was instructive, use
   `superpowers:capturing-knowledge` to record the diagnostic approach, even when
   no definitive root cause was found
```

- [ ] **Step 3: 提交**

```bash
git add skills/systematic-debugging/SKILL.md
git commit -m "fix: add Phase 4.5 knowledge capture reminder to systematic-debugging

Insert explicit decision gate after fix confirmation, update
No Root Cause guidance to reference capturing-knowledge."
```

---

### Task 3: 在 using-superpowers 引导中加入知识技能提及

**Files:**
- Modify: `skills/using-superpowers/SKILL.md`

- [ ] **Step 1: Skill Priority 新增第三级（"Implementation skills second" 行之后，第109行附近）**

在：
```markdown
2. **Implementation skills second** (frontend-design, mcp-builder) - these guide execution
```

之后插入：
```markdown
3. **Knowledge skills at key milestones** (capturing-knowledge, refreshing-knowledge) - when a bug is fixed, a non-trivial problem is solved, or a branch is being finished, capture the learning. Before starting a new plan or design, check `docs/solutions/` for relevant recorded knowledge.
```

- [ ] **Step 2: Red Flags 表新增一行（"I know what that means" 行之后，第99行附近）**

在：
```markdown
| "I know what that means" | Knowing the concept ≠ using the skill. Invoke it. |
```

之后插入：
```markdown
| "I'll document this later" | Context fades. Capture while it's fresh or it won't happen. |
```

- [ ] **Step 3: 提交**

```bash
git add skills/using-superpowers/SKILL.md
git commit -m "fix: add knowledge skills to using-superpowers skill priority

Add third priority level for capturing-knowledge and refreshing-knowledge,
add red flag for 'I'll document this later' rationalization."
```
