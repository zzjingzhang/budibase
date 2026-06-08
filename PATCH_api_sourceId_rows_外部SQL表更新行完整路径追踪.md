# PATCH /api/:sourceId/rows 更新外部SQL表行完整路径追踪

## 目录

1. [整体流程概览](#整体流程概览)
2. [pickApi 选择器：根据 isExternalTableID 选择 external](#pickapi-选择器根据-isexternaltableid-选择-external)
3. [external.patch 核心流程详解](#externalpatch-核心流程详解)
   - 3.1 [拒绝计算视图](#31-拒绝计算视图)
   - 3.2 [读取 beforeRow](#32-读取-beforerow)
   - 3.3 [只合并 getSourceFields 允许的字段](#33-只合并-getsourcefields-允许的字段)
   - 3.4 [执行 inputProcessing](#34-执行-inputprocessing)
   - 3.5 [执行 validate 验证](#35-执行-validate-验证)
   - 3.6 [通过 handleRequest(Operation.UPDATE) 调用 ExternalRequest](#36-通过-handlerequestoperationupdate-调用-externalrequest)
4. [generateIdForRow 重新计算 updatedId 与 refetch](#generateidforrow-重新计算-updatedid-与-refetch)
5. [outputProcessing 对返回 row 和 oldRow 的影响](#outputprocessing-对返回-row-和-oldrow-的影响)
   - 5.1 [preserveLinks=true 的影响](#51-preservelinkstrue-的影响)
   - 5.2 [squash=true 的影响](#52-squashtrue-的影响)

---

## 整体流程概览

```
PATCH /api/:sourceId/rows
        ↓
rowController.patch (入口)
        ↓
pickApi(tableId) → 选择 external 模块
        ↓
external.patch
  ├─ 拒绝计算视图 (isCalculationView)
  ├─ 读取 beforeRow (sdk.rows.external.getRow)
  ├─ 合并允许的字段 (getSourceFields)
  ├─ inputProcessing (类型转换、自动列等)
  ├─ validate (验证行数据)
  └─ handleRequest(Operation.UPDATE)
        ↓
ExternalRequest.run
  ├─ cleanupConfig
  ├─ prepareFilters
  ├─ inputProcessing (关系处理)
  ├─ makeExternalQuery (实际执行SQL)
  ├─ handleManyRelationships
  └─ sqlOutputProcessing
        ↓
generateIdForRow 重新计算 updatedId
        ↓
refetch 重新获取行 (sdk.rows.external.getRow)
        ↓
outputProcessing(row, { squash: true, preserveLinks: true })
outputProcessing(beforeRow, { squash: true, preserveLinks: true })
        ↓
返回 { row, table, oldRow }
```

---

## pickApi 选择器：根据 isExternalTableID 选择 external

### 位置

- 控制器层: [row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L53-L58)
- SDK 层: [sdk/workspace/rows/rows.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/rows.ts#L24-L34)
- isExternalTableID 定义: [integrations/utils/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/integrations/utils/utils.ts#L118)

### 控制器层 pickApi 实现

```typescript
function pickApi(tableId: string) {
  if (isExternalTableID(tableId)) {
    return external
  }
  return internal
}
```

### SDK 层 pickApi 实现

```typescript
function pickApi(tableOrViewId: string) {
  let tableId = tableOrViewId
  if (isViewId(tableOrViewId)) {
    tableId = getTableIdFromViewId(tableOrViewId)
  }

  if (isExternalTableID(tableId)) {
    return external
  }
  return internal
}
```

### 调用时机

在 `rowController.patch` 函数中：

```typescript
const api = pickApi(tableId)
const response = await api.patch(ctx)
```

### isExternalTableID 判断逻辑

`isExternalTableID` 来自 `@budibase/backend-core` 的 `sql.utils.isExternalTableID`，用于判断表ID是否为外部SQL表的格式（通常以特定前缀标识，如 `datasource_` 等）。

---

## external.patch 核心流程详解

### 位置

[api/controllers/row/external.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/external.ts#L45-L113)

### 完整代码

```typescript
export async function patch(ctx: UserCtx<PatchRowRequest, PatchRowResponse>) {
  const source = await utils.getSource(ctx)

  const { viewId, tableId } = utils.getSourceId(ctx)
  const sourceId = viewId || tableId

  if (sdk.views.isView(source) && helpers.views.isCalculationView(source)) {
    ctx.throw(400, "Cannot update rows through a calculation view")
  }

  const table = await utils.getTableFromSource(source)
  const { _id, ...rowData } = ctx.request.body

  const beforeRow = await sdk.rows.external.getRow(table._id!, _id, {
    relationships: true,
  })

  let dataToUpdate = cloneDeep(beforeRow)
  const allowedField = utils.getSourceFields(source)
  for (const key of Object.keys(rowData)) {
    if (!allowedField.includes(key)) continue

    dataToUpdate[key] = rowData[key]
  }

  dataToUpdate = await inputProcessing(
    ctx.user?._id,
    cloneDeep(source),
    dataToUpdate
  )

  const validateResult = await sdk.rows.utils.validate({
    row: dataToUpdate,
    source,
  })
  if (!validateResult.valid) {
    throw { validation: validateResult.errors }
  }

  const response = await handleRequest(Operation.UPDATE, source, {
    id: breakRowIdField(_id),
    row: dataToUpdate,
  })

  // The id might have been changed, so the refetching would fail. Recalculating the id just in case
  const updatedId =
    generateIdForRow({ ...beforeRow, ...dataToUpdate }, table) || _id
  const row = await sdk.rows.external.getRow(sourceId, updatedId, {
    relationships: true,
  })

  const [enrichedRow, oldRow] = await Promise.all([
    outputProcessing(source, row, {
      squash: true,
      preserveLinks: true,
    }),
    outputProcessing(source, beforeRow, {
      squash: true,
      preserveLinks: true,
    }),
  ])

  return {
    ...response,
    row: enrichedRow,
    table,
    oldRow,
  }
}
```

### 3.1 拒绝计算视图

#### 代码位置

[external.ts#L51-L53](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/external.ts#L51-L53)

```typescript
if (sdk.views.isView(source) && helpers.views.isCalculationView(source)) {
  ctx.throw(400, "Cannot update rows through a calculation view")
}
```

#### 设计原因

计算视图（Calculation View）是基于聚合函数（如 COUNT、SUM、AVG 等）生成的虚拟视图，其行数据是计算出来的结果，不对应真实的物理表行，因此：

- 无法定位到具体的物理行进行更新
- 更新计算结果没有意义，因为数据来源于底层表的聚合
- 避免用户产生误解，以为可以修改聚合结果

---

### 3.2 读取 beforeRow

#### 代码位置

[external.ts#L58-L60](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/external.ts#L58-L60)

```typescript
const beforeRow = await sdk.rows.external.getRow(table._id!, _id, {
  relationships: true,
})
```

#### 作用

1. **获取完整行数据**：PATCH 请求只包含要更新的字段，需要先获取完整的行数据作为基础
2. **提供 oldRow 用于事件和对比**：更新完成后需要返回旧行数据，用于事件触发（如 ROW_UPDATE）和前端对比
3. **用于重新计算 ID**：如果主键字段被修改，需要用旧数据 + 新数据来重新计算行ID
4. **携带 relationships**：`relationships: true` 确保获取关联数据，保证输出处理的完整性

---

### 3.3 只合并 getSourceFields 允许的字段

#### 代码位置

[external.ts#L62-L68](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/external.ts#L62-L68)

```typescript
let dataToUpdate = cloneDeep(beforeRow)
const allowedField = utils.getSourceFields(source)
for (const key of Object.keys(rowData)) {
  if (!allowedField.includes(key)) continue

  dataToUpdate[key] = rowData[key]
}
```

#### getSourceFields 实现

位置: [row/utils/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/utils/utils.ts#L113-L126)

```typescript
export function getSourceFields(source: Table | ViewV2): string[] {
  const isView = sdk.views.isView(source)
  if (isView) {
    const fields = Object.keys(
      helpers.views.basicFields(source, { visible: true })
    )
    return fields
  }

  const fields = Object.entries(source.schema)
    .filter(([_, field]) => field.visible !== false)
    .map(([columnName]) => columnName)
  return fields
}
```

#### 设计原因

1. **安全性**：防止用户通过 API 修改不可见的字段或内部字段
2. **视图限制**：如果通过视图访问，只能修改视图中可见的字段
3. **字段过滤**：对于表，只允许修改 `visible !== false` 的字段
4. **合并策略**：以 beforeRow 为基础，只覆盖允许的字段，保证数据完整性

---

### 3.4 执行 inputProcessing

#### 代码位置

[external.ts#L70-L74](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/external.ts#L70-L74)

```typescript
dataToUpdate = await inputProcessing(
  ctx.user?._id,
  cloneDeep(source),
  dataToUpdate
)
```

#### inputProcessing 实现

位置: [utilities/rowProcessor/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/utilities/rowProcessor/index.ts#L221-L287)

主要处理内容：

1. **字段清理**：移除不在 schema 中的字段（非内置列）
2. **公式字段移除**：删除公式字段值，因为它们是计算生成的
3. **类型转换**：根据字段类型进行值的强制转换（coerce）
4. **附件处理**：移除附件 URL，URL 是读取时生成的
5. **BB引用处理**：处理 BB_REFERENCE 和 BB_REFERENCE_SINGLE 类型字段
6. **自动列处理**：更新 auto 列（如 updatedAt、updatedBy 等）
7. **默认值处理**：为空字段填充默认值

---

### 3.5 执行 validate 验证

#### 代码位置

[external.ts#L76-L82](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/external.ts#L76-L82)

```typescript
const validateResult = await sdk.rows.utils.validate({
  row: dataToUpdate,
  source,
})
if (!validateResult.valid) {
  throw { validation: validateResult.errors }
}
```

#### 验证内容

基于表 schema 的约束进行验证，包括：

- 必填字段检查（presence 约束）
- 字段类型验证
- 选项值验证（inclusion 约束）
- 其他自定义验证规则

验证失败时抛出包含 validation 错误的对象。

---

### 3.6 通过 handleRequest(Operation.UPDATE) 调用 ExternalRequest

#### handleRequest 实现

位置: [external.ts#L33-L43](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/external.ts#L33-L43)

```typescript
export async function handleRequest<T extends Operation>(
  operation: T,
  source: Table | ViewV2,
  opts?: RunConfig
): Promise<ExternalRequestReturnType<T>> {
  return (
    await ExternalRequest.for<T>(operation, source, {
      datasource: opts?.datasource,
    })
  ).run(opts || {})
}
```

#### 调用参数

```typescript
const response = await handleRequest(Operation.UPDATE, source, {
  id: breakRowIdField(_id),
  row: dataToUpdate,
})
```

- `Operation.UPDATE`: 指定操作为更新
- `id: breakRowIdField(_id)`: 将编码后的行ID分解为原始主键值数组
- `row: dataToUpdate`: 要更新的行数据

#### ExternalRequest.run 流程

位置: [ExternalRequest.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/ExternalRequest.ts#L642-L799)

主要步骤：

1. **cleanupConfig**: 清理配置参数，处理行ID格式转换
2. **prepareFilters**: 准备过滤条件，将ID转换为WHERE条件
3. **buildExternalRelationships**: 构建外部表关系配置
4. **inputProcessing**: 处理关系字段（LINK类型），将链接转换为外键
5. **构建QueryJson**: 构建查询JSON对象
6. **makeExternalQuery**: 执行实际的SQL查询
7. **handleManyRelationships**: 处理多对多关系（创建/删除连接表记录）
8. **sqlOutputProcessing**: 处理输出，生成 _id、_rev、处理关系等

---

## generateIdForRow 重新计算 updatedId 与 refetch

### 代码位置

[external.ts#L89-L94](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/external.ts#L89-L94)

```typescript
// The id might have been changed, so the refetching would fail. Recalculating the id just in case
const updatedId =
  generateIdForRow({ ...beforeRow, ...dataToUpdate }, table) || _id
const row = await sdk.rows.external.getRow(sourceId, updatedId, {
  relationships: true,
})
```

### generateIdForRow 实现

位置: [row/utils/basic.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/utils/basic.ts#L46-L72)

```typescript
export function generateIdForRow(
  row: Row | undefined,
  table: Table,
  isLinked = false
): string {
  const primary = table.primary
  if (!row || !primary) {
    return ""
  }
  // build id array
  let idParts = []
  for (let field of primary) {
    let fieldValue = extractFieldValue({
      row,
      tableName: table.name,
      fieldName: field,
      isLinked,
    })
    if (fieldValue != null) {
      idParts.push(fieldValue)
    }
  }
  if (idParts.length === 0) {
    return ""
  }
  return generateRowIdField(idParts)
}
```

### 为什么需要重新计算 ID

1. **主键可能被修改**：外部SQL表的主键字段（如 id、uuid 等）可能在更新中被修改
2. **Budibase 的行 ID 是编码的**：外部表的行 `_id` 是由主键值编码生成的（`generateRowIdField`），不是固定的UUID
3. **refetch 需要正确的 ID**：如果主键变了，用原来的 `_id` 去查询会找不到行
4. **回退机制**：如果计算失败，则使用原来的 `_id` 作为后备

### 为什么需要 refetch

1. **获取数据库实际值**：有些字段可能由数据库触发器、默认值或计算列修改
2. **确保数据一致性**：返回数据库中的真实状态，而不是客户端传入的数据
3. **获取完整关系数据**：重新获取确保关系数据是最新的
4. **自动列更新**：如 updated_at 等自动列需要从数据库重新读取

---

## outputProcessing 对返回 row 和 oldRow 的影响

### 调用位置

[external.ts#L96-L105](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/external.ts#L96-L105)

```typescript
const [enrichedRow, oldRow] = await Promise.all([
  outputProcessing(source, row, {
    squash: true,
    preserveLinks: true,
  }),
  outputProcessing(source, beforeRow, {
    squash: true,
    preserveLinks: true,
  }),
])
```

### outputProcessing 核心实现

位置: [utilities/rowProcessor/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/utilities/rowProcessor/index.ts#L298-L347)

```typescript
export async function outputProcessing<T extends Row[] | Row>(
  source: Table | ViewV2,
  rows: T,
  opts: {
    squash?: boolean
    preserveLinks?: boolean
    fromRow?: Row
    skipBBReferences?: boolean
  } = {
    squash: true,
    preserveLinks: false,
    skipBBReferences: false,
  }
): Promise<T> {
  // ...

  // SQS returns the rows with full relationship contents
  // attach any linked row information
  let enriched = !opts.preserveLinks
    ? await linkRows.attachFullLinkedDocs(table.schema, safeRows, {
        fromRow: opts?.fromRow,
      })
    : safeRows

  if (!opts.squash && utils.hasCircularStructure(rows)) {
    opts.squash = true
  }

  enriched = await coreOutputProcessing(source, enriched, opts)

  if (opts.squash) {
    enriched = await linkRows.squashLinks(source, enriched)
  }

  return (wasArray ? enriched : enriched[0]) as T
}
```

### 5.1 preserveLinks=true 的影响

#### preserveLinks=false（默认值）的行为

当 `preserveLinks=false` 时，会调用 `attachFullLinkedDocs`：

位置: [db/linkedRows/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/db/linkedRows/index.ts#L157-L213)

- 从数据库获取链接文档（link documents）
- 获取关联的完整行数据
- 将关联行数据附加到主行的对应字段中
- 处理用户元数据等特殊情况

#### preserveLinks=true 的行为

当 `preserveLinks=true` 时：

- **跳过 attachFullLinkedDocs**：不额外查询和附加完整的链接文档
- **保留原始链接数据**：使用行中已有的关系数据，不进行额外的数据库查询
- **性能优化**：避免额外的数据库查询，提高性能
- **适用于已有完整关系数据的场景**：比如从外部数据库查询返回的行已经包含了 JOIN 的关系数据

#### 在 patch 中使用 preserveLinks=true 的原因

1. **refetch 时已包含 relationships**：`sdk.rows.external.getRow` 使用 `relationships: true` 参数，返回的行已经包含了关系数据
2. **避免重复查询**：不需要再通过内部链接机制去获取一次
3. **外部表的关系处理方式不同**：外部SQL表的关系是通过JOIN查询获取的，不是内部表的link document机制

---

### 5.2 squash=true 的影响

#### squashLinks 实现

位置: [db/linkedRows/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/db/linkedRows/index.ts#L249-L322)

```typescript
export async function squashLinks<T = Row[] | Row>(
  source: Table | ViewV2,
  enriched: T
): Promise<T> {
  // ...
  for (const row of enrichedArray) {
    for (let [column, schema] of Object.entries(rowTable.schema)) {
      if (schema.type !== FieldType.LINK || !Array.isArray(row[column])) {
        continue
      }
      const relatedTable = await getLinkedTable(schema.tableId, linkedTables)
      row[column] = row[column].map((link: Row) => {
        const obj: Record<string, unknown> = { _id: link._id }
        obj.primaryDisplay = getPrimaryDisplayValue(link, relatedTable)
        // ...视图列的额外处理
        return obj
      })
    }
  }
  return (isArray ? enrichedArray : enrichedArray[0]) as T
}
```

#### squash=true 的行为

当 `squash=true` 时：

- **压缩关联行**：将完整的关联行对象压缩为只包含 `_id` 和 `primaryDisplay` 的简化对象
- **减少数据量**：返回给前端的数据更小，传输更快
- **避免循环引用**：防止双向关系导致的循环引用问题
- **符合前端期望**：前端通常只需要关联记录的ID和显示名称

#### squash=false 的行为

当 `squash=false` 时：

- 保留完整的关联行数据
- 所有字段都包含在关联对象中
- 数据量更大，但信息更完整
- 可能存在循环引用问题（代码中有检测：`utils.hasCircularStructure`）

#### 在 patch 中使用 squash=true 的原因

1. **API 响应规范**：Budibase 的标准 API 响应使用 squash 格式
2. **性能优化**：减少响应数据大小
3. **前端兼容性**：前端组件期望这种格式的数据
4. **避免循环引用**：双向关系会导致循环引用，压缩后避免此问题

---

## 总结

PATCH /api/:sourceId/rows 更新外部SQL表行的完整流程体现了以下设计原则：

1. **分层架构**：通过 pickApi 实现内部表和外部表的统一接口，内部实现分离
2. **安全过滤**：通过 getSourceFields 限制可修改的字段范围
3. **数据完整性**：先读取再更新，确保数据一致性
4. **验证前置**：在执行数据库操作前进行数据验证
5. **ID 动态计算**：外部表主键可修改，需要重新计算行ID
6. **重新获取**：更新后重新获取数据，确保返回数据库真实状态
7. **输出处理**：通过 squash 和 preserveLinks 参数灵活控制输出格式
