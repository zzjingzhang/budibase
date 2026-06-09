# Workspace Migration 访问未完成迁移应用的完整流程分析

## 概述

当用户访问一个尚未完成 workspace migration 的应用 API 时，请求会经过一系列中间件检测、异步队列处理、前端响应等多个环节。本文档从服务端中间件到前端客户端，逐层解析完整的调用链路和行为逻辑。

---

## 一、服务端 API 中间件调用顺序

### 1.1 中间件挂载顺序

在 [index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/index.ts) 中，`workspaceMigrations` 中间件的挂载位置如下：

```
errorHandling → featureFlagCookie → compress → redirect → assetRoutes
→ auth.buildAuthMiddleware
→ auth.buildTenancyMiddleware
→ middleware.activeTenant()
→ pro.licensing()
→ currentWorkspace
→ middleware.csp            (若 DISABLE_CONTENT_SECURITY_POLICY 为 false)
→ auth.auditLog
→ migrations (workspaceMigrations)   ← 我们关注的位置
→ cleanup
→ mainRoutes / publicRoutes / staticRoutes
```

### 1.2 关键设计考量

`workspaceMigrations` 被放置在以下中间件**之后**：

- **auth**：需要认证信息来确定用户身份和权限
- **tenancy / activeTenant**：需要租户上下文，因为 migration 是按租户隔离的
- **licensing**：某些迁移功能可能与许可证相关
- **currentWorkspace**：需要当前 workspace 信息（`ctx.appId` 在此中间件中设置）
- **CSP**：内容安全策略需要尽早设置

它被放置在 `auth.auditLog` 之后、业务路由之前，确保：
- 审计日志能记录迁移状态
- 业务路由执行前已完成迁移检查或已将任务入队

---

## 二、workspaceMigrations 中间件入口

### 2.1 跳过条件

[workspaceMigrations.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/middleware/workspaceMigrations.ts) 的 `workspaceMigrations` 中间件在以下两种情况下直接跳过：

1. **`DISABLE_WORKSPACE_MIGRATIONS` 环境变量为 true**
   - 用途：测试环境或需要完全禁用迁移的场景
   - 直接 `return next()`，不执行任何迁移检查

2. **上下文中没有 `appId`**
   - 原因：workspace migration 是针对具体应用（workspace）的
   - 无 appId 的请求（如全局配置、健康检查等）不需要迁移检查
   - 直接 `return next()`

### 2.2 进入迁移检查

若以上两个条件都不满足，则调用 `checkMissingMigrations(ctx, next, appId)` 进入核心迁移检查逻辑。

---

## 三、checkMissingMigrations 核心逻辑

### 3.1 计算 latest enabled migration

[index.ts#L31-L44](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/index.ts#L31-L44) 中的 `getLatestEnabledMigrationId()` 函数：

- 遍历 `MIGRATIONS` 数组（按 id 排序，即时间顺序）
- **遇到第一个 `disabled: true` 的迁移时停止**——这意味着被禁用迁移之后的所有迁移也被视为禁用
- 返回最后一个启用的迁移 id
- 如果没有任何启用的迁移，返回 `undefined`

> **设计意图**：迁移是顺序执行的，如果某个迁移被禁用，其后续迁移也不能执行，因为它们可能依赖于前序迁移的数据结构变更。

### 3.2 确认 workspace metadata 存在

[index.ts#L62-L68](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/index.ts#L62-L68)：

```typescript
const workspaceExists = await context.doInWorkspaceContext(
  workspaceId,
  async () => !!(await sdk.workspaces.metadata.tryGet())
)
if (!workspaceExists) {
  return next()
}
```

- 使用 `tryGet()` 方法尝试获取 workspace metadata 文档
- 如果 metadata 不存在，说明 workspace 可能尚未初始化或已被删除
- 此时跳过迁移检查，直接继续后续中间件

### 3.3 检测 isWorkspaceFullyMigrated

[index.ts#L120-L130](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/index.ts#L120-L130)：

```typescript
export const isWorkspaceFullyMigrated = async (workspaceId: string) => {
  const latestMigration = getLatestEnabledMigrationId()
  if (!latestMigration) {
    return true
  }
  const latestMigrationApplied = await getWorkspaceMigrationVerions(workspaceId)
  return (
    getMigrationIndex(latestMigrationApplied) >=
    getMigrationIndex(latestMigration)
  )
}
```

- 从 workspace 的 migration metadata 文档中获取当前已应用的迁移版本
- 比较已应用迁移在 `MIGRATIONS` 数组中的索引与最新启用迁移的索引
- 如果已应用索引 >= 最新索引，则认为完全迁移

`getWorkspaceMigrationVerions` 的实现见 [workspaceMigrationMetadata.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/workspaceMigrationMetadata.ts)：
- 优先从缓存读取（cache key: `appmigrations_{VERSION}_{workspaceId}`）
- 缓存未命中则从数据库读取 `DesignDocuments.MIGRATIONS` 文档
- 如果文档不存在（404），返回空字符串（表示从未执行过迁移）
- 读取到的版本会缓存 1 天

### 3.4 未完成迁移时的处理

如果 workspace 未完全迁移，执行以下操作：

#### 3.4.1 添加迁移任务到队列

[index.ts#L71-L79](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/index.ts#L71-L79)：

```typescript
const queue = getAppMigrationQueue()
await queue.add(
  { appId: workspaceId },
  { jobId: `${workspaceId}_${latestMigration}` }
)
```

- 向 `APP_MIGRATION` 队列添加任务
- **jobId 格式**：`{workspaceId}_{latestMigration}`
- 使用此 jobId 的好处：同一 workspace 的同一目标迁移不会重复入队（Bull 队列的去重机制）

队列配置见 [queue.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/queue.ts)：
- 最大重试次数：3 次
- 并发数：5 个（每节点）
- 任务完成/失败后自动移除

#### 3.4.2 决定是否等待迁移完成

根据 `shouldSkipMigrationWait(ctx)` 的返回值，有两种处理路径：

---

## 四、shouldSkipMigrationWait 逻辑

### 4.1 判定条件

[index.ts#L21-L29](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/index.ts#L21-L29)：

```typescript
const BUILDER_APP_PACKAGE_ENDPOINT =
  /^\/api\/applications\/[^/]+\/appPackage\/?$/

function shouldSkipMigrationWait(ctx: UserCtx): boolean {
  return (
    BUILDER_APP_PACKAGE_ENDPOINT.test(ctx.path || "") &&
    ctx.headers[Header.CLIENT] === ClientHeader.BUILDER
  )
}
```

**同时满足两个条件才跳过等待**：
1. 请求路径匹配 builder 的 appPackage 端点（`/api/applications/{appId}/appPackage`）
2. 请求头 `x-budibase-client` 值为 `builder`

### 4.2 设计意图

为什么只对这个端点和 builder 客户端生效？

- **Builder 的 appPackage 端点**：这是 builder 获取应用包定义的核心接口。builder 需要知道应用正在迁移，以便显示相应的 UI 状态，但不需要同步等待迁移完成——builder 可以通过轮询或其他方式感知迁移完成。
- **ClientHeader.BUILDER**：确保只有 builder 客户端享受此优化，普通客户端（end user）仍然需要同步等待或显示迁移中页面。
- **性能考量**：builder 可能频繁调用此接口，如果每次都同步等待，会严重影响 builder 的响应速度。

### 4.3 两种路径的行为差异

| 路径 | skip wait 为 false（普通请求） | skip wait 为 true（builder appPackage） |
|------|------------------------------|--------------------------------------|
| 响应头 `MIGRATING_APP` | 超时后设置 | 立即设置 |
| 响应头 `SKIP_MIGRATING_WAIT` | 不设置 | 设置为 `"true"` |
| 前端行为 | 调用 `onMigrationDetected` | 不调用，由 builder 自行处理 |

---

## 五、waitForMigration 逻辑

### 5.1 轮询机制

[index.ts#L100-L118](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/index.ts#L100-L118)：

```typescript
const waitForMigration = async (
  appId: string,
  { timeoutMs }: { timeoutMs: number }
): Promise<{ applied: boolean }> => {
  const start = Date.now()
  const devId = db.getDevWorkspaceID(appId)

  while (Date.now() - start < timeoutMs) {
    if (await isWorkspaceFullyMigrated(devId)) {
      console.log(`Migration ran in ${Date.now() - start}ms`)
      return { applied: true }
    }
    await new Promise(r => setTimeout(r, 10))
  }
  return { applied: false }
}
```

### 5.2 为什么检查 dev workspace migration version

注意这里使用的是 `db.getDevWorkspaceID(appId)`，即 **dev workspace** 的 ID 来检查迁移状态。原因：

- 对于已发布的应用，迁移会同时应用于 prod 和 dev 两个 workspace
- **dev workspace 的迁移总是在 prod 之后执行**（详见 `processMigrations` 分析）
- 因此，dev workspace 迁移完成意味着整个迁移流程（包括 prod 和 dev）都已完成
- 用 dev workspace 作为"完成"的判定标准，可以确保用户访问时所有数据都已就绪

### 5.3 超时后设置 Header.MIGRATING_APP

如果在 `SYNC_MIGRATION_CHECKS_MS` 时间内迁移未完成：
- 返回 `{ applied: false }`
- 调用方设置响应头 `Header.MIGRATING_APP = workspaceId`
- 前端检测到此响应头后会触发相应的迁移中处理逻辑

---

## 六、processMigrations 迁移执行逻辑

### 6.1 整体流程

[migrationsProcessor.ts#L86-L212](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/migrationsProcessor.ts#L86-L212) 的 `processMigrations` 函数是迁移队列的实际处理器。

### 6.2 Published workspace 的 prod/dev 迁移顺序

```typescript
const devWorkspaceId = db.getDevWorkspaceID(workspaceId)
const prodWorkspaceId = db.getProdWorkspaceID(workspaceId)
const isPublished = await sdk.workspaces.isWorkspacePublished(prodWorkspaceId)
const workspaceIdToMigrate = isPublished ? prodWorkspaceId : devWorkspaceId
```

- **未发布的应用**：只有 dev workspace，直接迁移 dev
- **已发布的应用**：有 prod 和 dev 两个 workspace
  - `workspaceIdToMigrate` 设为 `prodWorkspaceId`（主迁移目标）
  - dev workspace 作为辅助目标

### 6.3 迁移执行顺序（for 循环内）

对于每一个待执行的迁移：

```
1. 检查迁移是否被 disabled → 若 disabled 则中断整个迁移流程
2. 判断是否需要在 workspaceIdToMigrate (prod) 上执行
3. 判断是否需要在 dev workspace 上执行
4. 若 prod 需要执行：执行迁移
5. 若 dev 需要执行（且已发布）：
   a. 先调用 syncDevApp() 同步 workspace
   b. 再执行迁移
6. 更新 prod 的迁移版本（如果执行了）
7. 更新 dev 的迁移版本（如果执行了）
```

### 6.4 Disabled migration 中断

```typescript
if (disabled) {
  console.log(`Migration ${migrationId} is disabled, stopping migration process`)
  return
}
```

- 如果待执行的迁移列表中遇到 `disabled: true` 的迁移，立即停止整个迁移过程
- 这与 `getLatestEnabledMigrationId` 的逻辑一致：被禁用的迁移之后的所有迁移都不会执行
- **注意**：这里是在"待执行迁移列表"上检查，而不是完整的 MIGRATIONS 列表。由于待执行列表是从当前版本向后截取的，所以只要第一个待执行迁移没被禁用，就会一直执行下去

### 6.5 syncDevApp 的作用

[migrationsProcessor.ts#L62-L72](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/migrationsProcessor.ts#L62-L72)：

```typescript
async function syncDevApp(devWorkspaceId: string): Promise<void> {
  await context.doInWorkspaceMigrationContext(devWorkspaceId, async () => {
    await sdk.workspaces.syncWorkspace(devWorkspaceId)
  })
}
```

- 在对已发布应用的 dev workspace 执行迁移前，先同步 workspace
- 确保 dev workspace 的数据与 prod workspace 同步后再执行迁移
- 这是因为发布时 prod 是从 dev 复制的，但迁移过程中 prod 可能已经发生了变化

### 6.6 updateWorkspaceMigrationMetadata

`updateMigrationVersion` 函数调用 `updateWorkspaceMigrationMetadata`，其作用：

- 更新 workspace 的 migration metadata 文档中的 `version` 字段
- 在 `history` 中记录迁移执行时间
- **清除缓存**（`cache.destroy(cacheKey)`），确保下次读取时能获取最新版本

---

## 七、前端 API 层处理

### 7.1 frontend-core 的 handleMigrations

[frontend-core/src/api/index.ts#L208-L218](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/api/index.ts#L208-L218)：

```typescript
const handleMigrations = (response: Response) => {
  if (!config.onMigrationDetected) {
    return
  }
  const migration = response.headers.get(Header.MIGRATING_APP)
  const shouldSkipWait = response.headers.get(Header.SKIP_MIGRATING_WAIT)

  if (migration && !shouldSkipWait) {
    config.onMigrationDetected(migration)
  }
}
```

调用时机：每次 API 请求成功后（status 200-399），在解析响应体之前。

触发条件（同时满足）：
1. 响应头中包含 `x-budibase-migrating-app`（`Header.MIGRATING_APP`）
2. 响应头中**不包含** `x-budibase-migrating-app-skip-wait`（`Header.SKIP_MIGRATING_WAIT`）
3. API 客户端配置了 `onMigrationDetected` 回调

> **设计意图**：对于 builder 的 appPackage 请求，虽然也返回 MIGRATING_APP 头，但同时设置了 SKIP_MIGRATING_WAIT，因此不会触发 `onMigrationDetected`。builder 有自己的迁移状态处理逻辑。

### 7.2 client API 的 onMigrationDetected

[client/src/api/api.ts#L118-L123](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/client/src/api/api.ts#L118-L123)：

```typescript
onMigrationDetected: _appId => {
  if (!window.MIGRATING_APP) {
    window.location.reload()
  }
},
```

行为：
- 检查 `window.MIGRATING_APP` 全局标志（防止重复 reload）
- 如果尚未设置，执行 `window.location.reload()` 触发页面刷新

### 7.3 页面刷新后的行为

页面刷新后，请求会到达 `serveApp` 静态页面控制器：

[static/index.ts#L420-L424](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/static/index.ts#L420-L424)：

```typescript
const [fullyMigrated, ...] = await Promise.all([
  isWorkspaceFullyMigrated(workspaceId),
  ...
])
```

然后在 props 中设置：
```typescript
appMigrating: !fullyMigrated,
```

HTML 模板 [BudibaseApp.svelte#L142-L146](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/static/templates/BudibaseApp.svelte#L142-L146)：

```svelte
{#if props.appMigrating}
  <script type="application/javascript" nonce={props.nonce}>
    window.MIGRATING_APP = true
  </script>
{/if}
```

### 7.4 Client 端的 UpdatingApp 组件

[client/src/index.ts#L231-L238](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/client/src/index.ts#L231-L238)：

```typescript
if (window.MIGRATING_APP) {
  if (!app) {
    app = mount(UpdatingApp, { target })
  }
  return
}
```

- `loadBudibase` 函数启动时检查 `window.MIGRATING_APP`
- 如果为 true，挂载 `UpdatingApp` 组件（显示"应用正在更新"的界面）
- 不会挂载正常的 `ClientApp` 组件
- 页面会持续显示更新中状态，直到迁移完成后用户手动刷新或通过其他机制刷新

---

## 八、完整流程时序图

```
用户访问应用 API
    │
    ▼
server/api/index.ts 中间件链
    │
    ├── auth (认证)
    ├── tenancy (租户)
    ├── activeTenant
    ├── licensing
    ├── currentWorkspace  ──→ 设置 ctx.appId
    ├── CSP
    ├── auditLog
    └── workspaceMigrations
            │
            ├── DISABLE_WORKSPACE_MIGRATIONS? ──是──→ next()
            ├── 无 appId? ────────────────────是──→ next()
            │
            └── checkMissingMigrations()
                    │
                    ├── 计算 latestEnabledMigration
                    ├── workspace metadata 存在? ──否──→ next()
                    ├── isWorkspaceFullyMigrated? ──是──→ next()
                    │
                    ├── 队列添加任务 (jobId: workspaceId_latestMigration)
                    │
                    ├── shouldSkipMigrationWait?
                    │     ├── 是 (builder appPackage)
                    │     │    ├── 设置 Header.MIGRATING_APP
                    │     │    ├── 设置 Header.SKIP_MIGRATING_WAIT
                    │     │    └── next() → builder 自行处理
                    │     │
                    │     └── 否 (普通请求)
                    │          ├── waitForMigration()
                    │          │    ├── 轮询 dev workspace 迁移状态
                    │          │    ├── 10ms 间隔, 超时 SYNC_MIGRATION_CHECKS_MS
                    │          │    └── 完成? ──是──→ next() (正常响应)
                    │          └── 超时
                    │               ├── 设置 Header.MIGRATING_APP
                    │               └── next()
                    │
                    ▼
           前端 API 层 (frontend-core)
                    │
                    ├── handleMigrations(response)
                    ├── 有 MIGRATING_APP 且无 SKIP_MIGRATING_WAIT?
                    │     └── 是 → 调用 onMigrationDetected(appId)
                    │
                    ▼
           Client API (client/src/api/api.ts)
                    │
                    ├── onMigrationDetected
                    ├── window.MIGRATING_APP 未设置?
                    │     └── 是 → window.location.reload()
                    │
                    ▼
           页面刷新 → serveApp 静态页面
                    │
                    ├── 检查 isWorkspaceFullyMigrated
                    ├── 设置 appMigrating = !fullyMigrated
                    ├── HTML 中设置 window.MIGRATING_APP = true
                    │
                    ▼
           Client 启动 (loadBudibase)
                    │
                    ├── window.MIGRATING_APP 为 true?
                    │     └── 是 → 挂载 UpdatingApp 组件
                    │          (显示"应用正在更新"界面)
                    │
                    └── 否则 → 正常挂载 ClientApp
```

---

## 九、关键 Header 常量

[shared-core/src/constants/api.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/shared-core/src/constants/api.ts)：

| 常量名 | Header 值 | 用途 |
|--------|-----------|------|
| `Header.MIGRATING_APP` | `x-budibase-migrating-app` | 标记应用正在迁移中，值为 workspaceId |
| `Header.SKIP_MIGRATING_WAIT` | `x-budibase-migrating-app-skip-wait` | 标记跳过等待，由前端自行处理迁移状态 |
| `Header.CLIENT` | `x-budibase-client` | 客户端类型标识（builder / client） |
| `ClientHeader.BUILDER` | `"builder"` | builder 客户端的标识值 |

---

## 十、核心文件索引

| 文件路径 | 主要职责 |
|---------|---------|
| [packages/server/src/api/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/index.ts) | API 路由和中间件挂载 |
| [packages/server/src/middleware/workspaceMigrations.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/middleware/workspaceMigrations.ts) | 迁移中间件入口 |
| [packages/server/src/workspaceMigrations/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/index.ts) | 迁移检查核心逻辑（checkMissingMigrations, waitForMigration 等） |
| [packages/server/src/workspaceMigrations/migrationsProcessor.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/migrationsProcessor.ts) | 迁移执行处理器 |
| [packages/server/src/workspaceMigrations/queue.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/queue.ts) | 迁移队列配置 |
| [packages/server/src/workspaceMigrations/workspaceMigrationMetadata.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/workspaceMigrationMetadata.ts) | 迁移元数据读写 |
| [packages/server/src/workspaceMigrations/migrations.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/workspaceMigrations/migrations.ts) | 迁移列表定义 |
| [packages/server/src/api/controllers/static/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/static/index.ts) | 静态页面服务（serveApp） |
| [packages/frontend-core/src/api/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/api/index.ts) | 前端 API 客户端 |
| [packages/client/src/api/api.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/client/src/api/api.ts) | Client 端 API 配置 |
| [packages/client/src/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/client/src/index.ts) | Client 启动入口 |
| [packages/shared-core/src/constants/api.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/shared-core/src/constants/api.ts) | 共享 Header 常量 |
