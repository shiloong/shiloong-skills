# gh CLI 使用规范与临时文件约定

tokenless-dev 所有子流程调用 GitHub CLI (`gh`) 及产生临时文件 / 工作目录时遵循的统一规范。

## 一、gh CLI 使用规范

### 1.1 禁用交互式分页器

所有 `gh` 命令**必须**禁用 pager，否则会在终端中阻塞：

```bash
GH_PAGER="" gh <command>
# 或
gh <command> | cat
```

### 1.2 传递 body / payload 一律用文件

涉及较长 body（PR body、review body、JSON payload）的命令**禁止**用 `--body "<heredoc>"`，因为：
- 引号转义容易出错
- 大量换行在 shell 中可能被解释
- 非交互场景下更稳定

统一用 `--body-file <file>` / `--input <file>`：

```bash
# PR body
GH_PAGER="" gh pr create --body-file "$BODY_FILE" ...

# API JSON payload
GH_PAGER="" gh api --method POST /repos/... --input "$PAYLOAD_FILE"

# Issue / PR comment body
GH_PAGER="" gh pr comment <N> --body-file "$BODY_FILE" ...
```

### 1.3 常见命令与错误码

| 命令 | 场景 | 常见失败 |
|------|------|---------|
| `gh auth status` | 环境校验 | 未登录 → 中止，提示 `gh auth login` |
| `gh pr view <N> --json ...` | review / fix 前读 PR | PR 不存在 / 无权限 |
| `gh pr diff <N>` | review 拉 diff | — |
| `gh pr create --body-file` | fix / release 创建 PR | 分支未 push；head/base 错误 |
| `gh pr comment <N> --body-file` | AI 评论 | PR 已关闭 |
| `gh api POST /repos/.../pulls/<N>/reviews --input` | review 提交评论 | **422 Line could not be resolved** — line 不在 diff 内（详见 [review-submit.md](review-submit.md)） |
| `gh release list / view / download` | release / distribute | tag 已存在；asset 缺失 |
| `gh api repos/{owner}/{repo}/commits/{sha}/pulls` | release 反查 commit 对应的 PR | 速率限制（HTTP 403），按 `x-ratelimit-reset` 等待 |

---

## 二、临时文件 / 工作目录位置规范

### 2.1 统一位置

所有子流程的临时文件 / 工作目录**必须**落在 workspace 内，禁止使用 `/tmp/`：

```
<workspace>/.qoder/tokenless-dev/<flow>-<target>.XXXXXX.<ext>
```

组成元素：

| 元素 | 含义 | 示例 |
|------|------|------|
| `<workspace>` | anolisa 仓库根目录（通常为当前 `pwd`） | `/home/user/codes/anolisa` |
| `.qoder/tokenless-dev/` | 固定子目录，全流程共享，与 cosh-dev 隔离 | — |
| `<flow>` | 子流程标识 | `fix` / `release` / `distribute` / `review` |
| `<target>` | 关键业务标识（issue 号 / PR 号 / 版本号） | `290` / `213` / `0.3.2` |
| `XXXXXX` | `mktemp` 生成的随机后缀，并发/重试隔离 | `aB3xYz` |
| `<ext>` | 扩展名，按内容选择 | `md` / `json` / 无扩展（目录） |

### 2.2 为什么不用 `/tmp/`

1. **工具作用域限制**：Qoder / Claude Code 等 harness 的 file 写入工具默认禁止写入 workspace 外文件。若流程中混用 shell 命令和 file 工具，前者可写 `/tmp/`，后者不行，切换时易出错。
2. **可审计**：失败残留的 payload 在工作区内可被 git 状态直接看到（虽被 `.gitignore` 排除），排查方便。
3. **与 workspace 生命周期对齐**：workspace 切走后 `.qoder/tokenless-dev/` 自然隔离。

### 2.3 `.gitignore` 保护

anolisa 根目录的 `.gitignore` 已有 `.qoder/` 条目，`.qoder/tokenless-dev/` 天然被忽略，**无需额外配置**。

> 若用户在非 anolisa 仓库执行（理论上仅 distribute 的 RPM 仓库侧才会跨仓库），临时文件仍写在 anolisa 仓库的 `.qoder/tokenless-dev/`，避免污染 RPM 仓库工作区。

### 2.4 标准命令模板

```bash
# 定义 workspace（一次性写在子流程开头）
WS="$(git rev-parse --show-toplevel)"

# 确保子目录存在（首次运行）
mkdir -p "$WS/.qoder/tokenless-dev"

# 创建临时文件
BODY_FILE="$(mktemp "$WS/.qoder/tokenless-dev/fix-${ISSUE}-prbody.XXXXXX.md")"

# 创建工作目录（distribute 用）
WORKDIR="$(mktemp -d "$WS/.qoder/tokenless-dev/distribute-${PLAIN_VERSION}.XXXXXX")"
```

注：GNU `mktemp` (coreutils 8.x+) 支持 template 末尾带 `.ext` 后缀的写法。若脚本需兼容更老版本，改用 `--suffix=.ext` 形式。

### 2.5 各子流程命名约定

| 流程 | 场景 | 命名 |
|------|------|------|
| issue-fix | PR body 文件 | `fix-<ISSUE>-prbody.XXXXXX.md` |
| release | CHANGELOG / PR body / spec.in changelog 草稿 | `release-<VERSION>-<purpose>.XXXXXX.md` |
| distribute | 下载/构建/打包工作目录 | `distribute-<PLAIN_VERSION>.XXXXXX` (目录) |
| review | gh api reviews JSON payload / diff / 反例 scratch | `review-<PR_NUMBER>.XXXXXX.{json,diff,tsc.log}` / `review-<PR_NUMBER>-scratch.XXXXXX/` (目录) |

### 2.6 清理约定

**流程成功完成时**：
```bash
rm -f "$BODY_FILE"        # 单文件
rm -rf "$WORKDIR"         # 工作目录
```

**流程失败或用户中止时**：**保留**临时文件，输出其路径给用户，方便排查。清理责任转移到下次运行前的清理（或由用户手动删）。

**下次运行前自愈**（可选）：
```bash
# 清理 24 小时前的残留（谨慎启用，distribute WORKDIR 下可能有 tar.gz / rpm 产物）
find "$WS/.qoder/tokenless-dev" -mindepth 1 -mmin +1440 -exec rm -rf {} + 2>/dev/null || true
```

### 2.7 硬门：不得污染 git working tree

子流程结束前**必须**验证：

```bash
# 1. .qoder/tokenless-dev/ 不被 git 追踪（已由 .gitignore 保证）
git check-ignore -q "$WS/.qoder/tokenless-dev/" && echo "ok"

# 2. git status 在 src/tokenless/** 范围内干净（review / distribute 是只读流程）
git status --porcelain -- src/tokenless/ | grep . && {
  echo "ERROR: src/tokenless/ 有未预期改动"; exit 1;
} || echo "ok"
```

这一条对 **review 流程**尤其关键（review 是只读的，不应产生任何 tokenless 代码改动）。详见 [review-submit.md](review-submit.md) 的"硬门 checklist"。

### 2.8 distribute 流程的跨仓库特例

distribute 同时操作两个仓库（anolisa 源 + 独立 tokenless RPM 仓库 `$WS_RPM`）：

- 临时下载/解压/构建目录：始终落在 anolisa 侧的 `$WS/.qoder/tokenless-dev/`
- RPM 仓库侧的产物（`tokenless-<v>.tar.gz`、`tokenless.spec`、`packages/*.rpm`）：由 distribute Phase 4-6 显式 `cp` 到 `$WS_RPM/` 对应位置，并在结束前 `git status -C "$WS_RPM"` 校验只动了预期文件
- `~/rpmbuild/` 由用户系统管理，distribute 不主动清理（避免污染其他 RPM 项目的中间产物）
