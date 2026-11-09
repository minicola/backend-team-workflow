# backend-team-workflow

**Java 后端**团队工作流插件（Spring / COLA 分层生态预设）。以 `team` 为总入口，串联 5 个固定角色 + 1 个条件触发角色（data-expert）的 skill 完成「需求 → 设计 → 开发 → 测试 → 审查」闭环。

> 定位说明：dev 复杂度评分表、tester 默认基线（JUnit 5 + Maven）、reviewer 检查清单（Spring/MyBatis/Lombok 等）均按 Java 生态预设；各角色会优先读取目标项目根 CLAUDE.md 的实际约定覆盖这些默认值，但在非 JVM 技术栈上未经验证。

## 角色

| Skill | 角色 | 入场时机 |
|---|---|---|
| `team` | 团队编排 | 总入口，按需调度其他 6 个角色（data-expert 为条件触发） |
| `analyst` | 需求分析师 | 收到 PRD / 需求变更时 |
| `tech-lead` | 技术负责人 | 需求分析完成后做技术设计 |
| `dev` | 后端开发 | 技术方案敲定后编码（fail-fast，拒绝防御式编码；关键业务逻辑行注释 + 精简 javadoc） |
| `tester` | 测试工程师 | 开发完成后写并跑测试（含集成边界、test 库真实数据取样） |
| `reviewer` | 代码审核 | 测试通过后做最终审查（含坏味道与防御式编码扫描） |
| `data-expert` | 数据治理审查员 | **条件触发**：变更涉及数据模型时，Phase 5 与 reviewer 并行启动（两者均为纯只读审查），审查迁移/索引/一致性 |
| `tidy` | 归档治理（工具，非团队成员） | `/team` 收尾自动以 `--auto` 执行；也可随时手动运行，交互整理存量 |

## 安装

```bash
# 方式一：本地路径直装（开发期）
/plugin marketplace add ~/claude-plugin/backend-team-workflow
/plugin install backend-team-workflow

# 方式二：通过本地 marketplace 集合（多插件共管）
/plugin marketplace add ~/claude-plugin
/plugin install backend-team-workflow@<marketplace-name>
```

安装后可通过 `/backend-team-workflow:team` 启动总入口。

## 前置依赖

工作流引用 5 个非本插件自带的外部技能/插件，建议目标环境提前装齐；缺失时各角色按 SKILL 内降级路径执行，不会卡死流程：

| 依赖 | 用途 | 缺失时降级 |
|---|---|---|
| `ralph-loop`（插件） | dev 编译迭代闭环（Step 3A） | 退化为手动「编译→修错」循环，同 3 轮上限 |
| `code-simplifier` | dev 代码简化（Step 4，提测/修复前） | 跳过自动优化，findings.md 标注「技能缺失未执行」，坏味道由 reviewer 报告交 dev 处理 |
| `simplify` | dev 代码优化（Step 4，与 code-simplifier 顺序执行） | 同上 |
| `claude-md-management`（可选） | Phase 6 用 `/revise-claude-md` 更新项目 CLAUDE.md | 提示用户手动更新 |
| `release-checklist`（可选） | Phase 6.0 上线卡点核对 + 生成上线工单 | 降级为读项目根 `release-checklist.yaml` 逐项核对，再缺则用内置通用卡点清单，产出 `release_checklist.md` |

## 工作区与归档治理

过程文件都在目标项目的 `.claude/workspace/`（已写入 `.git/info/exclude`，不入库）：

- 根目录：在途产物（`task_plan.md` / `architecture.md` / `findings.md` / `progress.md` / 各报告），以及 `attachments/`（各角色核心产物之外的过程文件，如分片 DDL、PoC 报告、配图）
- `archive/{YYYYMMDD_HHMMSS}_{需求}/`：`/team` 收尾归档，`summary.md` 顶部带「归档元数据」（分支、基准分支、HEAD、结论）
- `archive/INDEX.md`：全部归档的索引，由 `tidy` 按元数据重建
- `archive/_expired/`：过期区

过期规则（两段式自动）：归档的 HEAD 已合入基准分支且超过保留期 → 移入 `_expired/`；在 `_expired/` 满宽限期 → 删除，删除记录保留在 INDEX。信息不全的归档不会被自动处理，只列为人工候选。

```bash
git config backend-team-workflow.archiveRetentionDays 90   # 保留期，缺省 90 天
git config backend-team-workflow.archivePurgeDays 30       # 宽限期，缺省 30 天
/backend-team-workflow:tidy                                # 交互整理存量：遗留归档补元数据、散落文件归并、人工候选处置
```

`tidy` 只管理本工作流的文档：workspace 中的其他文件只列出，不移动也不删除；恢复过期归档只需把目录从 `_expired/` 移回 `archive/`。

## 开发

修改 `skills/<role>/SKILL.md` 后执行 `/reload-plugins` 即可热生效。

回归验证：`evals/` 下有针对 Phase 0 参数解析/前置校验的 `claude plugin eval` 用例（该 CLI 功能目前 early access；未开启时按 `evals/README.md` 人工对照 rubric 验收）。

## 注意

本插件接管 8 个 skill 后，`~/.claude/skills/{analyst,dev,reviewer,team,tech-lead,tester,data-expert,tidy}/` 应当移除以避免重复加载。
