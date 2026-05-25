# 环境验证

所有子流程的前置检查，确保工作环境就绪。

## 调用模式矩阵

各子流程按自身需要**裁剪**执行，未打勾的步骤直接跳过：

| 调用方 | 1.1-A git remote | 1.1-B gh auth | 1.1-C workspace clean | 1.4 sync main | 1.5-A Linux | 1.5-B Rust toolchain | 1.5-C rpmbuild |
|-----------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| fix       | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⬜ |
| release   | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⬜ |
| distribute| ✅ | ✅ | ⬜ | ⬜ | ✅ | ✅ | ✅ |
| review    | ✅ | ✅ | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |

调用方引用本文件时，明确声明执行的列即可，不必在各自文档中重复全部命令。

## 步骤 1.1-1.3（可并行）

使用 Task 工具同时启动 3 个子 agent：

- **子 agent A** — git remote 校验：
  ```bash
  git remote -v
  ```
  输出必须包含 `alibaba/anolisa`，否则中止：提示用户 `cd` 到仓库目录。

- **子 agent B** — gh 认证校验：
  ```bash
  GH_PAGER="" gh auth status
  ```
  必须已认证，否则中止：建议运行 `gh auth login`。

- **子 agent C** — 工作区校验：
  ```bash
  git status
  ```
  工作区必须干净（允许 untracked files，但不能有未提交的修改），否则中止：提示用户 commit 或 stash。

三个子 agent 全部返回后，**任一失败则中止**。

## 步骤 1.4（顺序执行）

同步 main 分支：

```bash
git checkout main && git pull --ff-only origin main
```

如果 pull 失败（如有冲突），中止并提示用户手动解决。

## 步骤 1.5（平台/工具链校验，按调用矩阵执行）

可并行启动 3 个子 agent（按需）：

- **子 agent A** — Linux 平台校验：
  ```bash
  uname -s
  ```
  必须为 `Linux`，否则中止：
  - 提示用户：tokenless 是 Linux-only 组件，本流程涉及 cargo/rpmbuild
  - 建议：通过 SSH 连接 Linux 远程开发机（用户内存中已记录开发机 `dev`）执行
  - 退路：用户可用 `--skip-platform-check` 显式跳过（但 build/test 失败概率高）

- **子 agent B** — Rust 工具链：
  ```bash
  cargo --version
  rustc --version
  command -v just || echo "just not installed"
  ```
  要求：
  - `cargo` ≥ 1.89（anolisa `src/tokenless/tokenless.spec.in` 要求 `BuildRequires: rust >= 1.89`）
  - `just` 可用（rtk setup 必需）
  - 缺失则中止，提示安装：`curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh` / `cargo install just`

- **子 agent C**（仅 distribute）— rpmbuild：
  ```bash
  command -v rpmbuild
  command -v rpm
  ```
  缺失则中止，提示安装：`sudo dnf install rpm-build rpmdevtools`（或 apt 等价物）

三个子 agent 全部返回后，**任一失败则中止**（按调用矩阵剪裁过的子集）。

## 步骤 1.6（可选 — 调用方决定）

预热 cargo 依赖：

```bash
cd src/tokenless && cargo fetch
```

主要服务于 fix 子流程的 Phase 1 末尾。若网络受限或仓库已 fetch 过，可跳过；Phase 5 build/test 阶段会按需自动 fetch。

调用方通过上方的**调用模式矩阵**决定是否执行此步骤。fix 默认执行；release / distribute / review 跳过（distribute 在 Phase 2 同步源码时另行处理）。
