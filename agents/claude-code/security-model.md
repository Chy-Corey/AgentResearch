# Claude Code — 安全模型分析

本文档分析 Claude Code 的安全模型，帮助理解 Agent 是如何保护用户数据和系统安全的。

---

## 1. 沙箱隔离

### 1.1 沙箱架构

Claude Code 使用 **多层沙箱** 来隔离工具执行：

```
用户请求
  ↓
Claude Code 主进程
  ↓
沙箱层
├─ 文件系统沙箱（限制访问路径）
├─ 命令执行沙箱（限制可执行命令）
├─ 网络沙箱（限制网络访问）
└─ 进程沙箱（限制子进程权限）
  ↓
实际执行
```

### 1.2 文件系统沙箱

```typescript
// utils/permissions/filesystem.ts 核心逻辑（简化）
function isPathAllowed(path: string): boolean {
  const allowedPaths = [
    getCwd(),                    // 当前工作目录
    getHomeDir(),                // 用户主目录
    getTempDir(),                // 临时目录
  ];
  
  const deniedPaths = [
    '/etc',                      // 系统配置
    '/var',                      // 系统变量
    '/usr',                      // 系统程序
    'node_modules',              // 依赖目录（防止修改第三方代码）
  ];
  
  // 检查路径是否在允许范围内
  const isAllowed = allowedPaths.some(allowed => 
    path.startsWith(allowed)
  );
  
  // 检查路径是否在拒绝列表中
  const isDenied = deniedPaths.some(denied => 
    path.includes(denied)
  );
  
  return isAllowed && !isDenied;
}
```

### 1.3 命令执行沙箱

BashTool 有专门的沙箱机制：

```typescript
// tools/BashTool/shouldUseSandbox.ts 核心逻辑（简化）
function shouldUseSandbox(command: string): boolean {
  // 1. 检查是否是危险命令
  const dangerousCommands = [
    'rm -rf',           // 递归删除
    'chmod 777',        // 修改权限
    'sudo',             // 提权
    'curl | bash',      // 远程执行
    'wget | sh',        // 远程执行
  ];
  
  if (dangerousCommands.some(cmd => command.includes(cmd))) {
    return true;
  }
  
  // 2. 检查是否是未知命令
  const knownCommands = ['git', 'npm', 'yarn', 'node', 'python', 'pip'];
  const firstCommand = command.split(' ')[0];
  
  if (!knownCommands.includes(firstCommand)) {
    return true; // 未知命令使用沙箱
  }
  
  return false;
}
```

### 1.4 沙箱实现

Claude Code 支持多种沙箱实现：

| 实现 | 说明 | 适用场景 |
|------|------|----------|
| SandboxManager | 内置沙箱管理器 | 默认实现 |
| Docker 沙箱 | 使用 Docker 容器隔离 | 高安全需求 |
| Firejail 沙箱 | 使用 Firejail 隔离 | Linux 环境 |

---

## 2. 权限模型

### 2.1 权限层级

Claude Code 的权限模型分为 **4 个层级**：

```
权限层级
├─ 全局权限（Global）
│   └─ ~/.claude/settings.json
│
├─ 项目权限（Project）
│   └─ .claude/settings.json
│
├─ 会话权限（Session）
│   └─ 运行时临时权限
│
└─ 单次权限（One-time）
    └─ 用户确认时的"Allow"选项
```

### 2.2 权限规则

权限规则使用 **工具 + 模式 + 动作** 的三元组：

```typescript
// types/permissions.ts 核心定义（简化）
type PermissionRule = {
  tool: string;           // 工具名称，如 "BashTool"
  pattern: string;        // 匹配模式，如 "git *" 或 "rm -rf *"
  action: 'allow' | 'deny'; // 动作
  source: 'global' | 'project' | 'session'; // 来源
};
```

### 2.3 权限检查流程

```typescript
// utils/permissions/permissions.ts 核心逻辑（简化）
async function hasPermissionsToUseTool(
  tool: Tool,
  input: unknown,
  context: ToolUseContext
): Promise<PermissionResult> {
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
  
  // 3. 检查权限规则
  const rules = await loadPermissionRules();
  const matchingRule = findMatchingRule(rules, tool, input);
  
  if (matchingRule) {
    if (matchingRule.action === 'allow') {
      return { behavior: 'allow' };
    } else {
      return { behavior: 'deny', reason: 'rule_denied' };
    }
  }
  
  // 4. 运行分类器（auto 模式）
  if (permissionMode === 'auto') {
    const classifierResult = await runClassifier(tool, input);
    if (classifierResult.safe) {
      return { behavior: 'allow', decisionReason: { type: 'classifier' } };
    }
  }
  
  // 5. 需要用户确认
  return { behavior: 'ask_user' };
}
```

### 2.4 权限模式

| 模式 | 说明 | 安全级别 | 适用场景 |
|------|------|----------|----------|
| Default | 每次危险操作都询问 | 最高 | 生产环境 |
| Auto-accept | 自动批准所有操作 | 最低 | 测试环境 |
| Auto | 分类器自动判断 | 中等 | 日常开发 |
| Bypass | 跳过所有权限检查 | 无 | 调试模式 |

---

## 3. 数据隐私

### 3.1 数据收集

Claude Code 收集的数据类型：

| 数据类型 | 用途 | 是否上传 |
|----------|------|----------|
| 对话消息 | 提供上下文 | 是（发送到 Anthropic API） |
| 工具调用结果 | 提供上下文 | 是（发送到 Anthropic API） |
| 错误日志 | 改进产品 | 是（匿名化后上传） |
| 使用统计 | 分析功能使用 | 是（匿名化后上传） |
| 配置信息 | 个性化设置 | 否（本地存储） |

### 3.2 数据传输安全

```typescript
// services/api/client.ts 核心逻辑（简化）
const client = new Anthropic({
  apiKey: getApiKey(),
  baseURL: getBaseUrl(),
  // 使用 HTTPS
  // 支持代理
  httpAgent: getProxyAgent(),
  // 超时设置
  timeout: 30000,
});
```

**安全措施**：
- 所有 API 调用都使用 HTTPS
- API Key 存储在本地，不上传
- 支持代理服务器
- 支持自定义 API 端点

### 3.3 本地数据存储

```
~/.claude/
├─ settings.json         # 配置文件（不包含敏感信息）
├─ memory/               # 记忆文件（本地存储）
├─ projects/             # 项目数据（本地存储）
│   └─ <project-hash>/
│       └─ <session>.jsonl  # 对话记录
└─ logs/                 # 日志文件（本地存储）
```

### 3.4 遥测控制

用户可以控制遥测数据的收集：

```json
// ~/.claude/settings.json
{
  "telemetry": {
    "enabled": false,           // 禁用所有遥测
    "errorReporting": false,    // 禁用错误报告
    "usageStats": false         // 禁用使用统计
  }
}
```

---

## 4. 命令白名单

### 4.1 白名单机制

Claude Code 使用 **白名单** 来控制可执行的命令：

```typescript
// utils/permissions/dangerousPatterns.ts 核心逻辑（简化）
const DANGEROUS_PATTERNS = [
  // 文件系统破坏
  /rm\s+-rf\s+[\/~]/,           // rm -rf / 或 rm -rf ~
  /chmod\s+777/,                // 修改权限
  /chown\s+/,                   // 修改所有者
  
  // 系统修改
  /sudo\s+/,                    // 提权
  /su\s+/,                      // 切换用户
  
  // 远程执行
  /curl\s+.*\|\s*(ba)?sh/,     // curl | bash
  /wget\s+.*\|\s*(ba)?sh/,     // wget | sh
  
  // 网络操作
  /nc\s+-l/,                    // 监听端口
  /ncat\s+-l/,                  // 监听端口
  
  // 编译执行
  /gcc\s+.*&&\s*\./,           // 编译并执行
  /g\+\+\s+.*&&\s*\./,         // 编译并执行
];

function isDangerousPattern(command: string): boolean {
  return DANGEROUS_PATTERNS.some(pattern => pattern.test(command));
}
```

### 4.2 命令分类

| 类别 | 示例 | 默认行为 |
|------|------|----------|
| 安全命令 | `git status`, `ls`, `cat` | 允许 |
| 低风险命令 | `npm install`, `git commit` | 允许（可能询问） |
| 中风险命令 | `git push`, `npm publish` | 询问 |
| 高风险命令 | `rm -rf`, `sudo`, `chmod 777` | 拒绝或询问 |
| 危险命令 | `curl \| bash`, `dd` | 拒绝 |

### 4.3 自定义白名单

用户可以在 `~/.claude/settings.json` 中自定义白名单：

```json
{
  "permissions": {
    "rules": [
      {
        "tool": "BashTool",
        "pattern": "docker *",
        "action": "allow"
      },
      {
        "tool": "BashTool",
        "pattern": "kubectl *",
        "action": "allow"
      }
    ]
  }
}
```

---

## 5. 安全设计亮点

### 5.1 分类器机制

Claude Code 使用 **小模型分类器** 来判断操作是否安全：

```
工具调用请求
  ↓
小模型分类器（快速、低成本）
  ├─ 安全 → 直接执行
  └─ 危险 → 弹出确认框
```

**优势**：
- 不需要用户每次都确认
- 可以学习用户的行为模式
- 减少用户的交互负担

### 5.2 推测性分类

在工具参数还没完全生成时，就开始运行分类器：

```typescript
// 核心逻辑（简化）
async function peekSpeculativeClassifierCheck(toolUseID: string) {
  // 在工具参数还没完全生成时就开始分类
  // 减少用户等待时间
}
```

### 5.3 权限持久化

用户的权限选择可以持久化：

| 选项 | 持久性 | 存储位置 |
|------|--------|----------|
| Allow | 单次 | 内存 |
| Allow for this session | 会话级 | 内存 |
| Always allow | 永久 | ~/.claude/settings.json |

### 5.4 审计日志

所有权限决策都会被记录：

```typescript
// hooks/toolPermission/permissionLogging.ts 核心逻辑（简化）
function logPermissionDecision(
  context: PermissionContext,
  decision: PermissionDecision
): void {
  logEvent('permission_decision', {
    tool: context.tool.name,
    input: JSON.stringify(context.input),
    decision: decision.behavior,
    reason: decision.decisionReason,
    timestamp: new Date().toISOString(),
  });
}
```

---

## 6. 安全最佳实践

### 6.1 对于用户

1. **使用 Default 模式**：除非你完全信任 Claude，否则使用默认模式
2. **配置权限规则**：为常用命令配置白名单，减少确认次数
3. **定期审查日志**：检查 `~/.claude/logs/` 中的权限决策日志
4. **不要在生产环境使用 Auto-accept**：生产环境应该使用 Default 模式

### 6.2 对于开发者

1. **最小权限原则**：工具只请求必要的权限
2. **输入验证**：所有工具输入都需要验证
3. **错误处理**：工具执行失败时应该优雅降级
4. **日志记录**：所有安全相关的操作都应该记录日志

---

## 总结

| 安全维度 | 实现方式 | 关键文件 |
|----------|----------|----------|
| 沙箱隔离 | 文件系统 + 命令执行 + 网络 | shouldUseSandbox.ts |
| 权限模型 | 规则 + 分类器 + 确认框 | permissions.ts |
| 数据隐私 | HTTPS + 本地存储 + 遥测控制 | client.ts |
| 命令白名单 | 正则匹配 + 自定义规则 | dangerousPatterns.ts |
| 审计日志 | 权限决策记录 | permissionLogging.ts |
