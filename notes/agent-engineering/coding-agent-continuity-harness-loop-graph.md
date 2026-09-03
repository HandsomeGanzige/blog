---
title: "Coding Agent 长任务为何会漂移：Harness、Loop 与 Graph 的状态边界"
date: 2026-09-03
tags:
  - coding-agent
  - agent-harness
  - context-engineering
  - durable-execution
source: conversation
---

# Coding Agent 长任务为何会漂移：Harness、Loop 与 Graph 的状态边界

Coding Agent 的上下文即将耗尽时，最直觉的做法是生成摘要，再开启一个新上下文。另一种更工程化的反应，是引入任务状态机、Checkpoint 或执行图，把每一步都保存下来。

两种做法都可能有效，也都可能制造虚假的安全感。真正的问题不是“历史有没有保存”，而是：**任务恢复时，哪些信息必须延续，哪些事实应该重新读取，哪些动作绝不能被重复执行？**

短答案是，把模型上下文视为可替换的认知缓存，把意图、实现、工作材料和执行状态分层管理。Harness 的价值不是替 Agent 记住一切，而是在正确的边界提供恢复入口、验证反馈和用户控制。

## 上下文还在，为什么方向仍然会偏？

因为“内容仍在窗口里”和“模型能够正确使用它”不是一回事。

[Lost in the Middle](https://arxiv.org/abs/2307.03172) 展示了长上下文中的位置效应：相关信息处在输入中部时，模型使用它的能力可能明显下降。[LLMs Get Lost in Multi-Turn Conversation](https://arxiv.org/abs/2505.06120) 则发现，在受控的多轮任务中，模型容易过早形成假设，并在后续信息到来后继续依赖此前的错误方向。

因此，目标漂移并不只发生在上下文被截断之后。过长历史、重复工具输出、失败轨迹和局部任务都可能逐渐挤压全局目标的注意力。

自然语言压缩能够缓解容量问题，却不能证明恢复是无损的。压缩器必须决定哪些内容值得保留，而“当时看似次要”的细节，可能正是后来定位问题所需的信息。

可以把 Coding Agent 接触的信息分成四层：

| 信息层 | 典型内容 | 合适载体 |
| --- | --- | --- |
| 意图事实 | 当前需求、验收标准、权限和用户决定 | 当前请求、规格和已确认的决策材料 |
| 实现事实 | 代码、测试、配置、diff 和运行结果 | 当前仓库与可重复验证 |
| 认知工作集 | 搜索结果、临时推理、任务简报和探索线索 | 模型上下文或 Agent 工作目录 |
| 执行状态 | 已提交步骤、外部副作用、重试和审批 | Checkpoint、Journal、Receipt 或宿主运行时 |

这四层不能相互替代。对话摘要适合恢复思考方向，但不适合证明代码仍然如此，也不能证明某个外部动作是否已经执行。

> 上下文可以告诉 Agent 去哪里找；权威事实必须能够重新读取或重新验证。

## 代码应该成为唯一事实吗？

不应该，但代码通常是最有价值的实现事实。

对于 Coding Agent，重新搜索当前代码、调用关系、测试、配置和 diff，往往比阅读一份详细的执行日志更可靠。执行日志描述的是 Agent 曾经看见什么，仓库描述的是系统现在是什么。

这也是为什么 Harness 不宜强制绑定某个搜索命令。它应该要求 Agent 找到相关符号、调用方、测试和项目约束，但把具体发现方式交给宿主环境。文本搜索、符号索引、语言服务器或用户自定义工具都可能是更合适的入口。

然而，代码无法完整回答三个问题：

1. 用户最终接受了什么结果？
2. 某个不明显的设计限制为什么存在？
3. 哪些方案经过讨论但仍未获得确认？

局部设计原因适合保存在靠近实现的注释、类型或测试中；跨模块的不变量、依赖方向和兼容性原则，则更适合进入架构文档、ADR、开发指南或项目级 Agent 指令。

这类架构材料不是 Agent 的临时记忆。它是后续开发者和 Agent 都需要遵循的项目事实，因此应使用与代码一致的术语、指向相关模块，并从项目入口文档中被发现。

与之相对，当前任务的简报、恢复指针和临时分析只服务于 Agent，可以放进统一的 Agent 工作目录。这个目录是否由 Git 忽略，是项目与用户的版本管理决定，不应成为 Agent 判断能否使用工作材料的前置条件。

所以更准确的规则不是“代码即唯一事实”，而是：

> 代码、测试和配置说明系统现在如何工作；项目文档保存无法从实现稳定推导的长期意图；Agent 工作材料只负责导航，不升级为项目真相。

## 一次上下文切换应该怎样恢复？

假设一个 Agent 正在重构权限模块。上下文结束前只留下这样一句话：

> 权限重构基本完成，继续补测试。

这份摘要缺少当前验收标准、已经修改的范围、未确认的兼容性决定和验证证据。新 Agent 很容易重复修改、漏掉风险，或者把“基本完成”误解成已经满足需求。

更有效的简报不需要复述全部历史，只需要保存：

- 当前结果和验收标准；
- 用户已确认、但无法从代码推导的约束；
- 尚未解决的问题；
- 相关文件、符号、错误和测试入口；
- 已知风险与下一步。

恢复后的 Agent 先读取这份导航，再检查当前工作区、diff、测试和项目文档。如果简报与代码冲突，以当前实现和最新用户要求为准；如果无法验证，就把结论降级为未知。

[Anthropic 的长运行 Agent Harness 复盘](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)也采用了类似思路：后续会话处理较小范围的工作，通过项目文件恢复进度，并验证当前环境，而不是依靠一段无限增长的历史。[Manus 的上下文工程总结](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)则强调“可恢复压缩”：删除正文时保留 URL 或文件路径，使信息以后仍能重新获取。

简报的价值因此不在于保存更多文字，而在于降低重新找到事实的成本。

## 什么时候普通 Loop 已经够用，什么时候才需要 Graph？

如果任务主要修改本地代码，每一步都便宜、可撤销并且能够通过仓库重新验证，一个简单反馈循环通常已经足够：

```text
检查当前事实
→ 做一个有边界的修改
→ 运行验证
→ 检查 diff 和结果
→ 决定继续、回退思路或结束
```

这类任务引入复杂 Graph，可能只是把原本能够重新计算的进度复制到另一套状态中。代码变化后，Graph 里的节点状态反而可能先过期。

当任务出现以下情况时，Checkpoint、Graph 或 Durable Runtime 才开始提供不可替代的价值：

- 调用可能产生费用或不可安全重复的外部服务；
- 任务需要暂停等待人工审批；
- 进程重启后必须从已提交步骤继续；
- 多个执行者并发修改共享状态；
- 重试必须知道某个副作用是否已经发生；
- 工作持续数天，无法仅靠重新检查仓库恢复。

[Microsoft Magentic-One](https://www.microsoft.com/en-us/research/publication/magentic-one-a-generalist-multi-agent-system-for-solving-complex-tasks/)将全局任务事实与短期进度判断放在不同 Ledger 中，并在停滞时重新规划。[LangGraph](https://docs.langchain.com/oss/python/langgraph/persistence)用 Checkpoint 保存图状态；[Temporal 的 AI 架构](https://go.temporal.io/platform-hub/ai-engineering/ai-reference-architecture)则进一步区分可重放的控制流程与不可随意重放的 LLM、工具和外部副作用。

这些机制解决的是执行恢复，不会自动保证任务理解正确。一个内容已经过期的 Checkpoint，只会更稳定地恢复到错误状态。

因此，Loop 与 Graph 不是“简单版”和“高级版”的关系，而是对应不同失败模型：

- 能从当前世界重新计算的内容，优先重新读取和验证；
- 不能安全重放的动作，才需要持久化执行语义；
- 无论采用哪种拓扑，完成都应由外部证据而不是 Agent 自评证明。

## 用户如何在不阅读 Agent 内存的情况下保持控制？

用户不应该依赖阅读 Agent 的内部简报，才能理解项目发生了什么。

真正面向用户的控制界面应当是：

- 当前目标和验收标准；
- 已确认的架构决定与项目约束；
- 可审阅的代码和文档 diff；
- 实际执行过的测试与可观察结果；
- 尚未验证的部分和残余风险；
- 涉及外部副作用时的明确授权。

如果一项架构建议要成为后续开发必须遵循的规则，就应进入项目文档；如果它仍是待选方案，就保持为提案或 Agent 工作材料。架构内容的作者可以是专门的架构 Agent，但“提出建议”和“让规范正式生效”仍然是两个不同动作。

多 Agent 系统也遵循同样边界。子 Agent 可以拥有独立上下文，处理原子化任务；全局协调者只需要接收结论、证据位置、修改范围、未验证假设和风险。复制完整对话既浪费上下文，也会把局部探索噪声重新带回全局。

设计 Coding Agent Harness 时，可以依次问四个问题：

1. 这条信息能否从当前代码或环境低成本重建？
2. 它是长期项目事实，还是当前 Agent 的工作线索？
3. 这一步是否产生不能安全重复的副作用？
4. 用户最终通过什么证据理解并控制结果？

如果能够重建，就保存入口而不是复制正文；如果需要长期遵循，就放进项目事实；如果不能安全重放，就交给持久执行机制；如果用户无法从最终 diff、测试和文档理解结果，Harness 就还没有形成完整的控制闭环。

这比“保留所有历史”更轻，也比“所有任务都建一张 Graph”更可靠。

## 延伸材料

- [OpenAI：长运行模型与 Compaction 指南](https://developers.openai.com/api/docs/guides/latest-model?model=gpt-5.5)
- [Anthropic：Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Anthropic：Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Microsoft：Magentic-One](https://www.microsoft.com/en-us/research/publication/magentic-one-a-generalist-multi-agent-system-for-solving-complex-tasks/)
- [LangGraph：Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
- [Temporal：AI reference architecture](https://go.temporal.io/platform-hub/ai-engineering/ai-reference-architecture)
- [Lost in the Middle](https://arxiv.org/abs/2307.03172)
- [LLMs Get Lost in Multi-Turn Conversation](https://arxiv.org/abs/2505.06120)

这些资料分别来自论文、官方文档和厂商工程实践，证据强度并不相同。它们支持“应该区分上下文、记忆和执行状态”这一方向，但尚不能证明某一种 Harness 结构适合所有 Coding Agent。
