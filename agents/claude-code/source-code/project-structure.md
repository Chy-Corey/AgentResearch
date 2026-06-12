# Claude Code — 项目结构解读

## 项目基本信息

| 属性 | 值 |
|------|-----|
| 项目名 | @anthropic-ai/claude-code |
| 版本 | 999.0.0-restored（从源码映射重建） |
| 包管理器 | Bun 1.3.5 |
| 运行时要求 | Bun >= 1.3.5 / Node >= 24.0.0 |
| 模块类型 | ESM |
| 语言 | TypeScript + React JSX |

## 核心文件 / 目录（必读）

> 以下是最关键的文件和目录，理解它们就理解了 Claude Code 的核心架构。

### 启动链路（从启动到运行）

```
package.json scripts  →  src/dev-entry.ts  →  src/main.tsx  →  src/entrypoints/cli.tsx
```

| 文件 | 作用 | 重要性 |
|------|------|--------|
| `src/dev-entry.ts` | 开发入口，package.json scripts 指向 | ★★★ |
| `src/main.tsx` | 主入口，应用初始化 | ★★★ |
| `src/entrypoints/cli.tsx` | CLI 入口，命令行参数解析 | ★★★ |

### 核心架构（Agent 的大脑）

| 文件/目录 | 作用 | 重要性 |
|-----------|------|--------|
| `src/Tool.ts` | **工具基类** — 所有工具的父类，定义工具接口 | ★★★ |
| `src/tools.ts` | **工具注册中心** — 工具的发现和调度 | ★★★ |
| `src/tools/` | **工具实现** — 50+ 个具体工具（BashTool、FileEditTool 等） | ★★★ |
| `src/Task.ts` | **任务基类** — Agent 任务的抽象 | ★★★ |
| `src/QueryEngine.ts` | **查询引擎** — 与 LLM 的交互核心 | ★★★ |
| `src/context.ts` | **上下文管理** — 构建发送给 LLM 的上下文 | ★★★ |
| `src/state/` | **状态管理** — 全局状态存储 | ★★☆ |

### 扩展机制（可扩展性核心）

| 文件/目录 | 作用 | 重要性 |
|-----------|------|--------|
| `src/skills/` | **技能系统** — 自定义 Skill 加载（CLAUDE.md 定义的命令） | ★★★ |
| `src/hooks/` | **钩子系统** — 85 个 React Hooks + 事件钩子 | ★★☆ |
| `src/plugins/` | **插件系统** — 插件加载机制 | ★★☆ |
| `src/commands/` | **命令系统** — 102 个斜杠命令的实现 | ★★☆ |

### UI 层（终端界面）

| 文件/目录 | 作用 | 重要性 |
|-----------|------|--------|
| `src/components/` | **UI 组件** — 146 个 React/Ink 组件 | ★★☆ |
| `src/ink/` | **终端渲染** — Ink 框架集成 | ★★☆ |
| `src/screens/` | **页面** — 不同视图 | ★☆☆ |

### 基础设施

| 文件/目录 | 作用 | 重要性 |
|-----------|------|--------|
| `src/bridge/` | **桥接层** — 与 IDE/桌面应用通信 | ★★☆ |
| `src/services/` | **服务层** — API、MCP、OAuth 等 | ★★☆ |
| `src/utils/` | **工具函数** — 333 个通用函数 | ★☆☆ |

### 阅读优先级建议

```
第一优先级（理解核心架构）：
  Tool.ts → tools/ → tools.ts → Task.ts → QueryEngine.ts → context.ts

第二优先级（理解扩展机制）：
  skills/ → hooks/ → plugins/ → commands/

第三优先级（理解启动和UI）：
  dev-entry.ts → main.tsx → entrypoints/cli.tsx → components/ → ink/
```

## 根目录结构

```
sources/claude-code/
├── src/                  # 源码主目录
├── docs/                 # 文档
├── shims/                # 本地垫片包（平台兼容层）
├── vendor/               # 第三方代码
├── package.json          # 项目配置和依赖
├── tsconfig.json         # TypeScript 配置
├── bun.lock              # Bun 依赖锁文件
├── AGENTS.md             # 贡献指南（编码风格、测试、提交规范）
├── README.md             # 项目说明
├── preview.png           # 预览图
├── image-processor.node  # 原生 Node 模块
└── .gitignore
```

## src/ 目录结构

`src/` 下共 **38 个子目录** + **15 个顶层文件**，按职责可分为四类：

### 核心模块（Core）

| 目录 | 文件数 | 职责 |
|------|--------|------|
| `tools/` | 54 | **工具系统** — 50+ 个工具的实现（BashTool、FileEditTool、GrepTool 等） |
| `commands/` | 102 | **命令系统** — 斜杠命令实现（/help、/config、/commit 等） |
| `services/` | 38 | **服务层** — API 调用、MCP、OAuth、插件、分析等后端服务 |
| `context/` | 9 | **上下文管理** — 构建和维护发送给 LLM 的上下文 |
| `state/` | 6 | **状态管理** — 全局状态（会话、配置、权限等） |
| `query/` | 5 | **查询引擎** — 与 LLM 的交互、流式输出处理 |
| `coordinator/` | 2 | **协调器** — 多 Agent 协调 |

### 扩展模块（Extension）

| 目录 | 文件数 | 职责 |
|------|--------|------|
| `skills/` | 5 | **技能系统** — 自定义 Skill 的加载和管理 |
| `hooks/` | 85 | **钩子系统** — React Hooks + 事件钩子 |
| `plugins/` | 2 | **插件系统** — 插件加载和管理 |
| `jobs/` | 1 | **任务调度** — 后台任务 |
| `tasks/` | 11 | **任务管理** — Agent 任务的创建和跟踪 |

### UI 模块（Interface）

| 目录 | 文件数 | 职责 |
|------|--------|------|
| `components/` | 146 | **UI 组件** — React/Ink 组件（对话框、输入框、进度条等） |
| `screens/` | 3 | **屏幕/页面** — 不同界面视图 |
| `ink/` | 50 | **终端渲染** — Ink 框架集成（终端 UI） |
| `keybindings/` | 15 | **快捷键** — 键盘快捷键绑定 |
| `outputStyles/` | 1 | **输出样式** — 终端输出格式化 |
| `vim/` | 5 | **Vim 模式** — Vim 键位支持 |

### 基础设施（Infrastructure）

| 目录 | 文件数 | 职责 |
|------|--------|------|
| `entrypoints/` | 6 | **入口点** — CLI、MCP、SDK 的入口文件 |
| `cli/` | 8 | **CLI 相关** — 命令行参数解析、配置 |
| `bridge/` | 33 | **桥接层** — 与外部系统（IDE、桌面应用）的通信 |
| `remote/` | 4 | **远程功能** — 远程连接、远程触发 |
| `ssh/` | 2 | **SSH** — SSH 连接支持 |
| `server/` | 3 | **服务器** — 内置 HTTP/WebSocket 服务 |
| `schemas/` | 1 | **Schema** — 数据结构定义 |
| `types/` | 16 | **类型定义** — TypeScript 类型 |
| `constants/` | 22 | **常量** — 全局常量定义 |
| `utils/` | 333 | **工具函数** — 通用工具函数（最大目录） |
| `migrations/` | 11 | **迁移** — 数据/配置迁移脚本 |
| `native-ts/` | 3 | **原生模块** — 平台特定的原生实现 |

### 功能模块（Feature）

| 目录 | 文件数 | 职责 |
|------|--------|------|
| `voice/` | 1 | **语音** — 语音输入支持 |
| `memdir/` | 9 | **记忆目录** — 持久化记忆存储 |
| `proactive/` | 2 | **主动建议** — 主动式建议和提示 |
| `buddy/` | 6 | **伙伴模式** — 辅助功能 |
| `moreright/` | 1 | **扩展权限** — 额外权限管理 |
| `upstreamproxy/` | 2 | **上游代理** — 代理配置 |
| `assistant/` | 3 | **助手** — 助手相关功能 |

### 顶层关键文件

| 文件 | 职责 |
|------|------|
| `main.tsx` | 主入口，应用启动 |
| `dev-entry.ts` | 开发入口（package.json scripts 指向） |
| `Tool.ts` | 工具基类定义 |
| `tools.ts` | 工具注册和调度 |
| `Task.ts` | 任务基类定义 |
| `tasks.ts` | 任务管理 |
| `QueryEngine.ts` | 查询引擎 |
| `query.ts` | 查询逻辑 |
| `context.ts` | 上下文管理 |
| `commands.ts` | 命令注册 |
| `setup.ts` | 初始化设置 |
| `history.ts` | 历史记录 |
| `cost-tracker.ts` | Token 消耗追踪 |
| `costHook.ts` | 成本钩子 |
| `ink.ts` | Ink 终端 UI 框架集成 |

## 辅助目录

### shims/ — 本地垫片

提供平台兼容性，包含 3 个本地包：
- `ant-claude-for-chrome-mcp` — Chrome MCP 集成
- `computer-use-input` — 计算机使用输入
- `computer-use-mcp` — 计算机使用 MCP
- `computer-use-swift` — Swift 平台支持
- `color-diff-napi` — 颜色差异计算（原生）
- `modifiers-napi` — 修饰键处理（原生）
- `url-handler-napi` — URL 处理（原生）

### vendor/ — 第三方代码

包含直接嵌入的第三方库代码。

## 目录规模概览

```
文件数排名 Top 10：
utils/        333  ← 最大，通用工具函数
components/   146  ← UI 组件
commands/     102  ← 斜杠命令
hooks/         85  ← 钩子
tools/         54  ← 工具实现
ink/           50  ← 终端渲染
services/      38  ← 服务层
bridge/        33  ← 桥接层
constants/     22  ← 常量
types/         16  ← 类型定义
```
