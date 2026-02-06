# 事件系统详解

事件系统是OpenCode组件间通信的核心机制，理解它对理解整个项目架构至关重要。

## 📋 目录

1. [什么是事件系统](#什么是事件系统)
2. [事件类型定义](#事件类型定义)
3. [事件发布机制](#事件发布机制)
4. [事件订阅机制](#事件订阅机制)
5. [完整的事件流](#完整的事件流)
6. [实践练习](#实践练习)

## 什么是事件系统

**概念**：事件系统是一种**发布-订阅**模式的通信机制。

**类比**：
- **发布者**（Publisher）：像广播电台，发送消息
- **订阅者**（Subscriber）：像收音机，接收消息
- **事件总线**（Bus）：像无线电波，传递消息

**好处**：
- **解耦**：组件之间不需要直接引用
- **灵活**：可以随时添加/移除订阅者
- **可扩展**：新增功能只需订阅相关事件

## 事件类型定义

### Session事件

**代码位置**：`packages/opencode/src/session/index.ts:105-138`

```typescript
export const Event = {
  Created: BusEvent.define(
    "session.created",
    z.object({
      info: Info,
    }),
  ),
  Updated: BusEvent.define(
    "session.updated",
    z.object({
      info: Info,
    }),
  ),
  Deleted: BusEvent.define(
    "session.deleted",
    z.object({
      info: Info,
    }),
  ),
  Diff: BusEvent.define(
    "session.diff",
    z.object({
      sessionID: z.string(),
      diff: Snapshot.FileDiff.array(),
    }),
  ),
  Error: BusEvent.define(
    "session.error",
    z.object({
      sessionID: z.string().optional(),
      error: MessageV2.Assistant.shape.error,
    }),
  ),
}
```

**说明**：
- `session.created`：Session创建时触发
- `session.updated`：Session更新时触发
- `session.deleted`：Session删除时触发
- `session.diff`：生成文件差异时触发
- `session.error`：发生错误时触发

---

### Message事件

**代码位置**：`packages/opencode/src/session/message-v2.ts:401-424`

```typescript
export const Event = {
  Updated: BusEvent.define(
    "message.updated",
    z.object({
      info: Info,
    }),
  ),
  Removed: BusEvent.define(
    "message.removed",
    z.object({
      sessionID: z.string(),
      messageID: z.string(),
    }),
  ),
  PartUpdated: BusEvent.define(
    "message.part.updated",  // ⭐ 最常用的事件！
    z.object({
      part: Part,
      delta: z.string().optional(),
    }),
  ),
  PartRemoved: BusEvent.define(
    "message.part.removed",
    z.object({
      sessionID: z.string(),
      messageID: z.string(),
      partID: z.string(),
    }),
  ),
}
```

**说明**：
- `message.updated`：Message更新时触发
- `message.removed`：Message删除时触发
- `message.part.updated`：Part更新时触发 ⭐ **最重要！**
- `message.part.removed`：Part删除时触发

---

### SessionStatus事件

**代码位置**：`packages/opencode/src/session/status.ts:27-42`

```typescript
export const Event = {
  Status: BusEvent.define(
    "session.status",
    z.object({
      sessionID: z.string(),
      status: Info,  // { type: "busy" | "retry" | "idle" }
    }),
  ),
  Idle: BusEvent.define(
    "session.idle",
    z.object({
      sessionID: z.string(),
    }),
  ),
}
```

**说明**：
- `session.status`：Session状态变化时触发
- `session.idle`：Session空闲时触发

---

### 其他事件

- **Todo.Event.Updated**：Todo列表更新
- **SessionCompaction.Event.Compacted**：Session压缩完成
- **Permission events**：权限请求/响应

## 事件发布机制

### 发布位置

**Session更新**：`packages/opencode/src/session/index.ts:229-246`

```typescript
export async function createNext(input: {...}) {
  const result: Info = { ... }

  // 1. 保存到存储
  await Storage.write(["session", project.id, result.id], result)

  // 2. 发布事件 ⭐
  Bus.publish(Event.Created, {
    info: result,
  })

  // 3. 可能自动分享
  if (!result.parentID && cfg.share === "auto") {
    share(result.id)
  }

  return result
}
```

**Part更新**：`packages/opencode/src/session/index.ts:425-434`

```typescript
export const updatePart = fn(UpdatePartInput, async (input) => {
  const part = "delta" in input ? input.part : input
  const delta = "delta" in input ? input.delta : undefined

  // 1. 保存到存储
  await Storage.write(["part", part.messageID, part.id], part)

  // 2. 发布事件 ⭐
  Bus.publish(MessageV2.Event.PartUpdated, {
    part,
    delta,  // 增量文本
  })

  return part
})
```

**Message更新**：`packages/opencode/src/session/index.ts:373-379`

```typescript
export const updateMessage = fn(MessageV2.Info, async (msg) => {
  await Storage.write(["message", msg.sessionID, msg.id], msg)

  // 发布事件
  Bus.publish(MessageV2.Event.Updated, {
    info: msg,
  })

  return msg
})
```

### 何时发布事件

| 操作 | 发布的事件 | 时机 |
|------|-----------|------|
| 创建Session | `session.created` | Session.create() |
| 更新Session | `session.updated` | Session.update() |
| 删除Session | `session.deleted` | Session.remove() |
| 创建Message | `message.updated` | MessageV2.create() |
| 更新Message | `message.updated` | Session.updateMessage() |
| 删除Message | `message.removed` | Session.removeMessage() |
| **更新Part** | **`message.part.updated`** | **Session.updatePart() ⭐** |
| 删除Part | `message.part.removed` | Session.removePart() |

## 事件订阅机制

### 订阅示例1：分享服务

**代码位置**：`packages/opencode/src/share/share.ts:50-66`

```typescript
export function init() {
  // 订阅Session更新事件
  Bus.subscribe(Session.Event.Updated, async (evt) => {
    await sync("session/info/" + evt.properties.info.id, evt.properties.info)
  })

  // 订阅Message更新事件
  Bus.subscribe(MessageV2.Event.Updated, async (evt) => {
    await sync("session/message/" + evt.properties.info.sessionID + "/" + evt.properties.info.id, evt.properties.info)
  })

  // 订阅Part更新事件
  Bus.subscribe(MessageV2.Event.PartUpdated, async (evt) => {
    await sync(
      "session/part/" +
        evt.properties.part.sessionID + "/" +
        evt.properties.part.messageID + "/" +
        evt.properties.part.id,
      evt.properties.part
    )
  })
}
```

**作用**：将session/message/part同步到云端分享服务

---

### 订阅示例2：Task工具

**代码位置**：`packages/opencode/src/tool/task.ts:117-127`

```typescript
// 订阅Part更新事件，追踪子任务进度
const unsub = Bus.subscribe(MessageV2.Event.PartUpdated, async (evt) => {
  if (evt.properties.part.sessionID !== session.id) return  // 过滤：只关心当前session
  if (evt.properties.part.messageID === messageID) return   // 过滤：忽略当前消息
  if (evt.properties.part.type !== "tool") return           // 过滤：只关心工具

  const part = evt.properties.part
  parts[part.id] = {
    id: part.id,
    tool: part.tool,
    state: {
      status: part.state.status,
      title: part.state.status === "completed" ? part.state.title : undefined,
    },
  }

  // 更新元数据，显示在主session中
  ctx.metadata({
    title: params.description,
    metadata: { summary: Object.values(parts) }
  })
})
```

**作用**：Task工具追踪子session中的工具执行进度

---

### 订阅示例3：CLI终端

**代码位置**：`packages/opencode/src/cli/cmd/run.ts:157-228`

```typescript
const events = await sdk.event.subscribe()

for await (const event of events.stream) {
  // 处理Part更新事件
  if (event.type === "message.part.updated") {
    const part = event.properties.part

    if (part.sessionID !== sessionID) continue  // 过滤

    if (part.type === "tool" && part.state.status === "completed") {
      // 显示工具执行结果
      const [tool, color] = TOOL[part.tool] ?? [part.tool, UI.Style.TEXT_INFO_BOLD]
      printEvent(color, tool, part.state.title || JSON.stringify(part.state.input))
    }

    if (part.type === "text" && part.time?.end) {
      // 显示文本
      process.stdout.write(part.text + EOL)
    }
  }

  // 处理权限请求事件
  if (event.type === "permission.asked") {
    const result = await select({
      message: `Permission required: ${permission.permission}`,
      options: [
        { value: "once", label: "Allow once" },
        { value: "always", label: "Always allow" },
        { value: "reject", label: "Reject" },
      ],
    })

    await sdk.permission.respond({
      sessionID,
      permissionID: permission.id,
      response: result,
    })
  }
}
```

**作用**：CLI订阅事件，实时显示AI输出和工具执行结果

## 完整的事件流

### 场景：用户输入"创建一个txt文件"

```
用户: opencode run "创建一个txt文件"
     ↓
┌─────────────────────────────────────────────────┐
│  1. 创建Session                                │
│     Bus.publish(Event.Created, { info })       │
└────────────┬────────────────────────────────────┘
             │
             ├─→ 分享服务订阅 → 同步到云端
             └─→ UI订阅 → 更新session列表
             │
             ▼
┌─────────────────────────────────────────────────┐
│  2. 创建User Message                           │
│     Bus.publish(Event.Updated, { info })       │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────┐
│  3. LLM开始流式输出                            │
│     ┌───────────────────────────────────────┐  │
│     │ text-delta: "我"                       │  │
│     │   → Session.updatePart({ text: "我" }) │  │
│     │   → Bus.publish(PartUpdated)            │  │
│     └───────────────────────────────────────┘  │
│     ┌───────────────────────────────────────┐  │
│     │ text-delta: "来"                      │  │
│     │   → Session.updatePart({ text: "我来" })│  │
│     │   → Bus.publish(PartUpdated)            │  │
│     └───────────────────────────────────────┘  │
└────────────┬────────────────────────────────────┘
             │
             ├────────────────────────────────┼─────────────┐
             │                                │             │
             ▼                                ▼             ▼
        ┌─────────┐                      ┌─────────┐  ┌──────────┐
        │ Web UI  │                      │ CLI UI  │  │Share服务  │
        │         │                      │         │  │          │
        │ 订阅    │                      │ 订阅    │  │ 订阅     │
        │ PartUpdated                     │PartUpdated│  │PartUpdated│
        │         │                      │         │  │          │
        │ 显示: 我  │                      │显示: 我  │  │同步到云端│
        │         │                      │         │  │          │
        └─────────┘                      └─────────┘  └──────────┘
```

### 事件传播时序图

```
SessionProcessor
     │
     │ Session.updatePart({ part })
     │
     ├─→ Storage.write() → 持久化
     │
     └─→ Bus.publish(PartUpdated, { part })
          │
          ├─→ [分享服务] 收到事件
          │     └─→ sync到云端
          │
          ├─→ [CLI] 收到事件
          │     └─→ 显示到终端
          │
          └─→ [Web UI] 收到事件
                └─→ 更新界面
```

## 实践练习

### 练习1：查看事件定义

```bash
# 搜索所有事件定义
grep -r "BusEvent.define" packages/opencode/src/session/

# 输出：
# packages/opencode/src/session/index.ts:     Session.Event
# packages/opencode/src/session/message-v2.ts: MessageV2.Event
# packages/opencode/src/session/status.ts:     SessionStatus.Event
# packages/opencode/src/session/compaction.ts: SessionCompaction.Event
# packages/opencode/src/session/todo.ts:       Todo.Event
```

### 练习2：查看事件发布点

```bash
# 搜索所有事件发布
grep -r "Bus.publish" packages/opencode/src/session/

# 主要发布点：
# Session.create() → session.created
# Session.update() → session.updated
# Session.updatePart() → message.part.updated ⭐
# Session.updateMessage() → message.updated
```

### 练习3：查看事件订阅者

```bash
# 搜索所有事件订阅
grep -r "Bus.subscribe" packages/opencode/src/

# 主要订阅者：
# packages/opencode/src/share/share.ts - 分享服务
# packages/opencode/src/tool/task.ts - Task工具
# packages/opencode/src/cli/cmd/github.ts - CLI GitHub工具
```

### 练习4：观察事件流

```bash
# 使用JSON格式输出，看所有事件
opencode run "测试" --format json

# 你会看到：
{"type":"message.part.updated","properties":{"part":{...}}}
{"type":"message.part.updated","properties":{"part":{...}}}
...
```

### 练习5：添加自定义订阅（模拟）

创建一个测试文件：

```typescript
// test-event-subscribe.ts
import { Bus } from "./packages/opencode/src/bus"
import { MessageV2 } from "./packages/opencode/src/session/message-v2"

// 订阅Part更新事件
const unsub = Bus.subscribe(MessageV2.Event.PartUpdated, (evt) => {
  console.log("Part updated:", evt.properties.part.type)

  if (evt.properties.part.type === "tool") {
    console.log("Tool:", evt.properties.part.tool)
    console.log("Status:", evt.properties.part.state.status)
  }
})

// ... 运行一些代码 ...

// 取消订阅
unsub()
```

## 总结

### 核心要点

1. **事件驱动架构**：组件间通过事件通信，解耦合
2. **发布-订阅模式**：
   - 发布者：`Bus.publish(Event, data)`
   - 订阅者：`Bus.subscribe(Event, handler)`
3. **最常用事件**：`message.part.updated` ⭐
4. **事件过滤**：订阅时根据sessionID、messageID等过滤
5. **异步处理**：事件处理是异步的，不会阻塞发布者

### 事件类型速查

| 事件类型 | 发布时机 | 主要订阅者 |
|---------|---------|-----------|
| `session.created` | Session创建 | 分享服务、UI |
| `session.updated` | Session更新 | 分享服务、UI |
| `message.updated` | Message更新 | 分享服务 |
| **`message.part.updated`** | **Part更新** ⭐ | **分享服务、CLI、Web UI** |
| `session.status` | 状态变化 | CLI、Web UI |
| `session.error` | 错误发生 | CLI、Web UI |
| `permission.asked` | 权限请求 | CLI |

### 学习路径

1. ✅ 理解事件系统的概念
2. ✅ 了解事件类型定义
3. ✅ 掌握发布和订阅机制
4. ⭐ **实践**：观察事件流
5. ⭐ **实践**：查看发布和订阅代码

### 下一步学习

- [05-工具系统](../05-工具系统/) - 理解工具如何使用事件
- [07-CLI和UI](../07-CLI和UI/) - 深入理解UI如何订阅事件

### 相关代码文件

- `packages/opencode/src/bus/bus-event.ts` - 事件定义
- `packages/opencode/src/session/index.ts` - 事件发布
- `packages/opencode/src/share/share.ts` - 订阅示例
- `packages/opencode/src/cli/cmd/run.ts` - CLI订阅示例
