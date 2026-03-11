# 架构设计

本文档介绍 Symphony 的系统架构和模块关系。

## 系统架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Symphony Service                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌────────────────────────────────────────┐ │
│  │ WORKFLOW.md │───▶│          Workflow Loader               │ │
│  └─────────────┘    │  (解析 YAML Front Matter + 模板)        │ │
│                     └───────────────────┬────────────────────┘ │
│                                         │                       │
│                     ┌───────────────────▼────────────────────┐ │
│                     │           Config Layer                 │ │
│                     │  (类型化配置、默认值、环境变量解析)      │ │
│                     └───────────────────┬────────────────────┘ │
│                                         │                       │
│  ┌──────────────────────────────────────▼────────────────────┐ │
│  │                    Orchestrator (GenServer)                │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │ │
│  │  │ 轮询调度    │  │ 状态管理    │  │ 重试队列        │   │ │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘   │ │
│  └────────┬─────────────────────┬───────────────────┬───────┘ │
│           │                     │                   │         │
│  ┌────────▼────────┐  ┌────────▼────────┐ ┌───────▼───────┐  │
│  │  Tracker Client │  │  Agent Runner   │ │   Workspace   │  │
│  │  (Linear API)   │  │  (Codex 管理)   │ │   Manager     │  │
│  └────────┬────────┘  └────────┬────────┘ └───────────────┘  │
│           │                    │                              │
└───────────┼────────────────────┼──────────────────────────────┘
            │                    │
            ▼                    ▼
    ┌───────────────┐    ┌───────────────┐
    │ Linear GraphQL│    │  Codex Agent  │
    │     API       │    │  (Subprocess) │
    └───────────────┘    └───────────────┘
```

## 抽象层次

Symphony 采用分层架构设计：

```
┌─────────────────────────────────────────┐
│  1. Policy Layer (策略层)               │
│     WORKFLOW.md 中的提示词和工作流规则    │
├─────────────────────────────────────────┤
│  2. Configuration Layer (配置层)        │
│     类型化配置、默认值、环境变量         │
├─────────────────────────────────────────┤
│  3. Coordination Layer (协调层)         │
│     Orchestrator：轮询、调度、重试       │
├─────────────────────────────────────────┤
│  4. Execution Layer (执行层)            │
│     Workspace + Agent Runner            │
├─────────────────────────────────────────┤
│  5. Integration Layer (集成层)          │
│     Linear API 适配器                   │
├─────────────────────────────────────────┤
│  6. Observability Layer (可观测层)      │
│     日志、仪表盘、API                    │
└─────────────────────────────────────────┘
```

## 核心模块

### 1. Workflow Loader

**职责**：加载和解析 `WORKFLOW.md`

```
输入：WORKFLOW.md 文件路径
输出：{config: map, prompt_template: string}
```

**处理流程**：
1. 读取文件内容
2. 解析 YAML Front Matter（`---` 之间的内容）
3. 提取 Markdown Body 作为提示词模板
4. 验证配置有效性

### 2. Config Layer

**职责**：提供类型化的配置访问

```
主要配置组：
├── tracker（跟踪器）
│   ├── kind: linear
│   ├── api_key: $LINEAR_API_KEY
│   └── project_slug: "project-name"
├── polling（轮询）
│   └── interval_ms: 5000
├── workspace（工作空间）
│   └── root: ~/code/workspaces
├── hooks（钩子）
│   ├── after_create
│   ├── before_run
│   ├── after_run
│   └── before_remove
├── agent（代理）
│   ├── max_concurrent_agents: 10
│   └── max_turns: 20
└── codex（Codex 配置）
    ├── command: codex app-server
    ├── approval_policy: never
    └── thread_sandbox: workspace-write
```

### 3. Orchestrator

**职责**：中央调度和状态管理

**内部状态**：
```elixir
%State{
  poll_interval_ms: 5000,           # 轮询间隔
  max_concurrent_agents: 10,        # 最大并发数
  running: %{},                     # 正在运行的任务
  claimed: MapSet.new(),            # 已预约的任务
  retry_attempts: %{},              # 重试队列
  completed: MapSet.new(),          # 已完成的任务
  codex_totals: %{...}              # Token 统计
}
```

**调度循环**：
```
每个 Tick：
1. 刷新配置（支持热更新）
2. 协调正在运行的任务
   ├── 检测停滞
   └── 刷新 Issue 状态
3. 验证配置有效性
4. 获取候选任务
5. 按优先级排序
6. 调度到并发槽位
```

### 4. Agent Runner

**职责**：管理单个任务的执行

**执行流程**：
```
1. 准备工作空间
   ├── 创建目录（如不存在）
   └── 执行 after_create 钩子
2. 构建 Prompt
   └── 使用模板引擎渲染
3. 启动 Codex 进程
   ├── 初始化会话
   ├── 启动 Thread
   └── 启动 Turn
4. 流式处理事件
   ├── 更新 Token 统计
   ├── 处理审批请求
   └── 检测完成/失败
5. 清理和报告
```

### 5. Workspace Manager

**职责**：工作空间生命周期管理

**安全约束**：
- 路径必须在 `workspace.root` 下
- 工作空间名称必须经过清理
- 只在安全目录执行代理

### 6. Tracker Client

**职责**：与 Linear API 交互

**主要操作**：
- `fetch_candidate_issues()`：获取候选任务
- `fetch_issues_by_states()`：按状态查询
- `fetch_issue_states_by_ids()`：批量刷新状态

## 数据流

### 任务调度流程

```
┌─────────┐    ┌─────────────┐    ┌──────────────┐    ┌──────────┐
│ Linear  │───▶│ Orchestrator│───▶│ Agent Runner │───▶│ Codex    │
│ (Poll)  │    │ (Dispatch)  │    │ (Execute)    │    │ (Run)    │
└─────────┘    └─────────────┘    └──────────────┘    └──────────┘
     │               │                    │                 │
     │               │                    │                 │
     │               │   ┌────────────────┘                 │
     │               │   │ 事件回调                         │
     │               │   │ (token 更新、完成、失败)          │
     │               │   ▼                                 │
     │               │  ┌──────────────┐                   │
     │               │  │ Orchestrator │◀──────────────────┘
     │               │  │ (Update)     │
     │               │  └──────┬───────┘
     │               │         │
     │               │    成功 │ 失败
     │               │    ┌────┴────┐
     │               │    ▼         ▼
     │               │ 延续重试   指数退避重试
     │               │
     │    状态刷新   │
     └───────────────┘
```

### Codex 通信协议

```
┌─────────────────┐                  ┌─────────────────┐
│  Symphony       │                  │  Codex Agent    │
│  (Client)       │                  │  (Server)       │
└────────┬────────┘                  └────────┬────────┘
         │                                    │
         │  initialize request                │
         │───────────────────────────────────▶│
         │                                    │
         │  initialize response               │
         │◀───────────────────────────────────│
         │                                    │
         │  initialized notification          │
         │───────────────────────────────────▶│
         │                                    │
         │  thread/start request              │
         │───────────────────────────────────▶│
         │                                    │
         │  thread/start response (threadId)  │
         │◀───────────────────────────────────│
         │                                    │
         │  turn/start request                │
         │───────────────────────────────────▶│
         │                                    │
         │  stream events...                  │
         │◀───────────────────────────────────│
         │                                    │
         │  turn/completed                    │
         │◀───────────────────────────────────│
         │                                    │
```

## 并发控制

### 全局并发

```
可用槽位 = max(max_concurrent_agents - running_count, 0)
```

### 按状态并发

可以针对特定状态设置不同的并发限制：

```yaml
agent:
  max_concurrent_agents: 10
  max_concurrent_agents_by_state:
    "in progress": 5    # 同时最多 5 个任务处于执行中
    "merging": 2        # 同时最多 2 个任务处于合并中
```

## 可观测性

### 日志系统

**必需字段**：
- `issue_id`：问题 ID
- `issue_identifier`：问题标识符
- `session_id`：会话 ID（Codex 相关）

### HTTP API（可选）

启用方式：`--port 4000` 或 `server.port: 4000`

**端点**：
- `GET /`：实时仪表盘
- `GET /api/v1/state`：系统状态 JSON
- `GET /api/v1/<issue_identifier>`：特定问题状态
- `GET /api/v1/refresh`：触发配置刷新

## 扩展机制

### 客户端工具

Symphony 可向 Codex 会话注入自定义工具：

- `linear_graphql`：执行 Linear GraphQL 查询

### 技能系统

仓库中的 `.codex/skills/` 目录定义了可复用的工作流技能：

```
.codex/skills/
├── commit/    # 提交代码
├── push/      # 推送分支
├── pull/      # 拉取更新
├── land/      # 合并 PR
├── linear/    # Linear 操作
└── debug/     # 调试辅助
```
