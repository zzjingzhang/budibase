# DELETE /api/:sourceId/rows 删除 API 深度分析

本文档详细分析了 Budibase 中行删除 API 在内部表与外部表上的实现差异，涵盖单行删除和批量删除两种场景。

---

## 1. 路由入口与控制器分发

### 1.1 路由定义

路由定义在 [row.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/row.ts#L82-L87)：

```typescript
writeRoutes
  .delete(
    "/api/:sourceId/rows",
    paramResource("sourceId"),
    trimViewRowInfoMiddleware,
    rowController.destroy
  )
```

- HTTP 方法：`DELETE`
- 路径：`/api/:sourceId/rows`
- 中间件：`paramResource`（资源权限）、`trimViewRowInfoMiddleware`（视图行信息裁剪）
- 处理器：`rowController.destroy`

### 1.2 控制器总入口 destroy

总入口函数位于 [row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L327-L332)：

```typescript
export async function destroy(
  ctx: UserCtx<DeleteRowRequest>,
  opts: { isAutomation?: boolean } = {}
) {
  return _destroy(ctx, opts)
}
```

---

## 2. isDeleteRows / isDeleteRow 分支判断

在 [_destroy](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L305-L325) 函数中，根据请求体格式判断是单行删除还是批量删除：

### 2.1 判断函数

| 函数 | 判断条件 | 含义 |
|------|---------|------|
| `isDeleteRows(input)` | `input.rows !== undefined && Array.isArray(input.rows)` | 批量删除 |
| `isDeleteRow(input)` | `input._id !== undefined` | 单行删除 |

### 2.2 分支逻辑

```typescript
if (isDeleteRows(ctx.request.body)) {
  response = await deleteRows(ctx, { isAutomation })
} else if (isDeleteRow(ctx.request.body)) {
  const deleteResp = await deleteRow(ctx, { isAutomation })
  response = deleteResp.response
  row = deleteResp.row
} else {
  ctx.status = 400
  response = { message: "Invalid delete rows request" }
}

// for automations include the row that was deleted
ctx.row = row || {}
ctx.body = response
```

**关键点**：
- 批量删除请求体包含 `{ rows: [...] }` 数组
- 单行删除请求体包含 `{ _id: "..." }`
- `ctx.row` 专门用于自动化场景，保存被删除的行数据

---

## 3. processDeleteRowsRequest：补全 _rev 和 fixRow

`processDeleteRowsRequest` 函数**仅在批量删除**时被调用，位于 [row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L222-L237)。

### 3.1 处理流程

```typescript
async function processDeleteRowsRequest(ctx: UserCtx<DeleteRowRequest>) {
  let request = ctx.request.body as DeleteRows
  const { tableId } = utils.getSourceId(ctx)

  const processedRows = request.rows.map(row => {
    let processedRow: Row = typeof row == "string" ? { _id: row, tableId } : row
    return !processedRow._rev
      ? addRev(fixRow(processedRow, ctx.params), tableId)
      : fixRow(processedRow, ctx.params)
  })

  const responses = await Promise.allSettled(processedRows)
  return responses
    .filter(resp => resp.status === "fulfilled")
    .map(resp => (resp as PromiseFulfilledResult<Row>).value)
}
```

### 3.2 fixRow 函数

定义在 [public/rows.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/public/rows.ts#L9-L23)：

```typescript
export function fixRow(row: Row, params: any) {
  if (!params || !row) return row
  if (params.rowId) row._id = params.rowId
  if (params.tableId) row.tableId = params.tableId
  if (!row.type) row.type = "row"
  return row
}
```

**补全内容**：
- `_id`：从 URL 参数 `params.rowId` 补全
- `tableId`：从 URL 参数 `params.tableId` 补全
- `type`：默认为 `"row"`

### 3.3 addRev 函数

定义在 [public/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/public/utils.ts#L6-L23)：

```typescript
export async function addRev(
  body: { _id?: string; _rev?: string },
  tableId?: string
): Promise<Row> {
  if (!body._id || (tableId && isExternalTableID(tableId))) {
    return body
  }
  // ... 从数据库获取文档并补充 _rev
  const db = context.getWorkspaceDB()
  const dbDoc = await db.get<any>(id)
  body._rev = dbDoc._rev
  return body
}
```

**关键点**：
- 外部表**不补 _rev**（外部表不需要 CouchDB 的 rev 机制）
- 内部表通过查询数据库获取当前 `_rev`
- 使用 `Promise.allSettled` 并行处理，仅保留成功的行

---

## 4. 内部表删除流程（internal.ts）

内部表删除逻辑位于 [row/internal.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/internal.ts)。

### 4.1 单行删除 destroy

```typescript
export async function destroy(ctx: UserCtx) {
  const db = context.getWorkspaceDB()
  const source = await utils.getSource(ctx)
  // 1. 计算视图检查
  if (sdk.views.isView(source) && helpers.views.isCalculationView(source)) {
    throw new HTTPError("Cannot delete rows through a calculation view", 400)
  }
  // 2. 获取 table
  let table: Table = sdk.views.isView(source) ? await sdk.views.getTable(source.id) : source
  // 3. 获取 row 和 _rev
  const { _id } = ctx.request.body
  let row = await db.get<Row>(_id)
  let _rev = ctx.request.body._rev || row._rev
  // 4. 校验 tableId 匹配
  if (row.tableId !== table._id) throw "Supplied tableId doesn't match the row's tableId"
  // 5. 输出处理（获取完整关系）
  row = await outputProcessing(table, row, { squash: false, skipBBReferences: true })
  // 6. 删除 link 关系
  await linkRows.updateLinks({ eventType: linkRows.EventType.ROW_DELETE, row, tableId: table._id! })
  // 7. 清理附件
  await AttachmentCleanup.rowDelete(table, [row])
  // 8. 更新静态公式
  await updateRelatedFormula(table, row)
  // 9. 用户元数据表特殊处理
  let response
  if (table._id === InternalTables.USER_METADATA) {
    ctx.params = { id: _id }
    await userController.destroyMetadata(ctx)
    response = ctx.body
  } else {
    response = await db.remove(_id, _rev)
  }
  return { response, row }
}
```

### 4.2 批量删除 bulkDestroy

```typescript
export async function bulkDestroy(ctx: UserCtx) {
  const { tableId } = utils.getSourceId(ctx)
  const table = await sdk.tables.getTable(tableId)
  let { rows } = ctx.request.body

  // 1. 输出处理 - 获得完整行用于自动化
  const processedRows = (await outputProcessing(table, rows, {
    squash: false,
    skipBBReferences: true,
  })) as Row[]

  // 2. 删除关系（异步并发）
  let updates: Promise<any>[] = processedRows.map(row =>
    linkRows.updateLinks({
      eventType: linkRows.EventType.ROW_DELETE,
      row,
      tableId: row.tableId,
    })
  )

  // 3. 删除行数据
  if (tableId === InternalTables.USER_METADATA) {
    // 用户元数据特殊路径
    updates = updates.concat(
      processedRows.map(row => {
        ctx.params = { id: row._id }
        return userController.destroyMetadata(ctx)
      })
    )
  } else {
    // 普通内部表 - 批量删除
    const db = context.getWorkspaceDB()
    await db.bulkDocs(processedRows.map(row => ({ ...row, _deleted: true })))
  }

  // 4. 清理附件
  await AttachmentCleanup.rowDelete(table, processedRows)
  // 5. 更新静态公式
  await updateRelatedFormula(table, processedRows)
  // 6. 等待关系更新完成
  await Promise.all(updates)

  return { response: { ok: true }, rows: processedRows }
}
```

### 4.3 Link 关系删除

通过 `linkRows.updateLinks` 调用，最终执行 [LinkController.rowDeleted()](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/db/linkedRows/LinkController.ts#L299-L308)：

```typescript
async rowDeleted() {
  const row = this._row!
  const linkDocs = await this.getRowLinkDocs(row._id!)
  if (linkDocs.length === 0) return null
  await this._db.bulkRemove(linkDocs, { silenceErrors: true })
  return row
}
```

**原理**：内部表使用独立的 Link Document（关联文档）存储关系，删除行时需要一并删除所有关联文档。

### 4.4 附件清理

[AttachmentCleanup.rowDelete](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/utilities/rowProcessor/attachments.ts#L151-L179)：

- 遍历行中所有附件列（`ATTACHMENTS`、`ATTACHMENT_SINGLE`、`SIGNATURE_SINGLE`）
- 提取每个附件的 `key`
- 调用 `objectStore.deleteFiles` 从对象存储中删除
- **生产环境保护**：检查生产环境是否仍在使用这些附件，避免误删

### 4.5 静态公式更新

[updateRelatedFormula](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/staticFormula.ts#L31-L94)：

- 检查 `table.relatedFormula`，判断是否有依赖本表的静态公式
- 遍历被删除行的关联表和关联行
- 对关联表中所有 `STATIC` 类型的公式字段重新计算并保存

### 4.6 用户元数据特殊路径

当表为 `InternalTables.USER_METADATA` 时：

- **单行**：调用 [userController.destroyMetadata](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/user.ts#L52-L65)
  ```typescript
  export async function destroyMetadata(ctx: UserCtx<void, DeleteUserMetadataResponse>) {
    const db = context.getWorkspaceDB()
    try {
      const dbUser = await sdk.users.get(ctx.params.id)
      await db.remove(dbUser._id!, dbUser._rev)
    } catch (err) {
      // error just means the global user has no config in this app
    }
    ctx.body = { message: `User metadata ${ctx.params.id} deleted.` }
  }
  ```
- **批量**：逐行调用 `destroyMetadata`
- **特点**：删除失败不会抛错（用户可能没有该应用的元数据配置）

### 4.7 单行 vs 批量差异总结

| 维度 | 单行 destroy | 批量 bulkDestroy |
|------|-------------|-----------------|
| 关系删除 | 同步执行 | 并发 Promise.all |
| 行删除方式 | `db.remove(_id, _rev)` | `db.bulkDocs([..._deleted: true])` |
| 用户元数据 | 直接调用 | 逐行调用 |
| 返回值 | `{ response, row }` | `{ response: { ok: true }, rows }` |
| 附件清理 | 传单行数组 | 传多行数组 |
| 静态公式 | 单行触发 | 多行批量触发 |

---

## 5. 外部表删除流程（external.ts）

外部表删除逻辑位于 [row/external.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/external.ts)。

### 5.1 单行删除 destroy

```typescript
export async function destroy(ctx: UserCtx) {
  const source = await utils.getSource(ctx)
  if (sdk.views.isView(source) && helpers.views.isCalculationView(source)) {
    throw new HTTPError("Cannot delete rows through a calculation view", 400)
  }
  const _id = ctx.request.body._id
  const { row } = await handleRequest(Operation.DELETE, source, {
    id: breakRowIdField(_id),
    includeSqlRelationships: IncludeRelationship.EXCLUDE,
  })
  return { response: { ok: true, id: _id }, row }
}
```

### 5.2 批量删除 bulkDestroy

```typescript
export async function bulkDestroy(ctx: UserCtx) {
  const { rows } = ctx.request.body
  const source = await utils.getSource(ctx)
  let promises: Promise<{ row: Row; table: Table }>[] = []
  for (let row of rows) {
    promises.push(
      handleRequest(Operation.DELETE, source, {
        id: breakRowIdField(row._id),
        includeSqlRelationships: IncludeRelationship.EXCLUDE,
      })
    )
  }
  const responses = await Promise.all(promises)
  const finalRows = responses.map(resp => resp.row).filter(row => row && row._id)
  return { response: { ok: true }, rows: finalRows }
}
```

### 5.3 handleRequest 与 Operation.DELETE

`handleRequest` 最终调用 [ExternalRequest.run()](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/ExternalRequest.ts#L642-L801) 执行 DELETE 操作。

DELETE 操作的核心流程：

```
1. cleanupConfig - 清理输入参数
2. prepareFilters - 准备删除过滤条件（基于主键）
3. 安全检查：Deletion must be filtered（防止全表删除）
4. 构建 QueryJson
5. 【关键】删除前移除关系: removeRelationshipsToRow()
6. 执行 makeExternalQuery(json) - 实际删除数据库记录
7. handleManyRelationships - 处理多方关系
8. sqlOutputProcessing - 输出处理
```

### 5.4 removeRelationshipsToRow：外部表关系处理

[ExternalRequest.removeRelationshipsToRow](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/ExternalRequest.ts#L612-L640)：

```typescript
async removeRelationshipsToRow(table: Table, rowId: string) {
  const row = await this.getRow(table, rowId)
  const related = await this.lookupRelations(table._id!, row)
  const promises: Promise<unknown>[] = []
  for (const column of Object.values(table.schema)) {
    if (!isRelationshipColumn(column) || isOneToMany(column)) continue
    const { rows, isMany, tableId } = related[relatedKey]!
    for (const row of rows) {
      const rowId = generateIdForRow(row, table)
      if (isMany) {
        promises.push(this.removeManyToManyRelationships(rowId, table))
      } else {
        promises.push(this.removeOneToManyRelationships(rowId, table, column.fieldName))
      }
    }
  }
  await Promise.all(promises)
}
```

**外部表关系处理方式**：
- **Many-to-Many**：删除中间表（junction table）中的关联记录
- **Many-to-One**：将关联表中的外键字段设为 `null`
- **One-to-Many**：跳过（由多方负责维护）

### 5.5 外部表单行 vs 批量差异

| 维度 | 单行 destroy | 批量 bulkDestroy |
|------|-------------|-----------------|
| 实现方式 | 单次 Operation.DELETE | 多次 Operation.DELETE 并发执行 |
| 并发模式 | 单个请求 | `Promise.all` 并发 |
| 关系处理 | 每个删除前移除关系 | 每个删除前独立移除关系 |
| 返回结果 | `{ response: { ok: true, id }, row }` | `{ response: { ok: true }, rows }` |

> **注意**：外部表的批量删除实际上是多个单行删除的并发执行，并非真正的批量 SQL 操作。

---

## 6. 内部表 vs 外部表删除对比

| 维度 | 内部表（Internal） | 外部表（External） |
|------|------------------|------------------|
| **存储引擎** | CouchDB（文档数据库） | 外部数据源（SQL/NoSQL 等） |
| **关系处理** | 删除独立的 Link Document | 操作外键 / 中间表 |
| **附件清理** | `AttachmentCleanup.rowDelete` | 无（附件字段由外部数据库存储） |
| **静态公式** | `updateRelatedFormula` 更新关联表 | 无（公式在查询时计算） |
| **用户元数据** | 特殊路径，调用 `userController.destroyMetadata` | 无此概念 |
| **_rev 机制** | 需要（CouchDB 乐观锁） | 不需要 |
| **批量删除** | `db.bulkDocs` 真正批量 | 多个单行并发执行 |
| **前置检查** | tableId 匹配校验 | 过滤条件非空检查 |

---

## 7. quotas、事件、WebSocket 与自动化

### 7.1 quotas.removeRow / removeRows

定义在 [pro/src/sdk/quotas/helpers/rows.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/quotas/helpers/rows.ts)：

```typescript
export const removeRow = async ({ tableId }: RowOpts = {}) => {
  return quotas.decrement(StaticQuotaName.ROWS, QuotaUsageType.STATIC, { id: tableId })
}

export const removeRows = async (change: number, { tableId }: RowOpts = {}) => {
  return quotas.decrementMany({
    change,
    name: StaticQuotaName.ROWS,
    type: QuotaUsageType.STATIC,
    opts: { id: tableId },
  })
}
```

**调用位置**（[row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts)）：
- 单行删除：`quotas.removeRow()`
- 批量删除：`quotas.removeRows(rows.length)`
- **例外**：`datasource_plus` 表不扣减行数配额

### 7.2 EventType.ROW_DELETE 事件

通过 `ctx.eventEmitter?.emitRow` 发射，位于 [deleteRow](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L294-L299) 和 [deleteRows](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L263-L269) 中：

```typescript
ctx.eventEmitter?.emitRow({
  eventName: EventType.ROW_DELETE,
  appId,
  row,
  user: sdk.users.getUserContextBindings(ctx.user),
})
```

**用途**：
- 触发自动化中的「行删除触发器」（Row Deleted Trigger）
- 携带数据：appId、被删除的 row、用户上下文

### 7.3 gridSocket.emitRowDeletion

定义在 [websockets/grid.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/websockets/grid.ts#L129-L137)：

```typescript
emitRowDeletion(ctx: Ctx, row: Row) {
  const source = getSourceId(ctx)
  const resourceId = source.viewId ?? source.tableId
  const room = `${ctx.appId}-${resourceId}`
  this.emitToRoom(ctx, room, GridSocketEvent.RowChange, {
    id: row._id,
    row: null,
  })
}
```

**用途**：
- 实时同步网格视图（Grid View）的行删除
- 发送 `RowChange` 事件到对应房间
- `row: null` 表示该行被删除
- 前端收到后从表格中移除对应行

### 7.4 ctx.row 在自动化中的用途

在 `_destroy` 函数中设置：

```typescript
// for automations include the row that was deleted
ctx.row = row || {}
```

**自动化调用链**：

自动化步骤 [deleteRow.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/steps/deleteRow.ts#L38-L44)：

```typescript
await destroy(ctx, { isAutomation: true })
return {
  response: ctx.body,
  row: ctx.row,
  success: ctx.body.ok,
}
```

**用途**：
- 自动化执行删除后，返回被删除的完整行数据
- 供后续自动化步骤使用（如通知、日志记录等）
- `isAutomation: true` 标志会跳过 quotas 检查和事件发射中的配额包装

> **注意**：单行删除时 `ctx.row` 是单个行对象；批量删除时 `ctx.row = {}`（空对象），因为批量删除返回的是行数组，不设置单个 ctx.row。

---

## 8. 完整调用流程图

```
DELETE /api/:sourceId/rows
        │
        ▼
rowController.destroy (index.ts)
        │
        ├─ isDeleteRows? ──► deleteRows() ──► pickApi(tableId).bulkDestroy()
        │                      │                    │
        │                      │                    ├─ 内部: internal.bulkDestroy
        │                      │                    │    ├─ outputProcessing
        │                      │                    │    ├─ linkRows.updateLinks (并发)
        │                      │                    │    ├─ db.bulkDocs(_deleted: true)
        │                      │                    │    ├─ AttachmentCleanup.rowDelete
        │                      │                    │    └─ updateRelatedFormula
        │                      │                    │
        │                      │                    └─ 外部: external.bulkDestroy
        │                      │                         └─ 循环调用 handleRequest(Operation.DELETE)
        │                      │                              └─ ExternalRequest.run
        │                      │                                   ├─ removeRelationshipsToRow
        │                      │                                   └─ makeExternalQuery
        │                      │
        │                      ├─ quotas.removeRows(n)
        │                      ├─ 循环 emit EventType.ROW_DELETE
        │                      └─ 循环 gridSocket.emitRowDeletion
        │
        └─ isDeleteRow? ──► deleteRow() ──► pickApi(tableId).destroy()
                               │                    │
                               │                    ├─ 内部: internal.destroy
                               │                    │    ├─ db.get(_id) 取 row
                               │                    │    ├─ outputProcessing
                               │                    │    ├─ linkRows.updateLinks
                               │                    │    ├─ AttachmentCleanup.rowDelete
                               │                    │    ├─ updateRelatedFormula
                               │                    │    └─ db.remove(_id, _rev)
                               │                    │
                               │                    └─ 外部: external.destroy
                               │                         └─ handleRequest(Operation.DELETE)
                               │
                               ├─ quotas.removeRow()
                               ├─ emit EventType.ROW_DELETE
                               └─ gridSocket.emitRowDeletion
```

---

## 9. 关键差异汇总表

| 特性 | 内部表单行 | 内部表批量 | 外部表单行 | 外部表批量 |
|------|-----------|-----------|-----------|-----------|
| **入口函数** | `internal.destroy` | `internal.bulkDestroy` | `external.destroy` | `external.bulkDestroy` |
| **_rev 补全** | 从 db 取或用请求体 | `processDeleteRowsRequest` 中补全 | 不需要 | 不需要 |
| **关系处理** | 删除 Link Document | 并发删除 Link Document | 删中间表/置空外键 | 逐行删中间表/置空外键 |
| **附件清理** | 有 | 有 | 无 | 无 |
| **静态公式** | 更新关联表 | 更新关联表 | 无 | 无 |
| **用户元数据** | 特殊路径 | 逐行特殊路径 | 无 | 无 |
| **数据库操作** | `db.remove()` | `db.bulkDocs(_deleted)` | `Operation.DELETE` | 多次 `Operation.DELETE` |
| **配额扣减** | `removeRow()` | `removeRows(n)` | `removeRow()` | `removeRows(n)` |
| **事件发射** | 1 次 ROW_DELETE | n 次 ROW_DELETE | 1 次 ROW_DELETE | n 次 ROW_DELETE |
| **WebSocket** | 1 次 emitRowDeletion | n 次 emitRowDeletion | 1 次 emitRowDeletion | n 次 emitRowDeletion |
| **ctx.row** | 单个行对象 | {} 空对象 | 单个行对象 | {} 空对象 |
