# Self-Host 单租户模式下 Cron 和 Email Automation 恢复与执行机制分析

## 1. 概述

在 self-host 单租户部署模式下，Redis 通常是临时性的（ephemeral），服务重启后 Bull 的 repeatable jobs 会丢失。为了确保 Cron 和 Email 类型的 automation 能够在服务重启后自动恢复执行，Budibase 设计了一套完整的重新水化（rehydrate）机制。

本文档详细分析了从服务启动初始化到定时任务恢复、再到邮件触发执行的完整流程。

---

## 2. 初始化入口：`init()` 函数

**文件位置**：[packages/server/src/automations/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/index.ts#L15-L29)

### 2.1 核心逻辑

```typescript
export async function init() {
  if (!automationsEnabled()) {
    return
  }
  const promise = automationQueue.process(async job => {
    await processEvent(job)
  })
  await rehydrateScheduledTriggers()
  await rebootTrigger()
  return promise
}
```

### 2.2 执行流程

1. **功能开关检查**：首先调用 `automationsEnabled()` 检查 automation 功能是否启用，未启用则直接返回。

2. **注册队列处理器**：通过 `automationQueue.process()` 注册队列处理器，所有进入队列的 automation job 都会调用 `processEvent(job)` 进行处理。这是一个持续运行的过程，promise 不会立即完成。

3. **重新水化定时触发器**：调用 `rehydrateScheduledTriggers()` 恢复所有因 Redis 重启而丢失的 repeatable jobs（Cron 和 Email 类型）。

4. **触发重启触发器**：调用 `rebootTrigger()` 执行所有配置为"服务重启时触发"的 automation。

### 2.3 关键设计说明

- 注册队列处理器和恢复定时任务是并行启动的，确保服务尽快开始处理任务
- `rehydrateScheduledTriggers()` 和 `rebootTrigger()` 都是异步执行的，确保初始化过程不会阻塞太久

---

## 3. 定时任务恢复：`rehydrateScheduledTriggers()`

**文件位置**：[packages/server/src/automations/rehydrate.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/rehydrate.ts#L38-L118)

### 3.1 设计背景

> 在许多 self-hosted 部署中，Redis 是临时性的。服务重启后，Bull 不再有任何 repeatable jobs，因此 CRON/Email automation 将不会再次触发，直到工作区重新发布（重新启用触发器）。在启动时，我们为所有生产环境工作区重新水化 repeatable jobs。

### 3.2 执行条件

函数首先检查运行环境，只有满足以下全部条件才会执行恢复逻辑：

- 不在子线程中运行（`!env.isInThread()`）
- 是 self-hosted 模式（`env.SELF_HOSTED`）
- 不是多租户模式（`!env.MULTI_TENANCY`）

### 3.3 核心流程

#### 步骤 1：获取现有 repeatable jobs

```typescript
const queue = automationQueue.getBullQueue()
const repeatableJobs = await queue.getRepeatableJobs()

const scheduled = new Set<string>()
for (const job of repeatableJobs) {
  if (!job.id) continue
  scheduled.add(scheduledKey(job.id, { cron: job.cron, every: job.every }))
}
```

通过 `scheduledKey` 函数将每个 job 转换为唯一标识字符串，存入 `Set` 中用于后续比较。

#### 步骤 2：`scheduledKey` 函数

**文件位置**：[packages/server/src/automations/rehydrate.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/rehydrate.ts#L13-L21)

```typescript
function scheduledKey(
  jobId: string,
  repeat: { cron?: string; every?: number }
) {
  if (repeat.cron) {
    return `${jobId}:cron:${repeat.cron}`
  }
  return `${jobId}:every:${repeat.every}`
}
```

该函数生成 job 的唯一调度键：
- Cron 类型：`{jobId}:cron:{cron表达式}`
- 固定间隔类型：`{jobId}:every:{间隔毫秒数}`

#### 步骤 3：遍历所有生产工作区

```typescript
const workspaceIds = await dbCore.getAllWorkspaces({
  dev: false,
  idsOnly: true,
})

for (const prodId of workspaceIds) {
  await context.doInWorkspaceContext(prodId, async () => {
    const automations = await getAllAutomations()
    // 处理每个 automation
  })
}
```

获取所有生产环境（非开发）工作区的 ID，逐个切换上下文并获取该工作区下的所有 automation。

#### 步骤 4：检查每个 automation 的触发器

对每个 automation 进行以下判断：
- 跳过没有触发器的 automation
- 跳过已禁用的 automation
- 跳过 reboot 类型的触发器（由 `rebootTrigger()` 单独处理）

#### 步骤 5：Cron 触发器恢复

```typescript
if (isCronTrigger(trigger)) {
  const inputs = trigger.inputs as CronTriggerInputs
  const cron = inputs.cron || ""
  const jobId = trigger.cronJobId?.toString()
  const key = jobId ? scheduledKey(jobId, { cron }) : null
  if (!key || !scheduled.has(key)) {
    promises.push(
      enableCronOrEmailTrigger(prodId, automation).then(result => {
        const scheduledJobId = result.automation.definition.trigger.cronJobId?.toString()
        if (scheduledJobId) {
          scheduled.add(scheduledKey(scheduledJobId, { cron }))
        }
      })
    )
  }
}
```

逻辑说明：
1. 从 automation 中提取 cron 表达式和 `cronJobId`
2. 使用 `scheduledKey` 生成调度键
3. 如果调度键不存在于 `scheduled` 集合中（即 Redis 中没有对应的 repeatable job），则调用 `enableCronOrEmailTrigger()` 重新创建
4. 创建成功后，将新的调度键加入集合，避免重复处理

#### 步骤 6：Email 触发器恢复

```typescript
else if (isEmailTrigger(trigger)) {
  const every = 30_000  // 30秒轮询一次
  const jobId = trigger.cronJobId?.toString()
  const key = jobId ? scheduledKey(jobId, { every }) : null
  if (!key || !scheduled.has(key)) {
    promises.push(
      enableCronOrEmailTrigger(prodId, automation).then(result => {
        const scheduledJobId = result.automation.definition.trigger.cronJobId?.toString()
        if (scheduledJobId) {
          scheduled.add(scheduledKey(scheduledJobId, { every }))
        }
      })
    )
  }
}
```

Email 触发器的恢复逻辑与 Cron 类似，区别在于：
- Email 触发器使用固定间隔 `every: 30_000`（每 30 秒轮询一次邮箱）
- 同样通过 `cronJobId` 来标识和匹配 job

---

## 4. 触发器启用：`enableCronOrEmailTrigger()`

**文件位置**：[packages/server/src/automations/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/utils.ts#L258-L365)

### 4.1 函数签名

```typescript
export async function enableCronOrEmailTrigger(
  appId: string,
  automation: Automation
): Promise<{
  enabled: boolean
  automation: Automation
  clearedRepeatableJobs: number
}>
```

### 4.2 前置检查

函数首先进行以下检查，不满足则直接返回：
- 触发器不存在
- automation 已禁用
- 是 reboot 触发器

### 4.3 Cron 触发器处理流程

#### 4.3.1 Cron 表达式校验

```typescript
if (isCronTrigger(trigger)) {
  const inputs = trigger.inputs as CronTriggerInputs
  const cronExp = inputs.cron || ""
  const validation = helpers.cron.validate(cronExp)
  if (!validation.valid) {
    throw new Error(
      `Invalid automation CRON "${cronExp}" - ${validation.err.join(", ")}`
    )
  }
  // ...
}
```

使用 `helpers.cron.validate()` 校验 cron 表达式的合法性，不合法则抛出错误。

#### 4.3.2 Legacy Job 迁移

```typescript
const existingJobId = trigger.cronJobId
if (existingJobId && isLegacyRepeatableJobId(existingJobId)) {
  const removedJobs = await removeLegacyRepeatableJob(
    existingJobId,
    appId,
    automation._id
  )
  clearedRepeatableJobs += removedJobs
}
```

**Legacy Job ID 判断**：
```typescript
function isLegacyRepeatableJobId(jobId?: string) {
  return !!jobId?.startsWith("repeat:")
}
```

如果现有的 `cronJobId` 以 `"repeat:"` 开头，说明是旧版本的 jobId，需要：
1. 调用 `removeLegacyRepeatableJob()` 从队列中移除旧的 repeatable job
2. 记录清除的 job 数量

**Legacy Job 移除逻辑**：
```typescript
async function removeLegacyRepeatableJob(
  legacyJobId: string,
  appId: string,
  automationId?: string
) {
  let count = 0
  try {
    const jobs = await automationQueue.getBullQueue().getRepeatableJobs()
    for (let job of jobs) {
      if (job.key.includes(legacyJobId)) {
        await automationQueue.getBullQueue().removeRepeatableByKey(job.key)
        count++
      }
    }
  } catch (err) {
    console.log("Failed to remove legacy repeatable job", { ... })
  }
  return count
}
```

#### 4.3.3 生成 Job ID

```typescript
const jobId =
  !existingJobId || isLegacyRepeatableJobId(existingJobId)
    ? `${appId}_cron_${utils.newid()}`
    : existingJobId
```

Job ID 生成规则：
- 如果没有现有 ID，或现有 ID 是 legacy 格式，则生成新 ID
- 新 ID 格式：`{appId}_cron_{随机ID}`
- 如果有有效现有 ID，则复用现有 ID

#### 4.3.4 添加 Repeatable Job

```typescript
await automationQueue.add(
  {
    automation,
    event: { appId },
  },
  { repeat: { cron: cronExp }, jobId }
)
```

通过 `automationQueue.add()` 添加一个 repeatable job 到 Bull 队列：
- Job 数据包含 automation 对象和 event（包含 appId）
- 配置 `repeat: { cron: cronExp }` 指定 cron 调度规则
- 使用生成的 `jobId` 作为 job 的唯一标识

#### 4.3.5 保存 CronJobId

```typescript
trigger.cronJobId = jobId

if (trigger.cronJobId !== existingJobId) {
  await dbCore.doWithDB(appId, async db => {
    const response = await db.put(automation)
    automation._id = response.id
    automation._rev = response.rev
  })
}
```

将 `cronJobId` 写入触发器配置中：
- 如果 ID 发生了变化（新创建或从 legacy 迁移），则将更新后的 automation 保存到数据库
- 如果 ID 未变，则无需保存

### 4.4 Email 触发器处理流程

Email 触发器的处理流程与 Cron 触发器基本一致，主要区别在于：

#### 4.4.1 输入验证

```typescript
if (isEmailTrigger(trigger)) {
  const inputs = trigger.inputs
  if (!inputs || !isValidEmailTriggerInputs(inputs)) {
    console.log("Automation email trigger inputs are not valid, disabling.", { ... })
    return { enabled: false, automation, clearedRepeatableJobs }
  }
  // ...
}
```

Email 触发器需要验证输入配置的有效性：
- 必须包含 host、port、username
- 根据 authType 不同，需要 password 或 datasourceId + authConfigId
- 验证不通过则不启用，直接返回

#### 4.4.2 Job ID 和调度配置

```typescript
const jobId =
  !existingJobId || isLegacyRepeatableJobId(existingJobId)
    ? `${appId}_email_${utils.newid()}`
    : existingJobId

await automationQueue.add(
  {
    automation,
    event: { appId },
  },
  { repeat: { every: 30_000 }, jobId }
)
```

区别：
- Job ID 前缀为 `{appId}_email_`
- 调度方式为固定间隔 `every: 30_000`（每 30 秒执行一次）

---

## 5. 邮件触发执行：`processEvent()` 中的 Email 处理

**文件位置**：[packages/server/src/automations/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/utils.ts#L87-L175)

### 5.1 `processEvent` 总体流程

`processEvent` 是所有 automation job 的统一入口，处理逻辑：

1. 从 job 中提取 workspaceId 和 automationId
2. 使用 `context.doInAutomationContext()` 设置自动化执行上下文
3. 根据触发器类型分发处理逻辑

### 5.2 Email 触发器处理

```typescript
if (isEmailTrigger(trigger)) {
  const { proceed, ...checkMailResult } = await checkMail(
    job.data.automation._id!,
    trigger.inputs
  )
  if (proceed === false) {
    return { skipped: true }
  }

  const { messages } = checkMailResult

  if (!messages) return { skipped: true }

  await Promise.all(
    messages?.map(async m => {
      const jobClone = cloneDeep(job)
      jobClone.data.event = { ...jobClone.data.event, ...m }
      const runFn = () => Runner.run(jobClone)
      await quotas.addAutomation(runFn, {
        automationId,
      })
    })
  )
  return {}
}
```

处理流程：
1. 调用 `checkMail()` 检查是否有新邮件
2. 如果 `proceed === false`，则跳过本次执行
3. 如果没有新邮件，也跳过
4. 对每封新邮件，深拷贝 job 对象，将邮件数据合并到 event 中
5. 通过 `Runner.run()` 逐个执行 automation
6. 使用 `quotas.addAutomation()` 进行配额控制

### 5.3 `checkMail()` 函数详解

**文件位置**：[packages/server/src/automations/email/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/email/index.ts#L28-L101)

#### 5.3.1 函数签名

```typescript
export const checkMail = async (
  automationId: string,
  imapInputs: EmailTriggerInputs
): Promise<CheckMailOutput> => {
```

返回值 `CheckMailOutput` 包含：
- `proceed`: boolean - 是否继续执行
- `reason`: string - 跳过原因（当 proceed 为 false 时）
- `messages`: array - 新邮件列表（当 proceed 为 true 时）

#### 5.3.2 核心执行流程

```typescript
let imapClient: Awaited<ReturnType<typeof getClient>> | null = null
const mailbox = imapInputs.mailbox || DEFAULT_MAILBOX
const stateKey = automationId

try {
  imapClient = await getClient(imapInputs)
  const lastSeenUid = await getLastSeenUid(stateKey, mailbox)
  const messages = await fetchMessages(imapClient, mailbox, lastSeenUid)
  // ...
} finally {
  if (imapClient) {
    await imapClient.logout().catch(console.log)
  }
}
```

步骤说明：
1. 创建 IMAP 客户端连接
2. 获取上次看到的最后一封邮件的 UID（`lastSeenUid`）
3. 获取 `lastSeenUid` 之后的所有邮件
4. 最后确保登出 IMAP 连接

#### 5.3.3 首次初始化跳过逻辑

```typescript
if (!lastSeenUid) {
  await setLastSeenUid(stateKey, mailbox, latestUid)
  return { proceed: false, reason: "init, now waiting" }
}
```

**关键设计**：当 `lastSeenUid` 不存在时（即首次运行），系统会：
1. 将当前最新邮件的 UID 设置为 `lastSeenUid`
2. 返回 `proceed: false`，跳过本次执行
3. 原因标记为 `"init, now waiting"`

**设计意图**：避免在首次启用邮件触发器时，触发所有历史邮件。只监听启用之后收到的新邮件。

#### 5.3.4 新邮件筛选

```typescript
const messagesWithUid = messages
  .map(message => ({
    message,
    uid: getMessageId(message),
  }))
  .filter(entry => entry.uid > 0)

messagesWithUid.sort((a, b) => a.uid - b.uid)
const latestUid = messagesWithUid[messagesWithUid.length - 1]?.uid

if (!latestUid) {
  return { proceed: false, reason: "no message id" }
}
```

首先为每封邮件提取 UID 并排序，找到最新的 UID。

```typescript
const unseenMessages = messagesWithUid.filter(
  ({ uid }) => uid > lastSeenUid
)

if (!unseenMessages.length) {
  return { proceed: false, reason: "no new mail" }
}
```

筛选出 UID 大于 `lastSeenUid` 的邮件（即新邮件）：
- 如果没有新邮件，返回 `proceed: false`，原因为 `"no new mail"`

#### 5.3.5 更新状态并返回结果

```typescript
await setLastSeenUid(stateKey, mailbox, latestUid)

const outputMessages = await Promise.all(
  unseenMessages.map(({ message }) => toOutputFields(message))
)

return {
  proceed: true,
  messages: outputMessages,
}
```

1. 将 `latestUid` 保存为新的 `lastSeenUid`
2. 将每封新邮件转换为输出字段格式
3. 返回 `proceed: true` 和新邮件列表

### 5.4 邮件状态存储

**文件位置**：[packages/server/src/automations/email/state.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/email/state.ts)

#### 5.4.1 `getLastSeenUid`

```typescript
export const getLastSeenUid = async (
  automationId: string,
  mailbox: string
): Promise<number | undefined> => {
  const db = context.getWorkspaceDB()
  const mailboxDoc = await db.tryGet<MailboxStateDoc>(
    getMailboxMetadataId(automationId, mailbox)
  )
  return mailboxDoc?.lastSeenUid
}
```

从工作区数据库中获取邮件状态文档，返回 `lastSeenUid`。

#### 5.4.2 `setLastSeenUid`

```typescript
export const setLastSeenUid = async (
  automationId: string,
  mailbox: string,
  lastSeenUid: number
) => {
  await updateEntityMetadata(
    MetadataType.AUTOMATION_EMAIL_STATE,
    getMailboxEntityId(automationId, mailbox),
    (metadata: MailboxStateDoc) => ({
      ...metadata,
      lastSeenUid,
    })
  )
}
```

将 `lastSeenUid` 保存到数据库的元数据中。元数据类型为 `AUTOMATION_EMAIL_STATE`。

#### 5.4.3 ID 生成规则

```typescript
const encodeMailbox = (mailbox: string) =>
  Buffer.from(mailbox, "utf8").toString("hex")

const getMailboxEntityId = (automationId: string, mailbox: string) =>
  `${automationId}:${encodeMailbox(mailbox)}`

const getMailboxMetadataId = (automationId: string, mailbox: string) =>
  generateMetadataID(
    MetadataType.AUTOMATION_EMAIL_STATE,
    getMailboxEntityId(automationId, mailbox)
  )
```

设计说明：
- 邮箱名称进行 hex 编码，避免包含 CouchDB 不允许的字符
- 实体 ID 格式：`{automationId}:{hex编码的邮箱名}`
- 最终元数据 ID 通过 `generateMetadataID` 生成

---

## 6. Reboot 触发器

**文件位置**：[packages/server/src/automations/triggers.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/triggers.ts#L287-L318)

### 6.1 `rebootTrigger()` 函数

```typescript
export async function rebootTrigger() {
  if (env.isInThread() || !env.SELF_HOSTED || env.MULTI_TENANCY) {
    return
  }
  const workspaceIds = await dbCore.getAllWorkspaces({
    dev: false,
    idsOnly: true,
  })
  for (let prodId of workspaceIds) {
    await context.doInWorkspaceContext(prodId, async () => {
      let automations = await getAllAutomations()
      let rebootEvents = []
      for (let automation of automations) {
        if (utils.isRebootTrigger(automation)) {
          const job = {
            automation,
            event: {
              appId: prodId,
              timestamp: Date.now(),
            },
          }
          rebootEvents.push(automationQueue.add(job, JOB_OPTS))
        }
      }
      await Promise.all(rebootEvents)
    })
  }
}
```

### 6.2 执行逻辑

1. **环境检查**：与 `rehydrateScheduledTriggers` 相同的运行条件检查
2. **遍历生产工作区**：获取所有生产工作区
3. **查找 reboot 触发器**：遍历每个工作区的 automation，找到 reboot 类型的触发器
4. **立即触发**：将 reboot automation 加入队列立即执行，附带当前时间戳

### 6.3 Reboot 触发器判断

**文件位置**：[packages/server/src/automations/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/utils.ts#L243-L251)

```typescript
export function isRebootTrigger(auto: Automation) {
  const trigger = auto ? auto.definition.trigger : null
  const isCron = trigger && isCronTrigger(trigger)
  if (!isCron) {
    return false
  }
  const inputs = trigger?.inputs as CronTriggerInputs
  return inputs.cron === REBOOT_CRON
}
```

Reboot 触发器本质上是一种特殊的 Cron 触发器，其 cron 表达式等于特殊常量 `REBOOT_CRON`。

---

## 7. 整体流程图

```
服务启动
    │
    ▼
init() 被调用
    │
    ├─► automationsEnabled()?
    │      │
    │      ├─ 否 ──► 返回
    │      │
    │      └─ 是 ──┐
    │               │
    ├───────────────► 注册 automationQueue.process(processEvent)
    │               │
    ├───────────────► rehydrateScheduledTriggers()
    │               │
    │               ├─ 环境检查（主线程 + self-host + 单租户）
    │               ├─ 获取所有现有 repeatable jobs（scheduled Set）
    │               ├─ 遍历所有 production workspace
    │               │   └─ 遍历所有 automations
    │               │       ├─ Cron trigger
    │               │       │   └─ scheduledKey 不存在？
    │               │       │       └─ 是 ──► enableCronOrEmailTrigger()
    │               │       └─ Email trigger
    │               │           └─ scheduledKey 不存在？
    │               │               └─ 是 ──► enableCronOrEmailTrigger()
    │               │
    │               └─ 保存更新后的 automations
    │
    └───────────────► rebootTrigger()
                    │
                    ├─ 环境检查（主线程 + self-host + 单租户）
                    ├─ 遍历所有 production workspace
                    │   └─ 遍历所有 automations
                    │       └─ isRebootTrigger?
                    │           └─ 是 ──► 立即加入队列执行
                    │
                    └─ 完成
```

### Email 触发执行流程

```
Bull 队列触发（每 30 秒）
    │
    ▼
processEvent(job)
    │
    ▼
isEmailTrigger(trigger)?
    │
    ├─ 否 ──► 其他处理逻辑
    │
    └─ 是 ──► checkMail(automationId, inputs)
                │
                ├─ 连接 IMAP 服务器
                ├─ 获取 lastSeenUid
                │   │
                │   ├─ lastSeenUid 不存在？
                │   │   ├─ 是 ──► 设置为最新邮件 UID，返回 proceed=false（首次初始化）
                │   │   └─ 否 ──► 继续
                │   │
                │   └─ 拉取 lastSeenUid 之后的邮件
                │       │
                │       ├─ 没有新邮件？
                │       │   └─ 是 ──► 返回 proceed=false
                │       │
                │       └─ 更新 lastSeenUid
                │
                └─ 返回 proceed=true + 新邮件列表
                    │
                    ▼
            对每封新邮件：
            - cloneDeep(job)
            - 合并邮件数据到 event
            - Runner.run(jobClone)
```

---

## 8. 关键设计要点总结

### 8.1 幂等性设计

- 通过 `scheduledKey` 比较机制，确保重复调用 `rehydrateScheduledTriggers` 不会重复创建 job
- `cronJobId` 持久化存储在 automation 文档中，服务重启后可以匹配现有 job
- Email 的 `lastSeenUid` 持久化存储，避免重复处理邮件

### 8.2 向后兼容

- 支持从 legacy job ID（`"repeat:"` 前缀）迁移
- 迁移时会清理旧的 repeatable job 并生成新的 job ID

### 8.3 安全性考虑

- Email 触发器首次初始化时只记录 `lastSeenUid`，不触发历史邮件，避免邮件风暴
- 所有操作都有 try/catch 保护，单个 automation 失败不影响其他

### 8.4 性能优化

- 使用 `Set` 进行调度键的存在性检查（O(1) 复杂度）
- 批量处理恢复任务（`Promise.all`）
- 邮件状态存储在数据库元数据中，不依赖 Redis

### 8.5 适用范围限制

- 仅在 self-host 单租户模式下启用恢复机制
- 多租户环境下有不同的管理方式
- 仅在主线程中执行，子线程不重复执行

---

## 9. 相关文件索引

| 文件 | 说明 |
|------|------|
| [packages/server/src/automations/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/index.ts) | 初始化入口，init 函数 |
| [packages/server/src/automations/rehydrate.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/rehydrate.ts) | 定时任务重新水化逻辑 |
| [packages/server/src/automations/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/utils.ts) | 工具函数：enableCronOrEmailTrigger、processEvent 等 |
| [packages/server/src/automations/triggers.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/triggers.ts) | 触发器相关：rebootTrigger、externalTrigger 等 |
| [packages/server/src/automations/email/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/email/index.ts) | 邮件检查：checkMail 函数 |
| [packages/server/src/automations/email/state.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/email/state.ts) | 邮件状态管理：lastSeenUid 存取 |
| [packages/server/src/automations/bullboard.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/bullboard.ts) | Bull 队列和 Bull Board 管理 |
