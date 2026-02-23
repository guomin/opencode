# 工具与执行引擎深度解析

OpenCode 的**工具与执行引擎（Tool & Execution Engine）**是连接 LLM 意图与本地物理环境（文件系统、终端、网络等）的核心桥梁。整个引擎的设计高度模块化，主要分为三个层次：**工具定义层（Definition）**、**注册与发现层（Registry）**、以及**底层执行引擎（Shell/Execution）**。

## 1. 工具定义层 (`src/tool/tool.ts`)

OpenCode 提供了一套标准化的工具定义范式，确保所有工具（内置、自定义、插件）都具备一致的接口、类型安全和输出控制。

- **Zod Schema 校验**：每个工具必须定义 `parameters`（Zod Schema）。在工具执行前，引擎会自动拦截并校验 LLM 传入的参数。如果校验失败，会抛出带有明确指导信息的错误，引导 LLM 自我修正（`"Please rewrite the input so it satisfies the expected schema."`）。
- **`Tool.define` 包装器**：这是创建工具的标准工厂函数。它不仅封装了参数校验，还**内置了输出截断（Truncation）逻辑**。无论工具返回多长的数据，`Tool.define` 都会调用 `Truncate.output` 确保输出不会撑爆 LLM 的上下文窗口。
- **上下文注入 (`Tool.Context`)**：工具的 `execute` 函数会接收一个强大的 `ctx` 对象，包含：
  - `ask()`：调用权限系统（PermissionNext），在执行高危操作前向用户弹窗确认。
  - `metadata()`：向 UI 发送实时的执行状态和元数据（如当前正在读取哪个文件）。
  - `abort`：监听中断信号，支持用户随时取消长时间运行的工具。

## 2. 注册与发现机制 (`src/tool/registry.ts`)

`ToolRegistry` 负责管理当前会话可用的所有工具，它支持动态加载和多来源聚合：

- **内置工具 (Built-in)**：如 `BashTool`, `ReadTool`, `WriteTool`, `GrepTool`, `GlobTool` 等，提供基础的系统操作能力。
- **工作区自定义工具 (Workspace Tools)**：引擎会扫描当前项目根目录下的 `{tool,tools}/*.{js,ts}` 文件。这意味着用户可以在自己的项目中编写专属工具，OpenCode 会自动发现并注册它们。
- **插件工具 (Plugin Tools)**：通过 `@opencode-ai/plugin` 体系加载的第三方扩展工具。
- **环境感知**：注册表会根据当前运行环境（CLI、Desktop、App）动态过滤工具。例如，`QuestionTool`（向用户提问）只在有交互界面的客户端中注册。

## 3. 底层执行引擎 (`src/shell/shell.ts` & `src/tool/bash.ts`)

这是 OpenCode 最硬核的部分之一，专门为了安全、稳定地执行终端命令而设计。

- **跨平台 Shell 解析 (`shell.ts`)**：
  - 为了保证 LLM 输出的 Bash 脚本能在 Windows 上顺利执行，`Shell.acceptable()` 会进行智能降级和探测。
  - 在 Windows 上，它会优先寻找 **Git Bash**（通过探测 `git.exe` 的路径推导 `bash.exe`），从而为 LLM 提供一致的 Unix-like 环境。如果找不到，才会 fallback 到 `cmd.exe`。
  - **进程树清理 (`killTree`)**：实现了跨平台的进程树杀死逻辑（Windows 使用 `taskkill /t`，Unix 使用 `-pid` 进程组 kill），确保超时或用户取消时，不会遗留僵尸进程。
- **AST 语法树分析 (`bash.ts`)**：
  - 在执行 Bash 命令前，OpenCode 并没有直接丢给 `child_process.spawn`，而是引入了 **`web-tree-sitter` (WASM)** 对 Bash 命令进行 AST（抽象语法树）解析。
  - **核心目的**：为了**安全和权限控制**。通过解析 AST，引擎可以精确提取出命令中涉及的目录、文件路径、重定向操作等，从而在执行前准确判断是否需要向用户请求权限（例如跨出了当前工作区）。
- **超时与防挂死机制**：`BashTool` 默认设置了超时时间（通常是 2 分钟），防止 LLM 运行了阻塞命令（如 `npm start` 或 `tail -f`）导致整个 Agent 流程卡死。

## 4. 输出截断与保护 (`src/tool/truncation.ts`)

由于工具的输出（如 `cat` 一个巨大的日志文件，或 `ls -R`）可能极其庞大，引擎在底层强制实施了截断策略：

- 限制最大行数（`MAX_LINES`）和最大字节数（`MAX_BYTES`）。
- 如果输出被截断，引擎会在返回给 LLM 的文本末尾添加提示（如 `... (truncated)`），并在 `metadata` 中标记 `truncated: true`，LLM 看到后会知道信息不全，可能会改用 `grep` 或其他方式精确查找。

## 5. 完整工作流总结

1. **意图识别**：LLM 决定调用某个工具。
2. **工具匹配**：`ToolRegistry` 匹配对应的工具定义。
3. **参数校验**：Zod 拦截并校验参数，失败则打回重试。
4. **安全审查**：触发 `PermissionNext` 权限检查；如果是 Bash 命令，使用 `web-tree-sitter` 分析 AST 确保安全。
5. **底层执行**：`Shell` 跨平台执行命令或调用 Node.js 原生 API。
6. **输出处理**：捕获输出并进行 `Truncate` 截断保护。
7. **状态同步**：返回结果给 LLM，并通过 `metadata` 更新 UI 状态。
