# 内部表行更新触发行更新 Automation 完整链路追踪

本文档追踪一次内部表行更新如何触发 "行更新" Automation 的完整调用链路，从 `rowController.patch` 成功后的事件发射开始，一直到 `automationQueue.process` 调用 `processEvent` 为止。

---

## 目录

1. [起点：rowController.patch 成功后发射事件](#1-起点rowcontrollerpatch-成功后发射事件)
2. [事件发射：BudibaseEmitter.emitRow → rowEmission](#2-事件发射budibaseemitteremitrow--rowemission)
3. [事件监听：triggers.ts 中的 emitter.on(ROW_UPDATE)](#3-事件监听triggersts-中的-emitteronrow_update)
4. [queueRelevantRowAutomations：核心过滤逻辑](#4-queuerelevantrowautomations核心过滤逻辑)
   - 4.1 [参数校验与 tableId 校验](#41-参数校验与-tableid-校验)
   - 4.2 [进入 Workspace Context](#42-进入-workspace-context)
   - 4.3 [加载所有 Automation](#43-加载所有-automation)
   - 4.4 [按 event / tableId / disabled 过滤](#44-按-event--tableid--disabled-过滤)
   - 4.5 [DevWorkspace 与 checkTestFlag 阻止机制](#45-devworkspace-与-checktestflag-阻止机制)
   - 4.6 [checkTriggerFilters：行级过滤](#46-checktriggerfilters行级过滤)
   - 4.7 [quotas.addAction 包裹 automationQueue.add](#47-quotasaddaction-包裹-automationqueueadd)
5. [队列消费：automationQueue.process → processEvent](#5-队列消费automationqueueprocess--processevent)
   - 5.1 [Server 启动时初始化消费端](#51-server-启动时初始化消费端)
   - 5.2 [BudibaseQueue.process 包装层](#52-budibasequeueprocess-包装层)
   - 5.3 [processEvent：真正执行 Automation](#53-processevent真正执行-automation)

---

## 1. 起点：rowController.patch 成功后发射事件

行更新的入口位于 [row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts) 的 `_patch` 函数（第 60-104 行）。

### 关键代码（第 86-93 行）：

```typescript
ctx.eventEmitter?.emitRow({
  eventName: EventType.ROW_UPDATE,
  appId,
  row,
  table,
  oldRow,
  user: sdk.users.getUserContextBindings(ctx.user),
})
```

### 说明：

- `_patch` 函数内部通过 `pickApi(tableId)` 选择内部表或外部表 API 执行实际的 patch 操作
- 成功后通过 `ctx.eventEmitter.emitRow()` 发射事件
- 事件包含的字段：
  - `eventName`: 事件类型，此处为 `EventType.ROW_UPDATE`
  - `appId`: 应用 ID（workspace ID）
  - `row`: 更新后的行数据
  - `table`: 表信息（可选）
  - `oldRow`: 更新前的行数据（仅 ROW_UPDATE 有）
  - `user`: 用户上下文绑定信息

> 注意：`ctx.eventEmitter` 是在 Koa 启动时通过 `app.context.eventEmitter = eventEmitter` 挂载的全局单例。

---

## 2. 事件发射：BudibaseEmitter.emitRow → rowEmission

### 2.1 BudibaseEmitter 类

定义在 [BudibaseEmitter.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/events/BudibaseEmitter.ts) 第 21-42 行。

`emitRow` 方法（第 22-38 行）接收结构化参数，然后委托给 `rowEmission` 工具函数。

### 2.2 rowEmission 工具函数

定义在 [events/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/events/utils.ts) 第 31-59 行。

```typescript
export function rowEmission({ emitter, eventName, appId, row, table, metadata, oldRow, user }: BBEventOpts) {
  let event: BBEvent = {
    row,
    oldRow,
    appId,
    tableId: row?.tableId,
    user,
  }
  if (table) {
    event.table = table
  }
  event.id = row?._id
  if (row?._rev) {
    event.revision = row._rev
  }
  if (metadata) {
    event.metadata = metadata
  }
  emitter.emit(eventName, event)
}
```

### 说明：

- 将结构化参数重新组装为扁平的 `BBEvent` 对象
- `tableId` 从 `row.tableId` 中提取
- 最终调用 Node.js 原生 `EventEmitter.emit(eventName, event)` 发射事件
- 事件名称为字符串形式的 `EventType.ROW_UPDATE`

### 2.3 全局 emitter 单例

在 [events/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/events/index.ts) 中创建：

```typescript
import BudibaseEmitter from "./BudibaseEmitter"
const emitter = new BudibaseEmitter()
export default emitter
```

---

## 3. 事件监听：triggers.ts 中的 emitter.on(ROW_UPDATE)

在 [automations/triggers.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/triggers.ts) 第 154-160 行注册了 `ROW_UPDATE` 事件的监听器。

### 关键代码：

```typescript
emitter.on(AutomationEventType.ROW_UPDATE, async function (event) {
  /* istanbul ignore next */
  if (!event || !event.row || !event.row.tableId) {
    return
  }
  await queueRowAutomations(event, AutomationEventType.ROW_UPDATE)
})
```

### 校验逻辑：

1. **event 存在性校验**：`!event || !event.row || !event.row.tableId`，若任一不满足则直接 return，不做任何处理
2. 通过校验后，调用 `queueRowAutomations(event, AutomationEventType.ROW_UPDATE)`

### queueRowAutomations 包装函数（第 132-141 行）：

```typescript
async function queueRowAutomations(
  event: AutomationRowEvent,
  type: AutomationEventType
) {
  try {
    await queueRelevantRowAutomations(event, type)
  } catch (err: any) {
    logging.logWarn("Unable to process row event", err)
  }
}
```

仅是一层 try-catch 包装，捕获异常并记录警告日志，防止单个事件异常影响整个事件循环。

---

## 4. queueRelevantRowAutomations：核心过滤逻辑

`queueRelevantRowAutomations` 是整个链路的核心函数，位于 [triggers.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/triggers.ts) 第 71-130 行。

### 4.1 参数校验与 tableId 校验

```typescript
const tableId = event.row.tableId
if (event.appId == null) {
  throw `No appId specified for ${eventType} - check event emitters.`
}

// make sure table exists and is valid before proceeding
if (!tableId || !(await doesTableExist(tableId))) {
  return
}
```

**校验步骤：**
1. 检查 `event.appId` 是否存在，不存在则抛出异常
2. 检查 `tableId` 是否存在且对应表真实存在
   - `doesTableExist` 定义在 [sdk/workspace/tables/getters.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/tables/getters.ts) 第 238-250 行
   - 内部调用 `getTable(tableId)`，成功返回 true，异常返回 false
   - 若表不存在则直接 return，不继续处理

### 4.2 进入 Workspace Context

```typescript
await context.doInWorkspaceContext(event.appId, async () => {
  // ... 所有后续逻辑都在 workspace context 内
})
```

`context.doInWorkspaceContext` 定义在 [backend-core/src/context/mainContext.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/context/mainContext.ts) 第 207-230 行。

**作用：**
- 设置 `appId`（即 workspaceId）到异步上下文
- 推导并设置 `tenantId`
- 后续所有数据库操作（如 `getWorkspaceDB()`）都会基于此上下文获取对应的数据库
- 使用 `newContext(updates, task)` 实现，基于 `AsyncLocalStorage`

### 4.3 加载所有 Automation

```typescript
async function getAllAutomations() {
  const db = context.getWorkspaceDB()
  let automations = await db.allDocs<Automation>(
    getAutomationParams(null, { include_docs: true })
  )
  return automations.rows.map(row => row.doc!)
}
```

**说明：**
- 通过 `context.getWorkspaceDB()` 获取当前 workspace 的数据库
- 使用 `getAutomationParams` 生成查询参数（基于 `DocumentType.AUTOMATION` 前缀）
- 调用 CouchDB 的 `allDocs` + `include_docs: true` 获取所有 automation 文档
- `getAutomationParams` 定义在 [server/src/db/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/db/utils.ts) 第 81-83 行

### 4.4 按 event / tableId / disabled 过滤

```typescript
automations = automations.filter(automation => {
  const trigger = automation.definition.trigger
  const triggerInputs = trigger?.inputs as RowTriggerInputs

  return (
    trigger &&
    trigger.event === eventType &&
    !automation.disabled &&
    triggerInputs?.tableId === event.row.tableId
  )
})
```

**四重过滤条件：**

| 条件 | 说明 |
|------|------|
| `trigger` | automation 必须定义了触发器 |
| `trigger.event === eventType` | 触发器事件类型必须匹配（ROW_UPDATE） |
| `!automation.disabled` | automation 必须处于启用状态 |
| `triggerInputs?.tableId === event.row.tableId` | 触发器配置的表 ID 必须与事件的表 ID 一致 |

只有同时满足四个条件的 automation 才会进入后续处理。

### 4.5 DevWorkspace 与 checkTestFlag 阻止机制

```typescript
// don't queue events which are for dev workspaces, only way to test automations is
// running tests on them, in production the test flag will never
// be checked due to lazy evaluation (first always false)
if (
  !env.ALLOW_DEV_AUTOMATIONS &&
  isDevWorkspaceID(event.appId) &&
  !(await checkTestFlag(automation._id!))
) {
  continue
}
```

**三重阻止逻辑（短路求值，从左到右）：**

1. **`!env.ALLOW_DEV_AUTOMATIONS`**：环境变量是否禁止开发环境 automation。若为 `false`（即允许），则直接跳过整个判断，不阻止
2. **`isDevWorkspaceID(event.appId)`**：是否为开发 workspace。通过 `workspaceId.startsWith(WORKSPACE_DEV_PREFIX)` 判断，定义在 [backend-core/src/docIds/conversions.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/docIds/conversions.ts) 第 6-11 行
3. **`!(await checkTestFlag(automation._id!))`**：该 automation 是否未处于测试状态

**`checkTestFlag` 实现**（[utilities/redis.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/utilities/redis.ts) 第 100-103 行）：

```typescript
export async function checkTestFlag(id: string) {
  const flag = await flagClient?.get(id)
  return !!(flag && flag.testing)
}
```

- 从 Redis 的 FLAGS 数据库中以 automation ID 为 key 取值
- 检查 `flag.testing` 是否为 true
- 测试标志的有效期为 60 秒（`AUTOMATION_TEST_FLAG_SECONDS`）
- 由 `withTestFlag` 函数在测试 automation 时设置

**设计意图：**
- 生产环境中（非 dev workspace），第一个条件 `!env.ALLOW_DEV_AUTOMATIONS` 可能为 true 也可能为 false，但 `isDevWorkspaceID` 一定为 false，所以整个判断为 false，不会阻止
- 开发环境中，默认禁止 automation 自动触发，只能通过测试功能（设置 test flag）来触发
- 惰性求值优化：生产环境下第一个条件就短路，不会执行 Redis 查询

### 4.6 checkTriggerFilters：行级过滤

```typescript
const shouldTrigger = await checkTriggerFilters(automation, {
  row: event.row,
  oldRow: event.oldRow,
})
```

`checkTriggerFilters` 定义在 [triggers.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/triggers.ts) 第 320-341 行。

```typescript
async function checkTriggerFilters(
  automation: Automation,
  event: { row: Row; oldRow: Row }
): Promise<boolean> {
  const trigger = automation.definition.trigger
  const triggerInputs = trigger.inputs as RowFilterInputs
  const filters = triggerInputs?.filters
  const tableId = triggerInputs?.tableId

  if (!filters) {
    return true
  }

  if (
    trigger.stepId === automations.triggers.definitions.ROW_UPDATED.stepId ||
    trigger.stepId === automations.triggers.definitions.ROW_SAVED.stepId
  ) {
    const newRow = await automationUtils.cleanUpRow(tableId, event.row)
    return rowPassesFilters(newRow, filters)
  }
  return true
}
```

**过滤逻辑：**

1. **无过滤器直接通过**：若 `!filters`，直接返回 `true`
2. **触发器类型判断**：仅 `ROW_UPDATED` 和 `ROW_SAVED` 类型的触发器支持行过滤，其他类型直接返回 `true`
3. **cleanUpRow 清洗行数据**
4. **rowPassesFilters 判断是否匹配**

#### 4.6.1 automationUtils.cleanUpRow

定义在 [automations/automationUtils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/automationUtils.ts) 第 106-109 行。

```typescript
export async function cleanUpRow(tableId: string, row: Row) {
  let table = await sdk.tables.getTable(tableId)
  return cleanInputValues(row, table.schema)
}
```

- 获取表结构 schema
- 调用 `cleanInputValues` 将行中的字符串值转换为正确的原始类型（数字、布尔值、关系数组等）
- 这是因为自动化中的值通常以字符串形式（模板字符串）传入，需要根据 schema 类型转换

#### 4.6.2 rowPassesFilters + dataFilters.runQuery

```typescript
function rowPassesFilters(row: Row, filters: SearchFilters) {
  const filteredRows = dataFilters.runQuery([row], filters)
  return filteredRows.length > 0
}
```

`dataFilters.runQuery` 定义在 [shared-core/src/filters.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/shared-core/src/filters.ts) 第 588 行起。

- 将单行包装为数组 `[row]` 传入查询函数
- `runQuery` 根据 `SearchFilters` 对数据进行过滤（支持等于、不等于、大于、小于、包含、逻辑运算符 AND/OR/NOT 等）
- 若过滤结果数组长度 > 0，说明行匹配过滤条件，返回 `true`

### 4.7 quotas.addAction 包裹 automationQueue.add

```typescript
if (shouldTrigger) {
  try {
    await quotas.addAction(ActionType.AUTOMATION_STEP, () =>
      automationQueue.add({ automation, event }, JOB_OPTS)
    )
  } catch (e) {
    logging.logAlert("Failed to queue automation", e)
  }
}
```

**关键要素：**

1. **`quotas.addAction`**：来自 `@budibase/pro` 包，用于配额管理
   - 第一个参数是动作类型 `ActionType.AUTOMATION_STEP`
   - 第二个参数是要执行的工作函数
   - 在配额内允许执行，超出配额则拒绝

2. **`automationQueue.add`**：将 automation 任务加入队列
   - 数据：`{ automation, event }`，包含完整的 automation 定义和事件数据
   - 选项：`JOB_OPTS = { removeOnComplete: true, removeOnFail: true }`，任务完成或失败后自动移除

3. **异常捕获**：队列添加失败时记录告警日志，但不抛出异常

---

## 5. 队列消费：automationQueue.process → processEvent

### 5.1 Server 启动时初始化消费端

在 [startup/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/startup/index.ts) 第 184-186 行：

```typescript
if (automationsEnabled()) {
  queuePromises.push(automations.init())
  // ...
}
```

`automations.init()` 定义在 [automations/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/index.ts) 第 15-29 行：

```typescript
export async function init() {
  if (!automationsEnabled()) {
    return
  }
  // this promise will not complete
  const promise = automationQueue.process(async job => {
    await processEvent(job)
  })
  // on init we need to rehydrate any scheduled triggers that may have been lost
  // due to reddis being ephemeral in some self hosted deployments.
  await rehydrateScheduledTriggers()
  // on init we need to trigger any reboot automations
  await rebootTrigger()
  return promise
}
```

**启动时做三件事：**
1. 调用 `automationQueue.process()` 注册消费处理器（常驻，promise 不会完成）
2. `rehydrateScheduledTriggers()`：重新注册定时触发器（防止 Redis 数据丢失）
3. `rebootTrigger()`：触发所有设置了"重启触发"的 automation

### 5.2 BudibaseQueue.process 包装层

`automationQueue` 是 `BudibaseQueue<AutomationData>` 的实例，定义在 [automations/bullboard.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/bullboard.ts) 第 12-26 行。

`BudibaseQueue` 类的 `process` 方法定义在 [backend-core/src/queue/queue.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/queue/queue.ts) 第 151-215 行。

**主要职责：**

1. **参数处理**：支持可选的并发数 `concurrency` 参数
2. **Datadog tracing 包装**：为每个任务添加 `queue.process` span，附带 queue 名称、job 信息、数据大小等标签
3. **metrics 统计**：成功/失败计数、耗时分布
4. **父 span 关联**：通过 `_parentSpanContext` 将入队时的 trace 与消费时的 trace 串联
5. **委托底层 Bull 队列**：最终调用 `this.queue.process(wrappedCb)`

### 5.3 processEvent：真正执行 Automation

`processEvent` 定义在 [automations/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/utils.ts) 第 87 行起。

**核心执行流程：**

```typescript
export async function processEvent(job: AutomationJob) {
  return tracer.trace("processEvent", async span => {
    const workspaceId = job.data.event.appId!
    const automationId = job.data.automation._id!
    // ... span tags ...

    const trigger = job.data.automation
      ? job.data.automation.definition.trigger
      : null

    const task = async () => {
      try {
        return await tracer.trace("task", async () => {
          // 邮件触发器特殊处理：拉取邮件后为每封邮件触发一次
          if (isEmailTrigger(trigger)) {
            // ... checkMail + 循环触发 ...
          }
          // Cron 触发器补充 timestamp
          const isCron = trigger && isCronTrigger(trigger)
          if (isCron && !job.data.event.timestamp) {
            job.data.event.timestamp = Date.now()
          }
          
          console.log("automation running", ...loggingArgs(job))
          const runFn = () => Runner.run(job)
          const result = await quotas.addAutomation(runFn, { automationId })
          console.log("automation completed", ...loggingArgs(job))
          return result
        })
      } catch (err) {
        // ... 错误处理 ...
        return { err }
      }
    }
    // ... 执行 task ...
  })
}
```

**关键要点：**

1. **Datadog tracing**：全程追踪，携带 appId、automationId、job 信息等标签
2. **邮件触发器特殊逻辑**：拉取邮件后为每封邮件单独触发一次 automation
3. **Cron 触发器补充时间戳**：运行时补充 `timestamp`
4. **`Runner.run(job)`**：真正的 automation 执行引擎（执行步骤链）
5. **`quotas.addAutomation`**：配额检查，确保在自动化使用配额内
6. **错误处理**：捕获所有异常，防止队列阻塞

---

## 完整链路总结图

```
rowController.patch 成功
        │
        ▼
ctx.eventEmitter.emitRow(EventType.ROW_UPDATE)
        │  [BudibaseEmitter.ts]
        ▼
rowEmission() 组装事件对象
        │  [events/utils.ts]
        ▼
EventEmitter.emit("row:update", event)
        │
        ▼
emitter.on(ROW_UPDATE) 监听器触发
        │  [triggers.ts L154-160]
        ▼
queueRowAutomations()  try-catch 包装
        │
        ▼
queueRelevantRowAutomations()  核心过滤
        │
        ├─► 参数校验：appId、tableId、表是否存在
        │
        ├─► context.doInWorkspaceContext(appId)
        │
        ├─► getAllAutomations()  加载所有 automation
        │
        ├─► filter 四连过滤：
        │     trigger存在 && event匹配 && !disabled && tableId匹配
        │
        ├─► DevWorkspace + checkTestFlag 阻止判断
        │     （dev环境默认阻止，除非有test flag）
        │
        ├─► checkTriggerFilters()  行级过滤
        │     └─► cleanUpRow() 类型转换
        │     └─► dataFilters.runQuery() 过滤匹配
        │
        └─► quotas.addAction(AUTOMATION_STEP, () =>
              automationQueue.add(job)
            )
                     │
                     ▼
              Bull 队列（Redis 存储）
                     │
                     ▼
automationQueue.process(callback)  消费端
        │  [automations/index.ts init]
        │  [BudibaseQueue.process 包装层]
        ▼
processEvent(job)
        │
        ├─► 邮件触发器特殊处理
        ├─► Cron 触发器补 timestamp
        └─► Runner.run(job)  执行自动化步骤
             └─► quotas.addAutomation  配额检查
```

---

## 关键文件索引

| 文件路径 | 作用 |
|----------|------|
| [row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts) | 行控制器，patch 成功后发射事件 |
| [BudibaseEmitter.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/events/BudibaseEmitter.ts) | 事件发射器类 |
| [events/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/events/utils.ts) | 事件对象组装工具 |
| [triggers.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/triggers.ts) | 事件监听与核心过滤逻辑 |
| [automationUtils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/automationUtils.ts) | 行数据清洗等工具函数 |
| [bullboard.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/bullboard.ts) | automation 队列定义 |
| [automations/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/index.ts) | automation 模块初始化，注册消费者 |
| [automations/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/utils.ts) | processEvent 队列消费处理 |
| [utilities/redis.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/utilities/redis.ts) | checkTestFlag 测试标志检查 |
| [startup/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/startup/index.ts) | Server 启动入口，初始化 automation |
| [queue/queue.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/queue/queue.ts) | BudibaseQueue 队列包装类 |
| [context/mainContext.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/context/mainContext.ts) | Workspace 上下文管理 |
| [filters.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/shared-core/src/filters.ts) | 数据过滤查询 runQuery |
