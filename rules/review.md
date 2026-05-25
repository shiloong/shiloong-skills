# Review 经验

Review 时按以下逻辑体系逐层审查，输出结论分三级：阻塞(block) / 建议修(suggest) / 可选清理(clean)。

## 1. 安全与信任链

- **不可伪造的身份源**: 凡依赖用户可控输入（`$HOME`/env var/CLI arg）推导身份（uid/gid/权限），必须用 syscall 或不可篡改源。`$HOME`/`dirs::home_dir()` 可被任意改写，不构成信任锚。应直接用 `libc::getuid()`/`rustix::process::getuid()` 等 OS syscall。
- **信任链传导**: 身份推断 → 文件所有者校验 → 信任判定。锚点一旦可伪造，后续全部失效。审查时画出完整信任链，检查每一步是否可被攻击者中断或注入。
- **静默降级**: 错误时 fallback 到高权限值（如 `unwrap_or(0)` 返回 root uid）比 crash 更危险。审查 fallback 值的权限语义。

## 2. 错误传播与静默忽略

- **构建步骤失败不应静默继续**: patch/compile/install 失败只打 WARNING 后继续 → 产物功能缺失但构建"成功" → 用户无感知。关键步骤（patch 应用、二进制安装、schema 迁移）失败必须 exit/hard fail。
- **`2>/dev/null || true` 审查**: 安装/部署步骤用此模式静默忽略失败 → 缺失组件被 symlink 指向空 → 运行时才暴露。install 步骤失败应中断构建，不应吞掉错误。
- **依赖缺失 hard fail vs warn**: 构建必需依赖（just/toon/jq）缺失 → die；运行时可选依赖缺失 → warn 并降级。区分场景，不一刀切。

## 3. 注释与实际一致性

- **"no network needed" 声明**: 凡 `cargo install`/`pip install`/`npm install` 步骤默认联网。声称"no network needed"时，必须有 `--offline` + vendored source 佐证。否则改注释承认联网需求，并确保构建环境有镜像源。
- **版本约束注释**: `BuildRequires: rust >= X.Y` 注释必须解释为什么是这个版本而非更低的。注释与实际约束矛盾（注释说 >=1.86 但 spec 写 >=1.89）时，补全解释（edition/API stability 等原因）。

## 4. 构建依赖闭环

- **首次用户体验**: `do_install_deps` 安装的依赖必须覆盖 `build` 步骤的所有前提。如果 build 需要 `just`/`toon`/特定 Rust 版本，deps 步骤必须安装它们。审查时从 build recipe 逆推所有前提，与 deps 步骤做集合差，差集即为遗漏。
- **依赖版本最低要求注释化**: 每个 `REQUIRED="X.Y.Z"` 应有注释说明触发原因（edition/feature/API 等），便于后续版本升级时判断是否可降低。

## 5. 跨文件重复与一致性

- **常量去重**: 同一常量（fallback 路径、版本号）在 N 个文件各定义一份 → 改一处漏其余。抽到共享模块/常量文件。跨语言重复（Rust/Python）若完全去重成本过高，至少 Python 端内部去重。
- **thiserror 属性冗余**: `#[from]` 已隐含 `#[source]`，同时标注两者冗余。thiserror 文档明确声明此规则。

## 6. 构建产物验证

- **patch fuzz 验证**: `patch --forward` 允许 fuzz 匹配 → context 行偏移仍成功但可能 patch 到错误位置。CI/发布前必须跑一次干净构建并验证 patch 精确匹配（0-fuzz），贴输出到 PR description。
- **缩进一致性**: shell 脚本 `else` 分支内行无缩进、注释缩进不一致 → 影响可读性但不影响功能。低优先级但应在改动触及该文件时顺手修。

## 7. 保留路径/写目标治理

- **保留路径集合完整性**: 路径校验只挡 `.anolisa/` 首段 → `.gitignore`/`.git/` 等功能关键文件仍可被 mem_write 覆盖或清空 → 保护机制被绕过。审查时枚举 mount root 下所有"功能依赖文件"（版本控制 .gitignore、身份 .git/、配置文件等），确认保留路径集合覆盖全部不可由工具写操作篡改的目标。视角: **信任链延伸**——保护 `.anolisa/` 只是锚点，锚点外围的功能依赖同样在信任链上，缺一即断。
- **写目标语义审查**: 每个写操作（mem_write/append/edit）的合法目标域应与保留路径集合互补。审查时从写操作入口逆推，检查是否所有保留路径在任何写路径下都不可达（包括 append/edit 等非全量写）。

## 8. 服务部署硬化与最小权限

- **systemd unit 全集审查**: 有 `NoNewPrivileges`/`PrivateTmp` 但缺 `ProtectKernelTunables`/`SystemCallFilter`/`LockPersonality`/`MemoryDenyWriteExecute`/`RestrictAddressFamilies`/`ReadOnlyPaths+ReadWritePaths` 等。审查时对照 systemd hardened service 清单逐项勾选，对每项缺失给出"不可加"的理由（如 `ProtectControlGroups` 与 `Delegate=` 冲突）或补加。
- **运行时目录持久化**: `/run` 下目录（session dir 等）在 tmpfs 上重启丢失，但代码 `create_dir_all` 可能递归创建。审查时区分：systemd 单元启动 → 需要 tmpfiles.d snippet 保证父目录存在；CLI 直接运行 → `create_dir_all` 备用路径够用。两种场景都要验证。
- **cgroup 写操作副作用边界**: 写 `+memory` 到 parent `cgroup.subtree_control` 在 delegated scope 内安全，但在共享 parent cgroup 下影响兄弟单元。审查时检查：① 是否检测运行环境（delegated vs 菜单）做 short-circuit；② warn-on-failure 的日志是否可让运维关联到"兄弟单元受影响"。
- **network 地址簇限制**: stdio-only 服务仍可 `AF_INET`/`AF_INET6` → 容器/受限环境下无意义的网络能力。`RestrictAddressFamilies=AF_UNIX` 是最小集，审查时根据服务 transport（stdio → AF_UNIX only, HTTP → +AF_INET）收紧。

## 9. 数据边界与资源上限

- **无 cap 的读操作**: `mem_read` 返回全文件内容，无 size limit → 多 GB 文件灌入 JSON-RPC response → 内存/延迟双重冲击。审查时对所有"返回全量数据"的工具检查：是否有可配置上限（`MAX_READ_BYTES` 等）；无上限时计算攻击者可触发的最大内存分配。
- **降级语义的运维可观测性**: `strategy=auto` 降级到 userland 时 `tracing::warn!` + `info` 输出区分 `(configured: auto)` vs actual，但测试只断言 `contains("mount strategy")` 不验证具体值。审查时对所有"意图 ≠ 实际"的降级路径，要求 info/diagnostic 输出同时暴露意图和实际，且测试断言验证两者。

## 10. Profile/策略边界声明

- **visibility-only gating 的部署声明**: `list_tools` 按 profile 过滤但 `call_tool` 不拦截 → off-profile 工具仍可被已配置客户端调用。代码注释已说明设计意图，但运维文档（README/spec）未声明 → 部署者误以为 profile=basic 是安全边界。审查时对所有"只影响可见性不影响可达性"的策略门控，要求运维可读的文档明确声明边界语义。

## 11. 文档/代码用词漂移

- **比喻性用词 vs 实际实现**: 模块 doc "spawns a best-effort commit" → 实际 inline 调用，无 `spawn_blocking`。开发者理解"spawn"为比喻，但新贡献者/审查者可能误认为异步。审查时对所有 doc/comment 中含"spawn"/"offload"/"async"/"background"等暗示异步的用词，确认是否与实现一致；不一致时改用精确动词（"performs"/"runs inline"/"calls"）。