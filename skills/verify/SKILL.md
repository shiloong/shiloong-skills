---
name: verify
description: Systematic self-verification and fix-loop for recent code changes. Runs build, lint, and test checks; if failures occur, fixes them and re-verifies until all pass. Use when the user invokes /verify or asks to validate recent modifications.
version: 0.1.0
---

# Verify — Self-Verification & Fix Loop

对刚完成的修改执行系统化验证。发现问题时自动修复并重新验证，直到全部通过。

## Trigger

- `/verify`
- `/verify <scope>` — 只验证特定组件
- "验证刚才的修改"
- "帮我确认这些改动没问题"

## Step 1: Determine scope

根据 `git diff --name-only HEAD`（或 `git diff --name-only` 如果未提交）确定变更涉及的组件，使用 CLAUDE.md scope 映射：

| 文件路径前缀 | scope | 检查命令 |
|-------------|-------|---------|
| `src/copilot-shell/` | cosh | npm ci → format → lint:ci → build → typecheck → test:ci |
| `src/agent-sec-core/` | sec-core | cargo fmt/clippy/test → python checks |
| `src/tokenless/` | tokenless | cargo fmt/clippy/test (指定 packages) |
| `src/agentsight/` | sight | cargo fmt --check → cargo test |
| `src/ws-ckpt/src/` | ckpt | cargo fmt/clippy/test --workspace |
| `src/os-skills/` | skill | python/shell specific checks |
| 其他 | chore | 基本检查 |

如果用户指定 `<scope>`，只运行该 scope 的检查。

## Step 2: Run verification pipeline

按变更组件运行对应的 CI 检查（见 CLAUDE.md "Pre-Commit CI 检查" 各组件命令）。

同时运行通用检查：
```bash
git diff --check HEAD  # 检查冲突标记和空白错误
```

## Step 3: Evaluate results

对每个检查命令的输出分类：
- **PASS**: 全部通过 → 记录并继续下一个检查
- **FAIL**: 有错误 → 进入 Step 4 修复循环
- **WARN**: 有警告但非阻断 → 记录，报告但不阻断

## Step 4: Fix loop (if failures found)

对每个 FAIL：

1. 读取失败输出的完整内容（不要总结，要看原始输出）
2. 定位根因：不是 suppress 错误，是解决根本原因
3. 修复代码
4. 重新运行该检查命令验证修复
5. 如果再次失败（第二轮仍在原地打转）→ **停止循环**，报告给用户，建议 `/clear` 重开

**最多 2 轮修复**。超过 2 轮说明上下文已被污染，止损比继续更高效。

## Step 5: Report

输出验证报告：

```
## Verification Report

### Scope: <scope(s)>
### Changes: <N files, +M/-K lines>

| Check | Status | Notes |
|-------|--------|-------|
| format | PASS | |
| lint | PASS | |
| build | PASS | |
| test | PASS | |
| diff-check | PASS | |

### Summary: ✅ All checks passed / ❌ <N> checks failed (see details)
```

如有修复，列出修复了什么：
```
### Fixes applied
- <file>: <what was fixed>
```

## Principles

1. **治本不治标**: 修复根因而非 suppress。禁止用 `unwrap_or`/`2>/dev/null`/`|| true` 掩盖
2. **最多 2 轮修复**: 超过说明上下文已污染，必须止损
3. **看原始输出**: 不总结错误信息，直接读完整 stack trace / lint output
4. **全部通过才算完成**: 任何一项检查未通过，整个验证不算完成