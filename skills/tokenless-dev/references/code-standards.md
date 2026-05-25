# 代码规范

tokenless 组件的编码规范与版权头规则。

## 语言与工具链

- **Rust**（edition 2024，`rustc` ≥ 1.89；详见 `src/tokenless/Cargo.toml` 和 `tokenless.spec.in`）
- **Python**（adapter 脚本，目前未强制 ruff/black，但需保持与现有风格一致）
- **Shell**（adapter 脚本，必须 `set -euo pipefail`）
- 所有代码和注释使用**英文**
- 由 `make fmt` / `make lint` 强制（配置在 `src/tokenless/`）

## Rust 关键规则

| 规则 | 配置 |
|------|------|
| `cargo fmt` | 必跑，CI 强制 |
| `cargo clippy -- -D warnings` | 任何 warning 等同 error，0 warning 门槛 |
| `unwrap()` / `expect()` | lib 代码禁用（`crates/tokenless-cli/src/` 和 `crates/tokenless-schema/src/` 中），仅允许在 tests/examples |
| 错误处理 | 使用 `thiserror` 派生错误类型 + `?` 操作符传播 |
| 公共 API | 所有 `pub fn` / `pub struct` / `pub enum` 必须有 `///` doc comment |
| 字符串 panic | 禁止 `panic!("...")` 字符串字面量，使用错误类型 |

其他约定：

- 跨 crate 引用使用包名（如 `tokenless_schema::SchemaCompressor`），不用相对路径
- 版本号在 workspace 级声明：`[workspace.package].version`；crate 自己的 `Cargo.toml` 用 `version.workspace = true` 继承
- rtk 不在 workspace（`[workspace] exclude = ["third_party/rtk"]`），clippy/test 必须用 `-p <crate>` 显式指定 crate

## Adapter 脚本规则

### Python (`adapters/tokenless/common/hooks/*.py`)

- Python 3，shebang `#!/usr/bin/env python3`
- 错误处理：所有 hook 必须 **fail-open**（缺失依赖、解析失败等情况退回到原始 input/output，不阻塞调用方）
- stderr 日志：用 `print(..., file=sys.stderr)` 输出可观测信息，不要打到 stdout（hook 协议要求 stdout 是结构化输出）
- 不引入第三方包依赖（系统 Python 即可运行）

### Shell (`adapters/tokenless/common/hooks/*.sh` / `tokenless-env-fix.sh`)

- `#!/bin/bash` + `set -euo pipefail`
- 检测依赖用 `command -v <bin> >/dev/null 2>&1` 而非 `which`
- 失败必须显式 `exit <code>`，不能依赖 `set -e` 隐式退出
- 跨发行版兼容：包管理器分支（apt/dnf/pacman/cargo）按 `command -v <pm>` 自动选择

## 版权头规则

tokenless 主要使用 Apache-2.0（详见 `src/tokenless/LICENSE`）。SPDX 头格式：

```rust
// SPDX-License-Identifier: Apache-2.0
// Copyright <当前年份> Alibaba Cloud
```

```python
# SPDX-License-Identifier: Apache-2.0
# Copyright <当前年份> Alibaba Cloud
```

```bash
#!/bin/bash
# SPDX-License-Identifier: Apache-2.0
# Copyright <当前年份> Alibaba Cloud
```

年份通过 `date +%Y` 动态获取。

| 场景 | 版权行 |
|------|--------|
| 新增文件 | `Copyright <当前年份> Alibaba Cloud` |
| 修改文件，变更量 > 文件总行数的 50% | `Copyright <当前年份> Alibaba Cloud`（替换原版权年份/主体） |
| 修改文件，变更量 ≤ 文件总行数的 50% | 保持原版权头不变 |

修改比例计算方式：`git diff --stat` 的变更行数 / `wc -l` 的文件总行数。

**例外**：`third_party/rtk/` 是 vendored 上游源码，**不修改其版权头**；任何针对 rtk 的修改通过 `third_party/patches/*.patch` 维护。

## 规范阅读清单

实施修复或开发前，阅读以下文件获取最新规范与上下文：

- `AGENT.md`（anolisa 仓库根） — commit 格式（强制 scope=tokenless）、分支命名、PR 规范
- `src/tokenless/README.md` — 整体架构、构建系统、三适配 target 概览
- `src/tokenless/CHANGELOG.md` — 最近发版的变更记录（理解当前 baseline）
- `src/tokenless/Cargo.toml` — workspace 配置、版本号、依赖
- `src/tokenless/Makefile` 与 `src/tokenless/justfile` — 构建/测试命令的权威来源
- `src/tokenless/tokenless.spec.in` — RPM 打包规则 + FHS 安装路径
