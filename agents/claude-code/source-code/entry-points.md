# Claude Code — 入口文件分析

## 启动命令

```bash
bun run dev          # → bun run ./src/dev-entry.ts
bun run start        # → bun run ./src/dev-entry.ts
bun run version      # → bun run ./src/dev-entry.ts --version
```

## 命令执行流程

以 `bun run dev --version` 为例：

```
bun run dev --version
       ↓
查找 package.json 的 scripts 字段
       ↓
scripts.dev = "bun run ./src/dev-entry.ts"
       ↓
实际执行：bun run ./src/dev-entry.ts --version
       ↓
dev-entry.ts 读取 process.argv = ["bun", "./src/dev-entry.ts", "--version"]
       ↓
const args = process.argv.slice(2)  →  args = ["--version"]
       ↓
匹配到 args.includes('--version')
       ↓
console.log(pkg.version)  →  输出版本号并退出
```

**关键点**：`bun run <script> <args>` 会自动把 `<args>` 透传给脚本的 `process.argv`，所以 `--version` 会出现在 `process.argv` 中。

## 启动链路

以 `bun run dev`（交互式模式）为例，完整调用链如下：

```
package.json scripts
  → src/dev-entry.ts                    # 第1层：开发入口
    → src/entrypoints/cli.tsx           # 第2层：CLI 引导
      → src/main.tsx                    # 第3层：主入口
        ├─ src/entrypoints/init.ts      # 第3.1层：核心初始化
        ├─ src/tools.ts                 # 第3.2层：工具注册
        ├─ src/commands.ts              # 第3.3层：命令注册
        ├─ src/plugins/bundled/         # 第3.4层：插件初始化
        └─ src/skills/bundled/          # 第3.5层：技能初始化
        → src/replLauncher.tsx          # 第4层：REPL 启动
          → src/components/App.tsx      # 第4.1层：App 外壳
          → src/screens/REPL.tsx        # 第5层：交互式 REPL 界面
```

### 每一层做了什么

| 层级 | 文件 | 职责 | 调用了谁 |
|------|------|------|----------|
| 入口 | `package.json` | 定义 `scripts.dev = "bun run ./src/dev-entry.ts"` | dev-entry.ts |
| 第1层 | `dev-entry.ts` | 设置全局宏配置 MACRO、扫描缺失导入 | cli.tsx |
| 第2层 | `cli.tsx` | 快速路径分发（12+ 个快速路径） | main.tsx |
| 第3层 | `main.tsx` | Commander.js 参数解析、并行初始化 | init.ts、tools.ts 等 |
| 3.1 | `init.ts` | 启用配置、设置代理、检测仓库、初始化遥测 | - |
| 3.2 | `tools.ts` | 加载 50+ 个工具（BashTool、FileEditTool 等） | - |
| 3.3 | `commands.ts` | 加载 100+ 个斜杠命令 | - |
| 3.4 | `plugins/bundled/` | 初始化内置插件 | - |
| 3.5 | `skills/bundled/` | 初始化内置技能 | - |
| 第4层 | `replLauncher.tsx` | 加载 App 和 REPL 组件，渲染终端 UI | App.tsx、REPL.tsx |
| 4.1 | `components/App.tsx` | 应用外壳：状态管理、主题、上下文 | - |
| 第5层 | `screens/REPL.tsx` | 交互式界面：输入框、对话流、工具调用展示 | - |

### 调用链对应的代码

```typescript
// 第1层：dev-entry.ts
await import('./entrypoints/cli.tsx');

// 第2层：cli.tsx
const { main: cliMain } = await import('../main.js');
await cliMain();

// 第3层：main.tsx
await init();                                    // 3.1 核心初始化
const tools = getTools();                         // 3.2 工具注册
const commands = getCommands();                   // 3.3 命令注册
initBuiltinPlugins();                             // 3.4 插件初始化
initBundledSkills();                              // 3.5 技能初始化
await launchRepl(root, appProps, replProps, ...); // 第4层

// 第4层：replLauncher.tsx
const { App } = await import('./components/App.js');
const { REPL } = await import('./screens/REPL.js');
await renderAndRun(root, <App><REPL /></App>);

// 第5层：用户看到的交互式界面
```

### 快速路径 vs 完整路径

```
快速路径（如 --version）：
  dev-entry.ts → cli.tsx → 直接输出 → 退出
  只加载 2 个文件，不进入第3层

完整路径（交互式）：
  dev-entry.ts → cli.tsx → main.tsx → init.ts + tools.ts + ... → replLauncher → REPL
  加载所有层级，进入 REPL 交互
```

---

## 1. dev-entry.ts — 开发入口

**作用**：开发环境的启动入口，负责宏配置和源码完整性检查。

### 关键逻辑

1. **设置全局宏配置** `MACRO`
   - VERSION、BUILD_TIME、PACKAGE_URL 等构建时信息
   - 从 `package.json` 读取版本号

2. **扫描缺失的导入**
   - 扫描 `src/` 和 `vendor/` 下所有 `.ts/.tsx/.js/.jsx` 文件
   - 检查每个 `import/require` 的目标文件是否存在
   - 统计缺失的相对导入数量

3. **快速路径**
   - `--version`：直接打印版本号并退出
   - `--help`：打印帮助信息并退出
   - 如果有缺失导入：打印诊断信息并退出（源码重建版特有）

4. **正常路径**
   - 如果没有缺失导入：`await import('./entrypoints/cli.tsx')` 进入 CLI

---

## 2. entrypoints/cli.tsx — CLI 引导入口

**作用**：Bootstrap 入口，根据命令行参数分发到不同的快速路径，避免加载完整 CLI 的开销。

### 设计理念

> "All imports are dynamic to minimize module evaluation for fast paths."
> 所有导入都是动态的，以最小化快速路径的模块加载。

### 快速路径分发

| 参数 | 目标 | 说明 |
|------|------|------|
| `--version` / `-v` | 直接输出 | 零模块加载，最快 |
| `--dump-system-prompt` | 输出系统提示词 | 用于 prompt 评估 |
| `--claude-in-chrome-mcp` | Chrome MCP 服务器 | Chrome 集成 |
| `--chrome-native-host` | Chrome 原生主机 | Chrome 集成 |
| `--computer-use-mcp` | 计算机使用 MCP | Computer Use 功能 |
| `--daemon-worker` | 守护进程 Worker | 后台服务 |
| `remote-control` / `rc` / `bridge` | 远程控制 | 桥接功能 |
| `daemon` | 守护进程 | 长期运行的服务 |
| `ps` / `logs` / `attach` / `kill` | 后台会话管理 | 会话管理 |
| `new` / `list` / `reply` | 模板任务 | 任务模板 |
| `environment-runner` | 环境运行器 | BYOC 无头运行 |
| `self-hosted-runner` | 自托管运行器 | 自托管服务 |
| `--worktree` + `--tmux` | tmux worktree | 工作树模式 |

### 正常路径

如果没有匹配的快速路径：

```typescript
const { main: cliMain } = await import('../main.js');
await cliMain();
```

加载 `main.tsx` 的完整 CLI。

---

## 3. main.tsx — 主入口

**作用**：完整的 CLI 初始化，包括参数解析、配置加载、工具注册、REPL 启动。

> 文件大小：~785KB（非常大，包含大量逻辑）

### 启动时的性能优化（并行初始化）

main.tsx 在模块加载阶段就开始并行执行任务：

```
模块加载开始
  ├─ profileCheckpoint('main_tsx_entry')     # 性能打点
  ├─ startMdmRawRead()                       # 并行：MDM 配置读取（macOS）
  ├─ startKeychainPrefetch()                 # 并行：钥匙串预取（OAuth + API Key）
  └─ 其他 import...                           # ~135ms 的模块加载
```

**目的**：让 I/O 密集型操作（MDM、钥匙串）与模块加载并行，减少启动时间。

### Commander.js 命令行定义

使用 `@commander-js/extra-typings` 定义 CLI：

- 主命令 `claude`
- 各种子命令（`config`、`update`、`mcp` 等）
- 各种选项（`--model`、`--permission-mode`、`--output-format` 等）

### 初始化流程

```
main() 被调用
  ├─ 解析命令行参数（Commander.js）
  ├─ 初始化 GrowthBook（功能标志）
  ├─ 初始化遥测和分析
  ├─ 加载配置（globalConfig、projectConfig）
  ├─ 加载远程管理设置
  ├─ 加载策略限制（Policy Limits）
  ├─ 初始化 OAuth 认证
  ├─ 初始化 MCP 服务器
  ├─ 注册工具（getTools()）
  ├─ 注册命令（getCommands()）
  ├─ 初始化内置插件（initBuiltinPlugins()）
  ├─ 初始化内置技能（initBundledSkills()）
  ├─ 显示设置屏幕（如果需要）
  └─ 进入 REPL 或执行命令
```

### 关键模块导入

| 模块 | 作用 |
|------|------|
| `getTools()` from `tools.ts` | 获取所有注册的工具 |
| `getCommands()` from `commands.ts` | 获取所有注册的命令 |
| `initBuiltinPlugins()` from `plugins/` | 初始化内置插件 |
| `initBundledSkills()` from `skills/` | 初始化内置技能 |
| `getMcpToolsCommandsAndResources()` | 获取 MCP 工具和资源 |
| `launchRepl()` from `replLauncher.tsx` | 启动 REPL |
| `init()` from `entrypoints/init.ts` | 核心初始化 |

---

## 4. replLauncher.tsx — REPL 启动器

**作用**：加载 App 和 REPL 组件，渲染终端 UI。

### 逻辑

```typescript
export async function launchRepl(root, appProps, replProps, renderAndRun) {
  const { App } = await import('./components/App.js');
  const { REPL } = await import('./screens/REPL.js');
  await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>);
}
```

- **App**：应用外壳组件（状态管理、主题等）
- **REPL**：交互式 REPL 界面（输入框、对话流、工具调用展示）
- 使用 **Ink**（React 终端 UI 框架）渲染

---

## 5. entrypoints/init.ts — 核心初始化

**作用**：执行应用的核心初始化逻辑，被 `main.tsx` 调用。

### 初始化步骤

```
init()
  ├─ enableConfigs()                    # 启用配置系统
  ├─ applySafeConfigEnvironmentVariables() # 应用安全的环境变量
  ├─ applyExtraCACertsFromConfig()      # 应用额外的 CA 证书
  ├─ setupGracefulShutdown()            # 设置优雅关闭
  ├─ initialize1PEventLogging()         # 初始化第一方事件日志
  ├─ initializeGrowthBook()             # 初始化功能标志
  ├─ configureGlobalAgents()            # 配置 HTTP 代理
  ├─ setShellIfWindows()                # Windows Shell 兼容
  ├─ detectCurrentRepository()          # 检测当前 Git 仓库
  ├─ initJetBrainsDetection()           # 检测 JetBrains IDE
  ├─ preconnectAnthropicApi()           # 预连接 Anthropic API
  ├─ initializePolicyLimitsLoadingPromise() # 初始化策略限制
  └─ initializeRemoteManagedSettingsLoadingPromise() # 初始化远程设置
```

### 设计特点

- **memoize**：`init()` 使用 `memoize` 包装，确保只执行一次
- **并行初始化**：多个 I/O 操作使用 `Promise.all` 并行执行
- **延迟加载**：大型模块（如 OpenTelemetry）通过 `import()` 延迟加载

---

## 其他入口点

### entrypoints/mcp.ts

MCP（Model Context Protocol）服务器入口，用于：
- 作为 MCP 服务器被其他应用调用
- 提供工具和资源给外部系统

### entrypoints/sdk/

SDK 入口，用于：
- 以编程方式调用 Claude Code
- 集成到其他应用中

---

## 启动性能关键路径

```
最快路径（--version）：
  dev-entry.ts → 直接输出 → 退出
  耗时：<10ms

快速路径（--help、remote-control 等）：
  dev-entry.ts → cli.tsx → 动态导入对应模块 → 执行 → 退出
  耗时：~50-100ms

完整路径（交互式 REPL）：
  dev-entry.ts → cli.tsx → main.tsx（并行初始化）→ init() → launchRepl()
  耗时：~200-500ms
```

---

## 入口文件总结

| 文件 | 职责 | 大小 |
|------|------|------|
| `dev-entry.ts` | 开发入口，宏配置 + 缺失导入检测 | 小 |
| `entrypoints/cli.tsx` | CLI 引导，快速路径分发 | 中 |
| `main.tsx` | 主入口，完整 CLI 初始化 | 大（785KB） |
| `replLauncher.tsx` | REPL 启动器，加载 UI 组件 | 小 |
| `entrypoints/init.ts` | 核心初始化，配置/认证/代理 | 中 |
| `entrypoints/mcp.ts` | MCP 服务器入口 | 小 |
| `entrypoints/sdk/` | SDK 入口 | 小 |
