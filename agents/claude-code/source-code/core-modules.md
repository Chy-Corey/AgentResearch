# Claude Code — 核心模块分析

本文档详细分析 Claude Code 的核心模块，帮助理解 Agent 的工作原理。

---

## 什么是 Agent？

在分析模块之前，先理解 Agent 的基本概念：

```
用户输入 → LLM（大脑）思考 → 决定使用什么工具 → 执行工具 → 得到结果 → 继续思考或回复用户
```

Agent = **LLM + 工具 + 循环**。LLM 负责思考和决策，工具负责执行具体操作，循环让 Agent 能够多步完成复杂任务。

Claude Code 就是一个这样的 Agent：它用 Claude 模型作为大脑，拥有 50+ 个工具，能在终端里帮你完成各种开发任务。

---

## 核心模块总览

```
┌─────────────────────────────────────────────────────────┐
│                    用户交互层                              │
│  components/  screens/  ink/  keybindings/               │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│                    Agent 核心层                            │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │ tools/   │  │commands/│  │ skills/ │  │ hooks/  │    │
│  │ 工具系统  │  │ 命令系统 │  │ 技能系统 │  │ 钩子系统 │    │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                 │
│  │ context/ │  │  state/ │  │ query/  │                 │
│  │ 上下文   │  │  状态    │  │ 查询引擎 │                 │
│  └─────────┘  └─────────┘  └─────────┘                 │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│                    服务层                                  │
│  services/api  services/mcp  services/oauth  services/plugins │
└─────────────────────────────────────────────────────────┘
```

---

## 1. 工具系统（tools/）

### 什么是工具？

工具是 Agent 能执行的具体操作。就像人的手能"拿、放、按、写"一样，Agent 通过工具来"读文件、写代码、运行命令、搜索"。

### 工具基类 Tool.ts

所有工具都继承自 `Tool.ts`，它定义了工具的通用接口：

```typescript
// Tool.ts 核心接口（简化）
interface Tool {
  name: string;           // 工具名称，如 "BashTool"
  description: string;    // 工具描述，告诉 LLM 这个工具能做什么
  inputSchema: JSONSchema; // 输入参数的格式定义
  
  // 核心方法
  isEnabled(): boolean;   // 是否启用
  validateInput(input): ValidationResult;  // 验证输入
  execute(input, context): Promise<ToolResult>;  // 执行工具
}
```

**关键设计**：LLM 不直接执行代码，而是通过"调用工具"来完成操作。Tool.ts 定义了工具的"说明书"，让 LLM 知道每个工具能做什么、怎么用。

### 工具分类

`src/tools/` 下有 **50+ 个工具**，按功能分为 7 大类：

#### 文件操作工具

| 工具 | 作用 | 举例 |
|------|------|------|
| `FileReadTool` | 读取文件内容 | 读取 package.json |
| `FileWriteTool` | 写入文件 | 创建新文件 |
| `FileEditTool` | 编辑文件（替换指定内容） | 修改函数名 |
| `GlobTool` | 按模式匹配文件 | 找到所有 .ts 文件 |
| `GrepTool` | 搜索文件内容 | 搜索 "TODO" 出现的位置 |
| `NotebookEditTool` | 编辑 Jupyter Notebook | 修改 notebook cell |

#### 终端工具

| 工具 | 作用 | 举例 |
|------|------|------|
| `BashTool` | 执行终端命令 | `npm install`、`git status` |
| `PowerShellTool` | 执行 PowerShell 命令 | Windows 环境下的命令 |

#### 交互工具

| 工具 | 作用 | 举例 |
|------|------|------|
| `AskUserQuestionTool` | 向用户提问 | "你想用哪种方案？" |
| `TodoWriteTool` | 创建/更新任务列表 | 显示工作进度 |
| `SendMessageTool` | 向其他 Agent 发消息 | 多 Agent 协作 |

#### 网络工具

| 工具 | 作用 | 举例 |
|------|------|------|
| `WebFetchTool` | 获取网页内容 | 读取 API 文档 |
| `WebSearchTool` | 搜索网络 | 搜索技术问题 |
| `WebBrowserTool` | 浏览器操作 | 自动化网页测试 |

#### 任务管理工具

| 工具 | 作用 | 举例 |
|------|------|------|
| `TaskCreateTool` | 创建后台任务 | 异步执行长时间操作 |
| `TaskOutputTool` | 获取任务输出 | 检查任务执行结果 |
| `TaskStopTool` | 停止任务 | 取消正在运行的任务 |
| `TaskListTool` | 列出所有任务 | 查看当前任务状态 |

#### 规划工具

| 工具 | 作用 | 举例 |
|------|------|------|
| `EnterPlanModeTool` | 进入规划模式 | 开始制定方案 |
| `ExitPlanModeTool` | 退出规划模式 | 提交方案供审批 |
| `VerifyPlanExecutionTool` | 验证方案执行 | 检查执行是否符合方案 |

#### 扩展工具

| 工具 | 作用 | 举例 |
|------|------|------|
| `SkillTool` | 调用自定义技能 | 执行 /skill 命令 |
| `MCPTool` | 调用 MCP 服务器工具 | 使用外部工具 |
| `AgentTool` | 创建子 Agent | 并行处理任务 |
| `WorkflowTool` | 执行工作流 | 运行多步骤自动化 |
| `ScheduleCronTool` | 定时任务 | 每天自动执行某操作 |

### 工具注册机制

`tools.ts` 负责注册所有工具：

```typescript
// tools.ts 核心逻辑（简化）
export function getTools(): Tools {
  const tools = [
    AgentTool,
    AskUserQuestionTool,
    BashTool,
    FileEditTool,
    FileReadTool,
    FileWriteTool,
    GlobTool,
    GrepTool,
    // ... 50+ 个工具
  ];
  
  // 根据功能标志过滤
  return tools.filter(tool => {
    if (tool.isEnabled && !tool.isEnabled()) return false;
    return true;
  });
}
```

**设计亮点**：
- 使用 `feature()` 函数做**条件加载**，未启用的功能不会被加载（减少启动时间）
- 某些工具用 `require()` 延迟加载，避免循环依赖

### 工具调用流程

```
用户："帮我把 main.ts 里的 TODO 注释删掉"
  ↓
LLM 思考：需要先读文件，再编辑文件
  ↓
LLM 决定调用：FileReadTool({ path: "main.ts" })
  ↓
工具系统：找到 FileReadTool，执行，返回文件内容
  ↓
LLM 看到文件内容，找到 TODO 注释
  ↓
LLM 决定调用：FileEditTool({ path: "main.ts", old: "...", new: "..." })
  ↓
工具系统：执行编辑，返回成功
  ↓
LLM：已完成，回复用户
```

---

## 2. 命令系统（commands/）

### 什么是命令？

命令是用户通过斜杠 `/` 触发的操作。比如 `/help`、`/config`、`/clear` 等。

### 命令分类

`commands/` 下有 **100+ 个命令**，按功能分类：

| 类别 | 示例命令 | 作用 |
|------|----------|------|
| 帮助 | `/help`、`/doctor` | 显示帮助信息、诊断问题 |
| 配置 | `/config`、`/model`、`/effort` | 修改配置、切换模型、调整努力程度 |
| 会话 | `/clear`、`/compact`、`/history` | 清空对话、压缩上下文、查看历史 |
| 开发 | `/commit`、`/diff`、`/pr` | Git 操作、查看差异、创建 PR |
| 扩展 | `/mcp`、`/install`、`/plugin` | 管理 MCP 服务器、安装插件 |
| 调试 | `/cost`、`/debug`、`/heapdump` | 查看消耗、调试信息、内存分析 |

### 命令与工具的关系

- **命令**：用户主动触发（`/` 开头）
- **工具**：LLM 自动选择调用

命令可以调用工具，但工具不能调用命令。命令是"用户意图的快捷方式"。

---

## 3. 技能系统（skills/）

### 什么是技能？

技能是用户自定义的可复用提示词（Prompt）。通过 `CLAUDE.md` 文件定义，可以用 `/skill-name` 调用。

### 技能文件结构

```
项目根目录/
├── CLAUDE.md              # 项目级配置和技能定义
├── .claude/
│   └── skills/            # 技能目录
│       ├── review.md      # /review 技能
│       └── deploy.md      # /deploy 技能
```

### 技能定义格式

```markdown
---
description: 代码审查技能
tools: ["FileReadTool", "GrepTool", "BashTool"]
---

请审查当前 git diff 的代码变更，关注：
1. 潜在的 bug
2. 性能问题
3. 代码风格
```

### 技能加载机制

`loadSkillsDir.ts` 负责加载技能：

1. 扫描 `~/.claude/skills/`（全局技能）
2. 扫描项目 `.claude/skills/`（项目技能）
3. 解析 Markdown 文件的 frontmatter（元数据）
4. 注册为可调用的命令

### 技能 vs 命令 vs 工具

| 概念 | 定义方式 | 调用方式 | 执行者 |
|------|----------|----------|--------|
| 工具 | TypeScript 代码 | LLM 自动选择 | 工具系统 |
| 命令 | TypeScript 代码 | 用户 `/command` | 命令系统 |
| 技能 | Markdown 文件 | 用户 `/skill` 或 LLM 调用 SkillTool | 技能系统 |

---

## 4. 钩子系统（hooks/）

### 什么是钩子？

钩子是 React Hooks，用于在 UI 组件中管理状态和副作用。Claude Code 使用 Ink（React 终端 UI 框架），所以用 React Hooks 来管理 UI 逻辑。

### 钩子分类

`hooks/` 下有 **85 个钩子**，按用途分类：

#### 输入相关

| 钩子 | 作用 |
|------|------|
| `useInputBuffer` | 管理用户输入缓冲区 |
| `useArrowKeyHistory` | 上下箭头键浏览历史输入 |
| `useCommandQueue` | 命令队列管理 |
| `useCopyOnSelect` | 选中即复制 |

#### 工具相关

| 钩子 | 作用 |
|------|------|
| `useCanUseTool` | 检查工具是否可用 |
| `toolPermission/` | 工具权限检查 |

#### UI 相关

| 钩子 | 作用 |
|------|------|
| `useBlink` | 光标闪烁动画 |
| `useElapsedTime` | 显示已用时间 |
| `useDiffData` | 差异数据展示 |
| `useDiffInIDE` | 在 IDE 中显示差异 |

#### 集成相关

| 钩子 | 作用 |
|------|------|
| `useIDEIntegration` | IDE 集成逻辑 |
| `useDirectConnect` | 直接连接 |
| `useDynamicConfig` | 动态配置 |

---

## 5. 上下文管理（context/）

### 什么是上下文？

上下文是发送给 LLM 的所有信息，包括：
- 系统提示词（System Prompt）
- 用户消息
- 对话历史
- 工具调用结果
- 项目信息（CLAUDE.md、Git 状态等）

### context.ts 核心函数

```typescript
// 获取系统上下文（系统提示词 + 项目信息）
getSystemContext(): Promise<SystemContext>

// 获取用户上下文（用户消息 + 附件）
getUserContext(): Promise<UserContext>
```

### 上下文构建过程

```
构建上下文
  ├─ 读取 CLAUDE.md（项目配置）
  ├─ 获取 Git 状态（分支、最近提交）
  ├─ 获取系统信息（OS、Shell、Node 版本）
  ├─ 加载 MCP 服务器信息
  ├─ 加载工具列表
  └─ 组装成完整的系统提示词
```

### 上下文窗口管理

LLM 有上下文窗口限制（如 200K tokens）。当对话太长时，需要：
1. **压缩**：`compact` 命令，总结历史对话
2. **截断**：移除最早的消息
3. **摘要**：将长对话压缩为摘要

---

## 6. 状态管理（state/）

### 管理哪些状态？

`state/AppStateStore.ts` 定义了应用的全局状态：

| 状态 | 说明 |
|------|------|
| `messages` | 对话消息列表 |
| `tools` | 可用工具列表 |
| `commands` | 可用命令列表 |
| `permissionMode` | 权限模式（auto-accept / require-approval） |
| `currentModel` | 当前使用的模型 |
| `mcpServers` | MCP 服务器连接状态 |
| `plugins` | 已加载的插件 |
| `agents` | 子 Agent 列表 |
| `todos` | 任务列表 |
| `sessionHooks` | 会话钩子状态 |

### 状态更新机制

```
用户操作 / LLM 响应
  ↓
触发状态更新（dispatch）
  ↓
Store 更新状态
  ↓
UI 组件重新渲染（Ink/React）
```

---

## 7. 查询引擎（query/）

### 什么是查询引擎？

查询引擎是 Agent 的"思考循环"——它负责：
1. 将用户消息 + 工具列表发送给 LLM
2. 接收 LLM 的响应
3. 如果 LLM 要调用工具，执行工具
4. 将工具结果发回 LLM
5. 重复直到 LLM 给出最终回复

### Agent 循环

```
用户输入
  ↓
┌─────────────────────────────┐
│  发送给 LLM                  │ ←──┐
│  LLM 返回：调用工具 / 最终回复  │    │
└─────────────────────────────┘    │
  ↓                                │
  LLM 要调用工具？                   │
  ├─ 是 → 执行工具 → 结果发回 LLM ──┘
  └─ 否 → 返回最终回复给用户
```

### query/config.ts

定义查询配置：

```typescript
type QueryConfig = {
  sessionId: string;
  gates: {
    streamingToolExecution: boolean;  // 是否流式执行工具
    emitToolUseSummaries: boolean;    // 是否输出工具使用摘要
    fastModeEnabled: boolean;         // 是否启用快速模式
  };
};
```

### 流式输出

LLM 的响应是流式的（逐字输出），查询引擎需要：
1. 实时接收 LLM 的 token
2. 检测是否包含工具调用
3. 如果是工具调用，暂停流式输出，执行工具
4. 将工具结果发回 LLM，继续流式输出

---

## 8. 服务层（services/）

### 服务分类

| 服务 | 作用 |
|------|------|
| `services/api/` | 与 Anthropic API 通信 |
| `services/mcp/` | MCP 服务器管理 |
| `services/oauth/` | OAuth 认证 |
| `services/plugins/` | 插件管理 |
| `services/analytics/` | 分析和遥测 |
| `services/lsp/` | 语言服务器协议 |
| `services/policyLimits/` | 策略限制 |
| `services/remoteManagedSettings/` | 远程管理设置 |
| `services/compact/` | 上下文压缩 |
| `services/SessionMemory/` | 会话记忆 |

### MCP 服务

MCP（Model Context Protocol）是 Claude Code 的扩展协议，允许外部工具作为"插件"接入。

```
Claude Code
  ├─ 内置工具（FileReadTool、BashTool 等）
  └─ MCP 工具（通过 MCP 协议接入的外部工具）
       ├─ Jira MCP
       ├─ Slack MCP
       └─ 自定义 MCP
```

---

## 模块协作流程

一个完整的用户请求处理流程：

```
1. 用户输入："帮我修复 main.ts 里的 bug"
   ↓
2. components/ 接收输入，传递给 query 引擎
   ↓
3. query/ 构建上下文（context/ + state/），发送给 LLM
   ↓
4. LLM 返回：先用 GrepTool 搜索 "bug"
   ↓
5. tools/ 执行 GrepTool，返回搜索结果
   ↓
6. query/ 将结果发回 LLM
   ↓
7. LLM 返回：用 FileReadTool 读取文件
   ↓
8. tools/ 执行 FileReadTool，返回文件内容
   ↓
9. query/ 将内容发回 LLM
   ↓
10. LLM 返回：用 FileEditTool 修复 bug
    ↓
11. tools/ 执行 FileEditTool，检查权限（hooks/）
    ↓
12. 用户确认（components/ 显示确认框）
    ↓
13. 执行编辑，返回结果
    ↓
14. LLM 返回最终回复："已修复 bug"
    ↓
15. components/ 显示回复给用户
```

---

## 总结

| 模块 | 职责 | 核心文件 |
|------|------|----------|
| tools/ | 工具实现（50+ 个） | Tool.ts、tools.ts |
| commands/ | 斜杠命令（100+ 个） | commands.ts |
| skills/ | 自定义技能 | loadSkillsDir.ts |
| hooks/ | UI 钩子（85 个） | useCanUseTool.tsx |
| context/ | 上下文构建 | context.ts |
| state/ | 全局状态管理 | AppStateStore.ts |
| query/ | Agent 循环 | config.ts、transitions.ts |
| services/ | 后端服务 | api/、mcp/、oauth/ |
