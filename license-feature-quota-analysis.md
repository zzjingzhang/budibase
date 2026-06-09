# Budibase License、Feature 和 Quota 系统分析

## 目录

1. [License 中间件：将 License 注入请求上下文](#1-license-中间件将-license-注入请求上下文)
2. [LicenseAuth：基于 License Key 的认证与上下文切换](#2-licenseauth基于-license-key-的认证与上下文切换)
3. [Features 系统：License Feature 到业务开关的映射](#3-features-系统license-feature-到业务开关的映射)
4. [Quotas 系统：Dry Run → 执行业务 → 更新用量的三段式流程](#4-quotas-系统dry-run--执行业务--更新用量的三段式流程)
5. [总结](#5-总结)

---

## 1. License 中间件：将 License 注入请求上下文

### 1.1 调用位置与时机

在 server 端的 API 主入口中，`pro.licensing()` 中间件被放置在认证 (`buildAuthMiddleware`) 和租户激活 (`activeTenant`) 之后，确保用户身份和租户上下文已就绪。

[api/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/index.ts#L62-L79)

```typescript
router
  .use(auth.buildAuthMiddleware([], { publicAllowed: true }))
  .use(auth.buildTenancyMiddleware([], [], { noTenancyRequired: true }))
  .use(middleware.activeTenant())
  .use(pro.licensing())  // <-- license 注入点
  .use(currentWorkspace)
```

这个位置的选择很关键：
- **在认证之后**：保证 `ctx.user` 已存在，license 可以挂到用户对象上
- **在 activeTenant 之后**：保证 tenantId 已在上下文中，用于从缓存中获取对应租户的 license
- **在 currentWorkspace 之前**：后续的 workspace 相关逻辑可能需要 license 信息

### 1.2 Licensing 中间件核心逻辑

[licensing.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/middleware/licensing.ts#L1-L46)

```typescript
const licensing = (
  opts: LicenseMiddlewareOptions = { checkUsersLimit: true }
) => {
  return async (ctx: any, next: any) => {
    const licensingCheck = opts.licensingCheck
      ? opts.licensingCheck
      : () => !!ctx.user

    if (licensingCheck(ctx)) {
      if (env.SELF_HOSTED && env.DEFAULT_LICENSE) {
        ctx.user.license = SELF_FREE_LICENSE
        return next()
      }

      ctx.user.license = await licenses.cache.getCachedLicense()

      if (
        opts.checkUsersLimit &&
        (utils.isServingApp(ctx) ||
          utils.isServingBuilder(ctx) ||
          utils.isServingBuilderPreview(ctx) ||
          utils.isPublicApiRequest(ctx))
      ) {
        await quotas.usageLimitIsExceeded({
          name: StaticQuotaName.USERS,
          type: QuotaUsageType.STATIC,
          usageChange: 0,
        })
      }
    }

    return next()
  }
}
```

#### 1.2.1 License 设置的两条路径

**路径一：自托管默认 License**

当环境同时满足 `SELF_HOSTED` 和 `DEFAULT_LICENSE` 时，直接使用 `SELF_FREE_LICENSE`。这是自托管场景下的快速路径，不需要访问远程 license 服务或缓存。

[licenses.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/constants/licenses.ts#L45-L76)

`SELF_FREE_LICENSE` 的特点：
- features 为空数组（无付费功能）
- 大部分 quota 为 `UNLIMITED`（-1），如用户数、应用数、行数等
- userGroups 为 0，customAIConfigurations 为 0，plugins 为 10
- plan.type 为 `PlanType.FREE`

**路径二：缓存 License**

非自托管场景下，通过 `licenses.cache.getCachedLicense()` 从缓存中获取 license。这是一个多级缓存机制：

[cache.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/licensing/cache/cache.ts#L63-L110)

```
getCachedLicense()
  ├─ 先检查 context 中是否有 license（doInLicenseContext 设置的）
  ├─ 再从 Redis 缓存中查找
  ├─ 若缓存未命中，调用 populateAndStoreLicense()
  │   ├─ 加锁防止缓存击穿
  │   ├─ 调用 Licenses.getLicense() 从 license 服务获取
  │   ├─ 若获取失败，使用 getFreeLicense() 作为兜底
  │   └─ 存入 Redis，TTL = 2小时（1小时过期 + 1小时宽限期）
  └─ 若缓存命中但已过期，后台异步刷新
```

缓存策略：
- **过期时间**：1 小时 (`EXPIRY_SECONDS`)
- **宽限期**：1 小时 (`STALE_GRACE_SECONDS`)，过期后仍可使用旧数据，同时后台刷新
- **总 TTL**：2 小时
- **锁机制**：使用分布式锁防止缓存击穿

#### 1.2.2 用户配额 Dry Run 检查

在特定场景下（serving app/builder/preview/public API），会对 `StaticQuotaName.USERS` 执行一次 dry run 检查：

```typescript
await quotas.usageLimitIsExceeded({
  name: StaticQuotaName.USERS,
  type: QuotaUsageType.STATIC,
  usageChange: 0,
})
```

这里 `usageChange: 0` 意味着**不增加用量，只是检查当前用量是否已超限**。如果用户数已超过配额限制，即使不新增用户，访问应用/构建器也会被阻止。

判定请求类型的工具函数：

[utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/utils/utils.ts#L49-L75)

| 函数 | 判断条件 | 说明 |
|------|---------|------|
| `isServingApp` | `/app_` 前缀、`/app/`、`/app-chat/` | 访问应用页面 |
| `isServingBuilder` | `/builder/workspace/` | 访问构建器 |
| `isServingBuilderPreview` | `/app/app_xxx/preview` | 构建器预览 |
| `isPublicApiRequest` | `/api/public/v` | 公共 API |

---

## 2. LicenseAuth：基于 License Key 的认证与上下文切换

### 2.1 适用场景

`licenseAuth` 中间件主要用于**自托管用户使用 Budibase Cloud 服务**的场景（如 Budibase AI）。此时用户的 license key 作为 API key 使用。

### 2.2 核心流程

[licenseAuth.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/middleware/licenseAuth.ts#L1-L41)

```typescript
export default async function (ctx: Ctx, next: Next) {
  await tracer.trace("licenseAuth", async span => {
    // 1. 提取 license key
    let licenseKey =
      ctx.request.headers[constants.Header.LICENSE_KEY] ||
      utils.getBearerToken(ctx)
    if (Array.isArray(licenseKey)) {
      licenseKey = licenseKey[0]
    }

    if (!licenseKey) {
      ctx.throw(403, "License key not provided")
    }

    // 2. 验证 license
    const license = await getLicenseFromKey(licenseKey)
    if (!license) {
      ctx.throw(403, "License not found or invalid")
    }

    if (!license.tenantId) {
      ctx.throw(403, "License does not have a tenant ID")
    }

    // 3. 设置追踪上下文
    tracer.setUser({ id: "anonymous", tenantId: license.tenantId })

    // 4. 进入自托管云上下文 + License 上下文
    await context.doInSelfHostTenantUsingCloud(license.tenantId, async () => {
      await context.doInLicenseContext(license, async () => {
        await next()
      })
    })
  })
}
```

### 2.3 License Key 的两种提取方式

1. **License-Key 请求头**：通过 `constants.Header.LICENSE_KEY` 指定的 header 名称
2. **Bearer Token**：通过 `utils.getBearerToken(ctx)` 从 Authorization header 中提取

如果两者同时存在，header 方式优先。

### 2.4 双重上下文切换

#### doInSelfHostTenantUsingCloud

[mainContext.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/context/mainContext.ts#L174-L180)

```typescript
export async function doInSelfHostTenantUsingCloud<T>(
  tenantId: string,
  task: () => T
): Promise<T> {
  const updates = { tenantId, isSelfHostUsingCloud: true }
  return newContext(updates, task)
}
```

设置 `isSelfHostUsingCloud: true` 的影响：
- **数据库访问**：不能使用常规的 global DB 和 workspace DB，而是使用 `StaticDatabases.SELF_HOST_CLOUD`
- **配额存储**：配额文档存储位置不同，避免为每个自托管租户创建独立的 global DB

#### doInLicenseContext

[mainContext.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/context/mainContext.ts#L182-L187)

```typescript
export async function doInLicenseContext<T>(
  license: License,
  task: () => T
): Promise<T> {
  return newContext({ license }, task)
}
```

将 license 直接放入上下文后，`getCachedLicense()` 会优先从 context 中读取，跳过 Redis 缓存和远程调用：

[cache.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/licensing/cache/cache.ts#L65-L74)

```typescript
const fromContext = context.getLicense()
if (fromContext) {
  span.addTags({ foundInContext: true, ... })
  return fromContext
}
```

---

## 3. Features 系统：License Feature 到业务开关的映射

### 3.1 核心检查逻辑

[features.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/features/features.ts#L7-L39)

```typescript
async function areFeaturesEnabled(
  featureFlags: Feature[] | Feature,
  license?: License
) {
  if (!Array.isArray(featureFlags)) {
    featureFlags = [featureFlags]
  }
  if (!license) {
    license = await cache.getCachedLicense()
  }
  for (let flag of featureFlags) {
    if (!license?.features.includes(flag)) {
      return false
    }
  }
  return true
}

export async function checkFeature(featureFlag: Feature, license?: License) {
  if (!(await areFeaturesEnabled(featureFlag, license))) {
    throw new FeatureDisabledWarning(featureFlag)
  }
}
```

核心机制：
- License 对象中包含 `features: Feature[]` 数组
- 检查就是判断目标 feature 是否在这个数组中
- 支持传入 license 参数直接检查，否则从缓存获取
- 不通过时抛出 `FeatureDisabledWarning`

### 3.2 Feature 与 Plan 的映射关系

不同的订阅计划对应不同的 feature 集合：

[features.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/licensing/licenses/features.ts#L1-L139)

| 计划类型 | 部分关键 Feature |
|---------|----------------|
| FREE | 空数组（无付费功能） |
| PREMIUM | WORKSPACE_BACKUPS, BRANDING, VIEW_PERMISSIONS, PDF |
| PRO | BUDIBASE_AI, SYNC_AUTOMATIONS |
| TEAM | USER_GROUPS, WORKSPACE_BACKUPS |
| BUSINESS | USER_GROUPS, ENFORCEABLE_SSO, AUDIT_LOGS, SCIM 等 |
| ENTERPRISE | 全部 feature，包括 MICROFRONTEND 等高级功能 |

### 3.3 常用 Feature 检查函数

#### 通用包装模式：checkBackups

[features.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/features/features.ts#L43-L50)

```typescript
export function checkBackups<Args extends any[], Return>(
  targetFunction: (...parameters: Args) => Return
): (...parameters: Args) => Promise<Return> {
  return async (...parameters: Args) => {
    await checkFeature(Feature.WORKSPACE_BACKUPS)
    return targetFunction(...parameters)
  }
}
```

这是一个高阶函数包装模式，用于包装现有的业务函数，在执行前先检查 feature。

#### 复合检查：checkSCIM

[features.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/features/features.ts#L158-L169)

```typescript
export const checkSCIM = async (): Promise<boolean> => {
  const featureFlag = Feature.SCIM

  const featureEnabled = await areFeaturesEnabled(featureFlag)
  const scimConfig = await configs.getSCIMConfig()

  if (!featureEnabled || !scimConfig?.enabled) {
    throw new FeatureDisabledWarning(featureFlag)
  }

  return true
}
```

SCIM 的特殊性：需要同时满足 **license 授权** 和 **配置启用** 两个条件。

#### 组合检查：多个 feature 同时满足

```typescript
export async function isWorkspaceImportExportPublicApiEnabled() {
  return areFeaturesEnabled([
    Feature.EXPANDED_PUBLIC_API,
    Feature.WORKSPACE_IMPORT_EXPORT,
  ])
}
```

### 3.4 Feature 中间件

[feature.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/middleware/feature.ts#L1-L17)

```typescript
export const requireFeature = (featureFlag: Feature) => {
  return async (ctx: any, next: Next) => {
    await features.checkFeature(featureFlag)
    await next()
  }
}
```

可以在路由级别直接使用，保护整个路由组：

```typescript
router.use('/scim', pro.middleware.requireSCIM, scimRoutes.routes())
```

### 3.5 FeatureDisabledWarning

[warnings.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/warnings/warnings.ts#L45-L64)

```typescript
export class FeatureDisabledWarning extends APIWarning {
  featureName: string
  constructor(featureName: string) {
    super(`Feature disabled: '${featureName}'`, APIWarningCode.FEATURE_DISABLED)
    this.featureName = featureName
    this.status = 400
  }
}
```

- 继承自 `APIWarning`
- 状态码 400
- 错误码 `FEATURE_DISABLED`
- 包含 `featureName` 字段，前端可根据此字段显示友好提示

---

## 4. Quotas 系统：Dry Run → 执行业务 → 更新用量的三段式流程

### 4.1 配额类型体系

#### 按使用方式分类

[quotas.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/types/src/sdk/licensing/quota.ts) （概念性）

| 类型 | 说明 | 示例 |
|-----|------|------|
| STATIC | 静态配额，总量限制，不随时间重置 | 用户数、应用数、行数 |
| MONTHLY | 月度配额，每月重置 | 查询次数、自动化运行次数、AI credits |
| CONSTANT | 常量型配置，不是用量限制 | 日志保留天数、备份保留天数 |

#### 按作用范围分类

- **租户级**：整个租户的总量限制（默认）
- **应用级**：每个应用单独限制（`APP_QUOTA_NAMES` 中的配额）
- **细粒度**：通过 `id` 参数指定具体资源

### 4.2 核心流程：tryIncrement 的三段式设计

[quotas.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/quotas/quotas.ts#L94-L148)

```typescript
const tryIncrement = async <T = any>(
  params: IncrementManyParams<T> | IncrementManyParams<T>[]
): Promise<T> => {
  const actions = Array.isArray(params) ? params : [params]

  // 第一段：Dry Run 预检查
  await updateUsage(
    actions.map(action => ({
      usageChange: action.change,
      name: action.name,
      type: action.type,
      opts: { dryRun: true, suppressErrorLog: ..., id: ... }
    }))
  )

  // 第二段：执行业务函数
  const results: any[] = []
  for (let action of actions) {
    let fn = action.opts?.fn
    if (fn) {
      results.push(await fn())
    }
  }

  // 第三段：真正更新用量
  await updateUsage(
    actions.map(action => ({
      usageChange: action.change,
      name: action.name,
      type: action.type,
      opts: { dryRun: false, valueFn: ..., ... }
    }))
  )

  return results[0]
}
```

#### 为什么要先 Dry Run？

这种设计模式的核心目的是**保证配额检查和业务操作的原子性语义**：

1. **前置校验**：在执行业务操作前就知道是否会超限，避免"做了一半发现配额不够"的尴尬
2. **快速失败**：不用等待业务操作完成就能返回错误
3. **事务语义**：虽然不是真正的数据库事务，但尽量保证"要么都成功，要么都不做"

如果业务函数执行成功但更新配额失败（极端情况），可以通过 `valueFn` 在下次更新时重新计算实际用量。

### 4.3 updateUsage 深度解析

[quotas.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/quotas/quotas.ts#L306-L452)

#### 4.3.1 读取 License Quota

```typescript
licensedQuota = await getLicensedQuota(
  QuotaType.USAGE,
  action.name,
  action.type
)
```

[quotas.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/quotas/quotas.ts#L454-L479)

```typescript
export const getLicensedQuota = async (
  quotaType: QuotaType,
  name: MonthlyQuotaName | StaticQuotaName | ConstantQuotaName,
  usageType?: QuotaUsageType
): Promise<Quota> => {
  const license = await licensing.cache.getCachedLicense()

  if (!license) {
    throw new Error("License not found for tenant id " + tenantId)
  }

  if (usageType && isStaticQuota(quotaType, usageType, name)) {
    return license.quotas[QuotaType.USAGE][QuotaUsageType.STATIC][name]
  } else if (usageType && isMonthlyQuota(quotaType, usageType, name)) {
    return license.quotas[QuotaType.USAGE][QuotaUsageType.MONTHLY][name]
  } else if (isConstantQuota(quotaType, name)) {
    return license.quotas[QuotaType.CONSTANT][name]
  } else {
    throw new Error("Invalid quota type")
  }
}
```

Quota 对象的结构（以 USERS 为例）：

[quotas.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/constants/quotas.ts#L31-L39)

```typescript
{
  name: "Users",      // 显示名称
  value: 5,           // 配额上限
  triggers: [80, 100] // 触发阈值百分比
}
```

#### 4.3.2 App Quota 与 Workspace Context

```typescript
try {
  appId = context.getWorkspaceId()
} catch (err) {
  // ignore error for now
}

const isAppQuota = APP_QUOTA_NAMES.includes(action.name)
if (isAppQuota && !appId) {
  throw new Error("App context required for quota update")
}
```

应用级配额（如每个应用的行数限制）需要 `workspaceId` 来区分不同应用的用量。如果不在 workspace 上下文中调用会直接报错。

#### 4.3.3 超限判断与 UsageLimitWarning

```typescript
if (
  licensedQuota.value !== constants.UNLIMITED &&
  totalValue > licensedQuota.value &&
  action.usageChange > 0  // 只有增加用量时才限制
) {
  throw new UsageLimitWarning(`${action.name}`)
}
```

三个条件同时满足才会抛错：
1. **不是无限配额**（`UNLIMITED = -1`）
2. **总用量超过配额**
3. **本次是增加用量**（`usageChange > 0`）

第三条很重要——**减少用量永远不会被阻止**，即使当前已经超限。这允许管理员在超限时仍能删除用户/应用来恢复正常。

[warnings.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/warnings/warnings.ts#L27-L43)

```typescript
export class UsageLimitWarning extends APIWarning {
  limitName: string

  constructor(limitName: string) {
    super(
      `Usage limit exceeded: '${limitName}'`,
      APIWarningCode.USAGE_LIMIT_EXCEEDED
    )
    this.limitName = limitName
  }

  getPublicWarning() {
    return { limitName: this.limitName }
  }
}
```

#### 4.3.4 Quota Triggers（警告触发）

当非 dry run 时，会检查并触发配额警告：

[quotas.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/quotas/quotas.ts#L261-L304)

```typescript
if (!action.opts?.dryRun) {
  triggers = await checkTriggers(action.type, action.name, totalValue, licensedQuota)
  triggersToApply = { ...triggersToApply, [action.name]: triggers }
}
```

`checkTriggers` 的逻辑：
1. 获取当前已触发的警告记录
2. 计算当前用量占配额的百分比
3. 遍历配额定义的 triggers 数组（如 `[80, 100]`）
4. 对每个未触发过且已达到的阈值，记录触发时间并发送通知
5. 使用分布式锁确保同一警告只发送一次

触发通知通过 license client 发送到授权服务：

```typescript
await licensing.client.triggerQuota(request)
```

#### 4.3.5 valueFn 的作用

在真正更新阶段（非 dry run），如果提供了 `valueFn`，会用它的返回值作为新的用量值，而不是简单的增减：

```typescript
if (!action.opts?.dryRun) {
  let fn = action.opts?.valueFn
  if (fn) {
    totalValue = await fn()
    appValue = totalValue
  }
  // ...
}
```

这在某些场景下很有用，比如用户数配额——不是每次加 1，而是直接查询数据库中的用户总数来校准。

[users.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/quotas/helpers/users.ts#L7-L49)

```typescript
export const addUsers = async <T>(
  change: number,
  changeCreator: number,
  addUsersFn?: () => Promise<T>
): Promise<T> => {
  const quotasToChange: IncrementManyParams[] = [
    {
      change,
      name: StaticQuotaName.USERS,
      type: QuotaUsageType.STATIC,
      opts: {
        fn: addUsersFn,
        valueFn: users.getUserCount,  // <-- 用实际查询结果校准
      },
    },
  ]
  // ...
}
```

### 4.4 usageLimitIsExceeded：便捷检查函数

[quotas.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/quotas/quotas.ts#L481-L499)

```typescript
export const usageLimitIsExceeded = async ({
  name,
  type,
  usageChange,
}: {
  name: MeteredQuotaName
  type: QuotaUsageType
  usageChange: number
}) => {
  try {
    await updateUsage({ usageChange, name, type, opts: { dryRun: true } })
    return false
  } catch (e: any) {
    if (e.code === APIWarningCode.USAGE_LIMIT_EXCEEDED) {
      return true
    }
    throw e
  }
}
```

这是对 `updateUsage` dry run 的封装，返回布尔值而不是抛出异常。在 licensing 中间件中就是用的这个函数。

### 4.5 decrement：减少用量

减少用量的逻辑简单很多，不需要三段式，直接调用 `updateUsage` 即可：

```typescript
export const decrement = (
  name: MeteredQuotaName,
  type: QuotaUsageType,
  opts: DecrementOptions = {}
) => {
  return updateUsage({ usageChange: -1, name, type, opts })
}
```

因为减少用量不会触发超限错误，所以不需要预检查。

---

## 5. 总结

### 5.1 整体架构图

```
HTTP Request
    │
    ▼
┌─────────────────────┐
│  buildAuthMiddleware │  ← 认证用户，设置 ctx.user
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ buildTenancyMiddleware │ ← 解析租户 ID
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   activeTenant()    │  ← 激活租户上下文
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   pro.licensing()   │  ← 注入 license 到 ctx.user.license
│  ├─ SELF_HOSTED?    │     是 → SELF_FREE_LICENSE
│  └─ 否              │     否 → 缓存获取
│     └─ USERS dryRun │     serving app/builder/public API 时检查
└─────────┬───────────┘
          │
          ▼
        业务路由
          │
          ├─ Feature 检查 → checkFeature / checkSCIM / checkBackups
          │                    ↓
          │                license.features.includes(flag)
          │
          └─ Quota 操作 → increment / tryIncrement
                           ↓
                   ┌───────────────────────┐
                   │  1. Dry Run 预检查    │
                   │     超限立即抛错      │
                   └───────────┬───────────┘
                               │
                   ┌───────────▼───────────┐
                   │  2. 执行业务函数 fn   │
                   └───────────┬───────────┘
                               │
                   ┌───────────▼───────────┐
                   │  3. 真正更新用量      │
                   │     - 检查 triggers   │
                   │     - 超限抛 Warning  │
                   │     - 写入数据库      │
                   └───────────────────────┘
```

### 5.2 LicenseAuth 特殊流程

```
API 请求（如 AI 服务）
    │
    ▼
┌─────────────────────┐
│   licenseAuth()     │
│  ├─ 提取 licenseKey  │  ← header 或 bearer token
│  ├─ 验证 license    │
│  ├─ 取 tenantId     │
│  └─ 双重上下文切换   │
│     ├─ doInSelfHostTenantUsingCloud
│     └─ doInLicenseContext
└─────────┬───────────┘
          │
          ▼
      业务处理
    (license 直接从
     context 中获取)
```

### 5.3 关键设计要点

| 模块 | 设计特点 | 目的 |
|-----|---------|------|
| Licensing 中间件 | 认证后注入，缓存优先 | 每个请求都能获取 license 信息 |
| License 缓存 | Redis + 后台刷新 + 宽限期 | 减少远程调用，提高可用性 |
| Features 检查 | 纯数组包含判断 | 简单高效，易于扩展 |
| Quotas 三段式 | Dry Run → 执行业务 → 更新用量 | 保证配额语义的正确性 |
| Quota Triggers | 百分比阈值 + 分布式锁 | 预警通知，避免重复发送 |
| 超限判断 | 只限制增量，允许减量 | 超限时仍可通过删除资源恢复 |
| App Quota | 依赖 workspace context | 支持应用级别的细粒度配额 |
| LicenseAuth | 双重上下文切换 | 支持自托管用户使用云服务 |

### 5.4 核心文件索引

| 模块 | 文件路径 |
|-----|---------|
| Licensing 中间件 | [licensing.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/middleware/licensing.ts) |
| License 缓存 | [cache.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/licensing/cache/cache.ts) |
| License 常量 | [licenses.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/constants/licenses.ts) |
| LicenseAuth 中间件 | [licenseAuth.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/middleware/licenseAuth.ts) |
| 上下文管理 | [mainContext.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/context/mainContext.ts) |
| Features SDK | [features.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/features/features.ts) |
| Feature-Plan 映射 | [features.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/licensing/licenses/features.ts) |
| Quotas SDK | [quotas.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/quotas/quotas.ts) |
| Quota 常量 | [quotas.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/constants/quotas.ts) |
| 用户配额助手 | [users.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/quotas/helpers/users.ts) |
| 警告类定义 | [warnings.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/warnings/warnings.ts) |
| Server API 入口 | [api/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/index.ts) |
