# /team 任意节点切入设计

- 日期：2026-05-22
- 状态：设计已批准，待 writing-plans
- 影响范围：`skills/team/SKILL.md`（主改造）、`skills/dev/SKILL.md`（少量同步）、`CLAUDE.md`（同步说明）

## 1. 背景与问题

当前 `/team` 强制从 Phase 1 (analyst) 进入完整 6 阶段流程；`/dev` 是另一独立 skill，仅做单 dev 编码 + code-simplifier，**不含** tester/reviewer 闭环。

存在的缺口：

- 用户已有需求文档 → 想跳过 analyst 直接进 tech-lead 设计：无入口
- 用户已有架构方案 → 想跳过分析+设计直接编码并保留测试审查闭环：无入口
- 用户已写好代码 → 想跑 tester+reviewer 验证：无入口
- 用户只想跑一次 P0 安全审查 → 无入口

`team/SKILL.md` 文末曾承认此缺口（"跳过分析设计但保留测试+审查"无入口支持）。本设计填补该缺口。

## 2. 设计目标

- `/team` 支持 `--from=<phase>` 参数，从 analyst / tech-lead / dev / tester / reviewer 任一节点切入
- 起步 phase 之前的角色不主动启动；起步 phase 及之后的角色按原闭环走
- 跳过的上游角色若被下游闭环需要（如 tester BLOCK 召回 dev）→ 按需首次启动，进入清单驱动模式
- 前置文件缺失立即报错暂停，由人工决策补齐路径
- 不破坏已有 `/team` 默认行为（缺省 `--from=analyst`）

## 3. 入口契约

`/team` 的 `argument-hint` 扩展为：

```
[--from=<phase>] <需求描述或PRD路径，--from=reviewer 时可选>
```

合法 `<phase>`：`analyst | tech-lead | dev | tester | reviewer`，缺省 = `analyst`。

| 参数值 | 起步 phase | 必备前置 (workspace) | 命令行需求 | 新建分支 | 备注 |
|---|---|---|---|---|---|
| *（缺省）* / `analyst` | Phase 1 | — | 必填 | 是 | 等价当前 `/team` |
| `tech-lead` | Phase 2 | `task_plan.md` | 可选 | 是 | 跳过需求分析 |
| `dev` | Phase 3 | `task_plan.md` + `architecture.md` | 可选 | 否 | 跳过分析+设计 |
| `tester` | Phase 4 | `task_plan.md` + `architecture.md` + 代码变更存在 | 可选 | 否 | 跳过分析+设计+编码 |
| `reviewer` | Phase 5 | 代码变更存在；前两份可选 | 可选 | 否 | 跳过分析+设计+编码+测试 |

> 「代码变更存在」= `git diff master...HEAD` 非空 **或** working tree 有未提交改动。

> `/dev` 不动：`/dev` 仍是"单 dev 编码无闭环"独立 skill。`/team --from=dev` 是真正"跳过分析+设计但保留测试+审查"的入口，两者差异写入文档。

## 4. Phase 0 入口路由

Phase 0 改造为"参数解析 + 前置校验"门控，5 种起点共用同一段路由。

### 4.1 参数解析
- 抽出 `--from=<phase>`（缺省 `analyst`），剩余 `$ARGUMENTS` 作为需求文本
- 非法 `<phase>` → 立即报错列出合法值，退出，不创建团队

### 4.2 前置校验

```
match --from in:
  analyst    → $ARGUMENTS 非空
  tech-lead  → exists(.claude/workspace/task_plan.md)
  dev        → exists(task_plan.md) AND exists(architecture.md)
  tester     → exists(task_plan.md) AND exists(architecture.md)
                AND (git diff master...HEAD 非空 OR working tree 有改动)
  reviewer   → (git diff master...HEAD 非空 OR working tree 有改动)
```

校验失败 → 统一报错模板：

```
[❌] --from=dev 需要以下前置文件:
   - .claude/workspace/architecture.md  (缺失)
补齐选项:
  1) 手写 architecture.md 后重跑 /team --from=dev "..."
  2) 退一档: /team --from=tech-lead "..." 让 tech-lead 重新设计
  3) 完整流程: /team "..."
  4) 从历史归档复活: ls .claude/workspace/archive/
```

退出，不创建团队。

### 4.3 工作空间初始化
- `.claude/workspace/` 不存在 → 创建
- `--from in [analyst, tech-lead]` 起步（"全新流程"语义）：
  - 任何已存在的下游产物视为脏工作区，交 §7.2 处理（人机交互）
  - 下游产物清单：`architecture.md` / `findings.md` / `progress.md` / `test_report.md` / `review_report.md`（`--from=analyst` 时还包括 `task_plan.md`）
- `--from in [dev, tester, reviewer]` 起步（"显式复用 workspace"语义）：
  - 保留 4.2 校验通过的前置文件 + `findings.md`（历史调研结论）
  - **静默清理**当前起步之后阶段的旧产物：
    - `--from=dev` → 清 `progress.md` / `test_report.md` / `review_report.md`
    - `--from=tester` → 清 `test_report.md` / `review_report.md`
    - `--from=reviewer` → 清 `review_report.md`

### 4.4 分支创建
- 统一命名格式：`fyx/feature/{YYYYMMDD}_{简短描述}`（带用户前缀 `fyx/`）
- `--from in [analyst, tech-lead]` → `git checkout -b fyx/feature/{YYYYMMDD}_{简短描述}`
- 其他 → 仅打印当前分支并提示"沿用当前分支，是否继续？Y/n"

> 此格式也适用于现有完整流程 `/team` 的 Phase 0.2 — 实施时需同步修改 team/SKILL.md 现有分支命名代码（旧 `feature/...` → 新 `fyx/feature/...`）。

### 4.5 TeamCreate
- team 名规则不变：`dev-team-{简短描述}`

### 4.6 TaskCreate
- 只创建从起步 phase 开始的 Phase 任务（如 `--from=dev` 只建 Phase 3-6 四个任务）

## 5. 各 Phase 起首守卫

Phase 1/2/3/4/5 各加一行起首守卫：

```
IF 起步 phase > 本 phase: 跳过整个 phase
```

实现为 team 在进入每个 phase 前的条件分支，跳过的 phase 不启动对应成员、不读写对应产物。

## 6. 成员生命周期调整

核心原则：**`--from=X` 等于声明 X 之前的角色不主动启动**；但若下游闭环路径需要它们，按需首次启动并进入对应模式。

### 6.1 跨阶段存活规则（按起步 phase 对齐）

| 起步 phase | analyst | tech-lead | dev | tester | reviewer |
|---|---|---|---|---|---|
| analyst | P1 启 / P1 关 | P2 启 / P3 关 | P3 启 / P5 APPROVE 后关 | 每轮启关 | 每轮启关 |
| tech-lead | — | P2 启 / P3 关 | P3 启 / P5 APPROVE 后关 | 每轮启关 | 每轮启关 |
| dev | — | **按需启动（dev 偏离时纠偏模式）** | P3 启 / P5 APPROVE 后关 | 每轮启关 | 每轮启关 |
| tester | — | — | **按需启动（BLOCK 时清单驱动模式）** | P4 每轮启关 | 每轮启关 |
| reviewer | — | — | **按需启动（BLOCK 时清单驱动模式）** | **按需启动（reviewer BLOCK 后回归）** | P5 每轮启关 |

### 6.2 dev 的两种启动 prompt

**A. 编码模式**（`--from in [analyst, tech-lead]` 时 Phase 3 首启）—— 沿用现有 prompt，读 architecture.md 从零实现。

**B. 清单驱动模式**（`--from in [dev, tester, reviewer]` 时按需首启或后续召回）：

```
你是开发团队的后端开发工程师（清单驱动模式）。
当前流程从 --from={X} 进入。

任务范围：根据 {test_report.md | review_report.md} 中列出的"问题/缺口清单"
逐项处理——**既包括 Bug 修复，也包括补齐报告指出的缺失实现**：
- 未覆盖的验收标准 → 补实现
- 未实施的安全控制（如缺少输入校验、缺少权限检查）→ 补实现
- Bug → 修复

约束：
1. 严格在报告清单内行动；清单外的代码即使你认为有问题也只能 SendMessage 报告，
   不要借机做无关重构或额外功能
2. 项目根 CLAUDE.md 必读；architecture.md 若存在必读，存在歧义时以 task_plan.md 为准
3. 完成后执行 code-simplifier 并保证编译通过
4. 通知 team lead 并保持 idle，Phase 4/5 代码冻结纪律继续生效
```

> 注：`--from=dev` 的 Phase 3 首启走 A 编码模式（按 architecture.md 从零实现）。B 模式只出现在 Phase 4/5 被召回时（无论起步 phase 为何）。

> **2026-05-22 实施修正（勘误）**：模式 B 仅用于 dev 在本任务中的首次 Agent() 启动（START_PHASE in [tester, reviewer]）；已存活 dev 的召回一律 SendMessage 修复模式指令，--from=dev 的 Phase 3 首启走模式 A。本节标题与上方注释中「按需首启或后续召回」「无论起步 phase 为何」的表述以此为准。

### 6.3 tech-lead 的纠偏模式
`--from=dev` 时若 dev 偏离 architecture，team 首次启动 tech-lead 进入"纠偏模式"，沿用现有纠偏 prompt，无需新模板。

### 6.4 不变的规则
- 同名成员重启前等 `shutdown_approved`（首次按需启动不存在前一轮，跳过该等待）
- tester / reviewer 每轮重启
- reviewer BLOCK → dev 修复 → **tester 回归**；即使 `--from=reviewer` 也按此规则首次启动 tester
- 闭环上限仍为 3 轮

### 6.5 `--from=reviewer` 的特殊路径
若 `--from=reviewer` 且 reviewer 第一轮直接 APPROVE → 跳过 dev/tester 全部启动，直入 Phase 6。这是唯一的"纯审查"路径。

## 7. 错误处理与边界

### 7.1 参数与前置校验失败（Phase 0 内拦截）

| 错误类型 | 触发条件 | 处理动作 |
|---|---|---|
| 非法 phase 值 | `--from=foo` 不在 5 个合法值中 | ❌ 报错列出合法值，退出，不创建团队 |
| 前置文件缺失 | §4.2 校验任一项失败 | ❌ 报错列具体缺失文件 + 4 种补齐选项，退出，不创建团队 |
| 代码变更不存在 | `--from in [tester, reviewer]` 且 `git diff master...HEAD` 与 working tree 均无变更 | ❌ 报错"没有可测试/审查的代码变更"，提示退回 `--from=dev` 或检查分支 |
| 需求文本缺失 | `--from=analyst` 且 `$ARGUMENTS` 为空 | ❌ 报错"analyst 起步必须提供 PRD 路径或需求描述" |

> `--from in [tech-lead, dev, tester, reviewer]` 时 `$ARGUMENTS` 可空，按 task_plan.md 内容继续。

### 7.2 脏工作区检测

`--from in [analyst, tech-lead]` 启动时，若 workspace 中已存在 §4.3 列出的下游产物清单中任一文件：

```
[⚠️] 检测到 workspace 中已有以下下游产物:
   - architecture.md (last modified: 2026-05-15)
   - test_report.md
继续将覆盖。请选择:
  1) 归档旧产物到 archive/ 后继续 (推荐)
  2) 直接覆盖
  3) 取消，先用 --from=tech-lead 复用现有产物
```

team 等用户输入再推进。

`--from in [dev, tester, reviewer]` 时不触发此告警（用户已显式声明复用 workspace）。

### 7.3 流程内异常（与起步 phase 无关）

| 场景 | 处理动作 |
|---|---|
| 闭环超 3 轮 | 暂停请求人工 |
| 任意 shutdown_request 超 60 秒无响应 | 记为孤儿候选，Phase 6.6 用 `ps` 验证 |
| 用户中断 | 关闭所有存活成员 → TeamDelete → 报告 workspace 当前状态 |
| `--from=reviewer` 直接 APPROVE | 跳过 dev/tester 启动，**仍执行** Phase 6 归档 |

### 7.4 不在本次 scope 内
- `--to=<phase>` 提前结束
- `--skip=<phase>` 中间跳过（如跑 analyst 跳过 tech-lead）
- 多份 PRD 合并起步
- 跨 team 实例复用 workspace

## 8. 纪律与文档同步

### 8.1 team/SKILL.md "纪律"节新增

在现有 9 条之后追加：

```
10. --from=X 等于声明 X 之前的角色不主动启动 — 但若下游闭环路径需要它们，
    按需首次启动并使用对应模式 prompt（dev → 清单驱动模式；tech-lead → 纠偏模式）。
    禁止"--from=X 就完全禁用 X 之前的角色"的简化理解，会破坏闭环可达性
11. --from in [dev, tester, reviewer] 时沿用当前分支 — 不新建 feature 分支；
    启动时打印当前分支供用户确认
12. --from=X 的前置校验失败必须立即报错暂停 — 不允许"静默回退到上游 phase 补生成"，
    否则用户会误以为跳过了实际没跳
13. Phase 6 归档/孤儿检查无论起步 phase 如何都必须执行 — 包括 --from=reviewer
    一次 APPROVE 的最短路径
```

### 8.2 team/SKILL.md 其他章节同步

- **frontmatter `argument-hint`**：扩展为 `[--from=<phase>] <需求描述或PRD路径，--from=reviewer 时可选>`
- **Phase 0**：完整替换为 §4 的"参数解析 + 前置校验 + 工作空间清理 + 条件分支 + 条件 TaskCreate"；分支命名一并改为 `fyx/feature/{YYYYMMDD}_{简短描述}`
- **Phase 1/2/3/4/5 起首**：各加一行 §5 起首守卫
- **"成员生命周期管理"表**：在原表下方新增 §6.1 表
- **"`/dev` 与本 skill 的关系"节**：补对比行—— `/team --from=dev` **含** Phase 4/5 闭环；`/dev` **不含**
- **"终止条件与升级机制"表**：新增 `--from=X 前置校验失败 → 暂停，列出补齐选项`

### 8.3 dev/SKILL.md 同步

- 加 3 行说明：识别 prompt 中的"清单驱动模式"标识，跳过 Step 0 的复杂度评估（清单模式不需要升级为 dev-leader 团队）
- 若 dev skill 自身也创建 feature 分支，命令同步为 `fyx/feature/{YYYYMMDD}_{简短描述}`

### 8.4 项目 CLAUDE.md 同步

`/Users/macpro/claude-plugin/backend-team-workflow/CLAUDE.md` 三处改动：
- "架构：6 角色 + Agent Team" 节—— 在生命周期表后加一句"具体存活范围会随 `--from=X` 调整，见 team/SKILL.md §6.1"
- "已知陷阱" 节—— 扩写"`/dev` ≠ 精简版 `/team`"那条：`/team --from=dev` 是真正"跳过分析+设计但保留测试+审查"的入口；`/dev` 仍不含闭环
- "已知陷阱" 节—— 更新分支命名示例：`feature/{YYYYMMDD}_{简短描述}` → `fyx/feature/{YYYYMMDD}_{简短描述}`（与 §4.4 一致）

### 8.5 其他 4 个角色 SKILL.md
analyst / tech-lead / tester / reviewer 的 SKILL.md **无需改动**。它们的 prompt 由 team 在 Agent() 时注入；team 注入了正确模式标识即可。

## 9. 验证策略

由于本仓库无构建/测试，验证依赖装到真实项目跑通：

1. **回归基线**：缺省 `/team "需求"` 行为与改造前完全一致（5 个回归用例覆盖每个 Phase）
2. **5 种新入口冒烟**：
   - `/team --from=tech-lead`（前置：手写 task_plan.md）
   - `/team --from=dev`（前置：手写 task_plan.md + architecture.md）
   - `/team --from=tester`（前置：上述两份 + 写一段代码）
   - `/team --from=reviewer`（前置：写一段代码）
   - `/team --from=reviewer` 直接 APPROVE 路径
3. **错误路径**：缺前置文件 → 报错模板正确显示；非法 phase 值 → 报错；脏工作区 → 3 选 1 提示
4. **闭环可达性**：`--from=reviewer` BLOCK 时正确召回 dev（清单驱动模式）+ 启动 tester 回归

每个用例的 pass 标准：team 行为符合本设计描述，且 Phase 6 归档与孤儿进程检查均执行。
