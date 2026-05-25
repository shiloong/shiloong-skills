# tokenless 代码关注点清单

tokenless 组件研发与评审共享的关注点字典，供所有 tokenless-dev 子流程引用。

## 使用方式

| 场景 | 引用方式 |
|------|---------|
| **review 子流程** | Phase 4 按本文件逐条核对 PR 产物，作为 4 维 + 横切触发 + 工程纪律的检查清单 |
| **issue-fix 子流程** | Phase 6 实施修复前，按"diff 关键词触发"一节做一次自检，提前规避常见隐患 |
| **future feat / refactor 子流程** | 同 issue-fix，写代码前过一遍"横切关注点触发表" |

---

## 核心四维（必过）

所有代码改动都必须满足以下 4 个维度：

| 维度 | 核心关切（tokenless 适配） |
|------|-------------------------|
| **A 对现有功能的影响** | 公共 API（`tokenless-schema` 库导出符号、CLI 子命令签名、SQLite stats 表/列）、配置兼容（`tool-ready-spec.json` schema、`cosh-extension.json`/`openclaw.plugin.json`/`plugin.yaml`）、删除/改名是否搜索过所有 callsite、破坏性变更是否有迁移路径 |
| **B 新增逻辑正确性与隐患** | Rust：`Result` 传播 vs `unwrap`/`expect`、`Option` 处理、`?` 在错误类型转换时的语义、SIMD/正则边界、文件/管道/SQLite 并发与锁、tokio/rayon 异步生命周期；Hook 脚本：fail-open 策略、stderr/stdout 分离、超时与跨进程信号 |
| **C 测试覆盖** | 新增功能是否有 `cargo test` 单测；被删除/重命名测试是否补回等效断言；mock 不是占位（能真正触发分支）；hook 改动同步更新 `tests/run-all-tests.sh` 集成测试 |
| **D 规范符合度** | 对照 [code-standards.md](code-standards.md)：`cargo fmt`、`cargo clippy -- -D warnings`、Apache-2.0 SPDX 头、`unwrap` 在 lib 代码中禁用、`pub` 必有 doc comment、跨 crate 用包名 |

---

## 横切关注点触发表（按 diff 关键词触发）

并非每条都适用于每个 PR。当 diff 命中"触发关键词"任一时，对应维度**必须**纳入检查；否则可跳过。

| # | 维度 | 触发关键词 / 路径 | 关注点 |
|---|------|-------------------|--------|
| 1 | **三适配 target 同步** | `adapters/tokenless/common/hooks/*` / `cosh-extension.json` / `openclaw/index.ts` / `openclaw.plugin.json` / `hermes/__init__.py` / `hermes/plugin.yaml` | 任何 hook 行为变更必须在 cosh / openclaw / hermes 三处的实现 + manifest + README 一致；schema 压缩仅 cosh/openclaw 支持，hermes 阻塞中（在 README 表格里标 ⏳）；新增 hook event 必须三处同步声明 |
| 2 | **SQLite stats schema** | `crates/tokenless-stats/**` / 含 `migrations/` 的目录 / `.sql` / `CREATE TABLE` / `ALTER TABLE` / 列名 | 任何列增加/类型变更必须配套迁移脚本；schema 版本号自增；测试覆盖旧数据升级路径（参考 0.3.1-2 的 `before_output`/`after_output` 列迁移）；schema 兼容性是公共 API（外部可能读 SQLite） |
| 3 | **Tool-Ready spec** | `adapters/tokenless/common/tool-ready-spec.json` / `tokenless-env-fix.sh` / `tool_ready_hook.sh` / `env-check` CLI | 新增 tool 依赖必须 4 类齐全（required / recommended / fallback / package manager），字符串简写也要兼容；fail-open（缺依赖不阻塞工具调用）；新增包管理器（apt/dnf/cargo/...）需在 env-fix.sh 加分支 |
| 4 | **RPM 安装路径（FHS）** | `Makefile` BINDIR/LIBEXECDIR/DATADIR / `tokenless.spec.in` `%{_bindir}`/`%{_libexecdir}`/`%{_datadir}` / `%files` 段 / `install -m` | helper 二进制必须在 `/usr/libexec/anolisa/tokenless/` + `/usr/bin/` symlink；adapter 资源在 `/usr/share/anolisa/adapters/tokenless/`；extension 在 `/usr/share/anolisa/extensions/tokenless/`；改动需 Makefile + spec.in 同步；`%post`/`%preun` scriptlet 中的清理路径要跟上 |
| 5 | **第三方 patch 维护** | `third_party/patches/*.patch` / `justfile setup-rtk` / `rtk_tag` 变更 | rtk 升级时 patch 必须 rebase；当前 justfile 在 patch 失败时仅打 WARNING——若改动是 patch 相关（如 stats 行为），需评估是否提升到 fail-fast；patch 文件本身需带 commit 链或注释说明针对哪个上游版本 |
| 6 | **rtk / toon 版本对齐** | `justfile` `rtk_tag` / `toon_ver` / `Makefile TOON_VER` / `tokenless.spec.in` `cargo install toon-format --version` / `package-tokenless.sh` `TAG` / `README.md` 版本表 | 这 5 处的版本号必须一一对应；任意一处升级需联动其他 4 处；尤其要警惕 monorepo 内的 justfile/Makefile/spec.in 与独立 RPM 仓库的 `package-tokenless.sh` `TAG` 不一致 |
| 7 | **cosh extension 契约** | `cosh-extension.json` / `common/hooks/*` 文件路径 / hook event 名 | cosh 仓库的 cosh-extension spec 是 contract；签名/hook event 名变更必须读 cosh-dev 文档确认兼容；本地修改后需在 cosh 中触发 cosh extension 自动发现验证；安装路径 `~/.copilot-shell/extensions/tokenless/` 与 `/usr/share/anolisa/extensions/tokenless/`（system）优先级 |
| 8 | **OpenClaw plugin** | `openclaw/index.ts` / `openclaw.plugin.json` / `openclaw/scripts/*.sh` / `package.json` | TS 源 + JS 编译产物双产物；spec.in `%build` 段同时支持 `npx esbuild` 与 `sed` 兜底（无 Node 环境时降级）；`plugin.json version` 必须与 Cargo.toml 同步；`detect/install/uninstall.sh` 必须幂等（多次执行结果一致）；`%preun` 里的 openclaw.json 清理逻辑参考已存在的 jq 段 |
| 9 | **Hermes plugin** | `hermes/__init__.py` / `hermes/plugin.yaml` / `hermes/scripts/*` | Python plugin，`register(ctx)` 钩三个 event（`pre_tool_call` / `transform_tool_result` / `on_session_start`）；任何 hook 行为变更需在三 event 维度全部检查；plugin.yaml 元数据准确；command rewriting 在 hermes 是 "block + suggest"（不能直接 modify args），需保留这条注释；schema 压缩在 hermes 仍 blocked，README 表格的 ⏳ 不要误改为 ✅ |
| 10 | **环境检测降级（fail-open）** | `tokenless-env-fix.sh` / `tool_ready_hook.sh` / `compress_response_hook.py` / `compress_schema_hook.py` / `rewrite_hook.py` | 缺失依赖必须降级而非阻塞；错误必须可观测（stderr 或 stats）；不得静默吞错导致 LLM 浪费 token 重试；hook 失败时必须输出原始 input 以保证调用方流水线不断 |
| 11 | **跨包公共 API** | `crates/tokenless-schema/src/lib.rs` 导出 / `pub fn` / `pub struct` / `pub enum` | schema 库被 cli 和潜在外部用户引用；签名变更必须 search 所有 callsite；非破坏可加 `#[deprecated]`，破坏需主版本号；新增 `pub` 项必须有 doc comment |
| 12 | **rtk vendoring + 排除 lint** | `Cargo.toml [workspace] exclude` / `Makefile`/`justfile` 中 `-p` 参数 / `cargo clean` | rtk 不在 workspace；clippy/test/clean 命令必须显式 `-p tokenless-cli -p tokenless-schema -p tokenless-stats` 避免误扫；新增 crate 时需同步加到 `[workspace] members` + Makefile 的 lint/fmt/test 命令 |
| 13 | **CHANGELOG + spec changelog 双写** | `src/tokenless/CHANGELOG.md` / `tokenless.spec.in` `%changelog` 段 / 独立 RPM 仓库的 `tokenless.spec` `%changelog` | release 时 anolisa 侧两处必须同步（CHANGELOG.md 简洁，spec.in 详细带日期/作者）；RPM 仓库的 `tokenless.spec` 由 distribute 流程从 spec.in 派生，**不要**手动改；如检测到 RPM 仓库 spec 与 monorepo spec.in 偏离，flag 给用户 |
| 14 | **Linux only 守门** | `cfg(target_os)` / `cfg(unix)` / 跨平台 API 调用 / Cargo `[target.'cfg(...)']` | tokenless README + AGENT.md 都明示 Linux-only；任何 macOS-friendly 改动（如增加 cfg 分支）需在 PR 描述说明 motivation；CI 默认 Ubuntu，本地开发非 Linux 必须用远程开发机（用户内存中的 `dev`） |

---

## 工程纪律（每次 PR 都看）

| # | 项目 | 说明 |
|---|------|------|
| 1 | **Scope 范围控制** | PR title / "Description" 段声明的范围 vs diff 实际范围是否一致；夹带不相关改动要指出（scope creep）；多组件改动时主 scope 选改动量最大的，但 `tokenless` 范围内的"crates vs adapters vs spec"夹带建议拆 PR |
| 2 | **PR 描述与 checklist 诚实度** | Type of Change 勾选与实际一致；Scope 勾选 `tokenless`；Testing 段不得留空或填 "N/A" 而未说明；Linux-only 改动需在 Additional Notes 提醒 |
| 3 | **CHANGELOG + spec changelog 双向同步** | 用户可见行为变更必须同步到 `src/tokenless/CHANGELOG.md` 和 `tokenless.spec.in` `%changelog` 段；内部重构可只更 CHANGELOG.md（看作"内部 build 变更"）；如改了 spec.in 但没改 CHANGELOG.md（或反之），flag |
| 4 | **可回滚性** | PR 是否可独立 revert；是否与其他未合入 PR 强耦合；SQLite migration 是否有降级路径或 backup 建议 |
| 5 | **lint 0-warning 门** | `cargo clippy -- -D warnings`：任何 clippy warning 等同 error，评审不得宽容 warning；CI 强制 |
| 6 | **Commit 质量** | Conventional Commits 格式（`<type>(tokenless): <desc>`，scope 强制）；必要时附 `Closes #<issue>`；本仓库允许 squash merge，不强制单 commit（参见 [pr-creation.md](pr-creation.md)） |

---

## 快速自检流程（issue-fix / feat 场景用）

写代码前按顺序过一遍：

1. **列出本次将要改动的文件路径集合**（从 Phase 3 信心评估的产物中拿）
2. **将文件路径集合 / 将要引入的关键词** 与上表"触发关键词"列匹配
3. **命中的每一条** 都在编码时落实（例：若命中 #1，则改 hook 时同时更新 cosh/openclaw/hermes 三处对应文件；若命中 #2，则同时写 migration 与升级测试）
4. **工程纪律 6 条** 在 Commit 前过一遍（例：#1 检查 diff 是否只在声明范围内；#3 检查是否要更新 CHANGELOG + spec.in）

自检产物不要求成文，目的是把"评审侧会发现的问题"提前到编码侧消化。
