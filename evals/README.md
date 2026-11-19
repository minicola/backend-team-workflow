# 插件回归 eval 套件

针对 team 主控 **Phase 0（参数解析 + 前置校验 + 运行环境门禁）** 的回归用例。选择这些路径的原因：全部在启动任何成员之前廉价、确定性地退出，不会真正启动 Agent Team、不产生多轮闭环成本。

## 状态

`claude plugin eval` 已可直接运行（Claude Code 2.1.328 实测，官方文档 code.claude.com/docs/en/plugin-evals）。改动 `skills/team/SKILL.md` 的 Phase 0 后跑一遍本套件；Phase 0.2 之后的控制流 eval 到不了，仍按 `manual-acceptance.md` 人工验收。

## 如何运行

```bash
# 运行全部用例（target 用本仓库路径）
claude plugin eval ~/claude-plugin/backend-team-workflow --ablation none --no-publish

# 只跑某个用例
claude plugin eval ~/claude-plugin/backend-team-workflow --ablation none --no-publish --case from-illegal-value

# CI：JSON 输出 + 成本上限（任一 case 低于 1.0 则 exit 1，threshold 缺省即 1.0）
claude plugin eval ~/claude-plugin/backend-team-workflow --ablation none --no-publish --trust-plugin --json results.json --max-cost-usd 6
```

两个参数每次都带，理由：

- `--ablation none`：缺省会再跑一组「不装插件」的基线臂。本套件的 prompt 都是 `/backend-team-workflow:team ...`，没有插件时该命令不存在，基线臂无意义且成本翻倍。
- `--no-publish`：缺省会把 HTML 报告发布到 claude.ai。

成本参考：每个 run 是一个完整的 claude 子会话，要加载 900 多行的 team SKILL，实测单 run 约 $0.31-0.36；4 个用例 × 缺省 3 runs ≈ $4。调试单个用例时加 `--runs 1`。

## 用例结构

每个用例一个目录：`prompt.md`（输入，可带 frontmatter 如 `max_turns`）+ `graders/*.md`（每个文件一个 grader，**必须带 frontmatter 的 `type`**，缺失时整个用例加载失败并报 `graders: Required`）。本套件每个用例固定三个 grader，全部通过才算过：

| grader | 类型 | 判定 |
|---|---|---|
| `rubric.md` | `llm` | 按 rubric 判定报错文案。**判分模型只看最后一条消息**，看不到工具调用，所以「没启动成员」「没写文件」不能只靠它 |
| `no-member-started.md` | `tool_used` | `Agent` 调用次数为 0 |
| `no-workspace-artifact.md` | `file_exists` | 运行中没有新建 `.claude/workspace/**` |

## 注意事项

- **工具门禁**：eval 缺省只给只读工具（Read / Glob / Grep / Skill / Agent 等），`Bash` / `Write` / `Edit` 须由运行者用 `--allow-tools` 授权，用例内声明无效。因此缺省运行下 `no-workspace-artifact` 恒真，只有带 `--allow-tools Bash Write` 运行时才真正起约束；`no-member-started` 始终有效（`Agent` 缺省可用）。
- **0.1b 门禁与用例的关系**：`from-illegal-value` / `from-dev-missing-prereq` / `no-requirement-text` 在 0.0/0.1 就退出，到不了 0.1b——门禁刻意排在参数/前置校验之后正是为了保住这三个场景的期望输出。`headless-env-gate` 用合法输入走到 0.1b，验证 headless 下报错退出而不是退化成普通 subagent 继续跑。
- **0.1b 之后到不了**：0.1b 同时要求交互式会话，eval 是 headless，0.2 脏工作区、0.3 主干保护与工作树洁净检查、以及需要完整团队运行的机制（tech-lead 轻量/标准分流、遗留必修项二选一、回归重试上限等）都无法用本套件覆盖，判定标准见 `manual-acceptance.md`。
- **可扩展方向**：`--base` 非法分支报错（0.0）与 `--from=tester/reviewer` 无代码变更（0.1）都在门禁之前，但要 git 仓库，需 `scaffold_script`（运行时 `--scaffold`）加 `--allow-tools Bash`；单角色 skill（如 `tidy`）不经 team 门禁，可独立成用例。
- **结果目录**：缺省写到 `evals/results/{时间戳}/`，已在 `.gitignore` 中忽略。

## 用例清单

| case | 覆盖点 | 期望 |
|---|---|---|
| `from-illegal-value` | 0.0 非法 `--from` 值 | 报错列出合法值，不启动任何团队成员 |
| `from-dev-missing-prereq` | 0.1 `--from=dev` 前置文件缺失 | 报错模板列缺失文件 + 4 个补齐选项，不启动任何团队成员 |
| `no-requirement-text` | 0.1 缺省 analyst 起步且需求文本为空 | 报错要求需求文本，不启动任何团队成员 |
| `headless-env-gate` | 0.1b 输入合法但运行环境不满足 | 报错说明 Agent Team 未启用 / 非交互式会话，不启动任何团队成员，不替代执行 |
