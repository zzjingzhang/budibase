# Budibase Automation Action 系统分析

## 概述

Budibase 的自动化（Automation）系统支持多种类型的 action 步骤，包括内置步骤、AI 步骤、自托管 Bash 步骤以及插件步骤。这些步骤通过统一的定义与实现分离机制进行管理，核心文件为 [actions.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/actions.ts)。

## 一、ACTION_IMPLS 与 BUILTIN_ACTION_DEFINITIONS 的 stepId 映射

### 1.1 双轨制设计

系统采用**定义（Definition）**与**实现（Implementation）**分离的双轨制设计，两者通过相同的 `stepId`（`AutomationActionStepId` 枚举值）进行映射：

| 对象 | 类型 | 来源 | 作用 |
|------|------|------|------|
| `BUILTIN_ACTION_DEFINITIONS` | `Record<string, AutomationStepDefinition>` | `@budibase/shared-core` 的 `automations.steps.*.definition` | 描述步骤的元信息、输入输出 schema、图标等 UI 展示信息 |
| `ACTION_IMPLS` | `ActionImplType` | `./steps/*.ts` 模块的 `run` 函数 | 步骤的实际运行逻辑 |

### 1.2 BUILTIN_ACTION_DEFINITIONS 定义

[actions.ts:86-122](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/actions.ts#L86-L122)

```typescript
export const BUILTIN_ACTION_DEFINITIONS: Record<string, AutomationStepDefinition> = {
  SEND_EMAIL_SMTP: automations.steps.sendSmtpEmail.definition,
  CREATE_ROW: automations.steps.createRow.definition,
  GET_ROW: automations.steps.getRow.definition,
  UPDATE_ROW: automations.steps.updateRow.definition,
  DELETE_ROW: automations.steps.deleteRow.definition,
  QUERY_ROWS: automations.steps.queryRows.definition,
  // ... 更多步骤
  EXECUTE_QUERY: automations.steps.executeQuery.definition,
  // AI 步骤
  CLASSIFY_CONTENT: automations.steps.classifyText.definition,
  PROMPT_LLM: automations.steps.promptLLM.definition,
  TRANSLATE: automations.steps.translate.definition,
  SUMMARISE: automations.steps.summarise.definition,
  GENERATE_TEXT: automations.steps.generate.definition,
  EXTRACT_FILE_DATA: automations.steps.extract.definition,
  // 逻辑步骤
  LOOP: automations.steps.loop.definition,
  BRANCH: automations.steps.branch.definition,
  COLLECT: automations.steps.collect.definition,
  // 向后兼容的小写 stepId
  discord: automations.steps.discord.definition,
  slack: automations.steps.slack.definition,
  // ...
}
```

定义文件统一存放在 [shared-core/src/automations/steps/](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/shared-core/src/automations/steps) 目录下，通过 [index.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/shared-core/src/automations/steps/index.ts) 聚合导出。

### 1.3 ACTION_IMPLS 实现

[actions.ts:52-84](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/actions.ts#L52-L84)

```typescript
const ACTION_IMPLS: ActionImplType = {
  SEND_EMAIL_SMTP: sendSmtpEmail.run,
  CREATE_ROW: createRow.run,
  GET_ROW: getRow.run,
  UPDATE_ROW: updateRow.run,
  DELETE_ROW: deleteRow.run,
  QUERY_ROWS: queryRow.run,
  EXECUTE_QUERY: executeQuery.run,
  // AI 步骤
  CLASSIFY_CONTENT: classifyText.run,
  PROMPT_LLM: promptLLM.run,
  TRANSLATE: translate.run,
  SUMMARISE: summarise.run,
  GENERATE_TEXT: generate.run,
  EXTRACT_FILE_DATA: extract.run,
  AGENT: agent.run,
  // 向后兼容
  discord: discord.run,
  slack: slack.run,
  // ...
}
```

实现文件存放在 [server/src/automations/steps/](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/steps) 目录下。

### 1.4 类型安全的 ActionImplType

`ACTION_IMPLS` 的类型 `ActionImplType` 通过条件类型根据 `SELF_HOSTED` 环境变量决定是否包含 `EXECUTE_BASH`：

[actions.ts:48-50](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/actions.ts#L48-L50)

```typescript
type ActionImplType = ActionImplementations<
  typeof env.SELF_HOSTED extends "true" ? Hosting.SELF : Hosting.CLOUD
>
```

底层类型定义在 [schema.ts:88-216](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/types/src/documents/workspace/automation/schema.ts#L88-L216)：

```typescript
export type ActionImplementations<T extends Hosting> = {
  [AutomationActionStepId.COLLECT]: ActionImplementation<...>,
  // ... 所有通用步骤
} & (T extends "self"
  ? {
      [AutomationActionStepId.EXECUTE_BASH]: ActionImplementation<BashStepInputs, BashStepOutputs>
    }
  : {})
```

## 二、SELF_HOSTED 分支与 EXECUTE_BASH

### 2.1 动态添加 EXECUTE_BASH

Bash 执行步骤仅在自托管环境下可用，通过运行时条件判断动态注入：

[actions.ts:124-136](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/actions.ts#L124-L136)

```typescript
// don't add the bash script/definitions unless in self host
// the fact this isn't included in any definitions means it cannot be ran at all
if (env.SELF_HOSTED) {
  // @ts-expect-error
  ACTION_IMPLS["EXECUTE_BASH"] = bash.run
  BUILTIN_ACTION_DEFINITIONS["EXECUTE_BASH"] = automations.steps.bash.definition

  if (env.isTest()) {
    BUILTIN_ACTION_DEFINITIONS["OPENAI"] = automations.steps.openai.definition
    BUILTIN_ACTION_DEFINITIONS["AGENT"] = automations.steps.agent.definition
  }
}
```

**关键要点：**
- 仅当 `env.SELF_HOSTED` 为 truthy 时，才将 `EXECUTE_BASH` 同时加入 `ACTION_IMPLS` 和 `BUILTIN_ACTION_DEFINITIONS`
- 由于 TypeScript 类型层面 `EXECUTE_BASH` 可能不存在，使用 `@ts-expect-error` 忽略类型错误
- 在测试环境（`env.isTest()`）下，额外将 `OPENAI` 和 `AGENT` 的定义加入 `BUILTIN_ACTION_DEFINITIONS`

### 2.2 EXECUTE_BASH 的定义结构

Bash 步骤的定义位于 [bash.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/shared-core/src/automations/steps/bash.ts)，包含：
- 输入：`command`（命令）和 `args`（JSON 数组参数）
- 输出：`stdout`（标准输出）和 `success`（是否成功）
- `internal: true` 标记为内部步骤
- 支持循环特性（`AutomationFeature.LOOPING`）

## 三、getActionDefinitions 函数详解

### 3.1 函数整体结构

[actions.ts:138-166](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/actions.ts#L138-L166)

```typescript
export async function getActionDefinitions(): Promise<
  Record<keyof typeof AutomationActionStepId, AutomationStepDefinition>
> {
  const actionDefinitions = { ...BUILTIN_ACTION_DEFINITIONS }

  if (env.SELF_HOSTED) {
    actionDefinitions["OPENAI"] = {
      ...automations.steps.openai.definition,
      deprecated: true,
    }
  }

  actionDefinitions["AGENT"] = automations.steps.agent.definition

  if (env.SELF_HOSTED) {
    const plugins = await sdk.plugins.fetch(PluginType.AUTOMATION)
    for (let plugin of plugins) {
      const schema = plugin.schema.schema as AutomationStepDefinition
      actionDefinitions[schema.stepId] = {
        ...schema,
        custom: true,
      }
    }
  }
  return actionDefinitions
}
```

### 3.2 OPENAI 的 deprecated 标记

在自托管环境下，`OPENAI` 步骤被显式标记为已废弃（`deprecated: true`）。这是因为 Budibase 已经用新的 AI 步骤（`CLASSIFY_CONTENT`、`PROMPT_LLM`、`TRANSLATE`、`SUMMARISE`、`GENERATE_TEXT`、`EXTRACT_FILE_DATA`）替代了旧的 `OPENAI` 步骤。

**注意：** `OPENAI` 的实现（`ACTION_IMPLS` 中的 `openai.run`）在初始化时就已存在，只是通过 `getActionDefinitions` 在自托管环境中将其 UI 定义标记为 deprecated。

### 3.3 AGENT 步骤的添加

`AGENT` 步骤被无条件添加到定义中（第 153 行），说明 AGENT 步骤在所有环境下都可见。

### 3.4 自托管插件的 custom 定义

在自托管环境下，函数会：
1. 通过 `sdk.plugins.fetch(PluginType.AUTOMATION)` 获取所有自动化类型的插件
2. 遍历每个插件，读取 `plugin.schema.schema` 作为步骤定义
3. 添加 `custom: true` 标记，标识这是自定义插件步骤
4. 以插件 schema 中的 `stepId` 为 key 存入 `actionDefinitions`

插件获取实现位于 [sdk/plugins/index.ts:16-30](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/sdk/plugins/index.ts#L16-L30)：

```typescript
export async function fetch(type?: PluginType): Promise<Plugin[]> {
  const db = tenancy.getGlobalDB()
  const response = await db.allDocs(
    dbCore.getPluginParams(null, { include_docs: true })
  )
  let plugins = response.rows.map((row: any) => row.doc) as Plugin[]
  plugins = await objectStore.enrichPluginURLs(plugins)
  if (type) {
    return plugins.filter((plugin: Plugin) => plugin.schema?.type === type)
  }
  return plugins
}
```

## 四、getAction 函数与插件加载机制

### 4.1 函数实现

[actions.ts:169-189](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/actions.ts#L169-L189)

```typescript
export async function getAction<TStep extends AutomationActionStepId, ...>(
  stepId: TStep
): Promise<ActionImplementation<TInputs, TOutputs> | undefined> {
  // 1. 优先查找内置实现
  if (ACTION_IMPLS[stepId as keyof ActionImplType] != null) {
    return ACTION_IMPLS[stepId as keyof ActionImplType] as unknown as ActionImplementation<TInputs, TOutputs>
  }

  // 2. 内置未命中时，尝试加载插件实现
  if (env.SELF_HOSTED) {
    const plugins = await sdk.plugins.fetch(PluginType.AUTOMATION)
    const found = plugins.find(plugin => plugin.schema.schema.stepId === stepId)
    if (!found) {
      throw new Error(`Unable to find action implementation for "${stepId}"`)
    }
    return (await getAutomationPlugin(found)).action
  }
}
```

### 4.2 加载流程

```
stepId → 检查 ACTION_IMPLS → 命中 → 返回内置实现
           ↓ 未命中
        检查是否 SELF_HOSTED → 否 → 返回 undefined
           ↓ 是
        sdk.plugins.fetch(PluginType.AUTOMATION)
           ↓
        按 stepId 查找匹配插件
           ↓ 找到
        getAutomationPlugin(plugin) → 返回 .action
           ↓ 未找到
        throw Error
```

### 4.3 getAutomationPlugin 插件加载器

[utilities/fileSystem/plugin.ts:81-83](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/utilities/fileSystem/plugin.ts#L81-L83)

```typescript
export const getAutomationPlugin = async (plugin: Plugin) => {
  return getPluginImpl(AUTOMATION_PATH, plugin)
}
```

底层 `getPluginImpl` 实现（[plugin.ts:43-75](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/utilities/fileSystem/plugin.ts#L43-L75)）的工作原理：
1. 检查本地临时目录中是否已存在该插件文件
2. 比较 hash 值，若相同则直接 `require` 本地文件
3. 若 hash 不同或文件不存在，从对象存储（MinIO）下载插件 JS
4. 写入本地文件并保存 hash 元数据
5. 通过 `require` 加载插件模块

这是一个带缓存的懒加载机制，避免重复下载和解析插件。

## 五、executeQuery Step 深度分析

### 5.1 整体架构

`executeQuery` 步骤采用**上下文模拟 + 控制器复用**的设计模式，通过构建一个模拟的 Koa `ctx` 对象，直接调用 query 控制器的方法，从而复用已有的查询执行逻辑。

入口文件：[executeQuery.ts](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/steps/executeQuery.ts)

### 5.2 run 函数实现

[executeQuery.ts:10-58](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/steps/executeQuery.ts#L10-L58)

```typescript
export async function run({
  inputs,
  appId,
  emitter,
  context,
}: {
  inputs: ExecuteQueryStepInputs
  appId: string
  emitter: ContextEmitter
  context: Record<string, any>
}): Promise<ExecuteQueryStepOutputs> {
  if (inputs.query == null) {
    return { success: false, response: { message: "Invalid inputs" } }
  }

  const { queryId, ...rest } = inputs.query

  const ctx: any = buildCtx(appId, emitter, {
    body: { parameters: rest },
    params: { queryId },
    user: context.user,
  })

  try {
    await queryController.executeV2AsAutomation(ctx)
    const { data, ...rest } = ctx.body

    return {
      response: data,
      info: rest,
      success: true,
    }
  } catch (err) {
    return {
      success: false,
      info: {},
      response: automationUtils.getError(err),
    }
  }
}
```

### 5.3 buildCtx 上下文构建

[utils.ts:41-64](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/automations/steps/utils.ts#L41-L64)

```typescript
export function buildCtx(appId: string, emitter?: ContextEmitter | null, opts: any = {}) {
  const ctx: any = {
    appId,
    user: opts.user || { appId },
    eventEmitter: emitter,
    throw: (_code: string, error: any) => {
      throw error
    },
  }
  if (opts.body) {
    ctx.request = { body: opts.body }
  }
  if (opts.params) {
    ctx.params = opts.params
  }
  if (opts.version) {
    ctx.version = opts.version
  }
  return ctx
}
```

**buildCtx 的作用：**
- 构造一个最小化的 Koa `ctx` 上下文对象，满足控制器方法的参数要求
- 注入 `appId`、`user`、`eventEmitter` 等必要属性
- 提供 `throw` 方法的兼容实现（直接抛出错误，而非 HTTP 状态码）
- 支持可选的 `request.body`、`params`、`version` 等属性

### 5.4 executeV2AsAutomation 复用控制器

[query/index.ts:494-498](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/query/index.ts#L494-L498)

```typescript
export async function executeV2AsAutomation(
  ctx: UserCtx<ExecuteQueryRequest, ExecuteV2QueryResponse>
) {
  return execute(ctx, { rowsOnly: false, isAutomation: true })
}
```

它调用内部的 `execute` 函数，并传入 `isAutomation: true` 选项。

### 5.5 绕过 CRUD Quota 的关键逻辑

[query/index.ts:415-480](file:///Users/zhangjing/Desktop/so-coders/0604-under/budibase/packages/server/src/api/controllers/query/index.ts#L415-L480)

```typescript
async function execute(ctx, opts = { rowsOnly: false, isAutomation: false }) {
  // ... 获取 query 和 datasource

  try {
    const inputs: QueryEvent = { /* ... */ }

    const { rows, pagination, extra, info } =
      query.queryVerb === "read" || opts.isAutomation
        ? await Runner.run<QueryResponse>(inputs)
        : await quotas.addAction(ActionType.CRUD, async () => {
            const response = await Runner.run<QueryResponse>(inputs)
            events.action.crudExecuted({ type: query.queryVerb })
            return response
          })

    // ... 处理返回结果
  } catch (err) {
    // ... 错误处理
  }
}
```

**关键判断条件（第 457 行）：**

```typescript
query.queryVerb === "read" || opts.isAutomation
```

- **普通 API 调用**（`isAutomation: false`）：
  - `read` 操作：直接执行，不经过 quota
  - 非 `read` 操作（create/update/delete）：通过 `quotas.addAction(ActionType.CRUD, ...)` 包裹，计入 CRUD 配额

- **自动化调用**（`isAutomation: true`）：
  - **所有操作**都直接执行，**不经过 quota 检查**，也不触发 `crudExecuted` 事件
  - 这意味着自动化中的查询执行不会消耗用户的 CRUD 操作配额

**设计意图：** 自动化是 Budibase 的核心功能之一，如果每次自动化执行查询都消耗配额，会严重限制自动化的使用场景。将自动化执行排除在配额之外，是产品设计上的策略选择。

### 5.6 数据流图

```
自动化步骤 executeQuery.run()
        ↓
   buildCtx() 构造模拟 ctx
        ↓
queryController.executeV2AsAutomation(ctx)
        ↓
   execute(ctx, { isAutomation: true })
        ↓
  ┌──────────────────────────┐
  │ queryVerb === "read" ||  │
  │       isAutomation       │
  └───────────┬──────────────┘
       true  │   false
             ↓
   Runner.run()  ←── 直接执行，跳过 quota
             ↑
             │ （自动化路径始终走此分支）
             ↓
   ctx.body = { data, pagination, ... }
        ↓
自动化步骤读取 ctx.body 并返回
```

## 六、总结

Budibase 的 automation action 系统采用了分层、可扩展的架构设计：

1. **定义与实现分离**：`BUILTIN_ACTION_DEFINITIONS` 负责 UI/schema 定义，`ACTION_IMPLS` 负责运行时逻辑，通过 `stepId` 映射关联

2. **环境差异化控制**：通过 `SELF_HOSTED` 环境变量控制 `EXECUTE_BASH` 的可用性，并在测试环境中额外开放 `OPENAI` 和 `AGENT` 定义

3. **插件扩展机制**：自托管环境支持通过插件自定义自动化步骤，`getActionDefinitions` 负责加载插件 schema，`getAction` 负责动态加载插件实现代码

4. **控制器复用模式**：以 `executeQuery` 为代表的步骤通过 `buildCtx` 模拟 Koa 上下文，复用 API 控制器逻辑，并利用 `isAutomation` 标志绕过 CRUD 配额限制

5. **版本演进兼容**：保留了旧版 `OPENAI` 步骤但标记为 deprecated，同时保留小写 stepId（discord、slack 等）以实现向后兼容
