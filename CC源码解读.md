# CC 源码解读

## 工作流程

```mermaid
flowchart TD
    A[用户输入] --> B[QueryEngine]
    B --> C{Agent Loop}
    B --> D[Context Layer]

    C -->|调用工具| E[Tool System]
    C -->|end_turn| F[返回结果]
    C -->|maxTurns / maxBudget| G[强制终止]

    E -->|权限检查| H[Permission Layer]
    H -->|通过| I[[执行工具]]
    I -->|结果追加| C

    D --> J[CLAUDE.md]
    D --> K[Git 状态]
    D --> L[当前日期]
```

## Agent Loop 流程

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

### 源码定位

Agent Loop 的核心实现在 `QueryEngine.submitMessage()` 中，关键代码位于 [QueryEngine.ts:675-686](src/QueryEngine.ts#L675-L686)：

```typescript
for await (const message of query({
  messages,
  systemPrompt,
  userContext,
  systemContext,
  canUseTool: wrappedCanUseTool,
  toolUseContext: processUserInputContext,
  fallbackModel,
  querySource: 'sdk',
  maxTurns,
  taskBudget,
})) {
  // 根据 message.type 分发处理
}
```

`query()` 是一个 **async generator**，每次 yield 一条消息。循环体内通过 `switch (message.type)` 处理不同类型：

| message.type | 处理逻辑 |
|---|---|
| `assistant` | 推入 mutableMessages，yield 给调用方 |
| `user` | 推入 mutableMessages，turnCount++ |
| `stream_event` | 累计 token 用量（message_start/message_delta/message_stop） |
| `attachment` | 处理 structured_output、**max_turns_reached** 等信号 |
| `system` | 处理 compact_boundary（上下文压缩）、api_error（重试） |

循环结束后还有两道守卫检查：
- **预算超限**：`getTotalCost() >= maxBudgetUsd` → yield `error_max_budget_usd`
- **结构化输出重试上限**：连续失败次数 ≥ `MAX_STRUCTURED_OUTPUT_RETRIES` → yield `error_max_structured_output_retries`