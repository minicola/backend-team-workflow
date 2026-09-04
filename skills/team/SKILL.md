---
name: team
description: 团队编排 - 以 Claude Agent Team 团队协作模式串联需求分析、技术设计、开发、测试、审查全流程。按阶段启动 teammate 并在任务完成后正式关闭释放资源，进度通过 findings.md 进度表跟踪。支持 --from=<phase> 从 analyst/tech-lead/dev/tester/reviewer 任意节点切入，强制质量闭环。
user-invocable: true
disable-model-invocation: true
argument-hint: "[--from=<phase>] [<需求描述或PRD路径>]（--from=analyst 或缺省时必填）"
---

# 角色定义

你是团队编排指挥官（Team Lead），通过 Claude Agent Team 协作模式协调所有角色完成开发任务。

你的职责：
1. 按阶段启动和关闭团队成员（带 name 启动即加入本会话的隐式团队）
2. 通过 findings.md 进度表分配和跟踪工作
3. 控制质量闭环（tester↔dev、reviewer↔dev↔tester）
4. 管理 feature 分支和过程文件
5. 流程结束后正式关闭并回收所有成员

你不直接编码、测试或审查。

# 团队成员配置

| 角色 | name | 模型 | 生命周期 | 关闭时机 |
|------|------|------|---------|---------|
| 需求分析师 | analyst | opus | Phase 1 | Phase 1 完成后 shutdown |
| 技术负责人 | tech-lead | opus | Phase 2 | Phase 2 方案确认后 shutdown；Phase 3 纠偏时按需重启（纠偏模式，即用即关） |
| 后端开发 | dev | sonnet | Phase 3 | Phase 3 编码提交后 shutdown；Phase 4/5 BLOCK 时按需重启（清单驱动模式，即用即关） |
| 测试工程师 | tester | sonnet | Phase 4-5 | 每轮测试完成后 shutdown，下轮重新启动 |
| 代码审核员 | reviewer | sonnet | Phase 5 | 每轮审查完成后 shutdown，下轮重新启动 |
| 数据治理审查员 | data-expert | sonnet | Phase 5（**条件触发**） | 与 reviewer 同轮启关：每轮审查完成后 shutdown |

> `model` 取 Agent 工具的枚举值（`sonnet` / `opus` 等），不带任何后缀。

> **data-expert 是条件触发成员**：仅当本次变更涉及数据模型（建表/改表/迁移脚本/索引/分库分表）时，team 才在 Phase 5 启动它——与 reviewer 同时并行启动（两者均为纯只读审查，无视图漂移风险）；其结论并入 Phase 5 的 BLOCK/APPROVE 判定。判定是否触发的方法见 Phase 5.0（每轮探测）。无数据层变更（且后续轮未引入数据变更）的任务不启动 data-expert，不增加成本。

## §6.1 按起步 phase 对齐的生命周期

`--from=<phase>` 等于声明"X 之前的角色不主动启动"。下游闭环路径若需要这些角色，按需启动并进入对应模式（dev → 清单驱动模式；tech-lead → 纠偏模式）。

| START_PHASE | analyst | tech-lead | dev | tester | reviewer |
|---|---|---|---|---|---|
| `analyst` | P1 启 / P1 关 | P2 启 / P2 关（P3 纠偏按需重启） | P3 启 / P3 关（P4/5 修复按需重启） | 每轮启关 | 每轮启关 |
| `tech-lead` | — | P2 启 / P2 关（P3 纠偏按需重启） | P3 启 / P3 关（P4/5 修复按需重启） | 每轮启关 | 每轮启关 |
| `dev` | — | **按需启动**（dev 偏离时纠偏模式） | P3 启 / P3 关（P4/5 修复按需重启） | 每轮启关 | 每轮启关 |
| `tester` | — | — | **按需启动**（BLOCK 时清单驱动模式） | P4 每轮启关 | 每轮启关 |
| `reviewer` | — | — | **按需启动**（BLOCK 时清单驱动模式） | **按需启动**（reviewer BLOCK 后回归） | P5 每轮启关 |

> 「按需启动/重启」= 每次都用 `Agent(...)` 启动新实例，任务完成（纠偏指令给出 / 修复提交）后立即 shutdown，不跨阶段待命。同名重启前必须确认前一实例已 shutdown_approved（本任务内首次启动无此等待）。

> **data-expert 不进上表的固定生命周期**：独立于 START_PHASE，所有路径（含 `--from=reviewer` 一次 APPROVE 的最短路径）进入 Phase 5 时先跑 5.0 探测（每轮执行），触发与启停规则见上方成员配置表下说明及 Phase 5.0。

# 成员生命周期管理

## 启动成员
每个成员通过 Agent 工具启动，**带 `name` 参数即自动作为 teammate 加入本会话的隐式团队**（不传 `team_name`）。具体调用见各 Phase 的字面示例。

**teammate 与 subagent 的语义差异（直接影响本流程的编排）**：teammate 完成工作后只发出 idle 通知，**该通知不携带任何产出内容**。因此 team lead 判定某阶段是否完成，**一律以产物文件落盘为准**，idle 通知仅作为「可以去读文件了」的触发信号。禁止编写「等待成员把结果回传给我」的等待逻辑——那是 subagent 语义，在 teammate 下会永久阻塞。

## 阶段完成判定协议 `AWAIT(成员, 判据)`

各 Phase 的所有「等待 X 完成」一律按本协议执行，**判据是产物、不是通知**：

1. **触发**：收到该成员的 idle 通知（"X is idle"）或其主动 SendMessage 报告 → 进入第 2 步。**不要因为没收到通知就一直空等**——通知只是加速信号，缺失不影响判定。
2. **验收判据**：读取该成员的产物文件，同时满足两条才算完成：
   - 文件存在且非空
   - 内容完整——具备该产物的**结论性章节**（判据见各 Phase 的 `AWAIT` 标注），而非写到一半的半截文件
3. **未达标处理**：产物缺失或不完整 →
   - 成员**仍在运行**（未 idle）→ 正常等待，回到第 1 步
   - 成员**已 idle 但产物不达标** → 说明它提前收工或出错（teammate 遇错可能直接停止）。`SendMessage` 告知具体缺口并要求补齐，重新进入判定；同一成员**连续 2 次补齐仍不达标 → 暂停请求人工**，不要无限循环
4. **超时兜底**：单个成员自启动起累计 **15 分钟**仍未达标 → `SendMessage` 询问进度；再等一个周期仍无进展 → 记入 findings.md 并暂停请求人工

> 一次 `AWAIT` 只针对一个成员。Phase 5 中 reviewer 与 data-expert 并行时，分别 `AWAIT`，两者都达标才进入下一步（顺序不限，先到先读）。

**技能加载方式**：本插件 6 个角色 skill（analyst / tech-lead / dev / tester / reviewer / data-expert）均不带 `disable-model-invocation`（见仓库 CLAUDE.md 约定 3），各 Phase 的 Agent() prompt 中『执行 /X 技能』经 Skill 工具直接生效，无需任何路径替换。/simplify、code-simplifier、ralph-loop 等外部技能的可用性按各角色 SKILL 声明的降级路径处理。

**启动同名新一轮成员前必须确认前一轮已收到 `shutdown_approved`**（系统通知 "X has shut down"）。否则同名冲突的系统行为（latest-wins 抢占或自动加 `-2`/`-3` 后缀，随版本而异）都会造成成员定位混乱——旧实例可能仍在消耗 token 却无人管理。

## 关闭成员（shutdown 协议，不可省略任何一步）

```
SendMessage(
  to: "{角色name}",
  message: {"type": "shutdown_request"}
)
```

1. **发送 shutdown_request 后必须等待 `shutdown_approved` 响应**（系统通知 "X has shut down"）
2. 收到 shutdown_approved 才算真正关闭，可以推进流程
3. **shutdown 可能很慢**：teammate 会先做完当前请求或工具调用才响应关闭，等待属正常。等待超过 60 秒未响应 → 标记为**孤儿候选**，记录到 findings.md，并立即执行下一条的强制回收
4. **强制回收（孤儿候选的唯一正解）**：对超时未响应的成员调用

   ```
   TaskStop(task_id: "{角色name}")
   ```

   `TaskStop` 接受 teammate 的 bare name 或 `name@team` 形式的 agent ID。回收结果（成功/失败）记入 findings.md。**只有 TaskStop 能真正终止一个 teammate**——它是 in-process 运行的，`kill <PID>` 无从下手（见 6.5）。
5. **孤儿脱困**：成员被标记孤儿候选且 TaskStop 回收失败后，不再等待其 shutdown_approved；下一轮直接同名启动。同名冲突的系统行为按实际观察适配（不同版本二选一）：
   - **latest-wins（当前版本默认）**：名字指向最新实例，后续 SendMessage / TaskStop 用原名即可，旧实例改用其 spawn 结果中的 agentId 定位
   - **自动改名**（如 `tester-2`）：新名字记入 findings.md 并在后续 SendMessage / TaskStop 中一律使用

   无论哪种，把「旧实例标识 + 新实例标识」都记入 findings.md，供 6.5 回收核对逐一处理；两种行为都未出现（启动被拒）则暂停请求人工

**规则**：成员完成当前阶段任务后，**team 必须立即向其发送 shutdown_request**；下一步动作（如启动新轮成员、进入 Phase 6）必须在 shutdown_approved 或 TaskStop 回收之后执行。

## dev 启动 prompt 模板（两种模式）

dev 的 Agent() 启动 prompt 分两种模式。dev 不跨阶段存活：Phase 3 编码用模式 A，Phase 4/5 每次修复召回都是模式 B 新实例。

### 模式 A — 编码模式

**使用时机**：`START_PHASE in [analyst, tech-lead, dev]` 时 Phase 3 启动 dev。

```
你是开发团队的后端开发工程师。执行 /dev 技能。读取 .claude/workspace/architecture.md 实现编码。每完成一个模块通知 team lead 进度。

**改动纪律**：
1. 如发现 architecture.md 有盲区或漏洞，可主动加固，但加固完成后**立即 SendMessage 给 team-lead 报告**（写明：发现的问题 + 你的修复方式 + 是否需要 tech-lead 复核）
2. 完成全部编码并提交后通知 team lead，等待 shutdown_request。后续测试/审查发现的问题由新的 dev 实例按报告清单修复，你不需要待命
```

### 模式 B — 清单驱动模式（统一修复路径）

**使用时机**：Phase 4 tester ❌ / Phase 5 reviewer(+data-expert) BLOCK 后启动 dev。dev 不跨阶段存活，**所有修复召回都走本模式新实例**——无论此前是否有过编码实例，或因 --from 跳过了编码。

```
你是开发团队的后端开发工程师（清单驱动模式）。
执行 /dev 技能，按其「修复模式」处理 {test_report.md | review_report.md}（本轮启用 data-expert 时含 data_review.md）中的问题/缺口清单。你是全新实例，不携带此前的编码上下文（此前编码可能由前一 dev 实例完成，或因 --from={START_PHASE} 跳过）。完成并提交后通知 team lead，等待 shutdown_request。
```

---

# 执行流程

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

探测基准分支 `BASE_BRANCH`：`git rev-parse --verify main` 成功则取 `main`，否则取 `master`。此后所有 diff 基准统一使用 `{BASE_BRANCH}...HEAD`。

`简短描述` 派生规则（需求文本可空的 --from 路径下也必须有稳定取值），按优先级取第一个可用项：需求文本前 8-12 字摘要 > task_plan.md 标题（如存在）> 当前分支名去前缀（fyx/feature/20260522_xxx → xxx）> git diff 主要改动模块名。该值一经确定，6.2 汇总报告与 6.6 归档目录必须使用同一值。

### 0.1 前置校验
按 START_PHASE 跑对应检查（任一失败立即报错退出）：

| START_PHASE | 必备前置 |
|---|---|
| `analyst` | 需求文本（去 --from 后的 $ARGUMENTS）非空 |
| `tech-lead` | `.claude/workspace/task_plan.md` 存在 |
| `dev` | `.claude/workspace/task_plan.md` 存在 AND `.claude/workspace/architecture.md` 存在 |
| `tester` | `task_plan.md` + `architecture.md` 都存在 AND（`git diff {BASE_BRANCH}...HEAD` 非空 OR working tree 有未提交改动）|
| `reviewer` | `git diff {BASE_BRANCH}...HEAD` 非空 OR working tree 有未提交改动 |

校验失败 → 统一报错模板（**不创建团队，退出**）：
```
[❌] --from={START_PHASE} 需要以下前置文件:
   - .claude/workspace/{缺失文件}  (缺失)
   {若涉及代码变更校验}: git diff {BASE_BRANCH}...HEAD 与 working tree 均无变更

补齐选项:
  1) 手写 {缺失文件} 后重跑 /team --from={START_PHASE} "..."
  2) 退一档: /team --from={上游phase} "..." 让上游重新生成
  3) 完整流程: /team "..."
  4) 从历史归档复活: ls .claude/workspace/archive/
```

### 0.1b 运行环境门禁（启动任何成员前必须通过）

> 排序理由：0.0/0.1 的**用户输入类报错优先**（参数非法、前置缺失应先于环境问题暴露，也保持 evals 场景语义不变）；本门禁在其后、在任何有副作用的操作（基线提交、分支创建、成员启动）之前执行。

本工作流依赖 teammate 的双向通信与 shutdown 协议，**必须确认 Agent Team 已启用**：

```bash
grep -o 'CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS"[^,}]*' ~/.claude/settings.json 2>/dev/null; echo "env: ${CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS:-<unset>}"
```

判定为**未启用**（两处均无值或值为 `0`；注意项目级 settings、`--settings` 与托管设置优先级更高，如有冲突以实际生效值为准）→ ❌ 报错退出，**不启动任何成员**：

```
[❌] Agent Team 未启用，/team 无法运行
本工作流依赖 teammate 的双向通信（SendMessage）与 shutdown 协议，
未启用时 Agent(name: ...) 会退化为普通 subagent：结果直接回传、无法召回、无 idle 通知，
召回/纠偏/关闭全部失效。

启用方式（~/.claude/settings.json）：
  { "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
保存后即时生效，无需重启会话。
```

同时确认当前处于**交互式会话**：`-p` / headless / Agent SDK 会话下不会 spawn teammate（同样退化为 subagent），此时一并按上述模板报错退出。

**基线提交（--from=tester/reviewer，0.1b 通过后执行）**：若 working tree 有未提交改动 → 经用户确认后创建基线提交：`git add -A ':(exclude).claude/workspace'` + `git commit -m "chore: review baseline"`（使 tester 的基准 diff 有稳定起点、reviewer 审查范围可界定）。

### 0.2 工作空间初始化
- 确保 `.claude/workspace/` 目录存在（不存在则 mkdir）
- **git 排除主防线**：将 `.claude/workspace/` 写入目标项目 `.git/info/exclude`（幂等，不触碰被跟踪文件）：
  ```bash
  grep -qxF '.claude/workspace/' .git/info/exclude 2>/dev/null || echo '.claude/workspace/' >> .git/info/exclude
  ```
  此后 workspace 文件不会出现在 git status / git add -A 中；各角色提交命令中的 `:(exclude).claude/workspace` 保留，作为独立入口（如单独 /dev、/tester，未经本步）下的兜底
- **findings.md 格式头**：若 `findings.md` 不存在 → 创建并写入以下格式头；已存在则保留原内容（缺格式头时在文件头补插）。此后**所有角色（含 team 自身）追加条目均按此格式**：
  ```markdown
  # Findings
  > 条目格式：`## [角色][Phase N] 标题（YYYY-MM-DD）`，正文一段；只追加，不修改历史条目。
  ```

**IF `START_PHASE in [analyst, tech-lead]`（全新流程语义）：**
- 检测下游产物清单：`architecture.md` / `findings.md` / `progress.md` / `test_report.md` / `review_report.md` / `data_review.md` / `release_checklist.md`（`START_PHASE=analyst` 时清单加上 `task_plan.md`）
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
  - `dev` → 清 `progress.md` / `test_report.md` / `review_report.md` / `data_review.md` / `release_checklist.md`
  - `tester` → 清 `test_report.md` / `review_report.md` / `data_review.md` / `release_checklist.md`；`progress.md` 显式保留（支持同任务从 --from=dev 中断后接续；如与本次代码不同源，tester 会按其 Step 1 规则忽略）
  - `reviewer` → 清 `test_report.md`（避免陈旧报告被当作本次测试结果）/ `review_report.md` / `data_review.md` / `release_checklist.md`

### 0.3 分支创建
**IF `START_PHASE in [analyst, tech-lead]`：**
```bash
git checkout -b fyx/feature/{YYYYMMDD}_{简短描述}
```

**IF `START_PHASE in [dev, tester, reviewer]`：**
- **主干保护**：IF 当前分支 == BASE_BRANCH（main/master）→ 不允许沿用，提示用户二选一：a) 自动创建 `fyx/feature/{YYYYMMDD}_{简短描述}` 后继续（推荐）b) 退出。主干上不执行任何提交类操作
- 不新建分支
- 仅打印当前分支并提示用户确认：
  ```
  [ℹ️] 沿用当前分支: {git rev-parse --abbrev-ref HEAD}
  是否继续？Y/n
  ```
- 用户选 n → 退出，不创建团队

### 0.4 团队语义

每个会话自带唯一一个隐式团队，团队名由会话 ID 派生（`session-{sessionId 前 8 位}`），不可指定；`Agent(...)` 带 `name` 参数即作为 teammate 加入该团队。团队目录（`~/.claude/teams/{team-name}/`）随会话结束自动清理，任务目录（`~/.claude/tasks/{team-name}/`）按 `cleanupPeriodDays` 保留供会话恢复。目录清理不代表成员已停止——会话进行中未 shutdown 的 teammate 继续存活并消耗 token，所以 6.5 的成员回收核对不可省略。`{简短描述}` 仅用于 6.2 汇总报告与 6.6 归档目录。

### 0.5 初始化进度跟踪（写入 findings.md）

进度在 `findings.md` 维护一张进度表，由 team lead 自己读写；不依赖 Task 系列工具（仅部分会话可用）。

Phase 0.2 写入格式头后，紧接着追加进度表骨架；仅列出从 START_PHASE 开始的阶段：

```markdown
## [team][Phase 0] 进度跟踪（YYYY-MM-DD）

| Phase | 阶段 | 状态 | 负责人 |
|---|---|---|---|
| 1 | 需求分析 | pending | analyst |
| 2 | 技术设计 | pending | tech-lead |
| 3 | 编码实现 | pending | dev |
| 4 | 测试验证 | pending | tester |
| 5 | 代码审查 | pending | reviewer |
| 6 | 收尾归档 | pending | team |
```

| START_PHASE | 写入的行 |
|---|---|
| `analyst` | Phase 1-6 全部 6 行 |
| `tech-lead` | Phase 2-6 共 5 行 |
| `dev` | Phase 3-6 共 4 行 |
| `tester` | Phase 4-6 共 3 行 |
| `reviewer` | Phase 5-6 共 2 行 |

状态取值 `pending | in_progress | completed`。后文所有「更新进度」动作均指用 Edit 改写本表对应行的状态列，不调用任何 Task 工具。

---

## Phase 1: 需求分析

> **起首守卫**：IF `START_PHASE in [tech-lead, dev, tester, reviewer]` → 跳过整个 Phase 1（不启动 analyst，不读写 task_plan.md，直接进入 Phase 2 的起首守卫）。

### 1.1 启动 analyst 成员
```
更新 findings.md 进度表：Phase 1 → in_progress（负责人 analyst）

Agent(
  name: "analyst",
  model: opus,
  prompt: "你是开发团队的需求分析师。执行 /analyst 技能。需求输入：{去除 --from 参数后的需求文本}。完成后将 task_plan.md 写入 .claude/workspace/，然后通知 team lead。"
)
```

### 1.2 等待 analyst 完成

`AWAIT(analyst, 判据: task_plan.md 存在且含验收标准章节)`。达标后：
- 读取 `.claude/workspace/task_plan.md`
- 检查是否有 [待确认] 项
  - 有 → **暂停**，展示给用户确认；用户答复后 SendMessage 给 analyst（仍存活）按答复更新 task_plan.md，确认更新完成后再执行 1.3 关闭
  - 无 → 继续

### 1.3 关闭 analyst
```
SendMessage(to: "analyst", message: {"type": "shutdown_request"})
```
更新 findings.md 进度表：Phase 1 → completed

---

## Phase 2: 技术设计

> **起首守卫**：IF `START_PHASE in [dev, tester, reviewer]` → 跳过整个 Phase 2（不启动 tech-lead，直接进入 Phase 3 的起首守卫）。tech-lead 在 Phase 3 dev 偏离时仍可"按需首次启动"进入纠偏模式，见 §6.1 表。

### 2.1 启动 tech-lead 成员
```
更新 findings.md 进度表：Phase 2 → in_progress（负责人 tech-lead）

Agent(
  name: "tech-lead",
  model: opus,
  prompt: "你是开发团队的技术负责人。执行 /tech-lead 技能（方案设计模式）。读取 .claude/workspace/task_plan.md 进行技术方案设计。完成后通知 team lead。"
)
```

### 2.2 等待 tech-lead 完成方案

`AWAIT(tech-lead, 判据: architecture.md 存在且含复杂度评分与方案结论)`。达标后：
- 读取 `.claude/workspace/architecture.md`
- **IF 多方案对比（标准模式，tech-lead 复杂度评分 > 6）**：展示方案对比摘要给用户 → 用户选择方案 → SendMessage 给 tech-lead（仍存活），由其将 architecture.md 更新为选定方案的详细设计，确认更新完成后继续
- **IF 单方案（轻量模式，tech-lead 复杂度评分 ≤ 6）**：展示方案摘要 + 免对比理由 + 开放问题，用户确认即继续（无 A/B 选择环节）；用户有异议 → SendMessage 给 tech-lead 按反馈调整，确认后继续

### 2.3 关闭 tech-lead
方案确认并更新完成后立即关闭（Phase 3 若需纠偏，按需重启新实例进入纠偏模式，即用即关）：
```
SendMessage(to: "tech-lead", message: {"type": "shutdown_request"})
```

更新 findings.md 进度表：Phase 2 → completed

---

## Phase 3: 编码实现

> **起首守卫**：IF `START_PHASE in [tester, reviewer]` → 跳过整个 Phase 3（不启动 dev，不读写 progress.md，直接进入 Phase 4 的起首守卫）。dev 在 Phase 4/5 闭环 BLOCK 时仍可"按需首次启动"进入清单驱动模式，见 §6.1 表与"dev 启动 prompt 模板"节。

### 3.1 启动 dev 成员
```
更新 findings.md 进度表：Phase 3 → in_progress（负责人 dev）

Agent(
  name: "dev",
  model: sonnet,
  prompt: {模式 A — 编码模式 prompt}  # 见"dev 启动 prompt 模板"节
)
```

### 3.2 监控 dev 进度

**偏离检测：**
dev 报告模块完成时，检查 findings.md。**偏离判定标准（命中任一即视为偏离）**：
1. dev 改动的文件不在 architecture.md 模块影响分析清单内
2. 实现的接口签名/表结构与详细设计不一致
3. dev 报告中自标「需要 tech-lead 复核」

仅是方案内的盲区加固且 dev 未请求复核 → 记录 findings.md，不触发纠偏。

- 发现偏离 → 按需启动 tech-lead 纠偏（tech-lead 不跨阶段存活，每次纠偏都是新实例；前置：若本任务已有过 tech-lead 实例，确认其已 shutdown_approved）；纠偏指令给出后立即发 shutdown_request：
  ```
  Agent(
    name: "tech-lead",
    model: opus,
    prompt: "你是开发团队的技术负责人。执行 /tech-lead 技能（纠偏模式）。dev 在 {模块} 偏离了技术方案，偏离详情：{内容}。读取 .claude/workspace/architecture.md 给出修正指令并追加到 findings.md，完成后通知 team lead。"
  )
  ```
- `AWAIT(tech-lead, 判据: findings.md 新增了本次纠偏的修正指令条目)` → 转发给 dev：
  ```
  SendMessage(
    to: "dev",
    message: "tech-lead 修正指令：{内容}。请按修正继续。"
  )
  ```

**环境前置未就绪处理：**
dev 开工前核对 architecture.md「环境前置清单」后报告存在未就绪的人工前置项（Nacos 配置、中间件资源等）：
- 将未就绪条目**连同其可直接复制执行的内容块**（完整 SQL / Nacos 配置全文 / 完整命令，来自 architecture.md 清单明细或 dev 上报消息）原样展示给用户，等待人工初始化——不要转述成摘要
- 用户确认就绪后 → SendMessage 通知 dev 继续

**ralph-loop 超限处理：**
dev 报告编译 3 轮未通过：
- 按需启动 tech-lead 分析（同 3.2 纠偏启动方式，完成后立即 shutdown）→ tech-lead 给出方案 → 转发 dev
- tech-lead 也无法解决 → **暂停**，请求人工介入

### 3.3 dev 完成编码

`AWAIT(dev, 判据: progress.md 存在且各模块标记完成 AND `git log {BASE_BRANCH}..HEAD` 有 dev 的新提交)`。

**dev 的判据以 git 提交为主**——产物是代码而非报告文件，仅凭 progress.md 不足以确认落盘。达标后核对：编码完成 + /simplify 与 code-simplifier 已执行 + 编译通过 + 改动已提交（feature 分支内，排除 .claude/workspace）；其中「编译通过 / 优化已执行」无法从 git 观测，以 dev 的报告为准，如报告缺失则 SendMessage 追问后再进入 3.4。

### 3.4 关闭 dev
编码提交确认后立即关闭（Phase 4/5 闭环若 BLOCK，按需重启新实例进入清单驱动模式，即用即关）：
```
SendMessage(to: "dev", message: {"type": "shutdown_request"})
```

更新 findings.md 进度表：Phase 3 → completed

---

## Phase 4: 测试验证（不可因质量原因省略；--from=reviewer 起步按起首守卫跳过）

> **起首守卫**：IF `START_PHASE = reviewer` → 跳过整个 Phase 4（reviewer 起步直接进入 Phase 5）。tester 在 Phase 5 reviewer BLOCK 后仍可"按需首次启动"进入回归测试，见 §6.1 表。

```
当前测试轮次 = 1
最大轮次 = 3
```

### 测试-修复闭环

**WHILE 当前测试轮次 ≤ 3:**

> 测试期间团队内没有存活的可改码成员（dev 已于 3.4 关闭或从未启动），代码视图漂移的主要来源只剩用户手工改动/外部进程。tester 的基准 diff 纪律仍然照常执行，作为兜底。

#### 4.1 启动 tester 成员

**前置**：如非首轮，确认前一轮 tester 已收到 shutdown_approved（避免同名冲突，见纪律 9）。

```
Agent(
  name: "tester",
  model: sonnet,
  prompt: "你是开发团队的测试工程师。执行 /tester 技能{当前轮次 > 1 ? '（回归测试模式）' : ''}。读取 workspace 中的 task_plan.md、architecture.md 进行测试；BASE_BRANCH={BASE_BRANCH}。代码视图一致性与文件:行号引用按 /tester 技能 Step 4/5 与纪律 8 执行。完成后通知 team lead。"
)
```

#### 4.2 等待 tester 完成

`AWAIT(tester, 判据: test_report.md 存在且含明确的 ✅/❌ 最终结论 + 「结论前复核」行)`。达标后读取 `.claude/workspace/test_report.md`。

> 「有结论」是硬判据：tester 若中途停止会留下无结论的半截报告，此时按 `AWAIT` 第 3 步要求补齐，**不得**把半截报告当作 ❌ 处理——那会误召 dev 修复不存在的问题。

**IF 收到 tester 的「代码视图漂移」上报（测试期间代码被改动）**：
1. 核实改动来源（此时团队内无存活的可改码成员——排查是否用户手工改动或外部进程所为），结论记入 findings.md
2. 漂移涉及被测模块 → 本轮作废，从头重测（轮次不变，重新 4.1）
3. 仅涉及无关文件 → 指示 tester 增量回归后继续

#### 4.3 关闭 tester
无论结果如何，当轮 tester 完成后立即关闭：
```
SendMessage(to: "tester", message: {"type": "shutdown_request"})
```

#### 4.4 判定

**IF 测试结论 = ✅ 通过：**
  → 更新 findings.md 进度表：Phase 4 → completed
  → BREAK，进入 Phase 5

**IF 测试结论 = ❌ 未通过：**

  **IF 当前测试轮次 = 3（已是最后一轮）→ 不再召回 dev，直接转「超限暂停」分支。**

  启动 dev 修复（统一路径：dev 不跨阶段存活，每次修复都是模式 B 新实例；前置：若本任务已有过 dev 实例，确认其已 shutdown_approved）：
  ```
  Agent(
    name: "dev",
    model: sonnet,
    prompt: {模式 B — 清单驱动模式 prompt，报告指向 test_report.md}  # 见"dev 启动 prompt 模板"节
  )
  ```

  → `AWAIT(dev, 判据: `git log {BASE_BRANCH}..HEAD` 出现本轮新提交 AND progress.md 记录了本轮修复项)`
  → 立即关闭 dev（等待 shutdown_approved）：
  ```
  SendMessage(to: "dev", message: {"type": "shutdown_request"})
  ```
  → 当前测试轮次 += 1
  → IF 当前测试轮次 ≤ 3 → 回到 4.1 启动新的 tester 实例（前置：确认前一轮 tester 已 shutdown_approved）；ELSE → 转「超限暂停」

**超限暂停（测试轮次耗尽）：**
  → **暂停**，展示 Bug 变化趋势和 findings.md，请求人工介入。请选择：
    1) 追加 1 轮（轮次上限 +1 后继续，含召回 dev 修复）
    2) 接受当前状态进入下一 Phase（风险自担，记录 findings.md）
    3) 终止流程（走异常终止路径，见终止条件表）
  暂停期间无存活成员待命（人工选追加轮次时 dev 按需重启）。人工选择后的任务状态归属：选 1 → Phase4 保持 in_progress 继续循环；选 2 → 更新 findings.md 进度表：Phase 4 → completed 并在 findings.md 记「测试超限接受，风险自担」后进入 Phase 5；选 3 → 保持 in_progress，走异常终止路径。

---

## Phase 5: 代码审查（不可跳过）

> **起首守卫**：无（reviewer 是 --from 支持的最后一个起始 phase）。本 phase 内的 dev 召回一律按需新实例（模式 B，见 5.4.1），tester 回归按 5.4.2 启动。

```
当前审查轮次 = 1
最大轮次 = 3
每轮回归重试上限 = 1（见 5.4.2，防回归子循环无界）
```

### 5.0 数据变更探测（决定是否启用 data-expert，每轮进入 5.1 前执行）

判断本次变更是否涉及数据模型，命中任一即 `DATA_CHANGE = true`：
- `architecture.md` 含「数据模型变更」非空内容（建表/改表/字段变更/索引/分库分表）
- `git diff {BASE_BRANCH}...HEAD`（或 working tree）命中以下任一：迁移脚本（Flyway/Liquibase/`.sql`）、Entity/表结构定义、Mapper XML / `@Query` 改动、索引或分片路由配置

```bash
# 参考探测（按项目实际路径调整）：命中即视为数据变更
git diff {BASE_BRANCH}...HEAD --name-only | grep -E '(migration|flyway|liquibase|\.sql$|/entity/|Mapper\.xml$|Entity\.java$)' 
```

- `DATA_CHANGE = true` → 从命中轮起每轮启动 data-expert（与 reviewer 并行启动，见 5.1）
- `DATA_CHANGE = false` → 本轮不启动 data-expert，5.1–5.4 仅按 reviewer 单角色走
- **逐轮升级规则**：探测只允许 false→true 升级——后续轮命中（如 dev 修复引入迁移/Entity 改动）则从该轮起启用 data-expert；一旦为 true 则后续轮保持 true，不做降级

> 每轮探测结论记入 findings.md 一行（`DATA_CHANGE=true/false + 命中依据`），便于回溯。

### 审查-修复闭环

**WHILE 当前审查轮次 ≤ 3:**

#### 5.1 启动 reviewer 成员（DATA_CHANGE=true 时与 data-expert 并行启动）

启动 reviewer：
```
Agent(
  name: "reviewer",
  model: sonnet,
  prompt: "你是开发团队的代码审核员。执行 /reviewer 技能{当前轮次 > 1 ? '（第 {N} 轮复审）' : ''}。审查当前代码变更（纯只读，不修改任何代码）。完成后通知 team lead。"
)
```

**IF `DATA_CHANGE = true`（5.0 探测命中）→ 同时并行启动 data-expert**（reviewer 与 data-expert 均为纯只读审查，无视图漂移风险，互不阻塞）：
```
Agent(
  name: "data-expert",
  model: sonnet,
  prompt: "你是开发团队的数据治理审查员。执行 /data-expert 技能{当前轮次 > 1 ? '（第 {N} 轮复审）' : ''}。审查本次变更的数据层部分（迁移/表结构/索引/Mapper/分片），产出 data_review.md。完成后通知 team lead。"
)
```
> 前置（非首轮）：确认前一轮 data-expert 已 shutdown_approved（避免同名冲突，见纪律 9）。

#### 5.2 等待审查完成

两者并行运行，**分别 AWAIT、先到先读**，不要串行空等：

- `AWAIT(reviewer, 判据: review_report.md 存在且含 BLOCK/APPROVE 明确结论)` → 读取 `.claude/workspace/review_report.md`
- IF `DATA_CHANGE = true`：`AWAIT(data-expert, 判据: data_review.md 存在且含 BLOCK/APPROVE 明确结论)` → 读取 `.claude/workspace/data_review.md`

两份都达标后才进入 5.3；任一未达标时按 `AWAIT` 第 3 步处理该成员，**不影响**另一成员的已达标结论。

#### 5.3 关闭本轮审查成员
无论结果如何，当轮成员完成后立即关闭（各自等待 shutdown_approved）：
```
SendMessage(to: "reviewer", message: {"type": "shutdown_request"})
```
IF `DATA_CHANGE = true`：
```
SendMessage(to: "data-expert", message: {"type": "shutdown_request"})
```

#### 5.4 判定（reviewer 与 data-expert 结论合并）

> 合并规则：**两者任一 BLOCK → 本轮 BLOCK**；两者都 APPROVE（或 data-expert 未启用且 reviewer APPROVE）→ APPROVE。dev 修复时合并两份报告的问题清单一次性处理。

**IF 结论 = APPROVE：**
  → **IF review_report.md 含「遗留必修项」**（坏味道严重档单独出现时的 APPROVE with mandatory-fix，见 reviewer 判定标准）→ **暂停询问用户**：
    1) 立即处理：按 5.4.1 启动 dev 修复遗留项 + 5.4.2 tester 回归（不占审查轮次、不再起 reviewer 复审——遗留项本就是 APPROVE 档；回归未通过时按 5.4.2 重试上限处理）。**回归通过后必须按 findings.md 条目格式记「遗留必修项已修复并回归通过（HI-xxx 清单）」**——6.0/6.2 引用遗留必修项时以此记录为准
    2) 接受遗留：转后续任务处理，记入 findings.md，Phase 6.2 汇总报告列出
  → 无需关闭 dev（dev 不跨阶段存活，此时无存活实例）
  → 更新 findings.md 进度表：Phase 5 → completed
  → BREAK，进入 Phase 6

**IF 结论 = BLOCK：**

  **IF 当前审查轮次 = 3（已是最后一轮）→ 不再召回 dev，直接转「超限暂停」分支。**

  ### 5.4.1 召回 dev 修复

  启动 dev 修复（统一路径：dev 不跨阶段存活，每次修复都是模式 B 新实例；前置：若本任务已有过 dev 实例，确认其已 shutdown_approved）：
  ```
  Agent(
    name: "dev",
    model: sonnet,
    prompt: {模式 B — 清单驱动模式 prompt，报告指向 review_report.md（本轮启用 data-expert 时一并指向 data_review.md）}  # 见"dev 启动 prompt 模板"节
  )
  ```

  → `AWAIT(dev, 判据: `git log {BASE_BRANCH}..HEAD` 出现本轮新提交 AND progress.md 记录了本轮修复项（含 data_review.md 清单项，若本轮启用）)`
  → 立即关闭 dev（等待 shutdown_approved）：
  ```
  SendMessage(to: "dev", message: {"type": "shutdown_request"})
  ```

  ### 5.4.2 修复后必须经过 tester 回归（防止修复引入新 bug）

  **分支 1：tester 本任务中已运行过（典型场景：`START_PHASE != reviewer`，或本 Phase 已有过回归轮）：**
  - 前置：确认其最后一轮已 shutdown_approved

  **分支 2：tester 本任务中从未启动（典型场景：`START_PHASE = reviewer` 首次回归）：**
  - 无需等待历史 shutdown_approved（不存在前一轮）

  两个分支共用启动命令：
  ```
  Agent(
    name: "tester",
    model: sonnet,
    prompt: "你是开发团队的测试工程师。执行 /tester 技能（回归测试模式）。仅回归 reviewer（及 data-expert，若本轮启用）要求修复的变更及关联影响；BASE_BRANCH={BASE_BRANCH}。代码视图一致性与文件:行号引用按 /tester 技能 Step 4/5 与纪律 8 执行。完成后通知 team lead。"
  )
  ```
  → `AWAIT(tester, 判据: test_report.md 已更新为本轮回归结论且含明确 ✅/❌)`，读取回归结论 → 关闭本轮 tester（等待 shutdown_approved 后再继续）：
  ```
  SendMessage(to: "tester", message: {"type": "shutdown_request"})
  ```
  → **回归结论判定**（每个审查轮次进入本步时 `回归重试计数 = 0`）：
    - **IF ❌ 未通过**（修复引入新问题）→ `回归重试计数 += 1`；IF 回归重试计数 > 1（同一审查轮内第二次回归仍未通过）→ 转「超限暂停」；ELSE 再次按需启动 dev（模式 B 新实例，仅修复回归报告新增问题，完成即关）→ 重新执行本步（重启 tester 回归）
    - **IF ✅ 通过** → 继续

  → 当前审查轮次 += 1
  → IF 当前审查轮次 ≤ 3 → 回到 5.1 启动新的 reviewer 实例（前置：确认前一轮 reviewer 已 shutdown_approved）；ELSE → 转「超限暂停」

**超限暂停（审查轮次耗尽）：**
  → **暂停**，展示 review_report.md 和 findings.md，请求人工介入。请选择：
    1) 追加 1 轮（轮次上限 +1 后继续，含召回 dev 修复）
    2) 接受当前状态进入下一 Phase（风险自担，记录 findings.md）
    3) 终止流程（走异常终止路径，见终止条件表）
  暂停期间无存活成员待命（人工选追加轮次时 dev 按需重启）。人工选择后的任务状态归属：选 1 → Phase5 保持 in_progress 继续循环；选 2 → 更新 findings.md 进度表：Phase 5 → completed 并在 findings.md 记「审查超限接受，风险自担」后进入 Phase 6；选 3 → 保持 in_progress，走异常终止路径。

---

## Phase 6: 收尾归档

### 6.0 上线就绪核对（归档前执行，产出上线工单）

> 本步是**汇总核对**——消费既有产物、逐项核对卡点，不做新的代码审查（不违反纪律 6"不替代角色执行"）。核对**不拦截流程**：所有结论进工单交用户决策。

**1. 收集证据**（按存在性读取，缺失的在工单「证据」列标注「无产物」）：
- `data_review.md` → 迁移回滚 / 在线 DDL / 数据一致性结论
- `review_report.md` → 安全章节、遗留必修项、可观测性落地核对（遗留必修项须与 findings.md 的「遗留必修项已修复并回归通过」记录交叉核对：已处理的在工单标注「已在 Phase 5 处理」，不再列为遗留风险）
- `test_report.md` → 测试结论、覆盖统计（含「集成测试缺失基建」标注）
- `architecture.md` → 开放问题、灰度与回滚预案、数据模型变更
- `findings.md` → 「测试/审查超限接受」等风险记录

**2. 确定卡点来源**（按优先级取第一个可用项，工单「卡点来源」注明用了哪级）：
1. release-checklist 技能可用（出现在可用技能列表且 Skill 工具可实际调用；调用被拦截即视为不可用，落到下一级）→ 执行 `/release-checklist`，把第 1 步证据作为输入，由其读取项目根 `release-checklist.yaml` 逐卡点核对并生成上线工单
2. 技能不可用但项目根存在 `release-checklist.yaml` → team 按该文件的卡点逐项核对
3. 两者皆无 → 按内置通用卡点清单核对：
   - **数据库**：迁移有回滚方案、上线执行顺序明确（先 DDL 后代码或兼容发布）
   - **配置**：新增配置项各环境值就位；功能/灰度开关及默认值
   - **依赖**：上下游 RPC 接口版本兼容；MQ topic/消费组就绪
   - **可观测**：关键路径日志/监控/告警已落地
   - **预案**：灰度计划、回滚步骤（代码 + 数据）、值班安排
   - **遗留风险**：遗留必修项、超限接受记录、architecture 开放问题

**3. 三档结论**（每个卡点必须落一档；✅ 必须附证据出处）：
- ✅ 已核实（证据：{文件:行号 或 章节}）
- ⚠️ 需人工确认（agent 无法核实的事项：运维审批、配置中心实值、外部依赖方通知等）
- ❌ 未通过（有证据表明缺失，注明缺口）

**4. 产出 `.claude/workspace/release_checklist.md`**（走来源 1 时把 /release-checklist 的工单保存/复制到该路径，保证归档路径一致）：
```markdown
# 上线清单 - {需求名称}

## 基本信息
| 分支 | 日期 | 涉及服务/模块 | 卡点来源 |
|------|------|--------------|---------|
| {分支} | {YYYY-MM-DD} | {模块清单} | /release-checklist / release-checklist.yaml / 内置通用清单 |

## 卡点核对
| 卡点 | 状态 | 证据 / 待办 |
|------|------|-------------|
| {卡点} | ✅/⚠️/❌ | {文件:行号 或 待办事项} |

## 建议上线步骤
1. {按依赖顺序：迁移 → 配置 → 部署 → 开关}

## 回滚预案
- 代码回滚：{方式}
- 数据回滚：{引用 data_review.md 回滚结论，无数据变更则写"不涉及"}

## 遗留风险
- {遗留必修项 / 超限接受 / 开放问题，逐条列出}
```

### 6.1 验证所有成员已正常关闭

按**本任务中实际启动过的成员实例**清单逐一确认收到 `shutdown_approved`（系统通知 "X has shut down"）。受 START_PHASE 与按需启动影响，未启动过的角色直接跳过，不计孤儿候选。全量流程下为：
- analyst / tech-lead（含 Phase 3 按需纠偏实例）/ dev（含 Phase 4/5 按需修复实例）/ tester（最后一轮）/ reviewer（最后一轮）
- IF `DATA_CHANGE = true`：data-expert（最后一轮）也必须确认已 shutdown_approved

如有成员仅 idle 但未发回 shutdown_approved（例如成员在 shutdown_request 到达前已 idle 完最后一轮工作）→ 标记为**孤儿候选**，在 Phase 6.5 逐个 `TaskStop` 回收并报告给用户。

### 6.2 汇总报告
```markdown
## 任务完成报告

### 需求: {需求名称}
### 分支: {git rev-parse --abbrev-ref HEAD 的实际输出}

### 各阶段完成情况
{从 findings.md 进度表提取各阶段状态}

### 质量数据
- 测试轮次: {N} 轮
- 审查轮次: {N} 轮
- 发现 Bug 数: {N}，已修复: {N}
- 审查问题数: {N}，已修复: {N}
- 遗留必修项: {N} 项（用户已选「接受遗留」，清单见 review_report.md「遗留必修项」）/ 已在 Phase 5 立即处理（回归通过，见 findings.md 记录）/ 无

### 产出文件
- task_plan.md / architecture.md / test_report.md / review_report.md{DATA_CHANGE ? ' / data_review.md' : ''} / release_checklist.md

### 上线就绪
- 卡点核对: ✅ {N} / ⚠️ 需人工确认 {N} / ❌ 未通过 {N}（工单见 release_checklist.md）
```

### 6.3 条件触发 CLAUDE.md 更新
检查 findings.md 是否包含新发现的项目模式/约定/陷阱：
- **有** → 提示用户更新项目 CLAUDE.md（装有 claude-md-management 插件时可用 /revise-claude-md）
- **无** → 跳过

### 6.5 孤儿成员回收（必须执行）

teammate 是 **in-process** 运行的——它跑在 lead 自己的 `claude` 进程内，**不是独立操作系统进程**。因此：

- ❌ `ps aux | grep agent-name` 恒为空，**不能**用来判断有无孤儿（进程表里根本没有这个字符串）
- ❌ `kill <PID>` 无对象可杀，且会误伤 lead 自身
- ✅ 唯一可靠手段是按 name 逐个 `TaskStop`

对 6.1 清单中**每一个已启动但未收到 `shutdown_approved`** 的成员执行：

```
TaskStop(task_id: "{角色name}")
```

处理规则：
1. 逐个回收，记录每次返回的成功/失败
2. 返回失败或提示 task 不存在 → 说明该成员实际已停止，视为已回收，不必再处理
3. 把「成员 name + 触发场景（哪个 Phase、为何未正常 shutdown）+ 回收结果」记入 `.claude/workspace/findings.md`，作为流程改进输入（归档在 6.6 才执行，此时文件仍在原路径）
4. 全部成员均已 `shutdown_approved`、无需回收时：仅输出一行确认（"全部成员已正常关闭，无需回收"）

> 用户侧交叉验证入口：`/tasks` 面板列出全部在跑的 agent，在面板中选中并按 `x` 亦可停止。若 6.5 执行后 `/tasks` 里仍能看到本次任务的成员，属异常，需报告用户。

### 6.6 归档（放在回收核对之后，保证 6.3/6.5 读写的 findings.md 仍在原路径）
```bash
# --from 短路径下部分产物可能不存在，缺失即跳过
mkdir -p .claude/workspace/archive/{YYYYMMDD}_{需求简称}
mv .claude/workspace/task_plan.md .claude/workspace/archive/{目录}/ 2>/dev/null || true
mv .claude/workspace/architecture.md .claude/workspace/archive/{目录}/ 2>/dev/null || true
mv .claude/workspace/progress.md .claude/workspace/archive/{目录}/ 2>/dev/null || true
mv .claude/workspace/test_report.md .claude/workspace/archive/{目录}/ 2>/dev/null || true
mv .claude/workspace/review_report.md .claude/workspace/archive/{目录}/ 2>/dev/null || true
mv .claude/workspace/findings.md .claude/workspace/archive/{目录}/ 2>/dev/null || true
# IF DATA_CHANGE=true（data_review.md 存在时）：
mv .claude/workspace/data_review.md .claude/workspace/archive/{目录}/ 2>/dev/null || true
mv .claude/workspace/release_checklist.md .claude/workspace/archive/{目录}/ 2>/dev/null || true
```

### 6.7 最终提示
- feature 分支已就绪
- 上线工单：`archive/{目录}/release_checklist.md`；**存在 ❌ 未通过或 ⚠️ 需人工确认卡点时在此逐条列出**，提醒用户上线前处理
- 是否需要合并到目标分支或创建 PR

更新 findings.md 进度表：Phase 6 → completed

---

# `/dev` 与本 skill 的关系（重要澄清）

`/dev` 触发的是本插件的独立 dev skill（`skills/dev/SKILL.md`，单 dev 编码 + /simplify + code-simplifier + 编译闭环），不创建团队、不含 tester/reviewer 闭环，**不等同于"精简版的 /team"**。

## 如何选择入口

| 你想要的效果 | 正确入口 |
|---|---|
| 完整流程：需求分析 + 架构设计 + 编码 + 测试 + 审查 + 归档 | `/team <需求>` |
| 已有需求文档（task_plan.md）→ 从技术设计开始 | `/team --from=tech-lead [<需求补充>]` |
| 已有需求 + 架构（task_plan.md + architecture.md）→ 从编码开始，**保留** 测试/审查闭环 | `/team --from=dev [<需求补充>]` |
| 已有代码 → 跑测试+审查闭环 | `/team --from=tester [<需求补充>]` |
| 已有代码 → 只做一次代码审查（含 P0 安全审查） | `/team --from=reviewer [<需求补充>]` |
| 单 dev 直接编码（自行写简版 architecture.md + 自建 feature 分支 + 完成后 /simplify + code-simplifier）**不含**测试/审查闭环 | `/dev <需求>` |

> `/team --from=dev` ≠ `/dev`：前者**含** Phase 4/5 闭环，后者**不含**。

---

# 终止条件与升级机制

各 Phase 内文与纪律 3 已定义的暂停场景（待确认项、方案确认、纠偏、闭环超限、前置校验失败等）从其规定。此外：

| 场景 | 处理方式 |
|------|---------|
| 任何阶段遇到不可解决的问题 | 暂停，展示 findings.md，请求人工介入 |
| **异常终止/人工中断** | 关闭所有存活成员（等待 shutdown_approved，超时按孤儿候选 TaskStop 强制回收）→ **孤儿成员回收核对（同 6.5，不可省略）** → 报告 workspace 当前状态（已生成产物清单 + 可用的 --from 复活入口）。团队目录随会话结束自动清理，无需手动删除 |

# 纪律

1. **tester 和 reviewer 阶段不可因质量原因省略** — 唯一例外是 `START_PHASE=reviewer` 按 Phase 4 起首守卫跳过 Phase 4（且 reviewer BLOCK 后修复仍必须经 tester 回归，见 5.4.2）
2. **reviewer BLOCK 后修复必须经过 tester 回归** — 防止修复引入新 bug
3. **闭环超限必须暂停** — 不允许无限循环消耗 token
4. **成员完成任务后必须 shutdown 并等待 shutdown_approved** — 协议细节见「关闭成员」节
5. **流程结束必须执行孤儿成员回收（6.5）** — 团队目录自动清理不代表成员已停止；teammate 是 in-process 的，只能用 `TaskStop(task_id: name)` 回收，`ps`/`kill` 一律无效
6. **不替代角色执行** — 编排指挥官只调度，不直接编码/测试/审查
7. **纠偏与修复一律按需新实例、完成即关** — tech-lead 纠偏、dev 修复每次都是 Agent() 新实例（纠偏模式/清单驱动模式），任务完成立即 shutdown；不留跨阶段待命成员，测试/审查期间团队内没有任何可改码的存活成员。例外：analyst（1.2）/ tech-lead（2.2）在等待用户确认并按答复更新自身产物期间的**同阶段**短暂存活，不属于跨阶段待命
8. **tester 提交报告前必须 verify 代码视图未变** — 通过 git diff 与测试基准对比；如发现测试期间代码被改动（用户手工改动/外部进程），立即暂停而非继续写结论
9. **启动同名新一轮成员前必须等前一轮 shutdown_approved** — 否则同名冲突（latest-wins 抢占或自动改名，随版本而异）导致成员定位混乱、旧实例失管（见「启动成员」节；孤儿候选场景除外，按「关闭成员」节第 5 条两种行为分别处理）
10. **--from=X 等于声明 X 之前的角色不主动启动** — 详见 §6.1 表（dev → 清单驱动模式；tech-lead → 纠偏模式）；禁止"--from=X 就完全禁用 X 之前的角色"的简化理解，会破坏闭环可达性
11. **--from in [dev, tester, reviewer] 时沿用当前分支** — 不新建 feature 分支；启动时打印当前分支供用户确认
12. **--from=X 的前置校验失败必须立即报错暂停** — 报错模板见 Phase 0.1，禁止静默回退到上游 phase 补生成
13. **Phase 6 归档/孤儿检查无论起步 phase 如何都必须执行** — 包括 --from=reviewer 一次 APPROVE 的最短路径
14. **data-expert 条件触发、与 reviewer 同档阻断** — 逐轮探测见 Phase 5.0，与 reviewer 并行启动见 5.1，结论合并规则（任一 BLOCK 即本轮 BLOCK）见 5.4
15. **团队隐式存在，进度走 findings.md 进度表** — 不创建/删除团队、不传 `team_name`、不用 Task 系列工具（见 0.4 / 0.5）
16. **阶段完成判定一律走 `AWAIT(成员, 判据)` 协议** — teammate 的 idle 通知不携带产出内容，判据是**产物落盘 + 有结论性章节**，idle 通知仅为触发信号；禁止编写「等成员把结果回传」的等待逻辑（subagent 语义），否则永久阻塞。成员已 idle 但产物不达标时按协议第 3 步补齐，连续 2 次不达标即暂停请求人工；单成员 15 分钟无进展触发超时兜底。协议全文见「阶段完成判定协议」节
17. **上线就绪核对（6.0）只出工单不拦截** — ✅ 必须附证据出处（文件:行号/章节），❌/⚠️ 卡点在 6.7 逐条列出交用户决策；不触发修复闭环，上线决策属于人
