# Review 结论结构化与提交规范

review 子流程专属的结论组织、证据校准、脏代码防护、GitHub 提交模板与硬门清单。

## 一、YAML Claim 结构化规范

### 1.1 为什么强制 YAML

散文式结论无法机械校验证据是否完备；结构化输出可以让"有没有 file:line 证据"、"Severity / Confidence 是否匹配"被自动检查，从源头杜绝臆测型评论。

### 1.2 Claim Schema

Phase 4 每个分析子 agent 产出的每条结论**必须**按此格式：

```yaml
- id: B-001                                # <dimension>-<seq>，全局唯一
  dimension: "B-新增逻辑正确性与隐患"        # 四维或横切维度名
  claim: "schema migration 缺少回滚路径"     # 一句话断言，不超过 40 字
  severity: Blocker                        # Blocker / Major / Minor / Info
  confidence: Speculative                  # 初始固定为 Speculative，由 Phase 5 更新
  evidence:                                # 至少一条；路径相对 src/tokenless/
    - path: crates/tokenless-stats/src/migrations.rs
      line: 42
      quote: "conn.execute(\"ALTER TABLE stats ADD COLUMN before_output INT\", [])?;"
  counter_hypothesis:                      # 至少一条反假设，每条带可执行的 verify
    - desc: "migration 模块已有 down 函数处理回滚"
      verify:
        action: grep_code                  # read_file / grep_code / run
        target: crates/tokenless-stats/src/
        look_for: "fn (down|rollback)\\("
  verification_tier_reached: null          # 由 Phase 5 填 tier-1/2/3/4
  verified_by: []                          # 填 file:line 或命令输出摘要
  final_action: null                       # 由 Phase 5 决策：keep / drop / downgrade
```

### 1.3 字段校验

| 字段 | 必填 | 校验 |
|------|------|------|
| `id` | ✅ | 形如 `<A|B|C|D|X1..X14|E1..E6>-<seq>` |
| `claim` | ✅ | 非空，≤ 40 字 |
| `severity` | ✅ | 枚举 |
| `confidence` | ✅ | 初始 `Speculative` |
| `evidence` | ✅ | ≥ 1 条，每条含 `path` + `line` + `quote`；路径存在性、quote 与代码文本一致性必须被验证 |
| `counter_hypothesis` | ✅ | ≥ 1 条；至少一条必须有可执行的 `verify` |

**缺字段的直接处置**：

- 缺 `evidence` → 自动标记 `confidence: Speculative` 并降级为 `Info`
- 缺 `counter_hypothesis` → 进入 Phase 5 前强制补齐，否则 drop

---

## 二、Severity × Confidence 分级门槛

Phase 5 完成后，每条 claim 的 `(severity, confidence)` 决定**评论动作**：

| | Verified | Likely | Speculative |
|-|----------|--------|-------------|
| **Blocker** | → 发 line comment + 计入 REQUEST_CHANGES 考虑 | → 发 line comment，前缀 "⚠️" | **禁止发**，必须降级或更多验证 |
| **Major** | → 发 line comment | → 发 line comment，前缀 "[待作者确认]" | → 降级为 Minor 或合并进 overall summary |
| **Minor** | → 发 line comment | → 合并进 overall summary | → 丢弃 |
| **Info** | → 发 line comment（scope 提醒、建议） | → 合并进 overall summary | → 丢弃 |

**Event 类型决策**：

- 任一 `Blocker × Verified` → `REQUEST_CHANGES`
- 仅有 `Major/Minor/Info` → `COMMENT`
- 全部 Verified 且无 Blocker/Major → 可考虑 `APPROVE`（由 Phase 7 交用户拍板）

---

## 三、四层反证 Pipeline

对 Phase 4 产出的每条 claim，按以下顺序执行反证，直到 confidence 确定或降级。

### Tier-1 静态代码反证（必跑）

执行 claim 中每条 `counter_hypothesis.verify`：
- `action: read_file` → 读取目标文件相关段落
- `action: grep_code` → 全仓库搜索模式
- 读取 claim evidence 所涉函数的被调用点、类型定义、测试文件

**结果处置**：
- 证据**支持** claim，反假设被逐一排除 → `confidence: Verified`
- 证据**推翻** claim → `final_action: drop`
- 证据不足 → 进入 Tier-2（仅当 `severity ∈ {Blocker, Major}`）

### Tier-2 编译器反证（条件触发）

触发条件：`confidence: Speculative` 且 `severity ∈ {Blocker, Major}` 且 claim 属于"类型/签名/导入/clippy 规则"类。

```bash
# 类型/签名类
cd src/tokenless && cargo check -p tokenless-cli -p tokenless-schema -p tokenless-stats

# 缩小到具体 crate
cd src/tokenless && cargo check -p <crate-name>

# clippy 规则类（如 unwrap、shadowing、type complexity）
cd src/tokenless && cargo clippy -p tokenless-cli -p tokenless-schema -p tokenless-stats --all-targets -- -D warnings
```

读 cargo 输出中**与 claim 相关的**诊断：
- claim 断言"类型不匹配" → cargo 无相关诊断 → 反驳 claim → `drop`
- cargo 确有对应诊断 → `confidence: Verified`，补充 `verified_by: ["cargo: <diagnostic>"]`
- cargo 无相关诊断但 claim 属行为类 → 进入 Tier-3

### Tier-3 运行时反证（严格触发，见第四节脏代码防护）

触发条件：**全部满足**才进入：
- `severity == Blocker`
- `confidence == Speculative`
- Tier-1 和 Tier-2 都未能定论
- claim 属于"运行态才能暴露"的类别（SQLite 并发、tokio/rayon 异步生命周期、hook 脚本副作用、FHS 路径在 RPM 包内的实际生效情况等）

### Tier-4 降级

仍为 Speculative 时：
- `severity` 降一档：Blocker → Major → Minor → Info
- 评论 body 前缀加 `[待作者确认]`
- overall summary 中不引用该 claim
- `final_action: downgrade`

---

## 四、Tier-3 运行时反证：脏代码防护流程

**核心原则**：反例代码**绝不**进入 PR 作者的分支，也**绝不**进入 tokenless 源码树。

### 4.1 触发判定（必须全部满足）

```
severity == Blocker
&& confidence == Speculative
&& Tier-1 结果 == 不定论
&& Tier-2 结果 == 不定论 (或 N/A)
&& claim.dimension 属于 {SQLite 并发, 异步生命周期, hook 副作用顺序, FHS 安装路径实测}
```

任一条件不满足 → 跳过 Tier-3，直接进入 Tier-4 降级。

### 4.2 隔离位置

```bash
WS="$(git rev-parse --show-toplevel)"
SCRATCH="$WS/.qoder/tokenless-dev/review-${PR_NUMBER}-scratch.XXXXXX"
mkdir -p "$(dirname "$SCRATCH")"
SCRATCH=$(mktemp -d "$SCRATCH")
```

- `.qoder/tokenless-dev/` 已被根 `.gitignore` 覆盖，不会污染 git
- 每次运行独立目录，`XXXXXX` 保证并发/重试隔离

### 4.3 反例文件命名约束

- 文件名**必须**以 `__review_scratch__` 开头（便于 `grep -r` 审计残留）
- 文件名**不得**以 `.rs` 放在 `crates/*/src/` 或 `crates/*/tests/` 内，避免被 `cargo test` 自动发现
- 首行**必须**带 SCRATCH 注释标记：

```rust
// REVIEW SCRATCH — DO NOT COMMIT. claim-id: B-001. pr: #<N>. ts: <ISO>.
```

```bash
#!/bin/bash
# REVIEW SCRATCH — DO NOT COMMIT. claim-id: X3-001. pr: #<N>. ts: <ISO>.
```

### 4.4 执行方式

按反例性质选择：

**A. Rust 独立可执行（首选，零依赖 cargo discovery）**：

```bash
cd "$SCRATCH"
# 复制相关 crate 源码 + 编写 main.rs（或独立 example）
cargo init --bin __review_scratch__B-001
# 在新 manifest 中以 path = "<absolute path to crate>" 引用 tokenless-schema 等依赖
cargo run --release
```

**B. Hook 脚本验证（Python / Shell）**：

```bash
cd "$SCRATCH"
# 模拟 hook input JSON，调用真实 hook 脚本
cat > __review_scratch__X10-001.input.json <<'EOF'
{"tool_name": "Shell", "tool_input": {"command": "ls"}}
EOF
cat __review_scratch__X10-001.input.json | \
  python3 "$WS/src/tokenless/adapters/tokenless/common/hooks/rewrite_hook.py"
```

**C. RPM 安装路径实测**（极少触发，需要本地有 rpmbuild + 沙盒环境）：

仅在 claim 涉及 spec.in `%files` 段或 `%post` scriptlet 时考虑。建议改用 Tier-1 静态分析 + 引用历史 RPM 验证记录，避免触发完整 rpmbuild。

**禁止**：
- 在 `crates/*/src/` 或 `crates/*/tests/` 下创建带 `#[test]` 标记的反例（会被 `cargo test` 自动发现）
- 对 tokenless 源码做任何修改（即便是"为了让反例跑通"也不行，这会污染 git working tree）
- 在 `~/rpmbuild/` 下直接构建反例 RPM（污染共享构建环境）

### 4.5 结果归档

```yaml
verification_tier_reached: tier-3
verified_by:
  - "scratch __review_scratch__B-001/main.rs output: SQLite locked under 4 concurrent writers, retry stalls 5s"
```

**可选增值**：将反例代码**改写**为面向作者的"建议补充的正式单测"，附在评论 body 中。**不直接提交**反例代码到 PR。

### 4.6 清理硬门（Phase 8 提交评论前执行）

```bash
# 1. 删除所有 scratch 目录
rm -rf "$WS/.qoder/tokenless-dev/review-${PR_NUMBER}-scratch."*

# 2. 验证无遗留
find "$WS/.qoder/tokenless-dev/" -name "__review_scratch__*" -print | grep . && {
  echo "ERROR: 存在未清理的反例文件"; exit 1;
}

# 3. 验证 src/tokenless/ 无改动
test -z "$(git status --porcelain -- src/tokenless/)" || {
  echo "ERROR: src/tokenless/ 有未预期改动"
  git status --porcelain -- src/tokenless/
  exit 1;
}

# 4. 验证 tokenless 代码未被 stash 或 branch 切换影响
git diff --stat -- src/tokenless/ | tee /dev/stderr | grep . && exit 1 || true
```

任一步骤失败 → 中止 Phase 8，输出残留路径给用户，等待手动确认。

---

## 五、GitHub Review 提交模板

### 5.1 获取 commit_id

```bash
HEAD_SHA=$(GH_PAGER="" gh pr view "$PR_NUMBER" --repo alibaba/anolisa --json headRefOid --jq '.headRefOid')
```

### 5.2 JSON payload 结构

```json
{
  "commit_id": "<HEAD_SHA>",
  "event": "COMMENT",
  "body": "<overall summary, markdown, 只引用 Verified claim>",
  "comments": [
    {
      "path": "src/tokenless/crates/tokenless-stats/src/migrations.rs",
      "line": 42,
      "side": "RIGHT",
      "body": "🔴 **migration 缺少 down 路径**\n\n本次新增的 `ALTER TABLE stats ADD COLUMN before_output INT` 没有配套的回滚/降级处理。在 0.3.1-2 已有 schema 演进先例，建议参照 ...（具体证据 + 建议）"
    }
  ]
}
```

### 5.3 line 必须落在 diff 内（422 防护）

GitHub API 要求 `line` 必须是**当前 diff 中出现过的行号**（新增行 side=RIGHT 或上下文行）。踩过的坑：

- ✅ 新增行（diff 中 `+` 开头的行）→ 合法
- ✅ 上下文行（diff 中无 `+`/`-` 的行）→ 合法
- ❌ diff 之外的行（评论想引用新代码的下游影响行）→ 返回 `422 Line could not be resolved`

**应对策略**：

1. 优先把 line 定位到**最近的新增行**
2. 若必须引用 diff 外的行，评论正文中用文字描述：`"This affects line 699 (in the final block), which is not in the diff."`
3. 提交前用脚本预校验（见 5.5）

### 5.4 payload 文件管理

```bash
WS="$(git rev-parse --show-toplevel)"
mkdir -p "$WS/.qoder/tokenless-dev"
PAYLOAD="$(mktemp "$WS/.qoder/tokenless-dev/review-${PR_NUMBER}.XXXXXX.json")"

# 生成 payload 内容到 $PAYLOAD
# ...

GH_PAGER="" gh api --method POST \
  "/repos/alibaba/anolisa/pulls/${PR_NUMBER}/reviews" \
  --input "$PAYLOAD"

# 成功后清理
rm -f "$PAYLOAD"
```

### 5.5 提交前预校验（可选但推荐）

```bash
# 从 gh pr diff 中提取所有"可评论"的行号（新增行 + 上下文行）
GH_PAGER="" gh pr diff "$PR_NUMBER" --repo alibaba/anolisa > "$WS/.qoder/tokenless-dev/review-${PR_NUMBER}.diff.txt"

# 对每条 comment 的 (path, line) 在 diff 文件里做存在性校验
# 若有任一 line 不在 diff → 修正（挪到相邻 diff 行）后再提交
```

---

## 六、证据完整性硬门 Checklist

Phase 8 提交评论前**必须**逐项确认，任一未过 → 回退到 Phase 5 补齐：

```
☐ 每条 claim 的 evidence 非空，每条 evidence.path 存在、quote 与源码一致
☐ 每条 severity == Blocker 的 claim 的 confidence == Verified
☐ 每条 confidence == Speculative 已 drop 或 downgrade（Tier-4）
☐ 每条 counter_hypothesis 都已被 Tier-1~3 显式反驳或确认（verified_by 非空）
☐ overall body 只引用 confidence == Verified 的 claim
☐ 所有 comments[].line 经 5.5 预校验落在 diff 内
☐ commit_id == 当前 PR headRefOid（不是过时的）
☐ event 类型符合 Severity × Confidence 分级（二节规则）
☐ .qoder/tokenless-dev/ 下无 __review_scratch__* 残留
☐ git status --porcelain -- src/tokenless/ 为空
☐ payload 临时文件在提交成功后已 rm
```

这份 checklist 对应 review.md 的 Phase 8 第一步。
