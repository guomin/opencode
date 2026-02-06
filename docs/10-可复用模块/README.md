# OpenCode 可复用模块详解

本文档详细列出了 OpenCode 项目中可以独立复用到其他项目的模块，按**颗粒度从大到小**排序。

## 📋 目录

1. [超大颗粒度（独立 Package）](#超大颗粒度独立-package)
2. [大颗粒度（完整子系统）](#大颗粒度完整子系统)
3. [中等颗粒度（单个功能模块）](#中等颗粒度单个功能模块)
4. [小颗粒度（工具函数）](#小颗粒度工具函数)
5. [复用指南](#复用指南)

---

## 超大颗粒度（独立 Package）

### 1. @opencode-ai/util - 基础工具库 ⭐⭐⭐⭐⭐

**位置**：`packages/util/src/`

**依赖**：仅 `zod`

**包含模块**：

| 模块 | 文件 | 功能 | 复用难度 |
|------|------|------|----------|
| 错误处理 | `error.ts` | 类型安全的错误定义 | ⭐ |
| 唯一标识 | `identifier.ts` | ULID ID 生成器 | ⭐ |
| URL 标识 | `slug.ts` | URL 友好标识 | ⭐ |
| 函数包装 | `fn.ts` | Zod 验证 + 函数包装 | ⭐ |
| 数组操作 | `array.ts` | 数组工具函数 | ⭐ |
| 二进制 | `binary.ts` | 二进制数据处理 | ⭐ |
| 编码 | `encode.ts` | 各种编码转换 | ⭐ |
| 立即执行 | `iife.ts` | 立即执行函数模式 | ⭐ |
| 懒加载 | `lazy.ts` | 懒加载工具 | ⭐ |
| 重试 | `retry.ts` | 重试机制 | ⭐ |
| 路径 | `path.ts` | 路径处理工具 | ⭐ |

#### 使用示例

```typescript
// 1. 唯一标识生成
import { Identifier } from '@opencode-ai/util/identifier'

// 降序时间戳 ID（用于排序，新ID在前）
const sessionID = Identifier.descending("session")  // "01HZABC123..."
const messageID = Identifier.ascending("message")  // 升序
const ulid = Identifier.ulid()                     // 随机 ULID

// 2. URL 友好标识
import { Slug } from '@opencode-ai/util/slug'
const slug = Slug.create()  // "calm-oval-84"

// 3. 类型安全的错误
import { NamedError } from '@opencode-ai/util/error'

const UserNotFoundError = NamedError.create(
  "UserNotFoundError",
  z.object({
    userId: z.string(),
    message: z.string(),
  })
)

// 抛出错误
throw UserNotFoundError.create({
  userId: "123",
  message: "User not found"
})

// 捕获错误
if (UserNotFoundError.is(error)) {
  console.log(error.properties.userId)
}

// 4. 函数包装器（自动验证）
import { fn } from '@opencode-ai/util/fn'

const InputSchema = z.object({
  name: z.string(),
  age: z.number(),
})

const createUser = fn(InputSchema, async (input) => {
  // input 已经被验证过了
  return { id: "123", ...input }
})

// 使用
const user = await createUser({ name: "Alice", age: 30 })
// 如果传入无效数据，自动抛出验证错误

// 5. 重试机制
import { Retry } from '@opencode-ai/util/retry'

const result = await Retry.run({
  times: 3,
  delay: 1000,
  handler: async () => {
    return await fetchAPI()
  }
})
```

#### 复用方式

```bash
# 方式 1: 直接复制
cp -r packages/util/src/* my-project/src/util/

# 方式 2: 改名为独立包
# 创建 my-util-package，复制代码，发布到 npm
```

---

### 2. @opencode-ai/sdk - HTTP 客户端生成器 ⭐⭐⭐⭐

**位置**：`packages/sdk/js/src/v2/`

**功能**：
- 从 OpenAPI 规范自动生成 TypeScript SDK
- 类型安全的 API 客户端
- 事件流订阅（SSE）
- 认证支持

#### 生成 SDK

```bash
# 安装工具
npm install @hey-api/openapi-ts

# 生成 SDK
openapi-ts -i openapi.json -o src/sdk/

# 生成的结构
# src/sdk/
#   gen/
#     sdk.gen.ts      # 自动生成的客户端
#   client.ts         # 客户端封装
#   index.ts          # 导出
```

#### 使用示例

```typescript
import { createOpencodeClient } from '@opencode-ai/sdk'

// 创建客户端
const client = createOpencodeClient({
  baseUrl: 'http://localhost:6250',
  token: 'your-token'  // 可选
})

// 调用 API（类型安全）
const sessions = await client.session.list()
console.log(sessions.data)

// 发送消息
await client.session.prompt({
  sessionID: '123',
  parts: [{ type: 'text', text: '帮我写代码' }]
})

// 订阅事件流
const events = await client.event.subscribe()
for await (const event of events.stream) {
  console.log(event.type, event.properties)
}
```

#### 复用场景

- 任何需要从 OpenAPI 生成 SDK 的项目
- 需要 SSE 事件流的 API 客户端
- TypeScript 类型安全的 HTTP 客户端

---

## 大颗粒度（完整子系统）

### 3. 事件总线 (Event Bus) ⭐⭐⭐⭐⭐

**位置**：`packages/opencode/src/bus/`

**文件**：
- `index.ts` - 发布订阅核心
- `bus-event.ts` - 事件定义工具
- `global.ts` - 全局总线

**依赖**：
- `Log` (轻量日志)
- `Instance` (实例状态管理，可替换为全局变量)

**复用难度**：⭐ 非常独立

#### 核心代码

```typescript
// bus-event.ts - 事件定义工具
import { z } from "zod"

export namespace BusEvent {
  export interface Definition<T extends z.ZodTypeAny> {
    type: string
    properties: T
  }

  export function define<T extends z.ZodTypeAny>(
    type: string,
    properties: T
  ): Definition<T> {
    return { type, properties }
  }
}

// index.ts - 发布订阅核心
export namespace Bus {
  const subscriptions = new Map<any, Subscription[]>()

  // 发布事件
  export async function publish<Definition extends BusEvent.Definition>(
    def: Definition,
    properties: z.output<Definition["properties"]>
  ) {
    const payload = {
      type: def.type,
      properties,
    }

    // 通知所有订阅者
    const subs = subscriptions.get(def.type) || []
    for (const sub of subs) {
      sub(payload)
    }

    // 通知通配符订阅者
    const wildcard = subscriptions.get("*") || []
    for (const sub of wildcard) {
      sub(payload)
    }
  }

  // 订阅事件
  export function subscribe<Definition extends BusEvent.Definition>(
    def: Definition,
    callback: (event: {
      type: string
      properties: z.output<Definition["properties"]>
    }) => void
  ): () => void {
    if (!subscriptions.has(def.type)) {
      subscriptions.set(def.type, [])
    }
    subscriptions.get(def.type)!.push(callback)

    // 返回取消订阅函数
    return () => {
      const subs = subscriptions.get(def.type)
      if (subs) {
        const index = subs.indexOf(callback)
        if (index > -1) subs.splice(index, 1)
      }
    }
  }
}
```

#### 使用示例

```typescript
import { Bus } from './bus'
import { BusEvent } from './bus-event'
import { z } from 'zod'

// 1. 定义事件
export const Event = {
  UserCreated: BusEvent.define(
    "user.created",
    z.object({
      id: z.string(),
      name: z.string(),
      email: z.string(),
    })
  ),

  UserUpdated: BusEvent.define(
    "user.updated",
    z.object({
      id: z.string(),
      changes: z.record(z.any()),
    })
  ),

  SessionCompleted: BusEvent.define(
    "session.completed",
    z.object({
      sessionId: z.string(),
      duration: z.number(),
    })
  ),
}

// 2. 发布事件
async function createUser(name: string, email: string) {
  const user = await db.users.create({ name, email })

  // 发布事件
  await Bus.publish(Event.UserCreated, {
    id: user.id,
    name: user.name,
    email: user.email,
  })

  return user
}

// 3. 订阅事件
const unsub = Bus.subscribe(Event.UserCreated, (evt) => {
  console.log('新用户创建:', evt.properties.name)

  // 发送欢迎邮件
  sendWelcomeEmail(evt.properties.email)
})

// 4. 订阅所有事件（通配符）
Bus.subscribe({ type: '*', properties: z.any() }, (evt) => {
  console.log('事件日志:', evt.type)
})

// 5. 取消订阅
unsub()
```

#### 实际应用场景

**场景 1：微服务间通信**
```typescript
// 订单服务创建订单后，发布事件
Bus.publish(Event.OrderCreated, { orderId: "123", amount: 100 })

// 库存服务订阅事件，扣减库存
Bus.subscribe(Event.OrderCreated, async (evt) => {
  await inventory.deduct(evt.properties.orderId)
})

// 通知服务订阅事件，发送通知
Bus.subscribe(Event.OrderCreated, async (evt) => {
  await notification.send('订单已创建')
})
```

**场景 2：UI 状态管理**
```typescript
// 订阅状态变更，更新 UI
Bus.subscribe(Event.UserUpdated, (evt) => {
  updateUserInUI(evt.properties.id, evt.properties.changes)
})

Bus.subscribe(Event.SessionCompleted, (evt) => {
  showNotification('会话已完成')
})
```

**场景 3：日志和监控**
```typescript
// 订阅所有事件，记录日志
Bus.subscribe({ type: '*', properties: z.any() }, (evt) => {
  logger.info('Event', { type: evt.type, properties: evt.properties })
})

// 发送到监控系统
Bus.subscribe({ type: '*', properties: z.any() }, (evt) => {
  metrics.increment(`event.${evt.type}`)
})
```

#### 复用步骤

```bash
# 1. 复制代码
mkdir -p my-project/src/bus
cp packages/opencode/src/bus/*.ts my-project/src/bus/

# 2. 简化依赖（如果不需要实例状态管理）
# 修改 index.ts，删除 Instance 相关代码，使用全局 Map

# 3. 使用
import { Bus } from './bus'
```

---

### 4. 存储系统 (Storage) ⭐⭐⭐⭐

**位置**：`packages/opencode/src/storage/storage.ts`

**功能**：
- 基于文件的键值存储
- JSON 序列化
- 原子操作
- 目录层级支持
- 自动迁移

**依赖**：
- `Log` - 日志
- `Filesystem` - 文件系统工具
- `Lock` - 文件锁
- `Bun` - Bun 运行时

**复用难度**：⭐⭐ 低依赖

#### 核心代码

```typescript
export namespace Storage {
  const root = path.resolve(os.homedir(), ".myapp/storage")

  // 写入
  export async function write(keys: string[], value: any) {
    const filepath = path.resolve(root, ...keys) + ".json"
    await Bun.write(filepath, JSON.stringify(value, null, 2))
  }

  // 读取
  export async function read<T>(keys: string[]): Promise<T | undefined> {
    const filepath = path.resolve(root, ...keys) + ".json"
    const file = Bun.file(filepath)
    if (!(await file.exists())) return undefined
    return await file.json()
  }

  // 列出
  export async function list(keys: string[]): Promise<string[]> {
    const dir = path.resolve(root, ...keys)
    const files: string[] = []
    for await (const file of new Bun.Glob("*.json").scan({ cwd: dir })) {
      files.push(file.replace('.json', ''))
    }
    return files
  }

  // 删除
  export async function remove(keys: string[]) {
    const filepath = path.resolve(root, ...keys) + ".json"
    await fs.remove(filepath)
  }
}
```

#### 使用示例

```typescript
import { Storage } from './storage'

// 1. 写入数据
await Storage.write(["user", "123"], {
  id: "123",
  name: "Alice",
  email: "alice@example.com"
})
// 保存到: ~/.myapp/storage/user/123.json

// 2. 读取数据
const user = await Storage.read(["user", "123"])
console.log(user.name)  // "Alice"

// 3. 列出所有用户
const userIds = await Storage.list(["user"])
console.log(userIds)  // ["123", "456", "789"]

// 4. 嵌套结构
await Storage.write(["project", "proj-001", "session", "sess-001"], {
  id: "sess-001",
  title: "我的会话"
})
// 保存到: ~/.myapp/storage/project/proj-001/session/sess-001.json

// 5. 删除
await Storage.remove(["user", "123"])
```

#### 实际应用场景

**场景 1：配置管理**
```typescript
// 保存配置
await Storage.write(["config", "database"], {
  host: "localhost",
  port: 5432,
})

// 读取配置
const dbConfig = await Storage.read(["config", "database"])
```

**场景 2：缓存系统**
```typescript
// 设置缓存
await Storage.write(["cache", "user:123"], {
  data: userData,
  expire: Date.now() + 3600000
})

// 读取缓存
const cached = await Storage.read(["cache", "user:123"])
if (cached && cached.expire > Date.now()) {
  return cached.data
}
```

**场景 3：会话存储**
```typescript
// 保存会话
await Storage.write(["session", sessionId], {
  id: sessionId,
  userId: "123",
  createdAt: Date.now(),
  data: {}
})
```

#### 复用步骤

```bash
# 1. 复制文件
cp packages/opencode/src/storage/storage.ts my-project/src/storage.ts

# 2. 调整根目录
# 修改 const root = path.resolve(os.homedir(), ".myapp/storage")

# 3. 复制依赖
cp packages/opencode/src/util/filesystem.ts my-project/src/util/
cp packages/opencode/src/util/lock.ts my-project/src/util/
```

---

### 5. 权限系统 (Permission) ⭐⭐⭐⭐

**位置**：`packages/opencode/src/permission/`

**文件**：
- `next.ts` - 权限检查引擎
- `arity.ts` - 参数数量检查
- `index.ts` - 权限定义

**功能**：
- 基于规则的权限控制
- 通配符匹配
- allow/deny/ask 三种动作
- 运行时权限检查

**依赖**：
- `Bus` - 事件总线
- `Storage` - 存储
- `Wildcard` - 通配符匹配

**复用难度**：⭐⭐ 中等依赖

#### 核心代码

```typescript
import { z } from "zod"

export namespace PermissionNext {
  export const Action = z.enum(["allow", "deny", "ask"])
  export type Action = z.infer<typeof Action>

  export const Rule = z.object({
    permission: z.string(),    // 权限名称（如 "edit", "bash"）
    pattern: z.string(),       // 匹配模式（如 "*.ts", "*"）
    action: Action,            // 动作
  })
  export type Rule = z.infer<typeof Rule>

  export type Ruleset = Rule[]

  // 检查权限
  export function check(input: {
    permission: string
    pattern: string
    ruleset: Ruleset
  }): { action: Action; rule?: Rule } {
    const { permission, pattern, ruleset } = input

    // 查找匹配的规则
    const rules = ruleset.filter(rule =>
      rule.permission === permission &&
      Wildcard.match(rule.pattern, pattern)
    )

    // 如果没有规则，默认允许
    if (rules.length === 0) {
      return { action: "allow" }
    }

    // 最后一条规则生效
    const lastRule = rules[rules.length - 1]
    return { action: lastRule.action, rule: lastRule }
  }

  // 询问权限（发布事件）
  export async function ask(input: {
    sessionID: string
    permission: string
    patterns: string[]
  }): Promise<Action> {
    const request = {
      id: Identifier.ascending("permission"),
      sessionID: input.sessionID,
      permission: input.permission,
      patterns: input.patterns,
    }

    // 保存请求
    await Storage.write(["permission", request.id], request)

    // 发布事件
    Bus.publish(Event.Asked, { request })

    // 等待响应（通过事件）
    // ... 等待逻辑
  }
}
```

#### 使用示例

```typescript
import { PermissionNext } from './permission'

// 1. 定义规则集
const ruleset: PermissionNext.Ruleset = [
  // 允许编辑所有 .ts 文件
  { permission: "edit", action: "allow", pattern: "*.ts" },

  // 禁止编辑 .json 文件
  { permission: "edit", action: "deny", pattern: "*.json" },

  // 其他文件询问
  { permission: "edit", action: "ask", pattern: "*" },

  // 允许在 src 目录执行 bash
  { permission: "bash", action: "allow", pattern: "src/*" },

  // 其他目录询问
  { permission: "bash", action: "ask", pattern: "*" },

  // 禁止删除操作
  { permission: "delete", action: "deny", pattern: "*" },
]

// 2. 检查权限
const result = PermissionNext.check({
  permission: "edit",
  pattern: "test.ts",
  ruleset,
})

console.log(result.action)  // "allow"

// 3. 在代码中使用
async function editFile(filepath: string) {
  const result = PermissionNext.check({
    permission: "edit",
    pattern: filepath,
    ruleset,
  })

  switch (result.action) {
    case "allow":
      await writeFile(filepath, content)
      break
    case "deny":
      throw new Error(`禁止编辑: ${filepath}`)
    case "ask":
      const response = await PermissionNext.ask({
        sessionID: "123",
        permission: "edit",
        patterns: [filepath],
      })
      if (response === "allow") {
        await writeFile(filepath, content)
      }
      break
  }
}

// 4. 通配符匹配
Wildcard.match("*.ts", "test.ts")        // true
Wildcard.match("src/*", "src/app.ts")    // true
Wildcard.match("*.ts", "test.js")        // false
```

#### 实际应用场景

**场景 1：AI 工具调用权限**
```typescript
// 定义规则
const ruleset = [
  { permission: "edit", action: "allow", pattern: "*.ts" },
  { permission: "edit", action: "ask", pattern: "*" },
  { permission: "bash", action: "ask", pattern: "rm -rf *" },
  { permission: "bash", action: "allow", pattern: "*" },
]

// AI 尝试调用工具
if (tool.name === "edit") {
  const result = PermissionNext.check({
    permission: "edit",
    pattern: tool.input.filepath,
    ruleset,
  })

  if (result.action === "deny") {
    throw new Error("权限不足")
  }

  if (result.action === "ask") {
    // 向用户询问
  }
}
```

**场景 2：Web API 权限**
```typescript
// 检查用户权限
app.post('/api/files/edit', async (req, res) => {
  const filepath = req.body.filepath
  const userRuleset = req.user.permissions

  const result = PermissionNext.check({
    permission: "edit",
    pattern: filepath,
    ruleset: userRuleset,
  })

  if (result.action === "deny") {
    return res.status(403).json({ error: "禁止访问" })
  }

  // 执行编辑
})
```

#### 复用步骤

```bash
# 1. 复制权限模块
cp packages/opencode/src/permission/next.ts my-project/src/permission.ts

# 2. 复制通配符工具
cp packages/opencode/src/util/wildcard.ts my-project/src/util/

# 3. 使用
import { PermissionNext } from './permission'
```

---

### 6. MCP 集成 ⭐⭐⭐⭐⭐

**位置**：`packages/opencode/src/mcp/`

**功能**：
- MCP 协议客户端（stdio, HTTP, SSE）
- 工具发现和调用
- 资源访问
- OAuth 认证流程
- 热重载支持

**依赖**：`@modelcontextprotocol/sdk`

**复用难度**：⭐⭐ 中等（但非常完整）

#### 使用示例

```typescript
import { MCP } from './mcp'

// 1. 连接到 MCP 服务器
const client = await MCP.connect({
  name: "filesystem",
  command: "npx",
  args: ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed"]
})

// 2. 列出可用工具
const tools = await client.listTools()
console.log(tools)
// [
//   { name: "read_file", description: "Read a file" },
//   { name: "write_file", description: "Write to a file" },
//   { name: "list_directory", description: "List directory" }
// ]

// 3. 调用工具
const result = await client.callTool("read_file", {
  path: "/path/to/file.txt"
})

// 4. 访问资源
const resources = await client.listResources()
const resource = await client.readResource("file:///path/to/file.txt")
```

#### 适用场景

- 任何需要集成 MCP 服务器的项目
- AI 工具扩展系统
- 插件架构

---

### 7. Provider 抽象 (LLM Provider) ⭐⭐⭐⭐⭐

**位置**：`packages/opencode/src/provider/`

**功能**：
- 支持 20+ LLM Provider
- 统一的 API 调用接口
- 自动认证管理
- 模型元数据

#### 使用示例

```typescript
import { Provider } from './provider'

// 1. 获取 Provider
const provider = await Provider.getProvider("openai")

// 2. 获取模型
const model = await Provider.getModel("openai", "gpt-4")

// 3. 调用 LLM
import { streamText } from 'ai'

const result = await streamText({
  model: model.languageModel,
  messages: [{ role: 'user', content: '你好' }],
})
```

#### 适用场景

- 任何需要调用 LLM 的项目
- AI 应用开发框架

---

### 8. Agent 系统 ⭐⭐⭐⭐

**位置**：`packages/opencode/src/agent/`

**功能**：
- Agent 定义和管理
- System prompt 模板
- 权限规则
- 模型配置

#### 使用示例

```typescript
import { Agent } from './agent'

// 获取 Agent
const agent = await Agent.get('default')

console.log(agent.prompt)      // System prompt
console.log(agent.permission)  // 权限规则
console.log(agent.model)       // 使用的模型
```

---

## 中等颗粒度（单个功能模块）

### 9. ID 生成器 (Identifier) ⭐⭐⭐⭐⭐

**位置**：`packages/opencode/src/id/id.ts`

**依赖**：`ulid`

```typescript
import { Identifier } from './id'

// 降序时间戳 ID（用于排序，新ID在前）
Identifier.descending("user")  // "01HZABC123..."

// 升序时间戳 ID
Identifier.ascending("message")

// 随机 ULID
Identifier.ulid()

// 验证 ID
Identifier.schema("user").parse("01HZABC123...")
```

---

### 10. 命令系统 (Command) ⭐⭐⭐

**位置**：`packages/opencode/src/command/`

**功能**：
- 命令注册和发现
- 命令参数验证
- 命令执行

```typescript
import { Command } from './command'

// 定义命令
const myCommand = {
  name: "deploy",
  description: "Deploy application",
  handler: async (args) => {
    console.log("Deploying...")
  }
}

// 注册命令
Command.register(myCommand)
```

---

### 11. 问答系统 (Question) ⭐⭐⭐

**位置**：`packages/opencode/src/question/`

**功能**：
- 向用户提问
- 选择题/填空题
- 事件驱动的问答

```typescript
import { Question } from './question'

// 发问
const request = await Question.ask({
  sessionID: "123",
  message: "选择框架",
  options: [
    { value: "react", label: "React" },
    { value: "vue", label: "Vue" },
  ],
})

// 等待回答
const answer = await Question.wait(request.id)
console.log(answer)  // { response: "react" }
```

---

### 12. 文件快照 (Snapshot) ⭐⭐⭐⭐

**位置**：`packages/opencode/src/snapshot/`

**功能**：
- 文件变更检测
- 生成 git diff
- 快照比较

```typescript
import { Snapshot } from './snapshot'

// 创建快照
const snapshot1 = await Snapshot.create()

// ... 修改文件 ...

// 比较差异
const diff = await Snapshot.diff(snapshot1)
console.log(diff.files)  // 变更的文件列表
```

---

### 13. Shell 封装 ⭐⭐⭐

**位置**：`packages/opencode/src/shell/`

**功能**：
- Shell 命令执行
- 伪终端（PTY）支持
- 输出流式读取

---

### 14. 补丁系统 (Patch) ⭐⭐⭐

**位置**：`packages/opencode/src/patch/`

**功能**：
- 文件补丁生成
- 补丁应用
- Git 风格 diff

---

### 15. 文件操作 (File) ⭐⭐⭐⭐

**位置**：`packages/opencode/src/file/`

**包含**：
- `watcher.ts` - 文件监控
- `ignore.ts` - .gitignore 支持
- `ripgrep.ts` - 快速搜索
- `time.ts` - 文件时间戳

---

## 小颗粒度（工具函数）

### 16. Lock - 分布式锁

```typescript
import { Lock } from './util/lock'

await Lock.run("my-lock", async () => {
  // 同一时间只能有一个进程执行这里
})
```

### 17. Retry - 重试机制

```typescript
import { Retry } from './util/retry'

await Retry.run({
  times: 3,
  delay: 1000,
  handler: async () => {
    return await fetchAPI()
  }
})
```

### 18. Wildcard - 通配符匹配

```typescript
import { Wildcard } from './util/wildcard'

Wildcard.match("*.ts", "test.ts")        // true
Wildcard.match("src/*", "src/app.ts")    // true
Wildcard.match("**/*.ts", "src/test.ts") // true
```

---

## 复用指南

### 方法 1：直接复制代码

```bash
# 复制需要的模块
mkdir -p my-project/src/bus
cp packages/opencode/src/bus/*.ts my-project/src/bus/

# 复制依赖
mkdir -p my-project/src/util
cp packages/opencode/src/util/log.ts my-project/src/util/
```

### 方法 2：提取为独立包

```bash
# 1. 创建新包
mkdir my-event-bus
cd my-event-bus
npm init -y

# 2. 复制代码
mkdir src
cp ../opencode/packages/opencode/src/bus/*.ts src/

# 3. 调整依赖
# 修改 import 路径，删除不需要的依赖

# 4. 发布
npm publish
```

### 方法 3：Fork 项目

```bash
# Fork OpenCode，删除不需要的模块
git clone https://github.com/opencode-dev/opencode.git
cd opencode

# 保留需要的模块，删除其他
rm -rf packages/app packages/desktop packages/web ...
rm -rf packages/opencode/src/cli packages/opencode/src/server ...

# 修改 package.json
# 重新构建
bun install
bun run build
```

---

## 推荐复用优先级

### 最高优先级（立即可用）
1. ✅ **@opencode-ai/util** - 零依赖，极简
2. ✅ **事件总线** - 解耦合神器
3. ✅ **ID 生成器** - 唯一标识必备

### 高优先级（简单配置）
4. ✅ **存储系统** - 本地数据持久化
5. ✅ **权限系统** - 安全控制
6. ✅ **问答系统** - 交互式 CLI

### 中优先级（需要适配）
7. ⚠️ **Provider 抽象** - LLM 集成
8. ⚠️ **Agent 系统** - AI 应用
9. ⚠️ **MCP 集成** - 插件系统

---

## 总结

OpenCode 项目中有大量可复用的模块，从简单的工具函数到完整的子系统。推荐优先复用：

1. **@opencode-ai/util** - 基础工具库，零依赖
2. **事件总线** - 解耦合，通用性强
3. **存储系统** - 文件存储，简单易用
4. **权限系统** - 安全控制，规则灵活

这些模块都经过实战验证，代码质量高，可以直接复用到你的项目中！
