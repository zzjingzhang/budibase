# POST /api/:sourceId/search 搜索条件构造分析报告

## 概述

本文档分析 Budibase 中 `POST /api/:sourceId/search` 接口在普通 table 和 viewV2 两种 source 下构造最终搜索条件的完整流程，涵盖 route 层验证、controller 层处理、SDK 搜索逻辑、internal SQS 查询等环节，并指出防止用户绕过 view 保存过滤条件的关键分支。

---

## 一、Route 层验证器

### 1.1 internalSearchValidator

**位置**: [validators.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/utils/validators.ts#L197-L213)

这是一个基于 Joi 的请求体验证中间件，验证搜索请求的顶层结构：

```typescript
export function internalSearchValidator() {
  return auth.joiValidator.body(
    Joi.object({
      tableId: OPTIONAL_STRING,
      query: filterObject(),       // 搜索过滤条件
      limit: OPTIONAL_NUMBER,
      sort: OPTIONAL_STRING,
      sortOrder: OPTIONAL_STRING,
      sortType: OPTIONAL_STRING,
      paginate: Joi.boolean(),
      countRows: Joi.boolean(),
      bookmark: Joi.alternatives()
        .try(OPTIONAL_STRING, OPTIONAL_NUMBER)
        .optional(),
    })
  )
}
```

其中 `filterObject()` 通过 `searchFiltersValidator()` 定义了所有允许的过滤操作符：
- 基础操作符：string, fuzzy, range, equal, notEqual, empty, notEmpty
- 数组操作符：oneOf, contains, notContains, containsAny
- 逻辑操作符：$and, $or（通过 conditions 嵌套）
- 元数据字段：allOr, onEmptyFilter
- **禁止字段**: fuzzyOr, documentType（使用 Joi.forbidden()）

### 1.2 searchRowRequestValidator

**位置**: [search.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/types/src/api/web/workspace/rows/search.ts#L57-L75)

这是基于 Zod 的验证器，与 internalSearchValidator 双重验证：

```typescript
const searchRowRequest = z.object({
  query: z.object({
    allOr: z.boolean().optional(),
    onEmptyFilter: z.nativeEnum(EmptyFilterOption).optional(),
    ...queryFilterValidation,  // 所有过滤操作符
  }).passthrough().optional(),
  paginate: z.boolean().optional(),
  bookmark: z.union([z.string(), z.number()]).nullish(),
  limit: z.number().optional(),
  sort: z.string().nullish(),
  sortOrder: z.nativeEnum(SortOrder).optional(),
  sortType: z.nativeEnum(SortType).nullish(),
  version: z.string().optional(),
  disableEscaping: z.boolean().optional(),
  countRows: z.boolean().optional(),
})
```

**关键安全措施**：通过 `fieldKey` 限制过滤字段的 key 不能是 `InternalSearchFilterOperator.COMPLEX_ID_OPERATOR`（即 `_id` 的特殊变体），防止通过该字段绕过安全控制。

### 1.3 路由注册

**位置**: [row.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/row.ts#L44-L50)

```typescript
readRoutes
  .post(
    "/api/:sourceId/search",
    internalSearchValidator(),       // Joi 验证
    validateBody(searchRowRequestValidator),  // Zod 验证
    paramResource("sourceId"),
    rowController.search
  )
```

viewV2 的专用路由：
```typescript
publicRoutes.post(
  "/api/v2/views/:viewId/search",
  internalSearchValidator(),
  validateBody(searchRowRequestValidator),
  authorizedResource(PermissionType.VIEW, PermissionLevel.READ, "viewId"),
  rowController.views.searchView
)
```

---

## 二、Controller 层处理

### 2.1 replaceTableNamesInFilters

**位置**: [row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L371-L412)

该函数将过滤器中的表名替换为关系字段名，支持通过关联表名进行过滤：

**处理流程**：
1. 遍历所有过滤器对象的 key
2. 用正则匹配 `relation.field` 格式
3. 检查 relation 是否是当前表的列名（如果是则跳过，避免误替换）
4. 在所有表中查找名称匹配的表
5. 查找当前表与目标表之间的关系字段
6. 将 `tableName.field` 替换为 `relationshipName.field`
7. 递归处理逻辑操作符（$and, $or）内的条件

### 2.2 enrichSearchContext

**位置**: [utils/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/utils/utils.ts#L192-L233)

该函数使用 Handlebars 模板和用户上下文绑定来丰富搜索条件中的动态值：

**处理流程**：
1. 遍历 filters 的所有键值对
2. 如果值为 null，保持为 null
3. 如果值为对象，递归丰富
4. 如果值为字符串，使用 `processStringSync` 进行模板处理
   - 支持 `{{currentUser.email}}` 等绑定
   - `noEscaping: true`：不转义
   - `escapeNewlines: true`：转义换行

### 2.3 search 控制器主逻辑

**位置**: [row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L334-L369)

```typescript
export async function search(ctx) {
  const { tableId, viewId } = utils.getSourceId(ctx)
  
  // 1. 替换表名为关系字段名
  if (query) {
    const allTables = await sdk.tables.getAllTables()
    query = replaceTableNamesInFilters(tableId, query, allTables)
  }

  // 2. 丰富用户上下文
  let enrichedQuery = await utils.enrichSearchContext(query, {
    user: sdk.users.getUserContextBindings(ctx.user),
  })

  // 3. 构造搜索参数并调用 SDK
  const searchParams = {
    query: enrichedQuery,
    tableId,
    viewId,
    // ... 其他参数
  }
  ctx.body = await sdk.rows.search(searchParams)
}
```

### 2.4 viewV2 专用控制器 searchView

**位置**: [views.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/views.ts#L12-L59)

viewV2 搜索的特殊处理：
1. 通过 `viewId` 获取 view 对象
2. 提取 view 中可见的字段（`visible: true`）作为 `fields` 参数
3. 排序优先级：请求中的 sort > view 中保存的 sort
4. 调用 `sdk.rows.search(searchOptions, { user: ... })`，传入 viewId 和 fields
5. 为返回的每一行添加 `_viewId` 标记

---

## 三、SDK rows.search 核心逻辑

**位置**: [search.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/search.ts#L38-L151)

### 3.1 整体流程

```
获取 source (table 或 view)
    ↓
有 query? → 是 → 获取 queryableFields → validateFilters
    ↓        否 → query = {}
    ↓
有 context? → 是 → enrichSearchContext(query, context)
    ↓
searchInputMapping (处理用户列映射)
    ↓
有 viewId? → 是 → 合并 view.query 与用户 query（$and）
    ↓
cleanupQuery + fixupFilterArrays
    ↓
onEmptyFilter === RETURN_NONE 且无过滤器? → 是 → 返回空
    ↓
外部表 → external.search
内部表 → internal.sqs.search
```

### 3.2 getQueryableFields

**位置**: [queryUtils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/queryUtils.ts#L68-L136)

获取可查询的字段列表，用于 validateFilters：

**返回的字段包括**：
1. `_id` 字段（始终允许）
2. 表中所有 visible 的字段
3. 关系字段及其关联表的字段（支持 `relationshipName.field` 和 `tableName.field` 两种格式）
4. 递归深度：只深入一层关系（防止循环），通过 `noRelationships: true` 控制

### 3.3 validateFilters

**位置**: [queryUtils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/queryUtils.ts#L28-L66)

验证过滤器中的字段是否都在允许的列表中：

**验证逻辑**：
1. 检查所有过滤操作符是否合法（isAllowedFilterKey）
2. 元数据键（allOr, onEmptyFilter 等）直接跳过
3. 逻辑操作符递归验证 conditions
4. 其他操作符遍历其所有字段 key
   - 检查 key（小写）是否在 validFields 中
   - 或 `removeKeyNumbering(key)` 后是否在 validFields 中（处理如 "1:fieldName" 的数字前缀）
   - 不合法则抛出 `HTTPError: Invalid filter field: ${key}`

### 3.4 view.query 合并逻辑

**位置**: [search.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/search.ts#L89-L112)

**这是防止用户绕过 view 过滤条件的核心机制**：

```typescript
if (options.viewId) {
  const view = source as ViewV2

  // 1. 丰富 view 保存的查询（支持动态绑定）
  let viewQuery = (await enrichSearchContext(view.query || {}, context))
  // 2. 转换 legacy 过滤器格式
  if (Array.isArray(viewQuery)) {
    viewQuery = dataFilters.buildQuery(viewQuery)
  }
  // 3. 处理用户列映射
  viewQuery = checkFilters(table, viewQuery)

  // 4. 关键：用 $and 合并 viewQuery 和用户 query
  const conditions = viewQuery ? [viewQuery] : []
  options.query = {
    $and: {
      conditions: [...conditions, options.query],
    },
  }
  // 5. onEmptyFilter 以 view 的为准
  if (viewQuery.onEmptyFilter) {
    options.query.onEmptyFilter = viewQuery.onEmptyFilter
  }
}
```

**核心安全设计**：使用 `$and` 将 view 保存的过滤条件与用户传入的过滤条件合并，确保 view 的过滤条件始终生效，用户只能在 view 过滤的结果范围内进一步筛选。

### 3.5 onEmptyFilter 处理

**位置**: [search.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/search.ts#L121-L129)

```typescript
if (
  !dataFilters.hasFilters(options.query) &&
  options.query.onEmptyFilter === EmptyFilterOption.RETURN_NONE
) {
  return { rows: [] }
}
```

当查询经过 cleanup 后没有实际过滤条件，且 `onEmptyFilter` 设置为 `RETURN_NONE` 时，直接返回空结果集。结合 view.query 合并逻辑，这意味着如果 view 设置了 `RETURN_NONE`，用户无法通过传空查询来绕过。

### 3.6 cleanupQuery

**位置**: [filters.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/shared-core/src/filters.ts#L168-L192)

清理查询中空字符串值的过滤器：
- 对 `NoEmptyFilterStrings` 中的操作符（string, equal, notEqual, contains 等）
- 删除值为 null、空字符串或空数组的过滤条件
- 递归处理逻辑操作符内的条件

### 3.7 fixupFilterArrays

**位置**: [filters.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/shared-core/src/filters.ts#L532-L557)

将数组过滤器的单个值转换为数组：
- 字符串值按逗号分割
- 其他类型值包装为单元素数组

---

## 四、Internal SQS 搜索

**位置**: [sqs.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/search/internal/sqs.ts)

### 4.1 cleanupFilters

**位置**: [sqs.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/search/internal/sqs.ts#L169-L219)

将过滤器中的列名转换为内部 SQS 使用的用户列格式：

**处理流程**：
1. 遍历所有表，构建 `userColumnMap`（列名 → 用户列名的映射）
2. 使用 `ColumnSplitter` 拆分列名（数字前缀、关系前缀、列名）
3. 对每个过滤字段的 key：
   - 拆分出 `numberPrefix`、`relationshipPrefix`、`column`
   - 如果 column 是任何表中的有效列名
   - 替换为 `${numberPrefix}${relationshipPrefix}${mapToUserColumn(column)}`
4. 递归处理逻辑操作符内的条件

### 4.2 buildInternalFieldList

**位置**: [sqs.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/search/internal/sqs.ts#L63-L167)

构建 SQS 查询需要的字段列表：

**对于 table source**：
- 所有 visible 的 schema 字段
- 受保护的内部列（PROTECTED_INTERNAL_COLUMNS）
- 关系字段会展开关联表的所有字段

**对于 view source**：
- 从 `helpers.views.basicFields(source)` 获取基本字段
- 如果包含公式字段，则需要获取表的所有字段（因为公式依赖其他字段）
- 计算视图（calculation view）不包含受保护内部列

**特殊处理**：
- 字段名格式：`${tableId}.${columnName}`
- 关系字段会递归展开（只深入一层）
- 有 `SQLITE_COLUMN_LIMIT = 2000` 的限制

### 4.3 分页多取一行

**位置**: [sqs.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/search/internal/sqs.ts#L456-L462)

```typescript
if (!isSearchingByRowID(searchFilters) && params.limit) {
  paginate = true
  request.paginate = {
    limit: params.limit + 1,  // 多取一行
    offset: bookmark,
  }
}
```

**设计意图**：多取一行数据来判断是否还有下一页，避免额外的 count 查询。

**结果处理**（第 487-493 行）：
```typescript
let nextRow = false
if (paginate && params.limit && rows.length > params.limit) {
  nextRow = true
  if (processed.length > params.limit) {
    processed.pop()  // 移除多余的那一行
  }
}
```

### 4.4 锁内重试（SQS 定义缺失时）

**位置**: [sqs.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/search/internal/sqs.ts#L524-L545)

当 SQS 搜索因为表/列定义缺失而失败时，会触发自动同步并重试：

```typescript
try {
  // ... 执行查询
} catch (err: any) {
  const msg = typeof err === "string" ? err : err.message
  
  // 需要重新同步定义的情况
  if (!opts?.retrying && resyncDefinitionsRequired(err.status, msg)) {
    await locks.doWithLock(
      {
        type: LockType.AUTO_EXTEND,
        name: LockName.SQS_SYNC_DEFINITIONS,
        resource: context.getWorkspaceId(),
      },
      sdk.tables.sqs.syncDefinition  // 在锁内执行同步
    )
    return search(options, source, { retrying: true })  // 重试一次
  }
  
  // 缺失列时优雅降级，返回空结果
  if (err.status === 400 && msg?.match(MISSING_COLUMN_REGEX)) {
    return { rows: [] }
  }
  
  throw new Error(...)
}
```

**需要重同步的条件**（`resyncDefinitionsRequired` 函数）：
1. 状态码 400 且匹配 "no such table: ..." → 表缺失
2. 状态码 400 且匹配 "no such column: ..." → 列缺失
3. 状态码 400 且匹配 "duplicate column name: ..." → 重复列
4. 状态码 404 且包含 SQLITE_DESIGN_DOC_ID → 设计文档缺失

---

## 五、防止用户绕过 view 保存过滤条件的分支

### 5.1 核心防护：$and 合并 view.query

**位置**: [search.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/search.ts#L89-L112)

**关键代码**：
```typescript
options.query = {
  $and: {
    conditions: [...conditions, options.query],
  },
}
```

**防护原理**：
- view 保存的 `query` 始终作为 `$and` 的第一个条件
- 用户传入的 `query` 作为第二个条件
- 两者是 **AND** 关系，用户无法通过任何方式排除 view 的过滤条件
- 用户只能在 view 过滤结果的基础上进一步缩小范围

### 5.2 onEmptyFilter 以 view 为准

**位置**: [search.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/search.ts#L109-L111)

```typescript
if (viewQuery.onEmptyFilter) {
  options.query.onEmptyFilter = viewQuery.onEmptyFilter
}
```

**防护原理**：
- 如果 view 设置了 `onEmptyFilter`（如 `RETURN_NONE`），则最终查询使用 view 的设置
- 即使用户不传任何查询条件，也无法绕过 view 的空过滤器行为
- 结合第 121-129 行的空查询检查：当 view 设置了 `RETURN_NONE` 且 view.query 为空时，直接返回空结果

### 5.3 validateFilters 字段白名单

**位置**: [queryUtils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/queryUtils.ts#L28-L66)

**防护原理**：
- 用户只能查询 `getQueryableFields` 返回的字段
- 不允许查询不可见的字段或内部字段
- 防止用户通过构造特殊字段名来绕过安全控制

### 5.4 searchRowRequestValidator 字段名限制

**位置**: [search.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/types/src/api/web/workspace/rows/search.ts#L15-L19)

```typescript
const fieldKey = z
  .string()
  .refine(s => s !== InternalSearchFilterOperator.COMPLEX_ID_OPERATOR, {
    message: `Key '${InternalSearchFilterOperator.COMPLEX_ID_OPERATOR}' is not allowed`,
  })
```

**防护原理**：禁止使用特殊的 `COMPLEX_ID_OPERATOR` 作为过滤字段名，防止通过内部操作符绕过验证。

### 5.5 viewV2 搜索的 fields 限制

**位置**: [views.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/views.ts#L25-L27)

```typescript
const viewFields = Object.entries(view.schema || {})
  .filter(([_, value]) => value.visible)
  .map(([key]) => key)
```

**防护原理**：
- viewV2 搜索时，只返回 view 中 visible 的字段
- 通过 `fields: viewFields` 参数限制返回的列
- 即使后端查询了更多字段，最终结果也会被过滤

---

## 六、普通 table vs viewV2 搜索条件构造对比

| 环节 | 普通 Table | ViewV2 |
|------|-----------|--------|
| **路由** | `/api/:sourceId/search`，sourceId 是 tableId | `/api/v2/views/:viewId/search`，viewId 是 view ID |
| **source 获取** | 直接通过 tableId 获取 table | 通过 viewId 获取 view，再通过 view.tableId 获取 table |
| **字段限制** | 所有 visible 字段 | view 中 visible 的字段 |
| **查询条件** | 仅用户传入的 query | view.query $and 用户 query |
| **onEmptyFilter** | 用户传入的（或默认） | 以 view 的为准 |
| **排序** | 用户传入的 sort | 请求 sort > view 保存的 sort |
| **返回字段** | 所有 visible 字段 | view 中 visible 的字段 |
| **结果标记** | 无 | 每行添加 `_viewId` |

---

## 七、完整调用链总结

```
POST /api/:sourceId/search
    ↓
[Route] internalSearchValidator (Joi)
    ↓
[Route] validateBody(searchRowRequestValidator) (Zod)
    ↓
[Controller] search()
    ├→ getSourceId() → 解析 tableId/viewId
    ├→ replaceTableNamesInFilters() → 表名→关系字段名
    └→ enrichSearchContext() → 用户上下文模板绑定
    ↓
[SDK] sdk.rows.search()
    ├→ 获取 source (table/view) + table
    ├→ getQueryableFields() → 可查询字段列表
    ├→ validateFilters() → 字段白名单验证
    ├→ enrichSearchContext() → 二次上下文绑定（如果有 context）
    ├→ searchInputMapping() → 用户列映射
    ├─ [view 分支] 合并 view.query
    │   ├→ enrichSearchContext(view.query)
    │   ├→ buildQuery() (legacy 转换)
    │   ├→ checkFilters()
    │   └→ $and 合并 + onEmptyFilter 覆盖
    ├→ cleanupQuery() → 清理空值过滤器
    ├→ fixupFilterArrays() → 数组值标准化
    ├→ onEmptyFilter RETURN_NONE 检查 → 空查询直接返回空
    └→ 内部表 → internal.sqs.search()
         ├→ cleanupFilters() → 列名→用户列名
         ├→ buildInternalFieldList() → 构建查询字段列表
         ├→ 分页 limit+1 → 多取一行判断下一页
         ├→ runSqlQuery() → 执行 SQLite 查询
         ├→ 结果处理 + 分页判断
         └─ [错误分支] 表/列缺失 → 锁内同步定义 → 重试一次
```

---

## 八、关键安全要点总结

1. **view 过滤条件不可绕过**：通过 `$and` 合并，view 的 query 始终生效
2. **onEmptyFilter 以 view 为准**：防止用户通过空查询绕过
3. **字段白名单验证**：validateFilters 确保只能查询可见字段
4. **双重验证**：Joi + Zod 两层验证请求体
5. **返回字段限制**：viewV2 搜索只返回 view 中 visible 的字段
6. **内部操作符禁用**：禁止使用 COMPLEX_ID_OPERATOR 等内部字段
