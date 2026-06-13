# Claude Code — 技术实现细节

本文档深入分析 Claude Code 源码中的关键技术实现，帮助理解 Agent 的"底层是怎么跑起来的"。

---

## 1. LLM 集成方式

### 1.1 API 调用链

Claude Code 通过 Anthropic SDK 与 Claude API 通信，核心调用链：

```
用户输入
  ↓
QueryEngine.ts（Agent 循环控制器）
  ↓
query.ts — 发起 API 请求
  ↓
services/api/claude.ts — 构建请求参数、调用 SDK
  ↓
@anthropic-ai/sdk — HTTP 请求发送到 Anthropic API
  ↓
流式接收响应（SSE）
  ↓
解析响应 → 决定是调用工具还是返回最终回复
```

### 1.2 关键文件

| 文件 | 职责 |
|------|------|
| `src/QueryEngine.ts` | Agent 循环主控制器，管理整个对话流程 |
| `src/query.ts` | 单次 API 调用逻辑，处理流式响应 |
| `src/services/api/claude.ts` | API 参数构建、SDK 调用、响应解析 |
| `src/services/api/client.ts` | HTTP 客户端配置（代理、超时等） |
| `src/services/api/withRetry.ts` | 重试逻辑（指数退避、错误分类） |

### 1.3 流式响应处理

Claude Code 使用 **SSE（Server-Sent Events）** 流式接收 LLM 响应：

```typescript
// services/api/claude.ts 核心逻辑（简化）
const stream = await client.messages.stream({
  model: model,
  max_tokens: maxTokens,
  system: systemPrompt,
  messages: messages,
  tools: toolSchemas,
  // ... 其他参数
});

// 逐块处理流式响应
for await (const event of stream) {
  switch (event.type) {
    case 'content_block_start':
      // 开始新的内容块（文本或工具调用）
      break;
    case 'content_block_delta':
      // 增量更新（逐字输出文本，或工具参数的增量）
      break;
    case 'content_block_stop':
      // 内容块完成
      break;
    case 'message_stop':
      // 整个消息完成
      break;
  }
}
```

**为什么用流式？** 
- 用户能实时看到 LLM 的输出，不用等全部生成完
- 可以在输出过程中提前检测工具调用，减少等待时间

### 1.4 模型选择机制

Claude Code 支持多模型切换：

```typescript
// utils/model/model.ts 核心逻辑（简化）
function getMainLoopModel(): string {
  // 1. 用户通过 /model 命令指定的模型
  const userModel = getUserSelectedModel();
  if (userModel) return userModel;

  // 2. 环境变量指定
  const envModel = process.env.CLAUDE_MODEL;
  if (envModel) return envModel;

  // 3. 默认模型
  return getDefaultSonnetModel();
}
```

支持的模型类型：
- **主模型**：Claude Sonnet（默认）、Claude Opus（高质量）
- **快速模式**：使用更轻量的模型，响应更快
- **小模型**：用于分类器、摘要等辅助任务

---

## 2. Prompt 工程

### 2.1 系统提示词结构

Claude Code 的系统提示词由 **4 个部分** 组成：

```
系统提示词（System Prompt）
│
├─ 静态部分（所有用户共享，硬编码在源码中）
│   ├─ 角色定义："你是一个交互式 Agent..."
│   ├─ 行为规则：如何使用工具、如何回复用户
│   ├─ 安全指令：禁止生成恶意代码、提示注入防护
│   └─ 工具使用说明：每个工具的详细说明
│
├─ 动态系统上下文（每个会话不同）
│   ├─ Git 状态：当前分支、最近提交、文件变更
│   ├─ 系统信息：OS、Shell、Node 版本
│   ├─ 工作目录：当前路径
│   └─ 时间信息：当前日期
│
├─ 用户上下文
│   ├─ CLAUDE.md 内容：项目配置、自定义指令
│   ├─ 记忆文件：~/.claude/memory/ 下的内容
│   └─ 附件：用户上传的图片、文件
│
└─ MCP 服务器信息
    ├─ 已连接的 MCP 服务器列表
    └─ MCP 提供的工具列表
```

### 2.2 系统提示词构建过程

```typescript
// utils/queryContext.ts 核心逻辑（简化）
async function fetchSystemPromptParts(): SystemPromptParts {
  // 1. 获取静态系统提示词
  const staticPrompt = getStaticSystemPrompt();

  // 2. 获取动态系统上下文
  const systemContext = await getSystemContext({
    cwd: getCwd(),
    gitStatus: await getGitStatus(),
    systemInfo: getSystemInfo(),
    time: new Date().toISOString(),
  });

  // 3. 获取用户上下文
  const userContext = await getUserContext({
    claudeMd: await readClaudeMd(),
    memory: await loadMemoryPrompt(),
    attachments: getAttachments(),
  });

  // 4. 获取 MCP 信息
  const mcpContext = await getMCPContext();

  return { staticPrompt, systemContext, userContext, mcpContext };
}
```

### 2.3 工具描述注入

每个工具的描述（description）会被注入到系统提示词中，告诉 LLM 有哪些工具可用：

```typescript
// Tool.ts 核心接口（简化）
interface Tool {
  name: string;           // 工具名称
  description: string;    // 工具描述（告诉 LLM 这个工具能做什么）
  inputSchema: JSONSchema; // 输入参数的 JSON Schema
  
  // 动态描述（根据上下文生成）
  description(input, context): Promise<string>;
}
```

**关键设计**：工具描述不是静态的，而是根据当前输入动态生成。例如，BashTool 的描述会根据当前操作系统和 Shell 类型变化。

### 2.4 上下文压缩策略

当对话太长时，Claude Code 使用多种策略压缩上下文：

| 策略 | 触发条件 | 实现方式 |
|------|----------|----------|
| Auto Compact | Token 数超过阈值 | 自动总结历史对话 |
| Manual Compact | 用户输入 `/compact` | 手动触发压缩 |
| Micro Compact | 每轮对话后 | 移除冗余的工具调用结果 |
| Reactive Compact | 接近上下文窗口限制 | 紧急压缩，移除最早的消息 |
| Snip Compact | 特定模式匹配 | 移除大块的代码输出 |

**Auto Compact 实现**（`services/compact/autoCompact.ts`）：

```typescript
// 核心逻辑（简化）
function getAutoCompactThreshold(model: string): number {
  const contextWindow = getContextWindowForModel(model);
  const reservedForOutput = 20_000; // 预留给输出的 token
  const buffer = 13_000; // 缓冲区
  return contextWindow - reservedForOutput - buffer;
}

// 检查是否需要压缩
function shouldAutoCompact(messages: Message[], model: string): boolean {
  const tokenCount = estimateTokenCount(messages);
  const threshold = getAutoCompactThreshold(model);
  return tokenCount > threshold;
}
```

---

## 3. RAG 实现

### 3.1 Claude Code 的 RAG 策略

Claude Code **没有使用传统的向量数据库 RAG**，而是采用了一种更轻量的"上下文注入"策略：

```
传统 RAG：文档 → 向量化 → 语义检索 → 注入上下文
Claude Code：配置文件 + Git 状态 + 记忆文件 → 直接注入上下文
```

### 3.2 上下文来源

| 来源 | 内容 | 注入方式 |
|------|------|----------|
| CLAUDE.md | 项目配置、自定义指令 | 每次对话都注入 |
| ~/.claude/memory/ | 用户记忆文件 | 每次对话都注入 |
| Git 状态 | 分支、最近提交、文件变更 | 每次对话都注入 |
| MCP 资源 | 外部数据源 | 按需注入 |
| 附件 | 用户上传的图片、文件 | 当次对话注入 |

### 3.3 技能搜索（实验性）

Claude Code 有一个实验性的技能搜索功能（`services/skillSearch/`）：

```typescript
// services/skillSearch/prefetch.ts 核心逻辑（简化）
async function prefetchRelevantSkills(query: string): Promise<Skill[]> {
  // 1. 获取所有可用技能
  const allSkills = await loadAllSkills();
  
  // 2. 根据用户查询匹配相关技能
  const relevantSkills = allSkills.filter(skill => 
    skill.description.includes(query) || 
    skill.keywords.some(kw => query.includes(kw))
  );
  
  return relevantSkills;
}
```

---

## 4. 记忆管理

### 4.1 记忆类型

Claude Code 有 **3 种记忆机制**：

| 类型 | 存储位置 | 生命周期 | 用途 |
|------|----------|----------|------|
| 会话记忆 | 内存（AppState） | 单次会话 | 当前对话的消息历史 |
| 文件记忆 | ~/.claude/memory/ | 持久化 | 跨会话的用户偏好、项目信息 |
| 会话存储 | ~/.claude/projects/ | 持久化 | 对话历史记录（可回放） |

### 4.2 文件记忆系统（memdir）

```typescript
// memdir/memdir.ts 核心逻辑（简化）
async function loadMemoryPrompt(): Promise<string> {
  const memoryDir = path.join(getHomeDir(), '.claude', 'memory');
  
  // 1. 读取所有记忆文件
  const files = await fs.readdir(memoryDir);
  const memoryFiles = files.filter(f => f.endsWith('.md'));
  
  // 2. 读取每个文件的内容
  const memories = await Promise.all(
    memoryFiles.map(async f => ({
      name: f,
      content: await fs.readFile(path.join(memoryDir, f), 'utf-8'),
    }))
  );
  
  // 3. 构建记忆提示词
  const memoryPrompt = memories
    .map(m => `## ${m.name}\n${m.content}`)
    .join('\n\n');
  
  return memoryPrompt;
}
```

### 4.3 会话存储

每次对话都会被记录到 `~/.claude/projects/` 目录下：

```
~/.claude/projects/
├── <project-hash>/
│   ├── <session-id>.jsonl    # 对话记录（JSONL 格式）
│   └── ...
```

JSONL 格式的优势：
- 可以逐行追加，不需要读取整个文件
- 可以流式处理，内存友好
- 可以按需回放特定消息

---

## 5. 错误处理

### 5.1 错误分类

Claude Code 将错误分为 **4 大类**：

```
错误类型
├─ API 错误（services/api/errors.ts）
│   ├─ 连接错误（APIConnectionError）
│   ├─ 超时错误（APIConnectionTimeoutError）
│   ├─ 速率限制（429 Too Many Requests）
│   ├─ 服务不可用（529 Service Overloaded）
│   └─ 认证错误（401 Unauthorized）
│
├─ 工具错误（各工具内部处理）
│   ├─ 文件不存在（FileReadTool）
│   ├─ 权限不足（BashTool）
│   ├─ 命令执行失败（BashTool）
│   └─ 输入验证失败（所有工具）
│
├─ 上下文错误
│   ├─ 上下文窗口超限（Prompt too long）
│   └─ Token 数估算错误
│
└─ 用户操作错误
    ├─ 中断请求（Ctrl+C）
    └─ 无效输入
```

### 5.2 重试机制（withRetry.ts）

```typescript
// services/api/withRetry.ts 核心逻辑（简化）
async function withRetry<T>(
  fn: () => Promise<T>,
  options: { maxRetries: number; baseDelay: number }
): Promise<T> {
  let lastError: Error;
  
  for (let attempt = 0; attempt <= options.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      
      // 判断是否可重试
      if (!isRetryableError(error)) {
        throw error;
      }
      
      // 指数退避：500ms, 1000ms, 2000ms, 4000ms...
      const delay = options.baseDelay * Math.pow(2, attempt);
      await sleep(delay);
    }
  }
  
  throw lastError;
}
```

**重试策略**：
- **429（速率限制）**：指数退避重试，最多 10 次
- **529（服务过载）**：指数退避重试，最多 3 次
- **连接错误**：立即重试，最多 3 次
- **认证错误**：不重试，提示用户重新登录

### 5.3 工具错误处理

每个工具都有自己的错误处理逻辑：

```typescript
// 工具执行流程（简化）
async function executeTool(tool: Tool, input: unknown): Promise<ToolResult> {
  try {
    // 1. 输入验证
    const validation = tool.validateInput(input);
    if (!validation.result) {
      return { error: validation.message };
    }
    
    // 2. 权限检查
    const permission = await checkPermission(tool, input);
    if (!permission.allowed) {
      return { error: 'Permission denied' };
    }
    
    // 3. 执行工具
    const result = await tool.execute(input, context);
    return result;
    
  } catch (error) {
    // 4. 错误处理
    return { error: errorMessage(error) };
  }
}
```

---

## 6. 流式输出实现

### 6.1 流式架构

```
Anthropic API（SSE 流）
  ↓
services/api/claude.ts（接收流式事件）
  ↓
query.ts（解析事件，构建消息）
  ↓
QueryEngine.ts（更新状态）
  ↓
components/（Ink/React 渲染）
  ↓
终端输出
```

### 6.2 流式事件类型

```typescript
// 流式事件类型
type StreamEvent =
  | { type: 'text_delta'; text: string }           // 文本增量
  | { type: 'tool_use_start'; name: string }       // 工具调用开始
  | { type: 'tool_use_delta'; input: string }      // 工具参数增量
  | { type: 'tool_use_stop' }                      // 工具调用完成
  | { type: 'thinking_delta'; thinking: string }   // 思考过程增量
  | { type: 'message_stop' }                       // 消息完成
```

### 6.3 流式工具执行

Claude Code 支持**流式工具执行**——在工具参数还没完全生成时就开始执行：

```typescript
// query.ts 核心逻辑（简化）
async function* processStream(stream: AsyncIterable<StreamEvent>) {
  let currentToolUse: ToolUseBlock | null = null;
  let toolInput = '';
  
  for await (const event of stream) {
    switch (event.type) {
      case 'tool_use_start':
        currentToolUse = { name: event.name, input: '' };
        break;
        
      case 'tool_use_delta':
        toolInput += event.input;
        // 如果工具参数已经足够，可以提前开始执行
        if (canStartExecution(toolInput)) {
          yield { type: 'start_tool', tool: currentToolUse, input: toolInput };
        }
        break;
        
      case 'tool_use_stop':
        // 工具参数完整，执行工具
        yield { type: 'execute_tool', tool: currentToolUse, input: toolInput };
        currentToolUse = null;
        toolInput = '';
        break;
    }
  }
}
```

### 6.4 Ink/React 终端渲染

Claude Code 使用 **Ink**（React 终端 UI 框架）来渲染流式输出：

```typescript
// components/ 核心逻辑（简化）
function StreamOutput({ messages }: { messages: Message[] }) {
  return (
    <Box flexDirection="column">
      {messages.map((msg, i) => (
        <Text key={i}>
          {msg.type === 'text' && msg.text}
          {msg.type === 'tool_use' && <ToolUseDisplay tool={msg} />}
          {msg.type === 'tool_result' && <ToolResultDisplay result={msg} />}
        </Text>
      ))}
    </Box>
  );
}
```

---

## 7. MCP 协议实现

### 7.1 MCP 客户端架构

```
Claude Code
  ↓
services/mcp/client.ts（MCP 客户端）
  ↓
@modelcontextprotocol/sdk（MCP SDK）
  ↓
传输层
├─ StdioClientTransport（本地进程，通过 stdin/stdout 通信）
├─ SSEClientTransport（远程服务器，通过 SSE 通信）
└─ StreamableHTTPClientTransport（远程服务器，通过 HTTP 流通信）
```

### 7.2 MCP 连接管理

```typescript
// services/mcp/client.ts 核心逻辑（简化）
class MCPClient {
  private connections: Map<string, MCPServerConnection> = new Map();
  
  async connect(serverConfig: MCPServerConfig): Promise<void> {
    // 1. 选择传输方式
    const transport = this.createTransport(serverConfig);
    
    // 2. 创建 MCP 客户端
    const client = new Client({
      name: 'claude-code',
      version: '1.0.0',
    });
    
    // 3. 连接到服务器
    await client.connect(transport);
    
    // 4. 获取服务器提供的工具列表
    const tools = await client.listTools();
    
    // 5. 注册到工具系统
    this.registerTools(serverConfig.name, tools);
    
    // 6. 保存连接
    this.connections.set(serverConfig.name, { client, transport, tools });
  }
  
  async callTool(serverName: string, toolName: string, input: unknown): Promise<ToolResult> {
    const connection = this.connections.get(serverName);
    if (!connection) {
      throw new Error(`MCP server ${serverName} not connected`);
    }
    
    // 调用远程工具
    const result = await connection.client.callTool({
      name: toolName,
      arguments: input,
    });
    
    return result;
  }
}
```

### 7.3 MCP 工具注册

MCP 服务器提供的工具会被动态注册到 Claude Code 的工具系统中：

```typescript
// 核心逻辑（简化）
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

---

## 8. Hook 系统实现

### 8.1 Hook 分类

Claude Code 的 Hook 分为 **2 大类**：

| 类型 | 位置 | 用途 |
|------|------|------|
| UI Hooks | `src/hooks/` | React Hooks，用于 UI 状态管理 |
| Permission Hooks | `src/hooks/toolPermission/` | 工具权限检查 |

### 8.2 权限检查流程

```typescript
// hooks/useCanUseTool.tsx 核心逻辑（简化）
async function hasPermissionsToUseTool(
  tool: Tool,
  input: unknown,
  context: ToolUseContext
): Promise<PermissionDecision> {
  // 1. 检查工具自己的权限配置
  const toolPermission = tool.getPermission?.(input);
  if (toolPermission?.behavior === 'allow') {
    return { behavior: 'allow' };
  }
  
  // 2. 检查全局权限模式
  const permissionMode = context.getAppState().permissionMode;
  if (permissionMode === 'auto-accept') {
    return { behavior: 'allow' };
  }
  
  // 3. 检查分类器（自动模式下的安全检查）
  if (permissionMode === 'auto') {
    const classifierResult = await runClassifier(tool, input);
    if (classifierResult.safe) {
      return { behavior: 'allow', decisionReason: { type: 'classifier' } };
    }
  }
  
  // 4. 需要用户确认
  return { behavior: 'ask_user' };
}
```

### 8.3 权限处理器

权限检查有 3 种处理器，根据不同的运行模式选择：

| 处理器 | 文件 | 用途 |
|--------|------|------|
| Interactive Handler | `interactiveHandler.ts` | 交互模式，弹出确认框 |
| Coordinator Handler | `coordinatorHandler.ts` | 协调器模式，由主 Agent 决策 |
| Swarm Worker Handler | `swarmWorkerHandler.ts` | 群体模式，由 Worker 自主决策 |

---

## 9. 关键设计模式总结

| 模式 | 应用场景 | 实现位置 |
|------|----------|----------|
| 流式处理 | LLM 响应、工具执行 | query.ts, claude.ts |
| 观察者模式 | 状态更新、UI 渲染 | state/AppStateStore.ts |
| 策略模式 | 权限检查、错误处理 | hooks/toolPermission/ |
| 工厂模式 | 工具创建、传输层创建 | Tool.ts, mcp/client.ts |
| 装饰器模式 | 重试逻辑、日志记录 | withRetry.ts |
| 异步生成器 | 流式事件处理 | query.ts |

---

## 10. 技术实现亮点

### 10.1 条件编译（Dead Code Elimination）

Claude Code 使用 `feature()` 函数实现条件编译：

```typescript
// 只在启用 REACTIVE_COMPACT 功能时才加载相关代码
const reactiveCompact = feature('REACTIVE_COMPACT')
  ? require('./services/compact/reactiveCompact.js')
  : null;
```

**好处**：
- 未启用的功能不会被打包，减小体积
- 可以通过功能标志控制功能发布

### 10.2 延迟加载（Lazy Loading）

某些模块使用延迟加载，避免启动时加载所有代码：

```typescript
// 延迟加载 React/Ink 组件
const messageSelector = () =>
  require('src/components/MessageSelector.js');
```

### 10.3 推测性分类器（Speculative Classifier）

在自动模式下，Claude Code 会**提前**运行分类器，判断工具调用是否安全：

```typescript
// 在工具参数还没完全生成时就开始分类
const classifierResult = await peekSpeculativeClassifierCheck(toolUseID);
if (classifierResult) {
  // 分类器已经完成，可以直接使用结果
  consumeSpeculativeClassifierCheck(toolUseID);
}
```

**好处**：减少用户等待时间，提高响应速度。

---

## 总结

| 技术领域 | 实现方式 | 关键文件 |
|----------|----------|----------|
| LLM 集成 | Anthropic SDK + SSE 流式 | claude.ts, query.ts |
| Prompt 工程 | 4 层上下文注入 | queryContext.ts, context.ts |
| RAG | 配置文件 + Git 状态注入 | context.ts, memdir.ts |
| 记忆管理 | 文件记忆 + 会话存储 | memdir.ts, sessionStorage.ts |
| 错误处理 | 分类 + 重试 + 降级 | errors.ts, withRetry.ts |
| 流式输出 | SSE + Ink/React | query.ts, components/ |
| MCP 协议 | SDK + 多传输层 | mcp/client.ts |
| 权限系统 | 分类器 + 处理器链 | useCanUseTool.tsx |
