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
4. **最小改动**: 每次改动只做必须做的，不多也不少。不附带无关依赖升级、格式调整或顺手优化。验证时逐行审视 diff，确保零冗余。
4. **目标驱动执行**: 将任务转成可验证目标——先写复现测试→让测试通过→验证无回归。多步骤给出 `步骤→verify:检查点` 计划。循环验证直到达成。

> 偏向谨慎而非速度。琐碎任务自行判断。

## 交互纪律

1. **验证优先**: 每次修改必须附带验证方法——运行测试、构建检查或可观察的行为变化。没有验证的代码不算完成。
2. **治本不治标**: 面对报错，解决根本原因，不仅消除症状（suppress/ignore）。禁止用 `unwrap_or`/`2>/dev/null`/`|| true` 掩盖错误。
3. **先规划再动手**: 涉及 ≥2 文件或不确定影响范围时，先进入 Plan 模式探索和规划，经用户确认后再实施。单文件小改可直接执行。
4. **一事一议**: 一个会话只推进一个任务主线。插问用 `/btw` 处理，不搅乱主上下文。
5. **约束前置**: 最重要的约束（如"绝不动配置文件"、"只改这行"）放在 prompt 最前面，不埋在中间。
6. **限缩探索范围**: 用 `@文件路径` 指定关注范围，不要无限探索整个仓库。研究型探索交给 SubAgent。
7. **陷入循环就止损**: 连续两轮在同一问题原地打转时，停止并汇报，而不是继续补补丁。建议用户 `/clear` 重开。
8. **反馈闭环**: 完成修改后，主动运行验证并报告结果。失败则继续修复直到通过，不要半途交给用户。

## 强制约束

**Commit**: `type(scope): description`，英文，小写开头，无句号。scope 必填（CI 硬阻断）。Breaking 加 `!`。scope→路径: cosh→copilot-shell, sec-core→agent-sec-core, skill→os-skills, sight→agentsight, tokenless, ckpt→ws-ckpt, ci→.github/workflows, docs, deps→lock/toml, chore→其他。

**分支**: `feature|fix|hotfix/release/<scope>/<desc>`，scope 同上。生命周期 ≤2周，超期 rebase 或关闭。

**Tag**: `<scope>/vX.Y.Z`，仅 main 分支，annotated tag (`git tag -a`)，禁止 force push/删除。

**版本**: 独立 SemVer。Bug→PATCH, Feat→MINOR, Breaking→MAJOR(须Issue讨论)。release 分支手动 bump。Rust 子项目(tokenless/sight/ckpt) bump 版本时必须同步更新 `Cargo.lock`（`cargo update -p <workspace-crates>`，仅更新 workspace 包，不附带外部依赖升级）。

**语言**: 代码和注释仅英文。

**SOB**: 必须且仅用 `Signed-off-by: Shile Zhang <shile.zhang@linux.alibaba.com>`，禁止其他。

**Push**: 禁止 `git push`，须用户人工手动执行。可 commit，不可 push。

**工作区边界**: 不得修改或删除工作区(`/root/anolisa/`)之外的文件。如需改动外部文件，先确认后再执行。

## SOP

**发布**: checkout -b release/<scope>/vX.Y → git-cliff 生成 CHANGELOG → PR `chore(<scope>): bump to vX.Y.Z` → rebase 合入 main(≥1 approve) → `git tag -a <scope>/vX.Y.Z`

**CHANGELOG**: 各子项目独立维护 Keep a Changelog，共用根 `cliff.toml`。`git cliff <scope>/vX.Y.Z..HEAD --include-path "src/<comp>/**" -o src/<comp>/CHANGELOG.md`。映射: feat→Added, fix→Fixed, refactor|perf→Changed, feat!/BREAKING→Breaking。docs/chore/test/ci 不出现。

**PR**: 用 `.github/pull_request_template.md` 模板。Related Issue 必填(close/fix/resolve #n 或 no-issue: 原因)。Checklist 按变更组件勾选。

**Merge**: 子任务→集成分支 = Squash；集成分支→main = Rebase。

**RPM 构建**: 重新编译构建 RPM 前必须清理历史构建中间文件及旧 RPM 包（`scripts/rpmbuild/` 下的 RPMS/SOURCES/BUILD/BUILDROOT/SRPMS + `cargo clean`）。安装新 RPM 包前必须显式卸载已安装的 RPM 包（`rpm -e <package>`），避免版本残留冲突。

## 开发模式

- **Fast Track**(单人+单功能+集中): main拉分支 → 开发 → PR → Squash Merge → 删分支
- **Collaborative**(多人/大功能/跨模块): main → feature/<scope>/<name> (集成分支) → feature/<scope>/<name>/<task-N> (子任务)。子任务 Squash 入集成，最终 Rebase 入 main。负责人定期 rebase main。

## Pre-Commit CI 检查

每次 commit 前按变更组件执行。commitlint(`.github/commitlint.config.json`): scope-empty=error(2)必填, scope-enum=warn(1), header≤120=error(2), body-line≤100=error(2)。有效scope: cosh,sec-core,skill,sight,tokenless,ckpt,deps,ci,docs,chore。PR lint(warning不阻断): 标题格式/分支命名/Issue关联。

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
cargo fmt -p tokenless-cli -p tokenless-schema -p tokenless-stats -- --check && cargo clippy -p tokenless-cli -p tokenless-schema -p tokenless-stats -- -D warnings && cargo test -p tokenless-cli -p tokenless-schema -p tokenless-stats
```
> CI 使用 Rust 1.89.0。本地 Rust 版本可能更高，导致 stable-only API 本地通过但 CI 失败。禁止使用高于 CI Rust 版本才稳定的 API；如需此类功能，必须用兼容 CI 版本的手写实现替代。本地 CI 检查前须确认未引入 CI 版本不可用的 unstable 特性。

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

## Review 经验

Review 时按 `.claude/rules/review.md` 中的逻辑体系逐层审查，输出结论分三级：阻塞(block) / 建议修(suggest) / 可选清理(clean)。