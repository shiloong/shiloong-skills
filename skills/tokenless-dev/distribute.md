# 分发打包流程（RPM 全链路）

tokenless (Rust) 组件从 GitHub Release 构建可安装 RPM / SRPM 的工作流。

> **交互原则**：源码同步前必须让用户选择自动/手动下载；RPM 仓库路径、spec 更新方式、SHA256 校验、构建断言缺项等关键点都需暂停等待用户决策。
> 
> **平台前置**：本流程涉及 `cargo` / `rpmbuild` 全链路，**必须在 Linux** 平台执行。env-check Phase 0 强制校验，不通过则中止建议切换到 Linux 远程开发机（用户内存中的 `dev`）。

## 输入

- `/tokenless-dev distribute` — 自动取最新 `tokenless/v*` release
- `/tokenless-dev distribute <version>` — 指定版本，支持 `0.3.2` / `v0.3.2` / `tokenless/v0.3.2` 三种写法
- 别名：`/tokenless-dev dist [version]` / `/tokenless-dev distribution [version]`

全流程变量约定：`PLAIN_VERSION` 表示纯版本号（如 `0.3.2`），`TAG` 表示完整 tag（如 `tokenless/v0.3.2`），两者一一对应。

## 跨仓库结构

本流程同时操作两个仓库：

| 变量 | 含义 | 典型路径 |
|------|------|---------|
| `$WS` | anolisa monorepo 根（含 `src/tokenless/` + `tokenless.spec.in`） | `~/workspace/anolisa` |
| `$WS_RPM` | 独立 tokenless RPM 打包仓库（含 `package-tokenless.sh` + 落地 `tokenless.spec`） | `~/workspace/tokenless` |
| `~/rpmbuild/` | 用户级 rpmbuild 工作根（不强制清理，避免污染其他项目） | `~/rpmbuild` |

## Phase 0：前置校验与上下文捕获

执行 [references/env-check.md](references/env-check.md) 调用模式矩阵中 `distribute` 列：git remote + gh auth + Linux 平台 + Rust 工具链 + rpmbuild（workspace 清洁度、sync main、cargo fetch 均跳过）。

| 检查项 | 命令 | 失败处理 |
|--------|------|----------|
| 在 anolisa 仓库内 | `git remote -v \| grep alibaba/anolisa` | 中止，提示 `cd` 到 anolisa 仓库根目录 |
| gh 已认证 | `GH_PAGER="" gh auth status` | 中止，建议 `gh auth login` |
| Linux 平台 | `[ "$(uname -s)" = "Linux" ]` | 中止，提示切到 Linux 远程开发机 |
| Rust ≥ 1.89 | `cargo --version` | 中止，提示升级（对齐 spec.in `BuildRequires`） |
| just 可用 | `command -v just` | 中止，提示 `cargo install just` |
| rpmbuild 可用 | `command -v rpmbuild` | 中止，提示 `sudo dnf install rpm-build` |
| 必要工具在 PATH | `command -v tar git python3 jq` 且 `command -v sha256sum \|\| command -v shasum` | 中止，提示安装 |

### 捕获初始上下文（在切入 `$WORKDIR` 之前必做）

```bash
WS="$(git rev-parse --show-toplevel)"               # anolisa 仓库根
mkdir -p "$WS/.qoder/tokenless-dev"                 # 临时文件根目录
if command -v sha256sum >/dev/null 2>&1; then
  SHA_CMD="sha256sum"                               # Linux 默认
else
  SHA_CMD="shasum -a 256"                           # 兜底
fi
```

### 确认 RPM 仓库路径 `$WS_RPM`

按 [references/interaction-guide.md](references/interaction-guide.md) 提问：先在正文展示候选路径与检测结果，再用极简 AskUserQuestion 让用户拍板。

候选路径推断顺序：
1. 用户在命令里显式带 `--rpm-dir <path>`（如有支持）
2. 环境变量 `$TOKENLESS_RPM_DIR`
3. 启发式：`$HOME/workspace/tokenless`、`$HOME/code/tokenless`
4. 询问用户手动指定

校验 `$WS_RPM`：
- `[ -d "$WS_RPM/.git" ]` — 是 git 仓库
- `[ -f "$WS_RPM/package-tokenless.sh" ]` — 含打包脚本
- `[ -f "$WS_RPM/tokenless.spec" ]` 或 `[ -f "$WS_RPM/tokenless.spec.in" ]` — 含 spec

任一不满足 → AskUserQuestion 让用户重新指定或中止。

遵循 [SKILL.md](SKILL.md) 全局约定「GH 命令规范」：所有 `gh` 调用前缀 `GH_PAGER=""`。
与用户的任何决策性交互遵循 [references/interaction-guide.md](references/interaction-guide.md)。

## Phase 1：确定目标版本与 release 元数据

### 步骤 1.1 — 列出可用 release

```bash
GH_PAGER="" gh release list --repo alibaba/anolisa --limit 30 | grep '^tokenless' | head -5
```

> 若无 `tokenless/v*` release，列出 `gh release list` 全部前 5 个，提示用户：tokenless 尚未发布过对应 release，可考虑直接从某个 tag 构建（见步骤 1.2 兜底）。

### 步骤 1.2 — 规范化版本号

**用户指定**：规范化输入：
- `0.3.2` → `TAG=tokenless/v0.3.2`，`PLAIN_VERSION=0.3.2`
- `v0.3.2` → 同上（去掉 `v` 前缀）
- `tokenless/v0.3.2` → 同上（去掉 `tokenless/v` 前缀）

校验 `PLAIN_VERSION` 必须匹配 `^\d+\.\d+\.\d+$`。格式不合法则中止。

**用户未指定**：取 `gh release list` 中第一个 `tokenless/v*` tag 作为 `TAG`，反解出 `PLAIN_VERSION`。

### 步骤 1.3 — 读取 release 元数据（可选）

```bash
GH_PAGER="" gh release view "$TAG" --repo alibaba/anolisa --json tagName,createdAt,assets,isDraft,isPrerelease
```

校验：
- `isDraft == false`（草稿 release 不允许 distribute）
- `tagName == $TAG`

> tokenless release 当前**未**自动上传源码 asset（不像 cosh 有官方分发 tarball），所以本流程不强依赖 release asset；源码通过下一阶段的浅克隆获取。若未来加上了 asset，可在此步骤记录 `EXPECTED_SHA256` 用于 Phase 2.5 校验源码包。

## Phase 2：浅克隆源码 & 准备 rtk

### 步骤 2.1 — 创建工作区

```bash
WORKDIR="$(mktemp -d "$WS/.qoder/tokenless-dev/distribute-${PLAIN_VERSION}.XXXXXX")"
cd "$WORKDIR"
```

全流程后续命令除非特别说明，均在 `$WORKDIR` 下执行。流程结束或失败后：
- 成功 → `rm -rf "$WORKDIR"`
- 失败/中止 → 保留 `$WORKDIR`，输出路径给用户

### 步骤 2.2 — 浅克隆 anolisa @ TAG

```bash
git clone --branch "$TAG" --depth 1 https://github.com/alibaba/anolisa.git anolisa
SRCDIR="$WORKDIR/anolisa/src/tokenless"
[ -d "$SRCDIR" ] || { echo "ERROR: src/tokenless 不存在 @ $TAG"; exit 1; }
```

失败处理：tag 不存在 / 网络失败 → 中止，提示用户检查 release 是否真实推送了 tag。

### 步骤 2.3 — 解析上游版本号一致性

```bash
UPSTREAM_VERSION=$(grep -E '^version[[:space:]]*=' "$SRCDIR/Cargo.toml" | head -1 | sed 's/.*"\([^"]*\)".*/\1/')
[ "$UPSTREAM_VERSION" = "$PLAIN_VERSION" ] || {
  echo "WARN: tag $TAG 中 Cargo.toml 版本 ($UPSTREAM_VERSION) 与请求版本 ($PLAIN_VERSION) 不一致"
  # 通过 AskUserQuestion 让用户决定：以 tag 为准 / 以 Cargo.toml 为准 / 中止
}
```

### 步骤 2.4 — 准备 rtk vendored 源码

```bash
cd "$SRCDIR"
just setup-rtk   # 内部执行：git clone rtk + 应用 patches/rtk-tokenless-stats.patch
```

`just setup-rtk` 当前在 patch 失败时仅打 WARNING；如果本次目标是发布 RPM，patch 失败需要**强制中止**（否则 rtk 行为可能与预期不符）：

```bash
# 校验 rtk 已就绪 + patch 已应用
[ -f "$SRCDIR/third_party/rtk/Cargo.toml" ] || { echo "ERROR: rtk 克隆失败"; exit 1; }
# patch 应用的标志：tokenless stats 相关符号出现在 rtk 源码中
grep -qr 'tokenless' "$SRCDIR/third_party/rtk/src/" 2>/dev/null || {
  echo "WARN: 未在 rtk 源码中检测到 tokenless 标记 — patch 可能未应用"
  # AskUserQuestion: 是否继续 (1. 继续 / 2. 中止)
}
```

## Phase 3：编译 & 生成源码 tarball

### 步骤 3.1 — 编译三个二进制

按 `package-tokenless.sh` 的 4-6 步执行，每步失败则中止并输出最后 40 行日志：

```bash
cd "$SRCDIR"

# tokenless
cargo build --release 2>&1 | tail -40

# rtk (用 --manifest-path，rtk 不在 workspace)
cargo build --release --manifest-path third_party/rtk/Cargo.toml 2>&1 | tail -40

# toon (从 crates.io 安装到 WORKDIR 局部根，避免污染用户全局 $HOME/.cargo/bin)
TOON_VER=$(grep -E '^TOON_VER[[:space:]]*:=' "$SRCDIR/Makefile" | head -1 | awk -F':=' '{print $2}' | tr -d ' ')
TOON_ROOT="$WORKDIR/toon-root"
cargo install toon-format --version "$TOON_VER" --root "$TOON_ROOT" --locked 2>&1 | tail -40
```

### 步骤 3.2 — 构建断言（缺一则中止）

| 断言 | 校验命令 |
|------|----------|
| `tokenless` 二进制存在 | `[ -x "$SRCDIR/target/release/tokenless" ]` |
| `rtk` 二进制存在 | `[ -x "$SRCDIR/third_party/rtk/target/release/rtk" ]` |
| `toon` 二进制存在 | `[ -x "$TOON_ROOT/bin/toon" ]` |
| 三个二进制可执行 | `"$SRCDIR/target/release/tokenless" --version && "$SRCDIR/third_party/rtk/target/release/rtk" --version && "$TOON_ROOT/bin/toon" --version` |
| 关键 adapter 资源完整 | `[ -d "$SRCDIR/adapters/tokenless" ] && [ -f "$SRCDIR/adapters/tokenless/manifest.json" ] && [ -f "$SRCDIR/adapters/tokenless/tokenless.spec.in" ]` |

> 注：当前 `tokenless.spec.in` 是从源码包中获取的（位于 `src/tokenless/`），与 RPM 仓库侧的 `tokenless.spec` 通过 `@VERSION@` 占位符派生。

### 步骤 3.3 — 准备源码 tarball

参考 `package-tokenless.sh` 第 7-8 步：把源码（含 vendored rtk）打 tar.gz。这里有**两种风格**可选，distribute 默认走"源码 + vendored rtk"风格（与上游 spec.in 的 `cargo build` 期望一致）：

```bash
TARBALL_NAME="tokenless-${PLAIN_VERSION}.tar.gz"
TARBALL_PATH="$WORKDIR/$TARBALL_NAME"

cd "$WORKDIR/anolisa/src"
# clean build 产物，保留 vendored 源码 + patch 应用结果
(cd tokenless && cargo clean --release 2>/dev/null || true)
tar -czf "$TARBALL_PATH" --exclude='target' --exclude='.git' tokenless
```

输出 tarball 大小 + sha256，写入 distribute 摘要。

## Phase 4：同步 RPM 仓库

切到 `$WS_RPM`，**不修改 git 历史**，只动跟当前版本相关的文件。

### 步骤 4.1 — 校验 RPM 仓库状态

```bash
cd "$WS_RPM"
git status --porcelain | grep . && {
  # 提示用户：RPM 仓库有未提交修改，建议先 commit/stash
  # AskUserQuestion: 继续（可能覆盖你的本地修改） / 中止
}
```

### 步骤 4.2 — 询问 spec 更新方式

按 [interaction-guide.md](references/interaction-guide.md)，先在正文展示对比信息（monorepo `spec.in` 的 changelog 顶部 vs `$WS_RPM/tokenless.spec` 的 changelog 顶部），再用极简 AskUserQuestion 提问：

选项：
- A. **自动同步**（推荐）：从 `$SRCDIR/tokenless.spec.in` 派生 `$WS_RPM/tokenless.spec`（`sed 's/@VERSION@/<PLAIN_VERSION>/g'`），覆盖现有 spec
- B. **手动维护**：跳过 spec 同步，使用 `$WS_RPM/tokenless.spec` 当前内容（需用户保证已 bump 到 `<PLAIN_VERSION>`）
- C. 取消

选 A 时：
```bash
sed "s/@VERSION@/${PLAIN_VERSION}/g" "$SRCDIR/tokenless.spec.in" > "$WS_RPM/tokenless.spec"
```

### 步骤 4.3 — 更新打包脚本 TAG

```bash
# 检测当前 TAG
CURRENT_TAG=$(grep -E '^TAG=' "$WS_RPM/package-tokenless.sh" | head -1 | sed 's/.*"\(.*\)".*/\1/')

if [ "$CURRENT_TAG" != "$TAG" ]; then
  # 通过 AskUserQuestion 让用户决定是否更新 TAG
  # 选 A → 直接 sed 修改
  sed -i.bak "s|^TAG=.*|TAG=\"$TAG\"|" "$WS_RPM/package-tokenless.sh" && rm -f "$WS_RPM/package-tokenless.sh.bak"
fi
```

### 步骤 4.4 — 把 tarball 拷贝到 RPM 仓库

```bash
cp "$TARBALL_PATH" "$WS_RPM/$TARBALL_NAME"
```

## Phase 5：rpmbuild

### 步骤 5.1 — 准备 rpmbuild 工作树

```bash
mkdir -p ~/rpmbuild/{BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS}
cp "$WS_RPM/$TARBALL_NAME" ~/rpmbuild/SOURCES/
cp "$WS_RPM/tokenless.spec" ~/rpmbuild/SPECS/tokenless.spec
```

### 步骤 5.2 — 执行 rpmbuild

```bash
RPMBUILD_LOG="$(mktemp "$WS/.qoder/tokenless-dev/distribute-${PLAIN_VERSION}.XXXXXX.rpmbuild.log")"

cd ~/rpmbuild
rpmbuild -ba SPECS/tokenless.spec 2>&1 | tee "$RPMBUILD_LOG"
```

> **默认带 BuildRequires 校验**（不加 `--nodeps`）；若用户的发行版没有 `BuildRequires: rust >= 1.89` 对应的可装包，AskUserQuestion 让用户决定：
> - A. 安装依赖（输出 `dnf install` 指引）
> - B. 退回到 `--nodeps`（用本地已装的 cargo / rustc，风险自担）
> - C. 中止

### 步骤 5.3 — 构建断言（缺一则中止）

| 断言 | 校验命令 |
|------|----------|
| 二进制 RPM 存在 | `ls ~/rpmbuild/RPMS/x86_64/tokenless-${PLAIN_VERSION}-*.x86_64.rpm` |
| SRPM 存在 | `ls ~/rpmbuild/SRPMS/tokenless-${PLAIN_VERSION}-*.src.rpm` |
| RPM 文件列表非空 | `rpm -qpl ~/rpmbuild/RPMS/x86_64/tokenless-${PLAIN_VERSION}-*.x86_64.rpm \| head -10` |
| 含 `/usr/bin/tokenless` | `rpm -qpl <rpm> \| grep -q '^/usr/bin/tokenless$'` |
| 含 `/usr/libexec/anolisa/tokenless/rtk` | `rpm -qpl <rpm> \| grep -q '^/usr/libexec/anolisa/tokenless/rtk$'` |
| 含 `/usr/libexec/anolisa/tokenless/toon` | `rpm -qpl <rpm> \| grep -q '^/usr/libexec/anolisa/tokenless/toon$'` |

任一断言失败 → 中止，输出 `$RPMBUILD_LOG` 最后 80 行供排查。

## Phase 6：落盘与输出

### 步骤 6.1 — 把 RPM/SRPM 拷贝到 RPM 仓库 packages/

```bash
mkdir -p "$WS_RPM/packages"
cp ~/rpmbuild/RPMS/x86_64/tokenless-${PLAIN_VERSION}-*.x86_64.rpm "$WS_RPM/packages/"
cp ~/rpmbuild/SRPMS/tokenless-${PLAIN_VERSION}-*.src.rpm "$WS_RPM/packages/"

OUT_RPM=$(ls "$WS_RPM/packages/tokenless-${PLAIN_VERSION}-"*.x86_64.rpm)
OUT_SRPM=$(ls "$WS_RPM/packages/tokenless-${PLAIN_VERSION}-"*.src.rpm)

OUT_RPM_SHA=$($SHA_CMD "$OUT_RPM" | awk '{print $1}')
OUT_SRPM_SHA=$($SHA_CMD "$OUT_SRPM" | awk '{print $1}')
```

### 步骤 6.2 — 校验 RPM 仓库的 git diff 范围

```bash
cd "$WS_RPM"
ALLOWED_PATTERNS='^\(tokenless\.spec\|package-tokenless\.sh\|tokenless-.*\.tar\.gz\|packages/.*\.rpm\)$'
UNEXPECTED=$(git status --porcelain | awk '{print $2}' | grep -vE "$ALLOWED_PATTERNS" || true)
if [ -n "$UNEXPECTED" ]; then
  echo "WARN: RPM 仓库出现非预期文件变更："
  echo "$UNEXPECTED"
  # 不主动 reset，但 flag 给用户在 6.4 摘要里
fi
```

### 步骤 6.3 — 清理临时目录

```bash
rm -rf "$WORKDIR"
rm -f "$RPMBUILD_LOG"
# ~/rpmbuild/ 由用户系统管理，不主动删
```

### 步骤 6.4 — 输出发布摘要

输出给用户的摘要需包含：

- **源**
  - tag：`<TAG>`
  - anolisa commit：`<git rev-parse HEAD>`（来自浅克隆的 anolisa 副本）
  - rtk version：`<rtk_tag from justfile>`
  - toon version：`<TOON_VER from Makefile>`
- **中间产物**
  - 源码 tarball：`$WS_RPM/<TARBALL_NAME>`
- **最终产物**
  - 二进制 RPM：`$OUT_RPM`，SHA256：`$OUT_RPM_SHA`
  - SRPM：`$OUT_SRPM`，SHA256：`$OUT_SRPM_SHA`
- **顶层清单**：`rpm -qpl $OUT_RPM | head -30`
- **RPM 仓库变更**：`cd "$WS_RPM" && git status --porcelain`（提示用户 commit/push spec.tarball.packages 变更，但**不自动 commit**）
- **安装命令示例**：
  ```bash
  sudo dnf install $OUT_RPM
  # 验证
  tokenless --version && rtk --version && toon --version
  ```

## 中止场景（distribute 流程专属）

| 条件 | 处理方式 |
|------|----------|
| 不在 anolisa 仓库内 | 中止，提示 `cd` 到仓库根目录 |
| `gh` 未认证 | 中止，提示 `gh auth login` |
| 平台非 Linux | 中止，提示用 Linux 远程开发机 |
| 缺 `cargo` / `just` / `rpmbuild` 等任一关键工具 | 中止，输出对应安装命令 |
| 指定版本无对应 tag | 中止，列出最近 5 个 `tokenless/v*` |
| `$WS_RPM` 路径不存在或缺关键文件 | 中止，AskUserQuestion 让用户重新指定 |
| `tokenless.spec.in @TAG` 与 RPM 仓库 spec 大幅偏离且用户选择"手动维护" | 警告，但允许继续 |
| 浅克隆失败（tag 不存在 / 网络） | 中止 |
| `just setup-rtk` 失败或 patch 未应用且用户选择中止 | 中止 |
| 任一 cargo 编译步骤失败 | 中止，输出最后 40 行日志 |
| 构建断言（步骤 3.2 / 5.3）缺项 | 中止，输出缺失项清单 |
| rpmbuild 失败 | 中止，输出 rpmbuild.log 最后 80 行 |
| 用户取消 spec 同步或 TAG 更新 | 视情况决定是否中止 |

## 不做的事

- 不修改 anolisa 仓库代码、不 commit、不推送
- **不**自动 commit RPM 仓库（`$WS_RPM`）的变更——只生成产物，用户决定是否 commit
- 不上传 RPM 到任何 yum/dnf repository 或 release asset
- 不清理 `~/rpmbuild/`（共享构建根，可能含其他项目产物）
- 不处理 `BuildRequires` 缺失（仅提示用户安装，不主动 `dnf install`）
- 不修复 vendored rtk 的 patch 失败（patch 失败需人工评估上游变更）
