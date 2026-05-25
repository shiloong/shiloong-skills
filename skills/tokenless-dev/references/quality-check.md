# 质量检查流水线

tokenless 组件的质量验证，支持两种执行模式。所有命令在 `src/tokenless/` 目录下运行。

> **平台前置**：tokenless 是 Linux-only 组件，本流水线必须在 Linux 平台执行（详见 [env-check.md](env-check.md) §1.5-A）。

> **rtk 预备**：所有 build/test 命令依赖 `third_party/rtk/`，首次执行前必须 `just setup-rtk` 完成 rtk 克隆 + patch（`make build` 已自动调用，`make lint`/`make test` 也走 justfile 的同名 recipe）。

## 分步模式

适用于增量开发场景（如 issue-fix），支持并行和重试。

### 每轮验证流程

**步骤 1（顺序执行）** — fmt 会修改文件，必须先行：

```bash
cd src/tokenless && make fmt
```

等价于：`cargo fmt -p tokenless-cli -p tokenless-schema -p tokenless-stats`。

**步骤 2-4（可并行）** — fmt 完成后，使用 Task 工具同时启动 3 个子 agent：

- 子 agent A：
  ```bash
  cd src/tokenless && cargo build --release
  ```
  > 不直接用 `make build`，避免触发 build-toon（`cargo install toon-format` 联网，本步骤不需要）；rtk 由 `cargo build` 触发的 build script 间接处理。如需 rtk 二进制也参与 lint/test，使用 `cargo build --release --manifest-path third_party/rtk/Cargo.toml`。

- 子 agent B：
  ```bash
  cd src/tokenless && cargo clippy -p tokenless-cli -p tokenless-schema -p tokenless-stats --all-targets -- -D warnings
  ```
  等价于 `make lint`（rtk 已 exclude）。0-warning 门。

- 子 agent C：
  ```bash
  cd src/tokenless && cargo test -p tokenless-cli -p tokenless-schema -p tokenless-stats --no-run
  ```
  仅编译测试，不运行（运行由步骤 5 串行执行，确保资源不冲突，例如 SQLite 临时文件）。

**步骤 5（顺序执行）** — 依赖步骤 2/4：

```bash
cd src/tokenless && cargo test -p tokenless-cli -p tokenless-schema -p tokenless-stats
```

等价于 `make test-tokenless`。

**步骤 6（条件触发）** — 当本次改动涉及 `adapters/tokenless/common/hooks/**` 或 `tests/run-all-tests.sh` 时必跑：

```bash
cd src/tokenless && bash tests/run-all-tests.sh
```

等价于 `make test-hooks`。前置：`jq`、`python3`、可选 `toon` 二进制（缺失时跳过 toon 子用例，不算失败）。

### 重试逻辑

- 全部通过 → 验证完成，继续后续流程。
- 任一失败 → 分析错误，修复后重试（从步骤 1 开始新一轮）。
- 最大重试次数由调用方指定（默认 3 轮）。
- 重试耗尽仍失败 → **保留当前代码**，向用户报告：
  - 失败的具体命令及完整错误输出
  - 已尝试的修复措施
  - 等待用户进一步指示后继续

## 一键模式

适用于完整验证场景（如 release），按顺序串行执行：

```bash
cd src/tokenless
make fmt
make build       # 内含 just setup-rtk + cargo build --release（tokenless + rtk）
make lint        # clippy -- -D warnings
make test        # test-tokenless + test-hooks
```

注意：
- `make build-toon` 涉及 `cargo install toon-format`（联网下载），不在本流程内强制；release 时若要确保 toon 二进制也最新，由 release.md Phase 4 显式触发。
- `make test` 含 hooks 集成测试（依赖 jq/python3）。release 流程必须确保这两个工具可用。
- 失败 → 输出完整错误信息，暂停等待用户决策（手动修复后告知继续 / 或中止流程）。

**注意**：rtk 是 vendored 上游源码（`[workspace] exclude = ["third_party/rtk"]`），**不参与** lint/test。命令必须显式 `-p <crate>`，否则会误扫 rtk。
