# DataProvider 数据请求与分页机制分析

本文档详细分析 Budibase 中 `DataProvider` 组件在 table datasource 和 query datasource 两种场景下，如何触发后端请求并维护分页状态。

## 目录

1. [DataProvider.svelte 核心逻辑](#1-dataprovidersvelte-核心逻辑)
2. [fetchData 与 DataFetchMap 选择机制](#2-fetchdata-与-datafetchmap-选择机制)
3. [BaseDataFetch.getInitialData 初始化流程](#3-basedatafetchgetinitialdata-初始化流程)
4. [TableFetch 表格数据源](#4-tablefetch-表格数据源)
5. [QueryFetch 查询数据源](#5-queryfetch-查询数据源)
6. [API Client 请求头处理机制](#6-api-client-请求头处理机制)

---

## 1. DataProvider.svelte 核心逻辑

[DataProvider.svelte](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/client/src/components/app/DataProvider.svelte) 是客户端数据提供的核心组件，负责协调数据获取、分页、过滤和自动刷新。

### 1.1 核心依赖

```svelte
import { fetchData, QueryUtils } from "@budibase/frontend-core"
import { createAutoRefresh } from "@/utils/autoRefresh"
const { styleable, Provider, ActionTypes, API, builderStore } = getContext("sdk")
```

### 1.2 QueryUtils.buildQuery 构建查询

`QueryUtils` 实际是 `@budibase/shared-core` 中的 `dataFilters`，在 [frontend-core/src/utils/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/utils/index.ts#L1) 中重新导出。

[buildQuery](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/shared-core/src/filters.ts#L461-L521) 函数将 `UISearchFilter`（UI 层的过滤器定义）转换为 `SearchFilters`（后端可识别的查询结构）：

- 输入：`UISearchFilter` 或旧版 `LegacyFilter[]`
- 输出：`SearchFilters`（包含逻辑运算符、条件组、空过滤器行为等）
- 支持递归嵌套的逻辑组（AND/OR）
- 默认空过滤行为：`EmptyFilterOption.RETURN_ALL`

```typescript
// DataProvider.svelte 第 38 行
$: defaultQuery = QueryUtils.buildQuery(filter)
```

### 1.3 queryExtensions 查询扩展机制

DataProvider 支持通过 `queryExtensions` 扩展查询条件，允许子组件向数据提供者添加额外过滤条件。

**核心方法：**

- [addQueryExtension](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/client/src/components/app/DataProvider.svelte#L140-L145)：添加扩展
- [removeQueryExtension](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/client/src/components/app/DataProvider.svelte#L147-L154)：移除扩展
- [extendQuery](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/client/src/components/app/DataProvider.svelte#L156-L177)：合并默认查询与扩展

**扩展逻辑：**
- 使用 `LogicalOperator.AND` 将默认查询与所有扩展合并
- 如果没有任何条件，返回空对象
- 空过滤器行为设为 `EmptyFilterOption.RETURN_NONE`

```typescript
const extendQuery = (defaultQuery, extensions) => {
  if (!Object.keys(extensions).length) {
    return defaultQuery
  }
  const extended = {
    [LogicalOperator.AND]: {
      conditions: [
        ...(defaultQuery ? [defaultQuery] : []),
        ...Object.values(extensions || {}),
      ],
    },
    onEmptyFilter: EmptyFilterOption.RETURN_NONE,
  }
  return (extended[LogicalOperator.AND]?.conditions?.length ?? 0) > 0
    ? extended
    : {}
}
```

### 1.4 createAutoRefresh 自动刷新

[createAutoRefresh](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/client/src/utils/autoRefresh.ts) 创建一个自动刷新管理器：

```typescript
export const createAutoRefresh = () => {
  let interval: ReturnType<typeof setInterval>

  const setUp = (autoRefresh, callback) => {
    clearInterval(interval)
    if (autoRefresh && callback) {
      // 最小间隔 10 秒
      interval = setInterval(callback, Math.max(10000, autoRefresh * 1000))
    }
  }

  const clear = () => {
    clearInterval(interval)
  }

  return { setUp, clear }
}
```

**使用方式：**
- 在 builder 中或未选中组件时禁用自动刷新
- 组件销毁时调用 `clear()` 清理定时器
- 刷新回调为 `fetch.refresh`

```typescript
// DataProvider.svelte 第 52-57 行
$: autoRefreshEnabled =
  !$builderStore.inBuilder || !$builderStore.selectedComponentId
$: autoRefreshActions.setUp(
  autoRefreshEnabled ? autoRefresh : null,
  fetch.refresh
)
```

### 1.5 ActionTypes 动作类型

[ActionTypes](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/client/src/constants.ts#L3-L16) 定义了 DataProvider 可响应的动作类型：

| 动作类型 | 作用 |
|---------|------|
| `RefreshDatasource` | 刷新数据源，调用 `fetch.refresh()` |
| `AddDataProviderQueryExtension` | 添加查询扩展 |
| `RemoveDataProviderQueryExtension` | 移除查询扩展 |
| `SetDataProviderSorting` | 设置排序，调用 `fetch.update()` |

这些动作通过 [Provider](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/client/src/components/context/Provider.svelte) 上下文传递给子组件。

### 1.6 fetch 实例的创建与更新

DataProvider 通过响应式声明管理 fetch 实例：

```svelte
$: fetch = createFetch(dataSource)
$: fetch.update({
    query,
    sortColumn,
    sortOrder,
    limit,
    paginate,
  })
```

`createFetch` 函数调用 `fetchData` 创建具体的 fetch 实例，并传入初始选项。当依赖变化时，`fetch.update()` 会检测选项是否真的变化，只有变化时才重新加载数据。

---

## 2. fetchData 与 DataFetchMap 选择机制

### 2.1 DataFetchMap 映射表

[DataFetchMap](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/fetch/index.ts#L18-L33) 定义了数据源类型到 Fetch 类的映射：

```typescript
export const DataFetchMap = {
  table: TableFetch,
  view: ViewFetch,
  viewV2: ViewV2Fetch,
  query: QueryFetch,
  link: RelationshipFetch,
  user: UserFetch,
  groupUser: GroupUserFetch,
  custom: CustomFetch,
  provider: NestedProviderFetch,
  field: FieldFetch,
  jsonarray: JSONArrayFetch,
  queryarray: QueryArrayFetch,
}
```

### 2.2 fetchData 工厂函数

[fetchData](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/fetch/index.ts#L79-L100) 是创建 fetch 实例的工厂函数：

```typescript
export const fetchData = ({ API, datasource, options }) => {
  const Fetch = DataFetchMap[datasource?.type] || TableFetch
  const fetch = new Fetch({ API, datasource, ...options })

  // 立即触发初始数据获取，不等待结果
  fetch.getInitialData()

  return fetch
}
```

**关键点：**
- 根据 `datasource.type` 从 `DataFetchMap` 选择对应的 Fetch 类
- 默认回退到 `TableFetch`
- 创建实例后**立即调用 `getInitialData()`**，异步加载初始数据
- 返回的 fetch 实例是 Svelte store，可通过 `$fetch` 订阅状态

---

## 3. BaseDataFetch.getInitialData 初始化流程

[BaseDataFetch](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/fetch/DataFetch.ts) 是所有 Fetch 类的抽象基类，定义了通用的数据获取和分页逻辑。

### 3.1 构造函数初始化

构造函数中初始化：
- **features**：功能标志（搜索、排序、分页支持）
- **options**：配置选项（limit、query、sortColumn、sortOrder、paginate 等）
- **store**：Svelte writable store，存储数据状态
- **derivedStore**：派生 store，计算 hasNextPage、hasPrevPage 等

### 3.2 getInitialData 完整流程

[getInitialData](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/fetch/DataFetch.ts#L201-L291) 是数据初始化的核心方法，执行以下步骤：

#### 步骤 1：获取数据源定义

```typescript
const definition = await this.getDefinition()
```

调用子类实现的 `getDefinition()` 获取数据源元数据。

#### 步骤 2：确定功能标志

```typescript
const features = await this.determineFeatureFlags()
this.features = {
  supportsSearch: !!features?.supportsSearch,
  supportsSort: !!features?.supportsSort,
  supportsPagination: paginate && !!features?.supportsPagination,
}
```

根据数据源类型确定是否支持服务端搜索、排序和分页。

#### 步骤 3：获取并丰富 Schema

```typescript
let schema = this.getSchema(definition)
if (!schema) {
  return
}
schema = this.enrichSchema(schema)
```

- `getSchema()` 从 definition 中提取 schema
- `enrichSchema()` 添加 JSON 字段的嵌套属性，确保 schema 结构正确

#### 步骤 4：确定排序列

```typescript
// 验证排序列是否有效
if (this.options.sortColumn && !schema[this.options.sortColumn]) {
  this.options.sortColumn = null
}

// 获取默认排序列
if (!this.options.sortColumn) {
  this.options.sortColumn = this.getDefaultSortColumn(definition, schema)
}
```

[getDefaultSortColumn](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/fetch/DataFetch.ts#L187-L196) 的逻辑：
- 优先使用 `definition.primaryDisplay`（如果存在且在 schema 中有效）
- 否则使用 schema 的第一个字段

#### 步骤 5：确定排序类型

```typescript
if (!this.options.sortColumn) {
  this.options.sortOrder = SortOrder.ASCENDING
  this.options.sortType = null
} else {
  this.options.sortType = SortType.STRING
  const fieldSchema = schema?.[this.options.sortColumn]
  if (
    fieldSchema?.type === FieldType.NUMBER ||
    fieldSchema?.type === FieldType.BIGINT ||
    fieldSchema?.responseType === FieldType.NUMBER ||
    ("calculationType" in fieldSchema && fieldSchema?.calculationType)
  ) {
    this.options.sortType = SortType.NUMBER
  }

  if (!this.options.sortOrder) {
    this.options.sortOrder = SortOrder.ASCENDING
  } else {
    this.options.sortOrder = this.options.sortOrder.toLowerCase() as SortOrder
  }
}
```

根据字段类型决定是按字符串还是数字排序。

#### 步骤 6：构建查询

```typescript
let query = this.options.query
if (!query) {
  query = buildQuery(filter ?? undefined) as TQuery
}
```

#### 步骤 7：更新 store 并获取第一页数据

```typescript
this.store.update($store => ({
  ...$store,
  definition,
  schema,
  query,
  loading: true,
  cursors: [],
  cursor: null,
}))

const page = await this.getPage()
this.store.update($store => ({
  ...$store,
  loading: false,
  loaded: true,
  pageNumber: 0,
  rows: page.rows,
  info: page.info,
  cursors: paginate && page.hasNextPage ? [null, page.cursor] : [null],
  error: page.error,
  resetKey: Math.random().toString(),
}))
```

**分页游标初始化：**
- `cursors[0]` 始终为 `null`（代表第一页的"前一个游标"）
- 如果有下一页，`cursors[1]` 为第一页返回的 cursor
- `pageNumber` 从 0 开始计数

### 3.3 getPage 获取单页数据

[getPage](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/fetch/DataFetch.ts#L296-L333) 获取单页数据，并根据功能标志决定是否在客户端进行补充处理：

```typescript
async getPage() {
  let { rows, info, hasNextPage, cursor, error } = await this.getData()

  // 不支持服务端搜索时，客户端搜索
  if (!this.features.supportsSearch && clientSideSearching) {
    rows = runQuery(rows, query)
  }

  // 不支持服务端排序时，客户端排序
  if (!this.features.supportsSort && clientSideSorting && sortType) {
    rows = sort(rows, sortColumn, sortOrder, sortType)
  }

  // 不支持服务端分页时，客户端限制
  if (!this.features.supportsPagination && clientSideLimiting) {
    rows = queryLimit(rows, limit)
  }

  return { rows, info, hasNextPage, cursor, error }
}
```

### 3.4 分页状态维护

#### 游标数组 (cursors)

`cursors` 数组是分页状态的核心：
- `cursors[i]` 表示第 `i` 页的**起始游标**
- `cursors[0] = null`（第一页不需要游标）
- `cursors[n]` 是第 n 页的起始游标
- 通过 `cursors[pageNumber + 1]` 判断是否有下一页

#### hasNextPage / hasPrevPage

```typescript
private hasNextPage(state): boolean {
  return state.cursors[state.pageNumber + 1] != null
}

private hasPrevPage(state): boolean {
  return state.pageNumber > 0
}
```

#### nextPage 下一页

```typescript
async nextPage() {
  const nextCursor = state.cursors[state.pageNumber + 1]
  this.store.update($store => ({
    ...$store,
    loading: true,
    cursor: nextCursor,
    pageNumber: $store.pageNumber + 1,
  }))
  const { rows, info, hasNextPage, cursor, error } = await this.getPage()

  this.store.update($store => {
    let { cursors, pageNumber } = $store
    if (hasNextPage) {
      cursors[pageNumber + 1] = cursor
    }
    return { ...$store, rows, info, cursors, loading: false, error }
  })
}
```

#### prevPage 上一页

```typescript
async prevPage() {
  const prevCursor = state.cursors[state.pageNumber - 1]
  this.store.update($store => ({
    ...$store,
    loading: true,
    cursor: prevCursor,
    pageNumber: $store.pageNumber - 1,
  }))
  const { rows, info, error } = await this.getPage()
  // ...
}
```

### 3.5 update 方法更新选项

[update](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/fetch/DataFetch.ts#L426-L449) 方法用于更新配置选项：

```typescript
async update(newOptions) {
  // 检测是否真的变化
  let refresh = false
  for (const [key, value] of Object.entries(newOptions || {})) {
    const oldVal = this.options[key] ?? null
    const newVal = value == null ? null : value
    if (JSON.stringify(newVal) !== JSON.stringify(oldVal)) {
      refresh = true
      break
    }
  }
  if (!refresh) {
    return
  }

  this.options = { ...this.options, ...cloneDeep(newOptions) }
  await this.getInitialData()
}
```

**关键点：**
- 使用 `JSON.stringify` 深度比较选项是否变化
- 只有变化时才重新获取数据
- 重新获取会调用 `getInitialData()`，重置分页状态

### 3.6 refresh 刷新当前页

[refresh](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/fetch/DataFetch.ts#L454-L489) 刷新当前页数据，不重置分页：

```typescript
async refresh() {
  if (get(this.store).loading) {
    return
  }
  this.store.update($store => ({ ...$store, loading: true }))
  const { rows, info, error, cursor } = await this.getPage()

  let { cursors } = get(this.store)
  const { pageNumber } = get(this.store)

  // 如果当前页数据为空且有上一页，回退到上一页
  if (!rows.length && pageNumber > 0) {
    this.store.update($store => ({
      ...$store,
      loading: false,
      cursors: cursors.slice(0, pageNumber),
    }))
    return await this.prevPage()
  }

  // 如果游标变化，标记后续页面为过期
  const currentNextCursor = cursors[pageNumber + 1]
  if (currentNextCursor != cursor) {
    cursors = cursors.slice(0, pageNumber + 1)
    cursors[pageNumber + 1] = cursor
  }

  this.store.update($store => ({
    ...$store,
    rows,
    info,
    loading: false,
    error,
    cursors,
  }))
}
```

---

## 4. TableFetch 表格数据源

[TableFetch](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/fetch/TableFetch.ts) 处理内部表数据的获取。

### 4.1 功能标志

```typescript
async determineFeatureFlags() {
  return {
    supportsSearch: true,
    supportsSort: true,
    supportsPagination: true,
  }
}
```

表格数据源全部支持服务端搜索、排序和分页。

### 4.2 getDefinition 获取表定义

```typescript
async getDefinition() {
  const { datasource } = this.options
  if (!datasource?.tableId) {
    return null
  }
  try {
    return await this.API.fetchTableDefinition(datasource.tableId)
  } catch (error) {
    this.store.update(state => ({ ...state, error }))
    return null
  }
}
```

调用 `API.fetchTableDefinition()` 获取表定义，结果会被缓存。

### 4.3 getData 获取数据

```typescript
async getData() {
  const { datasource, limit, sortColumn, sortOrder, sortType, paginate } =
    this.options
  const { tableId } = datasource
  const { cursor, query } = get(this.store)

  try {
    const res = await this.API.searchTable(tableId, {
      query,
      limit,
      sort: sortColumn,
      sortOrder: sortOrder ?? SortOrder.ASCENDING,
      sortType,
      paginate,
      bookmark: cursor,
    })
    return {
      rows: res?.rows || [],
      hasNextPage: res?.hasNextPage || false,
      cursor: res?.bookmark || null,
    }
  } catch (error) {
    return { rows: [], hasNextPage: false, error }
  }
}
```

**请求参数：**
- `query`：搜索过滤条件（SearchFilters）
- `limit`：每页条数
- `sort`：排序列
- `sortOrder`：排序方向
- `sortType`：排序类型（STRING/NUMBER）
- `paginate`：是否分页
- `bookmark`：分页游标

### 4.4 API.searchTable 调用

[searchTable](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/api/tables.ts#L89-L94) 构造 POST 请求：

```typescript
searchTable: async (sourceId, opts) => {
  return await API.post({
    url: `/api/${sourceId}/search`,
    body: opts,
  })
}
```

**请求路径：** `POST /api/:sourceId/search`

---

## 5. QueryFetch 查询数据源

[QueryFetch](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/fetch/QueryFetch.ts) 处理外部查询（SQL/REST 等）的数据获取。

### 5.1 功能标志

```typescript
async determineFeatureFlags() {
  const definition = await this.getDefinition()
  const supportsPagination =
    (!!definition?.fields?.pagination?.type &&
      !!definition?.fields?.pagination?.location &&
      !!definition?.fields?.pagination?.pageParam) ||
    !!definition?.fields?.pagination?.enabled
  return { supportsPagination }
}
```

**分页支持判断逻辑：**
- SQL 查询：通过 `fields.pagination.enabled` 判断
- REST 查询：需要同时配置 `type`、`location`、`pageParam`
- 搜索和排序：默认不支持服务端（会回退到客户端处理）

### 5.2 getDefinition 获取查询定义

```typescript
async getDefinition() {
  const { datasource } = this.options

  if (!datasource?._id) {
    return null
  }
  try {
    const definition = await this.API.fetchQueryDefinition(datasource._id)
    // 出于安全原因，服务器返回的定义会丢失 "fields" 属性
    // 但分页需要这个属性，所以从 datasource 中补回
    if (!definition.fields) {
      definition.fields = datasource.fields
    }
    return definition
  } catch (error) {
    return null
  }
}
```

**关键细节：**
- 服务器出于安全考虑，返回的查询定义会移除 `fields` 属性（包含敏感配置）
- 客户端从 `datasource.fields` 中补回 `fields`，用于分页配置
- `fields.pagination` 包含分页类型、位置、参数名等配置

### 5.3 getDefaultSortColumn 默认排序列

```typescript
getDefaultSortColumn() {
  return null
}
```

查询数据源没有默认排序列。

### 5.4 getData 获取数据

```typescript
async getData() {
  const { datasource, limit, paginate } = this.options
  const { supportsPagination } = this.features
  const { cursor, definition } = get(this.store)
  const paginationType = definition?.fields?.pagination?.type

  // 步骤 1：设置默认查询参数
  const parameters = Helpers.cloneDeep(datasource.queryParams || {})
  for (const param of datasource?.parameters || []) {
    if (!parameters[param.name]) {
      parameters[param.name] = param.default
    }
  }

  // 步骤 2：构造请求体
  const queryPayload: ExecuteQueryRequest = { parameters }
  if (paginate && supportsPagination) {
    const isPageBased =
      paginationType === "page" || definition?.fields?.pagination?.enabled
    const requestCursor = isPageBased ? parseInt(cursor || "1") : cursor
    queryPayload.pagination = { page: requestCursor, limit }
  }

  // 步骤 3：执行查询
  try {
    const res = await this.API.executeQuery(datasource?._id, queryPayload)
    const { data, pagination, ...rest } = res

    // 步骤 4：解析分页信息
    let nextCursor = null
    let hasNextPage = false
    if (paginate && supportsPagination) {
      const isPageBased =
        paginationType === "page" || definition?.fields?.pagination?.enabled

      if (isPageBased) {
        // 页码式分页：当前页 + 1
        nextCursor = queryPayload.pagination!.page! + 1
        hasNextPage = data?.length === limit && limit > 0
      } else {
        // 游标式分页：从响应中获取
        nextCursor = pagination?.cursor
        hasNextPage = nextCursor != null
      }
    }

    return {
      rows: data || [],
      info: rest,
      cursor: nextCursor,
      hasNextPage,
    }
  } catch (error) {
    return { rows: [], hasNextPage: false }
  }
}
```

#### 参数默认值填充

- 优先使用 `datasource.queryParams` 中的值
- 对于 `datasource.parameters` 中定义的参数，如果没有值，使用 `param.default`

#### 两种分页模式

| 模式 | 适用场景 | 游标类型 | 下一页判断 |
|------|---------|---------|-----------|
| Page-based (页码式) | SQL 查询、REST page 类型 | 数字页码 | 数据长度等于 limit |
| Cursor-based (游标式) | REST cursor 类型 | 字符串游标 | 响应中返回 cursor |

### 5.5 API.executeQuery 调用

[executeQuery](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/api/queries.ts#L43-L51) 构造 POST 请求：

```typescript
executeQuery: async (queryId, { pagination, parameters } = {}) => {
  return await API.post<ExecuteQueryRequest, ExecuteV2QueryResponse>({
    url: `/api/v2/queries/${queryId}`,
    body: {
      parameters,
      pagination,
    },
  })
}
```

**请求路径：** `POST /api/v2/queries/:queryId`

**请求体结构：**
```typescript
interface ExecuteQueryRequest {
  parameters?: Record<string, any>
  pagination?: {
    page: number | string | null
    limit: number
  }
}
```

---

## 6. API Client 请求头处理机制

### 6.1 API Client 架构

API Client 位于 [frontend-core/src/api/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/frontend-core/src/api/index.ts)，通过 `createAPIClient` 工厂函数创建。

**核心结构：**
- `makeApiCall`：底层请求函数
- `requestApiCall`：按 HTTP 方法封装
- `buildXxxEndpoints`：各业务领域端点构造函数
- 返回合并了所有端点的 API 对象

### 6.2 Session ID

```typescript
// frontend-core/src/api/index.ts 第 71 行
export const APISessionID = Helpers.uuid()
```

- 每个浏览器标签页生成一个唯一的 Session ID
- 用于标识请求来源，配合 WebSocket 消息忽略自己触发的事件
- 通过 `x-budibase-session-id` 请求头发送

### 6.3 API Version

```typescript
if (!external) {
  headers[Header.API_VER] = ApiVersion
}
```

- 内部请求附加 `x-budibase-api-version` 头
- 外部请求（`external: true`）不附加
- `ApiVersion` 常量定义在 `frontend-core/src/constants`

### 6.4 App ID

App ID 通过 `attachHeaders` 回调由使用方注入：

```typescript
// client/src/api/api.ts 第 23-25 行
if (window["##BUDIBASE_APP_ID##"]) {
  headers["x-budibase-app-id"] = window["##BUDIBASE_APP_ID##"]
}
```

- 从全局变量 `##BUDIBASE_APP_ID##` 读取
- 通过 `x-budibase-app-id` 请求头发送
- 用于服务端识别当前应用

### 6.5 CSRF Token

```typescript
// client/src/api/api.ts 第 42-45 行
const auth = get(authStore)
if (auth?.csrfToken) {
  headers["x-csrf-token"] = auth.csrfToken
}
```

- 从 `authStore` 中读取 CSRF token
- 仅在用户已认证时附加
- 通过 `x-csrf-token` 请求头发送
- 用于防止跨站请求伪造攻击

### 6.6 其他请求头

| Header | 说明 | 设置位置 |
|--------|------|---------|
| `Accept: application/json` | 接受 JSON 响应 | 基础配置 |
| `Content-Type: application/json` | JSON 请求体 | 有 body 时 |
| `x-budibase-type: client` | 客户端标识 | 非 builder 环境 |
| `x-budibase-embed-location` | 嵌入位置 | 嵌入模式 |
| `x-budibase-role` | 预览角色 | 开发者工具 |

### 6.7 迁移 Header 处理

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

**检测逻辑：**
- 响应头中包含 `x-budibase-migrating-app` 表示应用正在迁移
- `x-budibase-migrating-app-skip-wait` 表示是否跳过等待
- 在客户端中，检测到迁移会触发页面刷新，显示更新中界面

### 6.8 完整请求头示例

一次典型的内部 API 请求会包含以下请求头：

```
Accept: application/json
x-budibase-session-id: <uuid>
x-budibase-api-version: <version>
x-budibase-app-id: <appId>
x-budibase-type: client
Content-Type: application/json
x-csrf-token: <token>  (已认证时)
```

---

## 总结：完整调用链路

### Table Datasource 链路

```
DataProvider.svelte
  ↓ createFetch
fetchData()
  ↓ 根据 datasource.type = "table"
DataFetchMap.table → TableFetch
  ↓ new TableFetch().getInitialData()
BaseDataFetch.getInitialData()
  ├─→ getDefinition() → API.fetchTableDefinition() → GET /api/tables/:tableId
  ├─→ determineFeatureFlags() → { search: true, sort: true, pagination: true }
  ├─→ getSchema()
  ├─→ getDefaultSortColumn()
  └─→ getPage()
      ↓
      TableFetch.getData()
        ↓
        API.searchTable() → POST /api/:sourceId/search
          ↓
          makeApiCall() 附加 sessionId、apiVersion、appId、csrfToken 等
```

### Query Datasource 链路

```
DataProvider.svelte
  ↓ createFetch
fetchData()
  ↓ 根据 datasource.type = "query"
DataFetchMap.query → QueryFetch
  ↓ new QueryFetch().getInitialData()
BaseDataFetch.getInitialData()
  ├─→ getDefinition()
  │    ↓
  │    API.fetchQueryDefinition() → GET /api/queries/:queryId
  │    ↓ 补回 fields
  │    definition.fields = datasource.fields
  ├─→ determineFeatureFlags() → { supportsPagination: boolean }
  ├─→ getSchema()
  ├─→ getDefaultSortColumn() → null
  └─→ getPage()
      ↓
      QueryFetch.getData()
        ├─ 填充参数默认值
        ├─ 构造 ExecuteQueryRequest
        │   ├─ parameters
        │   └─ pagination { page, limit }
        └─→ API.executeQuery() → POST /api/v2/queries/:queryId
              ↓
              makeApiCall() 附加 sessionId、apiVersion、appId、csrfToken 等
```

### 分页状态维护

两种数据源都使用相同的分页机制（基于游标数组）：

| 状态字段 | 说明 |
|---------|------|
| `pageNumber` | 当前页码（从 0 开始） |
| `cursor` | 当前页的起始游标 |
| `cursors` | 所有页的游标数组，cursors[i] 是第 i 页的起始游标 |
| `hasNextPage` | cursors[pageNumber + 1] != null |
| `hasPrevPage` | pageNumber > 0 |

翻页时更新 cursor 并调用 `getPage()`，返回结果后更新 cursors 数组。`update()` 方法会重新调用 `getInitialData()` 重置所有分页状态。
