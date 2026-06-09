# Budibase 登录与密码重置安全路径追踪

## 概述

本文档详细追踪了 Budibase Worker 服务中 `POST /api/global/auth/:tenantId/login` 登录路径和密码重置相关安全路径的完整流程，包括中间件顺序、校验逻辑、锁定机制、会话管理等核心安全机制。

---

## 一、Worker API 中间件顺序

Worker API 的中间件链定义在 [packages/worker/src/api/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/index.ts) 中，按以下顺序执行：

### 1.1 中间件执行顺序

| 顺序 | 中间件 | 说明 | 相关文件 |
|------|--------|------|----------|
| 1 | `middleware.errorHandling` | 错误处理中间件 | [backend-core/middleware](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/middleware) |
| 2 | `middleware.featureFlagCookie` | 功能标志 Cookie 中间件 | [backend-core/middleware](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/middleware) |
| 3 | `compress` | 响应压缩中间件 (gzip/deflate) | koa-compress |
| 4 | `/health` | 健康检查端点 | - |
| 5 | `auth.buildAuthMiddleware(PUBLIC_ENDPOINTS)` | 认证中间件 | [backend-core/src/auth/auth.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/auth/auth.ts#L47) → [authenticated.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/middleware/authenticated.ts) |
| 6 | `auth.buildTenancyMiddleware(PUBLIC_ENDPOINTS, NO_TENANCY_ENDPOINTS)` | 租户中间件 | [backend-core/src/auth/auth.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/auth/auth.ts#L48) → [tenancy.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/middleware/tenancy.ts) |
| 7 | `middleware.activeTenant(ALLOW_INACTIVE_TENANT_ENDPOINTS)` | 活跃租户检查 | - |
| 8 | `auth.buildCsrfMiddleware({ noCsrfPatterns: NO_CSRF_ENDPOINTS })` | CSRF 防护中间件 | [backend-core/src/auth/auth.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/auth/auth.ts#L49) → [csrf.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/middleware/csrf.ts) |
| 9 | `pro.licensing()` | 授权许可中间件 | [pro/src/middleware/licensing.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/middleware/licensing.ts) |
| 10 | 自定义授权检查 | 检查 `ctx.publicEndpoint`、`ctx.isAuthenticated`、`ctx.user.budibaseAccess`、`ctx.internal` | [worker/src/api/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/index.ts#L160-L171) |
| 11 | `middleware.auditLog` | 审计日志中间件 | - |

### 1.2 PUBLIC_ENDPOINTS

定义在 [packages/worker/src/api/index.ts#L10-L70](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/index.ts#L10-L70)，包含以下公开端点：

- `POST /api/global/auth/:tenantId` - 所有 POST 认证路由（含 login）
- `GET /api/global/auth/:tenantId` - 所有 GET 认证路由
- `GET /api/global/configs/public` - 公开配置
- `GET /api/global/configs/checklist` - 检查清单
- `POST /api/global/users/init` - 用户初始化
- `POST /api/global/users/invite/accept` - 接受邀请
- `GET /api/system/environment` - 系统环境
- `GET /api/system/status` - 系统状态
- 等等...

**关键点**：登录端点 `/api/global/auth/:tenantId/login` 属于公开端点，通过 `buildAuthMiddleware` 时会被标记为 `ctx.publicEndpoint = true`，跳过认证检查。

### 1.3 CSRF 例外

`NO_CSRF_ENDPOINTS = [...PUBLIC_ENDPOINTS]`，即所有公开端点都免除 CSRF 检查。CSRF 中间件逻辑见 [csrf.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/middleware/csrf.ts)：

- 排除 GET、HEAD、OPTIONS 方法
- 排除非表单内容类型（application/json 等）
- 排除内部 API 请求（`ctx.internal`）
- 排除 `noCsrfPatterns` 匹配的端点
- 仅当用户会话中有 `csrfToken` 时才强制检查

### 1.4 pro.licensing 中间件

定义在 [pro/src/middleware/licensing.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/pro/src/middleware/licensing.ts)：

- 仅当 `licensingCheck(ctx)` 返回 true（默认是 `!!ctx.user`）时执行
- 自托管环境且有默认许可证时，设置 `SELF_FREE_LICENSE`
- 否则从缓存获取许可证并设置到 `ctx.user.license`
- 可选检查用户配额限制（`checkUsersLimit`）

---

## 二、Login Route：Joi 校验与 Lockout 中间件

登录路由定义在 [packages/worker/src/api/routes/global/auth.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/routes/global/auth.ts)。

### 2.1 路由定义

```typescript
loggedInRoutes
  .post(
    "/api/global/auth/:tenantId/login",
    buildAuthValidation(),  // Joi 校验
    lockout,                 // 锁定检查中间件
    authController.login     // 登录控制器
  )
```

**注意**：`loggedInRoutes` 是一个空的端点组（没有额外的前置中间件），见 [endpointGroups/standard.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/routes/endpointGroups/standard.ts#L23-L24)。

### 2.2 Joi 校验 (buildAuthValidation)

定义在 [auth.ts#L7-L13](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/routes/global/auth.ts#L7-L13)：

```typescript
function buildAuthValidation() {
  return auth.joiValidator.body(Joi.object({
    username: Joi.string().required(),
    password: Joi.string().required(),
  }).required().unknown(false))
}
```

校验规则：
- `username`：字符串，必填
- `password`：字符串，必填
- `.unknown(false)`：禁止额外字段

### 2.3 Lockout 中间件

定义在 [packages/worker/src/middleware/lockout.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/middleware/lockout.ts)。

**执行流程**：

1. 从请求体获取 `username`（即 email）
2. 如果没有 email，直接放行到下一个中间件
3. 通过 `userSdk.db.getUserByEmail(email)` 查找用户
4. 如果用户存在且被锁定（`cache.get(lockKey(email))` 存在）：
   - 设置响应头 `X-Account-Locked: 1`
   - 设置响应头 `Retry-After: {LOGIN_LOCKOUT_SECONDS}`
   - 抛出 403 错误："Account temporarily locked. Try again later."
5. 否则放行

**锁定 Key 格式**：`auth:login:lock:{lowercase_email}`

**环境变量**：
- `LOGIN_LOCKOUT_SECONDS`：锁定时长，默认 900 秒（15分钟），见 [environment.ts#L92](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/environment.ts#L92)

---

## 三、Controller.login 登录逻辑

登录控制器定义在 [packages/worker/src/api/controllers/global/auth.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/controllers/global/auth.ts#L114-L161)。

### 3.1 核心流程

```
请求到达
  ↓
检查 isPreventPasswordActions (SSO 用户禁止密码登录)
  ↓ 是 → 403 "Invalid credentials"
  ↓ 否
Passport Local 认证
  ├── 失败：
  │   ├── 若用户存在 → onFailed(email) 增加失败计数
  │   ├── 检查是否已锁定 → handleLockoutResponse
  │   └── passportCallback (返回错误)
  └── 成功：
      ├── clearFailureState(email) 清除失败状态
      ├── passportCallback (登录用户，设置 cookie/header)
      ├── 触发登录事件
      └── 返回响应
```

### 3.2 isPreventPasswordActions 检查

**位置**：[auth.ts#L120-L126](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/controllers/global/auth.ts#L120-L126)

```typescript
const dbUser = await userSdk.db.getUserByEmail(email)
if (dbUser && (await userSdk.db.isPreventPasswordActions(dbUser))) {
  console.log(`[auth] login prevented due to sso enforcement email=${normalizeEmail(email)}`)
  ctx.throw(403, "Invalid credentials")
}
```

**isPreventPasswordActions 实现**：见 [backend-core/src/users/db.ts#L89-L111](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/users/db.ts#L89-L111)

返回 `true`（禁止密码操作）的条件（任一满足）：
1. ~~SSO 维护模式且用户是 admin~~ → 返回 `false`（例外）
2. SSO 全局强制启用 (`UserDB.features.isSSOEnforced()`)
3. 用户本身是 SSO 用户 (`isSSOUser(user)`)
4. 用户是账户持有人且账户使用 SSO (`isSSOAccount(account)`)

### 3.3 失败处理 (onFailed)

**位置**：[auth.ts#L56-L72](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/controllers/global/auth.ts#L56-L72)

```typescript
const onFailed = async (email: string) => {
  if (!email) return
  const key = failKey(email)
  const currentAttempt = Number((await cache.get(key)) || 0) || 0
  const nextAttempt = currentAttempt + 1
  await cache.store(key, nextAttempt, env.LOGIN_LOCKOUT_SECONDS)
  
  if (nextAttempt >= env.LOGIN_MAX_FAILED_ATTEMPTS) {
    await cache.store(lockKey(email), "1", env.LOGIN_LOCKOUT_SECONDS)
    await cache.destroy(key)
  }
}
```

**关键点**：
- **仅对存在的用户执行**：`if (dbUser) { await onFailed(email) }`
- 失败计数 Key：`auth:login:fail:{lowercase_email}`
- 达到 `LOGIN_MAX_FAILED_ATTEMPTS`（默认 5 次）后：
  - 写入锁定 Key：`auth:login:lock:{lowercase_email}`
  - 清除失败计数 Key
  - 锁定时长：`LOGIN_LOCKOUT_SECONDS`（默认 900 秒）

### 3.4 成功处理 (clearFailureState)

**位置**：[auth.ts#L73-L77](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/controllers/global/auth.ts#L73-L77)

```typescript
const clearFailureState = async (email: string) => {
  if (!email) return
  await cache.destroy(failKey(email))
  await cache.destroy(lockKey(email))
}
```

登录成功后清除失败计数和锁定状态。

### 3.5 passportCallback

**位置**：[auth.ts#L79-L112](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/controllers/global/auth.ts#L79-L112)

```typescript
async function passportCallback(ctx, user, err = null, info = null) {
  if (err || !user) {
    return ctx.throw(403, info ? info : "Unauthorized")
  }

  const loginResult = await authSdk.loginUser(user)

  // 设置 Cookie (浏览器访问)
  setCookie(ctx, loginResult.token, Cookie.Auth, {
    sign: false,
    httpOnly: true,
  })
  
  // 设置 Header (API 访问)
  ctx.set(Header.TOKEN, loginResult.token)

  // 会话失效信息
  if (loginResult.invalidatedSessionCount > 0) {
    ctx.set("X-Session-Invalidated-Count", loginResult.invalidatedSessionCount.toString())
  }
}
```

---

## 四、Passport Local 策略

Passport Local 策略定义在 [packages/backend-core/src/middleware/passport/local.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/middleware/passport/local.ts)。

### 4.1 认证流程

```
authenticate(ctx, email, password, done)
  ↓
email 为空? → "Email Required"
  ↓
password 为空? → "Password Required"
  ↓
查找用户: users.getGlobalUserByEmail(email)
  ↓
用户不存在? → "Invalid credentials"
  ↓
用户状态 INACTIVE? → "Invalid credentials"
  ↓
用户无 password? → "This account has expired. Please reset your password"
  ↓
密码比对: compare(password, dbUser.password)
  ↓ 不匹配 → "Invalid credentials"
  ↓ 匹配
删除 password 字段
返回 done(null, dbUser)
```

### 4.2 关键检查点

| 检查项 | 条件 | 错误信息 |
|--------|------|----------|
| Email 必填 | `!email` | "Email Required" |
| Password 必填 | `!password` | "Password Required" |
| 用户存在 | `dbUser == null` | "Invalid credentials" |
| 用户非停用 | `dbUser.status === UserStatus.INACTIVE` | "Invalid credentials" |
| 有密码设置 | `!dbUser.password` | "This account has expired..." (EXPIRED) |
| 密码匹配 | `!compare(password, dbUser.password)` | "Invalid credentials" |

### 4.3 安全设计

- **用户枚举防护**：用户不存在、inactive、密码错误都返回相同的 "Invalid credentials" 错误
- **无密码用户**：可能是 SSO 用户或过期账户，返回专门的过期提示
- **密码清除**：认证成功后从返回用户对象中删除 password 字段

---

## 五、authSdk.loginUser 会话创建

`loginUser` 定义在 [packages/worker/src/sdk/auth/auth.ts#L18-L41](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/sdk/auth/auth.ts#L18-L41)。

### 5.1 执行流程

```
loginUser(user)
  ↓
生成 sessionId = newid()
  ↓
获取 tenantId
  ↓
sessions.createASession(userId, { sessionId, tenantId, email })
  ├── 检查现有会话数
  ├── 超过 MAX_SESSIONS_PER_USER? → 淘汰最旧会话
  ├── 生成 csrfToken (uuidv4)
  └── 存储会话到 Redis
  ↓
签发 JWT token
  ↓
返回 { token, invalidatedSessionCount }
```

### 5.2 createASession 会话创建与淘汰

定义在 [packages/backend-core/src/security/sessions.ts#L76-L120](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/security/sessions.ts#L76-L120)。

**会话淘汰逻辑**：

```typescript
const existingSessions = await getSessionsForUser(userId)

if (existingSessions.length >= MAX_SESSIONS_PER_USER) {
  const sortedSessions = existingSessions.sort(
    (a, b) => new Date(a.createdAt).getTime() - new Date(b.createdAt).getTime()
  )
  
  const sessionsToRemove = existingSessions.length - MAX_SESSIONS_PER_USER + 1
  const sessionIdsToInvalidate = sortedSessions
    .slice(0, sessionsToRemove)
    .map(session => session.sessionId)
  
  await invalidateSessions(userId, {
    sessionIds: sessionIdsToInvalidate,
    reason: "session limit exceeded",
  })
}
```

**关键参数**：
- `MAX_SESSIONS_PER_USER = 3`，定义在 [shared-core/src/login.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/shared-core/src/login.ts#L1)
- 淘汰策略：FIFO（按创建时间排序，淘汰最旧的）
- 会话过期时间：`SESSION_EXPIRY_SECONDS`，默认 7 天

### 5.3 JWT 签发

```typescript
const token = jwt.sign(
  {
    userId: user._id,
    sessionId,
    tenantId,
    email: user.email,
  },
  coreEnv.JWT_SECRET!
)
```

JWT Payload 包含：
- `userId`：用户 ID
- `sessionId`：会话 ID
- `tenantId`：租户 ID
- `email`：用户邮箱

### 5.4 Cookie 与 Header 设置

在 `passportCallback` 中设置：

| 类型 | 名称 | 属性 | 用途 |
|------|------|------|------|
| Cookie | `Cookie.Auth` | `httpOnly: true`, `sign: false` | 浏览器访问 |
| Header | `Header.TOKEN` | - | API 访问 |

---

## 六、密码重置 (Password Reset)

### 6.1 重置请求 (reset)

**控制器**：[auth.ts#L196-L260](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/controllers/global/auth.ts#L196-L260)

**路由**：`POST /api/global/auth/:tenantId/reset`，带 Joi 校验（email 必填）

#### 6.1.1 速率限制 (Rate Limiting)

采用**双重速率限制**：按 email 和按 IP 分别限制。

```typescript
// rate limit keys
const emailKey = `auth:pwdreset:email:${lcEmail}`
const ipKey = `auth:pwdreset:ip:${ip}`

// 应用速率限制
const nextEmail = await increment(emailKey, PASSWORD_RESET_RATE_EMAIL_WINDOW_SECONDS)
const nextIp = await increment(ipKey, PASSWORD_RESET_RATE_IP_WINDOW_SECONDS)

const emailLimited = nextEmail > PASSWORD_RESET_RATE_EMAIL_LIMIT
const ipLimited = nextIp > PASSWORD_RESET_RATE_IP_LIMIT
```

**速率限制参数**（见 [environment.ts#L95-L102](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/environment.ts#L95-L102)）：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `PASSWORD_RESET_RATE_EMAIL_LIMIT` | 3 | 每邮箱限制次数 |
| `PASSWORD_RESET_RATE_EMAIL_WINDOW_SECONDS` | 900 | 邮箱时间窗口（15分钟） |
| `PASSWORD_RESET_RATE_IP_LIMIT` | 20 | 每 IP 限制次数 |
| `PASSWORD_RESET_RATE_IP_WINDOW_SECONDS` | 900 | IP 时间窗口（15分钟） |

**触发限流时**：
- 设置 `X-RateLimit-*-Limit` 和 `X-RateLimit-*-Remaining` 响应头
- 设置 `Retry-After` 响应头
- 返回 429 状态码："Too many password reset requests. Try again later."

#### 6.1.2 重置逻辑 (authSdk.reset)

**定义**：[worker/src/sdk/auth/auth.ts#L54-L80](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/sdk/auth/auth.ts#L54-L80)

```typescript
export const reset = async (email: string) => {
  // 检查 SMTP 配置
  const configured = await emails.isEmailConfigured()
  if (!configured) {
    throw new HTTPError("Please contact your platform administrator, SMTP is not configured.", 400)
  }

  const user = await userSdk.core.getGlobalUserByEmail(email)
  
  // 用户不存在 → 静默返回 (不暴露用户存在性)
  if (!user) {
    return
  }

  // SSO 用户 → 静默返回 (不暴露 SSO 用户存在性)
  if (await userSdk.db.isPreventPasswordActions(user)) {
    return
  }

  // 发送密码重置邮件
  await emails.sendEmail(email, EmailTemplatePurpose.PASSWORD_RECOVERY, {
    user,
    subject: "{{ company }} platform password reset",
  })
  await events.user.passwordResetRequested(user)
}
```

**安全设计**：
- **用户枚举防护**：用户不存在或为 SSO 用户时，**静默成功返回**，不暴露任何信息
- 无论成功与否，控制器层都返回相同的响应：`{ message: "Please check your email for a reset link." }`

### 6.2 重置更新 (resetUpdate)

**控制器**：[auth.ts#L265-L279](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/controllers/global/auth.ts#L265-L279)

**路由**：`POST /api/global/auth/:tenantId/reset/update`，带 Joi 校验（resetCode + password 必填）

```typescript
export const resetUpdate = async (ctx) => {
  const { resetCode, password } = ctx.request.body
  try {
    await authSdk.resetUpdate(resetCode, password)
    ctx.body = { message: "password reset successfully." }
  } catch (err: any) {
    console.warn(err)
    // 隐藏错误细节
    ctx.throw(400, err.message || "Cannot reset password.")
  }
}
```

#### 6.2.1 authSdk.resetUpdate 实现

**定义**：[worker/src/sdk/auth/auth.ts#L85-L98](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/sdk/auth/auth.ts#L85-L98)

```typescript
export const resetUpdate = async (resetCode: string, password: string) => {
  // 验证重置码并获取 userId
  const { userId } = await cache.passwordReset.getCode(resetCode)

  // 获取用户并更新密码
  let user = await userSdk.db.getUser(userId)
  user.password = password
  user = await userSdk.db.save(user)

  // 失效重置码
  await cache.passwordReset.invalidateCode(resetCode)
  
  // 失效所有会话
  await sessions.invalidateSessions(userId)

  // 触发事件
  delete user.password
  await events.user.passwordReset(user)
}
```

#### 6.2.2 密码重置码缓存

**定义**：[backend-core/src/cache/passwordReset.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/cache/passwordReset.ts)

| 操作 | 说明 | TTL |
|------|------|-----|
| `createCode(userId, info)` | 创建重置码并存储 | 1 小时 |
| `getCode(code)` | 验证并返回 `{ userId, info }` | - |
| `invalidateCode(code)` | 删除重置码 | - |

#### 6.2.3 invalidateSessions 会话失效

**定义**：[backend-core/src/security/sessions.ts#L33-L74](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/security/sessions.ts#L33-L74)

```typescript
export async function invalidateSessions(userId, opts = {}) {
  const reason = opts?.reason || "unknown"
  let sessionIds = opts.sessionIds || []
  
  // 未指定 sessionIds 时失效所有会话
  if (sessionIds.length === 0) {
    const sessions = await getSessionsForUser(userId)
    sessionKeys = sessions.map(session => ({
      key: makeSessionID(session.userId, session.sessionId),
    }))
  }
  
  // 批量删除 Redis 中的会话
  const promises = sessionKeys.map(sk => client.delete(sk.key))
  await Promise.all(promises)
}
```

**密码重置时调用**：`sessions.invalidateSessions(userId)`（无 sessionIds 参数），即**失效该用户的所有会话**，强制所有设备重新登录。

---

## 七、完整登录流程时序图

```
Client  →  Nginx:10000  →  Worker:4002
  │           │                │
  │ POST /api/global/auth/:tenantId/login
  │ { username, password }
  │           │                │
  │───────────│───────────────→│
  │           │                │
  │           │            1. buildAuthMiddleware
  │           │               (标记 publicEndpoint = true)
  │           │                │
  │           │            2. buildTenancyMiddleware
  │           │               (设置 tenantId 上下文)
  │           │                │
  │           │            3. buildCsrfMiddleware
  │           │               (公开端点，跳过 CSRF)
  │           │                │
  │           │            4. pro.licensing()
  │           │               (无用户，跳过许可检查)
  │           │                │
  │           │            5. 自定义授权检查
  │           │               (publicEndpoint = true，放行)
  │           │                │
  │           │            6. Joi 校验
  │           │               (username + password 必填)
  │           │                │
  │           │            7. lockout 中间件
  │           │               (检查是否被锁定)
  │           │                │
  │           │            8. controller.login
  │           │               ├─ 检查 isPreventPasswordActions
  │           │               ├─ Passport Local 认证
  │           │               │   ├─ 查找用户
  │           │               │   ├─ 检查状态
  │           │               │   ├─ 检查密码存在
  │           │               │   └─ 密码比对
  │           │               ├─ 失败: onFailed (增加计数/锁定)
  │           │               ├─ 成功: clearFailureState
  │           │               ├─ authSdk.loginUser
  │           │               │   ├─ createASession (Redis)
  │           │               │   │  └─ MAX_SESSIONS_PER_USER 淘汰
  │           │               │   └─ 签发 JWT
  │           │               ├─ 设置 Cookie + Header
  │           │               └─ 触发登录事件
  │           │                │
  │←──────────│───────────────│
  │           │                │
```

---

## 八、关键安全机制总结

| 安全机制 | 描述 | 位置 |
|----------|------|------|
| **登录锁定** | 5 次失败后锁定 15 分钟 | [lockout.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/middleware/lockout.ts) + [auth.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/controllers/global/auth.ts#L56-L72) |
| **用户枚举防护** | 登录失败/重置请求不暴露用户存在性 | 多处 |
| **SSO 强制** | SSO 用户禁止密码操作 | [isPreventPasswordActions](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/users/db.ts#L89-L111) |
| **CSRF 防护** | 同步令牌模式，公开端点例外 | [csrf.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/middleware/csrf.ts) |
| **会话限制** | 每用户最多 3 个会话，FIFO 淘汰 | [sessions.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/security/sessions.ts#L76-L120) |
| **密码重置限流** | 邮箱+IP 双重速率限制 | [auth.ts#reset](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/controllers/global/auth.ts#L196-L260) |
| **重置后失效会话** | 密码重置后失效所有会话 | [auth.ts#resetUpdate](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/sdk/auth/auth.ts#L85-L98) |
| **HttpOnly Cookie** | 认证 Cookie 设为 HttpOnly | [passportCallback](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/api/controllers/global/auth.ts#L98-L101) |
| **JWT 认证** | Header/Cookie 双模式支持 | [loginUser](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/worker/src/sdk/auth/auth.ts#L18-L41) |
