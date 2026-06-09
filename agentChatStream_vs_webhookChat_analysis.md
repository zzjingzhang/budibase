# agentChatStream vs webhookChat 差异分析

## 概述

本文档从**权限**、**sessionId**、**持久化**三个维度，详细分析 `agentChatStream` 与 `webhookChat` 的实现差异，并深入解析关键辅助函数的设计逻辑。

---

## 一、resolveChatStreamRequest 深度解析

`resolveChatStreamRequest` 是 `agentChatStream` 的核心前置处理函数，负责请求解析、权限校验和冲突检测。

### 1.1 path 参数应用（applyChatStreamPathParams）

函数位置：[chatConversations.ts#L152-L177](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L152-L177)

该函数将 URL path 中的参数（`chatAppId`、`chatConversationId`）应用到请求体 `chat` 对象上，并执行冲突检测。

**chatAppId 冲突防护：**
- 如果 path 中存在 `chatAppId`，且 body 中也存在 `chat.chatAppId`，但两者不一致 → 抛出 400 错误：`"chatAppId in body does not match path"`
- 如果 path 中存在 `chatAppId`，将其赋值给 `chat.chatAppId`

**chatConversationId 冲突防护：**
- 如果 path 中 `chatConversationId` 存在且不为 `"new"`，同时 body 中 `chat._id` 也存在但不一致 → 抛出 400 错误：`"chatConversationId in body does not match path"`
- 如果 path 中 `chatConversationId` 存在且不为 `"new"`，将其赋值给 `chat._id`

这种设计确保了 URL 路径参数与请求体参数的一致性，防止参数注入或绕过路径限制。

### 1.2 Preview 权限控制

函数位置：[chatConversations.ts#L207-L266](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L207-L266)

**Preview 模式仅限 admin/builder 使用：**
```typescript
const isBuilderOrAdmin = usersSdk.users.isAdminOrBuilder(ctx.user)
const requestedPreview = chat.isPreview === true

if (requestedPreview && !isBuilderOrAdmin) {
  throw new HTTPError("Forbidden", 403)
}
```
- `chat.isPreview === true` 表示请求预览模式
- 非 builder/admin 用户请求 preview → 直接返回 403 Forbidden
- 这是在所有其他验证之前执行的，确保非授权用户无法进入后续逻辑

### 1.3 非 Preview 模式的严格验证

当 `canUsePreview` 为 false（即非预览模式）时，必须满足以下条件：

**1. chatAppId 必须存在：**
```typescript
if (!canUsePreview && !chatAppId) {
  throw new HTTPError("chatAppId is required", 400)
}
```

**2. chatApp 必须存在且为 live 状态：**
```typescript
chatApp = await db.tryGet<ChatApp>(chatAppId)
if (!chatApp) {
  throw new HTTPError("Chat app not found", 404)
}
assertChatAppIsLiveForUser(ctx, chatApp)
```

`assertChatAppIsLiveForUser` 逻辑（[chatApps.ts#L12-L17](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatApps.ts#L12-L17)）：
- builder/admin 用户：即使 chatApp 不是 live 也可以访问
- 普通用户：`chatApp.live !== true` → 抛出 403 "Chat app is not live"

**3. agent 必须已启用且用户可访问：**
```typescript
const chatAgentConfig = getEnabledChatAgentConfig(chatApp, agentId)
await assertCanAccessChatAgent(ctx, chatAgentConfig)
```

- `getEnabledChatAgentConfig`（[chatConversations.ts#L68-L77](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L68-L77)）：在 chatApp 的 agents 列表中查找 agentId，且必须 `isEnabled === true`，否则返回 400
- `assertCanAccessChatAgent`（[chatConversations.ts#L90-L97](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L90-L97)）：调用 `canAccessChatAppAgentForUser` 检查角色权限

`canAccessChatAppAgentForUser` 逻辑（[chatApps.ts#L24-L38](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatApps.ts#L24-L38)）：
- admin/builder 用户：始终返回 `true`
- agent 未配置 `roleId`：返回 `true`（公开访问）
- 其他情况：通过 `AccessController.hasAccess()` 检查用户角色是否满足 agent 的角色要求

---

## 二、getExistingChatForStream 安全机制

函数位置：[chatConversations.ts#L179-L205](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L179-L205)

该函数用于在流式对话中获取已存在的聊天记录，并实施两项关键安全防护。

### 2.1 防止改变 agentId

```typescript
if (chat.agentId && chat.agentId !== existingChat.agentId) {
  throw new HTTPError("agentId cannot be changed", 400)
}
```

- 如果请求体中指定了 `agentId`，且与现有对话的 `agentId` 不一致 → 拒绝请求
- 确保对话一旦创建，其所属 agent 不可被篡改
- 现有对话的 `agentId` 优先级更高（见 `resolveChatStreamRequest` 中 `const agentId = existingChat?.agentId || chat.agentId`）

### 2.2 跨用户访问防护

```typescript
if (existingChat.userId && existingChat.userId !== userId) {
  throw new HTTPError("Forbidden", 403)
}
```

- 如果对话有 `userId`（即属于某个用户），且与当前请求用户的 `userId` 不一致 → 返回 403 Forbidden
- 防止用户通过猜测 conversationId 访问他人的聊天记录
- `userId` 来自 `getGlobalUserId(ctx)`，从 `ctx.user.globalId / userId / _id` 中获取

### 2.3 其他防护措施

**chatApp 归属校验（非 preview 模式）：**
```typescript
if (!canUsePreview && existingChat.chatAppId !== chatAppId) {
  throw new HTTPError("chat does not belong to this chat app", 400)
}
```
- 非预览模式下，对话必须属于指定的 chatApp
- 预览模式跳过此检查（因为 preview 可以不指定 chatAppId）

---

## 三、agentChatStream 核心实现

函数位置：[chatConversations.ts#L389-L531](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L389-L531)

### 3.1 SSE 响应设置

`agentChatStream` 使用 Server-Sent Events (SSE) 进行流式响应：

```typescript
ctx.status = 200
ctx.set("Content-Type", "text/event-stream")
ctx.set("Cache-Control", "no-cache")
ctx.set("Connection", "keep-alive")
ctx.res.setHeader("X-Accel-Buffering", "no")
ctx.res.setHeader("Transfer-Encoding", "chunked")
```

关键 Header 说明：
- `Content-Type: text/event-stream`：SSE 标准 MIME 类型
- `Cache-Control: no-cache`：禁止缓存，确保实时性
- `Connection: keep-alive`：保持长连接
- `X-Accel-Buffering: no`：禁用 Nginx 代理缓冲，确保流式输出
- `Transfer-Encoding: chunked`：分块传输编码

后续通过 `ctx.respond = false` 禁用 Koa 的自动响应，直接操作 `ctx.res` 写入流。

### 3.2 chatId 与 sessionId 生成

```typescript
const chatId = chat._id ?? docIds.generateChatConversationID()
const sessionId = chat.transient ? chat.sessionId || chatId : chatId
```

**chatId 生成规则：**
- 如果 `chat._id` 存在（即已有对话），复用该 ID
- 否则通过 `docIds.generateChatConversationID()` 生成新的对话 ID

**sessionId 生成规则：**
- 非 transient 模式：`sessionId === chatId`
- transient 模式：优先使用 `chat.sessionId`，否则回退到 `chatId`
- transient 模式通常用于预览或临时对话，允许复用同一个 sessionId 保持上下文

### 3.3 pendingToolCalls 处理

```typescript
const pendingToolCalls = new Set<string>()
const result = await run.stream({
  pendingToolCalls,
})
```

- `pendingToolCalls` 是一个 `Set<string>`，用于跟踪尚未完成的工具调用
- 传入 `run.stream()` 后，AI SDK 会在工具调用开始时添加、完成时移除
- 在流结束时用于检测工具调用是否完整

在 `messageMetadata` 的 finish 阶段检测不完整的工具调用：
```typescript
const finishReason = (part as { finishReason?: string }).finishReason
const toolCallsIncomplete =
  pendingToolCalls.size > 0 || finishReason === "tool-calls"

if (toolCallsIncomplete) {
  return {
    ...,
    error: formatIncompleteToolCallError([]),
  }
}
```

两种不完整情况：
1. `pendingToolCalls.size > 0`：仍有工具调用未返回结果
2. `finishReason === "tool-calls"`：模型以工具调用结束，但没有后续处理

### 3.4 pipeUIMessageStreamToResponse 详解

`result.pipeUIMessageStreamToResponse(ctx.res, { ... })` 是流式输出的核心，包含多个关键回调。

**messageMetadata 回调：**
为流式消息的每个部分添加元数据。

- **start 阶段**（[chatConversations.ts#L437-L442](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L437-L442)）：
  - 返回 `toolDisplayNames`（如果有工具）
  - 返回 `createdAt`（流开始时间戳）

- **finish 阶段**（[chatConversations.ts#L443-L472](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L443-L472)）：
  - 返回 `ragSources`：使用的知识库来源
  - 返回 `createdAt` 和 `completedAt`：时间戳
  - 返回 `usage`：Token 使用统计（通过 `buildAgentMessageUsage` 构建）
  - 返回 `error`：如果工具调用不完整

**onFinish 回调（持久化逻辑）：**
（[chatConversations.ts#L496-L519](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L496-L519)）

```typescript
onFinish: async ({ messages }) => {
  await run.sessionLogIndexer.index()
  events.action.aiAgentExecuted({ agentId })

  if (chat.transient || !chatAppId) {
    return
  }

  const existingChat = chat._id
    ? await db.tryGet<ChatConversation>(chat._id)
    : null

  const chatToSave = prepareChatConversationForSave({
    chatId,
    chatAppId,
    userId,
    title,
    messages,
    chat,
    existingChat,
  })

  await db.put(chatToSave)
}
```

持久化条件：
- 非 `transient` 模式
- 存在 `chatAppId`（即非 preview 模式）

持久化流程：
1. 索引会话日志（`sessionLogIndexer.index()`）
2. 触发 `aiAgentExecuted` 事件
3. 查找现有对话（用于获取 `_rev` 等元数据）
4. 调用 `prepareChatConversationForSave` 构建保存对象
5. 写入数据库（`db.put`）

**prepareChatConversationForSave**（[helpers.ts#L21-L53](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/chatConversations/helpers.ts#L21-L53)）：

构建要保存的 `ChatConversation` 对象：
- `_id`: 使用 `chatId`
- `_rev`: 优先使用 `existingChat._rev`，否则使用 `chat._rev`
- `chatAppId`、`agentId`、`userId`：直接使用传入值
- `agentId`: 优先使用 `existingChat.agentId`（防止篡改）
- `title`: 优先使用传入的 `title`，否则使用 `chat.title`
- `messages`: 通过 `truncateToolPartsForSave` 处理后保存
- `createdAt`: 优先使用已有创建时间，否则使用当前时间
- `updatedAt`: 始终设置为当前时间
- `channel`: 如果有渠道信息则携带

---

## 四、webhookChat 核心实现

函数位置：[chatConversations.ts#L273-L387](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L273-L387)

### 4.1 输入要求与权限校验

**必须参数：**
```typescript
const chatAppId = chat.chatAppId
if (!chatAppId) {
  throw new HTTPError("chatAppId is required", 400)
}

const agentId = chat.agentId
if (!agentId) {
  throw new HTTPError("agentId is required", 400)
}
```

**校验逻辑：**
1. chatApp 必须存在（`db.tryGet<ChatApp>(chatAppId)`，不存在返回 404）
2. **只校验 agent 配置存在于 chatApp 中**（不要求启用，也不校验用户访问权限）：
```typescript
getConfiguredChatAgentConfig(chatApp, agentId)
```

`getConfiguredChatAgentConfig`（[chatConversations.ts#L79-L88](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L79-L88)）与 `getEnabledChatAgentConfig` 的区别：
- `getConfiguredChatAgentConfig`：只要 agent 在 chatApp 的 agents 列表中**存在**即可，不检查 `isEnabled`
- `getEnabledChatAgentConfig`：要求 `isEnabled === true`

3. 验证 agent 本身存在且有效：`sdk.ai.agents.getOrThrow(agentId)`

> **注意**：`webhookChat` 本身不做用户级别的权限校验（如角色访问控制、chatApp live 校验）。这些校验由调用方 `handleChatMessage` 在调用前完成。

### 4.2 chatId 与 sessionId 生成

```typescript
const providerPrefix = chat.channel?.provider || "chat"
const chatId = chat._id ?? docIds.generateChatConversationID()
const sessionId = `${providerPrefix}:${chatId}`
```

- `chatId`: 复用已有 ID 或生成新 ID
- `sessionId`: 格式为 `{provider}:{chatId}`，例如 `slack:chat_conv_xxx`
- 携带 provider 前缀是为了区分不同渠道的会话，避免不同渠道的会话 ID 冲突

### 4.3 消息流处理与结果消费

```typescript
const result = await run.stream()

const streamTask = onAssistantStream
  ? onAssistantStream(result.fullStream)
  : Promise.resolve()

const [textResult, responseResult, streamOutcome] = await Promise.allSettled([
  result.text,
  result.response,
  streamTask,
])
```

**三个并行任务：**

1. **`result.text`**：完整的助手回复文本
   - 来自 AI SDK 的流式文本聚合
   - `textResult.status === "rejected"` 时记录错误并抛出

2. **`result.response`**：响应元数据
   - 包含 `id`（请求 ID）等信息
   - 成功时将 `requestId` 添加到 `sessionLogIndexer`
   - 失败时记录错误

3. **`streamTask`**（可选）：流式输出任务
   - 如果调用方提供了 `onAssistantStream` 回调，则将 `result.fullStream` 传入
   - 用于实时将流式响应推送到第三方渠道（如 Slack、Discord）
   - 失败时记录错误并抛出（视为严重错误）

所有任务使用 `Promise.allSettled` 等待，确保即使部分失败也能收集所有结果。

**错误处理优先级：**
1. 先检查 `streamOutcome`（流推送失败）→ 直接抛出
2. 再检查 `textResult`（文本生成失败）→ 抛出
3. 最后检查 `responseResult`（响应元数据失败）→ 抛出

### 4.4 返回值设计

```typescript
const assistantText = textResult.value
const assistantMessage: ChatConversation["messages"][number] = {
  id: v4(),
  role: "assistant",
  parts: [{ type: "text", text: assistantText || "" }],
}

return {
  messages: [...chat.messages, assistantMessage],
  assistantText: assistantText || "",
  title,
}
```

**webhookChat 不直接写入数据库**，而是返回：
- `messages`: 包含用户消息和助手回复的完整消息列表
- `assistantText`: 助手的纯文本回复（方便调用方直接使用）
- `title`: 对话标题（基于最新问题截断生成）

持久化完全交由调用方处理（如 `handleChatMessage` 在收到结果后自行调用 `prepareChatConversationForSave` 和 `db.put`）。

---

## 五、核心差异对比表

| 维度 | agentChatStream | webhookChat |
|------|-----------------|-------------|
| **使用场景** | Web 端用户实时聊天（SSE 流式） | 第三方渠道集成（Slack/Discord/Teams/Telegram） |
| **入口方式** | HTTP API 路由（`/api/chatapps/:chatAppId/conversations/:chatConversationId/stream`） | 函数调用（由 webhook handler 内部调用） |
| **响应方式** | SSE 流式响应（`text/event-stream`） | Promise 返回完整结果 + 可选流回调 |
| **chatAppId** | 非 preview 模式必须提供；preview 模式可选 | 必须提供 |
| **chatApp 校验** | 非 preview：必须存在且 live；preview：不校验 | 只校验存在，不校验 live |
| **agent 校验** | 非 preview：必须启用 + 用户有权限；preview：只校验 agent 存在 | 只校验配置存在于 chatApp 中（不校验 isEnabled） |
| **Preview 模式** | 支持，仅限 admin/builder | 不支持 |
| **用户身份** | 从 Koa ctx.user 获取，必须已认证 | 由调用方传入 user 对象，可能是合成的公共用户 |
| **sessionId 格式** | `chatId`（非 transient）或自定义（transient） | `{provider}:{chatId}`（带渠道前缀） |
| **跨用户防护** | 严格校验 userId 匹配 | 不做校验（由调用方保证） |
| **agentId 防篡改** | 严格校验，已有对话的 agentId 不可变 | 不校验（由调用方保证） |
| **pendingToolCalls** | 跟踪并在流结束时检测不完整 | 不跟踪 |
| **持久化时机** | 流结束时在 `onFinish` 回调中自动保存 | 不保存，返回结果由调用方决定是否持久化 |
| **transient 模式** | 支持（不持久化） | 不支持（调用方控制） |
| **消息元数据** | 丰富（usage、ragSources、toolDisplayNames、时间戳） | 仅文本和消息结构 |
| **错误处理** | 通过 SSE error 事件推送 | 抛出异常由调用方处理 |

---

## 六、关键设计决策解析

### 6.1 为什么 agentChatStream 的权限校验更严格？

`agentChatStream` 是面向终端用户的公开 API，必须防御各种恶意请求：
- **参数一致性校验**：防止通过 body 覆盖 path 参数绕过路径路由
- **chatApp live 校验**：确保用户只能访问已发布的聊天应用
- **agent 启用 + 角色权限**：细粒度的访问控制
- **跨用户防护**：保护用户隐私，防止会话劫持

### 6.2 为什么 webhookChat 的权限校验更宽松？

`webhookChat` 是内部函数，调用方（`handleChatMessage`）已经做了前置校验：
- chatApp 存在性和渠道启用状态在 `handleChatMessage` 中检查
- 用户访问权限通过 `canAccessChatAppAgentForUser` 在调用前验证
- 因此 `webhookChat` 只需关注核心的 AI 对话逻辑，重复校验会增加冗余

### 6.3 为什么 agentChatStream 自己持久化，而 webhookChat 不持久化？

- **agentChatStream**：作为 API 端点，职责闭环，流式结束后直接保存
- **webhookChat**：作为可复用的服务函数，遵循单一职责原则，只负责 AI 对话逻辑，持久化交由调用方控制。不同渠道可能有不同的持久化策略（如 Redis 缓存、过期时间等）

### 6.4 为什么 sessionId 格式不同？

- **agentChatStream**：Web 端用户通过 chatId 就能唯一定位会话，不需要额外前缀
- **webhookChat**：多渠道场景下，不同渠道可能有相同的 chatId 语义，加上 `provider:` 前缀确保全局唯一性，也方便日志追踪和会话索引

---

## 七、代码位置索引

| 函数/模块 | 文件路径 |
|----------|---------|
| `agentChatStream` | [chatConversations.ts#L389-L531](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L389-L531) |
| `webhookChat` | [chatConversations.ts#L273-L387](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L273-L387) |
| `resolveChatStreamRequest` | [chatConversations.ts#L207-L266](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L207-L266) |
| `applyChatStreamPathParams` | [chatConversations.ts#L152-L177](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L152-L177) |
| `getExistingChatForStream` | [chatConversations.ts#L179-L205](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L179-L205) |
| `handleChatMessage` | [chatHandler.ts#L456-L735](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/webhook/chatHandler.ts#L456-L735) |
| `prepareChatConversationForSave` | [helpers.ts#L21-L53](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/chatConversations/helpers.ts#L21-L53) |
| `assertChatAppIsLiveForUser` | [chatApps.ts#L12-L17](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatApps.ts#L12-L17) |
| `canAccessChatAppAgentForUser` | [chatApps.ts#L24-L38](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatApps.ts#L24-L38) |
| Chat API 路由 | [chat.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/chat.ts) |
