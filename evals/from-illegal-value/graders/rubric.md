# 判定标准：非法 --from 值必须报错退出

对照 team SKILL Phase 0.0 的要求逐项判定，全部满足记满分，任一不满足记 0 分：

1. **报错并点名非法值**：输出明确指出 `coder` 是非法的 `--from` 值。
2. **列出全部合法值**：输出包含完整合法值清单 `analyst | tech-lead | dev | tester | reviewer`（五个都在，顺序不限）。
3. **不创建团队**：没有调用 TeamCreate / 创建 Agent Team / 启动任何角色成员的行为；没有创建 feature 分支或 `.claude/workspace/` 产物。
4. **干净退出**：报错后流程终止，没有转而按其他 --from 值或完整流程继续执行需求。
