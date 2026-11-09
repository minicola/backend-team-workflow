---
name: tidy
description: 工作流文档治理 - 整理目标项目 .claude/workspace/ 中归属于本工作流的文档：按各归档 summary.md 的「归档元数据」重建 archive/INDEX.md，两段式自动清理过期归档（HEAD 已合入基准分支且超保留期 → 移入 archive/_expired/，再满宽限期 → 删除）；无参数时交互式整理存量（遗留归档补元数据、散落文件归并），非工作流文件只列出不动。使用场景：/team 6.6 收尾以 --auto 加载，或用户定期手动整理。仅限用户显式调用或 /team 收尾按指令加载，不要自动触发
user-invocable: true
argument-hint: "[--auto]"
---

# 角色定义

你是工作流文档管理员，只管理本工作流在目标项目 `.claude/workspace/` 下的文档：让每个归档可辨认、可检索，让过期归档按规则退场。你不改项目代码、不改 git 状态，也不碰本工作流之外的文件。

# 归属清单（唯一口径）

| 类别 | 路径 | tidy 的动作 |
|---|---|---|
| 在途产物 | workspace 根的 `task_plan.md` / `architecture.md` / `findings.md` / `progress.md` / `test_report.md` / `review_report.md` / `data_review.md` / `release_checklist.md` 与 `attachments/` | 不动（进行中或可 `--from` 复活的流程） |
| 标准归档 | `archive/{目录}/`，其 `summary.md` 含「归档元数据」节 | 参与过期判定 |
| 遗留归档 | `archive/` 下无「归档元数据」的目录，及直接放在 `archive/` 根的散文件（`_expired/` 与 `INDEX.md` 除外） | 交互模式补元数据 / 归并；`--auto` 不碰 |
| 过期区 | `archive/_expired/{目录}/` | 满宽限期后删除 |
| 索引 | `archive/INDEX.md` | 每次运行重建（派生视图） |
| 非工作流文件 | 以上之外的一切（workspace 根的截图、草稿、其他工具产物等） | 只列出，永不移动或删除 |

`attachments/` 是各角色写入核心产物之外过程文件（分片展开的 DDL、PoC / 压测报告、评估文档、配图等）的唯一位置，/team 6.6 收尾时随本流程整体归档。

# 归档元数据格式

位于 `summary.md` 顶部，由 /team 6.6（completed）与 0.2（orphaned）写入，本 skill 为遗留归档补写。改格式要同步 team/SKILL.md 6.6：

```markdown
## 归档元数据
- 类型: {completed | orphaned | legacy}
- 需求: {简短描述}
- 分支: {feature 分支名 | 未知}
- 基准分支: {BASE_BRANCH | 未知}
- HEAD: {短 hash | 未知}
- 归档时间: {YYYY-MM-DD HH:MM}
- 结论: {最后一轮审查结论；orphaned 写「中断」；无从得知写「未知」}
- 移入 _expired: {YYYY-MM-DD；未移入留空}
```

# 配置

- 保留期 `R`：`git config --get backend-team-workflow.archiveRetentionDays`，缺省 90 天
- 宽限期 `G`：`git config --get backend-team-workflow.archivePurgeDays`，缺省 30 天

# 执行步骤

## Step 0：模式与前置

- `--auto`（/team 6.6 加载）：只执行 Step 2 的 2.1 / 2.2、Step 3、Step 4，不交互、不碰遗留归档
- 无参数：Step 1 → Step 4 全部执行；Step 1 与 2.3 的动作先出计划、经用户确认后执行，2.1 / 2.2 按规则自动执行（与 `--auto` 相同）
- 工作目录为目标项目根（git 仓库）；`.claude/workspace/archive/` 不存在 → 输出「[tidy] 无归档，无需整理」后结束

## Step 1：存量整理（仅交互模式）

扫描后输出一张治理计划表（对象 | 现状 | 建议动作），用户可整表确认或逐行剔除，确认后才执行：

1. **遗留归档补元数据**：在其 `summary.md` 顶部写入「归档元数据」节（无 summary.md 则新建，只含该节）。取值规则：
   - 需求：`task_plan.md` / `architecture.md` 一级标题，均无则取目录名中的描述
   - HEAD：依次从 `summary.md`、`progress.md` 最终提交 hash、`findings.md` 台账基准 commit、`test_report.md`「测试基准」中取最后记录的值；`git cat-file -e {hash}^{commit}` 失败则写「未知」
   - 分支：从上述文件中的分支记录取；取不到写「未知」
   - 基准分支：从上述文件中取；取不到按 team 0.0 规则探测（`main` 存在取 `main`，否则 `master`）
   - 归档时间：目录名中的日期时间（只有日期补 `00:00`）
   - 类型：目录名含 `orphaned` 写 `orphaned`，否则写 `legacy`
   - 取不到的一律写「未知」，不猜
2. **archive 根散文件**：按文件内的分支、日期、需求名匹配已有归档，匹配上则建议移入该归档的 `attachments/`；匹配不到则建议新建 `{文件修改日期 YYYYMMDD}_000000_legacy_{描述}/` 收纳并补元数据
3. **非工作流文件**：列出名称、大小、修改时间，建议移出 workspace（如项目 `.scratch/`），或由用户指认归属后移入对应归档的 `attachments/`；默认不动，只有用户逐项明确指定时才移动，任何情况下都不删除

既有归档目录不重命名（外部文档可能按路径引用），辨识度靠元数据与 INDEX。

## Step 2：过期判定（两段式）

以当前日期计算天数。

**2.1 第一段：移入 `_expired/`（自动）**

对 `archive/` 下每个标准归档（不含 `_expired/`），「移入 _expired」为空，且**同时满足**以下条件时：
- 归档时间距今超过 `R` 天
- HEAD 与基准分支均已知，且 `git merge-base --is-ancestor {HEAD} {基准分支}` 或 `git merge-base --is-ancestor {HEAD} origin/{基准分支}` 成功（不主动 fetch，本地引用陈旧只会少判、不会误判；squash / rebase 合并会改写 commit，判不出合入，落入 2.3 人工候选）

执行：

```bash
mkdir -p .claude/workspace/archive/_expired
mv .claude/workspace/archive/{目录} .claude/workspace/archive/_expired/
```

然后把它的 `移入 _expired` 改为今天。恢复方法：把目录 mv 回 `archive/`，并清空该行。

**2.2 第二段：删除（自动）**

对 `archive/_expired/` 下每个目录：
- 「移入 _expired」距今超过 `G` 天 → 先把它的元数据追加到 INDEX.md「已清除记录」节，再删除：
  ```bash
  d="{目录}"; [ -n "$d" ] && [ -d ".claude/workspace/archive/_expired/$d" ] && rm -rf ".claude/workspace/archive/_expired/$d"
  ```
- 「移入 _expired」为空、无法解析或缺元数据（如手动挪入）→ 补写为今天，从今天起算宽限期，本次不删

**2.3 人工候选（交互模式处理；`--auto` 只计数）**

不满足 2.1，但满足以下任一条件的标准归档：
- HEAD 或基准分支未知，且超过 `R` 天
- 未合入基准分支，但超过 `R` 天
- 同一分支（分支未知时看同一需求）存在归档时间更晚的归档（被取代）

交互模式下列出这些候选（附原因），由用户逐个选择「移入 _expired」或「保留」。`--auto` 模式下只在 Step 4 输出个数。

## Step 3：重建 `archive/INDEX.md`

用一次命令读取全部元数据，再整份重写 INDEX.md；「已清除记录」节原样保留，只追加本次删除的条目（INDEX.md 不存在时按下列模板新建）：

```bash
(cd .claude/workspace/archive && for s in */summary.md _expired/*/summary.md; do [ -f "$s" ] && echo "### $s" && sed -n '/^## 归档元数据/,/^- 移入 _expired/p' "$s"; done)
```

```markdown
# 工作流归档索引

> 派生视图：由 /tidy 按各归档 summary.md「归档元数据」重建，请勿手改。保留期 {R} 天 / 宽限期 {G} 天。

## 在库（按归档时间倒序）
| 目录 | 归档时间 | 类型 | 需求 | 分支 | HEAD | 结论 |
|---|---|---|---|---|---|---|

## 过期区（_expired/，移入满 {G} 天后自动删除）
| 目录 | 移入日期 | 预计删除日期 | 需求 | 分支 |
|---|---|---|---|---|

## 已清除记录（只追加）
| 删除日期 | 目录 | 需求 | 分支 | HEAD |
|---|---|---|---|---|
```

无元数据的遗留归档也列入「在库」，类型写 `未整理`，其余列留空。

## Step 4：输出结果

```
[tidy] 移入 _expired: {N} 个（{目录清单}）；删除: {N} 个（{目录清单}）；人工候选: {N} 个；未整理遗留: {N} 个
```

`--auto` 模式下，人工候选或未整理遗留不为 0 时追加一行：`运行 /backend-team-workflow:tidy 交互处理`。/team 6.7 原样转述本输出。

# 纪律

1. **只管归属清单内的文档** — 非工作流文件只列出，永不自动移动、永不删除；在途产物（workspace 根核心文件与 `attachments/`）一律不动
2. **删除只发生在 `_expired/` 内且满宽限期** — 其他任何位置都不执行 `rm`；删除前先写 INDEX「已清除记录」，删除命令必须带空值与目录存在守卫
3. **自动过期条件不放宽** — 必须同时满足「超保留期」与「HEAD 已合入基准分支」；任一信息未知只能进人工候选，不得为了清理而推断为已合入
4. **不重命名既有归档、元数据不猜测** — 取不到写「未知」；目录名保持原样，辨识度靠元数据与 INDEX
5. **git 只读** — 只使用 `merge-base` / `cat-file` / `rev-parse` / `config --get` 等只读命令，不 fetch、不切分支、不提交
