# AgentRuntime 前端到后端完整数据流程（含返回类型示例）

本文聚焦“**数据怎么流**”，不是只讲函数调用顺序。
我们按一条请求从前端到后端再回前端的过程，拆成可观测的数据层：

1. 前端输入数据（UI Message / Context）
2. Runtime 决策数据（Instruction / State / Event）
3. 服务层请求数据（ChatStreamPayload）
4. 后端路由与模型响应数据（SSE chunks）
5. 回流后的最终消息数据（文本 / 工具 / 沙盒文件）

---

## 1. 总体数据流图（Data Flow）

```text
[UI输入]
  用户消息(UIChatMessage)
      |
      v
[Store层]
  conversationLifecycle.sendMessage
      |
      v
[Runtime编排]
  internal_execAgentRuntime
    - AgentState (可持久化运行态)
    - AgentRuntimeContext (当前phase输入)
      |
      v
  AgentRuntime.step
    - Agent.runner -> AgentInstruction
    - executor执行 -> AgentEvent + newState + nextContext
      |
      v
[服务层]
  chatService.createAssistantMessageStream
    -> ChatStreamPayload
    -> fetchSSE('/webapi/chat/:provider')
      |
      v
[后端]
  POST /webapi/chat/[provider]
    - initModelRuntimeFromDB
    - modelRuntime.chat(...)
      |
      v
[SSE回流]
  StreamingHandler
    - updateMessage(content/reasoning/tools/images/search)
      |
      v
[最终落库/展示]
  assistant message / tool message / file attachment metadata
```

---

## 2. 前端输入层：用户消息怎么变成 Runtime 输入

### 2.1 UI 输入（示例）

```ts
const uiMessage = {
  id: 'u_001',
  role: 'user',
  content: '请帮我总结这个项目，并把总结写到 sandbox 文件里',
};
```

### 2.2 进入 `internal_execAgentRuntime`（示例）

```ts
internal_execAgentRuntime({
  context: {
    agentId: 'agent_default',
    topicId: 'topic_001',
    threadId: 'thread_001',
  },
  messages: [uiMessage],
  parentMessageId: 'a_placeholder_001',
  parentMessageType: 'assistant',
});
```

这里的关键数据是：

- `messages`: 当前会话窗口消息
- `context`: 会话维度上下文（agent/topic/thread）
- `parentMessageId`: 当前 assistant 回复链的父节点

---

## 3. Runtime 层：四类核心数据对象如何接力

## 3.1 `AgentState`（运行态）示例

```ts
const state = {
  operationId: 'op_001',
  status: 'running',
  stepCount: 1,
  messages: [{ role: 'user', content: '...' }],
  usage: {
    llm: { apiCalls: 1, tokens: { input: 120, output: 0, total: 120 }, processingTimeMs: 0 },
    tools: { totalCalls: 0, totalTimeMs: 0, byTool: [] },
    humanInteraction: { approvalRequests: 0, promptRequests: 0, selectRequests: 0, totalWaitingTimeMs: 0 },
  },
  cost: { currency: 'USD', total: 0, llm: { total: 0, byModel: [], currency: 'USD' }, tools: { total: 0, byTool: [], currency: 'USD' }, calculatedAt: '2026-03-11T00:00:00Z' },
};
```

## 3.2 `AgentRuntimeContext`（phase 输入）示例

```ts
const context = {
  phase: 'user_input',
  payload: { parentMessageId: 'a_placeholder_001' },
  session: { sessionId: 'agent_default', messageCount: 1, status: 'running', stepCount: 1 },
};
```

## 3.3 `AgentInstruction`（决策结果）示例

```ts
const instruction = {
  type: 'call_llm',
  payload: {
    messages: state.messages,
    model: 'gpt-4.1-mini',
    provider: 'openai',
    tools: state.tools,
    parentMessageId: 'a_placeholder_001',
  },
};
```

## 3.4 `AgentEvent`（执行事件）示例

```ts
[
  { type: 'llm_start', payload: { model: 'gpt-4.1-mini' } },
  { type: 'llm_stream', chunk: { content: '正在' } },
  { type: 'llm_stream', chunk: { content: '总结中...' } },
]
```

---

## 4. 服务层：请求体怎么组织成模型可消费数据

`call_llm` executor 会调用 `chatService.createAssistantMessageStream`，底层会组装 `ChatStreamPayload`。

### 4.1 `ChatStreamPayload` 示例

```ts
const chatPayload = {
  model: 'gpt-4.1-mini',
  provider: 'openai',
  stream: true,
  messages: [
    { role: 'system', content: 'You are a helpful assistant.' },
    { role: 'user', content: '请总结项目并写入sandbox文件' },
  ],
  tools: [
    {
      type: 'function',
      function: {
        name: 'write_sandbox_file',
        description: 'Write content to sandbox file and return file metadata',
        parameters: {
          type: 'object',
          properties: {
            filename: { type: 'string' },
            content: { type: 'string' },
          },
          required: ['filename', 'content'],
        },
      },
    },
  ],
};
```

---

## 5. 后端层：路由接收与模型返回

后端 `POST /webapi/chat/[provider]` 接收到上述 payload 后，调用模型 runtime 并返回 SSE。

### 5.1 SSE chunk 示例（文本）

```text
data: {"id":"chatcmpl_x","choices":[{"delta":{"content":"这是项目总结"}}]}

data: {"id":"chatcmpl_x","choices":[{"delta":{"content":"，共分三层..."}}]}
```

### 5.2 SSE chunk 示例（工具调用）

```text
data: {"id":"chatcmpl_x","choices":[{"delta":{"tool_calls":[{"index":0,"id":"call_1","function":{"name":"write_sandbox_file"}}]}}]}

data: {"id":"chatcmpl_x","choices":[{"delta":{"tool_calls":[{"index":0,"function":{"arguments":"{\"filename\":\"summary.md\",\"content\":\"...\"}"}}]}}]}
```

---

## 6. 回流层：三种典型返回结果（文本 / 工具 / 沙盒文件）

下面是你要求的“每个类型都举例”。

## 6.1 类型 A：纯文本返回（无工具）

### A-1 模型最终结果（Runtime 视角）

```ts
nextContext = {
  phase: 'llm_result',
  payload: {
    hasToolsCalling: false,
    result: { content: '这是项目总结：1) 前端 2) Runtime 3) 后端' },
    toolCalls: [],
  },
};
```

### A-2 前端最终消息（展示层）

```ts
assistantMessage = {
  id: 'a_001',
  role: 'assistant',
  content: '这是项目总结：1) 前端 2) Runtime 3) 后端',
  tools: undefined,
  files: undefined,
};
```

---

## 6.2 类型 B：工具返回（函数调用结果）

### B-1 模型先发 tool_calls

```ts
llmResult = {
  hasToolsCalling: true,
  toolsCalling: [
    {
      id: 'call_1',
      identifier: 'sandbox',
      apiName: 'write_sandbox_file',
      arguments: '{"filename":"summary.md","content":"项目总结..."}',
    },
  ],
};
```

### B-2 工具执行后写入 tool message

```ts
toolMessage = {
  id: 't_001',
  role: 'tool',
  tool_call_id: 'call_1',
  content: '{"success":true,"path":"/sandbox/summary.md","size":512}',
};
```

### B-3 再次 `call_llm` 汇总

```ts
assistantMessage = {
  id: 'a_002',
  role: 'assistant',
  content: '我已将总结写入 summary.md，文件大小约 512B。',
};
```

---

## 6.3 类型 C：沙盒文件返回（带文件元数据）

> 这里的关键是：**文件内容通常不直接全量塞进 assistant 文本**，而是通过工具结果 + 文件元数据引用。

### C-1 工具结果中的文件元数据示例

```ts
sandboxFileResult = {
  success: true,
  file: {
    id: 'file_001',
    name: 'summary.md',
    mimeType: 'text/markdown',
    size: 512,
    url: 'sandbox://files/file_001',
    previewText: '# 项目总结\n- 前端...\n- Runtime...\n- 后端...'
  }
};
```

### C-2 前端消息展示可采用“文本 + 文件卡片”

```ts
assistantMessage = {
  id: 'a_003',
  role: 'assistant',
  content: '总结已生成，请查看附件文件。',
  files: [
    {
      id: 'file_001',
      name: 'summary.md',
      mimeType: 'text/markdown',
      size: 512,
      url: 'sandbox://files/file_001',
    },
  ],
};
```

### C-3 UI 交互效果（示意）

- 气泡正文：`总结已生成，请查看附件文件。`
- 文件卡片：`summary.md (512B)`
- 点击卡片：读取 `sandbox://files/file_001` 并打开预览

---

## 7. 两条完整时序（带数据）

## 7.1 无工具调用时序

```text
user_input
 -> call_llm(payload.messages + tools)
 -> llm_result(hasToolsCalling=false, content='...')
 -> finish
 -> state.status = done
```

## 7.2 有工具 + 沙盒文件时序

```text
user_input
 -> call_llm(payload.messages + tools)
 -> llm_result(hasToolsCalling=true, toolsCalling=[write_sandbox_file])
 -> call_tool / call_tools_batch
 -> tool_result(content='{"success":true,"file":...}')
 -> call_llm(带上tool消息)
 -> llm_result(hasToolsCalling=false, content='已写入文件')
 -> finish
 -> state.status = done
```

---

## 8. 实战排查：按数据断点看问题

1. **看 instruction**：`llm_result` 后到底是 `finish` 还是 `call_tool`？
2. **看 tool message**：是否有 `tool_call_id` 对应上模型的 `call_xxx`？
3. **看附件元数据**：文件 URL / name / size 是否完整？
4. **看 nextContext.phase**：有没有正确从 `tools_batch_result` 回到 `call_llm`？
5. **看最终 state.status**：是否收敛为 `done`，还是停在 `waiting_for_human`。

---

## 9. 一页总结

- Runtime 是“数据状态机”，不是单次接口调用。
- 核心数据接力：`State -> Context -> Instruction -> Event -> nextContext`。
- 三类最终产物都可以统一在消息层表达：
  - 文本：assistant content
  - 工具：tool role message
  - 沙盒文件：tool result + assistant files metadata
