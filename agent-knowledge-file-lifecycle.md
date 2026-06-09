# Agent 知识文件上传生命周期

本文档追踪 `POST /api/agent/:agentId/files` 接口上传知识文件后的完整生命周期，涵盖从 HTTP 入口到 Gemini 向量化处理的全流程。

## 总览

```
HTTP 请求 → Controller 层 → SDK 层 (锁内建库) → knowledgeBase SDK (存文件+入队)
                                                          ↓
                                                  RAG 队列消费 (worker)
                                                          ↓
                                                  GeminiRagProcessor 处理
                                                          ↓
                                                  文件状态: READY
```

---

## 1. Controller 层：uploadAgentFile

**文件**: [files.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/files.ts#L113-L180)

**路由**: [aiAgents.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/aiAgents.ts#L101) `POST /api/agent/:agentId/files`

### 1.1 normalizeUpload：多文件字段归一化

[normalizeUpload](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/files.ts#L28-L36) 函数兼容多种表单字段名：

- `ctx.request.files?.file`
- `ctx.request.files?.knowledgeBaseFile`
- `ctx.request.files?.upload`

如果输入是数组，取第一个元素；否则直接返回。如果为空返回 `undefined`。

### 1.2 校验流程

按顺序执行以下校验，任一校验失败都会抛出 `HTTPError`：

| 校验项 | 说明 | 失败处理 |
|--------|------|----------|
| upload 是否存在 | 文件必填 | 抛 400 "file is required" |
| filepath / path | 检查上传文件的本地路径 | 抛 400 "Invalid upload payload" |
| filename | `originalFilename` 或 `name`，默认 "agent-document" | - |
| mimetype | `mimetype` 或 `type` | - |
| fileSize | `size` 字段，转 number | - |
| isKnowledgeFileSupported | 检查文件扩展名或 MIME 类型 | **先 unlink 临时文件**，再抛 400 |

### 1.3 isKnowledgeFileSupported 校验逻辑

**文件**: [knowledgeFiles.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/types/src/core/knowledgeFiles.ts#L125-L148)

支持判断按以下优先级：

1. **扩展名匹配**：检查 `KNOWLEDGE_FILE_EXTENSIONS`（.txt, .md, .pdf, .docx 等 20 种）
2. **MIME 类型白名单**：检查 `KNOWLEDGE_FILE_MIME_TYPES`（22 种 MIME 类型）
3. **text/* 通配**：所有 `text/` 开头的 MIME 类型都支持
4. **image/* 排除**：所有图片类型直接拒绝

### 1.4 unlinkSafe：安全删除临时文件

[unlinkSafe](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/files.ts#L38-L47) 函数在以下场景被调用：

- 不支持的文件类型：校验失败立即删除
- `finally` 块：无论成功失败，最终都会删除临时文件

删除失败只打日志，不抛出异常。

### 1.5 调用 SDK

读取文件 buffer 后，调用 `sdk.ai.rag.uploadFileForAgent`，参数包括：
- `filename`
- `mimetype`
- `size`
- `buffer`
- `uploadedBy`

### 1.6 错误处理

捕获 SDK 错误后，会判断是否为 Gemini 上游不可用（503 或包含 "upstream unavailable"/"service unavailable"），如是则打 `ai.gemini.upstream_unavailable` 事件日志。

---

## 2. SDK 层：uploadFileForAgent

**文件**: [files.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/rag/files.ts#L147-L165)

### 2.1 ensureKnowledgeBaseForAgent：锁内创建知识库

[ensureKnowledgeBaseForAgent](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/rag/files.ts#L84-L115) 是核心函数，使用分布式锁保证并发安全。

**锁配置**:
- 名称: `LockName.AGENT_RAG_KNOWLEDGE_BASE`
- 类型: `LockType.AUTO_EXTEND`（自动续期）
- 资源: `agentId`

**锁内逻辑**:
1. 通过 `agentsSdk.getOrThrow(agentId)` 获取 agent
2. 遍历 `agent.knowledgeBases` 查找已存在的知识库
3. 如已存在，直接返回现有知识库
4. 如不存在，调用 `knowledgeBaseSdk.create()` 创建 GEMINI 类型知识库
   - 名称: `Agent files (${agent._id})`
   - 类型: `KnowledgeBaseType.GEMINI`
5. 更新 agent 的 `knowledgeBases` 字段为新创建的知识库 ID

### 2.2 调用 knowledgeBase SDK

获取或创建知识库后，调用 `knowledgeBaseSdk.uploadKnowledgeBaseFile` 执行文件上传。

---

## 3. knowledgeBase SDK：uploadKnowledgeBaseFile

**文件**: [uploads.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/knowledgeBase/uploads.ts#L40-L141)

### 3.1 前置检查

- 获取 workspaceId
- 校验知识库是否存在（`findKnowledgeBase`），不存在抛 404

### 3.2 生成标识

- `fileId`: 通过 `docIds.generateKnowledgeBaseFileID(knowledgeBaseId)` 生成
- `objectStoreKey`: 格式为 `${workspaceId}/ai/knowledge-bases/${knowledgeBaseId}/files/${fileId}/${filename}`

### 3.3 上传对象存储

写入 `ObjectStoreBuckets.APPS` bucket：

```typescript
await objectStore.upload({
  bucket: ObjectStoreBuckets.APPS,
  filename: objectStoreKey,
  body: input.buffer,
  type: input.mimetype,
})
```

### 3.4 创建 KnowledgeBaseFile 记录

[createKnowledgeBaseFile](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/knowledgeBase/files.ts#L34-L74) 创建数据库记录，初始状态为 `PROCESSING`：

| 字段 | 值 |
|------|----|
| `status` | `KnowledgeBaseFileStatus.PROCESSING` |
| `ragSourceId` | 默认为 `_id`（后续会被真实值覆盖） |
| `errorMessage` | `undefined` |
| `processedAt` | `undefined` |

### 3.5 入队 RAG 处理

调用 `enqueueRagFileIngestion` 将文件加入处理队列：

```typescript
await enqueueRagFileIngestion({
  workspaceId,
  knowledgeBaseId: input.knowledgeBaseId,
  fileId: knowledgeBaseFile._id,
  objectStoreKey,
})
```

**入队成功**:
- 触发 `events.ai.ragFileUploaded` 事件
- 返回 KnowledgeBaseFile 记录

**入队失败**：
- 将文件状态更新为 `FAILED`
- 设置 `errorMessage`
- 抛出异常

### 3.6 对象存储上传失败的回滚

如果对象存储上传失败，会尝试删除已上传的文件（忽略不存在的情况），然后抛出异常。

---

## 4. RAG 队列：ragQueue.init

**文件**: [ragQueue.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/rag/ragQueue.ts)

### 4.1 队列配置

| 配置项 | 值 |
|--------|----|
| 队列名 | `JobQueue.RAG_INGESTION` |
| 并发数 | 默认 2 |
| 重试次数 | 5 次 |
| 退避策略 | 指数退避，初始 10 秒 |
| 超时 | 10 分钟 |
| 最大停滞次数 | 3 次 |
| 成功后移除 | 是 |
| 失败保留 | 1000 条 |

### 4.2 enqueueRagFileIngestion

[enqueueRagFileIngestion](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/rag/ragQueue.ts#L164-L173) 入队函数：

- 惰性初始化队列（`init()`）
- 使用 `fileId` 作为 jobId，保证幂等
- Job 数据包含: `workspaceId`, `knowledgeBaseId`, `fileId`, `objectStoreKey`

### 4.3 消费处理流程

[init](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/rag/ragQueue.ts#L54-L162) 函数注册消费处理器，在 `context.doInWorkspaceContext` 中执行：

#### 步骤 1：检查知识库是否存在

```typescript
knowledgeBaseConfig = await knowledgeBase.find(knowledgeBaseId)
if (!knowledgeBaseConfig) {
  await job.discard()  // 丢弃任务，不再重试
  return
}
```

#### 步骤 2：检查文件是否存在

```typescript
try {
  knowledgeBaseFile = await knowledgeBase.getKnowledgeBaseFileOrThrow(fileId)
} catch (error: any) {
  if (error?.status === 404) {
    await job.discard()  // 文件已删除，丢弃任务
    return
  }
  throw error
}
```

#### 步骤 3：补全 objectStoreKey

如果文件记录中没有 `objectStoreKey` 但 job 数据中有，则补全到文件记录。

#### 步骤 4：加载文件 buffer

[loadFileBuffer](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/rag/ragQueue.ts#L175-L186) 从对象存储读取文件流并拼接为 Buffer。

#### 步骤 5：调用 ingestKnowledgeBaseFile

```typescript
await ingestKnowledgeBaseFile(
  knowledgeBaseConfig,
  knowledgeBaseFile,
  buffer
)
```

### 4.4 错误处理：handleProcessingError

[handleProcessingError](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/rag/ragQueue.ts#L188-L214) 在处理失败时被调用：

- **非最后一次重试**：只打日志，由 bull 自动重试
- **最后一次重试失败**：
  - `file.status = FAILED`
  - 设置 `errorMessage`
  - 保存到数据库
  - 抛出异常（让 bull 标记 job 失败）

---

## 5. GeminiRagProcessor：ingestKnowledgeBaseFile

**文件**: [gemini.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/rag/processors/gemini.ts#L18-L85)

### 5.1 处理器实例化

通过 `getProcessor(knowledgeBase)` 工厂函数根据知识库类型创建对应处理器，目前仅支持 `GeminiRagProcessor`。

### 5.2 ingestKnowledgeBaseFile 流程

#### 步骤 1：调用 ingestGeminiFile

```typescript
const ingested = await ingestGeminiFile({
  vectorStoreId: this.knowledgeBase.config.googleFileStoreId,
  filename: input.filename,
  mimetype: input.mimetype,
  buffer: fileBuffer,
})
```

#### 步骤 2：更新文件状态为 READY

成功后更新文件记录：

| 字段 | 新值 |
|------|------|
| `status` | `KnowledgeBaseFileStatus.READY` |
| `ragSourceId` | `ingested.fileId`（Gemini 返回的真实 file_id） |
| `processedAt` | 当前时间 ISO 字符串 |
| `errorMessage` | `undefined` |

#### 步骤 3：触发 ragFileProcessed 事件

```typescript
events.ai.ragFileProcessed({
  knowledgeBaseId,
  fileId,
  sourceType: input.source?.type,
  processor: this.knowledgeBase.type,
})
```

### 5.3 ingestGeminiFile 实现

**文件**: [geminiFileStore.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/knowledgeBase/geminiFileStore.ts#L190-L237)

通过 LiteLLM 代理调用 Gemini RAG ingest API：

- **Endpoint**: `${LITELLM_URL}/v1/rag/ingest`
- **Method**: `POST`
- **Body**:
  ```json
  {
    "file": {
      "filename": "...",
      "content": "<base64 encoded>",
      "content_type": "..."
    },
    "ingest_options": {
      "name": "...",
      "vector_store": {
        "custom_llm_provider": "gemini",
        "vector_store_id": "...",
        "api_key": "..."
      }
    }
  }
  ```

返回 `{ fileId: payload.file_id }`。

如果返回 `status === "failed"` 且有 `error`，则抛出 HTTPError。

---

## 6. 完整状态流转

```
                上传中 (controller)
                      │
                      ▼
              PROCESSING (初始状态)
                      │
            ┌─────────┴─────────┐
            ▼                   ▼
        入队成功            入队失败 → FAILED
            │
    (RAG 队列消费)
            │
  ┌─────────┴──────────┐
  ▼                    ▼
重试中              处理成功 → READY
  │              (ragSourceId 已更新)
  │              (processedAt 已设置)
  │
  └── 重试耗尽 → FAILED
                (errorMessage 已设置)
```

---

## 7. 关键文件索引

| 层级 | 文件 | 关键函数 |
|------|------|----------|
| Controller | [files.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/files.ts) | `uploadAgentFile`, `normalizeUpload`, `unlinkSafe` |
| 类型定义 | [knowledgeFiles.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/types/src/core/knowledgeFiles.ts) | `isKnowledgeFileSupported` |
| RAG SDK | [files.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/rag/files.ts) | `uploadFileForAgent`, `ensureKnowledgeBaseForAgent`, `ingestKnowledgeBaseFile` |
| knowledgeBase | [uploads.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/knowledgeBase/uploads.ts) | `uploadKnowledgeBaseFile` |
| knowledgeBase | [files.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/knowledgeBase/files.ts) | `createKnowledgeBaseFile`, `updateKnowledgeBaseFile` |
| knowledgeBase | [crud.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/knowledgeBase/crud.ts) | `create`, `find` |
| RAG 队列 | [ragQueue.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/rag/ragQueue.ts) | `init`, `enqueueRagFileIngestion` |
| Gemini 处理器 | [gemini.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/rag/processors/gemini.ts) | `GeminiRagProcessor.ingestKnowledgeBaseFile` |
| Gemini 文件存储 | [geminiFileStore.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/knowledgeBase/geminiFileStore.ts) | `ingestGeminiFile`, `createGeminiFileStore` |
