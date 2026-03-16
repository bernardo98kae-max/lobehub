# Agent Runtime 新人入门（按“切图”方式一步步理解）

本文面向第一次接触 LobeHub `packages/agent-runtime` 的同学。
目标是把它“切成几张图”来理解：

1. **职责切图**：谁负责决策，谁负责执行。
2. **数据切图**：State / Context / Instruction / Event 如何流动。
3. **流程切图**：一次完整对话如何从 user_input 走到 finish。
4. **扩展切图**：如何插入工具、人审、任务编排、压缩。

---

## 1. 先看入口：这个包暴露了哪些能力

`agent-runtime` 的导出非常直接：

- `agents`：内置 Agent（例如 `GeneralChatAgent`）
- `core`：执行引擎（`AgentRuntime`）
- `types`：协议类型
- `utils` / `audit` / `groupOrchestration`

可先从 `src/index.ts` 建立目录级心智模型。

---

## 2. 职责切图：Agent 是“大脑”，Runtime 是“引擎”

```
User/Input --> Agent.runner(...) --> Instruction(s)
                               \-> AgentRuntime.step(...) --> Executor --> Event + newState + nextContext
```

### 2.1 Agent（大脑）

- 通过 `runner(context, state)` 决定“下一步做什么”。
- 输出的是 `AgentInstruction`（如 `call_llm`、`call_tool`、`finish` 等）。
- 自己不直接执行业务动作，主要负责**决策**。

### 2.2 Runtime（引擎）

- `step()` 执行一个回合：
  1. 更新 step 计数和状态时间
  2. 调用 Agent 产出 instruction
  3. 根据 instruction 分发到 executor
  4. 汇总 events / newState / nextContext
- 支持中断（interrupt）、恢复（resume）、成本与用量统计。

---

## 3. 数据切图：四个核心对象

### 3.1 `AgentState`（可持久化“护照”）

重点字段建议先记住：

- `messages`：会话消息
- `status`：`idle/running/waiting_for_human/done/error/interrupted`
- `stepCount`、`maxSteps`、`forceFinish`
- `pendingToolsCalling`、`pendingHumanPrompt/Select`
- `usage`、`cost`、`costLimit`
- `toolManifestMap`、`securityBlacklist`、`userInterventionConfig`

> 理解要点：`state` 是“事实”，不是“决策”。

### 3.2 `AgentRuntimeContext`（当前回合上下文）

- 核心是 `phase`：`init`、`user_input`、`llm_result`、`tool_result`、`tools_batch_result`、`human_abort` 等。
- `payload` 存放该 phase 的输入输出细节。

> 理解要点：`context.phase` 是当前回合的“路口牌”。

### 3.3 `AgentInstruction`（决策产物）

常见指令：

- 执行类：`call_llm`、`call_tool`、`call_tools_batch`
- 人审类：`request_human_approve/prompt/select`
- 收尾类：`finish`
- 扩展类：`compress_context`、`exec_task(s)`、`exec_client_task(s)`、`resolve_aborted_tools`

### 3.4 `AgentEvent`（执行事件流）

- 常见事件：`llm_start`、`llm_stream`、`tool_result`、`human_approve_required`、`interrupted`、`resumed` 等。
- UI/上层业务一般监听 event 去驱动展示。

---

## 4. 流程切图：一次“普通对话 + 工具调用”的主路径

下面以 `GeneralChatAgent` + `AgentRuntime` 为主线：

### Step 0：初始化

- 用 `AgentRuntime.createInitialState(...)` 创建初始 state。
- 进入 `context.phase = init` 或 `user_input`。

### Step 1：收到用户输入（`user_input`）

`GeneralChatAgent.runner()` 在 `user_input` 会：

1. 可选检查是否需要压缩上下文（`shouldCompress`）。
2. 需要压缩则返回 `compress_context`。
3. 否则返回 `call_llm`（携带 `messages`、工具定义等）。

### Step 2：Runtime 执行 `call_llm`

`AgentRuntime` 的 LLM executor 会：

1. 触发 `llm_start`
2. 迭代模型流式输出，持续发 `llm_stream`
3. 汇总 content / tool_calls
4. 产出下一阶段 `nextContext.phase = llm_result`

### Step 3：进入 `llm_result` 决策

`GeneralChatAgent.runner()` 对 `llm_result` 分三种：

1. **无工具调用** -> `finish`
2. **全是可自动执行工具** -> `call_tool` 或 `call_tools_batch`
3. **部分/全部需人工审批** -> 返回组合指令（先执行安全工具，再 `request_human_approve`）

### Step 4：工具执行后回流

- `tool_result` / `tools_batch_result` 阶段会再次决策：
  - 若仍有 pending 工具 -> `request_human_approve`
  - 否则 -> 再次 `call_llm`，让模型整合工具结果

### Step 5：收敛

最终走到 `finish`，Runtime 把状态标记为 `done`（或 error 路径）。

---

## 5. 人审（Human-in-the-loop）切图

`GeneralChatAgent` 的干预判断是“多层策略叠加”：

1. **全局审计规则（global audits）**：可强制拦截高风险工具。
2. **tool manifest 中的人审配置**：API 级优先于 tool 级。
3. **用户模式**：`manual` / `auto-run` / `allow-list` / `headless`。
4. **动态策略**：通过 `dynamicInterventionAudits` 运行时判定。

结果是把工具调用拆成：

- `toolsToExecute`（可直接执行）
- `toolsNeedingIntervention`（需审批）

---

## 6. 异常与控制切图

### 6.1 maxSteps 与 forceFinish

- step 超过 `maxSteps` 时，不一定立刻报错；会进入 `forceFinish` 流程。
- 其意图是让工具收尾后，下一次 LLM 输出最终文本并结束。

### 6.2 interrupt / resume

- `interrupt()` 会把状态置为 `interrupted` 并写入中断信息。
- `resume()` 可恢复执行，并可带入 context 继续跑下一步。

### 6.3 成本与用量

- `calculateUsage` / `calculateCost` 由 Agent 可选实现。
- Runtime 在 LLM/Tool 执行后更新 usage/cost，并检查 `costLimit`。

---

## 7. 你可以怎么“切入阅读”

推荐顺序（30~60 分钟一轮）：

1. `src/types/instruction.ts`：先把 phase 和 instruction 看懂。
2. `src/types/state.ts`：再看 state 字段，知道运行时会改什么。
3. `src/core/runtime.ts`：抓住 `step()` 主循环 + executor 分发。
4. `src/agents/GeneralChatAgent.ts`：逐 phase 看 runner 决策。
5. `examples/tools-calling.ts`：把抽象映射到可运行示例。

---

## 8. 给新人第一周的实操建议

1. **先不改功能，只打日志**：在 `runner` 各 phase 打印 context/state 关键字段。
2. **用单一问题跑通闭环**：例如“查时间 + 算式”这种带工具调用的问题。
3. **刻意制造分支**：
   - 一次无工具调用
   - 一次需审批工具调用
   - 一次中断后恢复
4. **每次只验证一个机制**：先验证 phase 流，再看 usage/cost，最后看人审策略。

这样你会更快把 `agent-runtime` 从“黑盒”变成“可预测状态机”。

---

## 9. 从前端到后端：一次请求的“逐步拆解”（核心代码版）

这一节把真实代码链路串起来：**用户点击发送** 后，消息如何经过前端 Store、`agent-runtime`、服务层，再到后端 `/webapi/chat/[provider]`。

### 9.1 前端入口：发送消息后启动 AgentRuntime

在 `conversationLifecycle.sendMessage` 完成“用户消息 + 占位 assistant 消息”创建后，会调用：

```ts
internal_execAgentRuntime({
  context,
  messages,
  parentMessageId,
  parentMessageType: 'assistant',
  ...
})
```

它是前端 AI 执行主入口（不是直接调用后端）。

### 9.2 运行时编排入口：`internal_execAgentRuntime`

`streamingExecutor.ts` 里的 `internal_execAgentRuntime` 做了几件关键事：

1. 创建 `execAgentRuntime` operation（用于取消、层级关联、UI 状态）。
2. 调用 `internal_createAgentState` 生成初始 `state/context`（把模型、工具、会话上下文整理好）。
3. 创建 `GeneralChatAgent` + `AgentRuntime`。
4. 注入 `createAgentExecutors(...)`，把指令执行桥接到当前前端能力（创建消息、流式更新、工具调用等）。
5. 进入 `while` 循环不断 `runtime.step(state, nextContext)`，直到 `done/error/waiting_for_human`。

> 这一步是“**前端状态机中枢**”：它既不是纯 UI，也不是纯网络层，而是把对话执行抽象成可中断、可恢复、可观察的 runtime 循环。

### 9.3 每一回合：`runtime.step` 如何推进

`AgentRuntime.step(...)`（`packages/agent-runtime/src/core/runtime.ts`）每轮会：

1. 增加 `stepCount`、更新 `lastModified`。
2. 根据 `context.phase` 调用 `agent.runner(...)` 产出 instruction（或 instruction 数组）。
3. 按 instruction type 调 executor：`call_llm` / `call_tool` / `call_tools_batch` / `request_human_approve` / `finish`...
4. 汇总 `events + newState + nextContext` 返回给外层循环。

所以你可以把它理解为：

- `GeneralChatAgent.runner` 决定“下一步是什么”
- `AgentRuntime` 负责“把这一步执行完并产出下一阶段输入”

### 9.4 `call_llm` 指令：如何真正连到模型

这块在前端并不直接写死 fetch，而是由 `createAgentExecutors` 注入 custom executor：

1. `call_llm` executor 先创建（或复用）assistant 消息。
2. 创建 `StreamingHandler`，把流式 chunk 映射成消息更新：content / reasoning / tools / grounding / images。
3. 调用 `chatService.createAssistantMessageStream(...)` 发起流式请求。

注意这里是关键桥梁：

- **AgentRuntime 只认识 instruction/event/state**
- **具体怎么请求模型、怎么更新消息，由 executors 决定**

这正是 runtime 可复用的原因。

### 9.5 Chat Service：前端服务层如何组装请求

`chatService.createAssistantMessageStream` -> `createAssistantMessage` -> `getChatCompletion`，核心工作：

1. 用 `resolvedAgentConfig` + `contextEngineering(...)` 组装最终 `modelMessages`。
2. 合并模型扩展参数、工具定义、stream 配置。
3. 通过 `fetchSSE(API_ENDPOINTS.chat(provider), ...)` 发到 `/webapi/chat/${provider}`。

`API_ENDPOINTS.chat(provider)` 在 `src/services/_url.ts` 明确映射到后端 chat 路由。

### 9.6 后端入口：`/webapi/chat/[provider]`

后端 route（`src/app/(backend)/webapi/chat/[provider]/route.ts`）逻辑很干净：

1. `checkAuth` 做鉴权包装。
2. `initModelRuntimeFromDB(serverDB, userId, provider)` 初始化模型 runtime（按用户配置）。
3. 读取 `ChatStreamPayload`。
4. 调 `modelRuntime.chat(data, { user, signal, traceOptions })` 返回流或非流响应。
5. 异常统一转换成标准 error response。

这意味着后端 route 主要是“**鉴权 + runtime 初始化 + 透传调用 + 错误规范化**”，而不是业务编排中心。

### 9.7 响应回流：如何回到前端状态机

SSE chunk 回来后：

1. `StreamingHandler` 持续更新 assistant 消息内容（含 reasoning/tools 等增量信息）。
2. `call_llm` executor 在流结束后生成 `llm_result` 对应的 `nextContext`。
3. 外层 `internal_execAgentRuntime` 收到 step 结果，继续下一轮：
   - 有工具调用 -> `tool_result/tools_batch_result` 路线
   - 无工具调用 -> `finish`

最终 operation 进入：

- `done`：完成
- `waiting_for_human`：暂停等审批
- `error`：失败收敛

### 9.8 一句话总览（真正的前后端分工）

- **前端 Store + AgentRuntime**：负责对话执行编排（状态机、指令循环、UI 同步、工具人审）。
- **前端 ChatService**：负责把“这一轮 LLM 调用”翻译成网络请求。
- **后端 chat route**：负责鉴权、读取用户模型配置、调用模型 runtime、返回标准响应。

这样拆分后，AgentRuntime 能同时支持：

1. 不同模型提供商
2. 不同工具执行方式（客户端/服务端）
3. 不同 UI 容器（Web/Desktop）

而核心执行语义（phase -> instruction -> event）保持不变。
