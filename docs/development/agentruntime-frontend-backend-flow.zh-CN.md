# AgentRuntime 前端到后端完整流程（含简单数据示例）

本文是对 `agent-runtime` 调用链路的“加细版”说明，重点回答两个问题：

1. 一条消息从前端点击发送，到后端模型返回，**每一步发生了什么**？
2. `llm_result` 后分叉为：
   - **无工具调用**（直接结束）
   - **有工具调用**（调用工具后再回到 LLM）

---

## 0. 先记住三层分工

- **前端编排层（Store + AgentRuntime）**
  - 负责“状态机循环”：`phase -> instruction -> executor -> event -> nextContext`
  - 入口是 `internal_execAgentRuntime`

- **前端服务层（chatService）**
  - 负责把本轮 `call_llm` 请求转为 `/webapi/chat/:provider` 的 SSE 请求

- **后端路由层（webapi/chat/[provider]）**
  - 鉴权、读取用户模型配置、初始化 runtime、调用模型并返回流式响应

---

## 1. 时序总览（发送后发生什么）

```text
用户点击发送
  -> conversationLifecycle.sendMessage
  -> internal_execAgentRuntime(...)
      -> internal_createAgentState(...) 组初始 state/context
      -> new GeneralChatAgent(...)
      -> new AgentRuntime(agent, { executors: createAgentExecutors(...) })
      -> while 循环 runtime.step(state, nextContext)
          -> agent.runner(context, state) 产出 instruction
          -> executor 执行 instruction
          -> 返回 events + newState + nextContext
  -> 直到 state.status = done / waiting_for_human / error
```

---

## 2. 前端入口：`conversationLifecycle.sendMessage`

发送消息后，`sendMessage` 在消息创建完成后，会调用：

```ts
await internal_execAgentRuntime({
  context: execContext,
  messages: displayMessages,
  parentMessageId: data.assistantMessageId,
  parentMessageType: 'assistant',
  ...
});
```

这一步的关键是：

- **不是直接 fetch 模型**，而是进入统一 runtime 执行器。
- 后续所有分支（工具、人审、继续生成、中断恢复）都在这个执行器里统一处理。

---

## 3. 运行时中枢：`internal_execAgentRuntime`

`streamingExecutor.ts` 中该函数完成以下动作：

1. 建立 `execAgentRuntime` operation（用于取消、trace、层级操作）。
2. 调 `internal_createAgentState`：
   - 组装初始 `AgentState`
   - 生成初始 `AgentRuntimeContext`（一般 phase 是 `init`）
3. 构造 `GeneralChatAgent` 与 `AgentRuntime`。
4. 注入 `createAgentExecutors`，把 instruction 落到真实能力上：
   - 创建/更新消息
   - 流式渲染
   - 调工具
   - 人工审批
5. 进入循环，不断执行 `runtime.step(state, nextContext)`。

循环中每步会先计算 `stepContext`（如 todo、激活工具、page editor 状态），再调用 runtime。

---

## 4. Runtime 单步：`AgentRuntime.step`

每个 step 的固定骨架：

1. 复制并更新 state（`stepCount+1`、更新时间）。
2. 调 `agent.runner(context, state)` 生成 instruction（可能是数组）。
3. 分发到 executor：
   - `call_llm`
   - `call_tool`
   - `call_tools_batch`
   - `request_human_approve`
   - `finish`
4. 汇总并返回：
   - `events`
   - `newState`
   - `nextContext`

你可以把它看成：

- Agent 决策“做什么”
- Runtime 保证“怎么稳定执行并进入下一阶段”

---

## 5. `call_llm` 如何走到后端

在 `createAgentExecutors` 的 `call_llm` executor 内：

1. 先创建（或复用）assistant 消息。
2. 启动 `StreamingHandler`，把 SSE chunk 映射成消息增量更新：
   - `content`
   - `reasoning`
   - `tools`
   - `grounding`
   - `images`
3. 调 `chatService.createAssistantMessageStream(...)`。

`chatService` 内部再走：

- `createAssistantMessage`（上下文工程 + 模型参数合并）
- `getChatCompletion`
- `fetchSSE(API_ENDPOINTS.chat(provider), ...)`

`API_ENDPOINTS.chat(provider)` 对应 `/webapi/chat/${provider}`。

---

## 6. 后端 `POST /webapi/chat/[provider]`

后端 route 主要做“薄编排”：

1. `checkAuth` 鉴权。
2. `initModelRuntimeFromDB(serverDB, userId, provider)` 初始化该用户的模型 runtime。
3. 读取 `ChatStreamPayload`。
4. 调 `modelRuntime.chat(data, { user, signal, traceOptions })`。
5. 把错误规范化后返回。

后端不负责 Agent 状态机编排，主要负责：

- 账户与模型配置隔离
- provider runtime 调用
- 标准化错误响应

---

## 7. 示例 A：无工具调用（最短路径）

> 目标：用户问“1+1 等于几？”，模型直接回答，不触发 tool_calls。

### 7.1 初始数据（简化）

```ts
state.messages = [
  { id: 'u1', role: 'user', content: '1+1 等于几？' }
];

context = {
  phase: 'user_input',
  payload: { parentMessageId: 'a1' }
};
```

### 7.2 决策与执行

1. `runner(user_input)` -> 返回 `call_llm`
2. `call_llm executor` 发起 SSE，流式更新 assistant
3. 流结束后，产出：

```ts
nextContext = {
  phase: 'llm_result',
  payload: {
    hasToolsCalling: false,
    result: { content: '1+1 = 2' },
    toolCalls: []
  }
};
```

4. 下一个 step：`runner(llm_result)` 发现没有工具调用 -> 返回 `finish`
5. Runtime 收敛，`state.status = 'done'`

### 7.3 结果

- 用户看到 assistant 最终文本“1+1 = 2”
- 整体路径：`user_input -> call_llm -> llm_result -> finish`

---

## 8. 示例 B：有工具调用（工具后再回 LLM）

> 目标：用户问“上海现在几点？顺便算 15*8+7”，模型先发 tool_calls。

### 8.1 初始数据（简化）

```ts
state.messages = [
  { id: 'u1', role: 'user', content: '上海现在几点？顺便算 15*8+7' }
];

context = {
  phase: 'user_input',
  payload: { parentMessageId: 'a1' }
};
```

### 8.2 LLM 第一次返回（带工具调用）

> 常见追问：**模型怎么知道有哪些工具？又怎么决定用哪个工具？**

可以按“工具供给 -> 请求携带 -> 模型决策”三步理解：

1. **工具供给（前端编排阶段）**
   - `internal_createAgentState` / `resolvedAgentConfig` 会把当前会话可用工具整理出来（受 agent 配置、禁用项、子任务过滤、人审策略等影响）。
   - `GeneralChatAgent.runner` 在 `user_input` 阶段返回 `call_llm` 时，会把 `state.tools`（工具 schema 列表）放进 payload。

2. **请求携带（call_llm executor）**
   - `createAgentExecutors.call_llm` 调 `chatService.createAssistantMessageStream(...)`。
   - `chatService.createAssistantMessage -> getChatCompletion` 最终把 `tools` 字段一起发给 `/webapi/chat/:provider`。
   - 对模型来说，它看到的是“用户消息 + 历史消息 + 工具定义（name/description/parameters）”。

3. **模型决策（LLM 侧）**
   - 模型基于语义匹配和参数 schema 选择是否生成 `tool_calls`。
   - 例如用户问“几点”，更容易触发 `get_time`；问“15*8+7”，更容易触发 `calculate`。
   - 如果模型判断不需要工具，就直接返回文本，进入“无工具调用”路径。

`call_llm` 结束后：

```ts
nextContext = {
  phase: 'llm_result',
  payload: {
    hasToolsCalling: true,
    toolsCalling: [
      { id: 'tc1', identifier: 'time', apiName: 'get_time', arguments: '{}' },
      { id: 'tc2', identifier: 'calc', apiName: 'calculate', arguments: '{"expression":"15*8+7"}' }
    ],
    parentMessageId: 'a1'
  }
};
```

### 8.3 `llm_result` 分叉

`GeneralChatAgent.runner(llm_result)` 会先做干预判断（是否需要人工审批），假设都可自动执行：

- 多个工具 -> 返回 `call_tools_batch`

```ts
{
  type: 'call_tools_batch',
  payload: {
    parentMessageId: 'a1',
    toolsCalling: [tc1, tc2]
  }
}
```

### 8.4 执行工具并回流

`call_tools_batch` executor 执行后，写入 tool 结果消息，例如：

```ts
{ role: 'tool', tool_call_id: 'tc1', content: '{"current_time":"2026-03-10T10:00:00Z"}' }
{ role: 'tool', tool_call_id: 'tc2', content: '{"result":127}' }
```

并给出：

```ts
nextContext = { phase: 'tools_batch_result', payload: { parentMessageId: 'a1' } }
```

随后 `runner(tools_batch_result)` -> 再次返回 `call_llm`，带上最新消息（含 tool 结果）。

### 8.5 第二次 LLM 收敛

第二次 `call_llm` 通常返回纯文本整合结果：

```text
上海当前时间是 ...
15*8+7 = 127
```

对应 `llm_result.hasToolsCalling = false`，下一步 `finish`，状态收敛到 `done`。

### 8.6 结果

典型路径：

`user_input -> call_llm -> llm_result(has tools) -> call_tools_batch -> tools_batch_result -> call_llm -> llm_result(no tools) -> finish`

---

## 9. 人审插入点（补充）

若工具命中人审规则（全局审计/manifest 配置/用户模式），在 `llm_result` 阶段会返回：

```ts
{ type: 'request_human_approve', pendingToolsCalling: [...] }
```

此时状态变为 `waiting_for_human`，当前 operation 会结束；用户审批后再开启后续执行。

---

## 10. 排查建议（按层定位）

- **UI 一直 loading**：先看 operation 是否未收敛（`execAgentRuntime` 状态）。
- **有流但不落消息**：看 `StreamingHandler` 回调是否更新到目标 messageId。
- **模型返回了工具却没执行**：看 `llm_result` 后的 instruction 是否变成 `request_human_approve`（可能被策略拦截）。
- **后端报模型错误**：看 `/webapi/chat/[provider]` route 返回的标准错误体与 provider 配置。

---

## 11. 最短记忆卡片

- 前端主循环：`internal_execAgentRuntime`。
- 决策函数：`GeneralChatAgent.runner(context, state)`。
- 执行引擎：`AgentRuntime.step(...)`。
- LLM 网络桥：`createAgentExecutors.call_llm -> chatService -> fetchSSE(/webapi/chat/:provider)`。
- 后端薄路由：鉴权 + 初始化模型 runtime + 调用 `modelRuntime.chat`。
