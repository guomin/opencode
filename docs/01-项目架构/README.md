# 项目架构详解

理解OpenCode的整体架构是学习项目的第一步。这个文档将帮助你建立项目的宏观认知。

## 📋 目录

1. [项目概述](#项目概述)
2. [目录结构](#目录结构)
3. [技术栈](#技术栈)
4. [核心模块](#核心模块)
5. [数据流](#数据流)
6. [运行模式](#运行模式)

## 项目概述

### OpenCode是什么？

**定义**：OpenCode是一个AI驱动的开发工具，类似于Claude Code的开源实现。

**核心功能**：
- **AI对话编程**：通过自然语言与AI协作完成开发任务
- **多模型支持**：支持OpenAI、Anthropic、Google、Azure等多种AI提供商
- **多端支持**：CLI工具、Web控制台、桌面应用、Slack集成
- **MCP支持**：Model Context Protocol，扩展AI能力
- **多语言**：支持中英文等多种语言

**类比**：
```
OpenCode ≈ Claude Code（开源版）
         ≈ Cursor（开源版）
         + 更多自定义能力
```

---

### 项目定位

**目标用户**：
- 开发者：AI辅助编程
- 团队：协作开发
- 企业：私有化部署

**核心价值**：
- 开源免费
- 可私有化部署
- 高度可定制
- 支持多种AI模型

## 目录结构

### Monorepo结构

OpenCode使用**Monorepo**（单体仓库）架构，使用Bun和Turbo管理。

```
opencode/
├── packages/              # 所有子包
│   ├── opencode/          # 核心CLI工具包 ⭐
│   ├── console/           # Web控制台（Solid.js）
│   │   ├── app/          # 控制台前端应用
│   │   ├── core/         # 核心业务逻辑
│   │   ├── mail/         # 邮件服务
│   │   └── resource/     # 资源管理
│   ├── desktop/          # 桌面应用（Tauri）
│   ├── web/              # 营销网站（Astro）
│   ├── app/              # 通用前端组件（Solid.js）
│   ├── sdk/              # SDK库（TypeScript/JavaScript）
│   ├── plugin/           # 插件系统
│   ├── script/           # 脚本工具
│   ├── slack/            # Slack集成
│   └── util/             # 工具库
├── docs/                 # 学习文档 📚
├── test/                 # 测试文件
└── package.json         # 根package.json
```

---

### 核心包详解

#### 1. packages/opencode/ - CLI工具包 ⭐

**作用**：核心CLI工具，用于命令行交互

**关键文件**：
```
packages/opencode/
├── src/
│   ├── cli/             # CLI命令定义
│   │   ├── cmd/          # 命令实现
│   │   │   ├── run.ts     # opencode run 命令
│   │   │   ├── serve.ts   # opencode serve 命令
│   │   │   ├── session.ts # session管理命令
│   │   │   └── ...
│   ├── session/         # Session系统 ⭐ 核心！
│   │   ├── index.ts      # Session核心逻辑
│   │   ├── prompt.ts     # 用户输入处理
│   │   ├── processor.ts  # LLM流式输出处理
│   │   └── llm.ts        # LLM调用
│   ├── tool/            # 工具定义
│   ├── agent/           # Agent系统
│   ├── provider/        # AI提供商抽象
│   ├── storage/         # 存储抽象
│   └── bus/             # 事件系统
└── test/               # 测试文件
```

**主要命令**：
```bash
opencode run "消息"              # 开始对话
opencode serve                       # 启动Web服务
opencode session list              # 列出所有session
opencode agent list                # 列出所有agent
```

---

#### 2. packages/console/ - Web控制台

**作用**：Web界面，用于可视化操作

**技术栈**：
- Solid.js（前端框架）
- Vite（构建工具）
- TailwindCSS（样式）

**关键文件**：
```
packages/console/
├── app/               # 前端应用
│   ├── src/
│   │   ├── entry-client.tsx    # 客户端入口
│   │   ├── entry-server.tsx    # 服务端入口
│   │   └── pages/             # 页面组件
│   └── ...
├── core/              # 核心业务逻辑
│   ├── auth/             # 认证
│   ├── billing/          # 计费
│   └── ...
└── resource/          # 资源管理
```

**主要功能**：
- Session管理
- 实时AI对话界面
- 文件浏览器
- 终端集成

---

#### 3. packages/sdk/ - SDK库

**作用**：为TypeScript/JavaScript提供编程接口

**使用示例**：
```typescript
import { createOpencodeClient } from "@opencode-ai/sdk/v2"

const sdk = createOpencodeClient({ baseUrl: "http://localhost:4096" })

// 创建session
const session = await sdk.session.create({ title: "测试" })

// 发送消息
await sdk.session.prompt({
  sessionID: session.data.id,
  parts: [{ type: "text", text: "帮我写代码" }]
})

// 订阅事件流
const events = await sdk.event.subscribe()
for await (const event of events.stream) {
  console.log(event)
}
```

---

#### 4. packages/desktop/ - 桌面应用

**技术栈**：
- Tauri（桌面应用框架）
- Rust + Web前端

**作用**：原生桌面应用，提供更好的性能和集成

---

#### 5. packages/util/ - 工具库

**作用**：通用工具函数

**关键模块**：
```
packages/util/
├── slug/          # URL友好ID生成
├── error/         # 错误处理
├── format/        # 代码格式化
└── ...
```

## 技术栈

### 核心技术

| 技术 | 用途 | 版本 |
|------|------|------|
| **Bun** | JavaScript运行时、包管理 | latest |
| **TypeScript** | 类型安全 | ^5.0 |
| **Solid.js** | Web前端框架 | latest |
| **Vite** | 前端构建工具 | latest |

### Web框架

| 技术 | 用途 | 位置 |
|------|------|------|
| **Hono** | Web服务框架 | packages/console |
| **Astro** | 营销网站 | packages/web |
| **Tauri** | 桌面应用 | packages/desktop |

### AI集成

| 技术 | 用途 |
|------|------|
| **Vercel AI SDK** | AI SDK抽象层 |
| **Anthropic API** | Claude模型 |
| **OpenAI API** | GPT模型 |
| **Google AI** | Gemini模型 |

### 数据库

| 技术 | 用途 |
|------|------|
| **PlanetScale** | 云数据库（生产） |
| **Drizzle ORM** | 数据库ORM |
| **本地文件** | Session/Message/Part存储 |

## 核心模块

### 1. Session系统 🎯

**核心概念**：Session是完整的对话会话

**关键文件**：
```
packages/opencode/src/session/
├── index.ts         # Session核心逻辑
├── message-v2.ts    # Message和Part定义
├── prompt.ts        # 用户输入处理
├── processor.ts     # LLM流式输出处理
└── llm.ts          # LLM调用
```

**详细文档**：[02-Session系统](../02-Session系统/)

---

### 2. Agent系统 🤖

**核心概念**：Agent是具有特定能力的AI助手

**关键文件**：
```
packages/opencode/src/agent/
├── agent.ts         # Agent定义
├── build.ts         # Agent构建
└── ...
```

**Agent类型**：
- **primary agent**：主Agent，处理一般任务
- **subagent**：子Agent，处理特定任务（code-reviewer, frontend-design等）

---

### 3. 工具系统 🛠️

**核心概念**：工具是AI可以调用的能力

**关键文件**：
```
packages/opencode/src/tool/
├── tool.ts          # 工具定义
├── registry.ts      # 工具注册
├── read.ts          # Read工具
├── write.ts         # Write工具
├── bash.ts          # Bash工具
└── task.ts          # Task工具（创建子session）
```

**工具列表**：
- `read`：读取文件
- `write`：写入文件
- `edit`：编辑文件
- `bash`：执行命令
- `todowrite/todoread`：Todo管理
- `task`：创建子任务
- 更多...

---

### 4. 权限系统 🔐

**核心概念**：控制AI的能力范围

**关键文件**：
```
packages/opencode/src/permission/
├── next.ts          # 新权限系统
├── index.ts         # 旧权限系统（兼容）
└── ...
```

**权限规则**：
```typescript
{
  permission: "edit",       // 权限名称
  action: "allow",         // allow 或 deny
  pattern: "*.ts"          // 匹配模式
}
```

---

### 5. 存储系统 💾

**核心概念**：持久化Session/Message/Part

**存储位置**：
```
~/.opencode/storage/
├── session/{projectID}/{sessionID}.json
├── message/{sessionID}/{messageID}.json
└── part/{messageID}/{partID}.json
```

**关键文件**：
```
packages/opencode/src/storage/
├── storage.ts       # 存储抽象
├── memory.ts        # 内存存储（测试用）
└── ...
```

---

### 6. 事件系统 📡

**核心概念**：组件间通信机制

**关键文件**：
```
packages/opencode/src/bus/
├── bus.ts           # 事件总线
└── bus-event.ts     # 事件定义
```

**详细文档**：[04-事件系统](../04-事件系统/)

---

### 7. LSP集成 🔌

**核心概念**：Language Server Protocol，提供代码智能

**关键文件**：
```
packages/opencode/src/lsp/
├── client.ts        # LSP客户端
└── ...
```

---

### 8. MCP支持 🌐

**核心概念**：Model Context Protocol，扩展AI能力

**关键文件**：
```
packages/opencode/src/mcp/
├── index.ts         # MCP集成
└── ...
```

## 数据流

### 用户输入 → AI响应流程

```
用户输入
     ↓
┌─────────────────────────────────────────┐
│  CLI: opencode run "帮我写代码"          │
│  └─> SessionPrompt.prompt()              │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  创建Session和Message                     │
│  ┌─────────────────────────────────────┐ │
│  │ Session.create()                      │ │
│   → storage/session/{project}/{session} │ │
│  └─────────────────────────────────────┘ │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  调用LLM                                  │
│  ┌─────────────────────────────────────┐ │
│  │ LLM.stream()                         │ │
│   → streamText() (AI SDK)               │ │
│  └─────────────────────────────────────┘ │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  处理流式输出                             │
│  ┌─────────────────────────────────────┐ │
│  │ SessionProcessor.process()           │ │
│   → text-delta: 创建TextPart          │ │
│   → tool-call: 创建ToolPart            │ │
│   → tool-result: 更新ToolPart          │ │
│  └─────────────────────────────────────┘ │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  更新存储 + 发布事件                       │
│  ┌─────────────────────────────────────┐ │
│  │ Session.updatePart()                │ │
│   → storage/part/{message}/{part}      │ │
│   → Bus.publish(PartUpdated)           │ │
│  └─────────────────────────────────────┘ │
└────────────┬────────────────────────────┘
             │
             ├─────────────┬───────────────┐
             │             │               │
             ▼             ▼               ▼
    ┌─────────┐   ┌─────────┐   ┌──────────┐
    │Web UI   │   │ CLI UI  │   │Share服务  │
    │订阅事件  │   │订阅事件  │   │订阅事件   │
    │更新界面  │   │显示输出 │   │同步云端   │
    └─────────┘   └─────────�   └──────────┘
```

## 运行模式

### 1. CLI模式

```bash
# 直接运行
opencode run "帮我写代码"

# 交互模式
opencode run

# 指定模型
opencode run "测试" --model anthropic/claude-3.5-sonnet

# 指定agent
opencode run "代码审查" --agent code-reviewer
```

---

### 2. Web服务模式

```bash
# 启动Web服务
opencode serve

# 访问
open http://localhost:4096
```

**功能**：
- Web界面
- API服务
- WebSocket实时通信

---

### 3. 桌面应用模式

```bash
# 构建桌面应用
bun run desktop:build

# 运行
bun run desktop:dev
```

---

## 架构设计原则

### 1. 模块化

每个包都是独立的模块：
- 清晰的职责划分
- 最小化依赖
- 可独立测试

### 2. 事件驱动

组件间通过事件通信：
- 松耦合
- 易扩展
- 可测试

### 3. 流式优先

所有AI输出都是流式的：
- 实时反馈
- 可中断
- 节省成本

### 4. 存储分离

Session/Message/Part分开存储：
- 便于管理
- 提高性能
- 方便压缩

### 5. 多端支持

CLI/Web/桌面使用相同核心逻辑：
- 代码复用
- 一致体验
- 统一API

## 实践练习

### 练习1：探索目录结构

```bash
# 查看所有包
ls packages/

# 查看核心包的src目录
ls packages/opencode/src/

# 查看测试目录
ls packages/opencode/test/
```

---

### 练习2：运行项目

```bash
# 1. 安装依赖
bun install

# 2. 运行CLI
bun run run "测试"

# 3. 启动Web服务
bun run serve

# 4. 运行测试
bun test
```

---

### 练习3：查看配置

```bash
# 查看package.json
cat packages/opencode/package.json

# 查看tsconfig
cat tsconfig.json

# 查看turbo配置
cat turbo.json
```

---

### 练习4：追踪调用链

**任务**：理解"opencode run"命令如何工作

```bash
# 1. 找到命令定义
grep -r "run.*message" packages/opencode/src/cli/

# 2. 查看命令实现
# packages/opencode/src/cli/cmd/run.ts

# 3. 追踪调用链
# run.ts → SessionPrompt.prompt() → LLM.stream() → SessionProcessor.process()
```

## 总结

### 核心要点

1. **Monorepo架构**：多个独立包，统一管理
2. **核心包**：packages/opencode/ 是CLI的核心
3. **技术栈**：Bun + TypeScript + Solid.js
4. **模块化**：Session、Agent、Tool、Permission等独立模块
5. **事件驱动**：组件间通过事件通信
6. **多端支持**：CLI/Web/Desktop共享核心逻辑

### 学习路径

1. ✅ 理解项目定位和架构
2. ⭐ **深入学习Session系统**：[02-Session系统](../02-Session系统/)
3. ⭐ **理解流式输出**：[03-LLM流式输出](../03-LLM流式输出/)
4. ⭐ **掌握事件系统**：[04-事件系统](../04-事件系统/)
5. 选择感兴趣的模块深入：工具、权限、LSP、MCP

### 下一步

- 选择一个核心模块深入学习
- 阅读相关源代码
- 动手实践

### 相关文档

- [02-Session系统](../02-Session系统/) - 深入理解Session
- [03-LLM流式输出](../03-LLM流式输出/) - 理解AI输出处理
- [04-事件系统](../04-事件系统/) - 理解事件驱动架构
- [学习路径](../学习路径.md) - 完整学习计划
