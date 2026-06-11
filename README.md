# backend-team-workflow

后端团队工作流插件。以 `team` 为总入口，串联 5 个固定角色 + 1 个条件触发角色（data-expert）的 skill 完成「需求 → 设计 → 开发 → 测试 → 审查」闭环。

## 角色

| Skill | 角色 | 入场时机 |
|---|---|---|
| `team` | 团队编排 | 总入口，按需调度其他 6 个角色（data-expert 为条件触发） |
| `analyst` | 需求分析师 | 收到 PRD / 需求变更时 |
| `tech-lead` | 技术负责人 | 需求分析完成后做技术设计 |
| `dev` | 后端开发 | 技术方案敲定后编码 |
| `tester` | 测试工程师 | 开发完成后写并跑测试（含集成边界） |
| `reviewer` | 代码审核 | 测试通过后做最终审查 |
| `data-expert` | 数据治理审查员 | **条件触发**：变更涉及数据模型时，Phase 5 在 reviewer 优化阶段（Step 1.1–1.3）完成后启动，与其只读审查阶段（Step 2–4）并行审查迁移/索引/一致性 |

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

工作流引用 4 个非本插件自带的外部技能/插件，建议目标环境提前装齐；缺失时各角色按 SKILL 内降级路径执行，不会卡死流程：

| 依赖 | 用途 | 缺失时降级 |
|---|---|---|
| `ralph-loop`（插件） | dev 编译迭代闭环（Step 3A） | 退化为手动「编译→修错」循环，同 3 轮上限 |
| `code-simplifier` | dev / reviewer 代码简化 | 跳过自动优化，报告标注「技能缺失未执行」，坏味道走人工审查报告 |
| `simplify` | reviewer 优化阶段（Step 1） | 同上 |
| `claude-md-management`（可选） | Phase 6 用 `/revise-claude-md` 更新项目 CLAUDE.md | 提示用户手动更新 |

## 开发

修改 `skills/<role>/SKILL.md` 后执行 `/reload-plugins` 即可热生效。

## 注意

本插件接管 7 个 skill 后，`~/.claude/skills/{analyst,dev,reviewer,team,tech-lead,tester,data-expert}/` 应当移除以避免重复加载。
