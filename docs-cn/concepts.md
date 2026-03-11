# 核心概念

本文档介绍 Symphony 的核心概念和术语，帮助你理解系统的工作原理。

## 核心实体

### Issue（问题）

来自 Linear 等问题跟踪器的任务单元。每个 Issue 包含：

| 字段 | 说明 |
|------|------|
| `identifier` | 人类可读的标识符，如 `ABC-123` |
| `title` | 问题标题 |
| `description` | 问题描述 |
| `state` | 当前状态（Todo、In Progress 等） |
| `priority` | 优先级（数字越小优先级越高） |
| `labels` | 标签列表 |
| `blocked_by` | 阻塞此问题的其他问题 |

### Workspace（工作空间）

为每个 Issue 创建的**隔离目录**，用于：

- 存放代码副本
- 运行编码代理
- 执行测试和验证

**工作空间路径规则**：`<workspace.root>/<sanitized_issue_identifier>`

例如：`~/code/workspaces/ABC-123`

### Workflow（工作流）

定义在 `WORKFLOW.md` 文件中的**执行策略**，包含：

1. **YAML Front Matter**：配置参数（跟踪器、并发数、超时等）
2. **Markdown Body**：发送给编码代理的提示词模板

### Session（会话）

一次编码代理的执行过程。一个 Issue 可能经历多个 Session：

- **Thread**：与代理的对话线程
- **Turn**：线程中的单次交互
- **Continuation**：同一 Thread 中的后续 Turn

## 状态管理

### Issue 跟踪器状态

Linear 中的标准状态流转：

```
Backlog → Todo → In Progress → Human Review → Merging → Done
                    ↑              ↓
                 Rework ←─────────┘
```

| 状态 | 说明 |
|------|------|
| `Backlog` | 待定，不在当前工作流范围内 |
| `Todo` | 待处理，准备开始工作 |
| `In Progress` | 进行中，代理正在执行 |
| `Human Review` | 等待人工审核 |
| `Merging` | 正在合并 |
| `Rework` | 需要返工 |
| `Done` | 已完成（终态） |

### 内部调度状态

Symphony 内部的 Issue 状态：

| 状态 | 说明 |
|------|------|
| `Unclaimed` | 未被领取，可以调度 |
| `Claimed` | 已被预约，防止重复调度 |
| `Running` | 正在执行 |
| `RetryQueued` | 已安排重试 |
| `Released` | 已释放（完成或取消） |

## 核心组件

### Orchestrator（调度器）

系统的**中央控制器**，负责：

- 定期轮询 Linear 获取候选任务
- 管理并发执行
- 处理重试和错误恢复
- 协调各组件工作

### Agent Runner（代理运行器）

负责**执行单个任务**：

1. 创建/复用工作空间
2. 构建提示词
3. 启动 Codex 进程
4. 处理代理事件
5. 报告执行结果

### Workspace Manager（工作空间管理器）

管理工作空间的**生命周期**：

- 创建目录
- 执行钩子脚本
- 清理已完成的工作空间

### Tracker Client（跟踪器客户端）

与 Linear API 交互：

- 获取候选任务
- 刷新任务状态
- 查询终态任务

## 钩子系统

钩子是在特定时机执行的**自定义脚本**：

| 钩子 | 执行时机 | 失败影响 |
|------|----------|----------|
| `after_create` | 工作空间首次创建后 | 阻止工作空间创建 |
| `before_run` | 每次代理运行前 | 阻止当前运行 |
| `after_run` | 每次代理运行后 | 仅记录日志 |
| `before_remove` | 工作空间删除前 | 仅记录日志 |

## 重试机制

### 重试类型

1. **延续重试**：正常完成后，Issue 仍处于活跃状态
   - 延迟：1 秒
   - 目的：继续处理尚未完成的任务

2. **失败重试**：执行出错、超时、停滞
   - 延迟：指数退避（10s → 20s → 40s → ...）
   - 上限：默认 5 分钟

### 重试条件

- Issue 仍处于活跃状态
- 有可用的并发槽位
- 没有未完成的阻塞项

## 模板系统

WORKFLOW.md 中的 Markdown 部分是 **Liquid 兼容的模板**：

### 可用变量

```liquid
{{ issue.identifier }}    {# 问题标识符 #}
{{ issue.title }}         {# 问题标题 #}
{{ issue.description }}   {# 问题描述 #}
{{ issue.state }}         {# 当前状态 #}
{{ issue.url }}           {# 问题链接 #}
{{ issue.labels }}        {# 标签列表 #}
{{ attempt }}             {# 重试次数（首次为 null）#}
```

### 条件逻辑

```liquid
{% if attempt %}
这是第 {{ attempt }} 次重试。
{% endif %}
```

## 安全约束

### 工作空间隔离

1. **路径限制**：工作空间必须在配置的根目录下
2. **名称清理**：只允许 `[A-Za-z0-9._-]` 字符
3. **执行目录**：编码代理只能在工作空间内执行

### 沙箱策略

通过 Codex 配置控制代理权限：

- `read-only`：只读模式
- `workspace-write`：只能写入工作空间
- `danger-full-access`：完全访问（危险）
