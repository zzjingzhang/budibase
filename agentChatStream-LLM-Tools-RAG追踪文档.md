# agentChatStream 完整流程追踪：LLM、工具与 RAG 来源准备

## 目录

1. [Controller 入口：agentChatStream 权限解析](#1-controller-入口agentchatstream-权限解析)
2. [prepareAgentChatRun：并行初始化三大核心组件](#2-prepareagentchatrun并行初始化三大核心组件)
3. [buildPromptAndTools：工具过滤与系统提示构建](#3-buildpromptandtools工具过滤与系统提示构建)
4. [agentRuntime：report_used_sources 注入与 ToolLoopAgent 配置](#4-agentruntimereport_used_sources-注入与-toolloopagent-配置)
5. [onStepFinish：工具调用收集、知识来源处理与配额统计](#5-onstepfinish工具调用收集知识来源处理与配额统计)
6. [Finish Metadata：ragSources 与 usage 写入](#6-finish-metadataragsources-与-usage-写入)

---

## 1. Controller 入口：agentChatStream 权限解析

### 1.1 入口函数

入口函数位于 [chatConversations.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L389-L531) 中的 `agentChatStream`。

```typescript
export async function agentChatStream(ctx: UserCtx<ChatAgentRequest, void>) {
  const db = context.getWorkspaceDB()
  const { agentId, chat, chatAppId, userId } =
    await resolveChatStreamRequest(ctx)
  // ...
}
```

### 1.2 权限解析流程

权限解析通过 `resolveChatStreamRequest` 函数完成，该函数执行以下检查：

#### 1.2.1 用户身份获取

- 通过 `getGlobalUserId(ctx)` 从 `ctx.user` 中提取全局用户 ID
- 优先级：`globalId` > `userId` > `_id`

#### 1.2.2 预览权限检查

```typescript
const isBuilderOrAdmin = usersSdk.users.isAdminOrBuilder(ctx.user)
const requestedPreview = chat.isPreview === true

if (requestedPreview && !isBuilderOrAdmin) {
  throw new HTTPError("Forbidden", 403)
}
```

- 只有管理员/构建者可以使用预览模式

#### 1.2.3 Chat App 存在性与状态检查

- 非预览模式必须提供 `chatAppId`
- 通过 `db.tryGet<ChatApp>(chatAppId)` 获取 Chat App
- 调用 `assertChatAppIsLiveForUser(ctx, chatApp)` 检查是否已上线

`assertChatAppIsLiveForUser` 实现见 [chatApps.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatApps.ts#L12-L17)：
- 管理员/构建者不受 live 状态限制
- 普通用户只能访问 `chatApp.live === true` 的应用

#### 1.2.4 Agent 权限检查

- 通过 `getEnabledChatAgentConfig(chatApp, agentId)` 检查 Agent 是否在 Chat App 中启用
- 调用 `assertCanAccessChatAgent(ctx, chatAgentConfig)` 进行角色权限验证

`canAccessChatAppAgentForUser` 实现见 [chatApps.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatApps.ts#L24-L38)：
- 管理员/构建者直接返回 `true`
- 未设置 `roleId` 的 Agent 允许所有用户访问
- 设置了 `roleId` 的 Agent 通过 `AccessController.hasAccess()` 检查用户角色权限

#### 1.2.5 已有会话权限检查

通过 `getExistingChatForStream` 验证：
- 会话必须存在
- 会话必须属于该 Chat App（非预览模式）
- 会话的 `userId` 必须匹配当前用户（如果设置了 userId）
- 不能切换 Agent

---

## 2. prepareAgentChatRun：并行初始化三大核心组件

### 2.1 函数定义

`prepareAgentChatRun` 位于 [agentRuntime.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/agents/agentRuntime.ts#L64-L231)，是 Agent 聊天运行时的核心准备函数。

### 2.2 并行调用三大组件

使用 `Promise.all` 并行执行三个初始化任务，提高启动效率：

```typescript
const [promptAndTools, llm, modelMessages] = await Promise.all([
  sdk.ai.agents.buildPromptAndTools(agent, {
    baseSystemPrompt: ai.agentSystemPrompt(user),
    includeGoal: false,
  }),
  sdk.ai.llm.createLLM(
    aiConfigId ?? agent.aiconfig,
    sessionId,
    undefined,
    agentId
  ),
  providedModelMessages ?? prepareModelMessages(chat?.messages ?? []),
])
```

#### 2.2.1 buildPromptAndTools

构建系统提示词和可用工具集（详见第 3 章）。

#### 2.2.2 createLLM

创建 LLM 客户端，实现见 [llm/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/llm/index.ts#L13-L42)：

```typescript
export async function createLLM(
  configId: string,
  sessionId?: string,
  span?: tracer.Span,
  agentId?: string
): Promise<LLMResponse> {
  const aiConfig = await sdk.ai.configs.find(configId)
  
  if (aiConfig.provider === BUDIBASE_AI_PROVIDER_ID) {
    await quotas.throwIfBudibaseAICreditsExceeded()
  }

  if (aiConfig.provider === BUDIBASE_AI_PROVIDER_ID && !env.SELF_HOSTED) {
    return createBBAIClient(...)
  }

  return createLiteLLMOpenAI(aiConfig, sessionId, span, agentId)
}
```

返回的 LLM 对象包含：
- `chat`：聊天模型实例
- `providerOptions`：提供者特定选项函数
- `contextWindowTokens`：上下文窗口大小

#### 2.2.3 prepareModelMessages

准备历史消息，实现见 [chatConversations/helpers.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/chatConversations/helpers.ts#L98-L109)：

```typescript
export const prepareModelMessages = async (
  messages: ChatConversationRequest["messages"]
): Promise<ModelMessage[]> => {
  const modelMessages = await convertToModelMessages(messages)

  return pruneMessages({
    messages: modelMessages,
    reasoning: "all",
    toolCalls: "before-last-2-messages",
    emptyMessages: "remove",
  })
}
```

消息裁剪策略：
- 保留所有推理内容
- 工具调用仅保留最近 2 条消息之前的
- 移除空消息

---

## 3. buildPromptAndTools：工具过滤与系统提示构建

### 3.1 函数定义

`buildPromptAndTools` 位于 [agents/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/agents/utils.ts#L143-L196)。

### 3.2 Helper 工具定义

首先定义了一组 helper 工具名称：

```typescript
const HELPER_TOOL_NAMES = new Set([
  "list_tables",
  "get_table",
  "list_automations",
  "get_automation",
  "list_knowledge_files",
  "search_knowledge",
])

const isHelperTool = (tool: Pick<AiToolDefinition, "name">) =>
  HELPER_TOOL_NAMES.has(tool.name)
```

### 3.3 工具过滤逻辑

#### 3.3.1 基于 enabledTools 过滤非 helper 工具

```typescript
const allTools = await getAvailableTools(agent.aiconfig)
const enabledToolNames = new Set(agent.enabledTools || [])
const enabledTools = addHelperTools(
  allTools.filter(
    tool => enabledToolNames.has(tool.name) && !isHelperTool(tool)
  ),
  allTools
)
```

- 从 `allTools` 中筛选出 `agent.enabledTools` 包含的工具
- **初始过滤时排除 helper 工具**（因为 helper 工具是自动添加的）
- 然后通过 `addHelperTools` 函数自动添加必要的 helper 工具

#### 3.3.2 自动添加表和 Automation 的 helper 工具

`addHelperTools` 函数（[utils.ts#L202-L238](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/agents/utils.ts#L202-L238)）：

**当启用了表工具时，自动加入 list_tables 和 get_table：**

```typescript
if (
  enabledTools.some(
    tool =>
      tool.sourceType === ToolType.EXTERNAL_TABLE ||
      tool.sourceType === ToolType.INTERNAL_TABLE
  )
) {
  for (const toolName of ["get_table", "list_tables"]) {
    if (seenTools.has(toolName)) continue
    let tool = toolByName.get(toolName)
    if (tool) {
      enabledTools.push(tool)
      seenTools.add(tool.name)
    }
  }
}
```

**当启用了 Automation 工具时，自动加入 list_automations 和 get_automation：**

```typescript
if (enabledTools.some(tool => tool.sourceType === ToolType.AUTOMATION)) {
  for (const toolName of ["get_automation", "list_automations"]) {
    if (seenTools.has(toolName)) continue
    let tool = toolByName.get(toolName)
    if (tool) {
      enabledTools.push(tool)
      seenTools.add(tool.name)
    }
  }
}
```

### 3.4 知识库工具自动添加

当 Agent 配置了 `knowledgeBases` 时，自动添加知识相关工具：

```typescript
const hasKnowledgeBases = agent.knowledgeBases?.some(Boolean) ?? false

if (
  hasKnowledgeBases &&
  !enabledTools.some(tool => tool.name === "list_knowledge_files")
) {
  enabledTools.push(createKnowledgeFilesTool(agentId))
}
if (
  hasKnowledgeBases &&
  !enabledTools.some(tool => tool.name === "search_knowledge")
) {
  enabledTools.push(createKnowledgeSearchTool(agentId))
}
```

#### 3.4.1 list_knowledge_files 工具

实现在 [knowledgeFiles.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/ai/tools/budibase/knowledgeFiles.ts#L192-L280)：
- 列出 Agent 关联的知识文件
- 支持文件名过滤（smart/exact/contains 三种匹配模式）
- 返回文件元数据：大小、状态、上传时间、MIME 类型等

#### 3.4.2 search_knowledge 工具

实现在 [knowledgeFiles.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/ai/tools/budibase/knowledgeFiles.ts#L282-L326)：
- 搜索知识文件中的相关内容
- 返回：`context`（文本内容）、`sources`（来源元数据）、`chunks`（文本块）

### 3.5 知识系统提示追加

当存在知识库时，在系统提示后追加知识使用说明：

```typescript
const systemPrompt = ai.composeAutomationAgentSystemPrompt({
  baseSystemPrompt,
  goal: includeGoal ? agent.goal : undefined,
  promptInstructions: agent.promptInstructions,
  includeGoal,
})

const resolvedSystemPrompt = hasKnowledgeBases
  ? `${systemPrompt}\n\nWhen users ask about attached files... call list_knowledge_files...\n\nFor factual questions... call search_knowledge before answering...\n\nIf you used search_knowledge context in your final answer, call report_used_sources immediately before your final response...`
  : systemPrompt
```

知识系统提示包含三个核心指令：
1. **文件相关问题**：使用 `list_knowledge_files`，不要猜测文件元数据
2. **事实性问题**：先调用 `search_knowledge`，找不到相关内容要明确说明
3. **引用来源**：最终回答前调用 `report_used_sources`，仅传递直接支持答案的 sourceIds

---

## 4. agentRuntime：report_used_sources 注入与 ToolLoopAgent 配置

### 4.1 report_used_sources 工具注入

在 `prepareAgentChatRun` 中，当检测到 `search_knowledge` 工具存在时，动态注入 `report_used_sources` 工具：

```typescript
const retrievedKnowledgeSourceById = new Map<
  string,
  NonNullable<AgentMessageMetadata["ragSources"]>[number]
>()
const usedKnowledgeSourceById = new Map<
  string,
  NonNullable<AgentMessageMetadata["ragSources"]>[number]
>()

const reportUsedSourcesTool = createReportUsedSourcesTool({
  getSourceById: sourceId => retrievedKnowledgeSourceById.get(sourceId),
  onAcceptedSources: accepted => setUsedKnowledgeSources(accepted),
})
if (tools.search_knowledge) {
  tools.report_used_sources = reportUsedSourcesTool
}
```

两个 Map 的作用：
- `retrievedKnowledgeSourceById`：存储所有 `search_knowledge` 返回的来源（候选池）
- `usedKnowledgeSourceById`：存储 `report_used_sources` 确认使用的来源（最终引用）

### 4.2 createReportUsedSourcesTool 实现

实现在 [reportUsedSources.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/ai/tools/budibase/knowledge/reportUsedSources.ts#L12-L44)：

```typescript
export const createReportUsedSourcesTool = ({
  getSourceById,
  onAcceptedSources,
}) =>
  tool({
    description:
      "Report the specific knowledge sources that were actually used in the final answer.",
    inputSchema: z.object({
      sourceIds: z.array(z.string().trim().min(1)).default([]),
    }),
    execute: async ({ sourceIds }) => {
      const accepted = []
      const ignored = []

      for (const sourceId of sourceIds || []) {
        const source = getSourceById(sourceId)
        if (!source) {
          ignored.push(sourceId)
          continue
        }
        accepted.push(source)
      }

      onAcceptedSources(accepted)

      return {
        accepted,
        acceptedCount: accepted.length,
        ignored,
        ignoredCount: ignored.length,
      }
    },
  })
```

工作原理：
- 接收 `sourceIds` 数组
- 通过 `getSourceById` 从检索到的来源中查找
- 有效的来源加入 `accepted`，无效的加入 `ignored`
- 通过 `onAcceptedSources` 回调更新已使用来源集合

### 4.3 ToolLoopAgent 配置

```typescript
const hasTools = Object.keys(tools).length > 0
const agentRunner = new ToolLoopAgent({
  model: wrapLanguageModel({
    model: llm.chat,
    middleware: extractReasoningMiddleware({
      tagName: "think",
    }),
  }),
  instructions: promptAndTools.systemPrompt || undefined,
  tools: hasTools ? tools : undefined,
  toolChoice: hasTools ? "auto" : "none",
  stopWhen: stepCountIs(30),
  providerOptions: llm.providerOptions?.(hasTools),
})
```

#### 4.3.1 stopWhen: stepCountIs(30)

- `ToolLoopAgent` 从 `ai` 库导入
- `stepCountIs(30)` 设置最大步骤数为 30
- 当工具循环执行了 30 步后，Agent 会停止执行
- 这是一个安全机制，防止无限工具调用循环

#### 4.3.2 模型包装

- 使用 `wrapLanguageModel` 包装原始模型
- 应用 `extractReasoningMiddleware` 中间件，从 `<think>` 标签中提取推理内容

---

## 5. onStepFinish：工具调用收集、知识来源处理与配额统计

### 5.1 onStepFinish 回调定义

在 `agentRunner.stream()` 调用中配置 `onStepFinish` 回调（[agentRuntime.ts#L158-L224](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/agents/agentRuntime.ts#L158-L224)）：

```typescript
async onStepFinish({
  content,
  toolCalls,
  toolResults,
  response,
  usage,
}) {
  // 使用量统计
  if (!contextUsage.input) {
    contextUsage.input = usage
  }
  contextUsage.output = usage
  
  sessionLogIndexer.addRequestId(response?.id)
  
  // 工具调用通知
  if (onToolCalls) {
    const toolNames = toolCalls
      .map(toolCall => toolCall.toolName)
      .filter(Boolean)
    if (toolNames.length) {
      onToolCalls(toolNames)
    }
  }
  
  // 更新待处理工具调用
  if (pendingToolCalls) {
    updatePendingToolCalls(pendingToolCalls, toolCalls, toolResults)
  }
  
  // 处理工具结果...
}
```

### 5.2 收集 toolCalls

```typescript
if (onToolCalls) {
  const toolNames = toolCalls
    .map(toolCall => toolCall.toolName)
    .filter(Boolean)
  if (toolNames.length) {
    onToolCalls(toolNames)
  }
}
```

- 从每一步的 `toolCalls` 中提取工具名称
- 通过 `onToolCalls` 回调通知外部（可选）

### 5.3 处理 search_knowledge 结果

遍历所有 `toolResults`，处理知识搜索结果：

```typescript
for (const toolResult of toolResults) {
  if (
    toolResult.toolName === "search_knowledge" &&
    !toolResult.preliminary
  ) {
    const output = toolResult.output as
      | { sources?: AgentMessageMetadata["ragSources"] }
      | undefined
    for (const source of output?.sources || []) {
      if (!source?.sourceId) {
        continue
      }
      const existing = retrievedKnowledgeSourceById.get(
        source.sourceId
      )
      retrievedKnowledgeSourceById.set(source.sourceId, {
        ...existing,
        ...source,
      })
    }
  }
  // ...
}
```

关键点：
- 只处理非 `preliminary`（初步）的结果
- 将检索到的来源存入 `retrievedKnowledgeSourceById` Map
- 支持合并（如果同一 sourceId 多次出现，后续会覆盖/合并）
- 这些来源是 `report_used_sources` 的有效来源池

### 5.4 处理 report_used_sources 结果

```typescript
if (
  toolResult.toolName === "report_used_sources" &&
  !toolResult.preliminary
) {
  const output = toolResult.output as
    | { accepted?: AgentMessageMetadata["ragSources"] }
    | undefined
  setUsedKnowledgeSources(output?.accepted)
}
```

- 当 `report_used_sources` 工具执行完成后
- 调用 `setUsedKnowledgeSources` 更新已使用的知识来源
- 这些来源最终会作为 `ragSources` 返回给前端

### 5.5 ActionType.AI_AGENT 配额统计

对每个工具结果增加配额：

```typescript
await quotas.addAction(ActionType.AI_AGENT, async () => {})
```

- 每执行一个工具调用，消耗一个 `AI_AGENT` 配额
- `quotas.addAction` 是配额系统的标准方法
- 注意：这是在 `for (const toolResult of toolResults)` 循环内，所以**每个工具结果都会触发一次配额消耗**

同样的模式也出现在自动化 Agent 步骤中，见 [automations/steps/ai/agent.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/steps/ai/agent.ts#L148-L150)：

```typescript
for (const _toolResult of toolResults) {
  await quotas.addAction(ActionType.AI_AGENT, async () => {})
}
```

---

## 6. Finish Metadata：ragSources 与 usage 写入

### 6.1 流式响应配置

在 `agentChatStream` 控制器中，通过 `result.pipeUIMessageStreamToResponse()` 配置消息流，其中 `messageMetadata` 回调用于为不同类型的消息部分添加元数据（[chatConversations.ts#L434-L472](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/chatConversations.ts#L434-L472)）。

### 6.2 finish 部分元数据

```typescript
if (part.type === "finish") {
  const usedKnowledgeSources = run.getUsedKnowledgeSourcesMetadata()
  
  const finishReason = (part as { finishReason?: string }).finishReason
  const toolCallsIncomplete =
    pendingToolCalls.size > 0 || finishReason === "tool-calls"

  const finishPart = part as {
    totalUsage?: LanguageModelUsage | undefined
  }
  const usage = buildAgentMessageUsage({
    inputUsage: run.contextUsage.input ?? finishPart.totalUsage,
    outputUsage: run.contextUsage.output ?? finishPart.totalUsage,
    maxTokens: run.contextWindowTokens,
    systemPromptTokens: run.systemPromptTokens,
  })

  return {
    ...sharedMetadata,
    ...(usedKnowledgeSources?.length
      ? { ragSources: usedKnowledgeSources }
      : {}),
    createdAt: streamStartTime,
    completedAt: Date.now(),
    ...(usage ? { usage } : {}),
    ...(toolCallsIncomplete && {
      error: formatIncompleteToolCallError([]),
    }),
  }
}
```

### 6.3 ragSources 写入

```typescript
const usedKnowledgeSources = run.getUsedKnowledgeSourcesMetadata()

return {
  ...,
  ...(usedKnowledgeSources?.length
    ? { ragSources: usedKnowledgeSources }
    : {}),
  ...
}
```

- `getUsedKnowledgeSourcesMetadata()` 返回 `Array.from(usedKnowledgeSourceById.values())`
- 只有当有已使用的知识来源时，才会在 finish metadata 中包含 `ragSources`
- 这些来源是经过 `report_used_sources` 确认的

### 6.4 usage 写入

`usage` 通过 `buildAgentMessageUsage` 函数构建，实现在 [usage.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/agents/usage.ts#L15-L47)：

```typescript
export const buildAgentMessageUsage = ({
  inputUsage,
  outputUsage,
  maxTokens,
  systemPromptTokens,
}: BuildAgentMessageUsageParams): AgentMessageUsage | undefined => {
  const output = outputUsage ?? inputUsage
  const totalInput = tokens(inputUsage.inputTokens)
  const system = Math.min(systemPromptTokens, totalInput)
  const cachedInput = Math.min(
    tokens(inputUsage.inputTokenDetails.cacheReadTokens),
    totalInput - system
  )
  const input = totalInput - system - cachedInput
  const reasoning = tokens(output.outputTokenDetails.reasoningTokens)
  const outputTokens = Math.max(0, tokens(output.outputTokens) - reasoning)
  
  const segments = [
    { type: "system", tokens: system },
    { type: "input", tokens: input },
    { type: "cachedInput", tokens: cachedInput },
    { type: "output", tokens: outputTokens },
    { type: "reasoning", tokens: reasoning },
  ]

  return {
    ...(maxTokens === undefined ? {} : { maxTokens }),
    segments: segments.filter(segment => segment.tokens > 0),
  }
}
```

使用量分段统计：
- **system**：系统提示词 token 数
- **input**：用户输入 token 数（扣除系统和缓存）
- **cachedInput**：缓存读取的输入 token 数
- **output**：输出 token 数（扣除推理）
- **reasoning**：推理过程 token 数

优先级：
- 优先使用 `run.contextUsage.input/output`（从 onStepFinish 累积的）
- 回退到 `finishPart.totalUsage`

### 6.5 finish metadata 完整结构

最终 finish 消息的元数据包含：

| 字段 | 说明 |
|------|------|
| `toolDisplayNames` | 工具可读名称映射（如果有工具） |
| `ragSources` | 已使用的 RAG 知识来源（如果有） |
| `createdAt` | 流开始时间 |
| `completedAt` | 流完成时间 |
| `usage` | Token 使用量统计（分 segment） |
| `error` | 工具调用不完整错误（如果有） |

---

## 总结：完整流程图

```
agentChatStream (controller)
    │
    ├─ resolveChatStreamRequest → 权限检查
    │   ├─ getGlobalUserId
    │   ├─ 预览权限检查
    │   ├─ ChatApp 存在性 + live 检查
    │   ├─ Agent 启用检查
    │   └─ Agent 角色权限检查 (canAccessChatAppAgentForUser)
    │
    └─ prepareAgentChatRun
        │
        ├─ [并行] buildPromptAndTools
        │   ├─ getAvailableTools → 获取所有可用工具
        │   ├─ 过滤：enabledTools && !isHelperTool
        │   ├─ addHelperTools
        │   │   ├─ 有表工具 → + list_tables, get_table
        │   │   └─ 有automation工具 → + list_automations, get_automation
        │   ├─ hasKnowledgeBases
        │   │   ├─ → + list_knowledge_files
        │   │   └─ → + search_knowledge
        │   ├─ composeAutomationAgentSystemPrompt
        │   └─ 知识库时追加知识系统提示
        │
        ├─ [并行] createLLM
        │   ├─ 检查配额 (Budibase AI)
        │   └─ createBBAIClient / createLiteLLMOpenAI
        │
        ├─ [并行] prepareModelMessages
        │   ├─ convertToModelMessages
        │   └─ pruneMessages
        │
        ├─ 初始化 retrievedKnowledgeSourceById Map
        ├─ 初始化 usedKnowledgeSourceById Map
        ├─ createReportUsedSourcesTool
        ├─ tools.search_knowledge 存在 → 注入 report_used_sources
        │
        └─ new ToolLoopAgent
            ├─ model: wrapLanguageModel + extractReasoningMiddleware
            ├─ instructions: systemPrompt
            ├─ tools
            ├─ stopWhen: stepCountIs(30)
            │
            └─ stream()
                ├─ onStepFinish
                │   ├─ 累积 contextUsage
                │   ├─ 收集 toolCalls
                │   ├─ updatePendingToolCalls
                │   ├─ 处理 search_knowledge 结果 → 存入 retrievedKnowledgeSourceById
                │   ├─ 处理 report_used_sources → 更新 usedKnowledgeSourceById
                │   └─ 每个 toolResult → quotas.addAction(ActionType.AI_AGENT)
                │
                └─ onFinish
                    └─ (controller 侧)
                        ├─ getUsedKnowledgeSourcesMetadata → ragSources
                        ├─ buildAgentMessageUsage → usage
                        └─ 写入 finish metadata
```
