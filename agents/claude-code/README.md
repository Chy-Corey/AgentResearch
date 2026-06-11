# Claude Code

## 基本信息

| 属性 | 值 |
|------|-----|
| 开发者 | Anthropic |
| 第三方仓库 | [泄漏源码](https://github.com/pengchengneo/Claude-Code) |
| 官方文档 | [官方文档](https://code.claude.com/docs/zh-CN/overview) |

## 定位与目标

产品定位：由 AI 驱动的编码工具

Claude Code 是一个代理编码工具，可以读取你的代码库、编辑文件、运行命令，并与你的开发工具集成。可在终端、IDE、桌面应用和浏览器中使用。

Claude Code 是一个由 AI 驱动的编码助手，可帮助你构建功能、修复错误和自动化开发任务。它理解你的整个代码库，可以跨多个文件和工具工作以完成任务。

## 核心特性概览

核心能力：

- 自动化繁琐任务（编写测试、修复 lint 错误、更新依赖）
- 构建功能和修复错误
- 创建提交和拉取请求
- 跨多个文件工作

扩展功能：

- MCP 集成 — 连接 Jira、Slack、Google Drive 等外部工具
- 自定义 — 使用 CLAUDE.md 文件、skills、hooks 进行个性化配置
- 多代理 — 生成多个代理并行处理任务
- 定时任务 — 按计划运行自动化工作（Routines、桌面计划任务）
- 多界面 — 在终端、VS Code、JetBrains、桌面应用、网页和 Slack 中使用

协作特性：

- 远程控制从手机继续工作
- GitHub Actions 和 GitLab CI/CD 集成
- 自动代码审查

## 目录

- [架构设计分析](architecture.md)
- [核心能力分析](core-capabilities.md)
- [技术实现细节](tech-implementation.md)
- [交互设计分析](interaction-design.md)
- [安全模型分析](security-model.md)
- [可扩展性分析](extensibility.md)
- [源码分析](source-code/)
- [优缺点总结](pros-cons.md)
