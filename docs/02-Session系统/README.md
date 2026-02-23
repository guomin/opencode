# Session系统详解

Session是OpenCode的核心概念，理解Session是理解整个项目的基础。

> **💡 进阶阅读**
> 
> 如果您想深入了解 OpenCode 是如何通过自研架构解决长文本截断、复杂任务拆解、流式 UI 渲染以及工具权限管控等痛点的，请阅读：
> 👉 [《会话与状态管理深度解析》](./会话与状态管理深度解析.md)

## 📋 目录

1. [Session是什么](#session是什么)
2. [Session的数据结构](#session的数据结构)
3. [Session的生命周期](#session的生命周期)
4. [父子Session关系](#父子session关系)
5. [Session存储机制](#session存储机制)

## Session是什么？

**定义**：Session是一个完整的对话会话，类似于ChatGPT的一个对话窗口。

**类比**：
- Session = 一个对话窗口
- Message = 对话中的一条消息
- Part = 消息的组成部分（文本、工具调用等）

**示例**：
```
Session: "实现用户认证功能"
├─ Message 1: User "帮我实现用户认证"
├─ Message 2: Assistant "我来规划一下任务"
│   └─ Part 1: TextPart "我来规划一下任务"
│   └─ Part 2: ToolPart (TodoWrite)
├─ Message 3: Assistant "开始执行任务1"
│   └─ Part 1: TextPart "开始执行..."
│   └─ Part 2: ToolPart (Task - 创建子Session)
└─ ...
```

## Session的数据结构

**代码位置**：`packages/opencode/src/session/index.ts`

```typescript
export const Info = z.object({
  id: Identifier.schema("session"),        // 唯一标识
  slug: z.string(),                         // URL友好标识
  projectID: z.string(),                    // 所属项目
  directory: z.string(),                    // 工作目录
  parentID: Identifier.schema("session").optional(),  // 父session ID
  title: z.string(),                         // 标题
  version: z.string(),                       // 版本号
  time: z.object({
    created: z.number(),                     // 创建时间
    updated: z.number(),                     // 更新时间
    compacting: z.number().optional(),       // 压缩时间
    archived: z.number().optional(),         // 归档时间
  }),
  permission: PermissionNext.Ruleset.optional(),  // 权限规则
  revert: z.object({...}).optional(),         // 撤销信息
  summary: z.object({...}).optional(),        // 摘要信息
  share: z.object({...}).optional(),          // 分享信息
})
```

**关键字段说明**：

| 字段 | 说明 | 示例 |
|------|------|------|
| `id` | 唯一标识，降序时间戳 | "01HZABC123..." |
| `parentID` | 父session ID，用于子任务 | undefined 或父session ID |
| `title` | 会话标题 | "实现用户认证功能" |
| `permission` | 权限规则 | [{ permission: "edit", action: "allow" }] |
| `time.created` | 创建时间戳 | 1704067200000 |

## Session的生命周期

### 1. 创建阶段

**触发时机**：
- 用户发起新对话
- AI创建子任务（Task工具）
- API直接调用

**代码位置**：`packages/opencode/src/session/index.ts:206-247`

```typescript
export async function createNext(input: {
  id?: string
  title?: string
  parentID?: string
  directory: string
  permission?: PermissionNext.Ruleset
}) {
  // 1. 生成ID和基础信息
  const result: Info = {
    id: Identifier.descending("session"),     // 降序时间戳ID
    slug: Slug.create(),                      // URL友好标识
    version: Installation.VERSION,
    projectID: Instance.project.id,
    directory: input.directory,
    parentID: input.parentID,                 // 父session（可选）
    title: input.title ?? createDefaultTitle(!!input.parentID),
    permission: input.permission,
    time: {
      created: Date.now(),
      updated: Date.now(),
    },
  }

  // 2. 保存到存储
  await Storage.write(["session", project.id, result.id], result)

  // 3. 发布事件
  Bus.publish(Event.Created, { info: result })

  // 4. 自动分享（如果配置允许）
  if (!result.parentID && cfg.share === "auto") {
    share(result.id)
  }

  return result
}
```

**实际操作**：
```bash
# CLI创建session
opencode run "测试"

# 等价于
Session.create({ title: "测试" })
```

### 2. 使用阶段

**活动**：
- 接收用户输入 → 创建User Message
- 调用LLM → 创建Assistant Message
- 执行工具 → 更新Message Part
- 更新状态 → `Session.touch()`

**代码示例**：
```typescript
// 接收用户输入
const message = await SessionPrompt.prompt({
  sessionID: "sess_001",
  parts: [{ type: "text", text: "帮我写代码" }]
})

// 内部流程：
// 1. 创建User Message
// 2. 创建Assistant Message
// 3. 调用LLM
// 4. 处理流式输出
// 5. 更新Session
```

### 3. 销毁阶段

**触发时机**：
- 用户主动删除
- 父session被删除（级联删除）
- 归档过期session

**代码位置**：`packages/opencode/src/session/index.ts:350-371`

```typescript
export const remove = fn(sessionID, async (sessionID) => {
  const session = await get(sessionID)

  // 1. 递归删除所有子session
  for (const child of await children(sessionID)) {
    await remove(child.id)  // 递归
  }

  // 2. 取消分享
  await unshare(sessionID).catch(() => {})

  // 3. 删除所有消息和部分
  for (const msg of await Storage.list(["message", sessionID])) {
    for (const part of await Storage.list(["part", msg.at(-1)!])) {
      await Storage.remove(part)
    }
    await Storage.remove(msg)
  }

  // 4. 删除session本身
  await Storage.remove(["session", project.id, sessionID])

  // 5. 发布事件
  Bus.publish(Event.Deleted, { info: session })
})
```

## 父子Session关系

### 为什么需要子Session？

**场景**：AI需要执行子任务

```
主Session: "实现用户认证"
│
├─ 子任务1: "设计数据库" → 子Session A
├─ 子任务2: "实现登录API" → 子Session B
└─ 子任务3: "编写测试" → 子Session C
```

**好处**：
1. **上下文隔离**：每个子任务独立，不污染主session
2. **权限控制**：子session权限更严格（禁止再创建子任务）
3. **清晰追踪**：可以看到每个子任务的执行情况

### 创建子Session

**代码位置**：`packages/opencode/src/tool/task.ts:68-97`

```typescript
return await Session.create({
  parentID: ctx.sessionID,  // ← 指向父session
  title: params.description + ` (@${agent.name} subagent)`,
  permission: [
    // 限制子session权限
    { permission: "todowrite", pattern: "*", action: "deny" },
    { permission: "todoread", pattern: "*", action: "deny" },
    { permission: "task", pattern: "*", action: "deny" },  // 禁止再创建子任务！
  ],
})
```

**权限对比**：

| 权限 | 主Session | 子Session | 说明 |
|------|-----------|----------|------|
| TodoWrite | ✅ 允许 | ❌ 禁止 | 避免Todo冲突 |
| Task | ✅ 允许 | ❌ 禁止 | 防止无限递归 |
| Edit | ✅ 允许 | ✅ 允许 | 需要编辑文件 |
| Bash | ✅ 允许 | ✅ 允许 | 需要执行命令 |

### UI显示

**Web UI**：
- 主session显示在主列表
- 子session默认隐藏，可以通过父session导航到子session

**CLI**：
```bash
# 只显示主session
$ opencode session list
sess_001  实现用户认证

# 查看子session
$ opencode session list --session sess_001
sess_002  设计数据库 (子)
sess_003  实现登录API (子)
sess_004  编写测试 (子)
```

## Session存储机制

### 存储路径

**根目录**：`~/.opencode/storage/`

**结构**：
```
~/.opencode/storage/
├── session/
│   └── {projectID}/
│       └── {sessionID}.json  ← Session基本信息
├── message/
│   └── {sessionID}/
│       ├── {messageID}.json  ← Message信息
│       └── ...
└── part/
    └── {messageID}/
        ├── {partID}.json  ← Part信息
        └── ...
```

### 实际示例

创建一个session后，可以查看：

```bash
# 查看session文件
cat ~/.opencode/storage/session/{projectID}/{sessionID}.json

{
  "id": "01HZABC123...",
  "slug": "abc123",
  "projectID": "proj_001",
  "directory": "/Users/owner/projects/opencode",
  "parentID": null,
  "title": "实现用户认证",
  "version": "0.0.1",
  "time": {
    "created": 1704067200000,
    "updated": 1704067260000
  },
  "permission": [...],
  "summary": {...}
}

# 查看message列表
ls ~/.opencode/storage/message/{sessionID}/

msg_001.json  ← User消息
msg_002.json  ← Assistant消息
msg_003.json  ← User消息
...

# 查看part列表
ls ~/.opencode/storage/part/{messageID}/

part_001.json  ← TextPart
part_002.json  ← ToolPart
part_003.json  ← ReasoningPart
...
```

## 实践练习

### 练习1：创建和查看Session

```bash
# 1. 创建一个session
opencode run "测试session"

# 2. 查看session存储
ls ~/.opencode/storage/session/

# 3. 查看具体内容
cat ~/.opencode/storage/session/{projectID}/{latest-session-id}.json

# 4. 查看messages
ls ~/.opencode/storage/message/{sessionID}/

# 5. 查看parts
ls ~/.opencode/storage/part/{messageID}/
```

### 练习2：观察父子Session

```bash
# 1. 创建主session
opencode run "帮我实现一个功能"

# 2. 在对话中，AI会创建子session
# 观察：父session的parentID为null
# 子session的parentID指向父session

# 3. 查看session列表
opencode session list

# 4. 查看子session
opencode session list --roots=false  # 显示所有session包括子session
```

### 练习3：追踪Session生命周期

```bash
# 1. 创建session
opencode run "测试"

# 2. 在另一个终端，监控storage目录
watch -n 1 'ls -la ~/.opencode/storage/session/*/'

# 3. 在第一个终端输入内容
# 观察storage目录的变化
```

## 总结

### 核心要点

1. **Session = 对话会话**：一个完整的对话窗口
2. **三层结构**：Session → Message → Part
3. **父子关系**：子session用于子任务，权限更严格
4. **文件存储**：JSON格式，按session/message/part分目录存储
5. **生命周期**：创建 → 使用（接收消息、调用LLM） → 销毁

### 下一步学习

- [03-LLM流式输出](../03-LLM流式输出/) - 理解AI输出如何被处理
- [04-事件系统](../04-事件系统/) - 理解Session如何与UI通信

### 相关代码文件

- `packages/opencode/src/session/index.ts` - Session核心逻辑
- `packages/opencode/src/session/message-v2.ts` - Message和Part定义
- `packages/opencode/src/session/prompt.ts` - 用户输入处理
