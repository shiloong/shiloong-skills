---
name: spec-interview
description: Reverse-interview skill for clarifying vague requirements. Claude acts as a senior architect, asking targeted questions about edge cases, trade-offs, and constraints until enough information is gathered to produce a SPEC.md. Use when the user invokes /spec-interview or has an unclear feature idea.
version: 0.1.0
---

# Spec Interview — Reverse-Interview for Requirement Clarification

需求模糊时，Claude 以资深架构师视角反向访谈用户，澄清边界条件、异常场景、权衡取舍，访谈完成后产出 SPEC.md。

## Trigger

- `/spec-interview <功能描述>`
- "我想做 X，帮我澄清需求"
- "帮我写个 spec"
- 需求描述含模糊词（"大概"、"类似"、"差不多"、"看情况"）

## Phase 1: Initial Assessment

Read the user's feature description and assess clarity. If any of these are unclear, proceed to interview:

- What exactly does "X" mean? (scope boundary)
- Who uses this? (target audience / role)
- When does this happen? (trigger condition / lifecycle)
- What happens when it fails? (error handling / fallback)
- What are the constraints? (performance / security / compatibility)

**If all are clear**: Skip interview, directly produce SPEC.md.

**If any are unclear**: Enter Phase 2.

## Phase 2: Structured Interview

以 AskUserQuestion 工具向用户提问。每次最多 3-4 个问题（不一次轰炸），聚焦最关键的缺失信息。

### Interview dimensions (按优先级)

**P0 — 必须澄清（阻塞 spec 产出）**:
1. **Scope boundary**: 这个功能包含什么、不包含什么？最小可行范围？
2. **Input/output contract**: 输入是什么格式/来源？输出是什么格式/去向？
3. **Success criteria**: 怎样算"做完了"？可观察的验证方法？

**P1 — 应该澄清（影响设计质量）**:
4. **Error / failure scenarios**: 依赖服务挂了怎么办？输入不合法怎么办？超时怎么办？
5. **Security / permission constraints**: 是否涉及身份验证？权限边界？数据隔离？
6. **Performance requirements**: 响应时间？吞吐量？数据量级？

**P2 — 可以澄清（影响实现细节）**:
7. **Compatibility / migration**: 是否需要兼容旧版？迁移路径？
8. **Trade-off preferences**: 一致性 vs 可用性？安全 vs 方便？性能 vs 可维护性？
9. **Naming / convention preferences**: 术语偏好？命名约定？

### Interview rules

- 每轮最多 4 个问题，不要一次问 10 个
- 问题要具体可回答，不要模糊开放题（"你觉得应该怎么做？" → ❌；"超时时应该重试还是返回错误？" → ✅）
- 提供 2-4 个选项供选择（用 AskUserQuestion），不要只给空白输入
- 用户说"不确定"或"看情况"时，给出推荐选项并标注 "(Recommended)" 和理由
- 记录每轮回答，积累信息直到 P0 全部澄清

## Phase 3: SPEC.md Production

当 P0 全部澄清后，产出 `SPEC.md` 文件到当前工作目录。

### SPEC.md structure

```markdown
# SPEC: <Feature Name>

## Overview
<1-2 sentence description>

## Scope
### In scope
- <item 1>
- <item 2>

### Out of scope
- <item 1>
- <item 2>

## Input/Output Contract
### Input
- <format, source, validation rules>

### Output
- <format, destination, error responses>

## Success Criteria
- <observable verification method 1>
- <observable verification method 2>

## Error Handling
- <scenario 1>: <response>
- <scenario 2>: <response>

## Constraints
- <security constraint>
- <performance constraint>
- <compatibility constraint>

## Trade-offs Decided
- <decision 1>: chose <option A> over <option B> because <reason>

## Implementation Notes
- <key files to modify>
- <patterns to follow>
- <existing utilities to reuse>
```

## Principles

1. **需求不清时不要猜**: 猜测比问清楚更危险——猜错了方向白干一轮
2. **P0 阻塞 P1 不阻塞**: P0 信息不全不产出 spec；P1/P2 信息不全可以产出但标注待定
3. **给选项而非空白**: 用 AskUserQuestion 提供选项+推荐，降低用户回答成本
4. **记录而非记忆**: 每轮回答写入 spec 文件，不靠上下文"记住"
5. **一事一 spec**: 一个功能一个 spec，不要把多个功能混在一个 spec 里