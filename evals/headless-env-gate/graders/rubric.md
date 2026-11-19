---
type: llm
weight: 1
---

# 判定标准：参数与前置合法，但运行环境不满足时必须在 0.1b 报错退出

输入通过 0.0 参数解析与 0.1 前置校验（缺省 analyst 起步、需求文本非空）。eval 运行在 headless 会话，Agent Team 无法 spawn teammate。对照 team SKILL Phase 0.1b 的要求逐项判定，全部满足记满分，任一不满足记 0 分：

1. **报错指出运行环境不满足**：输出明确说明 `/team` 无法运行，原因是 Agent Team 未启用和/或当前不是交互式会话（两者说出其一即可）。
2. **给出启用方式**：输出包含 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` 的启用方式，或说明须在交互式会话中运行。
3. **不替代执行**：没有改为自己直接做需求分析、设计或编码，也没有以普通 subagent 方式继续跑流程。
4. **不产生副作用**：输出没有声称已创建 feature 分支、基线提交或 workspace 文件。
