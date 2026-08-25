# 插件回归 eval 套件

针对 team 主控 **Phase 0（参数解析 + 前置校验）** 的回归用例。选择这些路径的原因：全部在创建团队之前廉价、确定性地退出，不会真正启动 Agent Team、不产生多轮闭环成本。

## 状态

`claude plugin eval` 目前为 **early access**（本机 CLI 已内置该子命令但未开启，运行会提示 early access）。开启前本套件作为**文档化的验收标准**使用：人工验证时按各 case 的 `prompt.md` 输入、对照 `graders/rubric.md` 判定。

## 开启后如何运行

```bash
# 校准（首次）：用官方模板核对 case 结构是否需要补充 case.yaml（runs/max_turns 等字段）
claude plugin eval init --bare _schema_probe && rm -rf evals/_schema_probe

# 运行全部用例（target 用本仓库路径）
claude plugin eval ~/claude-plugin/backend-team-workflow

# 只跑某个用例
claude plugin eval ~/claude-plugin/backend-team-workflow --case from-illegal-value

# CI：JSON 输出 + 阈值门禁（任一 case 低于 1.0 则 exit 1）
claude plugin eval ~/claude-plugin/backend-team-workflow --json results.json --threshold 1.0
```

## 校准注意事项

- **skill 命名空间**：prompt.md 中使用 `/backend-team-workflow:team`。eval 以路径为 target 加载插件时，slash 前缀可能是 `/team`（无命名空间）——首跑若未触发 skill，把各 prompt.md 的前缀改为实际生效的形式。
- **运行环境**：eval scaffold 目录默认为空（无 `.claude/workspace/`、无 git 历史），恰好构成 `--from=dev` 前置缺失的场景；若未来某 case 需要预置文件，用 `scaffold_script`（运行时需显式 `--scaffold`）。
- **0.1b 环境门禁与 eval 的关系**：team SKILL 在 0.1 之后有「Agent Team 未启用即报错退出」的运行环境门禁（0.1b）。现有三个用例都在 0.0/0.1 就退出，**到不了 0.1b**，不受影响——门禁刻意排在参数/前置校验之后正是为了保住这三个场景的期望输出。未来若新增需要跑到 0.1b 之后的用例（如 0.2 脏工作区、0.3 主干保护），eval 沙箱默认没有 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`，必须在 case 环境或 scaffold 中显式设置该变量为 `1`，否则会先撞上门禁报错。
- **用例扩展方向**（当前刻意未包含，避免真实启动团队的成本）：DATA_CHANGE 探测（Phase 5.0 grep 命中判定）、脏工作区三选一交互（0.2）、主干保护拦截（0.3）；以及需要完整团队运行才能触发的三个 v0.3.0 新机制——tech-lead 轻量/标准分流（评分 ≤6 单方案）、遗留必修项二选一（5.4 APPROVE 分支）、回归重试上限（5.4.2 同轮第二次回归失败即超限暂停）。

## 用例清单

| case | 覆盖点 | 期望 |
|---|---|---|
| `from-illegal-value` | 0.0 非法 `--from` 值 | 报错列出合法值，不启动任何团队成员 |
| `from-dev-missing-prereq` | 0.1 `--from=dev` 前置文件缺失 | 报错模板列缺失文件 + 4 个补齐选项，不启动任何团队成员 |
| `no-requirement-text` | 0.1 缺省 analyst 起步且需求文本为空 | 报错要求需求文本，不启动任何团队成员 |
