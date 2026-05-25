# 版本发布流程

tokenless (Rust) 组件的版本发布工作流。

> **交互原则**：CHANGELOG 审核是核心人工环节，必须展示草稿让用户确认后才写入文件（同时写两份 — `CHANGELOG.md` 与 `tokenless.spec.in` `%changelog` 段）。其他步骤如遇异常也需暂停等待用户决策。

## 输入

- `/tokenless-dev release <version>` — 指定目标版本号
- `/tokenless-dev release` — 不指定版本号，自动分析并建议

## Phase 1：环境验证 + 版本确定

### 步骤 1.1-1.5：环境验证

执行 [references/env-check.md](references/env-check.md) 中 release 列对应的步骤（git remote + gh auth + workspace clean + sync main + Linux + Rust toolchain；跳过 1.5-C rpmbuild、跳过 1.6 cargo fetch — 由 Phase 4 / 5 按需触发）。

### 步骤 1.6：确定版本号

读取当前 workspace 版本：
```bash
grep -E '^version[[:space:]]*=' src/tokenless/Cargo.toml | head -1 | sed 's/.*"\([^"]*\)".*/\1/'
```

> 注意是 `[workspace.package].version`；crate 自己的 `Cargo.toml` 用 `version.workspace = true` 继承，不要从单 crate 读。

获取最新的 tokenless release tag：
```bash
git tag --list 'tokenless/v*' | sort -V | tail -1
```

**如果用户指定了版本号**：
1. 校验格式必须匹配 `X.Y.Z`（纯数字三段式，不含 `v` 前缀）。如果用户输入了 `v` 前缀（如 `v0.3.3`）或 `tokenless/v` 前缀，自动去除并提示。格式不合法则中止。
2. 校验通过后，记为 `TARGET_VERSION`。保留 `LAST_TAG` 供 Phase 3 使用。

**如果用户未指定版本号**：
1. 对比 `CURRENT_VERSION`（Cargo.toml）与 `LAST_TAG` 对应的版本号：
   - 如果 `CURRENT_VERSION` 已高于 `LAST_TAG` 版本（已手动 bump），优先建议使用 `CURRENT_VERSION`
   - 否则，用 `git log` 分析自上一个 tag 以来的 tokenless 相关 commits：
     - 包含 `feat!` 或 `BREAKING CHANGE` → 建议 minor bump（pre-1.0 期）或 major bump（≥ 1.0）
     - 包含 `feat` → 建议 minor bump
     - 仅 `fix`/`docs`/`chore` → 建议 patch bump
2. 通过 AskUserQuestion 展示给用户：
   - 显示当前 Cargo.toml 版本
   - 显示最新 tag 版本
   - 显示建议版本及理由
   - 让用户确认或输入自定义版本号

## Phase 2：创建 Release 分支

```bash
git checkout -b release/tokenless/v<TARGET_VERSION>
```

## Phase 3：生成 CHANGELOG

### 步骤 3.1 — 收集 commits

复用 Phase 1.6 中的 `LAST_TAG`，获取 tokenless 相关 commits：
```bash
git log --no-merges --pretty=format:'%h|%s' <LAST_TAG>..HEAD -- src/tokenless/
```

> **首次发布边界**：若 `LAST_TAG` 为空（仓库尚无 `tokenless/v*` tag），取全历史：
> ```bash
> git log --no-merges --pretty=format:'%h|%s' HEAD -- src/tokenless/
> ```

### 步骤 3.2 — 获取 PR 号

**主策略**：解析 commit message 中已有的 `(#<number>)` 格式 PR 引用。

**回退策略**：仅对无 PR 引用的 commit 调用 GitHub API：
```bash
GH_PAGER="" gh api "repos/alibaba/anolisa/commits/<HASH>/pulls" --jq '.[0].number'
```

- 有 PR 号 → `(#<number>)`
- 无 PR 号 → `(<hash>)`

> **注意**：如遇 GitHub API 速率限制（HTTP 403），通过 `x-ratelimit-reset` 获取重置时间等待重试。等待超过 60 秒或重试仍失败则停止 API 调用，剩余 commit 使用 `(<hash>)` 格式并告知用户。

### 步骤 3.3 — 分类与格式化

按 Conventional Commits type 分类：

| 分类 | 匹配规则 | CHANGELOG.md 前缀 | spec.in changelog 前缀 |
|------|----------|-------------------|----------------------|
| Breaking Changes | `!:` 或 `BREAKING CHANGE` | `- **BREAKING**` | `- BREAKING:` |
| Features | type 为 `feat` | `- add` | `- feat:` |
| Bug Fixes | type 为 `fix` | `- fix` | `- fix:` |
| Misc | `docs`/`chore`/`refactor`/`perf`/`style`/`test` | `- update` 等 | `- chore/refactor/perf:` 等 |

格式化规则（参照现有 `src/tokenless/CHANGELOG.md` 风格——简洁动词开头的小写条目，不含 PR 号；spec.in `%changelog` 段则保留完整 commit 风格 `<type>(scope): <desc>`）：

- CHANGELOG.md：英文，动词原形开头（小写），不含 PR 号或 hash（与现有 0.3.0 / 0.2.0 / 0.1.0 段一致）
- spec.in `%changelog` 段：动词风格按现有 0.3.2-1 / 0.3.1-2 等条目风格——日期 + 作者 + 版本 `- N`，然后逐条 `- <change>`
- 不包含 release commit 本身（如 `chore(tokenless): release v*`）

### 步骤 3.4 — 展示草稿并等待确认

将分类后的 CHANGELOG **两份**草稿以 Markdown 格式直接输出给用户（**不写入文件**）：

````
以下是为 v<TARGET_VERSION> 生成的 CHANGELOG 草稿：

【CHANGELOG.md】

## <TARGET_VERSION>

- <Bug Fixes 条目，按现有风格>
- <Features 条目>
- <Misc 条目>

【tokenless.spec.in %changelog】

* <Day Mon DD YYYY> <Your Name> <email@...> - <TARGET_VERSION>-1
- <Bug Fixes>
- <Features>
- <Misc>

请审核以上内容。你可以：
1. 确认无误，我将写入两个文件
2. 提供修改意见，我来调整后重新展示
````

使用 AskUserQuestion 让用户选择确认或修改。空分类不展示。

> 日期、作者、邮箱通过 `git config user.name`、`git config user.email`、`date '+%a %b %d %Y'` 获取；与现有 spec.in 风格保持一致。

### 步骤 3.5 — 写入 CHANGELOG

用户确认后：
1. 将新版本内容插入 `src/tokenless/CHANGELOG.md` 的 `# Changelog` 标题之后、第一个 `## <旧版本>` 之前
2. 将新版本 `%changelog` 条目插入 `src/tokenless/tokenless.spec.in` 的 `%changelog` 段顶部（紧跟 `%changelog` 行之后，旧条目之前）

## Phase 4：更新版本号与 lock 文件

### 步骤 4.1 — 更新 Cargo workspace 版本号

更新 `src/tokenless/Cargo.toml` 的 `[workspace.package].version` 字段为 `TARGET_VERSION`。

```toml
[workspace.package]
version = "<TARGET_VERSION>"
```

> 不更新 `crates/*/Cargo.toml`：它们使用 `version.workspace = true` 继承（如未继承则 flag 给用户，可能是历史遗留需要补 inherit）。

使用 Edit 工具精确替换该字段值，不改动其他内容。

### 步骤 4.2 — 更新 lock 文件

```bash
cd src/tokenless
cargo update -p tokenless-cli -p tokenless-schema -p tokenless-stats
```

> 限定 `-p` 避免顺带升级所有间接依赖（造成 noisy diff 和潜在回归）。

如果 lock 文件无变更（极少见，比如版本号未变），跳过 add；正常会有 `Cargo.lock` 内对应 crate 段的版本更新。

## Phase 5：Preflight 检查

执行 [references/quality-check.md](references/quality-check.md) 中的**一键模式**（`make fmt && make build && make lint && make test`）。

> hooks 集成测试需要 `jq` + `python3`，缺失则告知用户并跳过该子步骤（不算 release 失败，但需在 PR Testing 段注明）。

## Phase 6：检查并提交变更

### 步骤 6.1 — 检查意外变更

```bash
git diff --name-only src/tokenless/
```

预期变更的文件**仅限**：
- `src/tokenless/CHANGELOG.md`
- `src/tokenless/Cargo.toml`
- `src/tokenless/Cargo.lock`
- `src/tokenless/tokenless.spec.in`

如果发现其他文件被修改（如 `.rs` 文件被 `cargo fmt` 修改、Makefile 等），暂停并告知用户：这些文件格式不一致或被 preflight 自动修复，建议先单独提交格式/修复变更，再重新触发 release。

### 步骤 6.2 — 提交

确认无意外变更后，只 add 明确的 release 文件：

```bash
git add src/tokenless/CHANGELOG.md \
        src/tokenless/Cargo.toml \
        src/tokenless/Cargo.lock \
        src/tokenless/tokenless.spec.in
git commit -m "chore(tokenless): release v<TARGET_VERSION>"
```

## Phase 7-8：Push & 创建 PR

执行 [references/pr-creation.md](references/pr-creation.md) 中的**直推模式**。调用前先导出临时文件所需的变量：

```bash
export FLOW=release
export TARGET=<TARGET_VERSION>
```

PR 标题：`chore(tokenless): release v<TARGET_VERSION>`

PR 模板填写要点：
- Description：`Release tokenless v<TARGET_VERSION>. Includes <N> commits since <LAST_TAG>.`
- Related Issue：`no-issue: release`
- Type of Change：勾选 `CI/CD or build changes`
- Scope：勾选 `tokenless`
- Testing：`make fmt && make build && make lint && make test passed (cargo + hooks).`

AI 评论内容微调为：
```
> [!CAUTION]
> **This release PR was generated and submitted by AI.**
> Please review all changes carefully before merging. Pay special attention to version numbers, CHANGELOG accuracy (CHANGELOG.md + tokenless.spec.in %changelog), Cargo.lock consistency, and three-adapter (cosh/openclaw/hermes) snapshot alignment.
```

## Phase 9：完成

### 步骤 9.1 — 创建 tag 消息文件

将本次版本的 CHANGELOG 条目写入 tag 消息文件（`dist/` 目录已被 `.gitignore` 忽略）：

```bash
mkdir -p dist
cat > dist/release.tokenless-<TARGET_VERSION>.tag <<'EOF'
Release Tokenless v<TARGET_VERSION>

<CHANGELOG.md 该版本条目，不含 ## 版本号标题>
EOF
```

### 步骤 9.2 — 输出发布摘要

- 版本号：`<TARGET_VERSION>`
- Release 分支名：`release/tokenless/v<TARGET_VERSION>`
- PR URL
- CHANGELOG 条目数
- Tag 消息文件路径：`dist/release.tokenless-<TARGET_VERSION>.tag`
- 关联的下游：**RPM 仓库**（如 `~/workspace/tokenless/`）的 `package-tokenless.sh` 的 `TAG` 需在 PR 合并 + tag 发布之后更新到 `tokenless/v<TARGET_VERSION>`；此动作由 `/tokenless-dev distribute` 流程接管，但用户也可手动执行

**重要提示**：告知用户 PR 合并后需**手动执行 tag 操作**来触发 Release 工作流：

```
PR 合并后，请执行以下操作完成发布：

1. 切换到 main 并拉取最新代码：
   git checkout main && git pull --ff-only origin main

2. 使用已生成的消息文件创建 annotated tag：
   git tag -a tokenless/v<TARGET_VERSION> -F dist/release.tokenless-<TARGET_VERSION>.tag

3. 推送 tag（将自动触发 release 工作流，如已配置）：
   git push origin tokenless/v<TARGET_VERSION>

4. tag 推送后，可运行 `/tokenless-dev distribute <TARGET_VERSION>` 构建 RPM。
```

使用 AskUserQuestion 确认用户已了解后续步骤。

## 中止场景（流程专属）

| 条件 | 处理方式 |
|------|----------|
| 版本号格式不合法 | 中止：提示正确格式为 `X.Y.Z` |
| 用户取消版本号确认 | 中止 |
| 用户多次修改 CHANGELOG 仍不满意 | 中止：提示用户手动编辑两份 changelog 后重新触发 |
| preflight 后发现意外文件变更 | 暂停：提示用户先单独提交 fmt/lint/build 变更 |
| 远程同名分支已存在且用户选择中止 | 中止 |
| PR 创建失败 | 输出错误信息，提示用户手动创建 |
| `crates/*/Cargo.toml` 未用 `version.workspace = true` | 暂停：提示用户先 inherit，避免漏改子 crate 版本 |
