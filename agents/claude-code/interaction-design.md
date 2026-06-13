# Claude Code — 交互设计分析

本文档分析 Claude Code 的交互设计，帮助理解 Agent 是如何与用户"对话"的。

---

## 1. CLI 设计

### 1.1 启动方式

Claude Code 支持多种启动方式：

| 方式 | 命令 | 用途 |
|------|------|------|
| 交互式 REPL | `claude` | 进入对话界面，持续交互 |
| 单次执行 | `claude -p "问题"` | 执行一次就退出 |
| 管道模式 | `echo "问题" \| claude` | 从 stdin 读取输入 |
| SDK 模式 | `claude --sdk` | 以编程方式调用 |

### 1.2 终端 UI 框架

Claude Code 使用 **Ink**（React 终端 UI 框架）来渲染界面：

```
Ink（React 终端渲染引擎）
  ↓
components/（UI 组件）
  ├─ App.tsx — 主应用组件
  ├─ StreamOutput.tsx — 流式输出显示
  ├─ PermissionRequest.tsx — 权限确认对话框
  ├─ TodoDisplay.tsx — 任务列表显示
  └─ ...
```

**为什么用 Ink？**
- 可以用 React 的组件化思想来构建终端 UI
- 支持状态管理，界面可以动态更新
- 支持键盘事件处理（快捷键、Vim 模式等）

### 1.3 快捷键系统

Claude Code 有完整的快捷键系统（`keybindings/`）：

| 快捷键 | 功能 |
|--------|------|
| `Enter` | 发送消息 |
| `Ctrl+C` | 中断当前操作 |
| `Ctrl+D` | 退出 |
| `↑/↓` | 浏览历史输入 |
| `Tab` | 自动补全 |
| `Esc` | 取消/返回 |

**Vim 模式**（`vim/`）：
- 支持 Vim 的移动命令（hjkl、w、b、e 等）
- 支持 Vim 的操作命令（d、y、p 等）
- 支持 Vim 的文本对象（iw、aw、i"、a" 等）

### 1.4 自定义键绑定

用户可以在 `~/.claude/keybindings.json` 中自定义快捷键：

```json
{
  "bindings": [
    { "key": "ctrl+shift+t", "action": "toggleTheme" },
    { "key": "ctrl+shift+m", "action": "switchModel" }
  ]
}
```

---

## 2. IDE 集成

### 2.1 VS Code 集成

Claude Code 可以作为 VS Code 扩展运行：

```
VS Code 扩展
  ↓
src/remote/（远程通信）
  ↓
Claude Code CLI（后端）
```

**集成功能**：
- 在 VS Code 终端中运行 Claude Code
- 可以将代码选区发送给 Claude Code
- 可以在 VS Code 中显示差异对比
- 支持 IDE 命令面板调用 Claude Code

### 2.2 JetBrains 集成

同样支持 JetBrains 系列 IDE（IntelliJ、WebStorm 等）。

### 2.3 桌面应用

Claude Code 有独立的桌面应用（Electron）：

```
Electron 桌面应用
  ↓
src/entrypoints/desktop.ts
  ↓
Claude Code CLI（后端）
```

### 2.4 Web 应用

Claude Code 也可以在浏览器中运行（claude.ai/code）：

```
Web 应用
  ↓
src/entrypoints/web.ts
  ↓
Claude Code CLI（后端）
```

---

## 3. 对话管理

### 3.1 消息类型

Claude Code 支持多种消息类型：

| 类型 | 说明 | 示例 |
|------|------|------|
| UserMessage | 用户输入的消息 | "帮我修复这个 bug" |
| AssistantMessage | LLM 的回复 | "我来看看代码..." |
| SystemMessage | 系统消息 | 错误信息、状态更新 |
| ToolUseMessage | 工具调用消息 | 调用 BashTool 执行命令 |
| ToolResultMessage | 工具执行结果 | 命令输出 |
| ProgressMessage | 进度消息 | 任务执行进度 |

### 3.2 消息队列

Claude Code 使用消息队列来管理用户输入：

```typescript
// utils/messageQueueManager.ts 核心逻辑（简化）
const messageQueue: QueuedMessage[] = [];

function enqueueMessage(message: QueuedMessage): void {
  // 按优先级插入队列
  const index = messageQueue.findIndex(m => m.priority < message.priority);
  if (index === -1) {
    messageQueue.push(message);
  } else {
    messageQueue.splice(index, 0, message);
  }
}

function dequeueMessage(): QueuedMessage | undefined {
  return messageQueue.shift();
}
```

### 3.3 中断处理

用户可以随时按 `Ctrl+C` 中断当前操作：

```typescript
// utils/abortController.ts 核心逻辑（简化）
function createAbortController(): AbortController {
  const controller = new AbortController();
  
  // 监听 Ctrl+C
  process.on('SIGINT', () => {
    controller.abort();
  });
  
  return controller;
}
```

---

## 4. 权限控制

### 4.1 权限模式

Claude Code 有 **3 种权限模式**：

| 模式 | 说明 | 安全级别 |
|------|------|----------|
| Default | 每次危险操作都询问用户 | 最高 |
| Auto-accept | 自动批准所有操作 | 最低 |
| Auto | 使用分类器自动判断，危险操作才询问 | 中等 |

### 4.2 权限检查流程

```
工具调用请求
  ↓
1. 检查工具自己的权限配置
   ├─ allow → 直接执行
   └─ 继续检查
  ↓
2. 检查全局权限模式
   ├─ auto-accept → 直接执行
   └─ 继续检查
  ↓
3. 检查权限规则（用户配置的规则）
   ├─ 匹配 allow 规则 → 直接执行
   ├─ 匹配 deny 规则 → 拒绝执行
   └─ 继续检查
  ↓
4. 运行分类器（auto 模式）
   ├─ 安全 → 直接执行
   └─ 危险 → 弹出确认框
  ↓
5. 用户确认
   ├─ 允许 → 执行
   └─ 拒绝 → 取消
```

### 4.3 权限规则

用户可以在 `~/.claude/settings.json` 中配置权限规则：

```json
{
  "permissions": {
    "rules": [
      {
        "tool": "BashTool",
        "pattern": "git *",
        "action": "allow"
      },
      {
        "tool": "BashTool",
        "pattern": "rm -rf *",
        "action": "deny"
      }
    ]
  }
}
```

### 4.4 分类器（Auto 模式）

在 Auto 模式下，Claude Code 使用分类器来判断操作是否安全：

```typescript
// utils/permissions/classifierDecision.ts 核心逻辑（简化）
async function classifyToolUse(
  tool: Tool,
  input: unknown
): Promise<ClassifierResult> {
  // 1. 检查是否是已知的危险模式
  if (isDangerousPattern(tool, input)) {
    return { safe: false, reason: 'dangerous_pattern' };
  }
  
  // 2. 使用小模型进行分类
  const classification = await smallModel.classify({
    tool: tool.name,
    input: input,
    context: 'Is this tool call safe?',
  });
  
  return {
    safe: classification.safe,
    confidence: classification.confidence,
    reason: classification.reason,
  };
}
```

---

## 5. 确认机制

### 5.1 确认对话框

当需要用户确认时，Claude Code 会显示一个确认对话框：

```
┌─────────────────────────────────────────────────┐
│  Claude wants to run: rm -rf /tmp/old-data      │
│                                                 │
│  This will permanently delete files.            │
│                                                 │
│  [Allow]  [Deny]  [Allow for this session]      │
└─────────────────────────────────────────────────┘
```

### 5.2 确认选项

| 选项 | 说明 | 持久性 |
|------|------|--------|
| Allow | 允许这一次 | 单次 |
| Deny | 拒绝这一次 | 单次 |
| Allow for this session | 本次会话内允许 | 会话级 |
| Always allow | 始终允许 | 永久 |

### 5.3 确认队列

当有多个工具调用需要确认时，Claude Code 使用队列管理：

```typescript
// components/permissions/PermissionRequest.tsx 核心逻辑（简化）
function PermissionRequest({ queue }: { queue: ToolUseConfirm[] }) {
  const current = queue[0]; // 只显示第一个待确认的
  
  if (!current) return null;
  
  return (
    <Box flexDirection="column">
      <Text>Claude wants to run: {current.command}</Text>
      <Text>{current.description}</Text>
      
      <Box>
        <Button onClick={current.onAllow}>Allow</Button>
        <Button onClick={current.onDeny}>Deny</Button>
        <Button onClick={current.onAllowSession}>Allow for this session</Button>
      </Box>
    </Box>
  );
}
```

---

## 6. 交互设计亮点

### 6.1 实时反馈

- LLM 输出是流式的，用户可以实时看到生成过程
- 工具执行有进度显示（Spinner、进度条）
- 长时间操作有已用时间显示

### 6.2 上下文感知

- 自动检测 Git 状态，显示当前分支
- 自动检测工作目录，显示相对路径
- 自动检测操作系统，调整命令语法

### 6.3 多模态支持

- 支持图片输入（截图、设计稿）
- 支持文件附件
- 支持代码选区

### 6.4 个性化配置

- 支持主题切换（亮色/暗色）
- 支持模型切换
- 支持输出风格切换（简洁/详细）

---

## 总结

| 交互维度 | 实现方式 | 关键文件 |
|----------|----------|----------|
| CLI 设计 | Ink/React 终端 UI | components/, ink/ |
| IDE 集成 | 扩展 + 远程通信 | remote/, entrypoints/ |
| 对话管理 | 消息队列 + 状态机 | types/message.ts |
| 权限控制 | 规则 + 分类器 + 确认框 | hooks/toolPermission/ |
| 确认机制 | 队列 + 多级选项 | PermissionRequest.tsx |
