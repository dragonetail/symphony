# Symphony 中文文档

## 项目简介

Symphony 是由 OpenAI 开发的**编码代理编排服务**，它将团队从"监督编码代理"转变为"管理工作"。

### 核心价值

传统的编码代理使用方式需要人工持续监督，而 Symphony 实现了：

- **自动化工作调度**：从 Linear 等问题跟踪器自动获取待办任务
- **隔离执行环境**：为每个任务创建独立的工作空间，避免冲突
- **持续自主运行**：代理自动完成任务直到需要人工审核
- **可观测性**：提供仪表盘和日志，方便调试和监控

### 工作流程概览

```
Linear 问题跟踪器
       ↓ (轮询获取任务)
    Orchestrator (调度器)
       ↓ (为每个问题创建)
    工作空间 (隔离环境)
       ↓ (启动)
    Codex Agent (编码代理)
       ↓ (执行完成后)
    Human Review (人工审核)
       ↓ (审核通过后)
    合并代码
```

## 快速开始

### 前置条件

1. **代码库准备**：确保你的代码库已采用 [Harness Engineering](https://openai.com/index/harness-engineering/) 最佳实践
2. **Linear API Token**：在 Linear 设置 → 安全与访问 → 个人 API 密钥中创建

### 安装运行

```bash
# 克隆仓库
git clone https://github.com/openai/symphony
cd symphony/elixir

# 安装依赖
mise trust
mise install
mise exec -- mix setup
mise exec -- mix build

# 设置环境变量
export LINEAR_API_KEY=your_linear_api_key

# 启动服务
mise exec -- ./bin/symphony ./WORKFLOW.md
```

### 最小配置示例

创建一个 `WORKFLOW.md` 文件：

```markdown
---
tracker:
  kind: linear
  project_slug: "your-project-slug"
workspace:
  root: ~/code/workspaces
hooks:
  after_create: |
    git clone git@github.com:your-org/your-repo.git .
agent:
  max_concurrent_agents: 10
  max_turns: 20
codex:
  command: codex app-server
---

You are working on a Linear issue {{ issue.identifier }}.

Title: {{ issue.title }}
Body: {{ issue.description }}
```

## 文档目录

| 文档 | 说明 |
|------|------|
| [核心概念](concepts.md) | 了解 Symphony 的核心概念和术语 |
| [架构设计](architecture.md) | 理解系统架构和模块关系 |
| [配置说明](configuration.md) | 详细的配置选项和示例 |
| [使用指南](usage.md) | 完整的使用流程和最佳实践 |

## 重要说明

> **警告**：Symphony 是一个工程预览版本，仅用于可信环境的测试评估。建议基于 `SPEC.md` 实现自己的加固版本。

## 许可证

本项目采用 [Apache License 2.0](../LICENSE) 许可证。
