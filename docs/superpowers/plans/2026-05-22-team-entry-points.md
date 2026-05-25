# /team 任意节点切入 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让 `/team` 通过 `--from=<phase>` 参数支持从 analyst / tech-lead / dev / tester / reviewer 任一节点切入，跳过的上游角色按需启动并进入清单驱动模式，且分支命名统一为 `fyx/feature/...`。

**Architecture:** 仅改写 markdown 指令文件（无源代码）。主改造在 `skills/team/SKILL.md`：Phase 0 重写为参数解析+前置校验门控；Phase 1-5 加起首守卫；Phase 4/5 召回逻辑加 dev "首次启动 vs SendMessage" 分支；新增清单驱动模式 prompt 模板。`skills/dev/SKILL.md` 与项目 `CLAUDE.md` 做小幅同步。

**Tech Stack:** 纯 markdown 编辑（Edit 工具），grep / sed 文本验证，git commit。无构建系统、无单元测试 — "验证"由最后一个 Task 在真实项目装插件冒烟完成。

**Spec:** `docs/superpowers/specs/2026-05-22-team-entry-points-design.md`（commits `060339f` + `36bdabe`）

**分支：** 在 `fyx/feature/20260522_team_entry_points` 上执行。

---

## File Structure

| 文件 | 操作 | 责任 |
|---|---|---|
| `skills/team/SKILL.md` | Modify | 主改造：frontmatter / Phase 0 / Phase 1-5 守卫 / 生命周期表 / 清单驱动 prompt / Phase 4 5 召回逻辑 / 纪律节 / /dev 关系节 / 终止条件表 |
| `skills/dev/SKILL.md` | Modify | 清单驱动模式识别（跳过 Step 2 复杂度评估）；Step 1 分支命名同步 |
| `CLAUDE.md` | Modify | 架构节、已知陷阱节三处同步 |
| `docs/superpowers/plans/2026-05-22-team-entry-points.md` | Created | 本计划文件 |

无新建可执行文件。

---

## Task 0: 准备工作分支

**Files:**
- 仅 git 操作

- [ ] **Step 0.1: 确认当前在 main，工作区干净**

Run: `git status && git rev-parse --abbrev-ref HEAD`
Expected: `On branch main` + `working tree clean`（容许 `.claude-plugin/marketplace.json` 未跟踪）

- [ ] **Step 0.2: 创建工作分支**

Run: `git checkout -b fyx/feature/20260522_team_entry_points`
Expected: `Switched to a new branch 'fyx/feature/20260522_team_entry_points'`

---

## Task 1: team SKILL frontmatter argument-hint 扩展

**Files:**
- Modify: `skills/team/SKILL.md` 第 6 行

- [ ] **Step 1.1: 改 argument-hint**

Edit `skills/team/SKILL.md`:

old_string:
```
argument-hint: <需求描述或PRD文件路径>
```

new_string:
```
argument-hint: "[--from=<phase>] <需求描述或PRD路径，--from=reviewer 时可选>"
```

- [ ] **Step 1.2: 验证 frontmatter 仍能被 yaml 解析**

Run: `head -10 skills/team/SKILL.md`
Expected: 前 7 行包含完整 frontmatter，第 6 行展示新 argument-hint。

- [ ] **Step 1.3: 提交**

```bash
git add skills/team/SKILL.md
git commit -m "feat(team): argument-hint 支持 --from=<phase> 参数"
```

---

## Task 2: team SKILL Phase 0 重写

**Files:**
- Modify: `skills/team/SKILL.md` 第 76-107 行（Phase 0 整段）

整段替换。包含 5 个子节：0.0 参数解析、0.1 前置校验、0.2 工作空间初始化（含脏工作区检测）、0.3 分支创建（fyx/feature/）、0.4 TeamCreate、0.5 条件 TaskCreate。

- [ ] **Step 2.1: 替换整段 Phase 0**

Edit `skills/team/SKILL.md`:

old_string:
```
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
```

new_string:
```
## Phase 0: 准备（参数解析 + 前置校验 + 条件初始化）

### 0.0 参数解析
从 $ARGUMENTS 解析 `--from=<phase>`：
- 缺省值：`analyst`
- 合法值：`analyst | tech-lead | dev | tester | reviewer`
- 非法值 → ❌ 报错列出合法值，**不创建团队，退出**：
  ```
  [❌] 非法的 --from 值: {收到的值}
  合法值: analyst | tech-lead | dev | tester | reviewer
  ```

剩余 $ARGUMENTS（去掉 --from=X 之后）作为需求文本，用于后续起步成员的 prompt。
设当前变量 `START_PHASE = {解析后的 phase 值}`。

### 0.1 前置校验
按 START_PHASE 跑对应检查（任一失败立即报错退出）：

| START_PHASE | 必备前置 |
|---|---|
| `analyst` | 需求文本（去 --from 后的 $ARGUMENTS）非空 |
| `tech-lead` | `.claude/workspace/task_plan.md` 存在 |
| `dev` | `.claude/workspace/task_plan.md` 存在 AND `.claude/workspace/architecture.md` 存在 |
| `tester` | `task_plan.md` + `architecture.md` 都存在 AND（`git diff master...HEAD` 非空 OR working tree 有未提交改动）|
| `reviewer` | `git diff master...HEAD` 非空 OR working tree 有未提交改动 |

校验失败 → 统一报错模板（**不创建团队，退出**）：
```
[❌] --from={START_PHASE} 需要以下前置文件:
   - .claude/workspace/{缺失文件}  (缺失)
   {若涉及代码变更校验}: git diff master...HEAD 与 working tree 均无变更

补齐选项:
  1) 手写 {缺失文件} 后重跑 /team --from={START_PHASE} "..."
  2) 退一档: /team --from={上游phase} "..." 让上游重新生成
  3) 完整流程: /team "..."
  4) 从历史归档复活: ls .claude/workspace/archive/
```

### 0.2 工作空间初始化
- 确保 `.claude/workspace/` 目录存在（不存在则 mkdir）

**IF `START_PHASE in [analyst, tech-lead]`（全新流程语义）：**
- 检测下游产物清单：`architecture.md` / `findings.md` / `progress.md` / `test_report.md` / `review_report.md`（`START_PHASE=analyst` 时清单加上 `task_plan.md`）
- 若清单中任一文件存在 → 触发脏工作区交互（暂停并向用户呈现 3 选 1）：
  ```
  [⚠️] 检测到 workspace 中已有以下下游产物:
     - {file} (last modified: {date})
     ...
  继续将覆盖。请选择:
    1) 归档旧产物到 archive/ 后继续 (推荐)
    2) 直接覆盖
    3) 取消，先用 --from={能复用的上游phase} 复用现有产物
  ```
  - 选 1 → `mkdir -p .claude/workspace/archive/{YYYYMMDD_HHMMSS}_orphaned/ && mv {下游产物} archive/{目录}/`
  - 选 2 → 删除下游产物清单中所有文件
  - 选 3 → 退出，不创建团队

**IF `START_PHASE in [dev, tester, reviewer]`（显式复用 workspace）：**
- 保留 0.1 校验通过的前置文件 + `findings.md`
- **静默清理**当前起步之后阶段的旧产物（覆盖即可，不需要交互）：
  - `dev` → 清 `progress.md` / `test_report.md` / `review_report.md`
  - `tester` → 清 `test_report.md` / `review_report.md`
  - `reviewer` → 清 `review_report.md`

### 0.3 分支创建
**IF `START_PHASE in [analyst, tech-lead]`：**
```bash
git checkout -b fyx/feature/{YYYYMMDD}_{简短描述}
```

**IF `START_PHASE in [dev, tester, reviewer]`：**
- 不新建分支
- 仅打印当前分支并提示用户确认：
  ```
  [ℹ️] 沿用当前分支: {git rev-parse --abbrev-ref HEAD}
  是否继续？Y/n
  ```
- 用户选 n → 退出，不创建团队

### 0.4 创建团队
```
TeamCreate(
  team_name: "dev-team-{简短描述}",
  description: "开发任务：{需求简述}（起步 phase: {START_PHASE}）"
)
```

### 0.5 初始化进度跟踪（按 START_PHASE 条件 TaskCreate）
仅创建从 START_PHASE 对应阶段开始的 Phase 任务：

| START_PHASE | 创建的 TaskCreate |
|---|---|
| `analyst` | Phase 1-6 全部 6 个任务 |
| `tech-lead` | Phase 2-6 共 5 个任务 |
| `dev` | Phase 3-6 共 4 个任务 |
| `tester` | Phase 4-6 共 3 个任务 |
| `reviewer` | Phase 5-6 共 2 个任务 |

每个 TaskCreate 使用与原版相同的 subject/description（Phase 1: 需求分析 / Phase 2: 技术设计 / Phase 3: 编码实现 / Phase 4: 测试验证 / Phase 5: 代码审查 / Phase 6: 收尾归档）。
```

- [ ] **Step 2.2: 验证 Phase 0 新内容存在且行号合理**

Run: `grep -n "Phase 0:" skills/team/SKILL.md | head -3 && grep -c "START_PHASE" skills/team/SKILL.md`
Expected: 第一行匹配 `## Phase 0: 准备（参数解析 + 前置校验 + 条件初始化）`；`START_PHASE` 出现 ≥ 8 次（在 0.0-0.5 各处引用）。

- [ ] **Step 2.3: 验证旧 Phase 0 内容已被移除**

Run: `grep -c "git checkout -b feature/{YYYYMMDD}_{简短描述}" skills/team/SKILL.md`
Expected: 0（旧的非前缀分支命名应已被删）

- [ ] **Step 2.4: 提交**

```bash
git add skills/team/SKILL.md
git commit -m "feat(team): Phase 0 重写为参数解析+前置校验，分支命名加 fyx/ 前缀"
```

---

## Task 3: team SKILL Phase 1-5 起首守卫插入

**Files:**
- Modify: `skills/team/SKILL.md` Phase 1/2/3/4/5 各 phase 标题下首行

每个 Phase 加一行起首守卫：`IF START_PHASE 对应序号 > 本 Phase 序号: 跳过整个 phase`。

约定：analyst=1 / tech-lead=2 / dev=3 / tester=4 / reviewer=5。

- [ ] **Step 3.1: Phase 1 起首守卫**

Edit `skills/team/SKILL.md`:

old_string:
```
## Phase 1: 需求分析

### 1.1 启动 analyst 成员
```

new_string:
```
## Phase 1: 需求分析

> **起首守卫**：IF `START_PHASE in [tech-lead, dev, tester, reviewer]` → 跳过整个 Phase 1（不启动 analyst，不读写 task_plan.md，直接进入 Phase 2 的起首守卫）。

### 1.1 启动 analyst 成员
```

- [ ] **Step 3.2: Phase 2 起首守卫**

Edit `skills/team/SKILL.md`:

old_string:
```
## Phase 2: 技术设计

### 2.1 启动 tech-lead 成员
```

new_string:
```
## Phase 2: 技术设计

> **起首守卫**：IF `START_PHASE in [dev, tester, reviewer]` → 跳过整个 Phase 2（不启动 tech-lead，直接进入 Phase 3 的起首守卫）。tech-lead 在 Phase 3 dev 偏离时仍可"按需首次启动"进入纠偏模式，见 §6.1 表。

### 2.1 启动 tech-lead 成员
```

- [ ] **Step 3.3: Phase 3 起首守卫**

Edit `skills/team/SKILL.md`:

old_string:
```
## Phase 3: 编码实现

### 3.1 启动 dev 成员
```

new_string:
```
## Phase 3: 编码实现

> **起首守卫**：IF `START_PHASE in [tester, reviewer]` → 跳过整个 Phase 3（不启动 dev，不读写 progress.md，直接进入 Phase 4 的起首守卫）。dev 在 Phase 4/5 闭环 BLOCK 时仍可"按需首次启动"进入清单驱动模式，见 §6.1 表与 §3.x 清单驱动模式 prompt。

### 3.1 启动 dev 成员
```

- [ ] **Step 3.4: Phase 4 起首守卫**

Edit `skills/team/SKILL.md`:

old_string:
```
## Phase 4: 测试验证（不可跳过）

```
当前测试轮次 = 1
```

new_string:
```
## Phase 4: 测试验证（不可跳过）

> **起首守卫**：IF `START_PHASE = reviewer` → 跳过整个 Phase 4（reviewer 起步直接进入 Phase 5）。tester 在 Phase 5 reviewer BLOCK 后仍可"按需首次启动"进入回归测试，见 §6.1 表。

```
当前测试轮次 = 1
```

- [ ] **Step 3.5: Phase 5 无需起首守卫（reviewer 是最后一个可起始角色）**

仅在 Phase 5 标题下加一行说明：

Edit `skills/team/SKILL.md`:

old_string:
```
## Phase 5: 代码审查（不可跳过）

```
当前审查轮次 = 1
```

new_string:
```
## Phase 5: 代码审查（不可跳过）

> **起首守卫**：无（reviewer 是 --from 支持的最后一个起始 phase）。但 `START_PHASE = reviewer` 时本 phase 内的 dev 召回与 tester 回归走"按需首次启动"分支，见 §6.1 表与本 phase 5.4 的"召回 dev"步骤。

```
当前审查轮次 = 1
```

- [ ] **Step 3.6: 验证 5 处守卫均已插入**

Run: `grep -c "起首守卫" skills/team/SKILL.md`
Expected: 5

- [ ] **Step 3.7: 提交**

```bash
git add skills/team/SKILL.md
git commit -m "feat(team): Phase 1-5 各加起首守卫，按 START_PHASE 跳过整段"
```

---

## Task 4: team SKILL 成员生命周期表新增"按起步 phase 对齐"小表

**Files:**
- Modify: `skills/team/SKILL.md` "团队成员配置" 节末尾（约第 32 行 sonnet[1m] 说明后）

在现有大表下方追加新的对齐表。

- [ ] **Step 4.1: 在团队成员配置节末尾追加 §6.1 表**

Edit `skills/team/SKILL.md`:

old_string:
```
> Sonnet 一律使用 `[1m]` 后缀启用 1M context 窗口，避免长流程下频繁触发自动压缩导致代码视图丢失。仅 model 字段处需带后缀，其余文档表述沿用 "sonnet"。
```

new_string:
```
> Sonnet 一律使用 `[1m]` 后缀启用 1M context 窗口，避免长流程下频繁触发自动压缩导致代码视图丢失。仅 model 字段处需带后缀，其余文档表述沿用 "sonnet"。

## §6.1 按起步 phase 对齐的生命周期

`--from=<phase>` 等于声明"X 之前的角色不主动启动"。下游闭环路径若需要这些角色，按需首次启动并进入对应模式（dev → 清单驱动模式；tech-lead → 纠偏模式）。

| START_PHASE | analyst | tech-lead | dev | tester | reviewer |
|---|---|---|---|---|---|
| `analyst` | P1 启 / P1 关 | P2 启 / P3 关 | P3 启 / P5 APPROVE 后关 | 每轮启关 | 每轮启关 |
| `tech-lead` | — | P2 启 / P3 关 | P3 启 / P5 APPROVE 后关 | 每轮启关 | 每轮启关 |
| `dev` | — | **按需启动**（dev 偏离时纠偏模式） | P3 启 / P5 APPROVE 后关 | 每轮启关 | 每轮启关 |
| `tester` | — | — | **按需启动**（BLOCK 时清单驱动模式） | P4 每轮启关 | 每轮启关 |
| `reviewer` | — | — | **按需启动**（BLOCK 时清单驱动模式） | **按需启动**（reviewer BLOCK 后回归） | P5 每轮启关 |

> 「按需启动」=该角色在本任务中是"首次出现"，team 使用 `Agent(...)` 启动而非 `SendMessage`。首次启动不存在前一轮 shutdown_approved 的等待。
```

- [ ] **Step 4.2: 验证新表存在**

Run: `grep -n "§6.1 按起步 phase 对齐的生命周期" skills/team/SKILL.md`
Expected: 命中一行。

- [ ] **Step 4.3: 提交**

```bash
git add skills/team/SKILL.md
git commit -m "feat(team): 新增 §6.1 按起步 phase 对齐的生命周期表"
```

---

## Task 5: team SKILL 新增"清单驱动模式 prompt 模板"

**Files:**
- Modify: `skills/team/SKILL.md` "成员生命周期管理" 大节末尾（"清理团队"小节之后、`# 执行流程` 之前的位置）

新增独立小节 §X.Y "dev 启动 prompt 模板"，把现有 Phase 3.1 的 prompt 抽出来命名为"编码模式"，并新增"清单驱动模式"。

- [ ] **Step 5.1: 在"清理团队"小节末尾追加新模板节**

Edit `skills/team/SKILL.md`:

old_string:
```
**TeamDelete 仅清理 config + 任务列表，不会 kill 成员进程**。Phase 6 必须执行孤儿进程检查（见 6.6）。

---

# 执行流程
```

new_string:
```
**TeamDelete 仅清理 config + 任务列表，不会 kill 成员进程**。Phase 6 必须执行孤儿进程检查（见 6.6）。

## dev 启动 prompt 模板（两种模式）

dev 的 Agent() 启动 prompt 分两种模式，由 START_PHASE 与触发时机决定使用哪个：

### 模式 A — 编码模式

**使用时机**：`START_PHASE in [analyst, tech-lead, dev]` 时 Phase 3 首次启动 dev。

```
你是开发团队的后端开发工程师。执行 /dev 技能。读取 .claude/workspace/architecture.md 实现编码。每完成一个模块通知 team lead 进度。完成全部编码后通知 team lead。

**改动纪律（贯穿你的整个生命周期）**：
1. **Phase 3 编码阶段**：如发现 architecture.md 有盲区或漏洞，可主动加固，但加固完成后**立即 SendMessage 给 team-lead 报告**（写明：发现的问题 + 你的修复方式 + 是否需要 tech-lead 复核）
2. **Phase 4/5 待命阶段（你已完成编码后保持存活）**：**严禁主动修改代码**。即使发现新 bug 也只能通过 SendMessage 报告，不许擅自动手——这会让 tester 报告基于过期代码视图、reviewer 审查白做
3. **修复阶段**（team-lead 召回你时）：仅修复指定的 bug 清单，不借机做其他改动；修复完成后立即返回待命
```

### 模式 B — 清单驱动模式

**使用时机**：`START_PHASE in [tester, reviewer]` 时 Phase 4 / Phase 5 闭环 BLOCK 后首次启动 dev（dev 在此场景下不存在 Phase 3 编码经历）。

```
你是开发团队的后端开发工程师（清单驱动模式）。
当前流程从 --from={START_PHASE} 进入，跳过了 Phase 3 编码阶段。

任务范围：根据 {test_report.md | review_report.md} 中列出的"问题/缺口清单"逐项处理——**既包括 Bug 修复，也包括补齐报告指出的缺失实现**：
- 未覆盖的验收标准 → 补实现
- 未实施的安全控制（如缺少输入校验、缺少权限检查）→ 补实现
- Bug → 修复

约束：
1. 严格在报告清单内行动；清单外的代码即使你认为有问题也只能 SendMessage 报告，不要借机做无关重构或额外功能
2. 项目根 CLAUDE.md 必读；architecture.md 若存在必读，存在歧义时以 task_plan.md 为准
3. 完成后执行 code-simplifier 并保证编译通过
4. 通知 team lead 并保持 idle，Phase 4/5 代码冻结纪律继续生效
```

> 注：`START_PHASE in [analyst, tech-lead, dev]` 时 Phase 4/5 闭环 BLOCK 后召回 dev 仍用 SendMessage（dev 已存活），SendMessage 内容即"修复模式"指令，由各 Phase 内具体场景描述（见 Phase 4.4 / 5.4）。模式 B 仅用于 dev 在本任务中"首次启动"的场景。

> **维护提醒**：模式 A 的字面 prompt = Phase 3.1 当前 dev 启动 prompt 的副本。修改任一处时必须同时更新两处，否则会出现编码模式启动行为与模板节描述不一致。

---

# 执行流程
```

- [ ] **Step 5.2: 验证两个模式都存在**

Run: `grep -c "模式 A — 编码模式\|模式 B — 清单驱动模式" skills/team/SKILL.md`
Expected: 2

- [ ] **Step 5.3: 提交**

```bash
git add skills/team/SKILL.md
git commit -m "feat(team): 新增 dev 启动 prompt 双模式（编码 / 清单驱动）"
```

---

## Task 6: team SKILL Phase 4 召回逻辑加 dev 首次启动分支

**Files:**
- Modify: `skills/team/SKILL.md` Phase 4.4 内 "IF 测试结论 = ❌ 未通过" 分支

原文是 SendMessage(dev) 修复模式；改为先判断 dev 是否已存活，未存活则用 Agent() 启动模式 B。

- [ ] **Step 6.1: 改 Phase 4.4 未通过分支**

Edit `skills/team/SKILL.md`:

old_string:
```
**IF 测试结论 = ❌ 未通过：**
  → 召回 dev 修复（dev 仍存活）：
  ```
  SendMessage(
    to: "dev",
    message: "修复模式：测试第 {N} 轮未通过。请读取 test_report.md 中的 Bug 清单进行修复。修复后重新执行 code-simplifier。**仅修复指定 Bug，不要借机做其他改动**；完成后通知 team lead 并立即返回待命，代码冻结纪律继续生效。"
  )
  ```
  → 等待 dev 修复完成
  → 当前测试轮次 += 1
  → 回到 4.1 启动新的 tester 实例（前置：确认前一轮 tester 已 shutdown_approved）
```

new_string:
```
**IF 测试结论 = ❌ 未通过：**

  **分支 1：dev 已存活（`START_PHASE in [analyst, tech-lead, dev]`）：**
  ```
  SendMessage(
    to: "dev",
    message: "修复模式：测试第 {N} 轮未通过。请读取 test_report.md 中的 Bug 清单进行修复。修复后重新执行 code-simplifier。**仅修复指定 Bug，不要借机做其他改动**；完成后通知 team lead 并立即返回待命，代码冻结纪律继续生效。"
  )
  ```

  **分支 2：dev 未存活（`START_PHASE = tester`，本任务中首次启动 dev）：**
  ```
  Agent(
    name: "dev",
    team_name: "dev-team-{简短描述}",
    model: sonnet[1m],
    prompt: {模式 B — 清单驱动模式 prompt}  # 见"dev 启动 prompt 模板"节
  )
  ```
  启动后立即把 test_report.md 的位置告知 dev（prompt 中已含读取指引）。

  → 等待 dev 修复完成
  → 当前测试轮次 += 1
  → 回到 4.1 启动新的 tester 实例（前置：确认前一轮 tester 已 shutdown_approved）
```

- [ ] **Step 6.2: 验证两个分支都在**

Run: `grep -n "分支 1：dev 已存活\|分支 2：dev 未存活" skills/team/SKILL.md`
Expected: 两行命中。

- [ ] **Step 6.3: 提交**

```bash
git add skills/team/SKILL.md
git commit -m "feat(team): Phase 4 BLOCK 后召回 dev 区分首次启动 / SendMessage"
```

---

## Task 7: team SKILL Phase 5 召回逻辑加 dev 首次启动 + tester 首次启动分支

**Files:**
- Modify: `skills/team/SKILL.md` Phase 5.4 内 "IF 结论 = BLOCK" 分支

需要两处改造：
1. 召回 dev 修复时区分"已存活/未存活"（同 Task 6）
2. 启动 tester 回归时区分"本任务中是否已启动过 tester"——`START_PHASE = reviewer` 时是首次启动；其他情况是新一轮启动（前置：确认前一轮 tester 已 shutdown_approved）

- [ ] **Step 7.1: 改 Phase 5.4 BLOCK 分支**

Edit `skills/team/SKILL.md`:

old_string:
```
**IF 结论 = BLOCK：**
  → 召回 dev 修复（dev 仍存活）：
  ```
  SendMessage(
    to: "dev",
    message: "修复模式：代码审查第 {N} 轮 BLOCK。请读取 review_report.md 中的问题清单进行修复。修复后重新执行 code-simplifier。**仅修复指定问题，不要借机做其他改动**；完成后通知 team lead 并立即返回待命，代码冻结纪律继续生效。"
  )
  ```
  → 等待 dev 修复完成
  → **修复后必须经过 tester 回归**（防止修复引入新 bug）：
    启动新的 tester 实例（前置：确认前一轮 tester 已 shutdown_approved）：
    ```
    Agent(
      name: "tester",
      team_name: "dev-team-{简短描述}",
      model: sonnet[1m],
      prompt: "你是开发团队的测试工程师。执行 /tester 技能（回归测试模式）。仅回归 reviewer 要求修复的变更及关联影响。完成后通知 team lead。代码视图一致性纪律：测试前后两次 git diff 对比、所有代码现状陈述引用文件:行号——同 Phase 4.1。"
    )
    ```
    → 等待 tester 完成 → 关闭 tester（等待 shutdown_approved 后再继续）：
    ```
    SendMessage(to: "tester", message: {"type": "shutdown_request"})
    ```
  → 当前审查轮次 += 1
  → 回到 5.1 启动新的 reviewer 实例（前置：确认前一轮 reviewer 已 shutdown_approved）
```

new_string:
```
**IF 结论 = BLOCK：**

  ### 5.4.1 召回 dev 修复

  **分支 1：dev 已存活（`START_PHASE in [analyst, tech-lead, dev, tester]`）：**
  > 说明：`START_PHASE = tester` 时若 Phase 4 触发过 BLOCK，已经在 Task 6 的分支 2 中首次启动过 dev；这里 dev 仍存活。
  ```
  SendMessage(
    to: "dev",
    message: "修复模式：代码审查第 {N} 轮 BLOCK。请读取 review_report.md 中的问题清单进行修复。修复后重新执行 code-simplifier。**仅修复指定问题，不要借机做其他改动**；完成后通知 team lead 并立即返回待命，代码冻结纪律继续生效。"
  )
  ```

  **分支 2：dev 未存活（`START_PHASE = reviewer` 且 Phase 4 未启动 dev，即本任务中首次启动 dev）：**
  ```
  Agent(
    name: "dev",
    team_name: "dev-team-{简短描述}",
    model: sonnet[1m],
    prompt: {模式 B — 清单驱动模式 prompt}  # 见"dev 启动 prompt 模板"节
  )
  ```

  → 等待 dev 修复完成

  ### 5.4.2 修复后必须经过 tester 回归（防止修复引入新 bug）

  **分支 1：tester 历史轮次已运行过（`START_PHASE != reviewer`）：**
  - 前置：确认前一轮 tester 已 shutdown_approved

  **分支 2：tester 本任务中首次启动（`START_PHASE = reviewer`）：**
  - 无需等待历史 shutdown_approved（不存在前一轮）

  两个分支共用启动命令：
  ```
  Agent(
    name: "tester",
    team_name: "dev-team-{简短描述}",
    model: sonnet[1m],
    prompt: "你是开发团队的测试工程师。执行 /tester 技能（回归测试模式）。仅回归 reviewer 要求修复的变更及关联影响。完成后通知 team lead。代码视图一致性纪律：测试前后两次 git diff 对比、所有代码现状陈述引用文件:行号——同 Phase 4.1。"
  )
  ```
  → 等待 tester 完成 → 关闭 tester（等待 shutdown_approved 后再继续）：
  ```
  SendMessage(to: "tester", message: {"type": "shutdown_request"})
  ```

  → 当前审查轮次 += 1
  → 回到 5.1 启动新的 reviewer 实例（前置：确认前一轮 reviewer 已 shutdown_approved）
```

- [ ] **Step 7.2: 验证 5.4.1/5.4.2 都存在**

Run: `grep -c "5.4.1 召回 dev 修复\|5.4.2 修复后必须经过 tester 回归" skills/team/SKILL.md`
Expected: 2

- [ ] **Step 7.3: 提交**

```bash
git add skills/team/SKILL.md
git commit -m "feat(team): Phase 5 BLOCK 后 dev/tester 召回区分首次启动 / SendMessage"
```

---

## Task 8: team SKILL "纪律"节追加 4 条

**Files:**
- Modify: `skills/team/SKILL.md` 文件末尾"纪律"节

- [ ] **Step 8.1: 追加 4 条纪律**

Edit `skills/team/SKILL.md`:

old_string:
```
9. **启动同名新一轮成员前必须等前一轮 shutdown_approved** — 避免系统自动加 `-2`/`-3` 后缀导致 SendMessage 定位错误
```

new_string:
```
9. **启动同名新一轮成员前必须等前一轮 shutdown_approved** — 避免系统自动加 `-2`/`-3` 后缀导致 SendMessage 定位错误
10. **--from=X 等于声明 X 之前的角色不主动启动** — 但若下游闭环路径需要它们，按需首次启动并使用对应模式 prompt（dev → 清单驱动模式；tech-lead → 纠偏模式）。禁止"--from=X 就完全禁用 X 之前的角色"的简化理解，会破坏闭环可达性
11. **--from in [dev, tester, reviewer] 时沿用当前分支** — 不新建 feature 分支；启动时打印当前分支供用户确认
12. **--from=X 的前置校验失败必须立即报错暂停** — 不允许"静默回退到上游 phase 补生成"，否则用户会误以为跳过了实际没跳
13. **Phase 6 归档/孤儿检查无论起步 phase 如何都必须执行** — 包括 --from=reviewer 一次 APPROVE 的最短路径
```

- [ ] **Step 8.2: 验证 13 条纪律齐全**

Run: `grep -cE "^[0-9]+\. \*\*" skills/team/SKILL.md`
Expected: ≥ 13

- [ ] **Step 8.3: 提交**

```bash
git add skills/team/SKILL.md
git commit -m "feat(team): 纪律节追加 4 条 --from 相关约束"
```

---

## Task 9: team SKILL "/dev 关系" 节 + "终止条件" 表更新

**Files:**
- Modify: `skills/team/SKILL.md` "如何选择入口"表 + "终止条件与升级机制"表

- [ ] **Step 9.1: 更新"如何选择入口"表，添加 /team --from=dev 行**

Edit `skills/team/SKILL.md`:

old_string:
```
| 你想要的效果 | 正确入口 |
|---|---|
| 完整流程：需求分析 + 架构设计 + 编码 + 测试 + 审查 + 归档 | `/team <需求>` |
| 单 dev 直接编码（自行写简版 architecture.md + 自建 feature 分支 + 完成后 code-simplifier）**不含测试/审查闭环** | `/dev <需求>` |
| 跳过需求分析/架构设计，但仍要 tester/reviewer 闭环 | **当前无任何入口支持此组合。** `/team` 无条件启动 Phase 1（analyst）和 Phase 2（tech-lead），不识别任何"跳过"参数或自然语言说明；`/dev` 则根本不含 tester/reviewer 闭环。若必须要这个组合，目前只能改走 `/team` 完整流程（多花 Phase 1/2 的 token 但功能达到）。改造路径见下方「若未来要实现"精简带闭环"入口」节。 |
```

new_string:
```
| 你想要的效果 | 正确入口 |
|---|---|
| 完整流程：需求分析 + 架构设计 + 编码 + 测试 + 审查 + 归档 | `/team <需求>` |
| 已有需求文档（task_plan.md）→ 从技术设计开始 | `/team --from=tech-lead [<需求补充>]` |
| 已有需求 + 架构（task_plan.md + architecture.md）→ 从编码开始，**保留** 测试/审查闭环 | `/team --from=dev [<需求补充>]` |
| 已有代码 → 跑测试+审查闭环 | `/team --from=tester [<需求补充>]` |
| 已有代码 → 只做一次代码审查（含 P0 安全审查） | `/team --from=reviewer [<需求补充>]` |
| 单 dev 直接编码（自行写简版 architecture.md + 自建 feature 分支 + 完成后 code-simplifier）**不含**测试/审查闭环 | `/dev <需求>` |

> `/team --from=dev` ≠ `/dev`：前者**含** Phase 4/5 闭环，后者**不含**。
```

- [ ] **Step 9.2: 删除已落地的 "若未来要实现「精简带闭环」入口" 节**

Edit `skills/team/SKILL.md`:

old_string:
```
## 若未来要实现"精简带闭环"入口

需要在 team skill 的 Phase 0 增加参数识别逻辑，或新增一个独立 skill（**不能**直接用 `/dev` 作为 slash command，因为会与现有 `dev` skill 碰撞命名空间）。

---

# 终止条件与升级机制
```

new_string:
```
---

# 终止条件与升级机制
```

- [ ] **Step 9.3: 更新"终止条件与升级机制"表，新增一行**

Edit `skills/team/SKILL.md`:

old_string:
```
| **异常终止/人工中断** | 关闭所有存活成员 → TeamDelete 清理 |
```

new_string:
```
| **异常终止/人工中断** | 关闭所有存活成员 → TeamDelete 清理 |
| **--from=X 前置校验失败** | 暂停，列出缺失文件 + 4 种补齐选项（手写 / 退一档 / 完整流程 / 历史归档复活），不创建团队 |
```

- [ ] **Step 9.4: 验证三处都改了**

Run: `grep -c "/team --from=dev\|/team --from=tech-lead\|--from=X 前置校验失败" skills/team/SKILL.md`
Expected: ≥ 3

- [ ] **Step 9.5: 验证旧"无任何入口支持"措辞已删除**

Run: `grep -c "当前无任何入口支持此组合" skills/team/SKILL.md`
Expected: 0

- [ ] **Step 9.6: 提交**

```bash
git add skills/team/SKILL.md
git commit -m "feat(team): /dev 关系表与终止条件表同步 --from 入口"
```

---

## Task 10: dev SKILL 加清单驱动模式识别 + 分支命名同步

**Files:**
- Modify: `skills/dev/SKILL.md` Step 1 第 45 行（分支命名）
- Modify: `skills/dev/SKILL.md` Step 2 前（在复杂度评估前加清单驱动模式跳过逻辑）

- [ ] **Step 10.1: 改 Step 1 分支命名**

Edit `skills/dev/SKILL.md`:

old_string:
```
- 自动创建 feature 分支: `feature/{日期}_{简短描述}`
```

new_string:
```
- 自动创建 feature 分支: `fyx/feature/{日期}_{简短描述}`
```

- [ ] **Step 10.2: 在"Step 2: 复杂度评估"前插入清单驱动模式识别**

Edit `skills/dev/SKILL.md`:

old_string:
```
## Step 2: 复杂度评估

读取 architecture.md 后按以下维度打分：
```

new_string:
```
## Step 1.6: 启动模式识别（清单驱动 vs 编码）

**如果 team-lead 在启动 prompt 中显式标注"（清单驱动模式）"**：
- 跳过 Step 2 复杂度评估（清单驱动场景不需要升级为 dev-leader 团队，规模由 test_report.md / review_report.md 中的问题清单决定）
- 跳过 Step 3A/3B 模块化实现路径，直接读取 test_report.md 或 review_report.md，按问题清单逐项处理
- 仍走 Step 1.5 的 Codex/Claude 引擎选择 + Step 4 的格式化/编译/code-simplifier 强制门禁

否则按下文 Step 2 -> 3 -> 4 -> 5 正常流程。

## Step 2: 复杂度评估

读取 architecture.md 后按以下维度打分：
```

- [ ] **Step 10.3: 验证两处改动**

Run: `grep -c "fyx/feature/{日期}_{简短描述}\|清单驱动模式" skills/dev/SKILL.md`
Expected: ≥ 2

- [ ] **Step 10.4: 提交**

```bash
git add skills/dev/SKILL.md
git commit -m "feat(dev): 识别清单驱动模式跳过复杂度评估，分支命名加 fyx/ 前缀"
```

---

## Task 11: 项目 CLAUDE.md 三处同步

**Files:**
- Modify: `/Users/macpro/claude-plugin/backend-team-workflow/CLAUDE.md` 三处

- [ ] **Step 11.1: 架构节生命周期表后加 §6.1 引用**

Edit `CLAUDE.md`:

old_string:
```
理解整体协作必须把 `skills/team/SKILL.md` 当作"主控代码"读：它是唯一驱动所有阶段切换、生命周期管理、闭环判定的逻辑。其余 5 个 skill 只是被它通过 `Agent(...)` 启动并通过 `SendMessage` 召回 / 关停的子角色。
```

new_string:
```
理解整体协作必须把 `skills/team/SKILL.md` 当作"主控代码"读：它是唯一驱动所有阶段切换、生命周期管理、闭环判定的逻辑。其余 5 个 skill 只是被它通过 `Agent(...)` 启动并通过 `SendMessage` 召回 / 关停的子角色。

> 注：以上生命周期适用于默认完整流程（`/team` 不带参数）。`/team --from=<phase>` 入口下具体存活范围会按起步 phase 调整，见 `skills/team/SKILL.md` §6.1 表。
```

- [ ] **Step 11.2: 已知陷阱节扩写 /dev ≠ /team 那条**

Edit `CLAUDE.md`:

old_string:
```
- **`/dev` ≠ 精简版 `/team`**：`/dev` 走独立 skill，**不含** tester/reviewer 闭环。详见 `skills/team/SKILL.md` 末尾「`/dev` 与本 skill 的关系」节。如需"跳过分析+设计但保留测试+审查"，目前没有任何入口支持。
```

new_string:
```
- **`/dev` ≠ 精简版 `/team`**：`/dev` 走独立 skill，**不含** tester/reviewer 闭环。如需"跳过分析+设计但保留测试+审查"，使用 `/team --from=dev`（含 Phase 4/5 闭环）；`/dev` 仍是"单 dev 编码无闭环"路径。详见 `skills/team/SKILL.md` 末尾「`/dev` 与本 skill 的关系」节与 §6.1 表。
```

- [ ] **Step 11.3: 已知陷阱节分支命名示例更新**

Edit `CLAUDE.md`:

old_string:
```
- **`/dev` 与 `/team` 都会创建 feature 分支**：`feature/{YYYYMMDD}_{简短描述}`。两者在同一项目接续使用时不要互相覆盖分支。
```

new_string:
```
- **`/dev` 与 `/team` 都会创建 feature 分支**：`fyx/feature/{YYYYMMDD}_{简短描述}`（带用户前缀 `fyx/`）。两者在同一项目接续使用时不要互相覆盖分支。`/team --from in [dev, tester, reviewer]` 不新建分支而沿用当前分支。
```

- [ ] **Step 11.4: 验证三处都改了**

Run: `grep -c "fyx/feature/{YYYYMMDD}_{简短描述}\|§6.1 表\|/team --from=dev" CLAUDE.md`
Expected: ≥ 3

- [ ] **Step 11.5: 提交**

```bash
git add CLAUDE.md
git commit -m "docs: CLAUDE.md 三处同步 --from 入口与 fyx/feature 命名"
```

---

## Task 12: 真实项目冒烟验证

**Files:**
- 无文件改动 — 在目标业务项目中执行

> 本仓库无构建/测试。"验证"等价于把 plugin 重新装载，然后在一个真实业务项目里跑 spec §9 的 5 个用例。

- [ ] **Step 12.1: 重装 plugin（本地开发期）**

Run（在 Claude Code 内）：`/plugin marketplace add ~/claude-plugin/backend-team-workflow` 然后 `/reload-plugins`
Expected: `backend-team-workflow` 重新加载，`/team` 显示新的 argument-hint。

- [ ] **Step 12.2: 用例 1 — 回归基线（缺省 /team）**

在一个有 CLAUDE.md 的目标项目里执行 `/team "微小改动，比如给某 Service 加日志"`。
Expected:
- 在目标项目内 `git branch --show-current` 输出 `fyx/feature/2026XXXX_xxx`（带 fyx/ 前缀）
- 走完 Phase 1-6 全流程，生命周期与改造前完全一致

- [ ] **Step 12.3: 用例 2 — `/team --from=tech-lead`**

预置：在目标项目手写一份 `.claude/workspace/task_plan.md`（5-10 行需求点即可）
Run: `/team --from=tech-lead`
Expected:
- 跳过 Phase 1（无 analyst 启动日志）
- 新建分支 `fyx/feature/...`
- 进入 Phase 2 tech-lead 设计

- [ ] **Step 12.4: 用例 3 — `/team --from=dev`**

预置：上一步基础上由 tech-lead 产出 `architecture.md`，然后清掉本次 team，重跑：
Run: `/team --from=dev`
Expected:
- 跳过 Phase 1+2
- **不新建分支**，打印当前分支并提示确认
- 进入 Phase 3 dev 编码（模式 A）

- [ ] **Step 12.5: 用例 4 — `/team --from=tester`**

预置：写一段代码改动（任意 mv 文件即可制造 git diff）
Run: `/team --from=tester`
Expected:
- 跳过 Phase 1+2+3
- 沿用当前分支
- 进入 Phase 4 tester 测试

- [ ] **Step 12.6: 用例 5 — `/team --from=reviewer` 直接 APPROVE**

预置：保留上一步的代码改动
Run: `/team --from=reviewer`
Expected:
- 跳过 Phase 1+2+3+4
- reviewer 第一轮 APPROVE → 直接进入 Phase 6 归档
- Phase 6.6 孤儿进程检查仍执行

- [ ] **Step 12.7: 错误路径 — 缺前置文件**

清空目标项目 `.claude/workspace/` 后 Run: `/team --from=dev`
Expected:
- 立即报错列出 `task_plan.md` + `architecture.md` 缺失
- 列出 4 种补齐选项
- 不创建团队，不切换分支

- [ ] **Step 12.8: 错误路径 — 非法 phase 值**

Run: `/team --from=foo "..."`
Expected:
- 立即报错列出合法 5 个值
- 不创建团队

- [ ] **Step 12.9: 闭环可达性 — `--from=reviewer` BLOCK 路径**

构造代码改动里故意留一个明显问题（例如 SQL 拼接漏洞），Run: `/team --from=reviewer`
Expected:
- reviewer 报 BLOCK
- team 首次启动 dev（模式 B 清单驱动模式 prompt）
- dev 修复后首次启动 tester 跑回归
- 进入下一轮 reviewer

- [ ] **Step 12.10: 全部用例通过后由用户决定合并方式**

向用户报告："冒烟全部通过，工作分支 fyx/feature/20260522_team_entry_points 准备合并 main。是用 ff merge / no-ff merge / squash merge / 还是开 PR 走 review 流程？"

等用户回复后执行对应命令。**不主动 push origin main**（属于 destructive 操作，需用户授权）。

> 任一用例失败 → 把 bug 反馈到本计划文档对应 Task 重做。

---

## Self-Review 结论

1. **Spec coverage**：spec §3-§8 + §9 验证策略全部映射到 Task 1-12。spec §7.4「不在 scope」明确排除 `--to=` / `--skip=` / 多 PRD / 跨实例 workspace —— 本 plan 也未引入这些。✅
2. **Placeholder scan**：所有 Edit 步骤都给出了完整 old/new content；所有命令都是可直接执行的字面命令；无 "TBD" / "implement later" / "similar to" 。✅
3. **Type consistency**：`START_PHASE`、`fyx/feature/{YYYYMMDD}_{简短描述}`、`模式 A — 编码模式 / 模式 B — 清单驱动模式`、`§6.1` 等命名在 Task 2-11 中保持一致。✅
4. **可读性**：每个 Task 平均 3-7 个 step，单 step 在 2-5 分钟内完成。Task 6/7/9 因为修改片段较大 step 略多，可接受。
