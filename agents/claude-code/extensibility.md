# Claude Code — 可扩展性分析

本文档分析 Claude Code 的可扩展性设计，帮助理解 Agent 是如何支持自定义功能的。

---

## 1. 插件系统

### 1.1 插件架构

Claude Code 的插件系统分为 **3 个层级**：

```
插件系统
├─ 内置插件（builtinPlugins.ts）
│   └─ 核心功能，随 Claude Code 一起发布
│
├─ 项目插件（.claude/plugins/）
│   └─ 项目特定的插件
│
└─ 用户插件（~/.claude/plugins/）
    └─ 用户自定义的插件
```

### 1.2 插件加载机制

```typescript
// plugins/builtinPlugins.ts 核心逻辑（简化）
const builtinPlugins = [
  // 内置插件列表
  { name: 'git', enabled: true },
  { name: 'npm', enabled: true },
  { name: 'docker', enabled: false }, // 默认禁用
];

// 插件加载流程
async function loadPlugins(): Promise<Plugin[]> {
  const plugins: Plugin[] = [];
  
  // 1. 加载内置插件
  plugins.push(...builtinPlugins.filter(p => p.enabled));
  
  // 2. 加载项目插件
  const projectPlugins = await loadProjectPlugins();
  plugins.push(...projectPlugins);
  
  // 3. 加载用户插件
  const userPlugins = await loadUserPlugins();
  plugins.push(...userPlugins);
  
  return plugins;
}
```

### 1.3 插件接口

```typescript
// 插件接口定义（简化）
interface Plugin {
  name: string;
  description: string;
  version: string;
  
  // 插件可以提供的功能
  tools?: Tool[];          // 自定义工具
  commands?: Command[];    // 自定义命令
  hooks?: Hook[];          // 自定义钩子
  
  // 生命周期方法
  onActivate?(): Promise<void>;
  onDeactivate?(): Promise<void>;
}
```

---

## 2. 工具注册

### 2.1 工具注册机制

Claude Code 的工具注册分为 **2 种方式**：

| 方式 | 说明 | 示例 |
|------|------|------|
| 静态注册 | 在 tools.ts 中硬编码 | FileReadTool、BashTool |
| 动态注册 | 运行时通过 MCP 或插件注册 | MCP 工具 |

### 2.2 静态注册

```typescript
// tools.ts 核心逻辑（简化）
import { AgentTool } from './tools/AgentTool/AgentTool';
import { BashTool } from './tools/BashTool/BashTool';
import { FileReadTool } from './tools/FileReadTool/FileReadTool';
// ... 导入所有工具

export function getTools(): Tools {
  const tools = [
    AgentTool,
    BashTool,
    FileReadTool,
    // ... 所有工具
  ];
  
  // 根据功能标志过滤
  return tools.filter(tool => {
    if (tool.isEnabled && !tool.isEnabled()) return false;
    return true;
  });
}
```

### 2.3 动态注册（MCP）

MCP 服务器提供的工具会在运行时动态注册：

```typescript
// services/mcp/client.ts 核心逻辑（简化）
function registerMCPTools(serverName: string, mcpTools: MCPTool[]) {
  for (const mcpTool of mcpTools) {
    const tool: Tool = {
      name: `mcp__${serverName}__${mcpTool.name}`,
      description: mcpTool.description,
      inputSchema: mcpTool.inputSchema,
      
      execute: async (input) => {
        return await mcpClient.callTool(serverName, mcpTool.name, input);
      },
    };
    
    // 注册到全局工具列表
    registerTool(tool);
  }
}
```

### 2.4 工具发现

Claude Code 支持 **工具发现**——在运行时发现新的工具：

```typescript
// 核心逻辑（简化）
async function discoverTools(): Promise<Tool[]> {
  const tools: Tool[] = [];
  
  // 1. 扫描 MCP 服务器
  const mcpServers = await loadMCPConfig();
  for (const server of mcpServers) {
    const mcpTools = await mcpClient.listTools(server);
    tools.push(...mcpTools);
  }
  
  // 2. 扫描插件
  const plugins = await loadPlugins();
  for (const plugin of plugins) {
    if (plugin.tools) {
      tools.push(...plugin.tools);
    }
  }
  
  return tools;
}
```

---

## 3. 自定义指令

### 3.1 CLAUDE.md 文件

Claude Code 支持通过 `CLAUDE.md` 文件来自定义行为：

```
项目根目录/
├─ CLAUDE.md              # 项目级配置
├─ .claude/
│   └─ CLAUDE.md          # 项目级配置（另一种位置）
└─ 子目录/
    └─ CLAUDE.md          # 目录级配置
```

### 3.2 CLAUDE.md 内容

```markdown
# 项目配置

## 代码风格
- 使用 TypeScript
- 使用 ESLint
- 使用 Prettier

## 命名规范
- 组件使用 PascalCase
- 函数使用 camelCase
- 常量使用 UPPER_SNAKE_CASE

## 自定义指令
- /review: 代码审查
- /deploy: 部署流程
```

### 3.3 技能系统（Skills）

技能是用户自定义的可复用提示词：

```
~/.claude/skills/           # 全局技能
.claude/skills/             # 项目技能
├─ review.md                # /review 技能
├─ deploy.md                # /deploy 技能
└─ test.md                  # /test 技能
```

**技能定义格式**：

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

### 3.4 技能加载机制

```typescript
// skills/loadSkillsDir.ts 核心逻辑（简化）
async function loadSkillsDir(dir: string): Promise<Skill[]> {
  const skills: Skill[] = [];
  
  // 1. 扫描目录下的所有 .md 文件
  const files = await fs.readdir(dir);
  const mdFiles = files.filter(f => f.endsWith('.md'));
  
  // 2. 解析每个文件
  for (const file of mdFiles) {
    const content = await fs.readFile(path.join(dir, file), 'utf-8');
    
    // 3. 解析 frontmatter
    const { data, content: prompt } = parseFrontmatter(content);
    
    skills.push({
      name: basename(file, '.md'),
      description: data.description,
      tools: data.tools,
      prompt: prompt,
    });
  }
  
  return skills;
}
```

---

## 4. MCP 支持

### 4.1 MCP 协议简介

MCP（Model Context Protocol）是 Claude Code 的扩展协议，允许外部工具作为"插件"接入。

```
Claude Code（MCP 客户端）
  ↓
MCP 协议（JSON-RPC 2.0）
  ↓
MCP 服务器（外部工具）
  ├─ Jira MCP
  ├─ Slack MCP
  ├─ GitHub MCP
  └─ 自定义 MCP
```

### 4.2 MCP 配置

用户可以在 `~/.claude/settings.json` 中配置 MCP 服务器：

```json
{
  "mcpServers": {
    "jira": {
      "command": "npx",
      "args": ["-y", "@anthropic/jira-mcp-server"],
      "env": {
        "JIRA_API_TOKEN": "your-token"
      }
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@anthropic/slack-mcp-server"],
      "env": {
        "SLACK_BOT_TOKEN": "your-token"
      }
    }
  }
}
```

### 4.3 MCP 传输方式

Claude Code 支持 **3 种 MCP 传输方式**：

| 传输方式 | 说明 | 适用场景 |
|----------|------|----------|
| Stdio | 通过 stdin/stdout 通信 | 本地 MCP 服务器 |
| SSE | 通过 Server-Sent Events 通信 | 远程 MCP 服务器 |
| Streamable HTTP | 通过 HTTP 流通信 | 远程 MCP 服务器 |

### 4.4 MCP 工具调用流程

```
用户请求
  ↓
LLM 决定调用 MCP 工具
  ↓
Claude Code 查找 MCP 连接
  ↓
通过 MCP 协议发送请求
  ↓
MCP 服务器执行工具
  ↓
返回结果给 Claude Code
  ↓
Claude Code 将结果发给 LLM
  ↓
LLM 继续处理
```

---

## 5. 扩展性设计亮点

### 5.1 模块化设计

- 工具、命令、技能都是独立的模块
- 可以单独添加、删除、修改
- 不影响其他模块

### 5.2 插件化架构

- 核心功能也是插件
- 可以替换核心功能
- 可以扩展新功能

### 5.3 协议化扩展

- MCP 协议是标准化的
- 任何语言都可以实现 MCP 服务器
- 可以复用社区的 MCP 服务器

### 5.4 配置化驱动

- 大部分功能通过配置文件控制
- 不需要修改源码
- 支持项目级和用户级配置

---

## 总结

| 扩展维度 | 实现方式 | 关键文件 |
|----------|----------|----------|
| 插件系统 | 内置 + 项目 + 用户 | builtinPlugins.ts |
| 工具注册 | 静态 + 动态（MCP） | tools.ts, mcp/client.ts |
| 自定义指令 | CLAUDE.md + Skills | loadSkillsDir.ts |
| MCP 支持 | JSON-RPC 2.0 + 多传输 | mcp/client.ts |
