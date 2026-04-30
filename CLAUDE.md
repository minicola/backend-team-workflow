# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库性质

这是一个 **Claude Code Plugin** 仓库，不是普通代码库：
- 没有源代码、没有构建系统、没有测试
- 产品就是 `skills/<role>/SKILL.md` 这 6 份 markdown 文件 + `.claude-plugin/plugin.json`
- "开发"= 编辑 SKILL.md 中的角色指令；"发布"= 提交并让用户重新 `/plugin install` 或 `/reload-plugins` 热生效

不存在 `npm test` / `mvn compile` 之类的入口。提交前的唯一"验证"就是把插件装到一个真实项目上跑一遍 `/team` 或 `/dev`。

## 常用操作

```bash
# 装到本地（开发期，从这个目录直装）
/plugin marketplace add ~/claude-plugins/backend-team-workflow
/plugin install backend-team-workflow

# 改完 SKILL.md 后热生效（不需要重启 Claude Code）
/reload-plugins

# 启动总入口（在目标业务项目里使用）
/backend-team-workflow:team <需求描述或PRD路径>
```

## 架构：6 角色 + Agent Team

整个插件是一个**编排 + 5 角色**的工作流，依赖 Claude Code 的 Agent Team API：`TeamCreate` / `Agent(team_name=…)` / `SendMessage` / `TeamDelete`。

```
team (编排，opus)
 ├─ Phase 1: analyst     (opus,   仅 Phase 1)
 ├─ Phase 2: tech-lead   (opus,   Phase 2-3 跨阶段存活，Phase 3 末尾才 shutdown)
 ├─ Phase 3: dev         (sonnet, Phase 3-5 跨阶段存活，reviewer APPROVE 才 shutdown)
 ├─ Phase 4: tester      (sonnet, 每轮重启 — 不跨轮复用)
 └─ Phase 5: reviewer    (sonnet, 每轮重启 — 不跨轮复用)
```

理解整体协作必须把 `skills/team/SKILL.md` 当作"主控代码"读：它是唯一驱动所有阶段切换、生命周期管理、闭环判定的逻辑。其余 5 个 skill 只是被它通过 `Agent(...)` 启动并通过 `SendMessage` 召回 / 关停的子角色。

## 跨文件的关键约定

读单个 SKILL.md 看不出来、改动时容易踩的不变量：

1. **共享工作区固定在目标项目（不是本仓库）的 `.claude/workspace/`**
   所有过程文件路径都是硬编码的，6 个角色之间通过它们传递状态：
   - `task_plan.md`（analyst 产出）
   - `architecture.md`（tech-lead 产出 / `/dev` 直入时由 dev 自写简版）
   - `findings.md`（多角色追加：调研结论、纠偏指令、修改记录）
   - `progress.md`（dev 写入）
   - `test_report.md` / `review_report.md`（tester / reviewer 产出）
   改动文件名或目录会同时打破多个 skill，需要全局替换。

2. **每个角色必读"目标项目"根目录的 CLAUDE.md**（注意：不是本仓库这份）
   analyst / tech-lead / dev / reviewer 的 Step 1 都是「读取项目根 CLAUDE.md」，从中提取 `Service Responsibility Boundary` / `Module Structure` / `Layer Dependencies` / `Tech Stack` / `Package Conventions` / `Important Constraints` 等小节并据此决策。新增角色或调整字段名时，要同步所有读取方。

3. **frontmatter `user-invocable: true` + `disable-model-invocation: true`**
   6 份 SKILL.md 都带这对字段：只能由用户用 `/<skill>` 主动触发，模型不能自动调用。修改时不要随手改掉。

4. **生命周期纪律（team/SKILL.md 末尾的"纪律"节）**
   - 成员每完成阶段任务必须发 `{"type": "shutdown_request"}`，不留空闲成员
   - tester / reviewer **每轮重启新实例**（不要复用旧的）
   - tech-lead 与 dev **跨阶段存活**，时机到了才 shutdown
   - 流程结束（含异常终止）必须 `TeamDelete()`

5. **质量闭环上限 = 3 轮**
   tester↔dev 和 reviewer↔dev↔tester 闭环超过 3 轮必须暂停请求人工。修改这个阈值要同时改 team/SKILL.md 的两处 WHILE 循环。

6. **reviewer BLOCK 后修复必须经过 tester 回归**（防止修复引入新 bug）
   这是 Phase 5 的一个非显然规则，写代码或改流程时容易省掉这步。

## 已知陷阱

- **`/dev` ≠ 精简版 `/team`**：`/dev` 走独立 skill，**不含** tester/reviewer 闭环。详见 `skills/team/SKILL.md` 末尾「`/dev` 与本 skill 的关系」节。如需"跳过分析+设计但保留测试+审查"，目前没有任何入口支持。
- **命名空间冲突**：装本插件后要清理用户级旧 skill 目录 `~/.claude/skills/{analyst,dev,reviewer,team,tech-lead,tester}/`（README 也提到过）。
- **`/dev` 与 `/team` 都会创建 feature 分支**：`feature/{YYYYMMDD}_{简短描述}`。两者在同一项目接续使用时不要互相覆盖分支。

## 改动 SKILL.md 的注意事项

- 每个 SKILL.md 末尾都有显式的「纪律」节，是该角色的强约束清单。改主流程前先确认不与这些纪律冲突，要么同步更新。
- 多个 SKILL.md 用近似但不完全一致的措辞描述同一规则（例如"读取项目根 CLAUDE.md"在 4 个角色里都有）。修订时要做整仓搜索，避免只改一处。
- 沿用已有 SKILL.md 的中文行文风格（角色定义 → 项目约束加载 → 执行步骤 → 纠偏/修复模式 → 纪律）。新角色若加入要保持骨架一致。
