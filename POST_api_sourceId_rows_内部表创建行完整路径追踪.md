# POST /api/:sourceId/rows 创建内部表行完整路径追踪

> 场景：普通用户 POST 请求创建内部表行，请求体没有 `_id` 字段。

## 整体调用链路概览

```
路由层 (routes/row.ts)
    ↓ paramResource("sourceId")
    ↓ trimViewRowInfoMiddleware
Controller 层 (controllers/row/index.ts)
    ↓ rowController.save → _save
    ↓ quotas.addAction + quotas.addRow (非 datasource_plus 表)
    ↓ events.action.crudExecuted
SDK 层 (sdk/workspace/rows/internal.ts)
    ↓ sdk.rows.save → pickApi → internal.save
    ↓ inputProcessing
    ↓ linkRows.updateLinks
    ↓ finaliseRow
        ↓ outputProcessing
        ↓ processFormulas (静态)
        ↓ processAIColumns
        ↓ db.put (写入数据库)
        ↓ updateRelatedFormula
        ↓ linkRows.squashLinks
Controller 层 (返回)
    ↓ ctx.eventEmitter.emitRow
    ↓ gridSocket.emitRowUpdate
```

---

## 1. 路由层：paramResource 和 trimViewRowInfoMiddleware

### 1.1 paramResource 中间件

**文件**：[middleware/resourceId.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/middleware/resourceId.ts#L47-L49)

`paramResource("sourceId")` 的作用是从路由参数中提取 `sourceId`，并将其设置到 `ctx.resourceId` 上。

```typescript
export function paramResource(main: string) {
  return new ResourceIdGetter("params").mainResource(main).build()
}
```

`ResourceIdGetter.build()` 返回的中间件会：
- 从 `ctx.request.params`（或 `ctx.params`）中读取参数
- 将 `params[sourceId]` 的值赋值给 `ctx.resourceId`
- 供后续的授权中间件等使用

### 1.2 trimViewRowInfoMiddleware 中间件

**文件**：[middleware/trimViewRowInfo.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/middleware/trimViewRowInfo.ts#L7-L31)

这个中间件的作用是：当 `sourceId` 是一个视图 ID 时，裁剪掉不在视图 schema 中的字段。

处理流程：
1. 从 `ctx` 或 `body._viewId` 中获取 `viewId`
2. 如果没有 `viewId`（只是表 ID），直接 `next()` 跳过
3. 如果是 DELETE 请求，也跳过裁剪
4. 调用 `sdk.views.get(viewId)` 获取视图
5. **请求阶段**：调用 `trimNonViewFields(ctx.request.body, view, "WRITE")` 裁剪请求体中不在视图可写字段列表中的字段
6. `await next()` 进入下游处理
7. **响应阶段**：调用 `trimNonViewFields(ctx.body, view, "READ")` 裁剪响应中不在视图可读字段列表中的字段

对于普通内部表（非视图）创建行的场景，因为 `viewId` 为 `undefined`，所以这个中间件会直接 `next()`，不做任何裁剪。

---

## 2. Controller 层：rowController.save 为什么不会转入 patch

### 2.1 save 入口函数

**文件**：[api/controllers/row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L113-L184)

`_save` 函数是实际的处理逻辑，入口如下：

```typescript
async function _save(
  ctx: UserCtx<SaveRowRequest, SaveRowResponse>,
  { isAutomation = false }: { isAutomation?: boolean } = {}
) {
  const { tableId, viewId } = utils.getSourceId(ctx)
  const sourceId = viewId || tableId
  const appId = ctx.appId
  const body = ctx.request.body

  // 如果有 _id，说明是更新，转去 patch
  if (body && body._id) {
    return patch(ctx as UserCtx<PatchRowRequest, PatchRowResponse>, {
      isAutomation,
    })
  }
  // ... 后续创建逻辑
}
```

### 2.2 为什么不会转入 patch

关键判断在第 129 行：

```typescript
if (body && body._id) {
  return patch(...)
}
```

**判定条件**：请求体中存在 `_id` 字段。

在本次场景中，请求体**没有 `_id`**，所以：
- 条件 `body && body._id` 为 `false`
- 不会转入 `patch` 函数
- 继续往下执行创建行的逻辑

对应的 `_patch` 函数也有一个反向判断（第 69 行）：
```typescript
// if it doesn't have an _id then its save
if (body && !body._id) {
  return save(ctx, { isAutomation })
}
```
这形成了 save 和 patch 之间的互转：有 `_id` 走更新，没 `_id` 走创建。

---

## 3. 非 datasource_plus 表如何触发 quotas.addAction 和 quotas.addRow

**文件**：[api/controllers/row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L134-L165)

`_save` 函数中根据表类型和调用来源有三种分支：

### 3.1 三种分支对比

| 分支条件 | quotas.addAction | quotas.addRow | events.action.crudExecuted 位置 |
|---------|-----------------|---------------|--------------------------------|
| datasource_plus 表 + 非 automation | ✅ 包裹外层 | ❌ 不调用 | 在 addAction 内部的 work 函数中 |
| 非 datasource_plus + isAutomation | ❌ 不调用 | ✅ 包裹 sdk.save | 不调用（automation 场景自己处理） |
| **非 datasource_plus + 非 automation** | **✅ 包裹外层** | **✅ 嵌套在内层** | **在 addAction 内部的 work 函数中** |

### 3.2 普通用户创建内部表的场景

本次场景是**普通用户（非 automation）+ 内部表（非 datasource_plus）**，走的是第三个分支（第 157-164 行）：

```typescript
} else {
  saveResult = await quotas.addAction(ActionType.CRUD, async () => {
    const response = await quotas.addRow(() =>
      sdk.rows.save(sourceId, ctx.request.body, ctx.user?._id)
    )
    events.action.crudExecuted({ type: "create" })
    return response
  })
}
```

执行顺序：
1. 最外层：`quotas.addAction(ActionType.CRUD, workFn)` —— 记录一次 CRUD 操作配额
2. workFn 内部第一层：`quotas.addRow(saveFn)` —— 记录行数配额
3. saveFn 内部：调用 `sdk.rows.save(...)` 执行实际保存
4. sdk.rows.save 返回后：`events.action.crudExecuted({ type: "create" })` —— 发出 CRUD 执行成功事件
5. 最终返回 `response` 给 `quotas.addAction`

**关键点**：
- `quotas.addAction` 在外层，确保 CRUD 操作计数
- `quotas.addRow` 在内层，确保行数计数
- `events.action.crudExecuted` 在 `quotas.addAction` 内部的 work 函数中，只有当 sdk.save 成功后才会触发
- datasource_plus 表不调用 `quotas.addRow`，因为外部表的行数不计入内部配额

---

## 4. sdk.rows.save 如何选择 internal.save

### 4.1 pickApi 选择器

**文件**：[sdk/workspace/rows/rows.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/rows.ts#L24-L43)

`sdk.rows.save` 是一个分发函数，内部通过 `pickApi` 选择具体实现：

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

export async function save(
  sourceId: string,
  row: Row,
  userId: string | undefined,
  opts?: { updateAIColumns: boolean }
) {
  return pickApi(sourceId).save(sourceId, row, userId, opts)
}
```

### 4.2 选择逻辑

1. **判断是否是视图 ID**：如果 `sourceId` 是视图 ID（`isViewId`），先从视图 ID 中提取出表 ID
2. **判断是否是外部表**：调用 `isExternalTableID(tableId)` 检查
   - 如果是外部表（如 MySQL、Postgres 等），返回 `external` 模块
   - 如果是内部表（CouchDB），返回 `internal` 模块

本次场景是内部表，所以最终选择的是 `internal.save`，即：
[sdk/workspace/rows/internal.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/internal.ts#L15-L65) 中的 `save` 函数。

---

## 5. inputProcessing 字段清理详解

**文件**：[utilities/rowProcessor/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/utilities/rowProcessor/index.ts#L221-L287)

`inputProcessing` 是行数据写入数据库前的核心预处理函数，负责清理和规范化输入数据。

### 5.1 函数签名

```typescript
export async function inputProcessing(
  userId: string | null | undefined,
  source: Table | ViewV2,
  row: Row,
  opts?: AutoColumnProcessingOpts
)
```

### 5.2 清理步骤详解

#### 步骤 1：清理 schema 外字段（第 230-242 行）

```typescript
for (const [key, value] of Object.entries(clonedRow)) {
  const field = table.schema[key]
  const isBuiltinColumn = isExternalTableID(table._id!)
    ? isExternalColumnName(key)
    : isInternalColumnName(key)
  // cleanse fields that aren't in the schema
  if (!field && !isBuiltinColumn) {
    delete clonedRow[key]
  }
  // field isn't found - might be a built-in column, skip over it
  if (!field) {
    continue
  }
  // ...
}
```

逻辑：
- 遍历行数据的所有字段
- 如果字段不在 `table.schema` 中，且不是内置列（如 `_id`、`_rev`、`tableId` 等），则**删除**该字段
- 如果是内置列但不在 schema 中，跳过不处理

#### 步骤 2：删除公式字段值（第 243-246 行）

```typescript
if (field.type === FieldType.FORMULA) {
  delete clonedRow[key]
}
```

逻辑：
- 公式字段的值是计算出来的，不允许用户输入
- 直接删除请求体中的公式字段值
- 公式值会在 `finaliseRow` 和 `outputProcessing` 阶段重新计算

#### 步骤 3：删除附件 URL（第 253-276 行）

```typescript
if (field.type === FieldType.ATTACHMENTS) {
  const attachments = clonedRow[key]
  if (attachments?.length) {
    attachments.forEach((attachment: RowAttachment) => {
      delete attachment.url
    })
  }
} else if (
  field.type === FieldType.ATTACHMENT_SINGLE ||
  field.type === FieldType.SIGNATURE_SINGLE
) {
  const attachment = clonedRow[key]
  if (attachment?.url) {
    delete clonedRow[key].url
  }
}
```

逻辑：
- 附件 URL 是读取时动态生成的（通过 `objectStore.getAppFileUrl`）
- 写入数据库时不需要保存 URL，只需要保存 `key`、`name` 等元数据
- 所以需要删除 `attachment.url` 字段
- 处理多种附件类型：多附件 (`ATTACHMENTS`)、单附件 (`ATTACHMENT_SINGLE`)、签名 (`SIGNATURE_SINGLE`)

#### 步骤 4：BB Reference 处理（第 269-275 行）

```typescript
} else if (
  value &&
  (field.type === FieldType.BB_REFERENCE_SINGLE ||
    helpers.schema.isDeprecatedSingleUserColumn(field))
) {
  clonedRow[key] = await processInputBBReference(value, field.subtype)
} else if (value && field.type === FieldType.BB_REFERENCE) {
  clonedRow[key] = await processInputBBReferences(value, field.subtype)
}
```

逻辑：
- BB Reference（用户、部门等引用类型）需要特殊处理
- 调用专门的输入处理函数规范化数据格式

#### 步骤 5：类型强制转换（第 248-249 行）

```typescript
else {
  clonedRow[key] = coerce(value, field.type)
}
```

逻辑：
- 对于非公式字段，根据 schema 中的字段类型进行类型转换
- 比如将字符串转为数字、日期等
- 转换规则定义在 `TYPE_TRANSFORM_MAP` 中

#### 步骤 6：自动列填充（第 284 行）

```typescript
await processAutoColumn(userId, table, clonedRow, opts)
```

`processAutoColumn` 函数（第 64-138 行）处理以下自动列类型：

| 自动列子类型 | 创建时 | 更新时 |
|------------|--------|--------|
| `CREATED_BY` | ✅ 设置为当前用户 ID | ❌ 不更新 |
| `CREATED_AT` | ✅ 设置为当前时间 | ❌ 不更新 |
| `UPDATED_BY` | ✅ 设置为当前用户 ID | ✅ 更新为当前用户 ID |
| `UPDATED_AT` | ✅ 设置为当前时间 | ✅ 更新为当前时间 |
| `AUTO_ID` | ✅ 分配自增 ID | ❌ 不更新 |

判断是否是创建的依据：`const creating = !row._rev`

#### 步骤 7：默认值填充（第 285 行）

```typescript
await processDefaultValues(table, clonedRow)
```

`processDefaultValues` 函数（第 140-188 行）：
- 遍历表 schema 中的所有字段
- 如果字段有 `default` 值且当前行中该字段为空（`null`、空字符串、空数组）
- 则使用模板引擎 `processStringSync` 处理默认值模板（支持 `{{ Current User }}` 等绑定）
- 然后通过 `coerce` 转换为正确的类型

---

## 6. 各关键操作所在层级定位

### 6.1 层级总览表

| 操作 | 所在层级 | 所在文件 | 调用位置 |
|------|---------|---------|---------|
| `paramResource` | 路由中间件层 | [middleware/resourceId.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/middleware/resourceId.ts) | 路由定义中 |
| `trimViewRowInfoMiddleware` | 路由中间件层 | [middleware/trimViewRowInfo.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/middleware/trimViewRowInfo.ts) | 路由定义中 |
| `quotas.addAction` | Controller 层 | [api/controllers/row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L158) | `_save` 函数 |
| `quotas.addRow` | Controller 层 | [api/controllers/row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L159) | `_save` 函数，嵌套在 addAction 内 |
| `events.action.crudExecuted` | Controller 层 | [api/controllers/row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L162) | `_save` 函数，quotas.addAction 的 work 回调中 |
| `sdk.rows.save` | SDK 层 | [sdk/workspace/rows/rows.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/rows.ts#L36-L43) | pickApi 分发 |
| `inputProcessing` | SDK 层 / 工具层 | [utilities/rowProcessor/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/utilities/rowProcessor/index.ts#L221-L287) | internal.save 中调用 |
| `linkRows.updateLinks` | SDK 层 | [sdk/workspace/rows/internal.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/rows/internal.ts#L54-L59) | internal.save 中，inputProcessing 之后 |
| `finaliseRow` | SDK 层 / Controller 混合 | [api/controllers/row/staticFormula.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/staticFormula.ts#L137-L183) | internal.save 末尾调用 |
| `ctx.eventEmitter.emitRow` | Controller 层 | [api/controllers/row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L168-L174) | `_save` 函数，sdk.save 返回后 |
| `gridSocket.emitRowUpdate` | Controller 层 | [api/controllers/row/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/row/index.ts#L178) | `_save` 函数末尾 |

### 6.2 各操作详细说明

#### linkRows.updateLinks

**位置**：SDK 层的 `internal.save` 函数中（第 54-59 行）

**作用**：更新关联行的链接文档。当保存一行数据时，如果该行有关联字段（Link 类型），需要更新对应的链接文档（LinkDocument）来维护关系。

**调用时机**：`inputProcessing` 之后，`finaliseRow` 之前。

```typescript
row = (await linkRows.updateLinks({
  eventType: linkRows.EventType.ROW_SAVE,
  row,
  tableId: row.tableId,
  table,
})) as Row
```

#### finaliseRow

**位置**：SDK 层 `internal.save` 的末尾调用，但函数定义在 `api/controllers/row/staticFormula.ts` 中（跨层调用）

**作用**：行保存的最后一步，完成：
1. `outputProcessing` —— 富集行数据（关联、附件 URL 等）
2. `processFormulas` —— 计算静态公式并存入数据库
3. `processAIColumns` —— 计算 AI 列
4. `db.put(row)` —— 实际写入数据库
5. `updateRelatedFormula` —— 更新关联表中的静态公式
6. `linkRows.squashLinks` —— 压平关联关系用于返回

**返回值**：`{ row, squashed, table }` —— 富集的行、压平的行、表对象

#### events.action.crudExecuted

**位置**：Controller 层 `_save` 函数中，`quotas.addAction` 的 work 回调内部

**作用**：发出 CRUD 操作执行成功的审计/分析事件

**触发时机**：`sdk.rows.save` 成功返回后，在 `quotas.addAction` 的回调函数中调用。只有在非 automation 场景才会在这里触发。

```typescript
saveResult = await quotas.addAction(ActionType.CRUD, async () => {
  const response = await quotas.addRow(() =>
    sdk.rows.save(sourceId, ctx.request.body, ctx.user?._id)
  )
  events.action.crudExecuted({ type: "create" })
  return response
})
```

#### ctx.eventEmitter.emitRow

**位置**：Controller 层 `_save` 函数中，`sdk.rows.save` 返回之后

**作用**：通过事件发射器发出行保存事件，用于触发自动化触发器等后续流程

**事件类型**：`EventType.ROW_SAVE`

**携带数据**：appId、row、table、user（用户上下文绑定）

```typescript
ctx.eventEmitter?.emitRow({
  eventName: EventType.ROW_SAVE,
  appId,
  row,
  table,
  user: sdk.users.getUserContextBindings(ctx.user),
})
```

#### gridSocket.emitRowUpdate

**位置**：Controller 层 `_save` 函数的最末尾

**作用**：通过 WebSocket（Grid Socket）向前端实时推送行更新消息，使前端表格能实时看到新增的行

**文件**：[websockets/grid.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/websockets/grid.ts)

```typescript
gridSocket?.emitRowUpdate(ctx, row || squashed)
```

---

## 7. 完整时序图（文字版）

```
POST /api/:sourceId/rows (body 无 _id)
│
├─→ paramResource("sourceId")
│    └─ 设置 ctx.resourceId = params.sourceId
│
├─→ trimViewRowInfoMiddleware
│    ├─ 检查是否是视图
│    └─ 非视图 → 直接 next()
│
└─→ rowController.save
     │
     └─→ _save
          │
          ├─ 检查 body._id → 没有，不转 patch
          │
          ├─ quotas.addAction(ActionType.CRUD, workFn)  ← 外层：CRUD 操作配额
          │    │
          │    ├─ quotas.addRow(saveFn)  ← 内层：行数配额
          │    │    │
          │    │    └─ sdk.rows.save(sourceId, body, userId)
          │    │         │
          │    │         └─ pickApi(sourceId) → internal.save
          │    │              │
          │    │              ├─ 生成 row._id（db.generateRowID）
          │    │              ├─ inputProcessing  ← 字段清理
          │    │              │    ├─ 删除 schema 外字段
          │    │              │    ├─ 删除公式字段值
          │    │              │    ├─ 删除附件 url
          │    │              │    ├─ BB Reference 处理
          │    │              │    ├─ 类型转换 coerce
          │    │              │    ├─ 自动列填充
          │    │              │    └─ 默认值填充
          │    │              │
          │    │              ├─ 行数据验证
          │    │              │
          │    │              ├─ linkRows.updateLinks  ← 更新关联文档
          │    │              │
          │    │              └─ finaliseRow  ← 最终保存
          │    │                   ├─ outputProcessing（富集）
          │    │                   ├─ processFormulas（静态公式）
          │    │                   ├─ processAIColumns
          │    │                   ├─ db.put  ← 写入数据库
          │    │                   ├─ updateRelatedFormula
          │    │                   └─ linkRows.squashLinks
          │    │
          │    └─ events.action.crudExecuted({ type: "create" })  ← CRUD 事件
          │
          ├─ ctx.eventEmitter.emitRow  ← 行保存事件（触发自动化等）
          │
          └─ gridSocket.emitRowUpdate  ← WebSocket 实时推送
```
