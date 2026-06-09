# SCIMv2 用户和组接口防护机制分析

本文档分析 Budibase 中 SCIMv2 用户和组接口如何防止普通资源被 SCIM 误改。

## 目录

1. [请求入口：Content-Type 转换](#1-请求入口content-type-转换)
2. [路由级中间件：三层防护](#2-路由级中间件三层防护)
3. [资源级中间件：scimUserOnly / scimGroupOnly](#3-资源级中间件scimuseronly--scimgrouponly)
4. [用户接口防护逻辑](#4-用户接口防护逻辑)
5. [组接口防护逻辑](#5-组接口防护逻辑)

---

## 1. 请求入口：Content-Type 转换

### 1.1 handleScimBody 中间件

**文件**：[handleScimBody.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/middleware/handleScimBody.ts)

SCIM 协议标准使用 `application/scim+json` 作为 Content-Type，但 Budibase 的 Koa 服务只识别 `application/json`。`handleScimBody` 中间件在请求进入路由处理前，将 SCIM 的 Content-Type 转换为标准 JSON 类型，确保 body 解析中间件能正常工作。

```typescript
export const handleScimBody = (ctx: Ctx, next: any) => {
  let type = ctx.req.headers["content-type"] || ""
  type = type.split(";")[0]

  if (type === "application/scim+json") {
    ctx.req.headers["content-type"] = "application/json"
  }

  return next()
}
```

**作用**：这是 SCIM 请求进入系统的第一道转换关口，确保请求体能够被正常解析。

---

## 2. 路由级中间件：三层防护

**文件**：[scim.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/routes/global/scim.ts)

所有 `/api/global/scim/v2` 路由依次经过以下三个中间件：

```typescript
router.use(proMiddleware.requireSCIM)
router.use(proMiddleware.doInScimContext)
router.use(auth.adminOnly)
```

### 2.1 requireSCIM — 功能开关检查

**文件**：[requireSCIM.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/middleware/requireSCIM.ts)

`requireSCIM` 调用 `features.checkSCIM()`，该函数同时检查**两个条件**：

1. **License 功能开关**：`Feature.SCIM` 是否在 License 中启用
2. **配置开关**：`configs.getSCIMConfig().enabled` 是否为 true

**文件**：[features.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/features/features.ts#L158-L169)

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

**防护作用**：只有 License 支持 SCIM 且管理员在设置中开启了 SCIM，接口才可用。任何一个条件不满足，请求都会被拒绝。

### 2.2 doInScimContext — 上下文标记

**文件**：[doInScimContext.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/middleware/doInScimContext.ts)

`doInScimContext` 将当前请求放入 SCIM 上下文中，设置 `isScim: true` 标记。底层实现：

**文件**：[mainContext.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/context/mainContext.ts#L346-L351)

```typescript
export function doInScimContext(task: any) {
  const updates: ContextMap = {
    isScim: true,
  }
  return newContext(updates, task)
}
```

**防护作用**：通过 `context.isScim()` 可以在业务逻辑中判断当前操作是否来自 SCIM 渠道，从而采取不同的处理策略（例如跳过某些普通用户的校验逻辑）。

### 2.3 adminOnly — 管理员权限验证

**来源**：`@budibase/backend-core` 的 `auth.adminOnly`

**文件**：[adminOnly.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/middleware/adminOnly.ts)

`adminOnly` 验证当前用户是否为管理员。SCIM 接口需要管理员权限才能访问。

**防护作用**：确保只有拥有管理员权限的请求（通过 Basic Auth 或 API Key）才能调用 SCIM 接口，防止普通用户通过 SCIM 接口操作数据。

---

## 3. 资源级中间件：scimUserOnly / scimGroupOnly

对于针对特定资源的操作（GET/PATCH/DELETE 单个用户或组），使用 `scimUserOnly` 和 `scimGroupOnly` 中间件进行防护。

**文件**：[scimSyncResourceRestricted.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/middleware/scimSyncResourceRestricted.ts)

### 3.1 核心逻辑

```typescript
export const scimUserOnly = (paramId: string) =>
  scimSyncChecks(users.getById, paramId, true)

export const scimGroupOnly = (paramId: string) =>
  scimSyncChecks(groups.get, paramId, true)

function scimSyncChecks(
  getter: (id: string) => Promise<User | UserGroup>,
  paramId: string,
  scimRequired: boolean
) {
  return async (ctx: Ctx, next: any) => {
    const id = ctx.params[paramId]

    if (typeof id !== "string") {
      ctx.throw(404)
    }

    const existingResource = await getter(id)
    if (!!existingResource.scimInfo?.isSync !== scimRequired) {
      ctx.throw(404)
    }

    return next()
  }
}
```

### 3.2 防护原理

中间件通过检查目标资源的 `scimInfo.isSync` 字段来判断该资源是否为 SCIM 同步资源：

- **scimUserOnly/scimGroupOnly**（`scimRequired = true`）：只允许操作 `scimInfo.isSync = true` 的资源
- **internalGroupOnly**（`scimRequired = false`）：只允许操作非 SCIM 同步的资源

如果资源的 `scimInfo.isSync` 状态与要求不符，返回 **404**（而不是 403），避免泄露资源存在性。

### 3.3 在路由中的应用

**用户路由**：

```typescript
router.get("/users/:id", proMiddleware.scimUserOnly("id"), userController.find)
router.patch("/users/:id", proMiddleware.scimUserOnly("id"), userController.update)
router.delete("/users/:id", proMiddleware.scimUserOnly("id"), userController.remove)
```

**组路由**：

```typescript
router.get("/groups/:id", proMiddleware.feature.requireFeature(Feature.USER_GROUPS), proMiddleware.scimGroupOnly("id"), groupController.find)
router.patch("/groups/:id", proMiddleware.feature.requireFeature(Feature.USER_GROUPS), proMiddleware.scimGroupOnly("id"), groupController.update)
router.delete("/groups/:id", proMiddleware.feature.requireFeature(Feature.USER_GROUPS), proMiddleware.scimGroupOnly("id"), groupController.remove)
```

**防护作用**：确保 SCIM 接口只能操作由 SCIM 创建/同步的资源，无法修改或删除普通用户手动创建的资源。

---

## 4. 用户接口防护逻辑

### 4.1 用户列表查询 — 只返回 SCIM 用户

**文件**：[users.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/scim/users.ts#L22-L45)

SCIM 用户列表查询通过 `builder.addEqual("scimInfo.isSync", true)` 过滤，只返回 SCIM 同步的用户：

```typescript
export async function get(params: GetParams): Promise<{ users: User[]; total: number }> {
  const db = tenancy.getGlobalDB()

  const builder = new dbCore.QueryBuilder<User>(db.name, SearchIndex.USER)
  builder.setIndexBuilder(dbCore.searchIndexes.createUserIndex)
  builder.setLimit(params.pageSize)
  builder.addEqual("scimInfo.isSync", true)  // 关键过滤条件

  // ...
}
```

### 4.2 用户创建 — 邮箱存在时合并而非报错

**文件**：[users.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/scim/users.ts#L51-L68)

当 SCIM 创建用户时，如果邮箱已存在：

- **已存在且是 SCIM 用户**：抛出 409 冲突错误
- **已存在但不是 SCIM 用户**：合并 `scimInfo`，清空 `password`，将其转化为 SCIM 用户

```typescript
export async function create(user: User) {
  const existingUser = await userDb.getUserByEmail(user.email)
  if (existingUser) {
    if (existingUser.scimInfo?.isSync) {
      throw new HTTPError(`User is already synched`, 409)
    }
    user = {
      ...existingUser,
      scimInfo: lodashMerge(existingUser.scimInfo, user.scimInfo),
      password: undefined,   // 清空密码，SCIM用户不由密码登录
      firstName: user.firstName,
      lastName: user.lastName,
      updatedAt: user.updatedAt,
    }
  }

  return await userDb.save(user, { requirePassword: false })
}
```

**防护作用**：
1. 防止重复创建 SCIM 用户（已同步的报 409）
2. 对已存在的普通用户，通过合并 `scimInfo` 并清空密码，安全地将其纳入 SCIM 管理
3. 清空 `password` 确保 SCIM 用户无法通过密码登录（通常配合 SSO 使用）

### 4.3 用户更新 — active=false 转换为删除

**文件**：[users.ts (controller)](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/controllers/global/scim/users.ts#L66-L114)

SCIM 协议中，用户禁用通过 `active = false` 表示。Budibase 将其转换为物理删除操作：

```typescript
function isDeactivation(request: ScimUpdateRequest) {
  const activeFieldChange = request.Operations.find(
    o => (o.op === "Replace" || o.op === "replace") && o.path === "active"
  )
  if (!activeFieldChange) {
    return false
  }

  return (
    activeFieldChange.value === false ||
    activeFieldChange.value?.toLowerCase?.() === "false"
  )
}

export const update = async (ctx: Ctx<ScimUpdateRequest, ScimUserResponse>) => {
  // ...
  if (isDeactivation(patchs)) {
    return remove(ctx)  // 直接调用删除逻辑
  }
  // ...
}
```

**防护作用**：
1. 符合 SCIM 协议规范，外部系统通过设置 `active=false` 禁用用户
2. 内部统一走删除流程，避免出现"禁用但还存在"的中间状态导致权限混乱
3. 结合 `scimUserOnly` 中间件，确保只能删除 SCIM 同步的用户

---

## 5. 组接口防护逻辑

### 5.1 组列表查询 — 只返回 SCIM 组

**文件**：[groups.ts (controller)](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/controllers/global/scim/groups.ts#L21-L47)

SCIM 组列表查询通过 `.filter(g => g.scimInfo?.isSync)` 过滤，只返回 SCIM 同步的组：

```typescript
let result = fetchedGroups
  .filter(g => g.scimInfo?.isSync)
  .map(mappers.group.toScimGroupResponse)
```

### 5.2 组创建 — 已存在非 SCIM 组清空成员后接管

**文件**：[groups.ts (sdk)](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/scim/groups.ts#L8-L32)

当 SCIM 创建组时，如果组名已存在：

- **已存在且是 SCIM 组**：抛出 409 冲突错误
- **已存在但不是 SCIM 组**：清空现有用户，合并 `scimInfo`，将其转化为 SCIM 组

```typescript
export async function create(data: UserGroup) {
  const existingGroup = await groupDb.getByName(data.name)

  let groupId
  if (!existingGroup) {
    const createdGroup = await groups.save(data)
    groupId = createdGroup.id!
  } else {
    if (existingGroup.scimInfo?.isSync) {
      throw new HTTPError(`Group is already synched`, 409)
    }

    groupId = existingGroup._id!
    if (existingGroup.users.length) {
      await groups.removeUsers(
        groupId,
        existingGroup.users.map(u => u._id)
      )
    }
    existingGroup.scimInfo = lodashMerge(existingGroup.scimInfo, data.scimInfo)
    await groups.save(existingGroup)
  }
  const group = await groups.get(groupId)
  return group
}
```

**防护作用**：
1. 防止重复创建同名 SCIM 组
2. 对已存在的普通组，**清空原有成员**后再纳入 SCIM 管理，避免 SCIM 系统不知情地继承了原有成员导致权限泄露
3. 清空成员确保 SCIM 组的成员列表完全由 SCIM 系统控制

### 5.3 组 PATCH — members 与字段操作分离，禁止 replace members

**文件**：[groups.ts (controller)](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/controllers/global/scim/groups.ts#L77-L157)

组的 PATCH 更新将操作分为两类处理：

#### 步骤一：分离成员操作与字段操作

使用 `lodash/groupBy` 按 `path === "members"` 将 Operations 分为两组：

```typescript
const { true: memberOps, false: fieldOps } = groupBy(
  patchs.Operations,
  p => p.path === "members"
)
```

#### 步骤二：字段操作 — 使用 scimPatch 统一处理

```typescript
if (fieldOps?.length) {
  const patchedScimGroup = scimPatch(scimGroup, fieldOps)
  // ...
  const groupToUpdate: UserGroup = {
    ...mappers.group.fromScimGroup(patchedScimGroup),
    _rev: group._rev,
  }
  await groups.save(groupToUpdate)
}
```

#### 步骤三：成员操作 — 单独处理，禁止 replace

```typescript
if (memberOps?.length) {
  const usersToAdd = []
  const usersToRemove = []
  for (const { op, value } of memberOps) {
    switch (op) {
      case "add":
      case "Add":
        // 添加成员...
        break
      case "remove":
      case "Remove":
        // 移除成员...
        break
      case "replace":
      case "Replace":
        throw new Error("Replacing members is not allowed")  // 关键：禁止替换
      default:
        utils.unreachable(op)
    }
  }
  // 执行添加/移除操作...
}
```

**防护作用**：
1. **分离处理**：成员操作和字段操作采用不同的处理路径，避免 `scimPatch` 库对 members 字段的意外处理
2. **禁止 replace**：SCIM 协议允许 `replace` 操作直接替换整个成员列表，但 Budibase 显式禁止了这种操作，防止一次性误操作清空或替换所有组成员
3. **增量操作**：只允许 `add` 和 `remove` 增量修改成员，更加安全可控
4. **成员身份校验**：添加/移除的成员通过 `scimUsers.find()` 查询，确保操作的是 SCIM 用户

---

## 总结：多层防护体系

| 防护层级 | 机制 | 作用 |
|---------|------|------|
| 入口层 | handleScimBody | 转换 Content-Type，确保请求体正常解析 |
| 功能层 | requireSCIM | 检查 License 功能 + 配置开关，双重把关 |
| 上下文层 | doInScimContext | 标记 SCIM 上下文，供业务逻辑区分渠道 |
| 权限层 | adminOnly | 确保只有管理员能调用 SCIM 接口 |
| 资源层 | scimUserOnly/scimGroupOnly | 只允许操作 `scimInfo.isSync=true` 的资源 |
| 业务层 | 用户 create 合并逻辑 | 邮箱存在时合并 scimInfo、清空 password |
| 业务层 | 用户 update active→remove | 禁用转换为删除，避免中间状态 |
| 业务层 | 组 create 清空成员 | 接管普通组前清空成员，防止权限泄露 |
| 业务层 | 组 PATCH 分离+禁止replace | 成员操作独立处理，禁止整体替换 |

这套防护体系从入口到业务逻辑层层递进，确保 SCIM 接口不会意外修改或删除普通资源，同时安全地将已有资源纳入 SCIM 管理。
