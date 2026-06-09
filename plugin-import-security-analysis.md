# Budibase Plugin 导入路径安全边界分析

## 一、路由入口限制：globalBuilderRoutes

### 1.1 路由定义

所有 plugin 相关的 API 路由均通过 `globalBuilderRoutes` 定义，位于 [plugin.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/plugin.ts)：

```
POST /api/plugin/upload  — controller.upload（文件上传）
POST /api/plugin         — controller.create（URL/NPM/GitHub 导入）
GET  /api/plugin         — controller.fetch
DELETE /api/plugin/:pluginId — controller.destroy
...
```

### 1.2 权限中间件

`globalBuilderRoutes` 在 [standard.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/endpointGroups/standard.ts#L6-L9) 中定义：

```typescript
export const globalBuilderRoutes = endpointGroupList.group(
  authorized(permissions.GLOBAL_BUILDER)
)
globalBuilderRoutes.lockMiddleware()
```

**安全要点：**
- 所有 `/api/plugin/*` 入口均要求 `GLOBAL_BUILDER` 权限
- `lockMiddleware()` 锁定后无法再添加新中间件，防止后续代码绕过权限校验
- 这是 plugin 系统的第一道安全边界——未授权用户无法访问任何 plugin API

---

## 二、控制器层：upload 与 create 的分流

### 2.1 controller.upload — 文件数组处理

位于 [plugin/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/plugin/index.ts#L29-L57)：

```typescript
export async function upload(ctx: UserCtx<UploadPluginRequest, UploadPluginResponse>) {
  const files = ctx.request.files
  const plugins =
    files && Array.isArray(files.file) && files.file.length > 1
      ? Array.from(files.file)
      : [files?.file]

  for (let plugin of plugins) {
    if (!plugin || Array.isArray(plugin)) continue
    const doc = await sdk.plugins.processUploaded(plugin, PluginSource.FILE)
    docs.push(doc)
  }
}
```

**处理逻辑：**
- 从 `ctx.request.files.file` 获取上传的文件
- 支持单文件或多文件（数组）上传
- 每个文件独立调用 `sdk.plugins.processUploaded`，source 为 `PluginSource.FILE`
- 单个文件失败不影响其他文件（循环内无 try/catch，实际整体在 try/catch 中）

### 2.2 controller.create — Source Switch 分流

位于 [plugin/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/plugin/index.ts#L59-L152)：

```typescript
export async function create(ctx: UserCtx<CreatePluginRequest, CreatePluginResponse>) {
  const { source, url, headers, githubToken } = ctx.request.body

  switch (source) {
    case PluginSource.NPM:
      const result = await npmUpload(url, name)
      break
    case PluginSource.GITHUB:
      const result = await githubUpload(url, name, githubToken)
      break
    case PluginSource.URL:
      const result = await urlUpload(url, name, headersObj)
      break
    default:
      ctx.throw(400, "Invalid source")
  }

  pluginCore.validate(metadata.schema)

  // 云环境只允许 component 插件
  if (!env.SELF_HOSTED && metadata.schema?.type !== PluginType.COMPONENT) {
    throw new Error("Only component plugins are supported outside of self-host")
  }

  // component 插件必须是 Svelte 5
  if (
    metadata.schema?.metadata?.svelteMajor !== 5 &&
    metadata.schema?.type === PluginType.COMPONENT
  ) {
    throw new Error("Only Svelte 5 plugins are supported on this branch")
  }

  const doc = await pro.plugins.storePlugin(metadata, directory!, source, origin)
  clientAppSocket?.emit("plugins-update", { name, hash: doc.hash })
}
```

**处理流程：**
1. 根据 `source` 字段分流到对应的 uploader
2. 上传器下载并解压 tarball，返回 metadata 和目录路径
3. 调用 `pluginCore.validate` 校验 schema
4. 云环境限制：只允许 `COMPONENT` 类型
5. Component 插件限制：必须 `svelteMajor === 5`
6. 调用 `pro.plugins.storePlugin` 持久化存储
7. 通过 socket 广播更新通知

---

## 三、四种导入路径的安全边界

### 3.1 文件上传：fileUpload

位于 [file.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/plugin/file.ts)

```typescript
export async function fileUpload(file: KoaFile) {
  const filename = file.originalFilename
  const filePath = file.filepath

  if (!filename || !filePath) {
    throw new Error("File is not valid - cannot upload.")
  }
  if (!filename.endsWith(".tar.gz")) {
    throw new Error("Plugin must be compressed into a gzipped tarball.")
  }
  // ... 解压并获取 metadata
}
```

**安全限制：**
- **仅接受 `.tar.gz` 后缀**：通过 `filename.endsWith(".tar.gz")` 校验文件名
- 原因：
  1. 统一插件打包格式，确保解压逻辑可预期
  2. 防止上传可执行文件（如 `.exe`、`.sh`）或恶意脚本
  3. 与后续 `extractTarball` 解压逻辑匹配
  4. 防止路径遍历——直接用文件名生成临时目录名

### 3.2 URL 导入：urlUpload

位于 [url.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/plugin/url.ts)

```typescript
function parseTarGzUrl(url: string): URL {
  let parsed: URL
  try {
    parsed = new URL(url)
  } catch {
    throw new Error("Invalid plugin URL.")
  }

  if (parsed.protocol !== "https:") {
    throw new Error("Plugin URL must use HTTPS.")
  }

  if (!parsed.pathname.endsWith(".tar.gz")) {
    throw new Error("Plugin must be compressed into a gzipped tarball.")
  }

  return parsed
}

export async function urlUpload(url: string, name = "", headers = {}) {
  parseTarGzUrl(url)

  const path = await downloadUnzipTarball(url, name, headers, {
    followRedirects: false,
  })
  // ...
}
```

**安全限制：**

| 限制项 | 实现方式 | 安全目的 |
|--------|----------|----------|
| **HTTPS 强制** | `parsed.protocol !== "https:"` | 防止中间人攻击，确保传输加密 |
| **.tar.gz 后缀** | `parsed.pathname.endsWith(".tar.gz")` | 确保下载的是预期格式，防止 SSRF 到任意端点 |
| **关闭重定向** | `followRedirects: false` | 防止开放重定向漏洞导致 SSRF 到内网 |

> **注意**：`.tar.gz` 检查的是 `pathname` 而非整个 URL，这可以防止通过 query string 或 fragment 绕过（如 `https://evil.com/ssrf?.tar.gz`）。安全测试 [security.spec.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/plugin/tests/security.spec.ts#L50-L68) 验证了这一点。

### 3.3 NPM 导入：npmUpload

位于 [npm.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/plugin/npm.ts)

```typescript
function isAllowedNpmHost(host: string): boolean {
  return host === "www.npmjs.com" || host === "registry.npmjs.org"
}

export async function npmUpload(url: string, name: string, headers = {}) {
  const parsedInput = parseNpmUrl(npmTarballUrl)
  if (!isAllowedNpmHost(parsedInput.hostname)) {
    throw new Error("The plugin origin must be from NPM")
  }

  if (!npmTarballUrl.includes(".tgz")) {
    // 包页 URL 解析逻辑
    if (
      parsedInput.hostname !== "www.npmjs.com" ||
      !parsedInput.pathname.startsWith("/package/")
    ) {
      throw new Error("The plugin origin must be from NPM")
    }
    const packageName = parsedInput.pathname.replace("/package/", "").trim()
    const npmPackageURl = `https://registry.npmjs.org/${packageName}`
    const response = await coreUtils.fetchWithBlacklist(npmPackageURl)
    let npmDetails = await response.json()
    const npmVersion = npmDetails["dist-tags"].latest
    npmTarballUrl = npmDetails?.versions?.[npmVersion]?.dist?.tarball
  }

  const path = await downloadUnzipTarball(npmTarballUrl, pluginName, headers)
  // ...
}
```

**安全限制：**

| 限制项 | 实现方式 | 安全目的 |
|--------|----------|----------|
| **白名单域名** | `host === "www.npmjs.com" \|\| host === "registry.npmjs.org"` | 只允许从官方 NPM 源下载，防止恶意 registry |
| **HTTPS 强制** | `parsed.protocol !== "https:"` 校验 | 传输加密 |
| **包页 → registry 解析** | 从 `dist-tags.latest` 获取版本，再从 `versions[x].dist.tarball` 获取 tarball URL | 用户输入友好的包名 URL，最终统一到官方 registry |
| **fetchWithBlacklist** | 调用 `coreUtils.fetchWithBlacklist` | 防止 SSRF 到内网 |

> **注意**：NPM tarball 使用 `.tgz` 后缀而非 `.tar.gz`，这是 NPM 的标准格式。

### 3.4 GitHub 导入：githubUpload

位于 [github.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/plugin/github.ts)

```typescript
function parseGithubUrl(url: string): URL {
  let parsed: URL
  try {
    parsed = new URL(url)
  } catch {
    throw new Error("Invalid Github URL")
  }

  if (parsed.protocol !== "https:" || parsed.hostname !== "github.com") {
    throw new Error("The plugin origin must be from Github")
  }
  return parsed
}

export async function githubUpload(url: string, name = "", token = "") {
  // 转换为 GitHub API URL
  const githubApiUrl = githubUrl.replace(
    "https://github.com/",
    "https://api.github.com/repos/"
  )

  // 获取仓库信息
  const pluginDetails = await request(githubApiUrl, headers, "Repository not found")

  // 获取 latest release
  const pluginLatestReleaseUrl = pluginDetails?.["releases_url"]
    ? pluginDetails?.["releases_url"].replace("{/id}", "/latest")
    : undefined

  const pluginReleaseDetails = await request(
    pluginLatestReleaseUrl,
    headers,
    "Github latest release not found"
  )

  // 查找 content_type 为 application/gzip 的资产
  const pluginReleaseTarballAsset = pluginReleaseDetails?.assets?.find(
    (x: any) => x?.["content_type"] === "application/gzip"
  )
  const pluginLastReleaseTarballUrl =
    pluginReleaseTarballAsset?.["browser_download_url"]

  // 下载 tarball
  path = await downloadUnzipTarball(pluginLastReleaseTarballUrl, pluginName, headers)
}
```

**安全限制：**

| 限制项 | 实现方式 | 安全目的 |
|--------|----------|----------|
| **仅 github.com** | `parsed.hostname !== "github.com"` | 只允许从 GitHub 官方导入，防止自建 Git 服务 |
| **HTTPS 强制** | `parsed.protocol !== "https:"` | 传输加密 |
| **Latest Release** | 通过 `releases_url/latest` 获取最新发布 | 确保下载的是正式发布版本 |
| **content_type 校验** | `x?.["content_type"] === "application/gzip"` | 只下载 gzip 格式的资产，防止下载其他类型文件 |
| **fetchWithBlacklist** | `request()` 内部调用 `coreUtils.fetchWithBlacklist` | 防止 SSRF |

> **注意**：GitHub release 资产 URL 通常指向 `github-releases.githubusercontent.com` 等域名，这些域名的 IP 不在黑名单中，属于正常的外网访问。

---

## 四、下载与解压：downloadUnzipTarball 与 fetchWithBlacklist

### 4.1 downloadUnzipTarball

位于 [utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/plugin/utils.ts)

```typescript
export async function downloadUnzipTarball(
  url: string,
  name: string,
  headers = {},
  { followRedirects = true }: { followRedirects?: boolean } = {}
) {
  const path = createTempFolder(name)

  try {
    await objectStore.downloadTarballDirect(url, path, headers, {
      followRedirects,
    })
    return path
  } catch (e: any) {
    deleteFolderFileSystem(path)
    throw e
  }
}
```

**职责：**
- 创建临时目录
- 调用 `objectStore.downloadTarballDirect` 下载并解压
- 失败时清理临时目录

### 4.2 objectStore.downloadTarballDirect

位于 [objectStore.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/objectStore/objectStore.ts#L706-L723)

```typescript
export async function downloadTarballDirect(
  url: string,
  path: string,
  headers = {},
  { followRedirects = true }: { followRedirects?: boolean } = {}
) {
  path = sanitizeKey(path)
  const response = await fetchWithBlacklist(
    url,
    { headers },
    { followRedirects }
  )
  if (!response.ok) {
    throw new Error(`unexpected response ${response.statusText}`)
  }

  await pipeline(response.body, zlib.createUnzip(), tar.extract(path))
}
```

**处理流程：**
1. 调用 `fetchWithBlacklist` 发起 HTTP 请求（带 SSRF 防护）
2. 检查响应状态
3. 流式处理：`response.body → zlib.createUnzip() → tar.extract(path)`

### 4.3 fetchWithBlacklist — SSRF 防护核心

位于 [outboundFetch.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/utils/outboundFetch.ts)

```typescript
export async function fetchWithBlacklist(
  url: string,
  request: TRequest = {} as TRequest,
  { followRedirects = true, ... } = {}
): Promise<TResponse> {
  let nextUrl = url
  let nextRequest: RedirectSafeRequest<TRequest> = {
    ...request,
    redirect: "manual",
  }

  for (let redirects = 0; redirects <= MAX_REDIRECTS; redirects++) {
    const pinnedIp = await resolveSafePinnedIp(nextUrl)
    let response = await fetchFn(nextUrl, {
      ...nextRequest,
      agent: makePinnedAgent(nextUrl, pinnedIp),
    })

    if (!isRedirect(response.status)) {
      return response
    }

    if (!followRedirects) {
      throw new Error("Redirects are not permitted.")
    }

    // 重定向处理：剥离敏感 headers、限制次数等
    // ...
  }

  throw new Error("Maximum redirect reached.")
}
```

**核心安全机制：**

#### 4.3.1 IP 黑名单

位于 [blacklist.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/blacklist/blacklist.ts#L6-L16)

```typescript
const DEFAULT_BLACKLIST = [
  "127.0.0.0/8",      // 回环地址
  "10.0.0.0/8",       // A 类内网
  "172.16.0.0/12",    // B 类内网
  "192.168.0.0/16",   // C 类内网
  "169.254.0.0/16",   // 链路本地
  "0.0.0.0/8",        // 本机
  "::1/128",          // IPv6 回环
  "fc00::/7",         // IPv6 唯一本地
  "fe80::/10",        // IPv6 链路本地
] as const
```

#### 4.3.2 DNS 解析与 IP 绑定

```typescript
async function resolveSafePinnedIp(url: string): Promise<string> {
  const parsed = parseUrl(url)
  const addresses = await resolveAddress(parsed.hostname)
  if (addresses.length === 0) {
    throw new Error("URL is blocked or could not be resolved safely.")
  }

  for (const address of addresses) {
    if (await isBlacklisted(address)) {
      throw new Error("URL is blocked or could not be resolved safely.")
    }
  }

  return addresses[0]
}

function makePinnedAgent(url: string, ip: string): http.Agent | https.Agent {
  // 通过自定义 lookup 函数绑定 IP，防止 DNS rebinding
  const lookup: LookupFunction = (_hostname, _options, callback) => {
    // 直接返回解析好的 IP，不再做 DNS 查询
    callback(null, ip, family)
  }
  return protocol === "https:"
    ? new https.Agent({ lookup })
    : new http.Agent({ lookup })
}
```

**关键设计：**
- **先发 DNS 解析，再校验黑名单**：确保所有解析出的 IP 都不在黑名单中
- **IP 绑定（Pinned IP）**：通过自定义 `lookup` 函数，将 HTTP 请求直接绑定到已校验的 IP，防止 DNS rebinding 攻击（DNS 在连接时返回不同 IP）
- **手动处理重定向**：不依赖 node-fetch 的自动重定向，每次重定向都重新做 DNS 解析和黑名单校验

#### 4.3.3 URL 基础校验

```typescript
function parseUrl(url: string): URL {
  let parsed: URL
  try {
    parsed = new URL(url)
  } catch {
    throw new Error("Invalid URL.")
  }

  if (!ALLOWED_PROTOCOLS.has(parsed.protocol)) {
    throw new Error("Only HTTP(S) URLs are allowed.")
  }

  if (parsed.username || parsed.password) {
    throw new Error("URL must not include credentials.")
  }

  return parsed
}
```

- 仅允许 `http:` 和 `https:` 协议
- URL 中不能包含用户名/密码

#### 4.3.4 重定向安全处理

```typescript
function shouldStripSensitiveHeadersForRedirect(
  currentUrl: string,
  redirectUrl: string
): boolean {
  return new URL(currentUrl).origin !== new URL(redirectUrl).origin
}

// 跨域重定向时剥离的敏感 headers
const SENSITIVE_REDIRECT_HEADERS = [
  "authorization",
  "cookie",
  "cookie2",
  "proxy-authorization",
]
```

- 最大重定向次数：5 次
- 跨域重定向时剥离敏感 headers
- 可配置 `followRedirects: false` 完全禁止重定向（urlUpload 使用此配置）

---

## 五、Schema 校验：pluginCore.validate

位于 [plugin/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/plugin/utils.ts)

```typescript
export function validate(schema: any) {
  switch (schema?.type) {
    case PluginType.COMPONENT:
      validateComponent(schema)
      break
    case PluginType.DATASOURCE:
      validateDatasource(schema)
      break
    case PluginType.AUTOMATION:
      validateAutomation(schema)
      break
    default:
      throw new Error(`Unknown plugin type - check schema.json: ${schema.type}`)
  }
}
```

### 5.1 Component 插件校验

```typescript
function validateComponent(schema: any) {
  const validator = joi.object({
    type: joi.string().allow(PluginType.COMPONENT).required(),
    metadata: joi.object().unknown(true).required(),
    hash: joi.string().optional(),
    version: joi.string().optional(),
    schema: joi
      .object({
        name: joi.string().required(),
        settings: joi.array().items(joi.object().unknown(true)).required(),
      })
      .unknown(true),
  })
  runJoi(validator, schema)
}
```

**必填字段：**
- `type`: 必须为 `component`
- `metadata`: 元数据对象（允许未知字段）
- `schema.name`: 组件名称
- `schema.settings`: 组件设置数组

### 5.2 Datasource 插件校验

```typescript
function validateDatasource(schema: any) {
  const validator = joi.object({
    type: joi.string().allow(PluginType.DATASOURCE).required(),
    metadata: joi.object().unknown(true).required(),
    hash: joi.string().optional(),
    version: joi.string().optional(),
    schema: joi.object({
      docs: joi.string(),
      plus: joi.boolean().optional(),
      isSQL: joi.boolean().optional(),
      auth: joi.object({ type: joi.string().required() }).optional(),
      features: joi.object(/* DatasourceFeature */).optional(),
      relationships: joi.boolean().optional(),
      description: joi.string().required(),
      friendlyName: joi.string().required(),
      type: joi.string().allow(...DATASOURCE_TYPES),
      datasource: joi.object().pattern(joi.string(), fieldValidator).required(),
      query: joi.object().pattern(joi.string(), queryValidator).unknown(true).required(),
      extra: joi.object().pattern(joi.string(), {
        type: joi.string().required(),
        displayName: joi.string().required(),
        required: joi.boolean(),
        data: joi.object(),
      }),
    }),
  })
  runJoi(validator, schema)
}
```

**必填字段：**
- `type`: 必须为 `datasource`
- `metadata`: 元数据对象
- `schema.description`: 数据源描述
- `schema.friendlyName`: 友好名称
- `schema.datasource`: 数据源字段定义
- `schema.query`: 查询类型定义

### 5.3 Automation 插件校验

```typescript
function validateAutomation(schema: any) {
  const validator = joi.object({
    type: joi.string().allow(PluginType.AUTOMATION).required(),
    metadata: joi.object().unknown(true).required(),
    hash: joi.string().optional(),
    version: joi.string().optional(),
    schema: joi.object({
      name: joi.string().required(),
      tagline: joi.string().required(),
      icon: joi.string().required(),
      description: joi.string().required(),
      type: joi.string().allow(AutomationStepType.ACTION, AutomationStepType.LOGIC).required(),
      stepId: joi.string().disallow(...AutomationStepIdArray).required(),
      inputs: joi.object().optional(),
      schema: joi.object({
        inputs: stepSchemaValidator,
        outputs: stepSchemaValidator,
      }).required(),
    }),
  })
  runJoi(validator, schema)
}
```

**必填字段：**
- `type`: 必须为 `automation`
- `metadata`: 元数据对象
- `schema.name`: 步骤名称
- `schema.tagline`: 标语
- `schema.icon`: 图标
- `schema.description`: 描述
- `schema.type`: 步骤类型（ACTION / LOGIC）
- `schema.stepId`: 步骤 ID（不能与内置步骤 ID 冲突）
- `schema.schema.inputs` / `outputs`: 输入输出 schema

### 5.4 部署环境限制

在 [plugin/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/plugin/index.ts#L114-L125) 和 [sdk/plugins/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/plugins/index.ts#L133-L145) 中还有两层额外限制：

```typescript
// 云环境只允许 component 插件
if (!env.SELF_HOSTED && metadata.schema?.type !== PluginType.COMPONENT) {
  throw new Error("Only component plugins are supported outside of self-host")
}

// component 插件必须是 Svelte 5
if (
  metadata.schema?.metadata?.svelteMajor !== 5 &&
  metadata.schema?.type === PluginType.COMPONENT
) {
  throw new Error("Only Svelte 5 plugins are supported on this branch")
}
```

| 限制项 | 条件 | 目的 |
|--------|------|------|
| **云环境仅 Component** | `!env.SELF_HOSTED && type !== COMPONENT` | 防止云环境中执行数据源/自动化插件的恶意代码 |
| **Component 必须 Svelte 5** | `svelteMajor !== 5 && type === COMPONENT` | 技术栈统一，确保前端兼容 |

> **注意**：`processUploaded`（文件上传路径）和 `create`（URL/NPM/GitHub 路径）都有这两层校验，实现了双重保障。

---

## 六、存储与广播：pro.plugins.storePlugin 与 Socket

### 6.1 pro.plugins.storePlugin

位于 [pro/src/sdk/plugins/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/sdk/plugins/index.ts)

```typescript
export async function storePlugin(
  metadata: PluginMetadata,
  directory: any,
  source: PluginSource,
  origin?: PluginOrigin
): Promise<Plugin> {
  const db = tenancy.getGlobalDB()

  // 1. 上传到对象存储
  const bucketPath = objectStore.getPluginS3Dir(name)
  const files = await objectStore.uploadDirectory(
    objectStore.ObjectStoreBuckets.PLUGINS,
    directory,
    bucketPath
  )

  // 2. 检查 JS 文件
  const jsFile = files.find((file: any) => file.name.endsWith(".js"))
  if (!jsFile) {
    throw new Error(`Plugin missing .js file.`)
  }

  // 3. Datasource 插件：JS 语法校验
  if (metadata.schema.type === PluginType.DATASOURCE) {
    const js = loadJSFile(directory, jsFile.name)
    const isolate = new ivm.Isolate({ memoryLimit: 8 })
    try {
      isolate.compileScriptSync(Module.wrap(js), { filename: jsFile.name })
    } catch (err: any) {
      throw new Error(`JS invalid: ${message}`)
    } finally {
      isolate.dispose()
    }
  }

  // 4. 保存到全局数据库
  let doc: Plugin = {
    _id: pluginId,
    _rev: rev,
    ...metadata,
    name,
    version,
    hash,
    description,
    source,
  }

  const write = async (): Promise<Plugin> => {
    const response = await db.put(doc)
    await events.plugin.imported(doc)
    return { ...doc, _rev: response.rev }
  }

  // 5. 配额检查
  if (!rev) {
    return await quotas.addPlugin(write)
  } else {
    return await write()
  }
}
```

**关键步骤：**

| 步骤 | 说明 | 安全意义 |
|------|------|----------|
| **上传对象存储** | 将解压后的插件文件上传到 S3/MinIO | 持久化存储，统一分发 |
| **JS 文件检查** | 必须存在 `.js` 文件 | 确保插件完整性 |
| **Datasource JS 校验** | 使用 `isolated-vm` 编译 JS 代码 | 提前发现语法错误，datasource 插件会在服务端执行 |
| **全局数据库存储** | 保存到 tenant 的全局 DB | 元数据持久化 |
| **配额检查** | 新增插件时调用 `quotas.addPlugin` | 防止无限制创建 |

### 6.2 Socket 广播

插件更新后通过 WebSocket 广播通知客户端：

**文件上传路径**（`processUploaded`）—— [sdk/plugins/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/plugins/index.ts#L148)：
```typescript
clientAppSocket?.emit("plugin-update", { name: doc.name, hash: doc.hash })
```

**URL/NPM/GitHub 路径**（`controller.create`）—— [plugin/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/plugin/index.ts#L142)：
```typescript
clientAppSocket?.emit("plugins-update", { name, hash: doc.hash })
```

> **注意**：两个路径的事件名不同：文件上传用 `plugin-update`，create 用 `plugins-update`。这可能是历史遗留问题，但功能类似——通知客户端插件已更新。

`clientAppSocket` 在 [websockets/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/websockets/index.ts) 中定义，是 `ClientAppSocket` 类的单例实例，用于与客户端应用通信。

---

## 七、安全边界总结

### 7.1 四层防御体系

| 层级 | 位置 | 作用 |
|------|------|------|
| **L1 路由权限** | `globalBuilderRoutes` | 只有 GLOBAL_BUILDER 可访问 plugin API |
| **L2 来源校验** | upload 各模块（file/url/npm/github） | 限制域名、协议、文件格式 |
| **L3 SSRF 防护** | `fetchWithBlacklist` + IP 黑名单 | 防止内网资源被访问 |
| **L4 Schema 校验** | `pluginCore.validate` | 确保插件元数据格式正确 |
| **L5 类型限制** | 云环境仅 component + Svelte 5 | 减少攻击面 |
| **L6 运行时防护** | `isolated-vm` 编译校验 | datasource 插件 JS 语法检查 |

### 7.2 四种导入方式对比

| 维度 | File | URL | NPM | GitHub |
|------|------|-----|-----|--------|
| **入口** | `/api/plugin/upload` | `/api/plugin` | `/api/plugin` | `/api/plugin` |
| **来源校验** | 文件名后缀 | 域名无限制（HTTPS+.tar.gz） | 白名单域名（2 个） | 白名单域名（1 个） |
| **格式校验** | `.tar.gz` | `.tar.gz` | `.tgz`（NPM 标准） | `application/gzip` content-type |
| **SSRF 防护** | 不涉及网络请求 | fetchWithBlacklist + 禁重定向 | fetchWithBlacklist | fetchWithBlacklist |
| **版本解析** | 无 | 无（指定 URL） | latest tag | latest release |
| **Token 支持** | 无 | 自定义 headers | 无 | GitHub token |

### 7.3 关键安全设计

1. **白名单优先**：NPM 和 GitHub 方式使用域名白名单，而非黑名单
2. **IP 绑定防 DNS rebinding**：先解析 DNS 并校验，再用固定 IP 建立连接
3. **重定向可控**：URL 方式完全禁重定向，NPM/GitHub 方式限制重定向次数
4. **云环境收敛**：云环境只允许 component 插件，大幅减少攻击面
5. **沙箱校验**：datasource 插件使用 isolated-vm 做 JS 语法检查
