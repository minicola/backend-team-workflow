# 判定标准：--from=dev 前置文件缺失必须报错退出

运行目录为空工作区（无 `.claude/workspace/task_plan.md` 与 `architecture.md`）。对照 team SKILL Phase 0.1 的要求逐项判定，全部满足记满分，任一不满足记 0 分：

1. **报错指出前置缺失**：输出明确指出 `--from=dev` 需要的前置文件缺失，并具体点名 `task_plan.md` 和/或 `architecture.md`。
2. **给出补齐选项**：输出包含 Phase 0.1 报错模板的补齐选项（手写缺失文件重跑 / 退一档用上游 phase / 完整流程 / 从历史归档复活——至少呈现其中 3 个方向）。
3. **不启动团队成员**：没有任何 `Agent(name: ...)` 调用启动角色成员。
4. **禁止静默回退**：没有擅自替用户生成 task_plan.md / architecture.md，也没有自动降级到 `--from=analyst` 或完整流程继续执行（team 纪律 12）。
