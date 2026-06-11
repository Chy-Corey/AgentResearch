# Agent Research

AI Agent 横向对比研究项目，从源码级别深入分析四个热门 Agent 的架构设计、技术实现与工程实践。

## 研究对象

| Agent | 开发者 | 定位 |
|-------|--------|------|
| **Claude Code** | Anthropic | CLI Agent，深度集成 Claude 模型 |
| **Codex** | OpenAI | CLI Agent，基于 Codex 模型 |
| **Hermes** | 待补充 | 待补充 |
| **OpenClaw** | 待补充 | 待补充 |

## 项目架构

```
AgentResearch/
│
├── overview/                          # 📊 横向对比（自上而下看全局）
│   ├── README.md                      #   对比总览入口
│   ├── architecture-comparison.md     #   架构设计对比
│   ├── capability-comparison.md       #   核心能力对比（代码生成、理解、编辑、工具调用等）
│   ├── tech-implementation-comparison.md # 技术实现对比（LLM集成、Prompt、RAG、记忆等）
│   ├── interaction-comparison.md      #   交互模式对比（CLI、IDE、权限、确认机制）
│   ├── performance-comparison.md      #   性能表现对比（延迟、Token效率、并发、缓存）
│   ├── security-comparison.md         #   安全机制对比（沙箱、权限模型、数据隐私）
│   ├── extensibility-comparison.md    #   可扩展性对比（插件、MCP、工具注册）
│   ├── ecosystem-comparison.md        #   生态系统对比（社区、集成、更新频率）
│   └── summary.md                     #   总结与结论
│
├── agents/                            # 🔬 各 Agent 深度分析（自下而上挖细节）
│   ├── claude-code/                   #   Claude Code
│   ├── codex/                         #   Codex
│   ├── hermes/                        #   Hermes
│   └── openclaw/                      #   OpenClaw
│   每个 Agent 目录结构一致：
│   ├── README.md                      #     概述与定位
│   ├── architecture.md                #     架构设计分析
│   ├── core-capabilities.md           #     核心能力分析
│   ├── tech-implementation.md         #     技术实现细节
│   ├── interaction-design.md          #     交互设计分析
│   ├── security-model.md              #     安全模型分析
│   ├── extensibility.md               #     可扩展性分析
│   ├── pros-cons.md                   #     优缺点总结
│   └── source-code/                   #     源码级分析
│       ├── project-structure.md       #       项目结构解读
│       ├── entry-points.md            #       入口文件分析
│       ├── core-modules.md            #       核心模块源码分析
│       ├── key-algorithms.md          #       关键算法与设计模式
│       └── dependency-analysis.md     #       依赖分析
│
├── methodology/                       # 📐 研究方法论
│   ├── README.md                      #   方法论概述与研究流程
│   ├── evaluation-criteria.md         #   评估标准定义（维度、权重、评分）
│   ├── testing-approach.md            #   测试方法（场景、指标、环境）
│   └── data-collection.md             #   数据收集方法
│
└── references/                        # 📚 参考资料
    ├── links.md                       #   官方链接汇总
    ├── tools.md                       #   使用的工具
    └── glossary.md                    #   术语表
```

## 研究维度

本项目从 **9 个维度** 对 Agent 进行系统性对比：

| 维度 | 关注点 |
|------|--------|
| 架构设计 | 整体架构、模块划分、数据流、状态管理、通信机制 |
| 核心能力 | 推理层（规划对不对）、行动层（工具选对没）、整体执行（完成了吗） |
| 技术实现 | LLM 集成、Prompt 工程、RAG 实现、记忆管理、错误处理、流式输出 |
| 交互体验 | CLI 设计、IDE 集成、对话管理、权限控制、确认机制 |
| 安全机制 | 沙箱隔离、权限模型、数据隐私、命令白名单 |
| 可扩展性 | 插件系统、工具注册、自定义指令、MCP 支持 |
| 社区活跃度 | GitHub 指标、第三方集成、社区内容、更新频率 |

## 快速导航

- [横向对比总览](overview/README.md) — 先看全局对比
- [研究方法论](methodology/README.md) — 了解评估标准
- [社区活跃度数据](references/community.md) — 各 Agent 的 GitHub 指标和社区生态
- [术语表](references/glossary.md) — 查阅专业术语

### 各 Agent 深度分析

| Agent | 入口 | 源码分析 |
|-------|------|----------|
| Claude Code | [概述](agents/claude-code/README.md) | [源码](agents/claude-code/source-code/) |
| Codex | [概述](agents/codex/README.md) | [源码](agents/codex/source-code/) |
| Hermes | [概述](agents/hermes/README.md) | [源码](agents/hermes/source-code/) |
| OpenClaw | [概述](agents/openclaw/README.md) | [源码](agents/openclaw/source-code/) |
