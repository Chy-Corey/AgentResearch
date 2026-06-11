# Claude Code 源码分析流程

本文档指导如何逐步分析 Claude Code 源码，每一步都有明确的目标、步骤和输出文件。

源码位置：`sources/claude-code/`

---

## 阶段一：项目结构概览

**目标**：理解整体目录组织，建立全局认知。

### 步骤

1. **浏览根目录**
   - 查看 `package.json`：了解项目名称、版本、构建工具、依赖数量
   - 查看 `tsconfig.json`：了解 TypeScript 配置
   - 查看 `README.md`：了解官方对项目的描述
   - 查看 `AGENTS.md`：了解是否有 Agent 相关说明

2. **浏览 `src/` 一级目录**
   - 不需要深入每个文件，只看目录名和目录内的文件数量
   - 用 `ls -la src/` 或文件浏览器查看
   - 记录每个目录的名称和推测的职责

3. **识别目录层级关系**
   - 哪些是核心目录（tools/、commands/、services/）
   - 哪些是辅助目录（utils/、types/、constants/）
   - 哪些是功能模块（voice/、vim/、remote/）

### 关键文件

| 文件 | 作用 |
|------|------|
| `package.json` | 项目元信息、依赖、构建脚本 |
| `tsconfig.json` | TypeScript 编译配置 |
| `src/main.tsx` | 主入口文件 |

### 输出

📄 填写到 [project-structure.md](project-structure.md)

内容要求：
- 根目录结构图
- `src/` 目录结构图（一级目录 + 推测职责）
- 核心目录 vs 辅助目录的分类

---

## 阶段二：入口与启动流程

**目标**：搞清楚程序从启动到运行的完整链路。

### 步骤

1. **找到启动命令**
   - 看 `package.json` 的 `scripts` 字段
   - 确定 `dev` / `start` 命令指向哪个文件
   - 当前项目：`bun run ./src/dev-entry.ts`

2. **追踪启动链路**
   ```
   package.json scripts
     → src/dev-entry.ts（或 src/main.tsx）
       → src/entrypoints/cli.tsx（CLI 入口）
         → 初始化逻辑
           → 进入交互模式 / 执行命令
   ```

3. **分析 CLI 入口** `src/entrypoints/cli.tsx`
   - 使用了什么 CLI 框架（commander / yargs / 自研）
   - 注册了哪些命令和参数
   - 如何处理用户输入

4. **分析初始化流程**
   - 配置加载（API Key、模型选择、权限设置）
   - 工具注册（tools/ 目录下的工具如何被加载）
   - 上下文初始化（context/ 目录的作用）
   - 状态初始化（state/ 目录的作用）

5. **分析交互模式**
   - REPL 循环怎么实现的
   - 用户输入如何传递给 LLM
   - LLM 响应如何展示

### 关键文件

| 文件 | 作用 |
|------|------|
| `src/dev-entry.ts` | 开发入口 |
| `src/main.tsx` | 主入口 |
| `src/entrypoints/cli.tsx` | CLI 入口点 |
| `src/entrypoints/init.ts` | 初始化逻辑 |
| `src/cli/` | CLI 相关实现 |
| `src/setup.ts` | 设置/配置 |

### 输出

📄 填写到 [entry-points.md](entry-points.md)

内容要求：
- 启动命令和入口文件映射
- 启动链路流程图（文字版）
- 初始化流程的关键步骤
- 交互模式的实现方式

---

## 阶段三：核心模块分析

**目标**：识别核心模块，理解各模块的职责和协作关系。

### 步骤

1. **分析工具系统** `src/tools/`
   - 统计工具数量（当前约 50+ 个）
   - 按类别分组：
     - 文件操作：FileReadTool、FileEditTool、FileWriteTool
     - 搜索：GrepTool、GlobTool
     - 终端：BashTool、PowerShellTool
     - 交互：AskUserQuestionTool、SkillTool
     - 任务管理：TaskCreateTool、TaskOutputTool、TodoWriteTool
     - 其他：WebFetchTool、WebSearchTool、WorkflowTool
   - 分析单个工具的实现结构（看一两个工具的源码）
   - 工具如何注册、如何被调用

2. **分析命令系统** `src/commands/`
   - 注册了哪些斜杠命令（/help、/clear、/config 等）
   - 命令的实现方式
   - 与工具系统的关系

3. **分析技能系统** `src/skills/`
   - Skill 是什么（自定义 slash command）
   - Skill 的定义格式和加载机制
   - SkillTool 如何调用 Skill

4. **分析钩子系统** `src/hooks/`
   - 有哪些 Hook 点
   - Hook 的触发时机和作用
   - 用户如何自定义 Hook

5. **分析上下文管理** `src/context/`
   - 上下文包含哪些信息
   - 上下文如何构建和更新
   - 上下文窗口管理策略

6. **分析状态管理** `src/state/`
   - 管理哪些状态（会话、配置、权限等）
   - 状态的持久化机制
   - 状态更新的触发方式

7. **分析查询引擎** `src/query/`
   - 查询引擎的作用
   - 与 LLM 的交互方式
   - 流式输出的实现

### 关键文件

| 目录 | 作用 |
|------|------|
| `src/tools/` | 工具实现（50+ 个工具） |
| `src/commands/` | 斜杠命令 |
| `src/skills/` | 技能系统 |
| `src/hooks/` | 钩子系统 |
| `src/context/` | 上下文管理 |
| `src/state/` | 状态管理 |
| `src/query/` | 查询引擎 |
| `src/services/` | 服务层 |
| `src/coordinator/` | 协调器 |

### 输出

📄 填写到 [core-modules.md](core-modules.md)

内容要求：
- 核心模块列表和职责说明
- 工具分类表（按功能分组）
- 模块间依赖关系图（文字版）
- 每个核心模块的关键实现要点

---

## 阶段四：关键算法与设计模式

**目标**：提取架构设计模式和关键实现细节。

### 步骤

1. **分析工具调用链路**
   ```
   用户输入 → LLM 决策调用工具 → 工具选择 → 参数解析 → 工具执行 → 结果返回 → LLM 继续
   ```
   - 追踪一个完整的工具调用流程
   - 看 Tool.ts 的基类设计
   - 看工具如何被发现和调度

2. **分析 Agent 循环**
   - Agent 的主循环怎么实现的
   - 如何决定继续调用工具还是返回结果
   - 最大迭代次数和退出条件

3. **分析上下文窗口管理**
   - 上下文超长时怎么处理
   - 是否有压缩/摘要机制
   - Token 计数方式

4. **分析权限控制**
   - 工具执行前的权限检查
   - 用户确认机制的实现
   - 沙箱隔离的实现

5. **分析流式输出**
   - LLM 响应的流式处理
   - 工具调用结果的增量展示
   - 终端渲染方式（Ink/React）

6. **识别设计模式**
   - 观察者模式（事件系统）
   - 策略模式（工具调度）
   - 责任链模式（中间件/Hook）
   - 工厂模式（工具创建）

### 关键文件

| 文件 | 关注点 |
|------|--------|
| `src/Tool.ts` | 工具基类设计 |
| `src/tools.ts` | 工具注册和调度 |
| `src/Task.ts` | 任务管理 |
| `src/QueryEngine.ts` | 查询引擎 |
| `src/query.ts` | 查询逻辑 |
| `src/context.ts` | 上下文管理 |
| `src/coordinator/` | 协调器 |

### 输出

📄 填写到 [key-algorithms.md](key-algorithms.md)

内容要求：
- 工具调用链路流程图
- Agent 循环的实现方式
- 关键设计模式及应用场景
- 核心算法的文字描述

---

## 阶段五：依赖分析

**目标**：理解技术选型，评估依赖合理性。

### 步骤

1. **分析 `package.json` 依赖**
   - 统计依赖总数
   - 按类别分组：
     - CLI 框架：commander、ink、chalk
     - LLM 集成：@anthropic-ai/sdk、@modelcontextprotocol/sdk
     - 工具执行：execa、shell-quote
     - 数据处理：zod、ajv、yaml、jsonc-parser
     - 终端渲染：ink、react、ansi 相关
     - 网络：axios、undici、ws
     - 监控：@opentelemetry/*
     - 其他：fuse.js（搜索）、diff（差异比较）

2. **分析本地依赖** `shims/`
   - 有哪些本地垫片
   - 为什么需要垫片（兼容性/平台特定）

3. **分析第三方代码** `vendor/`
   - 是否有 vendor 目录
   - 包含哪些第三方代码

4. **评估依赖选型**
   - 关键依赖的版本是否合理
   - 是否有不必要的依赖
   - 依赖的安全性

### 关键文件

| 文件 | 作用 |
|------|------|
| `package.json` | 依赖声明 |
| `bun.lock` | 依赖锁定 |
| `shims/` | 本地垫片 |
| `vendor/` | 第三方代码 |

### 输出

📄 填写到 [dependency-analysis.md](dependency-analysis.md)

内容要求：
- 依赖总数和分类表
- 关键依赖说明（做什么用的）
- 本地垫片分析
- 技术栈总结

---

## 分析顺序建议

```
阶段一（项目结构）→ 阶段二（入口启动）→ 阶段三（核心模块）→ 阶段四（设计模式）→ 阶段五（依赖分析）
```

每个阶段完成后，勾选 WORKFLOW.md 中对应的 checkbox。
