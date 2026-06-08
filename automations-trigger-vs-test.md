# POST /api/automations/:id/trigger 与 POST /api/automations/:id/test 差异分析

## 概述

本文档详细分析 Budibase 中两个自动化 API 端点的实现差异：
- `POST /api/automations/:id/trigger` — 用于触发已部署/生产环境的自动化
- `POST /api/automations/:id/test` — 用于在 builder 开发环境中测试自动化

两个端点的控制器实现均位于 [automation.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/automation.ts)，路由定义位于 [automation.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/automation.ts)。

---

## 一、trigger 端点分析

### 1.1 基本信息

- **路由位置**：[automation.ts#L83](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/automation.ts#L83-L83)
- **控制器位置**：[automation.ts#L273-L325](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/automation.ts#L273-L325)
- **权限**：需要 `AUTOMATION.EXECUTE` 权限
- **中间件**：`paramResource("id")`、recaptcha

### 1.2 核心逻辑分支

trigger 端点有两条主要执行路径，由 `hasCollectStep && features.isSyncAutomationsEnabled()` 条件决定：

```
                        ┌──────────────────────────────────┐
                        │         trigger 函数入口          │
                        └───────────────┬──────────────────┘
                                        │
                        ┌───────────────▼──────────────────┐
                        │  hasCollectStep &&               │
                        │  isSyncAutomationsEnabled() ?    │
                        └───────┬───────────────┬──────────┘
                                │               │
                       是 ──────┘               └────── 否
                                │                       │
                   ┌────────────▼───────────┐   ┌───────▼───────────────┐
                   │ 同步模式 (COLLECT + SYNC)│   │ 异步/限制模式         │
                   │ 调用 externalTrigger   │   │ 检查 dev workspace    │
                   │  getResponses: true    │   │ 限制 ALLOW_DEV_AUTO   │
                   │ 等待 steps 响应        │   │ 异步触发不等待结果     │
                   └────────────────────────┘   └───────────────────────┘
```

#### 分支一：COLLECT 步骤 + SYNC_AUTOMATIONS 开启

当自动化包含 COLLECT 步骤且同步自动化功能启用时，采用同步调用模式：

**代码位置**：[automation.ts#L279-L307](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/automation.ts#L279-L307)

```typescript
let hasCollectStep = sdk.automations.utils.checkForCollectStep(automation)
if (hasCollectStep && (await features.isSyncAutomationsEnabled())) {
  const response = await triggers.externalTrigger(
    automation,
    {
      fields: ctx.request.body.fields,
      user: sdk.users.getUserContextBindings(ctx.user),
      timeout: ctx.request.body.timeout * 1000 || env.AUTOMATION_THREAD_TIMEOUT,
    },
    { getResponses: true }
  )

  if (!("steps" in response)) {
    ctx.throw(400, "Unable to collect response")
  }

  let collectedValue = response.steps.find(
    step => step.stepId === AutomationActionStepId.COLLECT
  )
  ctx.body = collectedValue?.outputs
}
```

**关键细节**：

1. **COLLECT 步骤检测**：通过 [checkForCollectStep](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/shared-core/src/sdk/documents/automations.ts#L54-L56) 递归检查自动化定义中是否包含 COLLECT 步骤
2. **SYNC_AUTOMATIONS 特性**：通过 `features.isSyncAutomationsEnabled()` 检查同步自动化特性是否启用（pro 版特性）
3. **externalTrigger 参数**：
   - 传入 `{ getResponses: true }` 选项，表示需要等待并获取完整的步骤响应
   - 这会导致通过 `executeInThread` 同步执行自动化，而非放入队列异步执行
   - 参考 [externalTrigger](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/triggers.ts#L196-L285)
4. **响应提取**：从返回的 `steps` 数组中查找 `stepId === AutomationActionStepId.COLLECT` 的步骤，返回其 `outputs`
5. **超时处理**：使用请求体中的 `timeout`（秒）或环境变量 `AUTOMATION_THREAD_TIMEOUT` 作为超时时间

#### 分支二：非 COLLECT 或 SYNC 未开启

当不满足上述条件时，走异步触发分支，同时对开发环境进行限制：

**代码位置**：[automation.ts#L308-L324](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/automation.ts#L308-L324)

```typescript
else {
  if (
    ctx.appId &&
    !dbCore.isProdWorkspaceID(ctx.appId) &&
    !env.ALLOW_DEV_AUTOMATIONS
  ) {
    ctx.throw(400, "Only apps in production support this endpoint")
  }
  await triggers.externalTrigger(automation, {
    ...ctx.request.body,
    appId: ctx.appId,
    user: sdk.users.getUserContextBindings(ctx.user),
  })
  ctx.body = {
    message: `Automation ${automation._id} has been triggered.`,
  }
}
```

**关键细节**：

1. **dev workspace 限制**：
   - 使用 `dbCore.isProdWorkspaceID(ctx.appId)` 判断是否为生产环境
   - 如果是开发环境且 `env.ALLOW_DEV_AUTOMATIONS` 为 false，则抛出 400 错误
   - 这意味着 trigger 端点默认只对生产环境开放
2. **异步触发**：
   - 调用 `externalTrigger` 时不传 `getResponses` 选项（默认为 false）
   - 自动化会被添加到 Bull 队列 `automationQueue` 中异步执行
   - 接口立即返回成功消息，不等待执行结果
   - 参考 [externalTrigger#L280-L284](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/triggers.ts#L280-L284)

---

## 二、test 端点分析

### 2.1 基本信息

- **路由位置**：[automation.ts#L84-L89](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/automation.ts#L84-L89)
- **控制器位置**：[automation.ts#L338-L425](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/automation.ts#L338-L425)
- **权限**：需要 `AUTOMATION.EXECUTE` 权限
- **中间件**：`appInfoMiddleware({ appType: AppType.DEV })`、`paramResource("id")`
- **特点**：明确限定为 DEV 应用，只能在开发环境中使用

### 2.2 async 标志解析

test 端点支持同步和异步两种模式，由 `async` 标志控制：

**代码位置**：[automation.ts#L353-L357](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/automation.ts#L353-L357)

```typescript
const query = ctx.request.query as Record<string, unknown>
const bodyAsync = body?.async
const asyncFlag =
  isQsTrue(String(query?.async)) || isQsTrue(String(bodyAsync))
```

**关键细节**：
- `async` 标志可以从两个地方获取：
  - **URL query 参数**：如 `POST /api/automations/:id/test?async=true`
  - **请求体 body**：`{ async: true }`
- 使用 `isQsTrue` 函数将字符串值转换为布尔值
- 两者任一为 true 即视为异步模式

### 2.3 rowAction 时加载 table

如果自动化是 rowAction 类型且请求中包含 `tableId`，则预加载表数据：

**代码位置**：[automation.ts#L359-L365](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/automation.ts#L359-L365)

```typescript
let table: Table | undefined
if (coreSdk.automations.isRowAction(automation) && testBody.row?.tableId) {
  table = await sdk.tables.getTable(testBody.row?.tableId)
  if (!table) {
    throw new HTTPError("Table not found", 404)
  }
}
```

加载的 table 会在后续 `externalTrigger` 调用时传入，供行操作触发器使用。

### 2.4 进度记录与 WebSocket 推送

test 端点定义了 `emitProgress` 函数来处理测试进度：

**代码位置**：[automation.ts#L367-L386](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/automation.ts#L367-L386)

```typescript
const emitProgress = (event: ProgressEventInput) => {
  const payload: AutomationTestProgressEvent = {
    ...event,
    automationId: automation._id!,
    appId,
  }
  recordTestProgress(appId, automation._id!, payload)
  builderSocket?.emitToRoom(
    ctx,
    ctx.appId,
    BuilderSocketEvent.AutomationTestProgress,
    payload,
    { includeOriginator: true }
  )
}
```

#### recordTestProgress

**文件**：[testProgress.ts#L41-L74](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/testProgress.ts#L41-L74)

功能：
- 使用内存中的 `Map` 存储测试进度状态，key 为 `${appId}:${automationId}`
- 每个状态包含 `events`（按 blockId 索引的事件列表）、`lastUpdated`、`createdAt`
- 如果事件有 `blockId`，按步骤存储；如果是完成/错误状态，存储在 `__automation__` 键下
- `complete` 状态会设置 `state.result` 和 `state.completed = true`
- `error` 状态会设置 `state.error`
- 有 5 分钟 TTL 和每分钟清理机制，防止内存泄漏

#### builderSocket.emitToRoom

**文件**：[websocket.ts#L280-L292](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/websockets/websocket.ts#L280-L292)

功能：
- 向指定房间（这里是 `ctx.appId`，即当前应用）的所有连接 socket 推送事件
- 事件类型为 `BuilderSocketEvent.AutomationTestProgress`
- `{ includeOriginator: true }` 表示包含发起请求的客户端（默认会通过 `apiSessionId` 排除发起者）
- 这样 builder 前端可以实时展示自动化测试的执行进度

### 2.5 withTestFlag 的作用

`withTestFlag` 用于在测试执行期间设置一个 Redis 标志，使得开发环境中的自动化能够被执行：

**文件**：[redis.ts#L105-L116](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/utilities/redis.ts#L105-L116)

```typescript
export async function withTestFlag<R>(id: string, fn: () => Promise<R>) {
  await flagClient.store(id, { testing: true }, AUTOMATION_TEST_FLAG_SECONDS)
  try {
    return await fn()
  } finally {
    await devAppClient.delete(id)
  }
}
```

**工作原理**：

1. 在执行测试函数前，在 Redis 的 FLAGS 数据库中存储一个标志 `{ testing: true }`，TTL 为 60 秒
2. 执行测试函数
3. 在 `finally` 块中删除标志（注意：代码有 bug，删除的是 `devAppClient` 而非 `flagClient`）

**为什么需要 test flag**：

在 [triggers.ts#L107-L113](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/triggers.ts#L107-L113) 中，行事件触发的自动化会检查 dev workspace 限制：

```typescript
if (
  !env.ALLOW_DEV_AUTOMATIONS &&
  isDevWorkspaceID(event.appId) &&
  !(await checkTestFlag(automation._id!))
) {
  continue
}
```

`checkTestFlag` 检查 Redis 中是否存在该 automation 的测试标志，如果存在则允许在 dev 环境中执行。这确保了：
- 正常情况下开发环境的自动化不会被行事件触发
- 只有通过 test 端点显式测试的自动化才会在开发环境中执行

### 2.6 runTest 函数

**代码位置**：[automation.ts#L388-L407](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/automation.ts#L388-L407)

```typescript
const runTest = async () => {
  const occurredAt = new Date().getTime()
  const result = await withTestFlag(automation._id!, async () => {
    await updateTestHistory(automation, { ...testBody, occurredAt })
    const input = prepareTestInput(testBody)
    const user = sdk.users.getUserContextBindings(ctx.user)
    return await triggers.externalTrigger(
      { ...automation, disabled: false },
      { ...{ ...input, ...(table ? { table } : {}) }, appId, user },
      { getResponses: true, onProgress: emitProgress }
    )
  })
  await events.automation.tested(automation)
  emitProgress({
    status: "complete",
    occurredAt: Date.now(),
    result,
  })
  return result
}
```

**执行流程**：

1. **包装 testFlag**：使用 `withTestFlag` 确保开发环境可以执行
2. **更新测试历史**：调用 `updateTestHistory` 记录测试时间和参数
3. **准备输入**：`prepareTestInput` 处理 `id`/`revision` 到 `row._id`/`row._rev` 的映射
4. **调用 externalTrigger**：
   - 强制 `disabled: false`（即使自动化被禁用也能测试）
   - 传入 `getResponses: true` 同步执行并获取结果
   - 传入 `onProgress: emitProgress` 回调，实时推送进度
5. **触发 tested 事件**：调用 `events.automation.tested(automation)` 发布测试事件
6. **发送完成进度**：emit `status: "complete"` 的进度事件，包含完整结果

#### events.automation.tested

**文件**：[automation.ts#L51-L59](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/events/publishers/automation.ts#L51-L59)

发布 `AUTOMATION_TESTED` 事件，包含属性：
- `appId`
- `automationId`
- `triggerId`
- `triggerType`

该事件用于分析/审计目的，记录自动化被测试的情况。

### 2.7 异步模式与错误处理

**代码位置**：[automation.ts#L409-L424](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/automation.ts#L409-L424)

```typescript
clearTestProgress(appId, automation._id!)

if (asyncFlag) {
  ctx.status = 202
  ctx.body = { message: "Automation test started" }
  runTest().catch(err => {
    emitProgress({
      status: "error",
      occurredAt: Date.now(),
      message: err?.message || "Automation test failed",
    })
  })
  return
}

ctx.body = await runTest()
```

**执行流程**：

1. **清除历史进度**：开始前先调用 `clearTestProgress` 清除之前的测试进度
2. **异步模式**（`asyncFlag === true`）：
   - 设置 HTTP 状态码 202（Accepted）
   - 返回消息 "Automation test started"
   - 异步调用 `runTest()`，不等待结果
   - 如果 `runTest` 失败，通过 `emitProgress` 发送 `status: "error"` 事件
   - 客户端需要通过 WebSocket 或 `/test/status` 端点轮询结果
3. **同步模式**（默认）：
   - 直接 `await runTest()`
   - 返回完整的测试结果

---

## 三、关键差异对比

| 对比项 | trigger 端点 | test 端点 |
|--------|-------------|----------|
| **使用场景** | 生产/已部署应用触发自动化 | 开发环境中测试自动化 |
| **应用类型** | 不限（但 dev 受 ALLOW_DEV_AUTOMATIONS 限制） | 仅限 DEV 应用（appInfoMiddleware 强制） |
| **权限中间件** | paramResource + recaptcha | appInfoMiddleware(DEV) + paramResource |
| **默认执行方式** | 异步（队列），COLLECT+SYNC 时同步 | 始终同步执行（getResponses: true） |
| **COLLECT 步骤** | 与 SYNC_AUTOMATIONS 特性配合使用 | 不特殊处理，作为普通步骤执行 |
| **SYNC_AUTOMATIONS** | 影响是否同步执行 | 不影响（始终同步） |
| **dev workspace 限制** | 通过 ALLOW_DEV_AUTOMATIONS 控制 | 无限制（专门用于 dev 环境） |
| **test flag** | 不使用 | 使用 withTestFlag 绕开 dev 限制 |
| **进度反馈** | 无 | 通过 recordTestProgress + WebSocket 实时推送 |
| **async 模式** | 不支持 | 支持 query/body 中的 async 参数 |
| **事件发布** | 无专门事件 | 发布 events.automation.tested 事件 |
| **disabled 自动化** | 不能触发 | 强制 disabled: false，可以测试 |
| **table 加载** | 不预加载 | rowAction 时预加载 table |
| **返回值** | 成功消息 或 COLLECT 输出 | 完整 AutomationResults 或 202 状态 |

---

## 四、调用链图示

### 4.1 trigger 同步模式调用链

```
POST /api/automations/:id/trigger
        │
        ▼
  controller.trigger
        │
        ├─► hasCollectStep? (checkForCollectStep)
        ├─► isSyncAutomationsEnabled?
        │
        ▼  (两者皆为 true)
  triggers.externalTrigger(automation, params, { getResponses: true })
        │
        ├─► checkTriggerFilters
        │
        ▼
  executeInThread ────► Orchestrator.execute()
        │                      │
        │                      ├─► executeSteps 逐个执行
        │                      └─► 返回 AutomationResults
        │
        ▼
  查找 COLLECT step.outputs
        │
        ▼
  返回 collectedValue
```

### 4.2 trigger 异步模式调用链

```
POST /api/automations/:id/trigger
        │
        ▼
  controller.trigger
        │
        ├─► 非 prod 且 !ALLOW_DEV_AUTOMATIONS → 400 错误
        │
        ▼
  triggers.externalTrigger(automation, params)
        │
        ├─► checkTriggerFilters
        │
        ▼
  automationQueue.add(data)  ────► Bull 队列异步执行
        │
        ▼
  返回 { message: "Automation ... has been triggered." }
```

### 4.3 test 同步模式调用链

```
POST /api/automations/:id/test
        │
        ▼
  controller.test
        │
        ├─► 解析 async flag (query + body)
        ├─► rowAction? 加载 table
        ├─► 定义 emitProgress (记录 + WebSocket)
        ├─► clearTestProgress
        │
        ▼  (async === false)
  withTestFlag(id, runTest)
        │
        ├─► Redis SET { testing: true }
        │
        ├─► updateTestHistory
        ├─► prepareTestInput
        │
        ├─► triggers.externalTrigger(
        │     { ...automation, disabled: false },
        │     params,
        │     { getResponses: true, onProgress: emitProgress }
        │   )
        │        │
        │        └─► executeInThread ───► Orchestrator
        │                  │              (每步调用 onProgress)
        │                  └─► 返回 AutomationResults
        │
        ├─► events.automation.tested(automation)
        ├─► emitProgress({ status: "complete", result })
        │
        └─► Redis DELETE flag
        │
        ▼
  返回完整 result
```

### 4.4 test 异步模式调用链

```
POST /api/automations/:id/test?async=true
        │
        ▼
  controller.test
        │
        ├─► asyncFlag === true
        │
        ├─► ctx.status = 202
        ├─► ctx.body = { message: "Automation test started" }
        │
        └─► runTest().catch(err => emitProgress(error))
              (后台异步执行，客户端通过 WebSocket 接收进度)
```

---

## 五、关键文件索引

| 文件 | 作用 |
|------|------|
| [packages/server/src/api/controllers/automation.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/automation.ts) | trigger 和 test 控制器实现 |
| [packages/server/src/api/routes/automation.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/automation.ts) | 路由定义 |
| [packages/server/src/automations/triggers.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/triggers.ts) | externalTrigger 实现 |
| [packages/server/src/threads/automation.ts](file:////Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/threads/automation.ts) | 自动化执行器 Orchestrator |
| [packages/server/src/automations/testProgress.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/testProgress.ts) | 测试进度内存存储 |
| [packages/server/src/utilities/redis.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/utilities/redis.ts) | withTestFlag / checkTestFlag |
| [packages/server/src/websockets/websocket.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/websockets/websocket.ts) | emitToRoom 实现 |
| [packages/backend-core/src/events/publishers/automation.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/events/publishers/automation.ts) | automation.tested 事件发布 |
| [packages/shared-core/src/sdk/documents/automations.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/shared-core/src/sdk/documents/automations.ts) | checkForCollectStep 等工具函数 |
