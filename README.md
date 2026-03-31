# Claude Code 源码学习文档

> 📚 **基于 2026-03-31 泄露的官方源码（通过 npm source map 暴露）**

---

## 📖 文档导航

### 1. [ARCHITECTURE.md](./ARCHITECTURE.md) - 总体架构
- 📊 规模统计（3768 文件，98K+ 行代码）
- 🏗️ 技术栈（Bun, React/Ink, Zod, MCP SDK）
- 📁 核心目录结构
- 🔧 核心系统概览（工具、命令、服务、桥接、权限、查询引擎）
- 🧠 关键设计模式
- 🚀 启动流程

### 2. [TOOL_SYSTEM.md](./TOOL_SYSTEM.md) - 工具系统
- 🔧 Tool 接口定义
- 📝 工具注册机制
- 🛠️ 工具实现示例（Bash, Edit, Agent, MCP）
- 📜 Prompt 设计
- 🔐 权限系统
- 📊 进度状态
- ✅ 最佳实践

### 3. [QUERY_ENGINE.md](./QUERY_ENGINE.md) - 查询引擎
- 🧠 QueryEngine 类定义
- ⚙️ 配置接口
- 🔄 submitMessage 核心方法
- 📝 系统提示构建
- 🌐 API 调用流程
- 🔧 工具执行
- 🗜️ 上下文压缩
- 📈 使用量追踪

### 4. [MCP_INTEGRATION.md](./MCP_INTEGRATION.md) - MCP 集成
- 📡 MCP 协议概述
- 🔌 客户端架构
- 🛠️ MCPTool 实现
- ⚙️ 服务器配置
- 📦 资源访问
- 🔐 OAuth 认证
- 🔗 连接管理
- 🛡️ 权限控制

### 5. [DESIGN_PATTERNS.md](./DESIGN_PATTERNS.md) - 设计模式
- 🏗️ 架构模式（工具注册、查询引擎、插件架构）
- 🔐 安全模式（分层权限、沙箱执行、敏感数据保护）
- ⚡ 性能模式（并行预取、延迟加载、缓存、流式处理）
- 🔄 状态管理模式（不可变状态、事件驱动、会话持久化）
- 🛠️ 错误处理模式（分类重试、错误边界、优雅降级）
- 🧪 测试模式
- 📦 代码组织模式

---

## 🔍 快速查找

### 我想了解...

| 主题 | 文档 | 章节 |
|-----|------|------|
| 整体架构 | ARCHITECTURE.md | 全文 |
| 技术栈选择 | ARCHITECTURE.md | 技术栈 |
| 工具如何定义 | TOOL_SYSTEM.md | 工具类型定义 |
| 如何添加新工具 | TOOL_SYSTEM.md | 工具实现示例 |
| 权限如何检查 | TOOL_SYSTEM.md | 权限系统 |
| LLM 调用流程 | QUERY_ENGINE.md | submitMessage |
| 流式响应处理 | QUERY_ENGINE.md | API 调用 |
| MCP 工具集成 | MCP_INTEGRATION.md | MCPTool 实现 |
| 性能优化技巧 | DESIGN_PATTERNS.md | 性能模式 |
| 错误处理策略 | DESIGN_PATTERNS.md | 错误处理模式 |

---

## 📂 源码位置

```
/root/projects/myapp/claude-code-source/claude-code/
├── src/
│   ├── main.tsx          # 入口
│   ├── commands.ts       # 命令注册
│   ├── tools.ts          # 工具注册
│   ├── Tool.ts           # 工具类型
│   ├── QueryEngine.ts    # 查询引擎
│   ├── tools/            # 工具实现
│   ├── commands/         # 命令实现
│   ├── services/         # 服务层
│   ├── components/       # UI 组件
│   └── ...
```

---

## 🎯 核心要点

### 1. 工具系统
- **Zod Schema 验证** - 所有工具输入都有严格的类型和验证
- **分层权限检查** - 工具级 → 规则级 → 模式级 → 用户级
- **流式进度输出** - 长时间操作通过 AsyncGenerator 实时反馈
- **Feature Flags** - 通过 `bun:bundle` 实现编译时条件代码

### 2. 查询引擎
- **单一职责** - 只负责 LLM 交互循环，工具执行委托出去
- **流式设计** - 每个重要事件都 yield，支持实时 UI 更新
- **上下文压缩** - 自动压缩过长历史，保持响应速度
- **错误恢复** - 智能重试机制，区分临时/永久错误

### 3. MCP 集成
- **多传输支持** - Stdio, SSE, HTTP 三种传输方式
- **动态工具发现** - 自动加载 MCP 服务器的工具
- **OAuth 流程** - 标准授权码流程，支持 token 刷新
- **Elicitation** - 需要用户授权时的交互机制

### 4. 架构特点
- **Bun 运行时** - 更快的启动和执行
- **React + Ink** - 终端 UI 组件化
- **并行预取** - 启动时并行加载资源
- **插件架构** - 支持第三方扩展

---

## 🚀 实践建议

### 如果你想实现类似系统：

1. **工具系统** → 参考 TOOL_SYSTEM.md
   - 定义清晰的 Tool 接口
   - 使用 Zod 进行输入验证
   - 实现分层权限检查
   - 支持流式进度输出

2. **LLM 交互** → 参考 QUERY_ENGINE.md
   - 封装 QueryEngine 管理状态
   - 使用 AsyncGenerator 流式输出
   - 实现智能重试和错误恢复
   - 设计上下文压缩策略

3. **MCP 协议** → 参考 MCP_INTEGRATION.md
   - 使用官方 MCP SDK
   - 支持多种传输方式
   - 实现 OAuth 认证流程
   - 设计权限控制机制

4. **性能优化** → 参考 DESIGN_PATTERNS.md
   - 并行预取资源
   - 延迟加载重型模块
   - 使用缓存减少重复计算
   - 流式处理长时间操作

---

## 📚 扩展阅读

- [MCP 官方文档](https://modelcontextprotocol.io)
- [Ink 框架](https://github.com/vadimdemedes/ink)
- [Bun 运行时](https://bun.sh)
- [Zod 验证库](https://zod.dev)
- [Anthropic API](https://docs.anthropic.com)

---

*生成时间: 2026-04-01*  
*基于源码版本: 2026-03-31 leak*
