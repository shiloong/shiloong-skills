# PR 创建与提交

tokenless 组件的变更提交与 PR 创建通用流程，支持 Fork 和直推两种 Push 模式。

## Commit 规范

遵循 anolisa `AGENT.md` 的 Conventional Commits 格式，scope **强制**填写（缺失时 CI 直接 fail），仅英文：

```
<type>(tokenless): <小写描述，无句号>

<可选 body>
```

常用 type：
- `fix(tokenless):` — bug 修复，body 中附 `Closes #<issue-number>`
- `feat(tokenless):` — 新功能
- `chore(tokenless):` — release、构建脚本等维护性变更
- `refactor(tokenless):` — 重构（无行为变更）
- `perf(tokenless):` — 性能优化
- 破坏性变更：`<type>(tokenless)!: <desc>` + body 内描述迁移路径

> 注意：scope 始终是 `tokenless`，不是 `tokenless-cli` 或 `tokenless-schema` 之类的子项；按 AGENT.md 的 scope 推断表，`src/tokenless/` 下任何路径都归 `tokenless`。

## Push 模式

### Fork 模式

适用于非仓库成员提交（如 issue-fix）。

获取当前用户的 GitHub 用户名：
```bash
GH_USER=$(GH_PAGER="" gh api user --jq '.login')
```

确认 fork 仓库已存在，如不存在则自动 fork：
```bash
GH_PAGER="" gh api "repos/${GH_USER}/anolisa" --jq '.full_name' 2>/dev/null || \
  GH_PAGER="" gh repo fork alibaba/anolisa --clone=false
```

确保 fork remote 已添加，并推送分支：
```bash
git remote get-url fork 2>/dev/null || git remote add fork "https://github.com/${GH_USER}/anolisa.git"
git push -u fork <branch-name>
```

### 直推模式

适用于仓库成员直接推送（如 release）。

检查远程同名分支：
```bash
git ls-remote --heads origin <branch-name>
```

- 远程**不存在** → 直接 push：
  ```bash
  git push -u origin <branch-name>
  ```
- 远程**存在** → 使用 AskUserQuestion 询问用户：
  - 选项 1：强制覆盖推送（`git push -u origin <branch-name> --force`）
  - 选项 2：中止，用户手动处理

## PR 创建

先将 PR body 写入临时文件，通过 `--body-file` 传入，避免进入交互模式。临时文件位置遵循 [gh-cli-conventions.md](gh-cli-conventions.md) §二 的规范：

```bash
WS="$(git rev-parse --show-toplevel)"
mkdir -p "$WS/.qoder/tokenless-dev"
PR_BODY_FILE="$(mktemp "$WS/.qoder/tokenless-dev/${FLOW:-fix}-${TARGET:-pr}-prbody.XXXXXX.md")"
cat > "$PR_BODY_FILE" <<'PRBODY'
<按 PR 模板填写>
PRBODY
```

调用方需在调用此流程前定义：
- `FLOW` — 当前子流程名（`fix` / `release`）
- `TARGET` — 业务标识（如 issue 号 `213`、版本号 `0.3.2`）

创建 PR 并提取编号：

**Fork 模式**：
```bash
PR_URL=$(GH_PAGER="" gh pr create --draft --repo alibaba/anolisa \
  --head "${GH_USER}:<branch-name>" \
  --base main \
  --title "<title>" \
  --body-file "$PR_BODY_FILE")
PR_NUMBER=$(echo "$PR_URL" | grep -oE '[0-9]+$')
```

**直推模式**：
```bash
PR_URL=$(GH_PAGER="" gh pr create \
  --base main \
  --head "<branch-name>" \
  --title "<title>" \
  --body-file "$PR_BODY_FILE")
PR_NUMBER=$(echo "$PR_URL" | grep -oE '[0-9]+$')
```

清理临时文件：
```bash
rm -f "$PR_BODY_FILE"
```

## PR 模板

填写 `.github/pull_request_template.md` 的各字段（PR 内容使用英文）：

```markdown
## Description

<2-5 sentences: what changed, why, key implementation decision>

## Related Issue

<"closes #<number>" 或 "no-issue: <reason>">

## Type of Change

- [ ] Bug fix (non-breaking change that fixes an issue)
- [ ] New feature (non-breaking change that adds functionality)
- [ ] Breaking change
- [ ] CI/CD or build changes
- [ ] Refactoring
- [ ] Performance improvement
- [ ] Documentation update

## Scope

- [x] `tokenless`

## Checklist

- [x] I have read the Contributing Guide
- [x] My code follows the project's code style
- [x] I have added tests that prove my fix is effective or that my feature works
- [x] For `tokenless`: cargo test passes; clippy warnings resolved (`-D warnings`)
- [x] Lock files are up to date (`Cargo.lock`)
- [x] If hooks/adapters changed: `bash tests/run-all-tests.sh` passes
- [x] If FHS paths changed: Makefile and tokenless.spec.in are aligned

## Testing

<测试命令和结果摘要>

## Additional Notes

<补充说明，无则 <!-- N/A -->>
```

调用方根据流程类型勾选对应的 Type of Change，并填充具体内容。

## AI 评论

PR 创建成功后，立即添加 AI 生成提示评论：

```bash
GH_PAGER="" gh pr comment "$PR_NUMBER" --repo alibaba/anolisa --body "$(cat <<'EOF'
> [!CAUTION]
> **This PR was generated and submitted by AI.**
> Please review all changes carefully before merging. Pay special attention to logic correctness, edge cases, three-adapter (cosh/openclaw/hermes) consistency, and potential side effects.
EOF
)"
```

完成后输出 PR URL。
