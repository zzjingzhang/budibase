# AIAgent 创建与更新：部署密钥与知识配置保护机制分析

## 一、路由入口：builderAdminRoutes 权限控制

所有 Agent 的 CRUD 操作均通过 `builderAdminRoutes` 暴露，确保只有 Builder 或 Admin 角色才能访问。

**文件位置**：[aiAgents.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/aiAgents.ts#L27-L32)

```typescript
builderAdminRoutes
  .get("/api/agent", ai.fetchAgents)
  .post("/api/agent", createAgentValidator(), ai.createAgent)
  .put("/api/agent", updateAgentValidator(), ai.updateAgent)
  // ...
```

`builderAdminRoutes` 定义于 [standard.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/endpointGroups/standard.ts#L21-L22)，通过 `auth.builderOrAdmin` 中间件进行权限校验，且调用 `lockMiddleware()` 锁定中间件链，防止后续路由绕过权限检查。

---

## 二、Controller 层保护机制

### 2.1 createAgent：createdBy 转 globalId + 知识配置强制清空

**文件位置**：[agents.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/agents.ts#L301-L336)

#### （1）createdBy 转换为 globalId

```typescript
const createdBy = ctx.user?._id!
const globalId = db.getGlobalIDFromUserMetadataID(createdBy)
```

- 从请求上下文中获取当前用户的 `_id`（metadata ID）
- 通过 `db.getGlobalIDFromUserMetadataID()` 转换为全局用户 ID
- 确保 `createdBy` 字段存储的是全局唯一标识，而非应用内的局部 ID

#### （2）强制 knowledgeSources / knowledgeBases 为 undefined

```typescript
const createRequest: RequiredKeys<...> = {
  // ... 其他字段
  knowledgeSources: undefined,
  knowledgeBases: undefined,
}
```

- 创建 Agent 时，**忽略客户端提交的 `knowledgeSources` 和 `knowledgeBases`**
- 这两个字段被显式设置为 `undefined`，防止客户端通过创建接口直接注入知识配置
- 知识配置必须通过专门的知识管理接口（如文件上传、SharePoint 连接等）进行设置

---

### 2.2 updateAgent：stableSerialize 比较 + 禁止普通 endpoint 修改知识配置

**文件位置**：[agents.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/agents.ts#L338-L394)

#### （1）stableSerialize：稳定序列化比较

```typescript
const stableSerialize = (value: unknown): string => {
  if (Array.isArray(value)) {
    return `[${value.map(stableSerialize).join(",")}]`
  }
  if (value && typeof value === "object") {
    const entries = Object.entries(value as Record<string, unknown>).sort(
      ([a], [b]) => a.localeCompare(b)
    )
    return `{${entries
      .map(([key, item]) => `${JSON.stringify(key)}:${stableSerialize(item)}`)
      .join(",")}}`
  }
  return JSON.stringify(value)
}
```

- 对对象按 key 字母排序后再序列化
- 确保即使属性顺序不同，相同内容也会产生相同的序列化结果
- 用于精确比较知识配置是否发生了实质性变化

#### （2）禁止从普通 endpoint 修改知识配置

对 `knowledgeSources` 的校验：

```typescript
if (Object.prototype.hasOwnProperty.call(rawBody, "knowledgeSources")) {
  const incoming = normalizeKnowledgeSources(rawBody.knowledgeSources)
  const current = normalizeKnowledgeSources(existing.knowledgeSources || [])
  if (stableSerialize(incoming) !== stableSerialize(current)) {
    throw new HTTPError(
      "knowledgeSources cannot be updated from this endpoint",
      400
    )
  }
}
```

对 `knowledgeBases` 的校验：

```typescript
if (Object.prototype.hasOwnProperty.call(rawBody, "knowledgeBases")) {
  const incoming = normalizeKnowledgeBases(rawBody.knowledgeBases)
  const current = normalizeKnowledgeBases(existing.knowledgeBases || [])
  if (stableSerialize(incoming) !== stableSerialize(current)) {
    throw new HTTPError(
      "knowledgeBases cannot be updated from this endpoint",
      400
    )
  }
}
```

**保护逻辑**：

1. 首先检查请求体中是否包含 `knowledgeSources` 或 `knowledgeBases` 字段
2. 如果包含，则将传入值与当前数据库中的值进行归一化处理
3. 使用 `stableSerialize` 进行精确比较
4. 如果两者不一致，返回 **400 错误**，明确告知该 endpoint 不允许修改知识配置

**归一化处理**：

- `normalizeKnowledgeBases`：去空格、过滤空值、排序
- `normalizeKnowledgeSources`：提取 `id/type/config` 字段、按 `id:type` 排序

这样设计的目的是：知识配置的修改必须通过专门的知识管理接口（如文件上传、知识源同步等）进行，而不能通过通用的更新接口绕过校验。

---

## 三、SDK 层：密钥加密与保留机制

**文件位置**：[crud.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/agents/crud.ts)

### 3.1 SECRET_MASK 与 bbai_enc:: 前缀

```typescript
const SECRET_MASK = "********"
const SECRET_ENCODING_PREFIX = "bbai_enc::"
```

- `SECRET_MASK`：用于在返回给客户端时遮盖密钥的掩码字符串
- `SECRET_ENCODING_PREFIX`：标识数据库中存储的密钥是否已加密的前缀

### 3.2 encodeSecret / decodeSecret

```typescript
const encodeSecret = (value?: string): string | undefined => {
  if (!value || value.startsWith(SECRET_ENCODING_PREFIX)) {
    return value
  }
  return `${SECRET_ENCODING_PREFIX}${encryption.encrypt(value)}`
}

const decodeSecret = (value?: string): string | undefined => {
  if (!value || !value.startsWith(SECRET_ENCODING_PREFIX)) {
    return value
  }
  return encryption.decrypt(value.slice(SECRET_ENCODING_PREFIX.length))
}
```

**encodeSecret**：
- 如果值为空或已带有加密前缀，则原样返回
- 否则，使用 `encryption.encrypt()` 加密，并加上 `bbai_enc::` 前缀

**decodeSecret**：
- 如果值为空或不带加密前缀，则原样返回
- 否则，去掉前缀后使用 `encryption.decrypt()` 解密

### 3.3 各集成的密钥编解码

针对每个第三方集成（Discord、Slack、Telegram、MS Teams），都有对应的编码和解码函数：

| 集成类型 | 密钥字段 |
|---------|---------|
| Discord | `publicKey`, `botToken` |
| Slack | `botToken`, `signingSecret` |
| Telegram | `botToken`, `webhookSecretToken` |
| MS Teams | `appPassword` |

例如 Discord 的编码函数：

```typescript
const encodeDiscordIntegrationSecrets = (
  discordIntegration?: Agent["discordIntegration"]
) => {
  if (!discordIntegration) {
    return discordIntegration
  }
  return {
    ...discordIntegration,
    publicKey: encodeSecret(discordIntegration.publicKey),
    botToken: encodeSecret(discordIntegration.botToken),
  }
}
```

在 `create` 和 `update` 操作中，保存到数据库前会调用对应的 encode 函数加密密钥；从数据库读取后，会通过 `withAgentDefaults` 调用 decode 函数解密密钥。

### 3.4 mergeXxxIntegration：保留已保存密钥

以 `mergeDiscordIntegration` 为例：

```typescript
const mergeDiscordIntegration = ({
  existing,
  incoming,
}: {
  existing?: Agent["discordIntegration"]
  incoming?: Agent["discordIntegration"]
}) => {
  if (incoming === undefined) {
    return existing
  }
  if (!incoming) {
    return incoming
  }

  const merged = {
    ...(existing || {}),
    ...incoming,
  }

  if (incoming.publicKey === SECRET_MASK && existing?.publicKey) {
    merged.publicKey = existing.publicKey
  }

  if (incoming.botToken === SECRET_MASK && existing?.botToken) {
    merged.botToken = existing.botToken
  }

  return merged
}
```

**保留逻辑**：

1. 如果 `incoming` 为 `undefined`（未提交该集成配置），直接返回现有配置
2. 如果 `incoming` 为 falsy（如 `null` 或空对象），表示要清除配置，直接返回传入值
3. 先做基础合并：以现有配置为基础，用传入配置覆盖
4. **关键保护**：如果传入的密钥字段值等于 `SECRET_MASK`（即 `********`），说明客户端只是回传了掩码值而非真实密钥，则用已保存的真实密钥替换

这样设计的好处是：
- 客户端获取 Agent 时看到的是掩码后的密钥
- 更新时如果用户没有修改密钥，客户端回传掩码值
- 服务器检测到掩码值后，自动保留数据库中的真实密钥
- 只有当用户提交了新的密钥值（非掩码）时，才会更新

同样的模式也应用于 `mergeSlackIntegration`、`mergeTelegramIntegration` 和 `mergeMSTeamsIntegration`。

---

## 四、Live Agent 配置校验

**文件位置**：[utils.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/workspace/ai/agents/utils.ts#L298-L313)

```typescript
export const assertAgentHasValidConfig = async (agent: Agent) => {
  if (!agent.aiconfig) {
    throw new HTTPError(
      "Agent is not properly configured: missing AI config",
      422
    )
  }

  const aiConfig = await sdk.ai.configs.find(agent.aiconfig)
  if (!aiConfig) {
    throw new HTTPError(
      `Agent is not properly configured: AI config "${agent.aiconfig}" not found`,
      422
    )
  }
}
```

在 `create` 和 `update` 操作中，如果 Agent 被设置为 `live: true`，则会调用此函数进行校验：

```typescript
if (agent.live) {
  await assertAgentHasValidConfig(agent)
}
```

校验内容：
1. 检查 `aiconfig` 字段是否存在
2. 检查对应的 AI 配置在数据库中是否真实存在

这确保了上线（live）的 Agent 必须有有效的 AI 配置，防止部署一个无法正常工作的 Agent。

---

## 五、返回客户端：敏感信息脱敏

### 5.1 obfuscateAgentSecrets：密钥掩码

**文件位置**：[agents.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/agents.ts#L27-L62)

```typescript
const maskSecretFields = <T extends object>(obj: T, fields: (keyof T)[]): T => {
  const result = { ...obj }
  for (const field of fields) {
    if (result[field]) {
      result[field] = SECRET_MASK as T[typeof field]
    }
  }
  return result
}

const obfuscateAgentSecrets = (agent: Agent): Agent => ({
  ...agent,
  ...(agent.discordIntegration && {
    discordIntegration: maskSecretFields(agent.discordIntegration, [
      "publicKey",
      "botToken",
    ]),
  }),
  // ... Slack / MS Teams / Telegram 同理
})
```

对每个集成中的密钥字段，用 `********` 替换，确保真实密钥不会泄露到客户端。

### 5.2 withoutKnowledgeConfig：移除知识配置

**文件位置**：[agents.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/ai/agents.ts#L64-L71)

```typescript
const withoutKnowledgeConfig = <T extends Agent>(agent: T) => {
  const {
    knowledgeSources: _knowledgeSources,
    knowledgeBases: _knowledgeBases,
    ...rest
  } = agent
  return rest
}
```

在 `createAgent` 和 `updateAgent` 的返回结果中，会调用此函数移除 `knowledgeSources` 和 `knowledgeBases` 字段：

```typescript
ctx.body = withoutKnowledgeConfig(obfuscateAgentSecrets(agent))
```

这样做的目的：
- 减少敏感信息暴露面
- 知识配置通过专门的知识管理接口单独获取
- 遵循最小权限原则

---

## 六、整体安全架构总结

| 保护层面 | 机制 | 作用 |
|---------|------|------|
| **路由层** | `builderAdminRoutes` + `auth.builderOrAdmin` | 限制只有 Builder/Admin 才能操作 Agent |
| **Controller 层（创建）** | `knowledgeSources/knowledgeBases: undefined` | 防止通过创建接口注入知识配置 |
| **Controller 层（更新）** | `stableSerialize` 比较 + 400 错误 | 防止通过通用更新接口修改知识配置 |
| **Controller 层（返回）** | `obfuscateAgentSecrets` | 密钥掩码，防止泄露 |
| **Controller 层（返回）** | `withoutKnowledgeConfig` | 移除知识配置，减少暴露面 |
| **SDK 层（存储）** | `encodeSecret` / `decodeSecret` + `bbai_enc::` 前缀 | 数据库中密钥加密存储 |
| **SDK 层（更新）** | `mergeXxxIntegration` + `SECRET_MASK` 检测 | 自动保留已保存密钥，支持掩码回传 |
| **SDK 层（上线）** | `assertAgentHasValidConfig` | Live Agent 必须有有效 AI 配置 |
| **用户身份** | `getGlobalIDFromUserMetadataID` | 统一使用全局用户 ID |

通过多层防护机制，Budibase 确保了 AIAgent 的部署密钥和知识配置在创建、更新、存储、返回等各个环节都得到了有效保护。
