# CLAUDE.md

> 本地配置，禁止提交/推送（`.git/info/exclude` 已排除 git 追踪）。

## 项目

ANOLISA — Agentic OS monorepo。组件及技术栈：

- `cosh` = `src/copilot-shell/` (TypeScript/Node.js, 全平台)
- `sec-core` = `src/agent-sec-core/` (Rust+Python, 仅Linux)
- `sight` = `src/agentsight/` (Rust/eBPF, 仅Linux)
- `tokenless` = `src/tokenless/` (Rust, 仅Linux)
- `skill` = `src/os-skills/` (Python/Shell, 全平台)
- `ckpt` = `src/ws-ckpt/` (Rust, 仅Linux)

> sec-core/sight/tokenless/ckpt 禁止在 macOS/Windows 构建。

## 行为准则（Karpathy Guidelines）

1. **编码前思考**: 不假设，不确定就问。有歧义呈现多种解释。有更简方案就说。困惑时停下来。
2. **简洁优先**: 只写最少代码。不为单词用加抽象。不加未要求的功能/灵活性/错误处理。200行能变50行就重写。自问"资深工程师会觉得过度复杂吗？"
3. **精准修改**: 只改任务相关行。不改相邻代码/注释/格式。匹配现有风格。注意到死代码只提不删。清理自己改动造成的孤儿引用。检验：每行改动都能追溯到用户原话。
4. **目标驱动执行**: 将任务转成可验证目标——先写复现测试→让测试通过→验证无回归。多步骤给出 `步骤→verify:检查点` 计划。循环验证直到达成。

> 偏向谨慎而非速度。琐碎任务自行判断。

## 强制约束

**Commit**: `type(scope): description`，英文，小写开头，无句号。scope 必填（CI 硬阻断）。Breaking 加 `!`。scope→路径: cosh→copilot-shell, sec-core→agent-sec-core, skill→os-skills, sight→agentsight, tokenless, ckpt→ws-ckpt, ci→.github/workflows, docs, deps→lock/toml, chore→其他。

**分支**: `feature|fix|hotfix/release/<scope>/<desc>`，scope 同上。生命周期 ≤2周，超期 rebase 或关闭。

**Tag**: `<scope>/vX.Y.Z`，仅 main 分支，annotated tag (`git tag -a`)，禁止 force push/删除。

**版本**: 独立 SemVer。Bug→PATCH, Feat→MINOR, Breaking→MAJOR(须Issue讨论)。release 分支手动 bump。

**语言**: 代码和注释仅英文。

**SOB**: 必须且仅用 `Signed-off-by: Shile Zhang <shile.zhang@linux.alibaba.com>`，禁止其他。

**Push**: 禁止 `git push`，须用户人工手动执行。可 commit，不可 push。

**工作区边界**: 不得修改或删除工作区(`/root/anolisa/`)之外的文件。如需改动外部文件，先确认后再执行。

## SOP

**发布**: checkout -b release/<scope>/vX.Y → git-cliff 生成 CHANGELOG → PR `chore(<scope>): bump to vX.Y.Z` → rebase 合入 main(≥1 approve) → `git tag -a <scope>/vX.Y.Z`

**CHANGELOG**: 各子项目独立维护 Keep a Changelog，共用根 `cliff.toml`。`git cliff <scope>/vX.Y.Z..HEAD --include-path "src/<comp>/**" -o src/<comp>/CHANGELOG.md`。映射: feat→Added, fix→Fixed, refactor|perf→Changed, feat!/BREAKING→Breaking。docs/chore/test/ci 不出现。

**PR**: 用 `.github/pull_request_template.md` 模板。Related Issue 必填(close/fix/resolve #n 或 no-issue: 原因)。Checklist 按变更组件勾选。

**Merge**: 子任务→集成分支 = Squash；集成分支→main = Rebase。

## 开发模式

- **Fast Track**(单人+单功能+集中): main拉分支 → 开发 → PR → Squash Merge → 删分支
- **Collaborative**(多人/大功能/跨模块): main → feature/<scope>/<name> (集成分支) → feature/<scope>/<name>/<task-N> (子任务)。子任务 Squash 入集成，最终 Rebase 入 main。负责人定期 rebase main。

## Pre-Commit CI 检查

每次 commit 前按变更组件执行。commitlint(`.github/commitlint.config.json`): scope-empty=error(2)必填, scope-enum=warn(1), header≤120=error(2)。有效scope: cosh,sec-core,skill,sight,tokenless,ckpt,deps,ci,docs,chore。PR lint(warning不阻断): 标题格式/分支命名/Issue关联。

### copilot-shell
```bash
cd src/copilot-shell && npm ci
npm run format && git diff --exit-code
npm run lint:ci && npm run build && npm run typecheck
npm run test:ci --workspace=@copilot-shell/cli
npm run test:ci --workspace=@copilot-shell/core
```

### agent-sec-core
```bash
cd src/agent-sec-core
# Rust
cargo fmt --all --check && cargo clippy --workspace -- -D warnings && cargo test
# Python
make python-code-pretty && git diff-index --quiet HEAD --
pytest tests/integration-test/ tests/unit-test/ -v
make test-python-coverage
# 同步
cd agent-sec-cli && uv lock --check && cd ..
make export-requirements && git diff --quiet -- agent-sec-cli/requirements.txt
# openclaw
cd openclaw-plugin && npm install && cd .. && make test-openclaw-plugin-coverage
```
增量覆盖率门禁(PR): 新增代码 ≥80%，低于则 CI 失败。

### tokenless
```bash
cd src/tokenless
cargo fmt --all --check && cargo clippy --workspace -- -D warnings && cargo test --workspace
```

### agentsight
```bash
cd src/agentsight && cargo fmt --check && cargo test
```

### ws-ckpt
```bash
cd src/ws-ckpt/src
cargo fmt --all --check && cargo clippy --workspace -- -D warnings && cargo test --workspace
```

## 代码规范

TS: ESLint+Prettier | Python: Ruff+Black | Rust: `cargo fmt` + `cargo clippy -- -D warnings`。不隐藏错误。每次改动提升代码质量。