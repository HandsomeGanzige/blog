---
title: "别给 Coding Agent 再造检索系统：让代码本身成为上下文入口"
date: 2026-09-09
tags:
  - coding-agent
  - context-engineering
  - code-review
  - software-architecture
source: conversation
---

# 别给 Coding Agent 再造检索系统：让代码本身成为上下文入口

Coding Agent 接到一个需求后，首先要解决的通常不是“代码怎么写”，而是“真正控制这个行为的代码在哪里”。

当它搜索到几十个相似文件时，一个直觉方案是再建一套模块目录、业务词映射或上下文解析器，提前告诉 Agent 应该读取什么。问题在于，这些材料复制了代码中的事实：模块移动了、调用关系变了、旧实现被替换了，索引却可能仍然指向昨天的答案。

短答案是：**不要为 Agent 维护第二套代码地图，而要通过统一工程规范，让代码本身提供稳定、高信号的检索入口。**

## 为什么少读一点，往往比拥有更长上下文更重要？

因为上下文不是免费的存储空间，而是有限的注意力预算。

[Anthropic 的上下文工程指南](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)指出，随着上下文增加，模型对其中信息的准确召回和长距离推理能力可能下降。有效的上下文工程追求的不是装入尽可能多的材料，而是找到能够支持目标的最小高信号信息集合。

这正是 Coding Agent 检索代码时面临的问题。如果它为了修改一个业务判断，先读取多个页面、数份文档、全部配置和整组搜索结果，那么真正相关的类型、消费者和边界条件反而更容易被淹没。

更有效的方式是即时检索：

```text
需求中的对象与动作
→ 高信号路径或符号
→ 定义和引用
→ 行为所有者
→ 相关类型、测试和配置
```

Anthropic 将这种方式称为 just-in-time context：Agent 保留文件路径、符号和链接等轻量引用，在需要时逐步加载内容，而不是预先塞入完整资料。目录和命名本身也会提供用途信号，帮助 Agent 决定下一步读取什么。

因此，代码可检索性的收益不只是“搜索更方便”。它还可能带来：

- 更少的无关文件进入上下文；
- 更快排除同名但无关的实现；
- 更早找到真正的行为所有者；
- 更少遗漏调用方、类型和测试；
- Dev 与 Review 更快收敛到同一工程事实；
- 新 session 不必依赖上一轮 Agent 的长篇总结。

这些收益需要通过真实任务评估，不能预先承诺固定百分比，但背后的方向很明确：**减少上下文噪声，就是把有限注意力留给真正影响实现的事实。**

## 什么样的代码能让检索自然收敛？

假设一个商城提出需求：

> 订单取消后，不允许继续创建发货任务。

Agent 可能先搜索 `order`、`cancel`、`shipment`。如果代码中同时存在：

```ts
handleData(order);
processStatus(order);
doAction(order);
```

搜索很难判断哪个函数负责取消规则。即使找到了调用处，也必须继续读取实现才能知道它们做什么。

如果代码使用领域对象和动作：

```ts
cancelOrder(orderId);
canCreateShipment(order);
createShipment(orderId);
```

Agent 可以先找到 `createShipment` 的定义和引用，再追踪它是否调用 `canCreateShipment`。[GitHub 的代码导航文档](https://docs.github.com/en/repositories/working-with-files/using-files/navigating-code-on-github)展示的也是同一机制：以具名实体为入口，跳转到定义并查找全部引用。

但好的名称还不够。要让检索真正收敛，代码至少需要三个属性：

1. **可发现：** 文件和公共符号使用稳定的领域对象与动作。
2. **可追踪：** import、类型和调用关系能够从入口连接到行为所有者。
3. **可验证：** 测试、schema、配置和消费者能够证明 Agent 找到的是当前事实。

例如，“取消订单后不能发货”应当有一个权威规则：

```ts
export function canCreateShipment(order: Order): boolean {
  return order.status !== "cancelled";
}
```

页面、后台任务和 API 都使用同一规则。测试名称直接表达行为：

```ts
it("rejects shipment creation when the order is cancelled", () => {
  // ...
});
```

如果三个消费者分别判断字符串 `"cancelled"`，Agent 就必须逐个搜索和比较；下一次修改也可能只更新其中一处。统一所有者不仅减少重复代码，还缩小了每次任务需要加载的上下文。

## 如何让 Dev 和 Review 对“好代码”得到相同结论？

不能只告诉它们“命名要清晰”“代码要易于维护”。不同 Agent、模型和 session 对这些句子的理解并不一致。

更可靠的方式，是建立一份项目级工程规范。每条规则都应包含明确范围、正反例、验证方式和例外，而不是只表达愿望。

例如：

```text
[DOMAIN-001][MUST]

跨文件业务函数使用“领域对象 + 动作”命名。
禁止新增 handleData、processInfo、doAction 等无法表达业务结果的公共名称。

适用范围：
跨文件导出的函数、hook 和组件回调。

Dev：
新增公共函数前搜索已有领域术语和同类操作。

Review：
只有在名称无法区分实际业务行为，或与项目现有主称呼冲突时形成 finding。

自动检查：
可以禁止明确的通用名称；业务语义仍由 Review 判断。
```

Dev 和 Review 使用同一份规范，但职责不同：

| 角色 | 使用方式 |
|---|---|
| Dev | 按规则组织代码、类型、测试和必要说明 |
| Review | 独立检查 diff，并引用规则与具体失败方式 |
| 自动工具 | 检查能够机械判断的命名、依赖和类型约束 |

[Google 的 Code Review 指南](https://google.github.io/eng-practices/review/reviewer/standard.html)强调，技术事实和数据应当优先于个人偏好；纯风格问题应以团队 style guide 为权威。这个原则同样适用于 Agent：如果项目没有具体规则，Reviewer 就不应把自己的偏好包装成阻塞意见。

统一规范的价值不在于它一定是世界上最好的规范，而在于它为不同执行者提供了相同坐标。即使规则以后需要修订，Dev 和 Review 也能够引用同一条款讨论，而不是争论“我觉得这样更清晰”。

## 怎样落地而不制造新的枷锁？

第一步不是为整个仓库生成代码地图，而是选择少量高频、可判断的规则。

可以从以下类型开始：

- 公共符号如何命名；
- 业务规则由谁拥有；
- 哪些层之间禁止反向依赖；
- 外部字符串契约放在哪里；
- 动态 registry 如何定义未知输入；
- 行为变化必须检查哪些消费者和测试。

第二步，让 Dev 与 Review 读取同一份规范。规范描述结果，不规定必须使用 `rg`、语言服务器还是其他搜索工具。Agent 可以根据宿主能力自由调查，只要最终实现和证据满足规则。

第三步，把机械规则交给工具。TypeScript、ESLint、测试和依赖检查适合阻止明确违规；“两个抽象是否具有相同业务语义”仍需要 Review，但 Reviewer 必须给出调用方、变化路径或兼容风险。

第四步，只约束新增和被触及的代码。已有项目往往存在历史命名和依赖问题，如果一次性要求全仓合规，规范会从质量基线变成重构枷锁。更实际的目标是：每次修改都不增加新的歧义，并让被触及的区域逐步变得更容易检索。

## 什么时候才应该增加文档或索引？

只有当关键事实无法通过正常代码关系表达，而且重新推导的成本和风险都很高时，才需要额外说明。

增加前可以问：

1. 删除这段说明后，能否从目录、符号、类型、引用和测试中恢复事实？
2. 它记录的是稳定边界和原因，还是容易过期的实现清单？
3. 代码变化时，维护者是否会自然意识到需要更新它？

能够从代码廉价获得的调用链，不应复制进手工索引。架构边界、跨模块不变量、外部兼容原因和重要技术取舍，则值得进入项目文档。

下次准备提高 Coding Agent 的检索效率时，先检查三件事：

1. 需求中的领域概念能否命中稳定代码符号？
2. 从入口能否沿静态关系到达唯一行为所有者？
3. 类型、消费者和测试能否验证这条理解？

如果答案是否定的，优先修复代码的命名、局部性和所有权。**让现有搜索工具自然找到正确事实，通常比维护一套更聪明的检索系统更省上下文，也更不容易漂移。**

## 延伸阅读

- [Anthropic：Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [GitHub Docs：Navigating code on GitHub](https://docs.github.com/en/repositories/working-with-files/using-files/navigating-code-on-github)
- [Google Engineering Practices：The Standard of Code Review](https://google.github.io/eng-practices/review/reviewer/standard.html)
