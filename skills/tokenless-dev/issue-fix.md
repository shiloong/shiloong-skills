# Issue 修复流程

针对 `alibaba/anolisa` 仓库 tokenless 组件的自动化 Bug 修复。

## 输入

用户提供 issue 编号或 URL，提取 issue 编号后开始执行。

## Phase 1：环境验证

执行 [references/env-check.md](references/env-check.md) 中的全部步骤（**含**步骤 1.5-A Linux 校验 + 1.5-B Rust 工具链 + 1.6 `cargo fetch`）。跳过 1.5-C rpmbuild（fix 不打 RPM）。

如果 `cargo fetch` 拉取新依赖导致 `Cargo.lock` 变更，纳入后续 commit 一并提交。

## Phase 2 + 3.1：Issue 解析 & 仓库规范阅读（可并行）

Phase 1 通过后，使用 Task 工具同时启动 2 个子 agent：

**子 agent A — Issue 解析（Phase 2）**：

```bash
GH_PAGER="" gh issue view <number> --repo alibaba/anolisa --json title,body,labels,state,issueType,comments
```

执行准入检查（**任一不满足则整体中止**）：
- `state` == `OPEN`
- `labels` 包含 `component:tokenless`
- Issue 为 Bug 类型 — 满足其一即可：
  - `labels` 包含 `Bug`，或
  - `issueType.name` == `Bug`

从 issue 正文中提取：复现步骤、预期行为、实际行为、环境信息（特别留意 Anolis/RHEL 版本、cargo/rust 版本、是否走 RPM 安装还是 `make install`）。

从评论中提取（`comments` 数组）：
- 维护者提出的修复思路或提示
- 报告者补充的复现细节或上下文
- 讨论中达成的共识或决定

评论可能包含关键上下文 — 必须在代码分析前完整阅读。

**子 agent B — 仓库规范阅读（Phase 3.1）**：

阅读 [references/code-standards.md](references/code-standards.md) 中的规范阅读清单：
- `AGENT.md`（anolisa 根） — commit 格式（scope=tokenless 强制）、分支命名、PR 规范
- `src/tokenless/README.md` — 三适配 target 概览与构建系统
- `src/tokenless/CHANGELOG.md` — 最近发版的变更脉络
- `src/tokenless/Cargo.toml` + `Makefile` + `justfile` — 构建命令权威来源

两个子 agent 全部返回后，合并结果继续。

### 中止场景

| 条件 | 处理方式 |
|------|----------|
| Issue 不是 open 状态 | 中止：报告 issue 当前状态 |
| Issue 缺少 `component:tokenless` 标签 | 中止："不是 tokenless 组件的 issue" |
| Issue 不是 Bug（label 和 issueType 均不匹配） | 中止："不是 bug 类 issue" |

## Phase 3.2-3.4：代码搜索与根因分析（顺序执行）

依赖上方并行阶段的结果（issue 关键词 + 仓库规范）：

1. 使用 issue 中的关键词搜索 `src/tokenless/**`，重点区分以下分桶：
   - `crates/tokenless-cli/src/**`、`crates/tokenless-schema/src/**`、`crates/tokenless-stats/src/**`（核心逻辑）
   - `adapters/tokenless/common/hooks/**`（hook 脚本）
   - `adapters/tokenless/{common,openclaw,hermes}/`（manifest 与适配脚本）
   - `adapters/tokenless/common/tool-ready-spec.json` + `tokenless-env-fix.sh`（Tool-Ready）
   - `third_party/patches/**`（vendored 修补）
   - `Makefile` / `justfile` / `tokenless.spec.in`（构建系统）
2. 从复现步骤出发，追踪调用链，定位根因（Rust 侧用 `cargo doc --open` 或 `cargo expand` 辅助，hook 侧用模拟 input JSON 复现）。
3. 确定：需要修改的文件、相关的测试文件（`crates/*/tests/**`、`tests/run-all-tests.sh` 或新建 hook 测试）。

## Phase 4：可自动化修复门控

仅评估"该 bug 是否适合 AI 自动修复"。不满足则**停止并输出分析报告**，不继续执行。

**条件 A — 可无头验证**：修复可通过 `cargo test` 断言或 `tests/run-all-tests.sh` 的 bash 断言，无需手动终端交互。若 bug 涉及交互式 TTY、网络外部依赖、RPM 安装后才能复现（需要 rpmbuild 全链路）的部分，则停止。

**条件 B — 方案唯一**：根因明确且不存在多种可行修复方案。若存在不确定性或多方案选择（如 schema 列名是否要重命名、是否新增 cargo feature flag），则停止并展示各方案由用户拍板。

> **副作用 / 公共 API 影响 / 测试覆盖**等检查，延后到 Phase 6 步骤 0 对照 [references/tokenless-concerns.md](references/tokenless-concerns.md) 的核心四维 + 横切触发表逐条核对，避免在此重复评估。

## Phase 5：创建分支

遵循 AGENT.md 的分支命名规范：

```
fix/tokenless/<issue-number>-<short-desc>
```

示例：`fix/tokenless/231-stats-migration-rollback`

`short-desc` 生成规则：从 issue title 提取 2-4 个关键词，转为 kebab-case，总长度不超过 30 字符。去除冠词、介词等虚词。

```bash
git checkout -b fix/tokenless/<issue-number>-<short-desc>
```

## Phase 6：实施修复

**步骤 0：编码前自检** — 阅读 [references/tokenless-concerns.md](references/tokenless-concerns.md)，对照"快速自检流程"：

1. 列出 Phase 3 定位到的将要改动文件路径集合
2. 与文件中"横切关注点触发表"的触发关键词匹配（重点关注：#1 三适配 target 同步、#2 SQLite 迁移、#3 Tool-Ready spec、#4 FHS 路径、#6 rtk/toon 版本对齐、#10 fail-open、#13 CHANGELOG/spec changelog 双写）
3. 命中的每一维度在编码时必须落实（例：命中 #1 hook 改动 → 三适配 target 同步检查；命中 #2 → migration + 升级测试同步写）

自检产出不要求成文，目的是把"评审侧会发现的问题"提前到编码侧消化。

**步骤 1：编写修复代码** — 遵循 [references/code-standards.md](references/code-standards.md) 中的 Rust + adapter 脚本规范。

**步骤 2：处理版权头** — 遵循 [references/code-standards.md](references/code-standards.md) 中的版权头规则（Apache-2.0 SPDX + Alibaba Cloud；`third_party/rtk/` 例外）。

**步骤 3：新增或更新测试** — 覆盖修复后的代码路径。
- Rust 侧：`crates/*/src/**` 内 `#[cfg(test)] mod tests` 或 `crates/*/tests/*.rs`
- Hook 侧：在 `tests/` 下添加新的 `.sh` 测试用例，并在 `run-all-tests.sh` 中注册

## Phase 7：验证

执行 [references/quality-check.md](references/quality-check.md) 中的**分步模式**，最多 3 轮重试。

> 若本次改动涉及 `adapters/tokenless/common/hooks/**` 或 `tests/`，分步模式的步骤 6（`bash tests/run-all-tests.sh`）**必须**执行。

## Phase 8：提交 & 创建 PR

**暂存变更**（仅限 tokenless 范围内的文件）：
```bash
git add src/tokenless/
```

**Commit message**（遵循 [references/pr-creation.md](references/pr-creation.md) 中的 Commit 规范）：
```
fix(tokenless): <小写描述，无句号>

Closes #<issue-number>
```

**Push & PR**：执行 [references/pr-creation.md](references/pr-creation.md) 中的 **Fork 模式**。调用前先导出临时文件所需的变量：

```bash
export FLOW=fix
export TARGET=<issue-number>
```

PR 标题遵循 commit 格式：`fix(tokenless): <description>`。

PR 模板填写要点：
- Type of Change：勾选 `Bug fix`
- Scope：勾选 `tokenless`
- Related Issue：`closes #<issue-number>`
- Testing 部分：
  ```bash
  cd src/tokenless
  make fmt && cargo build --release && \
    cargo clippy -p tokenless-cli -p tokenless-schema -p tokenless-stats --all-targets -- -D warnings && \
    cargo test -p tokenless-cli -p tokenless-schema -p tokenless-stats
  # <测试结果摘要 + 涉及 hook 时附 tests/run-all-tests.sh 结果>
  ```

完成后输出 PR URL。
