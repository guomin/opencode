# 版本控制与安全（VCS & Permission）深度梳理

本章梳理 OpenCode 的**版本控制（Git 项目识别、分支/Worktree 沙箱）**与**安全体系（权限规则、边界判定、工具拦截）**。核心目标是：让 LLM 可以高效地在本地工作，同时把“不可逆/越界/高风险”的动作显式交给用户确认。

> 相关实现主要分布在：
> - `packages/opencode/src/project/*`（项目识别、实例边界）
> - `packages/opencode/src/project/vcs.ts`（当前分支追踪）
> - `packages/opencode/src/worktree/index.ts`（Git worktree 沙箱生命周期）
> - `packages/opencode/src/permission/next.ts`（规则化权限引擎）
> - `packages/opencode/src/tool/external-directory.ts`（越界目录拦截）
> - `packages/opencode/src/util/filesystem.ts`（路径规范化/包含关系判断）
> - `packages/opencode/src/session/prompt.ts`（工具调用上下文注入与权限挂载）

---

## 1. 关键术语与边界

- **Instance.directory**：当前会话/当前窗口实际工作的目录（可能是主工作区，也可能是一个沙箱 worktree）。
- **Instance.worktree**：项目的“根边界/根工作区”（Git 项目时指向 Git common dir 所在的工作区层级；非 Git 项目会被设为 `/`，并在 `containsPath` 中特殊处理以避免误判）。
- **Sandbox（沙箱）**：基于 `git worktree` 创建的隔离工作目录，OpenCode 默认把它放在全局数据目录下，并为其创建独立分支 `opencode/<name>`。

边界判定的核心 API 是 `Instance.containsPath(filepath)`：
- 只要路径在 `Instance.directory` 或 `Instance.worktree` 内，就视为“项目内路径”。
- 非 Git 项目时 `Instance.worktree` 可能为 `/`，为了避免“任何绝对路径都被认为在工作区内”的漏洞，`containsPath` 会跳过对 `/` 的 worktree 包含判断。

---

## 2. 版本控制：项目识别（Project ID）与 VCS 能力

### 2.1 Git 项目识别：`Project.fromDirectory`

OpenCode 会从当前目录向上查找 `.git`：
- 找到 `.git` 时，将该仓库视为一个 Git 项目。
- 通过 `git rev-list --max-parents=0 --all` 获取 root commit，并将其作为 **projectID**（并缓存到 `.git/opencode` 文件中，后续读取走缓存）。

这样做的好处：
- projectID 在同一仓库内稳定，不依赖目录名。
- 多个会话可以共享同一个 project 的配置与沙箱列表。

### 2.2 当前分支追踪：`project/vcs.ts`

`Vcs.branch()` 通过 `git rev-parse --abbrev-ref HEAD` 获取分支名，并用文件监听驱动更新：
- 订阅 `FileWatcher.Event.Updated`。
- 当 Git 元数据变化后，重新读取当前分支。
- 若变化，发布事件 `vcs.branch.updated`。

这类能力通常用于 UI 展示与“你正在对哪个分支做变更”的提示。

---

## 3. 版本控制：Worktree 沙箱（隔离工作区）

### 3.1 为什么用 `git worktree`

对 Agent 来说，“直接改主工作区”风险更高：
- 会污染当前分支、引入未审查改动
- 在多任务/多窗口并行时更容易相互干扰

Worktree 沙箱提供：
- 独立目录（隔离文件状态）
- 独立分支（隔离 Git 历史）
- 可一键重置到默认分支（回滚成本低）

### 3.2 创建：`Worktree.create`

关键步骤：
1. 仅 Git 项目允许创建沙箱（非 Git 会抛 `WorktreeNotGitError`）。
2. 生成唯一名称：`<adjective>-<noun>`，并保证：
   - 目录不存在
   - `refs/heads/opencode/<name>` 不存在
3. 执行 `git worktree add --no-checkout -b opencode/<name> <dir>`。
4. 在新 worktree 中执行 `git reset --hard` 填充文件。
5. 触发 `InstanceBootstrap` 引导新目录（用于初始化 LSP/配置等）。
6. 可选执行 start scripts（项目级 + worktree 额外脚本）。
7. 将 worktree 目录记录到 `Project.sandboxes`。

### 3.3 移除：`Worktree.remove`

1. `git worktree list --porcelain` 找到条目。
2. `git worktree remove --force <path>` 删除 worktree。
3. 若存在关联分支，`git branch -D <branch>` 删除分支。

### 3.4 重置：`Worktree.reset`（安全关键）

重置的目标是**把沙箱恢复到默认分支的干净状态**，并且强制校验“最终没有本地改动”。

关键动作：
- 选择目标（优先远端 HEAD，否则 main/master）：必要时 `git fetch`。
- `git reset --hard <target>`
- `git clean -fdx`
- 子模块强制同步：
  - `git submodule update --init --recursive --force`
  - `git submodule foreach --recursive git reset --hard`
  - `git submodule foreach --recursive git clean -fdx`
- `git status --porcelain=v1` 检查必须为空，否则认为重置失败。

这套组合拳的意义：把“清理副作用”的能力做成可复用的、安全的原子操作。

---

## 4. 安全：权限引擎（PermissionNext）

OpenCode 的权限不是简单的“是否允许工具”，而是：
- **permission**：权限域（如 `edit` / `read` / `external_directory` / `webfetch` / `doom_loop`）
- **pattern**：匹配对象（路径 glob、工具名、或自定义字符串）
- **action**：`allow | deny | ask`

### 4.1 配置到规则集：`PermissionNext.fromConfig`

配置结构来自 `Config.Permission`，支持两种写法：
- 单值：`{"bash": "deny"}` 等价于对 `*` 生效
- 对象：`{"read": {"*.env": "ask", "*": "allow"}}`

并支持 `~/`、`$HOME` 的展开。

规则的匹配策略：`evaluate()` 会 `findLast()`，即**最后一条匹配规则胜出**。
因此：
- 规则顺序非常重要
- `config.ts` 特意做了 preprocess/transform，用于尽可能保持原始 key 顺序，避免 zod 重排导致策略变化

### 4.2 询问/回复：`PermissionNext.ask` & `PermissionNext.reply`

工具执行前通过 `ctx.ask()` 进入权限流程：
- 若命中 `deny`：直接抛 `DeniedError`（硬阻断）
- 若命中 `ask`：发布 `permission.asked` 事件，并挂起 Promise 等待用户回复
- 回复 `once`：仅本次放行
- 回复 `always`：把 `always` 字段里的 glob 追加到已批准列表，并自动放行同 session 内可被覆盖的其它 pending
- 回复 `reject`：拒绝并可带纠正信息；还会拒绝同 session 下其它 pending 请求

另有 `disabled()`：若某工具在规则中对 `*` 设置为 `deny`，可用于在 UI 层直接禁用该工具。

### 4.3 Agent 默认权限与最小惊扰

Agent 默认权限在 `agent/agent.ts`：
- 基线：`{"*": "allow"}`（工具能力默认可用）
- 关键风险点强制 `ask`：如 `doom_loop`、`external_directory`
- `.env` 读取默认 `ask`（降低泄漏风险）
- 某些工具在特定 agent 中默认 `deny`（例如 plan 模式禁止 edit）

这体现了一个策略：**默认能做事，但在关键风险面前必须“停下来问一次”。**

---

## 5. 安全：工具侧如何落地（以写文件/越界目录为例）

### 5.1 越界目录：`tool/external-directory.ts`

`assertExternalDirectory(ctx, target)` 的判断逻辑：
- 若 target 不在 `Instance.containsPath()` 内，则触发 `external_directory` 权限：
  - `patterns: [<parentDir>/*]`
  - `always: [<parentDir>/*]`

这等价于把“目录访问”变成可被用户一次性授权的资源范围。

### 5.2 写文件：`tool/write.ts`

写文件工具在落盘前会：
1. `assertExternalDirectory`（避免写到工作区外）
2. 构造 diff（展示给用户/审计）
3. 触发 `edit` 权限，pattern 使用相对路径 `path.relative(Instance.worktree, filepath)`

### 5.3 编辑文件：`tool/edit.ts`

与 write 类似：
- 修改前先 `FileTime.assert`（防止并发改写导致的竞态）
- 生成 patch diff 并走 `edit` 权限

### 5.4 Windows 路径大小写绕过：`util/filesystem.ts`

`Filesystem.normalizePath()` 在 Windows 下使用 `realpathSync.native()` 获取规范大小写路径。
它主要用于：
- LSP 诊断路径对齐（避免大小写不一致导致查不到 diagnostics）
- 也间接降低“通过大小写变化绕过路径匹配”的概率（路径匹配仍需结合 `contains()` 与 pattern 规则理解）

---

## 6. 安全：在对话循环里挂载（Session 层）

`session/prompt.ts` 在构造工具执行上下文时，把 `ctx.ask()` 绑定到 `PermissionNext.ask()`：
- `ruleset = merge(agent.permission, session.permission)`
- 对 MCP 工具，会额外先走一次 `ctx.ask({ permission: <toolKey>, patterns: ["*"], always:["*"] })`，做到“外部能力也纳入权限体系”。

`session/processor.ts` 还实现了 `doom_loop` 防护：
- 当同一 toolName + 同一 input 连续出现达到阈值，会触发 `doom_loop` 权限询问。

---

## 7. 总结

- **版本控制**：Project 通过 Git root commit 建立稳定 projectID；通过 `git worktree` 为 Agent 提供隔离沙箱与可一键重置能力。
- **安全**：PermissionNext 提供“permission + pattern + action”的规则系统，结合 `Instance.containsPath` 的边界与工具侧的强制拦截，实现“默认可工作、关键动作必确认”。
- **工程落点**：Session 层统一注入 `ctx.ask`，使所有工具（内置/插件/MCP）都能被同一权限引擎治理。
