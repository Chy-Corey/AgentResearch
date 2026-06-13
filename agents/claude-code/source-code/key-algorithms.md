# Claude Code — 关键算法与设计模式

本文档深入分析 Claude Code 的核心算法和设计模式，帮助理解 Agent 的工作原理。

---

## 1. Agent 循环（Agent Loop）

### 什么是 Agent 循环？

Agent 循环是 Agent 的"思考-行动"循环。用户提出问题后，Agent 会：
1. 思考（调用 LLM）
2. 行动（调用工具）
3. 观察（获取工具结果）
4. 再思考... 直到完成任务

### 循环流程

```
用户输入
  ↓
┌─────────────────────────────────────────────────────────────┐
│                    Agent 循环开始                              │
│                                                             │
│  1. 构建上下文（系统提示词 + 对话历史 + 工具列表）                │
│  2. 发送给 LLM（Claude API）                                 │
│  3. 接收 LLM 响应                                            │
│     ├─ 如果是"调用工具" → 执行工具 → 将结果加入上下文 → 回到步骤2 │
│     └─ 如果是"最终回复" → 返回给用户                           │
│                                                             │
│                    Agent 循环结束                              │
└─────────────────────────────────────────────────────────────┘
```

### 上下文构建详解

"构建上下文"是 Agent 循环的第一步，也是最关键的一步。发送给 LLM 的内容由 **4 个部分** 组成：

```
发送给 Claude API 的请求
│
├─ system: SystemPrompt          ← 系统提示词（LLM 的"人设"和规则）
│   ├─ 静态部分（所有用户共享）
│   │   ├─ 角色定义："你是一个交互式 Agent，帮助用户完成软件工程任务"
│   │   ├─ 行为规则：如何使用工具、如何回复用户
│   │   ├─ 安全指令：禁止生成恶意代码、提示注入防护
│   │   └─ 工具使用说明：每个工具的详细说明
│   │
│   └─ 动态部分（每个用户/会话不同）
│       ├─ Git 状态：当前分支、最近提交、文件变更
│       ├─ CLAUDE.md 内容：项目配置和自定义指令
│       ├─ MCP 指令：已连接的 MCP 服务器说明
│       └─ 用户偏好：语言、输出风格
│
├─ messages: Message[]           ← 对话历史
│   ├─ 用户消息（UserMessage）
│   │   ├─ 用户输入的文本
│   │   └─ 附件（图片、文件）
│   ├─ 助手消息（AssistantMessage）
│   │   ├─ LLM 的文本回复
│   │   └─ 工具调用请求
│   └─ 工具结果（ToolResult）
│       ├─ 工具执行的输出
│       └─ 错误信息（如果有）
│
└─ tools: Tool[]                 ← 可用工具列表
    ├─ 内置工具（50+ 个）
    │   ├─ FileReadTool、FileEditTool、BashTool 等
    │   └─ 每个工具包含：名称、描述、输入参数 Schema
    └─ MCP 工具（外部工具）
        └─ 通过 MCP 协议接入的外部工具
```

#### 第1部分：系统提示词（System Prompt）

系统提示词定义了 LLM 的"人设"和行为规则。

**来源**：`constants/prompts.ts`

```typescript
// prompts.ts 核心结构（简化）
export async function getSystemPrompt(tools, model): Promise<SystemPrompt> {
  return [
    // ── 静态部分（所有用户共享，可缓存）──
    getSimpleIntroSection(),           // 角色定义
    getSimpleSystemSection(),          // 系统规则
    getSimpleDoingTasksSection(),      // 任务执行规则
    getToolUseSection(tools),          // 工具使用说明
    CYBER_RISK_INSTRUCTION,            // 安全指令

    // ── 动态边界 ──
    SYSTEM_PROMPT_DYNAMIC_BOUNDARY,    // 缓存边界标记

    // ── 动态部分（每个用户/会话不同）──
    getLanguageSection(language),       // 语言偏好
    getOutputStyleSection(style),       // 输出风格
    getMcpInstructionsSection(mcp),     // MCP 指令
    getHooksSection(),                  // 钩子说明
  ];
}
```

**关键设计**：
- 使用 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 标记分隔静态和动态内容
- 静态部分可以全局缓存（节省 Token 成本）
- 动态部分每个会话不同，不能缓存

#### 第2部分：系统上下文（System Context）

系统上下文包含当前环境的信息。

**来源**：`context.ts`

```typescript
// context.ts 核心逻辑（简化）
export const getSystemContext = memoize(async () => {
  // 1. 获取 Git 状态
  const gitStatus = await getGitStatus();
  // 返回：当前分支、主分支、文件变更、最近 5 条提交、用户名

  // 2. 获取系统注入（调试用）
  const injection = getSystemPromptInjection();

  return {
    ...(gitStatus && { gitStatus }),
    ...(injection && { cacheBreaker: injection }),
  };
});
```

**Git 状态包含**：
```
Current branch: main
Main branch: main
Git user: HongyuChen
Status:
M  agents/claude-code/source-code/key-algorithms.md
Recent commits:
abc1234 完成核心模块分析
def5678 补充启动链路说明
```

#### 第3部分：对话历史（Messages）

对话历史是用户和 Agent 之间的所有交互记录。

**来源**：`state/AppStateStore.ts`（存储）+ `utils/messages.ts`（格式化）

```typescript
// 消息类型
type Message =
  | UserMessage        // 用户输入
  | AssistantMessage   // LLM 回复
  | SystemMessage      // 系统消息（如压缩摘要）
  | ToolResultMessage  // 工具执行结果
  | ProgressMessage    // 进度消息
```

**消息流转**：
```
用户输入 "帮我修复 bug"
  → 创建 UserMessage，加入 messages 数组
  → 发送给 LLM
  → LLM 返回 "我来帮你看看"
  → 创建 AssistantMessage，加入 messages 数组
  → LLM 调用 FileReadTool
  → 执行工具，创建 ToolResultMessage，加入 messages 数组
  → 将完整 messages 发回 LLM
  → ...
```

#### 第4部分：工具列表（Tools）

工具列表告诉 LLM 它可以使用哪些工具。

**来源**：`tools.ts`（获取）+ `utils/api.ts`（转换为 API Schema）

```typescript
// tools.ts
export function getTools(): Tools {
  return [BashTool, FileEditTool, FileReadTool, ...].filter(t => t.isEnabled());
}

// utils/api.ts - 将工具转换为 API Schema
async function toolToAPISchema(tool: Tool): Promise<BetaToolUnion> {
  return {
    name: tool.name,                    // "BashTool"
    description: await tool.prompt(),   // "执行终端命令..."
    input_schema: zodToJsonSchema(tool.inputSchema),  // JSON Schema
  };
}
```

**工具 Schema 示例**（发送给 LLM 的格式）：
```json
{
  "name": "BashTool",
  "description": "执行终端命令。用于运行测试、安装依赖、Git 操作等。",
  "input_schema": {
    "type": "object",
    "properties": {
      "command": {
        "type": "string",
        "description": "要执行的终端命令"
      }
    },
    "required": ["command"]
  }
}
```

#### 补充：技能（Skills）如何进入上下文

技能不是作为独立部分加入上下文的，而是**嵌入在 SkillTool 的描述中**。

**加载流程**：
```
启动时
  ↓
loadSkillsDir() 扫描技能目录
  ├─ ~/.claude/skills/（全局技能）
  ├─ .claude/skills/（项目技能）
  └─ 内置技能（bundled/）
  ↓
解析每个技能的 frontmatter（元数据）
  ├─ name: 技能名称
  ├─ description: 技能描述
  ├─ whenToUse: 何时使用
  └─ allowedTools: 允许使用的工具
  ↓
注册为 Command（斜杠命令）
  ↓
构建系统提示词时
  ↓
getSkillToolCommands() 获取所有技能
  ↓
格式化为技能列表，嵌入 SkillTool 的描述中
  ↓
发送给 LLM
```

**技能在上下文中的样子**：

LLM 看到的 SkillTool 描述：
```
SkillTool: 调用用户定义的技能。

可用技能列表：
- /review: 代码审查技能 - 审查 git diff 中的代码变更
- /deploy: 部署技能 - 自动化部署流程
- /test: 测试技能 - 运行测试并分析结果

当用户输入 /review 时，使用 SkillTool({ name: "review" }) 调用该技能。
```

**关键设计**：
- 技能只在 SkillTool 的描述中列出**名称和摘要**（节省 Token）
- 技能的**完整内容**只在调用时才加载（延迟加载）
- 技能列表有 Token 预算限制（占上下文窗口的 1%）
- 每个技能描述最多 250 字符

**代码实现**：
```typescript
// tools/SkillTool/prompt.ts
export function formatCommandsWithinBudget(commands, contextWindowTokens) {
  const budget = getCharBudget(contextWindowTokens);  // 1% 的上下文窗口
  
  return commands
    .map(cmd => `- ${cmd.name}: ${cmd.description}`)
    .filter(desc => desc.length <= MAX_LISTING_DESC_CHARS)  // 最多 250 字符
    .join('\n');
}
```

### 上下文构建流程图

```
query() 被调用
  ↓
┌─ 构建系统提示词 ─────────────────────────────────────────────┐
│  getSystemPrompt(tools, model)                               │
│    ├─ 读取 prompts.ts 中的静态模板                            │
│    ├─ 获取语言偏好、输出风格                                   │
│    ├─ 获取 MCP 指令                                          │
│    └─ 拼接成完整的系统提示词                                   │
└──────────────────────────────────────────────────────────────┘
  ↓
┌─ 构建系统上下文 ─────────────────────────────────────────────┐
│  getSystemContext()                                          │
│    ├─ 执行 git status、git log 等命令                         │
│    └─ 格式化为文本                                           │
└──────────────────────────────────────────────────────────────┘
  ↓
┌─ 获取对话历史 ───────────────────────────────────────────────┐
│  appState.messages                                           │
│    └─ 从状态存储中读取所有消息                                 │
└──────────────────────────────────────────────────────────────┘
  ↓
┌─ 获取工具列表 ───────────────────────────────────────────────┐
│  getTools()                                                  │
│    ├─ 加载所有内置工具                                        │
│    ├─ 加载 MCP 工具                                          │
│    └─ 转换为 API Schema                                      │
└──────────────────────────────────────────────────────────────┘
  ↓
┌─ 组装 API 请求 ─────────────────────────────────────────────┐
│  callModel({                                                 │
│    system: systemPrompt,       // 系统提示词                  │
│    messages: messages,         // 对话历史                    │
│    tools: tools,               // 工具列表                    │
│    max_tokens: maxTokens,      // 最大输出 Token              │
│    model: model,               // 使用的模型                  │
│  })                                                          │
└──────────────────────────────────────────────────────────────┘
  ↓
发送给 Claude API
```

### 对应源码文件

| 上下文部分 | 源码文件 | 核心函数 |
|-----------|----------|----------|
| 系统提示词 | `constants/prompts.ts` | `getSystemPrompt()` |
| 系统上下文 | `context.ts` | `getSystemContext()`、`getGitStatus()` |
| 对话历史 | `state/AppStateStore.ts` | `appState.messages` |
| 工具列表 | `tools.ts` | `getTools()` |
| 工具 Schema | `utils/api.ts` | `toolToAPISchema()` |
| 消息格式化 | `utils/messages.ts` | `createUserMessage()`、`normalizeMessagesForAPI()` |

### 代码实现

`query.ts` 是 Agent 循环的核心实现：

```typescript
// query.ts 核心逻辑（简化）
async function* query(params: QueryParams): AsyncGenerator<StreamEvent> {
  while (true) {
    // 1. 构建上下文
    const messages = buildMessages(params);
    
    // 2. 发送给 LLM（流式）
    const stream = await callModel(messages, tools);
    
    // 3. 接收流式响应
    let assistantMessage = '';
    for await (const event of stream) {
      if (event.type === 'text') {
        assistantMessage += event.text;
        yield { type: 'text', text: event.text };  // 实时输出给用户
      }
      if (event.type === 'tool_use') {
        // 4. LLM 要调用工具
        const result = await executeTool(event.tool, event.input);
        // 5. 将工具结果加入上下文
        messages.push({ role: 'user', content: result });
        break;  // 继续循环，让 LLM 看到工具结果
      }
    }
    
    // 6. 如果 LLM 没有调用工具，说明是最终回复
    if (!hasToolUse) {
      yield { type: 'done', message: assistantMessage };
      return;
    }
  }
}
```

### 关键设计

- **流式处理**：LLM 的响应是逐 token 流式返回的，不是一次性返回
- **生成器模式**：使用 `AsyncGenerator` 实时 yield 事件，UI 可以边收边显示
- **循环退出条件**：LLM 不再调用工具时退出循环

---

## 2. 工具调用链路（Tool Invocation Chain）

### 完整调用链

```
LLM 返回工具调用请求
  ↓
query.ts 接收 ToolUseBlock
  ↓
工具调度（toolOrchestration.ts）
  ├─ 判断是否可并发执行
  ├─ 只读工具 → 并发执行
  └─ 写入工具 → 串行执行
  ↓
权限检查（useCanUseTool.tsx）
  ├─ 检查权限规则（permissions.ts）
  ├─ 如果需要确认 → 弹出确认框
  └─ 如果允许 → 继续执行
  ↓
工具执行（toolExecution.ts）
  ├─ 验证输入（validateInput）
  ├─ 执行工具（execute）
  └─ 返回结果
  ↓
结果处理
  ├─ 将结果加入消息历史
  ├─ 更新上下文
  └─ 继续 Agent 循环
```

### 工具调度策略

`toolOrchestration.ts` 实现了智能调度：

```typescript
// 工具调度核心逻辑（简化）
async function* runTools(toolUseMessages, canUseTool, context) {
  // 1. 将工具调用分批
  const batches = partitionToolCalls(toolUseMessages, context);
  
  for (const batch of batches) {
    if (batch.isConcurrencySafe) {
      // 2a. 只读工具：并发执行
      yield* runToolsConcurrently(batch.blocks, canUseTool, context);
    } else {
      // 2b. 写入工具：串行执行
      yield* runToolsSerially(batch.blocks, canUseTool, context);
    }
  }
}
```

**分批策略**：
- 连续的只读工具（如多个 FileReadTool）→ 并发执行
- 写入工具（如 FileEditTool、BashTool）→ 串行执行，避免冲突

### 并发控制

`StreamingToolExecutor.ts` 实现了流式工具执行：

```typescript
// 流式工具执行器（简化）
class StreamingToolExecutor {
  private tools: TrackedTool[] = [];
  
  // 添加工具到执行队列
  addTool(block: ToolUseBlock, assistantMessage: AssistantMessage) {
    const tool = findToolByName(this.toolDefinitions, block.name);
    this.tools.push({
      id: block.id,
      block,
      status: 'queued',
      isConcurrencySafe: tool.isConcurrencySafe(input),
    });
    
    // 如果可以并发，立即开始执行
    if (this.canStartConcurrently()) {
      this.startExecution();
    }
  }
  
  // 获取结果（按顺序返回）
  async* getRemainingResults() {
    for (const tool of this.tools) {
      if (tool.status === 'completed') {
        yield* tool.results;
      }
    }
  }
}
```

**关键设计**：
- 工具按接收顺序执行，结果按顺序返回
- 只读工具可以并发，写入工具必须串行
- 使用 `AbortController` 支持取消

---

## 3. 权限控制（Permission Control）

### 权限模型

Claude Code 有三层权限控制：

```
┌─────────────────────────────────────────┐
│  第1层：权限模式（PermissionMode）         │
│  - default：默认模式，需要确认高危操作      │
│  - auto-accept：自动接受所有操作           │
│  - plan：规划模式，只读不写                │
└─────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────┐
│  第2层：权限规则（PermissionRule）         │
│  - 工具级别的 allow/deny 规则            │
│  - 路径级别的 allow/deny 规则            │
│  - 配置来源：项目/全局/会话               │
└─────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────┐
│  第3层：用户确认（User Confirmation）      │
│  - 弹出确认框，显示将要执行的操作          │
│  - 用户选择：允许/拒绝/始终允许            │
└─────────────────────────────────────────┘
```

### 权限检查流程

`permissions.ts` 实现了权限检查：

```typescript
// 权限检查核心逻辑（简化）
async function hasPermissionsToUseTool(tool, input, context) {
  // 1. 检查权限模式
  if (context.permissionMode === 'auto-accept') {
    return { behavior: 'allow' };
  }
  
  // 2. 检查权限规则
  const rule = findMatchingRule(tool, input, context);
  if (rule) {
    if (rule.value === 'allow') {
      return { behavior: 'allow' };
    }
    if (rule.value === 'deny') {
      return { behavior: 'deny', reason: rule.reason };
    }
  }
  
  // 3. 需要用户确认
  return { behavior: 'ask', description: tool.description(input) };
}
```

### 权限规则来源

| 来源 | 优先级 | 说明 |
|------|--------|------|
| 会话级 | 最高 | 用户在当前会话中设置的"始终允许" |
| 项目级 | 中 | `.claude/settings.json` 中的规则 |
| 全局级 | 最低 | `~/.claude/settings.json` 中的规则 |

### BashTool 的特殊权限

BashTool 有额外的安全检查：

```typescript
// bashPermissions.ts 核心逻辑（简化）
async function checkBashPermissions(command, input) {
  // 1. 检查是否在沙箱中
  if (shouldUseSandbox()) {
    return { behavior: 'allow' };  // 沙箱中安全
  }
  
  // 2. 检查危险命令
  if (isDangerousCommand(command)) {
    return { behavior: 'deny', reason: '危险命令' };
  }
  
  // 3. 检查命令分类器（AI 判断）
  const classifierResult = await classifyCommand(command);
  if (classifierResult.isDangerous) {
    return { behavior: 'ask', description: '高危命令，是否执行？' };
  }
  
  return { behavior: 'allow' };
}
```

**安全机制**：
- 沙箱隔离（Docker/进程隔离）
- 危险命令检测（`rm -rf`、`curl | sh` 等）
- AI 分类器（用小模型判断命令是否危险）

---

## 4. 流式输出（Streaming Output）

### 流式处理流程

```
LLM API 返回流式响应
  ↓
claude.ts 接收 Stream
  ↓
逐 token 解析
  ├─ 文本 token → 实时显示给用户
  ├─ 工具调用 token → 缓存，等待完整调用
  └─ 思考 token → 显示思考过程
  ↓
工具调用完成
  ├─ 执行工具
  └─ 将结果发回 LLM
  ↓
继续接收流式响应
```

### 流式事件类型

```typescript
// 流式事件类型
type StreamEvent =
  | { type: 'text'; text: string }           // 文本输出
  | { type: 'thinking'; text: string }        // 思考过程
  | { type: 'tool_use'; tool: string; input: any }  // 工具调用
  | { type: 'tool_result'; result: any }      // 工具结果
  | { type: 'error'; error: string }          // 错误
  | { type: 'done' };                         // 完成
```

### 终端渲染

使用 **Ink**（React 终端 UI 框架）渲染：

```typescript
// REPL.tsx 核心逻辑（简化）
function REPL({ messages, isStreaming }) {
  return (
    <Box flexDirection="column">
      {/* 消息列表 */}
      {messages.map(msg => (
        <Message key={msg.id} message={msg} />
      ))}
      
      {/* 流式输出指示器 */}
      {isStreaming && <Spinner />}
      
      {/* 输入框 */}
      <Input onSubmit={handleSubmit} />
    </Box>
  );
}
```

---

## 5. 上下文压缩（Context Compaction）

### 问题

LLM 有上下文窗口限制（如 200K tokens）。当对话太长时，需要压缩。

### 压缩策略

```
对话历史太长？
  ↓
┌─────────────────────────────────────────┐
│  自动压缩（autoCompact）                  │
│  - 检测 token 数量                       │
│  - 如果接近上限，自动触发压缩              │
│  - 使用小模型总结历史对话                  │
│  - 保留关键信息，删除冗余                  │
└─────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────┐
│  微压缩（microcompact）                   │
│  - 更轻量的压缩                          │
│  - 只压缩最近的几条消息                   │
│  - 保留上下文连续性                       │
└─────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────┐
│  手动压缩（/compact 命令）                │
│  - 用户主动触发                          │
│  - 完整总结整个对话                       │
└─────────────────────────────────────────┘
```

### 压缩实现

```typescript
// autoCompact.ts 核心逻辑（简化）
async function autoCompactIfNeeded(messages, context) {
  const tokenCount = estimateTokenCount(messages);
  const maxTokens = getModelMaxTokens(context.model);
  
  if (tokenCount > maxTokens * 0.8) {
    // 接近上限，触发压缩
    const summary = await summarizeMessages(messages, context.model);
    return [
      createSystemMessage({ content: summary }),
      ...getRecentMessages(messages, 5),  // 保留最近 5 条
    ];
  }
  
  return messages;  // 不需要压缩
}
```

---

## 6. 设计模式（Design Patterns）

### 6.1 策略模式（Strategy Pattern）

工具系统使用策略模式，每个工具都是一个策略：

```typescript
// Tool 接口
interface Tool {
  name: string;
  description: string;
  inputSchema: JSONSchema;
  execute(input: any, context: ToolUseContext): Promise<ToolResult>;
}

// 不同工具实现不同策略
class FileReadTool implements Tool { /* ... */ }
class BashTool implements Tool { /* ... */ }
class GrepTool implements Tool { /* ... */ }
```

**好处**：新增工具只需实现接口，不需要修改核心代码。

### 6.2 观察者模式（Observer Pattern）

状态管理使用观察者模式：

```typescript
// Store 核心逻辑
class Store<T> {
  private listeners: Set<(state: T) => void> = new Set();
  
  subscribe(listener: (state: T) => void) {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }
  
  setState(newState: T) {
    this.state = newState;
    // 通知所有观察者
    this.listeners.forEach(listener => listener(newState));
  }
}
```

**好处**：UI 组件可以订阅状态变化，自动更新。

### 6.3 责任链模式（Chain of Responsibility）

权限检查使用责任链模式：

```
权限检查请求
  ↓
第1个处理器：PermissionMode 检查
  ├─ 如果 auto-accept → 直接允许
  └─ 否则 → 传递给下一个
  ↓
第2个处理器：PermissionRule 检查
  ├─ 如果有匹配规则 → 按规则处理
  └─ 否则 → 传递给下一个
  ↓
第3个处理器：用户确认
  ├─ 用户允许 → 允许
  └─ 用户拒绝 → 拒绝
```

**好处**：每个处理器只关心自己的逻辑，易于扩展。

### 6.4 生成器模式（Generator Pattern）

查询引擎使用生成器模式：

```typescript
// 使用 AsyncGenerator 实时 yield 事件
async function* query(params): AsyncGenerator<StreamEvent> {
  while (true) {
    const stream = await callModel(messages);
    for await (const event of stream) {
      yield event;  // 实时输出给 UI
    }
  }
}

// UI 消费生成器
for await (const event of query(params)) {
  if (event.type === 'text') {
    displayText(event.text);
  }
}
```

**好处**：支持流式处理，UI 可以边收边显示，不需要等待完整响应。

### 6.5 工厂模式（Factory Pattern）

工具注册使用工厂模式：

```typescript
// tools.ts 工厂函数
export function getTools(): Tools {
  const tools = [
    AgentTool,      // 工具类本身就是工厂
    BashTool,
    FileEditTool,
    // ...
  ];
  
  return tools.filter(tool => tool.isEnabled());
}

// 工具类是工厂
class BashTool implements Tool {
  static create(): BashTool {
    return new BashTool();
  }
}
```

### 6.6 依赖注入（Dependency Injection）

查询引擎使用依赖注入：

```typescript
// deps.ts 定义依赖
type QueryDeps = {
  callModel: typeof queryModelWithStreaming;
  microcompact: typeof microcompactMessages;
  autocompact: typeof autoCompactIfNeeded;
  uuid: () => string;
};

// 生产环境使用真实依赖
function productionDeps(): QueryDeps {
  return {
    callModel: queryModelWithStreaming,
    microcompact: microcompactMessages,
    autocompact: autoCompactIfNeeded,
    uuid: randomUUID,
  };
}

// 测试环境可以注入假依赖
const testDeps: QueryDeps = {
  callModel: mockCallModel,
  microcompact: mockMicrocompact,
  autocompact: mockAutocompact,
  uuid: () => 'test-uuid',
};
```

**好处**：便于测试，可以轻松替换依赖。

---

## 7. 并发控制（Concurrency Control）

### 工具并发策略

```
LLM 返回多个工具调用
  ↓
分批处理（partitionToolCalls）
  ├─ 只读工具（FileReadTool、GrepTool）→ 并发执行
  └─ 写入工具（FileEditTool、BashTool）→ 串行执行
```

### 并发限制

```typescript
// 最大并发数
function getMaxToolUseConcurrency(): number {
  return parseInt(process.env.CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || '10', 10);
}
```

### AbortController 机制

```typescript
// 创建子 AbortController
const siblingAbortController = createChildAbortController(
  toolUseContext.abortController
);

// 如果一个工具出错，取消其他并发工具
if (error) {
  siblingAbortController.abort();  // 取消兄弟工具
  // 注意：不会取消父级，Agent 循环继续
}
```

---

## 8. 错误处理（Error Handling）

### 错误分类

| 错误类型 | 处理方式 |
|----------|----------|
| API 错误 | 重试（指数退避） |
| 工具执行错误 | 返回错误信息给 LLM，让 LLM 决定下一步 |
| 权限拒绝 | 告知 LLM 权限不足 |
| 用户中断 | 清理状态，退出循环 |
| 上下文过长 | 触发压缩，重试 |

### 重试机制

```typescript
// withRetry.ts 核心逻辑（简化）
async function withRetry<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (isRetryableError(error) && i < maxRetries - 1) {
        await sleep(Math.pow(2, i) * 1000);  // 指数退避
        continue;
      }
      throw error;
    }
  }
}
```

---

## 总结

| 算法/模式 | 作用 | 核心文件 |
|-----------|------|----------|
| Agent 循环 | 多步任务执行 | query.ts |
| 工具调用链路 | 工具调度和执行 | toolOrchestration.ts |
| 权限控制 | 三层权限检查 | permissions.ts |
| 流式输出 | 实时显示 LLM 响应 | claude.ts |
| 上下文压缩 | 处理长对话 | autoCompact.ts |
| 策略模式 | 工具系统设计 | Tool.ts |
| 观察者模式 | 状态管理 | store.ts |
| 生成器模式 | 流式处理 | query.ts |
| 依赖注入 | 可测试性 | deps.ts |
