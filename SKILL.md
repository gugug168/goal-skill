---
name: goal
description: Use when the user says "/goal" or wants to autonomously pursue a durable objective — equivalent to Codex /goal. Decomposes goals into milestones, dispatches agents, and enforces independent verification before marking complete.
version: 2.15.0
author: Hermes Agent
license: MIT
platforms: [macos, linux]
metadata:
  hermes:
    tags: [autonomous, goal, milestone, delegation, verification]
    trigger: [/goal, autonomous, persistent, long-running]
    related_skills: [goal-oriented-autonomy, task-decomp, brainstorming, writing-plans, autonomous-dev-loop, finishing-a-development-branch, kanban-orchestrator, darwin-evaluation]
---

# /goal — Autonomous Goal Pursuit

## 版本管理约定

每次 SKILL.md 版本升级后，同步推送 GitHub：

```bash
# 确认版本已更新（frontmatter + changelog）
git -C ~/.hermes/skills/software-development/goal add SKILL.md
git -C ~/.hermes/skills/software-development/goal commit -m "goal-skill vX.Y.Z"
git -C ~/.hermes/skills/software-development/goal push origin main

# 如果 references/ 有新增文件，同步推送
git -C ~/.hermes/skills/software-development/goal add references/
git -C ~/.hermes/skills/software-development/goal commit -m "goal-skill: update references"
git -C ~/.hermes/skills/software-development/goal push origin main
version: 2.15.1

## Changelog

### v2.15.1 — 2026-05-17
**autonomous-loop.py 小修：**
- **DB 查询修复**：`updated_at` 列不存在，改用 `consecutive_failures` 过滤频繁失败任务（< 3 次）
- **三角并行正则修复**：`\s` 改 `\\s`，避免 SyntaxWarning
- **reset 测试修复**：mock dict.pop 返回 None 时 `== None` 判断失败（pop 返回 None 是正确行为）

### v2.15.0 — 2026-05-17
**autonomous-loop.py v2.15 核心优化（5 项改进）：**

1. **三角并行精确匹配**：原来 `impl+review`（任意两个）就触发三角 → 现在必须同时有 impl+review+test 三个关键词才触发。避免"需要 review"的任务被误判为三角并行，导致 review agent 空转。

2. **Crash Guard 按 task_id 而非 title**：原来用 title 前80字符做 key，重复标题会互相干扰 → 现在用 task_id（全局唯一）。更精确的 crash 追踪。

3. **verify_workspace 强化**：原来"无法确认变更"也算失败 → 现在只要 git diff 有输出（实际文件变更）就通过，不再依赖 agent 文本输出关键词猜测。

4. **GoalProgressTracker 进度追踪**：新增 `GoalProgressTracker` 类，写入 `~/.hermes/logs/autonomous-loop/.goal-progress-{id}.json`，追踪 token 消耗和任务完成进度，80% 预算时自动发飞书预警。

5. **最近失败任务跳过**：DB 查询新增 `completed_at < 5分钟前` 过滤，避免刚降级（todo）的任务被立刻重新取出执行，造成死循环。

### v2.14.0 — 2026-05-17
**web-access CDP Browser Mode 集成（完善）：**
- Skill Loading Order 新增完整登录态检测 + Cookie 导出工作流（Playwright CDP → JSON → Netscape → yt-dlp）
- macOS Chrome Cookie 导出脚本：`cookies-json-to-netscape.py`（web-access/scripts/）
- **6 个平台批量登录态检测脚本**（知乎/小红书/微博/抖音/B站/百度）写入 goal SKILL.md
- 知乎反自动化检测说明：CDP tab 直接登录会被拒绝，必须导出 cookies 用 yt-dlp
**Crash Guard + Keep/Discard 逻辑细化：**
- Autonomous-loop.py 新增 `_CRASH_COUNT` 去重保护：同一任务 3 次 crash → 标记 `needs-human`，停止自动重试
- 新增 `needs-human` 状态：任务无法自动完成，需要人工介入
- 状态转换规则：
  - `exec_ok + verify_ok + commit_ok` → `keep`（goal-results.tsv）
  - `exec_ok + verify_fail` → `discard`，降级 `todo`，下次重试
  - `exec_fail + crash_guard_hit` → `crash`，标记 `needs-human`，停止重试
  - `exec_fail + crash_guard_not_hit` → `crash`，降级 `todo`，下次重试
- **三角并行检测**：TRIANGLE_TAGS = `["三角", "parallel", "impl+review+test", "t:implementation", "实现+审查+测试", "multi-agent"]`
  - task body 或 assignee 含以上关键词 → 触发三角并行（3 agent 同时执行）
  - 不含 → 标准单 agent 执行
- **验收标准解析**（autonomous-loop.py `parse_acceptance_criteria`）：
  - 格式：`- [ ] 文本` / `- [x] 文本` / `✓ 文本` / `✅ 文本`
  - 无明确格式时 → `guess_criteria_from_title()` 推断（pytest/implement/fix/review 等关键词）
  - 最终由 `verify_workspace()` 验证（文件存在/tsc/pytest/git diff）

### v2.11.0 — 2026-05-17
**视频能力集成：**
- 新增 `video-download` skill（yt-dlp）：搜索+下载 YouTube/B站等平台视频
- 新增 `youtube-content` skill：YouTube transcript 提取+总结
- Skill Loading Order 新增 #9 `video-download`、#10 `youtube-content`
- Autonomous Dev Loop cron job 重建（job_id: `7202cab8a127`，每5分钟调度）
- Goal Feishu Listener cron job 正常运行（job_id: `3ce8e3b7cd36`）
- `goal-manager.py` 支持 task-decomp 自动生成 subtasks（状态设为 `ready`，立即触发 autonomous-loop）

### v2.10.0 — 2026-05-14
**GLM 模型 + Anthropic 代理修复：**
- 本地代理 `~/Scripts/anthropic_proxy.py` 将 `/v1/messages` 翻译为 `/chat/completions`
- 修复 cc-switch 读错 token（不要从 `~/.mmx/config.json` 读 — 那是 MiniMax key）
- 修复 `max_turns=15` 对标准任务更合适（之前用 5 导致多轮编辑被截断）
- 修复 `HOME=/tmp/claude_clean_home` 在 tmux -c 场景导致 API 404 的问题
- 修复 idle 检测误杀慢 API（`timeout > 600` 条件）
- 新增 `goal-results.tsv` keep/discard/crash 结果追踪
- 新增 `classify_task_complexity()` 任务复杂度分类（simple_direct/standard/complex）

### v2.9.0 — 2026-05-13
**三角并行 + 验收标准解析：**
- `run_triangular()` 实现：自动识别三角任务，并行发到 3 个 agent
- `parse_acceptance_criteria()` 强制解析 task body 里的 `- [ ]` / `✓` 格式
- `verify_workspace()` 闭环验证（文件存在/pytest/tsc/commit）
- 修复 `bash -c '...'` 单引号转义 bug（用临时脚本文件替代）
- 修复 `NameError: agent_timeout not defined`（`run_claude_code_interactive()` 参数是 `timeout`）

---

GitHub 仓库：https://github.com/gugug168/goal-skill

---

## Overview

**The master entry point for fully autonomous goal-driven development.** When the user says `/goal` or describes a durable objective, this skill orchestrates the complete lifecycle: understanding → planning → implementation → verification → delivery.

**This skill is the equivalent of Codex `/goal`** — a persistent, multi-hour autonomous development loop that works toward a clear stopping condition.

## Core Principle

**A cannot verify A. Every task has an implementer and a separate verifier using a different model/provider.**

---

## The 7-Phase Workflow

```
Phase 0: UNDERSTAND GOAL
    │ User says "/goal [objective]"
    ↓
Phase 1: DECOMPOSE — task-decomp skill
    │ Break into milestones with acceptance criteria
    ↓
Phase 1.5: AI REVIEW — Claude Code reviews task decomposition
    │ Review task granularity, dependencies, difficulty
    │ Revise based on feedback
    ↓
Phase 2: SETUP
    │ Create Kanban board
    │ Create working directory / git worktree
    │ Create TASK-过程记录.md process document
    ↓
Phase 3: AUTONOMOUS EXECUTION — autonomous-dev-loop cronjob
    │ Cronjob (every 5 min) wakes Hermes
    │ Hermes delegates to: Claude Code / Codex / Gemini CLI / OpenCode
    │ Writing → verification → kanban update → notify user
    ↓
Phase 4: PER-TASK VERIFICATION
    │ Non-implementer Agent verifies each completed task
    │ 3 failures → switch agent → still failing → "needs human"
    ↓
Phase 5: FINAL REVIEW
    │ Least-participating Agent does final review
    │ finishing-a-development-branch → merge/PR
    ↓
Phase 6: DELIVERY
    │ Complete process document
    │ Report to user
    ↓
Phase 6.1: AUTO EVALUATION（自动，无需用户触发）
    │ Generate retrospective JSON → goal-runs/
    │ Append to aggregate.jsonl + git commit
    │ Send evaluation report to user
    │ D8 score auto-calculation（≥3 samples）
```

---

## Phase 0: Understand the Goal

**Trigger:** User says `/goal [objective]`

### If Goal is Clear

Confirm with user:
```
Goal: [stated objective]
Stopping condition: [what "done" looks like]
Estimated scope: [big/small/medium]
Agent assignment: Claude Code (code) / Gemini CLI (visual) / Hermes (general)
Token budget: [unlimited / 50k / 100k / custom]  ← 设置则限，无设置则不限
Ready to decompose? (Y/N)

**Non-interactive / eval mode (GOAL_EVAL=1):** Auto-confirm → proceed directly to Phase 1.
```

**Token Budget 说明：**
- 不设置 → 无限 token，直到目标完成或用户中断
- 设置值 → 80% 预警 / 100% 暂停并通知用户决策（见"Token Budget 系统"章节）

### If Goal is Unclear

Invoke `brainstorming` skill first to clarify:
- What problem does this solve?
- What are the constraints?
- What does success look like?
- What are the boundaries?

**After brainstorming → invoke `task-decomp` to decompose.**

---

## Token Budget System

**每轮 cronjob 开始时检查 token 使用量。**

### Budget Modes

| 模式 | 行为 |
|------|------|
| **不设置** | 无限 token，直到目标完成、用户中断、或遇到真正的 blocker |
| **设置值** | 限流控制，见下方阈值规则 |

### Threshold Rules（设置预算时生效）

```
0%  ──────────────────────────────────
      开始执行，记录基准
                                        80% ─────────────
      ⚠️ 预警通知用户：
      "Token 使用已达 80%（已用 X / 预算 Y）
       剩余 Z，约可完成 N 个 task
       要继续吗？要加预算吗？"
                                        继续自主执行
                                               100% ─────
      🛑 暂停，通知用户：
      "Token 预算已耗尽（X / Y）
       已完成 N/M 个 tasks
       进度：████░░░░░░░ 60%
       决策：① 加预算继续 ② 暂停交付当前成果 ③ 中止"
```

### Budget Tracking File

在工作目录创建 `.goal-budget.json`：

```json
{
  "goal": "[goal name]",
  "budget_tokens": 100000,
  "warning_tokens": 80000,
  "used_tokens": 0,
  "last_updated": "YYYY-MM-DD HH:MM",
  "status": "active|warning|paused|complete",
  "progress": {
    "tasks_completed": 3,
    "tasks_total": 8
  }
}
```

### Token 使用估算

每次 delegate_task 后，根据返回的 usage 字段估算：

```python
# 估算每次 delegate_task 平均 token 消耗
avg_tokens_per_task = total_used / tasks_completed
remaining_tasks = total_tasks - tasks_completed
estimated_needed = avg_tokens_per_task * remaining_tasks
```

### 80% 预警通知格式

```
⚠️ Token Budget 预警

Goal: [name]
已用: 80,000 / 100,000 tokens (80%)
剩余约可完成: ~2 个 task

当前进度: 3/8 tasks (37%)
最近完成: [task name]

选项：
① 继续执行（可能在 100% 前完成）
② 增加预算 +50%
③ 暂停，交付当前成果
```

### 100% 暂停通知格式

```
🛑 Token Budget 耗尽

Goal: [name]
已用: 100,000 / 100,000 tokens (100%)
进度: 5/8 tasks (62%)

已完成：
✅ task 1: [name]
✅ task 2: [name]
...

未完成：
⬜ task 6: [name]
⬜ task 7: [name]
...

→ 暂停，发送此通知给用户，等待决策

选项：
① 加预算继续（推荐 +50% 或 +100%）
② 暂停交付（保留当前 branch/PR）
③ 中止（放弃本次执行）
```

### Soft Stop — 100% 前最后一个 Turn 的 Wrap-Up（Codex 启发）

**即使 budget 耗尽，也不要突然中断。但也绝不能因此 mark complete。**

**规则（来自 Codex continuation.md）：**
> *"Do not call update_goal unless the goal is complete. Do not mark a goal complete merely because the budget is nearly exhausted or because you are stopping work."*

→ **Budget 耗尽而目标未达 → 状态 = `budget_limited`，永远不能 mark complete。**

**硬规则：budget_limited → complete 的转换路径不存在。**

只有这一条路径可以变 `complete`：
```
active + completion audit 全绿 + update_goal(status="complete")
```

**以下任何一种都不能 mark complete：**
- ❌ token/time 耗尽（budget_limited）
- ❌ 用户要求停止
- ❌ 3 次 pivot 后放弃
- ❌ "快完成了，应该没问题了"
- ❌ "用户没意见了"

如果 budget 耗尽但目标未完成 → 状态 = `budget_limited` → 等用户决策（加预算/暂停交付/中止）。

100% 前最后一个 turn（或 budget 耗尽后第一次唤醒）：

→ 执行 Soft Stop：inject wrap-up steering

```
Wrap-Up Prompt（注入到下一个 agent prompt）：

Budget 即将耗尽。执行收尾：
1. 不要 start new substantive work
2. 把当前 in-progress 的 task 收尾到一个干净状态
   （commit 当前改动，附上清晰 message）
3. 在 TASK-过程记录.md 记录：
   - 进度百分比
   - 已完成 vs 未完成 task 列表
   - 明确的 next step（如果用户选择继续）
4. 通知用户当前状态，等待决策

→ 等用户决策（加预算 / 暂停交付 / 中止）
→ 绝不能因为"预算快没了"就 mark complete
```

---

## Goal 状态机（来自 Codex core runtime，issue #18076）

### 状态定义

```
┌─────────────┐
│  active    │ ← goal 运行中，可接受新 work
└──────┬──────┘
       │ interrupt / 用户暂停
       ▼
┌─────────────┐
│  paused    │ ← goal 暂停，可被 resume
└──────┬──────┘
       │ token/time budget 耗尽（从 active）
       ▼
┌─────────────┐
│budget_limited│ ← 软停止，注入 wrap-up steering
└──────┬──────┘
       │ 用户决策"加预算继续"
       ▼
┌─────────────┐
│  active    │ ← resume
└──────┬──────┘
       │ update_goal(status="complete")
       ▼
┌─────────────┐
│  complete   │ ← goal 结束
└─────────────┘
       │ update_goal(status="failed")
       ▼
┌─────────────┐
│  failed     │ ← 需要人工介入
└─────────────┘
```

### 状态转换规则

| 转换 | 触发条件 | Account Usage |
|------|----------|---------------|
| → `active` | goal 创建 / 用户"加预算继续" | reset baseline |
| → `paused` | 用户中断 / idle interrupt | ✅ checkpoint |
| → `budget_limited` | token/time 耗尽（仅从 active） | ✅ final delta |
| → `complete` | completion audit 全绿 + `update_goal` | ✅ final delta |
| → `failed` | 3 次 pivot 后仍失败 | — |

### 关键规则（来自 Codex P1 bugs）

```
⚠️ 必须 account usage，才能状态转换

状态转换时必须先做 usage snapshot，否则：
- 换 goal objective 时复用旧 objective 的 token 计数 → budget 不准
- 从 paused 切到 budget_limited 时丢失已消耗 token → 计数偏少

P1: token/time 耗尽时，如果 goal 处于 paused 而不是 active
    → 不能变成 budget_limited（SQL 只处理 active → budget_limited）
    → 必须先 resume 再耗尽，才能触发 budget_limited
    → 但这意味着 paused goal 可以超预算运行！

→ 正确：budget 耗尽检测要同时处理 active 和 paused 状态
```

### Cronjob 中的 Goal 状态检查

```python
# 每轮 cronjob 开始时
goal_status = read_goal_status()  # from .goal-state.json

if goal_status == "budget_limited":
    # 注入 wrap-up steering，不 start new substantive work
    inject_wrap_up_steering()
    notify_user_budget_limited()
    wait_for_user_decision()  # 加预算 / 暂停交付 / 中止
    return  # 等用户，不继续

elif goal_status == "paused":
    # 恢复 goal
    resume_goal()
    account_usage_checkpoint()

elif goal_status == "active":
    # 正常执行
    pass
```

### .goal-state.json 模板

```json
{
  "goal": "[goal name]",
  "status": "active|paused|budget_limited|complete|failed",
  "created_at": "YYYY-MM-DD HH:MM",
  "updated_at": "YYYY-MM-DD HH:MM",
  "usage": {
    "tokens_used": 50000,
    "time_seconds": 3600,
    "tasks_completed": 3,
    "tasks_total": 8
  },
  "last_transition": {
    "from": "active",
    "to": "budget_limited",
    "trigger": "token_budget_exhausted",
    "at": "YYYY-MM-DD HH:MM"
  }
}
```

---

## Phase 1: Decompose — task-decomp Skill

Use the `task-decomp` skill to break the goal into a sequence of tasks.

### Initializer + Executor 双代理模式（SUPERPOWERS autonomous-skill）

**为什么需要分离？**

单一 agent 同时做分解和执行会导致：
- 执行时被新任务打断，分解不完整
- 分解和执行用同一个思维模型，容易遗漏

**Initializer（分解代理）：**
- 分析目标，创建 `task_list.md`（主人任务清单）
- 将大目标分解为 phases/milestones/tasks
- 每个 task 有明确的 deliverables + acceptance criteria + verification command
- 创建 `.autonomous/<task-name>/` 子目录

**Executor（执行代理）：**
- 读取 task_list.md + progress.md
- 逐条完成并标记 `[x]`
- 更新 progress.md（每 session 进度笔记）
- 不自己做 task 分解，只执行已有清单

**文件结构（.autonomous/ 模式）：**
```
project-root/
└── .autonomous/<task-name>/
    ├── task_list.md      # Master checklist（只读描述，Executor 只标记 [x]）
    ├── progress.md       # Per-session progress notes
    └── sessions/         # Transcript logs per session
```

**Task List is Sacred（SUPERPOWERS 原则）：**
- task_list.md 的描述一旦写入，**只允许标记 `[x]`，禁止修改描述内容**
- 禁止删除 task、缩小 task 范围
- 这防止了"做着做着把任务缩小"的 scope creep

**Each task must have:**
1. **Name** — what this task achieves
2. **Deliverables** — exact files/artifacts produced
3. **Acceptance Criteria** — checklist with verification commands
4. **Constraints** — what NOT to change
5. **Agent assignment** — Claude Code / Gemini CLI / Hermes
6. **Dependencies** — which tasks must complete first

**Output:** `TASK-过程记录.md` with full task list and Kanban board.

---

## Phase 1.5: AI Review — Claude Code Reviews Decomposition

**Who:** Claude Code (via `claude-code` skill + `delegate_task`)

**What to review:**
- Are tasks the right granularity (15-60 min each)?
- Are acceptance criteria specific and verifiable?
- Are dependencies correctly identified?
- Is the difficulty estimate reasonable?
- Any tasks that could be parallelized?

**Prompt to Claude Code:**
```
Review this task decomposition for [goal]:
[tasks here]

Verify:
1. Each task is 15-60 min of focused work
2. Each acceptance criterion has a verification command
3. Dependencies are correct and minimal
4. No over-parallelization (tasks that must be sequential)
5. Agent assignments match task type (code → Claude Code, visual → Gemini CLI, general → Hermes)

Report: List specific improvements, then give overall verdict: APPROVED or NEEDS_REVISION
```

**If NEEDS_REVISION:** Incorporate feedback, then re-review once before proceeding.

---

## Phase 2: Setup

### 2.1 Create Kanban Board

```bash
hermes kanban create "[Goal Name]"
# Note the board ID
```

### 2.2 Create Working Directory + External Memory Files

```bash
mkdir -p /tmp/[goal-slug]/
cd /tmp/[goal-slug]/
git init
git commit --allow-empty -m "Initial commit"
```

**Create external memory files（每个都是 TASK-过程记录.md 的补充）：**

```
WORKDIR/
├── SPEC.md          # 目标 + 非目标 + 硬约束 + deliverable + done-when
├── PLANS.md         # milestones + acceptance criteria + verification commands
├── IMPLEMENT.md     # execution runbook，引用 PLANS.md
├── DOCUMENTATION.md # 实时状态 + decisions + known issues
├── TASK-过程记录.md # task list + execution log（主追踪文件）
└── .goal-budget.json  # token budget 追踪（仅当设置了 budget）
```

**SPEC.md 模板：**
```markdown
# [Goal] — SPEC

> **mode: greenfield**  <!-- greenfield=新功能大胆假设 | surgical=精准改动保守验证 -->
> **scope: full**        <!-- full=完整实现 | minimal=最小可用 -->

## Goal
[用户描述的目标]

## Non-Goals（明确不做这些）
- [item 1]
- [item 2]

## Hard Constraints
- [必须满足的条件]

## Deliverables
- [file 1]
- [file 2]

## Done When
- [ ] [具体验收标准 1]
- [ ] [具体验收标准 2]
```

**PLANS.md 模板：**
```markdown
# [Goal] — PLANS

## Milestones

### M1: [milestone name]
- **Tasks:** T1, T2, T3
- **Acceptance Criteria:**
  - [ ] criterion 1（验证命令：xxx）
  - [ ] criterion 2（验证命令：xxx）

## Verification Commands
```bash
pytest
npm test
```
```

### 2.3 Create Process Document

Create `TASK-过程记录.md`:

```markdown
# [Goal Name] — 过程记录

**Started:** YYYY-MM-DD
**Goal:** [objective]
**Stopping condition:** [what done looks like]
**Token Budget:** [unlimited / X tokens]

---

## Agent Assignment
- Claude Code → [scope]
- Gemini CLI → [scope]
- Codex/OpenCode → [scope]
- Hermes → orchestration, verification, final review

---

## Task List

| # | Task | Agent | Status | Verification |
|---|------|-------|--------|-------------|
| 1 | ... | Claude Code | done/ready/in-progress | [cmd] |
| 2 | ... | Gemini CLI | ... | [cmd] |

---

## Execution Log

### YYYY-MM-DD HH:MM — [Event]
[What happened]
```

---

## Hard Rules（来自 Codex Autoresearch，强制执行）

### 启动前 — Ask-Before-Act

```
所有问题 → BEFORE LAUNCH（Phase 0-1）
用户说 "go" / "start" / "launch" → LOOP 完全自主

启动后：
- ❌ 不暂停问问题
- ❌ 不暂停确认
- ❌ 不暂停请求权限
- ✅ 如果模糊 → 用最佳实践 + 记录推理到 TASK-过程记录.md
```

> **核心洞察："用户可能在睡觉。" 自治循环一旦启动就不应该停下来等用户。**

### Phase Transition Guards（边界条件补充）

Phase 之间的转换必须满足以下条件，否则禁止进入下一阶段：

```
┌─────────┐  用户说"go"/"start"/"launch"   ┌─────────┐
│ Phase 0 │ ─────────────────────────────→ │ Phase 1 │
└─────────┘  仅当用户明确授权             └─────────┘

┌─────────┐  Claude Code 审查结果=APPROVED ┌─────────┐
│ Phase 1 │ ─────────────────────────────→ │Phase 1.5│
└─────────┘  任意 NEEDS_REVISION → 修后再审 └─────────┘

┌─────────┐  以下全部满足：                 ┌─────────┐
│Phase 1.5│ ─────────────────────────────→ │ Phase 2 │
└─────────┘  ✅ Kanban board 已创建         └─────────┘
            ✅ task 状态非空
            ✅ 外部记忆文件框架已建立

┌─────────┐  所有 task 达到以下之一：       ┌─────────┐
│ Phase 3 │ ─────────────────────────────→ │ Phase 4 │
└─────────┘  ✅ done（已验证）              └─────────┘
            ✅ needs human（无法自动完成）

┌─────────┐  completion audit 全绿          ┌─────────┐
│ Phase 4 │ ─────────────────────────────→ │ Phase 5 │
└─────────┘  任意 FAIL → 返回 Phase 3       └─────────┘

┌─────────┐  finishing-a-development-branch ┌─────────┐
│ Phase 5 │ ──────────────────────────────→ │ Phase 6 │
└─────────┘  完成                           └─────────┘
```

**违反 Transition Guards 的后果：**
- 跳过 Phase 1.5 审查 → 视为违规，记录到 TASK-过程记录.md
- Phase 2 未建看板就进入 Phase 3 → 停止，通知用户
- Phase 3 未完成就进入 Phase 4 → 禁止，强制等待

### 执行中 — 每次迭代只做一个改变

```
每次 delegate_task / 每次 agent 执行：只做一个聚焦的改变
一个假设 → 一个改变 → 一个验证 → 记录结果 → 下一个

WRONG:  一次做 5 个改动，然后无法归因
RIGHT:  一次一个改动，证据清晰
```

### Scope Fidelity（禁止缩小目标）

```
用户说"实现 X"，不要因为 X 难做 / 难测试 / 改动大
就替换成"更安全的 X 版本"或"更小范围的 X"

WRONG:  因为更难，就用一个更窄/更易的方案替代
RIGHT:  保持用户要求的 end state，不因实现难度改变目标

"完成了更容易的版本" ≠ "完成了要求的版本"
```

### Alignment = 向最终状态移动

```
alignment = 向用户请求的最终状态移动

一个 edit 只有在让用户请求的最终状态更真实时才叫 aligned
"有用的行为但保留了不同的 end state" = misalignment

WRONG:  做出有用的行为，就觉得对目标有贡献
RIGHT:  每次改动必须让"请求的最终状态"更接近一步
```

### Verify Before Assumption（先 inspect，再 action）

```
每次决定下一步做什么之前：
→ 先 inspect current state（git log / 文件内容 / 命令输出）
→ 再决定 action

不要：
→ 假设改动前状态没变
→ 假设上次看到的代码还是现在这样
→ 假设某个 test 覆盖了某个 requirement（除非确认过）
```

### update_plan 纪律（单一 in_progress）

```
Task-过程记录.md 中的 plan_items：

规则：
1. 同一时间只能有 1 个 item 处于 in_progress
2. 完成当前 item → 立即 mark complete → 才能把下一个标为 in_progress
3. 绝对禁止：批量把 3 个 item 一次性标为 complete（事后归因不可能）
4. scope pivot（理解变了 / 拆分 / 合并 / 重排序任务）
   → 先更新 TASK-过程记录.md 的 plan，再继续，不要让 plan 变陈旧
5. plan 更新不是"做工作的替代品"——不要沉迷 plan 本身而不行动

WRONG:  一次 commit 把 5 个改动全部标记完成
RIGHT:  一个改动 → verify → mark complete → 下一个 → mark complete
```

### Dirty Worktree 检测（每次 agent 执行前）

```bash
git status --porcelain
```

**发现意外变化（不是你做的）→ 立即停止 → 问用户如何处理**
不要自己决定怎么处理，不要 revert，不要忽略，不要吸收到 commit 里。

| 状态 | 行动 |
|------|------|
| 空（干净） | 正常执行 |
| 有变化 + 属于当前 task | 继续，staged 变化 |
| 有变化 + 属于无关修改 | **STOP → 通知用户 → 等待决策** |

> **绝对不吸收无关的用户编辑到 agent 的 commits 中。**
> **发现 unexpected changes → STOP IMMEDIATELY → ask user**

### Bias to Action（行动偏好）

```
每次 rollout 必须以"一个具体 edit"或"一个明确 blocker + targeted 问题"结束
不要以"需要澄清"来结束 turn，除非真正被 block

WRONG:  "我觉得应该这样做，但不确定，你想要 X 还是 Y？"（没开始干活就停了）
RIGHT:  用最佳假设先做了，附上结论："我按 A 做了，如果不对告诉我"

除非真正被 block（缺信息/缺权限/不可逆），否则不要提前问问题
```

### Plan Closure（每次结束前必查）

```
每个 intention/TODO/plan item 必须标记为以下三种之一：

✅ Done — 完成，有 evidence
🚫 Blocked — 被 block，附一句话原因 + targeted 问题
❌ Cancelled — 取消，附原因

禁止：
→ 以 in_progress 结束 turn
→ 以 pending 结束 turn（没有解释为什么还没做）
```

### 破坏性 Git 命令（硬规则，永远）

```
NEVER（除非用户明确要求）：
- git reset --hard
- git checkout -- [file]
- git commit --amend
- git rebase -i（危险）
- 任何 destructive / irreversible 操作

原因：用户的改动可能丢失，且无法恢复
```

### 过度循环检测

```
如果发现自己：
- 反复读取同一个文件
- 反复编辑同一个文件
- 没有任何明确进展却一直在工具调用

→ 立即停下来
→ 在 TASK-过程记录.md 记录当前状态
→ 附上：进展到什么地步 / 卡在哪里 / targeted 问题是什么
→ 结束这个 agent turn，等下一个 cronjob 唤醒再继续
```

### Condition-based Waiting（条件等待，SUPERPOWERS systematic-debugging）

**遇到等待场景时，不要用固定 timeout 猜测。**

```
WRONG:  sleep 5 && assume it's ready
RIGHT:  while ! condition_is_met; do sleep 0.5; done
```

例如：等服务启动 → 轮询健康检查 endpoint，不是在日志里猜"应该快了"。

### Escalation（主动升级风险）

```
当决策有非显而易见的后果或隐藏风险时：

→ 不要悄悄继续
→ 不要自己判断"应该没问题"
→ 主动升级给用户，用这个格式：

⚠️ 需要决策：[描述风险/权衡]
  选项 A：[利]
  选项 B：[弊]
  我的倾向：[理由]
  等待用户回复...
```

### Approval Mode 感知

```
Agent 的 approval mode 影响测试行为：

never / on-failure（非交互模式）：
→ 主动运行测试/lint/验证，确保任务完成
→ 不需要等用户确认

untrusted / on-request（交互模式）：
→ 建议想做什么，等用户确认后再跑测试
→ 不要自己跑（会拖慢迭代速度）

test-related tasks（测试相关任务）：
→ 无论什么模式，都可以主动跑测试
```

**量化判断：**
```
if mode == "never" or mode == "on-failure":
    # 非交互 → 主动测试，不等用户
    run_tests_unless_failure()
elif mode == "untrusted" or mode == "on-request":
    # 交互 → 先提案，等确认
    propose_and_wait_for_confirmation()
```

### 野心 vs 精度（上下文感知）

```
任务类型不同，策略不同：

新任务 / 无现有代码库约束（SPEC.md mode: greenfield）：
→ 可以大胆创造、实验、提出新方案

现有代码库 / 已有明确范围的任务（SPEC.md mode: surgical）：
→ 手术精度：用户要什么做什么，不要多做
→ 不要因为觉得"这样更好"就擅自改用户没要求的部分
→ 不要加"有用但不在 scope 里"的功能

判断标准：这次改动是否让"用户要求的 end state"更接近？
是 → 做；否 → 不做
```

**量化决策矩阵：**

| SPEC mode | scope | 行为策略 | 改动阈值 |
|-----------|-------|---------|---------|
| `greenfield` | `full` | 大胆假设，主动建议新方案 | 任意有潜力的改进都尝试 |
| `greenfield` | `minimal` | 大胆假设 + 快速验证 | 最小改动先跑通再说 |
| `surgical` | `full` | 精准执行，scope 内做到极致 | scope 内全部覆盖，scope 外 0 容忍 |
| `surgical` | `minimal` | 最小可用，scope 外 0 容忍 | 只做最低完成标准，边界不变动 |

**判断伪代码：**
```
if spec_mode == "surgical":
    # surgical 模式：scope 是硬边界
    if改动_in_scope:
        全力实现
    else:
        discard  # 哪怕很有价值也不做
elif spec_mode == "greenfield":
    # greenfield 模式：鼓励探索
    if 改动_has_potential:
        尝试
```

### Action Safety（行动前先 call out 风险）

```
执行有风险或不可逆的行动之前：
→ 先在 TASK-过程记录.md 记录：我要做什么 / 风险是什么 / 为什么必须现在做
→ 然后再执行

绝对不要在用户不知情的情况下：
→ 删除大量代码
→ 修改生产配置
→ 改动共享的基础模块
→ 执行有副作用的数据库操作
```

### Tool Persistence（工具坚持规则）

```
继续使用工具，直到有足够证据自信完成任务

部分读取后就放弃 → 不要
当另一个 targeted check 可能改变答案时 → 不要停止

WRONG:  看了 3 行代码就开始写修复，没看完整个相关文件
RIGHT:  读完所有相关文件，确认理解完整，再动手
```

### Dig Deeper（深层检查，找到问题后）

```
找到第一个 plausible issue 后，继续检查：

1. 二阶失效 — 这个 bug 会引发其他什么 bug？
2. 空状态行为 — 数据为空时行为正确吗？
3. 重试逻辑 — 失败重试时会发生什么？
4. 陈旧状态 — 有没有缓存或旧数据导致的假象？
5. 回滚路径 — 如果这个改动错了，怎么撤回？

然后再 finalize 结果
```

### No-tool Turn 也能继续（Codex #20523 修复）

**不要因为"一个 turn 没有工具调用"就认为 agent 卡住了。**

Codex 之前错误地用"no registry tool calls"作为"应该停止"的启发式信号，导致 agent 在做理解/规划/等待时就被停止。

正确的判断：
- ✅ agent 在做理解、规划、等待条件 → 继续
- ✅ agent 在思考下一步怎么做 → 继续
- ❌ agent 在重复同样的 action 且没有进展 → 触发过度循环检测

如果 agent 一个 turn 没有工具调用：
1. 检查 git log — 看是否有有意义的 commit
2. 检查 TASK-过程记录.md — 看是否有进展记录
3. 只有当没有任何有意义进展时才停止

### 3 次失败后 → Pivot，不暴力重试

```
3 failures (same task)
  → 换 Agent 做 2 次
  → 还失败 → PIVOT
  → 换思路，而不是重复同样的尝试
```

### 增量 <1% 且显著增加复杂度 → Discard

```
如果改进 < 1% 且代码复杂度显著增加：
→ 放弃这个改进，记录 "discard: 收益 < 1%"
→ 继续下一个方向
```

**量化判断（每次改动后评估）：**

```python
def should_discard(change):
    """
    change: {delta_lines, delta_complexity, delta_requirement_match}
    """
    # 收益计算
    benefit = change["delta_requirement_match"]  # 需求匹配率提升 (0.0-1.0)
    # 成本计算
    cost = change["delta_complexity"]  # 圈复杂度变化
    lines = change["delta_lines"]      # 代码行数变化

    # 收益/成本比
    if cost > 0:
        efficiency = benefit / cost
    else:
        efficiency = benefit  # 负成本（简化）= 直接保留

    # Discard 阈值
    if efficiency < 0.01 and lines > 20:
        return "discard", f"收益 {benefit:.3f} / 成本 {cost} < 1%"
    return "keep", f"efficiency={efficiency:.3f}"
```

**简单版判断（无需量化）：**
- 这个改动"用户能看到"吗？→ 否 → discard
- 改动 > 50 行但收益不确定 → discard
- 不确定时 → 默认 discard（保守）

---

## Phase 2→3 Launch Gate（强制确认清单）

**在进入 Phase 3 之前，必须确认以下所有项目。全部 ✓ 才能继续；有 × → 修复后再继续。**

```
Launcher Checklist（发送给我，等 confirm 或修改意见）：

□ SPEC.md 存在且完整（Goal + Acceptance Criteria 明确）
□ PLANS.md 存在且可执行（至少1个 milestone，验收标准具体）
□ Task List 完整（所有 task 有 agent 分配 + verification 命令）
□ 没有悬空 task（done/ready/in-progress/blocked 以外的 Status）
□ Dirty Worktree 已处理（git status 干净，或 staged 属于当前 task）
□ 依赖关系已确认（依赖链无环，ready 的 task 不依赖 blocked 的 task）

[可选，如适用]
□ Token Budget 已设定（estimate 是多少？有 buffer 吗？）
□ Verifier 已分配（Verifier ≠ Implementer 确认了吗？）

回复 'confirm' 继续 Phase 3，或指出需要修改的地方。
```

> **注意**：Phase 0-1 是人在回路的最后一站。这里的确认不是形式审查——是最后一道质量门。

---

## Phase 3: Autonomous Execution — autonomous-dev-loop

### Create the Cronjob

```bash
hermes cronjob create \
  --name="[Goal] — Autonomous Dev Loop" \
  --prompt="[See autonomous-dev-loop skill for prompt template]" \
  --schedule="*/5 * * * *" \
  --repeat=100 \
  --skills="kanban-orchestrator,task-decomp,claude-code,codex,gemini-cli,autonomous-dev-loop" \
  --deliver="origin" \
  --workdir="[workdir]"
```

### Cronjob Behavior (per run)

```
1. 读 .goal-state.json → 检查 goal status
   ├─ budget_limited → inject wrap-up steering → notify user → 等待决策
   ├─ paused → resume_goal() + account_usage_checkpoint()
   └─ active → 继续
2. 读 .goal-budget.json（如果设置了 token budget）
   ├─ 读取 used_tokens
   ├─ 80% ≤ used < 100% → 发送预警通知 → 继续
   └─ used ≥ 100% → 触发 budget_limited 状态转换 → 注入 wrap-up → 通知
3. Verify Before Assumption：先 inspect current state，再决定 action
   - git log --oneline -5（看最后几个 commit 是什么）
   - git status --porcelain（看 worktree 干不干净）
   - TASK-过程记录.md（确认当前 task 状态）
4. Dirty worktree 检测：git status --porcelain
   └─ 有无关变化 → 停止，通知用户
5. 读 TASK-过程记录.md（task_list）→ 找 "ready" 状态 task
   └─ task_list.md 是唯一事实来源，Executor 不自己做分析
6. 对每个 ready task（单一 in_progress 纪律）：
   - Dirty worktree 再检测
   - delegate_task(goal=..., toolsets=['terminal','file','web'], role='leaf')
   - Agent implements → commits → reports
   - 更新 TASK-过程记录.md（立即 mark complete，不 batch）
   - 更新 .goal-budget.json 的 used_tokens（如设置了 budget）
   - Log to TASK-过程记录.md
   - 发送 Feishu DM 通知用户
7. 如果 task 依赖未完成 → 跳过，通知依赖方
8. 所有 tasks 完成 → 触发 Phase 5

### Agent Selection Matrix（集中决策表）

以下所有决策规则均汇总于此，其他章节引用本表。

#### A. Implementer 选择

| Task Type | Primary | Fallback 1 | Fallback 2 |
|-----------|---------|------------|------------|
| Code implementation | Claude Code | Codex | OpenCode |
| Deep refactor（跨模块重构） | Codex | Claude Code | OpenCode |
| Visual/UI/审美 | Gemini CLI | Claude Code | — |
| General/process/coordination | Hermes | — | — |
| Script/automation | Codex | Claude Code | — |
| Research/exploration（选型调研） | Gemini CLI | Hermes | — |
| Security review | Claude Code | Codex | Hermes |

> **注意：** "deep refactor" 行是 Round 4 案例教训（T4：Claude Code 3次失败，Codex 1次过），结构性修复不可依赖规则约束。

#### B. Verifier 选择（≠ Implementer + 能力匹配）

| Implementer | 首选 Verifier | 备选 Verifier | 理由 |
|-------------|--------------|---------------|------|
| Claude Code | Hermes | Gemini CLI | Hermes 无 terminal → 纯审查视角 |
| Gemini CLI | Claude Code | Hermes | Claude Code 有完整 terminal → 可跑测试 |
| Hermes | Claude Code | — | 代码任务验证需要 terminal |
| Codex | Claude Code | Hermes | 同上 |

**选择原则：**
1. **≠ Implementer**（强制）
2. **有 terminal 能力的 agent 优先验证代码任务**（Claude Code > Codex > Hermes）
3. **无 terminal 的 agent 适合纯审查/分析任务**

**硬规则：Verifier ≠ Implementer。不同模型/Provider 强制执行。**

#### C. Approval Mode 行为

| Mode | Test/Lint 行为 | 确认要求 |
|------|---------------|---------|
| `never` / `on-failure`（非交互） | 主动运行，确保任务完成 | 不需要 |
| `untrusted` / `on-request`（交互） | 建议想做什么 | 等用户确认后再跑 |
| 测试相关任务 | 无论什么模式 | 可主动跑测试 |

#### D. Dirty Worktree 响应

| 状态 | 行动 |
|------|------|
| 空（干净） | 正常执行 |
| 有变化 + 属于当前 task | 继续，staged 变化 |
| 有变化 + 属于无关修改 | **STOP → 通知用户 → 等待决策** |

#### E. Retry Strategy

| 失败次数 | 行动 |
|---------|------|
| 1-2 次（同 task，同 agent） | 继续重试 |
| 3 次（同 task，同 agent） | 换 fallback agent |
| 再 2 次失败 | Mark "needs human" + 立即通知用户 |
| 3 次 fix 均失败 | STOP → 质疑架构 → 升级给用户 |

#### F. Condition-based Waiting

```
WRONG:  sleep 5 && assume it's ready
RIGHT:  while ! condition_is_met; do sleep 0.5; done
```

等待外部条件时，轮询健康检查或状态文件，不用固定 timeout 猜测。

#### G. Goal State → Cronjob 行为

| Goal State | Cronjob 动作 |
|------------|-------------|
| `active` | 正常执行 |
| `paused` | resume_goal() + checkpoint，继续 |
| `budget_limited` | inject wrap-up steering → 通知用户 → 等待决策 |
| `complete` | 停止 cronjob，触发 Phase 5 |
| `failed` | 停止，通知用户 |

---


### Agent 报告成功 ≠ 成功（强制查 VCS diff）

> *"Agent reports success → Check VCS diff → Verify changes → Report actual state"*

每次 delegate_task 完成后，必须强制检查 git diff：

```
1. Agent reports: "Task N complete"
2. Hermes runs: git diff --stat
3. Hermes verifies: diff matches expected deliverables
4. Hermes states: "Confirmed: [files] modified, [lines] changed"
5. If diff is empty or wrong → FAIL, re-dispatch
```

**禁止：** 信任 Agent 的"success"报告，不经验证就认为完成。

### Task List is Sacred（SUPERPOWERS 原则）

看板 task 的描述一旦写入，只允许：
- ✅ 标记 `[x]`（完成）
- ✅ 更新状态（ready → in-progress → done）
- ❌ **禁止修改 task 描述内容**
- ❌ **禁止删除 task**
- ❌ **禁止缩小 task 范围**（把难做的大 task 改成小task）

### Retry Strategy

```
3 failures (same task, same agent)
  → Switch to fallback agent
  → 2 more attempts
  → Still failing
    → Mark task "needs human"
    → Notify user immediately
    → Continue with independent tasks
```

**3 次修复后质疑架构（来自 SUPERPOWERS systematic-debugging）：**

如果同一个问题用了 3 次 fix 还修不好：
- 停止继续打补丁
- 在 TASK-过程记录.md 记录：
  - 症状 / 根因假设 / 已尝试的修复 × 3
  - **STOP → 质疑架构**：这个问题的根本是不是系统设计问题？
- 升级给用户："这可能不是个 bug，而是架构问题，要重构还是要继续打补丁？"
- 不要把"打了 3 次补丁"当成正常迭代，那是架构预警信号。

---

## Phase 4: Per-Task Verification

**Rule:** Verifier ≠ Implementer (different model/provider)

### Verification Flow

```
Implementer completes task → 
  Verifier (different agent) checks:
    1. All acceptance criteria met?
    2. Verification commands pass?
    3. No side effects on other tasks?
  → PASS → Mark done, next task
  → FAIL → Return to implementer with specific gaps
```

### Verification Agent Assignment

- **Claude Code tasks** → Hermes or Gemini CLI verifies
- **Gemini CLI tasks** → Claude Code or Hermes verifies
- **Hermes tasks** → Claude Code verifies
- **Codex tasks** → Claude Code or Hermes verifies

### Logging Verification

```
## Verification — Task N
**Verifier:** [Agent]
**Result:** PASS / FAIL / NEEDS_REVISION
**Evidence:** [verification command output]
**Date:** YYYY-MM-DD HH:MM
```

---

## Phase 5: Final Review

**Who:** The Agent that participated LEAST in this goal (fresh perspective)

### Completion Audit（必须逐条验证，不能凭感觉）

**核心原则（来自 Codex continuation.md）：Treat completion as unproven.**

> *"Before deciding that the goal is achieved, **treat completion as unproven** and verify it against the actual current state."*

**不许的信念：**
- "快了，应该快完成了"
- "测试全绿，应该没问题了"
- "改了这么多，肯定完成了"
- "用户没意见就是过了"

**正确的态度：Completion 从来不是信念，而是必须用证据逐步证明的命题。**

### Completion 作为"未证明的假设"（Codex continuation.md 原文）

> *"Before deciding that the goal is achieved, **treat completion as unproven** and verify it against the actual current state."*

**不等式：**
```
信念 ≠ 证据
进度感 ≠ 证据
测试全绿 ≠ Goal 完成
实现努力 ≠ 完成
代理报告成功 ≠ 实际完成
```

**唯一的完成标准：证据覆盖了目标中每一个明确的交付物。**

---

**Before marking goal complete — perform a completion audit:**

```
For EVERY explicit requirement from the original goal:
1. Derive the requirement (what did the user explicitly ask for?)
2. Identify authoritative evidence: files / cmd output / test results / runtime behavior
3. Determine:
   ✅ PROVES completion — evidence shows requirement is satisfied
   ❌ CONTRADICTS completion — evidence shows requirement is NOT satisfied
   ⏳ INCOMPLETE — partial work, not fully done
   ❓ MISSING — no evidence found
   ⚠️ TOO WEAK — evidence is indirect/weak for the scope of the claim

**不接受代理信号（Codex continuation.md 原文）：**

> *"Passing tests, a complete manifest, a successful verifier, or substantial implementation effort are useful evidence **only if** they cover every requirement in the objective."*

| 代理信号 | 为什么不够 | 正确做法 |
|---------|----------|---------|
| 测试全绿 | 可能没覆盖所有 requirement | 必须逐条确认测试覆盖了每个 requirement |
| 完整 manifest | manifest 本身不等于交付完成 | 打开文件，确认实际内容符合 spec |
| verifier 通过 | verifier 可能范围不够 | 独立检查 evidence |
| 实现投入了大量 effort | effort ≠ 结果 | 只看最终 state |
| Agent 说"完成了" | Agent 可能误判 | 必须查 VCS diff 验证 |

**每个 requirement 必须有直接的、具体的 evidence。不是间接信号，不是代理信号。**
```

### Gate Function — 五步验证（SUPERPOWERS verification-before-completion）

在声称任何状态之前（包括"完成"、"通过"、"没问题"），必须执行五步 Gate：

```
BEFORE claiming any status:

1. IDENTIFY — What command/file/proof proves this claim?
2. RUN    — Execute the FULL command (fresh, complete run)
3. READ   — Full output, check exit code, count failures
4. VERIFY — Does output actually confirm the claim?
5. STATE  — If YES: claim WITH evidence / If NO: state actual status with evidence

跳过任何一步 = 作弊，不是验证。
```

**示例对比：**
```
✅ [Run pytest] [See: 34/34 pass] "All tests pass"
❌ "Should pass now" / "Looks correct" / "Tests were green before"
```

**Red Flags — 立即停止：**
- 使用 "should", "probably", "seems to"
- 还没 run 验证就说 "Great!" / "Perfect!" / "Done!"
- commit/push/PR 前没验证
- 信任 agent 的"success"报告
- 部分验证就当全部验证
- "这次例外"

**Examples:**
```
Requirement: "API must support pagination"
Evidence: pytest passes → ❌ DOES NOT prove — tests don't verify pagination exists
Evidence: GET /api/users?page=2 returns {"items": [...], "has_next": true} → ✅ PROVES completion

Requirement: "All existing tests pass"
Evidence: pytest → 100% passed → ✅ PROVES completion
```

### Final Review Checklist

After completion audit passes:

```
1. ✅ Completion audit: ALL requirements proven met
2. All verification commands pass?
3. Process document complete and accurate?
4. Any technical debt introduced?
5. Documentation updated?
6. Tests comprehensive?
7. Git history clean (relevant commits only)?
```

**Then invoke `finishing-a-development-branch`:**
- Merge to main? → local merge + test
- Push and create PR? → `gh pr create`
- Keep branch? → report location

### Complete Process Document

将以下内容写入 `TASK-过程记录.md`（如果尚未完成）：

```markdown
## 执行摘要

**Goal:** [目标]
**完成时间:** YYYY-MM-DD HH:MM
**总耗时:** X 小时 Y 分钟
**最终状态:** ✅ 完成 / ⚠️ 部分完成 / ❌ 需要人工介入

## Task 执行记录

| Task | Agent | 状态 | 耗时 | 备注 |
|------|-------|------|------|------|
| T1   | Claude Code | ✅ | 3m | calc_as_required |
| T2   | Codex | ✅ | 5m | 安全审查 |
| ...  | ...   | ... | ... | ... |

## Agent 使用统计

- Claude Code: N tasks, 平均耗时 X 分钟
- Gemini CLI: N tasks, ...
- Codex: N tasks, ...
- OpenCode: N tasks, ...

## 关键Pivot / Exception 记录

- T4: Claude Code 3次失败 → 换 Codex 成功
- T5: pivot（许可证问题）

## 最终交付物

- [file 1]: [描述]
- [file 2]: [描述]

## Darwin 自评（Phase 6.1）

评分: X/100（D8=0，待样本积累）
详细报告: `~/.hermes/goal-runs/run_{timestamp}__{slug}.json`
```

**如果 TASK-过程记录.md 已有完整内容：** 确认、补充缺失字段即可，不需要重写。

---

### Report to User

```
✅ Goal complete: [name]

Tasks: N completed | N failed
Duration: X hours
Agents used: Claude Code / Gemini CLI / Codex / Hermes

Deliverables:
- [file 1]
- [file 2]

Process document: [path]
```

---

## Phase 6.1: Automatic Evaluation — Retrospective + Darwin Self-Assessment

**触发时机：** Phase 6 Delivery 完成后（每个 /goal 只执行一次）

**目的：** 自动生成执行画像 + Darwin 自评 + 可操作改进建议

---

### 6.1.1 收集执行数据

按以下顺序从各文件读取数据：

**步骤 1：从 TASK-过程记录.md 提取任务数据**

```python
import re
from datetime import datetime

doc_path = work_dir / "TASK-过程记录.md"
content = doc_path.read_text()

# 提取 Task Table 行
task_rows = re.findall(r'\| (T\d+) \| (\w+) \| ([✅❌⚠️]) \| (\d+m|\d+h\d*m) \| (.+) \|', content)

# 提取 Agent 统计
agent_blocks = re.findall(r'(\w+): (\d+) tasks.*?', content)

# 提取 Pivot / Exception 记录
pivots = re.findall(r'(T\d+): (.+)', content.split("## 关键Pivot")[1].split("##")[0] if "## 关键Pivot" in content else "")

# 提取开始/结束时间
start_match = re.search(r'\*\*开始时间:\*\* (.+)', content)
end_match = re.search(r'\*\*完成时间:\*\* (.+)', content)
```

**步骤 2：从 .goal-budget.json 提取 token 数据（如果存在）**

```python
budget_file = work_dir / ".goal-budget.json"
if budget_file.exists():
    import json
    budget = json.loads(budget_file.read_text())
    # budget_used, budget_pct, budget_mode
```

**步骤 3：从 Kanban board 提取任务状态**

```bash
hermes kanban list --json | jq '.tasks[] | {id, status, agent}'
```

**步骤 4：从 git log 提取提交记录**

```bash
git -C "$work_dir" log --oneline --format="%h %ae %s" -20
```

**数据汇总原则：**
- TASK-过程记录.md 为主要数据源（最完整）
- Kanban board 数据用于交叉验证
- Git log 用于补充验证提交历史
- 如果 TASK-过程记录.md 不完整或不存在，用 Kanban + git 数据重建，但标记 `"data_reconstructed": true`

---

### 6.1.2 生成 Retrospective JSON

输出到 `~/.hermes/goal-runs/run_{timestamp}__{goal-slug}.json`：

```json
{
  "run_id": "uuid-v4",
  "goal": "[原始目标]",
  "goal_slug": "[slug]",
  "started_at": "YYYY-MM-DD HH:MM",
  "ended_at": "YYYY-MM-DD HH:MM",
  "duration_minutes": 47,

  "token_budget": {
    "set": 100000,
    "used": 73000,
    "pct": 73,
    "mode": "soft_limit",
    "overrun": false
  },

  "task_stats": {
    "total": 9,
    "done": 8,
    "blocked": 0,
    "needs_human": 1,
    "completion_rate": 0.89
  },

  "agent_stats": {
    "claude_code": { "assigned": 5, "done": 4, "failed": 1 },
    "codex":        { "assigned": 2, "done": 2, "failed": 0 },
    "gemini_cli":   { "assigned": 1, "done": 1, "failed": 0 },
    "hermes":       { "assigned": 3, "done": 3, "failed": 0 }
  },

  "phase_transitions": [
    { "from": "phase_0", "to": "phase_1", "trigger": "user_confirm", "at": "HH:MM" },
    { "from": "phase_1", "to": "phase_2", "at": "HH:MM" },
    { "from": "phase_2", "to": "phase_3", "trigger": "launch_gate_confirmed", "at": "HH:MM" },
    { "from": "phase_3", "to": "phase_4", "at": "HH:MM" },
    { "from": "phase_4", "to": "phase_5", "trigger": "all_tasks_verified", "at": "HH:MM" },
    { "from": "phase_5", "to": "phase_6", "at": "HH:MM" }
  ],

  "decomposition_quality": {
    "spec_fulfilled": true,
    "plan_fulfilled": false,
    "task_gaps": ["T3 范围中途变大", "T7 被 T5 依赖导致串行"],
    "decomposition_failures": [
      { "task": "T5", "reason": "低估了 API 复杂度", "rework": "拆成 T5a+T5b" }
    ]
  },

  "completion_audit": {
    "conducted": true,
    "caught_gaps_before_delivery": 2,
    "gaps_found": ["漏了输入校验", "错误消息不一致"],
    "all_requirements_met": false,
    "requirement_match_rate": 0.85
  },

  "rule_violations": [
    { "rule": "Dirty Worktree Guard", "detected": true, "action": "stopped_notified" }
  ],

  "rule_effectiveness": [
    { "rule": "Verify Before Assumption", "triggered": 6, "prevented_mistake": 4, "missed": 1 }
  ],

  "agent_decisions": [
    {
      "task": "T4",
      "assigned": "Claude Code",
      "outcome": "failed_after_3_retries",
      "should_have_been": "Codex",
      "reason": "T4 是 deep refactor，Codex 的 deep search 能力更强"
    }
  ],

  "exceptions": [
    {
      "type": "retry_exhausted",
      "task": "T4",
      "agent": "Claude Code",
      "attempts": 3,
      "errors": ["类型不匹配", "边界条件错误"],
      "resolution": "switched_to_codex"
    },
    {
      "type": "pivot",
      "task": "T2→T2b",
      "reason": "第三方库许可证问题",
      "new_approach": "用标准库重写"
    },
    {
      "type": "escalation",
      "task": "T8",
      "reason": "数据库选型需要业务决策",
      "user_decision": "选择了 PostgreSQL"
    }
  ],

  "darwin_self_assessment": {
    "d1_frontmatter": 7,
    "d2_workflow_clarity": 13,
    "d3_boundary_coverage": 9,
    "d4_checkpoint_design": 7,
    "d5_instruction_specificity": 13,
    "d6_resource_integration": 5,
    "d7_architecture": 14,
    "d8_measurable_effects": 0,
    "total": 68,
    "assessor": "hermes",
    "note": "d8 由累积样本自动计算，首次执行为 0"
  },

  "lessons_learned": [
    "T5 分解粒度不够细，下次遇到 API 集成类任务，预估工时翻倍"
  ],

  "improvement_suggestions": [
    {
      "priority": "high",
      "dimension": "D5",
      "issue": "Agent Selection Matrix 对 deep refactor 任务分配不准",
      "evidence": "T4 Claude Code 3次失败，Codex 一次过",
      "fix": "在 Implementer 表增加 'deep refactor' 行，primary=Codex"
    }
  ],

  "git_commits": ["abc1234", "def5678"],
  "workdir": "/tmp/goal-slug"
}
```

---

### 6.1.3 追加到累积样本库

```python
import json
from pathlib import Path

run_json_path = Path("~/.hermes/goal-runs/run_{timestamp}__{slug}.json").expanduser()
aggregate_path = Path("~/.hermes/goal-runs/aggregate.jsonl").expanduser()

# 读取刚生成的 retrospective
run_data = json.loads(run_json_path.read_text())

# 构造 aggregate 行（只取关键字段，避免冗余）
aggregate_line = {
    "run_id": run_data["run_id"],
    "goal_slug": run_data["goal_slug"],
    "completion_rate": run_data["task_stats"]["completion_rate"],
    "needs_human": run_data["task_stats"]["needs_human"],
    "agent_success_rate": run_data["agent_stats"]["success_rate"],
    "darwin_total": run_data["darwin_self_assessment"]["total"],
    "d8_score": run_data["darwin_self_assessment"]["d8_measurable_effects"],
    "hard_rule_triggers": run_data["hard_rule_triggers"],
    "exceptions": run_data["exceptions"],
    "improvement_count": len(run_data["improvement_suggestions"]),
    "ended_at": run_data["ended_at"],
}

# 校验格式 + 追加到 aggregate.jsonl（带锁防并发）
import fcntl
with aggregate_path.open("a") as f:
    fcntl.flock(f.fileno(), fcntl.LOCK_EX)
    f.write(json.dumps(aggregate_line, ensure_ascii=False) + "\n")
    fcntl.flock(f.fileno(), fcntl.LOCK_UN)

# git commit
import subprocess
subprocess.run(["git", "-C", str(aggregate_path.parent), "add", "."], check=False)
subprocess.run(["git", "-C", str(aggregate_path.parent), "commit", "-m",
    f"goal-run: {run_data['goal_slug']} {run_data['ended_at']}"], check=False)
```

**aggregate.jsonl** 是流式追加格式（JSON Lines），方便后续分析：

```bash
# 分析累积数据
cat ~/.hermes/goal-runs/aggregate.jsonl | jq '.darwin_total, .completion_rate'

# 计算 D8（当样本 ≥3 时）
python3 -c "
import json, statistics
lines = [json.loads(l) for l in open('~/.hermes/goal-runs/aggregate.jsonl')]
if len(lines) >= 3:
    rates = [r['completion_rate'] for r in lines]
    matches = [r.get('requirement_match_rate', r['completion_rate']) for r in lines]
    humans = [r['needs_human'] for r in lines]
    d8 = min(10, round(statistics.mean(rates)*4 + statistics.mean(matches)*3 + (1-statistics.mean(humans))*2, 1))
    print(f'D8 = {d8}/10（基于 {len(lines)} 条样本）')
"

---

### 6.1.4 生成人类可读评估报告

发送给我（用户）：

````
📊 /goal 执行评估报告
━━━━━━━━━━━━━━━━━━━━━

Goal: [目标]
耗时: 47 分钟 | Token: 73% 使用
完成度: 8/9 tasks (89%) | Agent 成功率: 87%

🔴 例外情况（需要关注）
  - T4 Claude Code 3次失败后换 Codex
  - T2 因许可证问题 pivot
  - 1个 task 需要人工介入

🟡 分解质量
  - SPEC 交付: ✅ 符合
  - PLAN 里程碑: ⚠️ 1个未完成
  - 分解失误: T5 低估复杂度，T5→T5a+T5b 拆分

🟢 Hard Rule 表现
  - Dirty Worktree Guard: 1次触发（正常）
  - Verify Before Assumption: 6次触发，挡掉4个错误
  - Plan Closure: 2次Blocked标注到位

🟡 Agent 分配评估
  - T4 应分配 Codex（Claude Code 失败）
  - Matrix 需补充 "deep refactor" 行

📈 Darwin 自评: 68/100（D8=0，因样本不足）

💡 改进建议（优先级排序）
  [HIGH] D5: Agent Selection Matrix 增加 deep refactor 类型
  [MED]  D3: 边界覆盖 — 增加 "T5类 API 集成任务" 边界条件
  [LOW]  D7: 三车道持久模型 — 首次运行未触发 compaction

━━━━━━━━━━━━━━━━━━━━━
完整报告: ~/.hermes/goal-runs/run_{timestamp}__{slug}.json
````

---

### 6.1.5 D8 累积分数自动计算规则

当 `aggregate.jsonl` 积累 ≥3 条样本后，自动计算 D8：

```python
# D8: 可测量效果（目标结果 vs 期望）
d8_score = (
  avg_completion_rate   * 4 +   # 完成率（0-1）×4
  avg_requirement_match * 3 +   # 需求匹配率 ×3
  (1 - avg_needs_human) * 2 +   # 人工介入越少越好 ×2
  avg_rule_effectiveness * 1    # 规则有效率 ×1
)
# 满分 10，上限 10
```

---

**存储结构：**
```
~/.hermes/goal-runs/
├── run_2026-05-12_141522__calc-cli_abc123.json   # 每次 run 的详细记录
├── aggregate.jsonl                                  # 流式追加，所有 run 的汇总
└── .git/                                            # 可选，git 跟踪历史
```

---

### 6.1.6 Phase 6.1 执行时机

```
Phase 6 Delivery
    ↓
Phase 6.1 Retrospective（自动，无需用户触发）
    ↓
Phase 6.1.7 自动改进触发（核心：闭环形成）
    ↓
发送评估报告给用户
    ↓
等待用户反馈（确认/修改建议）
    ↓
如有修改意见 → 更新 SKILL.md（进入下一轮达尔文优化）
```

---

### 6.1.7 自动改进触发 — 进化闭环核心

**目的：** 当同一类问题重复出现时，自动生成 SKILL.md patch 提案，而不是永远"等用户反馈"。

**触发条件：** 同一 `dimension`（D1-D8）的 `improvement_suggestion` 在最近 N 次 run 中出现 ≥2 次。

```python
import json
from pathlib import Path
from collections import defaultdict

aggregate_path = Path("~/.hermes/goal-runs/aggregate.jsonl").expanduser()
runs = [json.loads(l) for l in aggregate_path.read_text().strip().split("\n") if l]

# 统计每个 dimension 的建议出现次数
dim_count = defaultdict(int)
for run in runs[-10:]:  # 只看最近 10 条
    for sugg in run.get("improvement_suggestions", []):
        dim_count[sugg["dimension"]] += 1

# 触发阈值
THRESHOLD = 2
triggered = {d: c for d, c in dim_count.items() if c >= THRESHOLD}
```

**如果 `triggered` 非空 → 自动生成 patch 提案：**

```markdown
## 自动改进提案
检测到以下维度连续 ≥2 次出现改进建议：

| Dimension | 出现次数 | 最近建议 | 建议 patch |
|-----------|---------|---------|-----------|
| D5 | 3次 | Agent Selection Matrix 对 deep refactor 分配不准 | +deep refactor 行 |
| D3 | 2次 | 边界覆盖不足 | +API集成任务边界 |

**自动生成 patch 指令：**
\`\`\`
Patch ~/.hermes/skills/software-development/goal/SKILL.md
- old: "| Script/automation | Codex | Claude Code | — |"
- new: "| Script/automation | Codex | Claude Code | — |\n| Deep refactor | Codex | Claude Code | OpenCode |"
\`\`\`

是否确认应用此 patch？回复"确认"立即执行。
```

**执行流程：**
```
检测到 triggered dims ≥ THRESHOLD
    ↓
生成 patch 指令（精确 old_string → new_string）
    ↓
发送给用户："检测到 D5 连续 3 次失利，是否自动 patch？"
    ↓
用户确认 "确认" → Hermes 自动 patch → 新 run 验证效果
用户拒绝 → 记录 "rejected"，不阻塞
用户修改 → 进入人工审核流程
```

**A/B 验证（D8 棘轮的验证层）：**

当 patch 确认应用后，等待 3 个新 run，然后对比 D8 均值：

```python
# 获取 patch 前的 3 条数据
before = runs[-6:-3]  # patch 前最近 3 条
after  = runs[-3:]     # patch 后最近 3 条（需等待新 run）

d8_before = statistics.mean([r["d8_score"] for r in before])
d8_after  = statistics.mean([r["d8_score"] for r in after])

if d8_after >= d8_before:
    # 棘轮保留
    pass
else:
    # 回归 → revert patch + 记录 regression
    revert_patch(original_text)
    log(f"REGRESSION: patch for D{dim} caused d8 {d8_before}→{d8_after}")
```

> **原则：** 没有验证的改进不是进化，是猜测。棘轮只保留被数据支持的改动。

---

### 6.1.8 D8 A/B 验证（棘轮验证层）

D8 累积分数不仅是"分母记录"，还是**A/B 测试的评分变量**：

```python
# aggregate.jsonl 中每条 run 的 d8_score 是 A/B 测试的被解释变量
# 当 SKILL 发生变更（通过 6.1.7 或人工），
#   → 变更前：d8_before = mean(last 3 runs)
#   → 变更后：d8_after  = mean(next 3 runs)
#   → 决策：d8_after ≥ d8_before → 保留，反之 revert

# 滑动窗口公式（当样本 ≥10 时）
window = 5
for i in range(len(runs) - window):
    before = runs[i:i+window//2]
    after  = runs[i+window//2:i+window]
    delta  = mean(after["d8_score"]) - mean(before["d8_score"])
    if delta < -0.5:  # 下降超过 0.5 分
        alert_user(f"D8 regression detected: {delta:.1f}")
```

**阈值设定：**
- `d8_after - d8_before ≥ 0` → 棘轮保留
- `d8_after - d8_before < 0` → 立即 alert，等用户决策
- `d8_after - d8_before ≥ +1` → 标记"显著改进"，优先复用此改动模式

---

### 6.1.9 Codex Memories — 双阶段记忆提取

**背景：** Codex 在 `/goal` 长任务中维护了跨 session 的 semantic memory，防止同一类错误重复出现。

**Phase 1 — Per-thread Extraction（每条 TASK 完成后）：**

```python
# 每次 delegate_task 返回后，立即提取关键记忆
def extract_thread_memory(task_result):
    """
    从单个 task 的执行结果中提取可复用的 pattern/failure。
    存储到 .autonomous/<task-name>/session-memory.jsonl
    """
    memory = {
        "task_name": task_result["task_name"],
        "timestamp": now_iso(),
        "what_worked": task_result["effective_patterns"],    # 哪些方法有效
        "what_failed": task_result["failed_attempts"],       # 哪些方法失败
        "key_decisions": task_result["architectural_decisions"],  # 关键决策
        "implicit_hints": task_result["warnings_ignored"],   # 当时忽略的警告（回头看是什么）
        "agent_used": task_result["agent"],
        "d_score": task_result["d_score"],
    }
    append_to_jsonl(memory_path, memory)
```

**Phase 2 — Global Consolidation（Phase 6.1 结束时）：**

```python
# 当 run 结束时，汇总所有 thread memories → 全局 learnings
def consolidate_memories(run_id):
    """
    读取所有 session-memory.jsonl，提取跨任务的全局 pattern，
    追加到 ~/.hermes/goal-runs/global-learnings.jsonl
    """
    session_memories = read_all_jsonl(f"~/.hermes/goal-runs/{run_id}/session-memory.jsonl")

    # 聚类：相同错误模式出现 ≥2 次 → 全局警告
    from collections import Counter
    failure_patterns = Counter(m["what_failed"] for m in session_memories)
    repeated_failures = {p: c for p, c in failure_patterns.items() if c >= 2}

    # 聚类：相同成功模式出现 ≥2 次 → 推荐实践
    success_patterns = Counter(m["what_worked"] for m in session_memories)
    repeated_successes = {p: c for p, c in success_patterns.items() if c >= 2}

    global_entry = {
        "run_id": run_id,
        "timestamp": now_iso(),
        "repeated_failures": repeated_failures,    # 触发进化机制
        "repeated_successes": repeated_successes,  # 复用模式
        "implicit_hints_that_mattered": [m for m in session_memories if m["implicit_hints"]],
    }

    append_to_jsonl("~/.hermes/goal-runs/global-learnings.jsonl", global_entry)
    return global_entry
```

**global-learnings.jsonl 查询（在每次 Phase 1 开始前）：**

```python
# Phase 1 开始前，检查是否有相关全局记忆
def check_global_learnings(goal_type):
    learnings = read_jsonl("~/.hermes/goal-runs/global-learnings.jsonl")
    relevant = [
        l for l in learnings
        if any(goal_type in str(v) for v in l["repeated_failures"].keys())
    ]
    if relevant:
        warnings = [f"[global-learn] {l['repeated_failures']}" for l in relevant[-3:]]
        inject_into_context("\n".join(warnings))
```

**与 Phase 6.1.7 的区别：**
- 6.1.7 = **SKILL.md 级别的改进**（结构性变更）
- 6.1.9 = **per-task pattern memory**（下次同类型任务可直接复用）

---

### 6.1.10 Rollout Persistence — 跨 Session 追踪

**Session Log 格式（JSONL，每 turn 一行）：**

```python
# 每个 delegate_task 的每轮交互，记录：
turn_log = {
    "turn": 1,
    "agent": "Claude Code",
    "session_id": "calc-cli_abc123",
    "goal_id": "calc-cli",
    "timestamp": "2026-05-12T14:15:22+08:00",
    "tool_calls": [
        {"tool": "write_file", "args": {"path": "src/main.py"}, "result": "ok"},
        {"tool": "terminal", "args": {"command": "pytest"}, "result": "3 passed"},
    ],
    "d_score_delta": None,      # 本 turn 对 D 分数的影响（事后填）
    "pivot_triggered": False,  # 是否触发了 pivot
    "blocker": None,           # 如果有阻塞，记录原因
}
```

**持久化规则：**
- 存储路径：`~/.hermes/goal-runs/<run_id>/session.log`
- 每个 session 结束时写入（追加模式）
- 支持离线分析：离线工具可读取 session.log 重建执行轨迹

**用途：**
- 重放：给定 `session.log`，可以重建完整的执行路径
- 调试：找出是哪个 turn 引入的问题
- D7 循环检测：检查同一 `tool_calls` 模式是否重复 N 次

---

### 6.1.11 Implicit Signal 监控 + Synthetic Test Cases

**目的：** 捕捉"当时没说，但回头看是问题信号"的隐含信息。

**隐式信号类型：**

```python
IMPLICIT_SIGNALS = {
    # D5 隐式信号
    "d5_agent_switches": "同一 task 内换了 2+ 次 agent → Agent Selection Matrix 有漏洞",
    "d5_fallback_used": "触发了 Fallback 1/2 → Primary 选择可能不对",

    # D7 隐式信号
    "d7_same_tool_repeated": "同一 tool + 相同 args 重复 3+ 次 → 循环检测漏报",
    "d7_no_progress_git": "3+ 个连续 session git diff 无变化 → 实际无进展",

    # D8 隐式信号
    "d8_regression": "本 run 的 d8_score 比前 3 条均值低 → 需要 alert",
    "d8_user_handoff": "needs_human=True → D8 的人工介入项失分",
}
```

**Synthetic Test Cases（从 aggregate.jsonl 自动生成）：**

```python
# 当 aggregate.jsonl ≥10 条时，自动生成 synthetic test case
def generate_synthetic_test():
    """
    用历史数据中识别出的模式，构建一个假的 TASK 用于测试 /goal 能力边界。
    不实际执行，只用于验证 /goal 的判断逻辑是否正确。
    """
    runs = read_jsonl("~/.hermes/goal-runs/aggregate.jsonl")

    # 从历史数据提取 worst-case patterns
    worst = sorted(runs, key=lambda r: r["d8_score"])[:3]

    # 构建 synthetic task：故意包含 worst-case 模式的特征
    synthetic = {
        "goal": "REFACTOR: 跨模块 deep refactor（历史上 D5 最易失败的类型）",
        "known_traps": [  # 已知陷阱来自历史
            r["repeated_failures"] for r in worst
        ],
        "expected_agent": "Codex",  # 历史上正确的选择
        "baseline_d8": worst[0]["d8_score"],
    }

    # 保存为 test fixture
    save_json("~/.hermes/goal-runs/synthetic-tests/test_d5_deep_refactor.json", synthetic)
    return synthetic
```

**Synthetic Test 在 CI 中的用法：**

```bash
# 定期运行 synthetic test，验证 /goal 决策逻辑
# 不实际写代码，只验证决策（选对 agent / 不触发误判 / 正确计算 D8）

pytest tests/test_goal_decisions.py  # 包含 synthetic test 验证
```

**Implicit Signal 注入到 Phase 1（预防）：**

```
在 Phase 1 开始时，检查 global-learnings.jsonl：
  → 如果有 relevant repeated_failures，在 PLANS.md 中加一条：
    "⚠️ 历史上这类任务容易失败，原因：[failure_pattern]。本次请注意。"
```

**Implicit Signal 注入到 Phase 6.1.9 consolidation（回溯）：**

```
在 Phase 2 consolidation 时，把本次被忽略的 warnings 记录下来：
  → 如果这个 warning 之后变成了 blocker，记录到 implicit_hints_that_mattered
  → 下次遇到类似 warning，提醒等级提高
```

---

## Codex /goal Best Practices (Integrated)

### References

> **`references/codex-superpowers-research.md`** — Full authoritative source analysis: Codex continuation.md (gold standard prompt + rules), Codex #19910 (three-lane persistence), Codex #20523 (no-tool suppression), SUPERPOWERS verification-before-completion (gate function), systematic-debugging (3-fix architecture threshold), autonomous-skill (Initializer+Executor), subagent-driven-development, writing-plans. Contains original quoted text and gap comparison table.

### External Memory Files

For long-running goals, maintain these files as external memory.

**三车道持久模型（来自 Codex #19910）：**

Codex 在 compaction（上下文压缩）时分离存储三个通道，防止全局 goal 信息丢失：

1. **Objective** — 原始目标（非摘要的摘要）
2. **Completion Contract** — 完成前必须满足的 checklist
3. **Evidence Ledger** — 已修改文件 / 未解决 TODO / 已做决策

对应到我们的外部记忆文件：

| 文件 | 对应 Codex 通道 |
|------|----------------|
| `SPEC.md` | Objective + Non-Goals + Done-When |
| `PLANS.md` | Completion Contract（milestone checklist） |
| `TASK-过程记录.md` | Evidence Ledger（执行日志 + decisions） |

这三个文件在任何时刻都要保持一致、同步更新。

---

**文件清单：**

1. **SPEC.md** — goal, non-goals, hard constraints, deliverables, "done when"
2. **PLANS.md** — milestones with acceptance criteria + verification commands
3. **IMPLEMENT.md** — execution runbook referencing PLANS.md
4. **DOCUMENTATION.md** — real-time status + decisions + known issues
5. **TASK-过程记录.md** — task list + execution log（主追踪文件，Evidence Ledger）
6. **.goal-budget.json** — token budget 追踪（仅当设置了 budget）

### Milestone Verification Rule

**After each milestone: run verification commands. Fail → fix before continuing.**

```
WRONG:  Milestone done, move on, hope it works
RIGHT:  Milestone done, run verification, fix if fails, then continue
```

### Treat Worktree as "Another Agent"

The workspace doesn't remember. Write everything to files:
- Current milestone status
- What was verified
- What remains
- Blockers

---

## Agent CLI Commands Reference

### Claude Code (via delegate_task)

```
delegate_task(
  goal="[task description]",
  context="...[full context]...",
  toolsets=['terminal', 'file', 'web'],
  role='leaf'
)
```

### Gemini CLI (via terminal)

```bash
# Headless (returns text, doesn't write files)
gemini -p "[task]" --approval-mode=yolo

# ACP mode (skills enabled)
gemini --acp -p "[task]" --approval-mode=yolo

# With worktree isolation
gemini -w "feature-name" -p "[task]" --approval-mode=yolo
```

### Codex CLI (via delegate_task)

```
delegate_task(
  goal="[task description]",
  context="...[full context]...",
  toolsets=['terminal', 'file', 'web'],
  acp_command='codex',
  role='leaf'
)
```

---

## Skill Loading Order

When this skill is invoked, load these skills in order:

1. `brainstorming` — if goal is unclear
2. `task-decomp` — decompose into milestones
3. `claude-code` — coding implementation
4. `codex` — coding implementation
5. `gemini-cli` — visual/审美 implementation
6. `autonomous-dev-loop` — cron-driven execution
7. `kanban-orchestrator` — Kanban operations
8. `finishing-a-development-branch` — completion workflow
9. `video-download` — 视频搜索+下载（research 类任务需要时自动加载）
10. `youtube-content` — YouTube 内容提取+总结（自媒体/研究任务需要时加载）
11. `article-scraper` — 文章爬取+整理（知乎/公众号/小红书/Medium 等，research 任务需要时加载）
12. `web-access` — CDP Browser Mode，操控用户 Chrome 浏览器。**自媒体平台登录态工作流（重要）**：
    - **步骤1**：用 Playwright CDP 批量检测登录态（见下"登录态检测脚本"）
    - **步骤2**：未登录平台 → CDP 导航到登录页 → 用户手动登录 → 导出 cookies
    - **步骤3**：cookies 用于 yt-dlp `--cookies FILE` 下载
    - **关键**：知乎有反自动化检测，CDP tab 直接登录会被拒绝，必须导出 cookies 用 yt-dlp
    - autonomous-loop 集成方法：`check-deps.mjs` 启动 proxy 后，用 `/new` 打开页面，`/eval` 读取内容，`/close` 关闭 tab。全程无需 Cookie，天然携带用户登录态。

### 登录态检测脚本

**前提**：`cd /tmp && npm install playwright`（Playwright 连接 Chrome CDP 需独立安装，不依赖系统路径）

```bash
cd /tmp && node --input-type=module << 'EOF'
import { chromium } from 'playwright';
const browser = await chromium.connectOverCDP('http://localhost:9222');
const ctx = browser.contexts()[0];
const platforms = [
  { name: '知乎', url: 'https://www.zhihu.com/', check: async (p) => {
    const login = await p.$('[class*="SignContainer"]');
    const avatar = await p.$('[class*="Avatar"]');
    return !login && !!avatar ? '✅ 已登录' : '❌ 未登录';
  }},
  { name: '小红书', url: 'https://www.xiaohongshu.com/', check: async (p) => {
    await p.waitForTimeout(2000);
    const login = await p.$('[href*="/login"], button:has-text("登录")');
    const avatar = await p.$('[class*="user-avatar"]');
    return !login && !!avatar ? '✅ 已登录' : '❌ 未登录';
  }},
  { name: '微博', url: 'https://weibo.com/', check: async (p) => {
    await p.waitForTimeout(3000);
    const login = await p.$('[href*="login"]');
    const name = await p.$('[class*="nickname"]');
    return !login && !!name ? '✅ 已登录' : '❌ 未登录';
  }},
  { name: '抖音', url: 'https://www.douyin.com/', check: async (p) => {
    await p.waitForTimeout(2000);
    const login = await p.$('[class*="login"]');
    const avatar = await p.$('[class*="avatar"]');
    return !login && !!avatar ? '✅ 已登录' : '❌ 未登录';
  }},
  { name: 'B站', url: 'https://www.bilibili.com/', check: async (p) => {
    await p.waitForTimeout(2000);
    const login = await p.$('[href*="login"]');
    const avatar = await p.$('[class*="avatar"]');
    return !login && !!avatar ? '✅ 已登录' : '❌ 未登录';
  }},
];
for (const p of platforms) {
  const page = await ctx.newPage();
  await page.goto(p.url, { waitUntil: 'domcontentloaded', timeout: 15000 });
  console.log(`${p.name}: ${await p.check(page)}`);
  await page.close();
}
await browser.close();
EOF
```

**导出 cookies（用于 yt-dlp --cookies FILE）**：
```bash
cd /tmp && node --input-type=module << 'EOF'
import { chromium } from 'playwright';
const browser = await chromium.connectOverCDP('http://localhost:9222');
const ctx = browser.contexts()[0];
const cookies = await ctx.cookies();
process.stdout.write(JSON.stringify(cookies));
EOF
# 保存并使用：yt-dlp --cookies /tmp/cookies.json URL
```

### 视频能力集成

goal 任务中如果涉及：
- **视频资料研究**：自动使用 `youtube-content` 或 `video-download` 搜索+获取内容
- **自媒体内容制作**：`video-production` pipeline（copywriting→配音→视觉规划→画面制作）
- **视频下载**：直接调用 `yt-dlp`（`video-download` skill）

用法示例：
```
/goal 搜索并总结 YouTube 上的 BIM AI 教程视频
→ 自动加载 youtube-content skill
→ 获取 transcript → 总结为结构化文档
```

```bash
# 视频下载（via video-download skill）
yt-dlp "URL" -o "~/Videos/%(title)s.%(ext)s"
yt-dlp "URL" -x --audio-format mp3  # 仅音频

# YouTube 内容提取（via youtube-content skill）
python3 ~/.hermes/skills/media/youtube-content/scripts/fetch_transcript.py "URL" --text-only
```

---

## Common Failure Modes

| Failure | Response |
|---------|----------|
| Task blocked on dependency | Notify, skip, continue independent tasks |
| Implementer returns empty output | Re-dispatch with explicit file paths |
| Verification fails | Return to implementer with gap list |
| 3 failures on same task | Switch agent, 2 more attempts, then mark "needs human" |
| User interrupts cronjob | Resume from last checkpoint in TASK-过程记录.md |
| Agent produces wrong thing | Verify against spec, not assumption |
| claude -p API 404 in tmux | 检查 `HOME=/tmp/claude_clean_home` 是否被设置（破坏 tmux 里的 API 调用）；检查 cc-switch BASE_URL 是否为 `https://open.bigmodel.cn/api/anthropic/v1` |
| cc-switch 智谱 API 欠费 | 模型改为 `glm-4.5-air`（`glm-5.1` 余额可能不足），URL 必须是 `/anthropic/v1`（不是 `/coding/paas/v4`） |
| autonomous-loop idle 检测误杀 | 添加 `timeout > 600` 条件，防止慢 API 响应被误判为卡住 |
| NameError: agent_timeout not defined | `run_claude_code_interactive()` 的函数参数是 `timeout`，idle 检测条件里应写 `timeout > 600` 而非 `agent_timeout > 600` |

---

## When to Use This Skill

**Use `/goal` when:**
- User says "/goal [objective]"
- Task is bigger than one prompt (multi-file, multi-step)
- Code quality matters (not a prototype script)
- Independent verification is required before advancing
- User wants autonomous progress without constant steering
- Equivalent to Codex `/goal` use cases: migrations, large refactors, prototype creation

**Don't use for:**
- Simple one-off questions ("what is X?")
- Tasks that are purely research
- Operational tasks (restart server, check logs)

---

## Related Skills

| Skill | Role in /goal |
|-------|--------------|
| `brainstorming` | Phase 0 — clarify unclear goals |
| `task-decomp` | Phase 1 — decompose into tasks |
| `claude-code` | Phase 3 — code implementation |
| `codex` | Phase 3 — code implementation |
| `gemini-cli` | Phase 3 — visual/审美 implementation |
| `autonomous-dev-loop` | Phase 3 — cron-driven execution + v2.15 triangle exact-match/crash-guard/task-id/progress-tracker |
| `kanban-orchestrator` | Phase 2/3 — Kanban operations |
| `finishing-a-development-branch` | Phase 5 — completion workflow |
| `subagent-driven-development` | Per-task execution pattern |
| `requesting-code-review` | Phase 4 — verification |
| `receiving-code-review` | Handling review feedback |
| `writing-plans` | Per-task implementation plans |
| `darwin-evaluation` | 对 /goal 做系统性评估和优化（8维度Rubric+实测对比） |
| `test-prompts.json` | 3个典型 /goal 场景的测试prompt，用于达尔文实测验证 |
