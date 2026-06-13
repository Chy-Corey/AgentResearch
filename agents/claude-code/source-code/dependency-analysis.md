# Claude Code — 依赖分析

本文档详细分析 Claude Code 的依赖选型，理解其技术栈构成。

---

## 依赖概览

| 属性 | 值 |
|------|-----|
| 依赖总数 | 68 个 |
| 外部依赖 | 61 个（npm 包） |
| 本地垫片 | 7 个（shims/ 目录） |
| 包管理器 | Bun 1.3.5 |
| 模块类型 | ESM |

---

## 依赖分类

### 1. LLM 集成（核心）

与 Claude API 和 LLM 相关的依赖。

| 包名 | 作用 | 重要性 |
|------|------|--------|
| `@anthropic-ai/sdk` | **Anthropic 官方 SDK** — 调用 Claude API 的核心库 | ★★★ |
| `@anthropic-ai/claude-agent-sdk` | **Agent SDK** — Agent 开发框架 | ★★★ |
| `@anthropic-ai/sandbox-runtime` | **沙箱运行时** — 安全执行环境 | ★★☆ |
| `@anthropic-ai/mcpb` | **MCP 桥接** — MCP 协议实现 | ★★☆ |
| `@modelcontextprotocol/sdk` | **MCP SDK** — Model Context Protocol 官方 SDK | ★★★ |
| `@aws-sdk/client-bedrock-runtime` | **AWS Bedrock** — 通过 AWS 调用 Claude | ★★☆ |
| `google-auth-library` | **Google 认证** — GCP Vertex AI 认证 | ★☆☆ |

**关键说明**：
- `@anthropic-ai/sdk` 是最核心的依赖，所有 LLM 调用都通过它
- `@modelcontextprotocol/sdk` 实现 MCP 协议，用于扩展工具
- AWS Bedrock 和 GCP Vertex AI 是可选的 API 提供商

### 2. CLI 框架

命令行界面相关的依赖。

| 包名 | 作用 | 重要性 |
|------|------|--------|
| `@commander-js/extra-typings` | **CLI 框架** — 命令行参数解析（带类型） | ★★★ |
| `ink` | **终端 UI 框架** — 基于 React 的终端渲染 | ★★★ |
| `react` | **React** — UI 组件框架（Ink 依赖） | ★★★ |
| `react-reconciler` | **React 协调器** — Ink 的底层实现 | ★★☆ |
| `chalk` | **终端颜色** — 文本颜色和样式 | ★★☆ |
| `cli-boxes` | **终端框** — 绘制边框 | ★☆☆ |
| `figures` | **终端符号** — Unicode 符号（✓、✗ 等） | ★☆☆ |
| `indent-string` | **缩进** — 文本缩进处理 | ★☆☆ |
| `wrap-ansi` | **换行** — ANSI 文本自动换行 | ★☆☆ |
| `strip-ansi` | **清理** — 移除 ANSI 转义序列 | ★☆☆ |
| `get-east-asian-width` | **字符宽度** — 东亚字符宽度计算 | ★☆☆ |

**关键说明**：
- Claude Code 使用 **Ink**（React 终端 UI 框架）构建交互式界面
- 所有 UI 组件都是 React 组件，渲染到终端

### 3. 终端渲染

终端显示和格式化相关的依赖。

| 包名 | 作用 | 重要性 |
|------|------|--------|
| `asciichart` | **ASCII 图表** — 终端中绘制图表 | ★☆☆ |
| `code-excerpt` | **代码片段** — 提取代码片段 | ★☆☆ |
| `highlight.js` | **语法高亮** — 代码语法高亮 | ★★☆ |
| `marked` | **Markdown** — Markdown 解析和渲染 | ★★☆ |
| `diff` | **差异比较** — 文本差异对比 | ★★☆ |
| `qrcode` | **二维码** — 生成二维码（如 OAuth 链接） | ★☆☆ |

### 4. 文件系统和进程

文件操作和进程管理相关的依赖。

| 包名 | 作用 | 重要性 |
|------|------|--------|
| `execa` | **进程执行** — 执行子进程（比 child_process 更好用） | ★★★ |
| `tree-kill` | **进程终止** — 终止进程树 | ★★☆ |
| `signal-exit` | **退出信号** — 处理进程退出事件 | ★★☆ |
| `chokidar` | **文件监听** — 监听文件变化 | ★★☆ |
| `proper-lockfile` | **文件锁** — 防止并发写入冲突 | ★★☆ |
| `env-paths` | **路径** — 获取系统标准路径（配置、数据等） | ★☆☆ |
| `ignore` | **忽略规则** — .gitignore 风格的忽略规则 | ★★☆ |
| `picomatch` | **通配符** — 文件路径通配符匹配 | ★★☆ |

**关键说明**：
- `execa` 是执行终端命令的核心（如 `git status`、`npm install`）
- `chokidar` 用于监听文件变化（如 CLAUDE.md 变化时重新加载）

### 5. 数据处理

数据解析、验证和转换相关的依赖。

| 包名 | 作用 | 重要性 |
|------|------|--------|
| `zod` | **Schema 验证** — 数据结构定义和验证 | ★★★ |
| `ajv` | **JSON Schema** — JSON Schema 验证 | ★★☆ |
| `jsonc-parser` | **JSONC 解析** — 带注释的 JSON 解析 | ★★☆ |
| `yaml` | **YAML** — YAML 解析和生成 | ★★☆ |
| `shell-quote` | **Shell 引用** — Shell 命令参数解析 | ★★☆ |
| `lodash-es` | **工具库** — 通用工具函数（mapValues、pickBy 等） | ★★☆ |
| `type-fest` | **类型工具** — TypeScript 类型工具 | ★☆☆ |

**关键说明**：
- `zod` 用于定义工具的输入 Schema（如 BashTool 的参数格式）
- `ajv` 用于 JSON Schema 验证

### 6. 网络和 API

网络请求和 API 通信相关的依赖。

| 包名 | 作用 | 重要性 |
|------|------|--------|
| `axios` | **HTTP 客户端** — HTTP 请求 | ★★☆ |
| `undici` | **HTTP 客户端** — Node.js 原生 HTTP（更快） | ★★☆ |
| `ws` | **WebSocket** — WebSocket 客户端 | ★★☆ |
| `https-proxy-agent` | **代理** — HTTPS 代理支持 | ★☆☆ |

**关键说明**：
- `axios` 和 `undici` 都是 HTTP 客户端，可能在不同场景使用
- `ws` 用于 WebSocket 连接（如 MCP 通信）

### 7. 监控和遥测

性能监控和遥测相关的依赖。

| 包名 | 作用 | 重要性 |
|------|------|--------|
| `@opentelemetry/api` | **OpenTelemetry API** — 监控 API | ★★☆ |
| `@opentelemetry/api-logs` | **日志 API** — 结构化日志 | ★★☆ |
| `@opentelemetry/core` | **核心** — OpenTelemetry 核心 | ★★☆ |
| `@opentelemetry/resources` | **资源** — 资源信息 | ★☆☆ |
| `@opentelemetry/sdk-logs` | **日志 SDK** — 日志实现 | ★☆☆ |
| `@opentelemetry/sdk-metrics` | **指标 SDK** — 指标实现 | ★☆☆ |
| `@opentelemetry/sdk-trace-base` | **追踪 SDK** — 分布式追踪 | ★☆☆ |
| `@opentelemetry/semantic-conventions` | **语义约定** — 标准化标签 | ★☆☆ |

**关键说明**：
- OpenTelemetry 是可观测性框架，用于监控 Claude Code 的性能
- 包含日志、指标、追踪三大支柱

### 8. 功能标志

功能开关和 A/B 测试相关的依赖。

| 包名 | 作用 | 重要性 |
|------|------|--------|
| `@growthbook/growthbook` | **功能标志** — 功能开关和 A/B 测试 | ★★★ |

**关键说明**：
- GrowthBook 用于控制功能的启用/禁用（如 `feature('COORDINATOR_MODE')`）
- 支持 A/B 测试和渐进式发布

### 9. LSP（语言服务器协议）

代码智能相关的依赖。

| 包名 | 作用 | 重要性 |
|------|------|--------|
| `vscode-jsonrpc` | **JSON-RPC** — VS Code 通信协议 | ★★☆ |
| `vscode-languageserver-protocol` | **LSP 协议** — 语言服务器协议 | ★★☆ |
| `vscode-languageserver-types` | **LSP 类型** — LSP 类型定义 | ★☆☆ |

**关键说明**：
- LSP 用于获取代码智能信息（如定义跳转、引用查找）
- 让 Claude Code 能理解代码结构

### 10. 其他工具

| 包名 | 作用 | 重要性 |
|------|------|--------|
| `semver` | **版本号** — 语义化版本号解析 | ★☆☆ |
| `auto-bind` | **自动绑定** — 自动绑定 this | ★☆☆ |
| `auto-bind` | **自动绑定** — 自动绑定 this | ★☆☆ |
| `fuse.js` | **模糊搜索** — 模糊匹配（如命令搜索） | ★★☆ |
| `lru-cache` | **缓存** — LRU 缓存策略 | ★★☆ |
| `p-map` | **并发控制** — 并发执行 Promise | ★★☆ |
| `bidi-js` | **双向文本** — RTL 文本支持 | ★☆☆ |
| `emoji-regex` | **Emoji** — Emoji 正则匹配 | ★☆☆ |
| `stack-utils` | **堆栈** — 错误堆栈解析 | ★☆☆ |
| `supports-hyperlinks` | **超链接** — 终端超链接支持 | ★☆☆ |
| `usehooks-ts` | **React Hooks** — 常用 React Hooks | ★★☆ |
| `xss` | **XSS 防护** — 防止 XSS 攻击 | ★★☆ |

---

## 本地垫片（shims/）

`shims/` 目录包含 7 个本地包，用于平台兼容性和内部功能。

### Anthropic 内部垫片

| 包名 | 作用 |
|------|------|
| `@ant/claude-for-chrome-mcp` | Chrome 浏览器集成的 MCP 服务器 |
| `@ant/computer-use-input` | Computer Use 功能的输入处理 |
| `@ant/computer-use-mcp` | Computer Use 功能的 MCP 服务器 |
| `@ant/computer-use-swift` | Swift 平台的 Computer Use 支持 |

### 原生模块垫片

| 包名 | 作用 |
|------|------|
| `color-diff-napi` | 颜色差异计算（原生 N-API 模块） |
| `modifiers-napi` | 修饰键处理（如 Ctrl、Shift） |
| `url-handler-napi` | URL 处理（原生 N-API 模块） |

### 什么是垫片（Shim）？

**垫片 = 假包，让代码能编译通过，但功能不真正工作。**

源码中引用了 Anthropic 内部的包（如 `@ant/claude-for-chrome-mcp`）用于测试，但这些包在正式版本中不存在。如果没有垫片，`bun install` 会报错，代码跑不起来。

垫片的作用就是提供一个"假的"实现，让代码能编译和运行，但调用时功能不可用。

```
源码里写着：
import { runChromeMcp } from '@ant/claude-for-chrome-mcp'
  ↓
问题：这个包是 Anthropic 内部的，正式版没有
  ↓
如果不处理：bun install 报错，代码跑不起来
  ↓
解决方案：做一个"假包"（垫片）
  ↓
shims/ant-claude-for-chrome-mcp/
  └─ index.ts  ← 导出一个空函数
  ↓
代码能编译了，但调用时什么都不做（或报错"功能不可用"）
```

**为什么需要垫片？**

| 有垫片 | 没有垫片 |
|--------|----------|
| 代码能编译 | 编译报错 |
| 代码能运行 | 跑不起来 |
| 功能不可用但不崩溃 | 直接崩溃 |

**类比**：你要组装一台电脑，说明书说需要一个"无线网卡"，但你没有。你放了一个"假网卡"（垫片），电脑能组装完成，但上不了网。

---

## 技术栈总结

### 核心技术栈

| 层次 | 技术 | 说明 |
|------|------|------|
| 运行时 | Bun 1.3.5 | JavaScript 运行时（比 Node.js 更快） |
| 语言 | TypeScript | 类型安全的 JavaScript |
| 模块系统 | ESM | ES Modules（import/export） |
| CLI 框架 | Commander.js | 命令行参数解析 |
| UI 框架 | Ink + React | 终端 UI 渲染 |
| LLM SDK | @anthropic-ai/sdk | Claude API 调用 |
| 扩展协议 | MCP | Model Context Protocol |
| 验证 | Zod | 数据结构验证 |
| 监控 | OpenTelemetry | 可观测性 |
| 功能标志 | GrowthBook | 功能开关 |

### 架构特点

```
┌─────────────────────────────────────────────────────────┐
│                    用户界面层                              │
│  Ink + React（终端 UI）                                  │
└─────────────────────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────────────────────┐
│                    命令行层                                │
│  Commander.js（参数解析）                                 │
└─────────────────────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────────────────────┐
│                    核心逻辑层                              │
│  TypeScript + Zod（业务逻辑 + 验证）                      │
└─────────────────────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────────────────────┐
│                    LLM 集成层                              │
│  @anthropic-ai/sdk + MCP SDK（Claude API + 工具扩展）     │
└─────────────────────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────────────────────┐
│                    基础设施层                              │
│  execa（进程）+ fs（文件）+ ws（网络）+ OpenTelemetry（监控）│
└─────────────────────────────────────────────────────────┘
```

### 设计亮点

1. **Bun 运行时**：选择 Bun 而非 Node.js，启动更快、性能更好
2. **Ink UI 框架**：用 React 组件化方式构建终端 UI，代码更易维护
3. **Zod 验证**：用 Schema 定义工具输入，自动验证和类型推导
4. **MCP 协议**：标准化的工具扩展协议，支持第三方工具接入
5. **GrowthBook**：功能标志系统，支持渐进式发布和 A/B 测试
6. **OpenTelemetry**：标准化的可观测性框架，便于监控和调试

### 依赖选型特点

| 特点 | 说明 |
|------|------|
| **官方优先** | 核心功能使用 Anthropic 官方包 |
| **标准化** | 使用 OpenTelemetry、LSP 等标准协议 |
| **轻量级** | 选择轻量级库（如 execa 而非 child_process） |
| **类型安全** | 大量使用 TypeScript 和 Zod 验证 |
| **可扩展** | MCP 协议支持第三方扩展 |

---

## 依赖关系图

```
Claude Code
├─ @anthropic-ai/sdk ─────────── Claude API 调用
├─ @modelcontextprotocol/sdk ── MCP 协议
├─ @commander-js/extra-typings ─ CLI 参数解析
├─ ink + react ───────────────── 终端 UI
├─ zod ────────────────────────── 数据验证
├─ execa ──────────────────────── 进程执行
├─ chokidar ───────────────────── 文件监听
├─ ws ─────────────────────────── WebSocket
├─ axios + undici ─────────────── HTTP 请求
├─ @growthbook/growthbook ────── 功能标志
├─ @opentelemetry/* ──────────── 监控遥测
├─ vscode-* ──────────────────── LSP 协议
└─ shims/* ────────────────────── 平台兼容
```

---

## 总结

Claude Code 的依赖选型体现了以下特点：

1. **官方优先**：核心功能使用 Anthropic 官方包，保证兼容性
2. **标准化**：使用 MCP、LSP、OpenTelemetry 等标准协议
3. **现代化**：选择 Bun 运行时、ESM 模块、TypeScript
4. **可扩展**：MCP 协议支持第三方工具接入
5. **可观测**：OpenTelemetry 提供完整的监控能力
6. **安全**：沙箱运行时 + XSS 防护 + 输入验证
