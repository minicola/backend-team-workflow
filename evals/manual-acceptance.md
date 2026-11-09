# 人工验收场景（需真实团队运行）

以下场景 A-E 覆盖 2026-10-21 流程审计（WF-01 至 WF-06）修正的控制流，F 覆盖 v0.8.0 归档治理（其中 tidy 部分不经 team 门禁，可直接在临时仓库验收）。A-E 都要跑到 Phase 0.2 之后，依赖交互式会话与 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`，`claude plugin eval` 沙箱（headless）会先撞上 0.1b 门禁，因此只能在真实项目上人工验收。每个场景给出前置、操作与判定；判定只认文件与 git 状态，不认 lead 的口头结论。

验证分三类，报告时区分开，**不要把静态检查或模拟验证表述成完整团队运行通过**：

- **文件 / git 操作**：临时目录、临时仓库即可复现（标 ⓕ）
- **模型遵循指令**：需要真实 lead / 成员运行，按判定标准人工核对（标 ⓜ）
- **宿主行为**：通信、唤醒、关闭、TaskStop 返回值，只能真实运行观察（标 ⓗ）

| 场景 | 覆盖 | 类型 |
|---|---|---|
| A 空工作区初始化 | WF-01 | ⓕⓜ |
| B 旧报告残留 | WF-04 | ⓜ |
| C 未提交代码漂移 | WF-02 / WF-03 | ⓕⓜ |
| D 追加轮次 | WF-06 | ⓜ |
| E 回收调用失败 | WF-05 | ⓗ |
| F 归档治理 | v0.8.0 tidy / 6.6 / 0.2 | ⓕⓜ |

## A 空工作区初始化（team 0.2）

前置：目标项目在 BASE_BRANCH 上、工作树干净、`.claude/workspace/` 不存在或为空目录。

操作：`/backend-team-workflow:team <任意需求文本>`。

判定：
1. lead **不输出**「检测到 workspace 中已有以下下游产物」的三选一提示
2. 进入 Phase 1 前 `findings.md` 存在，首行为 `# Findings`，格式头之后紧接 `## [team][Phase 0] 进度跟踪` 条目，含进度表、运行状态块（`当前测试轮次 / 测试轮次上限 | 0 / 3`、`tester 启动次数 / data-expert 启动次数 | 0 / 0`）与成员实例台账
3. 反向用例：预置一份旧 `review_report.md` 再启动，必须出现三选一提示；选 3 后 workspace 内容与启动前逐字节一致（`diff -r`），且未创建团队成员；选 1 后旧文件进入 `archive/*_orphaned*/`（目录名带旧需求后缀，见场景 F 第 5 条），新 `findings.md` 格式头与进度区完整

## B 旧报告残留（AWAIT 本轮匹配）

前置：任一能进入 Phase 4 第 2 轮或 Phase 5 第 2 轮的任务（首轮 tester ❌ 或 reviewer BLOCK）。

操作：观察 lead 在启动第 2 轮 tester / reviewer 之后、成员写出新报告之前的行为（可在 dev 修复提交后、lead 启动新成员时用 `/tasks` 面板与 workspace 文件 mtime 对照）。

判定：
1. 台账中第 2 轮启动行含匹配期望值（tester「启动序号 {N}，基准 commit {hash}」，reviewer「期望轮次 {N}，基准 commit {hash}」），hash 等于启动时 `git rev-parse --short HEAD`
2. 上一轮 `test_report.md` / `review_report.md` 仍在原路径且带完整结论，lead **不据其**召回 dev、进入下一 Phase 或关闭成员
3. 新报告落盘后 tester「测试基准」启动序号 = 台账值、HEAD = 台账 hash；reviewer「审查轮次」= 当前审查轮次、「审查基准 commit」= 台账 hash，lead 才推进
4. 故意让成员写错匹配字段（人工把启动序号 / 审查轮次改为上一轮值）时，lead 按 AWAIT 第 3 步要求补齐，而不是推进
5. data-expert 启用时其匹配键是「data-expert 启动次数」，与 reviewer 的审查轮次无关
6. tester「测试轮次」保持闭环语义：Phase 4 第 2 轮报告写「第 2 轮」；漂移作废重测后的报告轮次与作废轮相同、启动序号 +1；Phase 5 回归报告轮次 = 上一份报告 +1

## C 未提交代码漂移（tester 快照 + dev 最终编译）

前置：任务已进入 Phase 4，tester 正在执行测试（Step 4 之后、Step 5 之前）。

操作 ⓕ：在业务路径（非 `src/test`）手工修改一个已跟踪文件但不提交；另在 `src/test` 下新增一个文件。

判定：
1. `test_report.md`「测试基准」含 HEAD、基准 diff、未提交改动三行；结论前复核发现业务文件差异，tester SendMessage 上报 team-lead 并**不写最终结论**；lead 按 4.2 处理（涉及被测模块即本轮作废重测）
2. 仅 `src/test` 下的改动不触发漂移上报
3. 反向用例 ⓕ（独立复现快照命令）：临时仓库中对业务文件做未提交修改 + 新增未跟踪业务文件 + 测试路径改动 + workspace 文件，`git status --porcelain -- . ':(exclude).claude/workspace' ':(exclude,glob)**/src/test/**'` 只列出两个业务文件
4. dev 侧：在 Phase 3 让 /simplify 或 code-simplifier 引入编译错误（或人工在其输出后注入），dev 不得报告完成；`progress.md` 记录的提交 hash 等于 `git rev-parse --short HEAD`，提交后 `git status --porcelain`（排除 workspace）为空；Step 4.5 修复后有追加提交

## D 追加轮次（轮次上限持久化）

前置：构造连续 3 轮 tester ❌（如 task_plan.md 中放一条实现无法满足的验收标准）。

判定：
1. 第 3 轮 ❌ 时 lead 不召回 dev，直接超限暂停并给出三选项
2. 选「追加 1 轮」后，运行状态块变为 `当前测试轮次 / 测试轮次上限 | 3 / 4`，findings.md 有「用户追加测试轮次，上限 → 4」记录，随后召回 dev 修复、启动第 4 轮 tester
3. 第 4 轮仍 ❌ 时再次超限暂停，而不是继续第 5 轮；未经用户选择 lead 不得再改上限
4. 在第 4 轮进行中触发上下文压缩（或让 lead 重读 findings.md）后，lead 仍按上限 4 运行
5. Phase 5 同样验证「审查轮次上限」；追加审查轮次后「每轮回归重试上限」仍为 1

## E 回收调用失败（6.5 / CONFIRM_SHUTDOWN 三分）

前置：任一完整跑到 Phase 6 的任务。

操作 ⓗ：
- 正常路径：观察 6.1 一次 ListAgents 与 6.5 输出
- 失败路径：在 6.5 前对某成员先手工 `/tasks` 停止，再让 lead 执行 TaskStop，观察「无此任务」分支；若能构造 TaskStop 超时 / 异常（如权限拒绝），观察「其他错误」分支

判定：
1. 「无此任务」→ 台账记「确认关闭」
2. 其他错误 → lead 再 ListAgents 复核；仍在列表则重试一次，仍失败记「回收未完成」
3. 6.2「成员回收」节与 6.7 最终提示列出「回收未完成」成员并提示 `/tasks` 手动停止；没有此类成员时写「全部成员已正常关闭」
4. 任何情况下 lead 不把一般工具错误写成「已回收」

## F 归档治理（tidy / team 6.6 / 0.2）

前置 ⓕ：临时 git 仓库，主干上有一个已合入的 feature 提交 `B` 和一个未合入的提交 `C`。`.claude/workspace/archive/` 下预置：
- 归档时间早于 90 天、HEAD=`B` 的标准归档 `M`
- 同样早于 90 天、HEAD=`C` 的标准归档 `U`
- HEAD 写「未知」的旧归档 `K`
- `_expired/` 下「移入 _expired」早于 30 天的 `P`，以及昨天移入的 `Q`
- 无元数据的遗留目录 `L`，`archive/` 根下一个散文件，workspace 根下一个无关截图

操作：`/backend-team-workflow:tidy --auto`；之后再运行一次无参数的 `/backend-team-workflow:tidy`。

判定：
1. `--auto`：`M` 移入 `_expired/`，且其「移入 _expired」= 今天；`P` 被删除，INDEX「已清除记录」有 `P`；`U`、`K`、`Q`、`L`、散文件、截图都不动；输出行中人工候选数 ≥ 2（`U`、`K`），未整理遗留 ≥ 1（`L`）
2. INDEX.md 的在库 / 过期区 / 已清除三节与目录实际状态一致；再跑一次 `--auto`，结果不变（幂等）
3. 交互模式：先出计划表、确认后才执行；`L` 补上元数据（取不到的字段写「未知」），目录名不变；截图只出现在清单里，未经逐项指定不移动、任何情况下不删除
4. ⓜ team 6.6：归档目录包含 `attachments/`；`summary.md` 首节为「归档元数据」且 HEAD = 收尾时的 `git rev-parse --short HEAD`；6.7 转述了 tidy 的输出行
5. ⓜ team 0.2 选 1：目录名形如 `{YYYYMMDD_HHMMSS}_orphaned_{旧需求}`，`summary.md` 只含元数据，类型 `orphaned`、结论「中断」
