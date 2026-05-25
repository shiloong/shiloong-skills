# PR Review 流程

针对 `alibaba/anolisa` 仓库 tokenless 组件已提交 PR 的结构化代码评审与评论回写。

## 输入

用户提供 PR 编号或 URL，支持：
- 编号：`290`
- 完整 URL：`https://github.com/alibaba/anolisa/pull/290`

提取 PR 编号后开始执行。

## 流程概览

| Phase | 动作 | 可并行 |
|-------|------|--------|
| 1 | 输入解析 & 环境验证 | 部分 |
| 2 | 规范准备 + PR 物料拉取 | ✅ |
| 3 | 信息采集（文件分桶） | — |
| 4 | 四维 + 横切 + 纪律分析 | ✅ |
| 5 | 四层反证 pipeline | 部分 |
| 6 | 结论分级与合入条件 | — |
| 7 | 与用户二次确认 | — |
| 8 | 提交 line-level review | — |

**重要特性**：review 是**纯只读流程**，不得修改 `src/tokenless/` 任何代码；临时文件位于 `<workspace>/.qoder/tokenless-dev/`，全流程结束必须清理干净。review 跨平台允许运行（不强制 Linux），但 Tier-2 Rust 编译反证仅在有 cargo 工具链的环境下可用。

---

## Phase 1：输入解析 & 环境验证

### 1.1 解析 PR 编号

从用户输入提取纯数字 `PR_NUMBER`。

### 1.2 环境验证

执行 [references/env-check.md](references/env-check.md) 调用模式矩阵中 `review` 列：仅 1.1-A (git remote) + 1.1-B (gh auth)，其余步骤跳过（只读流程允许脏工作区；不强制 Linux 平台 — Tier-2 编译反证若需要 cargo 而平台不支持，可在 Tier-2 触发时再提示）。

```bash
# 仅这两项并行
GH_PAGER="" gh auth status               # 必须已认证
git remote -v | grep alibaba/anolisa    # 必须在 anolisa 仓库
```

### 1.3 获取 PR 元数据

```bash
GH_PAGER="" gh pr view "$PR_NUMBER" --repo alibaba/anolisa \
  --json title,body,author,state,files,additions,deletions,baseRefName,headRefName,headRefOid,commits,url,labels
```

### 1.4 准入检查（任一不满足则中止）

| 条件 | 失败处理 |
|------|---------|
| `state` == `OPEN` | 中止：报告当前状态 |
| `files[].path` 至少一条落在 `src/tokenless/**` | 中止："PR 未涉及 tokenless 组件" |
| PR 不是 draft，或 draft 经用户确认评审 | 若 draft：使用 [interaction-guide.md](references/interaction-guide.md) 询问用户是否继续 |

保留以下变量供后续使用：`PR_NUMBER`、`HEAD_SHA`（= `headRefOid`）、`PR_TITLE`、`PR_BODY`、`FILES`（路径数组）。

---

## Phase 2：规范准备 + PR 物料拉取（并行）

使用 Task 工具同时启动 2 个子 agent：

**子 agent A — 规范与关注点清单阅读**：

- [references/tokenless-concerns.md](references/tokenless-concerns.md) — 核心四维、14 条横切触发、工程纪律
- [references/code-standards.md](references/code-standards.md) — Rust 规则、版权头、adapter 脚本规范
- [references/pr-creation.md](references/pr-creation.md)（仅取 Commit 规范、PR 模板要求）
- `AGENT.md`（仓库根） — commit/branch/PR 约束（scope=tokenless 强制）

**子 agent B — PR 物料拉取**：

```bash
# 拉 PR 分支到本地 ref，仅作为 `git show pr-${PR_NUMBER}:<path>` 的引用源；不要对此 ref 执行 checkout / switch
git fetch origin "pull/${PR_NUMBER}/head:pr-${PR_NUMBER}" --force

# 拉完整 diff 到临时文件
WS="$(git rev-parse --show-toplevel)"
mkdir -p "$WS/.qoder/tokenless-dev"
DIFF_FILE="$(mktemp "$WS/.qoder/tokenless-dev/review-${PR_NUMBER}.XXXXXX.diff")"
GH_PAGER="" gh pr diff "$PR_NUMBER" --repo alibaba/anolisa > "$DIFF_FILE"
```

两个子 agent 全部返回后，合并结果继续。

---

## Phase 3：信息采集（文件分桶）

按 diff 文件路径将 `FILES` 分桶：

| 桶 | 路径模式 | 在 Phase 4 的用途 |
|----|---------|-------------------|
| **核心逻辑（Rust）** | `crates/*/src/**/*.rs`（非 test） | 维度 A/B 分析主体 |
| **单元测试** | `crates/*/src/**/*tests*.rs` 或 `crates/*/tests/**/*.rs` | 维度 C 分析主体 |
| **公共 API** | `crates/tokenless-schema/src/lib.rs`、`crates/*/src/lib.rs` 中 `pub` 导出 | 维度 A + 横切 #11 |
| **SQLite 迁移** | `crates/tokenless-stats/**` 中 `.sql` / migrations / `CREATE TABLE` / `ALTER TABLE` | 横切 #2 |
| **Hook 脚本** | `adapters/tokenless/common/hooks/*.{py,sh}` | 横切 #1 / #10 |
| **Adapter manifest** | `adapters/tokenless/{manifest.json,common/cosh-extension.json,openclaw/openclaw.plugin.json,hermes/plugin.yaml}` | 横切 #1 / #7 / #8 / #9 |
| **Tool-Ready spec** | `adapters/tokenless/common/tool-ready-spec.json` / `tokenless-env-fix.sh` | 横切 #3 |
| **Patches** | `third_party/patches/*.patch` | 横切 #5（标红，第三方 monkey patch） |
| **构建系统** | `Makefile` / `justfile` / `Cargo.toml` (`[workspace]`) | 横切 #6 / #12 |
| **RPM spec** | `tokenless.spec.in` | 横切 #4 / #13 |
| **文档/CHANGELOG** | `*.md` / `src/tokenless/CHANGELOG.md` | 工程纪律 #3 |
| **配置/lockfile** | `Cargo.lock` | 横切 #6 / 工程纪律 #1 |

**diff 之外的代码读取**：Phase 4/5 中若需验证调用链、类型签名、被调用函数行为，按以下只读命令获取，**严禁** `git checkout` / `git switch` / `git restore --source` / `git apply` / `git cherry-pick` 等任何写工作区的操作：

- 读工作区当前文件 → Read 工具 / Grep
- 读 PR HEAD 版本文件 → `git show pr-${PR_NUMBER}:<path>`
- 读 base..PR 差异 → `git diff origin/main..pr-${PR_NUMBER} -- <path>` 或 `gh pr diff ${PR_NUMBER}`

---

## Phase 4：四维 + 横切 + 纪律分析（并行）

输出**结构化 YAML claim 数组**，schema 严格遵循 [references/review-submit.md](references/review-submit.md) §1.2。

### 4.1 触发匹配

根据 Phase 3 的分桶结果 + diff 关键词，对照 [tokenless-concerns.md](references/tokenless-concerns.md) §横切关注点触发表，确定**本 PR 命中的横切维度**子集。未命中的维度可跳过。

### 4.2 并行分析（子 agent 数量 = 4 + 命中的横切维度数）

使用 Task 工具同时启动多个子 agent，每个 agent 负责一个维度：

- **子 agent A** — 维度 A（对现有功能的影响）
- **子 agent B** — 维度 B（新增逻辑正确性与隐患）
- **子 agent C** — 维度 C（测试覆盖）
- **子 agent D** — 维度 D（规范符合度）
- **子 agent X-n** — 每个命中的横切维度（#1~#14 中的命中项）
- **子 agent E** — 工程纪律 6 条

每个 agent 的产出格式：

```yaml
# agent 输出示例
claims:
  - id: B-001
    dimension: "B-新增逻辑正确性与隐患"
    claim: "..."
    severity: ...
    confidence: Speculative
    evidence: [...]
    counter_hypothesis: [...]
    verification_tier_reached: null
    verified_by: []
    final_action: null
```

### 4.3 合并与去重

所有 agent 返回后合并 `claims` 数组。同一 (path, line, dimension) 下的重复 claim 合并为一条，保留最严厉的 `severity`。

---

## Phase 5：四层反证 Pipeline

对 Phase 4 合并后的每条 claim，按 [review-submit.md §三](references/review-submit.md) 的 Tier-1 → Tier-2 → Tier-3 → Tier-4 顺序执行。

### 5.1 Tier-1 静态反证（并行）

将 claims 按 `counter_hypothesis.verify.action` 分组（read_file / grep_code），并行执行。每条 claim 的 verify 完成后更新：

- `confidence`: Verified / 仍 Speculative
- `verified_by`: 填入证据
- `final_action`: drop / keep / 进入 Tier-2

### 5.2 Tier-2 编译器反证（按需）

仅当存在 `confidence: Speculative && severity ∈ {Blocker, Major}` 且属类型/签名/clippy 规则类的 claim 时，运行 cargo 反证：

```bash
WS="$(git rev-parse --show-toplevel)"
CARGO_LOG="$(mktemp "$WS/.qoder/tokenless-dev/review-${PR_NUMBER}.XXXXXX.cargo.log")"

cd src/tokenless
# 类型/签名 → cargo check
cargo check -p tokenless-cli -p tokenless-schema -p tokenless-stats 2>&1 | tee "$CARGO_LOG"

# clippy 规则 → cargo clippy
cargo clippy -p tokenless-cli -p tokenless-schema -p tokenless-stats --all-targets -- -D warnings 2>&1 | tee -a "$CARGO_LOG"
```

将 cargo 诊断与相关 claim 交叉比对，更新 `confidence` 和 `verified_by`。

> **跨平台例外**：若当前不在 Linux 平台或 cargo 工具链不可用，Tier-2 改为：跳过运行，记录 `verified_by: ["tier-2 skipped: <reason>"]` 并直接进入 Tier-3 触发判定。这种情况下 Speculative 的 Blocker/Major 大概率会走 Tier-4 降级。

### 5.3 Tier-3 运行时反证（严格触发，脏代码防护）

**仅对同时满足**以下条件的 claim 执行，严格按 [review-submit.md §四](references/review-submit.md) 的隔离 / 命名 / 执行 / 清理 流程：

- `severity == Blocker`
- `confidence == Speculative`
- Tier-1/2 无定论
- `dimension ∈ {SQLite 并发, 异步生命周期, Hook 副作用顺序, FHS 安装路径实测}`

反例代码一律落在 `<workspace>/.qoder/tokenless-dev/review-${PR_NUMBER}-scratch.XXXXXX/`。

**结束前必须跑清理硬门**（review-submit.md §4.6）。

### 5.4 Tier-4 降级

未通过 Tier-3 的 Speculative Blocker → 降级为 Major + 前缀 `[待作者确认]`，或更低。

---

## Phase 6：结论分级与合入条件

### 6.1 分级统计

按 Severity × Confidence 矩阵 ([review-submit.md §二](references/review-submit.md)) 汇总：

| | Verified | Likely | Speculative |
|-|----------|--------|-------------|
| Blocker | 计数 | 计数 | 必须为 0 |
| Major | 计数 | 计数 | 计数 |
| Minor | ... | ... | ... |
| Info | ... | ... | ... |

### 6.2 合入条件清单

根据分级结果，生成明确的合入条件列表：

```
合入前必须满足（🔴 Blocker × Verified）：
1. <claim summary 1>
2. <claim summary 2>

建议修复（🟡 Major × {Verified, Likely}）：
1. ...

可选说明（ℹ️ Info / Minor）：
1. ...
```

### 6.3 Event 类型初判

按**单向优先级**（自上而下，首条命中即停）：

1. 存在 `Blocker × Verified` → `REQUEST_CHANGES`
2. 全部 claim `confidence == Verified` 且**无** Blocker / Major → `APPROVE`（候选，Phase 7 由用户拍板）
3. 其他所有情况 → `COMMENT`

三分支互斥完备，与 [review-submit.md §二](references/review-submit.md) 保持一致。

---

## Phase 7：与用户二次确认

展示给用户：
1. **总体结论**（分级统计 + 合入条件清单）
2. **评论草案**（按优先级列出每条将要发出的 line comment 摘要）
3. **Event 类型**（初判值）

使用 [references/interaction-guide.md](references/interaction-guide.md) 的原则：**先正文呈现背景，再用极简 AskUserQuestion 提问**。

收集用户反馈：

- **项目级豁免**（例："本次 schema 迁移已与维护者口头确认无需 down 路径" → 从评论中剔除该类）
- **严重度调整**（用户可升/降级特定 claim）
- **Event 类型确认**（COMMENT / REQUEST_CHANGES / APPROVE）
- **整体取消**（用户可终止提交）

用户确认后，根据反馈更新 claims 的 `final_action` 和 event 类型。

---

## Phase 8：提交 line-level Review

### 8.1 硬门 Checklist

执行 [review-submit.md §六](references/review-submit.md) 的完整 checklist。**任一未过 → 回退到 Phase 5 补齐**。重点验证：

```bash
WS="$(git rev-parse --show-toplevel)"

# 1. 无 scratch 残留
find "$WS/.qoder/tokenless-dev/" -name "__review_scratch__*" -print | grep . && {
  echo "ERROR: scratch 残留"; exit 1;
}

# 2. tokenless 源码树无改动
test -z "$(git status --porcelain -- src/tokenless/)" || {
  echo "ERROR: src/tokenless/ 有未预期改动"
  git status --porcelain -- src/tokenless/
  exit 1;
}

# 3. 所有 comments[].line 在 diff 内（见 review-submit.md §5.5）
```

### 8.2 构造 payload

按 [review-submit.md §五](references/review-submit.md) 的 JSON 结构。overall body 用 Markdown 编写：

```
## Summary

<一句话总评>

### 🔴 Blockers
- <file>:<line> — <claim>

### 🟡 Major
- ...

### ℹ️ Info
- ...

### Scope 说明
<若有 scope creep 相关说明>
```

写入 payload 文件：

```bash
PAYLOAD="$(mktemp "$WS/.qoder/tokenless-dev/review-${PR_NUMBER}.XXXXXX.json")"
# 生成 JSON 到 $PAYLOAD
```

### 8.3 提交

```bash
RESP_FILE="$(mktemp "$WS/.qoder/tokenless-dev/review-${PR_NUMBER}.XXXXXX.resp.json")"
GH_PAGER="" gh api --method POST \
  "/repos/alibaba/anolisa/pulls/${PR_NUMBER}/reviews" \
  --input "$PAYLOAD" > "$RESP_FILE"
```

**422 处理**：若返回 `Line could not be resolved`：
- 读取 `$DIFF_FILE`，定位每条出错 line 的最近合法行（新增行或上下文行）
- 修正 payload 后重试
- 最多重试 2 次，仍失败 → 向用户报告具体 comment，让其人工定夺

### 8.4 成功后清理

```bash
# 读取 review URL（在清理前）
REVIEW_URL=$(jq -r '.html_url' "$RESP_FILE" 2>/dev/null)

# 删除本次运行的所有临时文件（payload / diff / cargo.log / resp.json；scratch 已在 Phase 5.3 清理）
rm -f "$WS/.qoder/tokenless-dev/review-${PR_NUMBER}."*

echo "Review submitted: $REVIEW_URL"
```

### 8.5 失败或中止的保留

若 Phase 8 任何步骤失败，或用户在 Phase 7 中止：**保留** `.qoder/tokenless-dev/` 下的相关文件，输出路径给用户便于排查。不自动清理。

---

## 中止场景汇总

| 条件 | 阶段 | 处理方式 |
|------|------|---------|
| PR state != OPEN | 1.4 | 中止：报告状态 |
| PR 未涉及 tokenless | 1.4 | 中止："不是 tokenless 组件的 PR" |
| draft 且用户拒绝评审 | 1.4 | 中止 |
| gh 未认证 | 1.2 | 中止：提示 `gh auth login` |
| Phase 8 硬门失败 | 8.1 | 回退 Phase 5 补齐；多次失败 → 人工介入 |
| Phase 8 提交 422 且重试耗尽 | 8.3 | 人工定夺：保留 payload，让用户手工调整或放弃 |
| 用户在 Phase 7 拒绝提交 | 7 | 终止：保留 `.qoder/tokenless-dev/` 下产物 |
