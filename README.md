# backend-team-workflow

后端团队工作流插件。以 `team` 为总入口，串联 5 个角色 skill 完成「需求 → 设计 → 开发 → 测试 → 审查」闭环。

## 角色

| Skill | 角色 | 入场时机 |
|---|---|---|
| `team` | 团队编排 | 总入口，按需调度其他 5 个角色 |
| `analyst` | 需求分析师 | 收到 PRD / 需求变更时 |
| `tech-lead` | 技术负责人 | 需求分析完成后做技术设计 |
| `dev` | 后端开发 | 技术方案敲定后编码 |
| `tester` | 测试工程师 | 开发完成后写并跑测试 |
| `reviewer` | 代码审核 | 测试通过后做最终审查 |

## 安装

```bash
# 方式一：本地路径直装（开发期）
/plugin marketplace add ~/claude-plugins/backend-team-workflow
/plugin install backend-team-workflow

# 方式二：通过本地 marketplace 集合（多插件共管）
/plugin marketplace add ~/claude-plugins
/plugin install backend-team-workflow@<marketplace-name>
```

安装后可通过 `/backend-team-workflow:team` 启动总入口。

## 开发

修改 `skills/<role>/SKILL.md` 后执行 `/reload-plugins` 即可热生效。

## 注意

本插件接管 6 个 skill 后，`~/.claude/skills/{analyst,dev,reviewer,team,tech-lead,tester}/` 应当移除以避免重复加载。
