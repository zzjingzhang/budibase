# Public API 调用链追踪文档

本文档追踪 Budibase Public API 中 `POST /api/public/v1/tables/:tableId/rows/search` 和 `PUT /api/public/v1/tables/:tableId/rows/:rowId` 两个端点的完整调用链，说明各中间件和控制器的作用机制。

---

## 1. public/index.ts 中间件应用流程

### 1.1 全局中间件（路由器级别）

[public/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/public/index.ts)

**Rate Limiter（速率限制）**

- 使用 `koa2-ratelimit` 库的 `RateLimit.middleware`
- 默认每秒 10 个请求（测试环境 100 个），可通过 `API_REQ_LIMIT_PER_SEC` 环境变量覆盖
- 非测试环境使用 Redis 存储（`Stores.Redis`），通过 `getPublicApiRedisConfig` 配置
- 仅在非开发环境（`!env.isDev()`）且未禁用（`!env.DISABLE_RATE_LIMITING`）时生效
- 通过 `publicRouter.use(limiter)` 应用到所有路由

**CORS（跨域资源共享）**

- 使用 `@koa/cors` 库
- 通过 `publicRouter.use(cors())` 应用到所有路由
- 位于 rate limiter 之后

### 1.2 端点级中间件（applyRoutes 函数）

`applyRoutes(endpoints, permType, resource, subResource?)` 函数为每类端点统一添加中间件，执行顺序如下：

| 顺序 | 中间件 | 作用 | 应用方式 |
|------|--------|------|----------|
| 1 | `publicApiMiddleware` | 验证 API key，可选验证 appId | `addMiddleware`（输入中间件） |
| 2 | `paramResource` / `paramSubResource` | 从 URL params 提取 resourceId / subResourceId 到 ctx | `addMiddleware`（输入中间件） |
| 3 | `authorized(permType, permLevel)` | 权限校验：resource role 或 base permission | `addMiddleware`（输入中间件） |
| 4 | `controller` | 业务逻辑控制器 | Endpoint 构造时传入 |
| 5 | `mapperMiddleware` | 输出格式映射（内部→外部） | `addOutputMiddleware`（输出中间件） |
| 6 | `testErrorHandling`（仅测试环境） | 测试错误处理 | `addMiddleware`（输入中间件，但在授权后） |

> **注意**：`addMiddleware` 添加的是输入中间件（控制器前执行），`addOutputMiddleware` 添加的是输出中间件（控制器后执行）。

### 1.3 Rows 端点的特殊配置

Rows 端点通过 `applyRoutes(rowEndpoints, PermissionType.TABLE, "tableId", "rowId")` 注册：

- `permType` = `PermissionType.TABLE`
- `resource` = `"tableId"`（主资源参数名）
- `subResource` = `"rowId"`（子资源参数名）
- 因为有 subResource，使用 `paramSubResource("tableId", "rowId")`
- `requiresAppId` = `true`（因为 `permType !== PermissionType.WORKSPACE && permType !== PermissionType.USER`）

### 1.4 Endpoint 类执行机制

[Endpoint.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/Endpoint/Endpoint.ts)

`Endpoint` 类的 `apply(router)` 方法按以下顺序组装路由参数：

```
[url, ...middlewares, controller, ...outputMiddlewares, complete]
```

即：输入中间件 → 控制器 → 输出中间件 → complete（空函数，终止执行链）

---

## 2. publicApiMiddleware：API Key 与 AppId 验证

[publicApi.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/middleware/publicApi.ts)

### 2.1 函数签名

```typescript
export function publicApiMiddleware({
  requiresAppId,
}: { requiresAppId?: boolean } = {})
```

### 2.2 验证逻辑

1. **AppId 验证（条件性）**：
   - 调用 `utils.getWorkspaceIdFromCtx(ctx)` 获取 appId
   - 当 `requiresAppId === true` 且未获取到 appId 时，抛出 400 错误
   - 错误信息：`Invalid app ID provided, please check the x-budibase-app-id header.`

2. **API Key 验证（始终执行）**：
   - 检查 `ctx.headers[constants.Header.API_KEY]`（即 `x-budibase-api-key` 请求头）
   - 不存在时抛出 400 错误
   - 错误信息：`Invalid API key provided, please check the x-budibase-api-key header.`

### 2.3 何时要求 appId

在 [public/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/public/index.ts#L113-L116) 中：

```typescript
const publicApiMiddleware = publicApi({
  requiresAppId:
    permType !== PermissionType.WORKSPACE && permType !== PermissionType.USER,
})
```

- `PermissionType.WORKSPACE` → 不需要 appId（workspace 级别操作）
- `PermissionType.USER` → 不需要 appId（全局用户操作）
- 其他类型（`TABLE`, `VIEW`, `QUERY`, `ROW` 等）→ 需要 appId

对于 rows 端点（`PermissionType.TABLE`），**必须**提供 appId。

---

## 3. rows.ts 中 Endpoint 对象注册 read/write

[rows.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/public/rows.ts)

### 3.1 分组方式

文件开头声明两个数组：
- `read = []` — 读操作端点
- `write = []` — 写操作端点

导出默认对象：`{ read, write }`

### 3.2 读操作端点（read 数组）

| 方法 | 路径 | 控制器 | 额外中间件 |
|------|------|--------|------------|
| GET | `/tables/:tableId/rows/:rowId` | `controller.read` | 无 |
| POST | `/tables/:tableId/rows/search` | `controller.search` | `externalSearchValidator()` |
| POST | `/views/:viewId/rows/search` | `controller.viewSearch` | `externalSearchValidator()` |

### 3.3 写操作端点（write 数组）

| 方法 | 路径 | 控制器 |
|------|------|--------|
| POST | `/tables/:tableId/rows` | `controller.create` |
| PUT | `/tables/:tableId/rows/:rowId` | `controller.update` |
| DELETE | `/tables/:tableId/rows/:rowId` | `controller.destroy` |

### 3.4 Endpoint 构造示例

```typescript
// search 端点（读操作）
read.push(
  new Endpoint(
    "post",
    "/tables/:tableId/rows/search",
    controller.search
  ).addMiddleware(externalSearchValidator())
)

// update 端点（写操作）
write.push(
  new Endpoint("put", "/tables/:tableId/rows/:rowId", controller.update)
)
```

---

## 4. Search：外部请求体到内部格式的转换

[public/rows.ts 控制器](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/public/rows.ts)

### 4.1 外部输入格式（Public API）

```json
{
  "query": { ... },
  "paginate": true,
  "bookmark": "some-bookmark",
  "limit": 100,
  "sort": {
    "column": "name",
    "order": "ascending",
    "type": "string"
  }
}
```

由 `externalSearchValidator()` 进行 Joi 验证，定义在 [validators.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/utils/validators.ts#L215-L231)。

### 4.2 buildSearchRequestBody 转换逻辑

```typescript
function buildSearchRequestBody(ctx: UserCtx) {
  let { sort, paginate, bookmark, limit, query } = ctx.request.body
  if (!sort) {
    sort = {}
  }
  return {
    sort: sort.column,      // 嵌套对象 → 扁平字段
    sortType: sort.type,    // 新增 sortType 字段
    sortOrder: sort.order,  // 重命名 order → sortOrder
    bookmark: convertBookmark(bookmark),  // 数字字符串转数字
    paginate,
    limit,
    query,
  }
}
```

### 4.3 convertBookmark 函数

[utilities/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/utilities/index.ts#L101-L107)

```typescript
export function convertBookmark(bookmark: string) {
  const IS_NUMBER = /^\d+\.?\d*$/
  if (typeof bookmark === "string" && bookmark.match(IS_NUMBER)) {
    return parseFloat(bookmark)
  }
  return bookmark
}
```

作用：如果 bookmark 是数字格式的字符串，转换为数字类型，否则保持原样。

### 4.4 search 控制器调用流程

```typescript
export async function search(ctx: UserCtx, next: Next) {
  ctx.request.body = buildSearchRequestBody(ctx)  // 转换请求体
  await rowController.search(ctx)                 // 调用内部 row 控制器
  await next()                                    // 继续执行输出中间件
}
```

---

## 5. update / destroy：addRev 与 fixRow 补全字段

### 5.1 fixRow 函数

[public/rows.ts 控制器](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/public/rows.ts#L9-L23)

```typescript
export function fixRow(row: Row, params: any) {
  if (!params || !row) {
    return row
  }
  if (params.rowId) {
    row._id = params.rowId       // URL 中的 rowId → row._id
  }
  if (params.tableId) {
    row.tableId = params.tableId // URL 中的 tableId → row.tableId
  }
  if (!row.type) {
    row.type = "row"             // 默认为 "row" 类型
  }
  return row
}
```

作用：从 URL params 中提取 rowId 和 tableId，补全到 row 对象中，确保内部控制器能正确处理。

### 5.2 addRev 函数

[public/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/public/utils.ts#L6-L23)

```typescript
export async function addRev(
  body: { _id?: string; _rev?: string },
  tableId?: string
): Promise<Row> {
  if (!body._id || (tableId && isExternalTableID(tableId))) {
    return body  // 外部表或无 _id 时跳过
  }
  let id = body._id
  if (body._id.startsWith(WORKSPACE_PREFIX)) {
    id = DocumentType.WORKSPACE_METADATA
  }
  const db = context.getWorkspaceDB()
  const dbDoc = await db.get<any>(id)
  body._rev = dbDoc._rev  // 从数据库获取最新 _rev
  body._id = id           // 更新 _id（可能是 workspace metadata ID）
  return body
}
```

作用：
- 根据 `_id` 从数据库读取当前文档，获取最新的 `_rev` 值
- 对于 CouchDB 操作，更新必须携带正确的 `_rev` 才能成功
- 外部表（`isExternalTableID`）跳过此步骤

### 5.3 update 端点流程

```typescript
export async function update(ctx: UserCtx, next: Next) {
  const { tableId } = ctx.params
  ctx.request.body = await addRev(
    fixRow(ctx.request.body, ctx.params),  // 先 fixRow 补 _id/tableId/type
    tableId                                // 再 addRev 补 _rev
  )
  await rowController.save(ctx)
  await next()
}
```

### 5.4 destroy 端点流程

```typescript
export async function destroy(ctx: UserCtx, next: Next) {
  const { tableId } = ctx.params
  ctx.request.body = await addRev(
    fixRow({ _id: ctx.params.rowId }, ctx.params),  // 构造只含 _id 的 body
    tableId                                         // 补 _rev
  )
  await rowController.destroy(ctx)
  ctx.body = ctx.row  // destroy 控制器不返回 body，从 ctx.row 取
  await next()
}
```

> **注意**：destroy 操作中，先构造一个只有 `_id` 的对象，通过 fixRow 补全 tableId/type，再通过 addRev 从数据库获取 `_rev`，最后交给内部 `rowController.destroy` 处理。

---

## 6. authorized 中间件：resource role 与 base permission 验证

[authorized.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/middleware/authorized.ts)

### 6.1 中间件签名

```typescript
const authorized = (
  permType: PermissionType,
  permLevel?: PermissionLevel,
  opts = { schema: false },
  resourcePath?: string
) => async (ctx: UserCtx, next: any) => { ... }
```

### 6.2 整体执行流程

1. **特殊豁免**：webhook 端点或内部请求（`ctx.internal`）直接跳过
2. **用户存在性检查**：无 `ctx.user` 时抛 401
3. **获取资源角色**：
   - 如果有 `resourceId`（由 paramResource/paramSubResource 设置），调用 `sdk.permissions.getResourcePerms(resourceId)` 获取资源级权限
   - 同样处理 `subResourceId`
   - 提取当前 `permLevel`（READ/WRITE）对应的角色数组
4. **公开资源检查**：如果资源角色包含 `PUBLIC`，直接放行
5. **认证检查**：未认证时抛 401
6. **Builder API 检查**：如果是 BUILDER / CREATOR / GLOBAL_BUILDER 类型，调用 `builderMiddleware`
7. **授权检查**：调用 `checkAuthorized`
8. **CSRF 保护**：最后执行 CSRF 中间件

### 6.3 checkAuthorized 函数

```typescript
const checkAuthorized = async (
  ctx: UserCtx,
  resourceRoles: any,
  permType: PermissionType,
  permLevel: PermissionLevel
) => {
  // Builder 类型 API：检查用户是否是对应级别的 builder
  // 不是对应 builder → 403
  
  // 非 builder 用户 → 检查资源授权
  if (!isBuilder) {
    await checkAuthorizedResource(ctx, resourceRoles, permType, permLevel)
  }
}
```

### 6.4 checkAuthorizedResource：resource role vs base permission

```typescript
const checkAuthorizedResource = async (
  ctx: UserCtx,
  resourceRoles: any,
  permType: PermissionType,
  permLevel: PermissionLevel
) => {
  const roleId = ctx.roleId || roles.BUILTIN_ROLE_IDS.PUBLIC
  const userRoles = await roles.getUserRoleHierarchy(roleId)
  
  if (resourceRoles.length > 0) {
    // 方式一：Resource Role（资源角色）
    // 用户角色与资源角色有交集 → 通过
    const found = userRoles.find(
      (role: any) => resourceRoles.indexOf(role._id) !== -1
    )
    if (!found) {
      ctx.throw(403, permError)
    }
  } else if (
    // 方式二：Base Permission（基础权限）
    !permissions.doesHaveBasePermission(permType, permLevel, userRoles)
  ) {
    ctx.throw(403, permError)
  }
}
```

**验证优先级**：

1. **Resource Role 优先**：如果资源配置了自定义角色（`resourceRoles.length > 0`），只检查用户是否拥有这些资源角色之一
2. **Base Permission 兜底**：如果资源没有配置自定义角色（使用默认权限），则通过 `permissions.doesHaveBasePermission(permType, permLevel, userRoles)` 检查用户角色是否具备该类型、该级别的基础权限

对于 rows 端点（`permType = PermissionType.TABLE`）：
- 读操作：`authorized(PermissionType.TABLE, PermissionLevel.READ)`
- 写操作：`authorized(PermissionType.TABLE, PermissionLevel.WRITE)`

---

## 7. mapperMiddleware：URL 判断与响应体映射

[mapper.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/public/middleware/mapper.ts)

### 7.1 中间件入口

```typescript
export default async (ctx: Ctx, next: () => Promise<void>) => {
  if (!ctx.body || noResponse(ctx) || isAttachment(ctx)) {
    return await next()  // 空响应或附件直接跳过
  }
  let urlParts = ctx.url.split("/")
  urlParts = urlParts.slice(4, urlParts.length)  // 去掉 /api/public/v1/
  // 根据 urlParts[0] 判断资源类型，调用对应的处理函数
  switch (urlParts[0]) {
    case Resource.TABLES:
      if (urlParts[2] === Resource.ROWS) {
        body = processRows(ctx)  // tables/:tableId/rows/...
      } else {
        body = processTables(ctx)
      }
      break
    // ... 其他资源类型
  }
  ctx.body = body
  await next()
}
```

### 7.2 URL 判断逻辑

URL 解析：`/api/public/v1/tables/:tableId/rows/search` → 去掉前 4 段 → `["tables", ":tableId", "rows", "search"]`

- `urlParts[0]` = `"tables"` → 进入 TABLES 分支
- `urlParts[2]` = `"rows"` → 调用 `processRows(ctx)`

判断是否为数组响应（`isArrayResponse`）：

```typescript
function isArrayResponse(ctx: Ctx) {
  return ctx.url.endsWith(Resource.SEARCH) || Array.isArray(ctx.body)
}
```

- URL 以 `"/search"` 结尾 → 数组响应（搜索结果）
- `ctx.body` 本身是数组 → 数组响应
- 否则 → 单对象响应

### 7.3 processRows 分发

```typescript
function processRows(ctx: Ctx) {
  if (isArrayResponse(ctx)) {
    return mapping.mapRowSearch(ctx)  // 搜索结果 → 列表映射
  } else {
    return mapping.mapRow(ctx)        // 单行 → 对象映射
  }
}
```

### 7.4 mapRowSearch：rows → data，删除 _rev

[mapping/rows.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/public/mapping/rows.ts)

```typescript
function row(body: any): RequiredKeys<Row> {
  delete body._rev  // 删除内部字段 _rev
  return {
    ...body,
    _id: body._id,
    tableId: body.tableId,
  }
}

function mapRowSearch(ctx: any): RowSearch {
  const rows = ctx.body.rows.map((body: any) => row(body))
  return {
    data: rows,           // rows → data
    hasNextPage: ctx.body.hasNextPage,
    bookmark: ctx.body.bookmark,
  }
}
```

**内部格式 → 外部格式映射**：

| 内部字段 | 外部字段 | 说明 |
|----------|----------|------|
| `ctx.body.rows` | `data` | 重命名，且每个 row 删除 `_rev` |
| `ctx.body.hasNextPage` | `hasNextPage` | 原样保留 |
| `ctx.body.bookmark` | `bookmark` | 原样保留 |

### 7.5 mapRow：单行映射

```typescript
function mapRow(ctx: any): { data: Row } {
  return {
    data: row(ctx.body),  // 包装在 data 中，删除 _rev
  }
}
```

对于 create / update / destroy / read 等返回单行的端点，响应体被包装为 `{ data: row }` 格式，且 row 中不包含 `_rev`。

---

## 8. 完整调用链汇总

### 8.1 POST /api/public/v1/tables/:tableId/rows/search

| 阶段 | 组件 | 操作 |
|------|------|------|
| 路由级 | rate limiter | 限流检查（非 dev 环境） |
| 路由级 | CORS | 跨域处理 |
| 输入 | externalSearchValidator | Joi 验证请求体格式 |
| 输入 | publicApiMiddleware | 验证 API key + appId |
| 输入 | paramSubResource | 从 params 提取 resourceId=tableId, subResourceId=rowId |
| 输入 | authorized | TABLE + READ 权限校验（resource role 或 base permission） |
| 控制器 | search (public) | `buildSearchRequestBody` 转换 sort/bookmark 等格式 |
| 控制器 | rowController.search | 内部搜索逻辑 |
| 输出 | mapperMiddleware | URL 判断 → processRows → mapRowSearch → rows→data，删 _rev |

### 8.2 PUT /api/public/v1/tables/:tableId/rows/:rowId

| 阶段 | 组件 | 操作 |
|------|------|------|
| 路由级 | rate limiter | 限流检查（非 dev 环境） |
| 路由级 | CORS | 跨域处理 |
| 输入 | publicApiMiddleware | 验证 API key + appId |
| 输入 | paramSubResource | 从 params 提取 resourceId=tableId, subResourceId=rowId |
| 输入 | authorized | TABLE + WRITE 权限校验（resource role 或 base permission） |
| 控制器 | update (public) | `fixRow` 补 _id/tableId/type，`addRev` 从 DB 取 _rev |
| 控制器 | rowController.save | 内部保存逻辑 |
| 输出 | mapperMiddleware | URL 判断 → processRows → mapRow → 包装 data，删 _rev |
