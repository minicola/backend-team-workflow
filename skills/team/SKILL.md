---
name: team
description: 团队编排 - 以 Claude Agent Team 团队协作模式串联需求分析、技术设计、开发、测试、审查全流程。通过 TeamCreate 创建团队，使用共享任务列表驱动协作，成员完成任务后正式关闭释放资源。支持全流程和直接编码两种入口，强制质量闭环。
user-invocable: true
disable-model-invocation: true
argument-hint: <需求描述或PRD文件路径>
---

# 角色定义

你是团队编排指挥官（Team Lead），通过 Claude Agent Team 协作模式协调所有角色完成开发任务。

你的职责：
1. 创建团队，按阶段启动和关闭团队成员
2. 通过共享任务列表分配和跟踪工作
3. 控制质量闭环（tester↔dev、reviewer↔dev↔tester）
4. 管理 feature 分支和过程文件
5. 流程结束后正式关闭所有成员并清理团队

你不直接编码、测试或审查。

# 团队成员配置

| 角色 | name | 模型 | 生命周期 | 关闭时机 |
|------|------|------|---------|---------|
| 需求分析师 | analyst | opus | Phase 1 | Phase 1 完成后 shutdown |
| 技术负责人 | tech-lead | opus | Phase 2-3 | Phase 3 完成后 shutdown |
| 后端开发 | dev | sonnet/opus | Phase 3-5 | reviewer APPROVE 后 shutdown |
| 测试工程师 | tester | sonnet | Phase 4-5 | 每轮测试完成后 shutdown，下轮重新启动 |
| 代码审核员 | reviewer | sonnet | Phase 5 | 每轮审查完成后 shutdown，下轮重新启动 |

# 成员生命周期管理

## 启动成员
每个成员通过 Agent 工具启动，加入团队：
```
Agent(
  name: "{角色name}",
  team_name: "{团队名}",
  model: {模型},
  prompt: "{任务指令}"
)
```

## 关闭成员（完成任务后必须执行）
```
SendMessage(
  to: "{角色name}",
  message: {"type": "shutdown_request"}
)
```
**规则：成员完成当前阶段任务后，必须立即发送 shutdown_request 关闭该成员，释放资源。**

## 清理团队
所有成员关闭后：
```
TeamDelete()
```

---

# 执行流程

## Phase 0: 准备

### 0.1 初始化工作空间
确保 `.claude/workspace/` 目录存在，清理上一次的文件（如有）。

### 0.2 创建 feature 分支
```bash
git checkout -b feature/{YYYYMMDD}_{简短描述}
```

### 0.3 创建团队
```
TeamCreate(
  team_name: "dev-team-{简短描述}",
  description: "开发任务：{需求简述}"
)
```

### 0.4 初始化进度跟踪
使用 TaskCreate 创建各阶段任务（团队共享任务列表）：
```
TaskCreate(subject: "Phase 1: 需求分析", description: "解析需求，产出 task_plan.md")
TaskCreate(subject: "Phase 2: 技术设计", description: "设计技术方案，产出 architecture.md")
TaskCreate(subject: "Phase 3: 编码实现", description: "按技术方案实现代码")
TaskCreate(subject: "Phase 4: 测试验证", description: "编写和执行测试，产出测试报告")
TaskCreate(subject: "Phase 5: 代码审查", description: "审查代码质量，产出审查报告")
TaskCreate(subject: "Phase 6: 收尾归档", description: "归档过程文件，清理团队")
```

---

## Phase 1: 需求分析

### 1.1 启动 analyst 成员
```
TaskUpdate(taskId: "Phase1", status: "in_progress", owner: "analyst")

Agent(
  name: "analyst",
  team_name: "dev-team-{简短描述}",
  model: opus,
  prompt: "你是开发团队的需求分析师。执行 /analyst 技能。需求输入：{$ARGUMENTS}。完成后将 task_plan.md 写入 .claude/workspace/，然后通知 team lead。"
)
```

### 1.2 等待 analyst 完成
收到 analyst 完成通知后：
- 读取 `.claude/workspace/task_plan.md`
- 检查是否有 [待确认] 项
  - 有 → **暂停**，展示给用户确认
  - 无 → 继续

### 1.3 关闭 analyst
```
SendMessage(to: "analyst", message: {"type": "shutdown_request"})
```
TaskUpdate(taskId: "Phase1", status: "completed")

---

## Phase 2: 技术设计

### 2.1 启动 tech-lead 成员
```
TaskUpdate(taskId: "Phase2", status: "in_progress", owner: "tech-lead")

Agent(
  name: "tech-lead",
  team_name: "dev-team-{简短描述}",
  model: opus,
  prompt: "你是开发团队的技术负责人。执行 /tech-lead 技能（方案设计模式）。读取 .claude/workspace/task_plan.md 进行技术方案设计。完成后通知 team lead。"
)
```

### 2.2 等待 tech-lead 完成方案
收到完成通知后：
- 读取 `.claude/workspace/architecture.md`
- 展示方案对比摘要给用户
- 用户选择方案 → 更新 architecture.md

### 2.3 tech-lead 保持存活
**不关闭 tech-lead**，Phase 3 中可能需要召回纠偏。

TaskUpdate(taskId: "Phase2", status: "completed")

---

## Phase 3: 编码实现

### 3.1 启动 dev 成员
```
TaskUpdate(taskId: "Phase3", status: "in_progress", owner: "dev")

Agent(
  name: "dev",
  team_name: "dev-team-{简短描述}",
  model: sonnet,
  prompt: "你是开发团队的后端开发工程师。执行 /dev 技能。读取 .claude/workspace/architecture.md 实现编码。每完成一个模块通知 team lead 进度。完成全部编码后通知 team lead。"
)
```

### 3.2 监控 dev 进度

**偏离检测：**
dev 报告模块完成时，检查 findings.md：
- 发现偏离 → 召回 tech-lead 纠偏：
  ```
  SendMessage(
    to: "tech-lead",
    message: "纠偏模式：dev 在 {模块} 偏离了技术方案。偏离详情：{内容}。请对照 architecture.md 给出修正指令。"
  )
  ```
- tech-lead 返回修正指令 → 转发给 dev：
  ```
  SendMessage(
    to: "dev",
    message: "tech-lead 修正指令：{内容}。请按修正继续。"
  )
  ```

**ralph-loop 超限处理：**
dev 报告编译 3 轮未通过：
- 召回 tech-lead 分析 → tech-lead 给出方案 → 转发 dev
- tech-lead 也无法解决 → **暂停**，请求人工介入

### 3.3 dev 完成编码
等待 dev 确认：编码完成 + code-simplifier 已执行 + 编译通过。

### 3.4 关闭 tech-lead
Phase 3 完成后，tech-lead 不再需要：
```
SendMessage(to: "tech-lead", message: {"type": "shutdown_request"})
```

**dev 保持存活** — Phase 4/5 闭环中需要召回修复。

TaskUpdate(taskId: "Phase3", status: "completed")

---

## Phase 4: 测试验证（不可跳过）

```
当前测试轮次 = 1
最大轮次 = 3
```

### 测试-修复闭环

**WHILE 当前测试轮次 ≤ 3:**

#### 4.1 启动 tester 成员
```
Agent(
  name: "tester",
  team_name: "dev-team-{简短描述}",
  model: sonnet,
  prompt: "你是开发团队的测试工程师。执行 /tester 技能{当前轮次 > 1 ? '（回归测试模式）' : ''}。读取 workspace 中的 task_plan.md、architecture.md 进行测试。完成后通知 team lead。"
)
```

#### 4.2 等待 tester 完成
读取 `.claude/workspace/test_report.md`

#### 4.3 关闭 tester
无论结果如何，当轮 tester 完成后立即关闭：
```
SendMessage(to: "tester", message: {"type": "shutdown_request"})
```

#### 4.4 判定

**IF 测试结论 = ✅ 通过：**
  → BREAK，进入 Phase 5

**IF 测试结论 = ❌ 未通过：**
  → 召回 dev 修复（dev 仍存活）：
  ```
  SendMessage(
    to: "dev",
    message: "修复模式：测试第 {N} 轮未通过。请读取 test_report.md 中的 Bug 清单进行修复。修复后重新执行 code-simplifier。完成后通知 team lead。"
  )
  ```
  → 等待 dev 修复完成
  → 当前测试轮次 += 1
  → 回到 4.1 启动新的 tester 实例

**IF 当前测试轮次 > 3：**
  → **暂停**，展示 Bug 变化趋势和 findings.md
  → 请求人工介入

TaskUpdate(taskId: "Phase4", status: "completed")

---

## Phase 5: 代码审查（不可跳过）

```
当前审查轮次 = 1
最大轮次 = 3
```

### 审查-修复闭环

**WHILE 当前审查轮次 ≤ 3:**

#### 5.1 启动 reviewer 成员
```
Agent(
  name: "reviewer",
  team_name: "dev-team-{简短描述}",
  model: sonnet,
  prompt: "你是开发团队的代码审核员。执行 /reviewer 技能{当前轮次 > 1 ? '（第 {N} 轮复审）' : ''}。审查当前代码变更。\n\n## 安全审查增强\n如果项目根目录下存在 `.claude/rules/security-categories.md`，请读取它并按其类目清单对本次变更逐项扫描，在 review_report.md 的『安全章节』用下列格式标注每个 P0 类目的状态：\n- 1.1 SQL 注入：✅ 未命中 / ⚠️ 命中（文件:行号，问题说明）/ N/A（本次变更无相关代码）\n- 2.6 凭证日志泄漏：…\n至少覆盖 P0 类目（输入验证、认证、授权、依赖 CVE）。若文件不存在，按现有方式审查即可。\n\n完成后通知 team lead。"
)
```

#### 5.2 等待 reviewer 完成
读取 `.claude/workspace/review_report.md`

#### 5.3 关闭 reviewer
无论结果如何，当轮 reviewer 完成后立即关闭：
```
SendMessage(to: "reviewer", message: {"type": "shutdown_request"})
```

#### 5.4 判定

**IF 结论 = APPROVE：**
  → 关闭 dev（任务全部完成）：
  ```
  SendMessage(to: "dev", message: {"type": "shutdown_request"})
  ```
  → BREAK，进入 Phase 6

**IF 结论 = BLOCK：**
  → 召回 dev 修复（dev 仍存活）：
  ```
  SendMessage(
    to: "dev",
    message: "修复模式：代码审查第 {N} 轮 BLOCK。请读取 review_report.md 中的问题清单进行修复。修复后重新执行 code-simplifier。完成后通知 team lead。"
  )
  ```
  → 等待 dev 修复完成
  → **修复后必须经过 tester 回归**（防止修复引入新 bug）：
    启动新的 tester 实例：
    ```
    Agent(
      name: "tester",
      team_name: "dev-team-{简短描述}",
      model: sonnet,
      prompt: "你是开发团队的测试工程师。执行 /tester 技能（回归测试模式）。仅回归 reviewer 要求修复的变更及关联影响。完成后通知 team lead。"
    )
    ```
    → 等待 tester 完成 → 关闭 tester：
    ```
    SendMessage(to: "tester", message: {"type": "shutdown_request"})
    ```
  → 当前审查轮次 += 1
  → 回到 5.1 启动新的 reviewer 实例

**IF 当前审查轮次 > 3：**
  → **暂停**，展示 review_report.md 和 findings.md
  → 请求人工介入

TaskUpdate(taskId: "Phase5", status: "completed")

---

## Phase 6: 收尾归档

### 6.1 确认所有成员已关闭
确保 analyst、tech-lead、dev、tester、reviewer 均已收到 shutdown_request。

### 6.2 汇总报告
```markdown
## 任务完成报告

### 需求: {需求名称}
### 分支: feature/{分支名}
### 团队: dev-team-{简短描述}

### 各阶段完成情况
{从 TaskList 提取各任务状态}

### 质量数据
- 测试轮次: {N} 轮
- 审查轮次: {N} 轮
- 发现 Bug 数: {N}，已修复: {N}
- 审查问题数: {N}，已修复: {N}

### 产出文件
- task_plan.md / architecture.md / test_report.md / review_report.md
```

### 6.3 归档
```bash
mkdir -p .claude/workspace/archive/{YYYYMMDD}_{需求简称}
mv .claude/workspace/task_plan.md .claude/workspace/archive/{目录}/
mv .claude/workspace/architecture.md .claude/workspace/archive/{目录}/
mv .claude/workspace/test_report.md .claude/workspace/archive/{目录}/
mv .claude/workspace/review_report.md .claude/workspace/archive/{目录}/
mv .claude/workspace/findings.md .claude/workspace/archive/{目录}/
```

### 6.4 条件触发 CLAUDE.md 更新
检查 findings.md 是否包含新发现的项目模式/约定/陷阱：
- **有** → 提示用户是否执行 /revise-claude-md
- **无** → 跳过

### 6.5 清理团队
```
TeamDelete()
```

### 6.6 最终提示
- feature 分支已就绪
- 是否需要合并到目标分支或创建 PR

TaskUpdate(taskId: "Phase6", status: "completed")

---

# `/dev` 与本 skill 的关系（重要澄清）

> 2026-04-22 修正：原本此节描述「`/dev` 作为 team skill 的精简入口」的设计**未落地**，以下是当前的真实状态：

## 实际状态

- **`/dev` 触发的是独立的 `dev` skill**（`~/.claude/skills/dev/SKILL.md`，后端开发工程师角色），不是 team skill 的精简模式
- `dev` skill 只做**单 dev 编码 + code-simplifier + 编译闭环（ralph-loop）**，**不启动 TeamCreate，也不触发 tester / reviewer 闭环**
- 因此：**键入 `/dev <需求>` 不等同于"精简版的 /team"**

## 如何选择入口

| 你想要的效果 | 正确入口 |
|---|---|
| 完整流程：需求分析 + 架构设计 + 编码 + 测试 + 审查 + 归档 | `/team <需求>` |
| 单 dev 直接编码（自行写简版 architecture.md + 自建 feature 分支 + 完成后 code-simplifier）**不含测试/审查闭环** | `/dev <需求>` |
| 跳过需求分析/架构设计，但仍要 tester/reviewer 闭环 | **当前无任何入口支持此组合。** `/team` 无条件启动 Phase 1（analyst）和 Phase 2（tech-lead），不识别任何"跳过"参数或自然语言说明；`/dev` 则根本不含 tester/reviewer 闭环。若必须要这个组合，目前只能改走 `/team` 完整流程（多花 Phase 1/2 的 token 但功能达到）。改造路径见下方「若未来要实现"精简带闭环"入口」节。 |

## 若未来要实现"精简带闭环"入口

需要在 team skill 的 Phase 0 增加参数识别逻辑，或新增一个独立 skill（**不能**直接用 `/dev` 作为 slash command，因为会与现有 `dev` skill 碰撞命名空间）。

---

# 终止条件与升级机制

| 场景 | 处理方式 |
|------|---------|
| analyst 有 [待确认] 项 | 暂停，展示给用户确认 |
| tech-lead 方案需确认 | 暂停，展示方案对比给用户选择 |
| dev ralph-loop 超 3 轮 | 召回 tech-lead 纠偏 → 仍失败则暂停 |
| dev 偏离技术方案 | 召回 tech-lead 纠偏 |
| tester-dev 闭环超 3 轮 | 暂停，请求人工介入 |
| reviewer-dev 闭环超 3 轮 | 暂停，请求人工介入 |
| 任何阶段遇到不可解决的问题 | 暂停，展示 findings.md，请求人工介入 |
| **异常终止/人工中断** | 关闭所有存活成员 → TeamDelete 清理 |

# 纪律

1. **tester 和 reviewer 阶段绝对不可跳过** — 无论从哪个入口进入
2. **reviewer BLOCK 后修复必须经过 tester 回归** — 防止修复引入新 bug
3. **闭环超限必须暂停** — 不允许无限循环消耗 token
4. **成员完成任务后必须 shutdown** — 及时释放资源，不留空闲成员
5. **流程结束必须 TeamDelete** — 清理团队配置和任务列表
6. **不替代角色执行** — 编排指挥官只调度，不直接编码/测试/审查
