---
name: code-review
description: 系统性代码 review 专家。基于 Google/Microsoft 工程实践与 Rust 社区规范,从安全性、正确性、性能、可维护性、一致性五个维度审查代码变更。自动关联 git 变更上下文。
triggers:
  - "code review"
  - "review"
  - "审查"
  - "review code"
  - "review this"
  - "code review checklist"
  - "review 分支"
  - "review branch"
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash(git:*)
  - Bash(cargo:*)
effort: high
tags: [review, quality, security, rust, checklist]
---

# Code Review Skill

基于 **Google Engineering Practices** (google.github.io/eng-practices), **Microsoft Code Review Checklist**, **Thoughtbot Code Review Guide**, **Atlassian Code Review Best Practices** 等业界标准,融合 Rust 社区规范的综合代码审查方法论。

## 核心原则

### 1. 审查优先序 (Google Standard)

审查时按以下顺序关注,不应颠倒:

**Design > Functionality > Complexity > Tests > Naming > Comments > Style**

- **Design (设计)**: 整体方案是否合理?是否有更简方案?是否需要改动架构?
- **Functionality (功能正确性)**: 是否实现了预期功能?边界情况、错误路径是否覆盖?
- **Complexity (复杂度)**: 是否过度工程?是否有不必要的抽象?200行能变50行?
- **Tests (测试)**: 覆盖是否适当?是否有边界/异常/并发测试?
- **Naming (命名)**: 是否自解释?是否遵循项目约定?
- **Comments (注释)**: 是否解释了"为什么"而非"是什么"?死注释是否清理?
- **Style (风格)**: 是否一致?是否符合项目规范?

### 2. 速度与粒度 (Google + Thoughtbot)

- **每次 review ≤ 400 行** (超过应要求拆分)
- **每个 reviewer 每小时 review ≤ 500 行** (超过注意力下降)
- **review 响应 ≤ 24 小时** (business day)
- **PR 生命周期 ≤ 48 小时** (避免 context switching 成本)

### 3. 审查态度 (Thoughtbot + Atlassian)

- **假设开发者是善意的**: 指出的每个问题都是技术性的,非人身
- **区分必须修复 vs 建议**: 用 `nit:`/`suggestion:`/`must:` 标记严重度
- **接受"足够好"的非完美方案**: 完美主义导致交付延迟
- **学会说"I don't know"**: 不确定的事注明,不要假装懂
- **对替代方案保持开放**: "这里可以考虑 X" vs "这里必须用 X"

## 五维审查清单

### 维度 A: 安全性 (Security) — 阻断级

每个发现须逐一确认,有疑问即阻断。

#### A1. 输入验证
- [ ] 所有外部输入(用户/API/文件/网络)是否有验证?
- [ ] 路径遍历是否防御? (`../` 绕过, 符号链接)
- [ ] 反序列化是否限制类型? (serde 反序列化任意类型风险)
- [ ] 整数溢出/下溢是否处理? (saturating/wrapping/checked ops)
- [ ] 格式化字符串注入? (println!/format! 中的用户输入)

#### A2. 命令/SQL 注入
- [ ] 拼接 shell 命令? → 必须用 `std::process::Command` 传参列表
- [ ] SQL 拼接? → 必须用参数化查询 (`rusqlite::params![]`)
- [ ] 外部命令参数是否正确转义?

#### A3. 内存安全 (Rust-Specific)
- [ ] 是否存在 `unsafe` 块? → 每个 unsafe 必须有 SAFETY 注释阐述不变量证明
- [ ] `unwrap()` / `expect()` 在生产路径? → 替换为 `?` + `.context()`
- [ ] 是否存在 `unsafe` 的 FFI 调用? → 检查边界/生命周期/对齐
- [ ] raw pointer 操作是否正确? → deref 时机与生命周期

#### A4. 敏感数据
- [ ] 日志/输出中是否暴露密钥/token/密码?
- [ ] 环境变量中是否有硬编码凭证?
- [ ] 错误消息是否泄露内部实现细节?

#### A5. 并发安全 (Rust-Specific)
- [ ] `Mutex`/`RwLock` 中毒是否正确恢复?
- [ ] `Send`/`Sync` 推导是否正确? (unsafe impl 必须验证)
- [ ] 是否存在 data race 风险? (共享可变状态)
- [ ] channel deadlock? (send/recv 配对)

### 维度 B: 正确性 (Correctness) — 阻断级

#### B1. 逻辑正确性
- [ ] 条件分支是否完整互斥? (if-else 链无覆盖漏洞)
- [ ] 循环终止条件是否正确? (是否可能无限循环?)
- [ ] 边界条件: 空值/零值/极大值/负值/null/空数组/空字符串
- [ ] off-by-one: 索引/截断位置/长度计算

#### B2. 错误处理
- [ ] 所有 `Result` 是否正确传播? (禁止无声吞掉 Err)
- [ ] `Option` 的 None 分支是否合理处理?
- [ ] 错误信息是否有足够上下文? (`.context()` 而非裸 `?`)
- [ ] panic 是否仅用于不可恢复情况? (禁止 panic 在生产路径)
- [ ] match 是否完备(exhaustive)? (缺分支的警告是否处理)

#### B3. 类型正确性
- [ ] 类型转换是否安全? (as 转换是否可能截断: `u64 as usize` on 32-bit)
- [ ] `FromStr`/`TryFrom` 实现是否处理了非法输入?
- [ ] 数字类型选择是否合适? (usize vs u64 vs i64 选择的语义)

#### B4. 资源管理
- [ ] 文件 handle 是否泄漏? (是否正确 close/drop?)
- [ ] 数据库连接是否正确管理? (连接池/超时/重连)
- [ ] 临时文件是否清理? (Drop/RAII 实现)

### 维度 C: 性能 (Performance) — 警告级

#### C1. 不必要的分配
- [ ] `clone()` 是否必要? → 是否可以 borrow `&T` 代替?
- [ ] `.to_string()` 是否在热路径? → strlen 不变的用 `&str`
- [ ] `Vec` 预分配? → 已知容量用 `Vec::with_capacity()`
- [ ] 循环内创建 String? → 移到循环外或复用 buffer
- [ ] `Box`/`Arc` 是否是必要开销?

#### C2. 计算复用
- [ ] 重复计算? → 提至循环外或缓存 (如 Regex 编译)
- [ ] 热路径中的序列化/反序列化? → 是否可避免?
- [ ] 不必要的排序/收集? → 惰性迭代器 (`iter()` vs `into_iter()`)
- [ ] Regex 是否用 `lazy_static!` / `LazyLock` 缓存?

#### C3. IO 与阻塞
- [ ] 不必要的 syscall? (多次 read/write vs 批量)
- [ ] 同步 IO 阻塞主流程? → 考虑 async (如适用)
- [ ] 大文件读取是否流式? (vs `read_to_string` 全部加进内存)

#### C4. 数据结构选择
- [ ] `Vec` vs `VecDeque` vs `LinkedList`?
- [ ] `HashMap` vs `BTreeMap`? (有序 vs 无序)
- [ ] `HashSet` vs `BTreeSet`?
- [ ] `String` vs `&str` vs `Cow<str>`?

### 维度 D: 可维护性 (Maintainability) — 建议级

#### D1. 不要的复杂度
- [ ] 是否存在过度抽象? → 只为当前需求写代码
- [ ] 是否引入不必要的 trait? → 单实现者不要 trait bound
- [ ] 是否有不必要的泛型? → 单类型不用泛型参数
- [ ] 是否有未使用的依赖/导入?

#### D2. 重复代码 (DRY by 3)
- [ ] 相同逻辑出现 ≥3 次? → 提取函数/方法
- [ ] 相同的结构体模式? → 考虑提取公共字段
- [ ] 测试中的重复 setup? → 提取 test helper/fixture

#### D3. 命名
- [ ] 是否能望文生义? (3个月后回来读还能懂)
- [ ] 是否与项目约定一致? (如 `with_*` builder patterns)
- [ ] 是否避免了缩写? (除项目约定的如 `fn`/`mod`/`cfg`)
- [ ] 是否避免了误导性命名? (`calculate` 函数不能有副作用)

#### D4. 注释
- [ ] 注释解释 "why" 而非 "what"? (代码已经说了"what")
- [ ] 是否有误导性/过时注释?
- [ ] 是否有被注释掉的代码块? (应删除,有 git 历史)
- [ ] 是否有 TODO/FIXME 无 ticket 编号?

#### D5. 模块边界
- [ ] 公开 API 是否最小化? (不该 pub 的内在不暴露)
- [ ] 模块职责是否单一明确?
- [ ] 循环依赖是否存在?

### 维度 E: 一致性 (Consistency) — 建议级

#### E1. 与项目约束一致
- [ ] Commit message 格式? (`type(scope): description`) 
- [ ] 分支命名? (`feature|fix/...`)
- [ ] 版本号规范? (SemVer)
- [ ] 代码风格工具通过? (`cargo fmt --check`, `cargo clippy`)

#### E2. 与现有代码风格一致
- [ ] 错误处理模式一致? (anyhow vs thiserror vs 自定义)
- [ ] 命名约定一致? (snake_case fields, CamelCase types)
- [ ] 测试模式一致? (unit vs integration vs fixture)
- [ ] 日志风格一致? (eprintln! vs tracing vs log crate)

#### E3. Configuration/Feature Flag 一致性
- [ ] 新配置项与现有配置结构一致?
- [ ] 环境变量命名模式一致? (`TOKENLESS_*` 前缀)
- [ ] Feature gate 命名一致?

## Rust 专项审查检查点

### R1. 所有权与借用
```rust
// ❌ 不必要的 clone
fn bad(input: &str) -> String {
    let owned = input.to_string();  // clone 复制
    owned.replace("a", "b")
}

// ✅ 借用已足够
fn good(input: &str) -> String {
    input.replace("a", "b")
}
```

### R2. 迭代器优于循环
```rust
// ❌ 手动 push
let mut result = Vec::new();
for x in items { result.push(transform(x)); }

// ✅ Iterator
let result: Vec<_> = items.iter().map(transform).collect();
```

### R3. 模式匹配完备性
```rust
// ❌ 使用 if let 但不处理 else (非 Rust 惯例)
if let Some(v) = opt { do_something(v); }  // None silently skipped

// ✅ match 完备, or if let + else
match opt {
    Some(v) => do_something(v),
    None => handle_none(),  // 显式处理
}
```

### R4. 泛型约束合理
```rust
// ❌ 过度泛型
fn process<T: AsRef<str>>(input: T) -> String { input.as_ref().to_uppercase() }

// ✅ 参数只需 &str 就用 &str
fn process(input: &str) -> String { input.to_uppercase() }
```

### R5. 派生合理
- 非必要的 `Clone`/`Copy` 派生增加维护负担
- `Default` 是否语义合理?
- `PartialEq`/`Eq` 的比较逻辑是否匹配业务?

## 审查工作流

### Step 0: 环境准备
```bash
pwd                                  # 确认工作目录
git branch --show-current            # 确认分支
git status                           # 确认干净工作区
```

### Step 1: 变更范围理解
```bash
# 了解要审查的内容量
git diff --stat origin/main...HEAD
git log --oneline origin/main...HEAD

# 识别高风险文件
git diff --stat origin/main...HEAD | awk '/\.rs$|\.py$|\.ts$/{print $0}'
```

### Step 2: 逐文件阅读 (按复杂度排序)
1. 先读公开 API 变更 (lib.rs / mod.rs / 新增 trait/struct)
2. 再读业务逻辑变更 (核心算法/流程)
3. 最后读配置文件/测试变更

### Step 3: 质量门禁
```bash
# Rust 项目
cargo check && cargo fmt --check
cargo clippy --workspace -- -D warnings
cargo test --workspace
cargo audit  # 如安装了 cargo-audit

# Python 项目
ruff check && black --check

# TypeScript 项目
npm run lint && npm run build && npx tsc --noEmit
```

### Step 4: 逐维审查
按 5 个维度 (A→E) 顺序过一遍,每个发现记录: `文件:行号 — 严重度 — 问题描述 — 建议修复`

### Step 5: 输出审查报告
格式:
```
## Code Review Report — <branch/PR name>

### Summary
- 审查文件: N
- 变更行数: +M / -K
- 严重问题: P 项 (阻断)
- 警告: Q 项
- 建议: R 项

### 严重问题 (Must Fix)
[编号] 文件:行号 — 描述 — 建议

### 警告 (Should Fix)
...

### 建议 (Nice to Have)
...

### 亮点
(值得保持的做法)
```

## 严重度标记约定

| 标记 | 含义 | 操作 |
|------|------|------|
| `MUST` | 阻断级 — 安全漏洞/逻辑错误/崩溃风险 | 必须修复后才能合并 |
| `SHOULD` | 警告级 — 性能问题/设计缺陷/潜在风险 | 应在合并前修复或记录 issue |
| `NIT` | 建议级 — 风格/命名/注释优化 | 作者自行判断是否采纳 |
| `PRAISE` | 正面反馈 | 值得保持的做法 |

## 注意事项

- **信任编译器和 linter 做工具的事**: 不要人工检查代码风格 (cargo fmt/clippy 已处理)
- **人工专注工具做不了的事**: 设计合理/逻辑正确/安全漏洞/架构一致性
- **一次 review 只给一个判断**: approve / request changes / comment (不犹豫)
- **review comment 必须可操作**: 每个建议附带具体示例代码或修改方向
- **如果一个 pattern 出现 ≥3 次,统一改而非逐一 comment**

## Design Doc 审查 (大型 PR 必读)

如果变更较大(>400 行),检查是否有设计文档:
- 设计是否与项目架构一致?
- 是否考虑替代方案? (README/Design Doc 中记录)
- 是否有迁移路径? (breaking change 的降级方案)
- 是否与其他组件存在接口冲突?

## 回溯与沉淀

审查完成后,若有通用模式/经验教训,保存到项目记忆中:
- 常见错误模式 → feedback memory (提醒未来审查注意)
- 安全漏洞类型 → security-review skill 更新
- 架构关键决策 → project memory