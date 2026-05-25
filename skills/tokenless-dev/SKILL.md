---
name: tokenless-dev
description: Unified development workflow for the tokenless component (Rust) in alibaba/anolisa. Supports sub-commands: "fix" for automated bug fixing from GitHub issues, "release" for version releasing (Cargo workspace + spec.in changelog), "distribute" (aliases: "dist" / "distribution") for building tokenless RPM/SRPM from a published GitHub release, "review" for structured code review on a submitted PR with line-level comments posted back to GitHub. Use when the user invokes /tokenless-dev fix <issue>, /tokenless-dev release [version], /tokenless-dev distribute [version], or /tokenless-dev review <pr>.
version: 0.1.0
---

# Tokenless Dev

alibaba/anolisa 仓库中 tokenless (Rust) 组件的统一研发工作流，覆盖源码 monorepo 与独立的 RPM 打包仓库两端。

## 触发与路由

解析用户输入的子命令，加载对应的子流程文件：

| 输入 | 子流程 | 说明 |
|------|--------|------|
| `/tokenless-dev fix <issue>` | [issue-fix.md](issue-fix.md) | 自动修复 Bug 类 Issue |
| `/tokenless-dev release [version]` | [release.md](release.md) | 版本发布（Cargo workspace + spec.in changelog 双写） |
| `/tokenless-dev distribute [version]`（别名 `dist` / `distribution`） | [distribute.md](distribute.md) | 从 GitHub Release 构建 RPM/SRPM 全链路 |
| `/tokenless-dev review <pr>` | [review.md](review.md) | 对已提交 PR 做结构化评审并回写 line-level 评论 |

- `<issue>` 支持编号（`213`）或完整 URL（`https://github.com/alibaba/anolisa/issues/213`）
- `<pr>` 支持编号（`290`）或完整 URL（`https://github.com/alibaba/anolisa/pull/290`）
- `[version]` 可选：
  - `release`：格式 `X.Y.Z`，不指定则自动分析并建议
  - `distribute`：支持 `0.3.2` / `v0.3.2` / `tokenless/v0.3.2` 三种写法，不指定则自动取最新 `tokenless/v*` release
- `distribute` 的三个字面量（`distribute` / `dist` / `distribution`）全部路由到同一个子流程

未匹配任何子命令时，列出可用子命令供用户选择。

## 关键路径

### 源码侧（anolisa monorepo）

| 资源 | 路径 |
|------|------|
| Monorepo 根 | `alibaba/anolisa` 仓库根 |
| 源码根 | `src/tokenless/` |
| Cargo workspace 配置 | `src/tokenless/Cargo.toml`（`[workspace.package].version` 是单一版本源） |
| Crates | `src/tokenless/crates/{tokenless-cli,tokenless-schema,tokenless-stats}/` |
| Adapters bundle | `src/tokenless/adapters/tokenless/{common,openclaw,hermes}/` |
| Adapter manifest | `src/tokenless/adapters/tokenless/manifest.json` |
| Cosh extension manifest | `src/tokenless/adapters/tokenless/common/cosh-extension.json` |
| OpenClaw plugin | `src/tokenless/adapters/tokenless/openclaw/{index.ts,openclaw.plugin.json,scripts/}` |
| Hermes plugin | `src/tokenless/adapters/tokenless/hermes/{__init__.py,plugin.yaml,scripts/}` |
| Tool-Ready spec | `src/tokenless/adapters/tokenless/common/tool-ready-spec.json` |
| Env-fix 脚本 | `src/tokenless/adapters/tokenless/common/tokenless-env-fix.sh` |
| Common hooks | `src/tokenless/adapters/tokenless/common/hooks/{*.py,*.sh}` |
| Vendored rtk | `src/tokenless/third_party/rtk/`（由 `just setup-rtk` 克隆 + patch） |
| 第三方 patches | `src/tokenless/third_party/patches/*.patch` |
| Build 系统 | `src/tokenless/Makefile` + `src/tokenless/justfile` |
| Hook 集成测试 | `src/tokenless/tests/run-all-tests.sh` |
| CHANGELOG | `src/tokenless/CHANGELOG.md` |
| RPM spec 模板 | `src/tokenless/tokenless.spec.in`（含 `@VERSION@` 占位符 + 主 `%changelog`） |
| Tag 格式 | `tokenless/v<version>`（如 `tokenless/v0.3.2`） |

### RPM 打包侧（独立仓库）

| 资源 | 路径 |
|------|------|
| RPM 打包仓库 | `<WS_RPM>`（用户本地路径，典型为 `~/workspace/tokenless/`） |
| 打包脚本 | `<WS_RPM>/package-tokenless.sh`（含 `TAG=tokenless/v<version>`） |
| 实例化 spec | `<WS_RPM>/tokenless.spec`（由 spec.in 派生，`@VERSION@` 已替换） |
| 源码 tarball | `<WS_RPM>/tokenless-<version>.tar.gz` |
| RPM 产物 | `~/rpmbuild/RPMS/x86_64/tokenless-<version>-1.*.x86_64.rpm` |
| SRPM 产物 | `~/rpmbuild/SRPMS/tokenless-<version>-1.*.src.rpm` |
| 输出目录 | `<WS_RPM>/packages/`（distribute 流程落盘位置） |

## 全局约定

### GH 命令规范

所有 `gh` 命令必须禁用交互式分页器，避免在终端中阻塞：

```bash
GH_PAGER="" gh <command>
# 或
gh <command> | cat
```

### 并行执行原则

工作流中标记 **可并行** 的步骤，必须使用 Task 工具同时启动多个子 agent 并行执行（在同一条消息中发送多个 Task 调用），以节省时间。

### 失败保留原则

所有子流程失败 / 用户中止时：
- **保留** `.qoder/tokenless-dev/` 下与本次运行相关的所有临时文件
- 向用户输出文件绝对路径列表，方便排查
- 不自动清理（清理责任交给下次运行前的自愈机制或用户手动）

成功完成时按 [references/gh-cli-conventions.md §2.6](references/gh-cli-conventions.md) 清理。

### 平台守门（tokenless 专属）

tokenless 是 **Linux only** 组件（README 与 anolisa AGENT.md 均明示）。

- 涉及 `cargo build` / `cargo test` / `make` / `rpmbuild` 的子流程（fix / release / distribute）在 Phase 1 (env-check) 必须校验 `uname -s == Linux`，否则中止并提示：
  - 切换到 Linux 开发机执行（参考用户内存中的远程开发机 `dev`，可通过 SSH 连接）
- review 子流程是只读流程，**不**走 cargo 工具链，跨平台允许运行

### 通用中止场景

| 条件 | 处理方式 |
|------|----------|
| 不在 anolisa 仓库中 | 中止：提示用户 `cd` 到仓库目录 |
| `gh` 未认证 | 中止：建议运行 `gh auth login` |
| 工作区有未提交修改 | 中止：提示用户 commit 或 stash（review / distribute 除外） |
| `git pull` 失败（如有冲突） | 中止：提示用户手动解决 |
| 平台非 Linux 且当前流程涉及 build/test | 中止：提示用 Linux 开发机 |

## 共享组件

子流程通过引用 `references/` 下的组件文件来复用通用逻辑：

| 组件 | 文件 | 职责 | 引用方 |
|------|------|------|--------|
| 环境验证 | [references/env-check.md](references/env-check.md) | git remote / gh auth / workspace / sync main / Linux / Rust toolchain / rpmbuild | fix / release / distribute / review(部分) |
| 代码规范 | [references/code-standards.md](references/code-standards.md) | Rust + adapter 脚本规范 / clippy 0-warning / 版权头规则 | fix / review |
| 质量检查 | [references/quality-check.md](references/quality-check.md) | fmt / clippy / build / test / hook 集成测试 流水线 | fix / release |
| PR 创建 | [references/pr-creation.md](references/pr-creation.md) | Fork / 直推模式 / anolisa PR 模板 / AI 评论 | fix / release |
| 交互规范 | [references/interaction-guide.md](references/interaction-guide.md) | AskUserQuestion 简洁问法 / 背景信息呈现 | 全部 |
| **tokenless 代码关注点** | [references/tokenless-concerns.md](references/tokenless-concerns.md) | 核心四维 + 14 条横切触发 + 工程纪律（含三适配 target / FHS / RPM / Linux） | review(检查) / fix(自检) |
| **gh CLI 与临时文件** | [references/gh-cli-conventions.md](references/gh-cli-conventions.md) | gh CLI 使用约定 + `<workspace>/.qoder/tokenless-dev/` 临时文件规范 | 全部 |
| **Review 结论与提交** | [references/review-submit.md](references/review-submit.md) | YAML claim / 四层反证 / tier-3 脏代码防护 / gh api JSON 模板 / 硬门 checklist | review 专属 |

## 扩展指南

新增子流程（如 `feat`、`refactor`、`hotfix`）：

1. 在根目录创建 `<name>.md`，编写流程步骤，按需引用 `references/` 组件
2. 在上方路由表中注册新的子命令映射
3. 如有新的通用逻辑，提取到 `references/` 下作为共享组件
