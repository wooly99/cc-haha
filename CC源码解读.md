# CC 源码解读

## 引子：CC 到底在干什么？

你在终端敲一句话，CC 就能帮你改代码、跑测试、读文档……它是怎么做到的？

答案只有一个循环：**调模型 → 拿到工具调用 → 执行工具 → 把结果喂回模型 → 再来一轮**。

这就是 `Agent Loop` —— 整个 CC 的心脏。

![反复分析](https://xmmx-bucket.oss-cn-beijing.aliyuncs.com/typora-imgs/2026-04/反复分析.jpg)

## 1. The Agent Loop

```mermaid
flowchart TD
    A[用户输入] --> B[拼好系统提示 + 历史消息 + 工具列表]
    B --> C[发给 Claude API]
    C --> D{收到流式响应}

    D -->|工具调用| E([执行工具])
    D -->|end_turn| F[返回结果给用户]
    D -->|超过最大轮次| G([强制终止])
    D -->|超过预算上限| G

    E --> H[结果追加到消息列表]
    H --> C
```

每轮结束，对话历史多一条工具结果消息。模型看到工具结果再决定下一步：继续调用工具，还是直接回复用户。

**这就是"Agent"和"普通聊天"的本质区别：模型不只是回答问题，它还能自己决定"我需要先做点什么"。**

### 源码定位

Agent Loop 由两层构成：

- **内层** `query()`（[query.ts:307](src/query.ts#L307)）— 真正的 while(true) 循环：调模型 → 收集 tool_use → 执行工具 → 拼回消息 → 下一轮
- **外层** `QueryEngine.submitMessage()`（[QueryEngine.ts:675](src/QueryEngine.ts#L675)）— 消费 `query()` yield 出的消息，按类型分发处理

#### 内层：query.ts — 真正的 Agent Loop

```typescript
// query.ts 第 307 行
// ⚠️ 以下为简化伪代码，突出核心骨架，省略了错误恢复 / compact / streaming 等逻辑
while (true) {
  // ① 调用模型 API（流式）
  for await (const message of callModel({ messages, systemPrompt, tools })) {
    yield message                              // 向外层推送流式消息
    if (message.type === 'assistant') {
      assistantMessages.push(message)
      // 收集本轮所有 tool_use 块
      const blocks = message.content.filter(c => c.type === 'tool_use')
      toolUseBlocks.push(...blocks)
      if (blocks.length > 0) needsFollowUp = true
    }
  }

  // ② 没有工具调用 → 模型说完了，正常退出
  if (!needsFollowUp) {
    return { reason: 'completed' }
  }

  // ③ 有工具调用 → 批量执行工具
  for await (const update of runTools(toolUseBlocks, canUseTool, context)) {
    yield update.message                       // 向外层推送工具结果
    toolResults.push(update.message)
  }

  // ④ 检查是否超过最大轮次
  if (maxTurns && nextTurnCount > maxTurns) {
    yield { type: 'max_turns_reached', turnCount: nextTurnCount }
    return { reason: 'max_turns' }
  }

  // ⑤ 把 assistant + toolResults 拼回消息列表，进入下一轮
  state = {
    messages: [...messages, ...assistantMessages, ...toolResults],
    turnCount: nextTurnCount,
  }
} // while (true)
```

#### 外层：QueryEngine.ts — 消费 + 分发

内层 query() 是一个 async generator，每 yield 一条消息，外层就接住并处理。可以理解为：**query() 是引擎，QueryEngine 是仪表盘**。

```typescript
// QueryEngine.ts 第 675 行（简化伪代码）
for await (const message of query({ messages, systemPrompt, tools, maxTurns })) {

  // 记录消息到会话历史
  if (message.type === 'assistant' || message.type === 'user') {
    messages.push(message)
    await recordTranscript(messages)           // 持久化会话
  }

  switch (message.type) {
    case 'assistant':
      this.mutableMessages.push(message)
      yield* normalizeMessage(message)         // 转为 SDK 事件推给调用方（终端、VSCode、桌面端）
      break

    case 'stream_event':
      // 实时累计 token 用量（用于预算控制）
      if (message.event.type === 'message_stop') {
        this.totalUsage = accumulateUsage(this.totalUsage, currentMessageUsage)
      }
      break

    case 'tool_use_summary':
      // 工具执行摘要（给 UI 展示一行简要说明）
      yield { type: 'tool_use_summary', summary: message.summary }
      break
  }

  // 超预算 → 终止
  if (maxBudgetUsd !== undefined && getTotalCost() >= maxBudgetUsd) {
    yield { type: 'result', subtype: 'error_max_budget_usd' }
    return
  }
}
```

> **为什么分两层？**
> - `query()` 是纯逻辑的 async generator，只管「调模型 → 跑工具 → 循环」
> - `QueryEngine` 负责工程侧关切：会话持久化、token 计费、SDK 事件格式化
> - 好处：想换模型供应商？只改 query 内部。想换 UI 框架？只改 QueryEngine 的 yield 格式。互不干扰。
>
> **为什么是两个终止条件？**
> - `maxTurns`（最大轮次）—— 防止死循环，在内层 query() 中检查
> - `maxBudgetUsd`（最大预算）—— 防止超支，在外层 QueryEngine 中检查
> - 一个管"跑了多少轮"，一个管"花了多少钱"，互补兜底

### 小结

Agent Loop 的本质就一句话：**while(true) { 问模型 → 跑工具 → 把结果喂回去 }，直到模型说"我说完了"或者触发安全阀。**

记住这个循环，后面的 Tool Use、TodoWrite、Subagent 都是往这个循环里"塞东西"。

## 2. Tool Use
