# 工作流

本文档记录项目的研究推进流程，按阶段拆分任务，逐步完成。

---

## 阶段一：基础准备

### 1.1 完善评估标准

- [x] 明确各评估维度的权重分配（7个维度）
- [x] 定义 5 分制评分标准（2=完成 / 1=有瑕疵 / 0=失败）
- [x] 制定各维度量化方法（架构设计、核心能力、技术实现、交互体验、安全机制、可扩展性、社区活跃度）
- [x] 设计核心能力 15 个统一测试任务（R1-R5, A1-A5, E1-E5）
- 📄 文件：[methodology/evaluation-criteria.md](methodology/evaluation-criteria.md)

### 1.2 收集资料

- [x] 收集 Claude Code 官方仓库、文档链接
- [x] 收集 Codex 官方仓库、文档链接
- [x] 收集 Hermes 官方仓库、文档链接
- [x] 收集 OpenClaw 官方仓库、文档链接
- [x] 收集相关技术博客链接
- [x] 收集各 Agent 的 GitHub 指标（Star、Fork、Issue 活跃度、贡献者数量）
- [x] 收集各 Agent 的社区生态信息（第三方集成、社区内容）
- 📄 文件：[references/links.md](references/links.md)、[references/community.md](references/community.md)

### 1.3 克隆源码

- [x] 克隆 Claude Code 源码到本地
- [x] 克隆 Codex 源码到本地
- [x] 克隆 Hermes 源码到本地
- [x] 克隆 OpenClaw 源码到本地

---

## 阶段二：Claude Code 深度分析

> 第一个样本 Agent，建立分析范式，后续 Agent 复用此模式。

### 2.1 基础信息

- [x] 填写基本信息表（开发者、仓库、文档、版本、许可证）
- [x] 明确产品定位与目标
- [x] 列出核心特性概览
- 📄 文件：[agents/claude-code/README.md](agents/claude-code/README.md)

### 2.2 源码分析

- [x] 阶段一：项目结构概览 → [project-structure.md](agents/claude-code/source-code/project-structure.md)
- [x] 阶段二：入口与启动流程 → [entry-points.md](agents/claude-code/source-code/entry-points.md)
- [x] 阶段三：核心模块分析 → [core-modules.md](agents/claude-code/source-code/core-modules.md)
- [ ] 阶段四：关键算法与设计模式 → [key-algorithms.md](agents/claude-code/source-code/key-algorithms.md)
- [ ] 阶段五：依赖分析 → [dependency-analysis.md](agents/claude-code/source-code/dependency-analysis.md)
- 📄 详细流程：[analysis-guide.md](agents/claude-code/source-code/analysis-guide.md)

### 2.3 架构与能力分析

- [ ] 分析整体架构与模块划分
- [ ] 分析数据流与状态管理
- [ ] 执行核心能力测试（R1-R5 推理层、A1-A5 行动层、E1-E5 整体执行）
- [ ] 记录 15 个任务的得分，填写结果记录表
- 📄 文件：[architecture.md](agents/claude-code/architecture.md)、[core-capabilities.md](agents/claude-code/core-capabilities.md)

### 2.4 技术实现分析

- [ ] 分析 LLM 集成方式
- [ ] 分析 Prompt 工程实现
- [ ] 分析 RAG 与记忆管理机制
- [ ] 分析错误处理与流式输出
- 📄 文件：[tech-implementation.md](agents/claude-code/tech-implementation.md)

### 2.5 交互与安全分析

- [ ] 分析 CLI 设计与 IDE 集成
- [ ] 分析权限控制与确认机制
- [ ] 分析沙箱隔离与数据隐私保护
- 📄 文件：[interaction-design.md](agents/claude-code/interaction-design.md)、[security-model.md](agents/claude-code/security-model.md)

### 2.6 可扩展性与总结

- [ ] 分析插件系统与 MCP 支持
- [ ] 总结优缺点与适用场景
- 📄 文件：[extensibility.md](agents/claude-code/extensibility.md)、[pros-cons.md](agents/claude-code/pros-cons.md)

---

## 阶段三：Codex 深度分析

> 复用阶段二的分析范式。

- [ ] 3.1 基础信息 → [README.md](agents/codex/README.md)
- [ ] 3.2 源码分析 → [source-code/](agents/codex/source-code/)
- [ ] 3.3 架构与能力 + 核心能力测试（R1-R5, A1-A5, E1-E5） → [architecture.md](agents/codex/architecture.md)、[core-capabilities.md](agents/codex/core-capabilities.md)
- [ ] 3.4 技术实现 → [tech-implementation.md](agents/codex/tech-implementation.md)
- [ ] 3.5 交互与安全 → [interaction-design.md](agents/codex/interaction-design.md)、[security-model.md](agents/codex/security-model.md)
- [ ] 3.6 可扩展性与总结 → [extensibility.md](agents/codex/extensibility.md)、[pros-cons.md](agents/codex/pros-cons.md)

---

## 阶段四：Hermes 深度分析

- [ ] 4.1 基础信息 → [README.md](agents/hermes/README.md)
- [ ] 4.2 源码分析 → [source-code/](agents/hermes/source-code/)
- [ ] 4.3 架构与能力 + 核心能力测试（R1-R5, A1-A5, E1-E5） → [architecture.md](agents/hermes/architecture.md)、[core-capabilities.md](agents/hermes/core-capabilities.md)
- [ ] 4.4 技术实现 → [tech-implementation.md](agents/hermes/tech-implementation.md)
- [ ] 4.5 交互与安全 → [interaction-design.md](agents/hermes/interaction-design.md)、[security-model.md](agents/hermes/security-model.md)
- [ ] 4.6 可扩展性与总结 → [extensibility.md](agents/hermes/extensibility.md)、[pros-cons.md](agents/hermes/pros-cons.md)

---

## 阶段五：OpenClaw 深度分析

- [ ] 5.1 基础信息 → [README.md](agents/openclaw/README.md)
- [ ] 5.2 源码分析 → [source-code/](agents/openclaw/source-code/)
- [ ] 5.3 架构与能力 + 核心能力测试（R1-R5, A1-A5, E1-E5） → [architecture.md](agents/openclaw/architecture.md)、[core-capabilities.md](agents/openclaw/core-capabilities.md)
- [ ] 5.4 技术实现 → [tech-implementation.md](agents/openclaw/tech-implementation.md)
- [ ] 5.5 交互与安全 → [interaction-design.md](agents/openclaw/interaction-design.md)、[security-model.md](agents/openclaw/security-model.md)
- [ ] 5.6 可扩展性与总结 → [extensibility.md](agents/openclaw/extensibility.md)、[pros-cons.md](agents/openclaw/pros-cons.md)

---

## 阶段六：横向对比

> 四个 Agent 分析完成后，基于收集的数据进行横向对比。

- [ ] 架构设计对比 → [overview/architecture-comparison.md](overview/architecture-comparison.md)
- [ ] 核心能力对比（汇总 15 个任务得分） → [overview/capability-comparison.md](overview/capability-comparison.md)
- [ ] 技术实现对比 → [overview/tech-implementation-comparison.md](overview/tech-implementation-comparison.md)
- [ ] 交互体验对比 → [overview/interaction-comparison.md](overview/interaction-comparison.md)
- [ ] 安全机制对比 → [overview/security-comparison.md](overview/security-comparison.md)
- [ ] 可扩展性对比 → [overview/extensibility-comparison.md](overview/extensibility-comparison.md)
- [ ] 社区活跃度对比 → [overview/ecosystem-comparison.md](overview/ecosystem-comparison.md)

---

## 阶段七：总结输出

- [ ] 撰写综合评价与结论 → [overview/summary.md](overview/summary.md)
- [ ] 完善术语表 → [references/glossary.md](references/glossary.md)
- [ ] 最终审校与排版

---

## 进度记录

| 阶段 | 状态 | 备注 |
|------|------|------|
| 阶段一：基础准备 | 🔲 进行中 | |
| 阶段二：Claude Code | 🔲 未开始 | |
| 阶段三：Codex | 🔲 未开始 | |
| 阶段四：Hermes | 🔲 未开始 | |
| 阶段五：OpenClaw | 🔲 未开始 | |
| 阶段六：横向对比 | 🔲 未开始 | |
| 阶段七：总结输出 | 🔲 未开始 | |
