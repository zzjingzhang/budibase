# Budibase 路由系统追踪：从副作用 import 到 Koa 路由注册

本文以新增 `GET /api/queries/accessible` 与已有 `GET /api/queries/:queryId` 为例，完整追踪从 `packages/server/src/api/routes/index.ts` 的副作用式 import 开始，到 `endpointGroupList.listAllEndpoints`、`EndpointGroup.endpointList`、`Endpoint.apply` 以及 `standard.ts` 中的权限组如何共同决定最终 Koa 路由顺序和中间件顺序的完整链路。

---

## 一、入口：副作用式 import 机制

### 1.1 入口文件

路由的入口是 [routes/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/index.ts)。它的核心模式是：

```typescript
import { endpointGroupList } from "./endpointGroups"

// 副作用式 import —— 只导入不引用
import "./query"
import "./datasource"
import "./auth"
// ... 数十个路由文件

const endpoints = endpointGroupList.listAllEndpoints()
const appRoutes = new Router()
for (let endpoint of endpoints) {
  endpoint.apply(appRoutes)
}
```

**关键洞察**：这些 `import "./xxx"` 没有引入任何变量，纯粹是为了触发模块的副作用执行——每个路由文件在被 import 时，会自行调用 `builderRoutes.get(...)`、`readRoutes.post(...)` 等方法，将 endpoint 注册到共享的 `endpointGroupList` 中。

### 1.2 endpointGroupList 的来源

`endpointGroupList` 定义在 [endpointGroups/standard.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/endpointGroups/standard.ts) 中，是一个 `EndpointGroupList` 类的单例：

```typescript
export const endpointGroupList = new EndpointGroupList()

export const builderRoutes = endpointGroupList.group(
  authorized(permissions.BUILDER)
)
builderRoutes.lockMiddleware()

export const readRoutes = endpointGroupList.group(...)  // 在各路由文件中创建
```

`EndpointGroupList` 位于 [backend-core/src/Endpoint/EndpointGroupList.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/Endpoint/EndpointGroupList.ts)。

---

## 二、核心类与协作关系

### 2.1 三类核心对象

| 类 | 所在文件 | 职责 |
|---|---|---|
| `EndpointGroupList` | [EndpointGroupList.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/Endpoint/EndpointGroupList.ts) | 管理所有 EndpointGroup，提供 `group()` 创建组、`listAllEndpoints()` 汇总并排序所有 endpoint |
| `EndpointGroup` | [EndpointGroup.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/Endpoint/EndpointGroup.ts) | 代表一组共享中间件的路由，提供 `.get()/.post()` 等方法添加 endpoint，`endpointList()` 将组中间件注入每个 endpoint |
| `Endpoint` | [Endpoint.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/Endpoint/Endpoint.ts) | 代表单个路由端点，持有 method、url、controller、middlewares、outputMiddlewares，`apply()` 将自身注册到 Koa Router |

### 2.2 协作流程概览

```
副作用 import 各路由文件
        │
        ▼
  各文件创建 EndpointGroup（readRoutes / writeRoutes 等）
  并向 group 中添加 endpoint（.get()/.post() 等）
        │
        ▼
  endpointGroupList.listAllEndpoints()
   ├─ 遍历所有 group，调用 group.endpointList()
   │    └─ 将 group 的 middlewares / outputMiddlewares 注入每个 endpoint
   └─ 静态路径排前面，参数路径排后面
        │
        ▼
  遍历所有 endpoint，调用 endpoint.apply(router)
   └─ 按顺序组装 middlewares → controller → outputMiddlewares → complete
      并调用 router[method](url, ...middlewares, controller, ...outputs, complete)
```

---

## 三、权限组机制：builderRoutes 与 readRoutes

### 3.1 标准权限组（standard.ts）

在 [standard.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/endpointGroups/standard.ts) 中预定义了多个全局权限组：

```typescript
export const builderRoutes = endpointGroupList.group(
  authorized(permissions.BUILDER)
)
builderRoutes.lockMiddleware()   // 锁定，不可再追加组中间件

export const publicRoutes = endpointGroupList.group()
publicRoutes.lockMiddleware()
```

这些组创建后**立即调用 `lockMiddleware()`**，意味着它们的组级中间件是固定的，不允许后续追加。

### 3.2 路由级权限组（以 query.ts 为例）

在 [query.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/query.ts) 中，创建了更细粒度的权限组：

```typescript
const readRoutes = endpointGroupList.group(
  {
    middleware: authorized(PermissionType.QUERY, PermissionLevel.READ),
    first: false,
  },
  recaptcha
)

readRoutes.get(
  "/api/queries/:queryId",
  paramResource("queryId"),
  queryController.find
)
```

这里 `readRoutes` 组拥有两个组中间件：
- `authorized(QUERY, READ)`：权限检查，`first: false` 表示注入到 endpoint 中间件数组尾部
- `recaptcha`：验证码校验，默认 `first: true`（见 `addGroupMiddleware` 默认值），注入到 endpoint 中间件数组头部

### 3.3 组中间件注入时机

`EndpointGroup.endpointList()` [L87-L101](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/Endpoint/EndpointGroup.ts#L87-L101) 是注入的关键：

```typescript
endpointList(): Endpoint[] {
  if (this.applied) {
    throw new Error("Already applied to router")
  }
  this.applied = true
  for (const endpoint of this.endpoints) {
    this.middlewares.forEach(({ fn, first }) =>
      endpoint.addMiddleware(fn, { first })
    )
    this.outputMiddlewares.forEach(({ fn, first }) =>
      endpoint.addOutputMiddleware(fn, { first })
    )
  }
  return this.endpoints
}
```

**执行逻辑**：
- 遍历组内所有 endpoint
- 对每个 endpoint，按 `first` 标志将组中间件插入 endpoint 的 `middlewares` 数组
  - `first: true` → `unshift`（插到最前）
  - `first: false` → `push`（插到最后）
- outputMiddlewares 同理注入到 `outputMiddlewares` 数组

### 3.4 单条 endpoint 的中间件顺序示例

以 `readRoutes.get("/api/queries/:queryId", paramResource("queryId"), queryController.find)` 为例，最终中间件顺序为：

| 顺序 | 中间件 | 来源 | 注入方式 |
|---|---|---|---|
| 1 | recaptcha | readRoutes 组 | first: true → unshift |
| 2 | paramResource("queryId") | endpoint 自身 | addEndpoint 时 push |
| 3 | authorized(QUERY, READ) | readRoutes 组 | first: false → push |
| 4 | queryController.find | endpoint 自身 | controller |

> **注意**：组中间件的 `first` 选项是相对于 endpoint 已有中间件的位置，而非组内中间件的顺序。组内中间件按添加顺序依次应用到 endpoint。

---

## 四、路由排序：静态路径为什么必须排在参数路径前面

### 4.1 排序逻辑

`EndpointGroupList.listAllEndpoints()` [L23-L41](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/Endpoint/EndpointGroupList.ts#L23-L41) 实现了排序：

```typescript
listAllEndpoints() {
  const endpoints = this.groups.flatMap(group => group.endpointList())
  const staticEndpoints = []
  const parameterizedEndpoints = []

  for (const endpoint of endpoints) {
    if (endpoint.url.includes(":")) {
      parameterizedEndpoints.push(endpoint)
    } else {
      staticEndpoints.push(endpoint)
    }
  }

  return [...staticEndpoints, ...parameterizedEndpoints]
}
```

**核心逻辑**：
- 将所有 endpoint 分为两类：静态路径（URL 不含 `:`）和参数路径（URL 含 `:`）
- 静态路径全部排在参数路径前面
- 同类内部保持原顺序（按 group 创建顺序 + 组内添加顺序）

### 4.2 为什么必须这样排序

Koa Router（`@koa/router`）按**注册顺序**匹配路由——第一个匹配到的路由会被命中，后续路由不再尝试。

以 `GET /api/queries/accessible` 为例：

**如果参数路径在前（错误情况）**：
```
注册顺序：
1. GET /api/queries/:queryId
2. GET /api/queries/accessible

请求 GET /api/queries/accessible 时：
  → 匹配第1条：queryId = "accessible" ✓ 命中
  → 第2条永远不会被执行
```

**如果静态路径在前（正确情况）**：
```
注册顺序：
1. GET /api/queries/accessible
2. GET /api/queries/:queryId

请求 GET /api/queries/accessible 时：
  → 匹配第1条：精确匹配 ✓ 命中

请求 GET /api/queries/someId 时：
  → 第1条不匹配
  → 匹配第2条：queryId = "someId" ✓ 命中
```

代码注释中也明确说明了这一点：
> endpoints /api/queries/:queryId and /api/queries/accessible can overlap, if the parameter comes before the accessible it'll be unreachable

### 4.3 新增 GET /api/queries/accessible 的正确做法

假设要将 `GET /api/queries/accessible` 添加到 `readRoutes` 组中：

```typescript
readRoutes.get("/api/queries/accessible", queryController.fetchAccessible)
readRoutes.get("/api/queries/:queryId", paramResource("queryId"), queryController.find)
```

即使代码中 `accessible` 写在 `:queryId` 前面（或后面），经过 `listAllEndpoints()` 的全局排序后，`/api/queries/accessible`（静态）一定会排在 `/api/queries/:queryId`（参数化）前面，保证可访问性。

---

## 五、lockMiddleware 与 group middleware 的追加

### 5.1 lockMiddleware 的作用

`EndpointGroup.lockMiddleware()` [L48-L50](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/Endpoint/EndpointGroup.ts#L48-L50)：

```typescript
lockMiddleware() {
  this.locked = true
}
```

`addGroupMiddleware()` 和 `addGroupMiddlewareOutput()` 均会检查 `this.locked`：

```typescript
addGroupMiddleware(middleware, opts = { first: true }) {
  if (this.locked) {
    throw new Error("Group locked, no more middleware can be added.")
  }
  this.middlewares.push({ fn: middleware, first: opts.first })
  return this
}
```

### 5.2 锁定后还能追加 group middleware 吗？

**不能。** 调用 `lockMiddleware()` 后，再调用 `addGroupMiddleware()` 或 `addGroupMiddlewareOutput()` 会直接抛出错误。

### 5.3 锁定后还能做什么？

锁定的是**组级中间件**，不是 endpoint 的添加。锁定后仍然可以：

- ✅ 通过 `.get() / .post() / .put()` 等添加新的 endpoint
- ✅ 每个新 endpoint 可以携带自己的 per-endpoint 中间件（在 controller 之前的参数）

例如 `builderRoutes` 在 `standard.ts` 中已锁定，但 `query.ts` 中仍然可以：

```typescript
builderRoutes
  .get("/api/queries", queryController.fetchQueries)
  .post("/api/queries", bodySubResource("datasourceId", "_id"), generateQueryValidation(), queryController.save)
```

这些 endpoint 会继承 `builderRoutes` 已锁定的组中间件（`authorized(permissions.BUILDER)`），同时可以添加自己特有的中间件（如 `bodySubResource`、`generateQueryValidation`）。

### 5.4 为什么要锁定？

锁定机制用于保证基础权限组的**安全性和一致性**，防止后续代码意外地向核心权限组（如 `builderRoutes`、`publicRoutes`）中插入中间件，避免引入安全漏洞或不可预期的行为。

---

## 六、Endpoint.apply 中的执行顺序

### 6.1 apply 方法实现

`Endpoint.apply()` [L53-L71](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/backend-core/src/Endpoint/Endpoint.ts#L53-L71)：

```typescript
apply(router: Router) {
  const method = this.method,
    url = this.url
  const middlewares = this.middlewares,
    controller = this.controller,
    outputMiddlewares = this.outputMiddlewares
  const complete = () => {}
  const params = [
    url,
    ...middlewares,
    controller,
    ...outputMiddlewares,
    complete,
  ]
  router[method](...params)
}
```

### 6.2 参数组装顺序

传递给 Koa Router 的参数顺序为：

```
[url] → [middlewares...] → [controller] → [outputMiddlewares...] → [complete]
```

以 `readRoutes.get("/api/queries/:queryId", paramResource("queryId"), queryController.find)` 为例，实际调用为：

```typescript
router.get(
  "/api/queries/:queryId",
  recaptcha,                     // middleware 1（组中间件，first:true）
  paramResource("queryId"),      // middleware 2（endpoint 中间件）
  authorized(QUERY, READ),       // middleware 3（组中间件，first:false）
  queryController.find,          // controller
  /* outputMiddlewares... */,    // 输出中间件
  complete                       // 空函数，终止链
)
```

### 6.3 controller 的位置

**controller 位于 middlewares 之后、outputMiddlewares 之前**，是整个中间件链的核心处理单元。

- `middlewares`：前置中间件，负责认证、权限校验、参数解析、验证等**进入控制器之前**的工作
- `controller`：业务逻辑的实际执行者
- `outputMiddlewares`：后置中间件，在 controller 之后执行，用于响应处理、日志记录等**控制器之后**的工作
- `complete`：一个空函数 `() => {}`，作为中间件链的末端，防止 `next()` 无限循环（代码注释："middlewares are circular so if they always keep calling next, it'll just keep looping"）

### 6.4 Koa 中间件的洋葱模型与执行顺序

在 Koa 的洋葱模型中，中间件执行分为"上行"（调用 `next()` 之前）和"下行"（`next()` 返回之后）两个阶段：

```
请求 →  middleware1 上行 →  middleware2 上行 →  controller  →  output1 上行 →  output2 上行 →  complete
                                                                          │
           middleware1 下行 ←  middleware2 下行  ←  返回  ←  output1 下行 ←  output2 下行  ←  返回
```

- `middlewares` 的上行阶段执行前置逻辑（鉴权、验证等）
- `controller` 执行业务逻辑
- `outputMiddlewares` 的上行阶段在 controller 之后执行
- 所有中间件的下行阶段按逆序执行

> **注意**：outputMiddlewares 被放置在 controller 之后，这意味着它们只有在 controller 内部调用了 `await next()` 时才会执行到。如果 controller 不调用 `next()`，链会在 controller 处终止，outputMiddlewares 不会被执行到。

---

## 七、完整流程总结

以新增 `GET /api/queries/accessible` 为例，从 import 到 Koa 注册的完整链路：

### 步骤 1：模块加载

- 导入 [routes/index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/index.ts)
  - 导入 `endpointGroupList`（触发 `standard.ts` 执行，创建 builderRoutes 等并锁定）
  - 副作用导入 `./query` 等路由文件

### 步骤 2：路由文件注册

- [query.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/routes/query.ts) 执行
  - 创建 `readRoutes` 组（含 authorized + recaptcha 两个组中间件）
  - 创建 `writeRoutes` 组
  - 向 `builderRoutes` 添加多个 endpoint
  - 向 `readRoutes` 添加 `GET /api/queries/accessible` 和 `GET /api/queries/:queryId`
  - 向 `writeRoutes` 添加多个 endpoint

### 步骤 3：汇总与排序

- `endpointGroupList.listAllEndpoints()` 调用
  - 按 group 创建顺序遍历各组
  - 每个 group 调用 `endpointList()`，将组中间件注入各自的 endpoint
  - 全局排序：静态路径在前，参数路径在后
  - `/api/queries/accessible`（静态）排在 `/api/queries/:queryId`（参数化）之前

### 步骤 4：注册到 Koa

- 遍历排序后的 endpoint 列表
- 每个 endpoint 调用 `apply(router)`
- 按 `url → middlewares → controller → outputMiddlewares → complete` 的顺序调用 `router[method](...)`
- Koa Router 按注册顺序保存路由，匹配时从上到下依次尝试

### 最终结果

新增的 `GET /api/queries/accessible` 能够被正确匹配，不会被 `GET /api/queries/:queryId` 遮蔽；两者各自享有 readRoutes 组的权限中间件和 recaptcha 保护。
