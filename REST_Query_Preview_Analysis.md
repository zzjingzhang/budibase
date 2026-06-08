# REST Query Preview 完整分析

## 目录

1. [整体调用链概述](#整体调用链概述)
2. [query.preview 构造 QueryEvent](#querypreview-构造-queryevent)
3. [Runner.run 与 QueryRunner.execute](#runnerrun-与-queryrunnerexecute)
4. [RestIntegration 的选择](#restintegration-的选择)
5. [RestIntegration._req 的 headers 合并逻辑](#restintegration_req-的-headers-合并逻辑)
6. [fetchWithBlacklist 安全机制详解](#fetchwithblacklist-安全机制详解)
7. [Blacklist 策略与 SELF_HOSTED/BLACKLIST_IPS 配置](#blacklist-策略与-self_hostedblacklist_ips-配置)
8. [OAuth2 401 重试机制](#oauth2-401-重试机制)
9. [为什么 127.0.0.1:5984 默认被阻止](#为什么-1270015984-默认被阻止)

---

## 整体调用链概述

REST Query Preview 的完整调用链路如下：

```
API Controller (preview)
  → 构造 QueryEvent
  → Runner.run (ThreadType.QUERY)
    → QueryRunner.execute
      → getIntegration(datasource.source) → RestIntegration
      → 富集参数与 headers
      → integration[queryVerb](query)  (如 read → _req)
        → RestIntegration._req
          → 合并 defaultHeaders + headers + authHeaders
          → disabledHeaders 删除指定 header
          → coreUtils.fetchWithBlacklist
            → parseUrl 校验协议/URL 合法性
            → resolveSafePinnedIp 解析 DNS + 检查黑名单
            → makePinnedAgent 固定 IP 防 DNS 重绑定
            → 处理重定向（跨 origin 剥离敏感 header）
          → 401 时 cleanStoredTokensForAuthConfig → 重试一次
```

---

## query.preview 构造 QueryEvent

### 入口位置

[packages/server/src/api/controllers/query/index.ts#L243-L413](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/query/index.ts#L243-L413)

### 核心逻辑

`preview` 函数是 REST Query 预览的 API 入口，主要完成以下工作：

1. **获取数据源与环境变量**：通过 `sdk.datasources.getWithEnvVars` 获取 datasource 及其环境变量
2. **构造 Query 对象**：从请求体中提取未保存的 query 配置
3. **构造 QueryEvent**：将 query、datasource、环境变量、用户上下文等打包成 QueryEvent
4. **调用 Runner.run**：将 QueryEvent 发送到查询线程执行

### QueryEvent 结构

```typescript
const inputs: QueryEvent = {
  appId: ctx.appId,
  queryVerb: query.queryVerb,      // create/read/update/patch/delete
  fields: query.fields,            // REST query 配置（path、headers、body 等）
  parameters: enrichParameters(query),
  transformer: query.transformer,
  schema: query.schema,
  nullDefaultSupport: query.nullDefaultSupport,
  queryId,
  datasource,                      // 完整的数据源配置
  environmentVariables: envVars,
  ctx: {
    user: sanitiseUserStructure(ctx.user),
    auth: { ...authConfigCtx },
  },
}
```

关键参数说明：
- `datasource`：包含 `source`（数据源类型，如 `REST`）、`config`（包含 `url`、`defaultHeaders`、`authConfigs` 等）
- `fields`：REST Query 的具体配置，包含 `path`、`headers`、`requestBody`、`authConfigId` 等
- `ctx.auth`：包含 OAuth 配置 ID 和 sessionId，用于 OAuth2 认证

---

## Runner.run 与 QueryRunner.execute

### Runner 初始化

[packages/server/src/api/controllers/query/index.ts#L47-L49](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/query/index.ts#L47-L49)

```typescript
const Runner = new Thread(ThreadType.QUERY, {
  timeoutMs: env.QUERY_THREAD_TIMEOUT,
})
```

### QueryRunner.execute 主流程

[packages/server/src/threads/query.ts#L77-L279](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/threads/query.ts#L77-L279)

核心执行步骤：

1. **获取 Integration 类**：
   ```typescript
   const Integration = await getIntegration(datasourceClone.source)
   const integration = new Integration(datasourceClone.config)
   ```

2. **富集参数与上下文**：
   - 处理数据源变量（静态变量、动态变量）
   - 富集 `defaultHeaders` 中的模板变量
   - 处理 base URL 模板解析

3. **选择执行方法**：
   ```typescript
   const fn = integration[queryVerb]
   let output = await fn.bind(integration)(query)
   ```

4. **错误重试逻辑**：
   - 401 且是 OAuth2 用户时，调用 `refreshOAuth2` 刷新 token
   - 其他错误时，失效缓存变量后重试一次（`hasRerun` 标记）

> **注意**：`makeExternalQuery` 函数位于 [packages/server/src/integrations/base/query.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/integrations/base/query.ts)，主要用于 SQL 类型的 DatasourcePlus 查询，而非 REST Query 的执行路径。REST Query 走的是 `QueryRunner` → `integration[queryVerb]` 路径。

---

## RestIntegration 的选择

### getIntegration 映射

[packages/server/src/integrations/index.ts#L60-L81](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/integrations/index.ts#L60-L81)

系统通过 `datasource.source` 字段来选择对应的 Integration 类：

```typescript
const INTEGRATIONS: Record<SourceName, IntegrationBaseConstructor | undefined> = {
  [SourceName.REST]: rest.integration,   // RestIntegration
  [SourceName.POSTGRES]: postgres.integration,
  // ... 其他数据源
}
```

当 `datasource.source === SourceName.REST` 时，选择 `RestIntegration`。

### RestIntegration 类

[packages/server/src/integrations/rest.ts#L241-L858](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/integrations/rest.ts#L241-L858)

`RestIntegration` 提供了五种 HTTP 方法对应的查询动词：

| queryVerb | HTTP 方法 | 内部调用 |
|-----------|-----------|----------|
| create | POST | `_req({ ...opts, method: HttpMethod.POST })` |
| read | GET | `_req({ ...opts, method: HttpMethod.GET })` |
| update | PUT | `_req({ ...opts, method: HttpMethod.PUT })` |
| patch | PATCH | `_req({ ...opts, method: HttpMethod.PATCH })` |
| delete | DELETE | `_req({ ...opts, method: HttpMethod.DELETE })` |

所有方法最终都调用内部的 `_req` 方法。

---

## RestIntegration._req 的 headers 合并逻辑

### _req 方法核心

[packages/server/src/integrations/rest.ts#L689-L837](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/integrations/rest.ts#L689-L837)

### headers 合并顺序

`_req` 方法中的 headers 合并遵循**后覆盖前**的原则：

```typescript
const authHeaders = await this.getAuthHeaders(authConfigId, authConfigType)

this.headers = {
  ...(this.config.defaultHeaders || {}),  // 第1层：数据源默认 headers
  ...headers,                             // 第2层：查询级别的 headers
  ...authHeaders,                         // 第3层：认证 headers（优先级最高）
}
```

合并优先级（从低到高）：
1. **defaultHeaders**：数据源级别的默认 headers，在 datasource.config 中配置
2. **headers**：单个 query 配置的 headers，会覆盖 defaultHeaders 中同名的项
3. **authHeaders**：认证相关的 headers（如 Authorization），优先级最高

### authHeaders 的生成

[packages/server/src/integrations/rest.ts#L659-L687](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/integrations/rest.ts#L659-L687)

`getAuthHeaders` 根据认证类型生成对应的 headers：

- **Basic Auth**：`Authorization: Basic <base64(username:password)>`
- **Bearer Token**：`Authorization: Bearer <token>`
- **OAuth2**：`Authorization: Bearer <access_token>`（token 从缓存或 OAuth2 服务获取）
- **API Key**：如果 location 是 header，则添加自定义 header

### disabledHeaders 删除机制

[packages/server/src/integrations/rest.ts#L711-L717](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/integrations/rest.ts#L711-L717)

合并完成后，`disabledHeaders` 可以禁用指定的 header：

```typescript
if (disabledHeaders) {
  for (let headerKey of Object.keys(this.headers)) {
    if (disabledHeaders[headerKey]) {
      delete this.headers[headerKey]
    }
  }
}
```

`disabledHeaders` 是一个对象，key 为 header 名称，value 为 truthy 值时表示禁用该 header。这通常用于在 UI 中临时关闭某些默认 header。

---

## fetchWithBlacklist 安全机制详解

### 位置

[packages/backend-core/src/utils/outboundFetch.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/utils/outboundFetch.ts)

`fetchWithBlacklist` 是 Budibase 所有出站 HTTP 请求的安全网关，提供了多层安全防护。

### 1. URL 解析与协议校验

[packages/backend-core/src/utils/outboundFetch.ts#L16-L33](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/utils/outboundFetch.ts#L16-L33)

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

安全检查点：
- **协议限制**：仅允许 `http:` 和 `https:` 协议，防止 `file://`、`ftp://` 等危险协议
- **禁止内嵌凭证**：URL 中不能包含 `username:password@` 形式的凭证

### 2. DNS 解析与黑名单检查

[packages/backend-core/src/utils/outboundFetch.ts#L39-L53](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/utils/outboundFetch.ts#L39-L53)

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
```

流程：
1. 解析 URL 获取 hostname
2. 调用 `resolveAddress` 进行 DNS 解析，获取所有 IP 地址
3. 逐个检查 IP 是否在黑名单中
4. 只要有一个 IP 在黑名单中，整个请求就被拒绝

### 3. Pinned Agent 防 DNS 重绑定

[packages/backend-core/src/utils/outboundFetch.ts#L55-L68](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/utils/outboundFetch.ts#L55-L68)

```typescript
function makePinnedAgent(url: string, ip: string): http.Agent | https.Agent {
  const protocol = new URL(url).protocol
  const lookup: LookupFunction = (_hostname, _options, callback) => {
    const family = ip.includes(":") ? 6 : 4
    if (typeof _options === "object" && _options?.all) {
      callback(null, [{ address: ip, family }])
      return
    }
    callback(null, ip, family)
  }
  return protocol === "https:"
    ? new https.Agent({ lookup })
    : new http.Agent({ lookup })
}
```

**DNS 重绑定攻击防护原理**：

普通的 HTTP 请求流程是：
1. 解析 DNS → 得到 IP
2. 检查 IP 是否安全
3. 建立连接 → 再次解析 DNS（可能得到不同的 IP）

攻击者可以让 DNS 第一次返回一个安全的 IP，第二次返回内网 IP，从而绕过检查。

**Pinned Agent 的解决方案**：
- 先解析一次 DNS，得到 IP 并通过黑名单检查
- 创建一个自定义的 Agent，将 lookup 函数固定为返回这个已验证的 IP
- 实际建立连接时，使用这个固定的 IP，而不是再次解析 DNS
- 这样就保证了检查和连接使用的是同一个 IP

### 4. 重定向处理与敏感 Header 剥离

#### 重定向循环

[packages/backend-core/src/utils/outboundFetch.ts#L165-L237](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/utils/outboundFetch.ts#L165-L237)

```typescript
for (let redirects = 0; redirects <= MAX_REDIRECTS; redirects++) {
  const pinnedIp = await resolveSafePinnedIp(nextUrl)
  let response = await fetchFn(nextUrl, {
    ...nextRequest,
    agent: makePinnedAgent(nextUrl, pinnedIp),
  })
  
  if (!isRedirect(response.status)) {
    return response
  }
  
  // 处理重定向...
  const redirectUrl = parseUrl(new URL(location, nextUrl).toString()).toString()
  nextRequest = nextRequestForRedirect(nextRequest, response.status)
  
  // 跨 origin 重定向时剥离敏感 header
  if (shouldStripSensitiveHeadersForRedirect(nextUrl, redirectUrl)) {
    nextRequest = stripSensitiveHeadersForRedirect(nextRequest)
  }
  
  nextUrl = redirectUrl
}
```

关键特性：
- **最多 5 次重定向**（`MAX_REDIRECTS = 5`）
- **每次重定向都重新检查黑名单**：每个重定向 URL 都会重新解析 DNS 并检查黑名单
- **每次重定向都使用 pinned agent**：防止重定向过程中的 DNS 重绑定

#### 跨 Origin 重定向的敏感 Header 剥离

[packages/backend-core/src/utils/outboundFetch.ts#L91-L110](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/utils/outboundFetch.ts#L91-L110)

```typescript
const SENSITIVE_REDIRECT_HEADERS = [
  "authorization",
  "cookie",
  "cookie2",
  "proxy-authorization",
]

function shouldStripSensitiveHeadersForRedirect(
  currentUrl: string,
  redirectUrl: string
): boolean {
  return new URL(currentUrl).origin !== new URL(redirectUrl).origin
}

function stripSensitiveHeadersForRedirect(request) {
  const headers = new Headers(request.headers as RequestInit["headers"])
  SENSITIVE_REDIRECT_HEADERS.forEach(header => headers.delete(header))
  return { ...request, headers }
}
```

**安全原理**：
- 当请求从一个 origin 重定向到另一个不同的 origin 时
- 自动移除 `Authorization`、`Cookie`、`Proxy-Authorization` 等敏感 headers
- 防止认证凭证被泄漏到第三方网站

---

## Blacklist 策略与 SELF_HOSTED/BLACKLIST_IPS 配置

### 位置

[packages/backend-core/src/blacklist/blacklist.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/blacklist/blacklist.ts)

### 默认黑名单

[packages/backend-core/src/blacklist/blacklist.ts#L6-L16](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/blacklist/blacklist.ts#L6-L16)

```typescript
const DEFAULT_BLACKLIST = [
  "127.0.0.0/8",      // 回环地址
  "10.0.0.0/8",       // A 类私网
  "172.16.0.0/12",    // B 类私网
  "192.168.0.0/16",   // C 类私网
  "169.254.0.0/16",   // 链路本地地址
  "0.0.0.0/8",        // 本网地址
  "::1/128",          // IPv6 回环
  "fc00::/7",         // IPv6 唯一本地地址
  "fe80::/10",        // IPv6 链路本地地址
] as const
```

默认黑名单包含了所有内网/私网 IP 段，这是为了防止 SSRF（Server-Side Request Forgery）攻击。

### 默认黑名单的应用条件

[packages/backend-core/src/blacklist/blacklist.ts#L21-L23](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/blacklist/blacklist.ts#L21-L23)

```typescript
function shouldApplyDefaultBlacklist() {
  return !(env.SELF_HOSTED && env.BLACKLIST_IPS !== undefined)
}
```

**逻辑解读**：

默认黑名单的应用条件是 `!(SELF_HOSTED && BLACKLIST_IPS !== undefined)`，即：

| SELF_HOSTED | BLACKLIST_IPS | 默认黑名单是否应用 |
|-------------|---------------|-------------------|
| false（云托管） | 任意 | ✅ 是（强制应用） |
| true（自托管） | 未设置 | ✅ 是（默认应用） |
| true（自托管） | 已设置 | ❌ 否（用户自定义） |

**关键结论**：
- **云托管环境**：默认黑名单始终生效，保障云平台安全
- **自托管环境**：
  - 默认情况下，内网 IP 仍然被阻止（安全默认）
  - 只有当用户显式设置了 `BLACKLIST_IPS` 环境变量时，默认黑名单才会被完全禁用
  - 用户可以通过 `BLACKLIST_IPS` 自定义需要阻止的 IP 列表

### BLACKLIST_IPS 配置

[packages/backend-core/src/blacklist/blacklist.ts#L117-L134](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/blacklist/blacklist.ts#L117-L134)

```typescript
const configuredBlacklist = env.BLACKLIST_IPS?.split(",") || []
for (const entry of configuredBlacklist) {
  const trimmed = entry.trim()
  if (!trimmed) continue

  const [ip] = trimmed.split("/")
  if (net.isIP(ip)) {
    addEntryToBlacklist(next, trimmed)
    continue
  }

  // 如果是域名，先解析 DNS，再添加所有解析到的 IP
  const addresses = await lookup(trimmed)
  for (const address of addresses) {
    addEntryToBlacklist(next, address)
  }
}
```

`BLACKLIST_IPS` 是一个逗号分隔的列表，支持：
- 单个 IP：`192.168.1.1`
- CIDR 网段：`10.0.0.0/8`
- 域名：`example.com`（会解析 DNS 并添加所有 IP）

### isBlacklisted 检查

[packages/backend-core/src/blacklist/blacklist.ts#L139-L155](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/blacklist/blacklist.ts#L139-L155)

```typescript
export async function isBlacklisted(address: string): Promise<boolean> {
  if (!blackList) {
    await refreshBlacklist()
  }

  let ips: string[]
  try {
    ips = await resolveAddress(address)
  } catch (e) {
    if (e instanceof TypeError) {
      console.error("Black list error: could not parse address: ", address)
      return true  // 解析失败时，默认阻止
    }
    return shouldApplyDefaultBlacklist()  // 其他错误时，根据配置决定
  }
  return ips.some(ip => blackList!.check(ip, getIpVersion(ip)))
}
```

---

## OAuth2 401 重试机制

### 位置

[packages/server/src/integrations/rest.ts#L830-L835](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/integrations/rest.ts#L830-L835)

### 重试逻辑

```typescript
if (response.status === 401 && retry401) {
  if (authConfigType === RestAuthType.OAUTH2 && authConfigId) {
    await sdk.oauth2.cleanStoredTokensForAuthConfig(authConfigId)
    return await this._req(query, false)
  }
}
```

### 重试条件

1. **HTTP 状态码为 401**（未授权）
2. **`retry401` 为 true**（首次调用时默认为 true）
3. **认证类型为 OAuth2**（`authConfigType === RestAuthType.OAUTH2`）
4. **存在 authConfigId**

### 只重试一次的机制

`_req` 方法的参数 `retry401` 控制是否重试：
- 首次调用：`retry401 = true`（默认值）
- 重试调用：`retry401 = false`（显式传入）

这样保证了最多只重试一次，避免无限循环。

### cleanStoredTokensForAuthConfig

[packages/server/src/sdk/workspace/oauth2/utils.ts#L192-L200](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/oauth2/utils.ts#L192-L200)

```typescript
export async function cleanStoredTokensForAuthConfig(
  authConfigId: string,
  datasourceId?: string
) {
  await cleanStoredToken(authConfigId)
  if (datasourceId) {
    await cleanStoredToken(`${datasourceId}:${authConfigId}`)
  }
}
```

该函数会清除两个可能的缓存 key：
1. `authConfigId` 直接作为 key
2. `${datasourceId}:${authConfigId}` 组合作为 key（如果提供了 datasourceId）

清除缓存后，下一次获取 token 时会重新向 OAuth2 服务器请求新的 access token。

### Token 获取与缓存

[packages/server/src/sdk/workspace/oauth2/utils.ts#L102-L116](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/oauth2/utils.ts#L102-L116)

```typescript
export async function getToken(id: string) {
  const token = await cache.withCacheWithDynamicTTL(
    cache.CacheKey.OAUTH2_TOKEN(id),
    async () => {
      const config = await get(id)
      if (!config) {
        throw new HTTPError(`oAuth config ${id} could not be found`, 400)
      }
      return fetchAndParseToken(config)
    }
  )

  await trackUsage(id)
  return token
}
```

token 是通过 Redis 缓存的，TTL 由 OAuth2 响应中的 `expires_in` 决定。当 401 发生时：
1. 清除缓存的 token
2. 重新请求（会重新获取新的 token）
3. 只重试一次，避免无限重试

---

## 为什么 127.0.0.1:5984 默认被阻止

访问 `http://127.0.0.1:5984` 被阻止的完整原因链：

### 1. URL 进入 fetchWithBlacklist

REST Query 预览的所有 HTTP 请求都会经过 `fetchWithBlacklist` 安全网关。

### 2. DNS 解析

`127.0.0.1` 本身就是一个 IP 地址，`resolveAddress` 直接返回 `["127.0.0.1"]`。

### 3. 黑名单检查

`isBlacklisted("127.0.0.1")` 检查该 IP 是否在黑名单中。

### 4. 默认黑名单匹配

默认黑名单中的 `127.0.0.0/8` 网段包含了 `127.0.0.1`，因此匹配成功。

### 5. 请求被拒绝

抛出错误：`"URL is blocked or could not be resolved safely."`

### 如何允许访问内网地址？

在自托管环境中，可以通过以下方式修改：

**方式一：设置 BLACKLIST_IPS 环境变量（禁用默认黑名单，自定义黑名单）**

```bash
# 注意：这会禁用所有默认内网阻止！
# 只添加你想阻止的 IP，留空表示不阻止任何 IP
BLACKLIST_IPS=""
```

但需要注意，设置 `BLACKLIST_IPS` 后，**默认黑名单会被完全禁用**，所有内网 IP 都可以访问，这可能带来安全风险。

**方式二：保持默认，使用公网地址**

推荐的做法是将需要访问的服务绑定到公网 IP 或使用域名，而不是直接访问内网地址。

**安全提示**：
- 阻止内网访问是为了防止 SSRF 攻击
- 攻击者可能通过构造请求访问内部服务（如 CouchDB、Redis 等）
- 在生产环境中，建议谨慎开启内网访问权限

---

## 附录：关键文件索引

| 文件 | 作用 |
|------|------|
| [query/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/query/index.ts) | Query Preview API 控制器 |
| [threads/query.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/threads/query.ts) | QueryRunner 执行器 |
| [integrations/rest.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/integrations/rest.ts) | RestIntegration 实现 |
| [outboundFetch.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/utils/outboundFetch.ts) | fetchWithBlacklist 安全网关 |
| [blacklist/blacklist.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/blacklist/blacklist.ts) | IP 黑名单管理 |
| [oauth2/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/oauth2/utils.ts) | OAuth2 token 管理 |
| [integrations/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/integrations/index.ts) | Integration 类映射 |
