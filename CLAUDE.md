# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库性质

这是一个 **Claude Code Plugin** 仓库，不是普通代码库：
- 没有源代码、没有构建系统、没有测试
- 产品就是 `skills/<role>/SKILL.md` 这 7 份 markdown 文件 + `.claude-plugin/plugin.json`（其中 `data-expert` 是 Phase 5 条件触发成员）
- "开发"= 编辑 SKILL.md 中的角色指令；"发布"= 提交并让用户重新 `/plugin install` 或 `/reload-plugins` 热生效

不存在 `npm test` / `mvn compile` 之类的入口。提交前的主要"验证"是把插件装到一个真实项目上跑一遍 `/team` 或 `/dev`；此外 `evals/` 下有针对 Phase 0 参数解析/前置校验的 `claude plugin eval` 回归用例（该 CLI 功能目前 early access，未开启时仅作为文档化的验收标准，详见 `evals/README.md`）。

## 常用操作

```bash
# 装到本地（开发期，从这个目录直装）
/plugin marketplace add ~/claude-plugin/backend-team-workflow
/plugin install backend-team-workflow

# 改完 SKILL.md 后热生效（不需要重启 Claude Code）
/reload-plugins

# 启动总入口（在目标业务项目里使用）
/backend-team-workflow:team <需求描述或PRD路径>

# 任意节点切入（跳过上游 phase）
/backend-team-workflow:team --from=tech-lead [<需求补充>]   # 预置 task_plan.md，从技术设计开始
/backend-team-workflow:team --from=dev [<需求补充>]         # 预置 task_plan + architecture，从编码开始（含测试/审查闭环）
/backend-team-workflow:team --from=tester [<需求补充>]      # 已有代码，跑测试+审查闭环
/backend-team-workflow:team --from=reviewer [<需求补充>]    # 已有代码，仅做一次审查（含 P0 安全审查）
/backend-team-workflow:team --base=develop <...>            # 主干不是 main/master 时显式指定 diff 基准分支（可与 --from 组合）
```

`--from=X` 等于声明 X 之前的角色不主动启动，但若下游闭环（tester BLOCK / reviewer BLOCK）需要它们，按需首次启动进入"清单驱动模式"。完整决策矩阵见 `skills/team/SKILL.md` §6.1 表。

## 前置依赖（外部技能/插件）

工作流引用 5 个非本插件自带的技能：`ralph-loop`（插件，dev 编译迭代闭环）、`code-simplifier` 与 `simplify`（dev 代码优化，Step 4 提测/修复前执行）、`claude-md-management` 的 `/revise-claude-md`（Phase 6 更新项目 CLAUDE.md，可选）、`release-checklist`（可选，Phase 6.0 上线卡点核对；缺失时降级为读项目根 `release-checklist.yaml`，再缺则用内置通用清单）。目标环境未装齐时不会卡死：各角色按各自 SKILL.md 声明的降级路径执行（跳过并在报告标注「技能缺失未执行」/ 退化为手动编译循环）。

## 架构：6 角色 + Agent Team（含 1 个条件触发成员）

整个插件是一个**编排 + 5 固定角色 + 1 条件触发角色**的工作流，依赖 Claude Code 的 Agent Team API：`Agent(name=…)` 启动 teammate / `SendMessage` 通信与 shutdown / `TaskStop` 强制回收。

> **API 版本红线（v2.1.178 起）**：`TeamCreate` 与 `TeamDelete` **已被移除**，`Agent` 的 `team_name` 入参**已废弃且被忽略**。当前语义是「每会话唯一隐式团队」：团队名由会话 ID 派生（`session-{前8位}`）不可指定，`Agent(...)` 带 `name` 即自动成为 teammate，会话退出时团队目录自动清理。另需 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 才启用 teammate（否则退化为普通 subagent，双向通信与召回全部失效），team/SKILL.md Phase 0.1b 有门禁探测（刻意排在 0.0/0.1 参数与前置校验之后——用户输入类报错优先，且不改变 evals 三个 Phase 0 场景的期望输出）。改动流程时**不要再写回这几个已移除的工具**。

```
team (编排，opus)
 ├─ Phase 1: analyst      (opus,   仅 Phase 1)
 ├─ Phase 2: tech-lead    (opus,   Phase 2 方案确认后关闭；Phase 3 纠偏时按需重启新实例，即用即关)
 ├─ Phase 3: dev          (sonnet, Phase 3 编码提交后关闭；Phase 4/5 BLOCK 时按需重启新实例（清单驱动模式），即用即关)
 ├─ Phase 4: tester       (sonnet, 每轮重启 — 不跨轮复用)
 └─ Phase 5: reviewer     (sonnet, 每轮重启 — 不跨轮复用，纯只读审查)
     └─ data-expert       (sonnet, 条件触发：仅当变更涉及数据模型时，与 reviewer 并行启动（均为纯只读审查），每轮重启)
```

理解整体协作必须把 `skills/team/SKILL.md` 当作"主控代码"读：它是唯一驱动所有阶段切换、生命周期管理、闭环判定的逻辑。其余 6 个 skill 只是被它通过 `Agent(...)` 启动并通过 `SendMessage` 召回 / 关停的子角色。

> **data-expert 的触发由 Phase 5.0 的 `DATA_CHANGE` 探测决定**（架构含数据模型变更 OR diff 命中迁移/Entity/Mapper/索引；探测每轮进入 5.1 前重跑，只允许 false→true 升级）。命中则与 reviewer 同轮启停、其 BLOCK 与 reviewer 同档阻断；未命中且后续轮未引入数据变更则不启动。详见 team/SKILL.md Phase 5.0 与成员配置表下方说明。

> 注：以上生命周期适用于默认完整流程（`/team` 不带参数）。`/team --from=<phase>` 入口下具体存活范围会按起步 phase 调整，见 `skills/team/SKILL.md` §6.1 表。

## 跨文件的关键约定

读单个 SKILL.md 看不出来、改动时容易踩的不变量：

1. **共享工作区固定在目标项目（不是本仓库）的 `.claude/workspace/`**
   所有过程文件路径都是硬编码的，角色之间通过它们传递状态：
   - `task_plan.md`（analyst 产出）
   - `architecture.md`（tech-lead 产出 / `/dev` 直入时由 dev 自写简版）
   - `findings.md`（多角色追加：调研结论、纠偏指令、修改记录、`DATA_CHANGE` 探测结论；统一条目格式 `## [角色][Phase N] 标题（YYYY-MM-DD）`，格式头由 team Phase 0.2 写入。唯一可原地改写的条目是 team 的「进度跟踪」：进度表 + 运行状态块 + 成员实例台账，是 team lead 上下文压缩后的恢复来源，6.1/6.5 回收核对只认台账）
   - `progress.md`（dev 写入）
   - `test_report.md` / `review_report.md`（tester / reviewer 产出）
   - `data_review.md`（data-expert 产出，**仅 `DATA_CHANGE=true` 时存在**；Phase 0.2 清理与 Phase 6 归档都把它与 `review_report.md` 同档处理）
   - `release_checklist.md`（team Phase 6.0 产出的上线工单：卡点三档结论 ✅/⚠️/❌ + 上线步骤 + 回滚预案；与其他报告同档清理/归档，只出工单不拦截流程）
   改动文件名或目录会同时打破多个 skill，需要全局替换。

2. **每个角色必读"目标项目"根目录的 CLAUDE.md**（注意：不是本仓库这份）
   analyst / tech-lead / dev / tester / reviewer / data-expert 的 Step 1 都是「读取项目根 CLAUDE.md」，从中提取 `Service Responsibility Boundary` / `Module Structure` / `Layer Dependencies` / `Tech Stack` / `Package Conventions` / `Important Constraints` 等小节并据此决策（data-expert 侧重 `Tech Stack`/ORM/分库分表与 `Naming Conventions`；tester 侧重 `Tech Stack`/构建工具/测试约定，缺失时按 JUnit5+mvn 默认并在报告标注）。dev 与 tester 另读取可选的 `Database Access` 节（dev Step 4.5 数据验证、tester Step 4.5 数据断言，见约定 10）。新增角色或调整字段名时，要同步所有读取方。

3. **frontmatter 调用控制字段（team 与角色不同，不要混淆）**
   - `team/SKILL.md` 保留 `user-invocable: true` + `disable-model-invocation: true`：编排入口只能由用户 `/team` 主动触发。
   - 其余 6 个角色 skill（含 data-expert）只保留 `user-invocable: true`，**不带** `disable-model-invocation`。带该字段时 skill 描述不进模型上下文、Skill 工具不可调用（官方文档 code.claude.com/docs/en/skills「Control who invokes a skill」），team 成员经 prompt 中的「执行 /X 技能」会被拦截；去掉后成员直接经 Skill 工具加载，team 的启动 prompt 不需要任何路径替换。（2026-09-03 在 Claude Code 2.1.259 实测：subagent 用 Read 读取带该字段的 SKILL.md 并未被拦截，所以「保留字段 + Read 加载」也能跑，但多一层 `${CLAUDE_PLUGIN_ROOT}` 路径拼接且无收益，不采用。）防自动触发改由各角色 description 末尾的「仅限用户显式调用或 /team 编排成员按启动指令加载，不要自动触发」声明承担，修改 description 时不要删掉这句。

4. **生命周期纪律（team/SKILL.md 末尾的"纪律"节）**
   - 成员每完成阶段任务，team 必须立即向其发 `{"type": "shutdown_request"}`，不留空闲成员
   - tester / reviewer / data-expert（命中时）**每轮重启新实例**（不要复用旧的）
   - tech-lead 与 dev **不跨阶段存活**：Phase 2/3 各自完成后立即 shutdown；纠偏（tech-lead）与修复（dev 清单驱动模式）一律按需新实例、即用即关，同名重启前须等前一实例 shutdown_approved
   - 流程结束（含异常终止）必须执行 Phase 6.5 **孤儿成员回收**：teammate 是 in-process 的，`ps aux | grep agent-name` 恒为空、`kill <PID>` 无效，唯一手段是对未 `shutdown_approved` 的成员逐个 `TaskStop(task_id: "{name}")`（团队目录本身随会话结束自动清理，不需要也无法手动删除）
   - 阶段完成判定统一走 **`AWAIT(成员, 判据)` 协议**（team/SKILL.md「阶段完成判定协议」节，全流程 14 个调用点，含 1.2/2.2 成员按用户答复更新产物后的等待，以及「修复实例上报处理」中的 tech-lead 纠偏等待）：teammate 的 idle 通知**不携带产出内容**，判据是产物落盘 + 有结论性章节，idle 通知仅为触发信号。写「等成员把结果回传」的等待逻辑会永久阻塞。新增等待点时必须引用该协议并声明判据，不要另起等待写法

5. **质量闭环上限 = 3 轮**
   tester↔dev 和 reviewer↔dev↔tester 闭环超过 3 轮必须暂停请求人工。修改这个阈值要同时改 team/SKILL.md 的两处 WHILE 循环。Phase 5 的 5.4.2 回归子循环另有独立上限（同一审查轮内重试 1 次，第二次回归仍未通过即超限暂停），与轮次上限共同构成硬边界。

6. **reviewer BLOCK 后修复必须经过 tester 回归**（防止修复引入新 bug）
   这是 Phase 5 的一个非显然规则，写代码或改流程时容易省掉这步。

7. **reviewer 与 data-expert 均为纯只读审查，代码优化全部前置到 dev**（reviewer/SKILL.md 纪律 1、dev/SKILL.md Step 4）
   /simplify 与 code-simplifier 由 dev 在提测/修复前执行，使所有代码优化都发生在 tester **之前**、天然处于测试保护下。改流程时不要把改码动作移回审查阶段——那会重新打开"审查员改码绕过测试"的盲区（并被迫恢复干净树门禁 / 改码回归 / 回滚的整套护栏与 data-expert 串行启动）。

8. **data-expert 与 reviewer 同档阻断、条件触发**（team/SKILL.md 纪律 14 + Phase 5.0）
   仅 `DATA_CHANGE=true` 时启动（探测逐轮重跑，false→true 单向升级）；其 BLOCK 与 reviewer BLOCK 等价（任一 BLOCK 即本轮 BLOCK），dev 修复时合并两份报告清单。

9. **Git 提交纪律贯穿闭环**（team Phase 0.0/0.3、dev Step 4/修复模式、tester Step 5）
   team 启动时探测 `BASE_BRANCH`（main/master，或 `--base=` 显式指定），主干上不执行任何提交类操作（0.3 拦截沿用主干）；0.3 的工作树洁净检查对所有入口生效（未提交改动先基线提交 `chore: baseline` 或退出，全流程入口从非主干起步须用户确认 diff 基准放大）；dev 完成编码/修复、tester 产出测试代码后均在 feature 分支内提交（排除 `.claude/workspace`，禁止 push）；reviewer 纯只读开审，审查范围 = 已提交 diff + 未提交改动（正常流程下应为空，如有则并入范围并在报告标注来源待查）。workspace 不入库的主防线是 team 0.2 幂等写入 `.git/info/exclude`（独立 /dev 入口自行补齐）；提交命令中的 `:(exclude)` 是独立入口未经 0.2 时的兜底——**两层都不要随手删**。

10. **数据验证通道（可选能力）**
   dev（Step 4.5 数据验证）与 tester（Step 4.5 数据断言）依赖目标项目 CLAUDE.md 的 `Database Access` 节声明：MCP 只读工具名（DBHub 多源命名 `execute_sql_{source_id}`，约定 source id 用 `dev` / `test`）、迁移执行方式、分片拓扑、环境自证锚点。三条不变量：
   - MCP 通道永远只读（SELECT/SHOW/DESCRIBE/EXPLAIN）；写操作（执行迁移、造测试数据）走项目原生工具链，不经 MCP
   - 验证/断言前必须环境自证（`SELECT @@hostname, DATABASE()` 对照锚点），严禁生产环境
   - 该节缺失或工具不可见时跳过并在 progress.md / test_report.md 标注原因，不阻塞流程
   目标项目配置模板见 `docs/20260826_database-access-setup.md`。改动节名、工具命名或不变量时要同步 dev、tester 两个 SKILL.md 与该模板。

11. **环境前置清单**
   tech-lead 的 architecture.md 模板固定含「环境前置清单」章节（Nacos 配置 / SQL / 中间件资源，每条标注初始化方式：**工作流内** = dev 编码阶段产出并经 Step 4.5 验证；**人工前置** = agent 无法自建，编码前须人工就绪）。**人工前置条目必须逐条附可直接复制执行的内容块与就绪核验**（完整 SQL / Nacos 配置全文 / 完整命令），禁止摘要式描述；Nacos 定位（namespace/group/dataId/key）须取自目标项目真实配置并与代码消费方一致，分片表 SQL 须按物理拓扑展开；未就绪上报与 team lead 展示时都原样携带内容块。dev Step 1 只核对人工前置项：SQL 类经 `Database Access` 只读通道执行条目自带的就绪核验语句，Nacos/中间件类向上确认；未就绪即暂停上报（/team 由 team lead 3.2 转人工确认）。`/dev` 直入时 dev 自写简版方案也必须含该章节。改动章节名、"工作流内/人工前置"语义或"可复制内容块"要求时要同步 tech-lead、dev、team 三个 SKILL.md。

12. **architecture.md 的「可观测性设计」「灰度与回滚预案」有下游消费者**
   reviewer 按「可观测性设计」核对代码落地（未落地 HIGH），team 6.0 按「灰度与回滚预案」填上线工单。tech-lead 模板与 `/dev` 简版方案都固定含这两节（不涉及写"无"）；改章节名要同步 tech-lead、dev、reviewer、team 四处。`DATA_CHANGE` 同理只有一个口径：dev Step 4.5 在 progress.md「数据验证」块首行写 `DATA_CHANGE=true/false`，team 5.0 与 tester 4.5 都以它为输入之一。

13. **修复实例的上报通路（team「修复实例上报处理」节 + dev 修复模式第 3 条）**
   Phase 4/5 的 dev 修复实例遇到「方案级问题」上报后由 team 启 tech-lead 纠偏，「争议项」（如测试用例本身错误）由 team lead 裁决、tester 下一轮修正用例；清单全部裁决跳过时 AWAIT dev 不再要求新提交。不要删掉这两条通路换回「dev 只能硬改或等超时」。

## 已知陷阱

- **`/dev` ≠ 精简版 `/team`**：`/dev` 走独立 skill，**不含** tester/reviewer 闭环。如需"跳过分析+设计但保留测试+审查"，使用 `/team --from=dev`（含 Phase 4/5 闭环）；`/dev` 仍是"单 dev 编码无闭环"路径。详见 `skills/team/SKILL.md` 末尾「`/dev` 与本 skill 的关系」节与 §6.1 表。
- **命名空间冲突**：装本插件后要清理用户级旧 skill 目录 `~/.claude/skills/{analyst,dev,reviewer,team,tech-lead,tester,data-expert}/`（README 也提到过）。
- **`/dev` 与 `/team` 都会创建 feature 分支**：`{BRANCH_PREFIX}feature/{YYYYMMDD}_{简短描述}`，前缀取 `git config backend-team-workflow.branchPrefix`，缺省 `fyx/`。两者在同一项目接续使用时不要互相覆盖分支。`/team --from in [dev, tester, reviewer]` 不新建分支而沿用当前分支（当前分支为 main/master 时被 0.3 主干保护拦截，须新建 feature 分支或退出）。

## 改动 SKILL.md 的注意事项

- 每个 SKILL.md 末尾都有显式的「纪律」节，是该角色的强约束清单。改主流程前先确认不与这些纪律冲突，要么同步更新。
- 多个 SKILL.md 用近似但不完全一致的措辞描述同一规则（例如"读取项目根 CLAUDE.md"在 6 个角色里都有）。修订时要做整仓搜索，避免只改一处。
- 沿用已有 SKILL.md 的中文行文风格（角色定义 → 项目约束加载 → 执行步骤 → 纠偏/修复模式 → 纪律）。新角色若加入要保持骨架一致。
- 改动 team SKILL 主流程前查阅本地 `docs/` 目录（被 `.gitignore` 忽略，但本地保留历次大改的 spec/plan）——这是设计决策的沉淀位置，可避免无意中违背已确认的方案。
