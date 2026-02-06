# OpenCode 学习文档

欢迎来到OpenCode学习文档！这里记录了OpenCode项目的核心概念和实现细节。

## 📚 文档结构

本文档按照"八面受敌法"组织，每个"面"代表项目的一个核心方面：

1. [01-项目架构](./01-项目架构/) - 理解项目整体结构和技术栈
2. [02-Session系统](./02-Session系统/) - 深入理解Session的生命周期
3. [03-LLM流式输出](./03-LLM流式输出/) - 理解AI输出如何被处理
4. [04-事件系统](./04-事件系统/) - 理解组件间通信机制
5. [05-工具系统](./05-工具系统/) - 理解工具定义和调用
6. [06-权限系统](./06-权限系统/) - 理解权限控制机制
7. [07-CLI和UI](./07-CLI和UI/) - 理解用户界面实现
8. [08-高级特性](./08-高级特性/) - 深入高级功能
9. [09-API使用指南](./09-API使用指南/) - 如何从外部调用OpenCode
10. [10-可复用模块](./10-可复用模块/) - 可复用到其他项目的模块

## 🎯 学习建议

### 循序渐进
按照01到08的顺序阅读，每个"面"都是独立的学习单元。

### 理论+实践
阅读文档时，建议配合以下活动：
- 运行CLI观察行为：`opencode run "测试"`
- 查看生成的文件：`~/.opencode/storage/`
- 使用JSON格式看事件流：`opencode run "测试" --format json`
- 阅读相关源代码

### 动手实践
学完每个"面"后，尝试：
- 修改一个小功能
- 添加一个日志
- 创建一个简单工具

## 🛠️ 快速开始

### 运行项目
```bash
# 安装依赖
bun install

# 运行CLI
bun run run "帮我创建一个README.md"

# 启动Web服务
bun run serve

# 运行测试
bun test
```

### 查看日志
```bash
# 查看详细日志
LOG=debug bun run run "测试"
```

### 查看存储
```bash
# Session存储位置
ls ~/.opencode/storage/session/

# Message存储位置
ls ~/.opencode/storage/message/{sessionID}/

# Part存储位置
ls ~/.opencode/storage/part/{messageID}/
```

## 📖 核心概念速查

### Session是什么？
- 一个完整的对话会话
- 包含多个Message
- 有自己的生命周期（创建、使用、销毁）
- 可以有父子关系（子session用于子任务）

### Message是什么？
- Session中的一条消息
- 分为User消息和Assistant消息
- 包含多个Part

### Part是什么？
- Message的组成部分
- 类型包括：text、tool、reasoning、file等
- 实时更新，触发UI刷新

### 流式输出为什么重要？
- 实时显示AI工作状态
- 可以中途中断
- 工具状态可见
- Token使用透明

### 事件如何工作？
1. 系统发布事件：`Bus.publish(Event.PartUpdated, { part })`
2. UI订阅事件：`Bus.subscribe(MessageV2.Event.PartUpdated, ...)`
3. 实时更新显示

## 🤝 贡献指南

欢迎补充和改进这些文档！

### 添加新文档
```bash
# 在对应目录创建markdown文件
vim docs/02-Session系统/新主题.md
```

### 改进现有文档
1. 找到相关文档
2. 编辑并补充内容
3. 提交PR

## 📞 获取帮助

- 查看[主README](../README.md)
- 查看[GitHub Issues](https://github.com/opencode-dev/opencode/issues)
- 加入社区讨论

---

**学习愉快！** 🚀
