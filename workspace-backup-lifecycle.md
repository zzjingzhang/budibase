# Workspace 备份生命周期追踪

本文档追踪 workspace 备份从**手动导出**到**队列恢复失败**的完整生命周期。

---

## 目录

1. [手动导出 API 端点](#1-手动导出-api-端点)
2. [导出逻辑 exports.ts](#2-导出逻辑-exportsts)
3. [导入逻辑 imports.ts](#3-导入逻辑-importsts)
4. [Pro 备份队列处理](#4-pro-备份队列处理)
5. [恢复失败回滚流程](#5-恢复失败回滚流程)
6. [完整生命周期流程图](#6-完整生命周期流程图)

---

## 1. 手动导出 API 端点

### 1.1 路由定义

路由定义在 [backup.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/backup.ts#L5-L10) 中：

```typescript
builderRoutes
  .post(
    "/api/backups/export",
    ensureTenantAppOwnershipMiddleware,
    controller.exportAppDump
  )
```

### 1.2 Tenant App Ownership 检查

使用 [ensureTenantAppOwnershipMiddleware](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/middleware/ensureTenantAppOwnership.ts#L4-L19) 中间件验证租户所有权：

```typescript
export async function ensureTenantAppOwnershipMiddleware(
  ctx: UserCtx,
  next: () => Promise<void> | void
) {
  const appId = await utils.getWorkspaceIdFromCtx(ctx)
  if (!appId) {
    ctx.throw(400, "appId must be provided")
  }

  const appTenantId = context.getTenantIDFromWorkspaceID(appId)
  const tenantId = tenancy.getTenantId()

  if (appTenantId !== tenantId) {
    ctx.throw(403, "Unauthorized")
  }
  await next()
}
```

**检查逻辑：**
- 从请求上下文中获取 `appId`（即 workspaceId）
- 从 `appId` 中解析出租户 ID
- 与当前请求的租户 ID 对比
- 不匹配则返回 403 错误

### 1.3 控制器处理

控制器 [backup.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/backup.ts#L14-L40) 中的 `exportAppDump` 函数：

```typescript
export async function exportAppDump(
  ctx: Ctx<ExportWorkspaceDumpRequest, ExportWorkspaceDumpResponse>
) {
  const { appId: workspaceId } = ctx.query as any
  const { excludeRows, encryptPassword } = ctx.request.body

  const [workspace] = await db.getWorkspacesByIDs([workspaceId])
  const workspaceName = workspace.name

  // 移除 120 秒请求超时限制
  ctx.req.setTimeout(0)

  const extension = encryptPassword ? "enc.tar.gz" : "tar.gz"
  const backupIdentifier = `${workspaceName}-export-${new Date().getTime()}.${extension}`
  ctx.attachment(backupIdentifier)
  ctx.body = await sdk.backups.streamExportWorkspace({
    workspaceId,
    excludeRows,
    encryptPassword,
  })

  // 触发导出事件
  await context.doInWorkspaceContext(workspaceId, async () => {
    const appDb = context.getWorkspaceDB()
    const app = await appDb.get<Workspace>(DocumentType.WORKSPACE_METADATA)
    await events.app.exported(app)
  })
}
```

**关键点：**

| 步骤 | 说明 | 代码位置 |
|------|------|----------|
| **设置无限超时** | `ctx.req.setTimeout(0)` 移除默认的 120 秒超时，防止大型备份导出超时 | [backup.ts#L24](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/backup.ts#L24-L24) |
| **扩展名决定** | 有 `encryptPassword` 时用 `.enc.tar.gz`，否则用 `.tar.gz` | [backup.ts#L26](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/backup.ts#L26-L26) |
| **调用导出函数** | 调用 `sdk.backups.streamExportWorkspace` 获取可读流作为响应 | [backup.ts#L29-L33](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/backup.ts#L29-L33) |

---

## 2. 导出逻辑 exports.ts

导出核心逻辑在 [exports.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/backups/exports.ts) 中。

### 2.1 流式导出入口

[streamExportWorkspace](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/backups/exports.ts#L202-L217) 函数：

```typescript
export async function streamExportWorkspace({
  workspaceId,
  excludeRows,
  encryptPassword,
}: {
  workspaceId: string
  excludeRows: boolean
  encryptPassword?: string
}) {
  const tmpPath = await exportWorkspace(workspaceId, {
    excludeRows,
    tar: true,
    encryptPassword,
  })
  return streamFile(tmpPath)
}
```

### 2.2 核心导出函数 exportWorkspace

[exportWorkspace](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/backups/exports.ts#L105-L193) 是主要的导出函数，执行以下步骤：

#### 步骤 1: 从对象存储检索目录

```typescript
const prodWorkspaceId = dbCore.getProdWorkspaceID(workspaceId)
const workspacePath = `${prodWorkspaceId}/`

const toExclude = [/\/\..+/]
if (config?.excludeRows) {
  toExclude.push(/\/attachments\/.*/)
}

const tmpPath = await objectStore.retrieveDirectory(
  ObjectStoreBuckets.APPS,
  workspacePath,
  toExclude
)
```

**说明：**
- 获取生产环境 workspace ID
- 从对象存储 `APPS` bucket 下载 workspace 目录到临时目录
- 始终排除隐藏文件（`/\/\..+/`）
- 当 `excludeRows=true` 时，额外排除 `attachments` 目录下的所有文件

#### 步骤 2: 整理目录结构

```typescript
const downloadedPath = join(tmpPath, workspacePath)
if (fs.existsSync(downloadedPath)) {
  const allFiles = await fsp.readdir(downloadedPath)
  for (let file of allFiles) {
    const path = join(downloadedPath, file)
    // 移出 workspace 目录，简化结构
    await fsp.rename(path, join(downloadedPath, "..", file))
  }
  await fsp.rmdir(downloadedPath)
}
```

#### 步骤 3: 导出数据库 (exportDB)

```typescript
const dbPath = join(tmpPath, DB_EXPORT_FILE)
await exportDB(workspaceId, {
  filter: defineFilter(config?.excludeRows),
  exportPath: dbPath,
})
```

[exportDB](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/backups/exports.ts#L55-L83) 函数使用 CouchDB 的 dump 功能导出数据库：

```typescript
export async function exportDB(
  dbName: string,
  opts: DBDumpOpts = {}
): Promise<string> {
  const exportOpts = {
    filter: opts?.filter,
    batch_size: 1000,
    batch_limit: 5,
    style: "main_only",
  } as const
  return dbCore.doWithDB(dbName, async db => {
    if (opts?.exportPath) {
      const path = opts?.exportPath
      const writeStream = fs.createWriteStream(path)
      await db.dump(writeStream, exportOpts)
      return path
    }
    // ...
  })
}
```

**数据库过滤器** [defineFilter](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/backups/exports.ts#L85-L96)：

```typescript
function defineFilter(excludeRows?: boolean) {
  const ids = [
    USER_METDATA_PREFIX,      // 用户元数据
    LINK_USER_METADATA_PREFIX, // 关联用户元数据
    AUTOMATION_LOG_PREFIX,     // 自动化日志
  ]
  if (excludeRows) {
    ids.push(TABLE_ROW_PREFIX) // 表行数据
  }
  return (doc: any) =>
    !ids.map(key => doc._id.includes(key)).reduce((prev, curr) => prev || curr)
}
```

**过滤的文档类型：**

| 类型 | 前缀常量 | 说明 |
|------|----------|------|
| 用户元数据 | `USER_METDATA_PREFIX` | 用户相关的元数据，始终排除 |
| 关联用户元数据 | `LINK_USER_METADATA_PREFIX` | 用户与其他实体的关联数据，始终排除 |
| 自动化日志 | `AUTOMATION_LOG_PREFIX` | 自动化执行日志，始终排除 |
| 表行数据 | `TABLE_ROW_PREFIX` | 数据表的行记录，仅在 `excludeRows=true` 时排除 |

#### 步骤 4: 可选加密（非附件文件）

```typescript
if (config?.encryptPassword) {
  const processDirectory = async (dirPath: string, relativePath = "") => {
    for (let file of await fsp.readdir(dirPath)) {
      const fullPath = join(dirPath, file)
      // 跳过 attachments - 太大不加密
      if (file !== ATTACHMENT_DIRECTORY) {
        const stats = await fsp.lstat(fullPath)
        if (stats.isFile()) {
          await encryption.encryptFile(
            { dir: dirPath, filename: file },
            config.encryptPassword!
          )
          await fsp.rm(fullPath)
        } else if (stats.isDirectory()) {
          await processDirectory(fullPath, relativeFilePath)
        }
      }
    }
  }
  await processDirectory(tmpPath)
}
```

**加密规则：**
- 只加密非 `attachments` 目录下的文件
- 附件文件不加密（体积太大）
- 加密后删除原文件，保留 `.enc` 后缀的加密文件

#### 步骤 5: 打包 tar

```typescript
if (config?.tar) {
  const tarPath = await tarFilesToTmp(tmpPath, await fsp.readdir(tmpPath))
  await fsp.rm(tmpPath, { recursive: true, force: true })
  return tarPath
}
```

[tarFilesToTmp](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/backups/exports.ts#L33-L46) 使用 gzip 压缩创建 tar 包。

---

## 3. 导入逻辑 imports.ts

导入核心逻辑在 [imports.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/backups/imports.ts) 中。

### 3.1 主导入函数 importApp

[importApp](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/backups/imports.ts#L247-L372) 执行完整的导入流程。

#### 步骤 1: 解 tar

```typescript
const isTar = template.file && template?.file?.type?.endsWith("gzip")
if (template.file && (isTar || isDirectory)) {
  tmpPath = isTar ? await untarFile(template.file) : template.file.path
  // ...
}
```

[untarFile](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/backups/imports.ts#L148-L157) 函数：

```typescript
export async function untarFile(file: { path: string }) {
  const tmpPath = join(budibaseTempDir(), uuid())
  await fsp.mkdir(tmpPath)
  await tar.extract({
    cwd: tmpPath,
    file: file.path,
  })
  return tmpPath
}
```

#### 步骤 2: 解密文件

```typescript
if (isTar && template.file.password) {
  await decryptFiles(tmpPath, template.file.password)
}
const contents = await fsp.readdir(tmpPath)
const stillEncrypted = !!contents.find(name => name.endsWith(".enc"))
if (stillEncrypted) {
  throw new Error("Files are encrypted but no password has been supplied.")
}
```

[decryptFiles](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/backups/imports.ts#L159-L184) 函数：

```typescript
async function decryptFiles(path: string, password: string) {
  try {
    const processDirectory = async (dirPath: string) => {
      for (let file of await fsp.readdir(dirPath)) {
        const inputPath = join(dirPath, file)
        if (!inputPath.endsWith(ATTACHMENT_DIRECTORY)) {
          const stats = await fsp.lstat(inputPath)
          if (stats.isFile() && inputPath.endsWith(".enc")) {
            const outputPath = inputPath.replace(/\.enc$/, "")
            await encryption.decryptFile(inputPath, outputPath, password)
            await fsp.rm(inputPath)
          } else if (stats.isDirectory()) {
            await processDirectory(inputPath)
          }
        }
      }
    }
    await processDirectory(path)
  } catch (err: any) {
    if (err.message === "incorrect header check") {
      throw new Error("File cannot be imported")
    }
    throw err
  }
}
```

**解密规则：**
- 跳过 `attachments` 目录
- 只解密 `.enc` 后缀的文件
- 解密后删除加密文件
- 密码错误（"incorrect header check"）时抛出 "File cannot be imported" 错误

#### 步骤 3: 阻止 Plugin 文件

```typescript
const isPlugin = !!contents.find(name => name === "plugin.min.js")
if (isPlugin) {
  throw new Error("Supplied file is a plugin - cannot import as app.")
}
```

**安全检查：** 如果发现 `plugin.min.js` 文件，拒绝导入（防止将插件当作应用导入）。

#### 步骤 4: 验证有效性

```typescript
const isInvalid = !contents.find(name => name === DB_EXPORT_FILE)
if (isInvalid) {
  throw new Error(
    "App export does not appear to be valid - no DB file found."
  )
}
```

必须包含 `db.txt`（`DB_EXPORT_FILE`）文件才是有效的应用导出。

#### 步骤 5: 上传对象存储内容

```typescript
if (importOpts.importObjStoreContents) {
  const promises = []
  const excludedFiles = [GLOBAL_DB_EXPORT_FILE, DB_EXPORT_FILE]

  for (let filename of contents) {
    const path = join(tmpPath, filename)
    if (excludedFiles.includes(filename)) {
      continue
    }
    filename = join(objectStoreProdAppId, filename)
    if ((await fsp.lstat(path)).isDirectory()) {
      promises.push(
        objectStore.uploadDirectory(ObjectStoreBuckets.APPS, path, filename)
      )
    } else {
      promises.push(
        objectStore.upload({
          bucket: ObjectStoreBuckets.APPS,
          path,
          filename,
        })
      )
    }
  }
  await Promise.all(promises)
  // ...
}
```

**上传逻辑：**
- 排除数据库导出文件（`db.txt` 和 `global.txt`）
- 目录用 `uploadDirectory`，文件用 `upload`
- 上传到目标 workspace 的对象存储路径

#### 步骤 6: 删除 Stale 非附件文件

```typescript
const uploadedFiles = await fsp.readdir(tmpPath, { recursive: true })

const filesToDelete: string[] = []
await utils.parallelForeach(
  objectStore.listAllObjects(
    objectStore.ObjectStoreBuckets.APPS,
    objectStoreProdAppId
  ),
  async file => {
    if (!file.Key) {
      return
    }
    const prefix = `${objectStoreProdAppId}/`
    if (!file.Key.startsWith(prefix)) {
      return
    }
    const relativePath = file.Key.slice(prefix.length)
    // 不删除附件文件 - 它们被生产环境行引用，不在导出中也无法重新上传
    if (
      !relativePath.startsWith(`${ATTACHMENT_DIRECTORY}/`) &&
      !uploadedFiles.includes(relativePath)
    ) {
      filesToDelete.push(file.Key)
    }
  },
  5
)

if (filesToDelete.length) {
  await objectStore.deleteFiles(
    objectStore.ObjectStoreBuckets.APPS,
    filesToDelete
  )
}
```

**删除规则：**
- 列出目标 workspace 在对象存储中的所有文件
- 对比本次上传的文件列表
- **只删除不在新上传列表中的非附件文件**
- `attachments/` 目录下的文件永远不删除（因为生产环境行数据引用它们，且不在导出范围内）

#### 步骤 7: 加载数据库 (load DB)

```typescript
dbStream = fs.createReadStream(join(tmpPath, DB_EXPORT_FILE))
// ...
const { ok } = await db.load(dbStream)
if (!ok) {
  throw "Error loading database dump from template."
}
```

使用 CouchDB 的 load 功能将数据库 dump 加载到目标数据库。

#### 步骤 8: 重写 Attachment Key

[updateAttachmentColumns](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/backups/imports.ts#L64-L98) 函数：

```typescript
export async function updateAttachmentColumns(prodAppId: string, db: Database) {
  const tables = await sdk.tables.getAllInternalTables(db)
  let updatedRows: Row[] = []
  for (let table of tables) {
    const { rows, columns } = await sdk.rows.getRowsWithAttachments(
      db.name,
      table
    )
    updatedRows = updatedRows.concat(
      rows.map(row => {
        for (let column of columns) {
          const columnType = table.schema[column].type
          if (
            columnType === FieldType.ATTACHMENTS &&
            Array.isArray(row[column])
          ) {
            row[column] = row[column].map((attachment: RowAttachment) =>
              rewriteAttachmentUrl(prodAppId, attachment)
            )
          } else if (
            (columnType === FieldType.ATTACHMENT_SINGLE ||
              columnType === FieldType.SIGNATURE_SINGLE) &&
            row[column]
          ) {
            row[column] = rewriteAttachmentUrl(prodAppId, row[column])
          }
        }
        return row
      })
    )
  }
  await db.bulkDocs(updatedRows)
}
```

[rewriteAttachmentUrl](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/backups/imports.ts#L49-L62) 函数：

```typescript
function rewriteAttachmentUrl(appId: string, attachment: RowAttachment) {
  // URL 格式: /prod-budi-app-assets/appId/attachments/file.csv
  const urlParts = attachment.key?.split("/") || []
  // 移除旧的 app ID
  urlParts.shift()
  // 添加新的 app ID
  urlParts.unshift(appId)
  const key = urlParts.join("/")
  return {
    ...attachment,
    key,
    url: "", // 检索时根据 key 计算
  }
}
```

**重写逻辑：**
- 遍历所有内部表的所有行
- 找到附件类型列（ATTACHMENTS、ATTACHMENT_SINGLE、SIGNATURE_SINGLE）
- 将附件 key 中的旧 appId 替换为新的 prodAppId
- 清空 url 字段（运行时根据 key 重新计算）

#### 步骤 9: 更新自动化配置

[updateAutomations](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/backups/imports.ts#L100-L126) 函数：

```typescript
async function updateAutomations(prodAppId: string, db: Database) {
  const automations = (
    await db.allDocs(
      getAutomationParams(null, {
        include_docs: true,
      })
    )
  ).rows.map(row => row.doc) as Automation[]
  const devId = dbCore.getDevWorkspaceID(prodAppId)
  let toSave: Automation[] = []
  for (let automation of automations) {
    const oldDevAppId = automation.appId,
      oldProdAppId = dbCore.getProdWorkspaceID(automation.appId)
    if (
      automation.definition.trigger?.stepId === AutomationTriggerStepId.WEBHOOK
    ) {
      const old = automation.definition.trigger.inputs as WebhookTriggerInputs
      automation.definition.trigger.inputs = {
        schemaUrl: old.schemaUrl.replace(oldDevAppId, devId),
        triggerUrl: old.triggerUrl.replace(oldProdAppId, prodAppId),
      }
    }
    automation.appId = devId
    toSave.push(automation)
  }
  await db.bulkDocs(toSave)
}
```

**更新内容：**
- 将所有自动化的 `appId` 更新为新的 dev workspace ID
- 对于 Webhook 触发器，更新 `schemaUrl` 和 `triggerUrl` 中的 app ID

#### 步骤 10: 清理 LiteLLM 导入数据

[sanitizeLiteLLMImportData](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/backups/imports.ts#L201-L245) 函数：

```typescript
async function sanitizeLiteLLMImportData(db: Database) {
  // 移除 LiteLLM key 配置
  const keyDocId = docIds.getLiteLLMKeyID()
  const keyDoc = await db.tryGet<LiteLLMKeyConfig>(keyDocId)
  if (keyDoc) {
    await db.remove(keyDoc)
  }

  // 处理 AI 配置中的 liteLLMModelId
  const aiConfigs = await db.allDocs<CustomAIProviderConfig>(
    docIds.getDocParams(DocumentType.AI_CONFIG, undefined, {
      include_docs: true,
    })
  )

  const updatedAIConfigs = aiConfigs.rows
    .map(row => row.doc)
    .filter((doc): doc is CustomAIProviderConfig => !!doc)
    .map(doc => ({
      ...doc,
      liteLLMModelId:
        doc.provider === BUDIBASE_AI_PROVIDER_ID && !environment.SELF_HOSTED
          ? doc.liteLLMModelId
          : IMPORT_PENDING_LITELLM_MODEL_ID,
    }))

  if (updatedAIConfigs.length) {
    await db.bulkDocs(updatedAIConfigs)
  }
}
```

**清理规则：**
- **删除**导入的 LiteLLM 密钥配置（安全考虑）
- 对于 Budibase AI provider 且非自托管环境，保留 `liteLLMModelId`
- 其他情况将 `liteLLMModelId` 设为 `__bb_import_pending_litellm_model__`（待用户重新配置）

---

## 4. Pro 备份队列处理

Pro 版本的备份队列处理在 [pro/src/sdk/backups/](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/backups/) 目录下。

### 4.1 备份状态定义

[backup.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/types/src/documents/workspace/backup.ts#L9-L14) 中的状态枚举：

```typescript
export enum BackupStatus {
  STARTED = "started",
  PENDING = "pending",
  COMPLETE = "complete",
  FAILED = "failed",
}
```

### 4.2 队列初始化

[queue.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/backups/queue.ts#L1-L22)：

```typescript
export function init() {
  backupQueue = new queue.BudibaseQueue<WorkspaceBackupQueueData>(
    queue.JobQueue.APP_BACKUP,
    {
      maxStalledCount: 3,
      jobOptions: {
        attempts: 3,
        removeOnFail: true,
        removeOnComplete: true,
      },
    }
  )
}
```

### 4.3 触发器 - 创建 PENDING Metadata

#### 触发备份

[triggerWorkspaceBackup](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/backups/backup.ts#L152-L195)：

```typescript
async function triggerWorkspaceBackup(
  workspaceId: string,
  trigger: BackupTrigger,
  opts: { createdBy?: string; name?: string } = {}
): Promise<string | undefined> {
  // 立即存储，获取 rev 和 id，状态为 PENDING
  let backup
  try {
    backup = await storeWorkspaceBackupMetadata({
      appId: workspaceId,
      trigger,
      timestamp: new Date().toISOString(),
      status: BackupStatus.PENDING,
      type: BackupType.BACKUP,
      ...opts,
    })
  } catch (err: any) {
    if (err.status === 409) {
      return // 同一毫秒已存在备份
    } else {
      throw err
    }
  }
  // 启动任务
  await getBackupQueue().add({
    docId: backup.id,
    docRev: backup.rev,
    appId: workspaceId,
    export: {
      trigger,
      ...opts,
    },
  })
  // ... 触发事件
  return backup.id
}
```

#### 触发恢复

[triggerWorkspaceRestore](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/backups/backup.ts#L197-L234)：

```typescript
async function triggerWorkspaceRestore(
  workspaceId: string,
  backupId: string,
  nameForBackup: string,
  createdBy?: string
): Promise<{ restoreId: string; metadata: any } | void> {
  const metadata = await getWorkspaceBackup(backupId)
  let restore
  try {
    restore = await storeWorkspaceBackupMetadata({
      appId: workspaceId,
      timestamp: new Date().toISOString(),
      status: BackupStatus.PENDING,
      type: BackupType.RESTORE,
      createdBy,
    })
  } catch (err: any) {
    if (err?.status === 409) {
      return
    } else {
      throw err
    }
  }
  await getBackupQueue().add({
    appId: workspaceId,
    docId: restore.id,
    docRev: restore.rev,
    import: {
      nameForBackup,
      backupId,
      createdBy,
    },
  })
  return { restoreId: restore.id, metadata }
}
```

**PENDING 阶段关键点：**
1. 先创建元数据文档，状态为 `PENDING`
2. 将任务添加到 Bull 队列
3. 返回备份/恢复 ID

### 4.4 队列处理器初始化

[processing.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/backups/processing.ts#L21-L39) 中的 `init` 函数：

```typescript
export async function init(opts: BackupProcessingOpts) {
  getBackupQueue().process(async (job: Job) => {
    const data = job.data as WorkspaceBackupQueueData
    try {
      if (data.export) {
        console.log("Exporting app backup:", data.appId, data.export.trigger)
        return exportProcessor(job, opts)
      } else if (data.import) {
        console.log("Importing app backup:", data.appId, data.import.backupId)
        return importProcessor(job, opts)
      }
    } catch (err: any) {
      logging.logAlert(
        `Failed to perform backup for app ID: ${data.appId}`,
        err
      )
    }
  })
}
```

### 4.5 导出处理器

[exportProcessor](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/backups/processing.ts#L390-L418)：

```typescript
async function exportProcessor(job: Job, opts: BackupProcessingOpts) {
  const data: WorkspaceBackupQueueData = job.data
  const appId = data.appId,
    trigger = data.export!.trigger,
    name = data.export!.name
  const tenantId = tenancy.getTenantIDFromWorkspaceID(appId) as string
  await tenancy.doInTenant(tenantId, async () => {
    try {
      const { rev } = await backups.updateBackupStatus(
        data.docId,
        BackupStatus.STARTED
      )
      return runBackup(trigger, tenantId, appId, {
        processing: opts,
        doc: { id: data.docId, rev },
        name,
      })
    } catch (err) {
      logging.logAlert("App backup error", err)
      const errorMessage = err instanceof Error ? err.message : String(err)
      await backups.trackBackupError(
        appId,
        data.docId,
        `Backup export failed: ${errorMessage}`
      )
    }
  })
}
```

**导出流程：**
1. 更新状态为 `STARTED`
2. 调用 `runBackup` 执行实际备份
3. 失败时记录错误

---

## 5. 恢复失败回滚流程

恢复（import/restore）的完整流程和失败回滚是最复杂的部分，在 [importProcessor](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/backups/processing.ts#L284-L388) 中实现。

### 5.1 恢复完整流程

```typescript
async function importProcessor(job: Job, opts: BackupProcessingOpts) {
  const data: WorkspaceBackupQueueData = job.data
  const appId = data.appId,
    backupId = data.import!.backupId,
    nameForBackup = data.import!.nameForBackup,
    createdBy = data.import!.createdBy
  const tenantId = tenancy.getTenantIDFromWorkspaceID(appId) as string
  return tenancy.doInTenant(tenantId, async () => {
    const devWorkspaceId = dbCore.getDevWorkspaceID(appId)
    const tempWorkspaceId = `${devWorkspaceId}_temp_${Date.now()}`

    // 1. 更新状态为 STARTED
    const { rev } = await backups.updateRestoreStatus(
      data.docId,
      data.docRev,
      BackupStatus.STARTED
    )
    
    // 2. 先导出当前状态作为回滚备份
    await runBackup(BackupTrigger.RESTORING, tenantId, appId, {
      processing: opts,
      createdBy,
      name: nameForBackup,
    })
    
    // 3. 下载备份文件到本地
    const path = await backups.downloadWorkspaceBackup(backupId)
    
    let status = BackupStatus.COMPLETE
    let promotedWorkspaceFiles: PromoteWorkspaceFilesResult | null = null
    
    try {
      // 4. 导入到临时数据库
      await opts.importWorkspaceFn(
        devWorkspaceId,
        dbCore.getDB(tempWorkspaceId),
        {
          file: { type: "application/gzip", path },
          key: path,
        },
        {
          objectStoreAppId: tempWorkspaceId,
          preserveLiteLLMConfig: true,
        }
      )
      
      // 5. 提升对象存储文件（promoteWorkspaceFiles）
      promotedWorkspaceFiles = await promoteWorkspaceFiles(
        tempWorkspaceId,
        devWorkspaceId
      )

      // 6. 替换数据库（Replication 切换）
      await removeExistingApp(devWorkspaceId)
      await new db.Replication({
        source: tempWorkspaceId,
        target: devWorkspaceId,
      }).replicate()
      
      // 7. 清理提升的文件
      try {
        await cleanupPromotedWorkspaceFiles(
          promotedWorkspaceFiles.sourceFileKeys,
          promotedWorkspaceFiles.targetFileKeys,
          devWorkspaceId
        )
      } catch (cleanupErr) {
        console.log("Failed to cleanup promoted restore files:", cleanupErr)
      }
    } catch (err: any) {
      // 失败回滚
      if (promotedWorkspaceFiles) {
        try {
          await rollbackPromotedWorkspaceFiles(
            promotedWorkspaceFiles.targetFileKeys,
            promotedWorkspaceFiles.rollbackFiles
          )
        } catch (rollbackErr) {
          console.log("Failed to rollback promoted restore files:", rollbackErr)
        }
      }
      logging.logAlert("App restore error", err)
      status = BackupStatus.FAILED
      // 跟踪错误
      const errorMessage = err instanceof Error ? err.message : String(err)
      await backups.trackBackupError(
        appId,
        backupId,
        `Backup restore failed: ${errorMessage}`
      )
    } finally {
      // 清理临时资源
      try {
        const tempDb = dbCore.getDB(tempWorkspaceId, { skip_setup: true })
        await tempDb.destroy()
      } catch (cleanupErr) { /* 忽略 */ }
      try {
        await clearWorkspaceFiles(tempWorkspaceId)
      } catch (cleanupErr) { /* 忽略 */ }
    }
    
    // 更新最终状态
    await backups.updateRestoreStatus(data.docId, rev, status)
    if (fs.existsSync(path)) {
      fs.rmSync(path, { force: true })
    }
  })
}
```

### 5.2 promoteWorkspaceFiles - 提升文件

[promoteWorkspaceFiles](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/backups/processing.ts#L130-L185)：

```typescript
async function promoteWorkspaceFiles(
  sourceWorkspaceId: string,
  workspaceId: string
): Promise<PromoteWorkspaceFilesResult> {
  const sourceProdWorkspaceId = dbCore.getProdWorkspaceID(sourceWorkspaceId)
  const targetProdWorkspaceId = dbCore.getProdWorkspaceID(workspaceId)
  const sourcePrefix = `${sourceProdWorkspaceId}/`
  const targetPrefix = `${targetProdWorkspaceId}/`
  const rollbackPrefix = `${sourcePrefix}__restore_rollback/${Date.now()}/`

  const sourceFileKeys = await listAppFiles(sourcePrefix)
  const uploadedTargetKeys = new Set<string>()
  const rollbackFiles: PromoteWorkspaceFileRollback[] = []
  
  try {
    for (const sourceKey of sourceFileKeys) {
      const relativePath = sourceKey.startsWith(sourcePrefix)
        ? sourceKey.slice(sourcePrefix.length)
        : sourceKey
      const targetKey = `${targetPrefix}${relativePath}`
      
      // 如果目标已存在，先备份到 rollback 位置
      const alreadyExists = await objectStore.objectExists(
        objectStore.ObjectStoreBuckets.APPS,
        targetKey
      )
      if (alreadyExists) {
        const rollbackKey = `${rollbackPrefix}${relativePath}`
        await copyAppFile(targetKey, rollbackKey)
        rollbackFiles.push({
          targetKey,
          rollbackKey,
        })
      }
      
      // 复制源文件到目标
      await copyAppFile(sourceKey, targetKey)
      uploadedTargetKeys.add(targetKey)
    }
  } catch (err) {
    // 部分提升失败时立即回滚
    if (uploadedTargetKeys.size) {
      try {
        await rollbackPromotedWorkspaceFiles(
          [...uploadedTargetKeys],
          rollbackFiles
        )
      } catch (rollbackErr) {
        console.log("Failed to rollback partially promoted restore files:", rollbackErr)
      }
    }
    throw err
  }
  
  return {
    sourceFileKeys,
    targetFileKeys: [...uploadedTargetKeys],
    rollbackFiles,
  }
}
```

**promote 阶段关键点：**

| 步骤 | 说明 |
|------|------|
| **列源文件** | 列出源 workspace（临时）的所有对象存储文件 |
| **备份现有文件** | 目标位置已存在的文件，先复制到 `__restore_rollback/{timestamp}/` 目录 |
| **复制文件** | 将源文件复制到目标位置 |
| **记录 rollback 信息** | 返回 rollback 所需的文件映射 |
| **部分失败回滚** | 提升过程中出错，立即回滚已提升的文件 |

### 5.3 rollbackPromotedWorkspaceFiles - 回滚提升的文件

[rollbackPromotedWorkspaceFiles](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/backups/processing.ts#L108-L128)：

```typescript
async function rollbackPromotedWorkspaceFiles(
  targetFileKeys: string[],
  rollbackFiles: PromoteWorkspaceFileRollback[]
) {
  const rollbackByTargetKey = new Map<string, string>()
  for (const rollbackFile of rollbackFiles) {
    rollbackByTargetKey.set(rollbackFile.targetKey, rollbackFile.rollbackKey)
  }
  const promotedNewFiles: string[] = []
  for (const targetKey of targetFileKeys) {
    const rollbackKey = rollbackByTargetKey.get(targetKey)
    if (rollbackKey) {
      // 有备份的文件，从备份恢复
      await copyAppFile(rollbackKey, targetKey)
    } else {
      // 新文件，直接删除
      promotedNewFiles.push(targetKey)
    }
  }
  if (promotedNewFiles.length) {
    await deleteAppFiles(promotedNewFiles)
  }
}
```

**回滚规则：**

| 场景 | 处理方式 |
|------|----------|
| 目标文件原有备份（rollback 文件） | 从 rollback 位置复制回目标 |
| 目标文件是新增的（无备份） | 直接删除 |

### 5.4 数据库切换 - Replication

```typescript
// 删除现有 dev 数据库
await removeExistingApp(devWorkspaceId)
// 从临时数据库复制到 dev 数据库
await new db.Replication({
  source: tempWorkspaceId,
  target: devWorkspaceId,
}).replicate()
```

**数据库切换策略：**
- 先销毁目标 dev 数据库
- 再通过 CouchDB Replication 将临时数据库数据复制到 dev 数据库
- 这样保证数据完全替换

### 5.5 trackBackupError - 跟踪错误

[trackBackupError](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/backups/backup.ts#L236-L269) 函数：

```typescript
async function trackBackupError(
  workspaceId: string,
  backupId: string,
  error: string
) {
  const prodWorkspaceId = db.getProdWorkspaceID(workspaceId)
  await context.doInWorkspaceContext(prodWorkspaceId, async () => {
    const database = context.getProdWorkspaceDB()

    const databaseExists = await database.exists()
    if (!databaseExists) {
      return
    }

    const metadata = await database.tryGet<Workspace>(
      DocumentType.WORKSPACE_METADATA
    )
    if (!metadata) {
      return
    }

    if (!metadata.backupErrors) {
      metadata.backupErrors = {}
    }

    if (!metadata.backupErrors[backupId]) {
      metadata.backupErrors[backupId] = []
    }

    metadata.backupErrors[backupId].push(error)
    await database.put(metadata)
    await cache.workspace.invalidateWorkspaceMetadata(metadata.appId, metadata)
  })
}
```

**错误跟踪方式：**
- 将错误存储在 workspace metadata 的 `backupErrors` 字段中
- 按 `backupId` 分组，每个备份可以有多个错误
- 更新后使缓存失效

---

## 6. 完整生命周期流程图

```
手动导出
  │
  ▼
┌───────────────────────────────────────────────────────┐
│  POST /api/backups/export                              │
│  - ensureTenantAppOwnershipMiddleware (验证租户权限)   │
│  - ctx.req.setTimeout(0) (无限超时)                   │
│  - 决定扩展名: .tar.gz 或 .enc.tar.gz                 │
│  - 调用 streamExportWorkspace                         │
└───────────────────────┬───────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────┐
│  exportWorkspace (exports.ts)                          │
│  1. objectStore.retrieveDirectory                      │
│     - 排除隐藏文件                                     │
│     - excludeRows 时排除 attachments                   │
│  2. exportDB (CouchDB dump)                            │
│     - 过滤: 用户元数据/关联用户元数据/自动化日志       │
│     - excludeRows 时额外过滤表行                      │
│  3. 可选加密 (非附件文件)                              │
│  4. tar + gzip 打包                                   │
└───────────────────────┬───────────────────────────────┘
                        │
                        ▼
                    下载到用户
                        │
                        ▼
                    用户上传恢复
                        │
                        ▼
┌───────────────────────────────────────────────────────┐
│  触发 restore (backup.ts)                              │
│  - 创建 PENDING 元数据                                 │
│  - 添加到 Bull 队列                                    │
└───────────────────────┬───────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────┐
│  importProcessor (processing.ts)                       │
│  1. 更新状态 STARTED                                   │
│  2. 先导出当前状态作为回滚备份 (runBackup)              │
│  3. 下载备份文件                                       │
│  4. 创建 temp workspace ID                             │
└───────────────────────┬───────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────┐
│  importApp (imports.ts) - 导入到 temp DB               │
│  1. untar 解包                                         │
│  2. decryptFiles 解密 (非附件)                         │
│  3. 检查 plugin (阻止 plugin.min.js)                   │
│  4. 上传对象存储内容到 temp workspace                   │
│  5. 删除 stale 非附件文件                              │
│  6. db.load 加载数据库                                 │
│  7. updateAttachmentColumns 重写附件 key               │
│  8. updateAutomations 更新自动化配置                   │
│  9. sanitizeLiteLLMImportData 清理 LiteLLM 配置        │
└───────────────────────┬───────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────┐
│  promoteWorkspaceFiles                                 │
│  - 遍历 temp workspace 文件                            │
│  - 目标已存在 → 备份到 __restore_rollback/{ts}/        │
│  - 复制 temp → dev 对象存储                            │
│  - 失败时立即部分回滚                                  │
└───────────────────────┬───────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────┐
│  数据库切换 (Replication)                              │
│  - removeExistingApp (销毁 dev DB)                     │
│  - Replication(temp → dev)                            │
└───────────────────────┬───────────────────────────────┘
                        │
          ┌─────────────┴─────────────┐
          │ 成功                       │ 失败
          ▼                            ▼
┌─────────────────────┐    ┌──────────────────────────────┐
│ cleanupPromoted     │    │ rollbackPromotedWorkspaceFiles│
│   - 删除 stale 文件 │    │   - 有备份的: 从 rollback 恢复│
│   - 删除源 temp 文件│    │   - 新文件: 直接删除          │
└─────────────────────┘    └──────────────┬───────────────┘
                                          │
                                          ▼
                                ┌──────────────────────┐
                                │ trackBackupError     │
                                │   - 存到 workspace   │
                                │     metadata.backupErrors
                                │   - 状态: FAILED     │
                                └──────────────────────┘
                                          │
                                          ▼
                                ┌──────────────────────┐
                                │ finally 清理          │
                                │   - 销毁 temp DB      │
                                │   - 清理 temp 文件    │
                                └──────────────────────┘
```

---

## 总结

### 导出阶段关键点

1. **安全验证**：`ensureTenantAppOwnershipMiddleware` 确保跨租户无法访问
2. **超时处理**：设置 `setTimeout(0)` 支持大文件长时间导出
3. **数据过滤**：导出时排除用户元数据、自动化日志等敏感数据
4. **选择性加密**：只加密非附件文件（附件太大加密成本高）
5. **流式响应**：直接返回文件流，减少内存占用

### 导入阶段关键点

1. **安全检查**：阻止 plugin 文件导入，防止混淆攻击
2. **附件保护**：stale 文件清理时永远不删除附件
3. **数据清洗**：LiteLLM 密钥被清除，模型 ID 标记为待配置
4. **引用更新**：附件 key 和自动化 URL 中的 app ID 都要重写

### 恢复队列关键点

1. **两阶段提交**：先导入临时数据库，成功后再切换
2. **文件级回滚**：promote 前先备份目标文件，失败可回滚
3. **数据库级回滚**：恢复前先备份当前状态（runBackup RESTORING）
4. **错误跟踪**：错误记录在 workspace metadata 中，用户可见
5. **资源清理**：finally 确保临时数据库和文件被清理
