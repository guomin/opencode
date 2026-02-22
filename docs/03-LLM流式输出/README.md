# LLM流式输出详解

LLM流式输出是OpenCode最核心的技术之一，理解它是理解整个项目的关键。

## 📋 目录

1. [为什么需要流式输出](#为什么需要流式输出)
2. [流式输出的整体流程](#流式输出的整体流程)
3. [LLM.stream() - 调用AI](#llmstream---调用ai)
4. [事件类型详解](#事件类型详解)
5. [SessionProcessor - 处理事件流](#sessionprocessor---处理事件流)
6. [实时更新机制](#实时更新机制)

## 为什么需要流式输出？

### 用户体验对比

**非流式输出**（传统方式）：
```
用户: "创建用户表"
     ↓
等待10秒... (完全不知道AI在做什么)
     ↓
AI: "我来帮你创建用户表..."
```

**流式输出**（OpenCode方式）：
```
用户: "创建用户表"
     ↓
🧠 让我分析一下需求... (立即看到AI在思考)
     ↓
我来帮您创建... (立即看到AI开始工作)
     ↓
⏳ Bash     sqlite3 schema.sql (看到工具执行中)
     ↓
✅ Created table users (看到工具完成)
```

### 流式输出的核心优势

| 优势 | 说明 | 用户体验 |
|------|------|----------|
| **实时反馈** | 立即看到AI工作 | 不焦虑，知道AI在工作 |
| **工具追踪** | 看到每个工具状态 | 知道执行了什么操作 |
| **可中断** | 随时Ctrl+C取消 | 发现方向不对可以立即停止 |
| **成本透明** | 实时看到token消耗 | 避免意外高额账单 |
| **打字机效果** | 逐字符显示 | 更自然、更友好 |

## 流式输出的整体流程

```
用户输入
     ↓
┌─────────────────────────────────────────────┐
│  SessionPrompt.prompt()                     │
│  ┌─────────────────────────────────────────┐ │
│  │ 1. 创建User Message                     │ │
│  │ 2. 创建Assistant Message                 │ │
│  │ 3. 调用 LLM.stream()                    │ │
│  └─────────────────────────────────────────┘ │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────┐
│  LLM.stream()                              │
│  ┌─────────────────────────────────────────┐ │
│  │ 1. 准备system prompt                    │ │
│  │ 2. 准备tools                            │ │
│  │ 3. 准备messages                         │ │
│  │ 4. 调用AI SDK的streamText()             │ │
│  └─────────────────────────────────────────┘ │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────┐
│  AI SDK返回流式事件                         │
│  for await (const value of stream) {       │
│    // value.type = "text-delta"            │
│    // value.type = "tool-call"             │
│    // ...                                   │
│  }                                          │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────┐
│  SessionProcessor.process()                │
│  ┌─────────────────────────────────────────┐ │
│  │ switch (value.type) {                   │ │
│  │   case "text-delta":                   │ │
│  │     → 创建/更新TextPart                │ │
│  │   case "tool-call":                    │ │
│  │     → 创建ToolPart (status: running)   │ │
│  │   case "tool-result":                  │ │
│  │     → 更新ToolPart (status: completed) │ │
│  │   ...                                  │ │
│  │ }                                      │ │
│  └─────────────────────────────────────────┘ │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────┐
│  Session.updatePart()                      │
│  ┌─────────────────────────────────────────┐ │
│  │ 1. 保存到存储                           │ │
│  │ 2. 发布事件: Bus.publish(PartUpdated)   │ │
│  └─────────────────────────────────────────┘ │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────┐
│  UI订阅事件 → 实时显示                      │
│  ┌─────────────────────────────────────────┐ │
│  │ text-delta → 逐字符显示                 │ │
│  │ tool-call → 显示 🔄 运行中              │ │
│  │ tool-result → 显示 ✅ 完成              │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

## LLM.stream() - 调用AI

**代码位置**：`packages/opencode/src/session/llm.ts:48-274`

```typescript
export async function stream(input: StreamInput) {
  // 1. 准备provider、模型、配置
  const [language, cfg, provider, auth] = await Promise.all([
    Provider.getLanguage(input.model),
    Config.get(),
    Provider.getProvider(input.model.providerID),
    Auth.get(input.model.providerID),
  ])

  // 2. 构建system prompt
  const system = [
    input.agent.prompt,
    ...input.system,
    input.user.system,
  ].filter(x => x).join("\n")

  // 3. 准备工具列表
  const tools = await resolveTools(input)

  // 4. 调用AI SDK的streamText
  return streamText({
    temperature: params.temperature,
    tools,
    maxOutputTokens,
    abortSignal: input.abort,
    messages: [
      { role: "system", content: system },
      ...input.messages,
    ],
    model: wrapLanguageModel({
      model: language,
      middleware: [
        extractReasoningMiddleware({ tagName: "think" })
      ],
    }),
  })
}
```

**返回值**：`StreamTextResult` - 一个流式迭代器

```typescript
// 使用方式
const stream = await LLM.stream(input)
for await (const value of stream.fullStream) {
  // 每个value都是一个小片段
}
```

## 事件类型详解

AI SDK返回的事件类型有很多，这里介绍最常用的几种。

> **💡 核心机制：事件是如何判断出来的？**
> 
> OpenCode 并没有自己手写正则或 JSON 解析器来处理底层数据流，而是完全依赖了 **Vercel AI SDK** 的标准化能力：
> 1. **抹平差异**：各大模型厂商（OpenAI, Anthropic 等）返回的 SSE (Server-Sent Events) 数据格式各不相同。Vercel AI SDK 底层的 Provider 负责监听这些原始 HTTP Stream。
> 2. **实时解析**：SDK 实时解析厂商的数据流，将其翻译成统一的、标准化的事件对象（如 `text-delta`, `tool-call`）。
> 3. **自动执行**：对于工具调用，SDK 会在后台自动执行传入的工具函数，并在执行完毕后抛出 `tool-result` 事件。
> 
> 这种设计极大地解耦了“模型 API 解析”和“业务逻辑处理”，使得 OpenCode 可以非常轻松地接入各种不同的新模型。

### 1. start 事件

```typescript
{ type: "start" }
```

**作用**：标记LLM开始工作

**处理**：
```typescript
case "start":
  SessionStatus.set(input.sessionID, { type: "busy" })
  break
```

**UI显示**：🔄 AI思考中...

---

### 2. reasoning-delta 事件（推理过程）

```typescript
{
  type: "reasoning-delta",
  id: "reason_123",
  text: "我"  // 增量文本
}
```

**作用**：Claude模型的thinking过程

**处理**：
```typescript
case "reasoning-delta":
  part.text += value.text  // 累积文本
  await Session.updatePart({ part, delta: value.text })
  break
```

**UI显示**：
```
🧠 让
🧠 让我
🧠 让我分析
🧠 让我分析一下
```

---

### 3. text-delta 事件（文本输出）

```typescript
{
  type: "text-delta",
  text: "我"  // 增量字符
}
```

**作用**：AI的文本回复，逐字符输出

**处理**：
```typescript
case "text-delta":
  if (currentText) {
    currentText.text += value.text  // 累积：我 → 我来 → 我来帮

    if (currentText.text) {
      await Session.updatePart({
        part: currentText,
        delta: value.text,  // 增量，用于UI流畅显示
      })
    }
  }
  break
```

**UI显示**：
```
我        → 显示
我来      → 追加"来"
我来帮    → 追加"帮"
...
```

---

### 4. tool-call 事件（工具调用）

```typescript
{
  type: "tool-call",
  toolCallId: "call_123",
  toolName: "bash",
  input: {
    command: "ls -la"
  }
}
```

**作用**：AI决定调用工具

**处理**：
```typescript
case "tool-call": {
  const part = await Session.updatePart({
    tool: value.toolName,
    state: {
      status: "running",  // 运行中
      input: value.input,
      time: { start: Date.now() },
    },
  })

  // 检测无限循环（连续3次调用同一工具）
  const lastThree = parts.slice(-3)
  if (lastThree.every(p =>
    p.tool === value.toolName &&
    JSON.stringify(p.state.input) === JSON.stringify(value.input)
  )) {
    await PermissionNext.ask({
      permission: "doom_loop",
      patterns: [value.toolName],
    })
  }
  break
}
```

**UI显示**：
```
⏳ Bash     ls -la
```

---

### 5. tool-result 事件（工具结果）

```typescript
{
  type: "tool-result",
  toolCallId: "call_123",
  output: {
    output: "文件列表...",
    title: "Listed files"
  }
}
```

**作用**：工具执行完成，返回结果

**处理**：
```typescript
case "tool-result": {
  await Session.updatePart({
    ...part,
    state: {
      status: "completed",  // 完成
      output: value.output.output,
      title: value.output.title,
    },
  })
  break
}
```

**UI显示**：
```
✅ Bash     Listed files
```

---

### 6. finish-step 事件（步骤结束）

```typescript
{
  type: "finish-step",
  finishReason: "stop",
  usage: {
    inputTokens: 1000,
    outputTokens: 500,
    ...
  }
}
```

**作用**：一个完整的响应步骤结束

**处理**：
```typescript
case "finish-step":
  // 1. 计算token使用
  const usage = Session.getUsage({ model, usage, metadata })

  // 2. 更新message元数据
  input.assistantMessage.finish = value.finishReason
  input.assistantMessage.cost += usage.cost
  input.assistantMessage.tokens = usage.tokens

  // 3. 保存step-finish part
  await Session.updatePart({
    type: "step-finish",
    tokens: usage.tokens,
    cost: usage.cost,
  })

  // 4. 生成文件patch
  if (snapshot) {
    const patch = await Snapshot.patch(snapshot)
    if (patch.files.length) {
      await Session.updatePart({
        type: "patch",
        files: patch.files,
      })
    }
  }

  // 5. 检查是否需要压缩
  if (await SessionCompaction.isOverflow({ tokens })) {
    needsCompaction = true
  }
  break
```

**UI显示**：
```
💰 Token: 1500, Cost: $0.002
📝 Modified: schema.sql, seed.sql
```

## SessionProcessor - 处理事件流

**代码位置**：`packages/opencode/src/session/processor.ts:45-401`

### 核心结构

```typescript
export function create(input: {
  assistantMessage: MessageV2.Assistant
  sessionID: string
  model: Provider.Model
  abort: AbortSignal
}) {
  const toolcalls: Record<string, MessageV2.ToolPart> = {}  // 追踪工具调用

  const result = {
    async process(streamInput: LLM.StreamInput) {
      while (true) {  // 支持重试的循环
        try {
          const stream = await LLM.stream(streamInput)

          for await (const value of stream.fullStream) {
            input.abort.throwIfAborted()  // 检查是否被取消

            switch (value.type) {
              // 处理各种事件类型...
            }
          }
        } catch (e) {
          // 错误处理和重试逻辑
          const retry = SessionRetry.retryable(error)
          if (retry !== undefined) {
            attempt++
            await sleep(delay)
            continue  // 重试
          }
        }
      }
    }
  }

  return result
}
```

### 处理循环

```
┌─────────────────────────────────────────┐
│  while (true) {                         │
│    try {                                │
│      for await (const value of stream) { │ ← 遍历事件流
│        switch (value.type) {             │ ← 处理事件
│          ...                              │
│        }                                │
│      }                                    │
│    } catch (e) {                         │ ← 错误处理
│      if (canRetry(e)) {                  │
│        retry                              │ ← 重试
│      } else {                            │
│        handleError(e)                     │ ← 处理错误
│        break                              │ ← 退出
│      }                                    │
│    }                                      │
│  }                                        │
└─────────────────────────────────────────┘
```

## 实时更新机制

### 1. 更新Part

**代码位置**：`packages/opencode/src/session/index.ts:425-434`

```typescript
export const updatePart = fn(UpdatePartInput, async (input) => {
  const part = "delta" in input ? input.part : input
  const delta = "delta" in input ? input.delta : undefined

  // 1. 保存到存储
  await Storage.write(["part", part.messageID, part.id], part)

  // 2. 发布事件 ⭐ 关键！
  Bus.publish(MessageV2.Event.PartUpdated, {
    part,
    delta,  // 增量文本，用于UI流畅显示
  })

  return part
})
```

### 2. 事件传播

```
SessionProcessor处理事件
     ↓
Session.updatePart({ part, delta })
     ↓
┌──────────────────────────────────────────┐
│  Bus.publish(MessageV2.Event.PartUpdated) │
└────────────┬─────────────────────────────┘
             │
             ├─────────────────────────────┼─────────────────┐
             │                             │                 │
             ▼                             ▼                 ▼
        ┌─────────┐                  ┌─────────┐      ┌──────────┐
        │ Web UI  │                  │ CLI UI  │      │ Share服务 │
        │ 订阅事件│                  │ 订阅事件│      │ 订阅事件  │
        └─────────┘                  └─────────┘      └──────────┘
             │                             │                 │
             ▼                             ▼                 ▼
      实时显示文本/工具状态            终端显示         同步到云端
```

### 3. UI订阅示例

**CLI订阅**：`packages/opencode/src/cli/cmd/run.ts:157-228`

```typescript
const events = await sdk.event.subscribe()

for await (const event of events.stream) {
  if (event.type === "message.part.updated") {
    const part = event.properties.part

    if (part.sessionID !== sessionID) continue  // 过滤

    if (part.type === "text" && part.time?.end) {
      // 显示文本
      process.stdout.write(part.text + EOL)
    }

    if (part.type === "tool" && part.state.status === "completed") {
      // 显示工具完成
      printEvent(color, tool, title)
    }
  }
}
```

## 实践练习

### 练习1：观察JSON事件流

```bash
# 使用JSON格式输出，看原始事件流
opencode run "创建一个txt文件" --format json
```

**输出示例**：
```json
{"type":"message.part.updated","timestamp":1704067200000,"sessionID":"sess_001","properties":{"part":{"type":"text","text":"我"}}}
{"type":"message.part.updated","timestamp":1704067200100,"sessionID":"sess_001","properties":{"part":{"type":"text","text":"我来"}}}
{"type":"message.part.updated","timestamp":1704067200200,"sessionID":"sess_001","properties":{"part":{"type":"tool","tool":"write","state":{"status":"running"}}}
{"type":"message.part.updated","timestamp":1704067201000,"sessionID":"sess_001","properties":{"part":{"type":"tool","tool":"write","state":{"status":"completed","title":"Created file"}}}
```

### 练习2：查看存储的Part

```bash
# 1. 创建session
opencode run "测试"

# 2. 找到sessionID
ls ~/.opencode/storage/session/*/

# 3. 查看messages
ls ~/.opencode/storage/message/{sessionID}/

# 4. 查看parts（能看到流式更新的痕迹）
ls ~/.opencode/storage/part/{messageID}/

# 5. 查看text part的增量更新
cat ~/.opencode/storage/part/{messageID}/part_001.json
# 会看到text字段是完整的文本（不是增量）

# 理解：增量只存在于事件流中，存储的是完整状态
```

### 练习3：追踪事件处理

在代码中添加日志：

```typescript
// packages/opencode/src/session/processor.ts

case "text-delta":
  log.info("text-delta", { text: value.text })  // 添加日志
  currentText.text += value.text
  break
```

运行：
```bash
LOG=debug opencode run "测试"
```

## 总结

### 核心要点

1. **流式输出是核心**：整个UI体验建立在流式输出之上
2. **事件驱动架构**：每个事件触发更新和UI刷新
3. **实时反馈**：用户立即看到AI工作状态
4. **可中断**：随时取消，节省token
5. **三层处理**：LLM.stream() → SessionProcessor → UI

### 学习路径

1. ✅ 理解为什么需要流式输出
2. ✅ 了解事件流的整体流程
3. ✅ 掌握主要事件类型
4. ⭐ **实践**：观察JSON事件流
5. ⭐ **实践**：查看存储的Part

### 下一步学习

- [04-事件系统](../04-事件系统/) - 深入理解事件如何传播

### 相关代码文件

- `packages/opencode/src/session/llm.ts` - LLM流式调用
- `packages/opencode/src/session/processor.ts` - 事件处理
- `packages/opencode/src/session/index.ts` - Part更新和事件发布
- `packages/opencode/src/cli/cmd/run.ts` - CLI事件订阅
