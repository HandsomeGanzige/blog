---
title: "有 Architect、Dev 和 Review，为什么 Agent 还是会写出糟糕的代码？"
date: 2026-09-08
tags:
  - coding-agent
  - multi-agent
  - software-architecture
  - code-review
  - harness-engineering
source: conversation
---

# 有 Architect、Dev 和 Review，为什么 Agent 还是会写出糟糕的代码？

一个 Coding Agent 团队已经配置了 Architect、Developer 和 Reviewer。架构师负责设计，开发负责实现，评审负责把关——看起来，它应该比单个 Agent 更不容易写出混乱的代码。

实际结果却可能没有明显改善：Agent 收到需求后仍然立即开始实现，组件越来越臃肿，新旧抽象并行存在，测试全部通过，代码却越来越难改。问题通常不是“角色还不够多”，而是这些角色只有名称，没有形成可阻断错误方向的工程机制。

短答案是：**多 Agent 只是任务拓扑，工程质量来自角色之间可验证的交付合同。**

## 为什么 Architect 在场，开发还是会直接开始？

因为“提供架构建议”和“决定当前是否可以实现”是两件事。

一个常见的 Architect 合同会要求它比较方案、分析权衡、列出风险。这些输出可能很专业，但如果没有说明当前决策是否已经就绪，Main Agent 仍然可以把建议当作参考，然后继续派发开发任务。

更有用的架构输出需要包含明确状态：

```text
ready
blocked-on-user
blocked-on-evidence
exploratory-only
```

如果状态是 `blocked-on-user`，说明不同选择会改变产品行为、公共接口或长期架构，必须由用户决定；如果是 `exploratory-only`，开发可以制作原型，却不能把原型默认为生产架构。

这一区别在组件库改造中尤其重要。假设项目已经有一套生产界面，Agent 又在隔离预览中实现了新的 `ui` 和 `modules`：

```text
生产路径：App → WorkspaceNavigation / ChatPane / Modal
预览路径：Preview → modules → ui
```

仅仅说明“预览暂不接入生产”还不够。团队还需要回答：

- 新体系最终是整体采用、部分提取，还是直接丢弃？
- 哪些旧能力必须保留？
- 两套实现可以共存多久？
- 每迁移一个组件，何时删除旧实现？
- 新生产依赖是否已经获得认可？

如果这些问题没有答案，继续完善组件库不是在落实架构，而是在扩大一个尚未决策的分支。

> Architect 的核心交付物不只是方案，而是“Development 现在被授权实现什么”。

## 为什么测试全部通过，代码仍然可能很差？

因为行为测试和结构质量回答的是不同问题。

测试可以很好地证明：

- 异步提交不会重复触发；
- 中文输入法的 Enter 不会误发送；
- 会话切换不会丢失草稿；
- 失败后可以安全重试。

但它们不会自动阻止一个 React 组件同时负责 bootstrap、会话状态、IPC 订阅、快捷键、响应式布局、弹窗和整个页面组合，也不会阻止父组件通过下面的方式操纵子组件：

```ts
document.querySelector('.chat-pane.active textarea')?.focus();
```

这段代码可能通过所有测试，却把父组件绑定到了子组件的 class 和 DOM 结构。将来只要调整 composer markup，焦点行为就可能悄悄失效。

同样，隔离组件测试能够证明新 `Composer` 自己工作正常，却不能证明它可以替换生产 Composer。生产代码也许区分：

```text
空闲 Enter       → prompt
运行中 Enter     → steer
运行中 Alt+Enter → follow-up
```

如果新接口只提供一个 `onQueue`，测试通过只能说明简化后的接口实现正确，不能说明原有能力得到保留。

因此，可靠交付至少需要两类证据：

| 证据 | 回答的问题 |
| --- | --- |
| 行为测试 | 功能在给定条件下是否正确 |
| 结构审查与边界检查 | 修改是否仍然容易理解、替换和继续演化 |

格式化、lint、依赖方向测试和架构边界检查可以机械发现一部分问题；职责是否合理、抽象是否保留领域语义，仍需要独立评审。两者不能互相替代。

## Reviewer 为什么没有把这些问题挡下来？

“寻找 actionable defects”会自然地把 Reviewer 引向 bug、安全问题和回归。职责膨胀、重复抽象和变化耦合通常还没有立即造成运行错误，很容易被降级为风格建议。

要让 Review 真正承担可维护性责任，任务合同必须明确检查：

- 责任边界是否清晰；
- 是否出现同一能力的重复实现；
- 依赖方向是否倒置；
- 新抽象能否保留现有能力；
- 组件之间是否通过内部 DOM、全局状态或隐式约定耦合；
- 测试是否只证明局部实现，而没有证明生产迁移；
- 复杂度是否来自真实需求，还是来自预设未来扩展。

Reviewer 还需要独立上下文和拒绝权。它应直接读取用户目标、已确认设计、当前 diff、项目规则和验证证据，而不是只接收 Developer 对自己实现的解释。

评审最终返回的也不应只是“看起来不错”，而应该是：

```text
approve
changes-required
escalate
```

其中 `escalate` 表示问题需要新的产品或架构决策，不能由 Reviewer 或 Developer 擅自选择。

## 更严格的流程会不会变成工程官僚主义？

会，所以不能让所有任务都经过同样的完整流程。

修改一个错别字和替换会话状态模型，不应该支付相同的编排成本。更实用的做法是先进行风险分类：

| 风险 | 典型变化 | 最小流程 |
| --- | --- | --- |
| trivial | 文案、明确的局部样式 | 直接实现并检查 diff |
| local | 单模块行为，不改变接口 | 简短探索、实现、针对性验证 |
| structural | 新抽象、公共 API、跨模块依赖 | 架构判断、计划、独立评审 |
| critical | 安全、并发、会话、数据迁移 | 架构门禁、强验证、多维独立评审 |

Agent 也不需要遇到任何不确定性都询问用户。能从代码、测试或项目文档确认的事实，应当自行调查；只有当不同答案会改变产品行为、兼容性、安全边界或长期架构时，才需要停止并提问。

Anthropic 的 Agent 工程实践强调，任务清晰后 Agent 才适合独立运行，并通过环境反馈获得 ground truth。OpenAI 的 Harness Engineering 实践则把计划当作一等工件，并使用自定义 linter 和结构测试约束关键依赖方向。Microsoft 的 maker-checker 模式和 Google ADK 的评估指南也都强调：独立检查需要明确标准，最终结果和关键执行轨迹都需要证据。

这些资料并没有证明所有项目都需要固定的阶段状态机。它们共同支持的是一个更小的原则：

> 对低风险实现保持快速；对不可逆或会塑造长期结构的决定，要求可检查的计划和独立证据。

## 如何判断一个多 Agent 团队是否真的有用？

不要统计调用了多少角色，而要检查它们是否改变了结果。

可以从团队自己的失败案例中整理一组评估任务，对比：

1. 普通单 Agent；
2. 带风险门禁和验证的单 Agent；
3. Architect、Developer、Reviewer 分离的多 Agent。

评估指标不应只有测试通过率，还应包括：

- 第一次编辑前是否发现关键约束；
- 应当提问时是否提问；
- 是否披露影响设计的假设；
- 是否制造重复架构；
- 是否违反依赖边界；
- Reviewer 是否发现并推动修复结构问题；
- 最终修改是否扩大了不必要的范围；
- 质量提升是否值得额外延迟和成本。

如果带门禁的单 Agent 已经表现得同样好，增加角色只会增加协调失败。如果独立 Reviewer 能稳定发现实现者遗漏的问题，它才真正提供了价值。

下次设计 Agent 工程团队时，可以先问四个问题：

1. 每个角色是否有独立且可验证的交付物？
2. 谁能宣布设计已经可以进入实现？
3. 哪些重要规则已经成为测试、lint 或结构检查？
4. 如果某个角色没有返回有效结果，Main 是否仍会错误地宣布完成？

只给 Agent 换上 Architect、Developer 和 Reviewer 的名字，不会自动产生架构、Clean Code 或反对意见。**当角色能够阻止错误方向、留下可核对证据，并在真实评估中改善结果时，它们才组成了工程团队。**

## 延伸阅读

- [Anthropic：Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- [Anthropic：Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Anthropic：Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [OpenAI：Harness engineering](https://openai.com/index/harness-engineering/)
- [Google ADK：Why evaluate agents](https://google.github.io/adk-docs/evaluate/)
- [Microsoft Azure Architecture Center：AI Agent Orchestration Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [SWE-agent](https://proceedings.neurips.cc/paper_files/paper/2024/file/5a7c947568c1b1328ccc5230172e1e7c-Paper-Conference.pdf)
- [Agentless](https://arxiv.org/html/2407.01489v2)
