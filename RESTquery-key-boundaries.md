# REST Query 关键边界追踪文档

## 目录

1. [路由权限差异](#1-路由权限差异)
2. [import 流程：无 datasourceId 时的 REST datasource 构造](#2-import-流程无-datasourceid-时的-rest-datasource-构造)
3. [save 流程：queryId 生成、nullDefaultSupport 与事件](#3-save-流程queryid-生成nulldefaultsupport-与事件)
4. [execute 流程：从 DB 取数到响应输出](#4-execute-流程从-db-取数到响应输出)

---

## 1. 路由权限差异

### 路由分组概览

在 [query.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/query.ts) 中，query 相关路由被划分到三个不同的权限组：

| 路由组 | 权限中间件 | 权限级别 | 包含端点 |
|--------|-----------|---------|---------|
| `builderRoutes` | `authorized(PermissionType.BUILDER)` | BUILDER 角色 | `save`、`preview`、`import`、`importInfo`、`fetch`、`destroy` |
| `writeRoutes` | `authorized(PermissionType.QUERY, PermissionLevel.WRITE)` | QUERY 写权限 | `executeV1`、`executeV2` |
| `readRoutes` | `authorized(PermissionType.QUERY, PermissionLevel.READ)` | QUERY 读权限 | `find` |

### builderRoutes 定义

`builderRoutes` 在 [standard.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/endpointGroups/standard.ts#L11-L14) 中定义：

```typescript
export const builderRoutes = endpointGroupList.group(
  authorized(permissions.BUILDER)
)
builderRoutes.lockMiddleware()
```

它要求用户具有 **BUILDER 角色**，属于较高级别的权限控制，用于应用构建者操作。

### 各端点权限对比

| 端点 | 路由组 | 权限要求 | 用途 |
|------|--------|---------|------|
| `POST /api/queries` (save) | builderRoutes | BUILDER | 保存/创建 query |
| `POST /api/queries/preview` | builderRoutes | BUILDER | 预览 query 执行结果 |
| `POST /api/queries/import` | builderRoutes | BUILDER | 导入 REST query（OpenAPI/cURL） |
| `POST /api/queries/import/info` | builderRoutes | BUILDER | 获取导入信息（预览） |
| `POST /api/queries/:queryId` (executeV1) | writeRoutes | QUERY WRITE | 执行 query（旧版，仅返回 rows） |
| `POST /api/v2/queries/:queryId` (executeV2) | writeRoutes | QUERY WRITE | 执行 query（新版，返回完整结构） |
| `GET /api/queries/:queryId` (find) | readRoutes | QUERY READ | 获取单个 query 详情 |
| `GET /api/queries` (fetch) | builderRoutes | BUILDER | 获取所有 query 列表 |
| `DELETE /api/queries/:queryId/:revId` | builderRoutes | BUILDER | 删除 query |

**关键差异总结**：
- **save/preview/import**：需要 BUILDER 角色，属于应用构建操作
- **executeV1/executeV2**：仅需要 QUERY 写权限，运行时用户也可调用
- **find**：仅需要 QUERY 读权限，读取门槛最低

---

## 2. import 流程：无 datasourceId 时的 REST datasource 构造

### 核心逻辑位置

import 的主逻辑位于 [query/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/query/index.ts#L77-L142) 的 `_import` 函数中。

### 流程步骤

当请求中没有 `datasourceId` 时，系统会自动创建一个新的 REST datasource：

#### 步骤 1：判断是否需要新建 datasource

```typescript
let datasourceId
if (!body.datasourceId) {
  // 构造新的 REST datasource 并保存
  // ...
} else {
  // 使用已有的 datasource
  datasourceId = body.datasourceId
}
```

#### 步骤 2：构造 Datasource 对象

从请求体中解构 datasource（如果有），丢弃 `_id` 和 `_rev`，然后构造新的 datasource：

```typescript
const {
  _id: _discardId,
  _rev: _discardRev,
  config: suppliedConfig,
  ...rest
} = body.datasource ? cloneDeep(body.datasource) : ({} as Datasource)

const config = suppliedConfig || {}
const datasource: Datasource = {
  ...rest,
  type: "datasource",
  source: rest.source || SourceName.REST,  // 默认 REST 类型
  name: rest.name || importInfo?.name,       // 从导入信息获取名称
  config: {
    ...config,
    defaultHeaders: config.defaultHeaders ?? {},        // 默认空 headers
    rejectUnauthorized: config.rejectUnauthorized ?? true, // 默认校验证书
    downloadImages: config.downloadImages ?? true,       // 默认下载图片
    url: config.url ?? importInfo?.url,                  // 从导入信息获取 URL
  },
}
```

**默认值说明**：
- `source`：默认为 `SourceName.REST`
- `name`：优先使用请求体中的 name，否则使用导入信息中的 name
- `defaultHeaders`：默认为空对象 `{}`
- `rejectUnauthorized`：默认为 `true`
- `downloadImages`：默认为 `true`
- `url`：优先使用请求体 config 中的 url，否则使用导入信息中的 url

#### 步骤 3：调用 importer.prepareDatasourceConfig

调用 [import/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/query/import/index.ts#L332-L363) 中的 `prepareDatasourceConfig` 方法，填充静态变量和安全头：

```typescript
importer.prepareDatasourceConfig(datasource)
```

该方法会：
- 填充 `staticVariables`（从 OpenAPI spec 的 server variables 中提取）
- 填充 `templateStaticVariables`（标记哪些变量是模板变量）
- 填充 `defaultHeaders`（从 OpenAPI spec 的 securitySchemes 中提取安全头）

#### 步骤 4：构造上下文并调用 saveDatasource

构造一个新的 `UserCtx`，将 datasource 放入 request body 中，然后调用 `saveDatasource`：

```typescript
const datasourceCtx: UserCtx<CreateDatasourceRequest> = merge(ctx, {
  request: {
    body: {
      datasource,
      tablesFilter: [],
    },
  },
})
await saveDatasource(datasourceCtx)
datasourceId = datasourceCtx.body.datasource._id
```

`saveDatasource` 来自 [datasource.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/datasource.ts#L272-L292)，最终调用 `sdk.datasources.save` 完成 datasource 的持久化。

#### 步骤 5：导入 queries

获取到 datasourceId 后，调用 `importer.importQueries` 导入具体的 query：

```typescript
importResult = await importer.importQueries(
  datasourceId,
  body.selectedEndpointId
)
```

在 [RestImporter.importQueries](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/query/import/index.ts#L256-L330) 中：
- 从导入源（OpenAPI/cURL）生成 query 列表
- 使用 `queryValidation()` 验证每个 query
- 为每个有效 query 生成 `_id`（通过 `generateQueryID`）
- 使用 `db.bulkDocs` 批量保存
- 触发 `events.query.imported` 和每个 query 的 `events.query.created` 事件

---

## 3. save 流程：queryId 生成、nullDefaultSupport 与事件

### 核心逻辑位置

save 的主逻辑位于 [query/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/query/index.ts#L169-L200) 的 `save` 函数中。

### 流程步骤

#### 步骤 1：验证 query 名称

```typescript
if (!query?.name.match(ValidQueryNameRegex)) {
  ctx.throw(400, "Invalid query name")
}
```

#### 步骤 2：生成 queryId（新建时）

通过判断 `_id` 和 `_rev` 是否都不存在来区分是新建还是更新：

```typescript
if (!query._id && !query._rev) {
  query._id = generateQueryID(query.datasourceId)
  // ...
}
```

`generateQueryID` 函数在 [db/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/db/utils.ts#L212-L216) 中定义：

```typescript
export function generateQueryID(datasourceId: string) {
  return `${
    DocumentType.QUERY
  }${SEPARATOR}${datasourceId}${SEPARATOR}${newid()}`
}
```

ID 格式：`query_<datasourceId>_<随机ID>`

#### 步骤 3：设置 nullDefaultSupport

`nullDefaultSupport` 标志用于控制参数默认值是为空字符串（旧行为）还是 `null`（新行为）。

**新建 query 时**：
```typescript
query.nullDefaultSupport = true
```

**更新 query 时**：
```typescript
const existingQuery = await db.get<Query>(query._id)
if (existingQuery.nullDefaultSupport && query.nullDefaultSupport == null) {
  query.nullDefaultSupport = true
}
```

规则：
- 新建 query：默认设为 `true`
- 更新 query：如果已有 flag 为 true 且请求中未设置，则保持为 true（防止降级）
- 允许通过 API 显式设置为 `false`

#### 步骤 4：确定事件函数

根据是新建还是更新，选择不同的事件函数：

```typescript
let eventFn
if (!query._id && !query._rev) {
  // ...
  eventFn = () => events.query.created(datasource, query)
} else {
  // ...
  eventFn = () => events.query.updated(datasource, query)
}
```

事件发布者定义在 [backend-core/src/events/publishers/query.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/events/publishers/query.ts) 中。

#### 步骤 5：保存到数据库并触发事件

```typescript
const response = await db.put(query)
await eventFn()
query._rev = response.rev
```

注意：事件是**在 db.put 成功之后**才触发的，确保数据已持久化。

---

## 4. execute 流程：从 DB 取数到响应输出

### 核心逻辑位置

execute 的主逻辑位于 [query/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/query/index.ts#L415-L480) 的 `execute` 函数中。

### 流程步骤

#### 步骤 1：从 DB 获取 query 和 datasource + envVars

```typescript
const db = context.getWorkspaceDB()
const query = await db.get<Query>(ctx.params.queryId)
const { datasource, envVars } = await sdk.datasources.getWithEnvVars(
  query.datasourceId
)
```

`getWithEnvVars` 在 [datasources.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/datasources/datasources.ts#L188-L192) 中定义：
- 首先从 DB 获取 datasource
- 然后调用 `enrichDatasourceWithValues` 填充环境变量值

#### 步骤 2：获取认证配置（非 automation 时）

```typescript
let authConfigCtx = {}
if (!opts.isAutomation) {
  authConfigCtx = getAuthConfig(ctx)
}
```

`getAuthConfig` 会从 cookie 中提取 sessionId 和 OIDC 配置 ID。

#### 步骤 3：验证并填充参数（enrichParameters）

`enrichParameters` 函数在 [query/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/query/index.ts#L222-L241) 中定义，包含两个关键操作：

##### 3.1 拒绝 handlebars 绑定

`validateQueryInputs` 函数检查参数中是否包含 handlebars 绑定（`{{ ... }}`），如果有则抛出错误：

```typescript
function validateQueryInputs(parameters: QueryEventParameters) {
  for (let entry of Object.entries(parameters)) {
    const [key, value] = entry
    if (typeof value !== "string") {
      continue
    }
    if (findHBSBlocks(value).length !== 0) {
      throw new Error(
        `Parameter '${key}' input contains a handlebars binding - this is not allowed.`
      )
    }
  }
}
```

**安全边界**：防止用户通过参数注入 handlebars 模板来执行任意代码或访问敏感数据。

##### 3.2 填充默认参数

```typescript
for (const parameter of query.parameters) {
  let value = requestParameters[parameter.name]
  if (value == null || value === "") {
    value = parameter.default
  }
  if (query.nullDefaultSupport && paramNotSet(value)) {
    value = null
  }
  requestParameters[parameter.name] = value
}
```

规则：
- 如果请求参数为空或未提供，使用 parameter.default
- 如果启用了 `nullDefaultSupport` 且值未设置（空字符串或 undefined），则设为 `null`

#### 步骤 4：构造执行输入并运行

```typescript
const inputs: QueryEvent = {
  appId: ctx.appId,
  datasource,
  queryVerb: query.queryVerb,
  fields: query.fields,
  pagination: ctx.request.body.pagination,
  parameters: enrichParameters(query, ctx.request.body.parameters),
  transformer: query.transformer,
  queryId: ctx.params.queryId,
  environmentVariables: envVars,
  nullDefaultSupport: query.nullDefaultSupport,
  ctx: {
    user: sanitiseUserStructure(ctx.user),
    auth: { ...authConfigCtx },
  },
  schema: query.schema,
}
```

#### 步骤 5：条件性 quota 包装（quotas.addAction）

根据 queryVerb 和 isAutomation 决定是否通过 quota 包装：

```typescript
const { rows, pagination, extra, info } =
  query.queryVerb === "read" || opts.isAutomation
    ? await Runner.run<QueryResponse>(inputs)
    : await quotas.addAction(ActionType.CRUD, async () => {
        const response = await Runner.run<QueryResponse>(inputs)
        events.action.crudExecuted({ type: query.queryVerb })
        return response
      })
```

**关键边界条件**：
- **read 操作**：不经过 quota，直接执行
- **automation 调用**：不经过 quota（`executeV2AsAutomation`）
- **非 read 且非 automation**：通过 `quotas.addAction(ActionType.CRUD, ...)` 包装，受配额限制
- 非 read 操作成功后会触发 `events.action.crudExecuted` 事件

#### 步骤 6：删除 extra.raw

```typescript
if (extra?.raw) {
  delete extra.raw
}
```

**安全边界**：移除原始响应数据，防止 transformer 用于隐藏数据时原始数据泄露。raw 数据可能包含敏感信息或未经过滤的数据。

#### 步骤 7：构造响应

根据 `rowsOnly` 参数返回不同格式：

```typescript
if (opts && opts.rowsOnly) {
  ctx.body = rows              // executeV1：仅返回 rows 数组
} else {
  ctx.body = { data: rows, pagination, ...extra, ...info }  // executeV2：完整结构
}
```

### executeV1 与 executeV2 的区别

| 版本 | rowsOnly | 响应格式 | 备注 |
|------|----------|---------|------|
| executeV1 | `true` | 直接返回 rows 数组 | 已废弃（DEPRECATED） |
| executeV2 | `false` | `{ data, pagination, ...extra, ...info }` | 推荐使用 |

```typescript
export async function executeV1(ctx) {
  return execute(ctx, { rowsOnly: true, isAutomation: false })
}

export async function executeV2(ctx) {
  return execute(ctx, { rowsOnly: false, isAutomation: false })
}
```

---

## 附录：关键文件索引

| 文件 | 说明 |
|------|------|
| [query.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/query.ts) | 路由定义与权限分组 |
| [query/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/query/index.ts) | query 控制器（save/preview/execute/import） |
| [query/import/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/query/import/index.ts) | REST query 导入器 |
| [datasource.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/datasource.ts) | datasource 控制器（save） |
| [db/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/db/utils.ts) | generateQueryID 等工具函数 |
| [standard.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/endpointGroups/standard.ts) | builderRoutes 等路由组定义 |
| [datasources.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/datasources/datasources.ts) | datasource SDK（getWithEnvVars） |
